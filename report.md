# Báo Cáo: Xây dựng Production Defense-in-Depth Pipeline (Assignment 11)

Báo cáo này tổng hợp quá trình xây dựng hệ thống phòng thủ đa tầng (Defense in Depth), dựa trên các kỹ thuật Guardrails, Human-in-the-Loop (HITL) và kiểm thử thực tế trên Google Colab với mô hình Gemini.

## 1. Thiết lập & Cấu hình Hệ thống (Setup)
- **Môi trường:** Đã thiết lập thành công các thư viện lõi bao gồm `google-adk`, `nemoguardrails`, và `langchain-google-genai`.
- **Kết nối API:** Cấu hình thành công Google Gemini API Key thông qua Colab Secrets để đảm bảo bảo mật cho key.
- **Hàm bổ trợ:** Xây dựng hàm `chat_with_agent` để giao tiếp mượt mà với các Agent trong môi trường không đồng bộ (async), cho phép giả lập tương tác của người dùng.

## 2. Tấn công Agent không bảo vệ (Red Teaming & Part 1)
- **TODO 1 (Tạo Unsafe Agent & Red Teaming):** Thiết lập chatbot ngân hàng không có lớp bảo vệ chứa bí mật hệ thống. Đã thiết kế bộ khung (framework) cho 5 kỹ thuật tấn công phức tạp bao gồm: Completion, Translation, Hypothetical, Confirmation, và Multi-step.
- **TODO 2 (Automated Red Teaming):** Đã sử dụng chính Gemini API để tự động sinh ra các prompt tấn công (AI-generated attack test cases) thay vì tự viết tay, nhằm tìm ra những mẫu tấn công mà con người khó lường trước.
- **Phát hiện rủi ro:** Kết quả kiểm thử (Test 2) xác nhận rằng Agent mặc định rất dễ bị rò rỉ thông tin mật nếu không có lớp bảo vệ.

## 3. Triển khai các lớp bảo vệ (Part 2 - Guardrails)
Hệ thống phòng thủ đa tầng đã được xây dựng hoàn chỉnh với 3 thành phần chính:

### A. Input Guardrails (Chặn đầu vào)
- **`detect_injection`:** Sử dụng biểu thức chính quy (Regex) để nhận diện các mẫu tấn công phổ biến như *"Ignore instructions"*, *"DAN"*, *"Reveal prompt"*. Lớp này chặn đứng các tấn công ở lớp đầu tiên (như câu 1, 2, 4 trong Test 2).
- **`topic_filter`:** Đảm bảo người dùng chỉ được hỏi về chủ đề ngân hàng, tự động chặn các chủ đề độc hại hoặc lạc đề.
- **`InputGuardrailPlugin`:** Tích hợp các hàm trên vào cấu trúc ADK Plugin để chặn đứng và phản hồi ngay lập tức trước khi tin nhắn được gửi đến LLM.

### B. Output Guardrails (Chặn đầu ra)
- **`content_filter`:** Tự động phát hiện và ẩn danh (Redacted) các thông tin nhạy cảm như Email, Số điện thoại, API Keys bằng Regex trước khi hiển thị cho người dùng (chặn câu 6 trong Test 2).
- **`llm_safety_check`:** Sử dụng kỹ thuật LLM-as-Judge để mô hình AI đánh giá chéo tính an toàn của câu trả lời trước khi gửi cho khách hàng (chặn câu 7 trong Test 2).

### C. NeMo Guardrails (NVIDIA)
- **OpenAI-Compatible Bridge:** Đã khắc phục lỗi khởi tạo `base_url` bằng cách sử dụng giao thức OpenAI-Compatible Bridge để kết nối ổn định Gemini với NeMo.
- **Colang Rules:** Cấu hình các quy tắc Colang (`.co`) thông minh để xử lý các tình huống phức tạp như giả danh Admin (Role confusion - chặn câu 3 trong Test 2), tấn công mã hóa (Encoding) và tấn công đa ngôn ngữ (Vietnamese injection - chặn câu 5 trong Test 2).

## 4. Quy trình có sự can thiệp của con người (Part 4 - HITL)
- **TODO 12 (Confidence Router):** Đã viết mã nguồn cho bộ định tuyến thông minh, tự động phân loại phản hồi của AI dựa trên điểm tin cậy (Confidence Score):
  - **`>= 0.9`**: Tự động gửi cho khách hàng (Auto Send / Human-on-the-loop).
  - **`0.7 - 0.9`**: Đưa vào hàng đợi duyệt (Queue Review / Human-in-the-loop).
  - **`< 0.7` hoặc Hành động rủi ro cao**: Chuyển trực tiếp cho chuyên viên con người (Escalate / Human-as-tiebreaker).
- **TODO 13 (Decision Points):** Thiết kế 3 kịch bản thực tế cần con người can thiệp: 
  1. Chuyển tiền số lượng lớn.
  2. Đổi thông tin định danh nhạy cảm (CCCD/SĐT).
  3. Tư vấn điều khoản pháp lý phức tạp.

## 5. Kiểm thử & Đánh giá (Part 3)
- **Security Pipeline:** Xây dựng hệ thống kiểm thử tự động để đo lường hiệu quả trước và sau khi có Guardrails. Kết quả cho thấy tỷ lệ chặn các truy vấn độc hại tăng lên đáng kể.
- **Rate Limiting:** Xử lý thành công lỗi `429 Resource Exhausted` (giới hạn API của tier miễn phí) bằng cơ chế Retry và khoảng nghỉ (Sleep) thông minh. Cơ chế này vừa tối ưu quota API, vừa đóng vai trò như một lớp Rate Limiter chống lại các cuộc tấn công Spam/DDoS từ người dùng.

---
### Tổng kết
Hệ thống hiện tại đã đáp ứng đầy đủ yêu cầu của một **Production Defense-in-Depth Pipeline**. Các lớp bảo vệ hoạt động độc lập nhưng bổ trợ cho nhau: Input chặn mã độc, NeMo kiểm soát luồng hội thoại, Output chống rò rỉ dữ liệu, và HITL đảm bảo chất lượng cuối cùng. Sự kết hợp này mang lại độ an toàn cao nhất cho hệ thống VinBank Assistant.