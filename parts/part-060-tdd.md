# Part 60: Test-Driven Development (TDD) ในทางปฏิบัติ

> ขั้นตอนที่ 591-600 ของหลักสูตร | ระดับ: สูง (จบหมวด Testing)

## สารบัญ

1. TDD คืออะไร
2. วงจร Red-Green-Refactor
3. ตัวอย่างเต็มรูปแบบ: สร้าง FizzBuzz ด้วย TDD
4. ตัวอย่างที่ซับซ้อนขึ้น: สร้าง Stack ด้วย TDD
5. TDD ช่วยขับเคลื่อนการออกแบบอย่างไร
6. Test Coverage: วัดผลแต่ไม่ใช่เป้าหมายสูงสุด
7. Test Doubles Recap: เมื่อไรใช้อะไร
8. ข้อจำกัดและข้อวิจารณ์ของ TDD
9. Best Practices ของการเขียน Test ที่ดี (FIRST Principles)
10. แบบฝึกหัดและสรุปหมวด Testing

---

## 1. TDD คืออะไร

**Test-Driven Development (TDD)** คือแนวทางการพัฒนาที่**เขียน test ก่อนเขียน
โค้ดจริง** — ฟังดูสวนทางกับสัญชาตญาณ แต่มีเหตุผลที่ลึกซึ้ง: การเขียน test
ก่อนบังคับให้เรา**คิดถึง API และพฤติกรรมที่ต้องการก่อนลงมือ implement**
ทำให้ได้ design ที่ดีกว่าและมั่นใจว่าโค้ดทำงานถูกต้องตั้งแต่บรรทัดแรก

## 2. วงจร Red-Green-Refactor

TDD ทำงานเป็นวงจรสั้น ๆ ซ้ำไปเรื่อย ๆ:

```
     ┌──────────────┐
     │   1. RED      │  เขียน test ที่ fail ก่อน (เพราะยังไม่มีโค้ดจริง)
     │  (test fails)  │
     └───────┬───────┘
             │
     ┌───────▼───────┐
     │   2. GREEN     │  เขียนโค้ดให้ "น้อยที่สุด" ที่ทำให้ test ผ่าน
     │ (test passes)   │  (ไม่ต้อง perfect แค่ผ่าน test ก่อน)
     └───────┬───────┘
             │
     ┌───────▼───────┐
     │  3. REFACTOR   │  ปรับปรุงโค้ดให้สะอาดขึ้น (ทบทวน Clean Code จาก Part 57)
     │ (คงพฤติกรรมเดิม) │  โดยที่ test ยังผ่านอยู่เสมอ (การันตีว่าไม่ทำอะไรพัง)
     └───────┬───────┘
             │
             └──────────► กลับไปที่ 1 (เขียน test ถัดไปสำหรับ feature ต่อไป)
```

## 3. ตัวอย่างเต็มรูปแบบ: สร้าง FizzBuzz ด้วย TDD

ทบทวนโจทย์ FizzBuzz จาก Part 6 — คราวนี้เราจะสร้างมันด้วย TDD ทีละขั้น

**รอบที่ 1 — RED: เขียน test แรกก่อนมี implementation เลย**

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class FizzBuzzTest {
    @Test
    void shouldReturnNumberAsStringWhenNotDivisibleByThreeOrFive() {
        assertEquals("1", FizzBuzz.convert(1)); // ยัง compile ไม่ผ่านเลย เพราะ FizzBuzz.convert() ไม่มีอยู่
    }
}
```

**รอบที่ 1 — GREEN: เขียนโค้ดน้อยที่สุดให้ผ่าน**

```java
public class FizzBuzz {
    public static String convert(int n) {
        return String.valueOf(n); // แค่พอให้ test แรกผ่าน ยังไม่ต้องสมบูรณ์
    }
}
```

**รอบที่ 2 — RED: เพิ่ม test สำหรับกรณี "Fizz"**

```java
class FizzBuzzTest {
    @Test
    void shouldReturnNumberAsStringWhenNotDivisibleByThreeOrFive() {
        assertEquals("1", FizzBuzz.convert(1));
    }

