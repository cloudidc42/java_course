# Part 21: Exception Handling ขั้นสูง

> ขั้นตอนที่ 201-210 ของหลักสูตร | ระดับ: กลาง (เริ่มหมวดโครงสร้างข้อมูลและอัลกอริทึม)

## สารบัญ

1. Custom Exception Classes
2. Exception Chaining (Cause)
3. `try-with-resources` แบบเต็มรูปแบบ
4. การสร้าง Resource ของตัวเองด้วย `AutoCloseable`
5. Multi-catch ทบทวนและกรณีขั้นสูง
6. Suppressed Exceptions
7. Checked Exception: ควรใช้เมื่อไร (ข้อถกเถียงในวงการ)
8. Exception Translation Pattern
9. Best Practices ระดับ Production
10. แบบฝึกหัดและสรุป

---

## 1. Custom Exception Classes

การสร้าง **Custom Exception** ของตัวเองช่วยให้ error message สื่อความหมายตรงกับ
domain ของแอปพลิเคชัน และให้ผู้เรียกใช้ catch เฉพาะเจาะจงได้ง่ายขึ้น

```java
// Custom Checked Exception: สืบทอดจาก Exception (ไม่ใช่ RuntimeException)
public class InsufficientFundsException extends Exception {
    private final double shortfall;

    public InsufficientFundsException(String message, double shortfall) {
        super(message);
        this.shortfall = shortfall;
    }

    public double getShortfall() {
        return shortfall;
    }
}
```

```java
// Custom Unchecked Exception: สืบทอดจาก RuntimeException
public class InvalidAccountException extends RuntimeException {
    public InvalidAccountException(String message) {
        super(message);
    }
}
```

```java
public class BankAccount {
    private double balance;

    public BankAccount(double balance) {
        this.balance = balance;
    }

    public void withdraw(double amount) throws InsufficientFundsException {
        if (amount > balance) {
            double shortfall = amount - balance;
            throw new InsufficientFundsException(
                "ยอดเงินไม่พอ ขาดอีก " + shortfall + " บาท", shortfall);
        }
        balance -= amount;
    }
}
```

```java
public class CustomExceptionDemo {
    public static void main(String[] args) {
        BankAccount account = new BankAccount(1000);

        try {
            account.withdraw(1500);
        } catch (InsufficientFundsException e) {
            System.out.println("ผิดพลาด: " + e.getMessage());
            System.out.println("ขาดอยู่: " + e.getShortfall() + " บาท"); // ใช้ custom field ได้
        }
    }
}
```

**หลักการเลือก Checked vs Unchecked สำหรับ Custom Exception**:
- **Checked** (extends `Exception`): ใช้เมื่อผู้เรียกใช้**ควรถูกบังคับให้จัดการ**
  เพราะเป็นสถานการณ์ที่คาดการณ์ได้และมีทางแก้ไข (เช่น เงินไม่พอ, ไฟล์ไม่พบ)
- **Unchecked** (extends `RuntimeException`): ใช้เมื่อเป็น**ข้อผิดพลาดจากการเขียน
  โค้ดผิด**หรือ**การใช้งาน API ผิดวิธี** ที่ผู้เรียกใช้ไม่ควรต้อง catch ทุกครั้ง
  (เช่น ส่ง argument ที่ไม่ถูกต้อง)

## 2. Exception Chaining (Cause)

**Exception Chaining** คือการ**ห่อ exception เดิม**ไว้ภายใน exception ใหม่ เพื่อ
รักษาข้อมูลต้นตอ (root cause) ไว้ ทั้งที่เปลี่ยนเป็น exception ที่สื่อความหมาย
กับระดับที่สูงขึ้นของแอปพลิเคชัน

```java
public class DataAccessException extends RuntimeException {
    public DataAccessException(String message, Throwable cause) {
        super(message, cause); // ส่ง cause ผ่าน constructor ของ Throwable
    }
}
```

