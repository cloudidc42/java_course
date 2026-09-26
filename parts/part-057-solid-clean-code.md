# Part 57: SOLID Principles และ Clean Code ในทางปฏิบัติ

> ขั้นตอนที่ 561-570 ของหลักสูตร | ระดับ: สูง

## สารบัญ

1. SOLID คืออะไร ทำไมสำคัญ
2. S: Single Responsibility Principle
3. O: Open/Closed Principle
4. L: Liskov Substitution Principle
5. I: Interface Segregation Principle
6. D: Dependency Inversion Principle
7. Clean Code: การตั้งชื่อ
8. Clean Code: เมธอดและฟังก์ชัน
9. Clean Code: คอมเมนต์และการจัดรูปแบบ
10. แบบฝึกหัดและสรุป

---

## 1. SOLID คืออะไร ทำไมสำคัญ

**SOLID** คือตัวย่อของ 5 หลักการออกแบบ OOP ที่ **Robert C. Martin (Uncle
Bob)** รวบรวมไว้ เพื่อช่วยให้โค้ด**ยืดหยุ่น บำรุงรักษาง่าย และขยายต่อได้**
โดยไม่ต้องแก้ไขโค้ดเดิมที่ทำงานอยู่แล้ว — เป็นการนำหลักการ OOP ทั้ง 4 เสา
(Part 11-16) มาประยุกต์ใช้อย่างเป็นระบบ

| ตัวอักษร | ชื่อเต็ม |
|---|---|
| **S** | Single Responsibility Principle |
| **O** | Open/Closed Principle |
| **L** | Liskov Substitution Principle |
| **I** | Interface Segregation Principle |
| **D** | Dependency Inversion Principle |

## 2. S: Single Responsibility Principle

**"Class หนึ่งควรมีเหตุผลให้เปลี่ยนแปลงเพียงเหตุผลเดียว"** — แต่ละ class
ควรทำหน้าที่**เดียว**ให้ดี (ทบทวนแนวคิดจาก Part 8 เรื่องการออกแบบเมธอดที่ดี
แต่ขยายไปถึงระดับ class)

```java
// ผิดหลัก SRP: class เดียวทำหลายหน้าที่ (คำนวณ, บันทึกไฟล์, ส่งอีเมล)
public class OrderProcessorBad {
    double calculateTotal(java.util.List<Double> prices) {
        return prices.stream().mapToDouble(Double::doubleValue).sum();
    }

    void saveToDatabase(double total) { // หน้าที่เกี่ยวกับ persistence
        System.out.println("บันทึกลงฐานข้อมูล: " + total);
    }

    void sendConfirmationEmail(String email) { // หน้าที่เกี่ยวกับการแจ้งเตือน
        System.out.println("ส่งอีเมลยืนยันไปที่: " + email);
    }
    // ถ้าต้องเปลี่ยนวิธีบันทึกฐานข้อมูล หรือเปลี่ยนผู้ให้บริการอีเมล ต้องแก้ class นี้
    // ทั้งที่ทั้งสองเรื่องไม่เกี่ยวข้องกับ "การคำนวณราคา" เลย - นี่คือปัญหา
}
```

```java
// ถูกหลัก SRP: แยกความรับผิดชอบชัดเจน แต่ละ class มีเหตุผลเดียวที่จะเปลี่ยน
public class PriceCalculator {
    double calculateTotal(java.util.List<Double> prices) {
        return prices.stream().mapToDouble(Double::doubleValue).sum();
    }
}

public class OrderRepository {
    void save(double total) {
        System.out.println("บันทึกลงฐานข้อมูล: " + total);
    }
}

public class EmailNotifier {
    void sendConfirmation(String email) {
        System.out.println("ส่งอีเมลยืนยันไปที่: " + email);
    }
}
```

## 3. O: Open/Closed Principle

**"Class ควรเปิดสำหรับการขยาย (extension) แต่ปิดสำหรับการแก้ไข
(modification)"** — เพิ่ม feature ใหม่ได้โดย**ไม่แก้ไขโค้ดที่ทำงานอยู่แล้ว**
(ทบทวนตัวอย่าง Strategy Pattern จาก Part 56 ซึ่งเป็นการนำหลักการนี้มาใช้จริง)

