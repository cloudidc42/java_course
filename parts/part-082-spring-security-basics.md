# Part 82: Spring Security เบื้องต้น: Authentication, Authorization

> ขั้นตอนที่ 811-820 ของหลักสูตร | ระดับ: สูงมาก

## สารบัญ

1. Authentication vs Authorization (ทบทวนความแตกต่างจาก Part 71)
2. Spring Security Filter Chain (ทบทวน Filter จาก Part 72)
3. การตั้งค่าเริ่มต้น: ค่า Default ของ Spring Security
4. `SecurityFilterChain`: ตั้งค่าเอง
5. `UserDetailsService` และ `UserDetails`
6. Password Encoding: `BCryptPasswordEncoder`
7. Authorization: `hasRole`, `hasAuthority`
8. Method-level Security: `@PreAuthorize`
9. Custom `AuthenticationProvider`
10. แบบฝึกหัดและสรุป

---

## 1. Authentication vs Authorization (ทบทวนความแตกต่างจาก Part 71)

ทบทวนจาก Part 71: **Authentication** คือ **"คุณเป็นใคร"** (ยืนยันตัวตน —
เช่น login ด้วย username/password), **Authorization** คือ **"คุณทำสิ่งนี้
ได้หรือไม่"** (ตรวจสอบสิทธิ์ — เช่น admin เท่านั้นที่ลบ user ได้)

```
Authentication: "คุณคือ Alice จริงหรือไม่?" -> ตรวจสอบ password ผ่านหรือไม่
Authorization:  "Alice มีสิทธิ์ลบ user คนอื่นหรือไม่?" -> ตรวจสอบ role/permission
```

**Spring Security** เป็น framework มาตรฐานสำหรับทั้งสองเรื่องนี้ใน Spring
ecosystem — ทำงานผ่าน**Filter Chain** (ทบทวนแนวคิดจาก Part 72)

## 2. Spring Security Filter Chain (ทบทวน Filter จาก Part 72)

Spring Security เป็นชุดของ **Servlet Filter** (ทบทวนจาก Part 72) ที่ทำงาน
**ก่อน**ที่ request จะไปถึง `DispatcherServlet`/Controller (ทบทวนจาก Part
77):

```
Request ──> SecurityContextPersistenceFilter ──> UsernamePasswordAuthenticationFilter
       ──> ExceptionTranslationFilter ──> FilterSecurityInterceptor ──> DispatcherServlet ──> Controller
       (Filter Chain แบบ Chain of Responsibility - ทบทวนจาก Part 56, 72)
```

แต่ละ Filter มีหน้าที่เฉพาะ (ทบทวน SRP จาก Part 57): ตรวจสอบ session, ตรวจ
สอบ credential, ตรวจสอบสิทธิ์ — ถ้า filter ใดปฏิเสธ request จะหยุดทันที
(คืน `401`/`403` — ทบทวนจาก Part 71) ไม่ไปถึง Controller เลย

## 3. การตั้งค่าเริ่มต้น: ค่า Default ของ Spring Security

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

เพียงเพิ่ม dependency นี้ (ทบทวน auto-configuration จาก Part 75) Spring
Boot จะ:
1. **บล็อกทุก endpoint** โดย default (ต้อง login ก่อนเข้าถึงอะไรได้)
2. สร้าง**login form** อัตโนมัติที่ `/login`
3. สร้าง**user ชื่อ "user"** พร้อม password แบบสุ่ม (แสดงใน console log
   ตอน startup — ทบทวน logging จาก Part 63)

**นี่แสดงหลักการสำคัญ**: Spring Security ออกแบบให้**ปลอดภัยโดย default**
("secure by default") — ผู้พัฒนาต้อง**เปิดเผยอย่างชัดเจน**ว่า endpoint ไหน
ควรเข้าถึงได้แบบสาธารณะ (ทบทวนหลักการ "fail fast/fail safe" จาก Part 76)

