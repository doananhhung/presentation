# Member 1 Presentation Script - System Architecture

## Slide 1: Title Slide - System Architecture

**Script:**

"Xin chào các thầy cô và các bạn. Tôi là thành viên số 1 trong nhóm, đảm nhận vai trò System Architect cho dự án này.

Trong phần trình bày của mình, tôi sẽ đi qua kiến trúc tổng thể của hệ thống, bao gồm: kiến trúc Event-Driven Core, cách triển khai, hệ thống Webhooks, và Audit Logs.

Các thành phần này là nền tảng cho toàn bộ hệ thống chat hỗ trợ khách hàng real-time của chúng tôi."

---

## Slide 2: System Overview

**Script:**

"Trước tiên, để các bạn có cái nhìn tổng quan về hệ thống chúng tôi đang xây dựng.

Về **Application Type**: Đây là một nền tảng Customer Support Chat Platform, cho phép real-time messaging giữa Visitor - người truy cập website - và Agent - nhân viên hỗ trợ. Hệ thống bao gồm một chat widget có thể nhúng vào bất kỳ website nào của khách hàng, và một dashboard quản lý dành cho các nhân viên hỗ trợ.

Về **Architecture Style**: Chúng tôi chọn kiến trúc Event-Driven Microservices. Các điểm đặc biệt là:
- **Real-time**: Sử dụng WebSocket thông qua Socket.IO để đảm bảo tin nhắn được truyền trong thời gian thực
- **Multi-tenant**: Hỗ trợ nhiều công ty khác nhau sử dụng cùng hệ thống, với dữ liệu được cô lập hoàn toàn theo từng Project
- **Decoupled**: Các thành phần giao tiếp thông qua EventEmitter2 Bus, giúp hệ thống linh hoạt và dễ mở rộng"

---

## Slide 3: System Components Overview

**Script:**

"Bây giờ chúng ta sẽ đi sâu vào các thành phần chính của hệ thống qua sơ đồ này.

Hệ thống được chia thành 5 tầng chính:

**Tầng Frontend** gồm hai phần:
- Agent Dashboard: Được viết bằng React, đây là giao diện làm việc của nhân viên hỗ trợ
- Chat Widget: Được viết bằng Preact - một phiên bản nhẹ hơn của React - để đảm bảo tải nhanh khi nhúng vào website khách hàng

**Tầng WebSocket Layer**: Sử dụng Socket.IO Gateway để xử lý tất cả các kết nối real-time. Đặc biệt, chúng tôi sử dụng cơ chế Project Rooms để cô lập các sự kiện theo từng project.

**Tầng Backend**: Xây dựng trên NestJS framework, bao gồm:
- REST Controllers để xử lý các API request
- Domain Services chứa business logic
- Auth Guards và RBAC để kiểm soát quyền truy cập

**Background Workers**: Xử lý các tác vụ nặng như gửi webhook mà không làm block main thread. Chúng tôi dùng BullMQ để quản lý queue.

**Cuối cùng là Infrastructure layer**: Gồm PostgreSQL để lưu trữ dữ liệu, và Redis phục vụ cho cache, queue, và pub/sub.

Các thành phần này làm việc phối hợp với nhau để tạo nên một hệ thống real-time hiệu quả và scalable."

---

## Slide 4: Multi-Tenancy with Projects

**Script:**

"Một trong những đặc điểm quan trọng nhất của hệ thống là khả năng Multi-Tenancy.

Về **Data Isolation**: Mọi entity trong hệ thống đều có projectId. Đây là đơn vị cô lập dữ liệu gốc. Điều này có nghĩa là:
- Project là container chứa tất cả dữ liệu của một công ty
- ProjectMember liên kết User với Project
- Mọi request đều phải validate project membership trước khi cho phép truy cập

Về **Role Hierarchy**: Hệ thống có 2 role chính:
- **MANAGER**: Có toàn quyền quản lý - bao gồm cấu hình hệ thống, xem báo cáo, và quản lý team members
- **AGENT**: Quyền chat với khách hàng và quản lý conversation

