# Part 39: Lambda Expressions พื้นฐานถึงขั้นสูง

> ขั้นตอนที่ 381-390 ของหลักสูตร | ระดับ: กลาง-สูง (จุดเปลี่ยนสำคัญของหลักสูตร)

## สารบัญ

1. Lambda Expression คืออะไร
2. Syntax ของ Lambda Expression ทุกรูปแบบ
3. Lambda กับ Functional Interface (ทบทวนเชิงลึกจาก Part 16)
4. Variable Capture ใน Lambda
5. Method Reference: กระชับกว่า Lambda อีกขั้น
6. Method Reference 4 รูปแบบ
7. Lambda ที่ throw Exception (ข้อจำกัดสำคัญ)
8. เมื่อไรใช้ Lambda เมื่อไรใช้ Anonymous Class (ทบทวนจาก Part 19)
9. ประสิทธิภาพของ Lambda เทียบกับ Anonymous Class
10. แบบฝึกหัดและสรุป

---

## 1. Lambda Expression คืออะไร

**Lambda Expression** (Java 8+) คือวิธีเขียน**function แบบไม่มีชื่อ (anonymous
function)** ที่กระชับกว่า anonymous class มาก — ใช้แทน implementation ของ
**functional interface** (interface ที่มี abstract method เดียว — ทบทวนจาก
Part 16) นี่คือจุดเปลี่ยนสำคัญที่สุดจุดหนึ่งในประวัติศาสตร์ภาษา Java เพราะเปิด
ทางให้เขียนโปรแกรมแบบ **functional programming** ควบคู่กับ OOP ได้

```java
import java.util.Comparator;
import java.util.ArrayList;
import java.util.List;

public class LambdaMotivationDemo {
    public static void main(String[] args) {
        List<String> names = new ArrayList<>(List.of("Charlie", "Alice", "Bob"));

        // แบบเก่า: Anonymous Class (ทบทวนจาก Part 19) - ยืดยาว
        names.sort(new Comparator<String>() {
            @Override
            public int compare(String a, String b) {
                return a.compareTo(b);
            }
        });

        // แบบใหม่: Lambda Expression - กระชับกว่ามาก ทำสิ่งเดียวกันทุกประการ
        names.sort((a, b) -> a.compareTo(b));

        System.out.println(names);
    }
}
```

## 2. Syntax ของ Lambda Expression ทุกรูปแบบ

รูปแบบพื้นฐาน: `(พารามิเตอร์) -> { เนื้อหา }`

```java
import java.util.function.*;

public class LambdaSyntaxDemo {
    public static void main(String[] args) {
        // ไม่มีพารามิเตอร์
        Runnable noParams = () -> System.out.println("ไม่มีพารามิเตอร์เลย");

        // พารามิเตอร์ 1 ตัว - วงเล็บใส่หรือไม่ใส่ก็ได้
        Function<Integer, Integer> square = x -> x * x;
        Function<Integer, Integer> squareWithParens = (x) -> x * x; // เหมือนกันทุกประการ

        // พารามิเตอร์หลายตัว - ต้องมีวงเล็บเสมอ
        BiFunction<Integer, Integer, Integer> add = (a, b) -> a + b;

        // ระบุชนิดข้อมูลของพารามิเตอร์อย่างชัดเจน (ปกติไม่จำเป็น เพราะ compiler อนุมานได้)
        BiFunction<Integer, Integer, Integer> addExplicit = (Integer a, Integer b) -> a + b;

        // body เป็น expression เดียว (implicit return - ไม่ต้องเขียน return หรือ { })
        Function<Integer, Integer> doubleIt = x -> x * 2;

        // body เป็น block หลายบรรทัด - ต้องมี { } และ return ชัดเจน
        Function<Integer, Integer> complexCalculation = x -> {
            int result = x * 2;
            result += 10;
            return result; // ต้องมี return เมื่อใช้ { } และมีการคืนค่า
        };

        System.out.println(square.apply(5));           // 25
        System.out.println(add.apply(3, 4));             // 7
        System.out.println(complexCalculation.apply(5)); // 20
        noParams.run();
    }
}
```

## 3. Lambda กับ Functional Interface (ทบทวนเชิงลึกจาก Part 16)

Lambda **ทำงานได้เฉพาะกับ functional interface** (abstract method เดียว)
เพราะ compiler ต้องรู้ว่า parameter/return type ของ lambda ควรเป็นแบบไหน
จาก signature ของ abstract method นั้น

