# Part 92: Spring Cloud: Service Discovery ด้วย Eureka

> ขั้นตอนที่ 911-920 ของหลักสูตร | ระดับ: ระดับมืออาชีพ/โลก

## สารบัญ

1. ปัญหา: Service ต้องรู้ที่อยู่ (IP/Port) ของ Service อื่นอย่างไร
2. Service Discovery คืออะไร: แนวคิดหลัก
3. Eureka Server: ตั้งค่า Service Registry
4. Eureka Client: การลงทะเบียน Service
5. การเรียก Service อื่นผ่านชื่อ (ไม่ใช่ IP) ด้วย `RestTemplate` + `@LoadBalanced`
6. OpenFeign: เรียก Service อื่นแบบ Declarative
7. Client-side Load Balancing
8. Health Check และการลบ Instance ที่ตายแล้ว
9. High Availability ของ Eureka Server เอง
10. แบบฝึกหัดและสรุป

---

## 1. ปัญหา: Service ต้องรู้ที่อยู่ (IP/Port) ของ Service อื่นอย่างไร

ทบทวนจาก Part 91: `OrderService` ต้องเรียก `InventoryService` ผ่าน HTTP
— แต่ **`InventoryService` อยู่ที่ IP ไหน? Port อะไร?**

```java
public class HardcodedServiceUrlProblem {
    /*
     * วิธีง่ายที่สุด (แต่มีปัญหามาก):
     *   String url = "http://192.168.1.50:8081/api/inventory/check";
     *
     * ปัญหา:
     *   1. ถ้า InventoryService restart แล้วได้ IP ใหม่ (เช่นใน Docker/Kubernetes - ปูทางสู่ Part 95, 96)
     *      -> URL ที่ hardcode ไว้ใช้ไม่ได้อีกต่อไป ต้องแก้ config ทุกที่ที่เรียก
     *   2. ถ้า scale InventoryService เป็น 5 instance (เพื่อรองรับโหลด) จะเรียก instance ไหน?
     *      ต้องมีวิธี "กระจาย" การเรียกไปยังหลาย instance เอง (load balancing)
     *   3. ถ้า instance หนึ่งตาย ต้องรู้ว่าตัวไหนยังอยู่ ตัวไหนตายไปแล้ว เพื่อไม่เรียกไปที่ตัวที่ตาย
     */
}
```

## 2. Service Discovery คืออะไร: แนวคิดหลัก

**Service Discovery** แก้ปัญหานี้ด้วย **Service Registry** — ตัวกลางที่
เก็บ**รายชื่อ instance ที่กำลังทำงานอยู่ทั้งหมด**ของทุก service พร้อม
ที่อยู่ปัจจุบัน

```
1. InventoryService instance เริ่มทำงาน -> ลงทะเบียนตัวเองกับ Eureka Server
   ("ฉันคือ inventory-service, อยู่ที่ 192.168.1.50:8081")

2. OrderService ต้องการเรียก InventoryService -> ถาม Eureka Server
   ("inventory-service อยู่ที่ไหนบ้าง?")

3. Eureka Server ตอบกลับรายชื่อ instance ทั้งหมดที่ยังมีชีวิตอยู่
   ("มี 3 instance: 192.168.1.50:8081, 192.168.1.51:8081, 192.168.1.52:8081")

4. OrderService เลือก instance หนึ่ง (load balancing - หัวข้อ 7) แล้วเรียกไปตรง ๆ
```

## 3. Eureka Server: ตั้งค่า Service Registry

```xml
<!-- pom.xml ของ eureka-server project -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
@EnableEurekaServer // ทำให้แอปนี้เป็น Service Registry เอง (ทบทวน @Enable* จาก Part 74, 87, 90)
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

```properties
# application.properties ของ eureka-server
server.port=8761
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
# eureka server ตัวนี้ไม่ต้องลงทะเบียนตัวเองหรือดึงข้อมูลจาก registry อื่น (มันคือ registry เอง)
```

## 4. Eureka Client: การลงทะเบียน Service

```xml
<!-- pom.xml ของ order-service project -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

