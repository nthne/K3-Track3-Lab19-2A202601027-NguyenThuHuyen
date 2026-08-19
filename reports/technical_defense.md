# Technical Defense — Lab 19

**Học viên:** Nguyễn Thu Huyền  
**Notebook:** Day19_GraphRAG_vs_FlatRAG_Production_Lab_Guide.ipynb  
**LLM provider:** OpenAI (chat model cấu hình qua OPENAI_MODEL; embedding text-embedding-3-small)

## 1. Coreference Resolution

Notebook dùng resolve_coref_batch() với prompt conservative: chỉ giải quyết đại từ khi tiền ngữ xuất hiện rõ trong cùng chunk, giữ nguyên số liệu/ngày tháng/ticker và ghi unresolved_mentions khi không chắc chắn. Đây là lựa chọn ưu tiên precision hơn recall. Nếu mô hình nối nhầm “the company” sang thực thể gần nhất nhưng không phải chủ thể, pipeline có thể tạo false edge; vì vậy fallback của run_coref() giữ lại văn bản gốc và đánh dấu batch lỗi thay vì tự bịa dữ kiện.

## 2. Entity Resolution và Lexical Guard

build_resolution_map() dùng embedding ANN với ngưỡng cosine 0.90, sau đó áp dụng lexical guard. strip_suffix() loại hậu tố pháp lý như Inc, Corp, Ltd, còn merge_guard() yêu cầu hai tên sau chuẩn hóa giống nhau hoặc có SequenceMatcher ratio tối thiểu 0.72. Union-Find chỉ hợp nhất các cặp đã qua guard.

Một cặp có thể có vector similarity cao nhưng vẫn bị từ chối nếu khác biệt tên cho thấy nguy cơ false merge, ví dụ Apple và Apple Music hoặc Sam Altman và Steve Altman. Bảng entity_resolution_audit_df ghi MERGE_MANUAL, MERGE_VECTOR hoặc REJECT_GUARD, giúp truy vết quyết định.

## 3. Super-node Mitigation

Các giới hạn chính:

    SUPER_NODE_DEGREE = 100
    SUPER_NODE_EDGE_CAP = 50
    GLOBAL_EDGE_CAP = 250
    MAX_GRAPH_CONTEXT_CHARS = 14000

Khi degree của node vượt 100, BFS chỉ lấy tối đa 50 cạnh mới nhất; toàn bộ context không vượt 250 cạnh và 14.000 ký tự. Ưu điểm là giảm graph explosion, latency và token usage. Rủi ro là các sự kiện lịch sử cũ có thể bị loại khi câu hỏi cần thông tin theo thời gian dài.

Trong benchmark 50 câu, graph_supernode_events bằng 0 cho cả 50 câu. Điều này cho thấy các truy vấn benchmark không kích hoạt node vượt ngưỡng, không phải bằng chứng rằng chính sách không tồn tại. Cell test_supernode_policy() vẫn là kiểm tra riêng cho chính sách cap.

## 4. Benchmark Flat RAG và GraphRAG

Benchmark thực tế có 50 câu: 5 factoid, 23 multi-hop và 22 cross-doc.

| Metric | Flat RAG | GraphRAG |
|---|---:|---:|
| Comprehensiveness | 2.640 | 2.360 |
| Faithfulness | 2.940 | 2.580 |
| Multi-hop reasoning | 2.640 | 2.340 |
| Latency (s) | 1.807 | 1.762 |
| Token usage | 642.44 | 597.52 |

GraphRAG dùng ít token hơn khoảng 44.92 token/câu và latency trung bình thấp hơn khoảng 0.045 giây, nhưng điểm chất lượng thấp hơn. Đây là kết quả quan trọng: GraphRAG không tự động tốt hơn nếu graph extraction còn thưa, seed matching thất bại hoặc quan hệ cần thiết chưa nằm trong allowlist.

## 5. Hai dạng failure case

Flat RAG có thể thất bại khi câu hỏi yêu cầu nối nhiều chunk bằng quan hệ trung gian. Top-k vector có thể lấy các bài nói về cùng chủ đề nhưng không giữ được đường đi A → B → C. GraphRAG có tiềm năng khắc phục bằng seed extraction, BFS và provenance trên cạnh.

Ngược lại, GraphRAG có thể thất bại khi seed không khớp node, quan hệ bị loại ở schema allowlist, extraction không tạo edge hoặc super-node cap cắt mất cạnh cần thiết. Khi đó Flat RAG đôi khi vẫn trả lời tốt hơn vì truy hồi trực tiếp chunk chứa câu trả lời.

## 6. Trade-off và scale 350MB

Flat RAG đơn giản hơn, index nhanh hơn và phù hợp factoid. GraphRAG có chi phí extraction, entity resolution, Neo4j ingestion và bảo trì schema, nhưng có lợi thế khi cần multi-hop, cross-document reasoning và provenance có cấu trúc.

Với dữ liệu khoảng 350MB, bottleneck đầu tiên thường là I/O, embedding và LLM extraction. Kiến trúc nên dùng streaming ingestion, async batch queue, checkpoint/resume, rate-limit retry, partition theo thời gian/chủ đề, ANN index thay cho so sánh toàn cặp và community partitioning.

## 7. Kiểm soát AI Coding Agent

- Không hard-code API key hoặc Neo4j password.
- Không dùng pairwise comparison toàn bộ dataset vì nguy cơ O(N²) và OOM.
- Không bỏ provenance để đổi lấy recall.
- Không coi điểm benchmark thấp hơn là lỗi hệ thống nếu chưa phân tích retrieval context và judge rationale.

## 8. OpenAI migration và kết luận

Coreference, NER/RE, seed extraction, answer generation, sufficiency check và judge đều dùng OpenAI client. Embedding cũng dùng text-embedding-3-small; vector được chuẩn hóa trước khi đưa vào FAISS inner-product index. API key được đọc từ Colab Secrets/environment.

Pipeline đáp ứng kiến trúc Production GraphRAG: preprocessing, strict extraction, entity resolution có audit, bulk Neo4j ingestion bằng UNWIND, hybrid retrieval, provenance, evaluation và self-correction. Kết quả benchmark cho thấy cần cải thiện graph coverage và seed/relation recall trước khi kết luận GraphRAG vượt Flat RAG về chất lượng.
