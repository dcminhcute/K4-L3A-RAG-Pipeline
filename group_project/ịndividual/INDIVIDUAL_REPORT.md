# Individual contribution report

- **Họ và tên:** Đoàn Quang Minh
- **Mã học viên:** [Điền mã học viên của bạn, ví dụ: K4-XXXXX]
- **Nhóm:** Nhóm RAG Pháp Luật Doanh Nghiệp (K4-L3A)
- **Repository/branch:** https://github.com/dcminhcute/K4-L3A-RAG-Pipeline (branch `main`)

---

## Phần việc đã thực hiện

| Module/deliverable | Việc tôi trực tiếp làm | File/commit/PR | Trạng thái |
|---|---|---|---|
| **Task 1: Thu thập Legal PDF** | Tìm kiếm, chọn lọc và tải 3 file văn bản quy phạm pháp luật thực tế từ `vanban.chinhphu.vn` (NĐ 359/2026/NĐ-CP về VAMC, NĐ 358/2026/NĐ-CP về DATC, NĐ 357/2026/NĐ-CP về cơ cấu vốn DNNN). | `data/landing/legal/*.pdf` (Commit `6306819`) | Done |
| **Task 2: Crawl News Articles** | Viết module cào dữ liệu tin tức kèm fallback `urllib`, xử lý giải nén gzip/brotli và UTF-8; cào 5 bài báo pháp luật giải thích NĐ 357, 358, 359. | `src/task2_crawl_news.py`, `data/landing/news/*.json` (Commit `6306819`) | Done |
| **Task 3: Chuẩn hóa Markdown** | Viết module chuyển đổi MarkItDown/pdfminer, chuẩn hóa metadata header (`source`, `title`, `doc_type`, `url`); tạo 8 file markdown chuẩn. | `src/task3_convert_markdown.py`, `data/standardized/` (Commit `6306819`) | Done |
| **Task 4: Chunking & Indexing** | Thiết kế bộ chia nhỏ `RecursiveCharacterTextSplitter` (chunk_size=500, overlap=50); embedding OpenAI `text-embedding-3-small` (1536 dim); index 44 chunks vào ChromaDB với cosine distance. | `src/task4_chunking_indexing.py`, `chroma_db/` (Commit `6306819`) | Done |
| **Task 5 & 6: Dense & Lexical Search** | Cài đặt `semantic_search()` dùng chung hàm embed query; Cài đặt `lexical_search()` dùng `BM25Plus` nạp corpus tự động từ ChromaDB. | `src/task5_semantic_search.py`, `src/task6_lexical_search.py` (Commit `6306819`) | Done |
| **Task 7: RRF Reranking** | Cài đặt thuật toán Reciprocal Rank Fusion ($k=60$), copy item để giữ nguyên dữ liệu đầu vào, gán nhãn `retrieval_method="hybrid"`. | `src/task7_reranking.py` (Commit `6306819`) | Done |
| **Task 9: Retrieval Pipeline** | Tích hợp luồng tìm kiếm kết hợp Dense + BM25, cơ chế so khớp ngưỡng fallback an toàn, bảo vệ pipeline không crash khi lỗi dịch vụ. | `src/task9_retrieval_pipeline.py` (Commit `6306819`) | Done |
| **Task 10: Generation & Citation** | Xây dựng thuật toán `reorder_for_llm` chống hiện tượng *lost-in-the-middle*; định dạng context có Title/Source; cơ chế Safe Refusal từ chối an toàn khi thiếu dữ liệu. | `src/task10_generation.py` (Commit `6306819`) | Done |
| **Chatbot UI** | Tích hợp toàn bộ pipeline vào Streamlit UI, hiển thị câu trả lời trích dẫn kèm điểm score và expander đối chiếu tài liệu nguồn. | `app.py` (Commit `6306819`) | Done |

---

## Quyết định kỹ thuật quan trọng

