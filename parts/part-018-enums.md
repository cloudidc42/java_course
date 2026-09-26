# Part 18: Enums แบบละเอียด

> ขั้นตอนที่ 171-180 ของหลักสูตร | ระดับ: OOP ขั้นกลาง

## สารบัญ

1. Enum คืออะไร ทำไมดีกว่า Constant ธรรมดา
2. การประกาศและใช้งาน Enum พื้นฐาน
3. Enum เป็น Class พิเศษ: มี Field, Constructor, Method ได้
4. Enum ใน `switch` Statement
5. เมธอดสำเร็จรูปของ Enum: `values()`, `ordinal()`, `name()`, `valueOf()`
6. Abstract Method ใน Enum (แต่ละค่ามีพฤติกรรมต่างกัน)
7. Enum Implement Interface
8. `EnumMap` และ `EnumSet` เบื้องต้น
9. เมื่อไรควรใช้ Enum
10. แบบฝึกหัดและสรุป

---

## 1. Enum คืออะไร ทำไมดีกว่า Constant ธรรมดา

**Enum (Enumeration)** คือชนิดข้อมูลพิเศษที่ใช้แทน**กลุ่มของค่าคงที่ที่จำกัด
(fixed set of constants)** เช่น วันในสัปดาห์, ทิศทาง, สถานะออร์เดอร์

ก่อนมี enum นักพัฒนามักใช้ `int` constant แทน ซึ่งมีปัญหามาก:

```java
public class OldStyleConstants {
    static final int STATUS_PENDING = 0;
    static final int STATUS_APPROVED = 1;
    static final int STATUS_REJECTED = 2;

    static void processOrder(int status) {
        if (status == STATUS_APPROVED) {
            System.out.println("ดำเนินการต่อ");
        }
    }

    public static void main(String[] args) {
        processOrder(1);       // ใช้งานได้ แต่ไม่มีใครรู้ว่า 1 คืออะไรถ้าไม่เปิดโค้ดดู
        processOrder(999);     // Error! compiler ไม่เตือนเลยว่า 999 ไม่ใช่ status ที่ถูกต้อง
    }
}
```

ปัญหา: **ไม่มี type safety** (ส่งเลขอะไรก็ได้ไม่มีการตรวจสอบ), **อ่านโค้ดยาก**
(ตัวเลขไม่สื่อความหมาย), **ไม่มี namespace** (ชื่อ constant อาจชนกันได้)

```java
public enum OrderStatus {
    PENDING, APPROVED, REJECTED // ตามธรรมเนียม: ตัวพิมพ์ใหญ่ทั้งหมด คั่นด้วย comma
}
```

```java
public class EnumBasicDemo {
    static void processOrder(OrderStatus status) {
        if (status == OrderStatus.APPROVED) {
            System.out.println("ดำเนินการต่อ");
        }
    }

    public static void main(String[] args) {
        processOrder(OrderStatus.APPROVED); // ชัดเจน อ่านง่าย
        // processOrder(999); // Error! compiler ปฏิเสธทันที เพราะ 999 ไม่ใช่ OrderStatus
    }
}
```

## 2. การประกาศและใช้งาน Enum พื้นฐาน

```java
public enum Day {
    MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY
}
```

```java
public class DayDemo {
    public static void main(String[] args) {
        Day today = Day.WEDNESDAY;

        System.out.println(today);            // WEDNESDAY (toString() ให้ชื่อ enum โดยอัตโนมัติ)
        System.out.println(today == Day.WEDNESDAY); // true - เปรียบเทียบด้วย == ได้อย่างปลอดภัย!
                                                       // เพราะแต่ละค่า enum มี instance เดียวเท่านั้นในทั้งโปรแกรม

        if (today == Day.SATURDAY || today == Day.SUNDAY) {
            System.out.println("วันหยุดสุดสัปดาห์");
        } else {
            System.out.println("วันทำงาน");
        }
    }
}
```

**ข้อสำคัญ**: enum ใช้ `==` เปรียบเทียบได้อย่างปลอดภัย (ต่างจาก String) เพราะ
Java รับประกันว่าแต่ละค่า enum จะมี**เพียง instance เดียวเท่านั้น**ตลอดทั้งโปรแกรม
(คล้ายกับ Singleton Pattern ที่จะเรียนใน Part 54)

## 3. Enum เป็น Class พิเศษ: มี Field, Constructor, Method ได้

