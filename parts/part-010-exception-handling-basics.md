# Part 10: Exception Handling เบื้องต้น (try-catch-finally, throw, throws)

> ขั้นตอนที่ 91-100 ของหลักสูตร | ระดับ: พื้นฐาน

## สารบัญ

1. Exception คืออะไร ทำไมต้องจัดการ
2. ลำดับชั้นของ Exception (Exception Hierarchy)
3. Checked vs Unchecked Exceptions
4. `try-catch` พื้นฐาน
5. หลาย `catch` blocks และ Multi-catch
6. `finally` block
7. `throw` และ `throws`
8. Exception ที่พบบ่อยและวิธีป้องกัน
9. แนวปฏิบัติที่ดีในการจัดการ Exception
10. แบบฝึกหัดและสรุป

---

## 1. Exception คืออะไร ทำไมต้องจัดการ

**Exception** คือเหตุการณ์ผิดปกติที่เกิดขึ้นระหว่างการรันโปรแกรม ทำให้การทำงานปกติ
หยุดชะงัก (เช่น หารด้วยศูนย์, เข้าถึง array นอกขอบเขต, ไฟล์ที่ต้องการเปิดไม่มีอยู่จริง)

หากไม่มีการจัดการ exception โปรแกรมจะ**หยุดทำงานทันที (crash)** และแสดง
**stack trace** ออกทาง console:

```java
public class UnhandledExceptionDemo {
    public static void main(String[] args) {
        int[] numbers = {1, 2, 3};
        System.out.println("ก่อนเกิดข้อผิดพลาด");
        System.out.println(numbers[10]); // โปรแกรมจะพังตรงนี้
        System.out.println("บรรทัดนี้จะไม่ถูกรันเลย");
    }
}
```

ผลลัพธ์:

```
ก่อนเกิดข้อผิดพลาด
Exception in thread "main" java.lang.ArrayIndexOutOfBoundsException:
    Index 10 out of bounds for length 3
    at UnhandledExceptionDemo.main(UnhandledExceptionDemo.java:5)
```

การจัดการ exception ช่วยให้โปรแกรม**รับมือกับความผิดพลาดอย่างสุภาพ (graceful)**
แทนที่จะพังทันที เช่น แจ้งข้อความที่เข้าใจง่ายแก่ผู้ใช้ บันทึก log แล้วลองใหม่
หรือใช้ค่า default แทน

## 2. ลำดับชั้นของ Exception (Exception Hierarchy)

```
                    Throwable
                   /          \
              Error            Exception
           (ร้ายแรงมาก           /        \
            แก้ไขไม่ได้      RuntimeException   (Checked Exceptions)
            เช่น OutOfMemory)   /    |    \        เช่น IOException,
                    NullPointer  Arith  IndexOutOf   SQLException,
                    Exception    metic  Bounds       ClassNotFoundException
                                Exception Exception
```

- **`Throwable`**: คลาสบนสุดของทุกสิ่งที่ throw/catch ได้ในภาษา Java
- **`Error`**: ปัญหาระดับร้ายแรงที่โปรแกรมทั่วไป**ไม่ควรพยายาม catch** เช่น
  `OutOfMemoryError`, `StackOverflowError` (มักเกิดจากปัญหาระดับ JVM หรือ hardware)
- **`Exception`**: ปัญหาที่โปรแกรมสามารถคาดการณ์และจัดการได้ แบ่งเป็น 2 กลุ่มย่อย
  ที่สำคัญมากในหัวข้อถัดไป

## 3. Checked vs Unchecked Exceptions

| ประเภท | Checked Exception | Unchecked Exception (RuntimeException) |
|---|---|---|
| ต้องจัดการหรือประกาศหรือไม่ | **ต้อง** (compiler บังคับ) | ไม่บังคับ |
| สืบทอดจาก | `Exception` (แต่ไม่ใช่ `RuntimeException`) | `RuntimeException` |
| ตัวอย่าง | `IOException`, `SQLException`, `ClassNotFoundException` | `NullPointerException`, `ArithmeticException`, `ArrayIndexOutOfBoundsException`, `IllegalArgumentException` |
| ใช้เมื่อ | ปัญหาที่คาดว่าจะเกิดจากภายนอก (I/O, network, database) ที่เรียกใช้ควรเตรียมรับมือ | ข้อผิดพลาดจาก bug ในโค้ด หรือการใช้งาน API ผิดวิธี |

