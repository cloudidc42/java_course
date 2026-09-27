# Part 76: Spring Boot: Configuration Properties, Profiles, Environment

> ขั้นตอนที่ 751-760 ของหลักสูตร | ระดับ: สูงมาก

## สารบัญ

1. `@ConfigurationProperties`: ทางเลือกที่ดีกว่า `@Value`
2. Validation ของ Configuration Properties
3. Nested Configuration Properties
4. Spring Profiles: แยก Config ตาม Environment
5. การกำหนด Active Profile
6. `@Profile` Annotation บน Bean
7. ลำดับความสำคัญของ Configuration Source
8. `Environment` Interface
9. Externalized Configuration ด้วย Environment Variables
10. แบบฝึกหัดและสรุป

---

## 1. `@ConfigurationProperties`: ทางเลือกที่ดีกว่า `@Value`

ทบทวนจาก Part 74: `@Value` ใช้ได้ดีกับค่าเดี่ยว ๆ แต่ถ้ามี config หลายค่า
ที่เกี่ยวข้องกัน **`@ConfigurationProperties`** จัดกลุ่มให้เป็น**object
เดียวที่ type-safe** (ทบทวนความสำคัญของ type safety จาก Part 3, 62)

```yaml
# application.yml
app:
  name: My E-Commerce App
  max-items-per-order: 10
  support-email: support@example.com
```

```java
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@Component
@ConfigurationProperties(prefix = "app") // ผูกกับทุก key ที่ขึ้นต้นด้วย "app."
public class AppProperties {
    private String name;
    private int maxItemsPerOrder;      // Spring แปลง kebab-case (max-items-per-order) เป็น camelCase อัตโนมัติ
    private String supportEmail;

    // ต้องมี getter/setter ครบ (Spring ใช้ในการ bind ค่า - ทบทวน JavaBeans convention จาก Part 13)
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public int getMaxItemsPerOrder() { return maxItemsPerOrder; }
    public void setMaxItemsPerOrder(int maxItemsPerOrder) { this.maxItemsPerOrder = maxItemsPerOrder; }
    public String getSupportEmail() { return supportEmail; }
    public void setSupportEmail(String supportEmail) { this.supportEmail = supportEmail; }
}
```

```java
@Service
public class OrderService {
    private final AppProperties appProperties;

    public OrderService(AppProperties appProperties) { // ฉีดเข้ามาเหมือน bean ทั่วไป (ทบทวน DI จาก Part 74)
        this.appProperties = appProperties;
    }

    void validateOrderSize(int itemCount) {
        if (itemCount > appProperties.getMaxItemsPerOrder()) {
            throw new IllegalArgumentException("เกินจำนวนสูงสุด: " + appProperties.getMaxItemsPerOrder());
        }
    }
}
```

**ข้อดีเทียบกับ `@Value`**: จัดกลุ่ม config ที่เกี่ยวข้องกันเป็น object เดียว
(ทบทวนแนวคิด SRP จาก Part 57), autocomplete ใน IDE ทำงานได้ (เพราะเป็น
Java object จริง ไม่ใช่ string key), และ**validate ได้** (หัวข้อ 2)

## 2. Validation ของ Configuration Properties

ผสาน Bean Validation (ปูทางสู่ Part 81) เข้ากับ configuration properties:

```java
import jakarta.validation.constraints.*;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.validation.annotation.Validated;

@Validated // เปิดใช้งาน validation
@ConfigurationProperties(prefix = "app")
public class ValidatedAppProperties {

    @NotBlank(message = "app.name ห้ามว่างเปล่า")
    private String name;

    @Min(1) @Max(100)
    private int maxItemsPerOrder;

    @Email(message = "support-email ต้องเป็นรูปแบบอีเมลที่ถูกต้อง")
    private String supportEmail;

    // getter/setter ...
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public int getMaxItemsPerOrder() { return maxItemsPerOrder; }
    public void setMaxItemsPerOrder(int v) { this.maxItemsPerOrder = v; }
    public String getSupportEmail() { return supportEmail; }
    public void setSupportEmail(String v) { this.supportEmail = v; }
}
```

