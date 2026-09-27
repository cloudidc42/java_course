# Part 90: WebSocket, Real-time Applications

> ขั้นตอนที่ 891-900 ของหลักสูตร | ระดับ: สูงมาก

## สารบัญ

1. ทำไม HTTP ธรรมดาไม่เหมาะกับ Real-time Communication
2. WebSocket Protocol คืออะไร
3. ทางเลือกอื่นก่อน WebSocket: Polling และ Long Polling
4. STOMP: Simple Text Oriented Messaging Protocol
5. ตั้งค่า Spring WebSocket + STOMP
6. เขียน Server-side: `@MessageMapping`, `SimpMessagingTemplate`
7. Broadcast ข้อความให้ทุกคนที่เชื่อมต่อ (Pub/Sub)
8. ส่งข้อความถึงผู้ใช้เฉพาะคน (Private Messaging)
9. WebSocket กับ Authentication (JWT)
10. แบบฝึกหัดและสรุป

---

## 1. ทำไม HTTP ธรรมดาไม่เหมาะกับ Real-time Communication

ทบทวนจาก Part 71: HTTP เป็นแบบ **request-response** — **client ต้องเป็น
ฝ่ายเริ่มขอข้อมูลก่อนเสมอ** server ไม่สามารถ "ส่งข้อมูลเข้ามาเอง" โดยที่
client ไม่ได้ขอได้ ปัญหานี้ชัดเจนมากในแอปที่ต้องการ**อัปเดตแบบทันที** เช่น
แชท, การแจ้งเตือน real-time, dashboard ที่ต้องอัปเดตกราฟสด ๆ

```java
public class WhyHttpFailsForRealtime {
    /*
     * แชทแอปด้วย HTTP ธรรมดา:
     *   User A ส่งข้อความ -> POST /messages -> server เก็บข้อความ
     *   User B จะเห็นข้อความนี้ได้อย่างไร? server ส่งไปให้ B เองไม่ได้!
     *   B ต้อง "ถาม" server ซ้ำ ๆ ว่า "มีข้อความใหม่ไหม?" (Polling - หัวข้อ 3)
     */
}
```

## 2. WebSocket Protocol คืออะไร

**WebSocket** เป็น protocol ที่เปิด **connection แบบสองทาง (full-duplex)
ค้างไว้ตลอด** ระหว่าง client กับ server — ทั้งสองฝั่ง**ส่งข้อมูลถึงกันได้
ทุกเมื่อ**โดยไม่ต้องรอให้อีกฝั่ง "ขอ" ก่อน (ต่างจาก HTTP request-response
โดยสิ้นเชิง)

```
HTTP (request-response):          WebSocket (full-duplex):
Client ──request──▶ Server        Client ══connection เปิดค้าง══ Server
Client ◀─response── Server              │                          │
(connection ปิดทุกครั้ง)                  ◀── message ────────────┤
                                          ├──── message ──────────▶
                                     (connection เดียวกัน ส่งได้ทั้งสองทาง
                                      ตลอดเวลาโดยไม่ต้องเปิดใหม่)
```

WebSocket เริ่มต้นด้วย **HTTP Handshake** (ทบทวน HTTP Header จาก Part
71) แล้ว "upgrade" เป็น WebSocket connection ที่**ค้างอยู่ตลอดจนกว่าจะ
ปิด**

## 3. ทางเลือกอื่นก่อน WebSocket: Polling และ Long Polling

```java
public class PollingVsWebSocketDemo {
    /*
     * Short Polling: client ยิง request ทุก ๆ N วินาที ถามว่า "มีอะไรใหม่ไหม"
     *   ข้อเสีย: ถ้า N สั้น -> เปลืองทรัพยากรมาก (ยิง request บ่อยเกินจำเป็น)
     *           ถ้า N นาน -> ข้อมูลอัปเดตช้า (ไม่ real-time จริง)
     *
     * Long Polling: client ยิง request แล้ว "รอ" (server ไม่ตอบทันที) จนกว่ามีข้อมูลใหม่หรือ timeout
     *   ดีกว่า short polling แต่ยังต้องเปิด connection ใหม่ทุกครั้งที่ตอบกลับ (overhead)
     *
     * WebSocket: เปิด connection ครั้งเดียว ใช้ซ้ำได้ตลอด ไม่มี overhead จากการเปิด/ปิด connection ซ้ำ ๆ
     *   เหมาะที่สุดสำหรับข้อมูลที่ต้อง real-time จริง ๆ และมีความถี่การส่งสูง
     */
}
```