```java
import java.io.FileReader;
import java.io.IOException;

public class CheckedExceptionDemo {
    public static void main(String[] args) {
        // Checked Exception: compiler บังคับให้ต้อง try-catch หรือประกาศ throws
        try {
            FileReader reader = new FileReader("ไม่มีไฟล์นี้จริง.txt");
        } catch (IOException e) {
            System.out.println("ไม่พบไฟล์: " + e.getMessage());
        }

        // Unchecked Exception: compiler ไม่บังคับ (แต่ก็ควรจัดการอยู่ดีถ้าคาดว่าจะเกิดได้)
        int[] arr = {1, 2, 3};
        int index = 10;
        System.out.println(arr[index]); // compile ผ่านสบาย ๆ แต่จะพังตอน runtime
    }
}
```

## 4. `try-catch` พื้นฐาน

```java
public class TryCatchDemo {
    public static void main(String[] args) {
        System.out.println("เริ่มโปรแกรม");

        try {
            int result = 10 / 0; // จุดที่คาดว่าอาจเกิด exception
            System.out.println("ผลลัพธ์: " + result); // จะไม่ถูกรัน
        } catch (ArithmeticException e) {
            System.out.println("เกิดข้อผิดพลาด: " + e.getMessage());
            System.out.println("ประเภทของ exception: " + e.getClass().getName());
        }

        System.out.println("โปรแกรมทำงานต่อได้ตามปกติ"); // ยังรันต่อได้ ไม่พัง!
    }
}
```

ผลลัพธ์:

```
เริ่มโปรแกรม
เกิดข้อผิดพลาด: / by zero
ประเภทของ exception: java.lang.ArithmeticException
โปรแกรมทำงานต่อได้ตามปกติ
```

### เมธอดสำคัญของ Exception object

```java
public class ExceptionMethodsDemo {
    public static void main(String[] args) {
        try {
            String text = null;
            text.length();
        } catch (NullPointerException e) {
            System.out.println("getMessage(): " + e.getMessage());
            System.out.println("toString(): " + e.toString());
            e.printStackTrace(); // พิมพ์ stack trace เต็มรูปแบบ (มักใช้ตอน debug)
        }
    }
}
```

## 5. หลาย `catch` blocks และ Multi-catch

สามารถดักจับ exception หลายชนิดได้ โดยเรียงจาก**เฉพาะเจาะจงไปหากว้าง** (subclass ต้อง
มาก่อน superclass เสมอ ไม่งั้น compile error เพราะ catch ที่กว้างกว่าจะครอบคลุมไปแล้ว):

```java
public class MultipleCatchDemo {
    static void process(int[] arr, int index, int divisor) {
        try {
            int value = arr[index];
            int result = value / divisor;
            System.out.println("ผลลัพธ์: " + result);
        } catch (ArrayIndexOutOfBoundsException e) {
            System.out.println("Index ไม่ถูกต้อง: " + e.getMessage());
        } catch (ArithmeticException e) {
            System.out.println("คำนวณผิดพลาด: " + e.getMessage());
        } catch (Exception e) {
            // catch-all: ดักจับ exception อื่น ๆ ที่ไม่คาดคิด (วางไว้ล่างสุดเสมอ)
            System.out.println("เกิดข้อผิดพลาดอื่น ๆ: " + e.getMessage());
        }
    }

    public static void main(String[] args) {
        int[] numbers = {10, 20, 30};
        process(numbers, 1, 0);   // ArithmeticException
        process(numbers, 10, 2);  // ArrayIndexOutOfBoundsException
        process(numbers, 1, 5);   // ทำงานปกติ
    }
}
```

### Multi-catch Syntax (Java 7+)