1. **Quyết định:** Sử dụng thuật toán `BM25Plus` thay cho `BM25Okapi` trong module tìm kiếm từ khóa ([`src/task6_lexical_search.py`](file:///d:/K4-L3A-RAG-Pipeline/src/task6_lexical_search.py)).
   - **Lý do/evidence:** Trên tập dữ liệu nhỏ hoặc các từ khóa xuất hiện ở 50% số văn bản trong corpus, công thức tính IDF của `BM25Okapi` tiêu chuẩn $(\ln\frac{N - n + 0.5}{n + 0.5})$ bị triệt tiêu về $0.0$, khiến kết quả tìm kiếm rỗng và fail contract test. `BM25Plus` bổ sung tham số cận dưới $\delta = 1.0$, đảm bảo mọi chunk có chứa từ khóa đều nhận được điểm số tương quan dương.
   - **Trade-off:** Phổ điểm tuyệt đối của BM25Plus bị dịch chuyển lên cao hơn, nhưng vì pipeline sử dụng RRF (chỉ dựa vào thứ hạng xếp hạng $rank$) nên không làm sai lệch chất lượng xếp hạng hỗn hợp.

2. **Quyết định:** Xác lập ngưỡng phân tách Fallback `SCORE_THRESHOLD = 0.40` dựa trên thực nghiệm đo khoảng cách giữa In-domain và Out-of-domain.
   - **Lý do/evidence:** Tôi đã thực hiện chạy đo đạc thực nghiệm với các truy vấn đại diện:
     - Các câu hỏi đúng chủ đề pháp lý doanh nghiệp (In-domain) cho điểm cosine similarity cao: từ **`0.5208` đến `0.7685`**.
     - Các câu hỏi ngoài lề (Out-of-domain: thời tiết, nấu ăn, phần mềm) có điểm trần chỉ đạt **`0.2841` đến `0.3176`**.
     Khoảng cách an toàn giữa 2 nhóm là $[0.32, 0.52]$. Ngưỡng $0.40$ giúp hệ thống tự tin phân loại câu hỏi không đủ cơ sở dữ liệu để kích hoạt Safe Refusal, ngăn chặn triệt để ảo giác thông tin từ LLM.
   - **Trade-off:** Các câu hỏi trong chủ đề nhưng dùng từ ngữ quá khác biệt hoặc paraphrase quá xa có thể đạt điểm dưới $0.40$ và bị từ chối; đòi hỏi người dùng diễn đạt rõ ràng câu hỏi.

---

## Kiểm thử và kết quả

- **Test đã chạy:**
  - `pytest tests/test_contracts.py -q`: Đạt **15/15 passed (100%)**.
  - `pytest tests/test_acceptance.py -k "test_corpus or test_standardized"`: Đạt **3/3 passed**.
- **Kết quả truy vấn thực tế:**
  - Query trong domain (`"VAMC có số vốn điều lệ là bao nhiêu và do ai quản lý?"`): Trả lời chính xác 5.000 tỷ đồng, có trích dẫn `(Document 1)`, `retrieval_source="hybrid"`.
  - Query ngoài domain (`"Cách làm bánh pizza hải sản tại nhà như thế nào?"`): Trả lời từ chối an toàn, `sources=[]`, `retrieval_source="none"`.
- **Lỗi đã phát hiện và xử lý:** 
  - Khắc phục lỗi server báo chí phản hồi nén `brotli/gzip` khiến file JSON bị dính ký tự nhị phân `\x00` (null byte).
  - Khắc phục lỗi `IndexError` của BM25 trên tập văn bản nhỏ bằng cách áp dụng `BM25Plus`.

---

## Điều còn hạn chế

- **Hạn chế:** Các tài liệu Nghị định gốc từ Chính phủ là file scan ảnh PDF dạng raster (đóng dấu đỏ), do đó công cụ trích xuất văn bản cơ bản chưa thể đọc trực tiếp các biểu mẫu phụ lục phức tạp nếu không có tầng OCR thị giác.
- **Thay đổi sẽ làm nếu có thêm thời gian:** Tích hợp mô hình OCR nâng cao (như DocTR hoặc Vision-LLM) để nhận diện cấu trúc biểu mẫu bảng biểu phụ lục trong các văn bản luật hành chính.

---

## Xác nhận đóng góp

Tôi xác nhận nội dung trên phản ánh đúng phần việc của mình và có thể giải thích hoặc chạy lại trong buổi demo.

- **Ngày:** 20/09/2026
- **Tên thành viên:** Đoàn Quang Minh