```java
// ผิดหลัก OCP: ทุกครั้งที่เพิ่มรูปทรงใหม่ ต้องแก้ไขเมธอดนี้ (เสี่ยงทำโค้ดเดิมพัง)
public class AreaCalculatorBad {
    double calculateArea(Object shape) {
        if (shape instanceof Circle c) {
            return Math.PI * c.radius() * c.radius();
        } else if (shape instanceof Square s) {
            return s.side() * s.side();
        }
        // ถ้าเพิ่ม Triangle ต้องมาแก้ if-else chain นี้อีก - ละเมิด "closed for modification"
        return 0;
    }
    record Circle(double radius) { }
    record Square(double side) { }
}
```

```java
// ถูกหลัก OCP: ใช้ polymorphism (ทบทวนจาก Part 15-16) - เพิ่มรูปทรงใหม่โดยไม่แก้โค้ดเดิม
public interface Shape {
    double area();
}

public record Circle(double radius) implements Shape {
    public double area() { return Math.PI * radius * radius; }
}

public record Square(double side) implements Shape {
    public double area() { return side * side; }
}

// เพิ่ม Triangle ใหม่ได้เลย โดยไม่แก้ไขโค้ดที่มีอยู่แล้วแม้แต่บรรทัดเดียว
public record Triangle(double base, double height) implements Shape {
    public double area() { return base * height / 2; }
}

public class AreaCalculatorGood {
    double calculateArea(Shape shape) {
        return shape.area(); // ทำงานกับ Shape ใดก็ได้ ไม่ต้องรู้ concrete type เลย
    }
}
```

## 4. L: Liskov Substitution Principle

**"Object ของ subclass ต้องสามารถแทนที่ object ของ superclass ได้โดยไม่ทำให้
โปรแกรมพัง"** — ทบทวนจาก Part 14-15: ถ้า `B extends A` แล้ว ทุกที่ที่ใช้ `A`
ควรใช้ `B` แทนได้โดยไม่เกิดพฤติกรรมที่ผิดคาด

```java
// ตัวอย่างคลาสสิกที่ละเมิด LSP: "Square is-a Rectangle" ทางคณิตศาสตร์ แต่ไม่ใช่ในโค้ด!
public class Rectangle {
    protected double width, height;

    public void setWidth(double width) { this.width = width; }
    public void setHeight(double height) { this.height = height; }
    public double area() { return width * height; }
}

public class SquareBad extends Rectangle {
    @Override
    public void setWidth(double width) {
        this.width = width;
        this.height = width; // บังคับให้ height เท่ากับ width เสมอ (เพราะสี่เหลี่ยมจัตุรัส)
    }
    @Override
    public void setHeight(double height) {
        this.width = height; // และในทางกลับกัน - ทำให้พฤติกรรมต่างจาก Rectangle ธรรมดา!
        this.height = height;
    }
}
```

```java
public class LSPViolationDemo {
    static void testRectangle(Rectangle r) {
        r.setWidth(5);
        r.setHeight(10);
        System.out.println("พื้นที่ที่คาดหวัง: 50, พื้นที่จริง: " + r.area());
    }

    public static void main(String[] args) {
        testRectangle(new Rectangle());  // พื้นที่ที่คาดหวัง: 50, พื้นที่จริง: 50.0 (ถูกต้อง)
        testRectangle(new SquareBad());   // พื้นที่ที่คาดหวัง: 50, พื้นที่จริง: 100.0 (!) ผิดคาด!
        // SquareBad แทนที่ Rectangle ไม่ได้อย่างปลอดภัย เพราะพฤติกรรมต่างกัน - ละเมิด LSP
    }
}
```

**วิธีแก้**: ไม่ควรให้ `Square` สืบทอดจาก `Rectangle` เลย ควรทำให้ทั้งคู่
`implements Shape` (interface กลาง) แยกจากกันอย่างอิสระแทน (เหมือนตัวอย่าง
ในหัวข้อ 3)

## 5. I: Interface Segregation Principle

**"ไม่ควรบังคับให้ class implement method ที่ไม่ได้ใช้"** — ควรแยก interface
ใหญ่ให้เป็น interface เล็ก ๆ ที่เฉพาะเจาะจง (ทบทวนแนวคิด interface จาก Part
16 — "fat interface" เป็นปัญหา)

