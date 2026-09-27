# Part 81: Spring Boot: Validation และ Global Exception Handling

> ขั้นตอนที่ 801-810 ของหลักสูตร | ระดับ: สูงมาก

## สารบัญ

1. ทำไมต้อง Validate ข้อมูลนำเข้า (ทบทวนจาก Part 71 — Client Error 4xx)
2. Bean Validation (Jakarta Validation) เบื้องต้น
3. Constraint Annotations ที่ใช้บ่อย
4. `@Valid` ใน Controller
5. การจัดการ Validation Error ด้วย `@RestControllerAdvice`
6. Custom Validation Annotation
7. Validation แบบ Group และ Nested Object
8. Validation ที่ Service Layer
9. Cross-field Validation
10. แบบฝึกหัดและสรุป

---

## 1. ทำไมต้อง Validate ข้อมูลนำเข้า (ทบทวนจาก Part 71 — Client Error 4xx)

ทบทวนหลักการจาก Part 12-13 (validation ใน constructor/setter) และ Part 71
(HTTP 400 Bad Request): **ข้อมูลที่มาจากภายนอก (client) ไม่สามารถเชื่อถือได้
เลย** — ต้อง validate ก่อนประมวลผลเสมอ เพื่อป้องกันข้อมูลผิดพลาดแพร่กระจาย
เข้าไปใน business logic และฐานข้อมูล (ทบทวนความเสี่ยง SQL Injection จาก Part
64 — validation คือแนวป้องกันชั้นแรกที่สำคัญ)

## 2. Bean Validation (Jakarta Validation) เบื้องต้น

**Bean Validation** เป็น specification มาตรฐาน (เหมือน JPA — ทบทวนจาก Part
79) — Spring Boot ใช้ **Hibernate Validator** เป็น implementation

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

```java
import jakarta.validation.constraints.*;

public record CreateUserRequest(
    @NotBlank(message = "ชื่อผู้ใช้ห้ามว่าง")
    String username,

    @Email(message = "รูปแบบอีเมลไม่ถูกต้อง")
    String email,

    @Min(value = 18, message = "อายุต้องมากกว่าหรือเท่ากับ 18 ปี")
    int age
) {}
```

**นี่คือหลักการเดียวกันกับที่ทำเองด้วย custom annotation ใน Part 52** (จำ
`@NotBlank` ที่เราสร้างเองใน `SimpleValidator` หัวข้อ 9 ได้ไหม?) — Bean
Validation คือเวอร์ชันมาตรฐานและครบครันของแนวคิดนั้น

## 3. Constraint Annotations ที่ใช้บ่อย

| Annotation | ตรวจสอบว่า |
|---|---|
| `@NotNull` | ไม่เป็น `null` (ทบทวนปัญหา null จาก Part 43) |
| `@NotBlank` | String ไม่เป็น null, ไม่ว่าง, ไม่มีแต่ whitespace |
| `@NotEmpty` | Collection/String ไม่เป็น null และไม่ว่างเปล่า (ทบทวน Part 22) |
| `@Size(min=, max=)` | ความยาวของ String/Collection อยู่ในช่วงที่กำหนด |
| `@Min`/`@Max` | ค่าตัวเลขอยู่ในช่วงที่กำหนด |
| `@Positive`/`@Negative` | ตัวเลขเป็นบวก/ลบ |
| `@Email` | รูปแบบอีเมลถูกต้อง (ทบทวน regex จาก Part 45) |
| `@Pattern(regexp=)` | ตรงกับ regex ที่กำหนด (ทบทวนจาก Part 45) |
| `@Past`/`@Future` | วันที่อยู่ในอดีต/อนาคต (ทบทวน `java.time` จาก Part 44) |

```java
import jakarta.validation.constraints.*;
import java.time.LocalDate;
import java.util.List;

public record ProductRequest(
    @NotBlank @Size(min = 3, max = 100)
    String name,

    @Positive(message = "ราคาต้องมากกว่า 0")
    double price,

    @Pattern(regexp = "^[A-Z]{3}-\\d{4}$", message = "รูปแบบ SKU ต้องเป็น XXX-9999")
    String sku,

    @NotEmpty(message = "ต้องมีอย่างน้อย 1 หมวดหมู่")
    List<String> categories,

    @Past(message = "วันที่ผลิตต้องเป็นอดีต")
    LocalDate manufacturedDate
) {}
```

## 4. `@Valid` ใน Controller

```java
import jakarta.validation.Valid;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/users")
public class UserController {

    @PostMapping
    public ResponseEntity<String> createUser(@Valid @RequestBody CreateUserRequest request) {
        // ถ้าข้อมูลไม่ผ่าน validation, Spring จะ throw MethodArgumentNotValidException
        // "ก่อน" ที่จะเข้ามาในเมธอดนี้เลย - โค้ดในนี้มั่นใจได้ว่า request ผ่าน validation แล้วเสมอ
        return ResponseEntity.ok("สร้างผู้ใช้: " + request.username());
    }
}
```