```java
@FunctionalInterface
interface Validator<T> {
    boolean isValid(T value);
}

public class CustomFunctionalInterfaceDemo {
    static <T> void validateAndPrint(T value, Validator<T> validator) {
        System.out.println(value + " ถูกต้อง: " + validator.isValid(value));
    }

    public static void main(String[] args) {
        Validator<String> notEmpty = s -> s != null && !s.isEmpty();
        Validator<Integer> isPositive = n -> n > 0;

        validateAndPrint("Hello", notEmpty);  // true
        validateAndPrint("", notEmpty);        // false
        validateAndPrint(-5, isPositive);       // false

        validateAndPrint("test", str -> str.length() > 2); // ใช้ lambda ตรง ๆ ก็ได้เช่นกัน
    }
}
```

**`@FunctionalInterface`** ไม่บังคับทางเทคนิค แต่**แนะนำให้ใส่เสมอ** เพราะ
compiler จะช่วยตรวจสอบว่า interface มี abstract method เดียวจริง (ถ้าเผลอเพิ่ม
ตัวที่สองเข้าไปจะ error ทันที — ป้องกัน bug จากการแก้ไข interface ผิดพลาด)

## 4. Variable Capture ใน Lambda

Lambda สามารถเข้าถึงตัวแปรจาก **enclosing scope** ได้ (เหมือน anonymous class
— ทบทวนแนวคิด effectively final จาก Part 19)

```java
import java.util.function.Supplier;

public class VariableCaptureDemo {
    static int instanceCounter = 0;

    static Supplier<String> createGreeter(String name) {
        int greetingCount = 0; // effectively final (ไม่มีการเปลี่ยนแปลงหลังจากนี้)

        return () -> "สวัสดี " + name + "! (นับครั้งที่สร้าง: " + greetingCount + ")";
        // lambda "capture" ค่าของ name และ greetingCount ไว้ ณ ตอนที่สร้าง
    }

    public static void main(String[] args) {
        Supplier<String> greeter = createGreeter("Alice");
        System.out.println(greeter.get()); // ยังใช้ name="Alice" ได้ แม้ createGreeter() คืนค่าไปแล้ว

        instanceCounter++; // แก้ไข static field ได้ปกติ (ไม่ต้องเป็น effectively final)
        Runnable r = () -> System.out.println("counter=" + instanceCounter); // อ่านค่าล่าสุดตอนเรียก run()
        instanceCounter++;
        r.run(); // "counter=2" - อ่านค่า ณ ตอนเรียก ไม่ใช่ตอนสร้าง lambda (เพราะเป็น static field ไม่ใช่ local variable)
    }
}
```

**ข้อแตกต่างสำคัญ**: local variable ที่ lambda capture ต้อง**effectively
final** (ทบทวนจาก Part 19) แต่**instance field/static field ไม่มีข้อจำกัดนี้**
เพราะไม่ได้ "capture ค่า" แต่ lambda จะไปอ่านค่าจาก field ผ่าน reference ของ
object/class ณ ตอนที่ถูกเรียกใช้จริง

## 5. Method Reference: กระชับกว่า Lambda อีกขั้น

**Method Reference** (`::`) ใช้แทน lambda ที่**เพียงเรียก method ที่มีอยู่แล้ว
ตรง ๆ โดยไม่ทำอะไรเพิ่ม** — กระชับกว่าและอ่านง่ายกว่า

```java
import java.util.List;
import java.util.function.Function;
import java.util.function.Consumer;

public class MethodReferenceDemo {
    public static void main(String[] args) {
        List<String> names = List.of("Alice", "Bob", "Charlie");

        // Lambda ที่แค่เรียก method ตรง ๆ
        names.forEach(name -> System.out.println(name));

        // Method Reference ที่เทียบเท่ากัน - กระชับกว่า
        names.forEach(System.out::println);

        // อีกตัวอย่าง: แปลง String เป็น Integer
        Function<String, Integer> parseLambda = s -> Integer.parseInt(s);
        Function<String, Integer> parseRef = Integer::parseInt; // เทียบเท่ากันทุกประการ

        System.out.println(parseRef.apply("123")); // 123
    }
}
```

## 6. Method Reference 4 รูปแบบ

