# Part 12: Constructors และ Constructor Chaining

> ขั้นตอนที่ 111-120 ของหลักสูตร | ระดับ: OOP พื้นฐาน

## สารบัญ

1. Constructor คืออะไร
2. Default Constructor
3. Parameterized Constructor
4. Constructor Overloading
5. Constructor Chaining ด้วย `this(...)`
6. ลำดับการทำงานตอนสร้าง Object
7. `static` initialization block และ instance initialization block
8. Copy Constructor (แนวคิด)
9. Builder Pattern เบื้องต้น (แนะนำ)
10. แบบฝึกหัดและสรุป

---

## 1. Constructor คืออะไร

**Constructor** คือเมธอดพิเศษที่ถูกเรียกใช้**อัตโนมัติ**ทันทีที่สร้าง object ด้วย
`new` มีหน้าที่**กำหนดค่าเริ่มต้น**ให้กับ fields ของ object นั้น

**กฎของ Constructor**:
- ชื่อต้อง**ตรงกับชื่อคลาสเป๊ะ ๆ**
- **ไม่มี return type** (แม้แต่ `void` ก็ไม่มี — ถ้าเขียน `void` เข้าไปจะกลายเป็น
  method ธรรมดา ไม่ใช่ constructor)
- ถูกเรียกโดยอัตโนมัติเมื่อใช้ `new`

```java
public class Student {
    String name;
    int age;
    String major;

    // Constructor: ชื่อตรงกับ class, ไม่มี return type
    public Student(String name, int age, String major) {
        this.name = name;
        this.age = age;
        this.major = major;
        System.out.println("สร้างนักเรียนใหม่: " + name);
    }
}
```

```java
public class StudentDemo {
    public static void main(String[] args) {
        Student s1 = new Student("Alice", 20, "Computer Science"); // เรียก constructor อัตโนมัติ
        Student s2 = new Student("Bob", 22, "Mathematics");

        System.out.println(s1.name + " เรียน " + s1.major);
        System.out.println(s2.name + " เรียน " + s2.major);
    }
}
```

## 2. Default Constructor

ถ้า**ไม่เขียน constructor เอง** Java จะสร้าง **default constructor** (ไม่รับ
พารามิเตอร์ ไม่ทำอะไรเลยนอกจากกำหนดค่าเริ่มต้นตามชนิดข้อมูล) ให้อัตโนมัติ

```java
public class SimpleClass {
    int value;
    String text;
    // ไม่มีการเขียน constructor เอง -> Java แอบสร้าง SimpleClass() { } ให้อัตโนมัติ
}
```

```java
public class DefaultConstructorDemo {
    public static void main(String[] args) {
        SimpleClass obj = new SimpleClass(); // เรียก default constructor ที่ Java สร้างให้
        System.out.println(obj.value); // 0
        System.out.println(obj.text);  // null
    }
}
```

**ข้อควรระวังสำคัญมาก**: ทันทีที่คุณเขียน constructor เองแม้แค่ตัวเดียว
**Java จะไม่สร้าง default constructor ให้อีกต่อไป**:

```java
public class Product {
    String name;
    double price;

    public Product(String name, double price) {
        this.name = name;
        this.price = price;
    }
}
```

```java
public class MissingDefaultConstructorDemo {
    public static void main(String[] args) {
        Product p1 = new Product("Laptop", 25000); // OK

        // Product p2 = new Product(); // Error! ไม่มี constructor ที่ไม่รับพารามิเตอร์แล้ว
        //                              // เพราะเราเขียน constructor เองแล้ว Java จึงไม่สร้างให้
    }
}
```

หากต้องการทั้งสองแบบ (มีและไม่มีพารามิเตอร์) ต้องเขียนเองทั้งคู่ (ดูหัวข้อถัดไป)

## 3. Parameterized Constructor

Constructor ที่รับพารามิเตอร์เพื่อกำหนดค่าเริ่มต้นตามที่ต้องการ (ตัวอย่างในหัวข้อ 1
คือ parameterized constructor อยู่แล้ว) มักใช้ร่วมกับการ validate ข้อมูลตั้งแต่
ตอนสร้าง object:

