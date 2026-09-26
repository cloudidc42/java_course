# Part 13: Encapsulation และ Access Modifiers

> ขั้นตอนที่ 121-130 ของหลักสูตร | ระดับ: OOP พื้นฐาน

## สารบัญ

1. Encapsulation คืออะไร ทำไมสำคัญ
2. Access Modifiers ทั้ง 4 ระดับ
3. การใช้ `private` fields + `public` Getter/Setter
4. Validation ใน Setter
5. Read-only Properties (มี Getter ไม่มี Setter)
6. JavaBeans Convention
7. Access Modifiers กับ Class, Method, Constructor
8. Package-Private (Default) Access ในทางปฏิบัติ
9. ประโยชน์ของ Encapsulation ในสถานการณ์จริง
10. แบบฝึกหัดและสรุป

---

## 1. Encapsulation คืออะไร ทำไมสำคัญ

**Encapsulation (การห่อหุ้ม)** คือหลักการ**ซ่อนรายละเอียดภายใน**ของ object และ
**เปิดเผยเฉพาะสิ่งที่จำเป็น**ผ่าน interface ที่ควบคุมได้ (public methods) — เปรียบเทียบ
กับรีโมททีวี: เราไม่จำเป็นต้องรู้วงจรอิเล็กทรอนิกส์ภายใน แค่กดปุ่มที่เปิดให้ใช้งานก็พอ

ใน Part 11 เราสร้าง class ที่ field เป็น public ทั้งหมด ซึ่งมีปัญหาใหญ่:

```java
public class BankAccountUnsafe {
    public double balance; // ปัญหา: ใครก็แก้ไขได้โดยตรง ไม่มีการตรวจสอบใด ๆ
}
```

```java
public class UnsafeDemo {
    public static void main(String[] args) {
        BankAccountUnsafe account = new BankAccountUnsafe();
        account.balance = 1000;

        account.balance = -99999; // แก้ไขตรง ๆ ได้เลย! ไม่มีการตรวจสอบใด ๆ ทั้งสิ้น
        System.out.println(account.balance); // -99999 (!!) ยอดเงินติดลบมหาศาลอย่างไร้เหตุผล
    }
}
```

Encapsulation แก้ปัญหานี้โดยทำให้ field เป็น `private` แล้วให้เข้าถึงผ่านเมธอด
ที่ควบคุมได้เท่านั้น:

```java
public class BankAccountSafe {
    private double balance; // private: เข้าถึงได้เฉพาะภายในคลาสนี้เท่านั้น

    public BankAccountSafe(double initialBalance) {
        setBalance(initialBalance); // ใช้ setter เพื่อให้ validate ตั้งแต่ตอนสร้างด้วย
    }

    public double getBalance() { // Getter: ทางเดียวที่จะอ่านค่าจากภายนอกได้
        return balance;
    }

    public void setBalance(double balance) { // Setter: ทางเดียวที่จะแก้ไขค่าได้ พร้อม validate
        if (balance < 0) {
            throw new IllegalArgumentException("ยอดเงินห้ามติดลบ");
        }
        this.balance = balance;
    }
}
```

```java
public class SafeDemo {
    public static void main(String[] args) {
        BankAccountSafe account = new BankAccountSafe(1000);

        // account.balance = -99999; // Error! balance เป็น private เข้าถึงจากภายนอกไม่ได้เลย

        try {
            account.setBalance(-99999); // ถูก validate และถูกปฏิเสธ
        } catch (IllegalArgumentException e) {
            System.out.println("แก้ไขไม่สำเร็จ: " + e.getMessage());
        }

        System.out.println("ยอดเงินปัจจุบัน: " + account.getBalance()); // ยังคงเป็น 1000
    }
}
```

## 2. Access Modifiers ทั้ง 4 ระดับ

Java มี access modifier 4 ระดับ เรียงจากเปิดกว้างที่สุดไปจำกัดที่สุด:

| Modifier | เข้าถึงได้จาก class เดียวกัน | package เดียวกัน | subclass ต่าง package | ทุกที่ |
|---|---|---|---|---|
| `public` | ✅ | ✅ | ✅ | ✅ |
| `protected` | ✅ | ✅ | ✅ | ❌ |
| (ไม่ใส่อะไร / default) | ✅ | ✅ | ❌ | ❌ |
| `private` | ✅ | ❌ | ❌ | ❌ |