```java
import java.sql.SQLException;

public class ExceptionChainingDemo {
    static void queryDatabase() {
        try {
            // จำลอง exception ระดับต่ำที่เกิดจากไลบรารีฐานข้อมูล
            throw new SQLException("Connection timeout");
        } catch (SQLException e) {
            // ห่อเป็น exception ระดับสูงที่สื่อความหมายกับ business logic มากกว่า
            // แต่ยังคงเก็บ e (สาเหตุดั้งเดิม) ไว้เพื่อ debug ได้
            throw new DataAccessException("ไม่สามารถดึงข้อมูลผู้ใช้ได้", e);
        }
    }

    public static void main(String[] args) {
        try {
            queryDatabase();
        } catch (DataAccessException e) {
            System.out.println("ข้อผิดพลาดระดับสูง: " + e.getMessage());
            System.out.println("สาเหตุดั้งเดิม: " + e.getCause().getMessage());
            e.printStackTrace(); // stack trace จะแสดงทั้ง exception ใหม่และ "Caused by:" ของเดิม
        }
    }
}
```

ผลลัพธ์ของ `printStackTrace()` จะแสดง chain ทั้งหมด:

```
DataAccessException: ไม่สามารถดึงข้อมูลผู้ใช้ได้
    at ExceptionChainingDemo.queryDatabase(...)
Caused by: java.sql.SQLException: Connection timeout
    at ExceptionChainingDemo.queryDatabase(...)
```

## 3. `try-with-resources` แบบเต็มรูปแบบ

ทบทวนจาก Part 10: `try-with-resources` ปิด resource ให้อัตโนมัติเมื่อออกจาก block
(ไม่ว่าสำเร็จหรือเกิด exception) — Part นี้จะลงรายละเอียดเรื่อง**หลาย resource**
และ**ลำดับการปิด**

```java
import java.io.BufferedReader;
import java.io.FileReader;
import java.io.FileWriter;
import java.io.IOException;

public class MultiResourceDemo {
    public static void main(String[] args) {
        // สามารถเปิดหลาย resource พร้อมกัน คั่นด้วย ; ภายใน try(...)
        try (BufferedReader reader = new BufferedReader(new FileReader("input.txt"));
             FileWriter writer = new FileWriter("output.txt")) {

            String line;
            while ((line = reader.readLine()) != null) {
                writer.write(line.toUpperCase());
                writer.write(System.lineSeparator());
            }

        } catch (IOException e) {
            System.out.println("เกิดข้อผิดพลาด: " + e.getMessage());
        }
        // ลำดับการปิด: resource จะถูกปิด "ย้อนกลับ" จากลำดับที่เปิด
        // (writer ปิดก่อน แล้วค่อยปิด reader) เหมือนการ pop stack
    }
}
```

## 4. การสร้าง Resource ของตัวเองด้วย `AutoCloseable`

Class ใด ๆ ที่ implement `AutoCloseable` (มีเมธอด `close()`) สามารถใช้กับ
`try-with-resources` ได้ทันที — มีประโยชน์มากเมื่อสร้าง resource ของตัวเอง เช่น
connection pool, custom logger ที่ต้อง flush/close

```java
public class DatabaseConnection implements AutoCloseable {
    private String name;

    public DatabaseConnection(String name) {
        this.name = name;
        System.out.println("เปิดการเชื่อมต่อ: " + name);
    }

    public void executeQuery(String sql) {
        System.out.println("รันคำสั่ง: " + sql);
    }

    @Override
    public void close() { // ไม่ throw checked exception ก็ได้ ขึ้นกับความจำเป็นของ resource นั้น
        System.out.println("ปิดการเชื่อมต่อ: " + name);
    }
}
```

