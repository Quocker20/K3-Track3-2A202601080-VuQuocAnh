# Báo Cáo Thực Hành & Thuyết Minh Kỹ Thuật — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Vũ Quốc Anh (2A202601080)
**Khóa học:** AICB-K34 · Track 3: GraphRAG  
**Ngày thực hiện:** 2026-08-19

---

## 📌 PHẦN 1: THUYẾT MINH KỸ THUẬT & PHÂN TÍCH CA LỖI

### 1. Coreference Resolution (Phân giải đại từ)
> **Tình huống thực tế:** Nêu ít nhất 1 tình huống cụ thể trong dữ liệu HackerNoon mà cơ chế Coreference Resolution phân giải sai hoặc gặp khó khăn. Hậu quả của nó đối với Knowledge Graph là gì?

*Trả lời:*
- **Ví dụ từ dữ liệu:** `chunk_id = ea76b71e88232361f308::c0000` (`data/cache/coref_samples.csv`, dòng 4)
  - Gốc: `"...with limited technology-based services. We expect this to continue for the rest of today. We're aiming to open Reading Rooms at St Pancras tomorrow..."`
  - Sau resolve: `"...with limited technology-based services. We're aiming to open Reading Rooms at St Pancras tomorrow..."`
- **Hiện tượng:** Không phải nhầm đối tượng — model **xóa hẳn câu** `"We expect this to continue for the rest of today."` vì không đủ context (không có tên tổ chức trong chunk) để resolve `We` an toàn, nên chọn bỏ câu thay vì giữ nguyên.
- **Hậu quả đối với Graph:** Mất 1 fact trước khi tới NER+RE (thời hạn gián đoạn dịch vụ) — không tạo False Edge nhưng gây **silent information loss**, khó phát hiện vì không có log lỗi như `COREF_BATCH_FAILED`.

---

### 2. Entity Resolution Threshold & Lexical Guard
> **Ngưỡng & Cơ chế Guard:** Bạn chọn ngưỡng cosine similarity là bao nhiêu cho vector matching? Trích dẫn 1 cặp thực thể có độ tương đồng vector cao ($> 0.85$) nhưng bị Lexical Guard chặn không cho gộp (Reject) và giải thích lý do.

*Trả lời:*
- **Ngưỡng cosine similarity:** `threshold = 0.90` (cell 2.2, `build_resolution_map`)
- **Dữ liệu tự nhiên chỉ sinh 1 candidate pair** (`X90A` vs `X90`, 0.920579, MERGE_VECTOR) — không đủ để quan sát REJECT_GUARD tự nhiên. Cell 5.1 chạy `merge_guard()` trên các cặp có chủ đích (không phải dữ liệu tự nhiên):

  | Left | Right | Similarity | Decision |
  |---|---|---|---|
  | Apple | Apple Music | 0.670162 | REJECT_GUARD |
  | Amazon | Amazon Web Services | 0.646989 | REJECT_GUARD |
  | Meta | Meta Quest | 0.625470 | REJECT_GUARD |
  | Apple | Apple Watch | 0.609479 | REJECT_GUARD |

- **Lý do chặn:** cả 4 cặp là quan hệ thương hiệu-mẹ vs sản phẩm/chi nhánh con (substring lexical match) — bản thể khác nhau dù embedding gần. **Giới hạn phát hiện được:** không cặp REJECT_GUARD nào vượt 0.85 trong test này; ngược lại `Sam Altman` vs `Steve Altman` (0.824) lại bị **MERGE_VECTOR** — guard chưa có rule cho pattern "2 người trùng họ khác tên đệm".

---

### 3. Đồ thị & Super-node Mitigation
> **Đặc trưng đồ thị & Cắt tỉa cạnh:** Top 3 thực thể có bậc (degree) cao nhất trong đồ thị là gì? Việc ưu tiên lấy $N$ cạnh ($N=50$) có `published_date` mới nhất tại các Super-node mang lại ưu điểm gì và có rủi ro tiềm ẩn nào?

*Trả lời:*
- **Top 3 Super-nodes:**

| Hạng | Tên thực thể | Loại thực thể (Type) | Bậc kết nối (Degree) |
|------|--------------|---------------------|----------------------|
| 1 | Microsoft | Company | 5 |
| 2 | Amazon Web Services | Company | 4 |
| 3 | ServiceNow | Company | 4 |

*(Nguồn: `graph_checks()` → `top_degree_df`, 193 nodes / 118 edges. Chưa node nào chạm ngưỡng production `SUPER_NODE_DEGREE=100` ở quy mô lab — cell 5.1 demo hạ ngưỡng xuống 2 trên `Microsoft` để chứng minh cơ chế cắt tỉa.)*

