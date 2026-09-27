# Part 73: Introduction to Spring Framework, IoC Container

> ขั้นตอนที่ 721-730 ของหลักสูตร | ระดับ: สูงมาก

## สารบัญ

1. Spring Framework คืออะไร ทำไมครองโลก Java Enterprise
2. Inversion of Control (IoC) คืออะไร
3. IoC Container: `ApplicationContext`
4. Bean คืออะไร
5. การประกาศ Bean ด้วย XML (ประวัติศาสตร์ที่ควรรู้)
6. การประกาศ Bean ด้วย Java Config (`@Configuration`, `@Bean`)
7. Component Scanning: `@Component`, `@Service`, `@Repository`
8. Bean Lifecycle
9. Spring Framework Modules ภาพรวม
10. แบบฝึกหัดและสรุป

---

## 1. Spring Framework คืออะไร ทำไมครองโลก Java Enterprise

**Spring Framework** (2003) เป็น framework ที่**ครองความนิยมสูงสุด**สำหรับ
การพัฒนา Java enterprise application — ทุกอย่างที่เราเรียนตั้งแต่ Part 1
(OOP, Collections, Concurrency, Design Patterns, JDBC) ล้วนเป็นพื้นฐานที่
Spring นำมาประกอบกันเป็นระบบที่ใช้งานง่ายและทรงพลัง

**ปัญหาที่ Spring แก้**: ก่อน Spring การพัฒนา Java EE (J2EE) ซับซ้อนมาก
(ต้องเขียน boilerplate code จำนวนมาก, EJB ที่ยุ่งยาก) — Spring เสนอแนวคิด
**Dependency Injection** (ทบทวนจาก Part 57 — DIP) เป็นหัวใจหลัก ทำให้เขียน
โค้ดที่ทดสอบง่าย (ทบทวนจาก Part 58-59) และยืดหยุ่นกว่ามาก

## 2. Inversion of Control (IoC) คืออะไร

ทบทวนจาก Part 57, 53: **IoC** คือหลักการที่**ควบคุมการสร้างและจัดการ object
ถูกย้ายจากโค้ดของเรา ไปให้ "framework" จัดการแทน**

```java
// แบบเดิม (ไม่มี IoC): เราควบคุมการสร้าง object เอง (ทบทวนจาก Part 11-12)
public class OrderServiceOld {
    private EmailService emailService = new SmtpEmailService(); // เราสร้างเอง ผูกติดกันแน่น
}
```

```java
// แบบ IoC: Framework สร้างและ "ฉีด" (inject) object ให้เรา (ทบทวนจาก Part 53, 57)
public class OrderServiceNew {
    private final EmailService emailService;

    public OrderServiceNew(EmailService emailService) { // Framework ส่ง object มาให้ผ่าน constructor
        this.emailService = emailService;
    }
}
```

