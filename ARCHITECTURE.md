# L3A Architecture Record

Team phải cập nhật tài liệu này cùng source. Mục tiêu là mô tả quyết định có thể kiểm chứng, không ghi prompt bí mật hoặc chain-of-thought.

## 1. System overview

Vẽ hoặc mô tả luồng từ `inputs/<case_id>.json` đến MCP calls, specialist agents, verifier, output và trace.

```text
                                inputs/<case_id>.json
                                          |
                                          | handoff
                                          ▼
               ┌────────────────────────────────────────────────────────┐
               │                     Coordinator                        │
               └──────────────────────────┬─────────────────────────────┘
                                          │ handoff
                                          ▼
                               ┌─────────────────────┐
                               │   Order Specialist  │ ──► get_order, get_order_items, get_sellers
                               └──────────┬──────────┘
                                          │ handoff
                                          ▼
                               ┌─────────────────────┐
                               │ Payment Specialist  │ ──► get_order_payments, get_refund_timeline*
                               └──────────┬──────────┘
                                          │ handoff
                                          ▼
                               ┌─────────────────────┐
                               │ Shipment Specialist │ ──► get_shipment_summary
                               └──────────┬──────────┘
                                          │ handoff
                                          ▼
                               ┌─────────────────────┐
                               │  Policy Specialist  │ ──► get_policy
                               └──────────┬──────────┘
                                          │ handoff
                                          ▼
                               ┌─────────────────────┐
                               │  Verifier/Resolver  │ ──► Độc lập đối soát & thẩm định Claims
                               └──────────┬──────────┘
                                          │
                                          ▼
                         outputs/<case_id>.json + traces/trace.jsonl
```

### Luồng xử lý chi tiết:
1. **Input Case**: Đọc `inputs/<case_id>.json`, giải mã yêu cầu khách hàng (`customer_request`), danh sách khiếu nại (`claims`) và phiên bản chính sách áp dụng (`policy_version`).
2. **Coordinator Initiation**: Tiếp nhận case, phát sự kiện trace `case_received`, phân công nhiệm vụ đầu tiên `task_assigned` cho `Order Specialist`.
3. **Specialist Data Harvesting**: Lần lượt từng Specialist thực thi nhiệm vụ theo thẩm quyền, tương tác với MCP Gateway để lấy bằng chứng được ký số (`evidence_ref`) và chuyển giao (`handoff`) cho Specialist kế tiếp.
4. **Verifier & Resolver**: Tổng hợp toàn bộ bằng chứng, đối soát trạng thái thực tế với lời khai của khách hàng, trích xuất chính sách đền bù và xác định trách nhiệm của các bên.
5. **Output & Trace Finalization**: Tạo file đầu ra tuân thủ chuẩn `outputs/<case_id>.json` và ghi nhận sự kiện `case_finalized`.

---

## 2. Agent Ownership

Để bảo đảm an toàn dữ liệu và tối ưu điểm số hiệu năng (Tool Efficiency Gate), mỗi Agent được phân định ranh giới trách nhiệm và quyền hạn gọi MCP Tool nghiêm ngặt:

| Actor | Input | Trách nhiệm | MCP Tools được phép | Output / Handoff |
| :--- | :--- | :--- | :--- | :--- |
| **Coordinator** | `case` object từ `inputs/<case_id>.json` | Tiếp nhận case, kiểm soát vòng đời luồng, điều phối phân công công việc | *Không trực tiếp gọi MCP tool* | Phân công `task_assigned` cho `order_specialist`; hoàn tất `case_finalized` |
| **Order Specialist** | `case_id`, `claimed_order_id` | Khai thác hồ sơ đơn hàng thực tế, chi tiết danh mục hàng hóa và người bán | `get_order`, `get_order_items`, `get_sellers` | `order_data`, `items_data`, `sellers_data`, evidence refs $\rightarrow$ Handoff sang `payment_specialist` |
| **Payment Specialist** | `case_id`, `order_id`, Customer context | Xác minh giao dịch thanh toán; đánh giá nhu cầu hoàn tiền có điều kiện | `get_order_payments`, `get_refund_timeline` *(chỉ gọi khi có claim hoàn tiền)* | `payments_data`, `refund_data`, evidence refs $\rightarrow$ Handoff sang `shipment_specialist` |
| **Shipment Specialist** | `case_id`, `order_id` | Kiểm tra tiến độ vận chuyển, trạng thái giao nhận và ngày giao đối tác vận chuyển | `get_shipment_summary` | `shipment_data`, evidence refs $\rightarrow$ Handoff sang `policy_specialist` |
| **Policy Specialist** | `case_id`, `policy_version` | Tra cứu bảng quy tắc xử lý đền bù theo đúng phiên bản chính sách | `get_policy` | `policy_rules`, evidence refs $\rightarrow$ Handoff sang `verifier` |
| **Verifier (Resolver)** | Tổng hợp toàn bộ Context & Evidence từ các Specialists | Đối soát xung đột dữ liệu, đánh giá từng khiếu nại (Claims), quyết định giải pháp tài chính và nguyên nhân gốc | *Không gọi MCP tool* (Chỉ suy luận dựa trên bằng chứng đã có) | Final JSON Output tuân thủ schema `outputs/<case_id>.json` |

