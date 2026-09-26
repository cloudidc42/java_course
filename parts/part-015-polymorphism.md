# Part 15: Polymorphism (โพลีมอร์ฟิซึม)

> ขั้นตอนที่ 141-150 ของหลักสูตร | ระดับ: OOP พื้นฐาน

## สารบัญ

1. Polymorphism คืออะไร
2. Compile-time Polymorphism vs Runtime Polymorphism
3. Upcasting: การมองลูกเป็นแม่
4. Dynamic Method Dispatch (หัวใจของ Runtime Polymorphism)
5. Downcasting และ `instanceof`
6. Pattern Matching for `instanceof` (Java 16+)
7. Polymorphism กับ Array และ Collection
8. Field Hiding (กับดักที่ต้องระวัง: Field ไม่ใช่ Polymorphic)
9. Static Method ไม่ใช่ Polymorphic (Method Hiding)
10. แบบฝึกหัดและสรุป

---

## 1. Polymorphism คืออะไร

**Polymorphism** (มาจากภาษากรีก แปลว่า "หลายรูปแบบ") คือความสามารถที่ **object ต่าง
ชนิดกันตอบสนองต่อคำสั่งเดียวกันแตกต่างกันไปตามชนิดจริงของมัน** — เราได้เห็นตัวอย่าง
เบื้องต้นมาแล้วใน Part 14 (`Circle.calculateArea()` vs `Square.calculateArea()`)
Part นี้จะเจาะลึกกลไกเบื้องหลังอย่างละเอียด

Polymorphism แบ่งเป็น 2 ประเภทหลัก:

| ประเภท | เรียกอีกชื่อ | ตัดสินใจตอนไหน | ตัวอย่าง |
|---|---|---|---|
| **Compile-time Polymorphism** | Static Binding | ตอน compile | Method Overloading |
| **Runtime Polymorphism** | Dynamic Binding | ตอน runtime | Method Overriding |

## 2. Compile-time Polymorphism vs Runtime Polymorphism

### Compile-time Polymorphism (Method Overloading — ทบทวนจาก Part 8)

```java
public class Calculator {
    int add(int a, int b) { return a + b; }
    double add(double a, double b) { return a + b; }
    int add(int a, int b, int c) { return a + b + c; }
}
```

```java
public class CompileTimeDemo {
    public static void main(String[] args) {
        Calculator calc = new Calculator();
        // Compiler รู้ตั้งแต่ compile-time แล้วว่าจะเรียก overload ตัวไหน
        // จากชนิดและจำนวนของ argument ที่ส่งเข้ามา (ไม่ต้องรอดูตอน runtime)
        System.out.println(calc.add(1, 2));       // เรียก add(int, int) แน่นอน
        System.out.println(calc.add(1.5, 2.5));    // เรียก add(double, double) แน่นอน
    }
}
```

### Runtime Polymorphism (Method Overriding)

นี่คือหัวใจสำคัญที่สุดของ OOP และเป็นสิ่งที่คนมักหมายถึงเวลาพูดถึง "Polymorphism"
เฉย ๆ (ไม่ระบุประเภท)

## 3. Upcasting: การมองลูกเป็นแม่

**Upcasting** คือการอ้างอิงถึง object ของ subclass ผ่านตัวแปรชนิด superclass
(หรือ interface) — ทำได้อัตโนมัติเสมอ เพราะ subclass **is-a** superclass

```java
public class Animal {
    public String makeSound() {
        return "เสียงสัตว์ทั่วไป";
    }
}

public class Dog extends Animal {
    @Override
    public String makeSound() {
        return "โฮ่ง โฮ่ง!";
    }
}

public class Cat extends Animal {
    @Override
    public String makeSound() {
        return "เมี้ยว~";
    }
}
```

