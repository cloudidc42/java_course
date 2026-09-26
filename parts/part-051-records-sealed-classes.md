# Part 51: Records, Sealed Classes, Pattern Matching (Java 17-21)

> ขั้นตอนที่ 501-510 ของหลักสูตร | ระดับ: สูง

## สารบัญ

1. Record คืออะไร แก้ปัญหาอะไร
2. Record Syntax และคุณสมบัติที่ได้มาโดยอัตโนมัติ
3. Custom Constructor และ Validation ใน Record
4. Record กับ Method เพิ่มเติมและ Static Members
5. Record Implement Interface
6. Sealed Classes และ Sealed Interfaces
7. Pattern Matching for `switch` (Java 21)
8. Record Patterns (Java 21): Deconstruction
9. Sealed + Record + Pattern Matching: การผสมผสานที่ทรงพลัง
10. แบบฝึกหัดและสรุป

---

## 1. Record คืออะไร แก้ปัญหาอะไร

ทบทวนจาก Part 11-13: การสร้าง **immutable data class** (class ที่เก็บข้อมูล
ล้วน ๆ ไม่มี behavior ซับซ้อน) ต้องเขียนโค้ดซ้ำซ้อนมาก — constructor, getter,
`equals()`, `hashCode()`, `toString()`

```java
// แบบเดิม (ก่อน Java 16): ต้องเขียนโค้ดซ้ำซ้อนมากสำหรับ data class ธรรมดา
public final class PointOld {
    private final int x;
    private final int y;

    public PointOld(int x, int y) {
        this.x = x;
        this.y = y;
    }

    public int getX() { return x; }
    public int getY() { return y; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof PointOld)) return false;
        PointOld p = (PointOld) o;
        return x == p.x && y == p.y;
    }

    @Override
    public int hashCode() {
        return java.util.Objects.hash(x, y);
    }

    @Override
    public String toString() {
        return "PointOld[x=" + x + ", y=" + y + "]";
    }
}
```

**`record`** (Java 16+) ให้ compiler**สร้างทุกอย่างข้างบนให้อัตโนมัติ**จาก
การประกาศเพียงบรรทัดเดียว:

```java
public record Point(int x, int y) { }
// ได้มาโดยอัตโนมัติทั้งหมด: constructor, x(), y() (getter), equals(), hashCode(), toString()
```

```java
public class RecordMotivationDemo {
    public static void main(String[] args) {
        Point p1 = new Point(3, 4);
        Point p2 = new Point(3, 4);

        System.out.println(p1);              // Point[x=3, y=4] (toString() อัตโนมัติ)
        System.out.println(p1.x());            // 3 (getter ชื่อ x() ไม่ใช่ getX())
        System.out.println(p1.equals(p2));      // true (equals() เปรียบเทียบทุก field อัตโนมัติ)
        System.out.println(p1.hashCode() == p2.hashCode()); // true
    }
}
```

## 2. Record Syntax และคุณสมบัติที่ได้มาโดยอัตโนมัติ

```java
public record Employee(String name, double salary, String department) { }
```

Record นี้ได้รับโดยอัตโนมัติ:
1. **Canonical constructor**: `new Employee("Alice", 35000, "IT")`
2. **Accessor methods**: `emp.name()`, `emp.salary()`, `emp.department()`
   (ไม่ใช่ `getName()` — ทบทวน JavaBeans convention จาก Part 13 ที่ record
   **ไม่ตาม** ธรรมเนียมนี้โดยเจตนา)
3. **`equals()`/`hashCode()`**: เปรียบเทียบทุก field (ทบทวนจาก Part 23, 27)
4. **`toString()`**: format `RecordName[field1=value1, field2=value2, ...]`
5. **Field เป็น `private final` โดยอัตโนมัติ** (record เป็น immutable โดย
   design — ทบทวนแนวคิดจาก Part 17)

```java
import java.util.Objects;

public class RecordAutoFeaturesDemo {
    public static void main(String[] args) {
        Employee emp = new Employee("Alice", 35000, "IT");

        System.out.println(emp.name());          // Alice
        System.out.println(emp.salary());          // 35000.0
        System.out.println(emp);                    // Employee[name=Alice, salary=35000.0, department=IT]

        // ทดสอบว่าเป็น immutable จริง - ไม่มี setter ให้เรียกเลย
        // emp.setSalary(40000); // Error! ไม่มีเมธอดนี้อยู่ - record ไม่มี setter

        // ทดสอบว่า field เป็น private จริง
        // System.out.println(emp.name); // Error! เข้าถึง field ตรง ๆ ไม่ได้ ต้องผ่าน accessor method
    }
}

record Employee(String name, double salary, String department) { }
```

## 3. Custom Constructor และ Validation ใน Record

