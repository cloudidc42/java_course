# Part 63: Logging ด้วย SLF4J และ Logback

> ขั้นตอนที่ 621-630 ของหลักสูตร | ระดับ: สูง

## สารบัญ

1. ทำไมต้องมี Logging Framework (แทน `System.out.println`)
2. SLF4J: Facade Pattern ในทางปฏิบัติ
3. การตั้งค่า SLF4J + Logback
4. Log Levels
5. Logback Configuration: `logback.xml`
6. Parameterized Logging: หลีกเลี่ยงปัญหาประสิทธิภาพ
7. MDC: Mapped Diagnostic Context
8. Log Rotation และ Appenders
9. Best Practices ของการเขียน Log ที่ดี
10. แบบฝึกหัดและสรุป

---

## 1. ทำไมต้องมี Logging Framework (แทน `System.out.println`)

ตลอดหลักสูตรนี้เราใช้ `System.out.println()` เพื่อความง่าย แต่ในโค้ด
production **ไม่ควรใช้เลย** เพราะมีข้อจำกัดมาก:

- **ควบคุมไม่ได้**: ไม่สามารถปิด/เปิด log บางระดับได้โดยไม่แก้ไขโค้ด
- **ไม่มี timestamp/context**: ไม่รู้ว่า log เกิดขึ้นเมื่อไร จาก class ไหน
- **ไม่ยืดหยุ่น**: เขียนไปที่ console อย่างเดียว ไม่ส่งไปไฟล์/ระบบ monitoring
  ได้ (Part 99)
- **Performance**: `println` เป็น synchronized method (ทบทวนจาก Part 47)
  ซึ่งช้าถ้าเรียกบ่อยมากในระบบ concurrent

**Logging Framework** แก้ปัญหาทั้งหมดนี้ ให้ควบคุม level, format, ปลายทาง
(console/file/network) ได้อย่างละเอียด

## 2. SLF4J: Facade Pattern ในทางปฏิบัติ

**SLF4J (Simple Logging Facade for Java)** เป็น**facade** (ทบทวน Facade
Pattern จาก Part 55) — ไม่ใช่ logging framework จริง แต่เป็น**interface
กลาง**ที่ให้โค้ดเรียกใช้ โดยไม่ผูกติดกับ implementation ใดตัวหนึ่ง (ทบทวน
Dependency Inversion Principle จาก Part 57)

```
โค้ดของเรา ──> SLF4J API (interface) ──> Logback / Log4j2 / java.util.logging
                                          (implementation จริง - สลับได้โดยไม่แก้โค้ด)
```

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class UserService {
    private static final Logger logger = LoggerFactory.getLogger(UserService.class);
    // ธรรมเนียมมาตรฐาน: logger เป็น static final, ใช้ชื่อ class ปัจจุบันเสมอ

    public void registerUser(String username) {
        logger.info("กำลังลงทะเบียนผู้ใช้: {}", username); // เขียน log ระดับ INFO
        // ... logic การลงทะเบียน ...
        logger.debug("บันทึกข้อมูลผู้ใช้ลงฐานข้อมูลสำเร็จ");
    }
}
```

**ประโยชน์ของ Facade Pattern ในบริบทนี้**: library ที่เราใช้ (เช่น Spring —
Part 73) เขียนโค้ดเรียก SLF4J API แต่**เราเลือก implementation จริงเองได้**
(Logback, Log4j2) โดยไม่ต้องแก้ไข library — เพียงเปลี่ยน dependency ใน
`pom.xml`/`build.gradle` (ทบทวนจาก Part 61-62)

## 3. การตั้งค่า SLF4J + Logback

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-classic</artifactId>
        <version>1.4.14</version>
    </dependency>
    <!-- logback-classic ดึง slf4j-api มาด้วยอัตโนมัติ (transitive dependency - ทบทวน Part 61) -->
</dependencies>
```

**Logback** เป็น implementation ที่นิยมที่สุด (เขียนโดยผู้สร้าง SLF4J
เองเช่นกัน) และเป็น**ค่า default ของ Spring Boot** (Part 75)

## 4. Log Levels

