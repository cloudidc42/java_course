# Part 85: RESTful API Design Best Practices, Versioning, Pagination

> ขั้นตอนที่ 841-850 ของหลักสูตร | ระดับ: สูงมาก

## สารบัญ

1. Richardson Maturity Model: REST มีหลายระดับ
2. หลักการตั้งชื่อ Resource URI ที่ดี
3. การใช้ HTTP Method และ Status Code อย่างถูกต้อง
4. API Versioning: 4 แนวทางหลัก
5. Pagination: Offset-based vs Cursor-based
6. Sorting และ Filtering ผ่าน Query Parameter
7. HATEOAS: Hypermedia as the Engine of Application State
8. Idempotency และ Idempotency Key
9. Rate Limiting เบื้องต้น
10. แบบฝึกหัดและสรุป

---

## 1. Richardson Maturity Model: REST มีหลายระดับ

ทบทวนจาก Part 78: REST ไม่ได้มีมาตรฐานตายตัว แต่ **Richardson Maturity
Model** ช่วยจัดระดับความ "RESTful" ของ API:

```
Level 0: ใช้ HTTP เป็นแค่ transport (RPC-style, endpoint เดียวทำทุกอย่าง)
Level 1: มี Resource แยกตาม URI (/orders, /customers)
Level 2: ใช้ HTTP Method/Status Code ถูกต้องตามความหมาย (GET/POST/PUT/DELETE)
Level 3: มี HATEOAS - response มี link บอกว่า "ทำอะไรต่อได้บ้าง"
```

API ส่วนใหญ่ในโลกจริงอยู่ที่ **Level 2** — Part นี้จะสอนวิธีทำ Level 2 ให้ดี
ที่สุด และแนะนำ Level 3 (หัวข้อ 7)

## 2. หลักการตั้งชื่อ Resource URI ที่ดี

```java
public class UriNamingBestPractices {
    /*
     * ดี:   GET  /api/orders           - list ทั้งหมด
     *       GET  /api/orders/42        - order เดียว
     *       POST /api/orders           - สร้างใหม่
     *       GET  /api/orders/42/items  - nested resource (items ของ order 42)
     *
     * ไม่ดี: GET  /api/getOrders        - ใช้ verb ใน URI (ซ้ำซ้อนกับ GET method)
     *       POST /api/orders/create    - ซ้ำซ้อน ("create" ซ้ำกับความหมายของ POST)
     *       GET  /api/order            - ใช้เอกพจน์ (ควรใช้พหูพจน์เสมอเพื่อความสม่ำเสมอ)
     *       GET  /api/Orders           - ใช้ตัวพิมพ์ใหญ่ (URI ควรเป็น lowercase-kebab-case)
     */
}
```

**หลักการ**: **Resource คือ Noun (คำนาม) ไม่ใช่ Verb (คำกริยา)** — การ
กระทำ (action) แสดงผ่าน **HTTP Method** อยู่แล้ว (ทบทวน Part 71, 78)

## 3. การใช้ HTTP Method และ Status Code อย่างถูกต้อง

ทบทวนตาราง HTTP Method จาก Part 71 และเพิ่มรายละเอียดเรื่อง **idempotency**
(หัวข้อ 8):

| Method | ใช้ทำอะไร | Idempotent? | Status Code ที่ถูกต้อง |
|---|---|---|---|
| GET | อ่านข้อมูล | ใช่ | 200 OK, 404 Not Found |
| POST | สร้างใหม่ | ไม่ | 201 Created (พร้อม `Location` header) |
| PUT | แทนที่ทั้ง resource | ใช่ | 200 OK หรือ 204 No Content |
| PATCH | แก้ไขบางฟิลด์ | ไม่เสมอไป | 200 OK |
| DELETE | ลบ | ใช่ | 204 No Content |

