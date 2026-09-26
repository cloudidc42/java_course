# Part 52: Annotations: การใช้งานและการสร้าง Custom Annotation

> ขั้นตอนที่ 511-520 ของหลักสูตร | ระดับ: สูง

## สารบัญ

1. Annotation คืออะไร ใช้ทำอะไร
2. Annotation มาตรฐานที่ใช้บ่อย
3. การสร้าง Custom Annotation
4. Meta-Annotations: `@Retention`, `@Target`
5. `@Retention` แบบละเอียด: SOURCE, CLASS, RUNTIME
6. Annotation ที่มี Element (พารามิเตอร์)
7. การอ่าน Annotation ด้วย Reflection (ปูทางสู่ Part 53)
8. `@Repeatable` Annotations
9. กรณีใช้งานจริงของ Custom Annotation
10. แบบฝึกหัดและสรุป

---

## 1. Annotation คืออะไร ใช้ทำอะไร

**Annotation** (Java 5+) คือ**metadata**ที่แนบไปกับโค้ด (class, method,
field, parameter) — ไม่มีผลต่อ logic การทำงานโดยตรง แต่ให้ข้อมูลเพิ่มเติมที่
**compiler, tool, หรือ framework** สามารถนำไปประมวลผลได้ เราใช้ annotation
มาตลอดทั้งหลักสูตรแล้ว (`@Override` จาก Part 14, `@FunctionalInterface` จาก
Part 16) — Part นี้จะสอนวิธีสร้างเองและเข้าใจกลไกเบื้องหลัง

```java
public class AnnotationMotivationDemo {
    @Override // annotation บอก compiler ว่า "ตั้งใจ override" (ทบทวนจาก Part 14)
    public String toString() {
        return "example";
    }

    @Deprecated // annotation บอกว่า method นี้ไม่ควรใช้อีกแล้ว
    public void oldMethod() {
        System.out.println("เมธอดเก่าที่ไม่แนะนำให้ใช้");
    }

    @SuppressWarnings("unchecked") // สั่งให้ compiler ไม่แสดง warning ประเภทที่ระบุ
    public void methodWithWarning() {
        java.util.List list = new java.util.ArrayList(); // raw type - จะมี warning ปกติ
    }
}
```

## 2. Annotation มาตรฐานที่ใช้บ่อย

| Annotation | ความหมาย |
|---|---|
| `@Override` | บอกว่าตั้งใจ override method จาก superclass/interface (ทบทวน Part 14) |
| `@Deprecated` | บอกว่า element นี้ไม่ควรใช้อีกแล้ว (compiler จะแสดง warning เมื่อมีการเรียกใช้) |
| `@SuppressWarnings` | ปิด warning ประเภทที่ระบุจาก compiler |
| `@FunctionalInterface` | บอกว่า interface นี้ต้องมี abstract method เดียว (ทบทวน Part 16) |
| `@SafeVarargs` | บอกว่า varargs method นี้ปลอดภัยจาก heap pollution (เกี่ยวกับ generics) |

```java
public class DeprecatedDemo {
    /**
     * @deprecated ใช้ {@link #newMethod()} แทน
     */
    @Deprecated(since = "2.0", forRemoval = true) // ระบุรายละเอียดเพิ่มเติมได้ (Java 9+)
    public void oldMethod() { }

    public void newMethod() { }

    public static void main(String[] args) {
        DeprecatedDemo demo = new DeprecatedDemo();
        demo.oldMethod(); // compiler จะแสดง warning: "oldMethod() has been deprecated"
    }
}
```

## 3. การสร้าง Custom Annotation

สร้าง annotation ด้วย keyword `@interface`:

```java
public @interface Author {
    String name();
    String date();
}
```

```java
public class DocumentedClass {
    @Author(name = "Somchai", date = "2024-01-15") // ใช้งาน annotation ที่สร้างเอง
    public void importantMethod() {
        System.out.println("เมธอดสำคัญ");
    }
}
```

**ข้อสำคัญ**: annotation ที่สร้างเองแบบนี้**ยังไม่มีผลอะไรเลย**ถ้าไม่มีอะไร
มาอ่านมันตอน runtime หรือ compile-time — annotation เป็นแค่ "ป้ายกำกับ"
พลังของมันอยู่ที่**เครื่องมือหรือโค้ดที่มาอ่านและประมวลผลมัน** (หัวข้อ 7)