```java
public class CustomAutoCloseableDemo {
    public static void main(String[] args) {
        try (DatabaseConnection conn = new DatabaseConnection("MainDB")) {
            conn.executeQuery("SELECT * FROM users");
        }
        // ผลลัพธ์:
        // เปิดการเชื่อมต่อ: MainDB
        // รันคำสั่ง: SELECT * FROM users
        // ปิดการเชื่อมต่อ: MainDB   <- close() ถูกเรียกอัตโนมัติแม้ไม่มี exception เกิดขึ้น
    }
}
```

## 5. Multi-catch ทบทวนและกรณีขั้นสูง

```java
public class AdvancedMultiCatchDemo {
    static void process(String type) throws Exception {
        if (type.equals("io")) throw new java.io.IOException("IO error");
        if (type.equals("sql")) throw new java.sql.SQLException("SQL error");
        throw new Exception("Unknown error");
    }

    public static void main(String[] args) {
        for (String type : new String[]{"io", "sql", "other"}) {
            try {
                process(type);
            } catch (java.io.IOException | java.sql.SQLException e) {
                // จัดการทั้ง IOException และ SQLException ด้วย logic เดียวกัน
                System.out.println("ข้อผิดพลาดที่คาดการณ์ได้: " + e.getClass().getSimpleName());
            } catch (Exception e) {
                System.out.println("ข้อผิดพลาดที่ไม่คาดคิด: " + e.getMessage());
            }
        }
    }
}
```

**ข้อจำกัดของ multi-catch**: exception ที่รวมกันด้วย `|` ต้อง**ไม่มีความสัมพันธ์
เป็น subtype ของกันและกัน** (ถ้า A เป็น subtype ของ B การเขียน `catch (A | B e)`
จะ compile error เพราะ B ครอบคลุม A อยู่แล้ว ซ้ำซ้อนไม่มีประโยชน์)

## 6. Suppressed Exceptions

เมื่อใช้ `try-with-resources` และเกิด exception **ทั้งใน try block และตอนปิด
resource (close())** Java จะ**เก็บ exception จาก close() ไว้เป็น "suppressed"**
แทนที่จะบัง exception หลักจาก try block ทิ้งไป — ป้องกันไม่ให้ข้อมูลสำคัญสูญหาย

```java
public class SuppressedResource implements AutoCloseable {
    @Override
    public void close() throws Exception {
        throw new Exception("Exception ตอนปิด resource!");
    }

    void doWork() throws Exception {
        throw new Exception("Exception ตอนทำงานหลัก!");
    }
}
```

```java
public class SuppressedExceptionDemo {
    public static void main(String[] args) {
        try (SuppressedResource resource = new SuppressedResource()) {
            resource.doWork();
        } catch (Exception e) {
            System.out.println("Exception หลัก: " + e.getMessage()); // "Exception ตอนทำงานหลัก!"

            // exception จาก close() ไม่ได้หายไป แต่ถูกเก็บไว้ใน suppressed exceptions
            for (Throwable suppressed : e.getSuppressed()) {
                System.out.println("Suppressed: " + suppressed.getMessage()); // "Exception ตอนปิด resource!"
            }
        }
    }
}
```

## 7. Checked Exception: ควรใช้เมื่อไร (ข้อถกเถียงในวงการ)

ในวงการ Java มีข้อถกเถียงมานานว่า checked exception ดีหรือไม่ดี:

**ฝ่ายสนับสนุน checked exception**: บังคับให้ผู้เรียกใช้ "ต้องคิด" ว่าจะจัดการ
ข้อผิดพลาดอย่างไร ทำให้ API สื่อสารชัดเจนผ่าน signature

**ฝ่ายคัดค้าน** (framework สมัยใหม่อย่าง Spring มักเลือกทางนี้): checked
exception ทำให้โค้ดรกด้วย try-catch ที่ไม่มีความหมาย (เช่น catch แล้วแค่ log
หรือแปลงเป็น unchecked แล้ว throw ต่อ) และขัดขวางการใช้ lambda/stream (Part 39-42)
เพราะ functional interface มาตรฐานไม่รองรับ throw checked exception

