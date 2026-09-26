# Part 38: Serialization และ Deserialization

> ขั้นตอนที่ 371-380 ของหลักสูตร | ระดับ: กลาง

## สารบัญ

1. Serialization คืออะไร ใช้ทำอะไร
2. `Serializable` Interface
3. การ Serialize และ Deserialize Object
4. `serialVersionUID`: ทำไมสำคัญ
5. `transient` Keyword
6. Serialization กับ Inheritance
7. ข้อจำกัดและความเสี่ยงด้านความปลอดภัยของ Java Serialization
8. ทางเลือกสมัยใหม่: JSON (ปูทางสู่ Part 70)
9. Custom Serialization ด้วย `writeObject`/`readObject`
10. แบบฝึกหัดและสรุป

---

## 1. Serialization คืออะไร ใช้ทำอะไร

**Serialization** คือกระบวนการแปลง**object ในหน่วยความจำ**ให้เป็น**ลำดับของ
bytes** ที่สามารถบันทึกลงไฟล์ ส่งผ่านเครือข่าย หรือเก็บไว้ใช้ทีหลังได้ —
**Deserialization** คือกระบวนการย้อนกลับ แปลง bytes กลับเป็น object เดิม

**กรณีใช้งาน**: บันทึก game save file, ส่ง object ผ่าน network (RMI), cache
object ไว้ใช้ทีหลัง, ส่ง object ระหว่าง JVM (แม้จะพบน้อยลงในโค้ดสมัยใหม่ที่
นิยมใช้ JSON แทน — ดูหัวข้อ 8)

```
Object ในหน่วยความจำ  --serialize-->  ลำดับ bytes  --เก็บ/ส่ง-->  ลำดับ bytes
        ↑                                                              |
        └──────────────────deserialize──────────────────────────────┘
```

## 2. `Serializable` Interface

