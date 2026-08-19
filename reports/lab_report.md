# Báo Cáo Thực Hành & Thuyết Minh Kỹ Thuật — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Nguyễn Thế Khoi
**Khóa học:** AICB-K34 · Track 3: GraphRAG
**Ngày thực hiện:** 19/08/2026

## Phạm vi lần chạy

Golden Dataset gồm 50 câu hỏi (G5000-01 đến G5000-50). Graph được xây dựng theo profile quick trước khi đánh giá toàn bộ Golden Set: 1,500 bài, tối đa 3,000 chunks và 80 chunks extraction. Vì graph có coverage nhỏ hơn phạm vi 5,000 dòng mà Golden Set tham chiếu, benchmark này phải được đọc như một baseline/failure analysis, không phải bằng chứng GraphRAG đã tối ưu.

Artifacts:

- 52 Entity nodes, 27 relationships.
- 0 relationship thiếu source_chunk_id hoặc published_date.
- Không có super-node event trong 50 truy vấn; degree cao nhất chỉ là 2.
- Kết quả chi tiết: outputs/graphrag_eval_results.csv.
- Bảng tổng hợp: outputs/graphrag_vs_flatrag_summary.csv.

---

## PHẦN 1: THUYẾT MINH KỸ THUẬT & PHÂN TÍCH CA LỖI

### 1. Coreference Resolution

resolve_coref_batch() dùng quy tắc conservative: chỉ resolve khi antecedent có mặt rõ ràng trong cùng chunk, còn mơ hồ thì giữ text gốc và ghi unresolved_mentions. Điều này giảm false edge nhưng làm giảm recall.

Lần chạy hiện tại không persist coref_df hoặc unresolved_mentions ra file; do đó không thể trích dẫn một chunk_id coreference sai mà không bịa dữ liệu. Đây là thiếu sót observability cần khắc phục: các lần chạy sau phải export coref_df và các record fallback cùng outputs. Tình huống rủi ro điển hình là “the company” trong chunk có cả bên mua lẫn bên bị mua; nếu resolve theo subject gần nhất, edge ACQUIRED hoặc INVESTED_IN sẽ bị gán sai chủ thể.

### 2. Entity Resolution Threshold & Lexical Guard

- Ngưỡng cosine similarity: 0.90.
- Candidate được sinh bằng FAISS ANN trong từng entity type; merge_guard() bỏ hậu tố pháp lý và yêu cầu SequenceMatcher >= 0.72; Union-Find tạo cụm canonical.
- Ngưỡng cao thiên về precision, phù hợp khi false merge có thể tạo nhiều cạnh sai.

entity_resolution_audit_df cũng chưa được export trong lần chạy này, nên không có cặp REJECT_GUARD thực nghiệm để trích dẫn trung thực. Cần export audit ở lần chạy kế tiếp. Ví dụ guard cần chặn là Sam Altman và Steve Altman: embedding có thể gần vì cùng họ/ngữ cảnh AI, nhưng đây là hai Person khác nhau. Apple (Company) và Apple Watch (Technology) đã được chặn ngay từ type guard.

### 3. Đồ thị & Super-node Mitigation

| Hạng | Tên thực thể | Loại | Degree |
|---|---|---|---:|
| 1 | Amazon Web Services (AWS) | Company | 2 |
| 2 | Apple | Company | 2 |
| 3 | GreenPages | Company | 1 |

Không node nào vượt ngưỡng degree 100; tổng số graph_supernode_events trong 50 truy vấn là 0. Vì vậy policy lấy tối đa 50 cạnh mới nhất chưa được kích hoạt thực nghiệm.

Về kiến trúc, cap 50 cạnh giảm bùng nổ BFS, token và latency. Rủi ro là mất evidence lịch sử; với truy vấn có mốc thời gian, nên filter theo khoảng thời gian của câu hỏi và phân bổ edge theo relation diversity, thay vì chỉ chọn latest 50.

### 4. So sánh Thực nghiệm: Flat RAG vs GraphRAG

| Tiêu chí | Flat RAG | GraphRAG | Delta Graph - Flat | Nhận xét |
|---|---:|---:|---:|---|
| Comprehensiveness (1–5) | 2.50 | 2.30 | -0.20 | Graph chưa đủ coverage. |
| Faithfulness (1–5) | 2.70 | 2.52 | -0.18 | Một số graph context dẫn tới chunk sai. |
| Multi-hop Reasoning (1–5) | 2.50 | 2.30 | -0.20 | Chưa thấy lợi thế trung bình của GraphRAG. |
| Latency trung bình (s) | 2.268 | 1.926 | -0.342 | Graph nhanh hơn trong run này do context nhỏ. |
| Token usage trung bình | 769.78 | 592.36 | -177.42 | Graph context ngắn vì chỉ có 27 edge. |

Kết quả theo group cũng nhất quán với chẩn đoán trên: cross-doc gần nhau về quality, factoid nghiêng về Flat RAG, và multi-hop vẫn chưa có lợi thế GraphRAG. Điều này không mâu thuẫn với mục tiêu GraphRAG; nó cho thấy extraction 80 chunks không bao phủ Golden Set 50 câu.

**Ca Flat RAG thất bại, GraphRAG thành công — G5000-18.** Câu hỏi yêu cầu gộp rows 1943, 2566, 3284 thành một Microsoft incident. Flat trả lời thiếu evidence, đạt 1/1/1. Graph trả lời rằng các outage đầu tháng 6 đều do cyberattack và cùng incident, trích Reuters chunk; đạt 4/5/4. Lợi thế đến từ graph context giữ được quan hệ Microsoft – service disruption – cyberattack, dù vẫn thiếu chi tiết hacktivist DDoS.

