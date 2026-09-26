# Part 54: Design Patterns: Creational (Singleton, Factory, Abstract Factory, Builder)

> ขั้นตอนที่ 531-540 ของหลักสูตร | ระดับ: สูง

## สารบัญ

1. Design Pattern คืออะไร ทำไมต้องเรียน
2. Singleton Pattern
3. Singleton แบบ Thread-safe
4. Enum Singleton (วิธีที่แนะนำที่สุด)
5. Factory Method Pattern
6. Abstract Factory Pattern
7. Builder Pattern (ทบทวนและขยายจาก Part 12)
8. Prototype Pattern
9. เมื่อไรควรใช้ Pattern ไหน
10. แบบฝึกหัดและสรุป

---

## 1. Design Pattern คืออะไร ทำไมต้องเรียน

**Design Pattern** คือ**วิธีแก้ปัญหาที่พบซ้ำ ๆ ในการออกแบบซอฟต์แวร์** ที่ถูก
กลั่นกรองและตกผลึกจากประสบการณ์ของนักพัฒนาจำนวนมาก — หนังสือ **"Design
Patterns: Elements of Reusable Object-Oriented Software"** (1994) โดย Gang
of Four (GoF) เป็นจุดเริ่มต้นที่ทำให้แนวคิดนี้เป็นที่รู้จักอย่างกว้างขวาง

Design Pattern แบ่งเป็น 3 กลุ่มหลัก:
- **Creational** (การสร้าง object): Singleton, Factory, Builder, Prototype
- **Structural** (การจัดโครงสร้าง object): Adapter, Decorator, Facade, Proxy
  (Part 55)
- **Behavioral** (พฤติกรรมและการสื่อสารระหว่าง object): Observer, Strategy,
  Command (Part 56)

**ประโยชน์**: ให้ **"ภาษากลาง"** ระหว่างนักพัฒนา (พูดว่า "ใช้ Factory Pattern"
สื่อสารได้ทันทีโดยไม่ต้องอธิบายรายละเอียด), แก้ปัญหาที่พิสูจน์แล้วว่าใช้ได้ผล
ดี ไม่ต้อง "คิดค้นใหม่" ทุกครั้ง

## 2. Singleton Pattern

**Singleton** รับประกันว่า class มี**instance เดียวเท่านั้นตลอดทั้งโปรแกรม**
และมีจุดเข้าถึง (access point) ที่เป็นสากล — ทบทวนจาก Part 13 ที่แนะนำ
`private constructor` เพื่อป้องกันการสร้าง object จากภายนอก

```java
public class DatabaseConnection {
    private static DatabaseConnection instance; // static field เก็บ instance เดียวที่มีอยู่
    private String connectionString;

    private DatabaseConnection() { // private constructor: สร้างจากภายนอกไม่ได้เลย
        connectionString = "jdbc:mysql://localhost/mydb";
        System.out.println("สร้างการเชื่อมต่อฐานข้อมูล (ทำครั้งเดียวเท่านั้น)");
    }

    public static DatabaseConnection getInstance() { // จุดเข้าถึงสากล
        if (instance == null) {
            instance = new DatabaseConnection(); // สร้างครั้งแรกที่มีการเรียกใช้เท่านั้น (lazy initialization)
        }
        return instance;
    }

    public void query(String sql) {
        System.out.println("รันคำสั่ง: " + sql);
    }
}
```

```java
public class SingletonDemo {
    public static void main(String[] args) {
        DatabaseConnection conn1 = DatabaseConnection.getInstance();
        DatabaseConnection conn2 = DatabaseConnection.getInstance();

        System.out.println(conn1 == conn2); // true - เป็น object เดียวกันจริง ๆ
        // "สร้างการเชื่อมต่อฐานข้อมูล" จะพิมพ์แค่ครั้งเดียวเท่านั้น (ตอนเรียก getInstance() ครั้งแรก)
    }
}
```

## 3. Singleton แบบ Thread-safe

