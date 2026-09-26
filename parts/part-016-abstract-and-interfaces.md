# Part 16: Abstract Classes และ Interfaces

> ขั้นตอนที่ 151-160 ของหลักสูตร | ระดับ: OOP พื้นฐาน (จบหมวด OOP หลัก)

## สารบัญ

1. Abstraction คืออะไร (เสาหลักสุดท้ายของ OOP)
2. Abstract Class และ Abstract Method
3. กฎของ Abstract Class
4. Interface คืออะไร
5. `default` และ `static` Methods ใน Interface (Java 8+)
6. Multiple Interface Implementation
7. Interface Inheritance (extends ระหว่าง interface)
8. Abstract Class vs Interface: เลือกใช้แบบไหนเมื่อไร
9. Functional Interface เบื้องต้น (ปูทางสู่ Lambda ใน Part 39)
10. แบบฝึกหัดและสรุป

---

## 1. Abstraction คืออะไร (เสาหลักสุดท้ายของ OOP)

**Abstraction** คือการ**ซ่อนรายละเอียดที่ซับซ้อน** และ**เปิดเผยเฉพาะสิ่งที่จำเป็น**
ต่อการใช้งาน — ต่างจาก Encapsulation (ซ่อนข้อมูลภายใน object) ตรงที่ Abstraction
เน้นที่การ**ออกแบบ contract/พิมพ์เขียวระดับสูง** ว่า "ต้องทำอะไรได้บ้าง" โดยไม่สนใจ
"ทำอย่างไร" (Encapsulation คือ "how", Abstraction คือ "what")

เปรียบเทียบ: การขับรถ เราแค่รู้ว่า "หมุนพวงมาลัย, เหยียบคันเร่ง, เหยียบเบรก" ได้
(Abstraction ระดับ interface) โดยไม่จำเป็นต้องรู้กลไกเครื่องยนต์ภายในทำงานยังไง
(รายละเอียด implementation ที่ถูกซ่อนไว้)

Java มี 2 กลไกหลักในการทำ Abstraction: **Abstract Class** และ **Interface**

## 2. Abstract Class และ Abstract Method

**Abstract Class** คือ class ที่**สร้าง object โดยตรงไม่ได้** (ต้องมี subclass
มาสืบทอดและ implement ส่วนที่ขาดไป) ใช้เมื่อมีคลาสแม่ที่มี logic ร่วมกันบางส่วน
แต่มีบาง method ที่**ต้องบังคับให้ subclass implement เอง**

```java
public abstract class Shape { // ประกาศด้วย abstract - สร้าง object ตรง ๆ ไม่ได้
    protected String color;

    public Shape(String color) { // abstract class มี constructor ได้ (เรียกผ่าน super() จาก subclass)
        this.color = color;
    }

    // Abstract method: ไม่มี body ต้องลงท้ายด้วย ; ทันที บังคับให้ subclass implement เอง
    public abstract double calculateArea();
    public abstract double calculatePerimeter();

    // Concrete method (method ปกติที่มี body): subclass ใช้ร่วมกันได้เลย ไม่ต้อง implement ซ้ำ
    public void printInfo() {
        System.out.println("รูปทรงสี " + color + " มีพื้นที่ " + calculateArea()
                          + " และเส้นรอบรูป " + calculatePerimeter());
    }
}
```

```java
public class Circle extends Shape {
    private double radius;

    public Circle(String color, double radius) {
        super(color); // เรียก constructor ของ abstract class ได้ตามปกติ
        this.radius = radius;
    }

    @Override
    public double calculateArea() { // บังคับ implement เพราะเป็น abstract ใน Shape
        return Math.PI * radius * radius;
    }

    @Override
    public double calculatePerimeter() { // บังคับ implement เช่นกัน
        return 2 * Math.PI * radius;
    }
}
```

