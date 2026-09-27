# Part 75: Spring Boot เบื้องต้น: Auto-configuration, Starter

> ขั้นตอนที่ 741-750 ของหลักสูตร | ระดับ: สูงมาก

## สารบัญ

1. ทำไมต้องมี Spring Boot (ปัญหาของ Spring แบบดั้งเดิม)
2. การสร้างโปรเจกต์ Spring Boot ด้วย Spring Initializr
3. `@SpringBootApplication`: 3 Annotation รวมเป็นหนึ่ง
4. Starter Dependencies
5. Auto-configuration ทำงานอย่างไร
6. Embedded Server
7. โครงสร้างโปรเจกต์ Spring Boot มาตรฐาน
8. `application.properties` vs `application.yml`
9. Spring Boot DevTools
10. แบบฝึกหัดและสรุป

---

## 1. ทำไมต้องมี Spring Boot (ปัญหาของ Spring แบบดั้งเดิม)

Spring Framework (Part 73-74) ทรงพลังมาก แต่**ตั้งค่าเยอะ**: ต้อง config
`DispatcherServlet`, ตั้งค่า database connection ด้วยมือ, deploy WAR ไปยัง
Tomcat แยกต่างหาก (ทบทวนจาก Part 72) — **Spring Boot** (2014) แก้ปัญหานี้
ด้วยหลักการ **"Convention over Configuration"** (ทบทวนแนวคิดจาก Part 20,
61): ให้ค่า default ที่สมเหตุสมผลสำหรับ 90% ของกรณีใช้งาน

```
Spring แบบดั้งเดิม:              Spring Boot:
- ตั้งค่า XML/Java Config เยอะ    - แค่เพิ่ม dependency แล้วรัน
- ต้องมี Tomcat แยก                - มี embedded Tomcat ในตัว
- Config database ด้วยมือ           - แค่ระบุ URL ใน properties
- Deploy WAR                          - รันด้วย java -jar ได้ทันที
```

## 2. การสร้างโปรเจกต์ Spring Boot ด้วย Spring Initializr

