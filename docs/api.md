# Run Log API

> Trạng thái: tài liệu thiết kế. API HTTP chưa được cài đặt trong phiên bản hiện tại.

## 1. Phạm vi

`run-log` hướng tới một execution platform bất đồng bộ cho workflow LLM. API nhận một yêu cầu chạy, đưa yêu cầu vào queue, cho phép xem trạng thái/kết quả, và cung cấp trace để giải thích các bước thực thi.

Phiên bản hiện tại của project chỉ có CLI `run-log` và chưa có dependency runtime. Không nên dùng các ví dụ dưới đây như endpoint đang hoạt động.

## 2. Quy ước chung

- Base URL dự kiến: `/api/v1`
- Content type: `application/json`
- ID là chuỗi ổn định, ví dụ `r_01J...` và `p_01J...`.
- Thời gian dùng ISO 8601 UTC, ví dụ `2026-10-08T10:15:30Z`.
- Client gửi `Idempotency-Key` khi tạo run. Cùng key trong cùng project phải trả về cùng một run.
- Lỗi dùng một schema thống nhất:

```json
{
  "error": {
    "code": "validation_error",
    "message": "input is invalid",
    "details": {"field": "input"},
    "request_id": "req_01J..."
  }
}
```

## 3. Endpoint dự kiến

### `POST /api/v1/projects/{project_id}/runs`

Tạo một run mới. Server chỉ xác nhận đã nhận job; worker mới thực hiện LLM.

Request:

```json
{
  "input": "Project A có bao nhiêu run lỗi trong 7 ngày qua?",
  "prompt_version": "v3",
  "model": "fake-llm",
  "metadata": {"source": "web"}
}
```

Response thành công: `202 Accepted`

```json
{
  "id": "r_01J...",
  "project_id": "project-a",
  "status": "queued",
  "created_at": "2026-10-08T10:15:30Z"
}
```

Lỗi thường gặp: `400 validation_error`, `404 project_not_found`, `409 idempotency_conflict`, `429 rate_limited`.

### `POST /api/v1/projects/{project_id}/conversations/{conversation_id}/runs`

Tạo một run mới trong một conversation đã có. Endpoint này dùng `messages` thay cho một chuỗi `input` đơn lẻ, giúp LLM nhận được đầy đủ ngữ cảnh của cuộc hội thoại nhiều lượt. Server lưu `conversation_id` cùng run và vẫn chỉ xác nhận đã nhận job; worker mới thực hiện LLM.

Request:

```json
{
  "messages": [
    {"role": "system", "content": "Bạn là trợ lý phân tích dữ liệu."},
    {"role": "user", "content": "Project A có bao nhiêu run lỗi?"},
    {"role": "assistant", "content": "Bạn muốn xem trong khoảng thời gian nào?"},
    {"role": "user", "content": "Trong 7 ngày qua."}
  ],
  "prompt_version": "v3",
  "model": "fake-llm",
  "metadata": {"source": "web"}
}
```

Quy tắc của `messages`:

- Phải là mảng không rỗng; mỗi phần tử có `role` và `content`.
- `role` chỉ nhận `system`, `user` hoặc `assistant`.
- Thứ tự phần tử là thứ tự lịch sử từ cũ đến mới; tin nhắn cuối thường là yêu cầu hiện tại của user.
- Kích thước tổng lịch sử phải nằm trong giới hạn context window của model.

Response thành công: `202 Accepted`

```json
{
  "id": "r_01J...",
  "project_id": "project-a",
  "conversation_id": "conv_01J...",
  "status": "queued",
  "created_at": "2026-10-08T10:15:30Z"
}
```

Lỗi thường gặp: `400 validation_error`, `404 project_not_found` hoặc `conversation_not_found`, `409 idempotency_conflict`, `413 context_too_large`, `429 rate_limited`.

### `GET /api/v1/runs/{run_id}`

Lấy trạng thái và kết quả hiện tại của run.

```json
{
  "id": "r_01J...",
  "project_id": "project-a",
  "status": "succeeded",
  "input": "Project A có bao nhiêu run lỗi trong 7 ngày qua?",
  "output": "Project A có 3 run lỗi trong 7 ngày qua.",
  "model": "fake-llm",
  "usage": {"input_tokens": 180, "output_tokens": 70, "total_tokens": 250},
  "latency_ms": 765,
  "created_at": "2026-10-08T10:15:30Z",
  "finished_at": "2026-10-08T10:15:31Z",
  "error": null
}
```

Trạng thái hợp lệ: `queued`, `running`, `succeeded`, `failed`, `cancelled`.

### `GET /api/v1/runs/{run_id}/trace?page=1&page_size=20`

