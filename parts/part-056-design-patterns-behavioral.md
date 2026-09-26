# Part 56: Design Patterns: Behavioral (Observer, Strategy, Command, Template Method, State)

> ขั้นตอนที่ 551-560 ของหลักสูตร | ระดับ: สูง (เริ่มหมวด Java ขั้นสูงและ Tooling)

## สารบัญ

1. Behavioral Pattern คืออะไร
2. Observer Pattern
3. Strategy Pattern
4. Command Pattern
5. Template Method Pattern
6. State Pattern
7. Chain of Responsibility Pattern
8. Iterator Pattern (ทบทวนจาก Part 27)
9. ตัวอย่างจริงใน Java Standard Library
10. แบบฝึกหัดและสรุปหมวด Design Patterns

---

## 1. Behavioral Pattern คืออะไร

**Behavioral Pattern** เกี่ยวข้องกับ**การสื่อสารและแบ่งความรับผิดชอบระหว่าง
object** — เน้นที่ "ใครทำอะไร" และ "สื่อสารกันอย่างไร" มากกว่าโครงสร้าง
(Structural — Part 55) หรือการสร้าง (Creational — Part 54)

## 2. Observer Pattern

**Observer** ให้ object หนึ่ง (**Subject**) **แจ้งเตือน**object อื่นหลายตัว
(**Observers**) โดยอัตโนมัติเมื่อสถานะเปลี่ยนแปลง — เป็นพื้นฐานของ **Event-
Driven Programming** และ GUI event handling

```java
import java.util.ArrayList;
import java.util.List;

public interface Observer {
    void onUpdate(double temperature);
}

public class WeatherStation { // Subject
    private List<Observer> observers = new ArrayList<>();
    private double temperature;

    public void subscribe(Observer observer) {
        observers.add(observer);
    }

    public void unsubscribe(Observer observer) {
        observers.remove(observer);
    }

    public void setTemperature(double temperature) {
        this.temperature = temperature;
        notifyObservers(); // แจ้งเตือนทุก observer โดยอัตโนมัติเมื่อสถานะเปลี่ยน
    }

    private void notifyObservers() {
        for (Observer observer : observers) {
            observer.onUpdate(temperature);
        }
    }
}

public class PhoneDisplay implements Observer {
    @Override
    public void onUpdate(double temperature) {
        System.out.println("[มือถือ] อุณหภูมิปัจจุบัน: " + temperature + "°C");
    }
}

public class WebDashboard implements Observer {
    @Override
    public void onUpdate(double temperature) {
        System.out.println("[Dashboard] อัปเดตกราฟ: " + temperature + "°C");
    }
}
```

```java
public class ObserverDemo {
    public static void main(String[] args) {
        WeatherStation station = new WeatherStation();
        station.subscribe(new PhoneDisplay());
        station.subscribe(new WebDashboard());

        station.setTemperature(28.5); // ทั้งสอง observer ได้รับแจ้งเตือนพร้อมกันอัตโนมัติ
        // [มือถือ] อุณหภูมิปัจจุบัน: 28.5°C
        // [Dashboard] อัปเดตกราฟ: 28.5°C
    }
}
```

**ตัวอย่างจริงใน Java**: `PropertyChangeListener` (Swing), GUI event
listener ทั้งหมด, และหลักการเบื้องหลัง **Reactive Programming** (Part 102)

## 3. Strategy Pattern

**Strategy** ให้ **algorithm ที่แตกต่างกันสับเปลี่ยนกันได้ตอน runtime** โดย
ห่อแต่ละ algorithm ไว้ใน object แยกกันที่ implement interface เดียวกัน (ทบทวน
polymorphism จาก Part 15 — ใกล้เคียงกับตัวอย่าง `Comparator` จาก Part 27)

```java
public interface DiscountStrategy {
    double applyDiscount(double price);
}

public class NoDiscount implements DiscountStrategy {
    public double applyDiscount(double price) { return price; }
}

public class PercentageDiscount implements DiscountStrategy {
    private double percentage;
    public PercentageDiscount(double percentage) { this.percentage = percentage; }
    public double applyDiscount(double price) { return price * (1 - percentage / 100); }
}

public class FixedAmountDiscount implements DiscountStrategy {
    private double amount;
    public FixedAmountDiscount(double amount) { this.amount = amount; }
    public double applyDiscount(double price) { return Math.max(0, price - amount); }
}

public class ShoppingCart {
    private DiscountStrategy discountStrategy;

    public ShoppingCart(DiscountStrategy discountStrategy) {
        this.discountStrategy = discountStrategy;
    }

    public void setDiscountStrategy(DiscountStrategy strategy) { // เปลี่ยน algorithm ได้ตอน runtime
        this.discountStrategy = strategy;
    }

    public double checkout(double totalPrice) {
        return discountStrategy.applyDiscount(totalPrice);
    }
}
```