**Spring Initializr** (https://start.spring.io) เป็นเครื่องมือสร้างโปรเจกต์
เริ่มต้นพร้อม dependency ที่เลือก — สร้างไฟล์ `pom.xml`/`build.gradle`
(ทบทวนจาก Part 61-62) ให้ครบถ้วนทันที

```xml
<!-- pom.xml ที่ได้จาก Spring Initializr -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.0</version>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId> <!-- ทบทวนจาก Part 61 -->
        </plugin>
    </plugins>
</build>
```

**`spring-boot-starter-parent`**: parent POM (ทบทวนแนวคิดจาก Part 61 —
Multi-Module) ที่กำหนด**version ของ dependency ทุกตัวที่เข้ากันได้กัน** ไม่
ต้องระบุ version เองทีละตัว ลดปัญหา dependency conflict (ทบทวนจาก Part 61)

## 3. `@SpringBootApplication`: 3 Annotation รวมเป็นหนึ่ง

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication // รวม 3 annotation เข้าด้วยกัน (ทบทวนแนวคิด composability จาก Part 52)
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args); // เริ่มต้น IoC Container + embedded server
    }
}
```

`@SpringBootApplication` เทียบเท่ากับ 3 annotation รวมกัน:

```java
@SpringBootConfiguration  // เทียบเท่า @Configuration (ทบทวนจาก Part 73)
@EnableAutoConfiguration   // เปิดใช้งาน auto-configuration (หัวข้อ 5)
@ComponentScan             // สแกนหา @Component ใน package นี้และย่อย (ทบทวนจาก Part 73)
```

**หลักการนี้เหมือนกับ meta-annotation ที่เรียนใน Part 52** — annotation
เดียวสามารถ "ห่อ" annotation หลายตัวไว้ด้วยกัน (`@SpringBootApplication`
เองก็ถูกกำกับด้วย `@Target`, `@Retention` แบบเดียวกับ custom annotation ที่
เราสร้างเอง)

## 4. Starter Dependencies

**Starter** คือ dependency ที่**รวม library ที่เกี่ยวข้องกันไว้เป็นชุด**
(ทบทวน transitive dependency จาก Part 61):

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

`spring-boot-starter-web` ดึงเข้ามาโดยอัตโนมัติ: Spring MVC (Part 77),
embedded Tomcat (หัวข้อ 6), Jackson (ทบทวนจาก Part 70), และอื่น ๆ อีกหลาย
ตัว — **ไม่ต้องประกาศ dependency เหล่านี้เองทีละตัว**

| Starter | รวม library อะไร |
|---|---|
| `spring-boot-starter-web` | Spring MVC, Tomcat, Jackson (สำหรับ REST API — Part 78) |
| `spring-boot-starter-data-jpa` | Hibernate, Spring Data JPA (Part 79-80) |
| `spring-boot-starter-security` | Spring Security (Part 82-83) |
| `spring-boot-starter-test` | JUnit 5, Mockito, AssertJ (ทบทวนจาก Part 58-59) |
| `spring-boot-starter-validation` | Bean Validation (Part 81) |

## 5. Auto-configuration ทำงานอย่างไร

**Auto-configuration** คือกลไกที่ Spring Boot **ตรวจสอบ classpath**
(ทบทวนจาก Part 20, 69) แล้ว**สร้าง bean ที่จำเป็นให้อัตโนมัติ**โดยไม่ต้อง
เขียน `@Bean` config เองเลย

```java
// ถ้า classpath มี H2 database driver (ทบทวน JDBC driver จาก Part 64)
// Spring Boot จะสร้าง DataSource bean ให้อัตโนมัติ (ทบทวน DataSource จาก Part 66)
// เทียบเท่ากับการเขียนโค้ดนี้เอง แต่ Spring Boot ทำให้แล้ว:
@Configuration
public class ManualDataSourceConfig {
    @Bean
    public DataSource dataSource() {
        // Spring Boot ตรวจจับ H2 บน classpath แล้วสร้าง config นี้ให้อัตโนมัติ
        return new com.zaxxer.hikari.HikariDataSource(); // ทบทวนจาก Part 66
    }
}
```

**หลักการภายใน**: Auto-configuration ใช้ **`@ConditionalOnClass`** (สร้าง
bean เฉพาะเมื่อ class นั้นมีอยู่บน classpath — ทบทวนแนวคิด conditional logic
คล้าย `if` ใน Part 5 แต่ทำงานที่ระดับ configuration) และ
**`@ConditionalOnMissingBean`** (สร้าง bean เฉพาะเมื่อยังไม่มีใครสร้างไว้
แล้ว — ให้ผู้ใช้ override ได้เสมอถ้าต้องการ config เอง)

```bash
# ดูรายการ auto-configuration ที่ Spring Boot ใช้ (มีประโยชน์มากตอน debug)
java -jar myapp.jar --debug
```

## 6. Embedded Server

Spring Boot **ฝัง web server ไว้ในตัว JAR** (ทบทวนจาก Part 72) — ค่า
default คือ **Tomcat** (เปลี่ยนเป็น Jetty หรือ Undertow ได้ผ่าน dependency)

```xml
<!-- เปลี่ยนจาก Tomcat เป็น Jetty (exclude Tomcat ก่อน แล้วเพิ่ม Jetty) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```

```bash
mvn clean package                     # สร้าง fat JAR (ทบทวนจาก Part 61)
java -jar target/myapp-1.0.0.jar       # รันได้ทันที ไม่ต้อง deploy ไปยัง Tomcat แยก!
# Started MyApplication in 2.345 seconds
# Tomcat started on port(s): 8080 (http)
```

**"Fat JAR"** (หรือ "Uber JAR"): JAR ที่รวม**ทุก dependency ไว้ในไฟล์เดียว**
(ทบทวนแนวคิด JAR จาก Part 20, 61) — ทำให้ deploy ง่ายมาก (แค่ไฟล์เดียว รันได้
ทุกที่ที่มี JRE)

## 7. โครงสร้างโปรเจกต์ Spring Boot มาตรฐาน

```
my-app/
├── pom.xml
└── src/
    ├── main/
    │   ├── java/com/example/myapp/
    │   │   ├── MyApplication.java          <- entry point (@SpringBootApplication)
    │   │   ├── controller/                  <- Web layer (Part 77-78)
    │   │   ├── service/                      <- Business logic layer
    │   │   ├── repository/                    <- Data access layer (Part 79)
    │   │   └── model/                          <- Entity/DTO
    │   └── resources/
    │       ├── application.properties           <- Configuration (หัวข้อ 8)
    │       ├── static/                            <- static files (CSS, JS)
    │       └── templates/                          <- Thymeleaf templates (Part 77)
    └── test/
        └── java/com/example/myapp/               <- test code (ทบทวนจาก Part 58-59)
```

**สังเกต**: นี่คือการนำ**Package by Layer** (ทบทวนจาก Part 20) มาประยุกต์ใช้
กับ Spring Boot — โปรเจกต์ขนาดใหญ่มักผสมกับ **Package by Feature** ด้วย

## 8. `application.properties` vs `application.yml`

```properties
# application.properties
server.port=8081
spring.datasource.url=jdbc:mysql://localhost:3306/mydb
spring.datasource.username=root
spring.jpa.hibernate.ddl-auto=update
logging.level.com.example=DEBUG
```

```yaml
# application.yml (โครงสร้างแบบ hierarchical - อ่านง่ายกว่าเมื่อ config ซับซ้อน)
server:
  port: 8081

spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb
    username: root
  jpa:
    hibernate:
      ddl-auto: update

logging:
  level:
    com.example: DEBUG