```java
// ปัญหา: checked exception ทำให้ใช้กับ Stream API ไม่สะดวก (จะเข้าใจเต็มที่ใน Part 41)
import java.util.List;
import java.util.stream.Collectors;

public class CheckedExceptionStreamProblem {
    static int parseOrThrow(String s) throws Exception { // checked exception
        if (!s.matches("\\d+")) throw new Exception("ไม่ใช่ตัวเลข");
        return Integer.parseInt(s);
    }

    public static void main(String[] args) {
        List<String> inputs = List.of("1", "2", "3");
        // inputs.stream().map(parseOrThrow).toList(); // Error! Stream.map ไม่รองรับ checked exception
        //                                               // ต้องห่อด้วย try-catch ภายใน lambda เอง ทำให้โค้ดรกมาก
    }
}
```

**แนวทางปฏิบัติในโค้ดสมัยใหม่ (แนะนำในหลักสูตรนี้)**: ใช้ **unchecked exception**
เป็นหลักสำหรับ custom exception ส่วนใหญ่ และสงวน checked exception ไว้เฉพาะกรณีที่
ผู้เรียกใช้**มีทางแก้ไขที่ชัดเจนจริง ๆ** เท่านั้น (เช่น retry การเชื่อมต่อเครือข่าย)

## 8. Exception Translation Pattern

**Exception Translation** คือการ**แปลง exception ระดับต่ำ**(low-level, มักมาจาก
library ภายนอก) **ให้เป็น exception ระดับสูง**ที่สื่อความหมายกับ business logic
ของแอปพลิเคชันเรา — ช่วยไม่ให้ layer บนของแอปพลิเคชันต้องรู้จัก exception ของ
library ภายนอกโดยตรง (ลด coupling)

```java
public class UserNotFoundException extends RuntimeException {
    public UserNotFoundException(String userId, Throwable cause) {
        super("ไม่พบผู้ใช้: " + userId, cause);
    }
}

public class UserRepository {
    User findById(String id) {
        try {
            return queryDatabaseForUser(id); // สมมติว่าอาจ throw SQLException จาก library ฐานข้อมูล
        } catch (java.sql.SQLException e) {
            // แปลง SQLException (รายละเอียดเชิงเทคนิคของฐานข้อมูล) ให้เป็น exception
            // ที่ business logic layer เข้าใจและจัดการได้โดยไม่ต้อง import java.sql เลย
            throw new UserNotFoundException(id, e);
        }
    }

    private User queryDatabaseForUser(String id) throws java.sql.SQLException {
        throw new java.sql.SQLException("Connection refused"); // จำลอง error จาก database driver
    }
}

class User { }
```

## 9. Best Practices ระดับ Production

1. **สร้าง exception hierarchy ของแอปพลิเคชันตัวเอง** — มี base exception class
   กลาง (เช่น `ApplicationException`) แล้วให้ custom exception อื่น ๆ สืบทอดจากมัน
2. **ใส่ข้อมูลที่เป็นประโยชน์ใน exception message** ระบุค่าที่ทำให้เกิดปัญหา
   (แต่ระวังไม่ใส่ข้อมูล sensitive เช่น password ลงใน log)
3. **Log exception ที่จุดที่จัดการจริง** (handle) เท่านั้น ไม่ log ซ้ำหลายที่
   ตลอด call stack (จะทำให้ log รกและสับสน)
4. **อย่า catch `Throwable` หรือ `Error`** ในโค้ดทั่วไป (ยกเว้นกรณีพิเศษเช่น
   framework ระดับ infrastructure) เพราะ `Error` มักบอกถึงปัญหาที่ร้ายแรงเกินกว่า
   จะแก้ไขได้ในระดับแอปพลิเคชัน
5. **ใช้ exception เพื่อจัดการสถานการณ์ผิดปกติจริง ๆ เท่านั้น** ไม่ใช่แทน
   control flow ปกติ (ทบทวนจาก Part 10)

