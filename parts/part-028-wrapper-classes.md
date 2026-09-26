# Part 28: Wrapper Classes, Autoboxing/Unboxing, Number Formatting

> ขั้นตอนที่ 271-280 ของหลักสูตร | ระดับ: กลาง

## สารบัญ

1. Wrapper Classes คืออะไร ทำไมต้องมี
2. ตารางเทียบ Primitive กับ Wrapper Class
3. Autoboxing และ Unboxing
4. กับดัก: `NullPointerException` จาก Unboxing
5. กับดัก: Integer Caching และการเปรียบเทียบด้วย `==`
6. เมธอด Utility สำคัญของ Wrapper Classes
7. การแปลงชนิดข้อมูลผ่าน Wrapper Classes
8. Performance: Autoboxing ใน Loop (กับดักด้านประสิทธิภาพ)
9. `Number` Abstract Class
10. แบบฝึกหัดและสรุป

---

## 1. Wrapper Classes คืออะไร ทำไมต้องมี

**Wrapper Class** คือ class ที่ **"ห่อ" primitive type ให้เป็น object** —
จำเป็นเพราะ Java Collections Framework (Part 22-25) และ Generics (Part 26)
**ทำงานกับ object เท่านั้น ไม่รับ primitive type โดยตรง**

```java
import java.util.ArrayList;
import java.util.List;

public class WhyWrapperDemo {
    public static void main(String[] args) {
        // List<int> numbers = new ArrayList<>(); // Error! Generics รับได้แค่ Reference Type
        List<Integer> numbers = new ArrayList<>(); // ต้องใช้ Wrapper Class (Integer) แทน
        numbers.add(10);
        numbers.add(20);
        System.out.println(numbers);
    }
}
```

## 2. ตารางเทียบ Primitive กับ Wrapper Class

| Primitive | Wrapper Class | ตัวอย่าง |
|---|---|---|
| `byte` | `Byte` | `Byte b = 10;` |
| `short` | `Short` | `Short s = 100;` |
| `int` | `Integer` | `Integer i = 42;` |
| `long` | `Long` | `Long l = 100L;` |
| `float` | `Float` | `Float f = 3.14f;` |
| `double` | `Double` | `Double d = 3.14;` |
| `char` | `Character` | `Character c = 'A';` |
| `boolean` | `Boolean` | `Boolean b = true;` |

ทุก Wrapper Class อยู่ใน package `java.lang` (import อัตโนมัติ) และเป็น
**immutable** (ทบทวนแนวคิดจาก Part 17) — เมื่อสร้างแล้วเปลี่ยนค่าไม่ได้ ต้องสร้าง
object ใหม่เสมอ

## 3. Autoboxing และ Unboxing

**Autoboxing** คือการแปลง primitive → Wrapper **โดยอัตโนมัติ**, **Unboxing**
คือการแปลง Wrapper → primitive **โดยอัตโนมัติ** — Java ทำให้ตั้งแต่ Java 5
เพื่อลดความยืดยาวของโค้ด

```java
public class AutoboxingDemo {
    public static void main(String[] args) {
        // Autoboxing: int -> Integer โดยอัตโนมัติ
        Integer boxed = 42; // เทียบเท่ากับ: Integer boxed = Integer.valueOf(42);

        // Unboxing: Integer -> int โดยอัตโนมัติ
        int unboxed = boxed; // เทียบเท่ากับ: int unboxed = boxed.intValue();

        System.out.println(boxed + " " + unboxed);

        // Autoboxing/Unboxing เกิดขึ้นอัตโนมัติในการคำนวณและเปรียบเทียบด้วย
        Integer a = 10;
        Integer b = 20;
        Integer sum = a + b; // unbox ทั้ง a, b -> บวกแบบ int -> box กลับเป็น Integer
        System.out.println(sum); // 30

        // Autoboxing ใน Collections ก็เกิดขึ้นอัตโนมัติเสมอ
        java.util.List<Integer> list = new java.util.ArrayList<>();
        list.add(5); // int 5 -> autobox เป็น Integer โดยอัตโนมัติ
        int first = list.get(0); // Integer -> unbox เป็น int โดยอัตโนมัติ
    }
}
```

## 4. กับดัก: `NullPointerException` จาก Unboxing

**นี่คือกับดักที่พบบ่อยที่สุดข้อหนึ่งในโค้ด Java จริง**: ถ้า Wrapper object
เป็น `null` แล้วพยายาม unbox จะเกิด `NullPointerException` ทันที