Singleton แบบข้างบน**ไม่ thread-safe** (ทบทวนปัญหา race condition จาก Part
46) — หลาย thread เรียก `getInstance()` พร้อมกันครั้งแรกอาจสร้าง instance
มากกว่า 1 ตัว

```java
public class ThreadSafeSingleton {
    private static volatile ThreadSafeSingleton instance; // volatile สำคัญมาก (ทบทวนจาก Part 47)

    private ThreadSafeSingleton() { }

    public static ThreadSafeSingleton getInstance() {
        if (instance == null) { // เช็คครั้งแรก (ไม่ล็อค - เร็วกว่าในกรณีปกติที่ instance สร้างแล้ว)
            synchronized (ThreadSafeSingleton.class) {
                if (instance == null) { // เช็คซ้ำภายใน lock (ป้องกัน race condition ตอนสร้างครั้งแรก)
                    instance = new ThreadSafeSingleton();
                }
            }
        }
        return instance;
    }
}
```

**รูปแบบนี้เรียกว่า "Double-Checked Locking"** — เช็คสองชั้นเพื่อประสิทธิภาพ
(หลีกเลี่ยงการ synchronized ทุกครั้งที่เรียก `getInstance()` หลังจากสร้าง
instance แล้ว) `volatile` จำเป็นเพื่อป้องกันปัญหา**instruction reordering**
ที่อาจทำให้ thread อื่นเห็น instance ที่ยัง initialize ไม่สมบูรณ์

**ทางเลือกที่ง่ายกว่า**: **Initialization-on-demand holder** (ใช้ static
nested class — ทบทวนจาก Part 19) ปลอดภัยจาก thread โดยธรรมชาติ เพราะ JVM
รับประกันว่า static initializer ของ class จะรันแค่ครั้งเดียวและ thread-safe
โดยกำเนิด:

```java
public class HolderSingleton {
    private HolderSingleton() { }

    private static class Holder { // static nested class จะไม่ถูกโหลดจนกว่าจะถูกอ้างอิงครั้งแรก
        static final HolderSingleton INSTANCE = new HolderSingleton();
    }

    public static HolderSingleton getInstance() {
        return Holder.INSTANCE; // lazy + thread-safe โดยธรรมชาติของ JVM classloading
    }
}
```

## 4. Enum Singleton (วิธีที่แนะนำที่สุด)

**Joshua Bloch** (ผู้เขียน Effective Java) แนะนำว่า **enum คือวิธี Singleton
ที่ปลอดภัยที่สุด** เพราะ JVM รับประกันว่าแต่ละค่า enum มีเพียง instance เดียว
เท่านั้นเสมอ (ทบทวนจาก Part 18) — ปลอดภัยจาก thread โดยธรรมชาติ และป้องกัน
การสร้าง instance ซ้ำผ่าน reflection หรือ deserialization (ทบทวนความเสี่ยง
จาก Part 38, 53) ได้ดีกว่าวิธีอื่นทั้งหมด

```java
public enum ConfigManager {
    INSTANCE; // enum มีค่าเดียว = มี instance เดียวเสมอ รับประกันโดย JVM

    private String appName = "MyApplication";

    public String getAppName() {
        return appName;
    }

    public void setAppName(String appName) {
        this.appName = appName;
    }
}
```

```java
public class EnumSingletonDemo {
    public static void main(String[] args) {
        ConfigManager.INSTANCE.setAppName("Java Course App");
        System.out.println(ConfigManager.INSTANCE.getAppName());

        System.out.println(ConfigManager.INSTANCE == ConfigManager.INSTANCE); // true เสมอ
    }
}
```

**ข้อควรพิจารณา**: Singleton Pattern ถูกวิจารณ์บ่อยว่าเป็น**"anti-pattern"**
ในบางกรณี เพราะทำให้ testing ยาก (global state ที่แชร์ระหว่าง test) และ
สร้าง hidden dependency — ในโค้ดสมัยใหม่ (โดยเฉพาะกับ Spring — Part 74) มัก
ใช้ **Dependency Injection** แทน ซึ่งให้ผลลัพธ์คล้าย Singleton (bean scope
`singleton` เป็นค่า default) แต่ testing และจัดการ lifecycle ได้ดีกว่ามาก

