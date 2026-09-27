# Part 74: Spring Core: Dependency Injection, Bean Scopes, Configuration

> ขั้นตอนที่ 731-740 ของหลักสูตร | ระดับ: สูงมาก

## สารบัญ

1. 3 วิธีของ Dependency Injection
2. Constructor Injection: วิธีที่แนะนำที่สุด
3. Field Injection: ทำไมไม่แนะนำ
4. Setter Injection: เมื่อไรเหมาะสม
5. `@Qualifier`: เมื่อมี Bean ชนิดเดียวกันหลายตัว
6. `@Primary`: กำหนด Bean เริ่มต้น
7. Bean Scopes แบบละเอียด
8. Circular Dependency ใน Spring
9. `@Value` และ Externalized Configuration
10. แบบฝึกหัดและสรุป

---

## 1. 3 วิธีของ Dependency Injection

ทบทวนจาก Part 73: Spring รองรับการฉีด dependency 3 วิธี — Part นี้จะเจาะลึก
แต่ละวิธีและเหตุผลว่าทำไมวิธีหนึ่งดีกว่าอีกวิธีในสถานการณ์ต่าง ๆ

## 2. Constructor Injection: วิธีที่แนะนำที่สุด

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class OrderService {
    private final PaymentGateway paymentGateway; // final! (ทบทวน immutability จาก Part 17)
    private final EmailService emailService;

    @Autowired // Spring 4.3+ ไม่ต้องเขียนถ้ามี constructor เดียว แต่เขียนไว้ชัดเจนก็ดี
    public OrderService(PaymentGateway paymentGateway, EmailService emailService) {
        this.paymentGateway = paymentGateway;
        this.emailService = emailService;
    }
}
```

**ทำไม Constructor Injection ดีที่สุด**:
1. **รองรับ `final` field**: object อยู่ในสถานะที่ถูกต้องสมบูรณ์ทันทีที่
   สร้าง (immutable dependency — ทบทวนหลัก immutability จาก Part 17)
2. **ป้องกัน `NullPointerException`**: ถ้า dependency จำเป็น ต้องระบุผ่าน
   constructor เสมอ ไม่มีทาง "ลืม" set ทีหลัง (ทบทวนปัญหา null จาก Part 43)
3. **ทดสอบง่ายที่สุด**: สร้าง object เพื่อทดสอบได้โดยไม่ต้องพึ่ง Spring
   container เลย — เขียน unit test ด้วยมือ `new OrderService(mockGateway,
   mockEmail)` ได้ตรง ๆ (ทบทวนจาก Part 58-59)
4. **บอก circular dependency ได้ตั้งแต่ startup** (หัวข้อ 8)

## 3. Field Injection: ทำไมไม่แนะนำ

```java
@Service
public class OrderServiceFieldInjection {
    @Autowired
    private PaymentGateway paymentGateway; // ฉีดตรงเข้า field เลย (ไม่ผ่าน constructor)

    @Autowired
    private EmailService emailService;
}
```

**ข้อเสียของ Field Injection** (แม้จะดู**สั้นและง่ายที่สุด**):
1. **field ต้อง non-final** เพราะ Spring set ค่าหลังสร้าง object แล้ว (ทบทวน
   ข้อจำกัด `final` จาก Part 17) — เสียโอกาสได้ immutability
2. **ทดสอบยากกว่า**: ต้องใช้ reflection (ทบทวนจาก Part 53) หรือ Spring test
   context เพื่อฉีด mock เข้า field ที่เป็น private
3. **ซ่อน dependency ที่มากเกินไป**: ถ้า class มี dependency 10 ตัวผ่าน
   field injection จะไม่มี "สัญญาณเตือน" ที่ชัดเจน (constructor ที่มี
   parameter 10 ตัวจะทำให้เห็นปัญหาทันที — ทบทวนแนวคิด SRP จาก Part 57)

**ข้อสรุป**: หลีกเลี่ยง Field Injection ในโค้ดใหม่ ยกเว้นในโค้ด test บางกรณี
ที่ความสะดวกสำคัญกว่า (แม้แต่ตรงนี้ก็ยังมีทางเลือกที่ดีกว่า)

## 4. Setter Injection: เมื่อไรเหมาะสม

```java
@Service
public class NotificationService {
    private EmailService emailService; // required dependency (ควรใช้ constructor injection)
    private SmsService smsService;      // optional dependency (เหมาะกับ setter injection)