```java
public class ApplicationException extends RuntimeException {
    public ApplicationException(String message) { super(message); }
    public ApplicationException(String message, Throwable cause) { super(message, cause); }
}

public class ValidationException extends ApplicationException {
    public ValidationException(String message) { super(message); }
}

public class ResourceNotFoundException extends ApplicationException {
    public ResourceNotFoundException(String message) { super(message); }
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** สร้าง custom unchecked exception `NegativeAmountException` และใช้ในเมธอด
`transfer(double amount)` ที่ throw exception นี้ถ้า amount ติดลบ

**เฉลย:**

```java
public class NegativeAmountException extends RuntimeException {
    public NegativeAmountException(double amount) {
        super("จำนวนเงินห้ามติดลบ: " + amount);
    }
}

public class Exercise1 {
    static void transfer(double amount) {
        if (amount < 0) {
            throw new NegativeAmountException(amount);
        }
        System.out.println("โอนเงิน " + amount + " บาทสำเร็จ");
    }

    public static void main(String[] args) {
        transfer(500);
        try {
            transfer(-100);
        } catch (NegativeAmountException e) {
            System.out.println(e.getMessage());
        }
    }
}
```

**2)** เขียนคลาส `Resource` ที่ implement `AutoCloseable` แล้วทดสอบด้วย
try-with-resources ว่า `close()` ถูกเรียกจริงแม้เกิด exception ใน try block

**เฉลย:**

```java
public class Exercise2Resource implements AutoCloseable {
    @Override
    public void close() {
        System.out.println("ปิด resource แล้ว");
    }
}

public class Exercise2 {
    public static void main(String[] args) {
        try (Exercise2Resource r = new Exercise2Resource()) {
            throw new RuntimeException("เกิดข้อผิดพลาดกลางทาง");
        } catch (RuntimeException e) {
            System.out.println("จับ exception ได้: " + e.getMessage());
        }
        // ผลลัพธ์แสดง "ปิด resource แล้ว" ก่อน "จับ exception ได้..." เสมอ
    }
}
```

**3)** เขียนตัวอย่าง Exception Translation ที่แปลง `NumberFormatException`
(low-level) เป็น custom exception `InvalidInputException` (business-level)

**เฉลย:**

```java
public class InvalidInputException extends RuntimeException {
    public InvalidInputException(String input, Throwable cause) {
        super("ข้อมูลนำเข้าไม่ถูกต้อง: " + input, cause);
    }
}

public class Exercise3 {
    static int parseUserAge(String input) {
        try {
            return Integer.parseInt(input);
        } catch (NumberFormatException e) {
            throw new InvalidInputException(input, e);
        }
    }

    public static void main(String[] args) {
        try {
            parseUserAge("abc");
        } catch (InvalidInputException e) {
            System.out.println(e.getMessage() + " (สาเหตุ: " + e.getCause().getMessage() + ")");
        }
    }
}
```

### สรุปเนื้อหา Part 21

- Custom exception ช่วยให้ error สื่อความหมายตรงกับ domain เลือก checked/unchecked
  ตามว่าผู้เรียกใช้มีทางแก้ไขจริงหรือไม่
- Exception chaining (`cause`) รักษาข้อมูลต้นตอไว้แม้แปลงเป็น exception ระดับสูง
- `try-with-resources` ปิดหลาย resource ตามลำดับย้อนกลับอัตโนมัติ, สร้าง resource
  เองได้ด้วย `AutoCloseable`
- Suppressed exceptions เก็บ exception จาก `close()` ไว้โดยไม่บัง exception หลัก
- แนวโน้มโค้ดสมัยใหม่นิยม unchecked exception มากกว่า เพื่อความสะดวกกับ
  lambda/stream
- Exception Translation แปลง exception ระดับ library เป็น exception ระดับ
  business logic ลด coupling ระหว่าง layer

**ต่อไป**: [Part 22 — Collections Framework: List](./part-022-collections-list.md)