```java
public class BankAccount {
    String accountNumber;
    double balance;

    public BankAccount(String accountNumber, double initialBalance) {
        if (initialBalance < 0) {
            throw new IllegalArgumentException("ยอดเงินเริ่มต้นห้ามติดลบ");
        }
        this.accountNumber = accountNumber;
        this.balance = initialBalance;
    }
}
```

```java
public class ParameterizedConstructorDemo {
    public static void main(String[] args) {
        BankAccount account = new BankAccount("ACC-001", 1000.0);
        System.out.println(account.accountNumber + ": " + account.balance);

        try {
            BankAccount invalid = new BankAccount("ACC-002", -500.0);
        } catch (IllegalArgumentException e) {
            System.out.println("สร้างบัญชีไม่สำเร็จ: " + e.getMessage());
        }
    }
}
```

## 4. Constructor Overloading

เช่นเดียวกับ method สามารถมี constructor**หลายตัว**ในคลาสเดียวกันได้ โดยมี
**พารามิเตอร์ต่างกัน** (Overloading — ทบทวนจาก Part 8)

```java
public class Rectangle {
    double width;
    double height;

    // Constructor 1: กำหนดทั้งกว้างและยาว
    public Rectangle(double width, double height) {
        this.width = width;
        this.height = height;
    }

    // Constructor 2: สี่เหลี่ยมจัตุรัส (กว้าง = ยาว)
    public Rectangle(double side) {
        this.width = side;
        this.height = side;
    }

    // Constructor 3: ไม่รับพารามิเตอร์ ใช้ค่า default
    public Rectangle() {
        this.width = 1.0;
        this.height = 1.0;
    }

    double area() {
        return width * height;
    }
}
```

```java
public class ConstructorOverloadingDemo {
    public static void main(String[] args) {
        Rectangle r1 = new Rectangle(5, 3);   // เรียก constructor 1
        Rectangle r2 = new Rectangle(4);       // เรียก constructor 2 (สี่เหลี่ยมจัตุรัส)
        Rectangle r3 = new Rectangle();         // เรียก constructor 3 (ค่า default)

        System.out.println("r1 พื้นที่: " + r1.area()); // 15.0
        System.out.println("r2 พื้นที่: " + r2.area()); // 16.0
        System.out.println("r3 พื้นที่: " + r3.area()); // 1.0
    }
}
```

## 5. Constructor Chaining ด้วย `this(...)`

**Constructor Chaining** คือการให้ constructor ตัวหนึ่ง**เรียก constructor อีกตัว**
ในคลาสเดียวกัน ผ่าน `this(...)` เพื่อลดโค้ดซ้ำซ้อน (ต้องเป็น**บรรทัดแรกสุด**ของ
constructor เท่านั้น)

```java
public class RectangleChained {
    double width;
    double height;
    String label;

    // Constructor หลัก (most complete) - ทำงานจริงทั้งหมด
    public RectangleChained(double width, double height, String label) {
        this.width = width;
        this.height = height;
        this.label = label;
        System.out.println("สร้างสี่เหลี่ยม: " + label);
    }

    // Chaining ไปหา constructor หลัก โดยใส่ label default
    public RectangleChained(double width, double height) {
        this(width, height, "ไม่มีชื่อ"); // ต้องเป็นบรรทัดแรกสุดเท่านั้น
    }

    // Chaining สำหรับสี่เหลี่ยมจัตุรัส
    public RectangleChained(double side) {
        this(side, side); // เรียก constructor ตัวที่ 2 ซึ่งจะไป chain ต่อไปตัวที่ 1 อีกที
    }
}
```

```java
public class ConstructorChainingDemo {
    public static void main(String[] args) {
        RectangleChained r1 = new RectangleChained(5, 3, "สี่เหลี่ยมผืนผ้า A");
        RectangleChained r2 = new RectangleChained(4, 6); // ได้ label = "ไม่มีชื่อ" อัตโนมัติ
        RectangleChained r3 = new RectangleChained(5);     // width=height=5, label="ไม่มีชื่อ"
    }
}
```

**ประโยชน์ของ Constructor Chaining**: ลด logic ซ้ำซ้อน มีจุดเดียวที่ทำงานหลัก
(single source of truth) ทำให้บำรุงรักษาง่ายขึ้นมาก — ถ้าต้องแก้ validation logic
แก้แค่ constructor หลักตัวเดียวพอ

## 6. ลำดับการทำงานตอนสร้าง Object

