# Part 84: Spring Boot Testing: MockMvc, @SpringBootTest, Testcontainers

> ขั้นตอนที่ 831-840 ของหลักสูตร | ระดับ: สูงมาก

## สารบัญ

1. ทำไม Spring Boot Application ต้องมี Testing Strategy หลายระดับ
2. Unit Test สำหรับ Service Layer (ทบทวน Mockito)
3. `@WebMvcTest` และ `MockMvc`: ทดสอบ Controller Layer แบบ Isolate
4. `@DataJpaTest`: ทดสอบ Repository Layer
5. `@SpringBootTest`: ทดสอบ Integration แบบเต็มระบบ
6. ปัญหาของ H2 In-Memory Database และเหตุผลที่ต้องมี Testcontainers
7. Testcontainers: รัน PostgreSQL จริงใน Docker สำหรับ Test
8. ทดสอบ REST API ด้วย `TestRestTemplate`
9. Test Slices สรุปเปรียบเทียบ
10. แบบฝึกหัดและสรุป

---

## 1. ทำไม Spring Boot Application ต้องมี Testing Strategy หลายระดับ

ทบทวน **Testing Pyramid** จาก Part 58: Unit Test มาก, Integration Test
ปานกลาง, End-to-End Test น้อย — Spring Boot application มี**หลายชั้น**
(Controller → Service → Repository → Database) แต่ละชั้นทดสอบต่างวิธีกัน:

```
┌─────────────────────────────────────┐
│  E2E / TestRestTemplate (น้อยที่สุด)   │  ← ทดสอบทั้งระบบผ่าน HTTP จริง
├─────────────────────────────────────┤
│  @SpringBootTest (Integration)        │  ← โหลด ApplicationContext เต็ม
├─────────────────────────────────────┤
│  @WebMvcTest / @DataJpaTest (Slice)   │  ← โหลดแค่ชั้นที่เกี่ยวข้อง
├─────────────────────────────────────┤
│  Unit Test + Mockito (มากที่สุด)       │  ← ทดสอบ class เดียวแบบ isolate
└─────────────────────────────────────┘
```

## 2. Unit Test สำหรับ Service Layer (ทบทวน Mockito)

ทบทวนจาก Part 59: mock dependencies ทั้งหมดของ Service เพื่อทดสอบ
**business logic ล้วน ๆ** โดยไม่แตะ database หรือ network จริง

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    @Mock
    private OrderRepository orderRepository; // ทบทวน @Mock จาก Part 59

    @InjectMocks
    private OrderService orderService;

    @Test
    void shouldThrowWhenOrderNotFound() {
        when(orderRepository.findById(1L)).thenReturn(Optional.empty());

        // ทบทวน assertThatThrownBy จาก Part 58 (AssertJ)
        org.assertj.core.api.Assertions.assertThatThrownBy(() -> orderService.getOrderById(1L))
                .isInstanceOf(OrderNotFoundException.class)
                .hasMessageContaining("1");

        verify(orderRepository, times(1)).findById(1L); // ทบทวน verify จาก Part 59
    }
}
```

## 3. `@WebMvcTest` และ `MockMvc`: ทดสอบ Controller Layer แบบ Isolate

`@WebMvcTest` โหลด**เฉพาะ Spring MVC layer** (Controller, filter,
`@ControllerAdvice`) โดย**ไม่โหลด Service/Repository จริง** — เร็วกว่า
`@SpringBootTest` มาก (ทบทวนแนวคิด Spring Context จาก Part 74)

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;

import static org.mockito.Mockito.when;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(OrderController.class) // โหลดแค่ controller นี้เท่านั้น
class OrderControllerTest {
    @Autowired
    private MockMvc mockMvc; // จำลอง HTTP request โดยไม่ต้องรัน server จริง (ทบทวน Servlet จาก Part 72)

    @MockBean // แทนที่ OrderService bean จริงด้วย mock ใน Spring Context
    private OrderService orderService;

    @Test
    void shouldReturnOrderById() throws Exception {
        var order = new OrderDto(1L, "PENDING", 199.99);
        when(orderService.getOrderById(1L)).thenReturn(order);

        mockMvc.perform(get("/api/orders/1")
                .contentType(MediaType.APPLICATION_JSON))
                .andExpect(status().isOk()) // ทบทวน HTTP status code จาก Part 71
                .andExpect(jsonPath("$.id").value(1))
                .andExpect(jsonPath("$.status").value("PENDING"));
    }

    @Test
    void shouldReturn404WhenOrderNotFound() throws Exception {
        when(orderService.getOrderById(999L))
                .thenThrow(new OrderNotFoundException(999L));

        mockMvc.perform(get("/api/orders/999"))
                .andExpect(status().isNotFound()); // ทดสอบ @RestControllerAdvice จาก Part 78 ทำงานถูกต้อง
    }

    @Test
    void shouldValidateRequestBody() throws Exception {
        String invalidJson = "{\"customerName\": \"\", \"amount\": -10}";

        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content(invalidJson))
                .andExpect(status().isBadRequest()) // ทดสอบ @Valid + Bean Validation จาก Part 81
                .andExpect(jsonPath("$.errors").isArray());
    }
}
```