    public NotificationService(EmailService emailService) {
        this.emailService = emailService;
    }

    @Autowired(required = false) // optional dependency
    public void setSmsService(SmsService smsService) {
        this.smsService = smsService;
    }
}
```

**ใช้ Setter Injection เมื่อ dependency เป็น optional** (ไม่จำเป็นต้องมี
เสมอ) — ผสมกับ Constructor Injection สำหรับ dependency ที่จำเป็น (required)
เพื่อได้ประโยชน์ทั้งสองแบบ

## 5. `@Qualifier`: เมื่อมี Bean ชนิดเดียวกันหลายตัว

ทบทวนจาก Part 16: ถ้ามี interface เดียวกันหลาย implementation Spring จะ
**สับสน**ว่าควรฉีดตัวไหน — `@Qualifier` ระบุให้ชัดเจน

```java
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Component;

interface PaymentGateway { void charge(double amount); }

@Component("creditCardGateway")
class CreditCardGateway implements PaymentGateway {
    public void charge(double amount) { System.out.println("จ่ายผ่านบัตรเครดิต: " + amount); }
}

@Component("paypalGateway")
class PaypalGateway implements PaymentGateway {
    public void charge(double amount) { System.out.println("จ่ายผ่าน PayPal: " + amount); }
}

@Component
class CheckoutService {
    private final PaymentGateway gateway;

    public CheckoutService(@Qualifier("paypalGateway") PaymentGateway gateway) {
        // ระบุชัดเจนว่าต้องการ bean ชื่อ "paypalGateway" ตัวใดตัวหนึ่งเท่านั้น
        this.gateway = gateway;
    }
}
```

**ถ้าไม่ระบุ `@Qualifier`** และมี bean มากกว่า 1 ตัวที่ตรงกับ type ที่ต้องการ
Spring จะ throw `NoUniqueBeanDefinitionException` ตอน startup (ดีกว่าการ
เดาผิดแล้วมี bug แบบเงียบ ๆ — ทบทวนหลักการ "fail fast" ที่สอดคล้องกับ static
typing จาก Part 3)

## 6. `@Primary`: กำหนด Bean เริ่มต้น

```java
import org.springframework.context.annotation.Primary;
import org.springframework.stereotype.Component;

@Component
@Primary // ถ้าไม่ระบุ @Qualifier ชัดเจน ให้ใช้ตัวนี้เป็นค่าเริ่มต้น
class CreditCardGateway implements PaymentGateway {
    public void charge(double amount) { System.out.println("จ่ายผ่านบัตรเครดิต (default): " + amount); }
}

@Component
class PaypalGateway implements PaymentGateway {
    public void charge(double amount) { System.out.println("จ่ายผ่าน PayPal: " + amount); }
}
```

`@Primary` ทำงานคล้าย **default case ใน switch expression** (ทบทวนจาก Part
5, 51) — ใช้เมื่อกรณีส่วนใหญ่ต้องการ implementation ตัวเดียว แต่บางที่อาจ
ต้องการตัวอื่นแทน (ใช้ `@Qualifier` เพื่อ override เฉพาะจุด)

## 7. Bean Scopes แบบละเอียด

ทบทวนและขยายจาก Part 73:

```java
import org.springframework.context.annotation.Scope;
import org.springframework.stereotype.Component;

@Component
@Scope("singleton") // ค่า default: instance เดียวทั้ง container (ทบทวน Part 54)
class ConfigService { }