```java
public class UpcastingDemo {
    public static void main(String[] args) {
        Animal animal1 = new Dog(); // Upcasting: ตัวแปรชนิด Animal ชี้ไปยัง object Dog
        Animal animal2 = new Cat(); // Upcasting: ตัวแปรชนิด Animal ชี้ไปยัง object Cat

        System.out.println(animal1.makeSound()); // "โฮ่ง โฮ่ง!" - เรียก method ของ Dog จริง!
        System.out.println(animal2.makeSound()); // "เมี้ยว~"    - เรียก method ของ Cat จริง!

        // แม้ตัวแปรจะประกาศเป็นชนิด Animal แต่ JVM รู้ว่า object จริงคือ Dog/Cat
        // และเรียก method เวอร์ชันที่ถูก override ไว้ ไม่ใช่เวอร์ชันของ Animal
    }
}
```

**ทำไม Upcasting ถึงมีประโยชน์?** เพราะเราสามารถเขียนโค้ดที่ทำงานกับ**ชนิดข้อมูล
ทั่วไป (Animal)** โดยไม่ต้องรู้ล่วงหน้าว่าเป็น Dog หรือ Cat และยังได้ behavior
ที่ถูกต้องเฉพาะของแต่ละชนิดโดยอัตโนมัติ:

```java
public class PolymorphicMethodDemo {
    // เมธอดนี้รับ Animal ชนิดใดก็ได้ ไม่ต้องเขียนแยกสำหรับ Dog, Cat, Bird ฯลฯ
    static void introduceAnimal(Animal animal) {
        System.out.println("สัตว์นี้ส่งเสียง: " + animal.makeSound());
    }

    public static void main(String[] args) {
        introduceAnimal(new Dog()); // ทำงานถูกต้องสำหรับ Dog
        introduceAnimal(new Cat()); // ทำงานถูกต้องสำหรับ Cat โดยไม่ต้องแก้โค้ดเมธอดเลย
        // ในอนาคตถ้ามี Bird extends Animal เพิ่มเข้ามา ก็เรียก introduceAnimal(new Bird())
        // ได้ทันทีโดยไม่ต้องแก้ไขเมธอดนี้เลยแม้แต่บรรทัดเดียว! นี่คือพลังของ Polymorphism
    }
}
```

## 4. Dynamic Method Dispatch (หัวใจของ Runtime Polymorphism)

**Dynamic Method Dispatch** คือกลไกที่ JVM ใช้ตัดสินใจว่าจะเรียก method เวอร์ชัน
ไหน**ตอน runtime** โดยดูจาก**ชนิดจริงของ object** (ไม่ใช่ชนิดของตัวแปรที่ประกาศไว้)

```java
public class DynamicDispatchDemo {
    public static void main(String[] args) {
        Animal[] animals = new Animal[3]; // array ของ Animal แต่เก็บ object ต่างชนิดกันได้
        animals[0] = new Dog();
        animals[1] = new Cat();
        animals[2] = new Animal();

        for (Animal animal : animals) {
            // JVM ตัดสินใจตอน runtime ว่าจะเรียก makeSound() เวอร์ชันไหน
            // โดยดูจากชนิดจริงของ object ใน array แต่ละตัว (Dynamic Dispatch)
            System.out.println(animal.makeSound());
        }
        // ผลลัพธ์: "โฮ่ง โฮ่ง!", "เมี้ยว~", "เสียงสัตว์ทั่วไป"
    }
}
```

**เปรียบเทียบ Static Binding (สำหรับ overloading) กับ Dynamic Binding (สำหรับ
overriding)**:

```
Static Binding (Overloading):
  compiler ตัดสินใจจาก "ชนิดของตัวแปรที่ประกาศ" ตอน compile-time

Dynamic Binding (Overriding):
  JVM ตัดสินใจจาก "ชนิดจริงของ object" ตอน runtime
```

## 5. Downcasting และ `instanceof`

**Downcasting** คือการแปลงตัวแปรชนิด superclass กลับไปเป็นชนิด subclass เพื่อ
เข้าถึง method ที่มีเฉพาะใน subclass เท่านั้น (ต้อง cast ด้วยมือ และเสี่ยง
`ClassCastException` ถ้า object จริงไม่ใช่ subclass ที่ cast ไป)