## 4. `@DataJpaTest`: ทดสอบ Repository Layer

`@DataJpaTest` โหลด**เฉพาะ JPA components** (Repository, EntityManager)
พร้อม **rollback transaction อัตโนมัติหลังทุก test** (ทบทวน `@Transactional`
จาก Part 65)

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

@DataJpaTest // ค่า default ใช้ H2 in-memory database แทน database จริง
class OrderRepositoryTest {
    @Autowired
    private OrderRepository orderRepository; // ทบทวน Spring Data JPA จาก Part 79

    @Test
    void shouldFindOrdersByStatus() {
        orderRepository.save(new Order("Alice", 100.0, "PENDING"));
        orderRepository.save(new Order("Bob", 200.0, "COMPLETED"));
        orderRepository.save(new Order("Carol", 150.0, "PENDING"));

        List<Order> pendingOrders = orderRepository.findByStatus("PENDING"); // query method ทบทวน Part 79

        assertThat(pendingOrders).hasSize(2)
                .extracting(Order::getCustomerName)
                .containsExactlyInAnyOrder("Alice", "Carol");
    }
    // แต่ละ test method rollback อัตโนมัติ - test อื่นไม่เห็นข้อมูลที่ insert ไว้ (isolation ทบทวน Part 65)
}
```

## 5. `@SpringBootTest`: ทดสอบ Integration แบบเต็มระบบ

`@SpringBootTest` โหลด **ApplicationContext ทั้งหมด** (ทุก Bean) เหมาะกับ
การทดสอบ**การทำงานร่วมกันของหลายชั้น** แต่**ช้ากว่า** test slice มาก

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.transaction.annotation.Transactional;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest // สตาร์ท ApplicationContext ทั้งระบบ เหมือนรันแอปจริง
@ActiveProfiles("test") // ใช้ application-test.properties (ทบทวน Profiles จาก Part 76)
@Transactional // rollback หลังทุก test เพื่อไม่ทำให้ test อื่นเพี้ยน
class OrderServiceIntegrationTest {
    @Autowired
    private OrderService orderService; // Service จริง ต่อกับ Repository จริง ต่อกับ database จริง

    @Autowired
    private OrderRepository orderRepository;

    @Test
    void shouldCreateOrderAndPersistToDatabase() {
        var request = new CreateOrderRequest("Dave", 299.99);

        OrderDto created = orderService.createOrder(request);

        // ตรวจสอบว่า business logic (Service) + persistence (Repository) ทำงานร่วมกันถูกต้อง
        assertThat(orderRepository.findById(created.id())).isPresent();
        assertThat(orderRepository.findById(created.id()).get().getStatus()).isEqualTo("PENDING");
    }
}
```