## 4. `SecurityFilterChain`: ตั้งค่าเอง

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class SecurityConfig {

    @Bean // ทบทวนการประกาศ bean ด้วย Java Config จาก Part 73
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()   // ทุกคนเข้าถึงได้ไม่ต้อง login
                .requestMatchers("/api/admin/**").hasRole("ADMIN") // ต้องมี role ADMIN (หัวข้อ 7)
                .anyRequest().authenticated()                        // อื่น ๆ ต้อง login ก่อนเสมอ
            )
            .csrf(csrf -> csrf.disable()); // ปิด CSRF protection (เหมาะกับ stateless REST API - หัวข้อถัดไปจะอธิบาย)

        return http.build();
    }
}
```

**CSRF (Cross-Site Request Forgery)**: การโจมตีที่หลอกให้ browser ของ
ผู้ใช้ที่ login อยู่แล้ว ส่ง request ที่ไม่ได้ตั้งใจไปยัง server — Spring
Security ป้องกันด้วย CSRF token โดย default สำหรับ**session-based
authentication** แต่ REST API ที่ใช้ **JWT** (Part 83, stateless) มักปิด
CSRF protection เพราะไม่ได้ใช้ cookie/session แบบเดิม (ทบทวนความแตกต่างจาก
Part 71)

## 5. `UserDetailsService` และ `UserDetails`

**`UserDetailsService`** เป็น interface ที่**เราต้อง implement เอง**เพื่อ
บอก Spring Security ว่าจะ**หา user จากไหน** (ทบทวนแนวคิด DIP จาก Part 57 —
Spring Security depend on interface นี้ ไม่ใช่ implementation เฉพาะเจาะจง):

```java
import org.springframework.security.core.userdetails.*;
import org.springframework.stereotype.Service;

@Service
public class CustomUserDetailsService implements UserDetailsService {
    private final UserRepository userRepository; // ทบทวน Repository จาก Part 79

    public CustomUserDetailsService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        AppUser user = userRepository.findByUsername(username)
                .orElseThrow(() -> new UsernameNotFoundException("ไม่พบผู้ใช้: " + username));
                // ทบทวน Optional.orElseThrow() จาก Part 43

        return org.springframework.security.core.userdetails.User.builder()
                .username(user.getUsername())
                .password(user.getPasswordHash()) // ต้องเป็น hash ที่เข้ารหัสแล้ว (หัวข้อ 6)
                .roles(user.getRole()) // เช่น "USER", "ADMIN"
                .build();
    }
}
```

## 6. Password Encoding: `BCryptPasswordEncoder`

**กฎเหล็กที่สำคัญที่สุดของความปลอดภัย**: **ห้ามเก็บ password เป็น plain
text ในฐานข้อมูลเด็ดขาด** (ทบทวนความเสี่ยงจาก Part 38, 65 — ถ้าฐานข้อมูลรั่ว
ไหล password ทุกคนจะถูกเปิดเผยทันที)

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;

@Configuration
public class PasswordConfig {
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(); // BCrypt: มี salt ในตัว, คำนวณช้าโดยตั้งใจ (ป้องกัน brute-force)
    }
}
```

```java
@Service
public class UserRegistrationService {
    private final PasswordEncoder passwordEncoder;
    private final UserRepository userRepository;

    public UserRegistrationService(PasswordEncoder passwordEncoder, UserRepository userRepository) {
        this.passwordEncoder = passwordEncoder;
        this.userRepository = userRepository;
    }

    public void register(String username, String rawPassword) {
        String hashedPassword = passwordEncoder.encode(rawPassword); // hash ก่อนเก็บเสมอ!
        AppUser user = new AppUser(username, hashedPassword);
        userRepository.save(user);
    }
}
```

**ทำไม BCrypt ปลอดภัยกว่า hash function ทั่วไป (MD5, SHA-256)**: BCrypt
**ออกแบบให้คำนวณช้าโดยตั้งใจ** (adjustable "cost factor") — ทำให้ผู้โจมตีที่
พยายาม brute-force เดา password (ลองทุกความเป็นไปได้ — ทบทวนแนวคิด
brute-force จาก Part 29 backtracking) ต้องใช้เวลานานมากในการเดาแต่ละครั้ง
นอกจากนี้ BCrypt มี **salt** (ค่าสุ่มที่ผสมเข้ากับ password ก่อน hash) ในตัว
โดยอัตโนมัติ ทำให้ password เดียวกันได้ hash ที่**ต่างกัน**ทุกครั้ง ป้องกัน
การโจมตีแบบ **rainbow table** (ทบทวนแนวคิด hash จาก Part 23-24 แต่ในบริบท
ความปลอดภัยแทนโครงสร้างข้อมูล)

