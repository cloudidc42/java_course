# Part 91: Microservices Architecture

> ขั้นตอนที่ 901-910 ของหลักสูตร | ระดับ: ระดับมืออาชีพ/โลก

## สารบัญ

1. Monolith vs Microservices
2. ข้อดีและข้อเสียของ Microservices
3. หลักการแบ่ง Service: Domain-Driven Design เบื้องต้น
4. Database per Service Pattern
5. การสื่อสารระหว่าง Service: Synchronous vs Asynchronous
6. ปัญหา Distributed Transaction: Saga Pattern
7. ปัญหา Distributed Data Query: API Composition และ CQRS
8. Bounded Context และ Anti-Corruption Layer
9. Strangler Fig Pattern: การย้ายจาก Monolith สู่ Microservices
10. แบบฝึกหัดและสรุป

---

## 1. Monolith vs Microservices

ทบทวนจาก Part 73-90: สิ่งที่เราสร้างมาตลอดคือ **Monolith** — แอปพลิเคชัน
เดียวที่รวม Controller, Service, Repository ทั้งหมดไว้ใน**process
เดียวกัน** deploy พร้อมกันเป็นไฟล์ JAR/WAR เดียว

```
Monolith:                          Microservices:
┌─────────────────────┐            ┌──────────┐  ┌──────────┐  ┌──────────┐
│  Order  Customer     │            │  Order   │  │ Customer │  │ Payment  │
│  Payment  Inventory  │            │ Service  │  │ Service  │  │ Service  │
│  (process เดียว)      │            │(process1)│  │(process2)│  │(process3)│
│  (database เดียว)     │            └────┬─────┘  └────┬─────┘  └────┬─────┘
└──────────────────────┘                  │ DB1         │ DB2         │ DB3
                                     (แต่ละ service มี database แยก - หัวข้อ 4)
```

**Microservices Architecture** คือการแบ่งแอปพลิเคชันใหญ่ออกเป็น
**service ย่อยที่ independent** — แต่ละ service มี**process แยกกัน**,
**deploy แยกกันได้**, และมักมี **database แยกกัน**

## 2. ข้อดีและข้อเสียของ Microservices

```java
public class MicroservicesTradeoffs {
    /*
     * ข้อดี:
     *   - แต่ละทีมพัฒนา/deploy service ของตัวเองได้อิสระ (ไม่ต้อง coordinate กับทีมอื่นทุกครั้ง)
     *   - Scale เฉพาะ service ที่ต้องการได้ (เช่น OrderService รับโหลดสูง -> scale แค่ตัวนี้)
     *   - เลือกเทคโนโลยีต่างกันได้ต่อ service (Service A ใช้ Java, Service B ใช้ Go)
     *   - ความล้มเหลวของ service หนึ่งไม่ทำให้ระบบทั้งหมดล่ม (ทบทวน fault isolation)
     *
     * ข้อเสีย:
     *   - ความซับซ้อนด้าน operations สูงขึ้นมาก (ต้อง monitor/deploy หลาย service - ปูทางสู่ Part 96, 99)
     *   - Distributed system เพิ่มปัญหาใหม่: network latency, partial failure, distributed transaction (หัวข้อ 6)
     *   - Testing integration ยากขึ้น (ทบทวน @SpringBootTest จาก Part 84 - ทดสอบ 1 service ไม่พอ)
     *   - ต้องมี infrastructure เพิ่ม (service discovery, API gateway - ปูทางสู่ Part 92, 93)
     */
}
```

**หลักการสำคัญ**: Microservices **ไม่ใช่คำตอบสำหรับทุกระบบ** — ระบบขนาด
เล็กที่ทีมเดียวดูแล Monolith ที่ออกแบบดี (แยก package ชัดเจน ทบทวน Part
20, 57) มักเหมาะสมกว่าและง่ายกว่ามาก Microservices เหมาะกับระบบขนาดใหญ่
ที่มีหลายทีมทำงานพร้อมกัน

## 3. หลักการแบ่ง Service: Domain-Driven Design เบื้องต้น

