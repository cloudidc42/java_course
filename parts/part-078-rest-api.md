# Part 78: Building REST API ด้วย Spring Boot

> ขั้นตอนที่ 771-780 ของหลักสูตร | ระดับ: สูงมาก

## สารบัญ

1. REST API vs Server-side Rendering (ทบทวนความแตกต่างจาก Part 77)
2. `@RestController`: `@Controller` + `@ResponseBody`
3. การสร้าง CRUD REST API แบบเต็มรูปแบบ
4. `ResponseEntity`: ควบคุม Response อย่างละเอียด
5. DTO (Data Transfer Object) Pattern
6. Content Negotiation
7. `@RestControllerAdvice`: จัดการ Exception แบบรวมศูนย์
8. CORS (Cross-Origin Resource Sharing)
9. Testing REST API ด้วย `MockMvc` (ปูทางสู่ Part 84)
10. แบบฝึกหัดและสรุป

---

## 1. REST API vs Server-side Rendering (ทบทวนความแตกต่างจาก Part 77)

Part 77 สร้าง web application ที่ **server render HTML** ส่งกลับไปให้
browser แสดงผลตรง ๆ — **REST API** ต่างออกไป: server ส่งคืน**ข้อมูลดิบ**
(มักเป็น JSON — ทบทวนจาก Part 70) ให้ **client** (อาจเป็น JavaScript
frontend, mobile app, หรือระบบอื่น) ไปจัดการแสดงผลเอง

```
Server-side Rendering (Part 77):        REST API (Part นี้):
Browser <── HTML ──── Server              Frontend/App <── JSON ──── Server
(server ทำทุกอย่าง)                        (client จัดการ UI เอง, server แค่ให้ข้อมูล)
```

**สถาปัตยกรรมสมัยใหม่**นิยม REST API มากขึ้นเรื่อย ๆ เพราะแยก
**frontend/backend** ออกจากกันอย่างชัดเจน (ทบทวนหลักการ Separation of
Concerns จาก Part 57) — backend เดียวรองรับได้ทั้ง web app, mobile app,
และระบบอื่นที่มาเรียกใช้ API เดียวกัน

## 2. `@RestController`: `@Controller` + `@ResponseBody`

```java
import org.springframework.web.bind.annotation.*;

@RestController // เทียบเท่า @Controller + @ResponseBody บนทุกเมธอด (ทบทวน meta-annotation จาก Part 52, 75)
@RequestMapping("/api/products")
public class ProductRestController {

    @GetMapping
    public java.util.List<String> listProducts() {
        return java.util.List.of("Laptop", "Mouse", "Keyboard");
        // Spring แปลง List<String> เป็น JSON array อัตโนมัติผ่าน Jackson (ทบทวนจาก Part 70)
    }
}
```

เรียก `GET /api/products` ได้ผลลัพธ์:

```json
["Laptop", "Mouse", "Keyboard"]
```

**ไม่ต้อง**ใส่ `@ResponseBody` ทุกเมธอดแบบใน Part 77 หัวข้อ 5 — `@RestController`
ทำให้**ทุกเมธอด**ในคลาสนี้คืนค่าเป็น response body โดยอัตโนมัติเสมอ

## 3. การสร้าง CRUD REST API แบบเต็มรูปแบบ

ทบทวน CRUD จาก Part 65 และ HTTP Methods จาก Part 71 มาประยุกต์ใช้:

```java
import org.springframework.web.bind.annotation.*;
import java.util.*;
import java.util.concurrent.atomic.AtomicLong;

record Product(Long id, String name, double price) {}

@RestController
@RequestMapping("/api/products")
public class ProductCrudController {
    private final Map<Long, Product> products = new java.util.concurrent.ConcurrentHashMap<>();
    private final AtomicLong idGenerator = new AtomicLong(); // ทบทวน Atomic classes จาก Part 50

    @GetMapping
    public List<Product> getAll() {
        return new ArrayList<>(products.values());
    }

    @GetMapping("/{id}")
    public Product getById(@PathVariable Long id) {
        Product product = products.get(id);
        if (product == null) {
            throw new ProductNotFoundException(id); // ทบทวน custom exception จาก Part 21
        }
        return product;
    }

    @PostMapping
    public Product create(@RequestBody Product request) {
        long id = idGenerator.incrementAndGet();
        Product product = new Product(id, request.name(), request.price());
        products.put(id, product);
        return product;
    }

    @PutMapping("/{id}")
    public Product update(@PathVariable Long id, @RequestBody Product request) {
        if (!products.containsKey(id)) {
            throw new ProductNotFoundException(id);
        }
        Product updated = new Product(id, request.name(), request.price());
        products.put(id, updated);
        return updated;
    }

    @DeleteMapping("/{id}")
    public void delete(@PathVariable Long id) {
        products.remove(id);
    }
}

class ProductNotFoundException extends RuntimeException {
    public ProductNotFoundException(Long id) {
        super("ไม่พบสินค้า id=" + id);
    }
}
```

## 4. `ResponseEntity`: ควบคุม Response อย่างละเอียด

การ return ค่าตรง ๆ (แบบหัวข้อ 3) ได้ status code `200 OK` เสมอ — ถ้าต้องการ
**ควบคุม status code, headers อย่างละเอียด** (ทบทวนจาก Part 71) ใช้
`ResponseEntity`:

```java
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/orders")
public class OrderResponseEntityController {

    @PostMapping
    public ResponseEntity<Order> createOrder(@RequestBody OrderRequest request) {
        Order created = new Order(1L, request.item(), "PENDING");

        return ResponseEntity
                .status(HttpStatus.CREATED) // 201 Created (ทบทวนความหมายจาก Part 71)
                .header("Location", "/api/orders/" + created.id())
                .body(created);
    }

    @GetMapping("/{id}")
    public ResponseEntity<Order> getOrder(@PathVariable Long id) {
        Order order = findOrder(id); // สมมติว่าอาจไม่เจอ

        if (order == null) {
            return ResponseEntity.notFound().build(); // 404 (ทบทวนจาก Part 71) ไม่มี body
        }
        return ResponseEntity.ok(order); // 200 พร้อม body
    }

    Order findOrder(Long id) { return null; } // placeholder
    record Order(Long id, String item, String status) {}
    record OrderRequest(String item) {}
}
```

## 5. DTO (Data Transfer Object) Pattern

**DTO** คือ object ที่ใช้**ส่งข้อมูลระหว่าง layer หรือระหว่างระบบ**
(ทบทวนแนวคิด separation of concerns จาก Part 57) — **ไม่ควรส่ง Entity
(ทบทวนจาก Part 79) ตรงจาก database ออกไปเป็น API response โดยตรง**

```java
// Entity: มี field ทั้งหมดที่ database เก็บ (รวม field ที่ sensitive/internal)
class UserEntity {
    Long id;
    String username;
    String passwordHash; // ห้ามส่งออกไปทาง API เด็ดขาด!
    String internalNotes; // ข้อมูลภายในที่ไม่ควรเปิดเผย
}

// DTO: มีแค่ field ที่ API ควรเปิดเผยให้ client เห็น
record UserResponseDto(Long id, String username) {
    static UserResponseDto from(UserEntity entity) { // static factory method (ทบทวนจาก Part 12)
        return new UserResponseDto(entity.id, entity.username);
    }
}
```

```java
@RestController
@RequestMapping("/api/users")
public class UserDtoController {
    @GetMapping("/{id}")
    public UserResponseDto getUser(@PathVariable Long id) {
        UserEntity entity = findUserEntity(id); // ดึงจาก database (ทบทวน Part 79)
        return UserResponseDto.from(entity); // แปลงเป็น DTO ก่อนส่งออก - ไม่มีทางรั่ว passwordHash เลย
    }

    UserEntity findUserEntity(Long id) { return new UserEntity(); } // placeholder
}
```

**เหตุผลสำคัญที่ต้องใช้ DTO**: (1) **ความปลอดภัย** — ป้องกันข้อมูล sensitive
รั่วไหลออกไปโดยไม่ตั้งใจ (ทบทวนความเสี่ยงจาก Part 101), (2) **ความยืดหยุ่น**
— เปลี่ยนโครงสร้าง database ได้โดยไม่กระทบ API contract ที่ client ใช้อยู่
(decoupling — ทบทวนหลัก DIP จาก Part 57)

