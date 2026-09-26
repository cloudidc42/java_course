# Part 43: Optional Class และการเขียนโค้ด Null-safe

> ขั้นตอนที่ 421-430 ของหลักสูตร | ระดับ: กลาง-สูง

## สารบัญ

1. ปัญหาของ `null` และ "The Billion Dollar Mistake"
2. `Optional<T>` คืออะไร
3. การสร้าง Optional
4. การตรวจสอบและดึงค่าจาก Optional
5. `map()`, `filter()`, `flatMap()` บน Optional
6. `orElse()`, `orElseGet()`, `orElseThrow()` เปรียบเทียบ
7. ข้อผิดพลาดที่พบบ่อยในการใช้ Optional
8. Optional กับ Field และ Method Parameter (ไม่แนะนำ)
9. `Optional` Primitive Specializations
10. แบบฝึกหัดและสรุป

---

## 1. ปัญหาของ `null` และ "The Billion Dollar Mistake"

**Tony Hoare** ผู้คิดค้นแนวคิด `null` reference ยอมรับในปี 2009 ว่ามันเป็น
**"ความผิดพลาดมูลค่าพันล้านดอลลาร์"** เพราะทำให้เกิด `NullPointerException`
(NPE) นับไม่ถ้วนตลอดประวัติศาสตร์การเขียนโปรแกรม — ปัญหาหลักคือ **`null` ไม่มี
สัญญาณเตือนใด ๆ ตอน compile-time** ว่าตัวแปรอาจไม่มีค่า

```java
public class NullProblemDemo {
    static String findUserEmail(String userId) {
        if (userId.equals("guest")) {
            return null; // ไม่มีอีเมลสำหรับ guest - เป็นเรื่องปกติทางธุรกิจ
        }
        return userId + "@example.com";
    }

    public static void main(String[] args) {
        String email = findUserEmail("guest");
        System.out.println(email.toUpperCase()); // NullPointerException! ไม่มีคำเตือนใด ๆ ตอน compile
                                                     // ผู้เรียกใช้ไม่รู้เลยว่าต้องเช็ค null ก่อน
    }
}
```

**ปัญหาสำคัญ**: จาก signature `String findUserEmail(String userId)` ไม่มีทาง
รู้ได้เลยว่าเมธอดนี้**อาจคืนค่า `null`** — ผู้เรียกใช้ต้องอ่าน**เอกสาร**หรือ
**source code**เพื่อรู้ ซึ่งมักถูกมองข้ามจนเกิด bug

## 2. `Optional<T>` คืออะไร

**`Optional<T>`** (Java 8+) คือ **container object** ที่**อาจมีค่าหรือไม่มี
ค่า** — ใช้เป็น**สัญญาณชัดเจนใน type system**ว่า "ค่านี้อาจไม่มีอยู่จริง"
บังคับให้ผู้เรียกใช้**ต้องจัดการกับกรณีที่ไม่มีค่าอย่างชัดเจน**

```java
import java.util.Optional;

public class OptionalMotivationDemo {
    static Optional<String> findUserEmail(String userId) {
        if (userId.equals("guest")) {
            return Optional.empty(); // ชัดเจนว่า "ไม่มีค่า" ผ่าน type system
        }
        return Optional.of(userId + "@example.com");
    }

    public static void main(String[] args) {
        Optional<String> email = findUserEmail("guest");

        // Signature ของ findUserEmail บอกชัดเจนแล้วว่า "อาจไม่มีค่า"
        // compiler ไม่บังคับให้เช็ค แต่ Optional ทำให้นักพัฒนา "คิดถึง" กรณีนี้มากขึ้นมาก
        // เพราะต้อง "แกะ" ค่าออกมาก่อนใช้งาน (ผ่านเมธอดต่าง ๆ ในหัวข้อถัดไป)

        System.out.println(email.isPresent()); // false
    }
}
```

## 3. การสร้าง Optional

