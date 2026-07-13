
# ViHistFact

ViHistFact nghiên cứu bài toán **kiểm chứng phát biểu lịch sử tiếng Việt dựa trên truy xuất bằng chứng và mô hình ngôn ngữ lớn**, sử dụng *Đại Việt Sử Ký Toàn Thư* làm kho sử liệu.
Dự án xây dựng bộ dữ liệu gồm 1.311 phát biểu được kiểm chứng thủ công theo ba nhãn `SUPPORTED`, `REFUTED` và `NOT_ENOUGH_EVIDENCE`, đồng thời đánh giá khả năng truy xuất, xác minh và hiệu quả end-to-end của hệ thống.
Nghiên cứu còn phân tích điểm nghẽn và lỗi theo loại phát biểu nhằm xác định hạn chế đến từ bước retrieval, verifier hay bản chất của dữ liệu lịch sử.

## Quy trình xử lý

```text
PDF sử liệu
    ↓
Trích xuất văn bản và metadata theo trang
    ↓
Chia chunk và tạo embedding
    ↓
Sinh claim bằng LLM → duyệt thủ công
    ↓
Chia tập dev/test theo chunk
    ↓
BM25 / Dense / Hybrid / Reranker
    ↓
Đánh giá retrieval, claim verification, end to end 
```

## Dataset và corpus

- Thư mục `copus/` (tên thư mục hiện có trong repository) chứa PDF gốc, văn bản đã trích xuất và các phiên bản corpus đã chia chunk; bản đầy đủ hiện có 5.343 chunk.
- Thư mục `dataset/` chứa chunk đầu vào, bản claim chờ duyệt, bộ dữ liệu cuối cùng và tệp thống kê quá trình duyệt.
- `final_dataset.json` gồm 1.311 claim: 460 `SUPPORTED`, 396 `REFUTED` và 455 `NOT_ENOUGH_EVIDENCE`.

## Notebook

| Notebook | Chức năng |
|---|---|
| `pdf_json.ipynb` | Trích xuất nội dung, tiêu đề và chú thích theo trang từ PDF sang JSON. |
| `Chunking.ipynb` | Chia văn bản thành các chunk theo cấu trúc và ngữ nghĩa, tạo embedding rồi đóng gói dữ liệu cho Qdrant. |
| `build_dataset.ipynb` | Dùng Gemini sinh các claim có nhãn từ chunk đầu vào và tạo bản nháp để con người kiểm duyệt. |
| `split_data.ipynb` | Chia dữ liệu thành dev/test theo `chunk_id`, đồng thời cân bằng tương đối nhãn và loại claim để tránh rò rỉ dữ liệu. |
| `run_retrieval_outputs.ipynb` | Chạy BM25, dense retrieval, hybrid retrieval và reranker, sau đó lưu top-k bằng chứng cho từng claim. |
| `retrieval_evaluation.ipynb` | Đọc các kết quả retrieval đã lưu, tính metric và chọn hệ số hybrid tốt nhất trên tập dev. |
| `verification-evaluation.ipynb` | Đánh giá mô hình kiểm chứng trong ba thiết lập: chỉ claim, gold evidence và retrieved evidence. |

## Môi trường

Các notebook được xây dựng để chạy chủ yếu trên **Google Colab** và **Kaggle Notebook**. Từng notebook đã có cell cài đặt thư viện cần thiết; tùy giai đoạn, bạn cần chuẩn bị:

- Python 3 và Jupyter Notebook;
- GPU cho bước embedding, reranking và chạy mô hình kiểm chứng;
- Gemini API key (tạo dataset và chia chunk bằng LLM);
- Qdrant URL/API key cho dense retrieval;
- quyền tải model từ Hugging Face.

Một số notebook đang sử dụng đường dẫn tuyệt đối dạng `/content/...` hoặc `/kaggle/...`. Hãy sửa các biến cấu hình đầu notebook để trỏ tới vị trí dữ liệu trong môi trường của bạn trước khi chạy.

## Cách chạy

1. Chạy `pdf_json.ipynb` để chuyển PDF trong `copus/` thành dữ liệu JSON theo trang.
2. Chạy `Chunking.ipynb` để tạo corpus chunk và đưa vector lên Qdrant.
3. Chuẩn bị `dataset/input_chunks.json`, sau đó chạy `build_dataset.ipynb` để tạo bản nháp claim.
4. Kiểm duyệt trường `human_review`, lọc các mẫu được chấp nhận thành `dataset/final_dataset.json`.
5. Chạy `split_data.ipynb` để tạo tập dev/test ở cả định dạng JSON và JSONL.
6. Chạy `run_retrieval_outputs.ipynb` riêng cho dev và test; dùng `retrieval_evaluation.ipynb` để chọn cấu hình retrieval.
7. Chạy `verification-evaluation.ipynb` để đo Accuracy, Macro-F1, confusion matrix và kết quả theo từng loại claim.

Các bước sau phụ thuộc vào output của bước trước. Trước mỗi lần chạy, cần kiểm tra lại đường dẫn dữ liệu, tên Qdrant collection, model và chế độ `dev`/`test` trong phần cấu hình của notebook tương ứng.

## Nhãn dữ liệu

- `SUPPORTED`: sử liệu xác nhận các thông tin chính trong claim.
- `REFUTED`: sử liệu thể hiện ít nhất một thông tin chính trong claim là sai hoặc mâu thuẫn.
- `NOT_ENOUGH_EVIDENCE`: bằng chứng hiện có không đủ để xác nhận hoặc bác bỏ claim.

## Hướng đánh giá

### Đánh giá từng thành phần

- **Retrieval:** so sánh BM25, dense, hybrid và hybrid kết hợp reranker dựa trên vị trí xuất hiện của gold evidence trong top-k.
- **Verification:** so sánh kết quả khi mô hình chỉ thấy claim, thấy bằng chứng chuẩn và thấy bằng chứng do hệ thống retrieval trả về.
- **Đánh giá end-to-end**: Quy trình đầy đủ nhận một claim, truy xuất top-k bằng chứng, đưa bằng chứng vào LLM verifier và trả về nhãn cuối cùng. Các cấu hình retrieval–verifier được so sánh bằng Accuracy, Macro-F1, Evidence Recall@5 và FEVER-style score; trong đó FEVER-style chỉ tính đúng khi hệ thống vừa dự đoán đúng nhãn vừa truy xuất đúng bằng chứng.


### Phân tích điểm nghẽn

Kết quả end-to-end sẽ được chia theo trạng thái của retrieval và verifier để xác định nguồn gây lỗi chính:


### Phân tích lỗi theo loại claim


