# Báo cáo Cá nhân (Individual Reflection Report)

- **Họ và tên:** Mai Ngọc Duy
- **Mã số sinh viên:** 2A202600736
- **Vai trò:** **AI Evaluation Benchmarking**

---

## 1. Đóng góp kỹ thuật

Trong Lab 14, tôi chịu trách nhiệm xây dựng và hoàn thiện hệ thống AI Evaluation Benchmarking từ dữ liệu kiểm thử, retrieval evaluation, multi-judge engine, benchmark runner đến regression gate và failure analysis.

### 1.1. Golden Dataset và Hard Cases

- Thiết kế bộ Golden Dataset gồm 58 test cases đa dạng, phản ánh các tình huống thực tế mà Agent phải xử lý trong môi trường production.
- Gán metadata đầy đủ cho từng case: `id`, `question`, `ground_truth`, `expected_doc_ids`, `difficulty`, `type`, giúp pipeline downstream như `engine/runner.py` và `analysis/auto_analyze.py` có thể tự động thống kê và phân loại lỗi.
- Phân bổ các loại câu hỏi kiểm thử chiến lược: Fact-check, Prompt Injection, Out-of-context, Ambiguous, Multi-turn Correction, Latency-stress và Conflicting Information.
- Thiết kế tiêu chí Pass/Fail dựa trên cả chất lượng câu trả lời và độ chính xác của tài liệu được truy xuất.

### 1.2. Retrieval, Multi-Judge và Benchmark Engine

- Xây dựng cơ chế đánh giá Retrieval bằng Hit Rate và MRR để đo việc hệ thống có tìm đúng tài liệu nguồn hay không.
- Phát triển module Multi-Judge trong `engine/llm_judge.py`, sử dụng nhiều Judge model để giảm thiên kiến khi chấm điểm câu trả lời.
- Thiết kế logic xử lý conflict: khi điểm số giữa các Judge lệch lớn, hệ thống có thể gọi thêm Tie-breaker Judge để đưa ra kết quả ổn định hơn.
- Tối ưu `engine/runner.py` bằng xử lý bất đồng bộ với `asyncio.Semaphore`, giúp benchmark nhiều cases song song mà vẫn kiểm soát được rate limit.
- Tích hợp pipeline chính trong `main.py`, thu thập các chỉ số như score, token usage, estimated cost và latency để phục vụ quyết định release.

### 1.3. Regression Gate và Failure Analysis

- Xây dựng module `engine/regression_gate.py` để tự động so sánh chất lượng phiên bản V1 và V2.
- Thiết kế các ngưỡng đánh giá trên nhiều trục: Quality Score Delta, Retrieval Hit Rate, Latency và Cost Increase.
- Phát triển `analysis/auto_analyze.py` để đọc kết quả benchmark, thống kê Pass/Fail, phân loại lỗi và hỗ trợ tạo phân tích 5 Whys cho các case tệ nhất.
- Cập nhật quy trình kiểm tra bài nộp bằng `check_lab.py`, đảm bảo dữ liệu và report có định dạng phù hợp trước khi submit.

---

## 2. Kết quả hoạt động tốt

- **Faithfulness cao:** Agent trả lời dựa trên context, hạn chế hallucination.
- **Retrieval V2 cải thiện rõ rệt:** Hit Rate tăng từ 0.76 lên 0.95, MRR tăng từ 0.70 lên 0.94.
- **Xử lý out-of-context tốt:** Các câu hỏi không có trong tài liệu được trả lời theo hướng không đủ thông tin thay vì bịa nội dung.
- **Prompt Injection được kiểm soát:** Agent từ chối các yêu cầu như debug mode, ignore context hoặc fake policy.
- **Benchmark có thêm góc nhìn vận hành:** Ngoài điểm chất lượng, hệ thống còn ghi nhận latency, token usage và estimated cost.

---

## 3. Vấn đề cần cải thiện

- **Relevancy còn thấp:** Agent có thể trả lời đúng context nhưng chưa khớp format hoặc mức chi tiết của expected answer.
- **Multi-turn context loss:** Agent chưa duy trì tốt ngữ cảnh giữa các lượt hội thoại.
- **Comparative questions:** Các câu hỏi so sánh như Team vs Pro hoặc Starter vs higher plans còn dễ bị trả lời thiếu.
- **Conflicting documents:** Khi tài liệu cũ và mới mâu thuẫn, Agent cần cơ chế versioning hoặc ưu tiên tài liệu mới hơn.
- **Fallback khi thiếu context:** Agent cần trả lời "không đủ thông tin" nhất quán hơn khi corpus không chứa chính sách cần hỏi.