```java
import java.util.Optional;

public class OptionalCreationDemo {
    public static void main(String[] args) {
        // Optional.of(): สร้างจากค่าที่ "ไม่ใช่ null แน่นอน" (ถ้าส่ง null เข้าไปจะ throw NPE ทันที)
        Optional<String> present = Optional.of("Hello");

        // Optional.empty(): สร้าง Optional ที่ไม่มีค่า
        Optional<String> empty = Optional.empty();

        // Optional.ofNullable(): สร้างจากค่าที่ "อาจเป็น null ก็ได้" (ปลอดภัยที่สุด)
        String maybeNull = null;
        Optional<String> safe = Optional.ofNullable(maybeNull); // ไม่ throw exception แม้ maybeNull เป็น null
        System.out.println(safe.isPresent()); // false

        String notNull = "value";
        Optional<String> safe2 = Optional.ofNullable(notNull);
        System.out.println(safe2.isPresent()); // true
    }
}
```

**หลักการเลือกใช้**: ใช้ `Optional.ofNullable()` เมื่อค่าที่ได้มา**อาจเป็น
null** (เช่น จากเมธอดของ library เก่าที่ยังคืน null), ใช้ `Optional.of()`
เมื่อ**มั่นใจ 100%** ว่าค่าไม่เป็น null (เพื่อให้ NPE เกิดขึ้นทันทีถ้าผิดคาด
แทนที่จะซ่อนปัญหาไว้)

## 4. การตรวจสอบและดึงค่าจาก Optional

```java
import java.util.Optional;

public class OptionalCheckDemo {
    public static void main(String[] args) {
        Optional<String> opt = Optional.of("Hello");

        // isPresent(): เช็คว่ามีค่าหรือไม่
        if (opt.isPresent()) {
            System.out.println(opt.get()); // get() ดึงค่าออกมา - throw NoSuchElementException ถ้าว่างเปล่า!
        }

        // isEmpty(): เช็คว่าไม่มีค่า (Java 11+, ตรงข้ามกับ isPresent())
        Optional<String> empty = Optional.empty();
        System.out.println(empty.isEmpty()); // true

        // ifPresent(): ทำงานกับค่าถ้ามีอยู่ (รับ Consumer - ทบทวนจาก Part 40) - แนะนำมากกว่า isPresent()+get()
        opt.ifPresent(value -> System.out.println("มีค่า: " + value));

        // ifPresentOrElse() (Java 9+): จัดการทั้งสองกรณีในคำสั่งเดียว - กระชับและปลอดภัยที่สุด
        opt.ifPresentOrElse(
            value -> System.out.println("มีค่า: " + value),
            () -> System.out.println("ไม่มีค่า")
        );
    }
}
```

**คำเตือนสำคัญ**: **`get()` โดยไม่เช็คก่อนเป็นแนวปฏิบัติที่ไม่ดี** เพราะทำให้
เกิด `NoSuchElementException` ได้เหมือนกับปัญหา NPE เดิม (เพียงแค่เปลี่ยนชนิด
exception) — ควรใช้ `ifPresent()`, `orElse()`, หรือ `map()` แทนเสมอ

## 5. `map()`, `filter()`, `flatMap()` บน Optional

Optional รองรับ functional operations แบบเดียวกับ Stream (Part 41-42) ทำให้
เขียนโค้ดที่**ไม่ต้องเช็ค null ซ้อนกันหลายชั้น**ได้อย่างกระชับ