Enum ใน Java **ไม่ใช่แค่ตัวเลขคงที่** แต่เป็น**class เต็มรูปแบบ** ที่สืบทอดจาก
`java.lang.Enum` โดยอัตโนมัติ สามารถมี field, constructor, และ method ได้เหมือน
class ทั่วไป (แต่**สร้าง object เพิ่มเองไม่ได้** ค่าถูกกำหนดตายตัวตอน compile)

```java
public enum Planet {
    MERCURY(3.303e+23, 2.4397e6),
    VENUS(4.869e+24, 6.0518e6),
    EARTH(5.976e+24, 6.37814e6),
    MARS(6.421e+23, 3.3972e6);

    private final double mass;   // field ของแต่ละค่า enum (private ตามหลัก encapsulation)
    private final double radius;

    // Constructor ของ enum เป็น private โดยปริยายเสมอ (ห้ามใช้ public/protected)
    Planet(double mass, double radius) {
        this.mass = mass;
        this.radius = radius;
    }

    double surfaceGravity() {
        final double G = 6.67300E-11;
        return G * mass / (radius * radius);
    }

    double surfaceWeight(double otherMass) {
        return otherMass * surfaceGravity();
    }
}
```

```java
public class PlanetDemo {
    public static void main(String[] args) {
        double earthWeight = 70; // กิโลกรัม
        double mass = earthWeight / Planet.EARTH.surfaceGravity();

        for (Planet planet : Planet.values()) { // values() คืนค่า array ของทุกค่าใน enum
            System.out.printf("น้ำหนักบนดาว %s: %.2f กก.%n",
                              planet, planet.surfaceWeight(mass));
        }
    }
}
```

## 4. Enum ใน `switch` Statement

Enum ทำงานได้ดีมากกับ `switch` (ทั้งแบบดั้งเดิมและ switch expression จาก Part 5)
โดย**ไม่ต้องระบุชื่อ enum นำหน้าใน case** (compiler รู้ชนิดอยู่แล้ว):

```java
public enum TrafficLight { RED, YELLOW, GREEN }

public class TrafficLightDemo {
    static String getAction(TrafficLight light) {
        // ใช้ switch expression (Java 14+) - ไม่ต้องเขียน TrafficLight.RED
        return switch (light) {
            case RED -> "หยุด";
            case YELLOW -> "เตรียมตัว";
            case GREEN -> "ไปได้";
        };
        // สังเกต: ไม่มี default! เพราะ compiler รู้ว่า enum นี้มีแค่ 3 ค่า
        // และ case ครอบคลุมครบทุกค่าแล้ว (exhaustive) - ถ้าเพิ่มค่าใหม่ใน enum
        // ในอนาคต compiler จะเตือนทันทีว่า switch นี้ไม่ครบถ้วนแล้ว
    }

    public static void main(String[] args) {
        System.out.println(getAction(TrafficLight.RED));    // หยุด
        System.out.println(getAction(TrafficLight.GREEN));  // ไปได้
    }
}
```

## 5. เมธอดสำเร็จรูปของ Enum

```java
public class EnumMethodsDemo {
    public static void main(String[] args) {
        // values(): คืนค่า array ของทุกค่าในลำดับที่ประกาศไว้
        for (Day day : Day.values()) {
            System.out.print(day + " ");
        }
        System.out.println();

        Day today = Day.WEDNESDAY;

        // name(): คืนชื่อของค่านั้นเป็น String (ตรงกับที่ประกาศไว้เป๊ะ)
        System.out.println(today.name()); // "WEDNESDAY"

        // ordinal(): คืนตำแหน่งลำดับ (เริ่มที่ 0) ตามที่ประกาศไว้ใน enum
        System.out.println(today.ordinal()); // 2 (MONDAY=0, TUESDAY=1, WEDNESDAY=2)

        // valueOf(): แปลง String กลับเป็นค่า enum (throw IllegalArgumentException ถ้าไม่ตรง)
        Day parsed = Day.valueOf("FRIDAY");
        System.out.println(parsed); // FRIDAY

        try {
            Day invalid = Day.valueOf("NOTADAY");
        } catch (IllegalArgumentException e) {
            System.out.println("ไม่พบค่า enum นี้: " + e.getMessage());
        }

        // compareTo(): เปรียบเทียบตาม ordinal (ลำดับที่ประกาศ)
        System.out.println(Day.MONDAY.compareTo(Day.FRIDAY)); // ค่าติดลบ (MONDAY มาก่อน)
    }
}
```