- **Ưu điểm & Rủi ro của Temporal Mitigation:**
  - *Ưu điểm:* Chặn bùng nổ context/token khi 1 node có rất nhiều cạnh, ưu tiên thông tin gần thời điểm hỏi nhất — hợp câu hỏi kiểu tin tức mới nhất.
  - *Rủi ro:* Cạnh liên quan sự kiện lịch sử xa của super-node có thể bị loại khỏi 50 cạnh mới nhất dù vẫn còn trong graph, khiến GraphRAG báo "không tìm thấy" thay vì trả lời đúng.

---

### 4. So sánh Thực nghiệm (Flat RAG vs GraphRAG)

#### Bảng tổng hợp Benchmark (LLM-as-a-Judge):

| Tiêu chí đánh giá | Flat RAG | GraphRAG | Độ chênh lệch ($\Delta$) | Nhận xét phân tích |
|-------------------|----------|----------|--------------------------|-------------------|
| **Comprehensiveness (1–5)** | | | | |
| **Faithfulness (1–5)** | | | | |
| **Multi-hop Reasoning (1–5)** | | | | |
| **Latency trung bình (s)** | | | | |
| **Token usage trung bình** | | | | |

#### Phân tích 2 Ca lỗi Điển hình:
1. **Ca lỗi Flat RAG thất bại (GraphRAG thành công):**
   - *Question ID & Câu hỏi:* 
   - *Tại sao Flat RAG thất bại?* [Ví dụ: Vector search không kết nối được 2 chunks chứa thông tin rời rạc...]
   - *GraphRAG đã giải quyết như thế nào?* [Ví dụ: Graph traversal qua cạnh A -> B -> C...]
2. **Ca lỗi GraphRAG thất bại (hoặc cả hai cùng sai):**
   - *Question ID & Câu hỏi:* 
   - *Nguyên nhân:* [Ví dụ: Thiếu seed entity, missing edge trong bước extraction, hoặc super-node cap cắt mất cạnh...]
   - *Đề xuất khắc phục:* [...]

---

### 5. Đánh đổi (Trade-offs) & Kiểm soát AI Coding Agent
> **Trade-offs, Agent Control & Scale 350MB:** 
> - So sánh sự đánh đổi giữa GraphRAG vs Flat RAG về Latency, Token và Indexing Overhead.
> - Trong lúc làm bài, AI Coding Agent từng đề xuất điều gì mà bạn **từ chối áp dụng**? Tại sao?
> - Nếu scale lên toàn bộ 350MB (~100,000 bài báo), bottleneck đầu tiên ở đâu và giải pháp xử lý là gì?

*Trả lời:*
- **Đánh đổi Quality vs Cost vs Latency:** GraphRAG tốn thêm 1 lượt LLM call (seed extraction) + truy vấn Neo4j (BFS 2-hop) trước khi generate, nên latency/token cao hơn Flat RAG (chỉ embed + FAISS search + generate). Đổi lại có provenance theo cạnh (`chunk=`, `evidence=`) và multi-hop reasoning mà similarity-only search không làm được. *(Số liệu định lượng: điền ở mục 4 sau khi có eval sạch trên golden set thật.)*
- **Quyết định từ chối AI Coding Agent:** Khi bàn near-dedup (Challenge A), Agent đề xuất cách đơn giản nhất trước: pairwise cosine similarity giữa toàn bộ chunk embedding. Từ chối vì (1) đề bài cấm rõ ràng cách này, (2) ở `LIMIT_ROWS=8000` sẽ ra ~64 triệu cặp so sánh — không scale lên 350MB/~100.000 bài. Yêu cầu đổi sang MinHash/LSH hoặc embedding + FAISS ANN với blocking (gần tuyến tính thay vì O(N²)).
  Một lần khác, khi viết `run_extraction()`, Agent đề xuất bỏ `response_format=json_object` khỏi lời gọi Groq để "tiết kiệm token", vì cho rằng prompt đã yêu cầu strict JSON là đủ. Từ chối vì `parse_json_object()` chỉ tìm `{`...`}` đầu/cuối rồi `json.loads` thẳng — không ép JSON mode thì model dễ chèn text dẫn/markdown fence làm parser vỡ hàng loạt (400 chunk/lần), rủi ro mất cả batch không tương xứng với vài chục token tiết kiệm được.
- **Giải pháp scale 350MB:** Bottleneck chính là LLM API calls tuần tự (coref + extraction) — ở 1.500 chunk đã cần ~180 lời gọi Groq và đã thực sự chạm giới hạn TPD 200.000/ngày khi chạy eval batch nhỏ trong lab này. Ở ~100.000 bài báo cần: (1) async/concurrent batching thay vòng `for` tuần tự; (2) near-dedup MinHash/LSH trước khi extract (không pairwise cosine O(N²) — đúng ràng buộc Challenge A) để giảm số chunk phải gọi LLM; (3) Entity Resolution dùng blocking theo `type` + ANN (FAISS/HNSW) thay vì so mọi cặp; (4) bulk insert Neo4j giữ nguyên `UNWIND` batch 1000, không phải bottleneck.

