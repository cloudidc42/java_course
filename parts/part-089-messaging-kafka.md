# Part 89: Messaging: Apache Kafka, Spring Kafka

> ขั้นตอนที่ 881-890 ของหลักสูตร | ระดับ: สูงมาก

## สารบัญ

1. Kafka ต่างจาก RabbitMQ อย่างไร
2. Kafka Concepts: Topic, Partition, Offset
3. Consumer Group: การกระจายงานระหว่าง Consumer
4. ติดตั้ง Spring Kafka และตั้งค่า Producer/Consumer
5. การส่งข้อความด้วย `KafkaTemplate`
6. การรับข้อความด้วย `@KafkaListener`
7. Partition Key: การันตีลำดับข้อความ
8. Kafka Retention: เก็บข้อความได้นานกว่า Queue ทั่วไป
9. เมื่อไหร่ควรใช้ Kafka เมื่อไหร่ควรใช้ RabbitMQ
10. แบบฝึกหัดและสรุป

---

## 1. Kafka ต่างจาก RabbitMQ อย่างไร

ทบทวนจาก Part 88: RabbitMQ เป็น **message broker** แบบ "ส่งแล้วลบ" (เมื่อ
consumer ACK แล้ว message จะหายจาก queue) — **Apache Kafka** ออกแบบมา
เพื่อ **event streaming** ที่มีปริมาณข้อมูลสูงมาก (high-throughput) และ
ต้องการ**เก็บ log ของทุก event ไว้ได้นาน**

```java
public class KafkaVsRabbitMQDemo {
    /*
     * RabbitMQ:
     *   - message ถูกลบทันทีที่ consumer ACK แล้ว
     *   - เหมาะกับ task queue (งานที่ต้องทำครั้งเดียวแล้วจบ เช่น ส่ง email)
     *   - routing ซับซ้อนได้ (Direct/Topic/Fanout Exchange - ทบทวน Part 88)
     *
     * Kafka:
     *   - message (เรียกว่า "record") ถูกเก็บไว้เป็น log แม้ consumer อ่านไปแล้ว (จนกว่าจะครบ retention period)
     *   - Consumer หลายตัว/หลายกลุ่มอ่าน log เดียวกันซ้ำได้ในเวลาต่างกัน (replay ได้!)
     *   - throughput สูงมาก (ออกแบบมาสำหรับ event stream ปริมาณมหาศาล เช่น log จาก IoT, clickstream)
     *   - เหมาะกับ Event Sourcing, real-time analytics, log aggregation
     */
}
```

## 2. Kafka Concepts: Topic, Partition, Offset

```
Topic: "order-events"
┌─────────────────────────────────────────────────┐
│ Partition 0: [msg0][msg1][msg2][msg3] ...        │  offset 0,1,2,3...
│ Partition 1: [msg0][msg1][msg2] ...              │  offset 0,1,2...
│ Partition 2: [msg0][msg1][msg2][msg3][msg4] ...  │  offset 0,1,2,3,4...
└─────────────────────────────────────────────────┘
```

- **Topic**: ชื่อ "ช่อง" ที่ event ถูกส่งเข้ามา (คล้าย Queue แต่เก็บ log
  ทั้งหมด ไม่ลบทิ้ง)
- **Partition**: Topic หนึ่งแบ่งเป็นหลาย partition เพื่อ**กระจายโหลด**
  (parallel processing) และเพิ่ม throughput
- **Offset**: ตำแหน่งของ record ใน partition — Consumer จดจำ offset ที่
  อ่านล่าสุด ทำให้อ่านต่อจากจุดที่หยุดไว้ได้ (ทบทวนแนวคิด cursor จาก
  Part 85)

## 3. Consumer Group: การกระจายงานระหว่าง Consumer

```java
public class ConsumerGroupDemo {
    /*
     * Topic "order-events" มี 3 partition
     * Consumer Group "email-service-group" มี 3 consumer instance
     *
     * Kafka จะ "แบ่ง" partition ให้แต่ละ consumer ในกลุ่มเดียวกันแบบอัตโนมัติ:
     *   Consumer A -> อ่าน Partition 0
     *   Consumer B -> อ่าน Partition 1
     *   Consumer C -> อ่าน Partition 2
     *
     * ถ้ามี Consumer Group อื่น "analytics-group" อ่าน topic เดียวกัน:
     *   จะได้รับ record ทั้งหมดแยกอิสระจาก "email-service-group"
     *   (แต่ละ consumer group มี offset ของตัวเอง - อ่าน log ซ้ำกันได้โดยไม่ชนกัน)
     */
}
```