```properties
# application.properties ของ order-service
spring.application.name=order-service
eureka.client.service-url.defaultZone=http://localhost:8761/eureka/
```

```java
@SpringBootApplication
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
        // เมื่อสตาร์ท, spring-cloud-starter-netflix-eureka-client จะลงทะเบียน
        // "order-service" กับ Eureka Server ที่ localhost:8761 โดยอัตโนมัติ
        // (ไม่ต้องเขียน code เพิ่ม - แค่มี dependency + config ก็เพียงพอ)
    }
}
```

## 5. การเรียก Service อื่นผ่านชื่อ (ไม่ใช่ IP) ด้วย `RestTemplate` + `@LoadBalanced`

```java
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestTemplate;

@Configuration
public class RestTemplateConfig {
    @Bean
    @LoadBalanced // ทำให้ RestTemplate นี้รู้จัก resolve ชื่อ service ผ่าน Eureka อัตโนมัติ
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

```java
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

@Service
public class InventoryServiceClient {
    private final RestTemplate restTemplate;

    public InventoryServiceClient(RestTemplate restTemplate) { this.restTemplate = restTemplate; }

    public boolean checkStock(Long productId, int quantity) {
        // ใช้ "inventory-service" (ชื่อจาก spring.application.name) แทน IP:Port ตรง ๆ!
        String url = "http://inventory-service/api/inventory/check?productId=" + productId + "&qty=" + quantity;
        return restTemplate.getForObject(url, Boolean.class);
        // @LoadBalanced RestTemplate จะถาม Eureka หาที่อยู่จริงของ "inventory-service" ให้อัตโนมัติ
        // แล้วเลือก instance หนึ่ง (round-robin โดย default - หัวข้อ 7) ไปเรียกจริง
    }
}
```

## 6. OpenFeign: เรียก Service อื่นแบบ Declarative

`RestTemplate` ยังต้องเขียน URL string เอง — **OpenFeign** ทำให้เรียก
service อื่นเหมือนเรียก**เมธอด Java ธรรมดา** (ทบทวนแนวคิด declarative
programming ที่เกี่ยวข้องกับ annotation-driven จาก Part 52)

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

```java
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.openfeign.EnableFeignClients;

@SpringBootApplication
@EnableFeignClients // เปิดใช้งาน Feign Client ทั้งแอป
public class OrderServiceApplication { /* ... */ }
```

```java
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;

@FeignClient(name = "inventory-service") // ใช้ชื่อ service จาก Eureka - ไม่ต้องรู้ IP เลย
public interface InventoryFeignClient {
    @GetMapping("/api/inventory/check") // เหมือนกับ @RequestMapping ฝั่ง InventoryService (ทบทวน Part 77)
    boolean checkStock(@RequestParam Long productId, @RequestParam int quantity);
}
```

```java
@Service
public class OrderService {
    private final InventoryFeignClient inventoryClient; // Spring inject implementation ให้อัตโนมัติ!

    public OrderService(InventoryFeignClient inventoryClient) { this.inventoryClient = inventoryClient; }