## 6. Content Negotiation

Spring MVC เลือก format ของ response (JSON, XML) ตาม **`Accept` header**
ของ request (ทบทวนจาก Part 71):

```java
@RestController
public class ContentNegotiationController {

    @GetMapping(value = "/api/data", produces = {"application/json", "application/xml"})
    public DataResponse getData() {
        return new DataResponse("Hello");
    }

    record DataResponse(String message) {}
}
```

```
Request: GET /api/data
Header: Accept: application/json  -> ได้ JSON: {"message":"Hello"}
Header: Accept: application/xml    -> ได้ XML: <DataResponse><message>Hello</message></DataResponse>
```

**ในทางปฏิบัติ**: JSON เป็น format ที่นิยมที่สุดเกือบทั้งหมดในปัจจุบัน
(ทบทวนเหตุผลจาก Part 38, 70) — XML พบใช้น้อยลงเรื่อย ๆ

## 7. `@RestControllerAdvice`: จัดการ Exception แบบรวมศูนย์

ทบทวนจาก Part 10, 21: การ catch exception ในทุกเมธอดซ้ำ ๆ ขัดกับหลัก DRY
(ทบทวนจาก Part 8) — **`@RestControllerAdvice`** จัดการ exception จาก**ทุก
Controller ในที่เดียว** (คล้ายแนวคิด Filter จาก Part 72 แต่ทำงานเฉพาะกับ
exception)

```java
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ProductNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ProductNotFoundException ex) {
        ErrorResponse error = new ErrorResponse(ex.getMessage(), 404);
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }

    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<ErrorResponse> handleBadRequest(IllegalArgumentException ex) {
        ErrorResponse error = new ErrorResponse(ex.getMessage(), 400);
        return ResponseEntity.badRequest().body(error);
    }

    @ExceptionHandler(Exception.class) // catch-all สำหรับ exception ที่ไม่คาดคิด (ทบทวนหลักการจาก Part 10)
    public ResponseEntity<ErrorResponse> handleGeneral(Exception ex) {
        ErrorResponse error = new ErrorResponse("เกิดข้อผิดพลาดภายในระบบ", 500);
        return ResponseEntity.internalServerError().body(error);
    }

    record ErrorResponse(String message, int status) {}
}
```

**ประโยชน์**: Controller (หัวข้อ 3) **ไม่ต้อง try-catch เองเลย** — แค่
`throw` exception ปกติ (ทบทวนจาก Part 10, 21) แล้ว `@RestControllerAdvice`
จะจัดการแปลงเป็น HTTP response ที่เหมาะสมให้เอง (ทบทวนหลัก Single
Responsibility จาก Part 57 — Controller โฟกัสที่ business flow, error
handling แยกไปอีก layer)

## 8. CORS (Cross-Origin Resource Sharing)

**CORS** เป็นกลไกความปลอดภัยของ browser ที่**บล็อก JavaScript จาก domain
หนึ่งเรียก API ของอีก domain**โดย default (ป้องกัน attack บางประเภท) —
ต้อง**อนุญาตอย่างชัดเจน**ถ้า frontend และ backend อยู่คนละ domain

```java
import org.springframework.web.bind.annotation.CrossOrigin;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/products")
@CrossOrigin(origins = "https://myfrontend.com") // อนุญาตเฉพาะ domain นี้เรียก API ได้
public class CorsController {
    @GetMapping
    public java.util.List<String> getProducts() {
        return java.util.List.of("Laptop", "Mouse");
    }
}
```