Điểm quan trọng nhất là: Dữ liệu của công ty A không bao giờ có thể lẫn với công ty B. Mỗi project là một môi trường độc lập hoàn toàn."

---

## Slide 5: Complete Message Flow

**Script:**

"Slide này mô tả luồng gửi tin nhắn HOÀN CHỈNH, từ khi người dùng nhấn nút 'Gửi' cho đến khi tin nhắn được deliver và xử lý background. Hệ thống được chia làm 4 phases:

**Phase 1 - OPTIMISTIC UI** (~5ms):
Ngay lập tức khi người gửi nhấn 'Gửi', giao diện hiển thị tin nhắn với status SENDING. Điều này tạo cảm giác hệ thống phản hồi CỰC NHANH.

**Phase 2 - CRITICAL PATH** (~60ms):
UI gửi tin nhắn qua Socket.IO hoặc REST API đến Gateway. Gateway trigger event qua EventEmitter. MessageService nhận event và lưu tin nhắn vào PostgreSQL. Quan trọng: Tin nhắn và outbox entry được lưu trong CÙNG MỘT TRANSACTION để đảm bảo data consistency.

**Phase 3 - REAL-TIME BROADCAST**:
Khi transaction commit thành công, PostgreSQL trigger NOTIFY event. Redis Pub/Sub nhận event và broadcast đến TẤT CẢ người nhận đang online - bao gồm cả Dashboard của Agent và Widget của Visitor. UI của người gửi cũng nhận confirm và cập nhật status từ SENDING thành SENT.

**Phase 4 - BACKGROUND PROCESSING** (async):
Song song với việc broadcast, MessageService enqueue webhook job vào BullMQ. Webhook processor xử lý job này và gửi HTTP POST đến external systems của khách hàng. Phase này chạy hoàn toàn background, KHÔNG ảnh hưởng đến tốc độ chat real-time.

Kết quả: Từ lúc nhấn 'Gửi' đến khi người nhận thấy tin nhắn chỉ mất ~60ms. Webhooks được xử lý sau, không làm chậm trải nghiệm người dùng."

---

## Slide 6: Visitor → Agent Message Flow

**Script:**

"Bây giờ chúng ta sẽ xem chi tiết luồng xử lý khi Visitor gửi tin nhắn đến Agent.

Luồng này được chia làm HAI paths:

**Critical Path - Real-time** (~60ms):
1. **Widget** gửi tin nhắn qua Socket.IO
2. **Gateway** nhận tin nhắn và phát event qua EventEmitter
3. **MessageService** xử lý và lưu TRỰC TIẾP vào **PostgreSQL** - không qua queue
4. **Outbox Pattern** kết hợp NOTIFY trigger đảm bảo exactly-once delivery
5. **Redis Pub/Sub** broadcast tin nhắn đến **Dashboard** của Agent

Chỉ trong ~60ms, Agent đã thấy tin nhắn!

**Background Path - Không block UX**:
Sau khi lưu DB, MessageService enqueue job vào **BullMQ** để:
- Gửi webhook đến external systems
- Gửi email notifications
- Process AI chatbot

Thiết kế này đảm bảo: Real-time messaging không bị chậm bởi các background tasks."

---

## Slide 7: Visitor → Agent: Step by Step

**Script:**

"Slide này chia rõ hai paths: Critical Path cho real-time, và Background Path cho các tác vụ không quan trọng.

**Critical Path** - Đảm bảo real-time (~60ms):

**Socket.IO**: Gửi tin nhắn với độ trễ cực thấp

**EventEmitter2**: Decouple components - Gateway không cần biết MessageService sẽ làm gì

**MessageService**: Xử lý business logic và LƯU TRỰC TIẾP vào PostgreSQL. Điểm quan trọng: KHÔNG qua queue, để đảm bảo tốc độ.

