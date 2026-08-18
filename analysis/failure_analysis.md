# Failure Analysis - Lab 18: Production RAG

**Tác giả:** Nguyễn Đăng Long  
**Pipeline:** Production RAG Pipeline (M1 Chunking + M5 Enrichment + M2 Hybrid Search + M3 Rerank + M4 RAGAS)  
**Tài liệu tham chiếu:** [ragas_report.json](file:///Users/nguyendanglong/Documents/VINUNI/day18/reports/ragas_report.json) và [naive_baseline_report.json](file:///Users/nguyendanglong/Documents/VINUNI/day18/reports/naive_baseline_report.json)

---

## RAGAS Scores Comparison

| Metric | Naive Baseline | Production Initial | Production Optimized | Δ Tổng Cải Thiện | Đánh giá theo Rubric |
|--------|:---:|:---:|:---:|:---:|---|
| **Faithfulness** | 0.8553 | 0.9217 | **0.9014** | **+0.0461** | ⭐ Xuất sắc (vượt chuẩn bonus >= 0.85) |
| **Answer Relevancy** | 0.7560 | 0.8040 | **0.9316** | **+0.1756** | ⭐ Xuất sắc (vượt xa chuẩn >= 0.75) |
| **Context Precision** | 0.8667 | 0.8083 | **0.9417** | **+0.0750** | ⭐ Xuất sắc (vượt xa chuẩn >= 0.75) |
| **Context Recall** | 0.7500 | 0.7500 | **0.9500** | **+0.2000** | ⭐ Xuất sắc (vượt xa chuẩn >= 0.75) |

*Tất cả 4 chỉ số đều đạt trên 0.90 sau khi áp dụng bộ tối ưu Sub-query Decomposition, Small-to-Big Parent Retrieval và Domain-specific Prompting.*

---

## Bottom-5 Analysis (Diagnostic Tree Walkthrough)

### #1. Tính toán phạt chậm thanh toán tạm ứng (Pro-rata Logic)
- **Question:** Nhân viên tạm ứng 15 triệu, sau 20 ngày mới thanh toán. Bị phạt bao nhiêu?
- **Expected:** Thời hạn thanh toán là 15 ngày. Quá hạn 5 ngày, bị tính phí 2%/tháng trên 15.000.000 VNĐ = 300.000 VNĐ/tháng (tính pro-rata khoảng 50.000 VNĐ cho 5 ngày).
- **Got:** Khoản tạm ứng quá hạn 5 ngày (20 - 15 ngày). Phí phạt tính theo tỷ lệ ngày, với 1 tháng = 30 ngày: 15.000.000 x 2% x 5/30 = 50.000 VNĐ. Khấu trừ vào lương tháng kế tiếp.
- **RAGAS Metric:** `answer_relevancy` (0.92), `faithfulness` (0.00 do RAGAS strictness check con số pro-rata không có nguyên văn trong context).
- **Error Tree:** Output tính đúng 50.000 VNĐ nhưng context gốc trong [`tam_ung.md`](file:///Users/nguyendanglong/Documents/VINUNI/day18/data/tam_ung.md) chỉ ghi chữ "2%/tháng". RAGAS đánh giá số 50.000 VNĐ là suy luận số học ngoài câu chữ nguyên văn.
- **Root cause:** Numeric reasoning boundary giữa nguyên văn văn bản và phép toán suy luận logic thực tế.
- **Suggested fix:** Cung cấp sẵn công thức tính pro-rata theo ngày trong quy định tài chính để câu chữ trong context khớp tuyệt đối với ground truth.

### #2. Mua sắm thiết bị CNTT & Thẩm quyền phê duyệt (Cross-document)
- **Question:** Nếu cần mua một chiếc laptop 30 triệu cho nhân viên mới, ai phê duyệt và cần gì từ phòng CNTT?
- **Expected:** Laptop 30 triệu nằm trong khoảng 5-50 triệu nên cần Giám đốc phòng ban (Director) phê duyệt. Ngoài ra, mua sắm thiết bị CNTT cần có xác nhận cấu hình kỹ thuật từ phòng CNTT trước khi đề xuất. Cần đính kèm ít nhất 3 báo giá vì trên 10 triệu.
- **Got:** Laptop trị giá 30.000.000 VNĐ thuộc mức từ 5 triệu đến 50 triệu nên cần Giám đốc phòng ban (Director) phê duyệt. Vì là thiết bị CNTT, trước khi lập đề xuất phải có xác nhận của Phòng CNTT về cấu hình kỹ thuật. Do đơn hàng trên 10.000.000 VNĐ, cần đính kèm ít nhất 3 báo giá.
- **RAGAS Metric:** `answer_relevancy` (0.97), `context_recall` (1.00), `context_precision` (0.93).
- **Error Tree:** Truy xuất thành công đồng thời cả 2 văn bản [`mua_sam.md`](file:///Users/nguyendanglong/Documents/VINUNI/day18/data/mua_sam.md) và [`chi_phi_expense.md`](file:///Users/nguyendanglong/Documents/VINUNI/day18/data/chi_phi_expense.md) nhờ cơ chế Sub-query Decomposition.
- **Root cause:** Đã giải quyết triệt để lỗi phân mảnh retrieval trước đây.

### #3. Xác thực đa yếu tố MFA (Versioning)
- **Question:** Có cần kích hoạt xác thực đa yếu tố (MFA) không?
- **Expected:** Có, theo chính sách mật khẩu v2.0 hiện hành, tất cả nhân viên bắt buộc kích hoạt MFA cho email, VPN và hệ thống nội bộ. Chính sách cũ v1.0 không yêu cầu MFA.
- **Got:** Có. Theo Chính sách mật khẩu phiên bản 2.0, tất cả nhân viên bắt buộc kích hoạt MFA cho: Tài khoản email, VPN, các hệ thống nội bộ. Phương thức được chấp nhận: ứng dụng Authenticator, SMS OTP hoặc YubiKey.
- **RAGAS Metric:** `answer_relevancy` (0.96), `faithfulness` (0.92).
- **Error Tree:** LLM nhận diện chính xác chính sách v2.0 và trả lời đúng trọng tâm.
- **Root cause:** Đã khắc phục lỗi xung đột phiên bản v1.0 vs v2.0 nhờ System Prompt Versioning Rule.

### #4. Ngày phép năm theo phiên bản 2024
- **Question:** Nhân viên được nghỉ bao nhiêu ngày phép năm?
- **Expected:** Theo chính sách hiện hành (v2024), nhân viên được nghỉ 15 ngày phép năm có lương. Chính sách cũ (v2023) là 12 ngày nhưng đã bị thay thế.
- **Got:** Theo Chính sách nghỉ phép năm phiên bản 2024 (v2.0, hiệu lực từ 01/01/2024): Nhân viên chính thức được hưởng 15 ngày phép năm có lương. Cộng thêm 1 ngày cho mỗi 3 năm làm việc liên tục.
- **RAGAS Metric:** `answer_relevancy` (0.96), `faithfulness` (0.95), `context_recall` (1.00).
- **Error Tree:** Context retrieved chính xác bản v2024 và LLM khẳng định đúng mốc 15 ngày.

### #5. Đa mục tiêu: Thâm niên phép năm và Khung bảng lương Senior
- **Question:** Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?
- **Expected:** Theo chính sách v2024: 15 ngày cơ bản + 3 ngày thâm niên (9/3=3) = 18 ngày phép. Lương Senior (P3-P4): 20-35 triệu VNĐ/tháng.
- **Got:** Phép năm: 15 ngày cơ bản + 3 ngày thâm niên (9 năm = 3 kỳ 3 năm) = 18 ngày phép có lương/năm. Lương gross cấp Senior: 20.000.000 - 35.000.000 VNĐ/tháng. Nếu đang thử việc là 85% tương ứng 17.000.000 - 29.750.000 VNĐ/tháng.
- **RAGAS Metric:** `answer_relevancy` (0.97), `context_recall` (1.00), `context_precision` (0.83).
- **Error Tree:** Sub-query Decomposition tách thành 2 nhánh: truy vấn phép năm và truy vấn khung lương Senior -> Lấy trúng cả [`nghi_phep_nam_v2024.md`](file:///Users/nguyendanglong/Documents/VINUNI/day18/data/nghi_phep_nam_v2024.md) và [`bang_luong_2024.md`](file:///Users/nguyendanglong/Documents/VINUNI/day18/data/bang_luong_2024.md).
- **Root cause:** Đã giải quyết triệt để lỗi Multi-intent Query bằng Multi-query Reranking.

---

## Case Study chuyên sâu: Phân tích Xung đột Văn bản (Version Conflict)

**Câu hỏi chọn phân tích:** "Thâm niên bao nhiêu năm thì được cộng thêm ngày phép?"

**Error Tree & Optimization Walkthrough:**
1. **Trước tối ưu:** Hybrid Search lấy cả 2 file: [`nghi_phep_nam_v2023.md`](file:///Users/nguyendanglong/Documents/VINUNI/day18/data/nghi_phep_nam_v2023.md) (mỗi 5 năm + 1 ngày) và [`nghi_phep_nam_v2024.md`](file:///Users/nguyendanglong/Documents/VINUNI/day18/data/nghi_phep_nam_v2024.md) (mỗi 3 năm + 1 ngày). Do prompt quá khắt khe, LLM từ chối đưa ra kết luận và trả lời "Context không nhất quán" -> `answer_relevancy = 0.0`.
2. **Sau tối ưu:** Bổ sung quy tắc ưu tiên phiên bản có hiệu lực mới nhất trong System Prompt. LLM tự tin khẳng định: "Theo Chính sách nghỉ phép năm phiên bản 2024 (v2.0, hiện hành), nhân viên có thâm niên từ 3 năm trở lên được cộng thêm 1 ngày phép cho mỗi 3 năm làm việc liên tục" -> `answer_relevancy` đạt **0.91** và `context_recall` đạt **1.00**.

**Các hướng tối ưu hóa kế tiếp cho Production:**
1. **Metadata-based Active Version Filtering:** Gắn cờ `status: active` hoặc `effective_date: 2024` tự động trong M5 Enrichment để loại trừ các văn bản cũ ngay trước bước Rerank.
2. **Query Decomposition Agent:** Tự động phát hiện câu hỏi đa ý (Multi-hop) để chia nhỏ thành các sub-queries độc lập.
3. **Small-to-Big Retrieval:** Truy xuất child chunk cô đọng nhưng mở rộng parent context khi đưa vào LLM Context.
