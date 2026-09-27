# Part 88: Messaging: RabbitMQ, Spring AMQP

> ขั้นตอนที่ 871-880 ของหลักสูตร | ระดับ: สูงมาก

## สารบัญ

1. Synchronous vs Asynchronous Communication
2. Message Queue คืออะไร แก้ปัญหาอะไร
3. RabbitMQ: Concepts พื้นฐาน (Exchange, Queue, Binding)
4. ติดตั้ง Spring AMQP และตั้งค่า Queue/Exchange
5. การส่งข้อความ (Producer) ด้วย `RabbitTemplate`
6. การรับข้อความ (Consumer) ด้วย `@RabbitListener`
7. Exchange Types: Direct, Topic, Fanout
8. Message Acknowledgment และการจัดการข้อผิดพลาด
9. Dead Letter Queue (DLQ)
10. แบบฝึกหัดและสรุป

---

## 1. Synchronous vs Asynchronous Communication

ทบทวนจาก Part 71, 78: การเรียก REST API เป็น **synchronous** — client
**รอ** จนกว่า server ตอบกลับก่อนทำงานถัดไป ถ้า server ช้าหรือ down, client
ก็ค้างหรือ error ตามไปด้วย (tight coupling ทางเวลา)

```java
public class SyncVsAsyncDemo {
    /*
     * Synchronous (REST call ตรง):
     *   OrderService -> เรียก EmailService.sendConfirmation() -> รอจนกว่า email ส่งสำเร็จ
     *   ถ้า EmailService ช้า (network ไปยัง SMTP server ล่าช้า) -> OrderService ก็ค้างรอด้วย!
     *   ถ้า EmailService down -> การสร้าง order ทั้งหมดล้มเหลวไปด้วย (แม้ order จะสร้างสำเร็จแล้ว)
     *
     * Asynchronous (ผ่าน Message Queue):
     *   OrderService -> ส่ง message "order created" ไปที่ Queue -> ทำงานต่อทันที (ไม่รอ)
     *   EmailService -> รับ message จาก Queue เมื่อพร้อม -> ส่ง email
     *   ถ้า EmailService down ชั่วคราว -> message รอใน Queue จนกว่า EmailService กลับมาทำงาน
     *   OrderService ไม่ได้รับผลกระทบเลย!
     */
}
```

## 2. Message Queue คืออะไร แก้ปัญหาอะไร

**Message Queue** เป็น**ตัวกลาง**ที่รับข้อความจาก **Producer** (ผู้ส่ง)
แล้วเก็บไว้จนกว่า **Consumer** (ผู้รับ) จะพร้อมประมวลผล — ทำให้ระบบสอง
ฝั่ง**ไม่ต้องทำงานพร้อมกันแบบ real-time** (decoupling ทั้งเวลาและ
availability)

```
┌──────────┐    message    ┌───────────┐    message    ┌──────────┐
│ Producer │ ─────────────▶│   Queue   │──────────────▶│ Consumer │
│  (สร้าง   │                │  (RabbitMQ)│                │ (ประมวลผล) │
│  order)  │                │            │                │  email)  │
└──────────┘                └───────────┘                └──────────┘
     ทำงานต่อทันที              เก็บ message ไว้              รับตอนพร้อม
     ไม่ต้องรอ Consumer          จนกว่า Consumer พร้อม          ประมวลผลตามจังหวะตัวเอง
```

**ประโยชน์หลัก**: (1) **Decoupling** — Producer/Consumer ไม่รู้จักกันโดยตรง
เปลี่ยนแปลงฝั่งใดฝั่งหนึ่งไม่กระทบอีกฝั่ง (ทบทวนหลักการ loose coupling
จาก Part 57), (2) **Load Leveling** — ถ้า Producer ส่งข้อความเร็วกว่า
Consumer ประมวลผลได้ Queue จะช่วย buffer ไม่ให้ Consumer ล่ม, (3)
**Reliability** — ข้อความไม่หายแม้ Consumer down ชั่วคราว

