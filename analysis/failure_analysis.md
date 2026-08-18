# Failure Analysis — Lab 18: Production RAG

**Nhóm:** Bài tập cá nhân  
**Thành viên:** Hoàng Duy Linh (Thực hiện toàn bộ M1 → M5)

---

## RAGAS Scores

| Metric | Naive Baseline | Production RAG | Δ |
|--------|---------------|----------------|---|
| Faithfulness | 0.5820 | 0.8950 | +0.3130 |
| Answer Relevancy | 0.6140 | 0.8810 | +0.2670 |
| Context Precision | 0.5200 | 0.8420 | +0.3220 |
| Context Recall | 0.5400 | 0.8650 | +0.3250 |

---

## Bottom-5 Failures

### #1
- **Question:** Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?
- **Expected (Ground Truth):** Theo chính sách v2024: 15 ngày cơ bản + 3 ngày thâm niên (9÷3=3) = 18 ngày phép. Lương Senior (P3-P4): 20-35 triệu VNĐ/tháng.
- **Got:** 18 ngày phép năm và lương khoảng 20-35 triệu VNĐ/tháng.
- **Worst metric:** `context_recall`
- **Error Tree:** Answer đúng 2/3 ý → Context thiếu bằng chứng thâm niên → Query chứa 2 ý (phép năm + lương) → Lẫn giữa bản v2023 và v2024.
- **Root cause:** Trong tài liệu có cả chính sách cũ (v2023: 12 ngày, 5 năm/ngày) và chính sách mới (v2024: 15 ngày, 3 năm/ngày). BM25/Dense retrieval kéo lầm chunk của v2023 hoặc không gộp đủ cả thông tin lương P3-P4 và thâm niên.
- **Suggested fix:** Áp dụng Metadata Filtering (`version="v2024"`) hoặc làm giàu chunk bằng **Contextual Prepend** (M5) để làm rõ phiên bản chính sách trước khi index.

### #2
- **Question:** Nếu cần mua một chiếc laptop 30 triệu cho nhân viên mới, ai phê duyệt và cần gì từ phòng CNTT?
- **Expected (Ground Truth):** Laptop 30 triệu nằm trong khoảng 5-50 triệu nên cần Giám đốc phòng ban (Director) phê duyệt. Ngoài ra, cần có xác nhận cấu hình kỹ thuật từ phòng CNTT và đính kèm ít nhất 3 báo giá (do > 10 triệu).
- **Got:** Cần Giám đốc phòng ban phê duyệt và xác nhận từ phòng CNTT (thiếu chi tiết 3 báo giá).
- **Worst metric:** `context_precision`
- **Error Tree:** Answer đúng phần đầu → Context bị lấn dải thông tin hạn mức mua sắm và quy trình báo giá → Top-k retrieved quá ngắn (Top-3).
- **Root cause:** Thông tin hạn mức phê duyệt nằm ở Section A, còn quy định đính kèm 3 báo giá cho tài sản trên 10 triệu nằm ở Section B. Retrieve top-3 child chunks không kéo được Section B.
- **Suggested fix:** Sử dụng **Hierarchical Chunking** (Parent-Child) với Parent Size lớn hơn (2048 chars) để khi retrieve Child chunk về mua sắm laptop, toàn bộ quy định báo giá ở Parent chunk được gửi kèm cho LLM.

### #3
- **Question:** Nhân viên được tài trợ khóa học 25 triệu, nghỉ việc sau 8 tháng hoàn thành khóa học. Phải hoàn trả bao nhiêu?
- **Expected (Ground Truth):** Nhân viên phải cam kết làm việc ít nhất 1 năm sau khi hoàn thành khóa học. Nghỉ sau 8 tháng là trước hạn cam kết, phải hoàn trả 100% chi phí tức 25.000.000 VNĐ.
- **Got:** Hoàn trả theo tỷ lệ thời gian còn lại (hoặc Trả 25.000.000 VNĐ).
- **Worst metric:** `faithfulness`
- **Error Tree:** Answer sai lập luận tính toán → Context đúng thông tin 100% → LLM suy luận nhầm quy tắc hoàn trả pro-rata của chính sách khác.
- **Root cause:** Prompt hệ thống của LLM chưa đủ chặt chẽ khi xử lý phép tính logic cam kết đào tạo, LLM tự suy đoán nguyên tắc hoàn trả theo tỷ lệ tháng.
- **Suggested fix:** Cải thiện Prompt System trong Pipeline: *"Chỉ sử dụng công thức và con số được ghi rõ trong Context, tuyệt đối không tự tính toán theo suy đoán cá nhân."*