    public OrderDto createOrder(CreateOrderRequest request) {
        // เรียกเหมือนเมธอด Java ธรรมดา แต่จริง ๆ แล้วเป็น HTTP call ไปที่ inventory-service ผ่าน Eureka
        boolean inStock = inventoryClient.checkStock(request.productId(), request.quantity());
        if (!inStock) throw new InsufficientStockException(request.productId());
        return new OrderDto(1L, "PENDING");
    }
}
```

**ข้อสังเกต**: `InventoryFeignClient` เป็นแค่ **interface** — เราไม่เขียน
implementation เลย! Spring Cloud OpenFeign **สร้าง proxy class ให้
อัตโนมัติ** ที่แปลง method call เป็น HTTP request จริง (ทบทวนแนวคิด
Dynamic Proxy ที่เกี่ยวข้องกับ Reflection จาก Part 53)

## 7. Client-side Load Balancing

```java
public class ClientSideLoadBalancingDemo {
    /*
     * Server-side Load Balancing (แบบดั้งเดิม): มี Load Balancer กลาง (เช่น Nginx)
     *   client -> Load Balancer -> เลือก instance -> forward request
     *   client ไม่รู้ว่ามีกี่ instance เลย รู้แค่ที่อยู่ของ Load Balancer เดียว
     *
     * Client-side Load Balancing (ที่ @LoadBalanced/OpenFeign ใช้):
     *   client ถาม Eureka ได้รายชื่อ instance ทั้งหมดมาเก็บไว้ (cache)
     *   client เลือก instance เอง (default: round-robin) แล้วเรียกไปตรง ๆ โดยไม่ผ่าน load balancer กลาง
     *   ข้อดี: ไม่มี single point of failure ที่ load balancer กลาง, latency ต่ำกว่า (ไม่ต้อง hop ผ่านตัวกลาง)
     */
}
```

## 8. Health Check และการลบ Instance ที่ตายแล้ว

```properties
# application.properties ของ order-service (Eureka Client)
eureka.instance.lease-renewal-interval-in-seconds=10
eureka.instance.lease-expiration-duration-in-seconds=30
```

```java
public class EurekaHeartbeatDemo {
    /*
     * Eureka Client ส่ง "heartbeat" (สัญญาณว่ายังมีชีวิตอยู่) ไปที่ Eureka Server ทุก ๆ
     * lease-renewal-interval-in-seconds (default 30 วินาที)
     *
     * ถ้า Eureka Server ไม่ได้รับ heartbeat ภายใน lease-expiration-duration-in-seconds
     * (default 90 วินาที) -> ถือว่า instance นั้น "ตายแล้ว" -> ลบออกจาก registry
     *
     * ทำให้ client อื่นที่ถาม Eureka หา instance ของ service นี้ จะไม่ได้รับ instance ที่ตายแล้วกลับมา
     * (ป้องกันการเรียกไปที่ instance ที่ตายไปแล้วโดยไม่รู้ตัว)
     */
}
```

## 9. High Availability ของ Eureka Server เอง

```java
public class EurekaHighAvailabilityDemo {
    /*
     * ปัญหา: ถ้า Eureka Server มีตัวเดียวและมันตาย -> service discovery ทั้งระบบล่มไปด้วย!
     * (Eureka Server กลายเป็น single point of failure)
     *
     * วิธีแก้: รัน Eureka Server หลาย instance ที่ "ลงทะเบียนซึ่งกันและกัน" (peer awareness)
     *   eureka-server-1 config: eureka.client.service-url.defaultZone=http://eureka-server-2:8762/eureka/
     *   eureka-server-2 config: eureka.client.service-url.defaultZone=http://eureka-server-1:8761/eureka/
     *
     * ทั้งสอง instance sync ข้อมูล registry กันเอง - ถ้าตัวหนึ่งตาย client ยัง fallback ไปใช้ตัวที่เหลือได้
     * (ทบทวนแนวคิด High Availability ที่เกี่ยวข้องกับ redundancy - ปูทางสู่ Part 96, 99)
     */
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียน `@FeignClient` interface สำหรับเรียก `PaymentService` ที่มี
endpoint `POST /api/payments` รับ `ChargeRequest` คืนค่า `PaymentResult`

**เฉลย:**
```java
@FeignClient(name = "payment-service")
public interface PaymentFeignClient {
    @PostMapping("/api/payments")
    PaymentResult charge(@RequestBody ChargeRequest request);
}
```

**2)** อธิบายว่าทำไม hardcode IP address ของ service ในระบบ microservices
เป็นแนวทางที่ไม่ยั่งยืน โดยเฉพาะเมื่อระบบ deploy บน container orchestration
เช่น Kubernetes

