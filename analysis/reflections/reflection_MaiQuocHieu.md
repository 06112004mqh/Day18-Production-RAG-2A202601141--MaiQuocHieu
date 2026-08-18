# Reflection: Production RAG Pipeline — Mai Quốc Hiếu

**Họ và tên:** Mai Quốc Hiếu  
**Mã SV / Lớp:** 2A202601141  
**Ngày thực hiện:** 18/08/2026  

---

## Phần 1: Mapping bài giảng vào Code (Lecture → Project Mapping)

| Lecture Concept | Module | Hàm / Class cụ thể | Observation & Kết quả thực tế |
|----------------|--------|--------------------|--------------------------------|
| **Semantic Chunking** | M1 | `chunk_semantic()` | Dùng OpenAI Embeddings (`text-embedding-3-small`) để tính cosine similarity giữa các câu liên tiếp, phân nhóm câu cùng ngữ cảnh mà không cắt nát đoạn văn. |
| **Hierarchical Chunking** | M1 | `chunk_hierarchical()` | Tạo cặp Parent (2048 chars) - Child (256 chars). Đảm bảo từng Child chunk lưu đúng `parent_id`, giúp việc Retrieval chính xác ở mức Child và trả lại full context ở mức Parent. |
| **Structure-Aware Chunking** | M1 | `chunk_structure_aware()` | Parse các Header Markdown (`#`, `##`, `###`), bảo toàn cấu trúc bảng biểu, code block và danh sách, gán metadata `section` cho từng chunk. |
| **BM25 + Dense Fusion (RRF)** | M2 | `reciprocal_rank_fusion()` | Kết hợp BM25 (đã được tách từ tiếng Việt qua `underthesea`) và Dense Search (Qdrant Vector DB). Công thức Reciprocal Rank Fusion ghép top-20 kết quả của 2 phương pháp giúp cân bằng keyword search và semantic search. |
| **Cosine / Cross-Encoder Reranking** | M3 | `CrossEncoderReranker.rerank()` | Rerank danh sách candidate top-20 từ Hybrid Search xuống Top 3. Sử dụng OpenAI vector cosine scoring giúp loại bỏ noise và nâng điểm Context Precision lên 90-93%. |
| **RAGAS 4 Metrics Evaluation** | M4 | `evaluate_ragas()` | Đánh giá tự động pipeline theo 4 tiêu chí chuẩn: Faithfulness (0.6050), Answer Relevancy (0.5530), Context Precision (0.9333), Context Recall (0.7750). |
| **Diagnostic Tree Failure Analysis** | M4 | `failure_analysis()` | Tự động phân loại lỗi từ các câu hỏi có điểm số thấp nhất để chỉ ra nguyên nhân gốc (Retrieval vs LLM Generation) và đề xuất hướng sửa chữa. |
| **Single-Call Chunk Enrichment** | M5 | `_enrich_single_call()` | Gộp 4 thao tác (Summarize, HyQA, Contextual Prepend, Auto Metadata) vào **1 LLM Call/chunk** giúp tối ưu 75% chi phí API OpenAI. |

---

## Phần 2: Khó khăn & Giải pháp (Challenges & Debugging)

### 1. Khó khăn về hạ tầng phần cứng & Mô hình cục bộ (Local Models)
- **Vấn đề (Exact Error):** Máy cá nhân phần cứng giới hạn, việc tải và chạy các mô hình local nặng như `BAAI/bge-m3` (Vector Embedding) hay `BAAI/bge-reranker-v2-m3` gây chiếm dung lượng RAM/VRAM lớn, dẫn đến giật lag và lỗi crash timeout.
- **Cách Debug & Giải pháp:**
  - Chuyển đổi toàn bộ kiến trúc Embedding và Scoring sang sử dụng **OpenAI API**: `text-embedding-3-small` (dimension 1536) cho Embeddings & Cosine Reranking, và `gpt-4o-mini` cho Generation & Enrichment.
  - Tích hợp cơ chế fallback linh hoạt trong tất cả các module (`m1_chunking`, `m2_search`, `m3_rerank`, `m5_enrichment`) để vừa ưu tiên OpenAI API tốc độ cao, vừa đảm bảo runnable trên môi trường offline nếu không có API key.

### 2. Lỗi serialization dữ liệu NumPy khi xuất báo cáo JSON
- **Vấn đề:** Khi chạy `save_report()`, Python ném ngoại lệ `TypeError: Object of type ndarray is not JSON serializable` do `ragas.evaluate` và `pandas` trả về các kiểu dữ liệu `np.ndarray` và `np.str_`.
- **Cách xử lý:** Viết hàm đệ quy `_convert()` biến đổi toàn bộ NumPy types, `ndarray`, `np.generic` về kiểu dữ liệu gốc Python (`float`, `int`, `list`, `str`) và truyền `default=str` vào `json.dump()`.

---

## Phần 3: Action Plan áp dụng vào Dự án Cá nhân

### Dự án: Hệ thống RAG hỗ trợ Tra cứu Văn bản & Quy trình Doanh nghiệp

#### 1. Hiện tại
- **RAG Pipeline hiện tại:** Naive Baseline (Split text cố định 500 chars, Dense search thuần túy).
- **Known issues:**
  - Cắt vụn văn bản gây mất ngữ cảnh câu lệnh.
  - Câu hỏi chứa thuật ngữ viết tắt tiếng Việt không match được bằng Dense Vector.
  - Không có metric đánh giá tự động chất lượng câu trả lời.

#### 2. Kế hoạch áp dụng kỹ thuật
1. **[x] Chunking strategy:** Áp dụng **Hierarchical Parent-Child Chunking** kết hợp **Structure-Aware** cho các tài liệu chính sách dạng Markdown/PDF.
2. **[x] Search:** Áp dụng **Hybrid Search (BM25 tiếng Việt + Qdrant Dense Vector + RRF)** để giải quyết vấn đề từ khóa chuyên ngành.
3. **[x] Reranking:** Sử dụng **Reranker** để tinh lọc Top-20 xuống Top-3 trước khi đưa vào LLM Context Window.
4. **[x] Enrichment:** Thực hiện **Contextual Prepend & HyQA** (Single-call mode) cho các đoạn văn bản chính sách quan trọng.
5. **[x] Evaluation:** Sử dụng khung đánh giá **RAGAS** để theo dõi hồi quy (regression tracking) mỗi khi thay đổi Prompt hoặc Vector Model.

#### 3. Timeline triển khai
- **Tuần 1:** Chuẩn hóa dữ liệu đầu vào (tách từ tiếng Việt underthesea + Markdown Structure Parser).
- **Tuần 2:** Dựng Qdrant Vector DB & Triển khai Hybrid Search RRF.
- **Tuần 3:** Tích hợp Reranker & Single-call Chunk Enrichment với OpenAI `gpt-4o-mini`.
- **Tuần 4:** Thiết lập RAGAS Auto Evaluation & Dashboard giám sát chỉ số Faithfulness / Recall.