ถ้าต้องการจัดการหลาย exception ด้วยโค้ดชุดเดียวกัน ใช้ `|` คั่นในบรรทัด catch เดียว:

```java
public class MultiCatchDemo {
    static void riskyOperation(int choice) {
        try {
            if (choice == 1) {
                throw new IllegalArgumentException("ค่าที่ 1 ไม่ถูกต้อง");
            } else {
                throw new IllegalStateException("สถานะที่ 2 ไม่ถูกต้อง");
            }
        } catch (IllegalArgumentException | IllegalStateException e) {
            // จัดการทั้งสองชนิดด้วยโค้ดเดียวกัน (ตัวแปร e เป็น effectively final)
            System.out.println("เกิดข้อผิดพลาด: " + e.getMessage());
        }
    }

    public static void main(String[] args) {
        riskyOperation(1);
        riskyOperation(2);
    }
}
```

## 6. `finally` block

`finally` เป็น block ที่**ทำงานเสมอ** ไม่ว่าจะเกิด exception หรือไม่ก็ตาม (ยกเว้น
JVM หยุดกะทันหันด้วย `System.exit()` หรือเครื่องดับ) มักใช้ปิดทรัพยากร (resource)
เช่น ไฟล์, การเชื่อมต่อฐานข้อมูล, socket

```java
public class FinallyDemo {
    public static void main(String[] args) {
        try {
            System.out.println("เปิดการเชื่อมต่อ...");
            int result = 10 / 0;
        } catch (ArithmeticException e) {
            System.out.println("จับข้อผิดพลาดได้: " + e.getMessage());
        } finally {
            System.out.println("ปิดการเชื่อมต่อ (ทำงานเสมอไม่ว่าจะเกิด exception หรือไม่)");
        }

        System.out.println("--- กรณีไม่เกิด exception ---");
        try {
            System.out.println("ทำงานปกติ");
        } finally {
            System.out.println("finally ก็ยังทำงานอยู่ดี แม้ไม่มี exception");
        }
    }
}
```

### `try-with-resources` (แนะนำมากกว่า finally สำหรับปิด resource — Part 21 จะลงลึก)

```java
import java.io.BufferedReader;
import java.io.FileReader;
import java.io.IOException;

public class TryWithResourcesPreviewDemo {
    public static void main(String[] args) {
        // resource (BufferedReader) จะถูกปิดให้อัตโนมัติเมื่อออกจาก try block
        // ไม่ว่าจะสำเร็จหรือเกิด exception ก็ตาม (ไม่ต้องเขียน finally { reader.close(); } เอง)
        try (BufferedReader reader = new BufferedReader(new FileReader("data.txt"))) {
            String line = reader.readLine();
            System.out.println(line);
        } catch (IOException e) {
            System.out.println("อ่านไฟล์ไม่สำเร็จ: " + e.getMessage());
        }
    }
}
```

## 7. `throw` และ `throws`

- **`throw`**: ใช้ **สั่งให้เกิด exception ขึ้นมาเอง** ณ จุดนั้น (มักใช้กับ validation)
- **`throws`**: ใช้ใน method signature เพื่อ**ประกาศว่าเมธอดนี้อาจโยน exception นี้ออกไป**
  (บังคับผู้เรียกใช้ต้องจัดการ ถ้าเป็น checked exception)

```java
public class ThrowThrowsDemo {
    // throws ประกาศว่าเมธอดนี้อาจโยน checked exception ออกไป
    static void readConfig(String path) throws java.io.IOException {
        if (path == null || path.isEmpty()) {
            throw new IllegalArgumentException("path ห้ามว่าง"); // throw สร้าง exception เอง
        }
        // ... โค้ดอ่านไฟล์จริง (จะเรียนใน Part 36) ...
        throw new java.io.IOException("ไฟล์ไม่พบ: " + path); // จำลองว่าอ่านไฟล์ไม่สำเร็จ
    }

    public static void main(String[] args) {
        try {
            readConfig("");
        } catch (IllegalArgumentException e) {
            System.out.println("ข้อมูลนำเข้าไม่ถูกต้อง: " + e.getMessage());
        } catch (java.io.IOException e) {
            System.out.println("ปัญหาเรื่องไฟล์: " + e.getMessage());
        }
    }
}
```

