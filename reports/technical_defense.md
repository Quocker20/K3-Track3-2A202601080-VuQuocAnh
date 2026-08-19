# Thuyết Minh Kỹ Thuật — 10 Câu Hỏi Bảo Vệ Kiến Trúc

**Học viên:** Vũ Quốc Anh
**Mã học viên:** 2A202601080
**Khóa học:** AICB-K34 · Track 3: GraphRAG
**Lab:** Day19 — Production-Grade GraphRAG vs Flat RAG

---

## 1. Coreference Resolution

> Nêu ít nhất 1 tình huống cụ thể trong dữ liệu mà cơ chế Coreference Resolution phân giải sai. Hậu quả của nó đối với Knowledge Graph là gì?

*Trả lời:*
- **Chunk / ví dụ cụ thể:** `chunk_id = ea76b71e88232361f308::c0000` (lấy từ `data/cache/coref_samples.csv`, dòng 4)
  - Gốc: `"...with limited technology-based services. We expect this to continue for the rest of today. We're aiming to open Reading Rooms at St Pancras tomorrow..."`
  - Sau resolve: `"...with limited technology-based services. We're aiming to open Reading Rooms at St Pancras tomorrow..."`
- **Hiện tượng sai:** Không phải resolve sai đối tượng — model **xóa hẳn câu `"We expect this to continue for the rest of today."`** thay vì thay `We`/`this` bằng antecedent rõ ràng. Vì chunk không chứa tên tổ chức (British Library), model không đủ context để gán `We` một cách an toàn nên chọn cách bỏ câu, vi phạm ngầm nguyên tắc "conservative, never invent — nhưng cũng không được xóa fact" mà `COREF_SYSTEM` không nói rõ.
- **Hậu quả đối với Graph:** Mất hẳn 1 fact (dịch vụ tiếp tục bị giới hạn "for the rest of today") trước khi tới bước NER+RE — không tạo ra False Edge, nhưng tạo **silent information loss**: pipeline NER+RE không bao giờ thấy câu này nên không thể trích triple nào liên quan đến thời hạn gián đoạn dịch vụ. Đây là failure mode nguy hiểm hơn cả sai vì không log lỗi (khác với `COREF_BATCH_FAILED`), phải soát thủ công bằng cách so `resolved_text` với `text` mới phát hiện được.

---

## 2. Entity Resolution Threshold & Lexical Guard

> Bạn chọn ngưỡng cosine similarity là bao nhiêu cho vector matching? Trích dẫn 1 cặp thực thể có độ tương đồng vector cao (> 0.85) nhưng bị Lexical Guard chặn không cho gộp và giải thích lý do.

*Trả lời:*
- **Ngưỡng cosine similarity:** `threshold = 0.90` (cell 2.2, `build_resolution_map`)
- **Dữ liệu tự nhiên (real extraction) chỉ sinh 1 cặp candidate** (`X90A` vs `X90`, similarity 0.920579, `MERGE_VECTOR`) — không đủ để quan sát `REJECT_GUARD`. Đã bổ sung cell 5.1 chạy `merge_guard()` trực tiếp trên các cặp bẫy có chủ đích (không phải dữ liệu tự nhiên) để chứng minh cơ chế:

  | Loại | Left | Right | Similarity | Decision |
  |---|---|---|---|---|
  | Company | Apple | Apple Music | 0.670162 | REJECT_GUARD |
  | Company | Amazon | Amazon Web Services | 0.646989 | REJECT_GUARD |
  | Company | Meta | Meta Quest | 0.625470 | REJECT_GUARD |
  | Company | Apple | Apple Watch | 0.609479 | REJECT_GUARD |

- **Lý do chặn:** cả 4 cặp đều là quan hệ **thương hiệu-mẹ vs sản phẩm/chi nhánh con** (substring lexical match: tên phải chứa trọn tên kia + có suffix riêng biệt như "Watch", "Music", "Quest", "Web Services"). Guard coi đây là 2 thực thể **khác nhau về mặt bản thể** (công ty mẹ ≠ sản phẩm của nó) dù embedding coi chúng gần nhau về ngữ nghĩa/lexical — nếu gộp sẽ tạo cạnh sai kiểu "Apple PARTNERED_WITH Apple Music" bị trộn vào node `Apple`.
- **Giới hạn phát hiện được:** trong bộ test này, **không có cặp REJECT_GUARD nào vượt 0.85** (cao nhất 0.670) — ngược lại, cặp `Sam Altman` vs `Steve Altman` đạt similarity 0.824214 nhưng bị **MERGE_VECTOR** (guard không chặn được vì đây là 2 tên người khác nhau về mặt lexical, không rơi vào pattern substring). Đây là **false-negative rủi ro thật** của guard hiện tại: guard chỉ bắt được pattern "brand vs sản phẩm/chi nhánh", không bắt được pattern "2 người trùng họ, khác tên đệm" — cần bổ sung rule riêng cho `type == "Person"` (ví dụ so khớp first-name).

