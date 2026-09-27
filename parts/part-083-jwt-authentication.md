# Part 83: Spring Security: JWT Authentication แบบเต็มระบบ

> ขั้นตอนที่ 821-830 ของหลักสูตร | ระดับ: สูงมาก

## สารบัญ

1. ทำไม REST API ไม่เหมาะกับ Session-based Authentication
2. JWT คืออะไร: โครงสร้าง 3 ส่วน
3. JWT ทำงานอย่างไรในทางปฏิบัติ
4. การสร้าง JWT ด้วย Java (jjwt library)
5. การตรวจสอบ (Verify) JWT
6. `JwtAuthenticationFilter`: เชื่อม JWT กับ Spring Security
7. Login Endpoint ที่คืนค่า JWT
8. Refresh Token Pattern
9. ข้อควรระวังด้านความปลอดภัยของ JWT
10. แบบฝึกหัดและสรุป

---

## 1. ทำไม REST API ไม่เหมาะกับ Session-based Authentication

ทบทวนจาก Part 71: **Session-based authentication** (ผ่าน cookie) ต้องให้
**server เก็บสถานะ (session)** ไว้ — ปัญหาคือใน**สถาปัตยกรรม
microservices** (ปูทางสู่ Part 91) ที่มีหลาย server instance รับ request
พร้อมกัน server ตัวหนึ่งอาจไม่รู้จัก session ที่สร้างจาก server อีกตัว
(ต้องใช้ shared session store เพิ่ม ซึ่งซับซ้อนและมี cost)

**JWT (JSON Web Token)** แก้ปัญหานี้ด้วยแนวคิด **stateless
authentication**: **ข้อมูลผู้ใช้ทั้งหมดอยู่ใน token เอง** ไม่ต้องเก็บ
session ฝั่ง server เลย — server ไหนก็ตรวจสอบ token ได้โดยไม่ต้องพึ่งพา
state ที่แชร์กัน (ทบทวนหลักการ statelessness ของ HTTP จาก Part 71 — JWT ทำ
ให้ authentication สอดคล้องกับธรรมชาติ stateless ของ HTTP อย่างสมบูรณ์)

## 2. JWT คืออะไร: โครงสร้าง 3 ส่วน

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhbGljZSIsInJvbGUiOiJVU0VSIn0.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
└──────── Header ────────┘└──────────── Payload ─────────────┘└──────────── Signature ────────────┘
```

แต่ละส่วนคือ **Base64URL-encoded** (ไม่ใช่การเข้ารหัส — ใครก็ decode ดูได้!
— ทบทวนความแตกต่างระหว่าง encoding และ encryption ที่สำคัญมากในหัวข้อ 9)

```java
public class JwtStructureDemo {
    public static void main(String[] args) {
        // Header: บอกว่าใช้ algorithm อะไร signature
        String header = "{\"alg\":\"HS256\",\"typ\":\"JWT\"}";

        // Payload (Claims): ข้อมูลผู้ใช้ - "sub" (subject/username), "role", "exp" (expiration)
        String payload = "{\"sub\":\"alice\",\"role\":\"USER\",\"exp\":1735689600}";

        // Signature: HMAC-SHA256(header + payload, secretKey) - ป้องกันการปลอมแปลง
        // ถ้าใครแก้ไข payload โดยไม่มี secret key, signature จะไม่ตรงกัน -> server ปฏิเสธ token ทันที
    }
}
```

## 3. JWT ทำงานอย่างไรในทางปฏิบัติ

```
1. Client login (POST /auth/login พร้อม username/password)
        │
        ▼
2. Server ตรวจสอบ credential (ทบทวน Part 82 - UserDetailsService, BCrypt)
        │
        ▼
3. Server สร้าง JWT (มี username, role, expiration) เซ็นด้วย secret key
        │
        ▼
4. Server ส่ง JWT กลับไปให้ client
        │
        ▼
