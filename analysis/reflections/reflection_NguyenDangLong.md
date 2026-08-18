# Individual Reflection - Lab 18: Production RAG Pipeline

**Học viên:** Nguyễn Đăng Long  
**Mã số học viên / Nhóm:** 2A202601934 · AICB-K34  
**Phạm vi phụ trách:** Triển khai toàn diện 5 Modules (M1 Chunking, M2 Hybrid Search, M3 Reranking, M4 RAGAS Evaluation, M5 Chunk Enrichment) và tích hợp hệ thống.  

---

## Phần 1: Mapping Bài giảng vào Code Thực tế (Lecture Mapping)

Bảng đối chiếu giữa các khái niệm lý thuyết trong bài giảng và hiện thực mã nguồn trong bài lab:

| Khái niệm Lý thuyết | Module | Hàm / Class cụ thể | Quan sát & Đánh giá thực nghiệm |
|---------------------|--------|---------------------|---------------------------------|
| Semantic Chunking | M1 | [`chunk_semantic()`](file:///Users/nguyendanglong/Documents/VINUNI/day18/src/m1_chunking.py#L89-L130) | Ngưỡng cosine similarity `0.85` giúp gom các câu cùng chủ đề mà không làm đứt đoạn ngữ cảnh câu, khắc phục nhược điểm cắt thô bạo theo độ dài cố định. |
| Hierarchical Chunking (Parent-Child) | M1 | [`chunk_hierarchical()`](file:///Users/nguyendanglong/Documents/VINUNI/day18/src/m1_chunking.py#L133-L200) | Chia Parent chunk 2048 ký tự và Child chunk 256 ký tự; truy xuất theo Child chunk để tăng độ chính xác nhưng trả về Parent context cho LLM giúp Faithfulness tăng vọt lên 0.9217. |
| Structure-Aware Chunking | M1 | [`chunk_structure_aware()`](file:///Users/nguyendanglong/Documents/VINUNI/day18/src/m1_chunking.py#L203-L245) | Parse các đề mục Markdown headers (`#`, `##`, `###`) để giữ nguyên vẹn bảng biểu quy chế và cấu trúc logic của tài liệu hành chính. |
| Vietnamese Word Segmentation | M2 | [`segment_vietnamese()`](file:///Users/nguyendanglong/Documents/VINUNI/day18/src/m2_search.py#L21-L30) | Sử dụng `underthesea` và thay thế ký tự `_` thành khoảng trắng để đảm bảo BM25 tokenize đồng nhất giữa query và corpus tiếng Việt. |
| BM25 + Dense Hybrid Search | M2 | [`BM25Search`](file:///Users/nguyendanglong/Documents/VINUNI/day18/src/m2_search.py#L33-L66), [`DenseSearch`](file:///Users/nguyendanglong/Documents/VINUNI/day18/src/m2_search.py#L69-L113) | Kết hợp ưu điểm tìm từ khóa chính xác của BM25 và hiểu ngữ nghĩa sâu của `BAAI/bge-m3` trên Qdrant vector database. |
| Reciprocal Rank Fusion (RRF) | M2 | [`reciprocal_rank_fusion()`](file:///Users/nguyendanglong/Documents/VINUNI/day18/src/m2_search.py#L116-L135) | Áp dụng công thức $RRF\_Score = \sum \frac{1}{k + rank + 1}$ ($k=60$) giúp dung hòa thứ hạng giữa dense score và sparse score mà không cần chuẩn hóa scale điểm. |
| Cross-Encoder Reranking | M3 | [`CrossEncoderReranker.rerank()`](file:///Users/nguyendanglong/Documents/VINUNI/day18/src/m3_rerank.py#L21-L53) | Model `BAAI/bge-reranker-v2-m3` đánh giá tương quan trực tiếp cặp `(query, document)`, lọc từ Top-20 ứng viên xuống Top-3 chất lượng nhất, loại bỏ hoàn toàn tài liệu nhiễu. |
| RAGAS 4 Metrics Evaluation | M4 | [`evaluate_ragas()`](file:///Users/nguyendanglong/Documents/VINUNI/day18/src/m4_eval.py#L29-L110) | Đánh giá khách quan 4 chiều: Faithfulness (0.9217), Answer Relevancy (0.8040), Context Precision (0.8083), Context Recall (0.7500). |
| Failure Diagnostic Tree | M4 | [`failure_analysis()`](file:///Users/nguyendanglong/Documents/VINUNI/day18/src/m4_eval.py#L113-L153) | Tự động phân loại nguyên nhân lỗi truy vấn vào Error Tree để đưa ra giải pháp khắc phục cụ thể theo từng điểm nghẽn. |
| Contextual Embeddings & HyQA | M5 | [`_enrich_single_call()`](file:///Users/nguyendanglong/Documents/VINUNI/day18/src/m5_enrichment.py#L157-L190) | Làm giàu chunk bằng 1 API call duy nhất sinh Summary, 3 câu hỏi giả định và vị trí tài liệu, giúp bắc cầu khoảng cách từ vựng giữa câu hỏi người dùng và nội dung văn bản. |

---

## Phần 2: Khó khăn Kỹ thuật & Cách Giải quyết (Debugging & Problem Solving)

### 1. Lỗi tương thích Proxy API với RAGAS Answer Relevancy
- **Exact Error Message:**  
  `BadRequestError: Error code: 400 - {'error': {'message': 'n>1 is not supported in responses compatibility mode ...'}}`
- **Nguyên nhân:**  
  Metric `answer_relevancy` trong Ragas mặc định gọi `llm.generate(prompt, n=strictness)` với `strictness=3` để sinh 3 câu hỏi đối chứng. Endpoint proxy `https://api.shopaikey.com/v1` không hỗ trợ tham số `n > 1` trong một request OpenAI-compatible.
- **Cách debug & Giải quyết:**  
  Kiểm tra mã nguồn của `ragas.metrics.answer_relevancy` bằng `inspect.getsource(answer_relevancy._ascore)`, phát hiện thuộc tính `strictness`. Thiết lập `answer_relevancy.strictness = 1` trước khi chạy `evaluate()`, giúp Ragas chạy trơn tru với 1 generation per prompt và tính toán đầy đủ 4 metrics.

### 2. Tách từ tiếng Việt với BM25 (Underthesea Compound Words)
- **Vấn đề:**  
  `underthesea.word_tokenize(text, format="text")` tạo ra các từ ghép nối bằng dấu gạch dưới (ví dụ: `nghỉ_phép`). Khi người dùng gõ query tìm kiếm `nghỉ phép` (phân tách bằng khoảng trắng), BM25 coi đây là 2 token riêng biệt và không khớp được với token `nghỉ_phép` trong corpus.
- **Cách giải quyết:**  
  Xử lý chuỗi bằng `.replace("_", " ")` sau khi tách từ, đảm bảo định dạng token đồng nhất giữa bước lập chỉ mục (index) và bước tìm kiếm (search).

### 3. Tối ưu chi phí và thời gian làm giàu dữ liệu (Module 5 Enrichment)
- **Vấn đề:**  
  Nếu gọi 4 hàm LLM riêng biệt (Summarize, HyQA, Contextual Prepend, Metadata) cho 116 chunks, hệ thống phải thực hiện 464 cuộc gọi API, gây nghẽn và tốn thời gian.
- **Cách giải quyết:**  
  Thiết kế hàm `_enrich_single_call()` với Structured JSON Prompt để lấy toàn bộ 4 thành phần trong 1 request duy nhất, giảm 75% số lượng cuộc gọi API và đạt điểm thưởng Bonus.

---

## Phần 3: Action Plan Áp dụng vào Project Cá nhân

```markdown
## Project: Hệ thống Trợ lý Pháp lý & Quy chế Doanh nghiệp Thông minh (Enterprise Policy Assistant)

### Hiện tại
- RAG pipeline hiện tại: Naive RAG sử dụng LangChain RecursiveCharacterTextSplitter, Dense Search đơn lẻ với OpenAI Embeddings, không có Reranking và chưa có quy trình đánh giá tự động.
- Known issues:
  1. Hay bị ảo giác (hallucination) khi gặp các câu hỏi suy luận đa tài liệu (Multi-hop).
  2. Gặp lỗi xung đột khi tài liệu chính sách được cập nhật theo từng năm (Version conflict giữa v2023 và v2024).
  3. Tỷ lệ tìm kiếm chính xác các thuật ngữ pháp lý viết tắt hoặc mã văn bản chưa cao.

### Plan áp dụng kiến thức Lab 18
1. [x] Chunking Strategy:
   - Áp dụng Hierarchical Chunking (Parent 2048 chars / Child 256 chars) cho văn bản dài.
   - Kết hợp Structure-Aware Chunking để bảo toàn các điều khoản và bảng biểu quy định.
2. [x] Search Strategy:
   - Xây dựng Hybrid Search kết hợp Vietnamese BM25 (xử lý chính xác số hiệu văn bản, từ khóa pháp lý) và Dense Search BGE-M3 trên Qdrant.
   - Kết hợp kết quả bằng thuật toán Reciprocal Rank Fusion (RRF, k=60).
3. [x] Reranking:
   - Triển khai Cross-Encoder `BAAI/bge-reranker-v2-m3` để chọn lọc Top-3 context có độ tương đồng thực sự trước khi sinh câu trả lời.
4. [x] Evaluation:
   - Tích hợp framework RAGAS với 4 chỉ số cốt lõi vào CI/CD pipeline để giám sát chất lượng định kỳ sau mỗi lần cập nhật dữ liệu.
5. [x] Enrichment & Metadata Filtering:
   - Tự động trích xuất metadata (năm hiệu lực, danh mục, trạng thái active) trong quá trình Ingestion.
   - Áp dụng Contextual Prepend để định vị ngữ cảnh của từng điều khoản trong bộ luật.

### Timeline triển khai
- Tuần 1: Nâng cấp tầng Ingestion - Implement Hierarchical Chunking và Auto Metadata Extraction.
- Tuần 2: Nâng cấp tầng Retrieval - Cấu hình Hybrid Search (BM25 + Qdrant) và Cross-Encoder Reranking.
- Tuần 3: Tích hợp RAGAS Evaluation và thiết lập bộ 100 câu hỏi test benchmark doanh nghiệp.
- Tuần 4: Fine-tune prompt generation và triển khai giao diện người dùng hoàn chỉnh.
```

---

## Phần 4: Tự Đánh giá (Self-Assessment)

| Tiêu chí | Tự chấm (1-5) | Minh chứng / Ghi chú |
|----------|---------------|----------------------|
| Hiểu bài giảng | 5/5 | Ánh xạ đầy đủ và chính xác toàn bộ 5 modules lý thuyết vào mã nguồn thực tế. |
| Code quality | 5/5 | 100% (37/37) unit tests pass, không còn TODO nào, code sạch, có typing và exception handling an toàn. |
| Pipeline & Metrics | 5/5 | RAGAS Faithfulness đạt 0.9217 (>=0.85), tất cả metrics đều >= 0.75, đạt toàn bộ tiêu chí Bonus (+10 điểm). |
| Problem solving | 5/5 | Phát hiện và khắc phục lỗi proxy RAGAS `answer_relevancy`, xử lý tokenization tiếng Việt BM25 hiệu quả. |
