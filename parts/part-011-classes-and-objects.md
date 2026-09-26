# Part 11: Classes และ Objects เบื้องต้น

> ขั้นตอนที่ 101-110 ของหลักสูตร | ระดับ: OOP พื้นฐาน

## สารบัญ

1. Object-Oriented Programming (OOP) คืออะไร
2. Class กับ Object ต่างกันอย่างไร
3. การสร้าง Class และ Fields
4. การสร้าง Object ด้วย `new`
5. Instance Methods
6. `this` keyword
7. Object ใน Memory: Heap และ Reference
8. `null` และการตรวจสอบ
9. คุณสมบัติ 4 เสาหลักของ OOP (ภาพรวม)
10. แบบฝึกหัดและสรุป

---

## 1. Object-Oriented Programming (OOP) คืออะไร

**OOP** คือแนวทางการเขียนโปรแกรมที่จำลอง**สิ่งต่าง ๆ ในโลกจริง**ให้เป็น "object"
ในโปรแกรม โดยแต่ละ object มี:

- **สถานะ (State)**: ข้อมูลที่ object นั้นเก็บไว้ (เช่น รถยนต์มีสี, ความเร็ว, น้ำมัน)
- **พฤติกรรม (Behavior)**: สิ่งที่ object นั้นทำได้ (เช่น รถยนต์ขับได้, เบรกได้, บีบแตรได้)

Java เป็นภาษา OOP **แบบเข้มข้น (pure-ish)** — เกือบทุกอย่างในโปรแกรม Java ต้องอยู่
ภายใน class (ยกเว้น primitive types) ต่างจากภาษาแบบ procedural (เช่น C) ที่เขียน
โปรแกรมเป็นลำดับคำสั่งตรง ๆ โดยไม่มีแนวคิดเรื่อง object

## 2. Class กับ Object ต่างกันอย่างไร

**Class คือ "พิมพ์เขียว" หรือ "แม่แบบ"** ที่นิยามว่า object ชนิดนี้จะมีข้อมูล
(fields) และพฤติกรรม (methods) อะไรบ้าง ส่วน**Object คือ "สิ่งที่สร้างขึ้นจริง"**
จากพิมพ์เขียวนั้น (เรียกว่า **instance**)

เปรียบเทียบ: `Class` คือ**พิมพ์เขียวบ้าน** ส่วน `Object` คือ**บ้านจริงที่สร้างขึ้น**
จากพิมพ์เขียวนั้น — พิมพ์เขียวเดียวสร้างบ้านได้หลายหลัง แต่ละหลังมีที่อยู่ (ตำแหน่ง
ในหน่วยความจำ) และสถานะของตัวเอง (สีทาบ้าน, ของตกแต่ง) แตกต่างกันได้

```java
// Class: พิมพ์เขียวของ "Dog"
public class Dog {
    // Fields: ข้อมูล/สถานะของ Dog แต่ละตัว
    String name;
    String breed;
    int age;

    // Methods: พฤติกรรมที่ Dog ทำได้
    void bark() {
        System.out.println(name + " เห่า: โฮ่ง โฮ่ง!");
    }

    void sleep() {
        System.out.println(name + " กำลังนอนหลับ");
    }
}
```

```java
public class DogDemo {
    public static void main(String[] args) {
        // สร้าง Object (instance) จาก Class Dog ด้วยคำสั่ง new
        Dog dog1 = new Dog();
        dog1.name = "โปโป้";
        dog1.breed = "ชิวาวา";
        dog1.age = 3;

        Dog dog2 = new Dog();
        dog2.name = "มะลิ";
        dog2.breed = "โกลเด้น รีทรีฟเวอร์";
        dog2.age = 5;

        // แต่ละ Object มีสถานะเป็นของตัวเอง แม้จะมาจาก Class เดียวกัน
        dog1.bark(); // โปโป้ เห่า: โฮ่ง โฮ่ง!
        dog2.bark(); // มะลิ เห่า: โฮ่ง โฮ่ง!

        System.out.println(dog1.name + " อายุ " + dog1.age + " ปี พันธุ์ " + dog1.breed);
        System.out.println(dog2.name + " อายุ " + dog2.age + " ปี พันธุ์ " + dog2.breed);
    }
}
```

## 3. การสร้าง Class และ Fields