Trả về execution metadata theo thứ tự thời gian và có phân trang để tránh trả về toàn bộ trace quá lớn trong một response. Mặc định mỗi trang có 20 span; trang đầu chứa các span gần nhất. Trace cần đủ để debug model, tool, retry và latency, nhưng không lưu chain-of-thought.

Query parameters:

- `page`: số trang bắt đầu từ `1`, mặc định `1`.
- `page_size`: số span mỗi trang, mặc định `20`, cho phép tối đa `100`.

Ví dụ request: `GET /api/v1/runs/r_01J.../trace?page=2&page_size=20`

```json
{
  "run_id": "r_01J...",
  "page": 1,
  "page_size": 20,
  "total": 3,
  "has_more": false,
  "spans": [
    {"type": "llm_call", "model": "fake-llm", "duration_ms": 420, "status": "succeeded"},
    {"type": "tool_call", "name": "get_project_stats", "duration_ms": 35, "status": "succeeded"},
    {"type": "llm_call", "model": "fake-llm", "duration_ms": 310, "status": "succeeded"}
  ]
}
```

Các trang được sắp xếp từ mới đến cũ; vì vậy `page=1` trả về tối đa 20 span gần nhất, còn `page=2` trả về 20 span tiếp theo. Khi `has_more` là `false`, client không cần gọi trang tiếp theo. Lỗi thường gặp: `400 invalid_pagination`, `404 run_not_found`.

### `GET /api/v1/runs/{run_id}/stream`

Stream kết quả của run bằng Server-Sent Events (SSE), để client nhận từng token ngay khi worker tạo ra thay vì chờ toàn bộ câu trả lời. Client nên gửi header `Accept: text/event-stream` và giữ kết nối mở đến khi nhận event `done` hoặc `error`.

Response: `200 OK`, `Content-Type: text/event-stream`

```text
event: start
data: {"run_id":"r_01J..."}

event: token
data: {"index":0,"text":"Project"}

event: token
data: {"index":1,"text":" A có"}

event: done
data: {"run_id":"r_01J...","status":"succeeded","usage":{"output_tokens":70}}
```

Các event hợp lệ:

- `start`: bắt đầu stream.
- `token`: một phần nội dung theo đúng thứ tự; `index` bắt đầu từ `0`.
- `done`: đã nhận đủ output; kết quả đầy đủ vẫn có thể lấy bằng `GET /api/v1/runs/{run_id}`.
- `error`: stream thất bại và chứa lỗi theo schema lỗi chung.

Nếu run chưa bắt đầu, server giữ kết nối và phát token khi có. Nếu run đã hoàn tất, server phát lại output theo từng phần rồi gửi `done`. Client cần xử lý việc mất kết nối bằng cách gọi lại trạng thái run; không nên tự nối tiếp token chỉ dựa vào một kết nối đã bị ngắt. Lỗi thường gặp: `404 run_not_found`, `409 stream_not_available`.

### `POST /api/v1/runs/{run_id}/cancel`

Yêu cầu huỷ run đang `queued` hoặc `running`. Response dự kiến: `202 Accepted`. Huỷ không đảm bảo dừng ngay một request đã gửi đến provider; worker phải kiểm tra trạng thái trước các bước tiếp theo.

### `GET /healthz` và `GET /readyz`

- `healthz`: process còn sống.
- `readyz`: service đã sẵn sàng nhận việc, bao gồm kiểm tra các dependency bắt buộc như database/queue.

Response tối thiểu:

```json
{"status": "ok"}
```

## 4. Quy tắc agent và tool

Worker thực hiện vòng lặp có giới hạn: gọi LLM → validate tool call → chạy tool trong allowlist → gửi kết quả lại cho LLM. Số bước tối đa dự kiến là 5.

Tool phải có schema input/output rõ ràng. Tool đọc dữ liệu được ưu tiên trước; chưa cho phép tool tự do ghi, xoá dữ liệu hoặc chạy code. Tool timeout và lỗi schema phải làm xuất hiện trong trace và được ánh xạ thành lỗi run phù hợp.

## 5. Bảo mật và vận hành

- Không trả API key trong response hoặc trace; redact secret trước khi ghi log.
- Giới hạn kích thước input/output và timeout cho mỗi run.
- Phân quyền theo project trước khi đọc run hoặc trace.
- Không đưa nội dung nhạy cảm vào metadata nếu không cần thiết.
- Dùng `request_id` để liên kết HTTP log với run/trace.

## 6. Tiêu chí hoàn thiện

API chỉ được xem là hoàn thiện khi có schema validation, idempotency, queue/worker, lưu trạng thái chuyển tiếp hợp lệ, timeout/retry có giới hạn, test cho các lỗi chính và tài liệu cập nhật theo implementation thực tế.