**Ca GraphRAG thất bại — G5000-26.** Câu hỏi cần Cohere và chương trình conversational customer-service agents trong Amazon AI-service expansion. Flat lấy đúng Cohere và đạt 4/5/4. Graph retrieval đi nhầm sang AMD AI-chip article, trả lời sai provider và đạt 1/1/1. Nguyên nhân là seed Amazon quá rộng trong graph thưa, cùng với missing/cạnh chưa được trích xuất cho Cohere. Khắc phục: tăng extraction coverage, re-rank edge theo lexical overlap với query và vector fallback khi graph evidence không chứa thuật ngữ trọng yếu.

### 5. Trade-offs, Agent Control & Scale 350MB

Flat RAG rẻ hơn ở indexing và hiệu quả khi evidence nằm trong một chunk. GraphRAG có overhead extraction, entity resolution và Neo4j ingestion, nhưng có thể tạo đường suy luận và provenance khi graph đủ coverage. Trong run này GraphRAG nhanh hơn 0.342 giây và ít hơn 177.42 token/truy vấn vì graph quá nhỏ; không nên diễn giải đó là lợi thế chi phí ở scale thực.

Đề xuất bị từ chối là so sánh cosine pairwise toàn bộ mention theo O(N²), vì không scale và dễ OOM. Thiết kế dùng ANN top-k, lexical guard và Union-Find.

Ở mức 350MB hoặc khoảng 100,000 bài, bottleneck đầu tiên là LLM extraction về chi phí/rate-limit. Giải pháp gồm async worker queue, retry tôn trọng Retry-After, checkpoint theo chunk, bulk UNWIND, HNSW/ANN blocking cho entity resolution và partition/community cho global retrieval.

---

## PHẦN 2: SUY NGẪM & KẾ HOẠCH ĐỒ ÁN

### 1. Mapping bài giảng vào code

| Khái niệm | Module | Hàm / khối code | Quan sát |
|---|---|---|---|
| Conservative Coreference | M1 | resolve_coref_batch, run_coref | Giảm false edge; cần export audit để review. |
| Dedup & scale guard | M1 | standardize_news, remove_near_duplicates, build_chunks | SHA-1 exact dedup và SimHash banding giảm repost. |
| Schema & provenance guard | M2 | run_extraction, validate_extracted_triples | Evidence phải xuất hiện trong source chunk trước ingestion. |
| Bulk Cypher ingestion | M2 | bulk_insert_nodes, bulk_insert_edges | UNWIND theo batch, không insert từng row. |
| Entity Resolution | M3 | build_resolution_map, merge_guard, UF | ANN sinh candidate, guard giảm false merge. |
| Hybrid retrieval | M4 | retrieve_flat_context, match_seeds, retrieve_graph_context | Kết hợp graph context có provenance và vector context. |
| Super-node cap | M4 | recent_edges, retrieve_graph_context | Policy có mặt nhưng chưa trigger do graph nhỏ. |
| LLM-as-a-Judge | M5 | judge_answer, run_evaluation, comparison_table | Chấm 3 tiêu chí 1–5 cho 50 câu. |

### 2. Quá trình Debugging & Bài học

Ba lỗi chính đã gặp:

1. Schema HackerNoon dump không có text/body mà dùng description và url. Loader được mở rộng để hỗ trợ hai cột này.
2. Model Groq llama-3.3-70b-versatile trả 404 vì không còn khả dụng; model được đổi sang openai/gpt-oss-120b.
3. Groq trả 429 vì quota token. Notebook được bổ sung LLM_PROVIDER=openai và REUSE_EXISTING_GRAPH=true để không rerun extraction đã ingest.

Bài học: cần validate schema và model availability bằng request nhỏ trước batch run; các intermediate artifact như coref_df và entity_resolution_audit_df phải được checkpoint/export để báo cáo có thể tái lập.

### 3. Kế hoạch áp dụng vào đồ án thực tế

Đề xuất dự án: hệ thống hỏi đáp và phân tích tin công nghệ theo sự kiện.

- Kiến trúc: Hybrid RAG là mặc định; bật GraphRAG cho câu hỏi có nhiều công ty, quan hệ và timeline.
- Nodes: Document, Company, Person, Technology, Event, Topic.
- Relations: ACQUIRED, INVESTED_IN, PARTNERED_WITH, DEVELOPED, MENTIONED_IN, HAS_STATUS.
- Entity resolution: canonical ID theo nguồn đáng tin cậy; alias table + ANN candidate + type/lexical guard.
- Super-node: cap theo thời gian, diversity theo relation và rerank theo query; với câu hỏi lịch sử dùng temporal filter thay vì latest-only.

---

## TỰ ĐÁNH GIÁ

| Tiêu chí | Điểm (1–5) | Ghi chú |
|---|---:|---|
| Mức độ hiểu bài giảng GraphRAG | 4 | Hiểu được pipeline và failure modes. |
| Khả năng kiểm soát AI Coding Agent | 4 | Từ chối O(N²), thêm validation và fallback provider. |
| Chất lượng đồ thị tri thức xây dựng | 2 | Provenance đúng nhưng coverage thấp: 52 nodes, 27 edges. |
| Khả năng phân tích và debug hệ thống | 4 | Xử lý schema mismatch, model 404 và quota 429. |