    @Test
    void shouldReturnFizzWhenDivisibleByThree() {
        assertEquals("Fizz", FizzBuzz.convert(3)); // fail! เพราะ convert() ยังคืนแค่ String.valueOf(n)
    }
}
```

**รอบที่ 2 — GREEN: แก้ไขให้ผ่านทั้งสอง test**

```java
public class FizzBuzz {
    public static String convert(int n) {
        if (n % 3 == 0) return "Fizz";
        return String.valueOf(n);
    }
}
```

**รอบที่ 3 — RED: เพิ่ม test สำหรับ "Buzz" และ "FizzBuzz"**

```java
class FizzBuzzTest {
    // ... test เดิม ...

    @Test
    void shouldReturnBuzzWhenDivisibleByFive() {
        assertEquals("Buzz", FizzBuzz.convert(5)); // fail!
    }

    @Test
    void shouldReturnFizzBuzzWhenDivisibleByBothThreeAndFive() {
        assertEquals("FizzBuzz", FizzBuzz.convert(15)); // fail! (15 % 3 == 0 ทำให้ได้ "Fizz" ผิด)
    }
}
```

**รอบที่ 3 — GREEN: แก้ไขให้ผ่านทั้งหมด (ต้องเช็ค 15 ก่อน 3 และ 5)**

```java
public class FizzBuzz {
    public static String convert(int n) {
        if (n % 15 == 0) return "FizzBuzz"; // ต้องเช็ค case ที่ specific ที่สุดก่อน (ทบทวน Part 5)
        if (n % 3 == 0) return "Fizz";
        if (n % 5 == 0) return "Buzz";
        return String.valueOf(n);
    }
}
```

**REFACTOR**: โค้ดนี้กระชับพออยู่แล้ว ไม่จำเป็นต้อง refactor เพิ่ม — ทุก test
ยังผ่านหมด (สังเกตว่า**test ทุกตัวที่เขียนไปแล้วยังต้องผ่านตลอด**แม้เพิ่ม
feature ใหม่ — นี่คือการป้องกัน regression โดยอัตโนมัติ)

## 4. ตัวอย่างที่ซับซ้อนขึ้น: สร้าง Stack ด้วย TDD

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class MyStackTest {

    @Test
    void newStackShouldBeEmpty() { // Test 1: RED -> GREEN
        MyStack<Integer> stack = new MyStack<>();
        assertTrue(stack.isEmpty());
    }

    @Test
    void afterPushStackShouldNotBeEmpty() { // Test 2: RED -> GREEN
        MyStack<Integer> stack = new MyStack<>();
        stack.push(1);
        assertFalse(stack.isEmpty());
    }

    @Test
    void popShouldReturnLastPushedElement() { // Test 3: RED -> GREEN
        MyStack<Integer> stack = new MyStack<>();
        stack.push(1);
        stack.push(2);
        assertEquals(2, stack.pop());
    }

    @Test
    void popOnEmptyStackShouldThrowException() { // Test 4: RED -> GREEN
        MyStack<Integer> stack = new MyStack<>();
        assertThrows(java.util.NoSuchElementException.class, stack::pop);
    }
}
```

Implementation ที่ทำให้ทุก test ผ่าน (สร้างขึ้นทีละขั้นตามลำดับ test):

```java
import java.util.ArrayDeque;
import java.util.Deque;
import java.util.NoSuchElementException;

public class MyStack<T> {
    private Deque<T> items = new ArrayDeque<>(); // ทบทวน ArrayDeque จาก Part 25

    public boolean isEmpty() {
        return items.isEmpty();
    }

    public void push(T item) {
        items.push(item);
    }

    public T pop() {
        if (isEmpty()) {
            throw new NoSuchElementException("Stack is empty");
        }
        return items.pop();
    }
}
```