```java
public class DomainDrivenDesignConcepts {
    /*
     * หลักการแบ่ง Microservices ที่ดีที่สุด: แบ่งตาม "Business Capability" หรือ "Bounded Context"
     * ไม่ใช่แบ่งตาม technical layer (เช่น ห้ามแบ่งเป็น "Controller Service", "Database Service")
     *
     * ตัวอย่างการแบ่งที่ดีสำหรับระบบ e-commerce:
     *   - Order Service: จัดการคำสั่งซื้อทั้งหมด (create, cancel, track status)
     *   - Inventory Service: จัดการสต็อกสินค้า
     *   - Payment Service: จัดการการชำระเงิน
     *   - Notification Service: จัดการการแจ้งเตือน (email, SMS, push)
     *
     * แต่ละ service ควบคุม "domain" ของตัวเองอย่างสมบูรณ์ ไม่มี service อื่นเข้าไปแก้ไข
     * ข้อมูลของ domain นั้นตรง ๆ (ต้องผ่าน API ของ service เจ้าของข้อมูลเท่านั้น)
     */
}
```

## 4. Database per Service Pattern

```java
public class DatabasePerServiceDemo {
    /*
     * กฎเหล็ก: แต่ละ Microservice มี database ของตัวเอง ห้าม service อื่นเข้าถึง database นั้นตรง ๆ
     *
     * ผิด: OrderService และ InventoryService ใช้ database เดียวกัน แล้ว query ข้าม table กันตรง ๆ
     *      -> ทำให้ service ทั้งสอง "coupled กันทาง database schema" เปลี่ยน schema ของฝั่งหนึ่ง
     *         กระทบอีกฝั่งทันที (สูญเสียประโยชน์หลักของ microservices - independent deployment)
     *
     * ถูก: OrderService มี OrderDB ของตัวเอง, InventoryService มี InventoryDB ของตัวเอง
     *      ถ้า OrderService ต้องรู้ข้อมูลสต็อก ต้องเรียกผ่าน InventoryService's API เท่านั้น
     *      (REST call ทบทวน Part 78 หรือผ่าน message queue ทบทวน Part 88)
     */
}
```

```java
// OrderService (มี OrderDB ของตัวเอง)
@Service
public class OrderService {
    private final OrderRepository orderRepository; // เชื่อมต่อ OrderDB เท่านั้น
    private final InventoryServiceClient inventoryClient; // เรียกผ่าน HTTP client ไม่ query database ตรง

    public OrderService(OrderRepository orderRepository, InventoryServiceClient inventoryClient) {
        this.orderRepository = orderRepository;
        this.inventoryClient = inventoryClient;
    }

    public OrderDto createOrder(CreateOrderRequest request) {
        boolean inStock = inventoryClient.checkStock(request.productId(), request.quantity());
        // เรียกผ่าน network (REST/gRPC) ไปที่ InventoryService - ไม่ query InventoryDB ตรง
        if (!inStock) throw new InsufficientStockException(request.productId());

        Order order = new Order(request.customerId(), request.productId(), "PENDING");
        orderRepository.save(order);
        return new OrderDto(order.getId(), order.getStatus());
    }
}
```

## 5. การสื่อสารระหว่าง Service: Synchronous vs Asynchronous

```java
public class InterServiceCommunicationDemo {
    /*
     * Synchronous (REST/gRPC ทบทวน Part 78):
     *   OrderService เรียก InventoryService ตรง ๆ แล้ว "รอ" คำตอบ
     *   ข้อดี: เข้าใจง่าย, ได้ผลลัพธ์ทันที
     *   ข้อเสีย: ถ้า InventoryService ช้าหรือ down, OrderService ก็ได้รับผลกระทบไปด้วย (cascading failure)
     *
     * Asynchronous (Message Queue/Event ทบทวน Part 88, 89):
     *   OrderService publish event "OrderCreated" แล้วทำงานต่อทันที
     *   InventoryService subscribe event นี้แล้วอัปเดตสต็อกเมื่อพร้อม
     *   ข้อดี: decoupling สูง, ทนต่อความล้มเหลวของ service อื่นได้ดีกว่า
     *   ข้อเสีย: eventual consistency (ข้อมูลอาจไม่ sync กันทันที - หัวข้อ 6)
     */
}
```

**คำแนะนำ**: ใช้ **synchronous** สำหรับ query ที่ต้องการผลลัพธ์ทันที (เช่น
"เช็คสต็อกก่อนแสดงหน้าสั่งซื้อ") และใช้ **asynchronous** สำหรับ
**side-effect** ที่ไม่ต้องรอผลทันที (เช่น "แจ้งเตือน", "อัปเดตสถิติ")

## 6. ปัญหา Distributed Transaction: Saga Pattern

