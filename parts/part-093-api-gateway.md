# Part 93: API Gateway

> ขั้นตอนที่ 921-930 ของหลักสูตร | ระดับ: ระดับมืออาชีพ/โลก

## สารบัญ

1. ปัญหา: Client ต้องรู้จัก Endpoint ของทุก Microservice
2. API Gateway คืออะไร แก้ปัญหาอะไร
3. Spring Cloud Gateway: ตั้งค่าพื้นฐาน
4. Route Configuration: การกำหนดเส้นทาง
5. Predicate และ Filter: ปรับแต่ง Request/Response
6. Cross-Cutting Concerns: Authentication ที่ Gateway
7. Rate Limiting ที่ระดับ Gateway
8. Circuit Breaker ที่ Gateway (เชื่อมโยงสู่ Part 94)
9. Gateway กับ Service Discovery (เชื่อมโยงกับ Part 92)
10. แบบฝึกหัดและสรุป

---

## 1. ปัญหา: Client ต้องรู้จัก Endpoint ของทุก Microservice

ทบทวนจาก Part 91: ระบบมี `OrderService`, `InventoryService`,
`PaymentService`, `CustomerService` แยกกัน — ถ้า **client (เว็บ/mobile
app) ต้องเรียกแต่ละ service โดยตรง** จะเกิดปัญหา:

```java
public class ClientCallingServicesDirectlyProblem {
    /*
     * Mobile App ต้องรู้จัก URL ของทุก service:
     *   http://order-service.internal/api/orders
     *   http://inventory-service.internal/api/inventory
     *   http://payment-service.internal/api/payments
     *   http://customer-service.internal/api/customers
     *
     * ปัญหา:
     *   1. Client ต้องจัดการ authentication แยกกับทุก service (ซ้ำซ้อนมาก)
     *   2. ถ้าเปลี่ยนจำนวน/โครงสร้าง service ภายใน -> client ทุกตัว (web, mobile, partner API) ต้องแก้ไขตาม
     *   3. ไม่มีจุดกลางสำหรับ cross-cutting concerns เช่น rate limiting, logging, CORS
     *   4. เปิด internal service ให้ client ภายนอกเข้าถึงตรง ๆ เพิ่มความเสี่ยงด้านความปลอดภัย
     */
}
```

## 2. API Gateway คืออะไร แก้ปัญหาอะไร

**API Gateway** เป็น **จุดเข้าเดียว (single entry point)** ที่รับ
request ทั้งหมดจาก client แล้ว**route ไปยัง microservice ที่เหมาะสม**
ภายใน — client เห็นแค่ Gateway ตัวเดียว ไม่รู้จัก service ภายในเลย

```
                          ┌─────────────────┐
Client ──── request ────▶│   API Gateway    │
(รู้จักแค่ Gateway            │  (single entry)  │
 ตัวเดียว)                   └────────┬─────────┘
                                       │ route ตาม path/rule
                    ┌──────────────────┼──────────────────┐
                    ▼                  ▼                  ▼
             OrderService      InventoryService     PaymentService
```

**ประโยชน์**: (1) **จุดกลางสำหรับ cross-cutting concerns** (auth, rate
limit, logging — หัวข้อ 6, 7), (2) **ซ่อนโครงสร้างภายใน**จาก client
(เปลี่ยน service ภายในได้โดย client ไม่รู้ตัว), (3) **ลด round-trip**
(บาง Gateway รวมหลาย request เป็นครั้งเดียวได้ - ทบทวน API Composition
จาก Part 91)

