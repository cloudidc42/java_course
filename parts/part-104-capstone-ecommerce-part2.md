# Part 104: Capstone Project: ระบบ E-Commerce แบบเต็มรูปแบบ (ตอนที่ 2)

> ขั้นตอนที่ 1031-1040 ของหลักสูตร | ระดับ: ระดับมืออาชีพ/โลก

## สารบัญ

1. เพิ่ม Authentication/Authorization ด้วย JWT
2. ป้องกัน IDOR: ผู้ใช้เข้าถึงแค่ order ของตัวเอง
3. Unit Testing: ทดสอบ Business Logic ของ OrderProcessingService
4. Integration Testing ด้วย Testcontainers
5. Async Notification ผ่าน RabbitMQ
6. Dockerize แอปพลิเคชันทั้งระบบ
7. Kubernetes Manifests สำหรับ Deploy
8. CI/CD Pipeline สำหรับ SimpleShop
9. Monitoring: เพิ่ม Custom Metrics
10. แบบฝึกหัดและสรุปโครงการ

---

## 1. เพิ่ม Authentication/Authorization ด้วย JWT

ทบทวนจาก Part 82, 83: เพิ่มระบบ authentication ให้ SimpleShop ที่สร้าง
ใน Part 103

```java
package com.simpleshop.api.security;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
public class SimpleShopSecurityConfig {
    @Bean
    public PasswordEncoder passwordEncoder() { return new BCryptPasswordEncoder(); } // ทบทวน Part 82

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http, JwtAuthenticationFilter jwtFilter)
            throws Exception {
        http
            .csrf(csrf -> csrf.disable()) // ทบทวน Part 83, 101 - ปลอดภัยเมื่อใช้ JWT ผ่าน header
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**", "/api/products/**").permitAll() // ดูสินค้าได้โดยไม่ต้อง login
                .requestMatchers("/api/orders/**").authenticated() // สั่งซื้อต้อง login
                .requestMatchers("/actuator/**").hasRole("ADMIN") // ทบทวน Part 99, 101 - จำกัดสิทธิ์ actuator
                .anyRequest().authenticated())
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }
}
```

```java
package com.simpleshop.api.controller;

import com.simpleshop.api.service.AuthService;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/auth")
public class AuthController {
    private final AuthService authService;

    public AuthController(AuthService authService) { this.authService = authService; }

    public record LoginRequest(String email, String password) {}
    public record TokenResponse(String token) {}

    @PostMapping("/login")
    public TokenResponse login(@RequestBody LoginRequest request) {
        // ทบทวน Part 82-83 - ตรวจสอบ credential ผ่าน UserDetailsService แล้ว generate JWT
        String token = authService.authenticate(request.email(), request.password());
        return new TokenResponse(token);
    }
}
```

## 2. ป้องกัน IDOR: ผู้ใช้เข้าถึงแค่ order ของตัวเอง

ทบทวนช่องโหว่จาก Part 101 หัวข้อ 5 — ต้องมั่นใจว่าลูกค้า A **ดูข้อมูล
order ของลูกค้า B ไม่ได้**

```java
package com.simpleshop.api.controller;

import com.simpleshop.api.service.OrderProcessingService;
import org.springframework.http.ResponseEntity;
import org.springframework.security.core.Authentication;
import org.springframework.web.bind.annotation.*;
import java.util.List;

@RestController
@RequestMapping("/api/orders")
public class SecureOrderController {
    private final OrderProcessingService orderProcessingService;

    public SecureOrderController(OrderProcessingService orderProcessingService) {
        this.orderProcessingService = orderProcessingService;
    }

    @GetMapping("/my-orders")
    public List<OrderProcessingService.OrderSummaryDto> getMyOrders(Authentication authentication) {
        // ใช้ email จาก JWT (Authentication.getName() - ทบทวน Part 83) ไม่ใช่จาก request parameter
        // ป้องกัน IDOR โดยธรรมชาติ เพราะ "ดึงจาก token เสมอ ไม่เคยเชื่อค่าที่ client ส่งมา"
        String customerEmail = authentication.getName();
        return orderProcessingService.getOrderHistory(customerEmail);
    }

    @GetMapping("/{id}")
    public ResponseEntity<OrderProcessingService.OrderSummaryDto> getOrder(
            @PathVariable Long id, Authentication authentication) {
        var order = orderProcessingService.getOrderByIdForCustomer(id, authentication.getName());
        // Service layer เช็ค ownership ภายใน (ทบทวน Part 101 หัวข้อ 5) - throw AccessDeniedException ถ้าไม่ตรงกัน
        return ResponseEntity.ok(order);
    }
}
```

