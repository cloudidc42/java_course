# Part 59: Mocking ด้วย Mockito

> ขั้นตอนที่ 581-590 ของหลักสูตร | ระดับ: สูง

## สารบัญ

1. ทำไมต้อง Mock: ปัญหาของการทดสอบ Dependency จริง
2. การตั้งค่า Mockito
3. การสร้าง Mock ด้วย `@Mock` และ `mock()`
4. `when().thenReturn()`: กำหนดพฤติกรรมของ Mock
5. การ Verify การเรียกใช้เมธอด
6. Argument Matchers
7. Mocking Exception และ Void Method
8. `@InjectMocks`: ฉีด Mock เข้า Object ที่ทดสอบ
9. Mock vs Stub vs Spy
10. แบบฝึกหัดและสรุป

---

## 1. ทำไมต้อง Mock: ปัญหาของการทดสอบ Dependency จริง

ทบทวนจาก Part 57 (DIP): class จริงมักมี**dependency** (database, API ภายนอก,
email service) — การทดสอบด้วย dependency จริงมีปัญหา:
- **ช้า**: เรียก database/network จริงใช้เวลานาน
- **ไม่แน่นอน**: network อาจล้ม, ข้อมูลใน database อาจเปลี่ยน
- **มี side effect**: การทดสอบอาจส่งอีเมลจริง หรือเขียนข้อมูลจริงลง database

```java
public interface EmailService {
    void sendEmail(String to, String message);
}

public class OrderService {
    private EmailService emailService;

    public OrderService(EmailService emailService) { // Dependency Injection (ทบทวนจาก Part 57)
        this.emailService = emailService;
    }

    public void completeOrder(String customerEmail) {
        // ... logic การจัดการคำสั่งซื้อ ...
        emailService.sendEmail(customerEmail, "คำสั่งซื้อของคุณเสร็จสมบูรณ์");
    }
}
```

การทดสอบ `completeOrder()` โดยใช้ `EmailService` จริง**จะส่งอีเมลจริงทุกครั้ง
ที่รัน test** — ไม่เหมาะสมเลย! **Mockito** แก้ปัญหานี้ด้วยการสร้าง**"ของปลอม"
(mock)** ที่เลียนแบบ behavior ของ dependency โดยไม่ต้องมี implementation จริง

## 2. การตั้งค่า Mockito

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
    <version>5.7.0</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-junit-jupiter</artifactId>
    <version>5.7.0</version>
    <scope>test</scope>
</dependency>
```

## 3. การสร้าง Mock ด้วย `@Mock` และ `mock()`

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class) // เปิดใช้งาน annotation ของ Mockito ร่วมกับ JUnit 5
class OrderServiceTest {

    @Mock // สร้าง mock object อัตโนมัติ (ไม่ต้องเขียน implementation จริงเลย)
    EmailService emailService;

    @Test
    void testCompleteOrder() {
        // หรือสร้างแบบไม่ใช้ annotation: EmailService mockEmail = mock(EmailService.class);

        OrderService orderService = new OrderService(emailService);
        orderService.completeOrder("test@example.com");

        // ยืนยันว่า sendEmail() ถูกเรียกจริง (ไม่ได้ส่งอีเมลจริง เพราะ emailService เป็น mock)
        verify(emailService).sendEmail("test@example.com", "คำสั่งซื้อของคุณเสร็จสมบูรณ์");
    }
}
```

**สิ่งที่เกิดขึ้นภายใน**: Mockito ใช้เทคนิคคล้าย Dynamic Proxy (ทบทวนแนวคิด
Proxy Pattern จาก Part 55 และ Reflection จาก Part 53) สร้าง object ที่
implement `EmailService` โดยอัตโนมัติ — ทุก method ของ mock **ไม่ทำอะไรเลย
โดยค่าเริ่มต้น** (คืนค่า `null`, `0`, `false` ตามชนิดข้อมูล) จนกว่าจะกำหนด
พฤติกรรมเอง (หัวข้อ 4)

## 4. `when().thenReturn()`: กำหนดพฤติกรรมของ Mock

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import static org.mockito.Mockito.*;
import static org.junit.jupiter.api.Assertions.*;

interface PaymentGateway {
    boolean processPayment(double amount);
    double getExchangeRate(String currency);
}

@ExtendWith(MockitoExtension.class)
class MockStubbingDemo {

    @Mock
    PaymentGateway paymentGateway;