```java
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import java.net.URI;

@RestController
@RequestMapping("/api/orders")
public class OrderController {
    private final OrderService orderService;

    public OrderController(OrderService orderService) { this.orderService = orderService; }

    @PostMapping
    public ResponseEntity<OrderDto> create(@RequestBody @jakarta.validation.Valid CreateOrderRequest req) {
        OrderDto created = orderService.createOrder(req);
        // ทบทวน Part 78: ต้องคืน 201 Created พร้อม Location header ชี้ไปที่ resource ใหม่
        return ResponseEntity.created(URI.create("/api/orders/" + created.id())).body(created);
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        orderService.deleteOrder(id);
        return ResponseEntity.noContent().build(); // 204 - ลบสำเร็จ ไม่มี body คืน
    }
}
```

## 4. API Versioning: 4 แนวทางหลัก

เมื่อ API เปลี่ยนแปลงแบบ **breaking change** (เช่น ลบ field, เปลี่ยน
โครงสร้าง response) ต้องมีวิธีให้ client เก่ายังใช้งานได้ต่อไปได้ ระหว่าง
client ใหม่ใช้ version ใหม่:

```java
public class ApiVersioningStrategies {
    /*
     * 1. URI Path Versioning (นิยมที่สุด, ชัดเจนที่สุด):
     *    GET /api/v1/orders
     *    GET /api/v2/orders
     *
     * 2. Query Parameter Versioning:
     *    GET /api/orders?version=2
     *
     * 3. Header Versioning (Custom Header):
     *    GET /api/orders
     *    Header: X-API-Version: 2
     *
     * 4. Media Type Versioning (Content Negotiation):
     *    GET /api/orders
     *    Header: Accept: application/vnd.company.v2+json
     */
}
```

```java
@RestController
@RequestMapping("/api/v1/orders") // Version เก่า - ยังรองรับ client เดิม
public class OrderControllerV1 {
    @GetMapping("/{id}")
    public OrderDtoV1 getOrder(@PathVariable Long id) {
        return new OrderDtoV1(id, "ok"); // response แบบเก่า (โครงสร้างเดิม)
    }
}

@RestController
@RequestMapping("/api/v2/orders") // Version ใหม่ - โครงสร้าง response เปลี่ยนไป (breaking change)
public class OrderControllerV2 {
    @GetMapping("/{id}")
    public OrderDtoV2 getOrder(@PathVariable Long id) {
        return new OrderDtoV2(id, "ok", java.time.Instant.now()); // เพิ่ม field ใหม่
    }
}
```

**คำแนะนำ**: **URI Path Versioning** เป็นที่นิยมที่สุดเพราะ**เข้าใจง่าย
ที่สุด**และ**cache ได้ง่าย** (URL ต่างกัน = cache แยกกันชัดเจน — ทบทวน
HTTP caching จาก Part 71) แม้จะไม่ purist ตามหลัก REST เท่าวิธีอื่นก็ตาม

## 5. Pagination: Offset-based vs Cursor-based

ทบทวนปัญหาจาก Part 79: ถ้า resource มีข้อมูลเป็นล้าน record การคืนค่า
ทั้งหมดในครั้งเดียวจะช้าและกิน memory มหาศาล — ต้องแบ่งหน้า (paginate)

### Offset-based Pagination

```java
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/orders")
public class OrderPaginationController {
    private final OrderRepository orderRepository;

    public OrderPaginationController(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @GetMapping
    public Page<OrderDto> listOrders(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        // GET /api/v1/orders?page=2&size=20 -> ข้าม 40 record แรก แล้วเอา 20 record ถัดไป
        Pageable pageable = PageRequest.of(page, size); // ทบทวน Pageable จาก Part 79
        return orderRepository.findAll(pageable).map(this::toDto);
    }

    private OrderDto toDto(Order order) {
        return new OrderDto(order.getId(), order.getStatus(), order.getAmount());
    }
}
```

**ปัญหาของ Offset-based**: ถ้ามีข้อมูลใหม่ถูก insert เข้ามาระหว่างที่ผู้ใช้
เลื่อนหน้า อาจทำให้ **record บางตัวถูกข้ามไปหรือซ้ำกัน** (เพราะ offset นับ
ตำแหน่งสัมบูรณ์ ไม่ใช่ตำแหน่งที่แท้จริงเทียบกับ record ก่อนหน้า) และ**ยิ่ง
offset สูง ยิ่งช้า** (database ต้อง scan ข้าม record จำนวนมากก่อนถึงที่
ต้องการ)