**`@Valid`** ทำงานคล้าย **Filter** (ทบทวนจาก Part 72) หรือ **AOP** —
ตรวจสอบข้อมูล**ก่อน**ที่ logic ในเมธอดจะรัน ทำให้ Controller **ไม่ต้องเขียน
if-else ตรวจสอบเงื่อนไขเองเลย** (ทบทวนหลัก SRP จาก Part 57 — แยกความ
รับผิดชอบเรื่อง validation ออกจาก business logic อย่างชัดเจน)

## 5. การจัดการ Validation Error ด้วย `@RestControllerAdvice`

ทบทวนจาก Part 78: ใช้ `@RestControllerAdvice` จัดการ
`MethodArgumentNotValidException` ที่เกิดจาก validation ล้มเหลว

```java
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import java.util.HashMap;
import java.util.Map;

@RestControllerAdvice
public class ValidationExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, String>> handleValidationErrors(MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();

        ex.getBindingResult().getFieldErrors().forEach(error -> {
            errors.put(error.getField(), error.getDefaultMessage()); // ทบทวน Map จาก Part 24
        });

        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(errors);
    }
}
```

ตัวอย่างผลลัพธ์เมื่อส่ง request ที่ไม่ผ่าน validation:

```json
{
  "username": "ชื่อผู้ใช้ห้ามว่าง",
  "age": "อายุต้องมากกว่าหรือเท่ากับ 18 ปี"
}
```

**ประโยชน์**: Client (frontend/mobile app) ได้รับข้อความ error ที่**ชี้
เฉพาะ field ที่ผิด**พร้อมเหตุผลชัดเจน — สร้าง UX ที่ดีกว่าการได้แค่ `400 Bad
Request` เปล่า ๆ โดยไม่รู้ว่าอะไรผิด

## 6. Custom Validation Annotation

เมื่อ constraint มาตรฐาน (หัวข้อ 3) ไม่พอ สร้าง annotation เองได้ (ทบทวน
เต็มรูปแบบจาก Part 52):

```java
import jakarta.validation.Constraint;
import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;
import jakarta.validation.Payload;
import java.lang.annotation.*;

@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = ThaiPhoneNumberValidator.class) // ระบุ class ที่ทำ validation logic จริง
public @interface ThaiPhoneNumber {
    String message() default "รูปแบบเบอร์โทรศัพท์ไทยไม่ถูกต้อง";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

```java
public class ThaiPhoneNumberValidator implements ConstraintValidator<ThaiPhoneNumber, String> {
    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        if (value == null) return true; // ปล่อยให้ @NotNull จัดการเรื่อง null แยก (Single Responsibility)
        return value.matches("^0\\d{9}$"); // ทบทวน regex จาก Part 45
    }
}
```

```java
public record ContactRequest(
    @ThaiPhoneNumber
    String phoneNumber
) {}
```

## 7. Validation แบบ Group และ Nested Object

```java
import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;

public record OrderRequest(
    @NotBlank
    String customerName,

    @Valid // สำคัญมาก! บอกให้ validate object ที่ nested อยู่ภายในด้วย (ทบทวน nested config จาก Part 76)
    ShippingAddress shippingAddress
) {
    public record ShippingAddress(
        @NotBlank String street,
        @NotBlank String city,
        @jakarta.validation.constraints.Pattern(regexp = "\\d{5}") String zipCode
    ) {}
}
```

**ถ้าไม่ใส่ `@Valid` บน nested object** — Spring จะ validate แค่ field
ระดับบนสุด (`customerName`) เท่านั้น field ภายใน `ShippingAddress` จะไม่ถูก
validate เลย เป็นกับดักที่พบบ่อยมาก

## 8. Validation ที่ Service Layer

`@Valid` ที่ Controller (หัวข้อ 4) ป้องกันข้อมูลจาก HTTP request — แต่
**business logic ที่ซับซ้อนกว่า** (เช่น เช็คว่า username ซ้ำในฐานข้อมูล
หรือไม่) ควร validate ที่ **Service layer** (ทบทวนแนวคิด layer จาก Part 65,
73):

```java
import org.springframework.stereotype.Service;

@Service
public class UserService {
    private final UserRepository repository;

    public UserService(UserRepository repository) { this.repository = repository; }

    public void createUser(CreateUserRequest request) {
        // Business rule validation ที่ @Valid ทำไม่ได้ (ต้องเช็คกับฐานข้อมูล)
        if (repository.existsByUsername(request.username())) {
            throw new DuplicateUsernameException(request.username()); // ทบทวน custom exception จาก Part 21
        }
        // ... สร้าง user ...
    }
}