    @Test
    void testStubbing() {
        // กำหนดว่า mock ควรคืนค่าอะไรเมื่อถูกเรียกด้วย argument ที่ระบุ
        when(paymentGateway.processPayment(100.0)).thenReturn(true);
        when(paymentGateway.processPayment(-50.0)).thenReturn(false);
        when(paymentGateway.getExchangeRate("USD")).thenReturn(35.5);

        assertTrue(paymentGateway.processPayment(100.0));
        assertFalse(paymentGateway.processPayment(-50.0));
        assertEquals(35.5, paymentGateway.getExchangeRate("USD"));

        // ถ้าเรียกด้วย argument ที่ไม่ได้กำหนดไว้ จะได้ค่า default (false สำหรับ boolean)
        assertFalse(paymentGateway.processPayment(999.0));
    }

    @Test
    void testMultipleReturnsInSequence() {
        // thenReturn หลายค่า: ครั้งแรกคืนค่าแรก, ครั้งที่สองเป็นต้นไปคืนค่าสุดท้ายเสมอ
        when(paymentGateway.processPayment(100.0))
            .thenReturn(true)
            .thenReturn(false); // ครั้งที่สองเป็นต้นไปจะได้ false

        assertTrue(paymentGateway.processPayment(100.0));   // ครั้งที่ 1
        assertFalse(paymentGateway.processPayment(100.0));   // ครั้งที่ 2
        assertFalse(paymentGateway.processPayment(100.0));   // ครั้งที่ 3 (คงค่าสุดท้ายไว้)
    }
}
```

## 5. การ Verify การเรียกใช้เมธอด

**`verify()`** ยืนยันว่า mock **ถูกเรียกใช้จริงตามที่คาดหวัง** — สำคัญมากเมื่อ
ทดสอบ method ที่ไม่คืนค่า (void) แต่มี side effect (เหมือนตัวอย่าง
`sendEmail()` ในหัวข้อ 3)

```java
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.junit.jupiter.api.extension.ExtendWith;
import org.junit.jupiter.api.Test;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class VerifyDemo {

    @Mock
    EmailService emailService;

    @Test
    void testVerifyCallCount() {
        emailService.sendEmail("a@test.com", "msg1");
        emailService.sendEmail("b@test.com", "msg2");

        verify(emailService, times(2)).sendEmail(any(), any()); // ถูกเรียกทั้งหมด 2 ครั้ง
        verify(emailService, atLeastOnce()).sendEmail(any(), any()); // ถูกเรียกอย่างน้อย 1 ครั้ง
        verify(emailService, never()).sendEmail("c@test.com", any()); // "ไม่เคย" ถูกเรียกด้วย argument นี้
    }

    @Test
    void testVerifyOrder() {
        emailService.sendEmail("a@test.com", "first");
        emailService.sendEmail("b@test.com", "second");

        var inOrder = inOrder(emailService); // ยืนยันลำดับการเรียกที่ถูกต้อง
        inOrder.verify(emailService).sendEmail("a@test.com", "first");
        inOrder.verify(emailService).sendEmail("b@test.com", "second");
    }
}
```

## 6. Argument Matchers

**Argument Matchers** ใช้เมื่อไม่ต้องการระบุค่า argument ที่แน่นอน แต่
ต้องการ**เงื่อนไข**แทน (ทบทวนแนวคิด `Predicate` จาก Part 40)

```java
import static org.mockito.Mockito.*;
import static org.mockito.ArgumentMatchers.*;

public class ArgumentMatchersDemo {
    void examples(PaymentGateway paymentGateway) {
        when(paymentGateway.processPayment(anyDouble())).thenReturn(true); // จำนวนอะไรก็ได้

        when(paymentGateway.processPayment(argThat(amount -> amount > 1000)))
            .thenReturn(false); // เงื่อนไขที่กำหนดเอง (คืนค่า false ถ้าจำนวนเงินมากกว่า 1000)

        verify(paymentGateway, atLeastOnce()).processPayment(eq(100.0)); // ระบุค่าตรง ๆ ผสมกับ matcher อื่นได้
    }
}
```

**ข้อสำคัญมาก**: **ถ้าใช้ matcher (เช่น `any()`) กับ argument ใดตัวหนึ่งใน
เมธอด ต้องใช้ matcher กับ**ทุก**argument ของเมธอดนั้น** ไม่สามารถผสมค่าตรง ๆ
กับ matcher ในเมธอดเดียวกันได้ (ยกเว้นใช้ `eq()` ครอบค่าธรรมดา):

```java
// ผิด: ผสม matcher กับค่าตรง ๆ โดยไม่ครอบด้วย eq()
// when(service.method(anyString(), "literal")).thenReturn(true); // Error!

// ถูก: ใช้ eq() ครอบค่าธรรมดาเมื่อผสมกับ matcher อื่น
// when(service.method(anyString(), eq("literal"))).thenReturn(true);
```

## 7. Mocking Exception และ Void Method

```java
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.junit.jupiter.api.extension.ExtendWith;
import org.junit.jupiter.api.Test;
import static org.mockito.Mockito.*;
import static org.junit.jupiter.api.Assertions.*;

