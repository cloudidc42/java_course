# Part 86: API Documentation with Swagger/OpenAPI

> ขั้นตอนที่ 851-860 ของหลักสูตร | ระดับ: สูงมาก

## สารบัญ

1. ทำไม API ต้องมี Documentation ที่เป็นมาตรฐาน
2. OpenAPI Specification คืออะไร
3. ติดตั้ง springdoc-openapi ใน Spring Boot
4. Swagger UI: ทดสอบ API ผ่านหน้าเว็บ
5. `@Operation`, `@Parameter`, `@ApiResponse`: อธิบาย endpoint แบบละเอียด
6. เอกสารสำหรับ Request/Response DTO ด้วย `@Schema`
7. เอกสาร Authentication (JWT Bearer) ใน Swagger
8. Grouping API ด้วย Tags
9. Generate Client Code จาก OpenAPI Spec
10. แบบฝึกหัดและสรุป

---

## 1. ทำไม API ต้องมี Documentation ที่เป็นมาตรฐาน

ทบทวนปัญหาจาก Part 78, 85: API ที่ดีต้องมี **contract ที่ชัดเจน** ระหว่าง
**ทีม backend** (ที่สร้าง API) และ**ทีม frontend/mobile/ทีมอื่น** (ที่เรียก
ใช้ API) — ถ้าไม่มี documentation ที่เป็นมาตรฐาน ทีมอื่นต้อง**เดา** จาก
source code หรือถามทีม backend ตลอดเวลา (ทบทวนแนวคิด "interface เป็น
contract" จาก Part 16)

**OpenAPI Specification** (เดิมชื่อ Swagger) เป็นมาตรฐานอุตสาหกรรมสำหรับ
อธิบาย REST API ในรูปแบบที่ **เครื่องอ่านได้** (machine-readable) — ทำให้
สร้าง documentation, ทดสอบ API, และ generate client code ได้อัตโนมัติ

## 2. OpenAPI Specification คืออะไร

OpenAPI spec เป็นไฟล์ JSON/YAML ที่อธิบาย API ทั้งหมด:

```yaml
openapi: 3.0.1
info:
  title: Order Management API
  version: "1.0"
paths:
  /api/v1/orders/{id}:
    get:
      summary: ดึงข้อมูล order ตาม id
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
      responses:
        '200':
          description: สำเร็จ
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/OrderDto'
        '404':
          description: ไม่พบ order
```

**springdoc-openapi** library จะ**สร้างไฟล์นี้อัตโนมัติ**จาก Controller
ของเราโดยไม่ต้องเขียน YAML เองเลย (ทบทวนแนวคิด annotation-driven จาก
Part 52)

## 3. ติดตั้ง springdoc-openapi ใน Spring Boot

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.3.0</version>
</dependency>
```

เท่านี้ก็เข้าถึง documentation ได้แล้วที่:
- `http://localhost:8080/v3/api-docs` — OpenAPI spec แบบ JSON (ทบทวน JSON
  จาก Part 70)
- `http://localhost:8080/swagger-ui.html` — Swagger UI แบบหน้าเว็บ (หัวข้อ
  4)

## 4. Swagger UI: ทดสอบ API ผ่านหน้าเว็บ

```java
import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Info;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OpenApiConfig {
    @Bean
    public OpenAPI apiInfo() {
        return new OpenAPI()
                .info(new Info()
                        .title("Order Management API")
                        .version("1.0")
                        .description("REST API สำหรับจัดการคำสั่งซื้อในระบบ e-commerce"));
        // ตั้งค่านี้ครั้งเดียว - Swagger UI จะแสดงข้อมูลนี้ที่ด้านบนของหน้าเว็บ
    }
}
```

Swagger UI ช่วยให้**ผู้ใช้ทดสอบ API ได้จริงจากหน้าเว็บ** โดยไม่ต้องเขียน
code หรือใช้ Postman — กรอกพารามิเตอร์แล้วกด "Try it out" ก็ยิง request
จริงได้ทันที (มีประโยชน์มากสำหรับทีม QA และทีม frontend ที่ไม่ใช่นักพัฒนา
backend)