```java
// ผิดหลัก ISP: interface ใหญ่เกินไป บังคับให้ทุก class implement method ที่อาจไม่เกี่ยวข้อง
public interface WorkerBad {
    void work();
    void eat();
    void sleep();
}

public class RobotWorkerBad implements WorkerBad {
    public void work() { System.out.println("หุ่นยนต์ทำงาน"); }
    public void eat() { throw new UnsupportedOperationException("หุ่นยนต์ไม่กินอาหาร!"); } // บังคับ implement ทั้งที่ไม่เกี่ยว
    public void sleep() { throw new UnsupportedOperationException("หุ่นยนต์ไม่นอน!"); }
}
```

```java
// ถูกหลัก ISP: แยก interface เล็ก ๆ ตามความสามารถที่แท้จริง
public interface Workable { void work(); }
public interface Eatable { void eat(); }
public interface Sleepable { void sleep(); }

public class HumanWorker implements Workable, Eatable, Sleepable {
    public void work() { System.out.println("คนทำงาน"); }
    public void eat() { System.out.println("คนกินอาหาร"); }
    public void sleep() { System.out.println("คนนอนหลับ"); }
}

public class RobotWorker implements Workable { // implement แค่ที่เกี่ยวข้องจริง ๆ
    public void work() { System.out.println("หุ่นยนต์ทำงาน"); }
}
```

## 6. D: Dependency Inversion Principle

**"Module ระดับสูงไม่ควร depend on module ระดับต่ำโดยตรง ทั้งคู่ควร depend
on abstraction (interface)"** — ทบทวนปัญหา circular dependency จาก Part 20
และปูทางสู่ Dependency Injection (Part 74)

```java
// ผิดหลัก DIP: OrderService (ระดับสูง) depend on MySQLDatabase (ระดับต่ำ) โดยตรง
public class MySQLDatabase {
    void save(String data) { System.out.println("บันทึกลง MySQL: " + data); }
}

public class OrderServiceBad {
    private MySQLDatabase database = new MySQLDatabase(); // ผูกติดกับ MySQL แน่นเกินไป!

    void placeOrder(String order) {
        database.save(order);
        // ถ้าต้องเปลี่ยนเป็น PostgreSQL หรือ MongoDB ต้องแก้ไข OrderServiceBad โดยตรง
    }
}
```

```java
// ถูกหลัก DIP: ทั้งสองฝั่ง depend on interface (abstraction) กลาง ไม่ผูกติดกันโดยตรง
public interface Database {
    void save(String data);
}

public class MySQLDatabaseGood implements Database {
    public void save(String data) { System.out.println("บันทึกลง MySQL: " + data); }
}

public class MongoDatabase implements Database {
    public void save(String data) { System.out.println("บันทึกลง MongoDB: " + data); }
}

public class OrderServiceGood {
    private Database database; // depend on interface เท่านั้น ไม่รู้จัก concrete class เลย

    public OrderServiceGood(Database database) { // รับผ่าน constructor (Dependency Injection - Part 74)
        this.database = database;
    }

    void placeOrder(String order) {
        database.save(order);
    }
}
```

```java
public class DIPDemo {
    public static void main(String[] args) {
        OrderServiceGood service1 = new OrderServiceGood(new MySQLDatabaseGood());
        service1.placeOrder("Order #1");

        OrderServiceGood service2 = new OrderServiceGood(new MongoDatabase()); // สลับ database ได้ทันที
        service2.placeOrder("Order #2");
        // OrderServiceGood ไม่ต้องแก้ไขเลยไม่ว่าจะเปลี่ยน database เป็นอะไรก็ตาม
    }
}
```

## 7. Clean Code: การตั้งชื่อ

ทบทวนและขยายจาก Part 2, 8: หลักการตั้งชื่อที่ดีตามแนวคิด **"Clean Code"**
(Robert C. Martin):

```java
public class NamingCleanCodeDemo {
    // ไม่ดี: ชื่อสั้นเกินไป ไม่สื่อความหมาย
    int d; // จำนวนวันที่ผ่านไป? หรืออะไร?
    List<int[]> l; // เก็บอะไร?

    // ดี: ชื่อสื่อความหมายชัดเจน อ่านแล้วเข้าใจทันทีโดยไม่ต้องอ่าน comment
    int daysSinceRegistration;
    List<int[]> coordinatePairs;

    // ไม่ดี: ชื่อที่บอกข้อมูลผิด (บอกว่าเป็น List แต่จริงเป็น Map)
    // Map<String, Integer> userList;

    // ดี: ชื่อตรงกับสิ่งที่มันเป็นจริง
    Map<String, Integer> userIdByUsername;

    // ไม่ดี: ใช้ชื่อที่คล้ายกันเกินไป สับสนง่าย
    void processData(int a1, int a2) { }

    // ดี: ชื่อพารามิเตอร์บอกบทบาทชัดเจน
    void calculateDistance(int startPoint, int endPoint) { }
}
```

