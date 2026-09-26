# Part 9: String, StringBuilder, StringBuffer

> ขั้นตอนที่ 81-90 ของหลักสูตร | ระดับ: พื้นฐาน

## สารบัญ

1. String คืออะไร และความ Immutable
2. String Pool และการสร้าง String
3. เมธอดสำคัญของ String
4. String Formatting (`String.format`, `printf`, text block)
5. เปรียบเทียบ String อย่างถูกต้อง
6. ทำไม String ต่อกันเยอะ ๆ ใน loop ถึงช้า
7. `StringBuilder` และ `StringBuffer`
8. ความแตกต่างระหว่าง StringBuilder กับ StringBuffer
9. Text Blocks (Java 15+)
10. แบบฝึกหัดและสรุป

---

## 1. String คืออะไร และความ Immutable

`String` เป็นคลาสใน `java.lang` ที่ใช้เก็บลำดับของตัวอักษร (sequence of characters)
แม้จะดูเหมือน primitive type เพราะมี literal syntax (`"text"`) แต่จริง ๆ แล้ว
**`String` เป็น reference type (class)** ที่มีคุณสมบัติพิเศษที่สุดข้อหนึ่งคือ
**Immutable (ไม่สามารถเปลี่ยนแปลงค่าได้หลังสร้าง)**

```java
public class ImmutableDemo {
    public static void main(String[] args) {
        String s1 = "Hello";
        String s2 = s1.concat(" World"); // concat "สร้าง String ใหม่" ไม่ได้แก้ s1

        System.out.println(s1); // "Hello" (ไม่เปลี่ยนแปลง!)
        System.out.println(s2); // "Hello World" (String ใหม่)

        s1 = s1 + " Java"; // นี่คือการ "reassign" ตัวแปร s1 ให้ชี้ไป String ใหม่
                            // ไม่ใช่การแก้ไข String เดิม (String เดิม "Hello" ยังอยู่ใน
                            // memory จนกว่า garbage collector จะเก็บกวาดทิ้ง)
        System.out.println(s1); // "Hello Java"
    }
}
```

**ทำไม String ต้อง Immutable?**

1. **ความปลอดภัย**: String มักใช้เก็บข้อมูลสำคัญ (username, path, URL) ถ้าเปลี่ยนแปลง
   ได้ระหว่างทาง อาจเกิดช่องโหว่ความปลอดภัย
2. **String Pool ทำงานได้**: เพราะ immutable การแชร์ object เดียวกันจึงปลอดภัย
   (อธิบายในหัวข้อถัดไป)
3. **Thread-safe โดยธรรมชาติ**: หลาย thread อ่าน String พร้อมกันได้อย่างปลอดภัย
   เพราะไม่มีทางที่ค่าจะถูกแก้ไขระหว่างทาง
4. **ใช้เป็น key ของ HashMap ได้อย่างปลอดภัย**: hashCode ของ String คงที่ตลอดชีวิต
   ของ object

## 2. String Pool และการสร้าง String

Java มีพื้นที่หน่วยความจำพิเศษเรียกว่า **String Constant Pool** (อยู่ใน heap)
ที่ใช้เก็บ String literal เพื่อ**นำกลับมาใช้ซ้ำ** ลดการสร้าง object ซ้ำซ้อน

```java
public class StringPoolDemo {
    public static void main(String[] args) {
        String s1 = "Java";           // สร้างใน String Pool (หรือใช้ตัวที่มีอยู่แล้ว)
        String s2 = "Java";           // ตัวนี้ "ใช้ตัวเดียวกับ s1" จาก Pool ไม่สร้างใหม่
        String s3 = new String("Java"); // บังคับสร้าง object ใหม่บน heap (นอก Pool)
        String s4 = s3.intern();      // .intern() ดึงมาจาก Pool (หรือเพิ่มเข้า Pool ถ้ายังไม่มี)

        System.out.println(s1 == s2); // true  - อ้างอิงถึง object เดียวกันใน Pool
        System.out.println(s1 == s3); // false - s3 เป็น object คนละตัวบน heap
        System.out.println(s1 == s4); // true  - intern() ทำให้กลับมาชี้ที่ Pool เดียวกัน
    }
}
```