Class ที่ต้องการ serialize ได้ ต้อง `implements Serializable` — เป็น
**marker interface** (interface ที่ไม่มี method ใดเลย เพียงแค่ "ประกาศ
เจตนา" ให้ JVM รู้ว่า class นี้อนุญาตให้ serialize ได้)

```java
import java.io.Serializable;

public class Student implements Serializable {
    private String name;
    private int age;
    private double gpa;

    public Student(String name, int age, double gpa) {
        this.name = name;
        this.age = age;
        this.gpa = gpa;
    }

    @Override
    public String toString() {
        return "Student{name='" + name + "', age=" + age + ", gpa=" + gpa + "}";
    }
}
```

## 3. การ Serialize และ Deserialize Object

```java
import java.io.*;

public class SerializationDemo {
    public static void main(String[] args) {
        Student student = new Student("Alice", 20, 3.75);

        // Serialize: เขียน object ลงไฟล์
        try (ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("student.ser"))) {
            out.writeObject(student);
            System.out.println("Serialize สำเร็จ");
        } catch (IOException e) {
            System.out.println("Serialize ไม่สำเร็จ: " + e.getMessage());
        }

        // Deserialize: อ่าน object กลับจากไฟล์
        try (ObjectInputStream in = new ObjectInputStream(new FileInputStream("student.ser"))) {
            Student loaded = (Student) in.readObject(); // ต้อง cast กลับเป็นชนิดเดิม
            System.out.println("Deserialize สำเร็จ: " + loaded);
        } catch (IOException | ClassNotFoundException e) {
            System.out.println("Deserialize ไม่สำเร็จ: " + e.getMessage());
        }
    }
}
```

**หมายเหตุ**: `readObject()` throw `ClassNotFoundException` (checked exception
— ทบทวนจาก Part 10) เพราะ JVM ต้องหา `.class` ของชนิดข้อมูลที่ deserialize
ให้เจอก่อน ถ้าไม่มี class นี้อยู่ใน classpath จะ deserialize ไม่ได้

## 4. `serialVersionUID`: ทำไมสำคัญ

**`serialVersionUID`** คือตัวเลขที่ใช้**ระบุ version ของ class** สำหรับ
serialization — ถ้าไม่ประกาศเอง JVM จะคำนวณให้อัตโนมัติจาก field/method
ของ class (ซึ่ง**อาจเปลี่ยนไปได้**เมื่อแก้ไข class แม้เพียงเล็กน้อย)

```java
import java.io.Serializable;

public class Employee implements Serializable {
    private static final long serialVersionUID = 1L; // ควรประกาศเองเสมอ (แนวปฏิบัติที่ดี)

    private String name;
    private double salary;

    public Employee(String name, double salary) {
        this.name = name;
        this.salary = salary;
    }
}
```

**ทำไมต้องประกาศเอง**: ถ้าไม่ประกาศและมีคน**แก้ไข class** (เช่น เพิ่ม field
ใหม่) ในภายหลัง `serialVersionUID` ที่ JVM คำนวณอัตโนมัติจะเปลี่ยนไป ทำให้
**object ที่ serialize ไว้ด้วย version เก่า deserialize ไม่ได้อีก** (จะเกิด
`InvalidClassException`) — การประกาศ `serialVersionUID` เองทำให้ควบคุมได้ว่า
เมื่อไรจะถือว่า "version เปลี่ยนจริง ๆ"

```java
public class VersionMismatchDemo {
    public static void main(String[] args) {
        // สมมติสถานการณ์: serialize Employee v1 (ไม่มี field ใหม่) เก็บไว้
        // จากนั้นแก้ไข class เพิ่ม field ใหม่ (เปลี่ยนเป็น v2) โดยไม่เปลี่ยน serialVersionUID
        // ผลลัพธ์: ยัง deserialize ได้ (field ใหม่จะได้ค่า default)
        // แต่ถ้าไม่ได้ประกาศ serialVersionUID เอง JVM อาจคำนวณ UID ใหม่ต่างจากเดิม
        // ทำให้ deserialize v1 ด้วยโค้ด v2 ไม่ได้เลย (InvalidClassException)
    }
}
```

## 5. `transient` Keyword

**`transient`** ใช้กำกับ field ที่**ไม่ต้องการให้ serialize** (เช่น password,
ข้อมูลที่คำนวณใหม่ได้เสมอ, resource ที่ผูกกับ session ปัจจุบันเท่านั้นอย่าง
`Thread` หรือ database connection)

```java
import java.io.Serializable;

public class UserSession implements Serializable {
    private static final long serialVersionUID = 1L;

    private String username;
    private transient String password; // ไม่ถูก serialize (ด้วยเหตุผลด้านความปลอดภัย)
    private transient long lastCalculatedValue; // ไม่จำเป็นต้องเก็บ คำนวณใหม่ได้เสมอ

    public UserSession(String username, String password) {
        this.username = username;
        this.password = password;
    }

    @Override
    public String toString() {
        return "UserSession{username='" + username + "', password='" + password + "'}";
    }
}
```

```java
import java.io.*;

public class TransientDemo {
    public static void main(String[] args) throws IOException, ClassNotFoundException {
        UserSession session = new UserSession("somchai", "secret123");

        try (ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("session.ser"))) {
            out.writeObject(session);
        }

        try (ObjectInputStream in = new ObjectInputStream(new FileInputStream("session.ser"))) {
            UserSession loaded = (UserSession) in.readObject();
            System.out.println(loaded);
            // UserSession{username='somchai', password='null'}
            // password เป็น null เพราะ transient ทำให้ไม่ถูก serialize ไปด้วย
        }
    }
}
```

## 6. Serialization กับ Inheritance

- ถ้า **superclass implements `Serializable`** subclass จะ serialize ได้
  โดยอัตโนมัติ (สืบทอดคุณสมบัตินี้มา — ทบทวนแนวคิด inheritance จาก Part 14)
- ถ้า **superclass ไม่ implement `Serializable`** แต่ subclass implement เอง
  **field ของ superclass จะไม่ถูก serialize** และ **superclass ต้องมี
  no-argument constructor** (ทบทวนจาก Part 12) เพื่อให้ deserialize สร้าง
  object ของ superclass ส่วนนั้นได้

```java
import java.io.Serializable;

class Vehicle { // ไม่ implement Serializable
    String brand = "Unknown";
    public Vehicle() { } // ต้องมี no-arg constructor
}

public class Car extends Vehicle implements Serializable {
    private static final long serialVersionUID = 1L;
    String model;

    public Car(String model) {
        this.model = model;
        this.brand = "Toyota"; // ตั้งค่า brand จาก superclass ที่ไม่ serialize
    }
}
```

```java
import java.io.*;

public class InheritanceSerializationDemo {
    public static void main(String[] args) throws IOException, ClassNotFoundException {
        Car car = new Car("Camry");
        car.brand = "Toyota";

        try (ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("car.ser"))) {
            out.writeObject(car);
        }

        try (ObjectInputStream in = new ObjectInputStream(new FileInputStream("car.ser"))) {
            Car loaded = (Car) in.readObject();
            System.out.println(loaded.model);  // "Camry" (serialize ได้ตามปกติ)
            System.out.println(loaded.brand);   // "Unknown" (!) กลับไปเป็นค่า default จาก
                                                  // constructor ของ Vehicle เพราะ Vehicle ไม่ serialize
        }
    }
}
```

## 7. ข้อจำกัดและความเสี่ยงด้านความปลอดภัยของ Java Serialization

**Java Serialization มีปัญหาด้านความปลอดภัยที่รู้จักกันดี**: การ deserialize
ข้อมูลที่**มาจากแหล่งที่ไม่น่าเชื่อถือ**เปิดช่องให้เกิด **Deserialization
Vulnerability** — ผู้โจมตีสามารถสร้าง byte stream ที่ปลอมแปลงเป็น object
อันตราย (malicious object) ทำให้เกิดการรันโค้ดที่ไม่ได้รับอนุญาต (Remote Code
Execution) ได้ในบางกรณี

**คำแนะนำสำคัญ** (ตรงกับหลักการ Security ใน Part 101):
1. **ห้าม deserialize ข้อมูลจากแหล่งที่ไม่น่าเชื่อถือเด็ดขาด** (เช่น input
   จากผู้ใช้ผ่านเครือข่ายโดยตรง)
2. พิจารณาใช้ **JSON** (Part 70) หรือ **Protocol Buffers** แทน Java native
   serialization สำหรับการสื่อสารระหว่างระบบ เพราะปลอดภัยกว่าและใช้งานร่วมกับ
   ภาษาอื่นได้ด้วย
3. ถ้าจำเป็นต้องใช้ Java serialization จริง ๆ ควรใช้ **whitelist ของ class ที่
   อนุญาตให้ deserialize** เท่านั้น (ObjectInputFilter — Java 9+)

## 8. ทางเลือกสมัยใหม่: JSON (ปูทางสู่ Part 70)

ในโปรเจกต์จริงสมัยใหม่ (โดยเฉพาะ REST API และ microservices — Part 78, 91)
**JSON เป็นทางเลือกที่นิยมกว่า Java Serialization มาก** เพราะ:
- **Human-readable**: อ่านและ debug ได้ง่ายด้วยตาเปล่า
- **Language-independent**: ใช้งานร่วมกับภาษาอื่น (JavaScript, Python, Go) ได้
  โดยตรง ไม่ผูกติดกับ Java เท่านั้น
- **ปลอดภัยกว่า**: ไม่มีปัญหา Remote Code Execution แบบ Java serialization

```java
// ตัวอย่างแนวคิด (จะเรียนเครื่องมือจริงใน Part 70 ด้วย Jackson/Gson)
public class JsonPreviewDemo {
    public static void main(String[] args) {
        // Java Serialization: {binary bytes ที่มนุษย์อ่านไม่ได้}
        // JSON: {"name":"Alice","age":20,"gpa":3.75}  <- อ่านง่าย ใช้ร่วมกับภาษาอื่นได้
    }
}
```

## 9. Custom Serialization ด้วย `writeObject`/`readObject`

สามารถ**ควบคุมกระบวนการ serialize/deserialize เองได้** โดย override เมธอด
พิเศษ `writeObject`/`readObject` (เช่น เพื่อเข้ารหัสข้อมูลก่อนบันทึก หรือคำนวณ
field ที่เป็น `transient` กลับมาใหม่หลัง deserialize):

```java
import java.io.*;

public class SecureAccount implements Serializable {
    private static final long serialVersionUID = 1L;
    private String accountId;
    private transient String pin; // ไม่ serialize ตรง ๆ ด้วยเหตุผลความปลอดภัย

    public SecureAccount(String accountId, String pin) {
        this.accountId = accountId;
        this.pin = pin;
    }

    private void writeObject(ObjectOutputStream out) throws IOException {
        out.defaultWriteObject(); // serialize field ปกติทั้งหมดก่อน (ยกเว้น transient)
        out.writeObject(encrypt(pin)); // เขียน pin แบบเข้ารหัสเองแยกต่างหาก
    }

    private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
        in.defaultReadObject(); // deserialize field ปกติก่อน
        String encryptedPin = (String) in.readObject();
        this.pin = decrypt(encryptedPin); // ถอดรหัสกลับมาเป็น pin จริง
    }

    private String encrypt(String value) { return new StringBuilder(value).reverse().toString(); } // ตัวอย่างง่าย ๆ
    private String decrypt(String value) { return new StringBuilder(value).reverse().toString(); }
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** สร้างคลาส `GameSave` ที่มี field `playerName`, `level`, `score` ที่
serialize ได้ พร้อมทดสอบ serialize/deserialize ให้ครบทั้งสองทาง

**เฉลย:**

```java
import java.io.*;

public class GameSave implements Serializable {
    private static final long serialVersionUID = 1L;
    String playerName;
    int level;
    long score;

    public GameSave(String playerName, int level, long score) {
        this.playerName = playerName;
        this.level = level;
        this.score = score;
    }

    @Override
    public String toString() {
        return playerName + " - Level " + level + " - Score " + score;
    }
}

public class Exercise1 {
    public static void main(String[] args) throws IOException, ClassNotFoundException {
        GameSave save = new GameSave("Player1", 5, 12500);
        try (ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("save.dat"))) {
            out.writeObject(save);
        }
        try (ObjectInputStream in = new ObjectInputStream(new FileInputStream("save.dat"))) {
            System.out.println(in.readObject());
        }
    }
}
```

**2)** ทดสอบว่า field ที่เป็น `transient` จะได้ค่าอะไรหลัง deserialize ถ้าเป็น
primitive type (เช่น `int`) เทียบกับ reference type

**เฉลย**: `transient` primitive field จะได้ค่า **default ของชนิดข้อมูลนั้น**
(เช่น `int` ได้ `0`, `boolean` ได้ `false`) ส่วน `transient` reference field
จะได้ `null` เสมอ — เพราะ deserialize ไม่ได้เรียก constructor ปกติ แต่สร้าง
object ขึ้นมาจาก byte stream โดยตรง และข้าม field ที่เป็น transient ไปทั้งหมด

**3)** อธิบายว่าทำไมองค์กรสมัยใหม่มักเลือกใช้ JSON แทน Java native
serialization สำหรับการสื่อสารระหว่างระบบ (microservices)

**เฉลย**: JSON เป็น text format ที่ human-readable และ language-independent
ทำให้ debug ง่ายและใช้งานร่วมกับระบบที่เขียนด้วยภาษาอื่นได้ (เช่น frontend
JavaScript, service ที่เขียนด้วย Python/Go) ในขณะที่ Java native serialization
ผูกติดกับ JVM เท่านั้นและมีความเสี่ยงด้านความปลอดภัยที่รู้จักกันดี (Remote Code
Execution จากการ deserialize ข้อมูลที่ไม่น่าเชื่อถือ) — ในระบบ microservices
ที่ต้องสื่อสารข้ามภาษาและข้าม network บ่อยครั้ง JSON จึงเป็นตัวเลือกที่
เหมาะสมกว่ามาก

### สรุปเนื้อหา Part 38

- Serialization แปลง object เป็น bytes เพื่อบันทึก/ส่งผ่านเครือข่าย,
  Deserialization คือกระบวนการย้อนกลับ
- Class ต้อง `implements Serializable` (marker interface) จึง serialize ได้
- ควรประกาศ `serialVersionUID` เองเสมอ เพื่อควบคุม compatibility ระหว่าง
  version ของ class
- `transient` ใช้ยกเว้น field จากการ serialize (เช่น password, resource ที่
  ผูกกับ session)
- Java Serialization มีความเสี่ยงด้านความปลอดภัย — ห้าม deserialize ข้อมูล
  จากแหล่งที่ไม่น่าเชื่อถือ, โค้ดสมัยใหม่นิยมใช้ JSON แทน

**ต่อไป**: [Part 39 — Lambda Expressions](./part-039-lambda-expressions.md)