```java
package com.example.demo;

public class AccessModifierDemo {
    public int publicField = 1;         // เข้าถึงได้ทุกที่
    protected int protectedField = 2;   // เข้าถึงได้ใน package เดียวกัน + subclass
    int defaultField = 3;               // (ไม่ใส่ modifier) เข้าถึงได้เฉพาะใน package เดียวกัน
    private int privateField = 4;       // เข้าถึงได้เฉพาะใน class นี้เท่านั้น

    public void showAll() {
        // ภายใน class เดียวกัน เข้าถึงได้ทุกตัวเสมอ ไม่ว่า modifier จะเป็นอะไร
        System.out.println(publicField + " " + protectedField + " "
                          + defaultField + " " + privateField);
    }
}
```

**หลักการเลือกใช้ในทางปฏิบัติ**:
- **Fields**: ควรเป็น `private` แทบทุกครั้ง (encapsulation)
- **Methods ที่เป็น API สาธารณะ**: ใช้ `public`
- **Methods ที่เป็น helper ภายในเท่านั้น**: ใช้ `private`
- **Methods/fields ที่ subclass ต้องใช้ต่อ**: ใช้ `protected` (จะเข้าใจชัดใน Part 14)

## 3. การใช้ `private` Fields + `public` Getter/Setter

รูปแบบมาตรฐานที่ใช้ทั่วโลก: field เป็น `private` ทั้งหมด แล้วเปิดเผยผ่าน getter/setter
ที่เป็น `public`

```java
public class Employee {
    private String name;
    private double salary;
    private String department;

    public Employee(String name, double salary, String department) {
        this.name = name;
        setSalary(salary); // ผ่าน setter เพื่อ validate ตั้งแต่ตอนสร้าง object
        this.department = department;
    }

    // Getter สำหรับ name
    public String getName() {
        return name;
    }

    // Setter สำหรับ name
    public void setName(String name) {
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("ชื่อห้ามว่างเปล่า");
        }
        this.name = name;
    }

    public double getSalary() {
        return salary;
    }

    public void setSalary(double salary) {
        if (salary < 0) {
            throw new IllegalArgumentException("เงินเดือนห้ามติดลบ");
        }
        this.salary = salary;
    }

    public String getDepartment() {
        return department;
    }

    public void setDepartment(String department) {
        this.department = department;
    }
}
```

## 4. Validation ใน Setter

ประโยชน์หลักของ encapsulation คือสามารถใส่ **business logic / validation** ใน
setter ได้ ทำให้มั่นใจได้ว่า object จะไม่อยู่ในสถานะที่ผิดปกติ (invalid state)
ตลอดช่วงชีวิตของมัน

```java
public class Product {
    private String name;
    private double price;
    private int stock;

    public Product(String name, double price, int stock) {
        setName(name);
        setPrice(price);
        setStock(stock);
    }

    public void setPrice(double price) {
        if (price <= 0) {
            throw new IllegalArgumentException("ราคาต้องมากกว่า 0");
        }
        this.price = price;
    }

    public void setStock(int stock) {
        if (stock < 0) {
            throw new IllegalArgumentException("จำนวนสินค้าห้ามติดลบ");
        }
        this.stock = stock;
    }

    public void setName(String name) {
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("ชื่อสินค้าห้ามว่าง");
        }
        this.name = name;
    }

    // เมธอดที่มี logic มากกว่าแค่ set ค่าตรง ๆ
    public void reduceStock(int quantity) {
        if (quantity <= 0) {
            throw new IllegalArgumentException("จำนวนที่ลดต้องมากกว่า 0");
        }
        if (quantity > this.stock) {
            throw new IllegalStateException("สินค้าไม่พอในสต็อก (เหลือ " + this.stock + " ชิ้น)");
        }
        this.stock -= quantity;
    }

    public String getName() { return name; }
    public double getPrice() { return price; }
    public int getStock() { return stock; }
}
```

```java
public class ProductDemo {
    public static void main(String[] args) {
        Product laptop = new Product("Laptop", 25000, 10);

        laptop.reduceStock(3);
        System.out.println("คงเหลือ: " + laptop.getStock()); // 7

        try {
            laptop.reduceStock(100); // เกินจำนวนสต็อก
        } catch (IllegalStateException e) {
            System.out.println("ผิดพลาด: " + e.getMessage());
        }
    }
}
```

## 5. Read-only Properties (มี Getter ไม่มี Setter)

หากต้องการให้ field กำหนดค่าได้ครั้งเดียวตอนสร้าง object แล้วห้ามเปลี่ยนอีกเลย
(immutable field) ให้ใช้ `private final` และมีแค่ getter (ไม่มี setter):