```
String Constant Pool (ส่วนพิเศษใน Heap)
┌───────────────┐
│    "Java"     │ <─────┬───── s1
└───────────────┘       ├───── s2
                         └───── s4 (หลัง intern())

Heap (ปกติ)
┌───────────────┐
│  new "Java"   │ <───────────  s3
└───────────────┘
```

**ข้อควรจำ**: การสร้าง String ด้วย `new String(...)` แทบไม่มีเหตุผลให้ใช้ในโค้ดจริง
ควรใช้ literal (`"text"`) เสมอ เพื่อประโยชน์จาก String Pool

## 3. เมธอดสำคัญของ String

```java
public class StringMethodsDemo {
    public static void main(String[] args) {
        String text = "  Hello, Java World!  ";

        System.out.println(text.length());              // 22 (นับรวม space)
        System.out.println(text.trim());                 // "Hello, Java World!" (ตัด space หัวท้าย)
        System.out.println(text.strip());                 // เหมือน trim แต่รองรับ Unicode whitespace ดีกว่า (Java 11+)
        System.out.println(text.toUpperCase());           // "  HELLO, JAVA WORLD!  "
        System.out.println(text.toLowerCase());           // "  hello, java world!  "

        String clean = text.trim();
        System.out.println(clean.charAt(0));               // 'H' (ตัวอักษรที่ index 0)
        System.out.println(clean.indexOf("Java"));          // 7 (ตำแหน่งที่เจอครั้งแรก)
        System.out.println(clean.lastIndexOf("o"));         // ตำแหน่งสุดท้ายที่เจอ 'o'
        System.out.println(clean.contains("World"));        // true
        System.out.println(clean.startsWith("Hello"));      // true
        System.out.println(clean.endsWith("!"));            // true
        System.out.println(clean.substring(7));             // "Java World!" (ตั้งแต่ index 7 ถึงจบ)
        System.out.println(clean.substring(7, 11));          // "Java" (index 7 ถึงก่อน 11)
        System.out.println(clean.replace("Java", "Python")); // "Hello, Python World!"
        System.out.println(clean.equals("Hello, Java World!")); // true
        System.out.println(clean.equalsIgnoreCase("HELLO, JAVA WORLD!")); // true
        System.out.println(clean.isEmpty());                // false
        System.out.println("".isEmpty());                    // true
        System.out.println("   ".isBlank());                 // true (Java 11+, เช็คว่าว่างหรือมีแต่ space)

        String[] parts = clean.split(", ");                  // แยกด้วย regex
        for (String part : parts) {
            System.out.println("ส่วนที่แยกได้: " + part);
        }

        String joined = String.join("-", "2024", "01", "15"); // ต่อด้วยตัวคั่น
        System.out.println(joined); // "2024-01-15"

        char[] chars = clean.toCharArray();                   // แปลงเป็น char array
        System.out.println(chars.length);

        System.out.println(clean.repeat(2)); // "Hello, Java World!Hello, Java World!" (Java 11+)
    }
}
```

## 4. String Formatting (`String.format`, `printf`, Text Block)

```java
public class StringFormattingDemo {
    public static void main(String[] args) {
        String name = "Somchai";
        int age = 25;
        double salary = 45000.5;

        // String.format คืนค่าเป็น String ใหม่ (ไม่พิมพ์เอง)
        String formatted = String.format("ชื่อ: %s, อายุ: %d ปี, เงินเดือน: %.2f บาท",
                                          name, age, salary);
        System.out.println(formatted);

        // printf พิมพ์ออก console ทันที ใช้ format specifier แบบเดียวกัน
        System.out.printf("ชื่อ: %s, อายุ: %d ปี%n", name, age);

        // ตัวระบุ format ที่ใช้บ่อย
        System.out.printf("%5d%n", 42);      // จัดขวาความกว้าง 5:  "   42"
        System.out.printf("%-5d|%n", 42);    // จัดซ้ายความกว้าง 5: "42   |"
        System.out.printf("%05d%n", 42);     // เติม 0 นำหน้า:     "00042"
        System.out.printf("%.3f%n", 3.14159); // ทศนิยม 3 ตำแหน่ง:  "3.142"
        System.out.printf("%x%n", 255);       // เลขฐาน 16:        "ff"
        System.out.printf("%,d%n", 1000000);  // คั่นหลักพัน:       "1,000,000"
    }
}
```