5. Client เก็บ JWT ไว้ (localStorage/memory) แล้วส่งแนบไปกับทุก request ถัดไป
        Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
        │
        ▼
6. Server ตรวจสอบ signature ของ JWT (ไม่ต้อง query database/session store!)
        │
        ▼
7. ถ้า signature ถูกต้องและยังไม่ expired -> อนุญาตให้เข้าถึง resource
```

## 4. การสร้าง JWT ด้วย Java (jjwt library)

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.3</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.3</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.3</version>
    <scope>runtime</scope>
</dependency>
```

```java
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.security.Keys;
import javax.crypto.SecretKey;
import java.util.Date;

public class JwtGenerationService {
    private final SecretKey secretKey = Keys.hmacShaKeyFor(
        "this-is-a-very-long-secret-key-that-should-be-at-least-256-bits".getBytes()
    ); // ทบทวนความสำคัญของ secret key ที่ยาวและปลอดภัยในหัวข้อ 9

    public String generateToken(String username, String role) {
        long now = System.currentTimeMillis();
        long expirationMs = 3600_000; // 1 ชั่วโมง (ทบทวน millisecond จาก Part 44)

        return Jwts.builder()
                .subject(username)
                .claim("role", role) // claim แบบกำหนดเอง (ทบทวนแนวคิด Map จาก Part 24)
                .issuedAt(new Date(now))
                .expiration(new Date(now + expirationMs))
                .signWith(secretKey) // เซ็น signature ด้วย secret key
                .compact(); // สร้าง JWT string สุดท้าย
    }
}
```

## 5. การตรวจสอบ (Verify) JWT

```java
import io.jsonwebtoken.Claims;
import io.jsonwebtoken.JwtException;
import io.jsonwebtoken.Jwts;
import javax.crypto.SecretKey;
import java.util.Optional;

public class JwtVerificationService {
    private final SecretKey secretKey; // ต้องเป็น secret key เดียวกับที่ใช้สร้าง token

    public JwtVerificationService(SecretKey secretKey) { this.secretKey = secretKey; }

    public Optional<Claims> verifyToken(String token) {
        try {
            Claims claims = Jwts.parser()
                    .verifyWith(secretKey)
                    .build()
                    .parseSignedClaims(token) // ตรวจสอบ signature + decode payload ในขั้นตอนเดียว
                    .getPayload();
            return Optional.of(claims); // ทบทวน Optional จาก Part 43 - ปลอดภัยกว่า throw exception ตรง ๆ
        } catch (JwtException e) { // signature ผิด, expired, malformed - ทั้งหมดจับได้ในนี้ (ทบทวน Part 10, 21)
            return Optional.empty();
        }
    }
}
```

**ข้อสำคัญ**: `parseSignedClaims()` **ตรวจสอบ signature โดยอัตโนมัติ** —
ถ้ามีใครแก้ไข payload (เช่น เปลี่ยน role จาก "USER" เป็น "ADMIN") โดยไม่มี
secret key ที่ถูกต้อง signature จะไม่ตรงกันและ throw `JwtException` ทันที
— นี่คือกลไกป้องกันการปลอมแปลงที่สำคัญที่สุดของ JWT

## 6. `JwtAuthenticationFilter`: เชื่อม JWT กับ Spring Security

ทบทวนแนวคิด Filter จาก Part 72, 82: สร้าง Filter ที่**ดักทุก request**
ตรวจสอบ JWT ใน header **ก่อน**ที่จะไปถึง Controller