---

## 📌 PHẦN 2: SUY NGẪM & KẾ HOẠCH ĐỒ ÁN (Reflection & Action Plan)

### 1. Mapping Bài giảng vào Code
| Khái niệm trong bài giảng | Module tương ứng | Hàm / Khối code cụ thể | Quan sát thực tế & Đánh giá |
|--------------------------|------------------|------------------------|-----------------------------|
| **Conservative Coreference** | Module 1 | `resolve_coref_batch()` | Hoạt động đúng phần lớn (400 chunk, 0 batch fail), nhưng phát hiện lỗi mới: model có thể **xóa hẳn câu** thay vì resolve khi antecedent mơ hồ (chunk `ea76b71e88232361f308::c0000`) — "conservative" chưa bao gồm "không được xóa fact". |
| **Schema & Allowlist Guard** | Module 2 | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS` | 100% relation trong 119 triples nằm trong whitelist, `invalid_provenance_edges = 0` sau ingest. |
| **Bulk Cypher Ingestion** | Module 2 | `bulk_insert_nodes()`, `bulk_insert_edges()` | Idempotent nhờ `MERGE`, batch 1000 — 193 nodes / 118 edges, chạy lại không nhân bản. |
| **Entity Resolution & Union-Find** | Module 3 | `build_resolution_map()`, `UF` | Dữ liệu tự nhiên chỉ sinh 1 candidate pair (quá ít để đánh giá guard tự nhiên). Test có chủ đích cho thấy guard chặn tốt "brand vs sản phẩm con" nhưng bỏ lọt "2 người trùng họ" (`Sam Altman`/`Steve Altman` 0.824 vẫn MERGE). |
| **Super-node Degree Cap** | Module 4 | `retrieve_graph_context()` | Đúng thiết kế, nhưng max degree thật của graph (5) thấp hơn nhiều ngưỡng production (100) nên chỉ chứng minh được bằng demo hạ ngưỡng tạm thời. |
| **LLM-as-a-Judge Evaluation** | Module 5 | `judge_answer()` | Chạy được, điểm hợp lý (5/5 cho câu factoid/multi-hop test), nhưng pipeline mong manh trước lỗi API ngoài — phải thêm resilience wrapper mới chạy hết golden set không crash. |

---

### 2. Quá trình Debugging & Bài học
- **Lỗi kỹ thuật phức tạp nhất gặp phải:** `run_evaluation()` crash hoàn toàn giữa chừng khi 1 câu golden gặp lỗi Groq — 2 kiểu: (1) `json_validate_failed` với `failed_generation` rỗng, lặp lại ổn định trên cùng câu hỏi dù `temperature=0.0`; (2) `429 rate_limit_exceeded` khi chạm TPD 200.000 token/ngày giữa batch eval. Hàm gốc không bắt exception per-question nên 1 câu lỗi làm mất toàn bộ kết quả đã chạy trước đó.
- **Cách bạn đã xử lý thành công:** Bọc thân vòng lặp bằng `_run_one_question()` — retry 1 lần sau `sleep(3)`, nếu vẫn lỗi thì ghi row `answer="ERROR: ..."` thay vì raise, để vòng `for` tiếp tục. Nhờ vậy pipeline chạy hết cả golden set, xuất đủ 4 CSV thay vì dừng giữa chừng. Bài học: với LLM API bên ngoài, per-item resilience quan trọng hơn per-batch retry.

---

### 3. Kế hoạch Áp dụng vào Đồ án Thực tế (Action Plan)
- **Tên đồ án / Dự án:** [Tên dự án]
- **Đặc thù bài toán & Lý do chọn giải pháp:** [Tại sao bài toán của bạn cần GraphRAG hay chỉ cần Flat/Hybrid RAG?]
- **Cấu trúc Node & Relation dự kiến:**
  - Nodes: `...`
  - Relations: `...`
- **Chiến lược xử lý Super-node & Entity Resolution:** [...]

---

## 🎯 TỰ ĐÁNH GIÁ
| Tiêu chí | Điểm tự chấm (1–5) | Ghi chú |
|----------|-------------------|---------|
| Mức độ hiểu bài giảng GraphRAG | | |
| Khả năng kiểm soát AI Coding Agent | | |
| Chất lượng đồ thị tri thức xây dựng | | |
| Khả năng phân tích và debug hệ thống | | |