ทบทวนปัญหาจาก Part 65: `@Transactional` การันตี ACID **ภายใน database
เดียว** — แต่ใน microservices การสร้าง order อาจต้องแก้ไขข้อมูลใน**หลาย
database** (OrderDB, InventoryDB, PaymentDB) พร้อมกัน ซึ่ง**ไม่มี
transaction ข้าม database ได้แบบ ACID ปกติ**

```java
public class SagaPatternDemo {
    /*
     * Saga Pattern: แบ่ง distributed transaction เป็นชุดของ "local transaction" ที่มี
     * "compensating transaction" (ธุรกรรมย้อนกลับ) ถ้าขั้นตอนใดขั้นตอนหนึ่งล้มเหลว
     *
     * ตัวอย่าง Order Saga:
     *   1. OrderService: สร้าง order สถานะ PENDING (local transaction)
     *   2. InventoryService: จองสต็อก (local transaction)
     *      ถ้าล้มเหลว -> compensate: OrderService ยกเลิก order (step 1 ย้อนกลับ)
     *   3. PaymentService: เก็บเงิน (local transaction)
     *      ถ้าล้มเหลว -> compensate: InventoryService ปลดจองสต็อก (step 2 ย้อนกลับ)
     *                              และ OrderService ยกเลิก order (step 1 ย้อนกลับ)
     *   4. OrderService: อัปเดตสถานะเป็น COMPLETED
     */
}
```

```java
@Service
public class OrderSagaOrchestrator {
    private final OrderServiceClient orderClient;
    private final InventoryServiceClient inventoryClient;
    private final PaymentServiceClient paymentClient;

    public OrderSagaOrchestrator(OrderServiceClient orderClient, InventoryServiceClient inventoryClient,
                                  PaymentServiceClient paymentClient) {
        this.orderClient = orderClient;
        this.inventoryClient = inventoryClient;
        this.paymentClient = paymentClient;
    }

    public void executeOrderSaga(CreateOrderRequest request) {
        Long orderId = orderClient.createPendingOrder(request);

        try {
            inventoryClient.reserveStock(request.productId(), request.quantity());
            try {
                paymentClient.chargeCustomer(request.customerId(), request.amount());
                orderClient.markCompleted(orderId); // ทุกขั้นตอนสำเร็จ
            } catch (Exception paymentFailed) {
                inventoryClient.releaseStock(request.productId(), request.quantity()); // compensate step 2
                orderClient.markCancelled(orderId); // compensate step 1
                throw paymentFailed;
            }
        } catch (Exception inventoryFailed) {
            orderClient.markCancelled(orderId); // compensate step 1
            throw inventoryFailed;
        }
        // นี่คือ "Orchestration-based Saga" - มี orchestrator ตัวกลางคุมทุกขั้นตอน
        // อีกแบบคือ "Choreography-based Saga" - แต่ละ service subscribe event กันเองไม่มีตัวกลาง (ทบทวน Part 88, 89)
    }
}
```

## 7. ปัญหา Distributed Data Query: API Composition และ CQRS

```java
public class ApiCompositionDemo {
    /*
     * ปัญหา: หน้า "รายละเอียด order" ต้องแสดงข้อมูลจาก 3 service:
     *   OrderService (สถานะ order) + CustomerService (ชื่อลูกค้า) + PaymentService (สถานะการเงิน)
     * ไม่มี database เดียวให้ JOIN ข้าม 3 ตารางนี้ได้เหมือน Monolith (ทบทวน SQL JOIN จาก Part 65)
     *
     * API Composition: สร้าง service กลาง (หรือ API Gateway - ปูทางสู่ Part 93) ที่เรียก 3 service
     * พร้อมกัน (parallel - ทบทวน CompletableFuture จาก Part 49) แล้วรวมผลลัพธ์เอง
     */
}
```

```java
import java.util.concurrent.CompletableFuture;

@Service
public class OrderDetailCompositionService {
    private final OrderServiceClient orderClient;
    private final CustomerServiceClient customerClient;
    private final PaymentServiceClient paymentClient;

    public OrderDetailCompositionService(OrderServiceClient orderClient,
            CustomerServiceClient customerClient, PaymentServiceClient paymentClient) {
        this.orderClient = orderClient;
        this.customerClient = customerClient;
        this.paymentClient = paymentClient;
    }

    record OrderDetailView(Long orderId, String status, String customerName, String paymentStatus) {}

    public OrderDetailView getOrderDetail(Long orderId) {
        // เรียกทั้ง 3 service พร้อมกัน (ทบทวน CompletableFuture.allOf จาก Part 49) ลด latency รวม
        CompletableFuture<OrderView> orderFuture = CompletableFuture.supplyAsync(() -> orderClient.get(orderId));
        CompletableFuture<CustomerView> customerFuture = CompletableFuture.supplyAsync(
                () -> customerClient.getByOrderId(orderId));
        CompletableFuture<PaymentView> paymentFuture = CompletableFuture.supplyAsync(
                () -> paymentClient.getByOrderId(orderId));

        CompletableFuture.allOf(orderFuture, customerFuture, paymentFuture).join();

        return new OrderDetailView(orderId, orderFuture.join().status(),
                customerFuture.join().name(), paymentFuture.join().status());
    }
    record OrderView(String status) {}
    record CustomerView(String name) {}
    record PaymentView(String status) {}
}
```