> **Nguyên tắc phân quyền công cụ**: Tuyệt đối không cho phép mọi Agent có quyền truy vấn toàn bộ MCP Tools. Đặc biệt, `get_refund_timeline` chỉ được kích hoạt bởi `Payment Specialist` khi có căn cứ khiếu nại hoàn tiền rõ ràng.

---

## 3. A2A Protocol

Giao thức giao tiếp giữa các tác nhân (Agent-to-Agent) tuân thủ các nguyên tắc:

- **Correlation theo `case_id`**: Mọi thông điệp và sự kiện phát ra bắt buộc phải mang đúng `case_id` của phiên xử lý hiện hành. Tuyệt đối không để rò rỉ dữ liệu hoặc dùng chéo `evidence_ref` giữa các case.
- **Tuyến tính một chiều (DAG - Directed Acyclic Graph)**: Luồng chuyển giao nhiệm vụ tuân thủ tuần tự cố định: `Coordinator` $\rightarrow$ `Order Specialist` $\rightarrow$ `Payment Specialist` $\rightarrow$ `Shipment Specialist` $\rightarrow$ `Policy Specialist` $\rightarrow$ `Verifier`. Thiết kế này loại trừ hoàn toàn nguy cơ lặp vô hạn (Infinite Loop) hoặc deadlock.
- **Observable Tracing**: Chỉ ghi nhận các sự kiện quan sát được phục vụ giám sát và đánh giá benchmark theo schema `day09-trace-event-v1`:
  - `case_received`, `task_assigned`: Tiếp nhận và phân công.
  - `tool_result_consumed`: Ghi nhận Agent đã tiêu thụ thành công một bằng chứng (`evidence_ref`).
  - `handoff`: Chuyển giao quyền điều khiển giữa 2 tác nhân.
  - `policy_decided`: Quyết định mã chính sách và mức tiền áp dụng.
  - `verification_completed`: Hoàn tất đối soát.
  - `case_finalized`: Đóng case.
  - *Tuyệt đối không trace nội dung prompt thô hoặc suy luận nội bộ.*
- **Timeout Management**: Cấu hình MCP Client timeout tổng 300 giây, kết nối 30 giây để xử lý an toàn các truy vấn mạng.

---

## 4. Evidence Lifecycle

Vòng đời của bằng chứng (Evidence) bảo đảm tính toàn vẹn và bất biến từ MCP Server đến kết quả báo cáo:

1. **Cách validate MCP server**:
   - Khi nhận phản hồi từ MCP Server, `EvidenceGateway` tự động kiểm tra định dạng qua hợp đồng `contracts/schemas/mcp-evidence-response-v1.schema.json`.
   - Bắt buộc phải có đủ: `schema_version`, `evidence_ref`, `result_hash`, `domain`, và `data`.
2. **Lưu `evidence_ref`**:
   - Giữ nguyên vẹn chuỗi định danh `evidence_ref` (dạng `ev_[A-Za-z0-9_-]{20,96}`) và hàm băm `result_hash`. Tuyệt đối không tự sinh hay sửa đổi mã evidence.
3. **Ánh xạ evidence vào case**: 
   - Mỗi kết luận khiếu nại (`claim_assessments`) phải có các `evidence_refs` hỗ trợ trực tiếp cho phán quyết đó.
   - Các evidence được tập hợp vào danh sách chung ở cấp root của output để phục vụ kiểm toán tự động.
4. **Emit sự kiện `tool_result_consumed`**:
   - Ngay sau khi một Specialist gọi MCP Tool thành công và nhận về bằng chứng hợp lệ, Agent bắt buộc phát sự kiện trace `tool_result_consumed` qua `trace.emit(...)`.
   - Sự kiện chứa đầy đủ: `case_id`, `event_type="tool_result_consumed"`, `actor` (Specialist đang thực thi), `tool_name` (tên công cụ vừa tiêu thụ), và danh sách `evidence_refs=[evidence_ref]`.
   - **Quy tắc bất biến**: Chỉ emit sự kiện khi tool trả về bằng chứng thực tế. Nếu tool bị lỗi, Not Found hoặc không được gọi (`call_safe` trả về `None`), tuyệt đối không phát sự kiện này để bảo đảm tính trung thực của trace.