## 7. Authorization: `hasRole`, `hasAuthority`

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http.authorizeHttpRequests(auth -> auth
        .requestMatchers("/api/admin/**").hasRole("ADMIN")           // ต้องมี role "ADMIN"
        .requestMatchers("/api/reports/**").hasAnyRole("ADMIN", "MANAGER") // มีอย่างน้อย 1 ใน 2 role
        .requestMatchers("/api/products/delete").hasAuthority("PRODUCT_DELETE") // authority ละเอียดกว่า role
        .anyRequest().authenticated()
    );
    return http.build();
}
```

**Role vs Authority**: **Role** เป็นแนวคิดกว้าง ๆ (`ADMIN`, `USER`) —
ภายใน Spring Security แท้จริงแล้ว role คือ authority ที่มี prefix `ROLE_`
(`hasRole("ADMIN")` เทียบเท่า `hasAuthority("ROLE_ADMIN")`) — **Authority**
ละเอียดกว่า สามารถกำหนดสิทธิ์เฉพาะเจาะจง (`PRODUCT_DELETE`,
`REPORT_EXPORT`) ให้ยืดหยุ่นกว่าการใช้ role อย่างเดียว (คล้ายแนวคิด
Interface Segregation Principle จาก Part 57 — แยกสิทธิ์ย่อยแทนใช้ role
ใหญ่ที่ครอบคลุมเกินจำเป็น)

## 8. Method-level Security: `@PreAuthorize`

นอกจาก config ที่ URL level (หัวข้อ 4) ยังตรวจสอบสิทธิ์ที่**ระดับเมธอด**ได้
(ทบทวนแนวคิด AOP/cross-cutting concern จาก Part 72, 79):

```java
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Service;

@Service
public class ProductAdminService {

    @PreAuthorize("hasRole('ADMIN')") // ตรวจสอบก่อนเข้าเมธอดนี้ (คล้าย @Valid จาก Part 81 แต่เป็นเรื่อง security)
    public void deleteProduct(Long id) {
        System.out.println("ลบสินค้า id=" + id);
    }

    @PreAuthorize("hasRole('ADMIN') or #username == authentication.name") // เงื่อนไขซับซ้อนได้ (SpEL)
    public void updateProfile(String username) {
        System.out.println("แก้ไขข้อมูลของ: " + username);
        // อนุญาตถ้าเป็น ADMIN หรือแก้ไขข้อมูลของตัวเองเท่านั้น
    }
}
```

ต้องเปิดใช้งานก่อนด้วย `@EnableMethodSecurity`:

```java
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;

@Configuration
@EnableMethodSecurity // เปิดใช้ @PreAuthorize, @PostAuthorize
public class MethodSecurityConfig { }
```

**ข้อดีของ method-level security**: ป้องกันได้แม้เรียกเมธอดนี้จาก**ที่ไหน
ก็ตาม**ในโค้ด (ไม่ใช่แค่จาก HTTP request) — เข้มงวดกว่าและปลอดภัยกว่า URL-
level config เพียงอย่างเดียว (defense in depth — มีหลายชั้นป้องกัน)

## 9. Custom `AuthenticationProvider`

สำหรับ logic การยืนยันตัวตนที่ซับซ้อนกว่า username/password ธรรมดา (เช่น
ตรวจสอบ 2FA, external identity provider):

```java
import org.springframework.security.authentication.*;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.AuthenticationException;
import org.springframework.stereotype.Component;

@Component
public class CustomAuthenticationProvider implements AuthenticationProvider {

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        String username = authentication.getName();
        String password = authentication.getCredentials().toString();