**สังเกต**: การเขียน test ก่อนทำให้เรา**ค้นพบ edge case** (`popOnEmptyStack`)
**ตั้งแต่ตอนออกแบบ** แทนที่จะไปเจอ bug ตอน production

## 5. TDD ช่วยขับเคลื่อนการออกแบบอย่างไร

**TDD** ไม่ใช่แค่เทคนิคการทดสอบ แต่เป็น**เทคนิคการออกแบบ** — เพราะการเขียน
test ก่อนบังคับให้ตอบคำถามสำคัญตั้งแต่ต้น:
- **API หน้าตาเป็นอย่างไร**: เรียกใช้ยังไง รับ-คืนค่าอะไร
- **Dependency คืออะไร**: ต้อง mock อะไรบ้าง (ทบทวนจาก Part 59) — ถ้า mock
  ยากมาก มักบ่งบอกว่า design มีปัญหา (coupling สูงเกินไป)
- **สัญญาระหว่าง class ชัดเจนหรือไม่**: test คือ "ตัวอย่างการใช้งานจริง"
  ของ API นั้น

โค้ดที่เขียนด้วย TDD มักมี**coupling ต่ำและ cohesion สูง** (ทบทวนหลักการ SOLID
จาก Part 57) เพราะโค้ดที่ testable ง่ายมักออกแบบมาดีโดยธรรมชาติ

## 6. Test Coverage: วัดผลแต่ไม่ใช่เป้าหมายสูงสุด

**Code Coverage** คือเปอร์เซ็นต์ของโค้ดที่ถูกรันระหว่าง test — เครื่องมือ
เช่น **JaCoCo** วัดค่านี้ได้ แต่**coverage สูงไม่ได้แปลว่า test ดี**:

```java
public class CoverageMisleadingDemo {
    static int divide(int a, int b) {
        return a / b; // บรรทัดนี้ถูก "cover" แต่ไม่ได้ทดสอบกรณี b=0 เลย!
    }

    // Test ที่ได้ 100% coverage แต่ไม่มีคุณภาพ
    // @Test void test() { assertEquals(5, divide(10, 2)); } // ไม่ทดสอบ edge case (b=0) เลย
}
```

**บทเรียนสำคัญ**: **100% coverage ไม่ได้การันตีว่าไม่มี bug** — Coverage บอก
แค่ว่า "บรรทัดนี้ถูกรัน" ไม่ได้บอกว่า "ถูกทดสอบครบทุก scenario ที่สำคัญ"
ควรใช้ coverage เป็น**เครื่องมือหาจุดที่ยังไม่มี test เลย** มากกว่าเป็น
**เป้าหมายที่ต้องไล่ตามให้ถึงตัวเลขที่กำหนด**

## 7. Test Doubles Recap: เมื่อไรใช้อะไร

ทบทวนและสรุปจาก Part 59:

| Test Double | ใช้เมื่อ |
|---|---|
| **Mock** (Part 59) | ต้องการ verify ว่า dependency ถูกเรียกใช้ตามที่คาดหวัง |
| **Stub** | ต้องการให้ dependency คืนค่าที่กำหนดไว้ล่วงหน้า โดยไม่สนใจ verify |
| **Fake** | Implementation แบบง่าย ๆ ที่ทำงานได้จริงแต่ไม่เหมาะกับ production (เช่น in-memory database แทน database จริง) |
| **Spy** (Part 59) | ต้องการ object จริงบางส่วน ผสมกับการปลอมบางเมธอด |

```java
import java.util.HashMap;
import java.util.Map;

// ตัวอย่าง Fake: in-memory implementation ใช้แทน database จริงใน test (เร็วกว่ามาก ไม่ต้องต่อ database จริง)
public class FakeUserRepository implements UserRepository {
    private Map<String, String> storage = new HashMap<>();

    public void save(String id, String name) { storage.put(id, name); }
    public String findById(String id) { return storage.get(id); }
}

interface UserRepository {
    void save(String id, String name);
    String findById(String id);
}
```