**Fields** (หรือเรียกว่า **instance variables**) คือตัวแปรที่ประกาศไว้ระดับคลาส
เก็บสถานะของแต่ละ object โดยแต่ละ object จะมีชุด fields เป็นของตัวเอง (แยกจากกัน
โดยสิ้นเชิงในหน่วยความจำ)

```java
public class BankAccount {
    // Fields (instance variables)
    String accountNumber;
    String ownerName;
    double balance;

    // ธรรมเนียมปฏิบัติที่ดี (จะเรียนเรื่อง encapsulation ใน Part 13):
    // ในโค้ดจริงควรใส่ private หน้า field เสมอ แต่ Part นี้ยังไม่ใส่เพื่อให้เห็นภาพง่ายก่อน
}
```

```java
public class BankAccountDemo {
    public static void main(String[] args) {
        BankAccount account1 = new BankAccount();
        account1.accountNumber = "1001-2002";
        account1.ownerName = "สมชาย ใจดี";
        account1.balance = 5000.0;

        BankAccount account2 = new BankAccount();
        account2.accountNumber = "1001-2003";
        account2.ownerName = "สมหญิง รักเรียน";
        account2.balance = 15000.0;

        System.out.println(account1.ownerName + " มีเงิน " + account1.balance + " บาท");
        System.out.println(account2.ownerName + " มีเงิน " + account2.balance + " บาท");
    }
}
```

## 4. การสร้าง Object ด้วย `new`

`new` keyword ทำหน้าที่ 3 อย่าง:

1. **จัดสรรหน่วยความจำ** บน heap สำหรับ object ใหม่
2. **เรียก constructor** เพื่อกำหนดค่าเริ่มต้น (Part 12 จะลงลึก)
3. **คืนค่า reference (ที่อยู่)** ของ object นั้นให้ตัวแปรที่รับไว้

```java
public class NewKeywordDemo {
    public static void main(String[] args) {
        Dog myDog = new Dog();
        //  ↑ตัวแปร reference   ↑สร้าง object ใหม่บน heap แล้วคืน reference มาเก็บใน myDog

        // ถ้าไม่กำหนดค่า field เอง field จะได้ค่าเริ่มต้นตามชนิดข้อมูล (Part 3)
        System.out.println(myDog.name); // null (เพราะ String ยังไม่ได้กำหนดค่า)
        System.out.println(myDog.age);  // 0   (เพราะ int ค่าเริ่มต้นคือ 0)
    }
}
```

## 5. Instance Methods

**Instance Method** คือเมธอดที่ต้องเรียกผ่าน object (instance) เท่านั้น ไม่สามารถ
เรียกผ่านชื่อคลาสตรง ๆ ได้ (ต่างจาก static method ที่เรียนใน Part 8) เพราะ instance
method มักต้องใช้ข้อมูลจาก fields ของ object นั้น ๆ

```java
public class Rectangle {
    double width;
    double height;

    // Instance method: คำนวณจาก field ของ object นี้โดยเฉพาะ
    double calculateArea() {
        return width * height;
    }

    double calculatePerimeter() {
        return 2 * (width + height);
    }

    void printInfo() {
        System.out.println("กว้าง: " + width + ", ยาว: " + height);
        System.out.println("พื้นที่: " + calculateArea()); // เรียก method อื่นในคลาสเดียวกันได้เลย
        System.out.println("เส้นรอบรูป: " + calculatePerimeter());
    }
}
```

```java
public class RectangleDemo {
    public static void main(String[] args) {
        Rectangle rect1 = new Rectangle();
        rect1.width = 5;
        rect1.height = 3;

        Rectangle rect2 = new Rectangle();
        rect2.width = 10;
        rect2.height = 4;

        rect1.printInfo(); // ใช้ width/height ของ rect1
        System.out.println("---");
        rect2.printInfo(); // ใช้ width/height ของ rect2 (คนละชุดข้อมูลกัน)
    }
}
```

## 6. `this` keyword

`this` คือ reference ที่**ชี้ไปยัง object ปัจจุบัน** (object ที่กำลังเรียกเมธอดนี้อยู่)
ใช้บ่อยที่สุดเมื่อชื่อพารามิเตอร์ซ้ำกับชื่อ field ทำให้ต้องแยกให้ชัดเจนว่าหมายถึงตัวไหน

