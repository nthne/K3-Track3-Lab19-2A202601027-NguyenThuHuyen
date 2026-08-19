# Reflection và Action Plan — Lab 19

## 1. Mapping bài giảng vào code

| Module | Khái niệm | Hàm/khối code | Kết quả quan sát |
|---|---|---|---|
| M1 | Streaming, dedup, chunking, coreference | load_stream_to_csv(), exact_dedup(), chunk_text(), run_coref() | Dataset được giới hạn theo scale guard; coreference có fallback conservative |
| M2 | Strict NER/RE và bulk ingestion | extract_batch(), run_extraction(), bulk_insert_nodes(), bulk_insert_edges() | Schema allowlist, evidence/confidence và UNWIND batch |
| M3 | Entity resolution | build_resolution_map(), merge_guard(), UF | OpenAI embedding + FAISS + lexical guard + audit |
| M4 | Flat RAG và hybrid GraphRAG | build_flat_index(), retrieve_graph_context(), answer_flat_rag(), answer_graph_rag() | BFS, provenance, vector context và super-node cap |
| M5 | Benchmark và LLM Judge | run_evaluation(), judge_answer(), comparison_table() | 50 câu hỏi, 3 nhóm, điểm 1–5, latency và token usage |

## 2. Bài học debugging

Lỗi quan trọng nhất là khác biệt giữa “code có vẻ đúng” và “toàn bộ pipeline chạy đúng”. Việc kiểm tra riêng context_sufficient() hoặc self_correcting_context() không thay thế benchmark đầy đủ. Cần kiểm tra cả dữ liệu đầu vào, Neo4j graph, provenance, output CSV và rationale của judge.

Một bài học khác là provider/model phải nhất quán. Notebook được chuyển từ Groq sang OpenAI ở cả chat generation, JSON tasks, judge và embedding. API key được đọc từ secret/environment, không ghi trực tiếp vào notebook.

## 3. Kết quả benchmark và suy ngẫm

Benchmark 50 câu cho kết quả:

| Metric | Flat RAG | GraphRAG |
|---|---:|---:|
| Comprehensiveness | 2.660 | 2.480 |
| Faithfulness | 2.960 | 2.760 |
| Multi-hop reasoning | 2.660 | 2.480 |
| Latency (s) | 1.794 | 1.806 |
| Token usage | 645.72 | 650.88 |

Kết quả cho thấy GraphRAG không nên được đánh giá chỉ bằng trực giác kiến trúc. GraphRAG có lợi thế về cấu trúc, provenance và khả năng mở rộng reasoning, nhưng chất lượng phụ thuộc mạnh vào graph coverage. Trong dataset hiện tại, Flat RAG vẫn tốt hơn về điểm Judge và GraphRAG cũng dùng nhiều token hơn một chút.

Near-dedup giúp điểm GraphRAG tăng so với lần chạy trước: comprehensiveness từ 2.360 lên 2.480, faithfulness từ 2.580 lên 2.760 và multi-hop reasoning từ 2.340 lên 2.480. Tuy nhiên GraphRAG vẫn chưa vượt Flat RAG; latency tăng lên 1.806 giây và token usage lên 650.88. Bài học là một cải tiến dữ liệu nhỏ có thể giúp chất lượng, nhưng cần đánh giá đồng thời quality, cost và latency.

## 4. Action plan cho đồ án thực tế

### Khi nào cần GraphRAG?

Chọn GraphRAG khi câu hỏi cần nối nhiều thực thể, quan hệ, thời gian hoặc nguồn tài liệu; cần giải thích provenance; hoặc cần các truy vấn có cấu trúc lặp lại. Với factoid đơn giản, Flat RAG thường rẻ và đủ tốt hơn.

### Thiết kế dự kiến

Nodes: Company, Person, Technology, Product, Document, Event.  
Relations: FOUNDED, WORKED_AT, INVESTED_IN, ACQUIRED, DEVELOPED, USES, PARTNERED_WITH, MENTIONED_IN.

Mỗi edge cần lưu source_chunk_id, published_date, evidence và confidence. Entity resolution cần kết hợp alias thủ công, ANN candidate search và lexical guard. Super-node cần cap theo degree, thời gian hoặc community.

### Kế hoạch triển khai

1. Xác định schema và provenance contract trước khi gọi LLM.
2. Dùng streaming ingestion và checkpoint để xử lý dữ liệu lớn.
3. Chạy extraction theo batch, retry có backoff và lưu lỗi.
4. Xây audit dashboard cho entity merge/reject và missing provenance.
5. Benchmark theo nhóm factoid, multi-hop và cross-doc.
6. Dùng failure analysis để sửa retrieval, không chỉ thay model.

## 5. Bonus đã thực hiện

- Community detection bằng NetworkX và ghi community_id vào Neo4j.
- Self-correction retrieval theo tuyến Hop 2 → Hop 3 → vector fallback.
- Chính sách super-node mitigation và test riêng đã có.

Near-dedup bằng SimHash đã được triển khai và chạy thật, loại 1 bài gần trùng sau exact dedup. MinHash/LSH vẫn là hướng mở rộng khác nếu cần tối ưu thêm.

## 6. Cam kết chất lượng

Trước khi nộp cần chạy lại Restart & Run All, xác nhận không có cell crash, kiểm tra invalid_provenance_edges == 0, bảo đảm hai CSV nằm trong outputs/, rà soát không còn placeholder trong báo cáo và đồng bộ README/requirements với OpenAI.