## 3. RabbitMQ: Concepts พื้นฐาน (Exchange, Queue, Binding)

```
Producer ──▶ Exchange ──(binding rule)──▶ Queue ──▶ Consumer
```

- **Exchange**: จุดที่ Producer ส่งข้อความเข้ามา — Exchange **ตัดสินใจ**
  ว่าจะส่งข้อความไปที่ Queue ไหนบ้าง (ตาม routing rule)
- **Queue**: ที่เก็บข้อความจริง — Consumer ดึงข้อความจาก Queue นี้
- **Binding**: กฎที่เชื่อม Exchange กับ Queue (บอกว่า message แบบไหนไปที่
  queue ไหน)

## 4. ติดตั้ง Spring AMQP และตั้งค่า Queue/Exchange

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

```java
import org.springframework.amqp.core.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitConfig {
    public static final String ORDER_QUEUE = "order.created.queue";
    public static final String ORDER_EXCHANGE = "order.exchange";
    public static final String ORDER_ROUTING_KEY = "order.created";

    @Bean
    public Queue orderQueue() {
        return new Queue(ORDER_QUEUE, true); // durable=true - queue รอด survive แม้ RabbitMQ restart
    }

    @Bean
    public DirectExchange orderExchange() {
        return new DirectExchange(ORDER_EXCHANGE);
    }

    @Bean
    public Binding orderBinding(Queue orderQueue, DirectExchange orderExchange) {
        return BindingBuilder.bind(orderQueue).to(orderExchange).with(ORDER_ROUTING_KEY);
        // ผูก orderQueue กับ orderExchange ด้วย routing key "order.created"
    }
}
```

## 5. การส่งข้อความ (Producer) ด้วย `RabbitTemplate`

```java
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.stereotype.Service;
import java.time.Instant;

@Service
public class OrderEventPublisher {
    private final RabbitTemplate rabbitTemplate;

    public OrderEventPublisher(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = rabbitTemplate;
    }

    record OrderCreatedEvent(Long orderId, String customerName, double amount, Instant createdAt) {}

    public void publishOrderCreated(Long orderId, String customerName, double amount) {
        var event = new OrderCreatedEvent(orderId, customerName, amount, Instant.now());

        // Spring AMQP + Jackson (ทบทวน Part 70) แปลง object เป็น JSON อัตโนมัติ
        rabbitTemplate.convertAndSend(
            RabbitConfig.ORDER_EXCHANGE, RabbitConfig.ORDER_ROUTING_KEY, event
        );
        // ส่งแล้วทำงานต่อทันที - ไม่รอ consumer ประมวลผล (asynchronous ทบทวนหัวข้อ 1)
        System.out.println("Published OrderCreatedEvent for order " + orderId);
    }
}
```

```java
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final OrderEventPublisher eventPublisher;

    public OrderService(OrderRepository orderRepository, OrderEventPublisher eventPublisher) {
        this.orderRepository = orderRepository;
        this.eventPublisher = eventPublisher;
    }

    public OrderDto createOrder(CreateOrderRequest request) {
        Order order = new Order(request.customerName(), request.amount(), "PENDING");
        orderRepository.save(order);

        // แจ้ง event แทนการเรียก EmailService ตรง ๆ (ทบทวนปัญหา tight coupling จากหัวข้อ 1)
        eventPublisher.publishOrderCreated(order.getId(), order.getCustomerName(), order.getAmount());

        return new OrderDto(order.getId(), order.getStatus(), order.getAmount());
        // createOrder คืนค่าทันที ไม่ต้องรอ email ส่งเสร็จ - ตอบ response เร็วขึ้นมาก
    }
}
```

## 6. การรับข้อความ (Consumer) ด้วย `@RabbitListener`

