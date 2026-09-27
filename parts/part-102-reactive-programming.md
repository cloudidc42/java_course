# Part 102: Advanced Concurrency และ Reactive Programming

> ขั้นตอนที่ 1011-1020 ของหลักสูตร | ระดับ: ระดับมืออาชีพ/โลก

## สารบัญ

1. ปัญหาของ Thread-per-Request Model แบบดั้งเดิม
2. Reactive Programming คืออะไร: แนวคิดหลัก
3. Reactive Streams Specification
4. Project Reactor: `Mono` และ `Flux`
5. Operators พื้นฐาน: map, flatMap, filter
6. Backpressure: การจัดการเมื่อ Producer เร็วกว่า Consumer
7. Spring WebFlux: Reactive Web Framework
8. WebClient: เรียก HTTP แบบ Non-blocking
9. Virtual Threads (Project Loom): ทางเลือกใหม่ใน Java 21
10. แบบฝึกหัดและสรุป

---

## 1. ปัญหาของ Thread-per-Request Model แบบดั้งเดิม

ทบทวนจาก Part 46-48: Spring MVC (Part 77) ใช้โมเดล **"1 request = 1
thread"** — thread นั้นจะ **block (รอ)** เมื่อเรียก database หรือ
external API จนกว่าจะได้ผลลัพธ์

```java
public class ThreadPerRequestProblemDemo {
    /*
     * ปัญหา: thread pool มีขนาดจำกัด (เช่น 200 threads - ทบทวน Part 48)
     *
     * ถ้าแต่ละ request ใช้เวลารอ database/external API 2 วินาที และมี 200 request เข้ามาพร้อมกัน:
     *   -> thread ทั้ง 200 ตัวถูกใช้ไปกับการ "รอ" (thread ไม่ได้ทำงานจริง แค่ block อยู่เฉย ๆ)
     *   -> request ที่ 201 ต้องรอจนกว่าจะมี thread ว่าง (ทบทวนปัญหา cascading failure จาก Part 94)
     *
     * แม้ CPU จะว่างอยู่ (thread ที่ block ไม่ได้ใช้ CPU จริง) แต่ระบบก็รับ request เพิ่มไม่ได้
     * เพราะ "จำนวน thread" คือ bottleneck ไม่ใช่ CPU หรือ memory
     */
}
```

## 2. Reactive Programming คืออะไร: แนวคิดหลัก

**Reactive Programming** ใช้โมเดล **non-blocking, event-driven** — thread
**ไม่ต้อง block รอ** เลย แต่ทำงานอื่นต่อไปได้ทันที แล้ว**รับการแจ้งเตือน
(callback)** เมื่อผลลัพธ์พร้อม

```java
public class ReactiveVsBlockingDemo {
    /*
     * Blocking (Thread-per-Request):
     *   Thread A: เรียก database -> [รอ 2 วินาที ทำอะไรไม่ได้เลย] -> ได้ผลลัพธ์ -> ตอบ response
     *
     * Non-blocking (Reactive):
     *   Thread A: เรียก database (ส่ง request แล้วปล่อยว่างทันที) -> ไปรับ request อื่นต่อ
     *   [2 วินาทีต่อมา] database ตอบกลับมา -> event loop เรียก callback ที่ลงทะเบียนไว้ -> ตอบ response
     *
     * ผลลัพธ์: Thread เดียวสามารถรองรับ concurrent request ได้มากกว่า thread-per-request หลายเท่า
     * เพราะไม่มี thread ไหน "ถูกจอง" ไว้เฉย ๆ ระหว่างรอ I/O เลย (เหมาะกับงานที่มี I/O รอมาก - I/O-bound)
     */
}
```

## 3. Reactive Streams Specification

