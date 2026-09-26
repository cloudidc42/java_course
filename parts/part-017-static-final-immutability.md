# Part 17: Static, Final และ Immutability

> ขั้นตอนที่ 161-170 ของหลักสูตร | ระดับ: OOP ขั้นกลาง

## สารบัญ

1. `static` Fields (Class Variables)
2. `static` Methods
3. `static` Nested Classes (เบื้องต้น)
4. `final` Variables
5. `final` Parameters
6. การสร้าง Immutable Class อย่างสมบูรณ์
7. Utility Class Pattern (all-static class)
8. Static Import
9. Memory Model: Method Area และ static
10. แบบฝึกหัดและสรุป

---

## 1. `static` Fields (Class Variables)

**Static field** (หรือ **class variable**) คือ field ที่**แชร์ร่วมกันระหว่างทุก
object ของ class นั้น** ไม่ได้แยกเป็นของแต่ละ object เหมือน instance field
มีอยู่เพียง**หนึ่งชุดค่าเดียว**ต่อคลาส ไม่ว่าจะสร้าง object กี่ตัวก็ตาม

```java
public class Counter {
    static int totalCounters = 0; // static field: แชร์ร่วมกันทุก object
    int id;                        // instance field: แยกกันในแต่ละ object

    public Counter() {
        totalCounters++; // เพิ่มค่าที่แชร์ร่วมกันทุกครั้งที่สร้าง object ใหม่
        id = totalCounters;
    }
}
```

```java
public class StaticFieldDemo {
    public static void main(String[] args) {
        Counter c1 = new Counter();
        Counter c2 = new Counter();
        Counter c3 = new Counter();

        System.out.println(c1.id); // 1 (instance field: แยกกันแต่ละ object)
        System.out.println(c2.id); // 2
        System.out.println(c3.id); // 3

        System.out.println(Counter.totalCounters); // 3 (static field: ค่าเดียวที่ทุก object แชร์ร่วมกัน)
        System.out.println(c1.totalCounters);       // 3 (เข้าถึงผ่าน instance ได้ แต่ไม่แนะนำ)
    }
}
```

```
Memory Model:

Method Area (ที่เก็บ static members - มีชุดเดียวสำหรับทั้งคลาส)
┌─────────────────────────┐
│ Counter.totalCounters=3 │
└─────────────────────────┘

Heap (แต่ละ object แยกกัน)
┌──────────┐  ┌──────────┐  ┌──────────┐
│ c1.id=1  │  │ c2.id=2  │  │ c3.id=3  │
└──────────┘  └──────────┘  └──────────┘
```

**แนวปฏิบัติที่ดี**: เข้าถึง static field ผ่าน**ชื่อคลาส** เสมอ (`Counter.totalCounters`)
ไม่ใช่ผ่าน instance (`c1.totalCounters`) เพื่อความชัดเจนว่าเป็น static

## 2. `static` Methods

ทบทวนจาก Part 8: **Static method เรียกผ่านชื่อคลาสได้เลยโดยไม่ต้องสร้าง object**
และ**เข้าถึงได้เฉพาะ static field/static method อื่น ๆ เท่านั้น** (เข้าถึง
instance field/instance method โดยตรงไม่ได้ เพราะไม่รู้ว่าจะอ้างอิงถึง object ไหน)

```java
public class MathUtils {
    static final double PI_APPROX = 3.14159;

    static double circleArea(double radius) { // static method: ไม่ต้องพึ่ง instance ใด ๆ
        return PI_APPROX * radius * radius;
    }

    static int square(int n) {
        return n * n;
    }

    int instanceField = 100;

    static void invalidExample() {
        // System.out.println(instanceField); // Error! static method เข้าถึง instance field ตรง ๆ ไม่ได้
        System.out.println(PI_APPROX);          // OK: static field เข้าถึงได้
    }
}
```

```java
public class StaticMethodDemo {
    public static void main(String[] args) {
        // เรียกผ่านชื่อคลาสตรง ๆ ไม่ต้องสร้าง object
        System.out.println(MathUtils.circleArea(5));
        System.out.println(MathUtils.square(7));
    }
}
```

**ตัวอย่างที่คุ้นเคยที่สุด**: `Math.sqrt()`, `Math.max()`, `Integer.parseInt()`,
`Arrays.sort()` ล้วนเป็น static method ทั้งสิ้น — ไม่ต้องสร้าง object `Math` หรือ
`Integer` ก่อนใช้งาน

## 3. `static` Nested Classes (เบื้องต้น)