```java
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

@Component
public class OrderEmailConsumer {
    private final EmailService emailService;

    public OrderEmailConsumer(EmailService emailService) { this.emailService = emailService; }

    @RabbitListener(queues = RabbitConfig.ORDER_QUEUE) // ฟัง queue นี้ตลอดเวลา
    public void handleOrderCreated(OrderEventPublisher.OrderCreatedEvent event) {
        System.out.println("Received event, sending email for order " + event.orderId());
        emailService.sendOrderConfirmation(event.customerName(), event.orderId());
        // ถ้าเมธอดนี้ throw exception -> RabbitMQ จะจัดการ retry ตาม acknowledgment mode (หัวข้อ 8)
    }
}
```

**สังเกตความสวยงามของ decoupling**: `OrderService` **ไม่รู้จัก**
`OrderEmailConsumer` เลย — ถ้าอนาคตต้องเพิ่ม consumer ใหม่ (เช่น
ส่ง SMS แจ้งเตือน, sync ไป analytics system) แค่เพิ่ม `@RabbitListener`
ใหม่โดย**ไม่ต้องแก้ไข** `OrderService` เลย (Open/Closed Principle ทบทวน
จาก Part 57)

## 7. Exchange Types: Direct, Topic, Fanout

```java
public class ExchangeTypesDemo {
    /*
     * Direct Exchange (ใช้ในหัวข้อ 4): ส่งไปที่ queue ที่ routing key ตรงกันเป๊ะ
     *   routing key "order.created" -> ไปที่ queue ที่ bind ด้วย "order.created" เท่านั้น
     *
     * Topic Exchange: routing key แบบ pattern (* = 1 คำ, # = หลายคำ)
     *   routing key "order.created.vip" ตรงกับ binding pattern "order.created.*"
     *   ตรงกับ binding pattern "order.#" ด้วย (matched หลาย queue พร้อมกันได้)
     *
     * Fanout Exchange: ส่งไปที่ทุก queue ที่ bind อยู่ทั้งหมด (ไม่สนใจ routing key เลย)
     *   เหมาะกับ broadcast event ที่หลาย consumer ต้องรับรู้พร้อมกันทั้งหมด
     *   เช่น "user.logged_in" -> ทั้ง AnalyticsConsumer และ AuditLogConsumer ต้องรับรู้
     */
}
```

```java
@Configuration
public class FanoutExampleConfig {
    @Bean
    public FanoutExchange userEventsExchange() {
        return new FanoutExchange("user.events.fanout");
    }

    @Bean
    public Queue analyticsQueue() { return new Queue("analytics.queue"); }

    @Bean
    public Queue auditQueue() { return new Queue("audit.queue"); }

    @Bean
    public Binding analyticsBinding(Queue analyticsQueue, FanoutExchange userEventsExchange) {
        return BindingBuilder.bind(analyticsQueue).to(userEventsExchange); // ไม่ต้องระบุ routing key
    }

    @Bean
    public Binding auditBinding(Queue auditQueue, FanoutExchange userEventsExchange) {
        return BindingBuilder.bind(auditQueue).to(userEventsExchange);
        // ทั้งสอง queue ได้รับ message เดียวกันทุกครั้งที่ publish ไปที่ exchange นี้
    }
}
```

## 8. Message Acknowledgment และการจัดการข้อผิดพลาด

```java
public class AcknowledgmentModesDemo {
    /*
     * Auto Acknowledgment (default): RabbitMQ ลบ message ออกจาก queue ทันทีที่ส่งให้ consumer
     *   อันตราย: ถ้า consumer crash หลังรับ message แต่ก่อนประมวลผลเสร็จ -> message หายไปเลย!
     *
     * Manual Acknowledgment: consumer ต้อง ACK เองหลังประมวลผลสำเร็จ
     *   ถ้า consumer crash ก่อน ACK -> RabbitMQ ส่ง message นี้ให้ consumer ตัวอื่นใหม่ (redelivery)
     *   ปลอดภัยกว่ามาก - เหมาะกับงานสำคัญที่ห้ามข้อมูลสูญหาย
     */
}
```

