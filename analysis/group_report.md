# Group Report - Lab 18: Production RAG

**Nhóm:** Cá nhân AICB-K34  
**Học viên:** Nguyễn Đăng Long  
**Ngày:** 18/08/2026  
**Repository:** [K34-Day18-Production-RAG-2A202601934-NguyenDangLong](https://github.com/Lolavine777/K34-Day18-Production-RAG-2A202601934-NguyenDangLong)

---

## Thành viên & Phân công Modules

| Tên | Module | Hoàn thành | Tests pass |
|-----|--------|:---:|:---:|
| Nguyễn Đăng Long | [M1: Chunking](file:///Users/nguyendanglong/Documents/VINUNI/day18/src/m1_chunking.py) | ✅ | 13/13 |
| Nguyễn Đăng Long | [M2: Hybrid Search](file:///Users/nguyendanglong/Documents/VINUNI/day18/src/m2_search.py) | ✅ | 5/5 |
| Nguyễn Đăng Long | [M3: Reranking](file:///Users/nguyendanglong/Documents/VINUNI/day18/src/m3_rerank.py) | ✅ | 5/5 |
| Nguyễn Đăng Long | [M4: Evaluation](file:///Users/nguyendanglong/Documents/VINUNI/day18/src/m4_eval.py) | ✅ | 4/4 |
| Nguyễn Đăng Long | [M5: Chunk Enrichment](file:///Users/nguyendanglong/Documents/VINUNI/day18/src/m5_enrichment.py) | ✅ | 10/10 |
| **Tổng cộng** | **Toàn bộ 5 Modules** | ✅ | **37/37 (100%)** |

---

## Kết quả RAGAS Benchmark (Toàn diện qua các giai đoạn)

| Metric | Naive Baseline | Production Initial | Production Optimized | Δ Cải thiện Toàn diện | Đánh giá |
|--------|:---:|:---:|:---:|:---:|---|
| **Faithfulness** | 0.8553 | 0.9217 | **0.9014** | **+0.0461** (+5.4%) | ⭐ Xuất sắc (vượt chuẩn bonus >= 0.85) |
| **Answer Relevancy** | 0.7560 | 0.8040 | **0.9316** | **+0.1756** (+23.2%) | ⭐ Đột phá (vượt xa chuẩn >= 0.75) |
| **Context Precision** | 0.8667 | 0.8083 | **0.9417** | **+0.0750** (+8.7%) | ⭐ Xuất sắc (vượt xa chuẩn >= 0.75) |
| **Context Recall** | 0.7500 | 0.7500 | **0.9500** | **+0.2000** (+26.7%) | ⭐ Đột phá (vượt xa chuẩn >= 0.75) |

*Tất cả 4 chỉ số đều vượt ngưỡng 0.90 sau khi áp dụng Parent Document Retrieval, Sub-query Decomposition và Domain Instruction Prompting.*

---

## Latency Breakdown Report (Thời gian thực thi từng giai đoạn)

| Giai đoạn Pipeline | Module đảm nhiệm | Thời gian thực thi | Tỷ trọng | Ghi chú tối ưu |
|-------------------|------------------|-------------------|----------|----------------|
| Chunking (Hierarchical) | [`src/m1_chunking.py`](file:///Users/nguyendanglong/Documents/VINUNI/day18/src/m1_chunking.py) | 0.1s | < 0.1% | 116 hierarchical chunks sinh nhanh trong memory |
| Chunk Enrichment (Combined) | [`src/m5_enrichment.py`](file:///Users/nguyendanglong/Documents/VINUNI/day18/src/m5_enrichment.py) | 2626.1s | 77.8% | 116 chunks x 1 single-call API (HyQA + Context + Meta) |
| Vector Indexing (Qdrant + BM25) | [`src/m2_search.py`](file:///Users/nguyendanglong/Documents/VINUNI/day18/src/m2_search.py) | 49.8s | 1.5% | BGE-M3 Dense embeddings + Underthesea BM25 |
| Cross-Encoder Reranker Load | [`src/m3_rerank.py`](file:///Users/nguyendanglong/Documents/VINUNI/day18/src/m3_rerank.py) | 0.0s | < 0.1% | Preloaded weights `bge-reranker-v2-m3` |
| Inference (20 queries + Subquery + Parent Context) | [`src/pipeline.py`](file:///Users/nguyendanglong/Documents/VINUNI/day18/src/pipeline.py) | ~589.3s | 17.5% | Query decomposition, parallel search, parent retrieval |
| RAGAS 4 Metrics Evaluation | [`src/m4_eval.py`](file:///Users/nguyendanglong/Documents/VINUNI/day18/src/m4_eval.py) | 108.9s | 3.2% | 80 evaluation jobs (4 metrics x 20 questions) |
| **Tổng thời gian End-to-End** | **src/pipeline.py** | **3374.2s** | **100.0%** | **Toàn bộ pipeline chạy mượt mà không lỗi** |

---

## Key Findings

1. **Biggest improvement:** Context Recall và Answer Relevancy tăng vọt từ 0.7500 lên 0.9500 và 0.8040 lên 0.9316 nhờ kỹ thuật **Small-to-Big Parent Retrieval** (truy xuất chính xác trên child chunks nhưng truyền toàn vẹn parent context cho LLM) và **Sub-query Decomposition** (xử lý triệt để các câu hỏi đa vế).
2. **Biggest challenge:** Tích hợp RAGAS với custom LLM provider (`gpt-5.6-luna` tại `https://api.shopaikey.com/v1`) gặp lỗi proxy do `answer_relevancy` mặc định gọi $n=3$ completions cùng lúc; giải quyết triệt để bằng cách thiết lập `answer_relevancy.strictness = 1`.
3. **Surprise finding:** Module 5 Enrichment với chế độ Combined Single-Call mode giúp đính kèm câu hỏi giả định (HyQA) và metadata tự động vào chunk, giúp BM25 tìm kiếm nhạy bén hơn hẳn đối với các câu hỏi sử dụng từ ngữ đồng nghĩa hoặc câu hỏi khái quát.

---

## Presentation Notes (Tổng kết báo cáo 5 phút)

1. **RAGAS scores (Naive vs Production Optimized):** Faithfulness tăng từ 0.8553 lên 0.9014 (+0.0461), Answer Relevancy tăng từ 0.7560 lên 0.9316 (+0.1756), Context Precision tăng từ 0.8667 lên 0.9417 (+0.0750), Context Recall tăng từ 0.7500 lên 0.9500 (+0.2000).
2. **Biggest win:** Bộ đôi M1 Hierarchical Small-to-Big Retrieval kết hợp M3 Cross-Encoder Reranker loại bỏ hoàn toàn các đoạn văn rác không liên quan, mở rộng ngữ cảnh đầy đủ giúp LLM không bị thiếu thông tin.
3. **Case study:** Phân tích câu hỏi xung đột phiên bản chính sách ngày phép (v2023 vs v2024), giải quyết bằng quy tắc ưu tiên phiên bản có hiệu lực mới nhất trong System Prompt.
4. **Optimization hoàn tất:** Triển khai thành công Sub-query Decomposition Agent và Small-to-Big Parent Context Resolution, đưa toàn bộ 4 chỉ số RAGAS vượt mốc 0.90.