## 5. Factory Method Pattern

**Factory Method** ย้าย**ตรรกะการสร้าง object ที่ซับซ้อน**ไปไว้ใน method
เฉพาะ แทนที่จะให้ผู้เรียกใช้เรียก `new` ตรง ๆ — เปิดทางให้เปลี่ยนชนิดของ object
ที่สร้างได้ในอนาคตโดยไม่กระทบผู้เรียกใช้ (ทบทวน polymorphism จาก Part 15)

```java
public interface Notification {
    void send(String message);
}

public class EmailNotification implements Notification {
    @Override
    public void send(String message) {
        System.out.println("ส่งอีเมล: " + message);
    }
}

public class SmsNotification implements Notification {
    @Override
    public void send(String message) {
        System.out.println("ส่ง SMS: " + message);
    }
}

public class NotificationFactory {
    public static Notification create(String type) { // Factory Method
        return switch (type) { // ทบทวน switch expression จาก Part 5
            case "email" -> new EmailNotification();
            case "sms" -> new SmsNotification();
            default -> throw new IllegalArgumentException("ไม่รู้จักประเภท: " + type);
        };
    }
}
```

```java
public class FactoryMethodDemo {
    public static void main(String[] args) {
        Notification notification = NotificationFactory.create("email");
        notification.send("สวัสดี!"); // ผู้เรียกไม่ต้องรู้เลยว่า class จริงคือ EmailNotification

        // ในอนาคตถ้าเพิ่ม PushNotification ใหม่ แก้แค่ NotificationFactory
        // โค้ดที่เรียกใช้ทั้งหมดไม่ต้องแก้ไขอะไรเลย (Open/Closed Principle - จะเรียนใน Part 56)
    }
}
```

## 6. Abstract Factory Pattern

**Abstract Factory** คือ "โรงงานของโรงงาน" — สร้าง**กลุ่มของ object ที่
เกี่ยวข้องกัน (family)** โดยไม่ต้องระบุ concrete class ตรง ๆ เหมาะกับกรณีที่
ต้อง**สลับ theme/platform ทั้งชุด**พร้อมกัน

```java
// Family ของ UI components
public interface Button { void render(); }
public interface Checkbox { void render(); }

// Concrete implementations สำหรับ theme "Light"
public class LightButton implements Button {
    public void render() { System.out.println("ปุ่มธีมสว่าง"); }
}
public class LightCheckbox implements Checkbox {
    public void render() { System.out.println("checkbox ธีมสว่าง"); }
}

// Concrete implementations สำหรับ theme "Dark"
public class DarkButton implements Button {
    public void render() { System.out.println("ปุ่มธีมมืด"); }
}
public class DarkCheckbox implements Checkbox {
    public void render() { System.out.println("checkbox ธีมมืด"); }
}

// Abstract Factory
public interface UIFactory {
    Button createButton();
    Checkbox createCheckbox();
}

public class LightUIFactory implements UIFactory {
    public Button createButton() { return new LightButton(); }
    public Checkbox createCheckbox() { return new LightCheckbox(); }
}

public class DarkUIFactory implements UIFactory {
    public Button createButton() { return new DarkButton(); }
    public Checkbox createCheckbox() { return new DarkCheckbox(); }
}
```

```java
public class AbstractFactoryDemo {
    static void renderUI(UIFactory factory) { // ไม่รู้เลยว่าเป็น theme ไหน แค่เรียกผ่าน interface
        Button button = factory.createButton();
        Checkbox checkbox = factory.createCheckbox();
        button.render();
        checkbox.render();
    }

    public static void main(String[] args) {
        boolean darkModeEnabled = true;
        UIFactory factory = darkModeEnabled ? new DarkUIFactory() : new LightUIFactory();

        renderUI(factory); // สร้าง button และ checkbox ที่เข้าธีมเดียวกันเสมอ (การันตี consistency)
    }
}
```

