# Báo cáo Submission - Memory System

## 1. Chiến lược quản lý Context

Hệ thống memory sử dụng kết hợp **sliding-window context** và **durable memory** để cân bằng giữa kích thước context và khả năng lưu giữ thông tin quan trọng.

Cơ chế sliding-window giới hạn context hiện tại bằng cách chỉ giữ lại các lượt hội thoại gần nhất. Điều này giúp context không tăng không giới hạn khi cuộc hội thoại kéo dài. Đồng thời, durable memory lưu giữ những thông tin quan trọng cần tồn tại qua nhiều session, chẳng hạn như **constraint, decision, deadline và các fact lâu dài**.

Một buffer đơn giản không phù hợp với các cuộc hội thoại dài vì số lượng token tăng gần như tuyến tính theo độ dài hội thoại và cuối cùng có thể vượt quá context budget. Việc kết hợp sliding-window với durable memory giải quyết vấn đề này bằng cách giữ lại context gần nhất đồng thời bảo toàn có chọn lọc những thông tin quan trọng về lâu dài.

---

## 2. Phân tích Memory Retrieval

### 2.1. Layer nào có hit rate thấp nhất?

Hit rate của từng memory layer được đánh giá dựa trên kết quả chi tiết trong `reports/benchmark.md`.

Việc đánh giá riêng từng layer rất quan trọng vì hệ thống memory tổng thể có thể đạt evidence hit rate cao ngay cả khi một layer riêng lẻ có hiệu quả retrieval thấp hơn các layer khác. Do đó, layer có hit rate thấp nhất là ứng viên chính để cải thiện cơ chế retrieval hoặc query routing.

### 2.2. Query nào retrieve nhiều token nhất?

Query có lượng token retrieval lớn nhất được xác định từ thống kê token chi tiết trong `reports/benchmark.md`.

Chỉ số này giúp đánh giá hiệu quả của context budget. Một query retrieve quá nhiều thông tin có thể cần cơ chế trimming mạnh hơn hoặc retrieval ranking tốt hơn để đảm bảo những evidence quan trọng nhất vẫn được giữ lại trong giới hạn token budget.

### 2.3. Case mixed (E07) cần kết hợp memory nào? Evidence nào bắt buộc?

Case mixed **E07** cần kết hợp thông tin từ nhiều memory layer thay vì chỉ dựa vào một nguồn duy nhất.

Các nguồn memory liên quan gồm:

- **Short-term memory:** thông tin từ cuộc hội thoại hiện tại.
- **Long-term memory:** các fact, preference, constraint và decision có tính lâu dài của user.
- **Episodic memory:** các trải nghiệm trước đây, hành động đã thực hiện, lỗi gặp phải và strategy đã thành công.
- **Semantic memory:** kiến thức domain dùng chung và các quy tắc tổng quát.

Evidence bắt buộc phải trực tiếp hỗ trợ từng phần của query. Router và cơ chế context assembly cần kết hợp evidence từ các memory layer phù hợp rồi merge chúng trong giới hạn token budget.

Điều này cho thấy kiến trúc multi-layer memory hiệu quả hơn việc chỉ sử dụng một cơ chế retrieval duy nhất cho các query mixed.

---

## 3. Token Budget và hiệu quả của Memory

Kết quả benchmark:

| Metric | Memory-enabled | No-memory | Delta |
|---|---:|---:|---:|
| Evidence hit rate | **100.0%** | **18.2%** | **+81.8%** |
| Passed cases | **11/11** | **2/11** | **+9** |
| Avg retrieval latency (ms) | **2760.9** | **0.1** | **+2760.8** |
| Avg token reduction | **20.8%** | **81.8%** | **-61.0%** |

Hệ thống memory-enabled đạt **100.0% evidence hit rate** và pass toàn bộ **11/11 cases**, trong khi baseline no-memory chỉ đạt **18.2% evidence hit rate** và pass **2/11 cases**.

Về token reduction, memory-enabled giảm trung bình **20.8%** so với full source context, trong khi no-memory giảm tới **81.8%**, chênh lệch **61.0 điểm phần trăm**.

Tuy nhiên, token reduction cao của no-memory không đồng nghĩa với hiệu quả tốt hơn. No-memory đạt reduction cao chủ yếu vì gần như không retrieve durable memory. Nói cách khác, nó tiết kiệm token bằng cách **loại bỏ context cần thiết**, thay vì lựa chọn và giữ lại những thông tin liên quan nhất.

Do đó, **không nên tối ưu token reduction một cách độc lập với evidence hit rate**. Một memory system hiệu quả phải giảm context không cần thiết nhưng đồng thời giữ lại evidence cần thiết để trả lời chính xác.

Memory-enabled đạt được sự cân bằng này bằng cách retrieve thông tin liên quan từ nhiều layer, sau đó áp dụng context budget và priority policy để giữ lại những evidence quan trọng nhất.

---

## 4. Các điểm rút ra chính

1. **Recent context và durable memory có vai trò khác nhau.**  
   Sliding-window context duy trì trạng thái của cuộc hội thoại hiện tại, trong khi durable memory lưu giữ những thông tin cần tồn tại qua nhiều session.

2. **Các memory layer có tính bổ trợ cho nhau.**  
   Short-term, long-term, episodic và semantic memory cung cấp các loại evidence khác nhau và cần được router lựa chọn dựa trên query.

3. **Mixed query cần kết hợp nhiều memory layer.**  
   E07 cho thấy một nguồn memory duy nhất không đủ khi query kết hợp thông tin hiện tại, kinh nghiệm trước đây và kiến thức domain.

4. **Token reduction không phải là metric duy nhất để đánh giá memory system.**  
   No-memory đạt **81.8% token reduction** nhưng chỉ có **18.2% evidence hit rate**. Trong khi đó, memory-enabled đạt **100.0% evidence hit rate** và vẫn giảm được **20.8%** số token so với full source context.

5. **Memory management là bài toán cân bằng giữa retrieval quality, context size và latency.**  
   Mục tiêu không phải là giảm số token nhiều nhất có thể, mà là giữ lại lượng context nhỏ nhất nhưng vẫn chứa đầy đủ evidence cần thiết để trả lời chính xác.