**เฉลย**: ใน container orchestration platform อย่าง Kubernetes (ปูทางสู่
Part 96) container ของแต่ละ service instance **ถูกสร้างและทำลายอยู่
ตลอดเวลา** (เช่น เมื่อ scale up/down, restart หลัง crash, หรือ rolling
update ระหว่าง deploy เวอร์ชันใหม่) — **ทุกครั้งที่ container ใหม่ถูก
สร้าง มันจะได้ IP address ใหม่เสมอ** ถ้า hardcode IP ไว้ใน config หรือ
code ค่านั้นจะ**ใช้งานไม่ได้ทันทีที่ container restart** ทำให้ต้องแก้ไข
config ด้วยมือทุกครั้งซึ่งเป็นไปไม่ได้ในทางปฏิบัติเมื่อมี container
จำนวนมาก Service Discovery (หัวข้อ 2) แก้ปัญหานี้เพราะ **instance ใหม่
ลงทะเบียนตัวเองกับ Eureka โดยอัตโนมัติทุกครั้งที่เริ่มทำงาน** — service
อื่นที่ต้องการเรียกก็แค่ถาม Eureka ด้วย**ชื่อ service** (ที่ไม่เปลี่ยน)
แทนที่จะต้องรู้ IP ที่เปลี่ยนไปเรื่อย ๆ

**3)** อธิบายความแตกต่างระหว่าง Server-side Load Balancing และ
Client-side Load Balancing ที่ Eureka + `@LoadBalanced`/OpenFeign ใช้

**เฉลย**: **Server-side Load Balancing** มี**เครื่อง load balancer กลาง
แยกต่างหาก** (เช่น Nginx หรือ hardware load balancer) ที่รับ request
ทั้งหมดจาก client ก่อน แล้วจึง**เลือกและ forward ไปยัง instance ที่
เหมาะสม** — client รู้จักแค่ที่อยู่ของ load balancer กลางเพียงจุดเดียว
ไม่รู้เลยว่ามี instance กี่ตัวอยู่ข้างหลัง ในขณะที่ **Client-side Load
Balancing** (ที่ Eureka Client ใช้ผ่าน `@LoadBalanced` หรือ OpenFeign)
**client เองเป็นผู้ถาม Eureka ให้ได้รายชื่อ instance ทั้งหมดมาเก็บไว้ใน
เครื่องตัวเอง (cache)** จากนั้น **client เลือก instance เอง** (ด้วย
algorithm เช่น round-robin) **แล้วเรียกไปที่ instance นั้นตรง ๆ**โดยไม่
ผ่านตัวกลางใด ๆ ข้อดีคือไม่มี single point of failure ที่ load balancer
กลาง และ latency ต่ำกว่าเพราะไม่ต้อง "hop" ผ่านตัวกลางก่อนถึงปลายทางจริง

### สรุปเนื้อหา Part 92

- ปัญหาของการ hardcode IP/Port ระหว่าง service ในระบบ microservices ที่
  instance เกิด/ตายอยู่ตลอดเวลา
- Service Discovery ใช้ Service Registry (Eureka Server) เก็บรายชื่อ
  instance ที่มีชีวิตอยู่ของทุก service
- Eureka Client ลงทะเบียนตัวเองอัตโนมัติเมื่อสตาร์ท และส่ง heartbeat
  สม่ำเสมอเพื่อยืนยันว่ายังมีชีวิตอยู่
- `@LoadBalanced` RestTemplate และ OpenFeign ช่วยเรียก service อื่นผ่าน
  ชื่อ (service name) แทนการ hardcode IP
- OpenFeign ทำให้เรียก service อื่นแบบ declarative ผ่าน interface โดยไม่
  ต้องเขียน implementation เอง
- Client-side Load Balancing กระจาย request ไปหลาย instance โดยไม่ต้อง
  พึ่งพา load balancer กลาง
- Eureka Server ควร deploy หลาย instance (peer awareness) เพื่อป้องกัน
  single point of failure

**ต่อไป**: [Part 93 — API Gateway](./part-093-api-gateway.md)