หรือตั้งค่าแบบ global สำหรับทุก controller:

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("https://myfrontend.com")
                .allowedMethods("GET", "POST", "PUT", "DELETE");
    }
}
```

## 9. Testing REST API ด้วย `MockMvc` (ปูทางสู่ Part 84)

ทบทวนจาก Part 58-59: ทดสอบ REST API โดยไม่ต้องรัน server จริง

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.test.web.servlet.MockMvc;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(ProductCrudController.class) // โหลดแค่ web layer ไม่ต้องมี database จริง
class ProductCrudControllerTest {

    @Autowired
    private MockMvc mockMvc; // จำลอง HTTP request โดยไม่ต้องมี network จริง

    @org.junit.jupiter.api.Test
    void shouldReturnEmptyListInitially() throws Exception {
        mockMvc.perform(get("/api/products"))
               .andExpect(status().isOk())      // ยืนยัน status code (ทบทวน assertion จาก Part 58)
               .andExpect(content().contentType("application/json"))
               .andExpect(jsonPath("$").isArray());
    }
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียน REST endpoint `PATCH /api/products/{id}/price` ที่รับ JSON
body `{"price": 199.99}` แล้วแก้ไขราคาสินค้าเฉพาะ field นั้น

**เฉลย:**

```java
@PatchMapping("/{id}/price")
public ResponseEntity<Product> updatePrice(@PathVariable Long id, @RequestBody PriceUpdateRequest req) {
    Product existing = products.get(id);
    if (existing == null) return ResponseEntity.notFound().build();

    Product updated = new Product(id, existing.name(), req.price());
    products.put(id, updated);
    return ResponseEntity.ok(updated);
}
record PriceUpdateRequest(double price) {}
```

**2)** เขียน `@RestControllerAdvice` ที่จัดการ `NumberFormatException`
(ทบทวนจาก Part 10) คืนค่า `400 Bad Request`

**เฉลย:**

```java
@ExceptionHandler(NumberFormatException.class)
public ResponseEntity<ErrorResponse> handleNumberFormat(NumberFormatException ex) {
    return ResponseEntity.badRequest().body(new ErrorResponse("รูปแบบตัวเลขไม่ถูกต้อง: " + ex.getMessage(), 400));
}
```

**3)** อธิบายว่าทำไมการส่ง Entity ตรงจาก database ออกไปเป็น API response
เสี่ยงต่อความปลอดภัย

**เฉลย**: Entity มักมี field ที่**ไม่ควรเปิดเผยให้ client เห็น** เช่น
`passwordHash`, ข้อมูลภายในที่ใช้เฉพาะใน business logic, หรือ relationship
ไปยัง entity อื่นที่อาจทำให้เกิดข้อมูลรั่วไหลเป็นทอด ๆ (เช่น
`User.orders.customer.creditCard` — ทบทวนแนวคิด relationship ที่จะเรียนใน
Part 80) — Jackson (ทบทวนจาก Part 70) จะ serialize**ทุก field ที่เข้าถึง
ได้**เป็น JSON โดยอัตโนมัติถ้าไม่ระบุ `@JsonIgnore` อย่างระมัดระวังทุกจุด
ซึ่งเสี่ยงพลาดได้ง่ายมาก — การใช้ DTO ที่นิยาม field ที่**ต้องการเปิดเผย
เท่านั้น**อย่างชัดเจน (whitelist approach) ปลอดภัยกว่าการพยายามซ่อน field
ที่ไม่ต้องการทีละตัวบน Entity (blacklist approach) มาก

### สรุปเนื้อหา Part 78

- REST API ส่ง JSON ให้ client จัดการ UI เอง ต่างจาก server-side rendering
  (Part 77)
- `@RestController` = `@Controller` + `@ResponseBody` บนทุกเมธอด
- `ResponseEntity` ควบคุม status code และ headers ได้อย่างละเอียด
- DTO แยกข้อมูลที่เปิดเผยผ่าน API ออกจาก Entity ภายใน — ป้องกันข้อมูล
  sensitive รั่วไหลและลด coupling
- `@RestControllerAdvice` จัดการ exception แบบรวมศูนย์ ไม่ต้อง try-catch
  ในทุก Controller
- CORS ต้องอนุญาตอย่างชัดเจนเมื่อ frontend/backend อยู่คนละ domain
- `MockMvc` ทดสอบ REST API โดยไม่ต้องรัน server จริง

**ต่อไป**: [Part 79 — Spring Data JPA เบื้องต้น](./part-079-spring-data-jpa.md)