### การสร้าง Custom Validation ด้วย `throw` (รูปแบบที่พบบ่อยมาก)

```java
public class ValidationDemo {
    static void setAge(int age) {
        if (age < 0 || age > 150) {
            throw new IllegalArgumentException("อายุต้องอยู่ระหว่าง 0-150 ปี, ได้รับ: " + age);
        }
        System.out.println("ตั้งค่าอายุเป็น " + age + " สำเร็จ");
    }

    public static void main(String[] args) {
        setAge(25); // ทำงานปกติ

        try {
            setAge(-5); // จะ throw IllegalArgumentException
        } catch (IllegalArgumentException e) {
            System.out.println("ข้อผิดพลาด: " + e.getMessage());
        }
    }
}
```

## 8. Exception ที่พบบ่อยและวิธีป้องกัน

| Exception | สาเหตุ | วิธีป้องกัน |
|---|---|---|
| `NullPointerException` | เรียกเมธอดหรือเข้าถึง field ของตัวแปรที่เป็น `null` | เช็ค `!= null` ก่อนใช้งาน, ใช้ `Optional` (Part 43) |
| `ArrayIndexOutOfBoundsException` | เข้าถึง index ที่ไม่มีอยู่จริงใน array | ตรวจสอบ `index < array.length` ก่อนเสมอ |
| `NumberFormatException` | แปลง String ที่ไม่ใช่ตัวเลขด้วย `Integer.parseInt()` | ตรวจสอบรูปแบบก่อนแปลง หรือใช้ try-catch |
| `ArithmeticException` | หารด้วยศูนย์ (เฉพาะ integer division) | ตรวจสอบ divisor ก่อนหารเสมอ |
| `ClassCastException` | cast object ผิดชนิดที่ไม่ใช่ subtype จริง | ใช้ `instanceof` ตรวจสอบก่อน cast |
| `ConcurrentModificationException` | แก้ไข collection ระหว่างวน loop ด้วย for-each | ใช้ `Iterator.remove()` หรือ `CopyOnWriteArrayList` |

```java
import java.util.Objects;

public class CommonExceptionsPreventionDemo {
    public static void main(String[] args) {
        // ป้องกัน NumberFormatException
        String userInput = "abc123";
        try {
            int number = Integer.parseInt(userInput);
        } catch (NumberFormatException e) {
            System.out.println("'" + userInput + "' ไม่ใช่ตัวเลขที่ถูกต้อง");
        }

        // ป้องกัน NullPointerException ด้วย Objects.requireNonNull
        try {
            String name = null;
            Objects.requireNonNull(name, "name ห้ามเป็น null");
        } catch (NullPointerException e) {
            System.out.println(e.getMessage());
        }

        // ป้องกัน ClassCastException ด้วย instanceof
        Object obj = "text";
        if (obj instanceof Integer) {
            Integer num = (Integer) obj;
        } else {
            System.out.println("obj ไม่ใช่ Integer ข้ามการ cast");
        }
    }
}
```

## 9. แนวปฏิบัติที่ดีในการจัดการ Exception

1. **อย่า catch แบบเงียบ (silent catch)**: การจับ exception แล้วไม่ทำอะไรเลยทำให้
   หา bug ยากมาก

```java
// แย่มาก: กลืน exception เงียบ ๆ ไม่มีใครรู้ว่าเกิดอะไรขึ้น
try {
    riskyOperation();
} catch (Exception e) {
    // ว่างเปล่า - ห้ามทำแบบนี้เด็ดขาด!
}
```

2. **catch เฉพาะเจาะจงเท่าที่จำเป็น** อย่า catch `Exception` กว้าง ๆ ถ้าไม่จำเป็น
3. **อย่าใช้ exception แทน control flow ปกติ** (เช่น ใช้ exception แทน if-else)
   เพราะ exception มี performance overhead และทำให้โค้ดอ่านยาก
