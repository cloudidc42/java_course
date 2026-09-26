# Part 14: Inheritance (การสืบทอด)

> ขั้นตอนที่ 131-140 ของหลักสูตร | ระดับ: OOP พื้นฐาน

## สารบัญ

1. Inheritance คืออะไร ทำไมต้องใช้
2. การสืบทอดด้วย `extends`
3. `super` keyword: เรียก constructor และ method ของคลาสแม่
4. Method Overriding
5. `@Override` annotation
6. กฎการ Override ที่ต้องรู้
7. Single Inheritance และข้อจำกัดของ Java
8. `Object` class: บรรพบุรุษของทุกคลาส
9. `final` class และ `final` method
10. แบบฝึกหัดและสรุป

---

## 1. Inheritance คืออะไร ทำไมต้องใช้

**Inheritance (การสืบทอด)** คือกลไกที่ให้คลาสหนึ่ง (**subclass / child class**)
**สืบทอด fields และ methods** จากอีกคลาสหนึ่ง (**superclass / parent class**)
ช่วยลดโค้ดซ้ำซ้อนและสร้างความสัมพันธ์แบบ **"is-a"** (เป็นชนิดหนึ่งของ)

ตัวอย่างปัญหาที่พบก่อนมี inheritance:

```java
// ไม่มี inheritance: โค้ดซ้ำซ้อนมหาศาลระหว่าง Dog และ Cat
public class Dog {
    String name;
    int age;
    void eat() { System.out.println(name + " กำลังกิน"); }
    void sleep() { System.out.println(name + " กำลังนอน"); }
    void bark() { System.out.println(name + " เห่า: โฮ่ง!"); }
}

public class Cat {
    String name;   // ซ้ำกับ Dog
    int age;       // ซ้ำกับ Dog
    void eat() { System.out.println(name + " กำลังกิน"); }   // ซ้ำกับ Dog
    void sleep() { System.out.println(name + " กำลังนอน"); }  // ซ้ำกับ Dog
    void meow() { System.out.println(name + " ร้อง: เมี้ยว!"); }
}
```

## 2. การสืบทอดด้วย `extends`

```java
// Superclass (Parent class): เก็บสิ่งที่ทุกสัตว์มีร่วมกัน
public class Animal {
    protected String name; // protected: subclass เข้าถึงได้โดยตรง (จะอธิบายในหัวข้อ 3)
    protected int age;

    public Animal(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public void eat() {
        System.out.println(name + " กำลังกิน");
    }

    public void sleep() {
        System.out.println(name + " กำลังนอน");
    }
}
```

```java
// Subclass (Child class): สืบทอด Animal ด้วย extends แล้วเพิ่มพฤติกรรมเฉพาะของตัวเอง
public class Dog extends Animal {
    public Dog(String name, int age) {
        super(name, age); // เรียก constructor ของคลาสแม่ (จะอธิบายในหัวข้อ 3)
    }

    public void bark() { // พฤติกรรมเฉพาะของ Dog เท่านั้น
        System.out.println(name + " เห่า: โฮ่ง โฮ่ง!"); // เข้าถึง name ได้เพราะเป็น protected
    }
}
```

```java
public class Cat extends Animal {
    public Cat(String name, int age) {
        super(name, age);
    }

    public void meow() { // พฤติกรรมเฉพาะของ Cat เท่านั้น
        System.out.println(name + " ร้อง: เมี้ยว~");
    }
}
```

```java
public class InheritanceDemo {
    public static void main(String[] args) {
        Dog dog = new Dog("โปโป้", 3);
        Cat cat = new Cat("มะลิ", 2);

        // สืบทอดมาจาก Animal
        dog.eat();   // ใช้ method ของ Animal ได้เลย ไม่ต้องเขียนซ้ำ
        dog.sleep();
        dog.bark();  // พฤติกรรมเฉพาะของ Dog

        cat.eat();
        cat.meow();  // พฤติกรรมเฉพาะของ Cat
    }
}
```