```java
public class ReactiveStreamsSpecDemo {
    /*
     * Reactive Streams เป็นมาตรฐาน (interface) ที่ library ต่าง ๆ (Project Reactor, RxJava, Akka Streams)
     * implement ตาม เพื่อให้ทำงานร่วมกันได้ (interoperability)
     *
     * 4 interface หลัก:
     *   Publisher<T>: แหล่งข้อมูล ("ฉันจะส่งข้อมูลให้เมื่อพร้อม")
     *   Subscriber<T>: ผู้รับข้อมูล ("ฉันจะรับข้อมูลและประมวลผล")
     *   Subscription: การเชื่อมโยงระหว่าง Publisher-Subscriber (ควบคุมว่าจะขอข้อมูลกี่ตัว - หัวข้อ 6)
     *   Processor<T,R>: ทำหน้าที่เป็นทั้ง Subscriber และ Publisher (แปลงข้อมูลระหว่างทาง)
     *
     * java.util.concurrent.Flow (ทบทวน Part 48-49) มี interface เหล่านี้ built-in ใน JDK ตั้งแต่ Java 9
     */
}
```

## 4. Project Reactor: `Mono` และ `Flux`

**Project Reactor** เป็น library หลักที่ Spring ใช้สำหรับ Reactive
Programming — มี 2 type หลัก:

```java
import reactor.core.publisher.Mono;
import reactor.core.publisher.Flux;

public class MonoFluxDemo {
    public Mono<OrderDto> findOrderById(Long id) {
        // Mono<T>: publisher ที่ส่งข้อมูล "0 หรือ 1 ตัว" - คล้าย Optional<T> (ทบทวน Part 43) แต่เป็น async
        return Mono.just(new OrderDto(id, "PENDING", 100.0));
    }

    public Flux<OrderDto> findAllOrders() {
        // Flux<T>: publisher ที่ส่งข้อมูล "0 ถึงหลายตัว" - คล้าย Stream<T> (ทบทวน Part 41-42) แต่เป็น async
        return Flux.just(
            new OrderDto(1L, "PENDING", 100.0),
            new OrderDto(2L, "COMPLETED", 200.0)
        );
    }
}
```

```java
public class MonoFluxLazinessDemo {
    /*
     * สำคัญมาก: Mono/Flux เป็น "lazy" - ไม่มีอะไรเกิดขึ้นจริงจนกว่าจะมีคน "subscribe"
     *
     * Mono<OrderDto> mono = orderService.findOrderById(42L); // ยังไม่ query database เลย!
     * mono.subscribe(order -> System.out.println(order)); // ตอนนี้ถึงเริ่มทำงานจริง
     *
     * คล้ายกับ Stream (Part 41) ที่ intermediate operation เป็น lazy จนกว่าจะมี terminal operation
     * แต่ Mono/Flux ขยายแนวคิดนี้ไปสู่ asynchronous execution เต็มรูปแบบ
     */
}
```

## 5. Operators พื้นฐาน: map, flatMap, filter

```java
import reactor.core.publisher.Mono;
import reactor.core.publisher.Flux;
import java.time.Duration;

public class ReactiveOperatorsDemo {
    public Flux<String> transformOrders(Flux<OrderDto> orders) {
        return orders
            .filter(order -> order.status().equals("COMPLETED")) // เหมือน Stream.filter (ทบทวน Part 41)
            .map(order -> "Order #" + order.id() + ": $" + order.amount()) // เหมือน Stream.map
            .take(10); // เอาแค่ 10 ตัวแรก
    }

    public Mono<CustomerDto> getOrderWithCustomer(Long orderId, OrderService orderService,
                                                    CustomerService customerService) {
        return orderService.findOrderById(orderId)
            .flatMap(order -> customerService.findCustomerById(order.customerId()));
            // flatMap: ใช้เมื่อผลลัพธ์ของ operation หนึ่งต้องเรียก operation async อีกตัว (คล้าย Part 43 - Optional.flatMap)
            // ถ้าใช้ map เฉย ๆ จะได้ Mono<Mono<CustomerDto>> ซ้อนกัน (ผิด) - flatMap "แบน" มันออกมาเป็น Mono<CustomerDto>
    }

    public Flux<Long> delayedSequence() {
        return Flux.interval(Duration.ofSeconds(1)).take(5);
        // สร้างค่า 0,1,2,3,4 ทุก 1 วินาที - แสดงธรรมชาติ asynchronous ของ Flux ที่ทำงานตามเวลาได้
    }
}
```