**คำว่า "Inversion"**: ปกติโค้ดของเรา**เรียกหา** dependency (`new
SmtpEmailService()`) แต่ตอนนี้**การควบคุมถูกกลับด้าน** — framework เป็นฝ่าย
**เรียกหาเรา**และ**ส่ง**dependency ที่ต้องการมาให้ (**"Don't call us, we'll
call you"** — Hollywood Principle)

## 3. IoC Container: `ApplicationContext`

**IoC Container** คือ**"โรงงานกลาง"**ที่สร้างและจัดการ object ทั้งหมดใน
แอปพลิเคชัน (ทบทวนแนวคิด Factory Pattern จาก Part 54 และ Simple DI
Container ที่เขียนเองใน Part 53) — Spring เรียก object ที่จัดการโดย
container นี้ว่า **Bean** (หัวข้อ 4)

```java
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

public class IoCContainerDemo {
    public static void main(String[] args) {
        ApplicationContext context =
            new AnnotationConfigApplicationContext(AppConfig.class); // สร้าง IoC Container

        OrderService orderService = context.getBean(OrderService.class); // ขอ bean จาก container
        orderService.processOrder();
        // เราไม่ได้เรียก "new OrderService(...)" เองเลย - container จัดการทุกอย่างให้
    }
}
```

## 4. Bean คืออะไร

**Bean** คือ **object ที่ถูกสร้างและจัดการโดย Spring IoC Container** —
container รู้จัก dependency ระหว่าง bean ทั้งหมด (ทบทวนแนวคิด Dependency
Graph คล้าย Topological Sort จาก Part 34) และ**ฉีด (inject) dependency ให้
ถูกต้องตามลำดับ**โดยอัตโนมัติ

```
IoC Container
┌────────────────────────────────────────┐
│  Bean: EmailService (สร้างก่อน - ไม่มี dependency)  │
│  Bean: OrderRepository (สร้างก่อน)                   │
│  Bean: OrderService (ต้องการ EmailService +           │
│         OrderRepository - สร้างทีหลัง แล้วฉีดทั้งสองเข้าไป)│
└────────────────────────────────────────┘
```

## 5. การประกาศ Bean ด้วย XML (ประวัติศาสตร์ที่ควรรู้)

Spring ยุคแรก (ก่อน Spring 3) ใช้ XML ประกาศ bean ทั้งหมด — ปัจจุบันไม่นิยม
ใช้แล้วแต่**ควรรู้จักไว้**เพราะระบบเก่าจำนวนมากยังใช้อยู่:

```xml
<!-- applicationContext.xml (แบบเก่า) -->
<beans xmlns="http://www.springframework.org/schema/beans">
    <bean id="emailService" class="com.example.SmtpEmailService"/>
    <bean id="orderService" class="com.example.OrderService">
        <constructor-arg ref="emailService"/> <!-- ฉีด emailService ผ่าน constructor -->
    </bean>
</beans>
```

## 6. การประกาศ Bean ด้วย Java Config (`@Configuration`, `@Bean`)

**Java Config** (Spring 3+) แทนที่ XML ด้วยโค้ด Java จริง (type-safe,
refactor-friendly กว่ามาก — ทบทวนข้อดีของ type-safety จาก Part 3, 62):

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration // บอก Spring ว่า class นี้นิยาม bean (ทบทวน annotation จาก Part 52)
public class AppConfig {

    @Bean // ทุกเมธอดที่มี @Bean จะถูกเรียกโดย Spring และผลลัพธ์กลายเป็น bean ใน container
    public EmailService emailService() {
        return new SmtpEmailService();
    }

    @Bean
    public OrderService orderService() {
        return new OrderService(emailService()); // ฉีด dependency ผ่านการเรียกเมธอดตรง ๆ
    }
}
```

```java
interface EmailService { void send(String message); }
class SmtpEmailService implements EmailService {
    public void send(String message) { System.out.println("ส่ง: " + message); }
}
class OrderService {
    private final EmailService emailService;
    public OrderService(EmailService emailService) { this.emailService = emailService; }
    public void processOrder() { emailService.send("คำสั่งซื้อเสร็จสมบูรณ์"); }
}
```

## 7. Component Scanning: `@Component`, `@Service`, `@Repository`

วิธีที่**นิยมที่สุดในปัจจุบัน**: ใช้ **annotation บน class โดยตรง** แล้วให้
Spring**สแกนหา**อัตโนมัติ (ทบทวนแนวคิด annotation scanning ผ่าน Reflection
จาก Part 52-53) ไม่ต้องเขียน `@Bean` method แยกทีละตัว

```java
import org.springframework.stereotype.*;

@Component // annotation ทั่วไป บอกว่า class นี้เป็น bean
class SmtpEmailService implements EmailService {
    public void send(String message) { System.out.println("ส่ง: " + message); }
}

@Service // เหมือน @Component แต่สื่อความหมายว่าเป็น business logic layer
class OrderService {
    private final EmailService emailService;

    @Autowired // บอก Spring ให้ฉีด dependency ที่ต้องการเข้ามาอัตโนมัติ (ทบทวน DI จาก Part 53, 57)
    public OrderService(EmailService emailService) {
        this.emailService = emailService;
    }

    public void processOrder() { emailService.send("คำสั่งซื้อเสร็จสมบูรณ์"); }
}

@Repository // เหมือน @Component แต่สื่อความหมายว่าเป็น data access layer (ทบทวน DAO จาก Part 65)
class OrderRepository { }

@Configuration
@ComponentScan("com.example") // สั่งให้ Spring สแกนหา @Component ทั้งหมดใน package นี้
class AppConfig { }
```

**`@Component`, `@Service`, `@Repository`, `@Controller`** (Part 77) ทำงาน
**เหมือนกันทุกประการ**ในเชิงเทคนิค (ทั้งหมดคือ `@Component` ที่มี
**meta-annotation** — ทบทวนแนวคิดจาก Part 52) — ต่างกันแค่**ชื่อที่สื่อ
ความหมาย**ของบทบาทในสถาปัตยกรรม (semantic clarity) ช่วยให้อ่านโค้ดเข้าใจ
บทบาทของแต่ละ class ได้ทันที

**ตั้งแต่ Spring 4.3**: ถ้า class มี**constructor เดียว** ไม่ต้องเขียน
`@Autowired` เลยก็ได้ (Spring ฉีดให้อัตโนมัติ) — แต่การเขียนไว้ชัดเจนก็ยัง
เป็นที่นิยมเพื่อความชัดเจน

## 8. Bean Lifecycle

```java
import org.springframework.stereotype.Component;
import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;

@Component
public class DatabaseConnectionManager {

    public DatabaseConnectionManager() {
        System.out.println("1. Constructor: สร้าง object");
    }

    @PostConstruct // ทำงานหลัง constructor และ dependency injection เสร็จสมบูรณ์แล้ว
    public void init() {
        System.out.println("2. @PostConstruct: เชื่อมต่อฐานข้อมูล (ทบทวน Part 64-66)");
    }

    @PreDestroy // ทำงานก่อน container จะทำลาย bean นี้ (เช่นตอนแอปพลิเคชัน shutdown)
    public void cleanup() {
        System.out.println("3. @PreDestroy: ปิดการเชื่อมต่อฐานข้อมูล");
    }
}
```

**Bean Scope**: ค่า default คือ **`singleton`** (มี instance เดียวทั้ง
container ตลอดชีวิตแอปพลิเคชัน — ทบทวน Singleton Pattern จาก Part 54 แต่
Spring จัดการให้โดยไม่ต้องเขียน private constructor เอง) — มี scope อื่น
เช่น `prototype` (สร้างใหม่ทุกครั้งที่ขอ), `request`/`session` (สำหรับ web
application — ผูกกับ HTTP request/session แต่ละครั้ง — ทบทวนจาก Part 71-72)

```java
import org.springframework.context.annotation.Scope;

@Component
@Scope("prototype") // สร้าง instance ใหม่ทุกครั้งที่มีการขอ bean นี้
public class ShoppingCart { }
```

## 9. Spring Framework Modules ภาพรวม

Spring ไม่ใช่ framework เดียว แต่เป็น**กลุ่มของ module** ที่ทำงานร่วมกัน:

| Module | ทำหน้าที่อะไร | Part ที่เกี่ยวข้อง |
|---|---|---|
| **Spring Core** | IoC Container, Dependency Injection (Part นี้) | 73-74 |
| **Spring MVC** | Web layer, Controller, REST API | 77-78 |
| **Spring Data** | เข้าถึงฐานข้อมูล (JPA, JDBC) | 79-80 |
| **Spring Security** | Authentication, Authorization | 82-83 |
| **Spring Boot** | Auto-configuration, ลด boilerplate | 75-76 |
| **Spring Cloud** | Microservices (service discovery, config server) | 92-94 |

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียน Java Config (`@Configuration` + `@Bean`) สำหรับ 2 bean:
`PaymentGateway` และ `OrderProcessor` ที่ต้องการ `PaymentGateway` เป็น
dependency

**เฉลย:**

```java
@Configuration
public class AppConfig {
    @Bean
    public PaymentGateway paymentGateway() {
        return new StripePaymentGateway();
    }

    @Bean
    public OrderProcessor orderProcessor() {
        return new OrderProcessor(paymentGateway());
    }
}

interface PaymentGateway { void charge(double amount); }
class StripePaymentGateway implements PaymentGateway {
    public void charge(double amount) { System.out.println("เก็บเงิน: " + amount); }
}
class OrderProcessor {
    private final PaymentGateway gateway;
    public OrderProcessor(PaymentGateway gateway) { this.gateway = gateway; }
}
```

**2)** แปลงโค้ดในข้อ 1 ให้ใช้ Component Scanning (`@Component`,
`@Autowired`) แทน

**เฉลย:**

```java
@Component
class StripePaymentGateway implements PaymentGateway {
    public void charge(double amount) { System.out.println("เก็บเงิน: " + amount); }
}

@Component
class OrderProcessor {
    private final PaymentGateway gateway;
    @Autowired
    public OrderProcessor(PaymentGateway gateway) { this.gateway = gateway; }
}
```

**3)** อธิบายว่า "Inversion of Control" กลับด้านอะไร เทียบกับโค้ดแบบเดิม

**เฉลย**: ในโค้ดแบบเดิม (ไม่มี IoC) **class ของเราเองเป็นฝ่ายควบคุม**การ
สร้าง dependency ที่ต้องการ (เรียก `new` ตรง ๆ ภายใน class — ทบทวนปัญหานี้
จาก Part 57 หัวข้อ DIP) ทำให้ผูกติดกับ concrete implementation แน่นเกินไป
— ด้วย IoC **การควบคุมนี้ถูกย้าย (invert) ไปให้ container** class ของเรา
เพียงแค่**ประกาศว่าต้องการ dependency ชนิดไหน** (ผ่าน constructor
parameter หรือ `@Autowired`) แล้ว container จะเป็นผู้**สร้างและส่ง**
(inject) dependency ที่ตรงกันมาให้เราเองโดยอัตโนมัติ — เราไม่ต้อง "เรียกหา"
dependency อีกต่อไป แต่ dependency ถูก "ส่งมาหาเรา" แทน (Hollywood
Principle ตามที่อธิบายในหัวข้อ 2)

### สรุปเนื้อหา Part 73

- Spring Framework ใช้ Dependency Injection เป็นหัวใจหลัก แก้ปัญหาความ
  ซับซ้อนของ Java EE ดั้งเดิม
- IoC (Inversion of Control): การควบคุมการสร้าง object ถูกย้ายจากโค้ดของ
  เราไปให้ framework จัดการแทน
- `ApplicationContext` คือ IoC Container ที่สร้างและจัดการ Bean ทั้งหมด
- ประกาศ Bean ได้ 3 วิธี: XML (legacy), Java Config (`@Bean`), Component
  Scanning (`@Component`) — วิธีหลังนิยมที่สุดในปัจจุบัน
- `@Component`/`@Service`/`@Repository`/`@Controller` เหมือนกันทางเทคนิค
  ต่างกันแค่ความหมายเชิงสถาปัตยกรรม
- Bean มี lifecycle (`@PostConstruct`/`@PreDestroy`) และ scope (`singleton`
  เป็น default)

**ต่อไป**: [Part 74 — Spring Core: Dependency Injection, Bean Scopes, Configuration](./part-074-spring-core-di.md)
