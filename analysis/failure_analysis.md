# Failure Analysis - Lab 18: Production RAG

**Tác giả:** Nguyễn Đăng Long  
**Pipeline:** Production RAG Pipeline (M1 Chunking + M5 Enrichment + M2 Hybrid Search + M3 Rerank + M4 RAGAS)  
**Tài liệu tham chiếu:** [ragas_report.json](file:///Users/nguyendanglong/Documents/VINUNI/day18/reports/ragas_report.json) và [naive_baseline_report.json](file:///Users/nguyendanglong/Documents/VINUNI/day18/reports/naive_baseline_report.json)

---

## RAGAS Scores Comparison

| Metric | Naive Baseline | Production | Δ | Đánh giá |
|--------|---------------|------------|---|----------|
| Faithfulness | 0.8553 | 0.9217 | +0.0664 | Xuất sắc (vượt chuẩn bonus >= 0.85) |
| Answer Relevancy | 0.7560 | 0.8040 | +0.0480 | Đạt chuẩn (>= 0.75) |
| Context Precision | 0.8667 | 0.8083 | -0.0583 | Đạt chuẩn (>= 0.75) |
| Context Recall | 0.7500 | 0.7500 | +0.0000 | Đạt chuẩn (>= 0.75) |

---

## Bottom-5 Failures Analysis

### #1. Câu hỏi mua sắm thiết bị CNTT & phân quyền phê duyệt
- **Question:** Nếu cần mua một chiếc laptop 30 triệu cho nhân viên mới, ai phê duyệt và cần gì từ phòng CNTT?
- **Expected:** Laptop 30 triệu nằm trong khoảng 5-50 triệu nên cần Giám đốc phòng ban (Director) phê duyệt. Ngoài ra, mua sắm thiết bị CNTT cần có xác nhận cấu hình kỹ thuật từ phòng CNTT trước khi đề xuất. Cần đính kèm ít nhất 3 báo giá vì trên 10 triệu.
- **Got:** Không tìm thấy thông tin về người phê duyệt mua laptop. Cần có xác nhận của phòng CNTT về cấu hình kỹ thuật trước khi đề xuất.
- **Worst metric:** `answer_relevancy` (0.0000) / `context_recall` (0.3333)
- **Error Tree:** Output thiếu vế người phê duyệt -> Context có đủ thông tin không? -> Context chỉ retrieve được chunk kỹ thuật CNTT ([`mua_sam.md`](file:///Users/nguyendanglong/Documents/VINUNI/day18/data/mua_sam.md)), thiếu chunk phân quyền phê duyệt tài chính ([`chi_phi_expense.md`](file:///Users/nguyendanglong/Documents/VINUNI/day18/data/chi_phi_expense.md)) -> Retrieval bị phân mảnh.
- **Root cause:** Multi-hop cross-document retrieval. Câu hỏi yêu cầu thông tin từ 2 văn bản khác nhau, khiến Reranker ưu tiên các chunk có độ khớp từ khóa cao về "laptop" mà bỏ rơi chunk về hạn mức chi phí.
- **Suggested fix:** Áp dụng kỹ thuật Sub-query Decomposition (tách thành 2 truy vấn: thẩm quyền phê duyệt mua tài sản 30tr và quy định mua máy tính từ IT) trước khi truy xuất.

### #2. Xung đột phiên bản chính sách ngày phép (v2023 vs v2024)
- **Question:** Thâm niên bao nhiêu năm thì được cộng thêm ngày phép?
- **Expected:** Theo chính sách v2024 hiện hành, nhân viên có thâm niên từ 3 năm trở lên được cộng thêm 1 ngày phép cho mỗi 3 năm. Chính sách cũ v2023 yêu cầu 5 năm.
- **Got:** Context không nhất quán: có đoạn quy định từ 5 năm được cộng 1 ngày phép mỗi 5 năm, đoạn khác quy định từ 3 năm được cộng 1 ngày phép mỗi 3 năm.
- **Worst metric:** `answer_relevancy` (0.0000)
- **Error Tree:** Output chỉ ra xung đột mà không khẳng định chính sách hiện hành -> Context chứa cả 2 phiên bản ([`nghi_phep_nam_v2023.md`](file:///Users/nguyendanglong/Documents/VINUNI/day18/data/nghi_phep_nam_v2023.md) và [`nghi_phep_nam_v2024.md`](file:///Users/nguyendanglong/Documents/VINUNI/day18/data/nghi_phep_nam_v2024.md)) -> LLM do dự do prompt yêu cầu trung thực tuyệt đối với context.
- **Root cause:** Temporal / Versioning ambiguity. Thiếu bộ lọc metadata hoặc quy tắc chỉ dẫn xử lý văn bản bị thay thế (superseded).
- **Suggested fix:** Bổ sung metadata `valid_year: 2024` và thêm System Prompt Rule: "Khi phát hiện các phiên bản chính sách khác nhau trong context, hãy ưu tiên áp dụng chính sách có hiệu lực mới nhất (v2024) và nêu rõ sự thay đổi so với bản cũ".

### #3. Tính toán tiền phạt chậm thanh toán tạm ứng
- **Question:** Nhân viên tạm ứng 15 triệu, sau 20 ngày mới thanh toán. Bị phạt bao nhiêu?
- **Expected:** Thời hạn thanh toán là 15 ngày. Quá hạn 5 ngày, bị tính phí 2%/tháng trên 15.000.000 VNĐ = 300.000 VNĐ/tháng (tính pro-rata khoảng 50.000 VNĐ cho 5 ngày).
- **Got:** Khoản phí là 2%/tháng x 15.000.000 = 300.000 VNĐ/tháng. Tuy nhiên, quy định không nêu cách tính cho 5 ngày quá hạn, nên không xác định được mức phạt chính xác theo số ngày.
- **Worst metric:** `answer_relevancy` (0.0000)
- **Error Tree:** Output nêu đúng công thức tháng nhưng từ chối chia ngày -> Context chỉ có quy định 2%/tháng ([`tam_ung.md`](file:///Users/nguyendanglong/Documents/VINUNI/day18/data/tam_ung.md)) -> LLM tuân thủ chặt chẽ nguyên tắc không tự suy diễn nếu không có chữ "pro-rata theo ngày".
- **Root cause:** Numeric reasoning boundary. Prompt cấm hallucination khiến LLM không dám suy luận tỷ lệ ngày thực tế.
- **Suggested fix:** Cho phép LLM thực hiện phép tính diễn giải số học với ghi chú rõ ràng về giả định pro-rata.

### #4. Đa mục tiêu: Thâm niên phép năm và Khung bảng lương
- **Question:** Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?
- **Expected:** Theo chính sách v2024: 15 ngày cơ bản + 3 ngày thâm niên (9/3=3) = 18 ngày phép. Lương Senior (P3-P4): 20-35 triệu VNĐ/tháng.
- **Got:** Nhân viên Senior có 9 năm thâm niên được 18 ngày phép năm có lương. Context không cung cấp thông tin về khoảng lương.
- **Worst metric:** `context_recall` (0.0000)
- **Error Tree:** Output đúng ngày phép nhưng thiếu hoàn toàn bảng lương -> Context chỉ lấy được [`nghi_phep_nam_v2024.md`](file:///Users/nguyendanglong/Documents/VINUNI/day18/data/nghi_phep_nam_v2024.md), rớt mất [`bang_luong_2024.md`](file:///Users/nguyendanglong/Documents/VINUNI/day18/data/bang_luong_2024.md) -> Retrieval bị nghẽn ở Top-3 Rerank.
- **Root cause:** Multi-intent Query. Câu hỏi ghép 2 chủ đề hoàn toàn tách biệt trong một câu đơn khiến vector search bị lệch trọng tâm sang chủ đề chiếm nhiều token hơn.
- **Suggested fix:** Triển khai Multi-Query expansion để tách các intent độc lập và thực hiện retrieval song song trước khi Rerank.

### #5. Thẩm quyền duyệt nghỉ phép không lương dài ngày
- **Question:** Nghỉ phép không lương 20 ngày cần ai phê duyệt?
- **Expected:** Nghỉ 16-30 ngày cần phê duyệt của Giám đốc điều hành (CEO). Lưu ý: nghỉ trên 14 ngày không lương, nhân viên phải tự đóng phần bảo hiểm của mình.
- **Got:** Giám đốc điều hành (CEO) phê duyệt.
- **Worst metric:** `context_precision` (0.5000)
- **Error Tree:** Output trả lời đúng trọng tâm nhưng context retrieved thừa một số đoạn không liên quan trực tiếp đến mốc 20 ngày.
- **Root cause:** Chunking size lớn mang theo cả các điều khoản về nghỉ 1-5 ngày và 6-15 ngày trong cùng một chunk context.
- **Suggested fix:** Tối ưu hóa kích thước Child Chunk trong [`src/m1_chunking.py`](file:///Users/nguyendanglong/Documents/VINUNI/day18/src/m1_chunking.py) để đạt độ cô đọng cao hơn.

---

## Case Study chuyên sâu: Phân tích Xung đột Văn bản (Version Conflict)

**Câu hỏi chọn phân tích:** "Thâm niên bao nhiêu năm thì được cộng thêm ngày phép?"

**Error Tree Walkthrough:**
1. **Output đúng không?** Output chưa hoàn thiện vì nêu ra mâu thuẫn giữa 2 văn bản thay vì khẳng định chính sách hiện hành có giá trị pháp lý cao nhất.
2. **Context đúng không?** Context retrieved được cả 2 file: [`nghi_phep_nam_v2023.md`](file:///Users/nguyendanglong/Documents/VINUNI/day18/data/nghi_phep_nam_v2023.md) (mỗi 5 năm + 1 ngày) và [`nghi_phep_nam_v2024.md`](file:///Users/nguyendanglong/Documents/VINUNI/day18/data/nghi_phep_nam_v2024.md) (mỗi 3 năm + 1 ngày).
3. **Query rewrite / Hybrid Search hoạt động ra sao?** Hybrid Search tìm kiếm rất nhạy bén khi lấy trúng cả 2 văn bản liên quan đến từ khóa "thâm niên" và "ngày phép".
4. **Điểm cần khắc phục:** Cần bổ sung cơ chế phân loại văn bản hết hiệu lực ngay từ khâu Chunk Enrichment (Module 5) hoặc lọc metadata theo trạng thái văn bản trước khi đưa vào LLM Context.

**Nếu có thêm thời gian phát triển, 3 hướng tối ưu hóa:**
1. **Metadata-based Active Version Filtering:** Gắn cờ `status: active` hoặc `effective_date: 2024` tự động trong M5 Enrichment để tự động loại bỏ các văn bản cũ.
2. **Query Decomposition Agent:** Tự động phát hiện câu hỏi đa ý (Multi-hop) để chia nhỏ thành các sub-queries độc lập.
3. **Adaptive Threshold Reranking:** Tự động điều chỉnh số lượng Top-K context dựa trên độ phân tán điểm số của Cross-Encoder.