## 6. Backpressure: การจัดการเมื่อ Producer เร็วกว่า Consumer

```java
public class BackpressureDemo {
    /*
     * ปัญหา: ถ้า Publisher ส่งข้อมูลเร็วกว่าที่ Subscriber ประมวลผลได้ทัน จะเกิดอะไรขึ้น?
     * (เช่น sensor ส่งข้อมูลทุก millisecond แต่ Subscriber ประมวลผลได้แค่ทุก 100ms)
     *
     * ถ้าไม่มีการจัดการ: ข้อมูลจะสะสมค้างใน memory จนล้น (OutOfMemoryError)
     *
     * Backpressure คือกลไกที่ Subscriber "บอก Publisher" ว่าตัวเองพร้อมรับข้อมูลกี่ตัว
     * ผ่าน Subscription.request(n) (ทบทวนหัวข้อ 3) - Publisher จะส่งแค่เท่าที่ Subscriber ขอเท่านั้น
     * ไม่ส่งท่วมจนเกินความสามารถของ Subscriber
     */
}
```

```java
import reactor.core.publisher.Flux;

public class BackpressureStrategyDemo {
    public Flux<Integer> handleOverflow() {
        return Flux.range(1, 1_000_000)
            .onBackpressureBuffer(1000) // เก็บ buffer สูงสุด 1000 ตัวถ้า consumer ตามไม่ทัน
            .onBackpressureDrop(dropped -> System.out.println("Dropped: " + dropped)); // เกินนั้นทิ้งไปเลย
        // กลยุทธ์อื่น: onBackpressureLatest (เก็บแค่ตัวล่าสุด), onBackpressureError (throw exception)
        // เลือกกลยุทธ์ตามความสำคัญของข้อมูล - ข้อมูล sensor อาจ drop ได้ แต่ transaction ทางการเงินไม่ควร drop
    }
}
```

## 7. Spring WebFlux: Reactive Web Framework

ทบทวนจาก Part 93: Spring Cloud Gateway ทำงานบน **WebFlux** — มาดูการใช้
WebFlux เขียน REST API แบบ reactive เต็มรูปแบบ

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

```java
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Mono;
import reactor.core.publisher.Flux;

@RestController
@RequestMapping("/api/orders")
public class ReactiveOrderController {
    private final ReactiveOrderRepository orderRepository; // ทบทวน Spring Data JPA จาก Part 79 แต่เป็น reactive version

    public ReactiveOrderController(ReactiveOrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @GetMapping("/{id}")
    public Mono<OrderDto> getOrder(@PathVariable Long id) {
        // เหมือนกับ Part 77 (@GetMapping) แต่คืน Mono แทนการคืนค่าตรง ๆ - controller thread ไม่ block รอ database
        return orderRepository.findById(id).map(this::toDto);
    }

    @GetMapping
    public Flux<OrderDto> getAllOrders() {
        return orderRepository.findAll().map(this::toDto);
        // ส่งข้อมูลกลับแบบ stream ทีละตัวเมื่อพร้อม ไม่ต้องรอให้ query เสร็จทั้งหมดก่อนส่ง (ต่างจาก Part 77 ที่ต้องรอ List ครบ)
    }

    private OrderDto toDto(Order order) {
        return new OrderDto(order.getId(), order.getStatus(), order.getAmount());
    }
}
```

