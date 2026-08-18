# Failure Analysis — Lab 18: Production RAG

**Nhóm:** Cá nhân  
**Thành viên:** Mai Quốc Hiếu (M1, M2, M3, M4, M5)

---

## RAGAS Scores

| Metric | Naive Baseline | Production RAG | Δ |
|--------|---------------|----------------|---|
| Faithfulness | 0.0000 | 0.7183 | +0.7183 |
| Answer Relevancy | 0.0000 | 0.6402 | +0.6402 |
| Context Precision | 0.0000 | 0.9000 | +0.9000 |
| Context Recall | 0.0000 | 0.7167 | +0.7167 |

---

## Bottom-5 Failures

### #1
- **Question:** Bảo hiểm sức khỏe PVI có hạn mức bao nhiêu cho nhân viên?
- **Expected:** Hạn mức bảo hiểm sức khỏe PVI cho nhân viên là 200.000.000 VNĐ/năm, bao gồm nội trú, ngoại trú và nha khoa.
- **Got:** Không tìm thấy.
- **Worst metric:** `context_recall` / `faithfulness` (0.00)
- **Error Tree:** Output sai ➔ Context bị thiếu chunk chứa thông tin hạn mức PVI.
- **Root cause:** Chunking chưa giữ trọn vẹn phần metadata hạn mức bảo hiểm PVI.
- **Suggested fix:** Cải thiện chunking strategy, dùng Parent-Child Hierarchical Chunking kích thước lớn hơn hoặc thêm Metadata Enrichment cho các file HR policy.

### #2
- **Question:** Thông tin lương thuộc cấp độ phân loại dữ liệu nào?
- **Expected:** Theo quy chế chi trả lương, thông tin lương được phân loại là dữ liệu Bí mật, cấm chia sẻ với đồng nghiệp. Dữ liệu Bí mật (cấp 3) phải mã hóa khi truyền.
- **Got:** Không tìm thấy.
- **Worst metric:** `context_recall` (0.00)
- **Error Tree:** Output sai ➔ Context thiếu ➔ Dense search không match được từ khóa chính xác "Bí mật (cấp 3)".
- **Root cause:** Từ khóa "phân loại dữ liệu" rơi vào chunk chính sách an toàn thông tin nhưng phần lương nằm ở chunk quy chế lương.
- **Suggested fix:** Tăng trọng số BM25 trong Hybrid Search để bắt từ khóa "phân loại dữ liệu" + "lương".

### #3
- **Question:** Bao lâu phải đổi mật khẩu một lần?
- **Expected:** Mật khẩu phải được thay đổi mỗi 120 ngày (chính sách v2.0).
- **Got:** Không tìm thấy.
- **Worst metric:** `context_recall` (0.00)
- **Error Tree:** Output sai ➔ Search trả về chunk chính sách cũ (90 ngày) hoặc bị Reranker lọc mất chunk v2.0.
- **Root cause:** Xung đột phiên bản giữa chính sách cũ (90 ngày) và chính sách v2.0 (120 ngày).
- **Suggested fix:** Thêm Metadata Version Filtering vào Qdrant để ưu tiên phiên bản v2.0 mới nhất.

### #4
- **Question:** Nhân viên được tài trợ khóa học 25 triệu, nghỉ việc sau 8 tháng hoàn thành khóa học. Phải hoàn trả bao nhiêu?
- **Expected:** Nhân viên phải cam kết làm việc ít nhất 1 năm sau khi hoàn thành. Nghỉ sau 8 tháng phải hoàn trả 100% chi phí (25.000.000 VNĐ).
- **Got:** Không tìm thấy.
- **Worst metric:** `context_recall` / `faithfulness` (0.00)
- **Error Tree:** Output sai ➔ Context lấy được phần tài trợ nhưng thiếu phần quy định cam kết đào tạo.
- **Root cause:** Đoạn văn bản về "cam kết đào tạo" nằm ở section khác với đoạn "chi phí tài trợ 25 triệu".
- **Suggested fix:** Dùng HyQA (Hypothesis Questions) để generate câu hỏi liên quan đến tính toán hoàn trả học phí khi nghỉ việc trước hạn.

### #5
- **Question:** Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?
- **Expected:** 15 ngày cơ bản + 3 ngày thâm niên = 18 ngày phép. Lương Senior (P3-P4): 20-35 triệu VNĐ/tháng.
- **Got:** Nhân viên có 9 năm thâm niên được 18 ngày phép (15 + 3). Không tìm thấy thông tin về lương trong context.
- **Worst metric:** `answer_relevancy` (0.00)
- **Error Tree:** Output đúng 1 phần ➔ Context lấy được bảng phép nhưng thiếu bảng lương Senior.
- **Root cause:** Câu hỏi đa ý (Multi-hop question: vừa hỏi phép năm vừa hỏi thang lương).
- **Suggested fix:** Áp dụng Query Decomposition (tách câu hỏi phức hợp thành 2 sub-queries: [Phép năm Senior 9 năm] và [Thang lương Senior]).

---

## Case Study (cho presentation)

**Question chọn phân tích:** *Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?*

**Error Tree walkthrough:**
1. Output đúng? ➔ Không, chỉ trả lời được ý 1 (18 ngày phép), thiếu ý 2 (thang lương 20-35 triệu).
2. Context đúng? ➔ Context chỉ retrieval được chunk quy định phép năm, không retrieve được chunk thang lương.
3. Query rewrite OK? ➔ Chưa có bước Query Decomposition để tách câu hỏi kép.
4. Fix ở bước: **Query Processing / Hybrid Retrieval & Decomposition**.

**Nếu có thêm 1 giờ, sẽ optimize:**
- Thêm module Query Decomposition (tách câu hỏi kép bằng LLM trước khi search).
- Sử dụng Parent-Child retrieval để khi retrieve child chunk phép năm sẽ tự động gom thêm context xung quanh.