Log level กำหนด**ความสำคัญ**ของข้อความ log เรียงจากน้อยไปมาก:

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class LogLevelsDemo {
    private static final Logger logger = LoggerFactory.getLogger(LogLevelsDemo.class);

    public static void main(String[] args) {
        logger.trace("รายละเอียดยิบย่อยที่สุด (มักใช้ debug ปัญหาลึก ๆ)");
        logger.debug("ข้อมูลสำหรับ debug ระหว่างพัฒนา");
        logger.info("เหตุการณ์ปกติที่น่าสนใจ (เช่น server เริ่มทำงาน, ผู้ใช้ล็อกอิน)");
        logger.warn("สถานการณ์ที่น่าสงสัยแต่ยังทำงานต่อได้");
        logger.error("ข้อผิดพลาดที่ต้องให้ความสนใจทันที");
    }
}
```

| Level | ใช้เมื่อ |
|---|---|
| `TRACE` | รายละเอียดยิบย่อยที่สุด (เปิดใช้เฉพาะตอน debug ปัญหาหนักจริง ๆ) |
| `DEBUG` | ข้อมูลที่มีประโยชน์ระหว่างพัฒนา (ปิดใน production ปกติ) |
| `INFO` | เหตุการณ์ระดับปกติที่น่าบันทึกไว้ (server start, transaction สำเร็จ) |
| `WARN` | สถานการณ์ผิดปกติแต่ระบบยังทำงานได้ (เช่น retry connection สำเร็จหลัง fail ครั้งแรก) |
| `ERROR` | ข้อผิดพลาดร้ายแรงที่ต้องดำเนินการ (exception ที่ไม่คาดคิด) |

**ในทางปฏิบัติ**: production environment มักตั้งค่าที่ `INFO` เป็นค่า
default (เพื่อไม่ให้ log ท่วมมากเกินไป) และเปิด `DEBUG`/`TRACE` ชั่วคราว
เฉพาะตอน troubleshoot ปัญหา

## 5. Logback Configuration: `logback.xml`

วางไฟล์นี้ไว้ที่ `src/main/resources/logback.xml` (ทบทวนโครงสร้างโปรเจกต์
จาก Part 20):

```xml
<configuration>
    <!-- Appender: กำหนดว่า log จะถูกเขียนไปที่ไหน -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n</pattern>
            <!-- %d=timestamp, %thread=ชื่อ thread, %level=log level, %logger=ชื่อคลาส, %msg=ข้อความ -->
        </encoder>
    </appender>

    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/application.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/application.%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>30</maxHistory> <!-- เก็บ log เก่าไว้แค่ 30 วัน -->
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <!-- Logger เฉพาะ package (override ค่า default สำหรับบางส่วนของโค้ด) -->
    <logger name="com.example.service" level="DEBUG"/>

    <!-- Root logger: ค่า default สำหรับทุกที่ที่ไม่ได้ override -->
    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="FILE"/>
    </root>
</configuration>
```

**ข้อดีสำคัญ**: เปลี่ยน log level หรือปลายทางได้**โดยไม่แก้ไข source code
เลย** (แก้แค่ XML แล้ว restart) — ต่างจากการใช้ `System.out.println` ที่ต้อง
แก้โค้ดและ compile ใหม่ทุกครั้ง

## 6. Parameterized Logging: หลีกเลี่ยงปัญหาประสิทธิภาพ

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class ParameterizedLoggingDemo {
    private static final Logger logger = LoggerFactory.getLogger(ParameterizedLoggingDemo.class);

    void demonstrateProblem(String username, int age) {
        // ไม่ดี: String concatenation เกิดขึ้น "เสมอ" แม้ log level นี้ถูกปิดอยู่!
        logger.debug("ผู้ใช้: " + username + ", อายุ: " + age); // สิ้นเปลืองถ้า DEBUG ถูกปิดใน production

        // ดี: parameterized logging - การแทนค่าเกิดขึ้น "เฉพาะเมื่อ" log level นี้เปิดอยู่จริง
        logger.debug("ผู้ใช้: {}, อายุ: {}", username, age); // {} คือ placeholder (ทบทวน printf จาก Part 9)

        // ยังดีกว่าถ้ามีการคำนวณที่ซับซ้อนใน argument (lazy evaluation - ทบทวนแนวคิดจาก Part 40, 43)
        // logger.debug("ผลลัพธ์: {}", expensiveCalculation()); // ยังคงเรียก expensiveCalculation() เสมออยู่ดี!
        // วิธีที่ดีที่สุดสำหรับกรณีนี้คือเช็ค logger.isDebugEnabled() ก่อน ถ้าการคำนวณ cost สูงมาก
        if (logger.isDebugEnabled()) {
            logger.debug("ผลลัพธ์: {}", "expensive result placeholder");
        }
    }
}
```