**นี่คือความแตกต่างสำคัญจาก RabbitMQ**: ใน RabbitMQ ถ้ามี consumer สอง
กลุ่มอยากอ่าน message เดียวกัน ต้องสร้าง queue แยกกัน (fanout - ทบทวน
Part 88) แต่ Kafka **consumer group แต่ละกลุ่มอ่าน topic เดียวกันได้เลย
โดยธรรมชาติ** เพราะ record ไม่ถูกลบทิ้งหลังอ่าน

## 4. ติดตั้ง Spring Kafka และตั้งค่า Producer/Consumer

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

```properties
# application.properties
spring.kafka.bootstrap-servers=localhost:9092
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer=org.springframework.kafka.support.serializer.JsonSerializer
spring.kafka.consumer.group-id=email-service-group
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.springframework.kafka.support.serializer.JsonDeserializer
spring.kafka.consumer.properties.spring.json.trusted.packages=*
```

## 5. การส่งข้อความด้วย `KafkaTemplate`

```java
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import java.time.Instant;

@Service
public class OrderEventProducer {
    private static final String TOPIC = "order-events";
    private final KafkaTemplate<String, OrderCreatedEvent> kafkaTemplate;

    public OrderEventProducer(KafkaTemplate<String, OrderCreatedEvent> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    record OrderCreatedEvent(Long orderId, String customerId, double amount, Instant createdAt) {}

    public void publish(Long orderId, String customerId, double amount) {
        var event = new OrderCreatedEvent(orderId, customerId, amount, Instant.now());

        // key = customerId -> การันตีว่า event ของ customer เดียวกันไปที่ partition เดียวกันเสมอ (หัวข้อ 7)
        kafkaTemplate.send(TOPIC, customerId, event);
        System.out.println("Published event for order " + orderId + " to Kafka");
    }
}
```

## 6. การรับข้อความด้วย `@KafkaListener`

```java
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

@Component
public class OrderEventEmailConsumer {
    private final EmailService emailService;

    public OrderEventEmailConsumer(EmailService emailService) { this.emailService = emailService; }

    @KafkaListener(topics = "order-events", groupId = "email-service-group")
    public void handleOrderCreated(OrderEventProducer.OrderCreatedEvent event) {
        System.out.println("Consumer group 'email-service-group' processing order " + event.orderId());
        emailService.sendOrderConfirmation(event.customerId(), event.orderId());
    }
}

@Component
public class OrderEventAnalyticsConsumer {
    @KafkaListener(topics = "order-events", groupId = "analytics-group") // consumer group คนละกลุ่ม
    public void trackOrderMetrics(OrderEventProducer.OrderCreatedEvent event) {
        System.out.println("Consumer group 'analytics-group' recording metrics for order " + event.orderId());
        // อ่าน topic เดียวกันกับ email consumer โดยอิสระ ไม่แย่ง record กัน (ทบทวนหัวข้อ 3)
    }
}
```

## 7. Partition Key: การันตีลำดับข้อความ

```java
public class PartitionKeyOrderingDemo {
    /*
     * ปัญหา: ถ้าไม่ระบุ key, Kafka กระจาย record แบบ round-robin ไปทุก partition
     *        -> event ของ customer เดียวกันอาจไปคนละ partition
     *        -> consumer อ่าน partition ต่างกันแบบ parallel -> ลำดับ event อาจสลับกัน!
     *        เช่น "OrderCreated" อาจถูกประมวลผลหลัง "OrderCancelled" ของ order เดียวกัน (ผิดลำดับ!)
     *
     * วิธีแก้: ใช้ key เดียวกันสำหรับ event ที่ "ต้องเรียงลำดับกัน" เสมอ (หัวข้อ 5 - customerId)
     *   Kafka การันตีว่า record ที่มี key เดียวกัน จะไปที่ partition เดียวกันเสมอ (hash(key) % numPartitions)
     *   และภายใน partition เดียวกัน record จะถูกอ่านตามลำดับที่เขียนเข้าไปเสมอ (FIFO ต่อ partition)
     */
}
```

