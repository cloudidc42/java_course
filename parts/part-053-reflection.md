# Part 53: Reflection API

> ขั้นตอนที่ 521-530 ของหลักสูตร | ระดับ: สูง

## สารบัญ

1. Reflection คืออะไร ใช้ทำอะไร
2. `Class<T>`: จุดเริ่มต้นของ Reflection
3. การตรวจสอบ Fields ด้วย Reflection
4. การตรวจสอบและเรียก Methods ด้วย Reflection
5. การสร้าง Object ด้วย Reflection
6. การเข้าถึง Private Members
7. การตรวจสอบ Constructors
8. ประสิทธิภาพและความเสี่ยงของ Reflection
9. กรณีใช้งานจริง: Simple Dependency Injection Container
10. แบบฝึกหัดและสรุป

---

## 1. Reflection คืออะไร ใช้ทำอะไร

**Reflection** คือความสามารถของโปรแกรม**ตรวจสอบและแก้ไข structure ของตัวเอง
ตอน runtime** — สามารถ**ดูว่า class มี field/method อะไรบ้าง, สร้าง object,
เรียก method, และเข้าถึง private field** ได้ทั้งหมด**โดยไม่ต้องรู้ชนิดข้อมูล
ล่วงหน้าตอน compile-time**

นี่คือกลไกเบื้องหลังของ framework สำคัญมากมาย: Spring (dependency injection),
JUnit/Mockito (testing), Jackson/Gson (JSON serialization — Part 70),
Hibernate (ORM — Part 80)

```java
public class ReflectionMotivationDemo {
    public static void main(String[] args) throws Exception {
        // ปกติเราต้องรู้ชนิดข้อมูลตอน compile-time เพื่อสร้าง object
        String normalString = new String("Hello"); // รู้ชนิดแน่นอน

        // Reflection ทำให้สร้าง object จาก "ชื่อ class" ที่เป็น String ได้ (ไม่รู้ชนิดตอน compile!)
        Class<?> clazz = Class.forName("java.lang.String");
        Object dynamicString = clazz.getDeclaredConstructor().newInstance();
        System.out.println(dynamicString.getClass().getName()); // java.lang.String

        // สถานการณ์นี้เกิดขึ้นบ่อยมากใน framework: ผู้ใช้ระบุชื่อ class ใน config file
        // (เช่น application.properties) แล้ว framework ต้องสร้าง object นั้นตอน runtime
    }
}
```

## 2. `Class<T>`: จุดเริ่มต้นของ Reflection

**`Class<T>`** คือ object ที่แทน "ชนิดข้อมูล" ของ class นั้น (ทุก class มี
`Class` object ของตัวเองเพียงหนึ่งตัวเท่านั้น ไม่ว่าจะสร้าง instance กี่ตัว)

```java
public class ClassObjectDemo {
    public static void main(String[] args) throws ClassNotFoundException {
        // 3 วิธีในการได้ Class object
        Class<?> clazz1 = String.class;                    // วิธีที่ 1: .class literal (compile-time known)
        Class<?> clazz2 = "hello".getClass();               // วิธีที่ 2: จาก instance ที่มีอยู่แล้ว
        Class<?> clazz3 = Class.forName("java.lang.String"); // วิธีที่ 3: จากชื่อ String (runtime known)

        System.out.println(clazz1 == clazz2); // true (Class object เดียวกันเสมอสำหรับ class เดียวกัน)
        System.out.println(clazz1 == clazz3);  // true

        System.out.println(clazz1.getName());          // java.lang.String
        System.out.println(clazz1.getSimpleName());      // String (ไม่มี package)
        System.out.println(clazz1.getPackageName());      // java.lang
        System.out.println(clazz1.isInterface());           // false
        System.out.println(clazz1.getSuperclass());          // class java.lang.Object

        // ตรวจสอบ interface ที่ implement
        for (Class<?> iface : clazz1.getInterfaces()) {
            System.out.println("implements: " + iface.getSimpleName());
        }
    }
}
```

## 3. การตรวจสอบ Fields ด้วย Reflection

```java
import java.lang.reflect.Field;

public class Person {
    private String name;
    public int age;
    protected double salary;
}
```