ตารางตัวระบุ format ที่ใช้บ่อย:

| ตัวระบุ | ความหมาย |
|---|---|
| `%s` | String |
| `%d` | จำนวนเต็ม (decimal) |
| `%f` | จำนวนทศนิยม |
| `%.Nf` | ทศนิยม N ตำแหน่ง |
| `%c` | ตัวอักษรเดียว |
| `%b` | boolean |
| `%n` | ขึ้นบรรทัดใหม่ (แนะนำแทน `\n` เพราะรองรับทุก OS) |
| `%%` | เครื่องหมาย % จริง ๆ |

## 5. เปรียบเทียบ String อย่างถูกต้อง

ทบทวนจาก Part 4 อย่างละเอียดขึ้น: **ห้ามใช้ `==` เปรียบเทียบเนื้อหา String เด็ดขาด**
ยกเว้นตั้งใจเช็ค reference equality จริง ๆ

```java
public class StringComparisonDemo {
    public static void main(String[] args) {
        String input = new String("password123"); // สมมติมาจาก user input (ไม่มาจาก literal)
        String correct = "password123";

        System.out.println(input == correct);       // false! เสี่ยง bug ร้ายแรงถ้าใช้เช็ค login
        System.out.println(input.equals(correct));   // true (ถูกต้อง)

        // compareTo: เปรียบเทียบตามลำดับตัวอักษร (lexicographic), คืนค่า int
        System.out.println("apple".compareTo("banana")); // ค่าติดลบ (apple มาก่อน banana)
        System.out.println("banana".compareTo("apple"));  // ค่าบวก
        System.out.println("apple".compareTo("apple"));   // 0 (เท่ากัน)

        // Objects.equals() ปลอดภัยกว่าเมื่อค่าอาจเป็น null
        String maybeNull = null;
        // maybeNull.equals("test");  // NullPointerException! ถ้า maybeNull เป็น null
        System.out.println(java.util.Objects.equals(maybeNull, "test")); // false (ปลอดภัย ไม่พัง)

        // เทคนิคป้องกัน: เขียนค่าที่รู้ว่าไม่ null ไว้ทางซ้ายของ .equals()
        System.out.println("test".equals(maybeNull)); // false (ปลอดภัยเช่นกัน)
    }
}
```

## 6. ทำไม String ต่อกันเยอะ ๆ ใน loop ถึงช้า

เพราะ String เป็น **immutable** ทุกครั้งที่ใช้ `+` ต่อ String จะมีการ**สร้าง object
String ใหม่ทั้งหมด** (คัดลอกของเก่า + เติมของใหม่) ถ้าทำใน loop หลายพันรอบ
จะเสียเวลาและหน่วยความจำมหาศาลโดยไม่จำเป็น:

```java
public class InefficiientStringDemo {
    public static void main(String[] args) {
        long start = System.currentTimeMillis();

        String result = "";
        for (int i = 0; i < 50_000; i++) {
            result += i; // ทุกรอบสร้าง String object ใหม่ทั้งหมด! O(n^2) โดยรวม
        }

        long end = System.currentTimeMillis();
        System.out.println("ใช้เวลา (ต่อด้วย +): " + (end - start) + " ms");
    }
}
```

## 7. `StringBuilder` และ `StringBuffer`