@Component
@Scope("prototype") // instance ใหม่ทุกครั้งที่ขอ bean นี้
class ReportGenerator { }

@Component
@Scope("request") // instance ใหม่ต่อ HTTP request (สำหรับ web application - ทบทวน Part 71)
class RequestContextHolder { }

@Component
@Scope("session") // instance ใหม่ต่อ HTTP session (ทบทวน Session จาก Part 71-72)
class UserPreferences { }
```

| Scope | Instance กี่ตัว | ใช้เมื่อ |
|---|---|---|
| `singleton` (default) | 1 ตัวทั้ง container | Stateless service ส่วนใหญ่ (ไม่มี mutable state) |
| `prototype` | ใหม่ทุกครั้งที่ขอ | Object ที่มี state เฉพาะแต่ละการใช้งาน |
| `request` | ใหม่ต่อ HTTP request | ข้อมูลที่เกี่ยวข้องกับ request เดียว (web only) |
| `session` | ใหม่ต่อ HTTP session | ข้อมูลผู้ใช้ที่ต้องคงอยู่ข้าม request (web only) |

**ข้อควรระวังสำคัญ**: ทบทวนปัญหา thread-safety จาก Part 46-47, 72 —
**singleton bean ที่มี mutable instance field เสี่ยง race condition**
เหมือนกับ Servlet (Part 72) เพราะ instance เดียวถูกใช้จากหลาย thread
พร้อมกัน (แต่ละ HTTP request อาจรันบนคนละ thread — ทบทวนจาก Part 48)

## 8. Circular Dependency ใน Spring

ทบทวนปัญหาจาก Part 20 (circular dependency ระหว่าง package): Spring ก็เจอ
ปัญหานี้ได้ถ้า bean A ต้องการ B และ B ต้องการ A

```java
@Component
class ServiceA {
    private final ServiceB serviceB;
    public ServiceA(ServiceB serviceB) { this.serviceB = serviceB; }
}

@Component
class ServiceB {
    private final ServiceA serviceA; // circular! A ต้องการ B, B ต้องการ A
    public ServiceB(ServiceA serviceA) { this.serviceA = serviceA; }
}
```

ด้วย **Constructor Injection** Spring จะ**ตรวจจับปัญหานี้ทันทีตอน
startup** และ throw `BeanCurrentlyInCreationException` — **นี่คือข้อดีอีก
ข้อของ Constructor Injection** (ทบทวนจากหัวข้อ 2): ปัญหานี้เกิดขึ้นแบบ
"เงียบ ๆ" ได้ถ้าใช้ Field Injection (Spring แก้ปัญหาด้วยการสร้าง proxy
half-baked object ชั่วคราว ซึ่งอาจนำไปสู่ bug ที่ตรวจจับยากในภายหลัง)

**วิธีแก้ circular dependency**: ทบทวนจาก Part 57 (DIP) — มักบ่งบอกว่า
design มีปัญหา ควร**แยก responsibility ใหม่**หรือดึง logic ที่ใช้ร่วมกัน
ออกมาเป็น class/interface ที่สาม

## 9. `@Value` และ Externalized Configuration

```properties
# application.properties (ทบทวน key-value config จาก Part 63 - logback.xml concept)
app.name=My E-Commerce App
app.max-items-per-order=10
payment.gateway.api-key=sk_test_12345
```

```java
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

@Component
public class AppConfig {

    @Value("${app.name}") // อ่านค่าจาก application.properties
    private String appName;

    @Value("${app.max-items-per-order:5}") // ค่า default = 5 ถ้าไม่มี key นี้ใน properties
    private int maxItemsPerOrder;