**ข้อควรระวังเรื่อง `ordinal()`**: ไม่ควรใช้ `ordinal()` เป็น business logic หรือ
บันทึกลงฐานข้อมูล เพราะถ้ามีการ**เพิ่มค่าใหม่แทรกกลาง enum** ในอนาคต ตำแหน่ง
ordinal ของค่าที่มีอยู่เดิมจะเปลี่ยนไปทั้งหมด ทำให้ข้อมูลเก่าผิดเพี้ยน — ถ้าต้องการ
เก็บ persistent value ควรใช้ field ที่กำหนดเองแทน (ดูตัวอย่างหัวข้อ 6)

## 6. Abstract Method ใน Enum (แต่ละค่ามีพฤติกรรมต่างกัน)

จุดที่ทรงพลังมากของ enum ใน Java คือสามารถให้**แต่ละค่ามี implementation ของตัวเอง**
ผ่าน abstract method — เป็นการรวม Polymorphism (Part 15) เข้ากับ enum:

```java
public enum Operation {
    ADD {
        @Override
        public double apply(double a, double b) { return a + b; }
    },
    SUBTRACT {
        @Override
        public double apply(double a, double b) { return a - b; }
    },
    MULTIPLY {
        @Override
        public double apply(double a, double b) { return a * b; }
    },
    DIVIDE {
        @Override
        public double apply(double a, double b) {
            if (b == 0) throw new ArithmeticException("หารด้วยศูนย์ไม่ได้");
            return a / b;
        }
    };

    public abstract double apply(double a, double b); // แต่ละค่า enum ต้อง implement เอง
}
```

```java
public class EnumAbstractMethodDemo {
    public static void main(String[] args) {
        double x = 10, y = 3;
        for (Operation op : Operation.values()) {
            System.out.printf("%s: %.2f%n", op, op.apply(x, y));
        }
        // ADD: 13.00
        // SUBTRACT: 7.00
        // MULTIPLY: 30.00
        // DIVIDE: 3.33
    }
}
```

### ตัวอย่างการใช้ field แทน `ordinal()` เพื่อความปลอดภัย

```java
public enum HttpStatus {
    OK(200), CREATED(201), BAD_REQUEST(400), NOT_FOUND(404), SERVER_ERROR(500);

    private final int code; // เก็บ business value เอง แทนพึ่งพา ordinal() ที่ไม่เสถียร

    HttpStatus(int code) {
        this.code = code;
    }

    public int getCode() {
        return code;
    }

    public static HttpStatus fromCode(int code) { // แปลงกลับจากตัวเลขเป็น enum
        for (HttpStatus status : values()) {
            if (status.code == code) return status;
        }
        throw new IllegalArgumentException("ไม่พบ HTTP status code: " + code);
    }
}
```

```java
public class HttpStatusDemo {
    public static void main(String[] args) {
        System.out.println(HttpStatus.NOT_FOUND.getCode()); // 404
        System.out.println(HttpStatus.fromCode(200));         // OK
    }
}
```

## 7. Enum Implement Interface

Enum สามารถ `implements` interface ได้ (แต่ **extends class ไม่ได้** เพราะ enum
สืบทอดจาก `java.lang.Enum` ไปแล้ว และ Java รองรับ single inheritance เท่านั้น):

```java
public interface Describable {
    String describe();
}

public enum Season implements Describable {
    SPRING, SUMMER, FALL, WINTER;

    @Override
    public String describe() {
        return switch (this) {
            case SPRING -> "ฤดูใบไม้ผลิ อากาศสดชื่น";
            case SUMMER -> "ฤดูร้อน อากาศร้อนจัด";
            case FALL -> "ฤดูใบไม้ร่วง อากาศเย็นสบาย";
            case WINTER -> "ฤดูหนาว อากาศหนาวเย็น";
        };
    }
}
```

```java
public class EnumInterfaceDemo {
    public static void main(String[] args) {
        for (Season season : Season.values()) {
            System.out.println(season + ": " + season.describe());
        }
    }
}
```

## 8. `EnumMap` และ `EnumSet` เบื้องต้น

Java มีโครงสร้างข้อมูลพิเศษที่ **optimize สำหรับ enum โดยเฉพาะ** ทำงานเร็วกว่า
HashMap/HashSet ธรรมดามาก (ใช้ bit vector ภายใน) — จะลงลึกเรื่อง Map/Set ใน
Part 23-24:

```java
import java.util.EnumMap;
import java.util.EnumSet;
import java.util.Map;
import java.util.Set;

public class EnumMapSetDemo {
    public static void main(String[] args) {
        // EnumMap: key เป็น enum เท่านั้น เรียงลำดับตาม ordinal อัตโนมัติ
        Map<Day, String> schedule = new EnumMap<>(Day.class);
        schedule.put(Day.MONDAY, "ประชุมทีม");
        schedule.put(Day.FRIDAY, "ส่งงาน");

        for (Map.Entry<Day, String> entry : schedule.entrySet()) {
            System.out.println(entry.getKey() + ": " + entry.getValue());
        }

        // EnumSet: เก็บชุดของค่า enum อย่างมีประสิทธิภาพสูง
        Set<Day> weekend = EnumSet.of(Day.SATURDAY, Day.SUNDAY);
        Set<Day> weekdays = EnumSet.complementOf((EnumSet<Day>) weekend); // ค่าที่เหลือทั้งหมด

        System.out.println("วันหยุด: " + weekend);
        System.out.println("วันทำงาน: " + weekdays);
    }
}
```

## 9. เมื่อไรควรใช้ Enum

ใช้ enum เมื่อ:
- มี**ชุดค่าที่จำกัดและรู้ล่วงหน้าตั้งแต่ตอน compile** (เช่น สถานะ, ทิศทาง, ระดับ,
  ประเภทการชำระเงิน)
- ต้องการ **type safety** ป้องกันการส่งค่าที่ไม่ถูกต้อง
- แต่ละค่าอาจมี**พฤติกรรมหรือข้อมูลเฉพาะตัว**ที่แตกต่างกัน

**ไม่ควรใช้ enum** เมื่อชุดค่าอาจ**เปลี่ยนแปลงบ่อยหรือมาจากภายนอก** (เช่น รายชื่อ
ประเทศที่อาจอัปเดตจากฐานข้อมูล ควรใช้ class ธรรมดาหรือดึงจาก database แทน)

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** สร้าง enum `Direction` (NORTH, SOUTH, EAST, WEST) พร้อมเมธอด `opposite()`
ที่คืนทิศทางตรงข้าม โดยใช้ switch expression

**เฉลย:**

```java
public enum Direction {
    NORTH, SOUTH, EAST, WEST;

    public Direction opposite() {
        return switch (this) {
            case NORTH -> SOUTH;
            case SOUTH -> NORTH;
            case EAST -> WEST;
            case WEST -> EAST;
        };
    }
}
```

**2)** สร้าง enum `MembershipLevel` (BRONZE, SILVER, GOLD) ที่มี field
`discountPercent` และเมธอด `calculateFinalPrice(double price)`

**เฉลย:**

```java
public enum MembershipLevel {
    BRONZE(5), SILVER(10), GOLD(20);

    private final int discountPercent;

    MembershipLevel(int discountPercent) {
        this.discountPercent = discountPercent;
    }

    public double calculateFinalPrice(double price) {
        return price * (1 - discountPercent / 100.0);
    }
}
```

**3)** สร้าง enum `Command` ที่ implement interface `Executable` (มี method
`execute()`) โดยแต่ละ command มี implementation ต่างกันผ่าน abstract method

**เฉลย:**

```java
public interface Executable {
    void execute();
}

public enum Command implements Executable {
    START {
        @Override
        public void execute() { System.out.println("เริ่มการทำงาน"); }
    },
    STOP {
        @Override
        public void execute() { System.out.println("หยุดการทำงาน"); }
    },
    RESTART {
        @Override
        public void execute() { System.out.println("รีสตาร์ทระบบ"); }
    };
}
```

### สรุปเนื้อหา Part 18

- Enum แทนกลุ่มค่าคงที่แบบ type-safe ดีกว่า int constant มาก เปรียบเทียบด้วย `==`
  ได้อย่างปลอดภัย
- Enum เป็น class เต็มรูปแบบ มี field, constructor (private เสมอ), method ได้
- Enum ทำงานร่วมกับ switch expression ได้ดีมาก และ compiler ช่วยเช็ค exhaustiveness
- แต่ละค่าใน enum สามารถมี implementation ของ abstract method เป็นของตัวเองได้
  (polymorphism)
- อย่าใช้ `ordinal()` เป็น business logic หรือบันทึกถาวร เพราะไม่เสถียรเมื่อ enum
  เปลี่ยนแปลง ควรใช้ field ที่กำหนดเองแทน
- Enum implement interface ได้ แต่ extends class อื่นไม่ได้

**ต่อไป**: [Part 19 — Nested Classes, Inner Classes, Anonymous Classes](./part-019-nested-classes.md)