**ความสัมพันธ์แบบ "is-a"**: `Dog` **is-a** `Animal`, `Cat` **is-a** `Animal`
(สุนัขเป็นสัตว์ชนิดหนึ่ง, แมวเป็นสัตว์ชนิดหนึ่ง) — นี่คือหลักการตัดสินว่าควรใช้
inheritance หรือไม่ ถ้าความสัมพันธ์ไม่ใช่ "is-a" ควรพิจารณา **composition** แทน
(เช่น "Car has-a Engine" ไม่ใช่ "Car is-a Engine")

## 3. `super` keyword: เรียก Constructor และ Method ของคลาสแม่

`super` มี 2 การใช้งานหลัก:

### 3.1 เรียก constructor ของคลาสแม่: `super(...)`

```java
public class Vehicle {
    protected String brand;
    protected int year;

    public Vehicle(String brand, int year) {
        this.brand = brand;
        this.year = year;
        System.out.println("สร้าง Vehicle: " + brand);
    }
}

public class Car extends Vehicle {
    private int numberOfDoors;

    public Car(String brand, int year, int numberOfDoors) {
        super(brand, year); // ต้องเป็นบรรทัดแรกสุดของ constructor เสมอ
        this.numberOfDoors = numberOfDoors;
        System.out.println("สร้าง Car เพิ่มเติม: " + numberOfDoors + " ประตู");
    }
}
```

```java
public class SuperConstructorDemo {
    public static void main(String[] args) {
        Car myCar = new Car("Toyota", 2024, 4);
        // ผลลัพธ์:
        // สร้าง Vehicle: Toyota
        // สร้าง Car เพิ่มเติม: 4 ประตู
    }
}
```

**สำคัญมาก**: ถ้าไม่เขียน `super(...)` เอง Java จะแอบใส่ `super()` (ไม่มีพารามิเตอร์)
ให้อัตโนมัติเป็นบรรทัดแรกของทุก constructor — ถ้าคลาสแม่**ไม่มี**
no-argument constructor จะทำให้ compile error ทันที ต้องเรียก `super(...)`
ที่มีพารามิเตอร์ให้ตรงเอง

```java
public class Base {
    public Base(String requiredValue) { // ไม่มี no-arg constructor
        System.out.println("Base: " + requiredValue);
    }
}

public class Derived extends Base {
    public Derived() {
        // Error ถ้าไม่เขียนบรรทัดนี้: implicit super() หา Base() ไม่เจอ
        super("ค่าที่จำเป็น");
    }
}
```

### 3.2 เรียก method ของคลาสแม่: `super.methodName()`

ใช้เมื่อ override method แล้วแต่ยังต้องการเรียก behavior เดิมของคลาสแม่ร่วมด้วย:

```java
public class Employee {
    protected String name;
    protected double baseSalary;

    public Employee(String name, double baseSalary) {
        this.name = name;
        this.baseSalary = baseSalary;
    }

    public double calculateSalary() {
        return baseSalary;
    }
}

public class Manager extends Employee {
    private double bonus;

    public Manager(String name, double baseSalary, double bonus) {
        super(name, baseSalary);
        this.bonus = bonus;
    }

    @Override
    public double calculateSalary() {
        // เรียก behavior เดิมของคลาสแม่ แล้วเพิ่ม logic เฉพาะของ Manager ต่อ
        return super.calculateSalary() + bonus;
    }
}
```

```java
public class SuperMethodDemo {
    public static void main(String[] args) {
        Manager manager = new Manager("Alice", 30000, 10000);
        System.out.println("เงินเดือนรวม: " + manager.calculateSalary()); // 40000
    }
}
```

## 4. Method Overriding

**Method Overriding** คือการที่ subclass เขียน method ที่มี**signature เดียวกัน
เป๊ะ ๆ** กับ method ในคลาสแม่ เพื่อ**เปลี่ยนพฤติกรรม**ให้เหมาะกับ subclass นั้น
(ต่างจาก Overloading ที่ signature ต้องต่างกัน — อย่าสับสน!)