Record รองรับการเพิ่ม validation logic ผ่าน **compact constructor** (ไม่ต้อง
เขียนพารามิเตอร์ซ้ำ):

```java
public record BankAccount(String accountNumber, double balance) {
    // Compact constructor: ไม่ต้องเขียน (String accountNumber, double balance) ซ้ำ
    // ไม่ต้องเขียน this.accountNumber = accountNumber; เอง (compiler ทำให้อัตโนมัติหลัง block นี้)
    public BankAccount {
        if (balance < 0) {
            throw new IllegalArgumentException("ยอดเงินห้ามติดลบ"); // ทบทวน validation จาก Part 12-13
        }
        accountNumber = accountNumber.trim(); // ปรับค่าก่อนกำหนดให้ field ได้ (normalize)
    }
}
```

```java
public class CompactConstructorDemo {
    public static void main(String[] args) {
        BankAccount account = new BankAccount("  ACC-001  ", 1000);
        System.out.println(account.accountNumber()); // "ACC-001" (ถูก trim แล้ว)

        try {
            BankAccount invalid = new BankAccount("ACC-002", -500);
        } catch (IllegalArgumentException e) {
            System.out.println("สร้างไม่สำเร็จ: " + e.getMessage());
        }
    }
}
```

สามารถเขียน**canonical constructor แบบเต็ม**ได้เช่นกัน (มีพารามิเตอร์ครบและ
ต้อง assign เองทุก field) แต่**compact constructor นิยมกว่ามาก**เพราะสั้นกว่า

## 4. Record กับ Method เพิ่มเติมและ Static Members

Record เพิ่ม method, static field/method ได้ตามปกติ (แต่**ห้ามเพิ่ม instance
field ใหม่** ที่ไม่ได้อยู่ใน record header):

```java
public record Rectangle(double width, double height) {
    // เพิ่ม instance method ได้ตามปกติ (คำนวณจาก field ที่มีอยู่)
    public double area() {
        return width * height;
    }

    public double perimeter() {
        return 2 * (width + height);
    }

    // เพิ่ม static factory method (ธรรมเนียมที่นิยมใน record)
    public static Rectangle square(double side) {
        return new Rectangle(side, side);
    }

    // static field ก็ใส่ได้
    public static final Rectangle UNIT = new Rectangle(1, 1);
}
```

```java
public class RecordMethodsDemo {
    public static void main(String[] args) {
        Rectangle rect = new Rectangle(5, 3);
        System.out.println(rect.area());       // 15.0
        System.out.println(rect.perimeter());   // 16.0

        Rectangle sq = Rectangle.square(4);
        System.out.println(sq.area());           // 16.0
        System.out.println(Rectangle.UNIT);        // Rectangle[width=1.0, height=1.0]
    }
}
```

## 5. Record Implement Interface

Record `implements` interface ได้ตามปกติ (แต่**`extends` class ไม่ได้** เพราะ
record สืบทอดจาก `java.lang.Record` โดยปริยายแล้ว — เหมือนข้อจำกัดของ enum
จาก Part 18):

```java
public interface Shape {
    double area();
}

public record Circle(double radius) implements Shape {
    @Override
    public double area() {
        return Math.PI * radius * radius;
    }
}

public record Square(double side) implements Shape {
    @Override
    public double area() {
        return side * side;
    }
}
```

```java
import java.util.List;

public class RecordInterfaceDemo {
    public static void main(String[] args) {
        List<Shape> shapes = List.of(new Circle(5), new Square(4));
        for (Shape shape : shapes) {
            System.out.println(shape + " มีพื้นที่ " + shape.area());
            // ใช้ polymorphism ได้ตามปกติ (ทบทวนจาก Part 15)
        }
    }
}
```

## 6. Sealed Classes และ Sealed Interfaces

**`sealed`** (Java 17+) จำกัดว่า**class/interface ใดบ้างที่สามารถ extends/
implements ได้** — ควบคุมลำดับชั้นการสืบทอดให้ชัดเจนและปิด (closed) แทนที่จะ
เปิดกว้างไม่จำกัดแบบเดิม (ทบทวนแนวคิด inheritance จาก Part 14, abstract class
จาก Part 16)

```java
public sealed interface Shape permits Circle, Square, Triangle { // ระบุ subtype ที่อนุญาตชัดเจน
    double area();
}

public record Circle(double radius) implements Shape {
    @Override public double area() { return Math.PI * radius * radius; }
}

public record Square(double side) implements Shape {
    @Override public double area() { return side * side; }
}

public record Triangle(double base, double height) implements Shape {
    @Override public double area() { return base * height / 2; }
}

// public class Hexagon implements Shape { } // Error! ไม่ได้อยู่ใน permits list - compile ไม่ผ่าน
```