**หลักการสำคัญ**: **Kafka การันตีลำดับ (ordering) เฉพาะภายใน partition
เดียวกันเท่านั้น** — ถ้าต้องการให้ event ของ entity เดียวกันเรียงลำดับ
ถูกต้องเสมอ **ต้องใช้ key ที่สื่อถึง entity นั้น** (เช่น orderId,
customerId) เป็น partition key

## 8. Kafka Retention: เก็บข้อความได้นานกว่า Queue ทั่วไป

```properties
# ตั้งค่าที่ broker (server.properties) หรือระดับ topic
log.retention.hours=168
```

```java
public class KafkaRetentionDemo {
    /*
     * RabbitMQ: message หายทันทีที่ consumer ACK (ทบทวน Part 88)
     * Kafka:    record เก็บไว้ตาม retention policy (เช่น 7 วัน) ไม่ว่า consumer จะอ่านไปแล้วหรือไม่
     *
     * ประโยชน์: ถ้ามี consumer ใหม่เพิ่มเข้ามาทีหลัง (เช่น สร้าง fraud-detection-group ขึ้นใหม่)
     *          สามารถ "replay" อ่าน event ย้อนหลังทั้งหมดภายใน retention period ได้
     *          (ตั้ง offset ของ consumer group ใหม่ให้กลับไปที่ offset 0 หรือ timestamp ที่ต้องการ)
     *          เหมาะมากสำหรับ Event Sourcing และการ debug/analytics ย้อนหลัง
     */
}
```

## 9. เมื่อไหร่ควรใช้ Kafka เมื่อไหร่ควรใช้ RabbitMQ

| ปัจจัย | RabbitMQ | Kafka |
|---|---|---|
| Throughput | ปานกลาง-สูง | สูงมาก (event stream ขนาดใหญ่) |
| Routing ซับซ้อน | ดีมาก (Direct/Topic/Fanout) | จำกัดกว่า (partition-based) |
| Message Replay | ทำไม่ได้ (ลบหลัง ACK) | ทำได้ (retention log) |
| Use Case ทั่วไป | Task Queue, RPC, Work Distribution | Event Streaming, Log Aggregation, Event Sourcing |
| ความซับซ้อนในการดูแล | ต่ำกว่า | สูงกว่า (ต้องจัดการ partition, consumer group) |

**คำแนะนำ**: ถ้าต้องการ **"งานที่ต้องทำครั้งเดียวแล้วจบ"** (ส่ง email,
สร้าง PDF) → **RabbitMQ** เหมาะกว่า (เรียบง่าย). ถ้าต้องการ **"เก็บ log
ของทุกอย่างที่เกิดขึ้นในระบบ"** เพื่อให้หลายทีม/หลายระบบนำไปใช้วิเคราะห์
ทั้ง real-time และย้อนหลัง (เช่น ระบบ microservices ขนาดใหญ่ที่ทำ Event
Sourcing — ปูทางสู่ Part 91) → **Kafka** เหมาะกว่า

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียน `KafkaTemplate.send()` สำหรับ event "PaymentProcessed" โดยใช้
`orderId` เป็น partition key เพื่อการันตีลำดับ

**เฉลย:**
```java
record PaymentProcessedEvent(Long orderId, String status, double amount) {}

public void publishPaymentProcessed(Long orderId, String status, double amount) {
    var event = new PaymentProcessedEvent(orderId, status, amount);
    kafkaTemplate.send("payment-events", String.valueOf(orderId), event);
    // orderId เป็น key -> event ของ order เดียวกันไปที่ partition เดียวกันเสมอ รักษาลำดับได้
}
```

**2)** อธิบายว่าทำไม Consumer Group สองกลุ่มที่อ่าน Topic เดียวกันไม่แย่ง
record กัน ในขณะที่ Consumer สองตัวใน**กลุ่มเดียวกัน**จะแบ่ง partition
กันอ่าน