```java
public class Shape {
    public double calculateArea() {
        return 0; // ค่า default สำหรับรูปทรงทั่วไปที่ไม่รู้สูตรเฉพาะ
    }

    public String describe() {
        return "รูปทรงที่มีพื้นที่: " + calculateArea();
    }
}

public class Circle extends Shape {
    private double radius;

    public Circle(double radius) {
        this.radius = radius;
    }

    @Override
    public double calculateArea() { // override ให้คำนวณตามสูตรวงกลม
        return Math.PI * radius * radius;
    }
}

public class Square extends Shape {
    private double side;

    public Square(double side) {
        this.side = side;
    }

    @Override
    public double calculateArea() { // override ให้คำนวณตามสูตรสี่เหลี่ยมจัตุรัส
        return side * side;
    }
}
```

```java
public class OverridingDemo {
    public static void main(String[] args) {
        Circle circle = new Circle(5);
        Square square = new Square(4);

        System.out.println(circle.describe()); // ใช้ calculateArea() ของ Circle
        System.out.println(square.describe());  // ใช้ calculateArea() ของ Square
        // สังเกตว่า describe() ไม่ได้ถูก override เลย แต่ผลลัพธ์ต่างกันเพราะ
        // calculateArea() ที่มันเรียกใช้ภายในถูก override — นี่คือจุดเริ่มต้นของ
        // Polymorphism ที่จะเรียนเต็มรูปแบบใน Part 15
    }
}
```

## 5. `@Override` Annotation

`@Override` เป็น annotation ที่**ควรใส่เสมอ**เมื่อตั้งใจ override method — มันไม่มี
ผลต่อการทำงานของโปรแกรม แต่ให้ **compiler ช่วยตรวจสอบ**ว่า signature ตรงกับ method
ในคลาสแม่จริงหรือไม่ ป้องกัน bug ที่พบบ่อยมาก:

```java
public class Animal {
    public void makeSound() {
        System.out.println("เสียงสัตว์ทั่วไป");
    }
}

public class Dog extends Animal {
    @Override
    public void makeSound() { // ถูกต้อง: signature ตรงกับคลาสแม่เป๊ะ
        System.out.println("โฮ่ง!");
    }

    // @Override
    // public void makesound() { }  // ถ้าใส่ @Override ตรงนี้ compiler จะ error ทันที!
    //                               // เพราะสะกดผิด (makesound ตัวเล็ก) ไม่ตรงกับ makeSound
    //                               // ถ้าไม่ใส่ @Override โค้ดนี้จะ compile ผ่านเงียบ ๆ
    //                               // กลายเป็น method ใหม่ที่ไม่เกี่ยวอะไรกับ override เลย (bug ร้ายแรง!)
}
```

**บทเรียนสำคัญ**: การใส่ `@Override` เสมอ ช่วยให้ compiler จับ typo หรือ signature
ที่ไม่ตรงกันได้ทันทีตอน compile-time แทนที่จะไปเจอ bug ตอน runtime ซึ่งอาจหาสาเหตุ
ได้ยากมาก

## 6. กฎการ Override ที่ต้องรู้

1. **Signature ต้องตรงกันเป๊ะ** (ชื่อ, จำนวน, ชนิด, ลำดับพารามิเตอร์)
2. **Return type ต้องเหมือนเดิม หรือเป็น subtype ของเดิม** (covariant return type)
3. **Access modifier ต้องเปิดกว้างเท่าเดิมหรือมากกว่า** (ห้ามแคบลง)
4. **ห้าม throw checked exception ที่กว้างกว่าเดิม**
5. **`static`, `final`, `private` methods ไม่สามารถ override ได้** (static คือ
   "hiding" ไม่ใช่ "overriding" — เรื่องนี้ subtle มาก จะอธิบายเพิ่มใน Part 15)

```java
public class Parent {
    protected Object getValue() {
        return "parent value";
    }

    protected void restrictedMethod() {
        System.out.println("Parent method");
    }
}

public class Child extends Parent {
    @Override
    public String getValue() { // OK: String เป็น subtype ของ Object (covariant return)
                                 // และ public เปิดกว้างกว่า protected (ถูกต้อง)
        return "child value";
    }

    @Override
    public void restrictedMethod() { // OK: ขยาย protected -> public ได้ (เปิดกว้างขึ้น)
        System.out.println("Child method");
    }

    // @Override
    // protected void getValue() { } // Error! จะแคบ access modifier ลงจาก public เป็น protected ไม่ได้
}
```