## 4. Meta-Annotations: `@Retention`, `@Target`

**Meta-Annotation** คือ annotation ที่ใช้กำกับ**annotation อื่น** (annotation
ของ annotation) — สองตัวที่สำคัญที่สุด:

```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME) // annotation นี้จะเก็บไว้ให้อ่านได้ตอน runtime (หัวข้อ 5)
@Target(ElementType.METHOD)          // annotation นี้ใช้ได้แค่กับ method เท่านั้น
public @interface LogExecutionTime {
}
```

**`@Target`** จำกัดว่า annotation ใช้ได้กับ element ประเภทไหนบ้าง:

| ElementType | ใช้กับ |
|---|---|
| `TYPE` | class, interface, enum, record |
| `FIELD` | field |
| `METHOD` | method |
| `PARAMETER` | parameter ของ method |
| `CONSTRUCTOR` | constructor |
| `LOCAL_VARIABLE` | ตัวแปร local |
| `ANNOTATION_TYPE` | annotation อื่น (สำหรับสร้าง meta-annotation) |

```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Target;

@Target({ElementType.METHOD, ElementType.FIELD}) // ใช้ได้ทั้งสองแบบ (array ของ ElementType)
public @interface MultiTargetAnnotation {
}
```

## 5. `@Retention` แบบละเอียด: SOURCE, CLASS, RUNTIME

**`RetentionPolicy`** กำหนดว่า annotation จะ**"อยู่ต่อ"ไปถึงขั้นไหน**ของ
lifecycle ของโค้ด:

| Policy | คงอยู่ถึงขั้นไหน | ตัวอย่างการใช้ |
|---|---|---|
| `SOURCE` | แค่ source code (`.java`) หายไปหลัง compile | `@Override`, `@SuppressWarnings` |
| `CLASS` | อยู่ใน `.class` file แต่ JVM ไม่โหลดตอน runtime (ค่า default) | เครื่องมือวิเคราะห์ bytecode |
| `RUNTIME` | อยู่ตลอดถึง runtime อ่านผ่าน Reflection ได้ (Part 53) | Framework เช่น Spring (`@Autowired`), JUnit (`@Test`) |

```java
public class RetentionDemo {
    public static void main(String[] args) {
        // @Override มี RetentionPolicy.SOURCE - หายไปหลัง compile, ไม่มีผลตอน runtime เลย
        // ใช้แค่ให้ compiler ตรวจสอบตอน compile-time เท่านั้น (ทบทวนจาก Part 14)

        // @LogExecutionTime (ที่เราสร้างในหัวข้อ 4) มี RetentionPolicy.RUNTIME
        // สามารถอ่านได้ด้วย Reflection API ตอน runtime (Part 53 จะสอนวิธีอ่าน)
    }
}
```

**หลักการเลือก**: ใช้ `RUNTIME` เมื่อต้องการให้ framework/library อ่าน
annotation ตอนโปรแกรมรันอยู่ (กรณีใช้บ่อยที่สุดสำหรับ custom annotation)

## 6. Annotation ที่มี Element (พารามิเตอร์)

```java
import java.lang.annotation.*;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Test {
    String description() default ""; // element ที่มีค่า default (ไม่บังคับต้องระบุ)
    int timeout() default 1000;
    boolean enabled() default true;
}
```

```java
public class TestAnnotationDemo {
    @Test(description = "ทดสอบการบวก", timeout = 500)
    public void testAddition() {
        System.out.println("ทดสอบ: 1+1=2");
    }

    @Test // ไม่ระบุ element ใด ๆ ใช้ค่า default ทั้งหมด
    public void testWithDefaults() {
        System.out.println("ทดสอบด้วยค่า default");
    }

    @Test(enabled = false)
    public void skippedTest() {
        System.out.println("ทดสอบนี้ถูก skip");
    }
}
```

**Element พิเศษชื่อ `value()`**: ถ้า annotation มี element ชื่อ `value` เพียง
ตัวเดียว (หรือเป็น element เดียวที่ไม่มีค่า default) สามารถละชื่อ element ได้:

```java
public @interface Author {
    String value(); // ชื่อ element พิเศษ "value"
}
```

```java
public class ValueElementDemo {
    @Author("Somchai") // เขียนสั้น ๆ ได้ เพราะ element ชื่อ "value" (เทียบเท่า @Author(value = "Somchai"))
    public void method() { }
}
```