```java
import java.util.function.*;

public class MethodReferenceTypesDemo {
    static class Calculator {
        int value;
        Calculator(int value) { this.value = value; }
        int addTo(int other) { return value + other; }
        static int staticAdd(int a, int b) { return a + b; }
    }

    public static void main(String[] args) {
        // รูปแบบ 1: ClassName::staticMethodName (เรียก static method)
        BiFunction<Integer, Integer, Integer> staticRef = Calculator::staticAdd;
        System.out.println(staticRef.apply(3, 4)); // 7

        // รูปแบบ 2: instance::instanceMethodName (เรียก method ของ object ที่มีอยู่แล้ว)
        Calculator calc = new Calculator(10);
        Function<Integer, Integer> instanceRef = calc::addTo;
        System.out.println(instanceRef.apply(5)); // 15 (10 + 5)

        // รูปแบบ 3: ClassName::instanceMethodName (เรียก instance method โดยรับ object เป็นพารามิเตอร์แรก)
        Function<String, Integer> lengthRef = String::length;
        System.out.println(lengthRef.apply("Hello")); // 5 (เทียบเท่า s -> s.length())

        // รูปแบบ 4: ClassName::new (Constructor Reference)
        Supplier<ArrayList<String>> listSupplier = ArrayList::new;
        java.util.List<String> newList = listSupplier.get();
        newList.add("test");
        System.out.println(newList); // [test]
    }

    static class ArrayList<T> extends java.util.ArrayList<T> { } // สมมติ import ปกติ
}
```

ตัวอย่างการใช้งานจริงที่พบบ่อย (Constructor Reference กับ Stream — ปูทางสู่
Part 41):

```java
import java.util.List;
import java.util.stream.Collectors;

public class ConstructorReferenceRealDemo {
    record Point(int x, int y) { }

    public static void main(String[] args) {
        List<Integer> xValues = List.of(1, 2, 3);
        List<Point> points = xValues.stream()
                                     .map(x -> new Point(x, x * 2)) // lambda
                                     .collect(Collectors.toList());
        System.out.println(points); // [Point[x=1, y=2], Point[x=2, y=4], Point[x=3, y=6]]
    }
}
```

## 7. Lambda ที่ throw Exception (ข้อจำกัดสำคัญ)

ทบทวนจาก Part 21: functional interface มาตรฐาน (`Function`, `Consumer`, ฯลฯ)
**ไม่รองรับ checked exception** ในเมธอด abstract ของมัน ทำให้ lambda ที่
throw checked exception จะ compile error

```java
import java.util.List;
import java.util.function.Function;

public class LambdaExceptionDemo {
    static int parseOrThrow(String s) throws NumberFormatException { // unchecked - ไม่มีปัญหา
        return Integer.parseInt(s);
    }

    static int riskyParse(String s) throws Exception { // checked exception
        if (!s.matches("\\d+")) throw new Exception("ไม่ใช่ตัวเลข");
        return Integer.parseInt(s);
    }

    public static void main(String[] args) {
        Function<String, Integer> ok = LambdaExceptionDemo::parseOrThrow; // OK: unchecked exception ไม่มีปัญหา

        // Function<String, Integer> bad = LambdaExceptionDemo::riskyParse; // Error! checked exception ไม่รองรับ

        // วิธีแก้: ห่อด้วย try-catch ภายใน lambda เอง แปลงเป็น unchecked
        Function<String, Integer> wrapped = s -> {
            try {
                return riskyParse(s);
            } catch (Exception e) {
                throw new RuntimeException(e); // แปลงเป็น unchecked (ทบทวน exception translation จาก Part 21)
            }
        };

        System.out.println(wrapped.apply("123"));
    }
}
```

## 8. เมื่อไรใช้ Lambda เมื่อไรใช้ Anonymous Class