```java
public class Person {
    String name;
    int age;

    // ไม่มีปัญหาถ้าชื่อพารามิเตอร์ต่างจาก field
    void setName(String newName) {
        name = newName; // ชัดเจนอยู่แล้วว่าหมายถึง field name
    }

    // ถ้าชื่อพารามิเตอร์ซ้ำกับ field ต้องใช้ this เพื่อแยกความกำกวม
    void setAge(int age) {
        this.age = age; // this.age = field, age (ไม่มี this) = พารามิเตอร์
        // age = age;   // ผิด! นี่คือ parameter ตั้งค่าให้ตัวเอง ไม่ได้ตั้งให้ field เลย
    }

    void introduce() {
        System.out.println("สวัสดี ฉันชื่อ " + this.name + " อายุ " + this.age + " ปี");
        // this. ในกรณีนี้ใส่หรือไม่ใส่ก็ได้ผลลัพธ์เหมือนกัน (ไม่มีความกำกวม)
        // แต่บางคนใส่เพื่อความชัดเจนว่ากำลังอ้างถึง field ของ object นี้
    }
}
```

```java
public class ThisDemo {
    public static void main(String[] args) {
        Person p = new Person();
        p.setName("Alice");
        p.setAge(30);
        p.introduce(); // สวัสดี ฉันชื่อ Alice อายุ 30 ปี
    }
}
```

## 7. Object ใน Memory: Heap และ Reference

ตัวแปร object ใน Java **ไม่ได้เก็บ object โดยตรง** แต่เก็บ **reference (ที่อยู่)**
ที่ชี้ไปยัง object ซึ่งถูกจัดสรรอยู่บน **heap memory**

```java
public class ReferenceMemoryDemo {
    public static void main(String[] args) {
        Dog dog1 = new Dog();
        dog1.name = "โปโป้";

        Dog dog2 = dog1; // dog2 ไม่ได้สร้าง object ใหม่! แค่คัดลอก reference เดียวกัน

        dog2.name = "มะลิ"; // แก้ไขผ่าน dog2 แต่กระทบ object เดียวกับที่ dog1 ชี้อยู่

        System.out.println(dog1.name); // "มะลิ" (!) เพราะ dog1 กับ dog2 ชี้ไป object เดียวกัน
        System.out.println(dog1 == dog2); // true - reference เดียวกันจริง ๆ

        Dog dog3 = new Dog(); // สร้าง object ใหม่จริง ๆ ด้วย new อีกครั้ง
        dog3.name = "มะลิ";
        System.out.println(dog1 == dog3);        // false - คนละ object แม้ชื่อจะเหมือนกัน
    }
}
```

```
Stack (ตัวแปร local)          Heap (Object จริง)
┌────────────┐
│ dog1 ────────────────────>  ┌──────────────┐
│            │                │ name: "มะลิ" │
│ dog2 ────────────────────>  │ (Object A)   │  <- dog1 และ dog2 ชี้ไป Object เดียวกัน
│            │                └──────────────┘
│ dog3 ────────────────────>  ┌──────────────┐
│            │                │ name: "มะลิ" │  <- Object B คนละตัวกับ Object A
└────────────┘                └──────────────┘
```

## 8. `null` และการตรวจสอบ

`null` คือค่าพิเศษที่หมายถึง "ไม่ได้ชี้ไปยัง object ใดเลย" — ตัวแปร reference ที่ยัง
ไม่ได้กำหนดค่าจะเป็น `null` โดยอัตโนมัติถ้าเป็น field (แต่ local variable ต้องกำหนด
เองเสมอ ตามที่เรียนใน Part 3)

```java
public class NullCheckDemo {
    public static void main(String[] args) {
        Dog myDog = null; // ยังไม่ชี้ไปยัง object ใดเลย

        try {
            myDog.bark(); // NullPointerException! เรียก method บน null
        } catch (NullPointerException e) {
            System.out.println("myDog ยังไม่ถูกสร้าง object จริง");
        }

        // วิธีป้องกันที่ถูกต้อง: ตรวจสอบก่อนใช้งานเสมอ
        if (myDog != null) {
            myDog.bark();
        } else {
            System.out.println("myDog เป็น null ข้ามการเรียกใช้");
        }

        myDog = new Dog(); // ตอนนี้ชี้ไปยัง object จริงแล้ว
        myDog.name = "บราวนี่";
        if (myDog != null) {
            myDog.bark();
        }
    }
}
```

## 9. คุณสมบัติ 4 เสาหลักของ OOP (ภาพรวม)

หลักสูตรนี้จะลงลึกทีละหัวข้อใน Part ถัดไป แต่ควรเห็นภาพรวมทั้ง 4 เสาหลักตั้งแต่ตอนนี้:

| เสาหลัก | ความหมายโดยย่อ | Part ที่สอน |
|---|---|---|
| **Encapsulation** (การห่อหุ้ม) | ซ่อนรายละเอียดภายใน เปิดเผยเฉพาะที่จำเป็นผ่าน public interface | Part 13 |
| **Inheritance** (การสืบทอด) | คลาสลูกสืบทอดคุณสมบัติจากคลาสแม่ ลดโค้ดซ้ำซ้อน | Part 14 |
| **Polymorphism** (พหุสัณฐาน) | object ต่างชนิดตอบสนองต่อคำสั่งเดียวกันแตกต่างกันได้ | Part 15 |
| **Abstraction** (นามธรรม) | ซ่อนความซับซ้อน เปิดเผยแค่สิ่งที่ผู้ใช้ต้องรู้ | Part 16 |

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** สร้างคลาส `Car` ที่มี field: `brand`, `model`, `year`, `speed` และ method:
`accelerate()` (เพิ่มความเร็ว 10), `brake()` (ลดความเร็ว 10 แต่ไม่ต่ำกว่า 0),
`printStatus()`

**เฉลย:**

```java
public class Car {
    String brand;
    String model;
    int year;
    int speed;

    void accelerate() {
        speed += 10;
    }

    void brake() {
        speed = Math.max(0, speed - 10);
    }

    void printStatus() {
        System.out.println(brand + " " + model + " (" + year + ") - ความเร็ว: " + speed + " km/h");
    }
}
```

```java
public class Exercise1 {
    public static void main(String[] args) {
        Car myCar = new Car();
        myCar.brand = "Toyota";
        myCar.model = "Camry";
        myCar.year = 2024;
        myCar.speed = 0;

        myCar.accelerate();
        myCar.accelerate();
        myCar.printStatus(); // ความเร็ว 20

        myCar.brake();
        myCar.printStatus(); // ความเร็ว 10
    }
}
```

**2)** อธิบายว่าทำไมโค้ดนี้ถึงพิมพ์ผลลัพธ์แบบนี้:

```java
Dog a = new Dog();
a.name = "A";
Dog b = a;
b.name = "B";
System.out.println(a.name);
```

**เฉลย**: พิมพ์ `"B"` เพราะ `b = a` เป็นการคัดลอก**reference** ไม่ใช่คัดลอก object
ดังนั้น `a` และ `b` ชี้ไปยัง object เดียวกันบน heap การแก้ไขผ่าน `b.name` จึงส่งผล
กระทบต่อ object เดียวกับที่ `a` ชี้อยู่ด้วย

**3)** เขียนโปรแกรมที่สร้าง `Person[]` array 3 คน แล้ววนพิมพ์ข้อมูลด้วย for-each
โดยต้องเช็ค null ก่อนเรียก method เสมอ

**เฉลย:**

```java
public class Exercise3 {
    public static void main(String[] args) {
        Person[] people = new Person[3];
        people[0] = new Person();
        people[0].setName("Alice");
        people[0].setAge(25);

        people[1] = new Person();
        people[1].setName("Bob");
        people[1].setAge(30);
        // people[2] ยังคงเป็น null โดยตั้งใจ

        for (Person p : people) {
            if (p != null) {
                p.introduce();
            } else {
                System.out.println("(ยังไม่มีข้อมูล)");
            }
        }
    }
}
```

### สรุปเนื้อหา Part 11

- Class คือพิมพ์เขียว, Object คือสิ่งที่สร้างขึ้นจริงจากพิมพ์เขียว (instance)
- Fields เก็บสถานะของแต่ละ object แยกจากกันโดยสิ้นเชิง แม้มาจาก class เดียวกัน
- `new` จัดสรรหน่วยความจำบน heap, เรียก constructor, และคืน reference
- `this` ใช้อ้างอิงถึง object ปัจจุบัน โดยเฉพาะเมื่อชื่อพารามิเตอร์ซ้ำกับ field
- ตัวแปร object เก็บ reference ไม่ใช่ object เอง — การคัดลอกตัวแปรคือคัดลอก reference
- ต้องตรวจสอบ `null` ก่อนเรียกใช้ method บน object เสมอ เพื่อป้องกัน
  `NullPointerException`

**ต่อไป**: [Part 12 — Constructors และ this keyword](./part-012-constructors.md)