รายละเอียดเต็มรูปแบบจะอยู่ใน Part 19 แต่ควรรู้พื้นฐานตั้งแต่ตอนนี้: **static nested
class** คือ class ที่ประกาศอยู่ภายใน class อื่น และมี `static` กำกับ — สร้าง object
ได้โดยไม่ต้องพึ่ง instance ของ class ภายนอก

```java
public class Outer {
    static class Node { // static nested class: มักใช้ทำโครงสร้างข้อมูล เช่น LinkedList (Part 32)
        int value;
        Node next;

        Node(int value) {
            this.value = value;
        }
    }
}
```

```java
public class StaticNestedClassDemo {
    public static void main(String[] args) {
        Outer.Node node = new Outer.Node(10); // สร้างได้โดยไม่ต้องมี object ของ Outer เลย
        System.out.println(node.value);
    }
}
```

## 4. `final` Variables

ทบทวนและขยายความจาก Part 3: `final` ทำให้ตัวแปร**กำหนดค่าได้เพียงครั้งเดียว**
หลังจากนั้นจะเปลี่ยนแปลงไม่ได้อีก ใช้ได้กับ local variable, field, parameter,
และ static field (constant)

```java
public class FinalVariableDemo {
    static final double TAX_RATE = 0.07; // static final = ค่าคงที่ระดับคลาส (constant)
    final String id;                     // final instance field: กำหนดได้ครั้งเดียว (ใน constructor)

    public FinalVariableDemo(String id) {
        this.id = id; // ต้องกำหนดค่าให้ final field ก่อนจบ constructor เสมอ (compiler บังคับ)
        // this.id = "changed"; // Error! กำหนดค่าซ้ำสองครั้งไม่ได้
    }

    public static void main(String[] args) {
        final int MAX_SIZE = 100; // local final variable
        // MAX_SIZE = 200; // Error! แก้ไขค่าไม่ได้อีก

        FinalVariableDemo obj = new FinalVariableDemo("ID-001");
        System.out.println(obj.id);
        System.out.println(TAX_RATE);
    }
}
```

**ข้อสำคัญ**: `final` กับ reference type หมายถึง**ตัวแปรห้ามชี้ไปยัง object อื่น**
แต่**เนื้อหาภายใน object ที่ชี้อยู่ยังแก้ไขได้** (ถ้า object นั้น mutable):

```java
import java.util.ArrayList;
import java.util.List;

public class FinalReferenceDemo {
    public static void main(String[] args) {
        final List<String> names = new ArrayList<>();

        names.add("Alice"); // OK! แก้ไขเนื้อหาภายใน object ได้ (ArrayList ยังคง mutable)
        names.add("Bob");
        System.out.println(names);

        // names = new ArrayList<>(); // Error! ห้ามให้ names ชี้ไปยัง object ใหม่
    }
}
```

## 5. `final` Parameters

Parameter ของ method ก็ `final` ได้ ป้องกันไม่ให้เผลอ reassign ค่าพารามิเตอร์
ภายในเมธอด (มักใช้เพื่อความชัดเจนของ intent มากกว่าจำเป็นทางเทคนิค):

```java
public class FinalParameterDemo {
    static int calculateDiscount(final double price, final double discountRate) {
        // price = price - 100; // Error! ห้าม reassign parameter ที่เป็น final
        return (int) (price * (1 - discountRate));
    }

    public static void main(String[] args) {
        System.out.println(calculateDiscount(1000, 0.1)); // 900
    }
}
```

## 6. การสร้าง Immutable Class อย่างสมบูรณ์

**Immutable Class** คือ class ที่**สถานะไม่สามารถเปลี่ยนแปลงได้เลยหลังสร้าง object**
(ตัวอย่างที่คุ้นเคยคือ `String`) มีข้อดีมากมาย: thread-safe โดยธรรมชาติ, ปลอดภัย
จากการแก้ไขโดยไม่ตั้งใจ, ใช้เป็น key ของ HashMap ได้อย่างปลอดภัย

**หลัก 5 ข้อในการสร้าง Immutable Class**:

1. ประกาศ class เป็น `final` (ป้องกัน subclass มาเปลี่ยนพฤติกรรม)
2. Field ทุกตัวเป็น `private final`
3. ไม่มี setter method ใด ๆ ทั้งสิ้น
4. ถ้า field เป็น mutable object (เช่น array, List, Date) ต้อง**คัดลอกป้องกัน
   (defensive copy)** ทั้งตอนรับเข้า constructor และตอนคืนค่าออกจาก getter
5. Class ไม่ควรอนุญาตให้ subclass override method ที่กระทบ invariant