## 3. Spring Cloud Gateway: ตั้งค่าพื้นฐาน

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ApiGatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(ApiGatewayApplication.class, args);
        // Spring Cloud Gateway ทำงานบน Spring WebFlux (reactive - ปูทางสู่ Part 102)
        // ต่างจาก Spring MVC ทั่วไปที่ใช้ใน Part 77 - รองรับ concurrent request จำนวนมากได้ดีกว่า
    }
}
```

## 4. Route Configuration: การกำหนดเส้นทาง

```yaml
# application.yml
spring:
  cloud:
    gateway:
      routes:
        - id: order-service-route
          uri: lb://order-service   # lb:// = ใช้ load balancer ผ่าน Eureka (ทบทวน Part 92)
          predicates:
            - Path=/api/orders/**    # request ที่ path ขึ้นต้นด้วย /api/orders ไปที่ order-service
        - id: inventory-service-route
          uri: lb://inventory-service
          predicates:
            - Path=/api/inventory/**
        - id: payment-service-route
          uri: lb://payment-service
          predicates:
            - Path=/api/payments/**
```

```java
public class RouteConfigExplanation {
    /*
     * Client ยิง request: GET http://api-gateway.com/api/orders/42
     *   -> Gateway เช็ค routes ทั้งหมด -> path ตรงกับ "order-service-route" (Path=/api/orders/**)
     *   -> forward ไปที่ order-service (ผ่าน Eureka หา instance จริง - ทบทวน Part 92)
     *   -> order-service ตอบกลับ -> Gateway ส่งต่อ response ให้ client
     *
     * Client ไม่รู้เลยว่า order-service อยู่ที่ไหน มี instance กี่ตัว - รู้จักแค่ Gateway ตัวเดียว
     */
}
```

## 5. Predicate และ Filter: ปรับแต่ง Request/Response

```java
import org.springframework.cloud.gateway.route.RouteLocator;
import org.springframework.cloud.gateway.route.builder.RouteLocatorBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class GatewayRouteConfig {
    @Bean
    public RouteLocator customRoutes(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("order-service-route", r -> r
                .path("/api/orders/**") // Predicate: เงื่อนไขว่า request ไหนเข้า route นี้
                .filters(f -> f
                    .addRequestHeader("X-Gateway-Source", "api-gateway") // Filter: แก้ไข request ก่อนส่งต่อ
                    .stripPrefix(0)) // ไม่ตัด path prefix ออก (ส่ง path เดิมไปที่ order-service)
                .uri("lb://order-service"))
            .build();
        // เขียนแบบ Java DSL ก็ได้ (ทางเลือกจาก YAML ในหัวข้อ 4) - เหมาะกับ logic ที่ซับซ้อนขึ้น
    }
}
```

```java
public class PredicateAndFilterConcepts {
    /*
     * Predicate: "เงื่อนไข" ว่า request ไหนควรเข้า route นี้ (Path, Method, Header, Query Param, ...)
     * Filter: "การแปลง" request ก่อนส่งต่อ หรือ response ก่อนส่งกลับ client
     *   - AddRequestHeader / AddResponseHeader
     *   - StripPrefix (ตัด path prefix ก่อนส่งต่อ เช่น /api/v1/orders/42 -> /orders/42)
     *   - RequestRateLimiter (หัวข้อ 7)
     *   - CircuitBreaker (หัวข้อ 8)
     */
}
```

## 6. Cross-Cutting Concerns: Authentication ที่ Gateway

ทบทวนจาก Part 83: แทนที่จะให้**ทุก microservice** ตรวจสอบ JWT เอง (ซ้ำซ้อน
มาก) สามารถ**ตรวจสอบที่ Gateway ครั้งเดียว** แล้วส่งต่อข้อมูลผู้ใช้ผ่าน
header ไปให้ service ภายใน

```java
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.http.HttpStatus;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

@Component
public class JwtAuthenticationGlobalFilter implements GlobalFilter, Ordered {
    private final JwtVerificationService jwtService; // ทบทวนจาก Part 83

    public JwtAuthenticationGlobalFilter(JwtVerificationService jwtService) { this.jwtService = jwtService; }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getPath().toString();
        if (path.startsWith("/api/auth")) return chain.filter(exchange); // login/register ไม่ต้องมี JWT