`StringBuilder` คือคลาสที่ใช้สร้างและแก้ไขข้อความแบบ **mutable** (แก้ไขได้โดยไม่ต้อง
สร้าง object ใหม่ทุกครั้ง) เหมาะมากสำหรับการต่อ String จำนวนมากใน loop

```java
public class StringBuilderDemo {
    public static void main(String[] args) {
        long start = System.currentTimeMillis();

        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < 50_000; i++) {
            sb.append(i); // แก้ไข buffer ภายในตัวเดิม ไม่สร้าง object ใหม่ทุกรอบ - เร็วกว่ามาก
        }
        String result = sb.toString(); // แปลงเป็น String ตอนจบเท่านั้น

        long end = System.currentTimeMillis();
        System.out.println("ใช้เวลา (StringBuilder): " + (end - start) + " ms");
        System.out.println("ความยาวผลลัพธ์: " + result.length());
    }
}
```

เมธอดสำคัญของ `StringBuilder`:

```java
public class StringBuilderMethodsDemo {
    public static void main(String[] args) {
        StringBuilder sb = new StringBuilder("Hello");

        sb.append(" World");           // ต่อท้าย: "Hello World"
        sb.insert(5, ",");              // แทรกที่ index 5: "Hello, World"
        sb.replace(0, 5, "Hi");         // แทนที่ index 0-4 ด้วย "Hi": "Hi, World"
        sb.deleteCharAt(0);             // ลบ 1 ตัวอักษรที่ index 0: "i, World"
        sb.delete(0, 2);                 // ลบช่วง index 0-1: ", World"
        sb.reverse();                    // กลับด้าน: "dlroW ,"
        System.out.println(sb);

        StringBuilder sb2 = new StringBuilder("Hello World");
        sb2.setCharAt(0, 'J');           // เปลี่ยนตัวอักษรที่ index 0
        System.out.println(sb2);         // "Jello World"

        System.out.println(sb2.length());     // ความยาวปัจจุบัน
        System.out.println(sb2.capacity());   // ความจุภายใน buffer (มากกว่า length เผื่อขยาย)
    }
}
```

## 8. ความแตกต่างระหว่าง StringBuilder กับ StringBuffer

| คุณสมบัติ | `StringBuilder` | `StringBuffer` |
|---|---|---|
| Thread-safe | ไม่ (เร็วกว่า) | ใช่ (เมธอดเป็น `synchronized`) |
| ความเร็ว | เร็วกว่า | ช้ากว่าเล็กน้อย เพราะมี lock overhead |
| ปีที่เพิ่มเข้ามา | Java 5 | Java 1.0 (เก่ากว่ามาก) |
| ควรใช้เมื่อ | โค้ด single-thread (ส่วนใหญ่ 95% ของกรณีใช้งาน) | หลาย thread แก้ไข buffer เดียวกันพร้อมกัน |

```java
public class StringBufferDemo {
    public static void main(String[] args) {
        // API เหมือนกับ StringBuilder ทุกประการ ต่างแค่ thread-safety
        StringBuffer buffer = new StringBuffer();
        buffer.append("Thread-safe ").append("String Buffer");
        System.out.println(buffer);
    }
}
```

**คำแนะนำในทางปฏิบัติ**: ใช้ `StringBuilder` เป็นค่าเริ่มต้นเสมอ เว้นแต่มีเหตุผล
ชัดเจนว่าต้องการ thread-safety (ซึ่งในทางปฏิบัติมักแก้ปัญหาด้วยวิธีอื่นที่ดีกว่า เช่น
ทำให้แต่ละ thread มี StringBuilder ของตัวเอง แล้วค่อยรวมผลลัพธ์)

## 9. Text Blocks (Java 15+)

**Text Block** ช่วยเขียนข้อความหลายบรรทัด (เช่น JSON, HTML, SQL) ได้อ่านง่ายกว่ามาก
โดยไม่ต้องมี `\n` และ `+` เกลื่อนโค้ด

