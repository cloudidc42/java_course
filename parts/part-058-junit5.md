# Part 58: Unit Testing ด้วย JUnit 5

> ขั้นตอนที่ 571-580 ของหลักสูตร | ระดับ: สูง

## สารบัญ

1. Unit Testing คืออะไร ทำไมต้องเขียน Test
2. การตั้งค่า JUnit 5 ด้วย Maven
3. Test แรกและ Assertions พื้นฐาน
4. Test Lifecycle: `@BeforeEach`, `@AfterEach`, `@BeforeAll`, `@AfterAll`
5. การทดสอบ Exception
6. Parameterized Tests
7. `@Nested` Tests: จัดกลุ่มการทดสอบ
8. Assertions ขั้นสูง: `assertAll`, `assertTimeout`
9. Test Naming และโครงสร้างที่ดี (AAA Pattern)
10. แบบฝึกหัดและสรุป

---

## 1. Unit Testing คืออะไร ทำไมต้องเขียน Test

**Unit Test** คือโค้ดที่**ทดสอบหน่วยเล็กที่สุดของโปรแกรม** (มักเป็นเมธอด
เดียว) แบบ**อัตโนมัติ** เพื่อยืนยันว่าทำงานถูกต้องตามที่คาดหวัง — เราเขียน
`main()` method ทดสอบด้วยมือมาตลอดทั้งหลักสูตร แต่วิธีนี้**ไม่ scale**
(ต้องรันด้วยมือทุกครั้ง, ไม่มีการยืนยันอัตโนมัติว่าผลลัพธ์ถูกต้อง)

```java
public class ManualTestingProblemDemo {
    static int add(int a, int b) { return a + b; }

    public static void main(String[] args) {
        System.out.println(add(2, 3)); // ต้องดูด้วยตาว่า "5" ถูกต้องหรือไม่ - ไม่ scale เมื่อมีเมธอดเยอะ ๆ
    }
}
```

**ประโยชน์ของ Unit Testing**:
- **ยืนยันความถูกต้องอัตโนมัติ**: ไม่ต้องดูผลลัพธ์ด้วยตา
- **ป้องกัน regression**: รัน test ทั้งหมดซ้ำได้ทุกครั้งที่แก้โค้ด มั่นใจว่า
  ไม่ทำของเดิมพัง
- **เป็นเอกสารที่ใช้งานได้จริง**: test อธิบายว่าโค้ดควรทำงานอย่างไร (ดีกว่า
  comment ที่อาจล้าสมัย — ทบทวนจาก Part 57)
- **ส่งเสริมการออกแบบที่ดี**: โค้ดที่ testable มักมี coupling ต่ำ (ทบทวน DIP
  จาก Part 57)

## 2. การตั้งค่า JUnit 5 ด้วย Maven

**JUnit 5** (หรือ "JUnit Jupiter") เป็นเวอร์ชันปัจจุบันที่ใช้กันแพร่หลาย
(Part 61 จะสอน Maven เต็มรูปแบบ แต่ให้เห็นตั้งแต่ตอนนี้):

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>5.10.0</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

โครงสร้างโฟลเดอร์มาตรฐาน (ทบทวนจาก Part 20):

```
src/
├── main/java/com/example/Calculator.java   <- source code หลัก
└── test/java/com/example/CalculatorTest.java <- test code (โครงสร้าง package ตรงกัน)
```

## 3. Test แรกและ Assertions พื้นฐาน

```java
// src/main/java/com/example/Calculator.java
package com.example;

public class Calculator {
    public int add(int a, int b) {
        return a + b;
    }

    public int divide(int a, int b) {
        if (b == 0) {
            throw new ArithmeticException("หารด้วยศูนย์ไม่ได้");
        }
        return a / b;
    }
}
```

```java
// src/test/java/com/example/CalculatorTest.java
package com.example;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class CalculatorTest {

    @Test // annotation บอก JUnit ว่านี่คือ test method (ทบทวน custom annotation จาก Part 52)
    void testAddition() {
        Calculator calc = new Calculator();
        int result = calc.add(2, 3);
        assertEquals(5, result); // ยืนยันว่า result เท่ากับ 5 (expected มาก่อน actual)
    }

    @Test
    void testAdditionWithNegativeNumbers() {
        Calculator calc = new Calculator();
        assertEquals(-1, calc.add(2, -3));
    }
}
```

Assertions พื้นฐานที่ใช้บ่อย:

```java
import static org.junit.jupiter.api.Assertions.*;

public class AssertionsDemo {
    void examples() {
        assertEquals(5, 2 + 3);              // เท่ากัน
        assertNotEquals(4, 2 + 3);            // ไม่เท่ากัน
        assertTrue(5 > 3);                     // เป็น true
        assertFalse(3 > 5);                     // เป็น false
        assertNull(null);                        // เป็น null
        assertNotNull("hello");                   // ไม่เป็น null
        assertSame(new int[0].getClass(), int[].class); // reference เดียวกัน (ทบทวน == จาก Part 4)
        assertArrayEquals(new int[]{1,2,3}, new int[]{1,2,3}); // array มีค่าเหมือนกันทุก element
        assertEquals(3.14159, Math.PI, 0.001);       // เทียบ double พร้อม delta (ความคลาดเคลื่อนที่ยอมรับได้)
    }
}
```

**ข้อสำคัญเรื่อง `assertEquals` กับ `double`**: การเทียบทศนิยมตรง ๆ ด้วย `==`
เสี่ยงผิดพลาดจาก floating-point precision (ทบทวนจาก Part 3) จึงต้องระบุ
**delta** (ค่าความคลาดเคลื่อนที่ยอมรับได้) เสมอ

## 4. Test Lifecycle: `@BeforeEach`, `@AfterEach`, `@BeforeAll`, `@AfterAll`

```java
import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

class LifecycleDemoTest {
    Calculator calculator;

    @BeforeAll // static method: รันครั้งเดียวก่อน test ทั้งหมดในคลาสนี้ (เช่น เชื่อมต่อฐานข้อมูล)
    static void setUpClass() {
        System.out.println("เริ่มการทดสอบทั้งหมด");
    }

    @BeforeEach // รันก่อน "ทุก" test method (ใช้สร้าง fresh state ป้องกัน test รบกวนกันเอง)
    void setUp() {
        calculator = new Calculator();
        System.out.println("เตรียม Calculator ใหม่");
    }

    @Test
    void testAdd() {
        assertEquals(5, calculator.add(2, 3));
    }

    @Test
    void testDivide() {
        assertEquals(2, calculator.divide(10, 5));
    }

    @AfterEach // รันหลัง "ทุก" test method (ใช้ล้างข้อมูล/ปิด resource)
    void tearDown() {
        System.out.println("ล้างข้อมูลหลังการทดสอบ");
    }

    @AfterAll // static method: รันครั้งเดียวหลัง test ทั้งหมด (เช่น ปิดการเชื่อมต่อฐานข้อมูล)
    static void tearDownClass() {
        System.out.println("จบการทดสอบทั้งหมด");
    }
}
```

**ทำไมต้องมี `@BeforeEach`**: สร้าง object ใหม่ **ทุกครั้ง**ก่อน test แต่ละตัว
เพื่อ**ป้องกัน test รบกวนกันเอง** (test isolation) — ถ้า test สองตัวใช้
`calculator` ตัวเดียวกัน และ test แรกแก้ไข state บางอย่าง อาจทำให้ test ที่
สองได้ผลลัพธ์ผิดเพี้ยนโดยไม่เกี่ยวกับ bug จริง

## 5. การทดสอบ Exception

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class ExceptionTestDemo {
    @Test
    void testDivideByZeroThrowsException() {
        Calculator calc = new Calculator();

        // assertThrows: ยืนยันว่าโค้ดใน lambda throw exception ชนิดที่ระบุ
        ArithmeticException exception = assertThrows(
            ArithmeticException.class,
            () -> calc.divide(10, 0)
        );

        assertEquals("หารด้วยศูนย์ไม่ได้", exception.getMessage()); // ตรวจสอบ message ด้วย
    }

    @Test
    void testNoExceptionThrown() {
        Calculator calc = new Calculator();
        assertDoesNotThrow(() -> calc.add(1, 2)); // ยืนยันว่า "ไม่" throw exception
    }
}
```

## 6. Parameterized Tests

**Parameterized Test** ทดสอบ**เมธอดเดียวด้วยหลายชุดข้อมูล** โดยไม่ต้องเขียน
`@Test` ซ้ำ ๆ (ทบทวนแนวคิด DRY จาก Part 8):

```java
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;
import org.junit.jupiter.params.provider.ValueSource;
import static org.junit.jupiter.api.Assertions.*;

class ParameterizedTestDemo {

