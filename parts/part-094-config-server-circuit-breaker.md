# Part 94: Config Server และ Circuit Breaker (Resilience4j)

> ขั้นตอนที่ 931-940 ของหลักสูตร | ระดับ: ระดับมืออาชีพ/โลก

## สารบัญ

1. ปัญหา: Configuration กระจัดกระจายในหลาย Microservice
2. Spring Cloud Config Server: Centralized Configuration
3. Config Client: ดึง Configuration จาก Config Server
4. Dynamic Refresh: เปลี่ยน Config โดยไม่ต้อง Restart
5. ปัญหา Cascading Failure ใน Distributed System
6. Circuit Breaker Pattern: แนวคิดหลัก
7. Resilience4j: ตั้งค่า Circuit Breaker
8. Retry Pattern และ Timeout
9. Bulkhead Pattern: การแยกทรัพยากร
10. แบบฝึกหัดและสรุป

---

## 1. ปัญหา: Configuration กระจัดกระจายในหลาย Microservice

ทบทวนจาก Part 76: `application.properties` ใช้จัดการ configuration ของ
แอปเดียว — ในระบบ microservices ที่มี**หลายสิบ service** แต่ละตัวมี
`application.properties` ของตัวเอง เกิดปัญหา:

```java
public class ScatteredConfigurationProblem {
    /*
     * ปัญหา:
     *   1. ค่า config ที่ใช้ร่วมกันหลาย service (เช่น database connection pool size, feature flag)
     *      ต้องแก้ไขในทุกไฟล์แยกกัน - ลืมแก้ตัวใดตัวหนึ่งได้ง่าย
     *   2. เปลี่ยน config ต้อง rebuild + redeploy service นั้นใหม่เสมอ (แม้แค่เปลี่ยนค่าตัวเลขตัวเดียว)
     *   3. ไม่มีที่เดียวให้ดู config ทั้งหมดของทุก service พร้อมกัน (ตรวจสอบยาก, audit ยาก)
     */
}
```

## 2. Spring Cloud Config Server: Centralized Configuration

**Config Server** เก็บ configuration ของ**ทุก service ไว้ที่เดียว**
(มักเก็บใน Git repository แยก) — service อื่นดึง config จากที่นี่ตอน
เริ่มทำงาน

```xml
<!-- pom.xml ของ config-server project -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.config.server.EnableConfigServer;

@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

```properties
# application.properties ของ config-server
server.port=8888
spring.cloud.config.server.git.uri=https://github.com/mycompany/config-repo
spring.cloud.config.server.git.default-label=main
```

```java
public class ConfigRepoStructureDemo {
    /*
     * ใน config-repo (Git repository แยกจาก code):
     *   order-service.properties       - config เฉพาะ order-service
     *   inventory-service.properties   - config เฉพาะ inventory-service
     *   application.properties         - config ที่ใช้ร่วมกันทุก service (shared config)
     *
     * ประโยชน์ของการเก็บใน Git: มี version history ของการเปลี่ยน config ทุกครั้ง (ทบทวนความสำคัญ
     * ของ version control) rollback ค่าที่ผิดพลาดได้ทันทีด้วย git revert
     */
}
```

## 3. Config Client: ดึง Configuration จาก Config Server

```xml
<!-- pom.xml ของ order-service -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```

```properties
# bootstrap.properties (หรือ application.properties ใน Spring Boot รุ่นใหม่) ของ order-service
spring.application.name=order-service
spring.config.import=configserver:http://localhost:8888
```

```java
public class ConfigClientFlowDemo {
    /*
     * ตอน order-service เริ่มทำงาน:
     *   1. อ่าน spring.application.name = "order-service"
     *   2. ยิง request ไปที่ Config Server: GET http://localhost:8888/order-service/default
     *   3. Config Server อ่าน order-service.properties + application.properties (shared) จาก Git
     *      แล้วรวมค่าทั้งหมดส่งกลับมาเป็น JSON
     *   4. order-service ใช้ค่าเหล่านี้เป็น configuration ของตัวเอง (เหมือนอ่านจากไฟล์ local)
     */
}
```

## 4. Dynamic Refresh: เปลี่ยน Config โดยไม่ต้อง Restart

```java
import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RefreshScope // ทำให้ bean นี้ "รีเฟรช" ค่าใหม่ได้โดยไม่ต้อง restart แอปทั้งตัว
public class FeatureFlagController {
    @Value("${feature.new-checkout-flow.enabled:false}")
    private boolean newCheckoutFlowEnabled;