```java
public class ImmutablePoint {
    private final double x; // final: กำหนดค่าได้ครั้งเดียวเท่านั้น (ตอน constructor)
    private final double y;

    public ImmutablePoint(double x, double y) {
        this.x = x;
        this.y = y;
    }

    public double getX() { return x; } // มีแค่ getter ไม่มี setter -> read-only จากภายนอก

    public double getY() { return y; }

    // ถ้าต้องการ "แก้ไข" ค่า ให้คืน object ใหม่แทนที่จะแก้ไขของเดิม (immutable pattern)
    public ImmutablePoint withX(double newX) {
        return new ImmutablePoint(newX, this.y);
    }
}
```

```java
public class ImmutableDemo {
    public static void main(String[] args) {
        ImmutablePoint p1 = new ImmutablePoint(3, 4);
        ImmutablePoint p2 = p1.withX(10); // ได้ object ใหม่ ต้นฉบับไม่เปลี่ยน

        System.out.println(p1.getX() + ", " + p1.getY()); // 3.0, 4.0 (เดิม ไม่เปลี่ยน)
        System.out.println(p2.getX() + ", " + p2.getY()); // 10.0, 4.0 (object ใหม่)
    }
}
```

**ข้อดีของ Immutable objects**: Thread-safe โดยธรรมชาติ, ปลอดภัยจากการแก้ไขโดย
ไม่ตั้งใจ, ใช้เป็น key ใน HashMap ได้อย่างปลอดภัย (จะลงลึกใน Part 51 เรื่อง Records)

## 6. JavaBeans Convention

**JavaBeans** เป็นธรรมเนียมมาตรฐานที่หลาย framework (Spring, Hibernate) คาดหวังให้
class ปฏิบัติตาม:

1. Field เป็น `private`
2. มี **no-argument constructor** (constructor ที่ไม่รับพารามิเตอร์)
3. Getter ตั้งชื่อ `getXxx()` (หรือ `isXxx()` สำหรับ `boolean`)
4. Setter ตั้งชื่อ `setXxx(value)`
5. Implement `Serializable` (จะเรียนใน Part 38)

```java
public class UserBean {
    private String username;
    private boolean active;

    public UserBean() { } // no-arg constructor ตามข้อกำหนด JavaBeans

    public String getUsername() { return username; }
    public void setUsername(String username) { this.username = username; }

    // สำหรับ boolean ใช้ isXxx() แทน getXxx() ตามธรรมเนียม (getXxx() ก็ยังใช้ได้แต่ isXxx() นิยมกว่า)
    public boolean isActive() { return active; }
    public void setActive(boolean active) { this.active = active; }
}
```

```java
public class JavaBeansDemo {
    public static void main(String[] args) {
        UserBean user = new UserBean(); // ใช้ no-arg constructor
        user.setUsername("somchai");
        user.setActive(true);

        System.out.println(user.getUsername() + " - active: " + user.isActive());
    }
}
```

## 7. Access Modifiers กับ Class, Method, Constructor

Access modifier ใช้ได้กับทั้ง top-level class, method, field, และ constructor
แต่**top-level class ใช้ได้แค่ `public` หรือ default เท่านั้น** (ไม่มี private/protected
สำหรับ top-level class เพราะไม่มี "คลาสแม่" ให้ protected/private เทียบกับอะไร)

```java
public class PublicClass { }     // เข้าถึงได้จากทุก package
class DefaultClass { }           // เข้าถึงได้เฉพาะใน package เดียวกัน (ต้องอยู่ไฟล์เดียวกัน หรือไฟล์แยกใน package เดียวกัน)
// private class InvalidClass {} // Error! top-level class เป็น private ไม่ได้
// protected class InvalidClass {} // Error! top-level class เป็น protected ไม่ได้
```

Constructor ก็มี access modifier ได้เช่นกัน — ใช้บ่อยมากในการทำ **Singleton
Pattern** (Part 54) ที่ทำให้ constructor เป็น `private` เพื่อป้องกันไม่ให้ผู้อื่น
สร้าง object จากภายนอกได้โดยตรง:

```java
public class Singleton {
    private static Singleton instance;
    private int data;

    private Singleton() { // private constructor: สร้างจากภายนอกไม่ได้เลย
        data = 0;
    }

    public static Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton(); // สร้างได้จากภายในคลาสเท่านั้น
        }
        return instance;
    }
}
```

## 8. Package-Private (Default) Access ในทางปฏิบัติ

เมื่อไม่ใส่ access modifier ใด ๆ (default/package-private) หมายความว่าเข้าถึงได้
เฉพาะจาก class ใน**package เดียวกัน**เท่านั้น มักใช้กับ helper class ที่ไม่ต้องการ
ให้ package ภายนอกเห็นหรือใช้งาน