## 4. STOMP: Simple Text Oriented Messaging Protocol

WebSocket protocol ดิบ ๆ (raw) จัดการแค่การส่ง byte ไปมา — **STOMP** เป็น
protocol ระดับสูงกว่าที่ทำงานบน WebSocket ให้มีแนวคิดแบบ **pub/sub**
(publish/subscribe) คล้ายกับ Message Queue ที่เรียนใน Part 88

```java
public class StompConceptDemo {
    /*
     * STOMP เพิ่มแนวคิด "destination" (คล้าย Topic ใน Kafka - Part 89):
     *   Client subscribe ไปที่ "/topic/chat-room-1"
     *   Server ส่งข้อความไปที่ "/topic/chat-room-1"
     *   -> ทุก client ที่ subscribe destination นี้จะได้รับข้อความพร้อมกัน
     *
     * ทำให้เขียนแอป real-time ง่ายขึ้นมาก เพราะไม่ต้องจัดการ raw byte เอง
     */
}
```

## 5. ตั้งค่า Spring WebSocket + STOMP

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

```java
import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.*;

@Configuration
@EnableWebSocketMessageBroker // เปิดใช้ STOMP over WebSocket (ทบทวน @Enable* จาก Part 74, 87)
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws") // client เชื่อมต่อ WebSocket ที่ path นี้
                .setAllowedOriginPatterns("*") // ทบทวน CORS จาก Part 71 - ควรจำกัด origin ใน production
                .withSockJS(); // fallback เป็น long-polling ถ้า browser ไม่รองรับ WebSocket จริง
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableSimpleBroker("/topic", "/queue"); // broker ในตัว จัดการ destination เหล่านี้
        registry.setApplicationDestinationPrefixes("/app"); // message จาก client ที่ขึ้นต้นด้วย /app ไปที่ @MessageMapping
    }
}
```

## 6. เขียน Server-side: `@MessageMapping`, `SimpMessagingTemplate`

```java
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.stereotype.Controller;
import java.time.Instant;

@Controller
public class ChatController {

    record ChatMessage(String sender, String content, Instant sentAt) {}

    @MessageMapping("/chat.send") // client ส่งไปที่ "/app/chat.send" (prefix จาก config หัวข้อ 5)
    @SendTo("/topic/public") // ส่งผลลัพธ์ไปที่ destination นี้ ให้ทุก subscriber ได้รับ
    public ChatMessage sendMessage(ChatMessage incoming) {
        System.out.println(incoming.sender() + " sent: " + incoming.content());
        return new ChatMessage(incoming.sender(), incoming.content(), Instant.now());
        // ทบทวนแนวคิด @MessageMapping คล้าย @RequestMapping (Part 77) แต่ทำงานผ่าน WebSocket ไม่ใช่ HTTP request-response
    }
}
```

**Client-side (JavaScript, สังเขปเพื่อความเข้าใจ)**:

```java
public class ClientSideJsExplanation {
    /*
     * const socket = new SockJS('/ws');
     * const stompClient = Stomp.over(socket);
     * stompClient.connect({}, () => {
     *     stompClient.subscribe('/topic/public', (message) => {
     *         console.log(JSON.parse(message.body)); // รับข้อความใหม่ทันทีที่มีคนส่ง
     *     });
     *     stompClient.send('/app/chat.send', {}, JSON.stringify({sender: 'Alice', content: 'Hello!'}));
     * });
     */
}
```

## 7. Broadcast ข้อความให้ทุกคนที่เชื่อมต่อ (Pub/Sub)

นอกจาก `@SendTo` ที่ตอบกลับทันทีในเมธอดเดียวกัน บางครั้งต้อง **broadcast
จากที่อื่นในแอป** (เช่น จาก Service layer ที่ไม่ได้เกิดจาก WebSocket
message โดยตรง) — ใช้ `SimpMessagingTemplate`:

```java
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.stereotype.Service;

@Service
public class OrderStatusNotifier {
    private final SimpMessagingTemplate messagingTemplate;

    public OrderStatusNotifier(SimpMessagingTemplate messagingTemplate) {
        this.messagingTemplate = messagingTemplate;
    }

    record OrderStatusUpdate(Long orderId, String newStatus) {}

    public void notifyStatusChange(Long orderId, String newStatus) {
        // ส่ง broadcast ไปที่ /topic/orders - ทุก client ที่ subscribe จะได้รับทันที
        messagingTemplate.convertAndSend("/topic/orders", new OrderStatusUpdate(orderId, newStatus));
        // เหมาะกับ dashboard ที่ต้องอัปเดตสถานะ order แบบ real-time โดยไม่ต้อง refresh หน้าเว็บ
    }
}
```

```java
@Service
public class OrderService {
    private final OrderStatusNotifier notifier;

    public OrderService(OrderStatusNotifier notifier) { this.notifier = notifier; }

    public void shipOrder(Long orderId) {
        // ... business logic เปลี่ยนสถานะ order เป็น SHIPPED ...
        notifier.notifyStatusChange(orderId, "SHIPPED"); // แจ้งทุก client ที่กำลังดู dashboard ทันที
    }
}
```

## 8. ส่งข้อความถึงผู้ใช้เฉพาะคน (Private Messaging)

Broadcast (หัวข้อ 7) เหมาะกับข้อมูลสาธารณะ แต่การแจ้งเตือนส่วนตัว (เช่น
"คำสั่งซื้อของคุณจัดส่งแล้ว") ต้องส่งถึง**ผู้ใช้คนเดียว**เท่านั้น:

```java
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.stereotype.Service;

@Service
public class PrivateNotificationService {
    private final SimpMessagingTemplate messagingTemplate;

    public PrivateNotificationService(SimpMessagingTemplate messagingTemplate) {
        this.messagingTemplate = messagingTemplate;
    }

    public void notifyUser(String username, String message) {
        // convertAndSendToUser ผูก destination กับ username โดยอัตโนมัติ (ต้องมี Principal - ทบทวนหัวข้อ 9)
        messagingTemplate.convertAndSendToUser(username, "/queue/notifications", message);
        // client subscribe ที่ "/user/queue/notifications" (Spring แปลง /user/ prefix ให้อัตโนมัติตาม session)
    }
}
```

## 9. WebSocket กับ Authentication (JWT)

ทบทวนจาก Part 83: WebSocket handshake เป็น HTTP request ครั้งเดียวตอน
เริ่มต้น — สามารถแนบ JWT ผ่าน query parameter หรือ header ตอน handshake
เพื่อระบุตัวผู้ใช้:

```java
import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.stomp.StompCommand;
import org.springframework.messaging.simp.stomp.StompHeaderAccessor;
import org.springframework.messaging.support.ChannelInterceptor;
import org.springframework.messaging.Message;
import org.springframework.messaging.MessageChannel;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;
import org.springframework.messaging.simp.config.ChannelRegistration;

@Configuration
public class WebSocketAuthConfig implements WebSocketMessageBrokerConfigurer {
    private final JwtVerificationService jwtService; // ทบทวนจาก Part 83

    public WebSocketAuthConfig(JwtVerificationService jwtService) { this.jwtService = jwtService; }

    @Override
    public void configureClientInboundChannel(ChannelRegistration registration) {
        registration.interceptors(new ChannelInterceptor() {
            @Override
            public Message<?> preSend(Message<?> message, MessageChannel channel) {
                StompHeaderAccessor accessor = StompHeaderAccessor.wrap(message);
                if (StompCommand.CONNECT.equals(accessor.getCommand())) {
                    String token = accessor.getFirstNativeHeader("Authorization");
                    var claims = jwtService.verifyToken(token.replace("Bearer ", ""));
                    if (claims.isEmpty()) {
                        throw new org.springframework.messaging.MessagingException("Invalid JWT");
                    }
                    // ตั้งค่า Principal ให้ Spring รู้ว่า connection นี้เป็นของใคร (ใช้กับ convertAndSendToUser หัวข้อ 8)
                    accessor.setUser(() -> claims.get().getSubject());
                }
                return message; // ทบทวนแนวคิด ChannelInterceptor คล้าย Filter/Interceptor จาก Part 72, 83
            }
        });
    }
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียน endpoint `@MessageMapping("/chat.typing")` ที่ broadcast
ไปที่ `/topic/typing` เพื่อแสดงสถานะ "กำลังพิมพ์..." ของผู้ใช้

**เฉลย:**
```java
record TypingEvent(String username, boolean isTyping) {}