**Outbox Pattern**: Đảm bảo exactly-once delivery. Tin nhắn và outbox entry được lưu trong cùng transaction - nếu server crash, không bị mất dữ liệu.

**Redis Pub/Sub**: Broadcast siêu nhanh đến Dashboard. Trong ~60ms kể từ khi visitor gửi, agent đã thấy tin nhắn!

**Background Path** - Không ảnh hưởng UX:

**BullMQ**: Chỉ dùng để queue các background jobs như webhook, email notification.

**Webhook Processor**: Chạy background, có thể mất vài trăm milliseconds nhưng không block chat real-time.

Kiến trúc này balance giữa tốc độ real-time và độ tin cậy."

---

## Slide 8: Agent → Visitor Message Flow

**Script:**

"Luồng ngược lại - khi Agent trả lời Visitor - được thiết kế với sự phân tách rõ ràng các concerns.

Đầu tiên, Agent gửi tin nhắn qua **REST API** thay vì Socket.IO. Điều này hợp lý vì Dashboard không cần optimize tốc độ gửi như Widget.

**MessageService** thực hiện một transaction phức tạp:
- Lưu tin nhắn vào PostgreSQL
- Đồng thời lookup **Redis Session** để lấy socketId của visitor đang online

Sau khi transaction thành công, điểm quan trọng là MessageService KHÔNG gọi trực tiếp Gateway. Thay vào đó, nó phát event 'agent.message.sent' qua **EventEmitter**.

**GatewayEventListener** - một component quan trọng mà chúng ta thường bỏ qua - lắng nghe event này thông qua decorator @OnEvent. Component này đóng vai trò bridge giữa business logic layer và WebSocket layer, giúp hệ thống decoupled và dễ maintain.

**EventsGateway** nhận lệnh từ listener và thực hiện dual-broadcast:
- Gửi event **AGENT_REPLIED** trực tiếp đến Widget của visitor cụ thể (sử dụng socketId đã lookup trước đó)
- Gửi event **NEW_MESSAGE** đến tất cả **Other Agents** trong project room để họ cập nhật dashboard real-time

Thiết kế này tuân thủ Single Responsibility Principle: MessageService lo business logic, GatewayEventListener lo event handling, EventsGateway lo WebSocket communication."

---

## Slide 9: Section Divider - Deployment & Tech Stack

**Script:**

"Tiếp theo, tôi sẽ trình bày về công nghệ và cấu trúc Monorepo mà chúng tôi sử dụng."

---

## Slide 10: Technology Stack

**Script:**

"Về technology stack:

**Backend** của chúng tôi:
- Runtime: Node.js phiên bản 18 trở lên
- Framework: NestJS - một framework TypeScript mạnh mẽ, hỗ trợ dependency injection và modular architecture
- Database: PostgreSQL cho data persistence
- Cache và Queue: Redis kết hợp BullMQ
- Real-time: Socket.IO

**Frontend**:
- Dashboard: React - framework phổ biến cho complex UI
- Widget: Preact - chỉ nặng 3KB, rất phù hợp cho embedded widget
- State Management: Zustand - nhẹ và đơn giản hơn Redux
- Styling: TailwindCSS cho productivity cao

**DevOps**:
- Container: Docker Compose phiên bản 2 trở lên
- Monorepo: npm workspaces để quản lý multiple packages

Tất cả các công nghệ này được chọn lựa kỹ càng để balance giữa performance, developer experience, và khả năng maintain."

---

## Slide 11: Monorepo Structure

**Script:**

"Dự án được tổ chức theo mô hình Monorepo với cấu trúc rất rõ ràng:

Thư mục **packages** chứa tất cả source code:

- **backend**: Chứa NestJS API và Worker processes, được chia thành các modules như:
  - auth: Xử lý Authentication
  - inbox: Quản lý Messages và Conversations  
  - gateway: WebSocket handling
  - webhooks: External integration

- **frontend**: Chứa cả React Dashboard và Preact Widget

- **shared-***: Các packages chứa shared DTOs và Types được dùng chung giữa frontend và backend