**เมื่อไหร่ควรใช้ WebFlux แทน Spring MVC**: WebFlux เหมาะกับระบบที่มี
**concurrent connection จำนวนมาก**และ**I/O-bound มาก** (เช่น API Gateway
ที่ทบทวนจาก Part 93, หรือระบบ streaming/real-time) — ถ้าทีมคุ้นเคยกับ
imperative code (Spring MVC) และ workload ไม่ได้ concurrent สูงมาก
Spring MVC ยังคงเรียบง่ายและเหมาะสมกว่าในหลายกรณี (ไม่ต้อง reactive ทุก
ระบบ)

## 8. WebClient: เรียก HTTP แบบ Non-blocking

ทบทวนจาก Part 91-92: `RestTemplate` เป็น**blocking client** —
`WebClient` เป็นทางเลือกแบบ non-blocking

```java
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Mono;

public class ReactiveInventoryClient {
    private final WebClient webClient;

    public ReactiveInventoryClient(WebClient.Builder webClientBuilder) {
        this.webClient = webClientBuilder.baseUrl("http://inventory-service").build();
    }

    public Mono<Boolean> checkStock(Long productId, int quantity) {
        return webClient.get()
                .uri("/api/inventory/check?productId={id}&qty={qty}", productId, quantity)
                .retrieve()
                .bodyToMono(Boolean.class);
                // thread ที่เรียกเมธอดนี้ "ไม่ block" รอ response - ปล่อยว่างทำงานอื่นได้ทันที
                // (ทบทวนความแตกต่างจาก RestTemplate.getForObject ที่ block รอ ใน Part 92 หัวข้อ 5)
    }

    public Mono<OrderDto> createOrderWithStockCheck(CreateOrderRequest request) {
        return checkStock(request.productId(), request.quantity())
            .flatMap(inStock -> {
                if (!inStock) return Mono.error(new InsufficientStockException(request.productId()));
                return Mono.just(new OrderDto(1L, "PENDING", request.amount()));
            });
        // ประกอบ operation แบบ async หลายขั้นตอนด้วย flatMap (ทบทวนหัวข้อ 5) - ไม่มี thread ไหน block เลย
    }
}
```

## 9. Virtual Threads (Project Loom): ทางเลือกใหม่ใน Java 21

ทบทวนปัญหาจากหัวข้อ 1: Reactive Programming แก้ปัญหา thread-per-request
ได้ แต่**เขียนโค้ดยากขึ้นมาก** (ทบทวนความซับซ้อนของ operator chain) —
**Virtual Threads** (Java 21+) เป็นทางออกใหม่ที่**เขียนโค้ดแบบ blocking
ธรรมดา แต่ได้ประสิทธิภาพใกล้เคียง reactive**

