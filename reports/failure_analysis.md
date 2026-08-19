# Failure Analysis — GraphRAG vs Flat RAG

## 1. Phạm vi và dữ liệu

Phân tích dựa trên outputs/graphrag_eval_results.csv, gồm 50 câu hỏi:

- 5 factoid
- 23 multi-hop
- 22 cross-doc

| Metric | Flat RAG | GraphRAG |
|---|---:|---:|
| Comprehensiveness | 2.640 | 2.360 |
| Faithfulness | 2.940 | 2.580 |
| Multi-hop reasoning | 2.640 | 2.340 |
| Latency (s) | 1.807 | 1.762 |
| Token usage | 642.44 | 597.52 |

GraphRAG tiết kiệm token và có latency trung bình thấp hơn, nhưng chất lượng thấp hơn trong lần chạy này.

## 2. Ca lỗi Flat RAG: mất liên kết giữa nhiều tài liệu

### Hiện tượng

Với câu hỏi multi-hop hoặc cross-doc, các dữ kiện cần thiết thường nằm ở nhiều chunk khác nhau. Flat RAG chỉ xếp hạng từng chunk theo độ tương đồng với toàn câu hỏi. Vì vậy có thể lấy được chunk nói về thực thể A nhưng bỏ qua chunk chứa quan hệ A–B hoặc bằng chứng thời gian của B.

### Root cause

1. Similarity search không biểu diễn trực tiếp quan hệ trong đồ thị.
2. Top-k cố định có thể bỏ sót một tài liệu ít giống về từ vựng nhưng quan trọng về quan hệ.
3. Các chunk cùng chủ đề có thể cạnh tranh nhau, làm mất diversity.

### Cách GraphRAG có thể khắc phục

Seed extraction xác định thực thể chính, Neo4j BFS nối các cạnh liên quan, sau đó linearize subgraph cùng source_chunk_id, published_date và evidence. Hybrid context vẫn bổ sung vector chunks để giảm nguy cơ graph thiếu edge.

### Giới hạn

Nếu entity resolution hoặc extraction không tạo được cạnh, GraphRAG không thể khắc phục lỗi dữ liệu nguồn. Vì vậy cần xem đồng thời flat_answer, graph_answer, context và judge rationale, không chỉ nhìn điểm tổng.

## 3. Ca lỗi GraphRAG: graph coverage thấp hoặc seed không khớp

### Hiện tượng

Trong lần benchmark, GraphRAG có điểm trung bình thấp hơn Flat RAG ở cả ba tiêu chí. Điều này phù hợp với các câu hỏi mà graph context rỗng/thưa hoặc seed extraction không tìm được node chính xác; khi đó phần graph không cung cấp thêm bằng chứng, còn vector fallback có thể bị giới hạn.

### Root cause

1. Allowlist quan hệ ưu tiên precision nên có thể loại các quan hệ diễn đạt khác nhau.
2. Seed fuzzy matching phụ thuộc embedding và ngưỡng 0.66.
3. Extraction chỉ chạy tối đa EXTRACTION_MAX_CHUNKS = 400, trong khi flat index có thể chứa nhiều chunk hơn.
4. Entity resolution guard có thể từ chối false merge đúng, nhưng cũng làm giảm recall.
5. Graph traversal chỉ đi theo cạnh đã ingest; missing edge tạo ra missing path.

### Tác động

GraphRAG vẫn trả lời được một phần nhờ vector context, nhưng câu trả lời có thể thiếu thực thể, thiếu quan hệ hoặc thiếu bằng chứng. Đây là lý do GraphRAG dùng ít token hơn nhưng điểm comprehensiveness/faithfulness thấp hơn.

### Khắc phục

- Mở rộng schema quan hệ có kiểm soát và version hóa schema.
- Log seed extraction, exact match, fuzzy match và seed không resolve.
- Chạy targeted re-extraction cho các câu hỏi benchmark bị điểm thấp.
- Tăng graph coverage theo batch thay vì tăng bừa context.
- Dùng hybrid retrieval có diversity theo tài liệu và entity.
- Ghi provenance đầy đủ để kiểm tra edge nào gây mất reasoning.

## 4. Super-node và temporal truncation

SUPER_NODE_DEGREE = 100, SUPER_NODE_EDGE_CAP = 50, GLOBAL_EDGE_CAP = 250 và giới hạn context 14.000 ký tự giúp tránh nổ context. Tuy nhiên, chọn 50 cạnh mới nhất có thể bỏ qua sự kiện lịch sử cũ. Trong benchmark hiện tại, graph_supernode_events bằng 0 cho cả 50 câu, nên chưa có bằng chứng thực nghiệm rằng cap đã ảnh hưởng đến một câu cụ thể.

## 5. Kết luận và kế hoạch kiểm chứng

Kết quả hiện tại không ủng hộ kết luận “GraphRAG luôn tốt hơn”. Kết luận chính xác hơn là:

> GraphRAG có chi phí token thấp hơn và latency tương đương, nhưng cần cải thiện graph coverage, seed resolution và relation recall để đạt chất lượng cao hơn Flat RAG trên bộ dữ liệu này.

Các bước tiếp theo:

1. Lấy 5 câu GraphRAG có điểm thấp nhất.
2. So sánh graph context với reference evidence.
3. Phân loại lỗi thành missing seed, missing edge, wrong canonical entity, insufficient hop hoặc judge disagreement.
4. Re-run targeted retrieval sau khi sửa đúng nguyên nhân.