```java
public class StrategyDemo {
    public static void main(String[] args) {
        ShoppingCart cart = new ShoppingCart(new NoDiscount());
        System.out.println(cart.checkout(1000)); // 1000.0

        cart.setDiscountStrategy(new PercentageDiscount(10));
        System.out.println(cart.checkout(1000)); // 900.0

        cart.setDiscountStrategy(new FixedAmountDiscount(150));
        System.out.println(cart.checkout(1000)); // 850.0
    }
}
```

**เปรียบเทียบกับ if-else**: ไม่ใช้ Strategy Pattern อาจเขียนเป็น if-else
ยาว ๆ ในเมธอดเดียว — Strategy Pattern ทำให้เพิ่ม algorithm ใหม่ได้โดยไม่ต้อง
แก้ไข `ShoppingCart` เลย (สอดคล้องกับหลักการ **Open/Closed Principle** ที่จะ
เรียนใน Part 57)

## 4. Command Pattern

**Command** ห่อ**คำขอ (request)** ให้เป็น object — ทำให้ส่งผ่าน, เก็บใน queue,
หรือ **undo/redo** ได้ (เป็นหลักการเบื้องหลังปุ่ม Undo ใน text editor เกือบ
ทุกตัว)

```java
public interface Command {
    void execute();
    void undo();
}

public class Light {
    boolean isOn = false;
    void turnOn() { isOn = true; System.out.println("ไฟเปิด"); }
    void turnOff() { isOn = false; System.out.println("ไฟปิด"); }
}

public class TurnOnCommand implements Command {
    private Light light;
    public TurnOnCommand(Light light) { this.light = light; }
    public void execute() { light.turnOn(); }
    public void undo() { light.turnOff(); }
}

public class RemoteControl {
    private java.util.Deque<Command> history = new java.util.ArrayDeque<>(); // ทบทวน Deque จาก Part 25

    public void pressButton(Command command) {
        command.execute();
        history.push(command); // เก็บ command ที่ทำไปแล้วไว้ใน stack เพื่อ undo ได้
    }

    public void pressUndo() {
        if (!history.isEmpty()) {
            Command lastCommand = history.pop();
            lastCommand.undo();
        }
    }
}
```

```java
public class CommandDemo {
    public static void main(String[] args) {
        Light light = new Light();
        RemoteControl remote = new RemoteControl();

        remote.pressButton(new TurnOnCommand(light)); // ไฟเปิด
        remote.pressUndo();                              // ไฟปิด (undo ทำงานได้!)
    }
}
```

## 5. Template Method Pattern

**Template Method** กำหนด**โครงร่างของ algorithm**ใน abstract class (ทบทวน
จาก Part 16) โดยให้ subclass **override เฉพาะบางขั้นตอน**ได้ ในขณะที่ลำดับ
การทำงานโดยรวมยังคงที่เสมอ

```java
public abstract class DataProcessor {
    // Template method: กำหนดลำดับขั้นตอนตายตัว (final ป้องกันไม่ให้ subclass เปลี่ยนลำดับ - ทบทวน Part 14)
    public final void process() {
        readData();
        validateData();
        transformData();
        saveData();
    }

    protected void readData() { System.out.println("อ่านข้อมูล (ขั้นตอนมาตรฐาน)"); }
    protected void saveData() { System.out.println("บันทึกข้อมูล (ขั้นตอนมาตรฐาน)"); }

    protected abstract void validateData(); // ต้อง implement เอง (แตกต่างกันไปตามชนิดข้อมูล)
    protected abstract void transformData(); // ต้อง implement เอง
}

public class CsvDataProcessor extends DataProcessor {
    @Override
    protected void validateData() { System.out.println("ตรวจสอบรูปแบบ CSV"); }
    @Override
    protected void transformData() { System.out.println("แปลง CSV เป็น object"); }
}

public class JsonDataProcessor extends DataProcessor {
    @Override
    protected void validateData() { System.out.println("ตรวจสอบรูปแบบ JSON"); }
    @Override
    protected void transformData() { System.out.println("แปลง JSON เป็น object"); }
}
```

```java
public class TemplateMethodDemo {
    public static void main(String[] args) {
        DataProcessor csv = new CsvDataProcessor();
        csv.process(); // ลำดับขั้นตอนเหมือนกันเสมอ แต่ validate/transform ต่างกันตามชนิดข้อมูล

        System.out.println("---");
        DataProcessor json = new JsonDataProcessor();
        json.process();
    }
}
```

