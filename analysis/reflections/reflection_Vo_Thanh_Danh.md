# Individual Reflection - Role 1

**Họ và tên:** Võ Thành Danh  
**MSSV:** 2A202600503  
**Vai trò:** Role 1 (Data / Retrieval / Failure Analysis)

## 1. Engineering Contribution

### 1.1. Những phần tôi trực tiếp triển khai
- Triển khai lại `data/synthetic_gen.py` để sinh Golden Dataset đúng contract:
  - 60 cases theo đúng phân phối yêu cầu của lab.
  - Mỗi case có đầy đủ trường bắt buộc: `case_id`, `question`, `expected_answer`, `expected_retrieval_ids`, `difficulty`, `case_type`, `tags`.
  - Có thêm `data/hard_cases_pack.json` để giữ tập red-team/hard cases.
- Hoàn thiện `engine/retrieval_eval.py`:
  - `HitRate@1/@3/@5`
  - `MRR`
  - `evaluate_case`, `evaluate_batch`
  - `top_k_sensitivity`
  - correlation tùy chọn giữa retrieval và answer score.
- Cập nhật `analysis/failure_analysis.md` theo hướng trung thực:
  - Không ghi số benchmark khi chưa có dữ liệu thật từ pipeline đầy đủ.
  - Chuẩn hóa framework phân cụm lỗi và mẫu 5 Whys để điền sau khi chạy thật.

### 1.2. Minh chứng kỹ thuật
- Unit tests thuộc phạm vi Role 1:
  - `tests/test_synthetic_gen.py`
  - `tests/test_retrieval_eval.py`
- Có thể chạy lại bằng:
  - `python data/synthetic_gen.py`
  - `python -m pytest tests/test_synthetic_gen.py tests/test_retrieval_eval.py -q`

## 2. Technical Depth

### 2.1. MRR (Mean Reciprocal Rank)
- **Định nghĩa:** MRR đo chất lượng thứ hạng của retriever thông qua vị trí tài liệu đúng đầu tiên trong danh sách trả về.
- **Công thức per-query:** nếu tài liệu đúng đầu tiên ở vị trí `r` (đánh số từ 1), điểm là `1/r`; nếu không có tài liệu đúng, điểm là `0`.
- **Công thức toàn tập:** `MRR = (1/N) * Σ(1/r_i)` với `N` là số query hợp lệ.
- **Ý nghĩa thực tế trong RAG:** MRR nhấn mạnh chất lượng top đầu. Tài liệu đúng xuất hiện sớm (top-1/top-2) giúp giảm xác suất model dùng context sai, từ đó giảm nguy cơ hallucination và câu trả lời lạc đề.
- **So sánh với HitRate@K:** HitRate@K cho biết “có tìm thấy hay không”, còn MRR phản ánh “tìm thấy sớm tới mức nào”. Vì vậy MRR hữu ích khi cần tối ưu ranking quality thay vì chỉ tối ưu recall thô.

### 2.2. Cohen's Kappa
- **Định nghĩa:** Cohen’s Kappa đo mức đồng thuận giữa hai bộ chấm điểm sau khi đã loại trừ phần đồng thuận xảy ra do ngẫu nhiên.
- **Công thức:** `kappa = (p_o - p_e) / (1 - p_e)`  
  - `p_o`: tỷ lệ đồng thuận quan sát được  
  - `p_e`: tỷ lệ đồng thuận kỳ vọng nếu hai bên chấm ngẫu nhiên theo phân phối nhãn
- **Diễn giải chuẩn:**  
  - `kappa ~ 1`: đồng thuận mạnh, hệ thống chấm ổn định  
  - `kappa ~ 0`: đồng thuận gần mức ngẫu nhiên  
  - `kappa < 0`: bất đồng có hệ thống
- **Giá trị kỹ thuật:** trong bài toán Multi-Judge, Kappa giúp tránh “agreement ảo” khi một nhãn chiếm đa số. Đây là chỉ số cần thiết để đánh giá độ tin cậy thực của hệ thống đánh giá.

### 2.3. Position Bias
- **Định nghĩa:** Position Bias là hiện tượng judge đánh giá tốt hơn cho phương án xuất hiện ở vị trí đầu (hoặc vị trí cố định), ngay cả khi chất lượng hai phương án tương đương.
- **Rủi ro:** nếu không kiểm soát bias này, kết quả benchmark có thể phản ánh thứ tự hiển thị thay vì phản ánh chất lượng thực.
- **Cách kiểm tra chuẩn:** thực hiện pairwise evaluation hai lượt:
  - Lượt 1: chấm theo thứ tự A/B  
  - Lượt 2: hoán đổi B/A  
  - So sánh độ lệch điểm và tỷ lệ đổi winner giữa hai lượt.