**ทำไม sealed classes สำคัญ**: ทำให้ compiler **รู้ทุก subtype ที่เป็นไปได้**
ของ `Shape` — สิ่งนี้เปิดทางให้ `switch` expression (Part 5, 18) **ตรวจสอบ
ความครบถ้วน (exhaustiveness) ได้อย่างสมบูรณ์** โดยไม่ต้องมี `default` case
(หัวข้อ 7)

**กฎของ sealed class**: subclass ที่อนุญาตต้องเป็นหนึ่งใน 3 แบบ:
- `final` (ห้ามสืบทอดต่ออีก — ทบทวนจาก Part 14)
- `sealed` (จำกัด subtype ต่อไปอีกชั้น)
- `non-sealed` (เปิดให้สืบทอดต่อได้อย่างอิสระอีกครั้ง)

## 7. Pattern Matching for `switch` (Java 21)

ทบทวนพื้นฐานจาก Part 5, 15, 18 — Java 21 ทำให้ `switch` ทำงานกับ **type
pattern** ได้เต็มรูปแบบ และเมื่อใช้กับ **sealed hierarchy** compiler จะ
**บังคับให้ครอบคลุมทุก subtype** (exhaustiveness checking):

```java
public class PatternMatchingSwitchDemo {
    static String describe(Shape shape) {
        return switch (shape) {
            case Circle c -> "วงกลมรัศมี " + c.radius();
            case Square s -> "สี่เหลี่ยมจัตุรัสด้าน " + s.side();
            case Triangle t -> "สามเหลี่ยมฐาน " + t.base();
            // ไม่ต้องมี default! เพราะ Shape เป็น sealed และครอบคลุมครบ 3 subtype แล้ว
            // ถ้าเพิ่ม subtype ใหม่ใน permits list ในอนาคต compiler จะเตือนทันทีว่า switch นี้ไม่ครบ
        };
    }

    public static void main(String[] args) {
        System.out.println(describe(new Circle(5)));
        System.out.println(describe(new Square(4)));
        System.out.println(describe(new Triangle(6, 3)));
    }
}
```

## 8. Record Patterns (Java 21): Deconstruction

**Record Pattern** ให้ **"แยกส่วน" (deconstruct)** record ออกเป็น field
ต่าง ๆ ได้ทันทีใน pattern matching (ทบทวน pattern matching for `instanceof`
จาก Part 15):

```java
public class RecordPatternDemo {
    record Point(int x, int y) { }
    record Line(Point start, Point end) { } // record ซ้อน record ได้

    static String describePoint(Object obj) {
        // แบบเดิม: ต้อง cast แล้วเรียก accessor เอง
        if (obj instanceof Point p) {
            return "จุดที่ (" + p.x() + ", " + p.y() + ")";
        }
        return "ไม่ทราบ";
    }

    static String describePointModern(Object obj) {
        // Record Pattern: แยกส่วน x, y ออกมาเป็นตัวแปรได้ทันที ไม่ต้องเรียก accessor เอง
        if (obj instanceof Point(int x, int y)) {
            return "จุดที่ (" + x + ", " + y + ")";
        }
        return "ไม่ทราบ";
    }

    static String describeLine(Line line) {
        // Nested record pattern: แยกส่วนซ้อนกันได้หลายชั้น
        if (line instanceof Line(Point(int x1, int y1), Point(int x2, int y2))) {
            return "เส้นจาก (" + x1 + "," + y1 + ") ถึง (" + x2 + "," + y2 + ")";
        }
        return "ไม่ทราบ";
    }

    public static void main(String[] args) {
        System.out.println(describePointModern(new Point(3, 4)));
        System.out.println(describeLine(new Line(new Point(0, 0), new Point(5, 5))));
    }
}
```

ใช้ร่วมกับ `switch` ได้อย่างทรงพลังมาก:

```java
public class RecordPatternSwitchDemo {
    record Point(int x, int y) { }

    static String classify(Point p) {
        return switch (p) {
            case Point(var x, var y) when x == 0 && y == 0 -> "จุดกำเนิด"; // guard condition (ทบทวนจาก Part 5)
            case Point(var x, var y) when x == 0 -> "อยู่บนแกน Y";
            case Point(var x, var y) when y == 0 -> "อยู่บนแกน X";
            case Point(var x, var y) -> "จุดทั่วไปที่ (" + x + ", " + y + ")";
        };
    }

    public static void main(String[] args) {
        System.out.println(classify(new Point(0, 0)));  // จุดกำเนิด
        System.out.println(classify(new Point(0, 5)));   // อยู่บนแกน Y
        System.out.println(classify(new Point(3, 4)));    // จุดทั่วไปที่ (3, 4)
    }
}
```

## 9. Sealed + Record + Pattern Matching: การผสมผสานที่ทรงพลัง