```java
import java.util.Optional;

public class OptionalTransformDemo {
    record Address(String city) { }
    record User(String name, Optional<Address> address) { }

    public static void main(String[] args) {
        User user = new User("Alice", Optional.of(new Address("Bangkok")));
        User userNoAddress = new User("Bob", Optional.empty());

        // map(): แปลงค่าภายใน Optional (ถ้ามีค่าอยู่) - ไม่ต้องเช็ค null เอง
        Optional<String> city = user.address().map(Address::city);
        System.out.println(city.orElse("ไม่ระบุ")); // "Bangkok"

        Optional<String> cityNoAddress = userNoAddress.address().map(Address::city);
        System.out.println(cityNoAddress.orElse("ไม่ระบุ")); // "ไม่ระบุ" (ไม่เกิด NPE เลย!)

        // filter(): เก็บค่าไว้เฉพาะถ้าตรงเงื่อนไข ไม่งั้นกลายเป็น empty
        Optional<String> longCity = city.filter(c -> c.length() > 5);
        System.out.println(longCity.isPresent()); // true ("Bangkok" มี 7 ตัวอักษร)

        // เปรียบเทียบกับโค้ดแบบเช็ค null ซ้อนกันหลายชั้น (แบบเก่าที่อ่านยากและเสี่ยง bug)
        // if (user != null && user.address != null && user.address.city != null) { ... }
        // Optional ทำให้ chain การเข้าถึงข้อมูลที่อาจไม่มีอยู่ได้อย่างปลอดภัยและอ่านง่ายกว่ามาก
    }
}
```

## 6. `orElse()`, `orElseGet()`, `orElseThrow()` เปรียบเทียบ

```java
import java.util.Optional;
import java.util.NoSuchElementException;

public class OrElseComparisonDemo {
    static String expensiveDefault() {
        System.out.println("คำนวณค่า default ที่ใช้เวลานาน...");
        return "default value";
    }

    public static void main(String[] args) {
        Optional<String> present = Optional.of("actual value");
        Optional<String> empty = Optional.empty();

        // orElse(): ค่า default ถูกคำนวณ "เสมอ" แม้ Optional มีค่าอยู่แล้ว (eager evaluation)
        String result1 = present.orElse(expensiveDefault());
        // สังเกต: "คำนวณค่า default..." จะถูกพิมพ์ออกมา แม้ present มีค่าอยู่แล้วก็ตาม! (สิ้นเปลือง)

        // orElseGet(): ค่า default ถูกคำนวณ "เฉพาะเมื่อจำเป็น" (lazy evaluation ผ่าน Supplier - Part 40)
        String result2 = present.orElseGet(OrElseComparisonDemo::expensiveDefault);
        // ไม่มีการพิมพ์ "คำนวณค่า default..." เลย เพราะ present มีค่าอยู่แล้ว ไม่จำเป็นต้องคำนวณ

        // orElseThrow(): throw exception ถ้าไม่มีค่า (ปลอดภัยกว่า get() เพราะระบุ exception ที่ชัดเจนได้)
        try {
            String result3 = empty.orElseThrow(() -> new IllegalStateException("ไม่พบข้อมูล"));
        } catch (IllegalStateException e) {
            System.out.println("Error: " + e.getMessage());
        }

        // orElseThrow() แบบไม่มีพารามิเตอร์ (Java 10+): throw NoSuchElementException ค่า default
        try {
            String result4 = empty.orElseThrow();
        } catch (NoSuchElementException e) {
            System.out.println("Error: ไม่พบค่าใน Optional");
        }
    }
}
```

**กฎทอง**: **ใช้ `orElseGet()` เสมอเมื่อค่า default มี cost สูงในการคำนวณ**
(เรียก database, network call, หรือ logic ซับซ้อน) เพื่อหลีกเลี่ยงการคำนวณ
ที่ไม่จำเป็น — ใช้ `orElse()` ได้เมื่อค่า default เป็น literal ธรรมดา ๆ
(เช่น `orElse(0)`, `orElse("")`)

## 7. ข้อผิดพลาดที่พบบ่อยในการใช้ Optional