4. **ให้ error message ที่ชัดเจน มีประโยชน์** ระบุว่าเกิดอะไรขึ้นและควรทำอย่างไรต่อ
5. **Log exception เสมอ** อย่างน้อยควร print stack trace หรือใช้ logging framework
   (Part 63) ในโค้ด production

```java
public class GoodExceptionPracticeDemo {
    static void processOrder(String orderId) {
        try {
            if (orderId == null || orderId.isBlank()) {
                throw new IllegalArgumentException("orderId ห้ามว่างเปล่า");
            }
            System.out.println("กำลังประมวลผลออร์เดอร์: " + orderId);
        } catch (IllegalArgumentException e) {
            System.err.println("[ERROR] ไม่สามารถประมวลผลออร์เดอร์ได้: " + e.getMessage());
            // ในโปรแกรมจริงควร log ผ่าน logging framework แทน System.err
        }
    }

    public static void main(String[] args) {
        processOrder("ORDER-001");
        processOrder("");
    }
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียนเมธอด `safeDivide(int a, int b)` ที่คืนค่า `0` แทนที่จะ crash เมื่อหารด้วย 0

**เฉลย:**

```java
public class Exercise1 {
    static int safeDivide(int a, int b) {
        try {
            return a / b;
        } catch (ArithmeticException e) {
            System.out.println("ไม่สามารถหารด้วยศูนย์ได้ คืนค่า 0 แทน");
            return 0;
        }
    }

    public static void main(String[] args) {
        System.out.println(safeDivide(10, 2)); // 5
        System.out.println(safeDivide(10, 0)); // 0
    }
}
```

**2)** เขียนเมธอด `parseAgeOrDefault(String input, int defaultValue)` ที่พยายามแปลง
String เป็นอายุ ถ้าแปลงไม่ได้ให้คืนค่า default

**เฉลย:**

```java
public class Exercise2 {
    static int parseAgeOrDefault(String input, int defaultValue) {
        try {
            return Integer.parseInt(input);
        } catch (NumberFormatException e) {
            return defaultValue;
        }
    }

    public static void main(String[] args) {
        System.out.println(parseAgeOrDefault("25", 0));   // 25
        System.out.println(parseAgeOrDefault("abc", -1)); // -1
    }
}
```

**3)** เขียนเมธอดตรวจสอบรหัสผ่านที่ throw `IllegalArgumentException` ถ้ารหัสผ่าน
สั้นกว่า 8 ตัวอักษร แล้วทดสอบด้วย try-catch

**เฉลย:**

```java
public class Exercise3 {
    static void validatePassword(String password) {
        if (password.length() < 8) {
            throw new IllegalArgumentException("รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร");
        }
        System.out.println("รหัสผ่านถูกต้อง");
    }

    public static void main(String[] args) {
        try {
            validatePassword("123");
        } catch (IllegalArgumentException e) {
            System.out.println("ผิดพลาด: " + e.getMessage());
        }
        validatePassword("securePass123");
    }
}
```

### สรุปเนื้อหา Part 10

- Exception คือเหตุการณ์ผิดปกติที่ทำให้โปรแกรมหยุดชะงัก ต้องจัดการเพื่อความ robust
- Checked exceptions ต้องจัดการหรือประกาศ `throws` เสมอ, Unchecked (RuntimeException)
  ไม่บังคับแต่ก็ควรจัดการถ้าคาดว่าจะเกิด
- `try-catch` ดักจับข้อผิดพลาด, `finally` ทำงานเสมอไม่ว่าจะเกิด exception หรือไม่
- `throw` สร้าง exception เอง, `throws` ประกาศว่าเมธอดอาจโยน exception ออกไป
- เรียง catch จากเฉพาะเจาะจงไปกว้างเสมอ ห้ามจับ exception แบบเงียบ ๆ โดยไม่ทำอะไร

**ต่อไป**: [Part 11 — Classes และ Objects เบื้องต้น](./part-011-classes-and-objects.md)