### Cursor-based Pagination

```java
public class CursorPaginationDemo {
    record CursorPage<T>(java.util.List<T> items, String nextCursor) {}

    // ใช้ "ตำแหน่งสุดท้ายที่เห็น" (เช่น id ล่าสุด) เป็น cursor แทน offset
    CursorPage<OrderDto> listOrdersAfterCursor(Long afterId, int limit) {
        // SQL: SELECT * FROM orders WHERE id > :afterId ORDER BY id LIMIT :limit
        // เร็วกว่า offset-based มาก (ใช้ index บน id ตรง ๆ ไม่ต้อง scan ข้าม)
        // และไม่มีปัญหาข้าม/ซ้ำ record แม้มีข้อมูลใหม่ insert เข้ามาระหว่างทาง
        return new CursorPage<>(java.util.List.of(), null);
    }
}
```

**ใช้เมื่อไหร่**: Offset-based เหมาะกับ UI ที่ต้อง**กระโดดไปหน้าที่ N ได้
โดยตรง** (เช่น "ไปหน้า 5"), Cursor-based เหมาะกับ **infinite scroll** และ
ข้อมูลที่เปลี่ยนแปลงบ่อย (feed ข่าว, timeline) ที่ความแม่นยำสำคัญกว่าการ
กระโดดหน้า

## 6. Sorting และ Filtering ผ่าน Query Parameter

```java
import org.springframework.data.domain.Sort;
import org.springframework.data.jpa.domain.Specification;

@RestController
@RequestMapping("/api/v1/orders")
public class OrderFilterController {
    private final OrderRepository orderRepository;

    public OrderFilterController(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @GetMapping("/search")
    public java.util.List<OrderDto> search(
            @RequestParam(required = false) String status,
            @RequestParam(required = false) Double minAmount,
            @RequestParam(defaultValue = "createdAt,desc") String sort) {
        // GET /api/v1/orders/search?status=PENDING&minAmount=100&sort=amount,asc

        String[] sortParts = sort.split(",");
        Sort sortSpec = Sort.by(sortParts[0])
                .with(sortParts[1].equalsIgnoreCase("desc") ? Sort.Direction.DESC : Sort.Direction.ASC);
        // ทบทวน Specification pattern จาก Part 79 สำหรับ dynamic filtering
        Specification<Order> spec = Specification.where(null);
        if (status != null) spec = spec.and((root, query, cb) -> cb.equal(root.get("status"), status));
        if (minAmount != null) spec = spec.and((root, query, cb) -> cb.ge(root.get("amount"), minAmount));

        return orderRepository.findAll(spec, sortSpec).stream()
                .map(o -> new OrderDto(o.getId(), o.getStatus(), o.getAmount()))
                .toList(); // ทบทวน Stream API จาก Part 41
    }
}
```

## 7. HATEOAS: Hypermedia as the Engine of Application State

**HATEOAS** ทำให้ response มี **link บอกว่า "ทำอะไรต่อได้บ้าง"** — client
ไม่ต้อง hardcode URL เอง เพราะ server บอกมาให้ในตัว response

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-hateoas</artifactId>
</dependency>
```

```java
import org.springframework.hateoas.EntityModel;
import org.springframework.hateoas.server.mvc.WebMvcLinkBuilder;
import org.springframework.web.bind.annotation.*;

import static org.springframework.hateoas.server.mvc.WebMvcLinkBuilder.*;

@RestController
@RequestMapping("/api/v1/orders")
public class OrderHateoasController {
    private final OrderService orderService;

    public OrderHateoasController(OrderService orderService) { this.orderService = orderService; }

    @GetMapping("/{id}")
    public EntityModel<OrderDto> getOrder(@PathVariable Long id) {
        OrderDto order = orderService.getOrderById(id);

        return EntityModel.of(order,
            linkTo(methodOn(OrderHateoasController.class).getOrder(id)).withSelfRel(),
            linkTo(methodOn(OrderHateoasController.class).cancelOrder(id)).withRel("cancel")
        );
        /* Response JSON จะมี:
           {
             "id": 1, "status": "PENDING",
             "_links": {
               "self": { "href": "/api/v1/orders/1" },
               "cancel": { "href": "/api/v1/orders/1/cancel" }
             }
           }
           client เห็น "cancel" link แล้วรู้ทันทีว่ายกเลิก order นี้ได้ - ไม่ต้อง hardcode URL เอง
        */
    }