```java
import java.util.Optional;

public class OptionalPitfallsDemo {
    public static void main(String[] args) {
        Optional<String> opt = Optional.of("test");

        // ผิด #1: ใช้ isPresent() + get() แทนวิธีที่ปลอดภัยกว่า (ทำงานถูกแต่ไม่ใช่แนวปฏิบัติที่ดี)
        if (opt.isPresent()) {
            System.out.println(opt.get().toUpperCase());
        }
        // ควรใช้แบบนี้แทน:
        opt.map(String::toUpperCase).ifPresent(System.out::println);

        // ผิด #2: เรียก get() โดยไม่เช็คก่อนเลย
        // String value = someOptional.get(); // อันตราย! อาจ throw NoSuchElementException

        // ผิด #3: ใช้ Optional เป็น null เปล่า ๆ (Optional ที่เป็น null เอง - ย้อนกลับไปที่จุดเริ่มต้นของปัญหา!)
        Optional<String> nullOptional = null; // ไม่ควรมี Optional ที่เป็น null เด็ดขาด
        // ควรคืน Optional.empty() เสมอ ไม่ใช่ null

        // ผิด #4: ใช้ == เปรียบเทียบค่าภายใน Optional (ทบทวนปัญหาจาก Part 9)
        Optional<String> a = Optional.of("hello");
        Optional<String> b = Optional.of("hello");
        System.out.println(a.equals(b)); // true - ใช้ equals() ถูกต้อง (Optional override equals() ให้แล้ว)
    }
}
```

## 8. Optional กับ Field และ Method Parameter (ไม่แนะนำ)

**Optional ถูกออกแบบมาสำหรับ return type เท่านั้น** — Java documentation เอง
ระบุชัดเจนว่า **ไม่ควรใช้ Optional เป็น field หรือ method parameter**

```java
public class OptionalMisuseDemo {
    // ไม่แนะนำ: Optional เป็น field ทำให้ serialization ยุ่งยาก (Optional ไม่ implement Serializable)
    // private Optional<String> nickname; // ไม่ควรทำแบบนี้

    // แนะนำ: field เก็บเป็น null ตามปกติ แล้ว "เปิดเผย" ผ่าน getter ที่คืน Optional
    private String nickname; // เก็บภายในแบบเดิม (null ได้ตามปกติ)

    public java.util.Optional<String> getNickname() { // getter คืน Optional เพื่อสื่อสารกับผู้เรียกใช้
        return java.util.Optional.ofNullable(nickname);
    }

    // ไม่แนะนำ: Optional เป็น parameter ทำให้ผู้เรียกใช้ต้องห่อค่าด้วย Optional.of() เสมอ (ยุ่งยากเกินจำเป็น)
    // void processName(Optional<String> name) { }  // ไม่ควรทำแบบนี้

    // แนะนำ: ใช้ method overloading แทน (ทบทวนจาก Part 8) หรือรับค่าตรง ๆ แล้วเช็ค null เอง
    void processName(String name) {
        String actualName = (name != null) ? name : "Unknown";
        System.out.println(actualName);
    }
}
```

**สรุปหลักการ**: **Optional เหมาะกับ return type ของเมธอดเท่านั้น** เพื่อ
สื่อสารกับผู้เรียกใช้ว่า "ค่านี้อาจไม่มีอยู่จริง" — ไม่ควรใช้เป็น field
(ทำให้ serialization/JavaBeans ยุ่งยาก) หรือ parameter (เพิ่มความซับซ้อนโดย
ไม่จำเป็น เมื่อ overloading หรือ null check ตรง ๆ ทำได้ง่ายกว่า)

## 9. `Optional` Primitive Specializations

เช่นเดียวกับ `java.util.function` (Part 40) มี Optional เวอร์ชัน primitive
เพื่อหลีกเลี่ยง autoboxing (ทบทวนจาก Part 28):