```java
public class UnboxingNullPitfall {
    static Integer getScore(boolean hasScore) {
        if (hasScore) {
            return 85;
        }
        return null; // คืนค่า null เมื่อไม่มีคะแนน
    }

    public static void main(String[] args) {
        Integer score = getScore(false); // ได้ null

        try {
            int result = score + 10; // Error! พยายาม unbox null -> NullPointerException
        } catch (NullPointerException e) {
            System.out.println("เกิด NullPointerException จากการ unbox ค่า null");
        }

        // วิธีป้องกัน: เช็ค null ก่อนเสมอ หรือใช้ Optional (Part 43)
        if (score != null) {
            int result = score + 10;
            System.out.println(result);
        } else {
            System.out.println("ไม่มีคะแนน");
        }
    }
}
```

**สถานการณ์ที่เกิดปัญหานี้บ่อยมาก**: ใช้ `Integer` เป็น field ในคลาสที่ map มา
จากฐานข้อมูล (ค่าที่เป็น `NULL` ในฐานข้อมูลจะกลายเป็น `null` ใน Java) แล้วนำไป
คำนวณโดยไม่เช็ค null ก่อน

```java
public class ConditionalUnboxingPitfall {
    public static void main(String[] args) {
        Integer count = null;
        boolean hasItems = false;

        // แม้ hasItems เป็น false, Java ยังต้อง unbox ทั้งสองด้านของ && ก่อนตัดสินใจ... 
        // จริง ๆ ไม่ใช่แบบนั้น (&& ยัง short-circuit ปกติ) แต่ตัวอย่างที่พังบ่อยคือ ternary:
        int result = hasItems ? count : 0; // ดูเหมือนปลอดภัย เพราะ hasItems=false เลือกฝั่ง 0

        // แต่ตัวอย่างนี้จะพัง เพราะ ternary ที่มีทั้ง int และ Integer จะ unbox "ทั้งสองฝั่ง"
        // เพื่อหาชนิดข้อมูลผลลัพธ์ร่วมกัน (ต้อง unbox count แม้ hasItems=false ก็ตาม!)
        try {
            int x = 5;
            int result2 = hasItems ? x : count; // NullPointerException! ต้อง unbox count เสมอ
                                                   // เพราะ ternary ต้องมีชนิดข้อมูลผลลัพธ์เดียวกัน (int)
        } catch (NullPointerException e) {
            System.out.println("Ternary กับดัก: unbox null โดยไม่รู้ตัว");
        }
    }
}
```

## 5. กับดัก: Integer Caching และการเปรียบเทียบด้วย `==`

Java **cache ค่า `Integer` ระหว่าง -128 ถึง 127** ไว้ล่วงหน้า (คล้าย String Pool
จาก Part 9) เพื่อประหยัดหน่วยความจำ — ทำให้การเปรียบเทียบด้วย `==` ให้ผลลัพธ์
**ไม่สอดคล้องกัน**ขึ้นกับค่าตัวเลข ซึ่งเป็นกับดักร้ายแรงมาก:

```java
public class IntegerCachingPitfall {
    public static void main(String[] args) {
        Integer a = 100;
        Integer b = 100;
        System.out.println(a == b); // true (!) เพราะ 100 อยู่ในช่วง cache (-128 ถึง 127)

        Integer c = 200;
        Integer d = 200;
        System.out.println(c == d); // false (!!) เพราะ 200 อยู่นอกช่วง cache -> สร้าง object ใหม่

        // ผลลัพธ์ที่ไม่สอดคล้องกันนี้คือกับดักร้ายแรง เพราะโค้ดทั้งสองดูเหมือนกันเป๊ะ
        // แต่ให้ผลต่างกันขึ้นกับค่าตัวเลขที่ใช้ทดสอบ!
    }
}
```

**กฎทอง**: **ใช้ `.equals()` เปรียบเทียบ Wrapper class เสมอ ไม่ใช้ `==`**
(เหมือนกับ String ใน Part 9) ยกเว้นตั้งใจเปรียบเทียบ primitive ที่ unbox แล้ว:

```java
public class CorrectComparisonDemo {
    public static void main(String[] args) {
        Integer a = 200;
        Integer b = 200;
        System.out.println(a.equals(b)); // true (ถูกต้องเสมอ ไม่ว่าค่าจะเป็นเท่าไร)

        // หรือ unbox เป็น primitive ก่อนเปรียบเทียบด้วย == (ปลอดภัยเพราะเทียบค่าจริง ไม่ใช่ reference)
        System.out.println(a.intValue() == b.intValue()); // true
    }
}
```