เมื่อเรียก `new SomeClass(...)` ลำดับการทำงานคือ:

1. จัดสรรหน่วยความจำบน heap และกำหนดค่าเริ่มต้นให้ทุก field ตามชนิดข้อมูล (default value)
2. รัน **instance initializer block** (ถ้ามี) และ **field initializer** ตามลำดับที่
   ปรากฏในโค้ด (จากบนลงล่าง)
3. รันโค้ดใน **constructor**
4. คืน reference ของ object ที่สร้างเสร็จแล้ว

```java
public class InitializationOrderDemo {
    int a = printAndReturn("field a", 1);

    {
        System.out.println("instance initializer block ทำงาน");
    }

    int b = printAndReturn("field b", 2);

    public InitializationOrderDemo() {
        System.out.println("constructor ทำงาน");
    }

    static int printAndReturn(String label, int value) {
        System.out.println("กำหนดค่า " + label + " = " + value);
        return value;
    }

    public static void main(String[] args) {
        new InitializationOrderDemo();
    }
}
```

ผลลัพธ์ (สังเกตลำดับ — field/block ทำงานก่อน constructor เสมอ):

```
กำหนดค่า field a = 1
instance initializer block ทำงาน
กำหนดค่า field b = 2
constructor ทำงาน
```

## 7. `static` Initialization Block และ Instance Initialization Block

- **Static Initialization Block**: ทำงาน**ครั้งเดียว**ตอนคลาสถูกโหลดเข้าสู่ JVM
  ครั้งแรก (ไม่ว่าจะสร้าง object กี่ตัวก็ตาม) ใช้กำหนดค่าเริ่มต้นให้ static field
  ที่มี logic ซับซ้อนกว่าการกำหนดค่าตรง ๆ
- **Instance Initialization Block**: ทำงานทุกครั้งที่สร้าง object ใหม่ (ก่อน
  constructor เสมอ) — ใช้น้อยกว่า static block มาก เพราะส่วนใหญ่ใส่ logic ใน
  constructor ได้อยู่แล้ว

```java
public class InitBlockDemo {
    static int totalCount;
    static String appName;

    // Static block: ทำงานครั้งเดียวตอนโหลดคลาส
    static {
        appName = "ระบบจัดการนักเรียน";
        totalCount = 0;
        System.out.println("โหลดคลาส InitBlockDemo ครั้งแรก - เตรียมค่า static");
    }

    int id;

    // Instance block: ทำงานทุกครั้งที่สร้าง object (ก่อน constructor)
    {
        totalCount++;
        id = totalCount;
    }

    public InitBlockDemo() {
        System.out.println("สร้าง object ลำดับที่ " + id);
    }

    public static void main(String[] args) {
        System.out.println(appName);
        new InitBlockDemo(); // สร้าง object ลำดับที่ 1
        new InitBlockDemo(); // สร้าง object ลำดับที่ 2
        System.out.println("จำนวน object ทั้งหมด: " + totalCount);
    }
}
```

## 8. Copy Constructor (แนวคิด)

Java ไม่มี copy constructor ในตัวเหมือน C++ แต่สามารถเขียนเองได้ เพื่อสร้าง object
ใหม่โดยคัดลอกค่าจาก object เดิม (คนละ reference กันโดยสิ้นเชิง):

```java
public class Point {
    double x;
    double y;

    public Point(double x, double y) {
        this.x = x;
        this.y = y;
    }

    // Copy constructor: รับ object ชนิดเดียวกันมาคัดลอกค่า
    public Point(Point other) {
        this.x = other.x;
        this.y = other.y;
    }
}
```

```java
public class CopyConstructorDemo {
    public static void main(String[] args) {
        Point original = new Point(3, 4);
        Point copy = new Point(original); // สร้าง object ใหม่ คัดลอกค่าจาก original

        copy.x = 999; // แก้ไข copy ไม่กระทบ original เลย เพราะเป็นคนละ object กัน

        System.out.println("original.x = " + original.x); // 3.0 (ไม่เปลี่ยน)
        System.out.println("copy.x = " + copy.x);           // 999.0
        System.out.println(original == copy); // false - คนละ object แน่นอน
    }
}
```

## 9. Builder Pattern เบื้องต้น (แนะนำ)