```java
import java.util.OptionalInt;
import java.util.OptionalDouble;
import java.util.stream.IntStream;

public class OptionalPrimitiveDemo {
    public static void main(String[] args) {
        OptionalInt maxValue = IntStream.of(3, 7, 2, 9, 4).max(); // คืน OptionalInt ไม่ใช่ Optional<Integer>
        System.out.println(maxValue.orElse(-1)); // 9 (ไม่มี autoboxing เกิดขึ้น)

        OptionalDouble average = IntStream.of(1, 2, 3, 4, 5).average();
        System.out.println(average.orElse(0.0)); // 3.0
    }
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียนเมธอด `findEmployeeById(int id)` ที่คืนค่า `Optional<Employee>`
แล้วใช้ `map()` และ `orElse()` เพื่อดึงชื่อพนักงานหรือคืน "ไม่พบพนักงาน"
ถ้าไม่เจอ

**เฉลย:**

```java
import java.util.List;
import java.util.Optional;

public class Exercise1 {
    record Employee(int id, String name) { }

    static List<Employee> employees = List.of(
        new Employee(1, "Alice"),
        new Employee(2, "Bob")
    );

    static Optional<Employee> findEmployeeById(int id) {
        return employees.stream().filter(e -> e.id() == id).findFirst();
    }

    public static void main(String[] args) {
        String name = findEmployeeById(1).map(Employee::name).orElse("ไม่พบพนักงาน");
        System.out.println(name); // Alice

        String notFound = findEmployeeById(99).map(Employee::name).orElse("ไม่พบพนักงาน");
        System.out.println(notFound); // ไม่พบพนักงาน
    }
}
```

**2)** แก้โค้ดนี้ให้ปลอดภัยขึ้นโดยไม่ใช้ `get()` โดยตรง:

```java
Optional<String> opt = getSomeValue();
if (opt.isPresent()) {
    System.out.println(opt.get().length());
}
```

**เฉลย:**

```java
Optional<String> opt = getSomeValue();
opt.map(String::length).ifPresent(System.out::println);
```

**3)** อธิบายว่าทำไม `orElseGet()` ดีกว่า `orElse()` เมื่อค่า default ต้องเรียก
database หรือ API ภายนอก

**เฉลย**: `orElse()` รับค่าเป็น**ผลลัพธ์ที่คำนวณเสร็จแล้ว** (eager evaluation)
ดังนั้นแม้ Optional มีค่าอยู่แล้ว ค่า default ก็ยังถูก**คำนวณเสมอ** (เพราะ Java
ต้อง evaluate argument ก่อนส่งเข้าเมธอด) — ถ้าการคำนวณนั้นคือการเรียก database
หรือ API จะเสียเวลาและ resource โดยไม่จำเป็น ในขณะที่ `orElseGet()` รับ
`Supplier` (ทบทวนจาก Part 40) ที่**เรียกใช้ต่อเมื่อจำเป็นจริง ๆ เท่านั้น**
(lazy evaluation) ทำให้ไม่มีการเรียก database/API โดยไม่จำเป็นเมื่อ Optional
มีค่าอยู่แล้ว

### สรุปเนื้อหา Part 43

- `Optional<T>` ทำให้ "ค่านี้อาจไม่มีอยู่จริง" ปรากฏชัดเจนใน type system แทน
  การใช้ `null` เงียบ ๆ
- `Optional.ofNullable()` ปลอดภัยที่สุดสำหรับค่าที่อาจเป็น null, `Optional.of()`
  ใช้เมื่อมั่นใจว่าไม่ null
- `map()`, `filter()` ทำงานแบบ functional เหมือน Stream ช่วยลดการเช็ค null
  ซ้อนกันหลายชั้น
- `orElseGet()` ใช้ lazy evaluation ดีกว่า `orElse()` เมื่อค่า default มี cost
  สูง
- Optional ควรใช้เป็น **return type ของเมธอดเท่านั้น** ไม่ควรใช้เป็น field
  หรือ parameter

**ต่อไป**: [Part 44 — Date and Time API (java.time)](./part-044-date-time-api.md)