| ใช้ **Lambda** เมื่อ | ใช้ **Anonymous Class** เมื่อ |
|---|---|
| Implement functional interface (abstract method เดียว) | Implement interface ที่มี abstract method หลายตัว |
| Implement interface ธรรมดา (ไม่ใช่ abstract class) | Extends abstract class |
| ต้องการโค้ดกระชับ อ่านง่าย | ต้องการ field/state เพิ่มเติมใน implementation |
| ไม่ต้องอ้างอิงถึง `this` ของตัวเอง (lambda's `this` คือ enclosing class) | ต้องการ `this` ที่อ้างถึง object ของ anonymous class เอง |

## 9. ประสิทธิภาพของ Lambda เทียบกับ Anonymous Class

Lambda **ไม่ได้สร้าง `.class` file แยกทุกครั้ง** แบบ anonymous class (ที่
compiler จะสร้างไฟล์ `Outer$1.class` แยกให้ทุกตัว) — Lambda ใช้กลไก
**`invokedynamic`** (bytecode instruction พิเศษที่เพิ่มมาตั้งแต่ Java 7)
สร้าง class ขึ้นมา**ตอน runtime แบบ lazy** (ครั้งแรกที่ถูกเรียกใช้เท่านั้น)
ทำให้**เริ่มโปรแกรมได้เร็วกว่า**และใช้หน่วยความจำ classloading น้อยกว่าเมื่อมี
lambda จำนวนมากในโปรแกรม

```java
public class LambdaVsAnonymousPerformance {
    public static void main(String[] args) {
        // ทั้งสองทำงานเหมือนกัน แต่ lambda ไม่สร้าง .class file แยกตอน compile
        Runnable lambdaVersion = () -> System.out.println("lambda");
        Runnable anonymousVersion = new Runnable() {
            @Override
            public void run() { System.out.println("anonymous"); }
        };
        // ผลลัพธ์การทำงานเหมือนกัน แต่ lambda มี classloading overhead น้อยกว่า
        // และไม่สร้าง object ใหม่ทุกครั้งที่เรียก (ถ้า lambda ไม่ capture ตัวแปรใด ๆ - JVM cache ไว้ใช้ซ้ำได้)
    }
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียน lambda expression สำหรับ `Comparator<Integer>` ที่เรียงจากมากไปน้อย

**เฉลย:**

```java
import java.util.Comparator;

public class Exercise1 {
    public static void main(String[] args) {
        Comparator<Integer> descending = (a, b) -> b - a;
        java.util.List<Integer> numbers = new java.util.ArrayList<>(java.util.List.of(3, 1, 4, 1, 5));
        numbers.sort(descending);
        System.out.println(numbers); // [5, 4, 3, 1, 1]
    }
}
```

**2)** เขียน custom functional interface `Transformer<T, R>` ที่มี method
`transform(T input)` คืนค่า `R` แล้วใช้ lambda และ method reference ทดสอบ

**เฉลย:**

```java
@FunctionalInterface
interface Transformer<T, R> {
    R transform(T input);
}

public class Exercise2 {
    public static void main(String[] args) {
        Transformer<String, Integer> toLength = String::length; // method reference
        Transformer<Integer, String> toBinary = Integer::toBinaryString; // method reference

        System.out.println(toLength.transform("Hello")); // 5
        System.out.println(toBinary.transform(10));         // "1010"
    }
}
```

**3)** อธิบายว่าทำไม lambda ที่ throw checked exception ไม่สามารถใช้กับ
`java.util.function.Function` ได้ตรง ๆ และมีวิธีแก้อย่างไร

**เฉลย**: เพราะ abstract method `apply()` ของ `Function<T, R>` ไม่ได้ประกาศ
`throws` ไว้ใน signature การเขียน lambda ที่ throw checked exception จะขัดกับ
signature นี้ทันที (checked exception ต้องถูกประกาศใน `throws` ของ method ที่
มันอาจเกิดขึ้น — ทบทวนจาก Part 10, 21) วิธีแก้คือห่อ logic ที่ throw checked
exception ด้วย try-catch ภายใน lambda เอง แล้วแปลง (translate) เป็น unchecked
exception (เช่น `RuntimeException`) ก่อน throw ออกไป

### สรุปเนื้อหา Part 39

- Lambda Expression คือ function แบบไม่มีชื่อที่ใช้แทน implementation ของ
  functional interface กระชับกว่า anonymous class มาก
- Syntax: `(พารามิเตอร์) -> expression` หรือ `(พารามิเตอร์) -> { statements;
  return value; }`
- Lambda capture ตัวแปร local ต้องเป็น effectively final, แต่ field ไม่มี
  ข้อจำกัดนี้
- Method Reference (`::`) มี 4 รูปแบบ: static method, instance method ของ
  object ที่มีอยู่, instance method ผ่าน class, และ constructor reference
- Lambda ไม่รองรับ checked exception ในตัว functional interface มาตรฐาน
  ต้องแปลงเป็น unchecked เอง
- Lambda ใช้ `invokedynamic` ไม่สร้าง `.class` แยกแบบ anonymous class ทำให้
  classloading เร็วกว่า

**ต่อไป**: [Part 40 — Functional Interfaces: Function, Predicate, Consumer, Supplier](./part-040-functional-interfaces.md)