**เหตุผลเชิงเทคนิค**: `"text " + variable` ต้องสร้าง `String` ใหม่**ทันที**
ไม่ว่า log level นั้นจะถูกเปิดหรือปิด (ทบทวนปัญหา String concatenation จาก
Part 9) — parameterized logging (`{}`) ให้ framework**ตัดสินใจก่อน**ว่าจะ
สร้าง string หรือไม่ ทำให้ไม่มี overhead เลยถ้า log level นั้นถูกปิดอยู่

## 7. MDC: Mapped Diagnostic Context

**MDC** ใช้แนบ**ข้อมูล context** (เช่น request ID, user ID) ไปกับ**ทุก log
statement**ของ thread นั้นโดยอัตโนมัติ — มีประโยชน์มากในระบบ web application
ที่ต้อง**ติดตาม request หนึ่งตัวข้ามหลาย log entry** (ปูทางสู่ distributed
tracing ใน Part 99)

```java
import org.slf4j.MDC;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class MDCDemo {
    private static final Logger logger = LoggerFactory.getLogger(MDCDemo.class);

    void handleRequest(String requestId) {
        MDC.put("requestId", requestId); // ผูก context กับ thread ปัจจุบัน (ทบทวน Thread จาก Part 46)
        try {
            logger.info("เริ่มประมวลผล request"); // log นี้จะมี requestId ติดไปด้วยอัตโนมัติ (ถ้า pattern กำหนดไว้)
            processBusinessLogic();
            logger.info("จบการประมวลผล request");
        } finally {
            MDC.clear(); // สำคัญมาก! ต้องล้าง MDC เสมอ ไม่งั้น thread pool (ทบทวน Part 48) จะ "รั่ว" context เก่าไปยัง request อื่น
        }
    }

    void processBusinessLogic() {
        logger.info("กำลังทำงาน business logic"); // ยังมี requestId เดิมติดไปด้วย แม้อยู่คนละเมธอด
    }
}
```

```xml
<!-- ปรับ pattern ใน logback.xml ให้แสดง MDC -->
<pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} [requestId=%X{requestId}] - %msg%n</pattern>
```

**ข้อควรระวังสำคัญ**: ต้องเรียก `MDC.clear()` ใน `finally` block เสมอ
(ทบทวน try-finally จาก Part 10, 21) เพราะ**thread pool นำ thread กลับมาใช้
ซ้ำ** (ทบทวนจาก Part 48) — ถ้าไม่ clear, context เก่าจาก request ก่อนหน้า
อาจ "รั่ว" ไปติดกับ log ของ request ถัดไปที่ใช้ thread เดียวกัน

## 8. Log Rotation และ Appenders

ทบทวนจากหัวข้อ 5: `RollingFileAppender` ป้องกันไฟล์ log**ใหญ่เกินไปจนเต็ม
disk** โดยแบ่งไฟล์ตามเวลาหรือขนาด:

```xml
<appender name="SIZE_BASED_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>logs/app.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
        <fileNamePattern>logs/app.%d{yyyy-MM-dd}.%i.log</fileNamePattern>
        <maxFileSize>10MB</maxFileSize> <!-- แบ่งไฟล์ใหม่ทุก 10MB -->
        <maxHistory>30</maxHistory>
        <totalSizeCap>1GB</totalSizeCap> <!-- ลบไฟล์เก่าถ้ารวมกันเกิน 1GB -->
    </rollingPolicy>
    <encoder>
        <pattern>%d{yyyy-MM-dd HH:mm:ss} %-5level - %msg%n</pattern>
    </encoder>
</appender>
```

## 9. Best Practices ของการเขียน Log ที่ดี

1. **เลือก log level ให้เหมาะสม** (ทบทวนหัวข้อ 4) — ไม่ใช้ `INFO` สำหรับทุก
   อย่าง (ทำให้ log ท่วมและหาข้อมูลสำคัญยาก)
2. **Log ข้อมูลที่มีประโยชน์ ไม่ใช่แค่ "เกิด error"**: ระบุ context ที่ทำให้
   debug ได้จริง (ค่า input, ID ที่เกี่ยวข้อง)