```java
public class AbstractClassDemo {
    public static void main(String[] args) {
        // Shape shape = new Shape("แดง"); // Error! สร้าง object จาก abstract class โดยตรงไม่ได้

        Shape circle = new Circle("แดง", 5); // สร้างผ่าน subclass ที่ implement ครบแล้วได้
        circle.printInfo(); // ใช้ concrete method จาก Shape ได้เลย
    }
}
```

## 3. กฎของ Abstract Class

1. ประกาศด้วย keyword `abstract` หน้า `class`
2. **สร้าง object โดยตรงไม่ได้** (`new Shape()` จะ compile error)
3. มีได้ทั้ง **abstract method** (ไม่มี body) และ **concrete method** (มี body ปกติ)
4. มี constructor ได้ (แม้สร้าง object ตรงไม่ได้ แต่ subclass เรียกผ่าน `super()` ได้)
5. มี field, static method, static field ได้ตามปกติเหมือน class ทั่วไป
6. **Subclass ที่ไม่ใช่ abstract ต้อง implement abstract method ทุกตัวให้ครบ**
   ถ้าไม่ implement ครบ subclass นั้นต้องประกาศเป็น `abstract` ด้วยเช่นกัน

```java
public abstract class Vehicle {
    public abstract void start();
    public abstract void stop();
}

// Subclass ที่ implement ไม่ครบ ต้องเป็น abstract ด้วย (ส่งต่อภาระให้ subclass ถัดไป)
public abstract class MotorVehicle extends Vehicle {
    @Override
    public void start() {
        System.out.println("สตาร์ทเครื่องยนต์");
    }
    // ยังไม่ implement stop() -> ต้องคง abstract ไว้
}

// Subclass สุดท้ายที่ implement ครบทุก abstract method แล้ว จึงเป็น concrete class ได้
public class Motorcycle extends MotorVehicle {
    @Override
    public void stop() {
        System.out.println("ดับเครื่องยนต์");
    }
}
```

## 4. Interface คืออะไร

**Interface** คือ "สัญญา (contract)" ที่กำหนดว่า class ที่ implement มันจะต้องมี
method อะไรบ้าง โดย**ไม่สนใจการ implement ภายในเลย** (จนกระทั่ง Java 8 ที่เริ่ม
อนุญาตให้มี default/static method ได้ — หัวข้อ 5)

```java
public interface Payable {
    void pay(double amount); // method ใน interface เป็น public abstract โดยปริยาย (ไม่ต้องเขียนเอง)
}

public interface Refundable {
    void refund(double amount);
}
```

```java
public class CreditCardPayment implements Payable, Refundable {
    // implements หลาย interface พร้อมกันได้ (ต่างจาก extends ที่ทำได้แค่ 1 class)

    @Override
    public void pay(double amount) {
        System.out.println("จ่ายเงิน " + amount + " บาทผ่านบัตรเครดิต");
    }

    @Override
    public void refund(double amount) {
        System.out.println("คืนเงิน " + amount + " บาทเข้าบัตรเครดิต");
    }
}
```

```java
public class InterfaceDemo {
    public static void main(String[] args) {
        CreditCardPayment payment = new CreditCardPayment();
        payment.pay(1000);
        payment.refund(200);

        // ใช้ Polymorphism ผ่าน interface type ได้เหมือน abstract class
        Payable payable = payment;
        payable.pay(500);
    }
}
```

**คุณสมบัติของ interface**:
- ทุก field ใน interface เป็น `public static final` (constant) โดยปริยาย
- ทุก method (ที่ไม่ใช่ default/static) เป็น `public abstract` โดยปริยาย
- **ไม่มี constructor** (สร้าง object จาก interface ตรง ๆ ไม่ได้เลย)
- Class implement interface ได้**หลายตัวพร้อมกัน**

```java
public interface Config {
    int MAX_RETRY = 3; // เท่ากับ: public static final int MAX_RETRY = 3;
}

public class ConfigDemo {
    public static void main(String[] args) {
        System.out.println(Config.MAX_RETRY); // เข้าถึงผ่านชื่อ interface ตรง ๆ ได้เลย (เหมือน static)
    }
}
```