class DuplicateUsernameException extends RuntimeException {
    DuplicateUsernameException(String username) {
        super("ชื่อผู้ใช้ '" + username + "' มีอยู่แล้ว");
    }
}
```

**หลักการแบ่งชั้น validation**: **Controller layer** (Bean Validation)
ตรวจสอบ**รูปแบบข้อมูล** (format, ความครบถ้วน) — **Service layer** ตรวจสอบ
**business rule** ที่ต้องพึ่งพา state ของระบบ (เช่น ความซ้ำซ้อน,
availability)

## 9. Cross-field Validation

เมื่อต้อง validate**ความสัมพันธ์ระหว่างหลาย field** (เช่น password กับ
confirmPassword ต้องตรงกัน) ใช้ **class-level constraint**:

```java
import jakarta.validation.Constraint;
import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;
import java.lang.annotation.*;

@Target({ElementType.TYPE}) // ใช้กับ class ทั้งตัว ไม่ใช่ field เดียว
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = PasswordMatchValidator.class)
public @interface PasswordMatch {
    String message() default "รหัสผ่านและการยืนยันรหัสผ่านไม่ตรงกัน";
    Class<?>[] groups() default {};
    Class<? extends jakarta.validation.Payload>[] payload() default {};
}
```

```java
public class PasswordMatchValidator implements ConstraintValidator<PasswordMatch, RegisterRequest> {
    @Override
    public boolean isValid(RegisterRequest request, ConstraintValidatorContext context) {
        return request.password().equals(request.confirmPassword());
    }
}

@PasswordMatch // ใส่ที่ class level
public record RegisterRequest(String username, String password, String confirmPassword) {}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียน `record CreateProductRequest` พร้อม validation: name (ห้ามว่าง,
ยาว 3-50 ตัวอักษร), price (ต้องเป็นบวก), stock (ต้องไม่ติดลบ)

**เฉลย:**

```java
public record CreateProductRequest(
    @NotBlank @Size(min = 3, max = 50) String name,
    @Positive double price,
    @Min(0) int stock
) {}
```

**2)** เขียน `@RestControllerAdvice` handler ที่จัดการ custom exception
`DuplicateUsernameException` จากหัวข้อ 8 คืนค่า `409 Conflict` (ทบทวน status
code จาก Part 71)

**เฉลย:**

```java
@ExceptionHandler(DuplicateUsernameException.class)
public ResponseEntity<String> handleDuplicate(DuplicateUsernameException ex) {
    return ResponseEntity.status(HttpStatus.CONFLICT).body(ex.getMessage());
}
```

**3)** อธิบายว่าทำไมการเช็ค "username ซ้ำหรือไม่" ควรทำที่ Service layer
แทน Bean Validation ที่ Controller

**เฉลย**: Bean Validation (`@NotBlank`, `@Size` ฯลฯ) ทำงานแบบ **stateless**
— ตรวจสอบแค่**รูปแบบของข้อมูลที่ส่งมาเอง** โดยไม่ต้องรู้ context ภายนอกใด ๆ
(เร็ว ไม่ต้องพึ่งพา resource อื่น) แต่การเช็คว่า username ซ้ำหรือไม่**ต้อง
ค้นหาในฐานข้อมูล** (ทบทวน Repository จาก Part 79) ซึ่งเป็น operation ที่มี
cost สูงกว่าและต้องพึ่งพา state ปัจจุบันของระบบ (ข้อมูลใน database
เปลี่ยนแปลงได้ตลอดเวลา) — การผสม logic แบบนี้เข้ากับ Bean Validation
annotation จะทำให้ validator ต้อง depend on Repository (ขัดกับหลักการ
separation of concerns ที่ Bean Validation framework ออกแบบมา) และทดสอบ
ยากกว่ามาก (ทบทวนปัญหาการทดสอบ dependency จาก Part 59) — การแยกไปที่ Service
layer ทำให้แต่ละ layer มีความรับผิดชอบที่ชัดเจนตามหลัก SRP (Part 57)

### สรุปเนื้อหา Part 81

- Bean Validation (Jakarta Validation) ให้ constraint annotation มาตรฐาน
  (`@NotBlank`, `@Min`, `@Email` ฯลฯ)
- `@Valid` ใน Controller parameter ตรวจสอบข้อมูลก่อนเข้าสู่ business logic
- `@RestControllerAdvice` + `MethodArgumentNotValidException` จัดการ error
  message แบบละเอียดต่อ field
- สร้าง Custom Validation Annotation ได้เมื่อ constraint มาตรฐานไม่พอ
  (ทบทวนหลักการจาก Part 52)
- ต้องใส่ `@Valid` บน nested object เองเสมอ ไม่งั้นจะไม่ถูก validate
- แยก validation เป็น 2 ชั้น: format validation ที่ Controller (Bean
  Validation), business rule validation ที่ Service layer

**ต่อไป**: [Part 82 — Spring Security เบื้องต้น: Authentication, Authorization](./part-082-spring-security-basics.md)