## 3. Unit Testing: ทดสอบ Business Logic ของ OrderProcessingService

ทบทวนจาก Part 58-59: ทดสอบ business logic แบบ isolate โดย mock
Repository

```java
package com.simpleshop.api.service;

import com.simpleshop.domain.model.Product;
import com.simpleshop.domain.repository.OrderRepository;
import com.simpleshop.domain.repository.ProductRepository;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import java.math.BigDecimal;
import java.util.List;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class OrderProcessingServiceTest {
    @Mock private OrderRepository orderRepository;
    @Mock private ProductRepository productRepository;
    @InjectMocks private OrderProcessingService orderProcessingService;

    @Test
    void shouldThrowWhenStockInsufficient() {
        Product product = new Product("Laptop", new BigDecimal("999.99"), 2); // สต็อกเหลือแค่ 2
        when(productRepository.findById(1L)).thenReturn(Optional.of(product));

        var request = new OrderProcessingService.CreateOrderRequest(
            "alice@example.com",
            List.of(new OrderProcessingService.OrderItemRequest(1L, 5)) // สั่งซื้อ 5 - เกินสต็อก!
        );

        // ทบทวน assertThatThrownBy จาก Part 58 - ทดสอบ business rule ที่เขียนไว้ใน Product.decreaseStock (Part 103)
        assertThatThrownBy(() -> orderProcessingService.createOrder(request))
                .isInstanceOf(InsufficientStockException.class);

        verify(orderRepository, never()).save(any()); // ทบทวน never() จาก Part 59 - ต้องไม่บันทึก order ที่ล้มเหลว
    }

    @Test
    void shouldCreateOrderSuccessfullyWhenStockAvailable() {
        Product product = new Product("Mouse", new BigDecimal("29.99"), 10);
        when(productRepository.findById(1L)).thenReturn(Optional.of(product));

        var request = new OrderProcessingService.CreateOrderRequest(
            "bob@example.com",
            List.of(new OrderProcessingService.OrderItemRequest(1L, 3))
        );

        orderProcessingService.createOrder(request);

        verify(productRepository).save(argThat(p -> p.getStockQuantity() == 7)); // 10 - 3 = 7
        verify(orderRepository).save(any());
    }
}
```

## 4. Integration Testing ด้วย Testcontainers

ทบทวนจาก Part 84: ทดสอบ concurrency จริงต้องใช้ database จริง (H2 อาจไม่
จำลอง locking behavior ได้แม่นยำเท่า PostgreSQL จริง)

```java
package com.simpleshop.api.service;

import com.simpleshop.domain.model.Product;
import com.simpleshop.domain.repository.ProductRepository;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import java.math.BigDecimal;
import java.util.List;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@Testcontainers
class OrderConcurrencyIntegrationTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired private OrderProcessingService orderProcessingService;
    @Autowired private ProductRepository productRepository;

    @Test
    void shouldNotOversellWhenMultipleCustomersOrderConcurrently() throws InterruptedException {
        // ทบทวน Part 103 หัวข้อ 7 - ทดสอบว่า Optimistic Locking + @Retryable ป้องกัน overselling จริง
        Product product = productRepository.save(new Product("Limited Item", new BigDecimal("50.00"), 5));

        int numberOfConcurrentCustomers = 10; // 10 คนพยายามสั่งซื้อพร้อมกัน แต่มีสต็อกแค่ 5
        ExecutorService executor = Executors.newFixedThreadPool(10); // ทบทวน Part 48
        CountDownLatch latch = new CountDownLatch(numberOfConcurrentCustomers);
        var successCount = new java.util.concurrent.atomic.AtomicInteger(0);

        for (int i = 0; i < numberOfConcurrentCustomers; i++) {
            final int customerId = i;
            executor.submit(() -> {
                try {
                    var request = new OrderProcessingService.CreateOrderRequest(
                        "customer" + customerId + "@example.com",
                        List.of(new OrderProcessingService.OrderItemRequest(product.getId(), 1))
                    );
                    orderProcessingService.createOrder(request);
                    successCount.incrementAndGet();
                } catch (Exception ignored) {
                    // คำสั่งซื้อที่เกินสต็อกจะ throw exception - เป็นพฤติกรรมที่ถูกต้อง
                } finally {
                    latch.countDown();
                }
            });
        }
        latch.await(); // ทบทวน CountDownLatch จาก Part 47 - รอทุก thread เสร็จก่อนตรวจสอบผลลัพธ์

        assertThat(successCount.get()).isEqualTo(5); // ต้องสำเร็จแค่ 5 คำสั่งเท่านั้น (เท่ากับสต็อกที่มี)
        Product updated = productRepository.findById(product.getId()).orElseThrow();
        assertThat(updated.getStockQuantity()).isEqualTo(0); // สต็อกต้องเหลือ 0 พอดี ไม่ติดลบ (ไม่ overselling)
    }
}
```