## 5. `default` และ `static` Methods ใน Interface (Java 8+)

ก่อน Java 8 interface มีแค่ abstract method ล้วน ๆ แต่ Java 8 เพิ่ม **default
method** (มี body, subclass ไม่จำเป็นต้อง override) และ **static method**
(เรียกผ่านชื่อ interface ได้เลย) เพื่อให้ขยาย interface ในอนาคตได้โดยไม่ทำให้
class ที่ implement อยู่แล้วพัง (backward compatibility)

```java
public interface Greetable {
    String getName(); // abstract method - ยังต้อง implement

    // default method: มี body ในตัว ไม่บังคับให้ override (แต่ override ได้ถ้าต้องการ)
    default void greet() {
        System.out.println("สวัสดี, " + getName() + "!");
    }

    // static method: เรียกผ่านชื่อ interface ตรง ๆ เหมือน utility method
    static Greetable of(String name) {
        return () -> name; // ใช้ lambda expression (จะเรียนใน Part 39) เพราะมี abstract method เดียว
    }
}
```

```java
public class Person implements Greetable {
    private String name;
    public Person(String name) { this.name = name; }

    @Override
    public String getName() { return name; }

    // ไม่จำเป็นต้อง override greet() เพราะมี default implementation ให้แล้ว
}
```

```java
public class DefaultMethodDemo {
    public static void main(String[] args) {
        Person p = new Person("Alice");
        p.greet(); // ใช้ default method จาก interface ได้เลย: "สวัสดี, Alice!"

        Greetable quick = Greetable.of("Bob"); // เรียก static method ของ interface
        quick.greet(); // "สวัสดี, Bob!"
    }
}
```

### ปัญหา Diamond Problem กับ default method

หาก class implement 2 interface ที่มี default method ชื่อเดียวกัน **ต้อง override
เองเพื่อแก้ความกำกวม** (compiler บังคับ ไม่ยอมให้เดาเอง):

```java
public interface InterfaceA {
    default void hello() { System.out.println("Hello from A"); }
}

public interface InterfaceB {
    default void hello() { System.out.println("Hello from B"); }
}

public class DiamondClass implements InterfaceA, InterfaceB {
    @Override
    public void hello() { // บังคับต้อง override เพื่อแก้ความกำกวมว่าจะใช้ตัวไหน
        InterfaceA.super.hello(); // เลือกเรียก default method ของ InterfaceA อย่างชัดเจน
        System.out.println("และเพิ่มเติมจาก DiamondClass เอง");
    }
}
```

## 6. Multiple Interface Implementation

จุดแข็งสำคัญของ interface คือ class หนึ่งสามารถ implement ได้**หลาย interface**
พร้อมกัน (แก้ข้อจำกัดของ single inheritance ใน class):

```java
public interface Flyable {
    void fly();
}

public interface Swimmable {
    void swim();
}

public interface Walkable {
    void walk();
}

public class Duck implements Flyable, Swimmable, Walkable {
    @Override
    public void fly() { System.out.println("เป็ดบินได้"); }

    @Override
    public void swim() { System.out.println("เป็ดว่ายน้ำได้"); }

    @Override
    public void walk() { System.out.println("เป็ดเดินได้"); }
}
```

```java
public class MultipleInterfaceDemo {
    public static void main(String[] args) {
        Duck duck = new Duck();
        duck.fly();
        duck.swim();
        duck.walk();

        // ใช้ polymorphism ผ่านแต่ละ interface ได้อิสระ
        Flyable flyer = duck;
        flyer.fly();
    }
}
```

## 7. Interface Inheritance (extends ระหว่าง interface)

Interface สามารถ `extends` interface อื่นได้ (และ**extends ได้หลายตัวพร้อมกัน**
ต่างจาก class ที่ extends class ได้แค่ 1 ตัว):