```java
public class FieldReflectionDemo {
    public static void main(String[] args) throws Exception {
        Class<?> clazz = Person.class;

        // getDeclaredFields(): ได้ field ทั้งหมดที่ประกาศใน class นี้ (รวม private ด้วย ไม่รวมของ superclass)
        for (Field field : clazz.getDeclaredFields()) {
            System.out.println(field.getName() + " : " + field.getType().getSimpleName());
        }
        // name : String
        // age : int
        // salary : double

        // getFields(): ได้แค่ field ที่ public (รวมของ superclass ด้วย)
        System.out.println("Public fields เท่านั้น:");
        for (Field field : clazz.getFields()) {
            System.out.println(field.getName());
        }
        // age (แค่ตัวเดียว เพราะ name เป็น private, salary เป็น protected)

        // อ่าน/เขียนค่าใน field ผ่าน reflection
        Person person = new Person();
        Field ageField = clazz.getDeclaredField("age");
        ageField.set(person, 25); // เขียนค่าเข้า field
        System.out.println(ageField.get(person)); // อ่านค่าออกมา: 25
    }
}
```

## 4. การตรวจสอบและเรียก Methods ด้วย Reflection

```java
import java.lang.reflect.Method;

public class Calculator {
    public int add(int a, int b) {
        return a + b;
    }

    private String secretMethod() {
        return "ข้อมูลลับ";
    }
}
```

```java
public class MethodReflectionDemo {
    public static void main(String[] args) throws Exception {
        Class<?> clazz = Calculator.class;
        Calculator calc = new Calculator();

        // ดูรายการ method ทั้งหมด
        for (Method method : clazz.getDeclaredMethods()) {
            System.out.println(method.getName() + " - พารามิเตอร์: " + method.getParameterCount());
        }

        // เรียก method แบบ dynamic ผ่าน reflection
        Method addMethod = clazz.getDeclaredMethod("add", int.class, int.class); // ระบุชื่อ + ชนิดพารามิเตอร์
        Object result = addMethod.invoke(calc, 3, 4); // invoke(instance, args...)
        System.out.println("ผลลัพธ์: " + result); // 7

        // เรียก private method ได้ด้วย (หัวข้อ 6 จะอธิบายเพิ่ม)
        Method secretMethod = clazz.getDeclaredMethod("secretMethod");
        secretMethod.setAccessible(true); // ต้องเปิดสิทธิ์ก่อน เพราะเป็น private
        Object secret = secretMethod.invoke(calc);
        System.out.println(secret); // "ข้อมูลลับ"
    }
}
```

## 5. การสร้าง Object ด้วย Reflection

```java
import java.lang.reflect.Constructor;

public class Product {
    String name;
    double price;

    public Product() { // no-arg constructor
        this.name = "Unknown";
        this.price = 0;
    }

    public Product(String name, double price) { // parameterized constructor
        this.name = name;
        this.price = price;
    }

    @Override
    public String toString() {
        return "Product{name='" + name + "', price=" + price + "}";
    }
}
```

```java
public class ObjectCreationReflectionDemo {
    public static void main(String[] args) throws Exception {
        Class<?> clazz = Product.class;

        // สร้าง object ด้วย no-arg constructor
        Object obj1 = clazz.getDeclaredConstructor().newInstance();
        System.out.println(obj1);

        // สร้าง object ด้วย constructor ที่มีพารามิเตอร์
        Constructor<?> constructor = clazz.getDeclaredConstructor(String.class, double.class);
        Object obj2 = constructor.newInstance("Laptop", 25000.0);
        System.out.println(obj2);
    }
}
```

## 6. การเข้าถึง Private Members

Reflection สามารถ**ข้าม access modifier** (`private`, `protected`) ได้ด้วย
`setAccessible(true)` — ทบทวนแนวคิด Encapsulation จาก Part 13 ซึ่ง Reflection
สามารถ**เจาะทะลุ**ได้ (แม้จะขัดกับหลักการ encapsulation แต่มีประโยชน์มากสำหรับ
framework/testing tools)

```java
import java.lang.reflect.Field;

public class SecretData {
    private String secret = "รหัสลับสุดยอด";
}
```

```java
public class AccessPrivateDemo {
    public static void main(String[] args) throws Exception {
        SecretData data = new SecretData();
        Field secretField = SecretData.class.getDeclaredField("secret");

        // secretField.get(data); // Error! IllegalAccessException - เข้าถึง private field ตรง ๆ ไม่ได้

        secretField.setAccessible(true); // "เปิดประตูหลัง" เข้าถึง private ได้
        String secret = (String) secretField.get(data);
        System.out.println(secret); // "รหัสลับสุดยอด"

        secretField.set(data, "รหัสลับใหม่"); // แก้ไข private field จากภายนอกได้ด้วย!
        System.out.println(secretField.get(data));
    }
}
```