@ExtendWith(MockitoExtension.class)
class ExceptionMockingDemo {

    @Mock
    PaymentGateway paymentGateway;

    @Mock
    EmailService emailService;

    @Test
    void testMockThrowsException() {
        // จำลองว่า dependency throw exception (ทดสอบ error handling โดยไม่ต้องสร้างสถานการณ์จริง)
        when(paymentGateway.processPayment(anyDouble()))
            .thenThrow(new RuntimeException("การเชื่อมต่อธนาคารล้มเหลว"));

        assertThrows(RuntimeException.class, () -> paymentGateway.processPayment(100.0));
    }

    @Test
    void testVoidMethodThrowsException() {
        // สำหรับ void method ต้องใช้ doThrow().when() แทน when().thenThrow() (syntax ต่างกัน)
        doThrow(new RuntimeException("ส่งอีเมลล้มเหลว"))
            .when(emailService).sendEmail(anyString(), anyString());

        assertThrows(RuntimeException.class,
            () -> emailService.sendEmail("test@example.com", "message"));
    }
}
```

## 8. `@InjectMocks`: ฉีด Mock เข้า Object ที่ทดสอบ

**`@InjectMocks`** สร้าง object ของ class ที่ทดสอบจริง (ไม่ใช่ mock) แล้ว
**ฉีด mock ทั้งหมดเข้าไปให้อัตโนมัติ** (ผ่าน constructor, setter, หรือ field
— เลือกวิธีที่เหมาะสมให้เอง เหมือน Dependency Injection ที่ทบทวนจาก Part 53,
57):

```java
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.junit.jupiter.api.extension.ExtendWith;
import org.junit.jupiter.api.Test;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class InjectMocksDemo {

    @Mock
    EmailService emailService; // mock ของ dependency

    @InjectMocks
    OrderService orderService; // object จริงที่ทดสอบ - Mockito จะฉีด emailService (mock) เข้าไปให้อัตโนมัติ

    @Test
    void testCompleteOrderWithInjectMocks() {
        orderService.completeOrder("customer@example.com");
        verify(emailService).sendEmail("customer@example.com", "คำสั่งซื้อของคุณเสร็จสมบูรณ์");
    }
}
```

**ข้อกำหนด**: `OrderService` ต้องมี constructor ที่รับ `EmailService` (ทบทวน
constructor injection จาก Part 57) — Mockito จะมองหา constructor ที่เหมาะสม
และฉีด mock ที่ประกาศไว้ (ด้วย `@Mock`) เข้าไปโดยอัตโนมัติ

## 9. Mock vs Stub vs Spy

| ประเภท | ความหมาย | ตัวอย่าง |
|---|---|---|
| **Mock** | Object ปลอมที่**verify การเรียกใช้ได้** (ตรวจสอบว่าถูกเรียกหรือไม่, กี่ครั้ง) | `mock(EmailService.class)` |
| **Stub** | Object ปลอมที่**คืนค่าที่กำหนดไว้ล่วงหน้า** เท่านั้น ไม่สนใจ verify | คล้าย mock ที่ใช้แค่ `when().thenReturn()` |
| **Spy** | **Object จริง**ที่ครอบด้วยความสามารถ verify/stub บางส่วน (เมธอดอื่นทำงานจริง) | `spy(new RealEmailService())` |

```java
import org.mockito.Spy;
import org.mockito.junit.jupiter.MockitoExtension;
import org.junit.jupiter.api.extension.ExtendWith;
import org.junit.jupiter.api.Test;
import static org.mockito.Mockito.*;
import static org.junit.jupiter.api.Assertions.*;

class RealCalculator {
    int add(int a, int b) { return a + b; }
    int multiply(int a, int b) { return a * b; }
}

@ExtendWith(MockitoExtension.class)
class SpyDemo {

    @Spy
    RealCalculator calculator = new RealCalculator(); // spy ครอบ object จริง

