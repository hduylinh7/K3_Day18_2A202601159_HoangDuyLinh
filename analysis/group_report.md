# Group Report — Lab 18: Production RAG Pipeline

**Nhóm:** Bài tập cá nhân  
**Họ tên học viên:** Hoàng Duy Linh (MSSV: 2A202601159)  
**Ngày hoàn thành:** 18/08/2026

---

## Thành viên & Phân công

| Tên | Module | Hoàn thành | Tests pass |
|-----|--------|-----------|-----------|
| Hoàng Duy Linh | M1: Chunking | ☑ | 13/13 |
| Hoàng Duy Linh | M2: Hybrid Search | ☑ | 5/5 |
| Hoàng Duy Linh | M3: Reranking | ☑ | 5/5 |
| Hoàng Duy Linh | M4: Evaluation | ☑ | 4/4 |
| Hoàng Duy Linh | M5: Enrichment Pipeline | ☑ | 10/10 |

---

## Kết quả RAGAS

| Metric | Naive | Production | Δ |
|--------|-------|-----------|---|
| Faithfulness | 0.5820 | 0.8950 | +0.3130 |
| Answer Relevancy | 0.6140 | 0.8810 | +0.2670 |
| Context Precision | 0.5200 | 0.8420 | +0.3220 |
| Context Recall | 0.5400 | 0.8650 | +0.3250 |

---

## Key Findings

1. **Biggest improvement:**  
   **Hybrid Search (BM25 + Dense + RRF) kết hợp Reranking (M2 + M3):** Việc kết hợp từ khóa tiếng Việt (đã tách từ bằng Underthesea) và vector embedding qua RRF giúp cải thiện điểm Context Recall và Context Precision lên **> 32%** so với Naive Vector Search đơn thuần.

2. **Biggest challenge:**  
   Xử lý tiếng Việt bị đứt từ trong BM25 (dấu gạch nối `_` của Underthesea) và quản lý Latency khi dùng Cross-Encoder Reranker (`BAAI/bge-reranker-v2-m3`). Giải pháp là thay thế `_` thành dấu cách và áp dụng Model Caching (Lazy Loading).

3. **Surprise finding:**  
   **Hierarchical (Parent-Child) Chunking (M1)** vượt trội hơn hẳn Basic Paragraph Chunking khi trả về ngữ cảnh cho LLM: Child chunk (256 chars) giúp tìm đúng vị trí, nhưng Parent chunk (2048 chars) cung cấp đủ thông tin xung quanh để LLM trả lời chính xác mà không bị hallucinate.

---

## Presentation Notes (5 phút)

1. **RAGAS scores (naive vs production):**  
   Tất cả 4 chỉ số RAGAS đều tăng mạnh từ mức trung bình ~0.56 lên **~0.87** nhờ kiến trúc 5 tầng chuẩn Production.
2. **Biggest win — module nào, tại sao:**  
   **Module 1 & Module 2:** Hierarchical Chunking giải quyết tận gốc vấn đề đứt ngữ cảnh; BM25 tiếng Việt + Dense RRF đảm bảo không bỏ sót từ khóa chuyên ngành.
3. **Case study — 1 failure, Error Tree walkthrough:**  
   Phân tích câu hỏi về "Thâm niên và ngày phép năm" -> Lỗi do retrieve nhầm phiên bản v2023 cũ -> Khắc phục bằng Metadata Filter & Contextual Prepend.
4. **Next optimization nếu có thêm 1 giờ:**  
   Tích hợp Query Rewriting / HyDE (Hypothetical Document Embeddings) và Metadata Filtering tự động theo thời gian phiên bản.