```java
public interface Animal {
    String getName();
}

public interface Pet extends Animal { // interface extends interface ได้
    String getOwnerName();
}

public interface TrainedPet extends Pet { // สืบทอดต่อกันเป็นทอด ๆ ได้
    void performTrick();
}

public class TrainedDog implements TrainedPet { // ต้อง implement ทุก method จากทุกชั้น
    private String name, ownerName;

    public TrainedDog(String name, String ownerName) {
        this.name = name;
        this.ownerName = ownerName;
    }

    @Override
    public String getName() { return name; }

    @Override
    public String getOwnerName() { return ownerName; }

    @Override
    public void performTrick() { System.out.println(name + " กระโดดผ่านห่วง!"); }
}
```

## 8. Abstract Class vs Interface: เลือกใช้แบบไหนเมื่อไร

| ปัจจัย | Abstract Class | Interface |
|---|---|---|
| Inheritance | สืบทอดได้แค่ 1 class | Implement ได้หลาย interface |
| State (fields) | มี instance field ที่ไม่ใช่ constant ได้ | มีแต่ `public static final` (constant) |
| Constructor | มีได้ | ไม่มี |
| Access Modifiers | มีได้ครบทุกระดับ | Method เป็น public โดยปริยาย (มี private method ได้ใน Java 9+) |
| ใช้เมื่อ | มี "is-a" relationship ชัดเจน และมี code ร่วมกันเยอะ | ต้องการกำหนด "capability/contract" ที่หลาย class ไม่เกี่ยวข้องกันนำไปใช้ได้ |
| ตัวอย่างจริง | `AbstractList`, `HttpServlet` | `Runnable`, `Comparable`, `List`, `Serializable` |

**หลักการตัดสินใจง่าย ๆ**:
- ถ้าความสัมพันธ์คือ **"is-a" และมี logic ร่วมกันเยอะ** ที่อยากให้ subclass ใช้ซ้ำ
  → **Abstract Class**
- ถ้าต้องการกำหนดแค่ **"ทำอะไรได้บ้าง (capability)"** โดยไม่สนใจว่า class นั้นจะ
  เป็นอะไรมาก่อน (เช่น ทั้ง `Duck` และ `Airplane` ต่างก็ "บินได้" แม้ไม่เกี่ยวข้องกัน
  เลยในเชิงลำดับชั้น) → **Interface**
- ในทางปฏิบัติสมัยใหม่ มักใช้ **interface เป็นหลัก** เพราะยืดหยุ่นกว่า (รองรับ
  multiple implementation) และใช้ abstract class เมื่อต้องการแชร์ state/logic
  ร่วมกันจริง ๆ

## 9. Functional Interface เบื้องต้น (ปูทางสู่ Lambda ใน Part 39)

**Functional Interface** คือ interface ที่มี**abstract method เพียงตัวเดียว**
(default/static method มีกี่ตัวก็ได้ ไม่นับ) — สำคัญมากเพราะเป็นพื้นฐานของ
**Lambda Expression** ที่จะเรียนเต็มรูปแบบใน Part 39

```java
@FunctionalInterface // annotation นี้ช่วยให้ compiler ตรวจสอบว่ามี abstract method แค่ 1 ตัวจริง
public interface Calculator {
    int calculate(int a, int b); // abstract method เดียวเท่านั้น
}
```

```java
public class FunctionalInterfaceDemo {
    public static void main(String[] args) {
        // แบบเดิม: implement ด้วย anonymous class (จะเรียนใน Part 19)
        Calculator addOld = new Calculator() {
            @Override
            public int calculate(int a, int b) {
                return a + b;
            }
        };

        // แบบใหม่ (Java 8+): ใช้ Lambda Expression กระชับกว่ามาก (แค่ปูพื้นตอนนี้)
        Calculator add = (a, b) -> a + b;
        Calculator multiply = (a, b) -> a * b;

        System.out.println(addOld.calculate(3, 4));   // 7
        System.out.println(add.calculate(3, 4));       // 7
        System.out.println(multiply.calculate(3, 4));  // 12
    }
}
```