    @Test
    void testSpy() {
        // เมธอดที่ไม่ได้ stub จะทำงานแบบ "จริง" (ไม่เหมือน mock ที่คืน default เสมอ)
        assertEquals(5, calculator.add(2, 3)); // ทำงานจริง ไม่ใช่ 0 (default ของ mock)

        // stub เฉพาะบางเมธอด (ใช้ doReturn().when() สำหรับ spy เพื่อป้องกันเรียกเมธอดจริงตอน stub)
        doReturn(999).when(calculator).multiply(2, 3);
        assertEquals(999, calculator.multiply(2, 3)); // ถูก stub ให้คืนค่าปลอม
    }
}
```

**หลักปฏิบัติ**: ใช้ **Mock เป็นค่าเริ่มต้นเสมอ** สำหรับ dependency ภายนอก
(database, API, email) — ใช้ **Spy เฉพาะเมื่อต้องการทดสอบ object จริงบางส่วน
ผสมกับการปลอมบางเมธอด** (พบไม่บ่อยในทางปฏิบัติ และมักบ่งบอกว่า class นั้น
ควรถูก refactor ให้แยกความรับผิดชอบดีกว่านี้ — ทบทวน SRP จาก Part 57)

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียน test สำหรับ `NotificationService` ที่มี dependency
`SmsGateway` (mock) ยืนยันว่า `sendSms()` ถูกเรียกด้วย argument ที่ถูกต้อง
เมื่อเรียก `notifyUser()`

**เฉลย:**

```java
interface SmsGateway {
    void sendSms(String phone, String message);
}

class NotificationService {
    private SmsGateway smsGateway;
    NotificationService(SmsGateway smsGateway) { this.smsGateway = smsGateway; }
    void notifyUser(String phone) {
        smsGateway.sendSms(phone, "คุณมีการแจ้งเตือนใหม่");
    }
}
```

```java
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.junit.jupiter.api.extension.ExtendWith;
import org.junit.jupiter.api.Test;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class Exercise1 {
    @Mock SmsGateway smsGateway;

    @Test
    void testNotifyUser() {
        NotificationService service = new NotificationService(smsGateway);
        service.notifyUser("0812345678");
        verify(smsGateway).sendSms("0812345678", "คุณมีการแจ้งเตือนใหม่");
    }
}
```

**2)** ใช้ `when().thenThrow()` ทดสอบว่า `OrderService` จัดการ exception จาก
`PaymentGateway` ได้อย่างเหมาะสม (สมมติว่ามี try-catch ภายใน)

**เฉลย (แนวคิด)**: `when(paymentGateway.processPayment(anyDouble())).
thenThrow(new RuntimeException("connection failed"));` แล้วเรียกเมธอดที่ใช้
`paymentGateway` พร้อม `assertThrows()` หรือตรวจสอบว่า class จัดการ exception
นั้นได้อย่างถูกต้อง (เช่น คืนค่า false แทนการปล่อยให้ exception หลุดออกไป)

**3)** อธิบายความแตกต่างระหว่าง Mock และ Spy ด้วยคำพูดของตัวเอง

**เฉลย**: **Mock** คือ object ปลอมทั้งหมด — ทุกเมธอดที่ไม่ได้ stub ไว้จะคืน
ค่า default (null, 0, false) เสมอ ไม่มี logic จริงใด ๆ อยู่เลย เหมาะสำหรับ
dependency ที่ไม่ต้องการให้ทำงานจริง (เช่น external service) ในขณะที่
**Spy** คือ object**จริง**ที่ถูก "ครอบ" ไว้ — เมธอดที่ไม่ได้ stub จะทำงาน
**ตามปกติจริง ๆ** (เรียก implementation จริง) มีแค่เมธอดที่ระบุ stub ไว้
เท่านั้นที่ถูกปลอมแปลงค่า เหมาะสำหรับกรณีที่ต้องการทดสอบ object จริงเป็นส่วน
ใหญ่ แต่ต้องการปลอมพฤติกรรมของบางเมธอดเท่านั้น (พบใช้งานน้อยกว่า mock มาก
ในทางปฏิบัติ)

### สรุปเนื้อหา Part 59

- Mockito สร้าง "ของปลอม" (mock) แทน dependency จริง ทำให้ test เร็ว
  แน่นอน และไม่มี side effect
- `when().thenReturn()` กำหนดพฤติกรรมของ mock, `verify()` ยืนยันว่า mock
  ถูกเรียกใช้ตามที่คาดหวัง
- Argument matchers (`any()`, `eq()`, `argThat()`) ใช้แทนค่า argument ที่
  แน่นอน — ต้องใช้ทุกตัวถ้าใช้ตัวหนึ่งในเมธอดเดียวกัน
- `doThrow().when()` ใช้กับ void method, `when().thenThrow()` ใช้กับ method
  ที่คืนค่า
- `@InjectMocks` ฉีด mock เข้า object จริงที่ทดสอบโดยอัตโนมัติ
- Mock ปลอมทั้งหมด, Spy ครอบ object จริง — ใช้ Mock เป็นค่าเริ่มต้นเสมอ

**ต่อไป**: [Part 60 — Test-Driven Development (TDD) ในทางปฏิบัติ](./part-060-tdd.md)