การรวม 3 feature นี้เข้าด้วยกันทำให้ Java เขียนโค้ดแบบ **Algebraic Data
Types** (แนวคิดจากภาษา functional เช่น Haskell, Scala) ได้อย่างปลอดภัยและ
กระชับ — เหมาะมากกับการจำลอง**ผลลัพธ์ที่เป็นไปได้หลายแบบ**อย่างชัดเจน:

```java
public sealed interface ApiResult<T> permits ApiResult.Success, ApiResult.Failure {
    record Success<T>(T data) implements ApiResult<T> { }
    record Failure<T>(String errorMessage, int statusCode) implements ApiResult<T> { }
}
```

```java
public class SealedResultDemo {
    static void handleResult(ApiResult<String> result) {
        String message = switch (result) {
            case ApiResult.Success<String>(var data) -> "สำเร็จ: " + data;
            case ApiResult.Failure<String>(var msg, var code) -> "ผิดพลาด [" + code + "]: " + msg;
            // ครอบคลุมครบทุกกรณีแล้ว - compiler การันตี ไม่มีทางลืม case ไหนไปได้เลย
        };
        System.out.println(message);
    }

    public static void main(String[] args) {
        handleResult(new ApiResult.Success<>("ข้อมูลผู้ใช้"));
        handleResult(new ApiResult.Failure<>("ไม่พบข้อมูล", 404));
    }
}
```

**ประโยชน์ที่สำคัญที่สุด**: ถ้าในอนาคตมีคนเพิ่ม subtype ที่ 3 (เช่น
`Pending`) เข้าไปใน `permits` list, **compiler จะแจ้ง error ทันทีที่จุดที่มี
switch expression ทุกจุดที่ไม่ครอบคลุม `Pending`** — ป้องกัน bug จากการลืม
จัดการกรณีใหม่ได้อย่างสมบูรณ์ตอน compile-time (ไม่ต้องรอไปเจอ bug ตอน runtime)

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** สร้าง record `Money(double amount, String currency)` ที่ validate ว่า
amount ต้องไม่ติดลบ และมี method `add(Money other)` (เทียบกับ Part 17
แบบเดิมที่ใช้ class ธรรมดา)

**เฉลย:**

```java
public record Money(double amount, String currency) {
    public Money {
        if (amount < 0) throw new IllegalArgumentException("จำนวนเงินห้ามติดลบ");
    }

    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("สกุลเงินต้องเหมือนกัน");
        }
        return new Money(this.amount + other.amount, this.currency);
    }
}
```

**2)** สร้าง sealed interface `TrafficLight` ที่มี record `Red`, `Yellow`,
`Green` แล้วใช้ switch expression บอกว่าควรทำอะไร

**เฉลย:**

```java
public sealed interface TrafficLight permits TrafficLight.Red, TrafficLight.Yellow, TrafficLight.Green {
    record Red() implements TrafficLight { }
    record Yellow() implements TrafficLight { }
    record Green() implements TrafficLight { }

    static String action(TrafficLight light) {
        return switch (light) {
            case Red r -> "หยุด";
            case Yellow y -> "เตรียมตัว";
            case Green g -> "ไปได้";
        };
    }
}
```

**3)** ใช้ Record Pattern แยกส่วน `record Employee(String name, Money salary)`
ใน switch เพื่อจัดกลุ่มตามช่วงเงินเดือน

**เฉลย:**

```java
public class Exercise3 {
    record Money(double amount, String currency) { }
    record Employee(String name, Money salary) { }

    static String categorize(Employee emp) {
        return switch (emp) {
            case Employee(var name, Money(var amount, var currency)) when amount >= 50000 ->
                name + " (เงินเดือนสูง)";
            case Employee(var name, Money(var amount, var currency)) ->
                name + " (เงินเดือนทั่วไป)";
        };
    }
}
```

### สรุปเนื้อหา Part 51

- `record` สร้าง immutable data class พร้อม constructor, accessor, equals,
  hashCode, toString อัตโนมัติจากบรรทัดเดียว
- Compact constructor ใส่ validation logic ได้โดยไม่ต้องเขียนพารามิเตอร์ซ้ำ
- `sealed` จำกัด subtype ที่อนุญาตให้สืบทอด ทำให้ compiler ตรวจสอบ
  exhaustiveness ได้
- Pattern matching for `switch` (Java 21) ทำงานร่วมกับ sealed hierarchy
  ครอบคลุมทุกกรณีโดยไม่ต้องมี `default`
- Record Pattern แยกส่วน record ออกเป็น field ได้ทันทีใน pattern matching
  (รวมถึง nested record)
- Sealed + Record + Pattern Matching รวมกันทำให้เขียน Algebraic Data Types
  ได้ปลอดภัยและกระชับ

**ต่อไป**: [Part 52 — Annotations: การใช้งานและการสร้าง Custom Annotation](./part-052-annotations.md)