Thư mục **docs**: Chứa toàn bộ documentation

Lợi ích của cấu trúc Monorepo này là:
- Code sharing dễ dàng giữa frontend và backend
- Build và deploy thống nhất
- Refactoring an toàn hơn vì thay đổi ảnh hưởng đến tất cả consumers ngay lập tức"

---

## Slide 12: Section Divider - Event-Driven Core

**Script:**

"Bây giờ chúng ta sẽ đi sâu vào phần quan trọng nhất của kiến trúc: Event-Driven Core và Socket.IO Room Isolation."

---

## Slide 13: Event Architecture

**Script:**

"Đây là sơ đồ kiến trúc Event của hệ thống.

**Domain Services** ở tầng Backend:
- ConversationService: Quản lý conversation lifecycle
- MessageService: Xử lý messages
- VisitorService: Quản lý visitor state

Khi các service này thực hiện một action quan trọng, họ không gọi trực tiếp các consumer. Thay vào đó, họ phát ra events thông qua **EventEmitter2 Bus**.

Các events tiêu biểu như:
- conversation.updated: Khi conversation được assign hoặc đổi status
- agent.message.sent: Khi agent gửi tin nhắn
- visitor.updated: Khi visitor information thay đổi

**GatewayEventListener** lắng nghe các events này. Ví dụ:
- handleConversationUpdated lắng nghe conversation.updated
- handleAgentMessageSent lắng nghe agent.message.sent

Các handler này sau đó gọi **EventsGateway** để broadcast đến các client thông qua WebSocket.

Kiến trúc này giúp hệ thống decoupled - Services chỉ cần quan tâm đến business logic, không cần biết ai sẽ xử lý events của họ."

---

## Slide 14: Socket.IO Room Isolation

**Script:**

"Một trong những thách thức lớn nhất trong multi-tenant realtime system là làm sao để cô lập events giữa các projects.

Chúng tôi giải quyết bằng Socket.IO Rooms.

Code minh họa ở đây là function handleJoinProjectRoom, được gọi khi agent muốn join vào project room.

**Bước 1 - Authentication**: Kiểm tra client phải đăng nhập. Nếu không có user data, throw WsException Unauthorized.

**Bước 2 - Authorization**: Validate xem user có phải là member của project này không bằng cách gọi projectService.validateProjectMembership. Nếu không phải member, sẽ throw exception.

**Bước 3 - Join Room**: Chỉ khi pass cả 2 bước trên, client mới được join vào room với tên là 'project:{projectId}'.

Khi broadcast events, chúng tôi dùng syntax:
```
this.server.to(`project:${projectId}`).emit(...)
```

Điều này đảm bảo event chỉ được gửi đến các clients trong room cụ thể đó.

Kết quả là: Agent của công ty A **hoàn toàn không thể** nhận được event của công ty B. Đây là security measure quan trọng nhất của hệ thống."

---

## Slide 15: Event Catalog

**Script:**

"Để có cái nhìn tổng quan, đây là catalog của các events trong hệ thống:

**Inbox Events**:
- conversationUpdated: Được trigger khi conversation được assign cho agent hoặc status thay đổi (ví dụ từ OPEN sang RESOLVED)
- newMessage: Trigger mỗi khi có tin nhắn mới, từ visitor hoặc agent

**Visitor Events**:
- visitorStatusChanged: Trigger khi visitor connect hoặc disconnect khỏi website
- visitorIsTyping: Trigger khi visitor đang gõ phím, cho phép agent thấy typing indicator
- visitorContextUpdated: Trigger khi visitor di chuyển giữa các trang, cho phép agent biết visitor đang xem trang nào

Tất cả các events này đều follow naming convention rõ ràng và được type-safe bằng TypeScript."

---

## Slide 16: Section Divider - Webhooks

**Script:**

"Phần tiếp theo tôi sẽ nói về hệ thống Webhooks - cho phép tích hợp với các external systems."

---

## Slide 17: Webhook Architecture

**Script:**