## 5. `@Operation`, `@Parameter`, `@ApiResponse`: อธิบาย endpoint แบบละเอียด

springdoc สร้าง documentation พื้นฐานให้อัตโนมัติจาก Controller แต่การ
เพิ่ม annotation ทำให้เอกสารสมบูรณ์และเข้าใจง่ายขึ้นมาก:

```java
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.Parameter;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import io.swagger.v3.oas.annotations.responses.ApiResponses;
import io.swagger.v3.oas.annotations.media.Content;
import io.swagger.v3.oas.annotations.media.Schema;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {
    private final OrderService orderService;

    public OrderController(OrderService orderService) { this.orderService = orderService; }

    @Operation(
        summary = "ดึงข้อมูล order ตาม id",
        description = "คืนค่ารายละเอียดของ order รวมถึงสถานะปัจจุบันและรายการสินค้า"
    ) // ทบทวนแนวคิด self-documenting code จาก Part 57 - annotation นี้คือ "documentation as code"
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "พบ order",
            content = @Content(schema = @Schema(implementation = OrderDto.class))),
        @ApiResponse(responseCode = "404", description = "ไม่พบ order ที่ระบุ")
    })
    @GetMapping("/{id}")
    public ResponseEntity<OrderDto> getOrder(
            @Parameter(description = "รหัส order ที่ต้องการค้นหา", example = "42")
            @PathVariable Long id) {
        return ResponseEntity.ok(orderService.getOrderById(id));
    }
}
```

Swagger UI จะแสดง summary, description, ตัวอย่าง parameter, และรายการ
response code ที่เป็นไปได้ทั้งหมด — ทำให้ผู้ใช้ API เข้าใจโดยไม่ต้องอ่าน
source code เลย

## 6. เอกสารสำหรับ Request/Response DTO ด้วย `@Schema`

```java
import io.swagger.v3.oas.annotations.media.Schema;

@Schema(description = "ข้อมูลคำขอสร้าง order ใหม่")
public record CreateOrderRequest(
        @Schema(description = "ชื่อลูกค้า", example = "สมชาย ใจดี", requiredMode = Schema.RequiredMode.REQUIRED)
        String customerName,

        @Schema(description = "จำนวนเงินรวม (บาท)", example = "1500.50", minimum = "0")
        double amount
) {}

@Schema(description = "ข้อมูล order ที่ตอบกลับ")
public record OrderDto(
        @Schema(description = "รหัส order", example = "42")
        Long id,

        @Schema(description = "สถานะปัจจุบันของ order",
                allowableValues = {"PENDING", "PAID", "SHIPPED", "COMPLETED", "CANCELLED"})
        String status,

        @Schema(description = "จำนวนเงินรวม (บาท)", example = "1500.50")
        double amount
) {}
```

**ข้อสังเกต**: การใช้ **record** (ทบทวน Part 51) ทำให้ DTO กระชับมาก และ
`@Schema` เพิ่มข้อมูลอธิบายที่ **compiler ไม่สามารถบอกได้เอง** เช่น
ค่าที่เป็นไปได้ของ `status` หรือความหมายทางธุรกิจของแต่ละ field

## 7. เอกสาร Authentication (JWT Bearer) ใน Swagger

ทบทวนจาก Part 83: API ที่ใช้ JWT ต้องบอก Swagger ว่าต้องแนบ
`Authorization: Bearer <token>` — เพื่อให้ผู้ใช้ทดสอบผ่าน Swagger UI ได้
โดยตรง

```java
import io.swagger.v3.oas.annotations.security.SecurityRequirement;
import io.swagger.v3.oas.annotations.security.SecurityScheme;
import io.swagger.v3.oas.annotations.enums.SecuritySchemeType;
import org.springframework.context.annotation.Configuration;

@Configuration
@SecurityScheme(
    name = "bearerAuth",
    type = SecuritySchemeType.HTTP,
    scheme = "bearer",
    bearerFormat = "JWT"
) // ประกาศ security scheme ครั้งเดียวสำหรับทั้งแอป
public class SwaggerSecurityConfig {}
```

