# Suy Ngẫm Cá Nhân & Kế Hoạch Đồ Án — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Vũ Quốc Anh
**Mã học viên:** 2A202601080
**Khóa học:** AICB-K34 · Track 3: GraphRAG
**Ngày thực hiện:** 2026-08-19

---

## 1. Mapping Bài Giảng vào Code

| Khái niệm trong bài giảng | Module tương ứng | Hàm / Khối code cụ thể | Quan sát thực tế & Đánh giá |
|---|---|---|---|
| Conservative Coreference | Module 1 | `resolve_coref_batch()`, `run_coref()` | Hoạt động đúng phần lớn (400 chunk, 0 batch fail) nhưng phát hiện 1 lỗi mới: model có thể **xóa hẳn câu** thay vì resolve khi antecedent mơ hồ (chunk `ea76b71e88232361f308::c0000`) — "conservative" chưa bao gồm "không được xóa fact", cần bổ sung rule. |
| Schema & Allowlist Guard | Module 2 | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS` | Giữ 100% relation trong 119 triples nằm trong whitelist — extraction không tự sáng tạo loại quan hệ lạ, `invalid_provenance_edges = 0` sau ingest. |
| Bulk Cypher Ingestion (`UNWIND`) | Module 2 | `bulk_insert_nodes()`, `bulk_insert_edges()` | Chạy idempotent nhờ `MERGE`, batch 1000 — 193 nodes / 118 edges ingest ổn định, chạy lại không nhân bản. |
| Entity Resolution & Union-Find | Module 3 | `build_resolution_map()`, `UF` | Dữ liệu tự nhiên ở quy mô lab chỉ sinh **1 candidate pair** (`X90A`/`X90`, 0.92, MERGE_VECTOR) — quá ít để quan sát guard tự nhiên. Test có chủ đích cho thấy guard chặn tốt pattern "brand vs sản phẩm con" nhưng **bỏ lọt** pattern "2 người trùng họ" (`Sam Altman`/`Steve Altman` similarity 0.824 vẫn bị MERGE) — giới hạn thật của thiết kế hiện tại. |
| Super-node Degree Cap | Module 4 | `retrieve_graph_context()` | Đúng thiết kế (`hop >= max_hops` guard), nhưng ở quy mô 1.500 bài báo max degree chỉ 5 nên `SUPER_NODE_DEGREE=100` không bao giờ kích hoạt tự nhiên — chỉ chứng minh được bằng cách hạ ngưỡng demo tạm thời. |
| LLM-as-a-Judge Evaluation | Module 5 | `judge_answer()`, `judge_json()` | Chạy được và cho điểm hợp lý (5/5 cho 2 câu factoid/multi-hop test), nhưng pipeline khá mong manh trước lỗi API bên ngoài (xem mục 2 bên dưới) — phải thêm resilience wrapper mới chạy hết golden set mà không crash toàn bộ. |

---

## 2. Quá Trình Debugging & Bài Học

- **Lỗi kỹ thuật phức tạp nhất gặp phải:** `run_evaluation()` (cell 4.3) crash hoàn toàn giữa chừng khi 1 câu golden gặp lỗi Groq — 2 kiểu lỗi khác nhau: (1) `json_validate_failed` với `failed_generation` rỗng, xảy ra lặp lại ổn định trên cùng 1 câu hỏi dù `temperature=0.0` (không phải do prompt riêng của câu đó); (2) `429 rate_limit_exceeded` khi chạm giới hạn Groq free tier (TPD 200.000 token/ngày) giữa batch eval. Vì `run_evaluation` gốc không bắt exception per-question, 1 câu lỗi làm mất toàn bộ kết quả các câu đã chạy trước đó (dù có `CHECKPOINT` ghi ra đĩa, hàm không đọc lại để resume).
- **Cách xử lý thành công:** Bọc thân vòng lặp bằng hàm `_run_one_question()` — retry 1 lần sau `sleep(3)`, nếu vẫn lỗi thì ghi lại row với `answer="ERROR: ..."` và điểm judge = `None` thay vì raise, để vòng `for` tiếp tục sang câu kế tiếp. Nhờ vậy full pipeline chạy hết 6 câu, xuất đủ 4 file CSV thay vì dừng giữa chừng.
- **Bài học rút ra:** Với pipeline gọi LLM API bên ngoài (rate limit, lỗi JSON mode không đoán trước được), **per-item resilience quan trọng hơn per-batch retry** — 1 điểm lỗi không nên làm hỏng toàn bộ batch đã thành công trước đó. Cũng nên tách rõ "lỗi hạ tầng" (rate limit, network) khỏi "điểm judge thật = 1" để không hiểu nhầm dữ liệu khi đọc CSV kết quả sau này.

---

## 3. Kế Hoạch Áp Dụng vào Đồ Án Thực Tế

- **Tên đồ án / dự án:** [...]
- **Đặc thù bài toán & lý do chọn giải pháp:** [Đồ án của bạn có cần GraphRAG không, hay Flat RAG / Hybrid RAG là đủ?]
- **Cấu trúc Node & Relation dự kiến (nếu áp dụng GraphRAG):**
  - Nodes: [...]
  - Relations: [...]
- **Chiến lược xử lý Entity Resolution & Super-node trong bài toán cụ thể:** [...]

---

## 4. Tự Đánh Giá

| Tiêu chí | Điểm tự chấm (1–5) | Ghi chú |
|---|---|---|
| Mức độ hiểu bài giảng GraphRAG | | |
| Khả năng kiểm soát AI Coding Agent | | |
| Chất lượng đồ thị tri thức xây dựng | | |
| Khả năng phân tích và debug hệ thống | | |