## 8. ข้อจำกัดและข้อวิจารณ์ของ TDD

**TDD ไม่ใช่เครื่องมือวิเศษที่เหมาะกับทุกสถานการณ์**:

1. **UI/exploratory code**: เมื่อยังไม่แน่ใจว่า API ควรหน้าตาเป็นอย่างไร
   (กำลัง prototype/สำรวจไอเดีย) การเขียน test ก่อนอาจทำให้ช้าและต้องเขียน
   ใหม่บ่อย — บางครั้ง "spike" (โค้ดทดลองแบบไม่มี test) ก่อนแล้วค่อยเขียน
   test ทีหลังก็เหมาะสมกว่า
2. **การเขียน test ก่อนไม่ได้ป้องกัน bad design โดยอัตโนมัติ**: ยังต้องอาศัย
   ความเข้าใจ SOLID (Part 57) และ Design Patterns (Part 54-56) ร่วมด้วย
3. **Over-mocking**: การ mock มากเกินไปอาจทำให้ test ผูกติดกับ
   implementation details มากกว่าพฤติกรรมที่แท้จริง (test เปราะบาง แก้โค้ด
   เล็กน้อยก็ทำ test พังทั้งที่ behavior ไม่เปลี่ยน)

**หลักปฏิบัติที่สมดุล**: ใช้ TDD เมื่อ requirement ชัดเจนพอที่จะเขียน test
ได้อย่างมีความหมาย ยืดหยุ่นได้เมื่อกำลังสำรวจไอเดียใหม่ ๆ

## 9. Best Practices ของการเขียน Test ที่ดี (FIRST Principles)

**FIRST** เป็นตัวย่อของหลักการที่ test ที่ดีควรมี:

- **Fast**: รันเร็ว (ทบทวนเหตุผลที่ต้อง mock จาก Part 59 — test ที่ช้าจะไม่มี
  ใครอยากรันบ่อย ๆ)
- **Independent**: แต่ละ test ไม่ควร depend on test อื่น (ทบทวน test
  isolation จาก Part 58)
- **Repeatable**: รันซ้ำได้ผลลัพธ์เหมือนกันทุกครั้ง (ไม่ควรพึ่งพา network,
  วันที่ปัจจุบัน, ค่าสุ่มที่ไม่ควบคุม)
- **Self-Validating**: test บอกผ่าน/fail ได้ชัดเจนในตัวเอง (ผ่าน assertion
  ไม่ต้องดูผลลัพธ์ด้วยตา)
- **Timely**: เขียนในเวลาที่เหมาะสม (TDD คือเขียนก่อน แต่แม้เขียนทีหลัง ก็
  ควรเขียนใกล้เคียงกับตอนที่เขียนโค้ดจริง ไม่ทิ้งไว้นาน)

```java
public class FIRSTPrincipleViolationDemo {
    // ผิด Repeatable: test ผลลัพธ์ขึ้นกับวันที่ปัจจุบัน (จะ fail ในบางวัน)
    // @Test void testIsWeekday() {
    //     assertTrue(LocalDate.now().getDayOfWeek() != DayOfWeek.SATURDAY);
    // }

    // ถูก: inject วันที่เข้าไปแทนพึ่งพา LocalDate.now() ตรง ๆ (testable และ repeatable)
    static boolean isWeekday(java.time.LocalDate date) {
        var day = date.getDayOfWeek();
        return day != java.time.DayOfWeek.SATURDAY && day != java.time.DayOfWeek.SUNDAY;
    }
    // @Test void testIsWeekday() {
    //     assertFalse(isWeekday(LocalDate.of(2024, 1, 13))); // วันเสาร์ที่แน่นอน - ผลลัพธ์เดิมเสมอ
    // }
}
```

## 10. แบบฝึกหัดและสรุปหมวด Testing

### แบบฝึกหัด

**1)** ใช้ TDD สร้างเมธอด `isPalindrome(String s)` (ทบทวนจาก Part 9) — เขียน
test ก่อน implementation ทีละขั้น