```java
import io.jsonwebtoken.Claims;
import jakarta.servlet.*;
import jakarta.servlet.http.*;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;
import java.io.IOException;
import java.util.List;
import java.util.Optional;

@Component
public class JwtAuthenticationFilter extends jakarta.servlet.http.HttpFilter {
    private final JwtVerificationService jwtService;

    public JwtAuthenticationFilter(JwtVerificationService jwtService) {
        this.jwtService = jwtService;
    }

    @Override
    protected void doFilter(HttpServletRequest req, HttpServletResponse resp, FilterChain chain)
            throws IOException, ServletException {

        String authHeader = req.getHeader("Authorization"); // ทบทวน HTTP Headers จาก Part 71

        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            String token = authHeader.substring(7); // ตัด "Bearer " ออก
            Optional<Claims> claimsOpt = jwtService.verifyToken(token);

            if (claimsOpt.isPresent()) {
                Claims claims = claimsOpt.get();
                String username = claims.getSubject();
                String role = claims.get("role", String.class);

                var authentication = new UsernamePasswordAuthenticationToken(
                    username, null, List.of(new SimpleGrantedAuthority("ROLE_" + role))
                ); // ทบทวน Role vs Authority จาก Part 82

                SecurityContextHolder.getContext().setAuthentication(authentication);
                // ตั้งค่า authentication ให้ Spring Security รู้ว่า request นี้คือใคร
            }
        }

        chain.doFilter(req, resp); // ส่งต่อไป filter/controller ถัดไป (ทบทวน Chain of Responsibility จาก Part 56, 72)
    }
}
```

```java
@Configuration
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http, JwtAuthenticationFilter jwtFilter)
            throws Exception {
        http
            .csrf(csrf -> csrf.disable()) // JWT เป็น stateless ไม่ต้องพึ่ง CSRF token (ทบทวนจาก Part 82)
            .sessionManagement(session -> session
                .sessionCreationPolicy(org.springframework.security.config.http.SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/auth/login", "/auth/register").permitAll()
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtFilter,
                org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }
}
```

## 7. Login Endpoint ที่คืนค่า JWT

```java
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/auth")
public class AuthController {
    private final AuthenticationManager authenticationManager;
    private final JwtGenerationService jwtService;

    public AuthController(AuthenticationManager authenticationManager, JwtGenerationService jwtService) {
        this.authenticationManager = authenticationManager;
        this.jwtService = jwtService;
    }

    record LoginRequest(String username, String password) {}
    record LoginResponse(String token) {}

    @PostMapping("/login")
    public LoginResponse login(@RequestBody LoginRequest request) {
        // ใช้ AuthenticationManager ตรวจสอบ credential ผ่าน UserDetailsService (ทบทวนจาก Part 82)
        authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(request.username(), request.password())
        ); // throw exception ถ้า credential ผิด (จับได้ที่ @RestControllerAdvice - ทบทวน Part 78)

        String token = jwtService.generateToken(request.username(), "USER");
        return new LoginResponse(token);
    }
}
```

## 8. Refresh Token Pattern

JWT ที่มีอายุสั้น (เช่น 15 นาที) ปลอดภัยกว่า (ถ้าถูกขโมยไป ใช้ได้ไม่นาน) แต่
ผู้ใช้ต้อง login ใหม่บ่อยเกินไป — **Refresh Token** แก้ปัญหานี้:

```java
public class RefreshTokenPatternDemo {
    // Access Token: อายุสั้น (15 นาที) ใช้เข้าถึง resource ทั่วไป
    // Refresh Token: อายุยาว (7 วัน) ใช้แค่แลกเป็น Access Token ใหม่เท่านั้น เก็บอย่างปลอดภัยกว่า

    record TokenPair(String accessToken, String refreshToken) {}

    TokenPair login(String username) {
        String accessToken = "..."; // สร้างด้วย JwtGenerationService (หัวข้อ 4) อายุ 15 นาที
        String refreshToken = "..."; // อายุยาวกว่า, เก็บใน database เพื่อเช็คว่าถูก revoke หรือยัง
        return new TokenPair(accessToken, refreshToken);
    }

    String refreshAccessToken(String refreshToken) {
        // ตรวจสอบ refreshToken ว่ายังไม่ expired และไม่ถูก revoke (เช็คกับฐานข้อมูล - ทบทวน Part 79)
        // ถ้าถูกต้อง -> สร้าง Access Token ใหม่กลับไป (ไม่ต้อง login ด้วย password ซ้ำ)
        return "new-access-token";
    }
}
```