5. **Evidence không được tái sử dụng giữa các case**:
   - Evidence chỉ hợp lệ cho duy nhất `case_id` được cấp phát. Mỗi case mới bắt đầu bằng danh sách `evidence_refs = []` hoàn toàn độc lập.

---

## 5. Failure Policy

Chính sách xử lý sự cố được thiết kế theo cơ chế **Graceful Degradation** và **Negative Evidence Exploitation**:

| Loại sự cố | Có Retry không? | Phương án Fallback | Trace Event / Ghi nhận |
| :--- | :--- | :--- | :--- |
| **MCP Tool Timeout / Network Error** | **Không** retry mù quáng để tránh cạn kiệt tài nguyên | Sử dụng hàm `call_safe()` trả về `None`, gán giá trị mặc định an toàn (`{}`, `[]`) | Không phát `tool_result_consumed` |
| **Record Not Found / Tool Failure** | **Không** retry | Ghi nhận dữ liệu thực tế không tồn tại. Coi đây là **Negative Evidence** (ví dụ: không tìm thấy refund timeline chứng minh đơn chưa từng được hoàn tiền) | Không phát `tool_result_consumed`; Verifier lấy căn cứ phán quyết `unsupported` |
| **Xung đột dữ liệu (Source Conflict)** | **Không** retry | Ưu tiên bản ghi chính thức từ MCP (`mcp_get_order`) hơn so với khiếu nại của khách (`customer_claim`) | Ghi nhận vào `data_conflicts` với mã `VERIFIED_AUTHORITATIVE_ORDER_RECORD` |
| **Thiếu hụt dữ liệu nghiêm trọng** | **Không** retry | Gán `primary_issue = "unsupported_claim"`, `case_status = "no_action"`, `refund = 0.0` | `policy_decided` với mã `UNSUPPORTED_CLAIM` |

> **Nguyên tắc cốt lõi**: Không chuyển đổi sự thiếu hụt bằng chứng thành dữ liệu suy đoán chủ quan. Lỗi "Not Found" từ MCP Tool được xem là một thuộc tính thông tin có giá trị nghiệp vụ, không phải là lỗi hệ thống cần retry.

---

## 6. Verification Invariants

Trước khi đóng gói output và kết thúc case, hệ thống luôn bảo đảm các bất biến (Invariants) sau:

1. **Schema Compliance**: Dữ liệu bắt buộc hợp lệ 100% với JSON Schema Draft 2020-12 theo `day09-l3a-output-v2.schema.json`.
2. **Entity Cardinality Bounds**:
   - `order_ids`, `item_ids`, `seller_ids`, `payment_references`, `shipment_ids` không vượt quá 20 phần tử mỗi mảng.
   - Toàn bộ danh sách thực thể và bằng chứng phải có tính duy nhất (`uniqueItems: true`).
3. **Evidence Linkage Consistency**:
   - Toàn bộ `evidence_ref` trong từng `claim_assessment` phải là tập con của mảng `evidence_refs` tổng.
4. **Financial Reconciliation Invariant**:
   - `financial_resolution.recommended_refund_brl` luôn bằng chính xác tổng `amount_brl` của tất cả các dòng trong `refund_lines`.
   - Nếu không có dòng hoàn tiền nào, giá trị bắt buộc là `0.0`.
5. **Confidence Range**:
   - Chỉ số tin cậy `confidence` của `assessment` và từng `claim_assessment` luôn nằm nghiêm ngặt trong khoảng $[0.0, 1.0]$.
6. **Responsibility & Action Consistency**:
   - Nếu `party_type == "seller"` và có `seller_ids`, gán đúng `party_id` của seller liên đới.

---

## 7. Reproducibility

Hệ thống bảo đảm tính tái lập kết quả thử nghiệm 100% trên môi trường chấm thi:

- **Môi trường thực thi**: Python 3.11+, quản lý gói qua Virtualenv (`.venv`).
- **Thư viện nòng cốt**:
  - `mcp`: Giao thức Model Context Protocol Client (Streamable HTTP).
  - `httpx2`: HTTP client bất đồng bộ quản lý kết nối và timeout.
  - `jsonschema` & `referencing`: Xác thực hợp đồng dữ liệu chuẩn Draft 2020-12.
- **Quy trình chạy và tái lập**:
  ```bash
  # 1. Chạy toàn bộ quy trình xử lý 100 cases
  day09 run

  # 2. Xác thực hợp lệ của Outputs và Trace Events
  day09 validate

  # 3. Đóng gói bài nộp
  day09 package --output dist/submission.zip
  ```
- **Bảo mật**: Mọi thông tin nhạy cảm (như `COMPETITION_TEAM_API_KEY`) được lưu trữ tại file cục bộ `.env` và bị loại trừ qua `.gitignore`. Quét tự động bằng regex ngăn chặn rò rỉ secret key vào artifacts trước khi đóng gói nộp bài.