        // custom logic การตรวจสอบ (เช่น เช็คกับ external system, ตรวจ 2FA)
        if (isValidUser(username, password)) {
            return new UsernamePasswordAuthenticationToken(
                username, password, java.util.List.of()); // สร้าง Authentication ที่สำเร็จ
        }
        throw new BadCredentialsException("Username หรือ password ไม่ถูกต้อง");
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication);
    }

    boolean isValidUser(String username, String password) { return true; } // placeholder
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียน `SecurityFilterChain` config ที่อนุญาต `/api/products` (GET)
สำหรับทุกคน แต่ต้อง login สำหรับ POST/PUT/DELETE

**เฉลย:**

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http.authorizeHttpRequests(auth -> auth
        .requestMatchers(HttpMethod.GET, "/api/products").permitAll()
        .requestMatchers(HttpMethod.POST, "/api/products").authenticated()
        .requestMatchers(HttpMethod.PUT, "/api/products/**").authenticated()
        .requestMatchers(HttpMethod.DELETE, "/api/products/**").authenticated()
        .anyRequest().denyAll()
    );
    return http.build();
}
```

**2)** เขียนเมธอด `register()` ที่ใช้ `BCryptPasswordEncoder` เข้ารหัส
password ก่อนบันทึก แล้วเขียนเมธอด `login()` ที่ใช้ `matches()` ตรวจสอบ
password ที่ผู้ใช้กรอกกับ hash ที่เก็บไว้

**เฉลย:**

```java
@Service
public class AuthService {
    private final PasswordEncoder encoder = new BCryptPasswordEncoder();

    public String register(String rawPassword) {
        return encoder.encode(rawPassword); // เก็บค่านี้ลงฐานข้อมูล
    }

    public boolean login(String rawPassword, String storedHash) {
        return encoder.matches(rawPassword, storedHash); // เทียบโดยไม่ต้อง decode hash กลับมา
    }
}
```

**3)** อธิบายว่าทำไม BCrypt ที่ "คำนวณช้า" ถึงเป็นคุณสมบัติที่ดี ไม่ใช่
ข้อเสีย

**เฉลย**: สำหรับผู้ใช้ทั่วไป ความช้าของ BCrypt (มักอยู่ที่หลักร้อยมิลลิวินาที
ต่อครั้ง) **ไม่มีผลกระทบที่สังเกตได้เลย** เพราะ login เกิดขึ้นไม่บ่อย แต่
สำหรับผู้โจมตีที่พยายาม brute-force เดา password (ทบทวนแนวคิดจาก Part 29 —
ลองทุกความเป็นไปได้) ความช้านี้**ทวีคูณอย่างมหาศาล**เมื่อต้องลองเดาหลาย
ล้าน/พันล้านครั้ง — ถ้า hash function เร็วมาก (เช่น MD5 ที่คำนวณได้เป็นล้าน
ครั้งต่อวินาทีด้วย hardware ทั่วไป) ผู้โจมตีจะเดา password ได้เร็วกว่า BCrypt
หลายพันเท่า BCrypt จึงแลก**ความช้าเล็กน้อยสำหรับผู้ใช้จริง**กับ**ความยากใน
การโจมตีที่เพิ่มขึ้นอย่างมหาศาล**ซึ่งเป็นการแลกเปลี่ยนที่คุ้มค่ามากในบริบท
ความปลอดภัย

### สรุปเนื้อหา Part 82

- Authentication ตรวจสอบตัวตน, Authorization ตรวจสอบสิทธิ์ — Spring
  Security จัดการทั้งสองผ่าน Filter Chain
- ค่า default ของ Spring Security คือบล็อกทุกอย่างก่อน (secure by default)
- `UserDetailsService` เป็น interface ที่ implement เองเพื่อบอกว่าหา user
  จากไหน
- **ห้ามเก็บ password เป็น plain text เด็ดขาด** — ใช้ `BCryptPasswordEncoder`
  เสมอ (มี salt + คำนวณช้าโดยตั้งใจ)
- `hasRole`/`hasAuthority` ควบคุม authorization ที่ URL level,
  `@PreAuthorize` ควบคุมที่ method level
- Custom `AuthenticationProvider` ใช้เมื่อ logic ยืนยันตัวตนซับซ้อนกว่า
  username/password ธรรมดา

**ต่อไป**: [Part 83 — Spring Security: JWT Authentication แบบเต็มระบบ](./part-083-jwt-authentication.md)