---

## 4. Chiều sâu kỹ thuật

### 4.1. Hit Rate và MRR

Hit Rate đo tỉ lệ câu hỏi mà Retriever tìm được ít nhất một tài liệu đúng trong Top-K. MRR đo thứ hạng của tài liệu đúng đầu tiên bằng nghịch đảo rank. Hai chỉ số này giúp tách bạch lỗi Retrieval và lỗi Generation, tránh đánh giá Agent chỉ bằng câu trả lời cuối cùng.

### 4.2. Multi-Judge Consensus

Một LLM Judge đơn lẻ có thể bị Position Bias, Verbosity Bias hoặc Style Bias. Vì vậy, việc dùng nhiều Judge và đo Agreement Rate giúp kết quả benchmark khách quan hơn. Khi các Judge lệch điểm lớn, Tie-breaker là cơ chế cần thiết để giảm rủi ro kết luận sai.

### 4.3. Quality vs Cost Trade-off

Đánh giá bằng LLM có chi phí cao, nên hệ thống cần cân bằng giữa chất lượng, tốc độ và ngân sách. Các câu hỏi FAQ đơn giản có thể dùng mô hình nhỏ hoặc tìm kiếm từ khóa, còn các câu hỏi rủi ro cao như pháp lý, bảo mật dữ liệu hoặc mâu thuẫn chính sách nên dùng mô hình mạnh hơn.

### 4.4. Golden Dataset là nền tảng của Benchmark

Benchmark chỉ đáng tin khi dataset đủ đa dạng và có ground truth rõ ràng. Nếu test cases quá dễ hoặc thiếu edge cases, hệ thống có thể đạt điểm cao nhưng không phản ánh đúng năng lực thực tế. Metadata như `type` và `difficulty` giúp phân tích lỗi tự động mà không cần hard-code rule.

---

## 5. Giải quyết vấn đề

1. **Khắc phục lỗi Unicode trên Windows**
   - Khi chạy benchmark trên PowerShell Windows, Python có thể crash do lỗi `charmap codec can't encode character`.
   - Cách xử lý là chạy script với môi trường UTF-8, ví dụ `PYTHONUTF8=1`, để terminal xử lý Unicode ổn định hơn.

2. **Thiết kế dataset cân bằng hơn**
   - Các case ban đầu quá sạch khiến Agent đạt điểm cao nhưng chưa lộ điểm yếu.
   - Tôi bổ sung các câu hỏi có viết tắt, sai chính tả nhẹ, thiếu ngữ cảnh và adversarial prompts để kiểm tra độ robust.

3. **Tách lớp Retrieval và Generation**
   - Mỗi case có `expected_doc_ids` để chấm Retrieval và `ground_truth` để chấm Generation.
   - Cách tách này giúp Failure Analysis xác định lỗi nằm ở truy xuất tài liệu hay tổng hợp câu trả lời.

4. **Nhận diện giới hạn của stateless architecture**
   - Các lỗi multi-turn cho thấy Agent không nhớ thông tin lượt trước.
   - Hướng cải thiện là thêm conversation memory và query rewriter để resolve các tham chiếu như "gói cao hơn" hoặc "cái đó".

---

## 6. Bài học rút ra

- Đánh giá AI không chỉ nhìn vào accuracy, mà phải cân bằng giữa quality, latency và cost.
- Retrieval tốt không đảm bảo câu trả lời cuối cùng tốt; cần đánh giá end-to-end.
- Semantic chunking và hybrid search phù hợp hơn fixed-size chunking hoặc keyword search thuần túy trong nhiều tình huống.
- Document versioning rất quan trọng khi corpus có tài liệu cũ và mới mâu thuẫn.
- Failure Analysis theo 5 Whys giúp biến điểm benchmark thành hành động cải tiến cụ thể.

---

## 7. Kế hoạch cải tiến cá nhân

- Tìm hiểu thêm semantic chunking và hybrid search như BM25 kết hợp vector search.
- Bổ sung conversation memory để xử lý multi-turn support chat.
- Thêm cơ chế versioning hoặc timestamp cho tài liệu để ưu tiên nguồn mới hơn.
- Cải thiện fallback "không đủ thông tin" khi context không chứa dữ liệu cần thiết.
- Tối ưu prompt và reranking để tăng relevancy mà không làm chi phí tăng quá cao.

---

*Báo cáo được hoàn thành trong khuôn khổ Lab 14 - AI Evaluation Benchmarking.*
