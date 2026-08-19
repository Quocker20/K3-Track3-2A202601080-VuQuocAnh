# Phân Tích Ca Lỗi — Flat RAG vs GraphRAG

**Học viên:** Vũ Quốc Anh
**Mã học viên:** 2A202601080
**Khóa học:** AICB-K34 · Track 3: GraphRAG
**Lab:** Day19 — Production-Grade GraphRAG vs Flat RAG

Phân tích theo quy trình root-cause: **Triệu chứng → Nguyên nhân gốc rễ → Điểm trong pipeline gây lỗi → Đề xuất khắc phục.**

---

## Ca lỗi 1 — Flat RAG thất bại, GraphRAG thành công

- **Question ID & câu hỏi:** [G0x — ...]
- **Câu trả lời Flat RAG:** [...]
- **Câu trả lời GraphRAG:** [...]
- **Điểm Judge (Flat vs Graph):** [...]

### Root-cause
- **Triệu chứng:** [Flat RAG thiếu/sai thông tin gì cụ thể]
- **Nguyên nhân gốc rễ:** [Ví dụ: vector search chỉ lấy top-k chunk theo similarity đơn lẻ, không kết nối được 2 chunk chứa 2 phần của câu trả lời multi-hop]
- **Vì sao GraphRAG giải quyết được:** [Graph traversal đi qua cạnh A → B → C nào, provenance nào]

---

## Ca lỗi 2 — GraphRAG thất bại (hoặc cả hai cùng sai)

- **Question ID & câu hỏi:** [G0x — ...]
- **Câu trả lời GraphRAG:** [...]
- **Điểm Judge:** [...]

### Root-cause
- **Triệu chứng:** [...]
- **Nguyên nhân gốc rễ:** [Ví dụ: thiếu seed entity match, missing edge do extraction bỏ sót quan hệ, hoặc super-node cap cắt mất cạnh cần thiết]
- **Điểm trong pipeline gây lỗi:** [seed matching / extraction / entity resolution / super-node cap — cụ thể hàm nào]
- **Đề xuất khắc phục:** [...]

---

## Ghi chú tổng hợp

- **Số lượng supernode_events quan sát được trong quá trình eval:** `0` (cột `graph_supernode_events` trong `outputs/graphrag_eval_results.csv`) — hợp lý vì degree cao nhất trong graph hiện tại là 5 (Microsoft), thấp hơn nhiều ngưỡng production `SUPER_NODE_DEGREE=100`. Cơ chế cắt tỉa chỉ được kích hoạt bằng demo hạ ngưỡng riêng (xem mục 3 `technical_defense.md`), không kích hoạt tự nhiên ở quy mô 1.500 bài báo của lab.
- **Tỷ lệ câu hỏi mà GraphRAG vượt trội Flat RAG theo nhóm (factoid/multi-hop/cross-doc):** *[Cần golden set thật + eval sạch (không lỗi Groq quota) để tính — hiện `outputs/graphrag_vs_flatrag_summary.csv` mới có 2/6 câu mock có điểm judge thật, chưa đủ để kết luận theo nhóm.]*
