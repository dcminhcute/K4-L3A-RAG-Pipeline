# Individual contribution report

- **Họ và tên:** Đoàn Quang Minh
- **Mã học viên:** [Điền mã học viên của bạn, ví dụ: K4-XXXXX]
- **Nhóm:** Nhóm RAG Pháp Luật Doanh Nghiệp (K4-L3A)
- **Vai trò trong nhóm:** **Data Lead** (Chịu trách nhiệm toàn bộ phân hệ Dữ liệu: Task 1, Task 2, Task 3)
- **Repository/branch:** https://github.com/dcminhcute/K4-L3A-RAG-Pipeline (branch `main`)

---

## Phần việc đã thực hiện

| Module/deliverable | Việc tôi trực tiếp làm | File/commit/PR | Trạng thái |
|---|---|---|---|
| **Task 1: Thu thập Legal PDF** | Khảo sát, chọn lọc và tải 3 file văn bản quy phạm pháp luật thực tế từ Cổng TTĐT Chính phủ (`vanban.chinhphu.vn`): NĐ 359/2026/NĐ-CP (VAMC), NĐ 358/2026/NĐ-CP (DATC), NĐ 357/2026/NĐ-CP (nguồn thu cơ cấu lại vốn DNNN). Đảm bảo mỗi file > 1024 bytes và đúng định dạng. | `data/landing/legal/*.pdf` (Commit `6306819`) | Done |
| **Task 2: Crawl News JSON** | Viết module cào dữ liệu tin tức báo chí pháp luật; thiết lập cơ chế giải mã nén `gzip`/`brotli`, chuẩn hóa UTF-8, loại bỏ thẻ HTML thừa; thu thập 5 bài viết phân tích chuyên sâu về 3 Nghị định trên từ Báo Nhân Dân, Báo Đấu Thầu, BNews, VOV. Đảm bảo đúng schema 4 trường bắt buộc (`url`, `title`, `date_crawled`, `content_markdown`). | `src/task2_crawl_news.py`, `data/landing/news/*.json` (Commit `6306819`) | Done |
| **Task 3: Chuẩn hóa Markdown** | Viết module chuyển đổi Markdown (`convert_legal_docs` và `convert_news_articles`) kèm metadata header (`source`, `title`, `doc_type`, `url`); đảm bảo tính idempotent (chạy lại không tạo file trùng lặp/rỗng), mỗi file đạt trên 200 ký tự. | `src/task3_convert_markdown.py`, `data/standardized/` (Commit `6306819`) | Done |

---

## Quyết định kỹ thuật quan trọng

1. **Quyết định: Xây dựng cơ chế giải nén (`gzip`/`brotli`) và xử lý bảng mã UTF-8 trong crawler Task 2 ([`src/task2_crawl_news.py`](file:///d:/K4-L3A-RAG-Pipeline/src/task2_crawl_news.py)).**
   - **Lý do/evidence:** Khi gửi request đến các máy chủ báo điện tử lớn (như Báo Nhân Dân, Báo Đấu Thầu), server tự động phản hồi bằng nội dung nén `brotli`/`gzip`. Trình đọc HTTP mặc định không tự giải nén dẫn đến nội dung bài viết bị biến thành chuỗi byte nhị phân chứa đầy ký tự null byte `\x00` (lỗi hiển thị ký tự rác trong file JSON). Tôi đã lập trình tầng xử lý đọc header `Content-Encoding`, tự động giải nén và decode UTF-8 để dữ liệu text luôn sạch, không bị dính ký tự lạ.
   - **Trade-off:** Tăng thêm độ phức tạp trong code xử lý ngoại lệ mạng và giải mã của crawler, nhưng đảm bảo 100% dữ liệu landing đạt chuẩn chất lượng cho các khâu chunking tiếp theo.

2. **Quyết định: Lựa chọn 5 bài báo phân tích có chủ đề bám sát và bổ trợ trực tiếp cho 3 văn bản Nghị định, thay vì cào tin tức ngẫu nhiên.**
   - **Lý do/evidence:** Về mặt kiểm thử kỹ thuật (`test_acceptance.py`), hệ thống chỉ kiểm tra số lượng và schema file JSON mà không kiểm tra ngữ nghĩa. Tuy nhiên, nếu dữ liệu tin tức rời rạc (ví dụ cào tin đời sống, giao thông), khi đưa vào cùng Vector Database sẽ làm loãng kho tri thức, gây nhiễu và làm giảm chỉ số Context Recall / Faithfulness của toàn hệ thống RAG. Việc chọn 5 bài báo phân tích thực tế về VAMC, DATC và cơ chế nộp ngân sách cổ phần hóa tạo thành một corpus đồng nhất: văn bản luật cung cấp điều khoản chính thức, bài viết báo chí cung cấp góc nhìn thực tiễn và giải thích áp dụng.
   - **Trade-off:** Đòi hỏi nhiều thời gian tra cứu, thẩm định và kiểm tra khả năng crawl của các bài báo tương thích hơn so với việc cào ngẫu nhiên 5 link bất kỳ trên mạng.

---

## Kiểm thử và kết quả

- **Các test chấp nhận (Acceptance Tests) đã pass 100%:**
  - `pytest tests/test_acceptance.py::test_corpus_has_required_legal_documents`: **PASSED** (đạt 3/3 file PDF hợp lệ, dung lượng từ 5MB - 10MB).
  - `pytest tests/test_acceptance.py::test_corpus_has_required_news_with_metadata`: **PASSED** (đạt 5/5 file JSON đủ 4 trường `url`, `title`, `date_crawled`, `content_markdown` không rỗng).
  - `pytest tests/test_acceptance.py::test_standardized_output_covers_both_source_types`: **PASSED** (đạt 8/8 file Markdown chuẩn hóa, mỗi file > 200 ký tự).
- **Lỗi đã phát hiện và cách xử lý trong khâu dữ liệu:**
  - *Lỗi 1 (WAF chặn 403 Forbidden):* Một số website cơ quan nhà nước (`moj.gov.vn`) chặn request tự động bằng tường lửa. Tôi đã khảo sát và chuyển hướng sang các nguồn báo chí chính thống mở (`nhandan.vn`, `baodauthau.vn`, `bnews.vn`).
  - *Lỗi 2 (Ký tự nhị phân do nén stream):* File JSON news bị dính ký tự corrupt; đã viết script xử lý giải nén chuẩn hóa và ghi đè lại toàn bộ dữ liệu sạch.

---

## Điều còn hạn chế

- **Hạn chế:** Các tài liệu Nghị định gốc tải từ Chính phủ là file scan ảnh PDF (dạng raster image có dấu đỏ), các thư viện đọc text thông thường (như `pdfminer`, `pypdf`) không trích xuất trực tiếp được text layer mà cần phải qua bộ OCR hoặc tóm tắt cấu trúc.
- **Thay đổi sẽ làm nếu có thêm thời gian:** Xây dựng thêm một pipeline OCR tự động (sử dụng PaddleOCR hoặc DocTR) kết hợp phân đoạn tài liệu pháp luật theo cấu trúc phân cấp (Chương -> Điều -> Khoản -> Điểm) để metadata trích xuất chi tiết hơn.

---

## Xác nhận đóng góp

Tôi xác nhận nội dung trên phản ánh đúng phần việc Data Lead của mình trong dự án nhóm và có thể giải thích hoặc chạy lại các bước thu thập, chuẩn hóa dữ liệu trong buổi demo.

- **Ngày:** 20/09/2026
- **Tên thành viên:** Đoàn Quang Minh