    @PostMapping("/{id}/cancel")
    public EntityModel<OrderDto> cancelOrder(@PathVariable Long id) {
        return EntityModel.of(orderService.cancelOrder(id));
    }
}
```

## 8. Idempotency และ Idempotency Key

ทบทวนตารางในหัวข้อ 3: **POST ไม่ idempotent** — ถ้า client ยิง POST
`/api/orders` ซ้ำ (เช่น เพราะ network timeout แล้ว retry) จะได้ order ใหม่
ซ้ำสองใบโดยไม่ตั้งใจ! **Idempotency Key** แก้ปัญหานี้:

```java
import org.springframework.web.bind.annotation.*;
import java.util.concurrent.ConcurrentHashMap;

@RestController
@RequestMapping("/api/v1/payments")
public class PaymentController {
    // เก็บ idempotency key ที่เคยประมวลผลแล้ว (จริงควรใช้ Redis ที่มี TTL - ปูทางสู่ Part 87)
    private final ConcurrentHashMap<String, PaymentDto> processedKeys = new ConcurrentHashMap<>();
    private final PaymentService paymentService;

    public PaymentController(PaymentService paymentService) { this.paymentService = paymentService; }

    @PostMapping
    public PaymentDto createPayment(
            @RequestHeader("Idempotency-Key") String idempotencyKey,
            @RequestBody CreatePaymentRequest request) {

        // ถ้าเคยเห็น key นี้แล้ว คืนผลลัพธ์เดิมกลับไป ไม่ประมวลผลซ้ำ (ทบทวน ConcurrentHashMap จาก Part 50)
        return processedKeys.computeIfAbsent(idempotencyKey,
                key -> paymentService.processPayment(request));
        // client ที่ retry ด้วย idempotency key เดิม (เพราะไม่แน่ใจว่า request แรกสำเร็จหรือไม่)
        // จะได้ผลลัพธ์เดิมกลับมาอย่างปลอดภัย ไม่ถูกเก็บเงินซ้ำสองครั้ง
    }
}
```

## 9. Rate Limiting เบื้องต้น

ป้องกัน client ยิง request มากเกินไปจนระบบล่ม (ทบทวนแนวคิด DoS
protection):

```java
import jakarta.servlet.*;
import jakarta.servlet.http.*;
import java.io.IOException;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;

public class SimpleRateLimitFilter extends HttpFilter {
    private final ConcurrentHashMap<String, AtomicInteger> requestCounts = new ConcurrentHashMap<>();
    private static final int MAX_REQUESTS_PER_MINUTE = 60;

    @Override
    protected void doFilter(HttpServletRequest req, HttpServletResponse resp, FilterChain chain)
            throws IOException, ServletException {
        String clientIp = req.getRemoteAddr(); // ทบทวน production ควรใช้ API key แทน IP (Part 82 - JWT sub)

        AtomicInteger count = requestCounts.computeIfAbsent(clientIp, k -> new AtomicInteger(0));
        if (count.incrementAndGet() > MAX_REQUESTS_PER_MINUTE) {
            resp.setStatus(429); // 429 Too Many Requests
            resp.getWriter().write("{\"error\":\"Rate limit exceeded\"}");
            return; // ไม่ chain.doFilter ต่อ - บล็อก request นี้
        }
        chain.doFilter(req, resp);
    }
    // ในระบบจริงควรใช้ sliding window algorithm + Redis (แชร์ข้ามหลาย server instance - ปูทางสู่ Part 87, 91)
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** ออกแบบ URI สำหรับ resource "ความคิดเห็น (comments) ของบทความ
(article) หนึ่ง" ตามหลักการในหัวข้อ 2

**เฉลย:**
```
GET    /api/v1/articles/42/comments        - list comments ของ article 42
POST   /api/v1/articles/42/comments        - เพิ่ม comment ใหม่
GET    /api/v1/articles/42/comments/7      - comment เดียว
DELETE /api/v1/articles/42/comments/7      - ลบ comment
```
ใช้ nested resource เพราะ comment "เป็นของ" article เสมอ (ไม่มี comment
ที่ไม่มี article เป็นเจ้าของ)

**2)** ทำไม PUT ต้อง idempotent แต่ POST ไม่จำเป็นต้อง idempotent