---

## 3. Super-node Analysis

> Top 3 thực thể có bậc (degree) cao nhất trong đồ thị là gì? Việc ưu tiên lấy 50 cạnh mới nhất mang lại ưu điểm và rủi ro gì?

*Trả lời:*

| Hạng | Tên thực thể | Loại | Degree |
|------|--------------|------|--------|
| 1 | Microsoft | Company | 5 |
| 2 | Amazon Web Services | Company | 4 |
| 3 | ServiceNow | Company | 4 |

*(Nguồn: `graph_checks()` → `top_degree_df`, đồ thị 193 nodes / 118 edges. Ở quy mô lab, chưa node nào chạm ngưỡng production `SUPER_NODE_DEGREE=100` nên cell 5.1 đã demo hạ tạm ngưỡng xuống 2 trên `Microsoft` (degree=5) để chứng minh cơ chế cắt tỉa hoạt động — ngưỡng production thật vẫn giữ 100.)*

- **Ưu điểm của temporal cap (50 cạnh mới nhất):** Chặn được bùng nổ context khi 1 node có hàng trăm/ngàn cạnh (ví dụ 1 công ty lớn xuất hiện ở rất nhiều bài báo) — giữ token budget cho LLM answer trong tầm kiểm soát, đồng thời ưu tiên thông tin **gần thời điểm câu hỏi nhất**, phù hợp câu hỏi kiểu "tin tức mới nhất về X".
- **Rủi ro:** Mất hoàn toàn context lịch sử xa — nếu câu hỏi hỏi về sự kiện cũ liên quan tới super-node (ví dụ "Microsoft mua lại công ty nào năm 2019?"), cạnh đó có thể đã bị cắt khỏi 50 cạnh mới nhất dù vẫn tồn tại trong graph, khiến GraphRAG trả lời sai kiểu "không tìm thấy thông tin" thay vì trả lời đúng.

---

## 4. So sánh Benchmark (Flat RAG vs GraphRAG)

> Điền bảng so sánh Quality vs Latency vs Token usage.

| Tiêu chí | Flat RAG | GraphRAG | Δ | Nhận xét |
|----------|----------|----------|---|----------|
| Comprehensiveness (1–5) | | | | |
| Faithfulness (1–5) | | | | |
| Multi-hop Reasoning (1–5) | | | | |
| Latency trung bình (s) | | | | |
| Token usage trung bình | | | | |

*(Nguồn: `outputs/graphrag_vs_flatrag_summary.csv`)*

---

## 5. Trade-offs & Kiểm soát AI Coding Agent

> So sánh đánh đổi chi phí/thời gian giữa GraphRAG và Flat RAG. Trong lúc làm bài, AI Coding Agent từng đề xuất điều gì mà bạn từ chối áp dụng? Tại sao?

*Trả lời:*
- **Trade-off Quality vs Cost vs Latency:** GraphRAG trả thêm 1 lượt LLM call (seed extraction) trước khi generate answer + phải truy vấn Neo4j (BFS 2-hop), nên tốn thêm latency/token so với Flat RAG (chỉ 1 vòng embed + FAISS search + generate). Đổi lại, GraphRAG có provenance rõ ràng theo cạnh (`chunk=`, `evidence=`) và khả năng multi-hop mà similarity-only search của Flat RAG không làm được. *(Số liệu định lượng cụ thể: điền sau khi eval trên golden set thật — xem mục 4/6/7/8.)*
- **Đề xuất bị từ chối / Lý do từ chối:** Khi bàn về near-dedup (Challenge A), AI Agent đề xuất trước tiên là cách đơn giản nhất: tính pairwise cosine similarity giữa toàn bộ chunk (`sentence-transformers` embedding rồi so từng cặp bằng vòng lặp lồng/`sklearn.metrics.pairwise`). Mình từ chối vì hai lý do: (1) đề bài đã cấm rõ ràng cách này ở Challenge A; (2) tự tính thử thấy ở quy mô `LIMIT_ROWS=8000` sẽ ra ma trận ~8.000×8.000 ≈ 64 triệu cặp so sánh — chạy được ở lab nhưng hoàn toàn không scale nếu sau này áp dụng lên 350MB/~100.000 bài báo (câu 10). Yêu cầu Agent đổi hướng sang MinHash/LSH hoặc embedding + FAISS ANN với blocking, độ phức tạp gần tuyến tính thay vì O(N²).