**ทำไมต้องมี 2 token**: **Access Token** อายุสั้นลด**ความเสี่ยง**ถ้าถูก
ขโมย (หมดอายุเร็ว), **Refresh Token** อายุยาวแต่**ใช้ได้แค่ endpoint เดียว**
(`/auth/refresh`) และ**เก็บใน database** ทำให้ **revoke ได้** (ถ้าผู้ใช้
logout หรือสงสัยว่าถูกขโมย สามารถลบ refresh token นั้นออกจากฐานข้อมูล
ทำให้ใช้ต่อไม่ได้ทันที — แก้ข้อจำกัดของ JWT ล้วน ๆ ที่ revoke ยาก ทบทวน
หัวข้อ 9)

## 9. ข้อควรระวังด้านความปลอดภัยของ JWT

1. **JWT ไม่ได้เข้ารหัส (encrypt) เพียง encode + sign**: **ห้ามเก็บข้อมูล
   sensitive** (password, credit card) ใน payload เด็ดขาด — ใครก็ decode
   ดูได้ (ทบทวนความแตกต่าง encoding/encryption จากหัวข้อ 2)

```java
public class JwtDecodeAnyoneCanReadDemo {
    public static void main(String[] args) {
        // ใครก็ paste JWT ไปที่ jwt.io แล้วเห็น payload ได้เลย โดยไม่ต้องมี secret key
        // secret key จำเป็นแค่สำหรับ "verify signature" ไม่ใช่สำหรับ "อ่านข้อมูล"
    }
}
```

2. **Secret Key ต้องยาวและสุ่มพอ**: (ทบทวนจากหัวข้อ 4) ถ้า secret key สั้น
   หรือเดาง่าย ผู้โจมตีสามารถ brute-force หา secret key แล้วปลอมแปลง JWT
   ของตัวเองได้ (ทบทวนแนวคิด brute-force จาก Part 29, 82)
3. **JWT ที่ออกไปแล้ว revoke ไม่ได้ง่าย ๆ** (จนกว่าจะ expired) — นี่คือ
   ข้อจำกัดสำคัญที่ Refresh Token Pattern (หัวข้อ 8) ช่วยบรรเทา
4. **เก็บ Access Token ฝั่ง client อย่างระมัดระวัง**: `localStorage` เสี่ยง
   ต่อ XSS attack (ถ้ามี JavaScript อันตรายรันบนหน้าเว็บ อ่าน localStorage
   ได้) — `httpOnly cookie` (ทบทวนจาก Part 71) ปลอดภัยกว่าในหลายกรณี
5. **ใช้ HTTPS เสมอ** (ทบทวนจาก Part 71) — ถ้าส่ง JWT ผ่าน HTTP ธรรมดา
   ใครดักฟังก็เอา token ไปใช้แทนตัวจริงได้ทันที (session hijacking)

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียนเมธอด `extractUsername(String token)` ที่ใช้ jjwt ดึงค่า
username จาก JWT โดยไม่ต้อง verify signature ก่อน (เช่น เพื่อ log เฉย ๆ)

**เฉลย (แนวคิด):** ควรใช้ `parseSignedClaims()` เสมอแม้จะแค่ต้องการ log
เพื่อป้องกันการอ่าน payload ของ token ที่ปลอมแปลง — ไม่ควรมีเมธอดที่ "ไม่
verify" ในระบบจริง เพราะเสี่ยงให้ผู้โจมตีปลอมแปลง claim แล้วให้ระบบเชื่อโดย
ไม่ตรวจสอบ

**2)** อธิบายว่าทำไม access token ควรมีอายุสั้น แต่ refresh token มีอายุยาว
ได้