```java
public class Dog extends Animal {
    @Override
    public String makeSound() { return "โฮ่ง!"; }

    public void fetch() { // method เฉพาะของ Dog เท่านั้น ไม่มีใน Animal
        System.out.println("คาบลูกบอลกลับมา!");
    }
}
```

```java
public class DowncastingDemo {
    public static void main(String[] args) {
        Animal animal = new Dog(); // Upcasting

        // animal.fetch(); // Error! ตัวแปร animal ชนิด Animal ไม่มีเมธอด fetch()
        //                  // แม้ object จริงจะเป็น Dog ก็ตาม (compiler มองแค่ชนิดที่ประกาศ)

        Dog dog = (Dog) animal; // Downcasting: แปลงกลับเป็น Dog เพื่อเรียก fetch()
        dog.fetch(); // ทำงานได้ถูกต้องเพราะ object จริงคือ Dog

        // Downcasting ที่ผิดพลาดจะเกิด ClassCastException ตอน runtime
        Animal justAnimal = new Animal();
        try {
            Dog wrongCast = (Dog) justAnimal; // Error! justAnimal ไม่ใช่ Dog จริง ๆ
        } catch (ClassCastException e) {
            System.out.println("แปลงชนิดไม่ได้: " + e.getMessage());
        }
    }
}
```

**หลักการป้องกัน**: ตรวจสอบด้วย `instanceof` ก่อน downcast เสมอ เพื่อป้องกัน
`ClassCastException`:

```java
public class SafeDowncastingDemo {
    static void handleAnimal(Animal animal) {
        System.out.println(animal.makeSound());

        if (animal instanceof Dog) { // ตรวจสอบก่อนเสมอ
            Dog dog = (Dog) animal;
            dog.fetch();
        }
    }

    public static void main(String[] args) {
        handleAnimal(new Dog()); // ทำงานทั้ง makeSound และ fetch
        handleAnimal(new Cat()); // ทำงานแค่ makeSound (ข้าม fetch เพราะไม่ใช่ Dog)
    }
}
```

## 6. Pattern Matching for `instanceof` (Java 16+)

Java 16 เพิ่ม syntax ที่รวมการเช็ค `instanceof` และการ cast เป็นขั้นตอนเดียว
ลดความยืดยาวและความเสี่ยงจากการลืม cast:

```java
public class PatternMatchingInstanceofDemo {
    static void handleAnimal(Animal animal) {
        System.out.println(animal.makeSound());

        // แบบเก่า: เช็คแล้วต้อง cast เองอีกที
        if (animal instanceof Dog) {
            Dog dog = (Dog) animal;
            dog.fetch();
        }

        // แบบใหม่ (Java 16+): เช็คและ cast พร้อมกันในบรรทัดเดียว ได้ตัวแปร dog มาใช้ทันที
        if (animal instanceof Dog dog) {
            dog.fetch(); // ใช้ dog ได้เลย ไม่ต้อง cast ซ้ำ
        }
    }

    public static void main(String[] args) {
        handleAnimal(new Dog());
    }
}
```

## 7. Polymorphism กับ Array และ Collection

Polymorphism มีประโยชน์อย่างมากเมื่อจัดการกับ collection ของ object ที่มีชนิด
แตกต่างกันแต่สืบทอดจากคลาสแม่เดียวกัน — เป็นรูปแบบที่ใช้บ่อยมากในโค้ด production จริง:

```java
public abstract class Shape { // จะเรียนเรื่อง abstract อย่างละเอียดใน Part 16
    public abstract double calculateArea();
}

public class Circle extends Shape {
    private double radius;
    public Circle(double radius) { this.radius = radius; }
    @Override
    public double calculateArea() { return Math.PI * radius * radius; }
}

public class Rectangle extends Shape {
    private double width, height;
    public Rectangle(double width, double height) { this.width = width; this.height = height; }
    @Override
    public double calculateArea() { return width * height; }
}
```