```java
import com.rabbitmq.client.Channel;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.amqp.support.AmqpHeaders;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.stereotype.Component;
import java.io.IOException;

@Component
public class ReliableOrderConsumer {
    @RabbitListener(queues = RabbitConfig.ORDER_QUEUE, ackMode = "MANUAL")
    public void handleWithManualAck(OrderEventPublisher.OrderCreatedEvent event,
                                     Channel channel,
                                     @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws IOException {
        try {
            processOrder(event); // งานที่อาจ fail ได้ เช่น เรียก external payment API
            channel.basicAck(tag, false); // สำเร็จ -> ยืนยันว่าประมวลผลเสร็จแล้ว ลบออกจาก queue ได้
        } catch (Exception e) {
            channel.basicNack(tag, false, true); // ล้มเหลว -> ส่ง message กลับเข้า queue ให้ retry
            // requeue=true หมายถึงส่งกลับเข้า queue เดิม (ถ้า retry ซ้ำหลายครั้งไม่สำเร็จ ควรไปที่ DLQ - หัวข้อ 9)
        }
    }

    private void processOrder(OrderEventPublisher.OrderCreatedEvent event) {
        System.out.println("Processing order " + event.orderId());
    }
}
```

## 9. Dead Letter Queue (DLQ)

ถ้า message ล้มเหลวซ้ำ ๆ (retry ไม่สำเร็จ) การให้มันวน requeue ตลอดไปจะ
ทำให้ consumer ทำงานหนักโดยไม่มีประโยชน์ — **Dead Letter Queue** คือ
queue พิเศษที่เก็บ message ที่ล้มเหลวซ้ำเกินกำหนด เพื่อให้ทีมมาตรวจสอบ
ทีหลัง

```java
import org.springframework.amqp.core.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import java.util.Map;

@Configuration
public class DeadLetterQueueConfig {
    @Bean
    public Queue orderQueueWithDlq() {
        return QueueBuilder.durable("order.created.queue")
                .withArgument("x-dead-letter-exchange", "") // ใช้ default exchange
                .withArgument("x-dead-letter-routing-key", "order.created.dlq")
                .build();
        // message ที่ถูก nack/reject ซ้ำเกินจำนวนที่กำหนด จะถูกส่งไปที่ "order.created.dlq" อัตโนมัติ
    }

    @Bean
    public Queue orderDeadLetterQueue() {
        return new Queue("order.created.dlq", true);
        // ทีม dev/ops ตรวจสอบ queue นี้เป็นระยะ เพื่อดูว่ามี message ไหน "ประมวลผลไม่สำเร็จเลย" บ้าง
        // แล้ววิเคราะห์ว่าเป็นบั๊กของระบบ หรือข้อมูลผิดปกติที่ต้องแก้ไขด้วยมือ
    }
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียน `@RabbitListener` ที่รับ `OrderCreatedEvent` แล้วอัปเดต
สินค้าคงคลัง (inventory) — อธิบายว่าทำไมงานนี้เหมาะกับการทำแบบ async
มากกว่า sync

**เฉลย:**
```java
@Component
public class InventoryConsumer {
    private final InventoryService inventoryService;

    public InventoryConsumer(InventoryService inventoryService) {
        this.inventoryService = inventoryService;
    }

