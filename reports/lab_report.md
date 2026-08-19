# Báo cáo thực hành và thuyết minh kỹ thuật — Lab 19

**Học viên:** Nguyễn Thu Huyền
**Khóa học:** AICB-K34 · Track 3: GraphRAG  
**Mô hình sử dụng:** OpenAI chat model và text-embedding-3-small

## 1. Tóm tắt kết quả

Pipeline đã chạy trên 50 câu hỏi Golden Dataset gồm 5 factoid, 23 multi-hop và 22 cross-doc. Kết quả trung bình:

| Metric | Flat RAG | GraphRAG |
|---|---:|---:|
| Comprehensiveness | 2.640 | 2.360 |
| Faithfulness | 2.940 | 2.580 |
| Multi-hop reasoning | 2.640 | 2.340 |
| Latency (s) | 1.807 | 1.762 |
| Token usage | 642.44 | 597.52 |

GraphRAG dùng ít hơn khoảng 44.92 token/câu và latency thấp hơn khoảng 0.045 giây, nhưng điểm chất lượng thấp hơn trong lần chạy này. Vì vậy kết luận thực nghiệm là GraphRAG có lợi thế về cấu trúc/provenance và chi phí context, nhưng graph coverage cần được cải thiện trước khi vượt Flat RAG về chất lượng.

## 2. Chuyển từ Grok/Groq sang OpenAI

Trong quá trình chạy ban đầu, các call tới Groq gặp lỗi xác thực API key và retry kéo dài, khiến các cell coreference và extraction không hoàn thành ổn định. Vì vậy pipeline được chuyển sang OpenAI để dùng một provider thống nhất.

Các thay đổi chính:

- Coreference, NER/RE, seed extraction, answer generation, sufficiency check và LLM Judge dùng OpenAI client.
- Embedding chuyển sang text-embedding-3-small.
- API key chỉ đọc từ Colab Secrets/environment qua get_secret(), không hard-code.
- JSON response vẫn được kiểm tra bằng parse_json_object() và có retry/backoff.

README và .env.example được giữ nguyên theo yêu cầu của bài làm; notebook thực thi và các báo cáo này mô tả cấu hình OpenAI thực tế đã dùng.

## 3. Thuyết minh kỹ thuật

### 3.1 Coreference Resolution

resolve_coref_batch() chỉ giải quyết đại từ khi tiền ngữ rõ ràng trong cùng chunk. Trường hợp mơ hồ được ghi trong unresolved_mentions và fallback giữ văn bản gốc. Cách làm này giảm false edge, dù có thể bỏ sót một số liên kết đúng.

### 3.2 Entity Resolution

build_resolution_map() dùng embedding ANN với threshold 0.90, lexical guard với SequenceMatcher ratio 0.72 và Union-Find. Audit table ghi MERGE_MANUAL, MERGE_VECTOR và REJECT_GUARD. Guard giúp tránh hợp nhất nhầm các tên gần giống như Apple với Apple Music hoặc Sam Altman với Steve Altman.

### 3.3 Super-node

SUPER_NODE_DEGREE = 100, SUPER_NODE_EDGE_CAP = 50, GLOBAL_EDGE_CAP = 250 và MAX_GRAPH_CONTEXT_CHARS = 14000. Node có degree trên 100 chỉ lấy 50 cạnh mới nhất. Benchmark có graph_supernode_events bằng 0 ở cả 50 câu, nghĩa là các truy vấn thực tế không kích hoạt ngưỡng; test_supernode_policy() vẫn kiểm tra chính sách độc lập.

### 3.4 Bulk ingestion và provenance

Neo4j được nạp bằng UNWIND theo batch 1000. Edge lưu source_chunk_id, published_date, evidence và confidence. Graph check đã cho invalid_provenance_edges = 0.

### 3.5 Flat RAG và GraphRAG

Flat RAG dùng FAISS để lấy top-k chunks. GraphRAG trích seed, resolve node, BFS theo hop, giới hạn super-node, linearize subgraph có provenance rồi kết hợp với vector context. Self-correction mở rộng Hop 2 → Hop 3 → vector fallback khi context chưa đủ.

### 3.6 LLM-as-a-Judge

Mỗi câu được chấm trên comprehensiveness, faithfulness và multi-hop reasoning theo thang 1–5. Kết quả gồm answer, rationale, latency và token usage cho cả hai phương pháp.

## 4. Phân tích failure modes

Flat RAG dễ thất bại khi thông tin bị tách thành nhiều chunk và top-k không lấy đủ các tài liệu cần nối. GraphRAG có thể khắc phục bằng đường đi giữa các entity.

GraphRAG lại dễ thất bại khi seed không resolve, extraction thiếu edge, relation bị loại bởi allowlist hoặc entity resolution giảm recall. Khi graph thưa, vector fallback không đủ mạnh và Flat RAG có thể đạt điểm cao hơn.

Chi tiết root-cause analysis được trình bày trong failure_analysis.md.

## 5. Trade-offs và scale 350MB

Flat RAG có indexing và vận hành đơn giản, phù hợp factoid. GraphRAG tốn chi phí extraction, entity resolution và Neo4j nhưng phù hợp reasoning có cấu trúc và provenance.

Khi scale lên khoảng 350MB, cần streaming ingestion, async batch extraction, checkpoint/resume, retry rate-limit, ANN index, partition theo community/thời gian và targeted re-extraction. Không nên chạy so sánh pairwise toàn bộ entity vì có nguy cơ O(N²) và OOM.

## 6. Reflection và action plan

Các module trong bài được mapping vào các hàm preprocessing, extraction, ingestion, resolution, retrieval và evaluation. Community Detection bằng NetworkX và Self-Correction đã được triển khai. Near-dedup chưa được bật trong bản notebook này; đây là hướng mở rộng sau exact dedup.

Đối với đồ án thực tế, GraphRAG chỉ nên được chọn khi câu hỏi cần multi-hop, cross-document hoặc provenance có cấu trúc. Với factoid đơn giản, Flat/Hybrid RAG có thể rẻ và đủ tốt hơn.

## 7. Checklist trước khi nộp

- Notebook chạy không có cell crash.
- invalid_provenance_edges = 0.
- Entity audit có các quyết định merge/reject minh bạch.
- Super-node policy có test riêng.
- Golden Dataset có đủ reference answer.
- Hai file benchmark nằm trong outputs/.
- Ba báo cáo kỹ thuật, failure analysis và reflection đã hoàn thiện.
- README và .env.example được giữ nguyên theo yêu cầu; cấu hình OpenAI thực tế được mô tả trong báo cáo này.