ถ้า config ไม่ผ่าน validation **แอปพลิเคชันจะ fail ตั้งแต่ startup** (ไม่ใช่
ตอนรันจริง) — นี่คือหลักการ **"fail fast"** เดียวกับที่ static typing
(Part 3) และ Constructor Injection (Part 74) ใช้ — ตรวจจับปัญหาให้เร็วที่สุด
ก่อนที่จะกระทบผู้ใช้จริง

## 3. Nested Configuration Properties

```yaml
app:
  name: My App
  payment:
    gateway: stripe
    timeout-seconds: 30
    retry:
      max-attempts: 3
      delay-ms: 500
```

```java
@ConfigurationProperties(prefix = "app")
public class AppProperties {
    private String name;
    private Payment payment = new Payment(); // nested object

    public static class Payment {
        private String gateway;
        private int timeoutSeconds;
        private Retry retry = new Retry();

        public static class Retry {
            private int maxAttempts;
            private long delayMs;
            // getter/setter ...
            public int getMaxAttempts() { return maxAttempts; }
            public void setMaxAttempts(int v) { this.maxAttempts = v; }
            public long getDelayMs() { return delayMs; }
            public void setDelayMs(long v) { this.delayMs = v; }
        }
        // getter/setter สำหรับ gateway, timeoutSeconds, retry ...
        public String getGateway() { return gateway; }
        public void setGateway(String v) { this.gateway = v; }
        public int getTimeoutSeconds() { return timeoutSeconds; }
        public void setTimeoutSeconds(int v) { this.timeoutSeconds = v; }
        public Retry getRetry() { return retry; }
        public void setRetry(Retry v) { this.retry = v; }
    }
    // getter/setter สำหรับ name, payment ...
    public String getName() { return name; }
    public void setName(String v) { this.name = v; }
    public Payment getPayment() { return payment; }
    public void setPayment(Payment v) { this.payment = v; }
}
```

**หมายเหตุ**: ตัวอย่างนี้แสดงโครงสร้าง nested ที่ลึก — ในโค้ดจริงมักใช้
**record** (ทบทวนจาก Part 51, รองรับตั้งแต่ Spring Boot 3.0) เพื่อลดโค้ด
getter/setter ที่ยืดยาวแบบนี้

## 4. Spring Profiles: แยก Config ตาม Environment

**Profile** ให้แยก config สำหรับ**environment ต่างกัน** (dev, staging,
production) — เป็นปัญหาที่พบทุกโปรเจกต์จริง: database ของ dev ต่างจาก
production, log level ต่างกัน, ฯลฯ

```
src/main/resources/
├── application.yml              <- ค่า default ที่ใช้ร่วมกันทุก environment
├── application-dev.yml           <- override สำหรับ dev
├── application-staging.yml        <- override สำหรับ staging
└── application-prod.yml            <- override สำหรับ production
```

```yaml
# application-dev.yml
spring:
  datasource:
    url: jdbc:h2:mem:devdb  # H2 in-memory database (ทบทวน JDBC จาก Part 64) - สะดวกสำหรับ dev
logging:
  level:
    com.example: DEBUG
```

```yaml
# application-prod.yml
spring:
  datasource:
    url: jdbc:mysql://prod-db.example.com:3306/mydb
logging:
  level:
    com.example: WARN  # production ไม่ต้องการ log DEBUG ที่ท่วมมาก (ทบทวนจาก Part 63)
```

## 5. การกำหนด Active Profile

```bash
# ผ่าน command line argument
java -jar myapp.jar --spring.profiles.active=prod

# ผ่าน environment variable (นิยมมากใน container/cloud - ปูทางสู่ Part 95)
export SPRING_PROFILES_ACTIVE=prod
java -jar myapp.jar

# ผ่าน application.yml (ค่า default ถ้าไม่ระบุอย่างอื่น)
```

```yaml
# application.yml
spring:
  profiles:
    active: dev  # ค่า default สำหรับ local development
```

**เปิดหลาย profile พร้อมกัน**: `--spring.profiles.active=prod,monitoring`
— config จากทั้งสอง profile จะถูก**รวมกัน** (merge) ถ้ามี key ซ้ำ profile
ที่ระบุหลังจะ override ก่อน

## 6. `@Profile` Annotation บน Bean