**คำเตือนสำคัญด้านความปลอดภัย**: การใช้ `setAccessible(true)` **ขัดกับหลัก
encapsulation โดยตรง** และอาจถูกจำกัดโดย **Security Manager** หรือ **Module
System** (Part 69) ในสภาพแวดล้อมที่มีการควบคุมความปลอดภัยสูง — ควรใช้เฉพาะใน
กรณีที่จำเป็นจริง ๆ (เช่น เขียน testing framework, ORM) ไม่ใช่ในโค้ด business
logic ทั่วไป

## 7. การตรวจสอบ Constructors

```java
import java.lang.reflect.Constructor;
import java.lang.reflect.Modifier;

public class ConstructorReflectionDemo {
    public static void main(String[] args) throws Exception {
        Class<?> clazz = Product.class;

        for (Constructor<?> constructor : clazz.getDeclaredConstructors()) {
            System.out.print("Constructor พารามิเตอร์: ");
            for (Class<?> paramType : constructor.getParameterTypes()) {
                System.out.print(paramType.getSimpleName() + " ");
            }
            System.out.println("(modifiers: " + Modifier.toString(constructor.getModifiers()) + ")");
        }
        // Constructor พารามิเตอร์:  (modifiers: public)
        // Constructor พารามิเตอร์: String double (modifiers: public)
    }
}
```

## 8. ประสิทธิภาพและความเสี่ยงของ Reflection

**Reflection มี performance overhead สูงกว่าการเรียกโค้ดตรง ๆ อย่างมาก**
เพราะ JVM ไม่สามารถ optimize การเรียกผ่าน reflection ได้ดีเท่าการเรียกแบบ
static (compile-time known) — ทุกครั้งที่เรียกผ่าน `invoke()` ต้องมีการ
ตรวจสอบสิทธิ์และชนิดข้อมูลเพิ่มเติมที่ compiler ปกติทำไว้ล่วงหน้าแล้ว

```java
public class ReflectionPerformanceDemo {
    public static void main(String[] args) throws Exception {
        Calculator calc = new Calculator();

        long start = System.nanoTime();
        for (int i = 0; i < 1_000_000; i++) {
            calc.add(1, 2); // เรียกตรง ๆ - JVM optimize ได้เต็มที่ (JIT compilation)
        }
        System.out.println("เรียกตรง: " + (System.nanoTime() - start) / 1_000_000 + " ms");

        var method = Calculator.class.getDeclaredMethod("add", int.class, int.class);
        start = System.nanoTime();
        for (int i = 0; i < 1_000_000; i++) {
            method.invoke(calc, 1, 2); // เรียกผ่าน reflection - ช้ากว่ามาก
        }
        System.out.println("เรียกผ่าน reflection: " + (System.nanoTime() - start) / 1_000_000 + " ms");
    }
}
```

**หลักปฏิบัติ**: ใช้ reflection เฉพาะเมื่อจำเป็นจริง ๆ (framework-level code,
เครื่องมือ tooling) — ในโค้ด business logic ทั่วไปที่รู้ชนิดข้อมูลล่วงหน้า
**ไม่ควรใช้ reflection** เพราะเสียประสิทธิภาพโดยไม่จำเป็นและทำให้ compiler
ช่วยตรวจสอบ type safety ไม่ได้ (ทบทวนประโยชน์ของ static typing จาก Part 3)

## 9. กรณีใช้งานจริง: Simple Dependency Injection Container

ตัวอย่างที่แสดงให้เห็นว่า Reflection + Custom Annotation (ทบทวนจาก Part 52)
รวมกันเป็นหลักการเบื้องหลังของ **Spring's `@Autowired`** (Part 74):

```java
import java.lang.annotation.*;
import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Map;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface Inject { }

public class SimpleContainer {
    private Map<Class<?>, Object> registry = new HashMap<>();

    public void register(Class<?> type, Object instance) {
        registry.put(type, instance);
    }

    public void injectDependencies(Object target) throws Exception {
        for (Field field : target.getClass().getDeclaredFields()) {
            if (field.isAnnotationPresent(Inject.class)) {
                field.setAccessible(true);
                Object dependency = registry.get(field.getType()); // หา instance ที่ตรงกับชนิดข้อมูล
                field.set(target, dependency); // "ฉีด" (inject) dependency เข้าไปใน field
            }
        }
    }
}

interface EmailService { void send(String message); }

class SmtpEmailService implements EmailService {
    public void send(String message) { System.out.println("ส่งอีเมล: " + message); }
}

class NotificationController {
    @Inject
    EmailService emailService; // ไม่ต้องสร้างเอง - container จะฉีดให้อัตโนมัติ
}
```