## 6. State Pattern

**State** ให้ object**เปลี่ยนพฤติกรรมตามสถานะภายใน** — คล้าย State Machine
(finite state machine) แต่ implement ด้วย OOP โดยแต่ละสถานะเป็น class แยกกัน
(ทบทวน polymorphism จาก Part 15)

```java
public interface OrderState {
    void next(OrderContext context);
    String getStatus();
}

public class PendingState implements OrderState {
    public void next(OrderContext context) { context.setState(new ShippedState()); }
    public String getStatus() { return "รอดำเนินการ"; }
}

public class ShippedState implements OrderState {
    public void next(OrderContext context) { context.setState(new DeliveredState()); }
    public String getStatus() { return "จัดส่งแล้ว"; }
}

public class DeliveredState implements OrderState {
    public void next(OrderContext context) {
        System.out.println("คำสั่งซื้อจบสมบูรณ์แล้ว ไม่มีสถานะถัดไป");
    }
    public String getStatus() { return "ส่งถึงแล้ว"; }
}

public class OrderContext {
    private OrderState state = new PendingState(); // เริ่มที่สถานะแรก

    public void setState(OrderState state) { this.state = state; }
    public void nextStatus() { state.next(this); }
    public String getStatus() { return state.getStatus(); }
}
```

```java
public class StateDemo {
    public static void main(String[] args) {
        OrderContext order = new OrderContext();
        System.out.println(order.getStatus()); // รอดำเนินการ

        order.nextStatus();
        System.out.println(order.getStatus()); // จัดส่งแล้ว

        order.nextStatus();
        System.out.println(order.getStatus()); // ส่งถึงแล้ว
    }
}
```

**เปรียบเทียบกับ enum จาก Part 18**: สำหรับ state machine ที่ไม่ซับซ้อนมาก
การใช้ `enum` กับ abstract method (ทบทวนตัวอย่าง `Operation` จาก Part 18)
มักเพียงพอและง่ายกว่า — State Pattern (แบบ class เต็มรูปแบบ) เหมาะกับกรณีที่
แต่ละสถานะมี logic ซับซ้อนมากจนควรแยกเป็น class ของตัวเอง

## 7. Chain of Responsibility Pattern

**Chain of Responsibility** ส่งคำขอผ่าน**ลูกโซ่ของ handler** โดยแต่ละ
handler ตัดสินใจว่าจะจัดการเองหรือส่งต่อให้ handler ถัดไป

```java
public abstract class SupportHandler {
    protected SupportHandler next;

    public void setNext(SupportHandler next) {
        this.next = next;
    }

    public abstract void handle(int severity);
}

public class Level1Support extends SupportHandler {
    public void handle(int severity) {
        if (severity <= 2) {
            System.out.println("Level 1 Support จัดการเรื่องนี้");
        } else if (next != null) {
            next.handle(severity); // ส่งต่อให้ handler ถัดไปในลูกโซ่
        }
    }
}

public class Level2Support extends SupportHandler {
    public void handle(int severity) {
        if (severity <= 5) {
            System.out.println("Level 2 Support จัดการเรื่องนี้");
        } else if (next != null) {
            next.handle(severity);
        }
    }
}

public class ManagerSupport extends SupportHandler {
    public void handle(int severity) {
        System.out.println("ผู้จัดการจัดการเรื่องนี้ (ระดับความรุนแรง " + severity + ")");
    }
}
```

```java
public class ChainOfResponsibilityDemo {
    public static void main(String[] args) {
        SupportHandler level1 = new Level1Support();
        SupportHandler level2 = new Level2Support();
        SupportHandler manager = new ManagerSupport();

        level1.setNext(level2);
        level2.setNext(manager); // สร้างลูกโซ่: level1 -> level2 -> manager

        level1.handle(1);  // Level 1 Support จัดการเรื่องนี้
        level1.handle(4);  // Level 2 Support จัดการเรื่องนี้
        level1.handle(10); // ผู้จัดการจัดการเรื่องนี้
    }
}
```

## 8. Iterator Pattern (ทบทวนจาก Part 27)

**Iterator Pattern** ที่เรียนไปแล้วใน Part 27 (`Iterable`/`Iterator`) ก็คือ
Behavioral Pattern ตัวหนึ่ง — ให้วิธี**เข้าถึง element ของ collection ทีละตัว
โดยไม่เปิดเผยโครงสร้างภายใน** ของ collection นั้น

## 9. ตัวอย่างจริงใน Java Standard Library