## 6. ปัญหาของ H2 In-Memory Database และเหตุผลที่ต้องมี Testcontainers

`@DataJpaTest` ค่า default ใช้ **H2** (in-memory database) เพราะเร็วและไม่
ต้องติดตั้งอะไร แต่มีปัญหาสำคัญ: **H2 ไม่ใช่ PostgreSQL/MySQL จริง** —
syntax SQL บางอย่าง, JSON column type, หรือ database-specific function
อาจทำงานต่างกัน ทำให้ **test ผ่านบน H2 แต่ fail บน production database
จริง** (ทบทวนแนวคิด "test environment ต้องเหมือน production" จาก Part 66)

**Testcontainers** แก้ปัญหานี้: รัน **database จริง (เช่น PostgreSQL) ใน
Docker container ชั่วคราว** เฉพาะช่วง test แล้วปิดทิ้งอัตโนมัติ

## 7. Testcontainers: รัน PostgreSQL จริงใน Docker สำหรับ Test

```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>1.19.3</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <version>1.19.3</version>
    <scope>test</scope>
</dependency>
```

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@Testcontainers // จัดการ lifecycle ของ container อัตโนมัติ (start ก่อน test, stop หลัง test)
class OrderRepositoryTestcontainersTest {

    @Container // สร้าง PostgreSQL container จริงจาก Docker image
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test");

    @DynamicPropertySource // ตั้งค่า datasource ของ Spring ให้ชี้ไปที่ container ที่สร้างขึ้นแบบ dynamic
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private OrderRepository orderRepository;

    @Test
    void shouldWorkAgainstRealPostgresDatabase() {
        orderRepository.save(new Order("Eve", 500.0, "PENDING"));

        // ทดสอบกับ PostgreSQL จริง - มั่นใจได้ว่า SQL/JSON/index behavior ตรงกับ production 100%
        assertThat(orderRepository.findByStatus("PENDING")).isNotEmpty();
    }
}
```

**ข้อดีสำคัญ**: Testcontainers ทำให้ CI/CD pipeline (ปูทางสู่ Part 97) รัน
test ด้วย database ตัวเดียวกับ production ได้ **โดยไม่ต้องติดตั้ง
PostgreSQL ลงในเครื่อง CI จริง** — แค่มี Docker ก็เพียงพอ

## 8. ทดสอบ REST API ด้วย `TestRestTemplate`

```java
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.SpringBootTest.WebEnvironment;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT) // รัน embedded server จริงที่ port สุ่ม
class OrderApiEndToEndTest {
    @Autowired
    private TestRestTemplate restTemplate; // ยิง HTTP request จริงไปที่ embedded server