3. **อย่า log ข้อมูล sensitive**: password, credit card number, personal
   data (ทบทวนความเสี่ยงด้านความปลอดภัยจาก Part 38 — จะลงลึกใน Part 101)
4. **Log exception พร้อม stack trace เสมอ** เมื่อจับ exception ไว้:

```java
try {
    riskyOperation();
} catch (Exception e) {
    logger.error("การประมวลผลล้มเหลว สำหรับ orderId={}", orderId, e); // e เป็น argument สุดท้าย
    // Logback จะพิมพ์ stack trace เต็มรูปแบบให้อัตโนมัติ (ทบทวน Exception จาก Part 10, 21)
}
```

5. **ใช้ MDC สำหรับ context ที่ต้องติดตามข้าม request** (หัวข้อ 7)

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** แก้ไขโค้ดนี้ให้ใช้ parameterized logging แทน string concatenation:

```java
logger.info("ผู้ใช้ " + username + " ทำการสั่งซื้อ order #" + orderId + " มูลค่า " + amount);
```

**เฉลย:**

```java
logger.info("ผู้ใช้ {} ทำการสั่งซื้อ order #{} มูลค่า {}", username, orderId, amount);
```

**2)** เขียน method `processOrder(String orderId)` ที่ใช้ MDC ผูก orderId
กับทุก log statement ระหว่างการประมวลผล

**เฉลย:**

```java
void processOrder(String orderId) {
    MDC.put("orderId", orderId);
    try {
        logger.info("เริ่มประมวลผลคำสั่งซื้อ");
        validateOrder();
        logger.info("ตรวจสอบคำสั่งซื้อสำเร็จ");
    } finally {
        MDC.clear();
    }
}

void validateOrder() {
    logger.info("กำลังตรวจสอบข้อมูล"); // มี orderId ติดไปด้วยอัตโนมัติ
}
```

**3)** อธิบายว่าทำไม `logger.debug("value: " + x)` มี performance overhead
มากกว่า `logger.debug("value: {}", x)` แม้ log level DEBUG จะถูกปิดอยู่

**เฉลย**: `"value: " + x` เป็น**การดำเนินการที่ Java compiler แปลงเป็นการสร้าง
`StringBuilder` และเรียก `.toString()`**ทันทีที่บรรทัดนี้ถูกรัน (ทบทวนจาก
Part 9) — การสร้าง String นี้เกิดขึ้น**ก่อน**ที่จะส่งเข้าเมธอด `debug()`
เสียอีก ดังนั้นแม้ log level DEBUG จะถูกปิดอยู่ (ทำให้เมธอด `debug()` ไม่ทำ
อะไรเลยภายใน) การสร้าง String ก็ยังเกิดขึ้นอยู่ดีโดยเปล่าประโยชน์ — ในขณะที่
`logger.debug("value: {}", x)` ส่ง `x` เป็น argument ดิบ ๆ ไปให้เมธอด ซึ่ง
ภายในจะ**เช็คก่อนว่า DEBUG level เปิดอยู่หรือไม่**ก่อนที่จะแทนค่า `{}` ด้วย
`x` (สร้าง String จริง) — ถ้า DEBUG ถูกปิด จะไม่มีการสร้าง String ใด ๆ เกิดขึ้น
เลย ประหยัด CPU และหน่วยความจำได้มากในระบบที่ log จำนวนมาก

### สรุปเนื้อหา Part 63

- SLF4J เป็น facade (interface กลาง) ให้เลือก logging implementation จริง
  (Logback) ได้โดยไม่ผูกติดโค้ด
- Log levels (TRACE < DEBUG < INFO < WARN < ERROR) กำหนดความสำคัญของข้อความ
- `logback.xml` กำหนด appender (console/file), pattern, และ level ของแต่ละ
  package โดยไม่ต้องแก้โค้ด
- Parameterized logging (`{}`) หลีกเลี่ยง performance overhead จาก String
  concatenation ที่ไม่จำเป็น
- MDC ผูก context (เช่น request ID) กับทุก log statement ของ thread ปัจจุบัน
  — ต้อง `clear()` เสมอเพื่อป้องกัน context รั่วไหลข้าม request

**ต่อไป**: [Part 64 — JDBC และการเชื่อมต่อฐานข้อมูล](./part-064-jdbc.md)