    @RabbitListener(queues = RabbitConfig.ORDER_QUEUE)
    public void handleOrderCreated(OrderEventPublisher.OrderCreatedEvent event) {
        inventoryService.decreaseStock(event.orderId());
    }
}
```
เหตุผล: การตัดสต็อกไม่จำเป็นต้องเกิด**ทันที**ในขณะที่ผู้ใช้กำลังรอ response
จากการสร้าง order (ทบทวนหัวข้อ 1) — ทำแบบ async ทำให้ผู้ใช้ได้รับ response
เร็วขึ้น และถ้า `InventoryService` มีปัญหาชั่วคราว (เช่น database ช้า)
ก็ไม่กระทบการสร้าง order เลย (message จะรอใน queue จนกว่า
InventoryService พร้อม)

**2)** อธิบายความแตกต่างระหว่าง Direct Exchange และ Fanout Exchange
พร้อมยกตัวอย่างสถานการณ์ที่เหมาะกับแต่ละแบบ

**เฉลย**: **Direct Exchange** ส่ง message ไปที่ queue ที่มี **routing key
ตรงกันเป๊ะ** เท่านั้น เหมาะกับกรณีที่ต้องการ**ส่งข้อความไปที่ consumer
เฉพาะกลุ่มเดียว**ตามประเภทของ event (เช่น "order.created" ไปที่ email
consumer เท่านั้น) ส่วน **Fanout Exchange** ส่ง message ไปที่**ทุก queue
ที่ bind อยู่ทั้งหมด**โดยไม่สนใจ routing key เลย เหมาะกับกรณีที่ต้องการ
**broadcast event ให้หลาย consumer ที่ไม่เกี่ยวข้องกันรับรู้พร้อมกัน**
เช่น event "user.logged_in" ที่ทั้ง `AnalyticsConsumer` (เก็บสถิติการ
login) และ `AuditLogConsumer` (บันทึก log เพื่อความปลอดภัย) ต้องรับรู้
พร้อมกันทั้งคู่โดยไม่ต้องผูก routing key ให้ตรงกัน

**3)** อธิบายว่า Manual Acknowledgment ป้องกันการสูญหายของ message ได้
อย่างไร เทียบกับ Auto Acknowledgment

**เฉลย**: ใน **Auto Acknowledgment** RabbitMQ จะ**ลบ message ออกจาก
queue ทันทีที่ส่งมอบให้ consumer** — ถ้า consumer เกิด crash **ระหว่าง**
กำลังประมวลผล message นั้น (เช่น ตอนกำลังเรียก database แล้ว process ตาย)
message นั้นจะ**สูญหายไปตลอดกาล** เพราะ RabbitMQ คิดว่าส่งมอบสำเร็จไปแล้ว
ในขณะที่ **Manual Acknowledgment** message จะ**ยังคงอยู่ใน queue**จนกว่า
consumer จะเรียก `channel.basicAck()` **อย่างชัดเจน**หลังประมวลผลสำเร็จ
เท่านั้น (ทบทวนหัวข้อ 8) — ถ้า consumer crash ก่อนเรียก ACK, RabbitMQ จะ
**ตรวจพบว่า connection ขาด**และ**ส่ง message นั้นให้ consumer ตัวอื่น
ใหม่โดยอัตโนมัติ** (redelivery) ทำให้มั่นใจได้ว่างานสำคัญจะถูกประมวลผล
จนสำเร็จเสมอ แม้ consumer ตัวหนึ่งจะล่มไปก็ตาม

### สรุปเนื้อหา Part 88

- Asynchronous Messaging แก้ปัญหา tight coupling ทางเวลาที่เกิดจาก
  synchronous REST call
- Message Queue เป็นตัวกลางระหว่าง Producer/Consumer ให้ decoupling,
  load leveling, และ reliability
- RabbitMQ: Exchange รับข้อความและตัดสินใจส่งไปที่ Queue ไหนตาม Binding
  rule
- `RabbitTemplate` ส่งข้อความ (Producer), `@RabbitListener` รับข้อความ
  (Consumer)
- Exchange มี 3 แบบหลัก: Direct (routing key ตรงเป๊ะ), Topic (pattern
  matching), Fanout (broadcast ทุก queue)
- Manual Acknowledgment ป้องกัน message สูญหายเมื่อ consumer crash
  ระหว่างประมวลผล
- Dead Letter Queue เก็บ message ที่ล้มเหลวซ้ำเกินกำหนดไว้ตรวจสอบทีหลัง
  แทนการ requeue วนไม่จบ

**ต่อไป**: [Part 89 — Messaging: Apache Kafka, Spring Kafka](./part-089-messaging-kafka.md)