    @Test
    void shouldCreateOrderViaRealHttpCall() {
        var request = new CreateOrderRequest("Frank", 89.99);

        ResponseEntity<OrderDto> response = restTemplate.postForEntity(
                "/api/orders", request, OrderDto.class
        ); // นี่คือ HTTP request จริง ผ่าน network stack จริง (ทบทวน Part 71)

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody().customerName()).isEqualTo("Frank");
    }
}
```

## 9. Test Slices สรุปเปรียบเทียบ

| Annotation | โหลดอะไรบ้าง | ความเร็ว | ใช้เมื่อไหร่ |
|---|---|---|---|
| Unit Test + Mockito | ไม่โหลด Spring Context เลย | เร็วที่สุด | ทดสอบ business logic ล้วน ๆ |
| `@WebMvcTest` | Controller, Filter, `@ControllerAdvice` | เร็ว | ทดสอบ HTTP layer, validation, JSON mapping |
| `@DataJpaTest` | Repository, EntityManager, H2 | เร็ว-ปานกลาง | ทดสอบ query method, mapping |
| `@SpringBootTest` | ทุก Bean ทั้งระบบ | ช้า | Integration test ข้ามหลายชั้น |
| `@SpringBootTest` + Testcontainers | ทุก Bean + database จริง | ช้าที่สุด | Integration test ที่ต้องมั่นใจ 100% กับ production behavior |

**หลักการ**: ใช้ Unit Test **ให้มากที่สุด** (เร็ว, isolate ง่าย) และใช้
`@SpringBootTest`/Testcontainers **ให้น้อยที่สุดเท่าที่จำเป็น** (ทบทวน
Testing Pyramid จาก Part 58)

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียน `@WebMvcTest` ทดสอบว่า endpoint `DELETE /api/orders/{id}`
คืนค่า HTTP 204 No Content เมื่อลบสำเร็จ

**เฉลย:**
```java
@Test
void shouldReturn204WhenDeleteSucceeds() throws Exception {
    doNothing().when(orderService).deleteOrder(1L);

    mockMvc.perform(delete("/api/orders/1"))
            .andExpect(status().isNoContent());

    verify(orderService, times(1)).deleteOrder(1L);
}
```

**2)** อธิบายว่าทำไม `@DataJpaTest` ที่ใช้ H2 อาจไม่พบ bug ที่เกิดขึ้นจริง
บน production ที่ใช้ PostgreSQL

**เฉลย**: H2 ไม่ใช่ PostgreSQL — มีความแตกต่างในหลายจุด เช่น การจัดการ
`JSON`/`JSONB` column type, การ enforce constraint บางแบบ, syntax เฉพาะของ
`PostgreSQL` function (เช่น `ILIKE`, window function บางตัว), และพฤติกรรม
ของ locking/isolation level ที่อาจต่างกัน (ทบทวน Transaction Isolation
จาก Part 65) — query ที่ทำงานถูกต้องบน H2 อาจ throw exception หรือให้ผล
ต่างกันบน PostgreSQL จริง การใช้ Testcontainers รัน PostgreSQL จริงจึง
ช่วยจับ bug เหล่านี้ได้ตั้งแต่ตอน test แทนที่จะไปพบตอน production

**3)** ทำไมควรใช้ `@Transactional` คู่กับ `@SpringBootTest` ในสถานการณ์ทดสอบ
ทั่วไป

**เฉลย**: `@Transactional` บน test class ทำให้ Spring **เปิด transaction
ก่อนแต่ละ test method และ rollback อัตโนมัติหลังจบ** (ทบทวน `@Transactional`
จาก Part 65) — ทำให้**ข้อมูลที่ insert/update ระหว่าง test ไม่ถูกบันทึกจริง
ลง database** และ**ไม่กระทบ test อื่นที่รันต่อจากกัน** (test isolation) ถ้า
ไม่มี `@Transactional` ข้อมูลจาก test หนึ่งจะค้างอยู่ในฐานข้อมูลและอาจทำให้
test ถัดไปที่นับจำนวน record ได้ผลลัพธ์ผิดเพี้ยน (flaky test)

### สรุปเนื้อหา Part 84

- Spring Boot ต้องการ testing strategy หลายระดับ: Unit → Slice → Integration
- Unit Test + Mockito: เร็วที่สุด, ทดสอบ business logic แบบ isolate
- `@WebMvcTest`+`MockMvc`: ทดสอบ Controller/HTTP layer โดยไม่โหลด Service
  จริง
- `@DataJpaTest`: ทดสอบ Repository ด้วย H2 พร้อม rollback อัตโนมัติ
- `@SpringBootTest`: โหลด ApplicationContext เต็มระบบ สำหรับ integration
  test
- Testcontainers: รัน database จริง (PostgreSQL) ใน Docker เพื่อความมั่นใจ
  100% ว่า test สอดคล้องกับ production
- `TestRestTemplate`: ทดสอบ REST API ผ่าน HTTP จริงแบบ end-to-end
- เลือกระดับ test ให้เหมาะสม: ใช้ Unit Test มากที่สุด, Integration Test
  น้อยที่สุดเท่าที่จำเป็น

**ต่อไป**: [Part 85 — RESTful API Design Best Practices, Versioning, Pagination](./part-085-rest-api-best-practices.md)