        String authHeader = exchange.getRequest().getHeaders().getFirst("Authorization");
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }

        var claims = jwtService.verifyToken(authHeader.substring(7));
        if (claims.isEmpty()) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }

        // ตรวจสอบผ่านแล้ว - แนบ username ผ่าน header ใหม่ให้ service ภายในใช้ได้เลย ไม่ต้อง verify JWT ซ้ำ
        ServerHttpRequest mutatedRequest = exchange.getRequest().mutate()
                .header("X-Authenticated-User", claims.get().getSubject())
                .build();
        return chain.filter(exchange.mutate().request(mutatedRequest).build());
    }

    @Override
    public int getOrder() { return -1; } // รันก่อน filter อื่น ๆ ทั้งหมด (ทบทวนแนวคิด Filter Chain จาก Part 72, 88)
}
```

**ประโยชน์สำคัญ**: `OrderService`, `InventoryService` ฯลฯ **ไม่ต้องมี
logic ตรวจสอบ JWT เลย** — เชื่อ header `X-Authenticated-User` ที่ Gateway
ยืนยันมาให้แล้ว (ลด code ซ้ำซ้อนข้าม service — DRY ทบทวนจาก Part 57)

## 7. Rate Limiting ที่ระดับ Gateway

ทบทวนจาก Part 85: การทำ Rate Limiting **ที่ Gateway** ดีกว่าทำที่แต่ละ
service เพราะเป็น**จุดเดียวที่ควบคุมทุก request จากภายนอกได้ทั้งหมด**

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-service-route
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10  # อนุญาต 10 request/วินาที ต่อ key
                redis-rate-limiter.burstCapacity: 20   # อนุญาต burst สูงสุด 20 request ในช่วงสั้น ๆ
```

```java
public class RateLimiterKeyResolverExplanation {
    /*
     * Spring Cloud Gateway ใช้ Redis (ทบทวน Part 87) เก็บ counter ของแต่ละ "key"
     * (default: ตาม IP address, ปรับให้ใช้ user id จาก JWT ได้ผ่าน KeyResolver custom bean)
     * ทำให้ rate limit แชร์กันข้ามทุก Gateway instance ได้ (ถ้ามี Gateway หลาย instance)
     */
}
```