```java
import java.util.ArrayList;
import java.util.List;

public class PolymorphicCollectionDemo {
    public static void main(String[] args) {
        List<Shape> shapes = new ArrayList<>(); // list เก็บ Shape ทุกชนิดที่สืบทอดมาได้
        shapes.add(new Circle(5));
        shapes.add(new Rectangle(4, 6));
        shapes.add(new Circle(3));

        double totalArea = 0;
        for (Shape shape : shapes) {
            totalArea += shape.calculateArea(); // เรียก method ที่ถูก override ตามชนิดจริง
        }

        System.out.println("พื้นที่รวมทั้งหมด: " + totalArea);
        // ไม่ต้องรู้เลยว่าแต่ละตัวเป็น Circle หรือ Rectangle - โค้ดทำงานถูกต้องเสมอ
    }
}
```

## 8. Field Hiding (กับดักที่ต้องระวัง: Field ไม่ใช่ Polymorphic)

**นี่คือกับดักที่สำคัญมาก**: **Field ไม่มี polymorphism แบบ method** — การเข้าถึง
field จะถูกตัดสินจาก**ชนิดของตัวแปรที่ประกาศ** (compile-time / static binding)
ไม่ใช่ชนิดจริงของ object เหมือน method

```java
public class ParentField {
    String label = "Parent Label"; // field ชื่อเดียวกับใน ChildField
}

public class ChildField extends ParentField {
    String label = "Child Label"; // นี่คือ "field hiding" ไม่ใช่ overriding!
}
```

```java
public class FieldHidingDemo {
    public static void main(String[] args) {
        ParentField obj = new ChildField(); // Upcasting

        System.out.println(obj.label); // "Parent Label" (!!) ผิดจากที่คาดหวังมาก

        ChildField childObj = new ChildField();
        System.out.println(childObj.label); // "Child Label" (ถ้าตัวแปรชนิด ChildField ตรง ๆ)

        // เปรียบเทียบกับ method ที่เป็น polymorphic จริง:
        // obj.someMethod() จะเรียกเวอร์ชันของ ChildField เสมอ (dynamic binding)
        // แต่ obj.label จะได้ค่าของ ParentField เสมอ (static binding ตามชนิดตัวแปร)
    }
}
```

**บทเรียนสำคัญ**: **อย่าตั้งชื่อ field ซ้ำกับคลาสแม่** เพราะจะทำให้เกิดความสับสน
พฤติกรรมที่ไม่คาดคิด — นี่คือเหตุผลสำคัญข้อหนึ่งที่ควรทำให้ field เป็น `private`
เสมอ (encapsulation จาก Part 13) และเข้าถึงผ่าน getter method ที่ **เป็น
polymorphic ได้จริง**:

```java
public class ParentSafe {
    private String label = "Parent Label";
    public String getLabel() { return label; } // เป็น method จึงมี polymorphism ถูกต้อง
}

public class ChildSafe extends ParentSafe {
    // ไม่ประกาศ field label ซ้ำ ให้ override getter แทน
    @Override
    public String getLabel() { return "Child Label (overridden)"; }
}
```

```java
public class SafeFieldAccessDemo {
    public static void main(String[] args) {
        ParentSafe obj = new ChildSafe();
        System.out.println(obj.getLabel()); // "Child Label (overridden)" - ถูกต้องตามที่คาดหวัง!
    }
}
```

## 9. Static Method ไม่ใช่ Polymorphic (Method Hiding)

เช่นเดียวกับ field, **static method ก็ไม่ใช่ polymorphic** — การเรียก static
method ผ่าน object variable จะถูกตัดสินจากชนิดของตัวแปรที่ประกาศ (ไม่ใช่ชนิดจริง)
เรียกว่า **method hiding** (ไม่ใช่ overriding)

```java
public class ParentStatic {
    static void staticMethod() {
        System.out.println("Parent static method");
    }
}

public class ChildStatic extends ParentStatic {
    static void staticMethod() { // นี่คือ "hiding" ไม่ใช่ "overriding"
        System.out.println("Child static method");
    }
}
```