    @ParameterizedTest
    @ValueSource(ints = {2, 4, 6, 8, 100}) // ทดสอบด้วยหลายค่า ไม่ต้องเขียน @Test ซ้ำ
    void testIsEven(int number) {
        assertTrue(number % 2 == 0);
    }

    @ParameterizedTest
    @CsvSource({
        "2, 3, 5",   // a=2, b=3, expected=5
        "10, 20, 30",
        "-5, 5, 0"
    })
    void testAddition(int a, int b, int expected) {
        Calculator calc = new Calculator();
        assertEquals(expected, calc.add(a, b));
    }
}
```

**ผลลัพธ์**: การทดสอบด้วย `@CsvSource` 3 บรรทัดข้างบน**เทียบเท่ากับเขียน
`@Test` แยก 3 method** แต่กระชับกว่ามาก และเพิ่ม test case ใหม่ทำได้เพียง
เพิ่มบรรทัดข้อมูล ไม่ต้องเขียนโค้ดใหม่

## 7. `@Nested` Tests: จัดกลุ่มการทดสอบ

**`@Nested`** จัดกลุ่ม test ที่เกี่ยวข้องกันเป็นหมวดหมู่ ทำให้ผลลัพธ์การรัน
test อ่านง่ายขึ้น (โดยเฉพาะเมื่อทดสอบหลาย scenario ของเมธอดเดียวกัน):

```java
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class BankAccountTest {

    @Nested
    class WhenBalanceIsPositive {
        BankAccount account = new BankAccount(1000);

        @Test
        void canWithdraw() {
            account.withdraw(500);
            assertEquals(500, account.getBalance());
        }

        @Test
        void canDeposit() {
            account.deposit(500);
            assertEquals(1500, account.getBalance());
        }
    }

    @Nested
    class WhenBalanceIsZero {
        BankAccount account = new BankAccount(0);

        @Test
        void cannotWithdraw() {
            assertThrows(IllegalStateException.class, () -> account.withdraw(100));
        }
    }

    static class BankAccount {
        double balance;
        BankAccount(double balance) { this.balance = balance; }
        void withdraw(double amount) {
            if (amount > balance) throw new IllegalStateException("ยอดเงินไม่พอ");
            balance -= amount;
        }
        void deposit(double amount) { balance += amount; }
        double getBalance() { return balance; }
    }
}
```

## 8. Assertions ขั้นสูง: `assertAll`, `assertTimeout`

```java
import org.junit.jupiter.api.Test;
import java.time.Duration;
import static org.junit.jupiter.api.Assertions.*;

class AdvancedAssertionsDemo {

    @Test
    void testMultipleProperties() {
        Calculator calc = new Calculator();

        // assertAll: รันทุก assertion แม้บางตัวจะ fail (รายงาน fail ทั้งหมดพร้อมกัน)
        // ต่างจากการเขียน assertEquals แยกกันหลายบรรทัด ที่จะหยุดทันทีที่ตัวแรก fail
        assertAll("การคำนวณของ Calculator",
            () -> assertEquals(5, calc.add(2, 3)),
            () -> assertEquals(2, calc.divide(10, 5)),
            () -> assertEquals(-1, calc.add(2, -3))
        );
    }

    @Test
    void testPerformance() {
        assertTimeout(Duration.ofMillis(100), () -> {
            Thread.sleep(50); // จำลองงานที่ควรเสร็จภายใน 100ms
        });
    }
}
```

## 9. Test Naming และโครงสร้างที่ดี (AAA Pattern)

**AAA Pattern** (Arrange-Act-Assert) เป็นโครงสร้างมาตรฐานของ unit test ที่ดี:

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class AAAPatternDemo {

    @Test
    void shouldReturnSumWhenAddingTwoPositiveNumbers() { // ชื่อบอกสถานการณ์และผลลัพธ์ที่คาดหวังชัดเจน
        // Arrange: เตรียมข้อมูลและ object ที่ต้องใช้
        Calculator calculator = new Calculator();
        int a = 5;
        int b = 3;

        // Act: เรียกเมธอดที่ต้องการทดสอบ
        int result = calculator.add(a, b);

        // Assert: ตรวจสอบผลลัพธ์
        assertEquals(8, result);
    }
}
```