@MessageMapping("/chat.typing")
@SendTo("/topic/typing")
public TypingEvent notifyTyping(TypingEvent event) {
    return event; // ส่งต่อไปให้ผู้ใช้อื่นทุกคนที่ subscribe /topic/typing เห็นสถานะทันที
}
```

**2)** อธิบายว่าทำไม Long Polling ยังมี overhead มากกว่า WebSocket แม้จะ
ดีกว่า Short Polling แล้วก็ตาม

**เฉลย**: **Long Polling** ยังคงเป็น**การยิง HTTP request ใหม่ทุกครั้ง**
หลังจากได้รับ response แต่ละครั้ง (client ต้อง "เปิด request ใหม่" ทันที
ที่ response ก่อนหน้าจบลง เพื่อรอข้อมูลถัดไป) แต่ละ HTTP request มี
**overhead จากการเปิด connection ใหม่และส่ง header ครบชุดทุกครั้ง**
(ทบทวน HTTP Header จาก Part 71) ในขณะที่ **WebSocket เปิด connection
เพียงครั้งเดียว**และใช้ connection เดียวกันนั้น**ส่งข้อความไปมาได้ตลอดไป**
โดยไม่ต้องเปิด/ปิด connection ใหม่ซ้ำ ๆ ทำให้ overhead ต่ำกว่ามาก
โดยเฉพาะเมื่อความถี่ของการส่งข้อมูลสูง (เช่น แชทที่มีข้อความเข้ามาต่อเนื่อง)

**3)** ทำไมการแจ้งเตือนส่วนตัว (เช่น "คำสั่งซื้อของคุณจัดส่งแล้ว") ควรใช้
`convertAndSendToUser` แทนการ broadcast ผ่าน `/topic/...` ธรรมดา

**เฉลย**: การ broadcast ผ่าน `/topic/...` (หัวข้อ 7) จะส่งข้อความไปให้
**ทุก client ที่ subscribe destination นั้นทั้งหมด** — ถ้าใช้วิธีนี้กับ
การแจ้งเตือนส่วนตัว ผู้ใช้ทุกคนที่ subscribe topic เดียวกันจะเห็นการแจ้ง
เตือนของคนอื่นด้วย ซึ่งเป็น**การรั่วไหลของข้อมูลส่วนตัว**อย่างร้ายแรง
(ทบทวนหลักการ least privilege ที่เกี่ยวข้องกับ Authorization จาก Part 82)
`convertAndSendToUser` (หัวข้อ 8) ผูกข้อความกับ **username ที่ระบุใน
Principal ของ connection นั้นโดยเฉพาะ** (ทบทวนหัวข้อ 9 - การตั้ง Principal
ผ่าน JWT ตอน handshake) ทำให้ Spring ส่งข้อความไปถึง**เฉพาะ session ของ
ผู้ใช้คนนั้นเท่านั้น** ไม่มีผู้ใช้คนอื่นเห็นข้อความนี้เลย

### สรุปเนื้อหา Part 90

- HTTP request-response ไม่เหมาะกับ real-time communication เพราะ server
  ส่งข้อมูลเข้ามาเองไม่ได้
- WebSocket เปิด connection แบบ full-duplex ค้างไว้ ส่งข้อมูลได้ทั้งสองทาง
  ตลอดเวลา
- STOMP เพิ่มแนวคิด pub/sub (destination) บน WebSocket ทำให้เขียนแอป
  real-time ง่ายขึ้น
- `@MessageMapping`+`@SendTo` จัดการข้อความที่มาจาก client โดยตรง,
  `SimpMessagingTemplate` broadcast จากที่อื่นในแอป (เช่น Service layer)
- `convertAndSendToUser` ส่งข้อความถึงผู้ใช้เฉพาะคน ป้องกันข้อมูลส่วนตัว
  รั่วไหล
- WebSocket authentication ทำผ่าน interceptor ที่ตรวจสอบ JWT ตอน CONNECT
  command และผูก Principal เข้ากับ session

**หมวดที่ 5 (Web Development: Part 71-90) เสร็จสมบูรณ์แล้ว! รวม 90/105
Part — เหลือแค่หมวดสุดท้าย: Professional/World-class Level (Part 91-105)**

**ต่อไป**: [Part 91 — Microservices Architecture](./part-091-microservices-architecture.md)