**Bean** ก็เลือกได้ว่าจะ**สร้างเฉพาะบาง profile**เท่านั้น (ทบทวนแนวคิด
conditional bean creation จาก Part 75 — auto-configuration):

```java
import org.springframework.context.annotation.Profile;
import org.springframework.stereotype.Component;

@Component
@Profile("dev") // สร้าง bean นี้เฉพาะเมื่อ active profile คือ "dev" เท่านั้น
class MockPaymentGateway implements PaymentGateway {
    public void charge(double amount) {
        System.out.println("[MOCK] จำลองการชำระเงิน: " + amount + " (ไม่มีการเรียกเงินจริง)");
    }
}

@Component
@Profile("prod")
class RealPaymentGateway implements PaymentGateway {
    public void charge(double amount) {
        System.out.println("[REAL] เรียกเก็บเงินจริงผ่าน Stripe API: " + amount);
    }
}
```

**ประโยชน์**: ทดสอบ business logic บน dev environment ได้โดย**ไม่เรียก
service จริงที่มีค่าใช้จ่ายหรือผลกระทบจริง** (เช่น payment gateway, email
service) — คล้ายแนวคิด Mock จาก Part 59 แต่ทำงานที่ระดับ configuration
ทั้งแอปพลิเคชัน ไม่ใช่แค่ใน unit test

## 7. ลำดับความสำคัญของ Configuration Source

Spring Boot รวม config จากหลายแหล่ง — ถ้ามี key เดียวกันซ้ำกัน**ลำดับ
ความสำคัญ**จะตัดสินว่าใครชนะ (เรียงจากสูงสุดไปต่ำสุด บางส่วน):

```
1. Command line arguments        (--server.port=9090)
2. Environment variables          (SERVER_PORT=9090)
3. application-{profile}.yml       (application-prod.yml)
4. application.yml                  (ค่า default)
5. @PropertySource ที่ระบุใน @Configuration
6. ค่า default ที่ hardcode ในโค้ด
```

**ตัวอย่างการใช้ประโยชน์**: ตั้งค่า database password ผ่าน**environment
variable**ใน production (ปลอดภัยกว่าเก็บใน `application.yml` ที่อาจถูก
commit ลง git โดยไม่ตั้งใจ — ทบทวนความเสี่ยงด้านความปลอดภัยที่จะลงลึกใน Part
101) โดยที่ `application.yml` ยังมีค่า default สำหรับ local development

## 8. `Environment` Interface

```java
import org.springframework.core.env.Environment;
import org.springframework.stereotype.Component;

@Component
public class EnvironmentDemo {
    private final Environment environment;

    public EnvironmentDemo(Environment environment) { // ฉีด Environment เข้ามาได้เหมือน bean ทั่วไป
        this.environment = environment;
    }

    void printInfo() {
        String[] activeProfiles = environment.getActiveProfiles();
        String dbUrl = environment.getProperty("spring.datasource.url");
        int port = environment.getProperty("server.port", Integer.class, 8080); // ค่า default ถ้าไม่พบ

        System.out.println("Active profiles: " + java.util.Arrays.toString(activeProfiles));
        System.out.println("Database URL: " + dbUrl);
        System.out.println("Port: " + port);
    }
}
```

`Environment` ใช้เมื่อต้องการ**อ่าน config แบบ dynamic** (ไม่รู้ key
ล่วงหน้าตอน compile) — สำหรับกรณีทั่วไป `@ConfigurationProperties` หรือ
`@Value` (type-safe กว่า) ยังเป็นตัวเลือกที่ดีกว่า

## 9. Externalized Configuration ด้วย Environment Variables

**Container/Cloud environment** (Docker — Part 95, Kubernetes — Part 96)
นิยมส่ง config ผ่าน **environment variable** มากกว่าไฟล์ (ทบทวนหลักการ
**12-Factor App** ที่จะลงลึกเชิงสถาปัตยกรรมในภาพรวมของหลักสูตรช่วงหลัง):

```bash
docker run -e SPRING_DATASOURCE_URL=jdbc:mysql://db:3306/mydb \
           -e SPRING_DATASOURCE_USERNAME=root \
           -e SPRING_PROFILES_ACTIVE=prod \
           myapp:latest
```