**ธรรมเนียมการตั้งชื่อ test method ที่นิยม**:
- `shouldXxxWhenYyy()`: "ควรทำ X เมื่อเงื่อนไข Y"
- `testXxx_Yyy_Zzz()`: method_scenario_expectedResult
- ชื่อควรอ่านแล้วเข้าใจว่ากำลังทดสอบอะไร**โดยไม่ต้องดูเนื้อโค้ดข้างใน** —
  รายงานผลการทดสอบที่ fail ควรบอกปัญหาได้ทันทีจากชื่อ test เพียงอย่างเดียว

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียน unit test ครบถ้วนสำหรับเมธอด `isPrime(int n)` (ทบทวนจาก Part 6)
ครอบคลุมกรณี: จำนวนเฉพาะ, ไม่ใช่จำนวนเฉพาะ, เลข 0, 1, และเลขติดลบ

**เฉลย:**

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;
import static org.junit.jupiter.api.Assertions.*;

class PrimeCheckerTest {
    static boolean isPrime(int n) {
        if (n < 2) return false;
        for (int i = 2; i * i <= n; i++) if (n % i == 0) return false;
        return true;
    }

    @ParameterizedTest
    @ValueSource(ints = {2, 3, 5, 7, 11, 13})
    void shouldReturnTrueForPrimeNumbers(int n) {
        assertTrue(isPrime(n));
    }

    @ParameterizedTest
    @ValueSource(ints = {0, 1, 4, 6, 8, 9, -5})
    void shouldReturnFalseForNonPrimeNumbers(int n) {
        assertFalse(isPrime(n));
    }
}
```

**2)** เขียน `@Nested` test class สำหรับ `Stack` (ทบทวนจาก Part 25, 31) แบ่ง
เป็น "WhenEmpty" และ "WhenNotEmpty"

**เฉลย:**

```java
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;
import java.util.ArrayDeque;
import java.util.Deque;
import static org.junit.jupiter.api.Assertions.*;

class StackTest {
    @Nested
    class WhenEmpty {
        Deque<Integer> stack = new ArrayDeque<>();

        @Test
        void isEmptyReturnsTrue() {
            assertTrue(stack.isEmpty());
        }

        @Test
        void popThrowsException() {
            assertThrows(java.util.NoSuchElementException.class, stack::pop);
        }
    }

    @Nested
    class WhenNotEmpty {
        Deque<Integer> stack = new ArrayDeque<>(java.util.List.of(1, 2, 3));

        @Test
        void peekReturnsTopElement() {
            assertEquals(1, stack.peek());
        }
    }
}
```

**3)** อธิบายว่าทำไม `@BeforeEach` สำคัญสำหรับ test isolation

**เฉลย**: `@BeforeEach` รันก่อน**ทุก**test method สร้าง state ใหม่ (fresh
state) เสมอ — ถ้าไม่มี `@BeforeEach` และ test หลายตัวใช้ shared object
เดียวกัน ผลลัพธ์ของ test หนึ่งอาจ**รั่วไหล**ไปกระทบ test อื่นที่รันหลังจากนั้น
(เช่น test แรกเพิ่มข้อมูลเข้า list แล้ว test ที่สองที่คาดว่า list เป็น list
ว่างจะ fail อย่างไม่คาดคิด) — การมี fresh state ทุกครั้งทำให้แต่ละ test เป็น
**อิสระจากกันอย่างสมบูรณ์** (test independence) ไม่ว่าจะรันตามลำดับไหนหรือรัน
แบบ parallel ก็ได้ผลลัพธ์เหมือนกันเสมอ (ทบทวนความสำคัญของการหลีกเลี่ยง shared
mutable state จาก Part 50)

### สรุปเนื้อหา Part 58

- Unit Test ยืนยันความถูกต้องอัตโนมัติ ป้องกัน regression และเป็นเอกสารที่
  ใช้งานได้จริง
- `@Test` ประกาศ test method, assertion methods (`assertEquals`,
  `assertTrue` ฯลฯ) ยืนยันผลลัพธ์ที่คาดหวัง
- `@BeforeEach`/`@AfterEach` รันก่อน/หลังทุก test เพื่อ test isolation,
  `@BeforeAll`/`@AfterAll` รันครั้งเดียว
- `assertThrows` ทดสอบว่าโค้ด throw exception ที่คาดหวัง
- `@ParameterizedTest` ทดสอบเมธอดเดียวด้วยหลายชุดข้อมูลโดยไม่ต้องเขียนซ้ำ
- AAA Pattern (Arrange-Act-Assert) เป็นโครงสร้างมาตรฐานของ test ที่ดี

**ต่อไป**: [Part 59 — Mocking ด้วย Mockito](./part-059-mockito.md)