"Webhook architecture của chúng tôi được thiết kế để xử lý high-volume events một cách reliable.

**Trigger**: Khi có message created hoặc các events quan trọng khác

**System Flow**:
1. Event được publish lên **Redis Pub/Sub**
2. **Dispatcher** lắng nghe Redis và enqueue các webhook jobs vào **BullMQ Queue**
3. **Processor** consume các jobs từ queue và thực hiện HTTP POST đến **Customer Server**

Lợi ích của kiến trúc này:
- **Asynchronous**: Không block main request flow
- **Reliable**: BullMQ hỗ trợ retry automatic nếu customer server down
- **Scalable**: Có thể scale số lượng processors độc lập
- **Traceable**: Mỗi webhook delivery được log đầy đủ"

---

## Slide 18: Webhook Components & Security

**Script:**

"Về các components:

**Dispatcher**: Lắng nghe Redis Pub/Sub và enqueue các webhook jobs vào BullMQ

**Processor**: Thực hiện HTTP POST đến customer URL, implement retry logic, và tính toán HMAC signature

**Delivery Log**: Theo dõi chi tiết trạng thái của từng lần gửi webhook - success, failed, pending retry

Đặc biệt quan trọng là **SSRF Protection** - Server-Side Request Forgery Protection:

Chúng tôi implement 4 layers bảo vệ:

1. **HTTPS only**: Chỉ chấp nhận webhook URL với https:// protocol, không cho phép http, file, hoặc các protocol khác

2. **DNS Validation**: Resolve hostname trước khi gửi request để validate nó là domain hợp lệ

3. **Block Private IPs**: Reject các IP thuộc dải private như:
   - 127.0.0.0/8 (localhost)
   - 10.0.0.0/8 (private network)
   - 192.168.0.0/16 (private network)
   - Và các dải IP internal khác
   
   Điều này ngăn attacker dùng webhook để scan internal network của chúng ta.

4. **HMAC Signature**: Mỗi webhook request đều có header X-Hub-Signature-256 chứa HMAC signature. Customer server có thể verify request thật sự đến từ hệ thống của chúng tôi.

Các biện pháp này đảm bảo webhook system vừa flexible vừa secure."

---

## Slide 19: Section Divider - Audit Logs

**Script:**

"Phần cuối cùng tôi muốn trình bày là hệ thống Audit Logs - rất quan trọng cho security compliance và investigation."

---

## Slide 20: Audit System

**Script:**

"Hệ thống Audit của chúng tôi có các đặc điểm sau:

**Mục đích**: Phục vụ Security compliance - đáp ứng các yêu cầu về audit trail cho các hành động quan trọng trong hệ thống.

**Cơ chế**: Sử dụng Decorator-based Interceptor. Developers chỉ cần thêm một decorator lên method, hệ thống sẽ tự động log.

Ví dụ rất đơn giản:
```typescript
@Auditable({ 
  action: AuditAction.UPDATE, 
  entity: 'Conversation' 
})
@Patch(':id/assign')
async assign(@Body() dto) { ... }
```

Mỗi khi method assign được gọi, hệ thống tự động tạo audit log với action là UPDATE và entity là Conversation.

**Pattern**: Chúng tôi áp dụng Fail-Open pattern - nghĩa là nếu audit logging fails, operation vẫn tiếp tục. Không để audit system làm crash business logic.

**Storage**: Audit logs được lưu trong PostgreSQL với JSONB columns để store flexible metadata. Điều này cho phép query hiệu quả và lưu trữ các custom fields."

---

## Slide 21: Sensitive Data Redaction

**Script:**

"Một vấn đề quan trọng khi logging là **Sensitive Data Redaction**.

Chúng tôi định nghĩa một list các SENSITIVE_KEYS như:
- password
- token
- secret
- authorization
- apikey
- creditcard, cvv, ssn

Khi log request body hoặc response, hệ thống sẽ tự động scan và redact các fields này.