## 7. Single Inheritance และข้อจำกัดของ Java

**Java รองรับ single inheritance เท่านั้น** สำหรับ class — หนึ่ง class สืบทอดได้
จาก**เพียง class เดียว** (ต่างจาก C++ ที่รองรับ multiple inheritance)

```java
public class ClassA { }
public class ClassB { }

// public class ClassC extends ClassA, ClassB { } // Error! Java ไม่รองรับ multiple inheritance สำหรับ class
public class ClassC extends ClassA { } // ถูกต้อง: สืบทอดได้แค่ 1 class เท่านั้น
```

**เหตุผลที่ Java ไม่รองรับ multiple inheritance**: เพื่อหลีกเลี่ยงปัญหา
**Diamond Problem** (ถ้า class ทั้งสองมี method ชื่อเดียวกัน compiler จะไม่รู้ว่า
ควรใช้ตัวไหน) แต่ Java แก้ปัญหานี้บางส่วนด้วย **interface** ที่รองรับการ
"implement" ได้หลายตัว (Part 16)

```java
public interface Flyable { void fly(); }
public interface Swimmable { void swim(); }

// class implement ได้หลาย interface พร้อมกัน (ต่างจาก extends ที่ทำได้แค่ 1 class)
public class Duck implements Flyable, Swimmable {
    @Override
    public void fly() { System.out.println("เป็ดบินได้"); }

    @Override
    public void swim() { System.out.println("เป็ดว่ายน้ำได้"); }
}
```

## 8. `Object` class: บรรพบุรุษของทุกคลาส

ทุกคลาสใน Java **สืบทอดจาก `java.lang.Object` โดยอัตโนมัติ** แม้จะไม่เขียน
`extends Object` เอง (ถ้าเขียน `extends` คลาสอื่น จะสืบทอดจากคลาสนั้น ซึ่งท้ายที่สุด
ก็จะสืบทอดจาก `Object` อยู่ดี เพราะทุก chain ของ inheritance จบที่ `Object` เสมอ)

```java
public class AnyClass {
    // ไม่ได้เขียน extends อะไรเลย แต่จริง ๆ แล้ว "extends Object" อยู่แล้วโดยปริยาย
}
```

`Object` มี method สำคัญที่ทุกคลาสได้รับมาโดยอัตโนมัติ (มักถูก override):

```java
public class Point {
    private int x, y;

    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }

    @Override
    public String toString() { // ควบคุมว่าเวลา print object จะแสดงข้อความอะไร
        return "Point(" + x + ", " + y + ")";
    }

    @Override
    public boolean equals(Object obj) { // ควบคุมการเปรียบเทียบเนื้อหา (จะลงลึกใน Part 27)
        if (this == obj) return true;
        if (!(obj instanceof Point)) return false;
        Point other = (Point) obj;
        return this.x == other.x && this.y == other.y;
    }

    @Override
    public int hashCode() { // ต้อง override คู่กับ equals เสมอ (จะอธิบายเหตุผลใน Part 24)
        return java.util.Objects.hash(x, y);
    }
}
```

```java
public class ObjectMethodsDemo {
    public static void main(String[] args) {
        Point p1 = new Point(1, 2);
        Point p2 = new Point(1, 2);

        System.out.println(p1); // เรียก toString() อัตโนมัติ -> "Point(1, 2)"
        System.out.println(p1.equals(p2)); // true (เพราะ override equals แล้ว)
        System.out.println(p1 == p2);      // false (คนละ object ใน heap)

        // ถ้าไม่ override toString() การ print object จะได้ผลลัพธ์แปลก ๆ เช่น:
        // Point@1b6d3586 (ชื่อคลาส@hashcode แบบ hexadecimal - ไม่มีประโยชน์)
    }
}
```

## 9. `final` Class และ `final` Method