**นี่คือ test ที่สำคัญที่สุดของทั้งโครงการ**: มันพิสูจน์ว่า design
decision ในหัวข้อ 7 ของ Part 103 (Optimistic Locking + Retry) **ทำงานได้
จริงภายใต้ concurrent load** ไม่ใช่แค่ทำงานถูกต้องใน scenario เดียวแบบ
sequential — นี่คือความแตกต่างสำคัญระหว่าง unit test (หัวข้อ 3) กับ
integration test ที่ทดสอบ concurrency จริง

## 5. Async Notification ผ่าน RabbitMQ

ทบทวนจาก Part 88: แจ้งเตือนลูกค้าแบบ async หลังสร้าง order สำเร็จ

```java
package com.simpleshop.api.service;

import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Component;
import org.springframework.transaction.event.TransactionalEventListener;

@Component
public class OrderEventPublisher {
    private final RabbitTemplate rabbitTemplate;

    public OrderEventPublisher(RabbitTemplate rabbitTemplate) { this.rabbitTemplate = rabbitTemplate; }

    public record OrderCreatedEvent(Long orderId, String customerEmail) {}

    @TransactionalEventListener // ทบทวน Part 65, 88 - ส่ง event หลัง transaction commit สำเร็จเท่านั้น
    public void onOrderCreated(OrderCreatedEvent event) {
        // สำคัญมาก: ถ้าส่ง message ก่อน transaction commit และ transaction rollback ทีหลัง
        // จะเกิด "แจ้งเตือนลูกค้าว่าสั่งซื้อสำเร็จ" ทั้งที่ order ไม่ได้ถูกบันทึกจริง (data inconsistency)
        rabbitTemplate.convertAndSend("order.exchange", "order.created", event);
    }
}
```

```java
package com.simpleshop.api.service;

import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

@Component
public class OrderNotificationConsumer {
    @RabbitListener(queues = "order.created.queue") // ทบทวน Part 88
    public void sendConfirmationEmail(OrderEventPublisher.OrderCreatedEvent event) {
        System.out.println("Sending confirmation email to " + event.customerEmail()
                + " for order " + event.orderId());
        // ทำงานแบบ async - ไม่ทำให้ createOrder() (Part 103 หัวข้อ 7) ต้องรอ email ส่งเสร็จก่อนตอบ response
    }
}
```

## 6. Dockerize แอปพลิเคชันทั้งระบบ

ทบทวนจาก Part 95: multi-stage build สำหรับ multi-module project

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /build
COPY pom.xml .
COPY simpleshop-domain/pom.xml simpleshop-domain/
COPY simpleshop-api/pom.xml simpleshop-api/
RUN mvn dependency:go-offline
COPY simpleshop-domain/src simpleshop-domain/src
COPY simpleshop-api/src simpleshop-api/src
RUN mvn clean package -DskipTests

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=build /build/simpleshop-api/target/simpleshop-api-1.0.0.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

```yaml
# docker-compose.yml สำหรับ local development
version: '3.8'
services:
  simpleshop-api:
    build: .
    ports: ["8080:8080"]
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://shop-db:5432/simpleshop
      - SPRING_RABBITMQ_HOST=rabbitmq
    depends_on: [shop-db, rabbitmq]
  shop-db:
    image: postgres:16-alpine
    environment:
      - POSTGRES_DB=simpleshop
      - POSTGRES_PASSWORD=secret
  rabbitmq:
    image: rabbitmq:3-management-alpine
```

## 7. Kubernetes Manifests สำหรับ Deploy