## 8. Circuit Breaker ที่ Gateway (เชื่อมโยงสู่ Part 94)

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: inventory-service-route
          uri: lb://inventory-service
          predicates:
            - Path=/api/inventory/**
          filters:
            - name: CircuitBreaker
              args:
                name: inventoryCircuitBreaker
                fallbackUri: forward:/fallback/inventory
```

```java
public class CircuitBreakerAtGatewayPreview {
    /*
     * ถ้า inventory-service ล้มเหลวซ้ำ ๆ (timeout, error) Circuit Breaker "เปิด" (open)
     * -> Gateway หยุดส่ง request ไปที่ inventory-service ชั่วคราว (ไม่รอ timeout ทุกครั้งอีก)
     * -> ส่ง request ไปที่ fallbackUri แทน (เช่น คืนค่า default หรือ error message ที่สุภาพ)
     * รายละเอียดเต็มรูปแบบของ Circuit Breaker (Resilience4j) จะเรียนใน Part 94
     */
}
```

## 9. Gateway กับ Service Discovery (เชื่อมโยงกับ Part 92)

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

```properties
spring.application.name=api-gateway
eureka.client.service-url.defaultZone=http://localhost:8761/eureka/
spring.cloud.gateway.discovery.locator.enabled=true
```

```java
public class GatewayServiceDiscoveryIntegration {
    /*
     * "lb://order-service" ในหัวข้อ 4 ทำงานได้เพราะ Gateway เป็น Eureka Client ด้วย (ทบทวน Part 92)
     * เมื่อ Gateway ต้อง route ไปที่ "order-service" มันจะถาม Eureka หา instance จริงที่ยังมีชีวิตอยู่
     * แล้วเลือก instance หนึ่ง (client-side load balancing) ก่อนส่ง request ต่อ
     *
     * spring.cloud.gateway.discovery.locator.enabled=true ยังทำให้ Gateway สร้าง route อัตโนมัติ
     * ให้ทุก service ที่ลงทะเบียนกับ Eureka โดยไม่ต้องเขียน route config เองทีละตัว (convention-based)
     */
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียน route configuration (YAML) สำหรับ `customer-service` ที่รับ
path `/api/customers/**` พร้อม filter ที่เพิ่ม request header
`X-Request-Source: mobile-app`

**เฉลย:**
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: customer-service-route
          uri: lb://customer-service
          predicates:
            - Path=/api/customers/**
          filters:
            - AddRequestHeader=X-Request-Source, mobile-app
```

**2)** อธิบายว่าทำไมการตรวจสอบ JWT ที่ Gateway (แทนที่แต่ละ microservice
ตรวจสอบเอง) ช่วยลดความซับซ้อนของระบบ

**เฉลย**: ถ้าทุก microservice (`OrderService`, `InventoryService`,
`PaymentService`, ...) ต้อง**ตรวจสอบ JWT เอง** จะเกิด**code ที่ทำหน้าที่
เดียวกันซ้ำซ้อนในหลาย service** (ทบทวนหลักการ DRY จาก Part 57) — ทุก
service ต้องมี dependency กับ JWT library, ต้องรู้ secret key, และต้อง
ดูแล logic การ verify token ให้ sync กันตลอดเวลา (ถ้าแก้ logic ที่หนึ่ง
ต้องแก้ทุก service) การตรวจสอบที่ **Gateway เพียงจุดเดียว** (หัวข้อ 6)
ทำให้**logic การ authentication อยู่ที่เดียว** — service ภายในแค่**เชื่อ
header ที่ Gateway ยืนยันมาให้แล้ว** (เช่น `X-Authenticated-User`) ลด
ความซับซ้อนและจุดที่อาจเกิด bug ด้านความปลอดภัยลงอย่างมาก และทำให้
เปลี่ยนกลไก authentication ในอนาคต (เช่น เปลี่ยนจาก JWT เป็น OAuth2) ทำ
ที่ Gateway จุดเดียวโดยไม่ต้องแก้ทุก service

**3)** อธิบายว่าทำไม API Gateway ควรทำ Rate Limiting เอง แทนที่จะให้แต่ละ
microservice ทำ Rate Limiting ของตัวเอง

**เฉลย**: Rate Limiting มีจุดประสงค์เพื่อ**จำกัดจำนวน request ทั้งหมดจาก
client ภายนอกหนึ่งราย** (ทบทวนหัวข้อ 7, Part 85) — ถ้าทำที่แต่ละ
microservice แยกกัน แต่ละ service จะเห็นแค่**ส่วนหนึ่ง**ของ request
ทั้งหมดที่ client นั้นส่งมา (เพราะ client เดียวอาจเรียกหลาย service ใน
การทำงานหนึ่งครั้ง) ทำให้**นับจำนวน request ของ client นั้นได้ไม่ถูกต้อง
ครบถ้วน** และ client ที่ถูกบล็อกจาก rate limit ที่ service หนึ่งอาจยัง
เรียก service อื่นได้ตามปกติ (ไม่ได้ถูกจำกัดจริง) การทำ Rate Limiting
**ที่ Gateway** ซึ่งเป็น**จุดเดียวที่ทุก request จากภายนอกต้องผ่าน**
(หัวข้อ 2) ทำให้เห็น**ภาพรวมทั้งหมดของ request จาก client นั้น**และ
สามารถจำกัดได้อย่างถูกต้องและครบถ้วนในจุดเดียว

### สรุปเนื้อหา Part 93

- API Gateway เป็นจุดเข้าเดียวที่ client เห็น ซ่อนโครงสร้าง microservices
  ภายในทั้งหมด
- Spring Cloud Gateway ใช้ Route + Predicate + Filter กำหนดว่า request
  แบบไหนไปที่ service ไหน และปรับแต่งอย่างไรก่อน/หลัง
- Cross-cutting concerns (authentication, rate limiting, logging) ควร
  ทำที่ Gateway เพียงจุดเดียว ลดความซ้ำซ้อนข้าม service
- `lb://service-name` ทำงานร่วมกับ Eureka Service Discovery (Part 92)
  เพื่อ route ไปยัง instance จริงพร้อม load balancing
- Rate Limiting ที่ Gateway เห็นภาพรวมของ request จาก client แต่ละราย
  ได้ถูกต้องกว่าทำแยกที่แต่ละ service
- Circuit Breaker ที่ Gateway ป้องกันการรอ timeout ซ้ำ ๆ เมื่อ service
  ภายในมีปัญหา (รายละเอียดเต็มใน Part 94)

**ต่อไป**: [Part 94 — Config Server และ Circuit Breaker (Resilience4j)](./part-094-config-server-circuit-breaker.md)