    @GetMapping("/api/features")
    public String checkFeature() {
        return "New checkout flow enabled: " + newCheckoutFlowEnabled;
    }
}
```

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```java
public class DynamicRefreshFlowDemo {
    /*
     * 1. แก้ไข "feature.new-checkout-flow.enabled=true" ใน Git config-repo แล้ว push
     * 2. เรียก POST http://order-service/actuator/refresh (Spring Boot Actuator endpoint)
     * 3. Bean ที่มี @RefreshScope จะถูกสร้างใหม่ด้วยค่า config ล่าสุด - ไม่ต้อง restart service เลย!
     *
     * มีประโยชน์มากสำหรับ feature flag ที่ต้องการเปิด/ปิด feature แบบ real-time
     * โดยไม่กระทบผู้ใช้ที่กำลังใช้งานอยู่ (ทบทวนความสำคัญของ zero-downtime deployment)
     */
}
```

## 5. ปัญหา Cascading Failure ใน Distributed System

ทบทวนจาก Part 91: ถ้า `InventoryService` ช้าหรือ down และ `OrderService`
เรียกแบบ **synchronous** (รอผลลัพธ์) โดยไม่มีการป้องกัน จะเกิด**ปฏิกิริยา
ลูกโซ่**:

```java
public class CascadingFailureDemo {
    /*
     * 1. InventoryService เริ่มช้า (database ตัวมันเองมีปัญหา) - แต่ละ request ใช้เวลา 30 วินาที
     * 2. OrderService ที่เรียก InventoryService ก็ต้อง "รอ" นานขึ้นตามไปด้วย
     * 3. Thread pool ของ OrderService ถูกใช้จนหมด (ทุก thread กำลังรอ InventoryService)
     * 4. OrderService ไม่มี thread เหลือรับ request ใหม่ -> OrderService "ดูเหมือนตาย" ไปด้วย!
     * 5. Service อื่นที่เรียก OrderService ก็เจอปัญหาเดียวกันต่อเป็นทอด ๆ (cascading)
     *
     * ผลลัพธ์: ปัญหาเล็กที่ InventoryService ลุกลามจนระบบทั้งหมดล่ม (แม้ Order/Payment/Customer
     * Service ไม่มี bug อะไรเลยก็ตาม)
     */
}
```

## 6. Circuit Breaker Pattern: แนวคิดหลัก

ทบทวนแนวคิดวงจรไฟฟ้า (circuit breaker ในบ้าน): เมื่อไฟฟ้าลัดวงจร
**เบรกเกอร์ตัดไฟทันที** เพื่อป้องกันความเสียหายลุกลาม — Software Circuit
Breaker ทำงานคล้ายกัน

```
CLOSED (ปกติ) ──ล้มเหลวเกิน threshold──▶ OPEN (ตัดการเรียก)
     ▲                                         │
     │                                    รอสักพัก (wait duration)
     │                                         ▼
     └──── สำเร็จ ──── HALF_OPEN (ทดลองเรียกดูบางส่วน)
```

- **CLOSED**: ทำงานปกติ ปล่อยให้ request ผ่านไปเรียก service จริง
- **OPEN**: หลังล้มเหลวเกินเกณฑ์ที่กำหนด → **หยุดเรียก service นั้นทันที**
  (ไม่ต้องรอ timeout ทุกครั้ง) คืนค่า fallback ทันที
- **HALF_OPEN**: หลังรอสักพัก ทดลองปล่อย request บางส่วนผ่านไปดูว่า
  service กลับมาทำงานปกติหรือยัง

## 7. Resilience4j: ตั้งค่า Circuit Breaker

```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
</dependency>
```

```properties
resilience4j.circuitbreaker.instances.inventoryService.sliding-window-size=10
resilience4j.circuitbreaker.instances.inventoryService.failure-rate-threshold=50
resilience4j.circuitbreaker.instances.inventoryService.wait-duration-in-open-state=10s
resilience4j.circuitbreaker.instances.inventoryService.permitted-number-of-calls-in-half-open-state=3
```

```java
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import org.springframework.stereotype.Service;

@Service
public class InventoryServiceClient {
    private final RestTemplate restTemplate;

    public InventoryServiceClient(RestTemplate restTemplate) { this.restTemplate = restTemplate; }

    @CircuitBreaker(name = "inventoryService", fallbackMethod = "checkStockFallback")
    public boolean checkStock(Long productId, int quantity) {
        String url = "http://inventory-service/api/inventory/check?productId=" + productId;
        return restTemplate.getForObject(url, Boolean.class);
        // ถ้าเรียกล้มเหลวเกิน 50% ใน 10 request ล่าสุด (sliding-window-size=10) -> circuit เปิด
    }