```java
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public final class ImmutableStudent { // (1) final class
    private final String name;             // (2) private final field
    private final int age;
    private final List<String> subjects;   // mutable object -> ต้อง defensive copy

    public ImmutableStudent(String name, int age, List<String> subjects) {
        this.name = name;
        this.age = age;
        // (4) defensive copy ตอนรับเข้า - ป้องกันผู้เรียกแก้ไข list ต้นฉบับแล้วกระทบเรา
        this.subjects = new ArrayList<>(subjects);
    }

    public String getName() { return name; }   // (3) ไม่มี setName()
    public int getAge() { return age; }         // (3) ไม่มี setAge()

    public List<String> getSubjects() {
        // (4) defensive copy ตอนคืนค่าออก - ป้องกันผู้เรียกแก้ไข list ภายในของเรา
        return Collections.unmodifiableList(subjects);
        // หรือ return new ArrayList<>(subjects); ก็ได้เช่นกัน
    }

    // ถ้าต้องการ "เปลี่ยน" ค่า ให้คืน object ใหม่แทน (immutable pattern - "wither" method)
    public ImmutableStudent withAge(int newAge) {
        return new ImmutableStudent(this.name, newAge, this.subjects);
    }
}
```

```java
import java.util.List;

public class ImmutableClassDemo {
    public static void main(String[] args) {
        List<String> originalSubjects = new java.util.ArrayList<>(List.of("Math", "Science"));
        ImmutableStudent student = new ImmutableStudent("Alice", 20, originalSubjects);

        originalSubjects.add("History"); // แก้ไข list ต้นฉบับหลังสร้าง object แล้ว
        System.out.println(student.getSubjects()); // [Math, Science] - ไม่ได้รับผลกระทบเลย!
                                                     // เพราะ constructor ทำ defensive copy ไว้แล้ว

        try {
            student.getSubjects().add("Art"); // พยายามแก้ไข list ที่ได้จาก getter
        } catch (UnsupportedOperationException e) {
            System.out.println("แก้ไข list ที่ได้จาก getter ไม่ได้ เพราะเป็น unmodifiable");
        }

        ImmutableStudent olderStudent = student.withAge(21); // ได้ object ใหม่แทนการแก้ไขของเดิม
        System.out.println(student.getAge());       // 20 (ต้นฉบับไม่เปลี่ยน)
        System.out.println(olderStudent.getAge());   // 21 (object ใหม่)
    }
}
```

## 7. Utility Class Pattern (All-static Class)

**Utility Class** คือ class ที่มีแต่ static method ล้วน ๆ (เช่น `Math`,
`Collections`, `Arrays`) ไม่มีเจตนาให้สร้าง object เลย — ตามธรรมเนียมที่ดี
ควรป้องกันไม่ให้ใครสร้าง object ได้โดยใช้ **private constructor**:

```java
public final class StringUtils { // final: ไม่ควรให้ subclass สืบทอด (ไม่มีเหตุผลที่จะทำ)
    private StringUtils() { // private constructor: ป้องกันการสร้าง object จากภายนอก
        throw new AssertionError("ห้ามสร้าง object ของ utility class นี้");
    }

    public static boolean isNullOrEmpty(String s) {
        return s == null || s.isEmpty();
    }

    public static String capitalize(String s) {
        if (isNullOrEmpty(s)) return s;
        return Character.toUpperCase(s.charAt(0)) + s.substring(1);
    }

    public static String reverse(String s) {
        return new StringBuilder(s).reverse().toString();
    }
}
```

```java
public class UtilityClassDemo {
    public static void main(String[] args) {
        System.out.println(StringUtils.isNullOrEmpty(""));      // true
        System.out.println(StringUtils.capitalize("hello"));    // "Hello"
        System.out.println(StringUtils.reverse("hello"));       // "olleh"

        // StringUtils utils = new StringUtils(); // Error! constructor เป็น private
    }
}
```

## 8. Static Import

**Static Import** ช่วยให้เรียกใช้ static member ได้โดยไม่ต้องพิมพ์ชื่อคลาสนำหน้า
(ทบทวนจาก Part 2) ใช้อย่างระมัดระวัง เพราะอาจทำให้โค้ดอ่านยากขึ้นถ้าใช้เยอะเกินไป:

```java
import static java.lang.Math.PI;
import static java.lang.Math.pow;
import static java.lang.Math.sqrt;

public class StaticImportDemo {
    public static void main(String[] args) {
        // ไม่ต้องพิมพ์ Math. นำหน้า
        double area = PI * pow(5, 2);
        double hypotenuse = sqrt(pow(3, 2) + pow(4, 2));

        System.out.println("พื้นที่วงกลม: " + area);
        System.out.println("ด้านตรงข้ามมุมฉาก: " + hypotenuse);
    }
}
```