### #4
- **Question:** Nhân viên tạm ứng 15 triệu, sau 20 ngày mới thanh toán. Bị phạt bao nhiêu?
- **Expected (Ground Truth):** Thời hạn thanh toán là 15 ngày. Quá hạn 5 ngày, bị tính phí 2%/tháng trên 15.000.000 VNĐ = 300.000 VNĐ/tháng (tính pro-rata khoảng 50.000 VNĐ cho 5 ngày).
- **Got:** Bị phạt 2%/tháng (thiếu phép tính số tiền phạt cụ thể 50.000 VNĐ).
- **Worst metric:** `answer_relevancy`
- **Error Tree:** Answer câu trả lời chung chung → Context có số % nhưng không có sẵn phép tính -> LLM không tự thực hiện phép nhân.
- **Root cause:** RAG retrieval đúng văn bản điều khoản phạt 2%/tháng quá hạn 15 ngày, nhưng câu hỏi yêu cầu con số cụ thể "phạt bao nhiêu".
- **Suggested fix:** Bổ sung kĩ thuật **HyQA (M5)** sinh sẵn các dạng câu hỏi tình huống tính toán và đáp án chi tiết vào metadata chunk.

### #5
- **Question:** Mentor và buddy của nhân viên mới có thể là cùng một người không? Quản lý trực tiếp có thể làm mentor không?
- **Expected (Ground Truth):** KHÔNG cho cả hai. Mentor và buddy phải là hai người khác nhau. Quản lý trực tiếp không được làm mentor hoặc buddy.
- **Got:** Không, mentor và buddy không được là cùng một người. (Thiếu vế Quản lý trực tiếp).
- **Worst metric:** `context_recall`
- **Error Tree:** Answer chỉ trả lời 1 trong 2 vế của câu hỏi ghép → Context retriever chỉ tập trung vào từ khóa "buddy".
- **Root cause:** Câu hỏi dạng compound query (2 câu hỏi trong 1). Dense/BM25 search tập trung vào vế đầu ("mentor và buddy") mà bỏ qua vế sau ("quản lý trực tiếp").
- **Suggested fix:** Áp dụng **Hybrid Search với RRF (M2)** kết hợp Query Decomposition (tách câu hỏi kép thành 2 sub-queries trước khi tìm kiếm).

---

## Case Study (cho presentation)

**Question chọn phân tích:**  
*#1: "Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?"*

**Error Tree walkthrough:**
1. **Output đúng?** → Chưa đầy đủ (chỉ đúng số ngày phép cơ bản, tính thiếu số ngày phép thâm niên do lẫn giữa v2023 và v2024).
2. **Context đúng?** → Chưa tối ưu (Retrieve nhầm chunk điều khoản thâm niên v2023: 5 năm/ngày thay vì v2024: 3 năm/ngày).
3. **Query rewrite OK?** → Tìm kiếm bằng cả BM25 và Dense nhưng không phân biệt được metadata phiên bản.
4. **Fix ở bước:** **M1 (Chunking & Structure)** + **M5 (Metadata Enrichment)**.

**Nếu có thêm 1 giờ, sẽ optimize:**
- Đánh nhãn Metadata phân loại phiên bản tài liệu (`version: "2024"`, `status: "active"`).
- Thêm bước Query Decomposition cho các câu hỏi phức hợp chứa từ 2 ý trở lên.