    // fallback method - ถูกเรียกทันทีเมื่อ circuit เป็น OPEN โดยไม่ต้องรอ timeout จริง
    public boolean checkStockFallback(Long productId, int quantity, Exception ex) {
        System.out.println("Circuit OPEN - inventory-service unavailable, using fallback for product " + productId);
        return false; // ค่า default ที่ปลอดภัย: สมมติว่าสินค้าไม่พร้อม แทนที่จะรอ timeout จริง
    }
}
```

**ผลลัพธ์**: เมื่อ `InventoryService` มีปัญหา, `checkStock()` จะ**ไม่รอ
timeout ซ้ำ ๆ ทุก request**อีกต่อไป — circuit เปิดแล้วเรียก
`checkStockFallback` **ทันที** ทำให้ **thread ของ OrderService ไม่ถูก
ครอบครองนาน** (แก้ปัญหา cascading failure จากหัวข้อ 5 ได้โดยตรง)

## 8. Retry Pattern และ Timeout

```properties
resilience4j.retry.instances.inventoryService.max-attempts=3
resilience4j.retry.instances.inventoryService.wait-duration=500ms
resilience4j.timelimiter.instances.inventoryService.timeout-duration=2s
```

```java
import io.github.resilience4j.retry.annotation.Retry;
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import org.springframework.stereotype.Service;

@Service
public class ResilientInventoryClient {
    @Retry(name = "inventoryService") // ลองใหม่อัตโนมัติถ้าล้มเหลว (เหมาะกับความล้มเหลวชั่วคราว - transient failure)
    @CircuitBreaker(name = "inventoryService", fallbackMethod = "fallback")
    public boolean checkStock(Long productId) {
        return callInventoryService(productId);
    }

    public boolean fallback(Long productId, Exception ex) { return false; }

    private boolean callInventoryService(Long productId) {
        return true; // เรียก service จริง
    }
    // ลำดับการทำงาน: Retry ครอบ CircuitBreaker - ลอง retry ก่อน ถ้า retry ครบแล้วยังล้มเหลว circuit ถึงจะนับความล้มเหลว
}
```

**ข้อควรระวัง**: **Retry ต้องใช้ร่วมกับ Idempotency** (ทบทวนจาก Part
85) — ถ้า operation ไม่ idempotent (เช่น "เก็บเงิน") การ retry ซ้ำอาจทำให้
เก็บเงินซ้ำสองครั้งโดยไม่ตั้งใจ! ต้องแน่ใจว่า operation ปลอดภัยต่อการเรียก
ซ้ำก่อนใส่ `@Retry`

## 9. Bulkhead Pattern: การแยกทรัพยากร

ทบทวนแนวคิดจากเรือ (bulkhead compartment ในเรือ): ถ้าห้องหนึ่งของเรือ
น้ำท่วม **ผนังกันน้ำ (bulkhead)** ป้องกันไม่ให้น้ำไหลไปห้องอื่น เรือยัง
ลอยได้

```properties
resilience4j.bulkhead.instances.inventoryService.max-concurrent-calls=10
resilience4j.bulkhead.instances.inventoryService.max-wait-duration=1s
```

```java
import io.github.resilience4j.bulkhead.annotation.Bulkhead;
import org.springframework.stereotype.Service;

@Service
public class BulkheadProtectedClient {
    @Bulkhead(name = "inventoryService") // จำกัด concurrent call ไปที่ inventory-service สูงสุด 10 พร้อมกัน
    public boolean checkStock(Long productId) {
        return true;
    }
    /*
     * ถ้าไม่มี Bulkhead: request จำนวนมากไปที่ InventoryService ที่มีปัญหา อาจใช้ thread
     * ของ OrderService ทั้งหมด (เหมือนหัวข้อ 5) ทำให้ endpoint อื่นของ OrderService ที่ไม่เกี่ยว
     * กับ InventoryService เลยก็ได้รับผลกระทบไปด้วย (เพราะ thread pool ใช้ร่วมกัน)
     *
     * Bulkhead จำกัดจำนวน concurrent call ไปยัง dependency แต่ละตัวแยกกัน - เหมือนแบ่งห้องเรือ
     * ถ้า InventoryService มีปัญหา กระทบแค่ "ห้อง" ของมัน ไม่ลามไปกระทบ endpoint อื่นที่ไม่เกี่ยวข้อง
     */
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เพิ่ม `@CircuitBreaker` ให้เมธอด `chargePayment` ที่เรียก
`PaymentService` พร้อม fallback ที่คืนค่าสถานะ "PENDING_RETRY"

**เฉลย:**
```java
@CircuitBreaker(name = "paymentService", fallbackMethod = "chargeFallback")
public PaymentResult chargePayment(ChargeRequest request) {
    return paymentServiceClient.charge(request);
}

public PaymentResult chargeFallback(ChargeRequest request, Exception ex) {
    return new PaymentResult("PENDING_RETRY", null);
    // แจ้งผู้ใช้ว่าระบบจะลองเก็บเงินใหม่ทีหลัง แทนที่จะรอ timeout หรือแสดง error ทันที
}
```

**2)** อธิบายว่าทำไม Circuit Breaker ช่วยป้องกัน Cascading Failure ได้
ดีกว่าการปล่อยให้ request ล้มเหลว/timeout ตามปกติ