---

## 6. Flat RAG thắng nhóm câu hỏi nào?

*Trả lời:* [...]

---

## 7. GraphRAG thắng nhóm câu hỏi nào?

*Trả lời:* [...]

---

## 8. Latency / Token Trade-off

> Phân tích cụ thể mức chênh lệch latency và token usage giữa hai hệ thống.

*Trả lời:* [...]

---

## 9. Đề xuất của AI Coding Agent bị từ chối (chi tiết)

> Mô tả cụ thể 1 đề xuất kỹ thuật mà AI Coding Agent đưa ra trong quá trình làm bài mà bạn quyết định KHÔNG áp dụng, và lý do kỹ thuật.

*Trả lời:*

Trong lúc viết `run_extraction()`, AI Agent đề xuất bỏ `response_format={"type":"json_object"}` khỏi lời gọi Groq (cell 1.6) để "tiết kiệm token" vì JSON mode buộc model phải in đủ ngoặc/khoá — lý do Agent đưa ra là prompt đã yêu cầu "Return strict JSON only" nên response_format là dư thừa. Mình từ chối vì:
1. **Kỹ thuật:** `parse_json_object()` (cell 1.6) tìm `{`...`}` đầu/cuối chuỗi trả về rồi `json.loads` thẳng — nếu không ép JSON mode, model rất dễ chèn thêm câu dẫn ("Here is the JSON:") hoặc markdown fence trước/sau, làm `parse_json_object` parse sai vị trí ngoặc và toàn bộ `run_coref()`/`run_extraction()` (chạy hàng trăm lần) sẽ fail hàng loạt thay vì fail lẻ tẻ.
2. **Đánh đổi không đáng:** JSON mode chỉ tốn thêm vài chục token/lời gọi, trong khi rủi ro là mất cả batch (400 chunk coref, 400 chunk extraction) nếu parser vỡ — không tương xứng.
Quyết định: giữ nguyên `response_format=json_object`, chỉ tối ưu token ở chỗ khác (giảm `batch_size`, truncate `text` input khi payload lớn — đúng như rủi ro đã ghi ở `docs/IMPLEMENTATION_PLAN.md` §Stage 2).

---

## 10. Scale lên 350MB (~100,000 bài báo)

> Nếu scale lên toàn bộ dataset 350MB, bottleneck đầu tiên xuất hiện ở đâu? Giải pháp kiến trúc là gì?

*Trả lời:*
- **Bottleneck đầu tiên:** LLM API calls tuần tự (coref + extraction), mỗi cell hiện chạy vòng `for` đồng bộ, batch nhỏ (4–5 chunk/lần). Ở 1.500 chunk (lab hiện tại) đã cần ~180 lời gọi Groq; scale lên ~100.000 bài báo (ước lượng hàng chục ngàn chunk sau dedup) sẽ cần hàng chục ngàn lời gọi tuần tự → hàng giờ chạy, và dễ chạm token-per-day quota (đã thực sự gặp giới hạn TPD 200.000 của Groq free tier khi chạy eval batch nhỏ trong lab này).
- **Giải pháp đề xuất:**
  1. **Async/concurrent batching** cho coref + extraction (asyncio + semaphore giới hạn concurrency) thay vì vòng `for` tuần tự, kết hợp queue + backoff riêng cho rate-limit thay vì retry chặn cứng trong `groq_chat`.
  2. **Near-dedup trước khi extract** (MinHash/LSH — theo đúng ràng buộc Challenge A, cấm pairwise cosine O(N²)) để giảm số chunk phải gọi LLM ngay từ đầu — nhiều bài báo repost/gần giống nhau trên HackerNoon.
  3. **Entity Resolution ở quy mô lớn:** union-find hiện tại là O(N²) cặp candidate nếu so mọi node — cần blocking theo `type` + ANN (FAISS/HNSW) để chỉ so các cặp có khả năng trùng cao, giống cách near-dedup xử lý bài báo.
  4. **Bulk insert Neo4j** đã dùng `UNWIND` batch 1000 — vẫn ổn ở quy mô lớn hơn, không phải bottleneck chính.