## 7. การอ่าน Annotation ด้วย Reflection (ปูทางสู่ Part 53)

Annotation ที่มี `@Retention(RUNTIME)` สามารถอ่านได้ตอน runtime ผ่าน
**Reflection API** (จะเรียนเต็มรูปแบบใน Part 53) — นี่คือกลไกที่ framework
อย่าง Spring และ JUnit ใช้ในการ**ค้นหาและเรียก method ที่มี annotation ที่
กำหนด**โดยอัตโนมัติ

```java
import java.lang.reflect.Method;

public class ReadAnnotationDemo {
    public static void main(String[] args) throws Exception {
        Class<?> clazz = TestAnnotationDemo.class;

        for (Method method : clazz.getDeclaredMethods()) {
            if (method.isAnnotationPresent(Test.class)) { // เช็คว่า method นี้มี @Test หรือไม่
                Test testAnnotation = method.getAnnotation(Test.class); // ดึง annotation object ออกมา

                if (testAnnotation.enabled()) { // อ่านค่า element ของ annotation
                    System.out.println("กำลังรัน: " + method.getName()
                            + " (" + testAnnotation.description() + ")");
                    method.invoke(new TestAnnotationDemo()); // เรียก method นั้นแบบ dynamic
                } else {
                    System.out.println("ข้าม: " + method.getName());
                }
            }
        }
    }
}
```

**นี่คือหลักการเบื้องหลัง JUnit ทั้งหมด** (Part 58) — JUnit สแกนหา method ที่
มี `@Test` annotation แล้วเรียกให้อัตโนมัติ ไม่ต้องเขียน `main()` เรียกเองเลย

## 8. `@Repeatable` Annotations

ปกติ annotation เดียวใช้ซ้ำกับ element เดียวไม่ได้มากกว่า 1 ครั้ง — Java 8
เพิ่ม `@Repeatable` ให้ทำได้:

```java
import java.lang.annotation.*;

@Repeatable(Schedules.class) // ต้องระบุ "container annotation" ที่จะเก็บหลาย ๆ ตัว
public @interface Schedule {
    String day();
}

@Retention(RetentionPolicy.RUNTIME)
public @interface Schedules {
    Schedule[] value(); // container ต้องมี element ชื่อ value เป็น array ของ annotation ที่ repeat
}
```

```java
public class RepeatableDemo {
    @Schedule(day = "Monday")
    @Schedule(day = "Wednesday")
    @Schedule(day = "Friday")
    public void weeklyMeeting() { }

    public static void main(String[] args) throws Exception {
        var method = RepeatableDemo.class.getMethod("weeklyMeeting");
        Schedule[] schedules = method.getAnnotationsByType(Schedule.class);
        for (Schedule s : schedules) {
            System.out.println("ประชุมวัน: " + s.day());
        }
    }
}
```

## 9. กรณีใช้งานจริงของ Custom Annotation

**ตัวอย่างสร้าง Custom Annotation สำหรับ Validation** (แนวคิดเดียวกับ Bean
Validation ที่จะเรียนใน Part 81):

```java
import java.lang.annotation.*;
import java.lang.reflect.Field;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface NotBlank {
    String message() default "ห้ามเป็นค่าว่าง";
}

public class SimpleValidator {
    static void validate(Object obj) throws Exception {
        for (Field field : obj.getClass().getDeclaredFields()) {
            if (field.isAnnotationPresent(NotBlank.class)) {
                field.setAccessible(true); // เข้าถึง private field ได้ผ่าน reflection
                Object value = field.get(obj);
                if (value == null || value.toString().isBlank()) {
                    NotBlank annotation = field.getAnnotation(NotBlank.class);
                    throw new IllegalArgumentException(
                        field.getName() + ": " + annotation.message());
                }
            }
        }
    }
}

public class User {
    @NotBlank(message = "ชื่อผู้ใช้ห้ามว่าง")
    String username;

    public User(String username) {
        this.username = username;
    }
}
```

```java
public class SimpleValidatorDemo {
    public static void main(String[] args) throws Exception {
        SimpleValidator.validate(new User("somchai")); // ผ่าน ไม่มี error

        try {
            SimpleValidator.validate(new User(""));
        } catch (IllegalArgumentException e) {
            System.out.println("Validation ล้มเหลว: " + e.getMessage());
        }
    }
}
```