```java
@RestController
@RequestMapping("/api/v1/orders")
@SecurityRequirement(name = "bearerAuth") // บอกว่า endpoint ทั้งหมดในนี้ต้องใช้ JWT
public class SecuredOrderController {
    @GetMapping("/{id}")
    public OrderDto getOrder(@PathVariable Long id) {
        return new OrderDto(id, "PENDING", 100.0);
    }
}
```

หลังตั้งค่านี้ Swagger UI จะมีปุ่ม **"Authorize"** ให้ผู้ใช้กรอก JWT token
ครั้งเดียว แล้ว Swagger จะแนบ header `Authorization: Bearer <token>` ให้
ทุก request ที่ทดสอบผ่านหน้าเว็บโดยอัตโนมัติ

## 8. Grouping API ด้วย Tags

เมื่อ API มีหลาย Controller (Order, Customer, Payment, ...) การจัดกลุ่ม
ด้วย `@Tag` ช่วยให้ Swagger UI **อ่านง่ายขึ้นมาก**:

```java
import io.swagger.v3.oas.annotations.tags.Tag;

@RestController
@RequestMapping("/api/v1/orders")
@Tag(name = "Orders", description = "จัดการคำสั่งซื้อ - สร้าง, ค้นหา, ยกเลิก")
public class TaggedOrderController { /* ... */ }

@RestController
@RequestMapping("/api/v1/customers")
@Tag(name = "Customers", description = "จัดการข้อมูลลูกค้า")
public class TaggedCustomerController { /* ... */ }
```

Swagger UI จะแสดง endpoint เป็นกลุ่ม "Orders" และ "Customers" แยกกัน
คล้ายกับการจัดโฟลเดอร์ ทำให้ค้นหา endpoint ที่ต้องการได้เร็วขึ้นเมื่อ API
มีขนาดใหญ่ (หลายสิบ-หลายร้อย endpoint)

## 9. Generate Client Code จาก OpenAPI Spec

เพราะ OpenAPI spec เป็น **machine-readable** เครื่องมืออย่าง
`openapi-generator` สามารถ**สร้าง client code อัตโนมัติ**ในภาษาต่าง ๆ
(TypeScript, Java, Python) จากไฟล์ spec เดียวกัน — ทีม frontend ไม่ต้อง
เขียน HTTP client เองเลย