**เฉลย**: **PUT** มีความหมายว่า "**แทนที่ทั้ง resource ด้วยข้อมูลนี้**" — 
ไม่ว่าจะยิง PUT ซ้ำกี่ครั้งด้วย body เดียวกัน ผลลัพธ์สุดท้ายก็เหมือนกัน
(resource ถูกแทนที่ด้วยค่าเดิม) จึง idempotent โดยธรรมชาติ ในขณะที่
**POST** มีความหมายว่า "**สร้าง resource ใหม่**" — ยิง POST ซ้ำสองครั้ง
ด้วย body เดียวกันจะได้ resource ใหม่**สองชิ้นที่แยกกัน** (มี id ต่างกัน)
ซึ่งไม่ idempotent โดยธรรมชาติ (ทบทวนหัวข้อ 8 - นี่คือเหตุผลที่ Idempotency
Key จำเป็นสำหรับ POST)

**3)** อธิบายข้อดีของ Cursor-based Pagination เทียบกับ Offset-based
Pagination ในกรณีของ feed ข่าวที่มีข้อมูลใหม่เข้ามาตลอดเวลา

**เฉลย**: ใน Offset-based Pagination ถ้าผู้ใช้ดูหน้า 1 (record 1-20) แล้วมี
ข่าวใหม่ 5 รายการถูก insert เข้ามาที่ตำแหน่งบนสุดก่อนที่ผู้ใช้จะเลื่อนไปหน้า
2 ระบบจะคำนวณ offset=20 ใหม่โดยอิงจากลำดับที่**เปลี่ยนไปแล้ว** ทำให้ผู้ใช้
**เห็นข่าวซ้ำ 5 รายการที่เคยเห็นในหน้า 1** (เพราะข่าวใหม่ 5 รายการเลื่อน
ทุกอย่างลงมา) ในทางกลับกัน Cursor-based Pagination ใช้ "id ล่าสุดที่เคยเห็น"
เป็นจุดอ้างอิง (`WHERE id < :lastSeenId ORDER BY id DESC`) ซึ่ง**ไม่ขึ้นกับ
ตำแหน่งสัมบูรณ์**เลย แม้จะมีข่าวใหม่ insert เข้ามาก่อนหน้า cursor ผู้ใช้ก็
ยังเห็นข่าวถัดไปที่ถูกต้องโดยไม่ซ้ำหรือขาดหาย

### สรุปเนื้อหา Part 85

- URI ควรตั้งชื่อเป็น noun พหูพจน์ ไม่ใช้ verb ซ้ำซ้อนกับ HTTP Method
- ใช้ HTTP Method/Status Code ให้ตรงความหมาย: POST→201, DELETE→204,
  GET→200/404
- API Versioning มี 4 แนวทาง — URI Path Versioning นิยมที่สุดเพราะเข้าใจ
  ง่ายและ cache ได้ดี
- Offset-based Pagination ใช้งานง่ายแต่ช้าเมื่อ offset สูงและมีปัญหา
  ข้าม/ซ้ำข้อมูล; Cursor-based เร็วกว่าและแม่นยำกว่าสำหรับข้อมูลที่
  เปลี่ยนแปลงบ่อย
- HATEOAS ทำให้ response มี link บอก action ที่ทำได้ต่อ ลด hardcode ฝั่ง
  client
- Idempotency Key ป้องกันการสร้าง resource ซ้ำจาก POST ที่ retry
- Rate Limiting ป้องกันการใช้งาน API เกินขนาดที่กำหนด

**ต่อไป**: [Part 86 — API Documentation with Swagger/OpenAPI](./part-086-api-documentation.md)