Kết quả trong log, bạn sẽ thấy:
```json
{
  "email": "user@example.com",
  "password": "[REDACTED]",
  "token": "[REDACTED]"
}
```

Email được giữ nguyên vì nó không sensitive, nhưng password và token được redact.

Hai điểm quan trọng:
1. **Case-insensitive**: Matching không phân biệt hoa thường, nên 'Password', 'PASSWORD', 'password' đều được redact
2. **Recursive**: Hệ thống scan deep vào nested objects và arrays

Điều này đảm bảo chúng tôi comply với các data protection regulations như GDPR, PCI-DSS khi store audit logs."

---

## Slide 22: Section Divider - Summary

**Script:**

"Bây giờ tôi sẽ tổng kết lại những gì đã trình bày."

---

## Slide 23: Architecture Recap

**Script:**

"Trong phần trình bày của mình, tôi đã đi qua 6 chủ đề chính:

**Kiến trúc**: Hệ thống được xây dựng theo Event-Driven Microservices architecture trên nền tảng NestJS

**Multi-tenancy**: Cô lập dữ liệu hoàn toàn theo Project với Role-Based Access Control

**Real-time Communication**: Sử dụng Socket.IO Rooms để isolate events và EventEmitter2 để decouple components

**Message Flow**: Áp dụng Optimistic UI pattern cho trải nghiệm người dùng tốt hơn, và Outbox Pattern để đảm bảo message reliability

**External Integration**: Hệ thống Webhooks với đầy đủ SSRF Protection để integration an toàn với external systems

**Compliance**: Audit Logs system với Fail-Open pattern và Sensitive Data Redaction để comply với security regulations

Tất cả các quyết định kiến trúc này đều hướng đến mục tiêu: Xây dựng một hệ thống scalable, secure, và maintainable."

---

## Slide 24: Handoff to Next Presenter

**Script:**

"Như vậy là tôi đã hoàn thành phần trình bày về System Architecture.

Tôi đã covered các topics:
- System Architecture Overview
- Multi-tenancy và Project Isolation
- Message Flow Patterns
- Event-Driven Core
- Webhooks và Security
- Audit Logs

Phần tiếp theo sẽ được trình bày bởi Member 2 - Core Developer phụ trách Authentication. Bạn ấy sẽ đi sâu vào:
- JWT Authentication mechanism
- OAuth Integration với third-party providers
- Two-Factor Authentication - 2FA
- Session Management

Cảm ơn các bạn đã lắng nghe. Tôi xin dừng phần trình bày của mình tại đây và chuyển microphone cho Member 2."

---

## Tips for Presentation Delivery

### General Guidelines:
1. **Pace**: Nói với tốc độ vừa phải, khoảng 120-150 từ/phút
2. **Pause**: Dừng ngắn sau mỗi ý quan trọng để audience absorb
3. **Eye Contact**: Nhìn vào audience, không chỉ đọc slides
4. **Gestures**: Sử dụng tay chỉ vào các phần quan trọng trên slides

### Technical Terms:
- Đọc rõ các thuật ngữ tiếng Anh như "Event-Driven", "Socket.IO", "SSRF Protection"
- Giải thích ngắn gọn các concept phức tạp bằng ví dụ thực tế

### Time Management:
- Mỗi slide nên mất khoảng 45-90 giây
- Tổng thời gian ~15-20 phút cho 24 slides
- Dành 2-3 phút cuối cho Q&A nếu cần

### Handling Questions:
- Nếu không biết câu trả lời, thành thật nói "Đây là câu hỏi hay, tôi sẽ research thêm và trả lời sau"
- Redirect technical details sang members khác nếu phù hợp
- Keep answers concise, không đi quá sâu

### Pre-presentation Checklist:
- [ ] Đọc qua script ít nhất 2 lần
- [ ] Practice với timer
- [ ] Chuẩn bị demo (nếu có)
- [ ] Test slides transitions
- [ ] Backup slides trên USB/cloud
- [ ] Uống nước trước khi trình bày

Good luck! 🎤