**CQRS (Command Query Responsibility Segregation)**: แนวทางที่ซับซ้อนกว่า
คือให้แต่ละ service **ส่ง event ไปสร้าง "read model" รวม**ไว้ล่วงหน้า
(denormalized view) ใน database แยก ทำให้ query หน้ารายละเอียดแบบนี้ทำได้
เร็วในครั้งเดียวโดยไม่ต้องเรียกหลาย service ทุกครั้ง (แลกกับความซับซ้อนใน
การ sync ข้อมูล — เหมาะกับระบบที่ query บ่อยกว่าการเขียนมาก)

## 8. Bounded Context และ Anti-Corruption Layer

```java
public class BoundedContextDemo {
    /*
     * Bounded Context: แต่ละ service มี "โมเดลข้อมูลของตัวเอง" ที่อาจต่างจาก service อื่น
     * แม้จะพูดถึง "concept" เดียวกันในชื่อก็ตาม
     *
     * ตัวอย่าง: "Customer" ใน OrderService สนใจแค่ {customerId, name, shippingAddress}
     *          "Customer" ใน MarketingService สนใจ {customerId, email, preferences, segments}
     * ทั้งสองไม่ต้องมี model เดียวกันทุกฟิลด์ - แต่ละ service เก็บแค่ข้อมูลที่ตัวเองต้องใช้
     *
     * Anti-Corruption Layer: เมื่อ service ต้องคุยกับระบบภายนอกที่มี model ต่างกันมาก
     * (เช่น legacy system หรือ third-party API) ควรมี "adapter layer" แปลงข้อมูลระหว่างกัน
     * เพื่อไม่ให้โมเดลที่ไม่ดีของระบบภายนอก "รั่วไหล" เข้ามาปนกับโมเดลภายในของเราเอง
     */
}
```

## 9. Strangler Fig Pattern: การย้ายจาก Monolith สู่ Microservices