**เฉลย**: โดยปกติเมื่อ dependency (เช่น `InventoryService`) มีปัญหา
**ทุก request ที่เรียกไปยังมันจะต้องรอจนกว่าจะ timeout จริง** (เช่น 30
วินาที) ก่อนที่จะรู้ว่าล้มเหลว — ถ้ามี request จำนวนมากเรียกไปพร้อมกัน
**thread จำนวนมากจะถูก "ผูกไว้" รอ timeout นานหลายวินาทีพร้อมกันหมด**
(ทบทวนหัวข้อ 5) ทำให้ thread pool ของ service ที่เรียกหมดลงอย่างรวดเร็ว
และไม่มี thread เหลือรับ request อื่น ๆ เลย (แม้จะไม่เกี่ยวกับ
InventoryService ก็ตาม) Circuit Breaker แก้ปัญหานี้โดย**ตรวจจับ pattern
ความล้มเหลวที่เกิดซ้ำ ๆ**และ**เปลี่ยนสถานะเป็น OPEN**ทันทีที่เกินเกณฑ์ —
หลังจากนั้น**ทุก request ที่ตามมาจะได้รับ fallback ทันทีโดยไม่ต้องรอ
timeout จริงเลย** ทำให้ thread ไม่ถูกครอบครองนาน ๆ และระบบยังมีทรัพยากร
เหลือรองรับ request อื่นที่ไม่เกี่ยวข้องกับ dependency ที่มีปัญหา

**3)** อธิบายว่าทำไมการใส่ `@Retry` กับ operation "เก็บเงินจากบัตร
เครดิต" โดยไม่ระมัดระวังอาจเป็นอันตราย และควรแก้ไขอย่างไร

**เฉลย**: `@Retry` (หัวข้อ 8) ทำงานโดย**เรียก method เดิมซ้ำโดยอัตโนมัติ
เมื่อเกิดข้อผิดพลาด** — ถ้า operation "เก็บเงินจากบัตรเครดิต" **ไม่
idempotent** (ทบทวนแนวคิด idempotency จาก Part 85) การ retry อาจเกิด
สถานการณ์ที่**การเรียกครั้งแรกสำเร็จจริงที่ธนาคารแล้ว แต่ response กลับมา
ช้าหรือขาดหายไปเพราะ network timeout** — ระบบเข้าใจผิดว่าล้มเหลวและ
**retry เก็บเงินซ้ำเป็นครั้งที่สอง** ทำให้ลูกค้าถูกเก็บเงินสองครั้งจาก
คำสั่งซื้อเดียว วิธีแก้คือใช้ **Idempotency Key** (ทบทวนจาก Part 85) ส่ง
ไปพร้อมกับทุก request รวมถึง request ที่ retry — ฝั่ง `PaymentService`
จะตรวจสอบว่า key นี้เคยประมวลผลไปแล้วหรือยัง ถ้าเคยแล้วจะคืนผลลัพธ์เดิม
กลับไปโดยไม่เก็บเงินซ้ำ ทำให้ `@Retry` ปลอดภัยที่จะใช้ได้แม้กับ operation
ที่มีผลกระทบทางการเงิน

### สรุปเนื้อหา Part 94

- Spring Cloud Config Server เก็บ configuration ของทุก microservice ไว้
  ที่เดียว (มักเก็บใน Git) แก้ปัญหา config กระจัดกระจาย
- `@RefreshScope` + Actuator `/refresh` endpoint ทำให้เปลี่ยน config ได้
  โดยไม่ต้อง restart service
- Cascading Failure เกิดจาก dependency ที่มีปัญหาทำให้ thread ของ service
  ที่เรียกถูกครอบครองจนหมด ลุกลามไปทั้งระบบ
- Circuit Breaker (CLOSED/OPEN/HALF_OPEN) ตรวจจับความล้มเหลวซ้ำ ๆ และ
  ตัดการเรียกทันทีโดยไม่ต้องรอ timeout จริง
- Resilience4j ให้ `@CircuitBreaker`, `@Retry`, `@Bulkhead` annotation
  จัดการ resilience ได้สะดวก
- Retry ต้องใช้ร่วมกับ Idempotency Key เสมอสำหรับ operation ที่ไม่
  idempotent (เช่น การเก็บเงิน)
- Bulkhead แยกทรัพยากร (thread/connection) สำหรับ dependency แต่ละตัว
  ป้องกันไม่ให้ปัญหาของ dependency หนึ่งลุกลามไปกระทบส่วนอื่น

**ต่อไป**: [Part 95 — Docker: Containerization](./part-095-docker-containerization.md)