- **Hướng xử lý:** randomize thứ tự đầu vào, chấm lặp nhiều lần và tổng hợp bằng thống kê (mean/median + variance) để giảm sai lệch do vị trí.

### 2.4. Trade-off Chi phí và Chất lượng
- **Quan sát kỹ thuật:** tăng `top_k` thường cải thiện xác suất chứa tài liệu đúng trong context, nhưng không miễn phí.
- **Chi phí phát sinh khi tăng `top_k`:**
  - tăng số chunk đưa vào prompt,
  - tăng token input và chi phí inference,
  - tăng latency do context dài hơn.
- **Rủi ro chất lượng:** nếu không có cơ chế reranking/lọc nhiễu, mở rộng `top_k` có thể kéo theo nhiều đoạn ít liên quan, làm model suy luận kém chính xác hơn.
- **Nguyên tắc tối ưu:** không tối ưu một chiều. Cần theo dõi đồng thời nhóm chỉ số chất lượng (`HitRate@K`, `MRR`, độ chính xác đầu ra) và nhóm chỉ số vận hành (token usage, latency, cost/request) trước khi chốt cấu hình retrieval.

## 3. Problem Solving

### 3.1. Vấn đề gặp phải và cách giải quyết
1. **Sự cố encoding trên môi trường Windows (console cp1258)**
   - **Triệu chứng:** script SDG lỗi khi in một số ký tự Unicode (emoji), làm gián đoạn pipeline tạo dữ liệu.
   - **Phân tích nguyên nhân:** output log phụ thuộc code page terminal, không ổn định giữa các môi trường chạy.
   - **Giải pháp thực hiện:** chuẩn hóa log về ASCII thuần để loại bỏ phụ thuộc môi trường.
   - **Kết quả:** script chạy ổn định trên terminal hiện tại, không còn lỗi do encoding.

2. **Rủi ro lệch chuẩn dữ liệu SDG**
   - **Triệu chứng:** khi sinh dữ liệu tự động, dễ phát sinh thiếu trường, sai kiểu dữ liệu hoặc lệch phân phối case.
   - **Phân tích nguyên nhân:** thiếu bước kiểm định bắt buộc trước khi ghi file đầu ra.
   - **Giải pháp thực hiện:** bổ sung `validate_dataset_schema` để kiểm tra:
     - số lượng tối thiểu,
     - phân phối case type theo đúng kế hoạch,
     - đầy đủ trường bắt buộc và kiểu dữ liệu hợp lệ.
   - **Kết quả:** giảm nguy cơ “data quality drift”, đảm bảo `golden_set.jsonl` nhất quán và sẵn sàng cho benchmark.

3. **Rủi ro báo cáo sai do dùng số mô phỏng**
   - **Triệu chứng:** một số nội dung báo cáo ban đầu có thể bị hiểu là kết quả benchmark thật dù chưa chạy pipeline đầy đủ.
   - **Phân tích nguyên nhân:** chưa tách rõ ràng giữa “thiết kế framework” và “kết quả đo thực nghiệm”.
   - **Giải pháp thực hiện:** loại bỏ toàn bộ phần mô phỏng/không kiểm chứng khỏi reflection và failure report; chỉ giữ nội dung có thể đối chiếu trực tiếp bằng code, test và artifact hợp lệ.
   - **Kết quả:** báo cáo phản ánh đúng trạng thái thực tế của Role 1, giảm rủi ro bị trừ điểm vì thiếu tính trung thực dữ liệu.

### 3.2. Bài học rút ra
- Trong hệ thống đánh giá AI, sai lệch nhỏ ở dữ liệu hoặc cách diễn giải metric có thể dẫn đến kết luận sai toàn pipeline.
- Quy trình đúng cần ưu tiên: **schema-first, metric-first, evidence-first**.
- Mọi claim về chất lượng/bonus chỉ nên công bố khi có run thật và có artifact truy xuất được.

## 4. Cam kết trung thực dữ liệu
- Tôi cam kết:
  - Không dùng số liệu mô phỏng để khai báo thành tích benchmark.
  - Chỉ cập nhật điểm/bonus khi có kết quả chạy thật từ pipeline đầy đủ sau khi Role 2 hoàn tất.