**เฉลย**: Kafka เก็บ **offset (ตำแหน่งที่อ่านล่าสุด) แยกตาม Consumer
Group** — แต่ละ consumer group มี "ตัวชี้" ของตัวเองที่ชี้ไปยังตำแหน่งใน
log ที่ตัวเองอ่านถึงแล้ว (ทบทวนหัวข้อ 2, 3) ทำให้ consumer group A
("email-service-group") และ consumer group B ("analytics-group") **อ่าน
record เดียวกันได้ทั้งคู่แบบอิสระ** โดยไม่แย่งกัน เพราะแต่ละกลุ่มมี offset
ของตัวเอง แต่ภายใน**กลุ่มเดียวกัน** Kafka ออกแบบมาเพื่อ**กระจายงาน** —
partition หนึ่งจะถูกอ่านโดย consumer **เพียงตัวเดียว**ในกลุ่มนั้นเท่านั้น
(เพื่อป้องกันการประมวลผล record ซ้ำซ้อนภายในกลุ่มเดียวกัน) ทำให้ต้อง
"แบ่ง" partition ให้แต่ละ consumer ในกลุ่มรับผิดชอบคนละส่วน

**3)** ทีมหนึ่งต้องการระบบที่เก็บ log การทำธุรกรรม (transaction log) ไว้
90 วัน เพื่อให้ทีม fraud-detection ที่เพิ่งตั้งขึ้นใหม่สามารถย้อนไปวิเคราะห์
ธุรกรรมเก่า ๆ ได้ ควรเลือก RabbitMQ หรือ Kafka และทำไม

**เฉลย**: ควรเลือก **Kafka** เพราะโจทย์ต้องการ**เก็บ log ธุรกรรมไว้ได้นาน
(90 วัน) แม้จะถูกอ่านไปแล้วก็ตาม** เพื่อให้ระบบใหม่ (fraud-detection) ที่
**ยังไม่มีอยู่ตอนที่ธุรกรรมเกิดขึ้น**สามารถกลับไป "replay" อ่านข้อมูล
ย้อนหลังทั้งหมดได้ (ทบทวนหัวข้อ 8) — คุณสมบัตินี้ RabbitMQ **ทำไม่ได้เลย**
เพราะ message จะถูกลบทิ้งทันทีหลัง consumer ACK สำเร็จ (ทบทวน Part 88)
ดังนั้นถ้า fraud-detection-group ถูกสร้างขึ้นหลังจากธุรกรรมเกิดขึ้นแล้ว
มันจะไม่มีทางเห็นข้อมูลเก่าเลยถ้าใช้ RabbitMQ แต่ด้วย Kafka สามารถตั้ง
consumer group ใหม่ให้เริ่มอ่านจาก offset เริ่มต้น (earliest) เพื่อดึง
ข้อมูลย้อนหลังทั้งหมดภายใน retention period ได้ทันที

### สรุปเนื้อหา Part 89

- Kafka ออกแบบมาเพื่อ event streaming ที่มี throughput สูงและต้องการเก็บ
  log ไว้ได้นาน ต่างจาก RabbitMQ ที่ลบ message หลัง ACK
- Topic แบ่งเป็นหลาย Partition เพื่อกระจายโหลดและเพิ่ม throughput
- Consumer Group แต่ละกลุ่มมี offset อิสระ อ่าน topic เดียวกันได้โดยไม่
  แย่งกัน แต่ consumer ภายในกลุ่มเดียวกันแบ่ง partition กันอ่าน
- `KafkaTemplate` (Producer) และ `@KafkaListener` (Consumer) เป็น API
  หลักของ Spring Kafka
- Partition Key การันตีว่า record ของ entity เดียวกันไปที่ partition
  เดียวกันเสมอ รักษาลำดับได้
- Kafka Retention ทำให้ replay ข้อมูลย้อนหลังได้ เหมาะกับ Event Sourcing
  และ analytics
- เลือก RabbitMQ สำหรับ task queue ทั่วไป, เลือก Kafka สำหรับ event
  streaming ปริมาณสูงที่ต้องการเก็บ log ไว้นาน

**ต่อไป**: [Part 90 — WebSocket, Real-time Applications](./part-090-websocket-realtime.md)