เมื่อ class มี field จำนวนมากและหลายตัวเป็น optional การมี constructor ที่รับ
พารามิเตอร์เยอะ ๆ ("telescoping constructor") จะอ่านยากและสับสนง่าย
**Builder Pattern** (จะลงลึกเต็มรูปแบบใน Part 54) ช่วยแก้ปัญหานี้:

```java
public class UserProfile {
    private final String username; // จำเป็น
    private final String email;    // จำเป็น
    private final int age;         // optional
    private final String bio;      // optional

    private UserProfile(Builder builder) {
        this.username = builder.username;
        this.email = builder.email;
        this.age = builder.age;
        this.bio = builder.bio;
    }

    static class Builder {
        private final String username;
        private final String email;
        private int age = 0;
        private String bio = "";

        Builder(String username, String email) { // ค่าที่จำเป็นต้องกำหนดผ่าน constructor
            this.username = username;
            this.email = email;
        }

        Builder age(int age) {
            this.age = age;
            return this; // คืน this เพื่อให้เขียนแบบ method chaining ได้
        }

        Builder bio(String bio) {
            this.bio = bio;
            return this;
        }

        UserProfile build() {
            return new UserProfile(this);
        }
    }

    void printProfile() {
        System.out.println(username + " (" + email + "), อายุ: " + age + ", bio: " + bio);
    }
}
```

```java
public class BuilderPatternPreviewDemo {
    public static void main(String[] args) {
        UserProfile user = new UserProfile.Builder("somchai", "somchai@email.com")
                .age(28)
                .bio("นักพัฒนาซอฟต์แวร์")
                .build(); // เขียนแบบ chaining อ่านง่าย ชัดเจนว่าแต่ละค่าคืออะไร

        user.printProfile();
    }
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียนคลาส `Book` ที่มี field: `title`, `author`, `price` พร้อม constructor
ที่ validate ว่า `price` ต้องไม่ติดลบ (throw `IllegalArgumentException` ถ้าผิด)

**เฉลย:**

```java
public class Book {
    String title;
    String author;
    double price;

    public Book(String title, String author, double price) {
        if (price < 0) {
            throw new IllegalArgumentException("ราคาหนังสือห้ามติดลบ");
        }
        this.title = title;
        this.author = author;
        this.price = price;
    }
}
```

**2)** เขียนคลาส `Circle` ที่มี constructor overload 2 แบบ: รับรัศมี (radius) หรือ
ไม่รับอะไรเลย (ใช้ radius = 1 เป็นค่า default) โดยใช้ constructor chaining

**เฉลย:**

```java
public class Circle {
    double radius;

    public Circle(double radius) {
        this.radius = radius;
    }

    public Circle() {
        this(1.0); // constructor chaining
    }

    double area() {
        return Math.PI * radius * radius;
    }
}
```

**3)** เขียนโปรแกรมทดสอบ copy constructor ของคลาส `Point` (จากตัวอย่างในหัวข้อ 8)
พิสูจน์ว่าแก้ไข copy ไม่กระทบต้นฉบับ

**เฉลย**: (ดูโค้ดในหัวข้อ 8 — `CopyConstructorDemo` คือคำตอบที่สมบูรณ์แล้ว)

### สรุปเนื้อหา Part 12

- Constructor ชื่อตรงกับ class เสมอ ไม่มี return type ถูกเรียกอัตโนมัติเมื่อใช้ `new`
- ถ้าไม่เขียน constructor เอง Java จะสร้าง default constructor ให้ แต่ถ้าเขียนเอง
  แม้ตัวเดียว default constructor จะหายไปทันที
- Constructor Overloading คือมีหลาย constructor พารามิเตอร์ต่างกัน
- Constructor Chaining ด้วย `this(...)` ช่วยลดโค้ดซ้ำซ้อน (ต้องเป็นบรรทัดแรกเสมอ)
- ลำดับตอนสร้าง object: field initializer/instance block → constructor
- Static block ทำงานครั้งเดียวตอนโหลดคลาส, Instance block ทำงานทุกครั้งที่สร้าง object
- Builder Pattern ช่วยแก้ปัญหา constructor ที่มีพารามิเตอร์เยอะเกินไป

**ต่อไป**: [Part 13 — Encapsulation และ Access Modifiers](./part-013-encapsulation.md)
