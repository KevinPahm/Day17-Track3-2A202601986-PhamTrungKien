# Báo cáo Submission - Memory System

## 1. Context và Durable Memory

Hệ thống kết hợp **sliding-window context** với **durable memory**. Sliding-window giữ các lượt hội thoại gần nhất để giới hạn context, trong khi durable memory bảo toàn các thông tin quan trọng như constraint, decision, preference và deadline qua nhiều session. Buffer thuần túy không phù hợp với hội thoại dài vì token tăng theo lịch sử và có thể vượt context budget.

**Short-term memory** là layer quan trọng nhất cho trạng thái hiện tại vì chứa thông tin mới nhất. Tuy nhiên, không layer nào đủ cho mọi loại query: long-term giữ facts/preferences, episodic giữ kinh nghiệm và strategy, còn semantic giữ kiến thức domain dùng chung. Vì vậy cần router và merged context để lựa chọn đúng evidence.

## 2. Benchmark Analysis

Memory-enabled đạt **100.0% evidence hit rate** và **11/11 cases**, trong khi no-memory chỉ đạt **18.2%** và **2/11 cases**.

Layer có hit rate thấp nhất cần được xác định từ bảng chi tiết trong `reports/benchmark.md`. Query retrieve nhiều token nhất cũng được xác định từ thống kê retrieval trong file này; đây là ứng viên cần tối ưu ranking hoặc trimming.

Case mixed **E07** cần kết hợp **long-term** để lấy preference của Minh và **semantic** để lấy policy payment. Evidence bắt buộc là preference Python của Minh và rule `PAYMENT-RULE-3`, bao gồm `Idempotency-Key`, `exponential-backoff` và `max-3-retries`.

Memory-enabled giảm trung bình **20.8%** token so với full source context, trong khi no-memory giảm **81.8%**. No-memory có reduction cao vì gần như không retrieve context cần thiết, dẫn tới hit rate thấp. Do đó token reduction phải được đánh giá cùng evidence hit rate, không nên tối ưu reduction một cách độc lập.

## 3. Recency và Compaction

E08 cho thấy **recency** quan trọng khi các fact mới có thể thay thế hoặc giới hạn phạm vi của thông tin cũ; retrieval cần ưu tiên evidence mới và đúng scope. E10 cho thấy **compaction** giúp giữ constraint quan trọng như deadline dù các turn trung gian đã bị loại khỏi recent context. Durable notes vì vậy cần bảo toàn thông tin có giá trị lâu dài thay vì lưu toàn bộ transcript.

## 4. Trade-off và Memory Poisoning

Memory system phải cân bằng **retrieval quality, token budget và latency**: giảm context quá mạnh có thể làm mất evidence, còn retrieve quá nhiều làm tăng chi phí và latency. Durable memory cũng có rủi ro **memory poisoning**, vì thông tin sai hoặc instruction độc hại có thể tồn tại lâu dài và ảnh hưởng các session sau. Do đó memory ingestion cần consent, minimization, provenance và guardrail; không nên tự động ghi mọi nội dung hội thoại vào durable memory.