## 6. เมธอด Utility สำคัญของ Wrapper Classes

```java
public class WrapperUtilityDemo {
    public static void main(String[] args) {
        // การแปลงระหว่าง String และตัวเลข
        int parsed = Integer.parseInt("123");        // String -> int
        String str = Integer.toString(456);            // int -> String
        Integer boxed = Integer.valueOf("789");         // String -> Integer

        // ค่าคงที่สำคัญ
        System.out.println(Integer.MAX_VALUE);          // 2147483647
        System.out.println(Integer.MIN_VALUE);           // -2147483648
        System.out.println(Double.MAX_VALUE);
        System.out.println(Character.MAX_VALUE);

        // เมธอดเปรียบเทียบ (static) - ปลอดภัยกว่าการลบตรง ๆ (ทบทวนจาก Part 27)
        System.out.println(Integer.compare(5, 10));      // ค่าติดลบ
        System.out.println(Integer.max(5, 10));           // 10
        System.out.println(Integer.min(5, 10));            // 5
        System.out.println(Integer.sum(5, 10));             // 15

        // การแปลงเลขฐาน
        System.out.println(Integer.toBinaryString(10));    // "1010"
        System.out.println(Integer.toHexString(255));       // "ff"
        System.out.println(Integer.toOctalString(8));        // "10"
        System.out.println(Integer.parseInt("1010", 2));      // 10 (แปลงจากฐาน 2)

        // Character utility methods
        System.out.println(Character.isDigit('5'));         // true
        System.out.println(Character.isLetter('A'));         // true
        System.out.println(Character.isUpperCase('A'));       // true
        System.out.println(Character.toLowerCase('A'));        // 'a'
        System.out.println(Character.isWhitespace(' '));        // true
    }
}
```

## 7. การแปลงชนิดข้อมูลผ่าน Wrapper Classes

```java
public class ConversionDemo {
    public static void main(String[] args) {
        // ทบทวนจาก Part 3: การแปลง String <-> ตัวเลขที่ปลอดภัย
        String input = "42.5";

        try {
            double value = Double.parseDouble(input);
            int intValue = (int) value; // ต้อง cast เอง เพราะ narrowing conversion
            System.out.println(intValue); // 42
        } catch (NumberFormatException e) {
            System.out.println("แปลงไม่สำเร็จ: ข้อมูลไม่ใช่ตัวเลข");
        }

        // Wrapper -> Wrapper ชนิดอื่น
        Integer intObj = 100;
        double doubleValue = intObj.doubleValue(); // ใช้เมธอดของ Number (หัวข้อ 9)
        long longValue = intObj.longValue();
        System.out.println(doubleValue + " " + longValue);
    }
}
```

## 8. Performance: Autoboxing ใน Loop (กับดักด้านประสิทธิภาพ)

Autoboxing ที่เกิดขึ้นซ้ำ ๆ ใน loop จำนวนมาก**ทำให้เกิด object จำนวนมหาศาลโดยไม่
จำเป็น** เพิ่มภาระให้ garbage collector และทำให้โปรแกรมช้าลงอย่างมีนัยสำคัญ:

```java
public class AutoboxingPerformanceDemo {
    public static void main(String[] args) {
        long start = System.nanoTime();

        Long sum = 0L; // ใช้ Wrapper class แทน primitive
        for (long i = 0; i < 10_000_000; i++) {
            sum += i; // ทุกรอบ: unbox sum -> บวก -> box กลับเป็น Long object ใหม่! (สร้าง object 10 ล้านตัว!)
        }

        long boxedTime = System.nanoTime() - start;
        System.out.println("ใช้เวลา (Wrapper): " + boxedTime / 1_000_000 + " ms");

        start = System.nanoTime();
        long sumPrimitive = 0L; // ใช้ primitive ตรง ๆ
        for (long i = 0; i < 10_000_000; i++) {
            sumPrimitive += i; // ไม่มีการสร้าง object เลย เร็วกว่ามาก
        }
        long primitiveTime = System.nanoTime() - start;
        System.out.println("ใช้เวลา (primitive): " + primitiveTime / 1_000_000 + " ms");
    }
}
```

**หลักการปฏิบัติ**: ใช้ **primitive type ในการคำนวณที่ต้อง loop จำนวนมาก** เสมอ
สงวน Wrapper class ไว้เฉพาะเมื่อจำเป็นต้องใช้กับ Generics/Collections เท่านั้น

## 9. `Number` Abstract Class