**หลักการสำคัญ**: ชื่อที่ดีควร**ตอบคำถามได้ครบ**: ทำไมมันถึงมีอยู่, มันทำอะไร,
และใช้อย่างไร — ถ้าต้องเขียน comment อธิบายว่าตัวแปรนี้คืออะไร มักหมายความว่า
ชื่อนั้นยังไม่ดีพอ

## 8. Clean Code: เมธอดและฟังก์ชัน

ทบทวนจาก Part 8:

```java
public class MethodCleanCodeDemo {
    // ไม่ดี: เมธอดทำหลายอย่างเกินไป (validate + calculate + save + notify)
    void processOrderBad(Order order) {
        if (order.getTotal() <= 0) throw new IllegalArgumentException("invalid total");
        double tax = order.getTotal() * 0.07;
        double finalTotal = order.getTotal() + tax;
        // saveToDatabase(finalTotal);
        // sendEmail(order.getCustomerEmail());
        System.out.println("รวม: " + finalTotal);
    }

    // ดี: แยกเป็นเมธอดย่อยที่ทำหน้าที่เดียว ชื่อสื่อความหมาย (ทบทวน SRP จากหัวข้อ 2)
    void processOrderGood(Order order) {
        validateOrder(order);
        double finalTotal = calculateFinalTotal(order);
        saveOrder(finalTotal);
        notifyCustomer(order);
    }

    void validateOrder(Order order) {
        if (order.getTotal() <= 0) throw new IllegalArgumentException("invalid total");
    }

    double calculateFinalTotal(Order order) {
        double tax = order.getTotal() * 0.07;
        return order.getTotal() + tax;
    }

    void saveOrder(double total) { System.out.println("บันทึก: " + total); }
    void notifyCustomer(Order order) { System.out.println("แจ้งลูกค้า"); }

    static class Order {
        double getTotal() { return 100; }
        String getCustomerEmail() { return "test@example.com"; }
    }
}
```

**กฎที่นิยม**: เมธอดควร**สั้น** (แนะนำไม่เกิน 20 บรรทัด), ทำ**สิ่งเดียว**และ
ทำให้ดี, มีระดับ**abstraction เดียวกันตลอดทั้งเมธอด** (ไม่ผสม logic ระดับสูง
กับรายละเอียดระดับต่ำในเมธอดเดียวกัน)

## 9. Clean Code: คอมเมนต์และการจัดรูปแบบ

ทบทวนจาก system prompt ของหลักสูตรนี้เอง (หลักการเขียนโค้ดที่ดี):

```java
public class CommentsCleanCodeDemo {
    // ไม่ดี: comment อธิบาย "อะไร" ที่โค้ดทำอยู่แล้ว (ซ้ำซ้อน ไม่มีประโยชน์)
    // เพิ่มค่า i ทีละ 1
    int i = 0;
    // i++;

    // ไม่ดี: comment ที่ล้าสมัย (ไม่ตรงกับโค้ดจริงแล้ว - อันตรายกว่าไม่มี comment เลย)
    // คำนวณราคารวมไม่รวม VAT
    double total = 100 * 1.07; // จริง ๆ รวม VAT ไปแล้ว! comment โกหก

    // ดี: comment อธิบาย "ทำไม" ที่ไม่ชัดเจนจากโค้ดเอง (เหตุผลเชิงธุรกิจหรือ workaround)
    // ใช้ 0.9 แทน 1.0 เพราะ third-party API มี rate limit ที่ยังไม่แน่ชัด
    // (อ้างอิง: ticket JIRA-1234)
    double safetyMargin = 0.9;
}
```