```java
public class VirtualThreadsDemo {
    /*
     * Platform Thread (ธรรมดา - ทบทวน Part 46): ผูกกับ OS thread จริง 1:1 - สร้างได้จำกัด (หลักพัน)
     * Virtual Thread (ใหม่ใน Java 21): JVM จัดการเอง ไม่ผูกกับ OS thread ตายตัว - สร้างได้หลักล้าน!
     *
     * เมื่อ Virtual Thread "block" (รอ I/O เช่น database call):
     *   JVM จะ "ปลด" มันออกจาก OS thread ที่ใช้อยู่ชั่วคราว ให้ OS thread นั้นไปรับ Virtual Thread อื่นทำงานต่อ
     *   พอ I/O เสร็จ ก็ค่อยหา OS thread ว่างมาทำงานต่อให้ Virtual Thread ตัวนั้นใหม่
     *
     * ผลลัพธ์: เขียนโค้ดแบบ blocking ธรรมดา (อ่านง่ายกว่า reactive มาก) แต่ได้ throughput สูง
     * เพราะ thread ที่ "ดูเหมือน block" ไม่ได้ผูกครอง OS thread จริงไว้ตลอดเวลาที่รอ
     */
}
```

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class VirtualThreadUsageDemo {
    public void handleRequestsWithVirtualThreads() {
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 0; i < 100_000; i++) {
                final int requestId = i;
                executor.submit(() -> {
                    // โค้ด blocking ธรรมดา (เหมือน Spring MVC ปกติ - ทบทวน Part 77) แต่รันบน Virtual Thread
                    processOrderBlocking(requestId); // เรียก database/external API แบบ blocking ได้เลย
                });
            }
        } // executor.close() รอทุก task เสร็จอัตโนมัติ (ทบทวน try-with-resources จาก Part 36)
    }

    private void processOrderBlocking(int requestId) {
        // เขียนแบบ imperative ธรรมดา อ่านง่ายกว่า Mono/Flux chain มาก - นี่คือข้อดีหลักของ Virtual Threads
    }
}
```

**ข้อสรุปสำคัญ**: Virtual Threads **ไม่ได้มาแทนที่ Reactive Programming
ทั้งหมด** — Reactive ยังมีข้อดีเรื่อง backpressure (หัวข้อ 6) และ
operator ที่ทรงพลังสำหรับ stream ข้อมูลที่ซับซ้อน แต่สำหรับงานทั่วไปที่
แค่ต้องการ "รองรับ concurrent request จำนวนมากโดยไม่ block thread เปล่า
ๆ" Virtual Threads ให้ผลลัพธ์คล้ายกันด้วยโค้ดที่**อ่านง่ายกว่ามาก**

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียนเมธอดที่ใช้ `WebClient` เรียก 2 service พร้อมกัน
(InventoryService และ PaymentService) แล้วรวมผลลัพธ์ด้วย `Mono.zip`

**เฉลย:**
```java
public Mono<OrderResult> processOrderReactive(Long productId, ChargeRequest chargeRequest) {
    Mono<Boolean> stockCheck = inventoryClient.checkStock(productId, 1);
    Mono<PaymentResult> paymentResult = paymentClient.charge(chargeRequest);

    return Mono.zip(stockCheck, paymentResult)
            .map(tuple -> new OrderResult(tuple.getT1(), tuple.getT2()));
    // Mono.zip รอทั้งสอง Mono เสร็จพร้อมกัน (คล้าย CompletableFuture.allOf จาก Part 49)
    // โดยไม่มี thread ไหน block รอเลยตลอดกระบวนการ
}
```

**2)** อธิบายว่าทำไม Reactive Programming เหมาะกับงาน I/O-bound แต่ไม่ได้
ช่วยอะไรมากสำหรับงาน CPU-bound (เช่น การคำนวณทางคณิตศาสตร์ที่ซับซ้อน)

**เฉลย**: ข้อดีหลักของ Reactive Programming (หัวข้อ 1, 2) คือการ**ไม่
ผูกครอง thread ไว้เฉย ๆ ระหว่างที่รอ I/O** (เช่น รอ database ตอบกลับ,
รอ network response) — ช่วงเวลาที่ thread "รอ" นี้**ไม่ได้ใช้ CPU จริง
เลย** ดังนั้นการปลดปล่อย thread ให้ไปทำงานอื่นระหว่างรอจึงเพิ่ม
throughput ได้มาก (รองรับ concurrent request ได้มากขึ้นด้วยจำนวน thread
เท่าเดิม) ในทางกลับกัน **งาน CPU-bound** (เช่น การคำนวณที่ซับซ้อน, การ
ประมวลผลภาพ) **ใช้ CPU เต็มที่ตลอดเวลาที่ทำงาน ไม่มีช่วง "รอเฉย ๆ" ให้
ปลดปล่อย thread เลย** — ไม่ว่าจะเขียนแบบ reactive หรือ blocking ธรรมดา
งานนั้นก็ต้อง**ใช้ CPU cycle จำนวนเท่ากันอยู่ดี** การเปลี่ยนมาใช้ Reactive
Programming กับงาน CPU-bound จึง**ไม่ได้ช่วยลดเวลาทำงานหรือเพิ่ม
throughput เลย** (อาจเพิ่มความซับซ้อนของโค้ดโดยไม่ได้ประโยชน์ใด ๆ) —
สำหรับงาน CPU-bound การ**เพิ่มจำนวน CPU core**หรือใช้**Parallel Stream**
(ทบทวน Part 42) เหมาะสมกว่ามาก

**3)** อธิบายว่า Virtual Threads (Java 21) แก้ปัญหาความซับซ้อนของโค้ด
แบบ Reactive Programming ได้อย่างไร โดยยังคงประสิทธิภาพที่ดี

**เฉลย**: โค้ดแบบ **Reactive Programming** (Mono/Flux, operator chain
อย่าง `map`/`flatMap` ในหัวข้อ 5) มีข้อเสียสำคัญคือ**อ่านและเขียนยากกว่า
โค้ด imperative ธรรมดามาก** โดยเฉพาะเมื่อ logic มีความซับซ้อน (เช่น
if-else ที่ซ้อนกันหลายชั้น หรือ error handling ที่ต้องกระจายไปตาม
operator หลายตัว) — นักพัฒนาต้อง**คิดแบบ asynchronous callback ตลอดเวลา**
ซึ่งเป็น mental model ที่ต่างจากการเขียนโค้ดทั่วไปมาก **Virtual Threads**
(หัวข้อ 9) แก้ปัญหานี้โดยให้นักพัฒนา**เขียนโค้ดแบบ blocking ธรรมดา**
(imperative, อ่านจากบนลงล่างตามปกติ เหมือนที่เรียนมาตลอดหลักสูตร) แต่
**JVM จัดการเบื้องหลังให้**โดย**ปลด thread ออกจาก OS thread จริงเมื่อ
block** (แทนที่จะครอบครองไว้เฉย ๆ) ทำให้**ได้ throughput สูงใกล้เคียงกับ
Reactive Programming**โดยที่**โค้ดยังคงเรียบง่ายและอ่านง่ายเหมือนโค้ด
blocking ทั่วไป** — เป็นการแก้ปัญหาที่ระดับ JVM/runtime แทนที่จะผลักภาระ
ความซับซ้อนไปให้นักพัฒนาต้องเขียนโค้ดที่ซับซ้อนขึ้นเอง

### สรุปเนื้อหา Part 102

- Thread-per-Request Model มีข้อจำกัดเมื่อ thread จำนวนมาก block รอ I/O
  พร้อมกัน (thread pool หมด)
- Reactive Programming ใช้โมเดล non-blocking event-driven ทำให้ thread
  ไม่ถูกผูกครองระหว่างรอ I/O
- Mono (0-1 ค่า) และ Flux (0-many ค่า) เป็น type หลักของ Project Reactor
  ที่ Spring ใช้
- Operator (map, flatMap, filter) ประกอบ transformation แบบ async ได้
  คล้าย Stream API แต่ทำงานแบบ lazy และ non-blocking
- Backpressure ป้องกัน Publisher ส่งข้อมูลท่วม Subscriber ที่ตามไม่ทัน
- Spring WebFlux + WebClient ให้ REST API และ HTTP client แบบ
  non-blocking เต็มรูปแบบ
- Virtual Threads (Java 21) ให้ throughput สูงคล้าย Reactive แต่เขียนโค้ด
  แบบ blocking ธรรมดาที่อ่านง่ายกว่ามาก
- Reactive/Virtual Threads เหมาะกับงาน I/O-bound; งาน CPU-bound ควรใช้
  Parallel Stream หรือเพิ่ม CPU core แทน

**ต่อไป**: [Part 103 — Capstone Project: ระบบ E-Commerce แบบเต็มรูปแบบ (ตอนที่ 1)](./part-103-capstone-ecommerce-part1.md)