Wrapper class ทั้งหมด (`Integer`, `Double`, `Long`, ฯลฯ) สืบทอดจาก **abstract
class `Number`** (ทบทวนแนวคิด abstract class จาก Part 16) ทำให้แปลงข้าม
ชนิดข้อมูลตัวเลขได้ง่ายผ่าน interface ร่วมกัน:

```java
public abstract class Number {
    public abstract int intValue();
    public abstract long longValue();
    public abstract float floatValue();
    public abstract double doubleValue();
}
```

```java
public class NumberClassDemo {
    static void printAsAllTypes(Number number) { // รับ Number ใดก็ได้ผ่าน polymorphism
        System.out.println("int: " + number.intValue());
        System.out.println("long: " + number.longValue());
        System.out.println("double: " + number.doubleValue());
    }

    public static void main(String[] args) {
        printAsAllTypes(42);      // Integer (autobox เป็น Integer ก่อน)
        printAsAllTypes(3.14);     // Double
        printAsAllTypes(100L);      // Long
    }
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** อธิบายว่าทำไมโค้ดนี้พิมพ์ผลลัพธ์ที่ดูขัดแย้งกัน:

```java
Integer a = 127, b = 127;
Integer c = 128, d = 128;
System.out.println(a == b); // ?
System.out.println(c == d); // ?
```

**เฉลย**: `a == b` ได้ `true` เพราะ 127 อยู่ในช่วง Integer cache (-128 ถึง 127)
ทั้งสองตัวแปรจึงชี้ไปยัง object เดียวกันจาก cache แต่ `c == d` ได้ `false`
เพราะ 128 อยู่นอกช่วง cache ทำให้ Java สร้าง object ใหม่แยกกันสองตัว — บทเรียน
คือควรใช้ `.equals()` เปรียบเทียบ Wrapper class เสมอ ไม่ใช้ `==`

**2)** เขียนเมธอด `safeAdd(Integer a, Integer b)` ที่คืนค่า `null` ถ้าตัวใด
ตัวหนึ่งเป็น `null` แทนที่จะเกิด `NullPointerException`

**เฉลย:**

```java
public class Exercise2 {
    static Integer safeAdd(Integer a, Integer b) {
        if (a == null || b == null) {
            return null;
        }
        return a + b;
    }

    public static void main(String[] args) {
        System.out.println(safeAdd(5, 10));   // 15
        System.out.println(safeAdd(5, null)); // null
    }
}
```

**3)** เขียนโปรแกรมที่วัดความแตกต่างของประสิทธิภาพระหว่างการใช้ `int[]` และ
`List<Integer>` ในการหาผลรวมของตัวเลข 1 ล้านตัว

**เฉลย:**

```java
import java.util.ArrayList;
import java.util.List;

public class Exercise3 {
    public static void main(String[] args) {
        int n = 1_000_000;

        int[] array = new int[n];
        for (int i = 0; i < n; i++) array[i] = i;

        List<Integer> list = new ArrayList<>();
        for (int i = 0; i < n; i++) list.add(i); // autoboxing ทุกครั้งที่ add

        long start = System.nanoTime();
        long sum1 = 0;
        for (int value : array) sum1 += value;
        System.out.println("Array: " + (System.nanoTime() - start) / 1_000_000 + " ms");

        start = System.nanoTime();
        long sum2 = 0;
        for (int value : list) sum2 += value; // unboxing ทุกครั้งที่วน loop
        System.out.println("List: " + (System.nanoTime() - start) / 1_000_000 + " ms");
    }
}
```

### สรุปเนื้อหา Part 28

- Wrapper class ห่อ primitive ให้เป็น object เพื่อใช้กับ Generics/Collections
  ได้ (primitive ใช้ตรง ๆ กับ Generics ไม่ได้)
- Autoboxing/Unboxing เกิดขึ้นอัตโนมัติ แต่ unbox ค่า `null` จะเกิด
  `NullPointerException` ทันที
- Integer cache (-128 ถึง 127) ทำให้ `==` ให้ผลลัพธ์ไม่สอดคล้องกัน — ใช้
  `.equals()` เสมอ
- Autoboxing ใน loop จำนวนมากส่งผลเสียต่อประสิทธิภาพอย่างมาก ใช้ primitive แทน
  เมื่อทำได้
- ทุก Wrapper class สืบทอดจาก abstract class `Number` มีเมธอด `intValue()`,
  `doubleValue()` ฯลฯ ร่วมกัน

**ต่อไป**: [Part 29 — Recursion ขั้นสูง: Backtracking, Memoization](./part-029-recursion-advanced.md)