**หลักการ**: โค้ดที่ดีควร**อธิบายตัวเองได้**ผ่านชื่อตัวแปร/เมธอดที่ดี — เขียน
comment เฉพาะเมื่อมี**เหตุผลที่ไม่ชัดเจนจากโค้ดเอง** (constraint แปลก ๆ,
workaround, decision ทางธุรกิจ) ไม่ใช่อธิบายว่าโค้ดทำอะไร (ซึ่งโค้ดที่ดีบอกอยู่
แล้ว)

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** ระบุว่าโค้ดนี้ละเมิดหลักการ SOLID ข้อไหน และอธิบายวิธีแก้:

```java
class ReportGenerator {
    void generateAndEmailReport(List<Data> data) {
        String report = formatAsHtml(data);
        SmtpClient client = new SmtpClient();
        client.send(report);
    }
    String formatAsHtml(List<Data> data) { return "html..."; }
}
```

**เฉลย**: ละเมิด **Single Responsibility Principle** เพราะ class เดียวทำ
สองหน้าที่ที่ไม่เกี่ยวข้องกัน: (1) จัดรูปแบบรายงาน และ (2) ส่งอีเมล — ควรแยก
เป็น `ReportFormatter` และ `EmailSender` คนละ class และยังละเมิด
**Dependency Inversion Principle** เพราะ `ReportGenerator` สร้าง
`SmtpClient` ขึ้นมาโดยตรง (ควร depend on interface `EmailSender` แทน แล้วรับ
ผ่าน constructor)

**2)** ปรับโค้ดในหัวข้อ 3 (`AreaCalculatorBad`) ให้ปฏิบัติตาม Open/Closed
Principle โดยเพิ่ม `Pentagon` เข้าไปโดยไม่แก้ไขโค้ดเดิม

**เฉลย**: (ดูโค้ดในหัวข้อ 3 — เพียงสร้าง `record Pentagon(...) implements
Shape` ใหม่ พร้อม override `area()` โดยไม่ต้องแก้ไข `AreaCalculatorGood` หรือ
`Shape` interface เลย)

**3)** อธิบายว่าทำไม `SquareBad extends Rectangle` ละเมิด Liskov
Substitution Principle ทั้งที่ในทางคณิตศาสตร์ สี่เหลี่ยมจัตุรัสก็เป็น
สี่เหลี่ยมผืนผ้าชนิดหนึ่ง

**เฉลย**: LSP ไม่ได้ตัดสินจากความสัมพันธ์ทางแนวคิด (conceptual "is-a") แต่
ตัดสินจาก**พฤติกรรม**ในโค้ด — `Rectangle` มี invariant (คุณสมบัติที่คงที่)
ว่า `setWidth()` และ `setHeight()` เป็นอิสระจากกัน แต่ `SquareBad` ละเมิด
invariant นี้ (การ `setWidth()` ไปกระทบ `height` ด้วย) ทำให้โค้ดที่ทดสอบผ่าน
กับ `Rectangle` (เช่น `testRectangle()` ในหัวข้อ 4) **ล้มเหลวเมื่อใช้กับ
`SquareBad`** ซึ่งขัดกับหลักการที่ subclass ต้องแทนที่ superclass ได้โดย
ไม่ทำให้โปรแกรมพัง — นี่คือเหตุผลที่ "is-a" ทางภาษาธรรมชาติไม่ใช่เกณฑ์ที่
เพียงพอสำหรับการออกแบบ inheritance ในโค้ดจริง

### สรุปเนื้อหา Part 57

- **S**RP: class ควรมีเหตุผลให้เปลี่ยนแปลงเพียงเหตุผลเดียว
- **O**CP: เปิดสำหรับขยาย ปิดสำหรับแก้ไข — ใช้ polymorphism/interface แทน
  if-else chain
- **L**SP: subclass ต้องแทนที่ superclass ได้โดยไม่ทำให้โปรแกรมพัง
- **I**SP: แยก interface ใหญ่เป็นเล็ก ๆ เฉพาะเจาะจง ไม่บังคับ implement สิ่ง
  ที่ไม่เกี่ยวข้อง
- **D**IP: depend on abstraction (interface) ไม่ใช่ concrete class โดยตรง
- Clean Code: ตั้งชื่อสื่อความหมาย, เมธอดสั้นและทำสิ่งเดียว, comment อธิบาย
  "ทำไม" ไม่ใช่ "อะไร"

**ต่อไป**: [Part 58 — Unit Testing ด้วย JUnit 5](./part-058-junit5.md)