```java
public class SimpleContainerDemo {
    public static void main(String[] args) throws Exception {
        SimpleContainer container = new SimpleContainer();
        container.register(EmailService.class, new SmtpEmailService());

        NotificationController controller = new NotificationController();
        container.injectDependencies(controller); // emailService ยังเป็น null ก่อนบรรทัดนี้

        controller.emailService.send("สวัสดี!"); // ทำงานได้เพราะถูก inject แล้ว
    }
}
```

**นี่คือหลักการพื้นฐานของ Dependency Injection** ที่ Spring Framework
(Part 73-74) ใช้เป็นแกนหลักทั้งระบบ — เพียงแต่ Spring มีความซับซ้อนกว่านี้มาก
(รองรับ constructor injection, scope, lifecycle callbacks ฯลฯ)

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียนเมธอด `printAllFieldsAndValues(Object obj)` ที่ใช้ reflection
พิมพ์ชื่อและค่าของทุก field ใน object (รวม private)

**เฉลย:**

```java
import java.lang.reflect.Field;

public class Exercise1 {
    static void printAllFieldsAndValues(Object obj) throws Exception {
        for (Field field : obj.getClass().getDeclaredFields()) {
            field.setAccessible(true);
            System.out.println(field.getName() + " = " + field.get(obj));
        }
    }

    public static void main(String[] args) throws Exception {
        printAllFieldsAndValues(new Product("Mouse", 500));
    }
}
```

**2)** เขียนเมธอด generic `createInstance(String className)` ที่สร้าง object
จากชื่อ class เป็น String (ใช้ `Class.forName()`)

**เฉลย:**

```java
public class Exercise2 {
    static Object createInstance(String className) throws Exception {
        Class<?> clazz = Class.forName(className);
        return clazz.getDeclaredConstructor().newInstance();
    }

    public static void main(String[] args) throws Exception {
        Object obj = createInstance("java.util.ArrayList");
        System.out.println(obj.getClass().getName()); // java.util.ArrayList
    }
}
```

**3)** อธิบายว่าทำไม Reflection ช้ากว่าการเรียกโค้ดตรง ๆ และควรใช้เมื่อไร

**เฉลย**: การเรียกผ่าน reflection ต้องผ่านขั้นตอนเพิ่มเติมที่การเรียกโค้ดตรง ๆ
ไม่ต้องทำ: ตรวจสอบสิทธิ์การเข้าถึง (access check), ค้นหา method/field ที่
ตรงกันจาก metadata, และ boxing/unboxing พารามิเตอร์ (เพราะ `invoke()` รับ
`Object...`) — นอกจากนี้ JIT compiler (ทบทวนจาก Part 1) ไม่สามารถ optimize
การเรียกผ่าน reflection ได้ดีเท่าการเรียกแบบ static ที่รู้ชนิดข้อมูลแน่นอน
ตั้งแต่ compile-time ควรใช้ reflection เฉพาะในโค้ดระดับ framework/tooling
ที่ต้องทำงานกับ class ที่ไม่รู้จักล่วงหน้า (เช่น dependency injection, ORM,
serialization library) ไม่ใช่ในโค้ด business logic ทั่วไปที่รู้ชนิดข้อมูล
อยู่แล้ว

### สรุปเนื้อหา Part 53

- Reflection ตรวจสอบและแก้ไข structure ของโปรแกรมตอน runtime โดยไม่ต้องรู้
  ชนิดข้อมูลล่วงหน้า
- `Class<T>` เป็นจุดเริ่มต้นของ reflection ทุกอย่าง (ได้จาก `.class`,
  `getClass()`, หรือ `Class.forName()`)
- `Field`, `Method`, `Constructor` ใช้ตรวจสอบและเรียกใช้งาน member ต่าง ๆ
  แบบ dynamic
- `setAccessible(true)` เจาะทะลุ encapsulation เข้าถึง private ได้ — ใช้อย่าง
  ระมัดระวัง
- Reflection มี performance overhead สูง เหมาะกับ framework-level code
  ไม่เหมาะกับ business logic ทั่วไป
- Reflection + Custom Annotation คือหลักการเบื้องหลัง Spring DI, JUnit,
  Jackson, Hibernate

**ต่อไป**: [Part 54 — Design Patterns: Creational (Singleton, Factory, Builder)](./part-054-design-patterns-creational.md)