```java
public class OpenApiGeneratorConcept {
    /*
     * ขั้นตอน:
     * 1. รัน Spring Boot app -> ดึง spec จาก /v3/api-docs (JSON)
     * 2. รัน: openapi-generator-cli generate -i api-docs.json -g typescript-axios -o ./client
     * 3. ได้ TypeScript client ที่มี type-safe function เรียก API ทุกตัวอัตโนมัติ
     *    เช่น: ordersApi.getOrder(42) -> คืน Promise<OrderDto> ที่มี type ตรงกับ backend
     *
     * ประโยชน์: ถ้า backend เปลี่ยน DTO (เพิ่ม field, เปลี่ยน type) generate ใหม่ครั้งเดียว
     * frontend จะเห็น compile error ทันทีถ้าใช้ field ที่ไม่มีแล้ว - ป้องกัน bug จาก API mismatch
     */
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เพิ่ม `@Operation` และ `@ApiResponses` ให้กับ endpoint
`POST /api/v1/orders` ที่คืนค่า 201 เมื่อสำเร็จ และ 400 เมื่อ validation
ล้มเหลว

**เฉลย:**
```java
@Operation(summary = "สร้าง order ใหม่", description = "สร้างคำสั่งซื้อใหม่ในระบบพร้อมตรวจสอบข้อมูลนำเข้า")
@ApiResponses({
    @ApiResponse(responseCode = "201", description = "สร้างสำเร็จ",
        content = @Content(schema = @Schema(implementation = OrderDto.class))),
    @ApiResponse(responseCode = "400", description = "ข้อมูลนำเข้าไม่ถูกต้อง (validation failed)")
})
@PostMapping
public ResponseEntity<OrderDto> create(@RequestBody @Valid CreateOrderRequest req) {
    OrderDto created = orderService.createOrder(req);
    return ResponseEntity.created(URI.create("/api/v1/orders/" + created.id())).body(created);
}
```

**2)** ทำไมการเขียน OpenAPI spec เป็น YAML ด้วยมือ (แทนการใช้
springdoc-openapi generate อัตโนมัติ) มีความเสี่ยงที่จะทำให้ documentation
"หลุด sync" กับ code จริง

**เฉลย**: ถ้าเขียน spec แยกจาก code ด้วยมือ **ทุกครั้งที่ developer แก้ไข
Controller** (เพิ่ม field, เปลี่ยน path, เปลี่ยน status code) **ต้องจำ
ไปแก้ไฟล์ YAML ด้วยตัวเองเสมอ** — ในทางปฏิบัติมักถูกลืมหรือทำไม่ทัน ทำให้
documentation แสดงข้อมูลที่**ไม่ตรงกับ behavior จริงของ API** (เช่น บอกว่า
field เป็น optional แต่จริง ๆ required แล้ว) ผู้ใช้ API ที่เชื่อ
documentation จะเขียนโค้ดผิดพลาด ในทางกลับกัน springdoc-openapi
**generate spec จาก annotation ใน source code โดยตรง** ทำให้
documentation **sync กับ code เสมอโดยอัตโนมัติ** (ทบทวนหลักการ "single
source of truth" ที่เกี่ยวข้องกับ DRY จาก Part 57)

**3)** อธิบายประโยชน์ของการใช้ `@SecurityScheme` ร่วมกับ Swagger UI
สำหรับ API ที่ใช้ JWT

**เฉลย**: การประกาศ `@SecurityScheme` ทำให้ Swagger UI**แสดงปุ่ม
"Authorize"** ที่ผู้ใช้กรอก JWT token ได้**ครั้งเดียว** จากนั้น Swagger จะ
**แนบ header `Authorization: Bearer <token>` ให้ทุก request ที่ทดสอบผ่าน
หน้าเว็บโดยอัตโนมัติ** โดยไม่ต้องพิมพ์ header ซ้ำทุกครั้งที่ทดสอบ endpoint
ใหม่ ทำให้ผู้ใช้ (ทั้งนักพัฒนาและ QA) **ทดสอบ endpoint ที่ต้อง
authentication ได้สะดวกมาก**โดยไม่ต้องใช้เครื่องมืออื่นอย่าง Postman เพื่อ
จัดการ token เอง — เพิ่มความเร็วในการทดสอบและลด friction ในการทำงานร่วมกัน
ระหว่างทีม

### สรุปเนื้อหา Part 86

- OpenAPI Specification เป็นมาตรฐานอุตสาหกรรมสำหรับอธิบาย REST API แบบ
  machine-readable
- springdoc-openapi generate documentation อัตโนมัติจาก Controller ไม่
  ต้องเขียน YAML เอง
- Swagger UI ให้ผู้ใช้ทดสอบ API ผ่านหน้าเว็บได้จริงโดยไม่ต้องเขียนโค้ด
- `@Operation`/`@ApiResponse`/`@Parameter`/`@Schema` เพิ่มรายละเอียดให้
  documentation สมบูรณ์
- `@SecurityScheme` + `@SecurityRequirement` ทำให้ Swagger UI รองรับการ
  ทดสอบ endpoint ที่ต้อง JWT authentication
- `@Tag` จัดกลุ่ม endpoint ให้อ่านง่ายเมื่อ API มีขนาดใหญ่
- OpenAPI spec ใช้ generate client code อัตโนมัติในหลายภาษาได้

**ต่อไป**: [Part 87 — Caching with Redis, Spring Cache](./part-087-caching-redis.md)