ทบทวนจาก Part 96: Deployment + Service + HPA สำหรับ production

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: simpleshop-api
spec:
  replicas: 3
  selector:
    matchLabels: { app: simpleshop-api }
  template:
    metadata:
      labels: { app: simpleshop-api }
    spec:
      containers:
        - name: simpleshop-api
          image: mycompany/simpleshop-api:1.0.0
          ports: [{ containerPort: 8080 }]
          readinessProbe:
            httpGet: { path: /actuator/health/readiness, port: 8080 }
            initialDelaySeconds: 20
          livenessProbe:
            httpGet: { path: /actuator/health/liveness, port: 8080 }
            initialDelaySeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  name: simpleshop-api
spec:
  selector: { app: simpleshop-api }
  ports: [{ port: 80, targetPort: 8080 }]
```

## 8. CI/CD Pipeline สำหรับ SimpleShop

ทบทวนจาก Part 97: pipeline ที่รัน unit test + integration test
(Testcontainers ในหัวข้อ 4) ก่อน deploy

```yaml
# .github/workflows/ci.yml
name: SimpleShop CI/CD
on:
  push: { branches: [main] }
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: '21', distribution: 'temurin' }
      - name: Run unit tests
        run: mvn test
      - name: Run integration tests (Testcontainers needs Docker)
        run: mvn verify -Dtest='**/*IntegrationTest'
  build-and-deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build and push Docker image
        run: |
          docker build -t ghcr.io/mycompany/simpleshop-api:${{ github.sha }} .
          docker push ghcr.io/mycompany/simpleshop-api:${{ github.sha }}
      - name: Deploy to Kubernetes
        run: kubectl set image deployment/simpleshop-api simpleshop-api=ghcr.io/mycompany/simpleshop-api:${{ github.sha }}
```

## 9. Monitoring: เพิ่ม Custom Metrics

ทบทวนจาก Part 99: ติดตาม business metric ที่สำคัญของ SimpleShop

```java
package com.simpleshop.api.service;

import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.stereotype.Component;

@Component
public class OrderMetrics {
    private final MeterRegistry meterRegistry;

    public OrderMetrics(MeterRegistry meterRegistry) { this.meterRegistry = meterRegistry; }

    public void recordOrderCreated(java.math.BigDecimal amount) {
        meterRegistry.counter("simpleshop.orders.created").increment();
        meterRegistry.summary("simpleshop.orders.amount").record(amount.doubleValue());
    }