    public void printConfig() {
        System.out.println(appName + " รองรับสูงสุด " + maxItemsPerOrder + " รายการต่อออร์เดอร์");
    }
}
```

**ประโยชน์**: แยก**configuration ที่เปลี่ยนตาม environment** (dev, staging,
production) ออกจาก**โค้ดที่ compile แล้ว** — เปลี่ยนค่าได้โดยไม่ต้อง
recompile (ทบทวนหลักการเดียวกับ `logback.xml` จาก Part 63) — Part 76 จะสอน
`@ConfigurationProperties` ซึ่งเป็นวิธีที่ดีกว่า `@Value` สำหรับ config
กลุ่มใหญ่

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** แปลง `NotificationService` จากหัวข้อ 4 (ที่ใช้ Field Injection) ให้
ใช้ Constructor Injection สำหรับ required dependency ทั้งคู่

**เฉลย:**

```java
@Service
public class NotificationService {
    private final EmailService emailService;
    private final SmsService smsService;

    public NotificationService(EmailService emailService, SmsService smsService) {
        this.emailService = emailService;
        this.smsService = smsService;
    }
}
```

**2)** สร้าง 2 bean ที่ implement `Logger` interface เดียวกัน (`ConsoleLogger`,
`FileLogger`) แล้วใช้ `@Primary` กับ `ConsoleLogger` และ `@Qualifier` เพื่อ
เลือก `FileLogger` ในบางที่

**เฉลย:**

```java
interface Logger { void log(String msg); }

@Component
@Primary
class ConsoleLogger implements Logger {
    public void log(String msg) { System.out.println("[Console] " + msg); }
}

@Component("fileLogger")
class FileLogger implements Logger {
    public void log(String msg) { System.out.println("[File] " + msg); }
}

@Component
class AuditService {
    private final Logger logger;
    public AuditService(@Qualifier("fileLogger") Logger logger) { this.logger = logger; }
}
```

**3)** อธิบายว่าทำไม Constructor Injection ตรวจจับ circular dependency ได้
ดีกว่า Field Injection

**เฉลย**: Constructor Injection บังคับให้ Spring**สร้าง object ให้เสร็จ
สมบูรณ์ทันที**ตอนเรียก constructor (ต้องมี dependency ทุกตัวพร้อมก่อนถึง
จะสร้างได้ — ทบทวนจาก Part 12) ถ้า A ต้องการ B และ B ต้องการ A พร้อมกัน
Spring จะติดอยู่ในวงจร "ต้องสร้าง A ก่อนถึงจะสร้าง B ได้ แต่ต้องสร้าง B ก่อน
ถึงจะสร้าง A ได้" ซึ่งเป็นไปไม่ได้ทางตรรกะ — Spring จึง throw exception
ทันทีตอน startup ในทางตรงกันข้าม Field Injection ทำให้ Spring สร้าง
**object เปล่า** (ยังไม่มี dependency) ก่อน แล้วค่อย set field ทีหลัง ทำให้
สามารถสร้าง A แบบ "ครึ่ง ๆ กลาง ๆ" ส่งให้ B ใช้ได้ (แม้ A ยังไม่มี B ของ
ตัวเองครบ) — ปัญหานี้อาจไม่ throw exception ทันที แต่ทำให้เกิด behavior
ที่ผิดเพี้ยนซึ่งตรวจจับได้ยากกว่ามากในภายหลัง

### สรุปเนื้อหา Part 74

- Constructor Injection แนะนำที่สุด: รองรับ `final`, ทดสอบง่าย, ตรวจจับ
  circular dependency ได้ตั้งแต่ startup
- Field Injection สะดวกแต่มีข้อเสียหลายอย่าง ไม่แนะนำในโค้ดใหม่
- Setter Injection เหมาะกับ optional dependency
- `@Qualifier` ระบุ bean ที่ต้องการเมื่อมีหลาย implementation, `@Primary`
  กำหนดค่า default
- Bean scope: `singleton` (default), `prototype`, `request`, `session` —
  ระวัง thread-safety กับ mutable state ใน singleton
- `@Value` อ่าน externalized configuration จาก properties file แยกจาก
  compiled code

**ต่อไป**: [Part 75 — Spring Boot เบื้องต้น: Auto-configuration, Starter](./part-075-spring-boot-intro.md)