```java
public class TextBlockDemo {
    public static void main(String[] args) {
        // แบบเก่า: อ่านยาก มี escape เต็มไปหมด
        String jsonOld = "{\n" +
                         "  \"name\": \"Somchai\",\n" +
                         "  \"age\": 25\n" +
                         "}";

        // แบบใหม่: Text Block (เริ่มด้วย \"\"\" แล้วขึ้นบรรทัดใหม่)
        String jsonNew = """
                {
                  "name": "Somchai",
                  "age": 25
                }
                """;

        System.out.println(jsonOld);
        System.out.println("---");
        System.out.println(jsonNew);

        // ใช้กับ SQL ก็อ่านง่ายขึ้นมาก
        String sql = """
                SELECT id, name, email
                FROM users
                WHERE active = true
                ORDER BY name
                """;
        System.out.println(sql);
    }
}
```

**กฎสำคัญของ Text Block**: การเยื้อง (indentation) ของ closing `"""` จะกำหนดว่า
เยื้องส่วนไหนจะถูกตัดออกโดยอัตโนมัติ (incidental whitespace stripping)

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียนโปรแกรมตรวจสอบว่าข้อความที่รับเข้ามาเป็น **Palindrome** หรือไม่
(อ่านจากหน้าไปหลังกับหลังไปหน้าเหมือนกัน เช่น "level", "racecar")

**เฉลย:**

```java
public class Exercise1 {
    static boolean isPalindrome(String text) {
        String cleaned = text.toLowerCase().replaceAll("[^a-z0-9]", "");
        String reversed = new StringBuilder(cleaned).reverse().toString();
        return cleaned.equals(reversed);
    }

    public static void main(String[] args) {
        System.out.println(isPalindrome("level"));               // true
        System.out.println(isPalindrome("A man a plan a canal Panama")); // true
        System.out.println(isPalindrome("hello"));                 // false
    }
}
```

**2)** เขียนโปรแกรมนับจำนวนคำ (word count) ในประโยคที่รับเข้ามา

**เฉลย:**

```java
public class Exercise2 {
    public static void main(String[] args) {
        String sentence = "Java is a powerful programming language";
        String[] words = sentence.trim().split("\\s+");
        System.out.println("จำนวนคำ: " + words.length); // 6
    }
}
```

**3)** ใช้ `StringBuilder` เขียนเมธอดที่รับ array ของ String แล้วต่อรวมกันโดยคั่นด้วย
comma และครอบด้วยวงเล็บเหลี่ยม เช่น `["a", "b", "c"]` -> `[a, b, c]`

**เฉลย:**

```java
public class Exercise3 {
    static String formatList(String[] items) {
        StringBuilder sb = new StringBuilder("[");
        for (int i = 0; i < items.length; i++) {
            sb.append(items[i]);
            if (i < items.length - 1) {
                sb.append(", ");
            }
        }
        sb.append("]");
        return sb.toString();
    }

    public static void main(String[] args) {
        System.out.println(formatList(new String[]{"a", "b", "c"})); // [a, b, c]
    }
}
```

### สรุปเนื้อหา Part 9

- `String` เป็น immutable object — ทุกการ "แก้ไข" จะได้ String ใหม่เสมอ
- String Pool ช่วยประหยัดหน่วยความจำโดยแชร์ literal ที่มีเนื้อหาเดียวกัน
- ใช้ `.equals()` เปรียบเทียบเนื้อหาเสมอ ไม่ใช้ `==` (ยกเว้นตั้งใจเช็ค reference)
- ใช้ `StringBuilder` แทนการต่อ `+` ใน loop เพื่อประสิทธิภาพที่ดีกว่ามาก
- `StringBuffer` เหมือน `StringBuilder` แต่ thread-safe (ช้ากว่าเล็กน้อย)
- Text Block (`"""`) ช่วยเขียนข้อความหลายบรรทัดให้อ่านง่ายขึ้นมาก

**ต่อไป**: [Part 10 — Exception Handling เบื้องต้น](./part-010-exception-handling-basics.md)