    public void recordOversellAttemptBlocked() {
        // metric นี้สำคัญมาก - ถ้าค่าสูงผิดปกติ อาจบ่งบอกว่ามีสินค้าขายดีที่สต็อกไม่พอ ควรเติมสต็อกเพิ่ม
        meterRegistry.counter("simpleshop.orders.oversell_blocked").increment();
    }
    // Grafana dashboard (ทบทวน Part 99) แสดง metric นี้ให้ทีมธุรกิจเห็นแนวโน้มการขายแบบ real-time
}
```

## 10. แบบฝึกหัดและสรุปโครงการ

### แบบฝึกหัด

**1)** เพิ่ม endpoint `GET /api/orders/my-orders` ที่มี pagination
(ทบทวน Part 85) แทนการคืนค่าทั้งหมดในครั้งเดียว

**เฉลย:**
```java
@GetMapping("/my-orders")
public Page<OrderSummaryDto> getMyOrders(Authentication auth,
        @RequestParam(defaultValue = "0") int page, @RequestParam(defaultValue = "10") int size) {
    return orderProcessingService.getOrderHistoryPaged(auth.getName(), PageRequest.of(page, size));
}
```

**2)** อธิบายว่าทำไม `OrderEventPublisher.onOrderCreated` ต้องใช้
`@TransactionalEventListener` แทน `@EventListener` ธรรมดา ในบริบทของ
ระบบนี้

**เฉลย**: `@EventListener` ธรรมดาจะ**ประมวลผล event ทันทีที่ publish**
ซึ่งอาจเกิดขึ้น**ก่อนที่ transaction ของ `createOrder`** (Part 103 หัวข้อ
7) **จะ commit สำเร็จจริง** — ถ้า transaction เกิด rollback ทีหลัง (เช่น
เพราะ `@Retryable` ใช้ครบจำนวนครั้งแล้วยังล้มเหลว) **ข้อความแจ้งเตือนที่
ส่งไปแล้วจะกลายเป็นข้อมูลเท็จ** (แจ้งลูกค้าว่า order สำเร็จ ทั้งที่จริง
แล้ว order ไม่ได้ถูกบันทึกในระบบเลย) `@TransactionalEventListener` (โดย
default ใช้ `AFTER_COMMIT` phase) **รอให้ transaction commit สำเร็จ
สมบูรณ์ก่อน**ถึงจะประมวลผล event — การันตีว่า**จะไม่มีการแจ้งเตือนที่ผิด
พลาดจาก transaction ที่ล้มเหลว**เกิดขึ้นเด็ดขาด สอดคล้องกับหลักการ
Consistency ที่เน้นมาตลอดในระบบธุรกรรมทางการเงิน (ทบทวน Part 100 หัวข้อ
10)

**3)** สรุปว่า `OrderConcurrencyIntegrationTest` (หัวข้อ 4) พิสูจน์อะไร
ที่ Unit Test ในหัวข้อ 3 พิสูจน์ไม่ได้

**เฉลย**: **Unit Test** (หัวข้อ 3) ทดสอบ `OrderProcessingService`
**แบบ isolate โดย mock `ProductRepository`** — มันพิสูจน์ได้แค่ว่า
**logic ภายในเมธอดถูกต้องตาม scenario ที่กำหนดไว้ทีละกรณี** (เช่น "ถ้า
สต็อกไม่พอ ต้อง throw exception") แต่**ไม่ได้ทดสอบพฤติกรรมจริงของ
database เมื่อมีหลาย transaction แข่งกันเข้าถึงข้อมูลเดียวกันพร้อมกัน
จริง ๆ** เพราะ mock ไม่มี concurrency behavior ที่แท้จริง (mock แค่คืน
ค่าตามที่สั่งไว้ ไม่มี locking, ไม่มี version conflict) **Integration
Test ด้วย Testcontainers** (หัวข้อ 4) ใช้ **PostgreSQL จริง** และ**สร้าง
thread หลายตัวยิง request พร้อมกันจริง** (ทบทวน `ExecutorService` จาก
Part 48) — มันพิสูจน์ได้ว่า**กลไก Optimistic Locking (`@Version`) และ
`@Retryable` ที่ออกแบบไว้ทำงานถูกต้องภายใต้ race condition จริง** ซึ่งเป็น
สิ่งที่ **สำคัญที่สุดสำหรับระบบนี้** (ป้องกัน overselling) แต่**เป็นสิ่ง
ที่ unit test ที่ mock ทุกอย่างไม่มีทางจับ bug ประเภทนี้ได้เลย** แม้ unit
test จะผ่านหมดทุกตัวก็ตาม นี่คือเหตุผลที่ต้องมี**ทั้งสองระดับของการ
ทดสอบ**ทำงานเสริมกัน (ทบทวน Testing Pyramid จาก Part 58, 84)

### สรุปเนื้อหา Part 104 และสรุปโครงการ Capstone ทั้งหมด

- เพิ่ม JWT Authentication/Authorization และป้องกัน IDOR ด้วยการดึงข้อมูล
  ผู้ใช้จาก token เสมอ ไม่เชื่อค่าจาก client
- Unit Test (mock) ทดสอบ business logic แบบ isolate; Integration Test
  (Testcontainers) พิสูจน์ concurrency behavior ที่ unit test ทำไม่ได้
- `@TransactionalEventListener` ป้องกัน async notification ที่ผิดพลาด
  จาก transaction ที่ยังไม่ commit หรือ rollback ไปแล้ว
- Docker multi-stage build + docker-compose สำหรับ local development;
  Kubernetes manifests (Deployment/Service/Probes) สำหรับ production
- CI/CD Pipeline รัน unit + integration test ก่อน build image และ deploy
  เสมอ (fail fast)
- Custom Metrics ติดตาม business KPI (จำนวน order, ยอดขาย, การพยายาม
  overselling ที่ถูกบล็อก)

**ตลอด Part 103-104 เราได้สร้างระบบ E-Commerce ที่ใช้แนวคิดจากทั้ง
หลักสูตร 104 Part มาประกอบกันเป็นระบบที่ใช้งานได้จริง**: Domain-Driven
Design, Transaction Management, Concurrency Control, Security,
Testing ครบทุกระดับ, Caching, Async Messaging, Containerization,
Orchestration, CI/CD, และ Observability — นี่คือภาพรวมของงานที่วิศวกร
ซอฟต์แวร์มืออาชีพต้องทำในระบบจริง

**ต่อไป**: [Part 105 — Interview Preparation และ Career Roadmap](./part-105-career-roadmap.md)