**คำแนะนำ**: ใช้ static import เฉพาะกับ method/constant ที่ใช้บ่อยมากและชัดเจน
ในบริบท (เช่น `Math.*` หรือใน unit test เช่น `assertEquals` จาก JUnit) หลีกเลี่ยง
การใช้แบบ wildcard (`import static SomeClass.*`) ในโค้ดขนาดใหญ่ เพราะทำให้ตามหา
ที่มาของ method ยาก

## 9. Memory Model: Method Area และ `static`

Static members ถูกเก็บใน**Method Area** (บางเอกสารเรียก Metaspace ใน Java 8+)
ซึ่งเป็นพื้นที่หน่วยความจำที่**แชร์ร่วมกันทุก object** ของคลาสนั้น และ**ถูกโหลด
เพียงครั้งเดียว**ตอนคลาสถูกโหลดเข้าสู่ JVM ครั้งแรก (จะลงลึกเรื่อง JVM memory
model ทั้งหมดใน Part 67)

```
JVM Memory Structure (ภาพรวมคร่าว ๆ):

┌─────────────────────────────────────┐
│         Method Area / Metaspace       │  <- static fields, method bytecode, class metadata
├─────────────────────────────────────┤
│              Heap                     │  <- object instances ทั้งหมด (new)
├─────────────────────────────────────┤
│         Stack (แยกต่อ thread)          │  <- local variables, method call frames
└─────────────────────────────────────┘
```

**ผลกระทบเชิงปฏิบัติ**: static field มีอายุยืนตลอดโปรแกรม (จนกว่าคลาสจะถูก
unload) จึงต้องระวังการเก็บข้อมูลจำนวนมากไว้ใน static field เพราะอาจทำให้เกิด
**memory leak** ได้ง่าย (ข้อมูลไม่ถูกเก็บกวาดโดย garbage collector เพราะยังมี
reference จาก static field อยู่เสมอ)

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** สร้าง class `IdGenerator` ที่มี static field เก็บลำดับ ID ล่าสุด และ static
method `generateNextId()` ที่คืนค่า ID ถัดไปทุกครั้งที่เรียก (เริ่มจาก 1001)

**เฉลย:**

```java
public class IdGenerator {
    private static int currentId = 1000;

    public static int generateNextId() {
        currentId++;
        return currentId;
    }
}
```

```java
public class Exercise1 {
    public static void main(String[] args) {
        System.out.println(IdGenerator.generateNextId()); // 1001
        System.out.println(IdGenerator.generateNextId()); // 1002
        System.out.println(IdGenerator.generateNextId()); // 1003
    }
}
```

**2)** สร้าง Immutable class `Money` ที่มี field `amount` (double) และ `currency`
(String) พร้อมเมธอด `add(Money other)` ที่คืนค่า `Money` object ใหม่ (ไม่แก้ไขของเดิม)

**เฉลย:**

```java
public final class Money {
    private final double amount;
    private final String currency;

    public Money(double amount, String currency) {
        this.amount = amount;
        this.currency = currency;
    }

    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("สกุลเงินต้องเหมือนกัน");
        }
        return new Money(this.amount + other.amount, this.currency);
    }

    public double getAmount() { return amount; }
    public String getCurrency() { return currency; }
}
```

**3)** สร้าง utility class `MathHelper` ที่มี static method `isPrime(int n)` และ
`gcd(int a, int b)` (หา ห.ร.ม.) พร้อม private constructor ป้องกันการสร้าง object

**เฉลย:**

```java
public final class MathHelper {
    private MathHelper() { }

    public static boolean isPrime(int n) {
        if (n < 2) return false;
        for (int i = 2; i * i <= n; i++) {
            if (n % i == 0) return false;
        }
        return true;
    }

    public static int gcd(int a, int b) {
        return (b == 0) ? a : gcd(b, a % b);
    }
}
```

### สรุปเนื้อหา Part 17

- Static field แชร์ร่วมกันทุก object ของ class, static method เรียกผ่านชื่อคลาส
  ได้เลยโดยไม่ต้องสร้าง object
- `final` variable กำหนดค่าได้ครั้งเดียว, สำหรับ reference type คือห้ามชี้ไป object
  อื่น (แต่เนื้อหาภายในยังแก้ไขได้ถ้า object เดิม mutable)
- Immutable class: `final` class + `private final` fields + ไม่มี setter +
  defensive copy สำหรับ mutable fields
- Utility class ใช้ `private constructor` ป้องกันการสร้าง object
- Static members ถูกเก็บใน Method Area/Metaspace แชร์ร่วมกันทุก object และมีอายุ
  ยืนตลอดโปรแกรม ต้องระวัง memory leak

**ต่อไป**: [Part 18 — Enums แบบละเอียด](./part-018-enums.md)