**นี่คือหลักการเบื้องหลัง `@NotBlank`, `@NotNull`, `@Size` ของ Bean Validation
(Jakarta Validation)** ที่ใช้จริงใน Spring Boot (Part 81) — เพียงแค่ระบบจริง
มีความซับซ้อนและครอบคลุมมากกว่าตัวอย่างง่าย ๆ นี้มาก

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** สร้าง custom annotation `@Priority(int value)` ที่ใช้กับ method แล้ว
เขียนโปรแกรมที่สแกนหา method ทั้งหมดใน class ที่มี annotation นี้ เรียงตาม
priority

**เฉลย:**

```java
import java.lang.annotation.*;
import java.lang.reflect.Method;
import java.util.*;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@interface Priority {
    int value();
}

class Tasks {
    @Priority(3) public void low() { System.out.println("low"); }
    @Priority(1) public void high() { System.out.println("high"); }
    @Priority(2) public void medium() { System.out.println("medium"); }
}

public class Exercise1 {
    public static void main(String[] args) throws Exception {
        List<Method> methods = new ArrayList<>(List.of(Tasks.class.getDeclaredMethods()));
        methods.sort(Comparator.comparingInt(m -> m.getAnnotation(Priority.class).value()));

        Tasks tasks = new Tasks();
        for (Method m : methods) {
            m.invoke(tasks); // high, medium, low
        }
    }
}
```

**2)** อธิบายว่าทำไม `@Override` ไม่มีผลต่อการทำงานตอน runtime เลย

**เฉลย**: `@Override` มี `@Retention(RetentionPolicy.SOURCE)` หมายความว่า
annotation นี้**ถูกทิ้งไปตั้งแต่ตอน compile** ไม่ปรากฏอยู่ใน bytecode
(`.class` file) เลย มันมีประโยชน์เฉพาะช่วง**compile-time**เท่านั้น ที่ให้
compiler ตรวจสอบว่า method signature ตรงกับ method ในคลาสแม่จริงหรือไม่
(ทบทวนจาก Part 14) — เมื่อ compile เสร็จแล้ว annotation นี้ก็ไม่มีตัวตนอีก
ต่อไป จึงไม่มีทางอ่านหรือใช้ประโยชน์จากมันตอน runtime ได้เลย

**3)** ทำไม framework อย่าง JUnit และ Spring ถึงต้องใช้ `RetentionPolicy.
RUNTIME` สำหรับ annotation ของตัวเอง (เช่น `@Test`, `@Autowired`)

**เฉลย**: เพราะ framework เหล่านี้ต้อง**สแกนหา annotation ตอนโปรแกรมกำลัง
รันอยู่จริง**ผ่าน Reflection API (ทบทวนหัวข้อ 7) เพื่อรู้ว่า method ไหนคือ
test case ที่ต้องรัน (JUnit) หรือ field ไหนที่ต้อง inject dependency เข้าไป
(Spring) — ถ้าใช้ `SOURCE` หรือ `CLASS` retention annotation จะไม่ปรากฏให้
Reflection API เห็นตอน runtime เลย ทำให้ framework ไม่สามารถทำงานตามที่
ออกแบบไว้ได้

### สรุปเนื้อหา Part 52

- Annotation เป็น metadata ที่แนบกับโค้ด ไม่มีผลต่อ logic โดยตรง แต่ tool/
  framework นำไปประมวลผลได้
- สร้าง custom annotation ด้วย `@interface`, กำกับด้วย meta-annotation
  `@Retention` และ `@Target`
- `RetentionPolicy.RUNTIME` จำเป็นถ้าต้องการอ่าน annotation ผ่าน Reflection
  ตอน runtime (framework ส่วนใหญ่ใช้แบบนี้)
- Annotation มี element (พารามิเตอร์) พร้อมค่า default ได้, element ชื่อ
  `value` ละชื่อได้ตอนใช้งาน
- `@Repeatable` ให้ annotation เดียวใช้ซ้ำได้หลายครั้งบน element เดียวกัน
- Custom annotation + Reflection เป็นหลักการเบื้องหลัง JUnit, Spring และ
  Bean Validation

**ต่อไป**: [Part 53 — Reflection API](./part-053-reflection.md)