```java
public class MethodHidingDemo {
    public static void main(String[] args) {
        ParentStatic obj = new ChildStatic();
        obj.staticMethod(); // "Parent static method" (!) ตัดสินจากชนิดตัวแปร ไม่ใช่ dynamic dispatch

        // แนวปฏิบัติที่ดี: เรียก static method ผ่านชื่อคลาสตรง ๆ เสมอ ไม่ผ่าน instance
        ChildStatic.staticMethod(); // "Child static method" (ชัดเจน ไม่กำกวม)
        ParentStatic.staticMethod(); // "Parent static method"
    }
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** สร้าง class `Employee` มี method `calculateBonus()` คืนค่า 1000 แล้วสร้าง
`SalesEmployee extends Employee` ที่ override ให้คืนค่า 5000 ทดสอบด้วยการเก็บใน
`List<Employee>` แล้ววนคำนวณโบนัสรวม

**เฉลย:**

```java
import java.util.ArrayList;
import java.util.List;

public class Employee {
    double calculateBonus() { return 1000; }
}

public class SalesEmployee extends Employee {
    @Override
    double calculateBonus() { return 5000; }
}

public class Exercise1 {
    public static void main(String[] args) {
        List<Employee> employees = new ArrayList<>();
        employees.add(new Employee());
        employees.add(new SalesEmployee());
        employees.add(new SalesEmployee());

        double total = 0;
        for (Employee e : employees) {
            total += e.calculateBonus();
        }
        System.out.println("โบนัสรวม: " + total); // 11000
    }
}
```

**2)** เขียนเมธอด `printDetails(Animal animal)` ที่ใช้ pattern matching
`instanceof` ตรวจสอบว่าเป็น `Dog` หรือ `Cat` แล้วเรียก method เฉพาะของแต่ละชนิด

**เฉลย:**

```java
public class Exercise2 {
    static void printDetails(Animal animal) {
        System.out.println(animal.makeSound());
        if (animal instanceof Dog dog) {
            dog.fetch();
        } else if (animal instanceof Cat cat) {
            System.out.println("แมวกำลังข่วนเสาลับเล็บ");
        }
    }
}
```

**3)** อธิบายว่าทำไมโค้ดนี้พิมพ์ `"Animal Type"` แทนที่จะเป็น `"Dog Type"`:

```java
class Animal { String type = "Animal Type"; }
class Dog extends Animal { String type = "Dog Type"; }

Animal a = new Dog();
System.out.println(a.type);
```

**เฉลย**: เพราะ field ไม่มี polymorphism — การเข้าถึง `a.type` ถูกตัดสินจาก
**ชนิดของตัวแปร a ที่ประกาศไว้เป็น `Animal`** (static/compile-time binding)
ไม่ใช่ชนิดจริงของ object (`Dog`) ต่างจาก method ที่ใช้ dynamic binding

### สรุปเนื้อหา Part 15

- Polymorphism แบ่งเป็น Compile-time (Overloading) และ Runtime (Overriding)
- Upcasting: มองตัวแปร subclass ผ่านชนิด superclass ได้เสมออัตโนมัติ
- Dynamic Method Dispatch: JVM ตัดสินใจเรียก method เวอร์ชันไหนจาก**ชนิดจริงของ
  object** ตอน runtime
- Downcasting ต้อง cast เอง และเสี่ยง `ClassCastException` — ควรเช็ค `instanceof`
  ก่อนเสมอ
- **Field และ static method ไม่มี polymorphism** — ถูกตัดสินจากชนิดตัวแปรที่ประกาศ
  (static binding) เป็นกับดักสำคัญที่ต้องจำ
- Polymorphism ทำให้เขียนโค้ดที่ทำงานกับ collection ของ object หลายชนิดได้อย่าง
  ยืดหยุ่นและขยายง่ายในอนาคต

**ต่อไป**: [Part 16 — Abstract Classes และ Interfaces](./part-016-abstract-and-interfaces.md)