## 7. Builder Pattern (ทบทวนและขยายจาก Part 12)

ทบทวนจาก Part 12: Builder Pattern แก้ปัญหา constructor ที่มีพารามิเตอร์เยอะ
เกินไป — Part นี้จะแสดงเวอร์ชันที่สมบูรณ์ขึ้นพร้อม validation:

```java
public class HttpRequest {
    private final String url;      // จำเป็น
    private final String method;    // จำเป็น
    private final String body;       // optional
    private final int timeout;        // optional (มีค่า default)

    private HttpRequest(Builder builder) {
        this.url = builder.url;
        this.method = builder.method;
        this.body = builder.body;
        this.timeout = builder.timeout;
    }

    public static class Builder {
        private final String url;
        private final String method;
        private String body = "";
        private int timeout = 30;

        public Builder(String url, String method) { // ค่าที่จำเป็นต้องกำหนดตอนสร้าง Builder
            this.url = url;
            this.method = method;
        }

        public Builder body(String body) {
            this.body = body;
            return this; // method chaining (ทบทวนจาก Part 12)
        }

        public Builder timeout(int timeout) {
            if (timeout <= 0) throw new IllegalArgumentException("timeout ต้องมากกว่า 0");
            this.timeout = timeout;
            return this;
        }

        public HttpRequest build() {
            return new HttpRequest(this);
        }
    }

    @Override
    public String toString() {
        return method + " " + url + " (timeout=" + timeout + "s, body=" + body + ")";
    }
}
```

```java
public class BuilderPatternDemo {
    public static void main(String[] args) {
        HttpRequest request = new HttpRequest.Builder("https://api.example.com/users", "POST")
                .body("{\"name\":\"Alice\"}")
                .timeout(60)
                .build();

        System.out.println(request);
        // POST https://api.example.com/users (timeout=60s, body={"name":"Alice"})
    }
}
```

## 8. Prototype Pattern

**Prototype** สร้าง object ใหม่โดย**คัดลอกจาก object ที่มีอยู่แล้ว** (ทบทวน
copy constructor จาก Part 12) เหมาะเมื่อการสร้าง object ใหม่จาก scratch มี
cost สูง แต่การคัดลอกจาก object ที่มีอยู่แล้วเร็วกว่า

```java
public class Document implements Cloneable {
    String title;
    java.util.List<String> paragraphs;

    public Document(String title) {
        this.title = title;
        this.paragraphs = new java.util.ArrayList<>();
    }

    @Override
    public Document clone() {
        try {
            Document cloned = (Document) super.clone(); // shallow copy จาก Object.clone()
            cloned.paragraphs = new java.util.ArrayList<>(this.paragraphs); // deep copy สำหรับ mutable field
            return cloned;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError(e); // ไม่มีทางเกิดขึ้นเพราะ implement Cloneable แล้ว
        }
    }
}
```

```java
public class PrototypeDemo {
    public static void main(String[] args) {
        Document template = new Document("Template");
        template.paragraphs.add("เนื้อหามาตรฐาน");

        Document copy1 = template.clone();
        copy1.title = "เอกสาร 1";
        copy1.paragraphs.add("เนื้อหาเฉพาะเอกสาร 1");

        System.out.println(template.paragraphs);  // [เนื้อหามาตรฐาน] (ไม่ถูกกระทบ - deep copy ทำงานถูกต้อง)
        System.out.println(copy1.paragraphs);       // [เนื้อหามาตรฐาน, เนื้อหาเฉพาะเอกสาร 1]
    }
}
```

**ข้อควรระวัง**: `Object.clone()` (จาก `Cloneable` interface) มีชื่อเสียงไม่
ดีในวงการ Java เพราะ design ที่แปลกและเสี่ยง bug ง่าย — ในโค้ดสมัยใหม่มักใช้
**copy constructor** หรือ **static factory method** (`copyOf()`) แทน

## 9. เมื่อไรควรใช้ Pattern ไหน

