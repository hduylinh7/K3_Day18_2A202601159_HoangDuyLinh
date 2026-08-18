# Individual Reflection — Lab 18: Production RAG Pipeline

**Tên:** Hoàng Duy Linh  
**Mã SV:** 2A202601159  
**Module phụ trách:** Toàn bộ Module 1 → Module 5 (Bài tập cá nhân)

---

## 1. Đóng góp kỹ thuật & Mapping Bài giảng

| Lecture Concept | Module | Hàm / Class cụ thể trong Code | Nhận xét / Observation |
|---|---|---|---|
| **Semantic Chunking** | M1 | `chunk_semantic()` | Tách câu bằng Regex, dùng `all-MiniLM-L6-v2` tính Cosine Sim. Giữ nguyên trọn vẹn ngữ nghĩa từng chủ đề. |
| **Hierarchical (Parent-Child)** | M1 | `chunk_hierarchical()` | Tạo Parent (2048) & Child (256). Search trên Child để đạt precision, gửi Parent cho LLM để có context depth. |
| **Structure-Aware Chunking** | M1 | `chunk_structure_aware()` | Parse Markdown Headers (`#`, `##`), lưu tên mục vào `metadata.section`. Không làm vỡ bảng biểu / code. |
| **Vietnamese BM25 + Dense Fusion** | M2 | `segment_vietnamese()`, `BM25Search`, `DenseSearch`, `reciprocal_rank_fusion()` | Dùng `underthesea` tách từ ghép (thay `_` bằng ` `), kết hợp BM25 & Qdrant Dense qua thuật toán RRF $1/(k+r)$. |
| **Cross-Encoder Reranking** | M3 | `CrossEncoderReranker.rerank()` | Dùng `BAAI/bge-reranker-v2-m3` đánh giá cặp (Query, Doc). Rerank Top-20 về Top-3. |
| **RAGAS Evaluation & Error Tree** | M4 | `evaluate_ragas()`, `failure_analysis()` | Đánh giá 4 chỉ số (Faithfulness, Relevancy, Precision, Recall) và tự động chẩn đoán nguyên nhân thất bại. |
| **Contextual Prepend & Enrichment** | M5 | `summarize_chunk()`, `generate_hypothesis_questions()`, `contextual_prepend()`, `_enrich_single_call()` | Bổ sung ngữ cảnh vị trí tài liệu (Anthropic style), câu hỏi giả định (HyQA) và tóm tắt chunk trước khi index. |

---

## 2. Khó khăn & Cách giải quyết (Real Debugging Experience)

1. **Lỗi `underthesea` nối từ ghép bằng `_` trong BM25 (M2):**
   * *Hiện tượng:* Query "nghỉ phép" không tìm thấy tài liệu có chứa "nghỉ_phép".
   * *Nguyên nhân:* BM25 split theo khoảng trắng nên câu thành token `nghỉ_phép`, còn query tách thành 2 token `nghỉ` và `phép`.
   * *Giải pháp:* Đã bổ sung `.replace("_", " ")` trong `segment_vietnamese()` để đồng bộ tokenization.

2. **Lỗi Reload Model làm treo / giật lag trong Reranking (M3):**
   * *Hiện tượng:* Hàm `rerank()` bị chậm bất thường và báo timeout.
   * *Nguyên nhân:* `_load_model()` khởi tạo lại `CrossEncoder` ở mỗi lần gọi do đặt lệnh `self._model = ...` ngoài khối `if self._model is None:`.
   * *Giải pháp:* Sửa lại cơ chế Caching (Lazy Loading) chuẩn xác để model chỉ load đúng 1 lần duy nhất vào bộ nhớ.

3. **Tương thích Groq / LLM Provider linh hoạt (M5 & Pipeline):**
   * *Hiện tượng:* Cần dùng Groq API thay vì OpenAI.
   * *Giải pháp:* Bổ sung helper `get_llm_client()` hỗ trợ tự động nhận diện `GROQ_API_KEY` (model `llama-3.3-70b-versatile`) và tích hợp sẵn chế độ Extractive Fallback khi không có API key.

---

## 3. Action Plan áp dụng vào Project cá nhân

### Project: Hệ thống RAG Hỏi đáp Quy chế & Tài liệu Doanh nghiệp

#### Tình trạng hiện tại:
* Đang dùng Naive Chunking (tách theo 500 ký tự) làm mất thông tin điều khoản hợp đồng.
* Chưa có Reranker nên kết quả retrieval còn nhiều nhiễu.

#### Plan áp dụng từ Lab 18:
1. **Chunking Strategy:** Chuyển sang **Hierarchical Chunking (Parent 2048 / Child 256)** kết hợp **Structure-Aware Chunking** cho các tài liệu quy trình dạng Markdown.
2. **Search Engine:** Cài đặt **Hybrid Search (BM25 + Qdrant Dense)** với Underthesea tokenizer cho tiếng Việt.
3. **Reranking:** Tích hợp `bge-reranker-v2-m3` để lọc lại Top-3 kết quả tốt nhất trước khi đưa vào LLM.
4. **Enrichment:** Sử dụng kĩ thuật **Contextual Prepend** để đính kèm tên văn bản / chương mục vào từng chunk.
5. **Evaluation:** Thiết lập bộ test set 50 câu và chạy **RAGAS Evaluation** hàng tuần để giám sát chất lượng Pipeline.

---

## 4. Tự đánh giá

| Tiêu chí | Tự chấm (1-5) | Ghi chú |
|----------|---------------|---------|
| Hiểu bài giảng | 5/5 | Nắm vững toàn bộ kiến thức 5 modules trong Production RAG Pipeline |
| Code quality | 5/5 | Code sạch, truyền tham số chuẩn, hỗ trợ fallback & caching đầy đủ |
| Problem solving | 5/5 | Tự debug và khắc phục các lỗi về tokenizer, model caching và LLM integration |
| Delivery | 5/5 | Hoàn thành 100% yêu cầu bài lab và đẩy đủ các file báo cáo phân tích |
