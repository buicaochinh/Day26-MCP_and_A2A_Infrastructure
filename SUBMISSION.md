# Báo Cáo Thực Hành: Xây Dựng Hệ Thống Multi-Agent với A2A Protocol

**Môn học / Lab:** Day 26 - Track 02: MCP and A2A Infrastructure

## I. Tổng Quan Các Bước Đã Thực Hiện
Trong quá trình thực hành, hệ thống đã được triển khai và hoàn thành qua 5 phân hệ (stages):
1. **Stage 1 (Direct LLM):** Thiết lập gọi trực tiếp LLM, cấu hình môi trường (.env) và thêm tham số temperature để ổn định output.
2. **Stage 2 (RAG & Tools):** Cập nhật cơ sở kiến thức pháp lý (luật lao động) và tích hợp công cụ (Tool) tính toán thời hiệu khởi kiện.
3. **Stage 3 (Single Agent):** Chuyển đổi sang kiến trúc ReAct Agent tự chủ, cung cấp thêm công cụ tra cứu án lệ (`search_case_law`).
4. **Stage 4 (Multi-Agent In-Process):** Thiết kế LangGraph định tuyến điều kiện (conditional routing) tự động phân luồng câu hỏi cho 3 agents chuyên gia chạy song song: Thuế, Tuân thủ (Compliance), và Quyền riêng tư (Privacy/GDPR).
5. **Stage 5 (Distributed A2A System):** Cấu hình chạy các Agents dưới dạng Microservices phân tán độc lập, tinh chỉnh system prompt cho Tax Agent và kiểm thử tính năng service discovery thông qua Registry Service.

## II. Trả Lời Câu Hỏi Ôn Tập

### 1. Khi nào nên dùng Single-Agent thay vì Multi-Agent?
- **Phạm vi bài toán hẹp:** Khi câu hỏi/nhiệm vụ chỉ yêu cầu kiến thức chuyên môn của một lĩnh vực duy nhất. Việc chia nhỏ thành nhiều Agent lúc này là không cần thiết và làm tăng độ phức tạp của mã nguồn.
- **Yêu cầu độ trễ thấp (Low Latency):** Multi-agent tiêu tốn thời gian để định tuyến, truyền tin và gộp kết quả từ nhiều cụm. Single agent giúp xử lý toàn bộ luồng công việc từ đầu tới cuối một cách đồng bộ và nhanh hơn.
- **Tiết kiệm tài nguyên:** Sử dụng Multi-agent đồng nghĩa với việc tiêu tốn nhiều memory, CPU và đặc biệt là API Tokens (do phải chia nhỏ prompt và truyền context nhiều lần). Single-agent kết hợp Tools là giải pháp tối ưu chi phí hơn.

### 2. Ưu điểm của A2A (Agent-to-Agent) Protocol so với REST/gRPC thông thường?
- **Ngữ nghĩa hướng AI (AI-Native):** REST hay gRPC chủ yếu xử lý việc trao đổi dữ liệu thô (JSON/Protobuf). A2A Protocol (của Google) chuẩn hóa các khái niệm dành riêng cho AI như `Agent Card` (hồ sơ năng lực), `Task` (nhiệm vụ/mục tiêu), và `Message History` (lịch sử ngữ cảnh).
- **Tiêu chuẩn hóa Ủy quyền (Delegation):** Giao thức A2A cho phép một Agent "giao việc" cho một Agent khác với mục tiêu và context cụ thể, thay vì chỉ gọi API lấy dữ liệu rồi tự xử lý.
- **Độ linh hoạt cao:** So với strict-typing của REST/gRPC, A2A truyền tải Prompt ngôn ngữ tự nhiên, giúp các Agents có thể giao tiếp và phối hợp suy luận linh hoạt kể cả với các tình huống chưa được lập trình cứng (hard-code) từ trước.

### 3. Làm thế nào để ngăn chặn vòng lặp vô hạn (Infinite Delegation Loops) trong A2A?
*(Ví dụ: Agent A hỏi Agent B -> B không biết lại hỏi ngược lại Agent A)*
- **Giới hạn độ sâu ủy quyền (Delegation Depth):** Cài đặt một ngưỡng tối đa (vd: `MAX_DELEGATION_DEPTH = 3`). Mỗi lần đi qua 1 Agent (hop), biến đếm sẽ tăng lên. Khi đạt ngưỡng, yêu cầu sẽ bị ngắt và từ chối.
- **Truyền Trace ID / Routing History:** Đính kèm lịch sử định tuyến vào request. Khi một Agent nhận được request, nó sẽ kiểm tra xem mình đã nằm trong lịch sử xử lý chưa. Nếu có, nó sẽ từ chối ủy quyền tiếp hoặc trả luôn kết quả hiện tại để bẻ gãy vòng lặp.

### 4. Tại sao cần Registry service? Có thể hardcode URLs không?
- **Có thể hardcode URL không?** Hoàn toàn có thể (ví dụ hardcode `http://localhost:10102` cho Tax Agent). Tuy nhiên, đây là Anti-Pattern đối với hệ thống phân tán và Microservices.
- **Tại sao cần Registry (Service Discovery)?**
  - **Dynamic Routing:** Các Agent không cần biết chính xác địa chỉ IP của nhau. Khi cần, chúng chỉ gọi `discover("tax_specialist")`, Registry sẽ tự động trả về endpoint khả dụng.
  - **Khả năng mở rộng (Scalability & Load Balancing):** Khi hệ thống quá tải, ta có thể khởi chạy thêm nhiều server Tax Agent. Registry sẽ đóng vai trò Load Balancer tự động phân phối request đều cho các server.
  - **Khả năng chịu lỗi (Fault Tolerance):** Nếu một Agent bị lỗi hoặc sập (crash), nó sẽ không còn trong danh sách của Registry. Các request sẽ không bị gửi nhầm tới một server "chết", giúp hệ thống hoạt động đáng tin cậy hơn.