Java มี functional interface สำเร็จรูปให้ใช้มากมายใน `java.util.function`
(`Function`, `Predicate`, `Consumer`, `Supplier`) ซึ่งจะเรียนอย่างละเอียดใน Part 40

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** สร้าง abstract class `Employee` ที่มี field `name` และ abstract method
`calculateSalary()` แล้วสร้าง 2 subclass: `FullTimeEmployee` (เงินเดือนคงที่)
และ `Freelancer` (เงินเดือน = ชั่วโมง x อัตราต่อชั่วโมง)

**เฉลย:**

```java
public abstract class Employee {
    protected String name;
    public Employee(String name) { this.name = name; }
    public abstract double calculateSalary();
}

public class FullTimeEmployee extends Employee {
    private double monthlySalary;
    public FullTimeEmployee(String name, double monthlySalary) {
        super(name);
        this.monthlySalary = monthlySalary;
    }
    @Override
    public double calculateSalary() { return monthlySalary; }
}

public class Freelancer extends Employee {
    private double hours, ratePerHour;
    public Freelancer(String name, double hours, double ratePerHour) {
        super(name);
        this.hours = hours;
        this.ratePerHour = ratePerHour;
    }
    @Override
    public double calculateSalary() { return hours * ratePerHour; }
}
```

**2)** สร้าง 2 interface: `Drawable` (มี method `draw()`) และ `Resizable`
(มี method `resize(double factor)`) แล้วสร้าง class `Square` ที่ implement ทั้งคู่

**เฉลย:**

```java
public interface Drawable {
    void draw();
}

public interface Resizable {
    void resize(double factor);
}

public class Square implements Drawable, Resizable {
    private double side;
    public Square(double side) { this.side = side; }

    @Override
    public void draw() { System.out.println("วาดสี่เหลี่ยมจัตุรัสด้าน " + side); }

    @Override
    public void resize(double factor) { this.side *= factor; }
}
```

**3)** อธิบายความแตกต่างระหว่าง abstract method กับ default method ใน interface
พร้อมยกตัวอย่างว่าเมื่อไรควรใช้แบบไหน

**เฉลย**: Abstract method **บังคับ**ให้ทุก class ที่ implement ต้องเขียน
implementation เอง ใช้เมื่อพฤติกรรมนั้น**แตกต่างกันไปในแต่ละ class อย่างแน่นอน**
(เช่น `calculateArea()` ที่แต่ละรูปทรงคำนวณต่างกัน) ส่วน default method **มี
implementation สำเร็จรูปให้แล้ว** ใช้เมื่อพฤติกรรมนั้น**เหมือนกันในเกือบทุก class**
หรือใช้เพื่อ**เพิ่ม method ใหม่เข้า interface เดิมโดยไม่ทำให้ class ที่ implement
อยู่แล้วพัง** (backward compatibility)

### สรุปเนื้อหา Part 16

- Abstraction ซ่อนความซับซ้อน เปิดเผยแค่ "ต้องทำอะไร" ไม่ใช่ "ทำอย่างไร"
- Abstract Class สร้าง object ตรงไม่ได้ มีทั้ง abstract และ concrete method ได้
- Interface เป็น "สัญญา" ล้วน ๆ (จนถึง Java 8 เพิ่ม default/static method ได้)
- Class implement ได้หลาย interface พร้อมกัน แต่ extends class ได้แค่ 1 ตัว
- เลือก abstract class เมื่อมี "is-a" ชัดเจนและมี code ร่วมกันเยอะ, เลือก interface
  เมื่อต้องการกำหนด capability ที่หลาย class ไม่เกี่ยวข้องกันนำไปใช้ได้
- Functional Interface (abstract method เดียว) คือรากฐานของ Lambda Expression

**จบหมวด OOP หลัก 4 เสา (Encapsulation, Inheritance, Polymorphism, Abstraction)
ครบถ้วนแล้ว!**

**ต่อไป**: [Part 17 — Static, Final และ Immutability](./part-017-static-final-immutability.md)