**การแปลงชื่อ**: Spring Boot แปลง environment variable
`SPRING_DATASOURCE_URL` เป็น property `spring.datasource.url` โดยอัตโนมัติ
(underscore เป็น dot, ตัวใหญ่เป็นตัวเล็ก) — ทำให้ config ที่เขียนไว้ใน
`application.yml` และที่ส่งผ่าน environment variable ใช้**กลไกเดียวกัน**
โดยไม่ต้องแก้โค้ด

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียน `@ConfigurationProperties` class สำหรับ config การเชื่อมต่อ
Redis (host, port, timeout) พร้อม validation ว่า port ต้องอยู่ระหว่าง 1-65535

**เฉลย:**

```java
@Validated
@ConfigurationProperties(prefix = "redis")
public class RedisProperties {
    @NotBlank
    private String host;

    @Min(1) @Max(65535)
    private int port;

    private int timeoutMs = 5000; // ค่า default

    public String getHost() { return host; }
    public void setHost(String host) { this.host = host; }
    public int getPort() { return port; }
    public void setPort(int port) { this.port = port; }
    public int getTimeoutMs() { return timeoutMs; }
    public void setTimeoutMs(int timeoutMs) { this.timeoutMs = timeoutMs; }
}
```

**2)** สร้าง 2 bean ที่ implement `EmailService` โดยใช้ `@Profile` แยก
`dev` (พิมพ์ log แทนส่งจริง) และ `prod` (ส่งอีเมลจริง)

**เฉลย:**

```java
@Component
@Profile("dev")
class LoggingEmailService implements EmailService {
    public void send(String to, String msg) {
        System.out.println("[DEV LOG] อีเมลถึง " + to + ": " + msg);
    }
}

@Component
@Profile("prod")
class SmtpEmailService implements EmailService {
    public void send(String to, String msg) {
        System.out.println("[PROD] ส่งอีเมลจริงถึง " + to);
    }
}
```

**3)** อธิบายว่าทำไม environment variable มักถูกใช้เก็บ credential
(password, API key) มากกว่าเก็บไว้ใน `application.yml`

**เฉลย**: `application.yml` เป็นไฟล์ที่**เก็บอยู่ใน source code repository**
(ทบทวนโครงสร้างโปรเจกต์จาก Part 20, 75) ถ้า commit credential ลงไปตรง ๆ
จะทำให้**ทุกคนที่มีสิทธิ์เข้าถึง git repository เห็น credential นั้นได้**
(รวมถึงประวัติ git ที่ลบไม่ได้ง่าย ๆ แม้จะลบออกจาก commit ล่าสุดแล้วก็ตาม)
ซึ่งเป็นความเสี่ยงด้านความปลอดภัยที่ร้ายแรง (จะลงลึกใน Part 101) — Environment
variable ถูกกำหนดที่**ระดับ runtime environment** (เครื่อง server, container
— ทบทวนจาก Part 95-96) แยกออกจาก source code โดยสิ้นเชิง ทำให้ credential
ไม่ถูก commit ลง git เลย และสามารถจัดการสิทธิ์การเข้าถึงได้แยกจากสิทธิ์การ
เข้าถึง source code (เช่นผ่าน secret management tool เฉพาะทาง)

### สรุปเนื้อหา Part 76

- `@ConfigurationProperties` จัดกลุ่ม config ที่เกี่ยวข้องกันเป็น type-safe
  object ดีกว่า `@Value` สำหรับ config หลายค่า
- Validation บน configuration properties ทำให้แอปพลิเคชัน fail ตั้งแต่
  startup ถ้า config ผิด (fail fast)
- Spring Profiles แยก config ตาม environment (dev/staging/prod) ผ่านไฟล์
  `application-{profile}.yml`
- `@Profile` บน bean เลือกสร้าง bean เฉพาะ profile ที่กำหนด
- Configuration source มีลำดับความสำคัญ: command line > environment
  variable > profile-specific file > default file
- Environment variable เหมาะกับการเก็บ credential และ config ที่เปลี่ยนตาม
  runtime environment (container/cloud)

**ต่อไป**: [Part 77 — Spring MVC: Controller, Model, View, Thymeleaf](./part-077-spring-mvc.md)