- **`final` class**: ห้าม subclass สืบทอดต่อ (เช่น `String`, `Integer` เป็น
  `final class` ป้องกันไม่ให้ใครมาสืบทอดแล้วเปลี่ยนพฤติกรรมพื้นฐาน)
- **`final` method**: ห้าม subclass override method นี้ (ล็อค behavior ไว้)

```java
public final class ImmutableConfig { // final class: ไม่มีใครสืบทอดต่อได้
    private final String value;
    public ImmutableConfig(String value) { this.value = value; }
    public String getValue() { return value; }
}

// public class ExtendedConfig extends ImmutableConfig { } // Error! ImmutableConfig เป็น final

public class SecuritySettings {
    public final void validateAccess() { // final method: subclass override ไม่ได้
        System.out.println("ตรวจสอบสิทธิ์แบบมาตรฐาน");
    }
}

public class CustomSettings extends SecuritySettings {
    // public void validateAccess() { }  // Error! ห้าม override method ที่เป็น final
}
```

**เหตุผลที่ใช้ `final`**: ป้องกันไม่ให้ subclass เปลี่ยนแปลง behavior ที่สำคัญต่อ
ความปลอดภัยหรือความถูกต้องของระบบ (เช่น validation logic, security checks)

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** สร้าง superclass `Shape` ที่มีเมธอด `calculateArea()` คืนค่า 0 แล้วสร้าง
subclass `Triangle` ที่ override ให้คำนวณพื้นที่สามเหลี่ยมจริง (ฐาน x สูง / 2)

**เฉลย:**

```java
public class Shape {
    public double calculateArea() {
        return 0;
    }
}

public class Triangle extends Shape {
    private double base, height;

    public Triangle(double base, double height) {
        this.base = base;
        this.height = height;
    }

    @Override
    public double calculateArea() {
        return base * height / 2;
    }
}
```

**2)** สร้างคลาส `Person` ที่มี constructor รับ `name`, `age` และคลาส `Student`
ที่ extends `Person` เพิ่ม field `studentId` โดยใช้ `super(...)` ในการเรียก
constructor ของคลาสแม่

**เฉลย:**

```java
public class Person {
    protected String name;
    protected int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
}

public class Student extends Person {
    private String studentId;

    public Student(String name, int age, String studentId) {
        super(name, age);
        this.studentId = studentId;
    }
}
```

**3)** ทำนายว่าโค้ดนี้จะ compile ผ่านหรือไม่ เพราะเหตุใด:

```java
public class Parent {
    public String getName() { return "parent"; }
}
public class Child extends Parent {
    @Override
    public int getName() { return 1; }
}
```

**เฉลย**: **Compile ไม่ผ่าน** เพราะ return type ของ `getName()` ใน `Child`
(`int`) ไม่ใช่ subtype ของ return type ในคลาสแม่ (`String`) — กฎ covariant return
type กำหนดว่า return type ต้องเหมือนเดิมหรือเป็น subtype เท่านั้น `int` กับ
`String` ไม่มีความสัมพันธ์แบบสืบทอดกันเลย

### สรุปเนื้อหา Part 14

- Inheritance ให้ subclass สืบทอด fields/methods จาก superclass ด้วย `extends`
  ลดโค้ดซ้ำซ้อน สร้างความสัมพันธ์แบบ "is-a"
- `super(...)` เรียก constructor ของคลาสแม่ (ต้องเป็นบรรทัดแรกสุด), `super.method()`
  เรียก method ของคลาสแม่
- Method Overriding: signature ต้องตรงกันเป๊ะ, ใส่ `@Override` เสมอเพื่อให้ compiler
  ช่วยตรวจสอบ
- Java รองรับ single inheritance สำหรับ class เท่านั้น แต่ implement ได้หลาย
  interface
- ทุกคลาสสืบทอดจาก `Object` โดยอัตโนมัติ ได้ `toString()`, `equals()`, `hashCode()`
  มาให้ override
- `final` class ห้ามสืบทอดต่อ, `final` method ห้าม override

**ต่อไป**: [Part 15 — Polymorphism](./part-015-polymorphism.md)