| Pattern | ใช้เมื่อ |
|---|---|
| **Singleton** | ต้องการ instance เดียวทั่วทั้งแอปพลิเคชัน (config, connection pool) — แต่ระวังปัญหา testing/hidden dependency |
| **Factory Method** | การสร้าง object มี logic ตัดสินใจซับซ้อน (เลือก subclass ตามเงื่อนไข) |
| **Abstract Factory** | ต้องสร้าง object หลายตัวที่เกี่ยวข้องกัน (family) ให้สอดคล้องกันเสมอ |
| **Builder** | Object มีพารามิเตอร์จำนวนมาก หลายตัวเป็น optional |
| **Prototype** | การสร้าง object ใหม่มี cost สูงกว่าการคัดลอกจากที่มีอยู่ |

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** สร้าง Singleton `Logger` (ใช้วิธี Enum Singleton) ที่มี method
`log(String message)` พิมพ์ข้อความพร้อม timestamp

**เฉลย:**

```java
public enum Logger {
    INSTANCE;

    public void log(String message) {
        System.out.println("[" + java.time.LocalDateTime.now() + "] " + message);
    }
}
```

**2)** สร้าง Factory Method สำหรับสร้าง `PaymentMethod` (CreditCard, PayPal,
BankTransfer) ตามชื่อที่รับมาเป็น String

**เฉลย:**

```java
public interface PaymentMethod {
    void pay(double amount);
}

public class PaymentFactory {
    public static PaymentMethod create(String type) {
        return switch (type) {
            case "credit_card" -> amount -> System.out.println("จ่ายผ่านบัตรเครดิต: " + amount);
            case "paypal" -> amount -> System.out.println("จ่ายผ่าน PayPal: " + amount);
            default -> throw new IllegalArgumentException("ไม่รู้จักวิธีจ่ายเงินนี้");
        };
    }
}
```

**3)** อธิบายว่าทำไม Enum Singleton ปลอดภัยกว่า Singleton แบบ class ธรรมดา

**เฉลย**: JVM รับประกันว่าแต่ละค่า enum จะมี**เพียง instance เดียว**เสมอตลอด
ทั้งโปรแกรม (ทบทวนจาก Part 18) โดย**ไม่ต้องเขียน synchronization logic เอง**
เลย — นอกจากนี้ enum ยัง**ป้องกันการสร้าง instance ซ้ำผ่าน reflection**
(ต่างจาก class ธรรมดาที่ยังสามารถถูก reflection เจาะทะลุ private constructor
ได้ — ทบทวนจาก Part 53) และ**ป้องกันปัญหาจาก deserialization** ที่อาจสร้าง
instance ใหม่ขึ้นมาซ้ำ (ทบทวนความเสี่ยงจาก Part 38) — ด้วยเหตุนี้ Enum
Singleton จึงเป็นวิธีที่ Effective Java แนะนำว่าปลอดภัยที่สุดในทุกกรณี

### สรุปเนื้อหา Part 54

- Design Pattern คือวิธีแก้ปัญหาการออกแบบที่พบซ้ำ ๆ แบ่งเป็น Creational,
  Structural, Behavioral
- Singleton รับประกัน instance เดียว — Enum Singleton ปลอดภัยที่สุด (ทน
  reflection, deserialization, thread-safe โดยธรรมชาติ)
- Factory Method ย้าย logic การสร้าง object ที่ซับซ้อนออกจากผู้เรียกใช้
- Abstract Factory สร้าง object หลายตัวที่เกี่ยวข้องกัน (family) ให้สอดคล้อง
  กันเสมอ
- Builder แก้ปัญหา constructor ที่มีพารามิเตอร์เยอะเกินไป (ทบทวนจาก Part 12)
- Prototype สร้าง object ใหม่โดยคัดลอกจากที่มีอยู่ — ในทางปฏิบัติมักใช้ copy
  constructor แทน `Cloneable`

**ต่อไป**: [Part 55 — Design Patterns: Structural (Adapter, Decorator, Facade, Proxy, Composite)](./part-055-design-patterns-structural.md)