```java
package com.example.internal;

class InternalHelper { // default access: ใช้ได้แค่ใน package com.example.internal เท่านั้น
    static void doInternalWork() {
        System.out.println("ทำงานภายใน package เท่านั้น");
    }
}

public class PublicService {
    public void serve() {
        InternalHelper.doInternalWork(); // เรียกได้เพราะอยู่ package เดียวกัน
    }
}
```

## 9. ประโยชน์ของ Encapsulation ในสถานการณ์จริง

1. **ป้องกันสถานะที่ผิดปกติ (invalid state)**: validation ใน setter ป้องกันไม่ให้
   object มีค่าที่ผิดกฎทางธุรกิจ
2. **เปลี่ยน implementation ภายในได้โดยไม่กระทบผู้ใช้**: เช่น เปลี่ยนจากเก็บ
   `celsius` เป็นเก็บ `fahrenheit` ภายใน แต่ผู้ใช้ยังเรียก `getCelsius()` เหมือนเดิม

```java
public class Temperature {
    private double fahrenheit; // เปลี่ยนวิธีเก็บข้อมูลภายในได้ โดยไม่กระทบผู้ใช้ API

    public void setCelsius(double celsius) {
        this.fahrenheit = celsius * 9 / 5 + 32; // แปลงเก็บเป็น fahrenheit ภายใน
    }

    public double getCelsius() { // ผู้ใช้ยังคงเรียกใช้ celsius เหมือนเดิม ไม่รู้ภายในเปลี่ยน
        return (fahrenheit - 32) * 5 / 9;
    }
}
```

3. **เพิ่ม logic เสริมได้ในภายหลังโดยไม่ต้องแก้โค้ดผู้ใช้**: เช่น เพิ่ม logging,
   caching ใน getter/setter ได้ทันทีโดยผู้เรียกใช้ไม่ต้องแก้โค้ดตัวเอง

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียนคลาส `Temperature` ที่เก็บอุณหภูมิเป็น Celsius (private) พร้อม getter/setter
ที่ validate ว่าอุณหภูมิต้องไม่ต่ำกว่า -273.15 (absolute zero)

**เฉลย:**

```java
public class Temperature {
    private double celsius;

    public Temperature(double celsius) {
        setCelsius(celsius);
    }

    public double getCelsius() {
        return celsius;
    }

    public void setCelsius(double celsius) {
        if (celsius < -273.15) {
            throw new IllegalArgumentException("อุณหภูมิต่ำกว่า absolute zero ไม่ได้");
        }
        this.celsius = celsius;
    }
}
```

**2)** เขียนคลาส `ImmutableCoordinate` ที่มี field `latitude`, `longitude` เป็น
`private final` พร้อม getter อย่างเดียว (ไม่มี setter)

**เฉลย:**

```java
public class ImmutableCoordinate {
    private final double latitude;
    private final double longitude;

    public ImmutableCoordinate(double latitude, double longitude) {
        this.latitude = latitude;
        this.longitude = longitude;
    }

    public double getLatitude() { return latitude; }
    public double getLongitude() { return longitude; }
}
```

**3)** เขียนคลาส `Wallet` ที่มี `balance` (private) พร้อมเมธอด `deposit(amount)` และ
`withdraw(amount)` ที่ validate ไม่ให้ยอดติดลบและไม่ให้ถอนเกินยอดคงเหลือ

**เฉลย:**

```java
public class Wallet {
    private double balance;

    public Wallet(double initialBalance) {
        if (initialBalance < 0) throw new IllegalArgumentException("ยอดเริ่มต้นห้ามติดลบ");
        this.balance = initialBalance;
    }

    public void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("จำนวนฝากต้องมากกว่า 0");
        balance += amount;
    }

    public void withdraw(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("จำนวนถอนต้องมากกว่า 0");
        if (amount > balance) throw new IllegalStateException("ยอดเงินไม่เพียงพอ");
        balance -= amount;
    }

    public double getBalance() { return balance; }
}
```

### สรุปเนื้อหา Part 13

- Encapsulation คือการซ่อนรายละเอียดภายใน เปิดเผยเฉพาะที่จำเป็นผ่าน public interface
- Access modifiers มี 4 ระดับ: `public` > `protected` > default > `private`
- รูปแบบมาตรฐาน: field เป็น `private` ทั้งหมด เข้าถึงผ่าน public getter/setter
- Setter ควรมี validation logic เพื่อป้องกันสถานะที่ผิดปกติ
- ใช้ `private final` + getter อย่างเดียวสำหรับ read-only/immutable properties
- JavaBeans convention: no-arg constructor + getXxx/setXxx/isXxx เป็นมาตรฐานที่หลาย
  framework คาดหวัง

**ต่อไป**: [Part 14 — Inheritance](./part-014-inheritance.md)