```

**ทั้งสองแบบใช้ key เดียวกัน** เพียงแค่ syntax ต่างกัน — YAML นิยมมากกว่า
เมื่อ config มีโครงสร้างซับซ้อน (nested มาก) เพราะไม่ต้องพิมพ์ prefix ซ้ำ
(`spring.datasource.` ซ้ำหลายบรรทัด) — **ห้ามใช้ tab ใน YAML เด็ดขาด**
(ต้องใช้ space เท่านั้น เป็นกับดักคลาสสิกของมือใหม่)

## 9. Spring Boot DevTools

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <scope>runtime</scope>
    <optional>true</optional>
</dependency>
```

**DevTools** เปิดใช้ **automatic restart** — เมื่อแก้ไขโค้ดและ save,
แอปพลิเคชัน**restart ให้อัตโนมัติ**เร็วกว่าการปิด-เปิด JVM ใหม่ทั้งหมด
(ทบทวน classloading จาก Part 20, 67 — DevTools ใช้ 2 classloader แยกกัน
สำหรับ code ที่เปลี่ยนบ่อยกับที่ไม่เปลี่ยน) — มีประโยชน์มากตอน development
แต่**ควร exclude ออกจาก production build เสมอ** (`optional=true` ช่วยเรื่อง
นี้บางส่วน)

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียน `application.yml` ที่กำหนด server port เป็น 9090 และตั้งค่า
logging level ของ package `com.example.myapp.service` เป็น `DEBUG`

**เฉลย:**

```yaml
server:
  port: 9090

logging:
  level:
    com.example.myapp.service: DEBUG
```

**2)** อธิบายว่า `@SpringBootApplication` เทียบเท่ากับการเขียน annotation
อะไร 3 ตัว และแต่ละตัวทำหน้าที่อะไร

**เฉลย**: เทียบเท่ากับ `@SpringBootConfiguration` (ทำหน้าที่เหมือน
`@Configuration` — ประกาศว่า class นี้เป็นแหล่งของ bean definition, ทบทวน
จาก Part 73), `@EnableAutoConfiguration` (เปิดใช้งานกลไก auto-configuration
ที่ตรวจสอบ classpath แล้วสร้าง bean ที่จำเป็นให้อัตโนมัติ ตามหัวข้อ 5), และ
`@ComponentScan` (สั่งให้ Spring สแกนหา `@Component`, `@Service`,
`@Repository` ฯลฯ ใน package ของ class นี้และ package ย่อยทั้งหมด — ทบทวน
จาก Part 73)

**3)** อธิบายว่าทำไม "Fat JAR" ทำให้ deploy แอปพลิเคชัน Spring Boot ง่ายกว่า
การ deploy WAR แบบเดิม

**เฉลย**: WAR แบบเดิม (ทบทวนจาก Part 72) ต้อง**ติดตั้ง Servlet Container
(Tomcat) แยกต่างหาก**บนเครื่อง server ก่อน แล้วค่อย deploy ไฟล์ WAR เข้าไปใน
container นั้น — ต้องจัดการ version ของ Tomcat ให้ตรงกับที่แอปพลิเคชัน
ต้องการ และต้องดูแล container แยกจากแอปพลิเคชัน ในขณะที่ Fat JAR **รวม
embedded server (Tomcat) ไว้ในไฟล์เดียวกับแอปพลิเคชันเอง** (ทบทวนหัวข้อ 6)
ทำให้ deploy คือแค่**คัดลอกไฟล์ JAR ไฟล์เดียวไปที่ server แล้วรันด้วย `java
-jar`** — ไม่ต้องติดตั้งหรือจัดการ container แยก ลดความซับซ้อนในการ deploy
ลงอย่างมาก และเหมาะกับการ deploy ผ่าน container/cloud (Docker — Part 95)
ที่นิยมใช้ "self-contained" artifact แบบนี้

### สรุปเนื้อหา Part 75

- Spring Boot ใช้หลักการ "Convention over Configuration" ลด boilerplate
  ของ Spring แบบดั้งเดิมอย่างมาก
- `@SpringBootApplication` รวม `@SpringBootConfiguration` +
  `@EnableAutoConfiguration` + `@ComponentScan`
- Starter dependencies รวม library ที่เกี่ยวข้องกันไว้เป็นชุด ลดการประกาศ
  dependency ทีละตัว
- Auto-configuration ตรวจสอบ classpath แล้วสร้าง bean ที่จำเป็นให้
  อัตโนมัติ (ใช้ `@ConditionalOnClass`/`@ConditionalOnMissingBean`)
- Embedded server (Tomcat) ทำให้ deploy เป็น Fat JAR เดียว รันด้วย `java
  -jar` ได้ทันที
- `application.properties`/`application.yml` แยก configuration จาก
  compiled code — YAML เหมาะกับ config ที่ซับซ้อน

**ต่อไป**: [Part 76 — Spring Boot: Configuration Properties, Profiles](./part-076-spring-boot-config.md)