```java
public class StranglerFigPatternDemo {
    /*
     * การย้าย Monolith ขนาดใหญ่ไปเป็น Microservices ทั้งหมดในครั้งเดียวเสี่ยงสูงมาก (big-bang rewrite)
     * Strangler Fig Pattern: ย้ายทีละส่วน โดยให้ Monolith และ Microservices ใหม่ทำงานคู่กันไปก่อน
     *
     * ขั้นตอน:
     *   1. เลือก feature หนึ่งจาก Monolith (เช่น "Notification") ที่ตัดออกมาได้ง่ายที่สุด
     *   2. สร้าง NotificationService ใหม่แยกออกมา
     *   3. ใช้ API Gateway (ปูทางสู่ Part 93) route request ที่เกี่ยวกับ notification ไปที่ service ใหม่
     *      route ส่วนอื่นที่ยังไม่ย้ายไปที่ Monolith เดิมตามปกติ
     *   4. ทำซ้ำกับ feature อื่น ๆ ทีละตัว จนกว่า Monolith จะเหลือน้อยลงเรื่อย ๆ (ถูก "รัด" จนหายไป)
     *
     * ข้อดี: ความเสี่ยงต่ำกว่า big-bang rewrite มาก, ทดสอบและ rollback แต่ละ feature ได้อิสระ
     */
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** ระบบ e-commerce มี Monolith เดียวจัดการทั้ง Order, Customer,
Product ถ้าจะแบ่งเป็น Microservices ตามหลัก Domain-Driven Design ควรแบ่ง
อย่างไร

**เฉลย**: แบ่งตาม **business capability** ไม่ใช่ technical layer —
`OrderService` (จัดการวงจรชีวิตของคำสั่งซื้อ), `CustomerService` (จัดการ
ข้อมูลลูกค้าและ profile), `ProductService` (จัดการ catalog สินค้าและ
ราคา) แต่ละ service ควบคุม database และ business logic ของ domain
ตัวเองอย่างสมบูรณ์ (ทบทวนหัวข้อ 3, 4) — ไม่แบ่งแบบ "ControllerService",
"DatabaseService" เพราะนั่นคือการแบ่งตาม technical layer ซึ่งขัดกับหลัก
DDD และทำให้แต่ละ service ยัง coupled กันสูงอยู่

**2)** อธิบายว่าทำไม Database per Service Pattern สำคัญต่อความเป็น
independent ของ microservices

**เฉลย**: ถ้าหลาย service **ใช้ database เดียวกัน**และ query ข้าม table
กันตรง ๆ การเปลี่ยน schema ของ table หนึ่ง (เช่น เปลี่ยนชื่อ column,
เพิ่ม constraint) จะ**กระทบ service อื่นที่ query table นั้นทันที**
ทำให้ทีมที่ดูแล service หนึ่งไม่สามารถ deploy การเปลี่ยนแปลงได้อย่าง
อิสระ (ต้อง coordinate กับทุกทีมที่ใช้ database เดียวกัน) ซึ่งขัดกับ
เป้าหมายหลักของ microservices ที่ต้องการให้ **แต่ละทีม deploy service
ของตัวเองได้โดยไม่ต้องรอทีมอื่น** (ทบทวนหัวข้อ 2) การให้แต่ละ service มี
database ของตัวเองทำให้**การเปลี่ยนแปลง schema ภายในเป็นเรื่องภายในของ
service นั้นเท่านั้น** ตราบใดที่ API ที่เปิดให้ service อื่นเรียกยังคงมี
contract เดิม

**3)** อธิบาย Saga Pattern และเหตุผลที่จำเป็นในระบบ microservices ที่ไม่
สามารถใช้ `@Transactional` แบบ ACID ข้าม database ได้

**เฉลย**: ใน Monolith เดียวที่มี database เดียว `@Transactional`
(ทบทวน Part 65) สามารถการันตีว่า**ทุกการเปลี่ยนแปลงสำเร็จพร้อมกันหรือ
ล้มเหลวพร้อมกันทั้งหมด (atomicity)** แต่ใน microservices ที่แต่ละ
service มี database แยกกัน (หัวข้อ 4) **ไม่มีกลไก transaction ที่ครอบคลุม
หลาย database ได้แบบ ACID ปกติ** เพราะแต่ละ database เป็นระบบอิสระที่ไม่
รู้จักกัน **Saga Pattern** แก้ปัญหานี้โดยแบ่งงานเป็น**ชุดของ local
transaction ที่เกิดขึ้นทีละขั้นตอนในแต่ละ service** และถ้าขั้นตอนใด
ขั้นตอนหนึ่งล้มเหลว จะเรียก **compensating transaction** (ธุรกรรมย้อนกลับ)
เพื่อ**ยกเลิกผลของขั้นตอนก่อนหน้าที่สำเร็จไปแล้ว** ทำให้ระบบกลับสู่สถานะ
ที่สมเหตุสมผล แม้จะไม่ใช่ atomicity แบบ ACID เป๊ะ ๆ ก็ตาม (เรียกว่า
eventual consistency)

### สรุปเนื้อหา Part 91

- Microservices แบ่งแอปพลิเคชันเป็น service อิสระที่ deploy/scale แยกกัน
  ได้ แลกกับความซับซ้อนด้าน distributed system ที่เพิ่มขึ้นมาก
- แบ่ง service ตาม Domain-Driven Design (business capability) ไม่ใช่
  technical layer
- Database per Service Pattern ทำให้แต่ละ service independent อย่างแท้จริง
- การสื่อสารระหว่าง service มีทั้ง synchronous (REST) และ asynchronous
  (message queue) — เลือกให้เหมาะกับสถานการณ์
- Saga Pattern แก้ปัญหา distributed transaction ด้วย local transaction +
  compensating transaction
- API Composition/CQRS แก้ปัญหา query ข้อมูลที่กระจายอยู่หลาย service
- Bounded Context ทำให้แต่ละ service มีโมเดลข้อมูลของตัวเองได้โดยไม่ต้อง
  เหมือนกันทุก service
- Strangler Fig Pattern ช่วยย้ายจาก Monolith สู่ Microservices แบบทีละส่วน
  ลดความเสี่ยง

**ต่อไป**: [Part 92 — Spring Cloud: Service Discovery ด้วย Eureka](./part-092-service-discovery-eureka.md)