**เฉลย (ลำดับ TDD):**

```java
// รอบ 1: test "level" -> true
// รอบ 2: test "hello" -> false
// รอบ 3: test "" -> true (edge case: string ว่าง)
// รอบ 4: test "A man a plan a canal Panama" -> true (ignore case + space)

public class Palindrome {
    public static boolean isPalindrome(String s) {
        String cleaned = s.toLowerCase().replaceAll("[^a-z0-9]", "");
        return cleaned.equals(new StringBuilder(cleaned).reverse().toString());
    }
}
```

**2)** อธิบายว่าทำไม "Independent" (จาก FIRST) สำคัญสำหรับ test suite ขนาด
ใหญ่ที่รันแบบ parallel

**เฉลย**: ถ้า test ไม่ independent (เช่น test B ต้องรันหลัง test A เสมอ
เพราะพึ่งพา side effect ที่ A สร้างไว้) การรัน test แบบ parallel (เพื่อความ
เร็ว — ทบทวนแนวคิดจาก Part 46-50) จะทำให้ผลลัพธ์**ไม่แน่นอน** (test อาจผ่าน
หรือ fail แบบสุ่มขึ้นกับลำดับการรันจริง) ทำให้ debug ยากมากและบั่นทอนความ
เชื่อมั่นในการทดสอบทั้งระบบ — test ที่ independent อย่างสมบูรณ์รันในลำดับไหน
หรือรันพร้อมกันก็ได้ผลลัพธ์เหมือนกันเสมอ

**3)** วิจารณ์ test ต่อไปนี้ตามหลัก FIRST และเสนอวิธีแก้:

```java
@Test
void testUserRegistration() {
    UserService service = new UserService(new RealDatabase(), new RealEmailSender());
    service.register("test@example.com");
    // เช็คผลลัพธ์โดยดู log ที่พิมพ์ออกมาด้วยตา
}
```

**เฉลย**: ละเมิดหลาย FIRST principles: **Fast** (ใช้ database และ email
sender จริง ทำให้ช้า), **Repeatable** (ผลลัพธ์อาจต่างกันถ้า database มี
ข้อมูลเดิมอยู่แล้ว), **Self-Validating** (ต้องดู log ด้วยตา ไม่มี assertion
อัตโนมัติ) — วิธีแก้: ใช้ Mockito mock ทั้ง `RealDatabase` และ
`RealEmailSender` (ทบทวนจาก Part 59), ใช้ `verify()`/`assertEquals()` แทน
การดู log ด้วยตา

### สรุปเนื้อหา Part 60 และหมวด Testing (Part 58-60)

- TDD คือเขียน test ก่อนโค้ดจริง ทำงานเป็นวงจร Red-Green-Refactor
- การเขียน test ก่อนช่วยขับเคลื่อนการออกแบบ API ที่ดีและค้นพบ edge case
  ตั้งแต่ต้น
- Code Coverage เป็นเครื่องมือหาจุดที่ยังไม่มี test ไม่ใช่เป้าหมายสูงสุด
  (coverage สูงไม่การันตีว่าไม่มี bug)
- Mock, Stub, Fake, Spy เป็น test doubles ที่เลือกใช้ตามสถานการณ์
- TDD ไม่เหมาะกับทุกสถานการณ์ (เช่น exploratory code) — ต้องยืดหยุ่นได้
- FIRST Principles (Fast, Independent, Repeatable, Self-Validating, Timely)
  เป็นเกณฑ์ตัดสิน test ที่ดี

**จบหมวด Testing (Part 58-60) อย่างสมบูรณ์! ต่อไปจะเข้าสู่เรื่อง Build Tools
(Maven, Gradle) ซึ่งเป็นเครื่องมือที่ใช้จัดการ dependency และรัน test เหล่านี้
ในโปรเจกต์จริง**

**ต่อไป**: [Part 61 — Build Tools: Maven เบื้องต้นถึงขั้นสูง](./part-061-maven.md)