**เฉลย**: Access token ถูกส่งไปพร้อมกับ**ทุก request**ไปยัง API (ทบทวนจาก
หัวข้อ 3) ทำให้มี**ความเสี่ยงถูกดักฟังหรือขโมยสูงกว่า**ในทางสถิติ (ยิ่งใช้
บ่อย ยิ่งมีโอกาสรั่วไหล) — อายุสั้นจึงจำกัด**ความเสียหายสูงสุด**ถ้าถูกขโมย
(หมดอายุเร็ว ใช้ต่อไม่ได้) ในขณะที่ refresh token ถูกส่งไป**แค่ endpoint
เดียว** (`/auth/refresh`) ไม่บ่อยเท่า access token และ**เก็บไว้ในฐานข้อมูล
ฝั่ง server** ทำให้ revoke ได้ทันทีถ้าสงสัยว่าถูกขโมย (ทบทวนหัวข้อ 8) —
ความเสี่ยงโดยรวมของ refresh token จึงต่ำกว่า ทำให้ยอมรับอายุที่ยาวกว่าได้

**3)** อธิบายว่าทำไมการเก็บข้อมูล credit card number ไว้ใน JWT payload
เป็นความคิดที่แย่มาก

**เฉลย**: JWT payload เป็นแค่ **Base64URL encoding** ไม่ใช่การเข้ารหัส
(ทบทวนหัวข้อ 2, 9) — **ใครก็ตามที่มี JWT string นั้นสามารถ decode ดูเนื้อหา
ใน payload ได้ทันที**โดยไม่ต้องมี secret key เลย (secret key ใช้แค่ verify
signature ว่าไม่ถูกแก้ไข ไม่ได้ใช้ป้องกันการอ่านเนื้อหา) — ถ้า JWT ถูกส่งผ่าน
network ที่ไม่ปลอดภัย ถูก log ไว้โดยไม่ตั้งใจ หรือรั่วไหลผ่านช่องทางใดก็ตาม
credit card number ที่อยู่ใน payload จะถูกเปิดเผยทันที นี่คือเหตุผลที่ข้อมูล
sensitive ต้องเก็บไว้ในฐานข้อมูลฝั่ง server เท่านั้น (ที่มีการควบคุมการ
เข้าถึงอย่างเข้มงวด) โดย JWT ควรมีแค่ identifier (เช่น user ID) ที่ใช้ค้นหา
ข้อมูลนั้นจากฐานข้อมูลอีกที ไม่ใช่เก็บข้อมูล sensitive ไว้ตรง ๆ

### สรุปเนื้อหา Part 83

- JWT เหมาะกับ REST API/microservices เพราะเป็น stateless authentication
  ไม่ต้องเก็บ session ฝั่ง server
- JWT มี 3 ส่วน: Header, Payload, Signature — encode ด้วย Base64URL ไม่ใช่
  เข้ารหัส (ใครก็อ่าน payload ได้)
- Signature ป้องกันการปลอมแปลง แต่ไม่ป้องกันการอ่านเนื้อหา — ห้ามเก็บข้อมูล
  sensitive ใน payload
- `JwtAuthenticationFilter` ตรวจสอบ token ในทุก request ก่อนถึง Controller
  (ทบทวน Filter Chain จาก Part 72, 82)
- Refresh Token Pattern แก้ปัญหา JWT revoke ยาก — access token อายุสั้น,
  refresh token อายุยาวแต่ revoke ได้ผ่านฐานข้อมูล
- ต้องใช้ HTTPS เสมอ, secret key ต้องยาวและสุ่มพอ, ระมัดระวังการเก็บ token
  ฝั่ง client (XSS risk)

**ต่อไป**: [Part 84 — Spring Boot Testing: MockMvc, @SpringBootTest, Testcontainers](./part-084-spring-boot-testing.md)