```java
public class RealWorldBehavioralPatternsDemo {
    public static void main(String[] args) {
        // Strategy Pattern: Comparator เป็น Strategy สำหรับการเรียงลำดับ (ทบทวนจาก Part 27)
        java.util.List<String> names = new java.util.ArrayList<>(java.util.List.of("Bob", "Alice"));
        names.sort((a, b) -> a.compareTo(b)); // "strategy" การเรียงที่สับเปลี่ยนได้

        // Observer Pattern: ActionListener ใน Swing, PropertyChangeListener
        // Runnable ที่ส่งเข้า Thread ก็ถือเป็นรูปแบบหนึ่งของ Command Pattern

        // Template Method: AbstractList, AbstractMap ใน Collections Framework
        // กำหนดโครงร่างเมธอดหลัก (เช่น equals, toString) ให้ subclass override บางส่วนเท่านั้น
    }
}
```

## 10. แบบฝึกหัดและสรุปหมวด Design Patterns

### แบบฝึกหัด

**1)** สร้าง Observer Pattern สำหรับระบบแจ้งเตือนสต็อกสินค้าใกล้หมด (มี
`StockObserver` แจ้งเตือนทาง email และ SMS)

**เฉลย:**

```java
import java.util.ArrayList;
import java.util.List;

interface StockObserver {
    void onLowStock(String product, int quantity);
}

class EmailAlert implements StockObserver {
    public void onLowStock(String product, int quantity) {
        System.out.println("ส่งอีเมลแจ้งเตือน: " + product + " เหลือ " + quantity + " ชิ้น");
    }
}

class Inventory {
    private List<StockObserver> observers = new ArrayList<>();
    public void subscribe(StockObserver o) { observers.add(o); }

    public void updateStock(String product, int quantity) {
        if (quantity < 10) {
            for (StockObserver o : observers) o.onLowStock(product, quantity);
        }
    }
}
```

**2)** ใช้ Strategy Pattern สร้างระบบคำนวณค่าจัดส่งที่มี 3 วิธี: Standard,
Express, SameDay

**เฉลย:**

```java
interface ShippingStrategy {
    double calculate(double weight);
}

class StandardShipping implements ShippingStrategy {
    public double calculate(double weight) { return weight * 10; }
}

class ExpressShipping implements ShippingStrategy {
    public double calculate(double weight) { return weight * 20 + 50; }
}
```

**3)** อธิบายว่า Strategy Pattern ต่างจาก Template Method Pattern อย่างไร

**เฉลย**: Strategy Pattern ใช้ **composition** — object หลักถือ reference
ไปยัง strategy object และ**เรียกทั้ง algorithm ทั้งชุดผ่าน interface เดียว**
(สับเปลี่ยน strategy ได้อย่างอิสระตอน runtime, ทุก strategy เป็น class แยก
กันโดยสมบูรณ์) ในขณะที่ Template Method ใช้ **inheritance** — abstract class
กำหนด**โครงร่างของ algorithm ที่ตายตัว** (ลำดับขั้นตอนคงที่) แล้วให้ subclass
override**เฉพาะบางขั้นตอนย่อย**เท่านั้น (ไม่ใช่ algorithm ทั้งชุด) พูดง่าย ๆ
Strategy คือ "เปลี่ยน algorithm ทั้งหมด" ส่วน Template Method คือ "เปลี่ยน
บางส่วนของ algorithm ที่มีโครงร่างเดียวกัน"

### สรุปเนื้อหา Part 56 และหมวด Design Patterns (Part 54-56)

- Observer แจ้งเตือนหลาย object โดยอัตโนมัติเมื่อสถานะเปลี่ยน (พื้นฐานของ
  event-driven programming)
- Strategy สับเปลี่ยน algorithm ได้ตอน runtime ผ่าน composition
- Command ห่อคำขอเป็น object ทำให้ queue/undo/redo ได้
- Template Method กำหนดโครงร่าง algorithm ตายตัว ให้ subclass override บาง
  ขั้นตอนผ่าน inheritance
- State เปลี่ยนพฤติกรรมตามสถานะภายใน แต่ละสถานะเป็น class แยกกัน
- Chain of Responsibility ส่งคำขอผ่านลูกโซ่ของ handler จนกว่าจะมีตัวจัดการ

**จบหมวด Design Patterns ทั้ง 3 กลุ่ม (Creational, Structural, Behavioral)
อย่างสมบูรณ์! Part 57 จะเชื่อมโยงเข้ากับหลักการออกแบบระดับสูงกว่า (SOLID)**

**ต่อไป**: [Part 57 — SOLID Principles และ Clean Code](./part-057-solid-clean-code.md)
