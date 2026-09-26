# Part 45: Regular Expressions (Regex) ใน Java

> ขั้นตอนที่ 441-450 ของหลักสูตร | ระดับ: กลาง

## สารบัญ

1. Regex คืออะไร ใช้ทำอะไร
2. Syntax พื้นฐานของ Regex
3. `Pattern` และ `Matcher`
4. เมธอด Regex ของ `String`
5. Groups: การจับกลุ่มใน Regex
6. Named Groups
7. Quantifiers และ Greedy vs Lazy Matching
8. Character Classes และ Anchors ที่ใช้บ่อย
9. ตัวอย่าง Regex ที่ใช้จริงบ่อยที่สุด
10. แบบฝึกหัดและสรุป

---

## 1. Regex คืออะไร ใช้ทำอะไร

**Regular Expression (Regex)** คือภาษาขนาดเล็กสำหรับ**นิยามรูปแบบของข้อความ**
ใช้ในการ**ค้นหา, ตรวจสอบความถูกต้อง (validation), และแทนที่ข้อความ** — เป็น
เครื่องมือที่ทรงพลังมากแต่ก็อ่านยากถ้าไม่คุ้นเคย ("regex คือภาษาที่เขียนได้
ง่ายแต่อ่านยาก")

```java
public class RegexMotivationDemo {
    public static void main(String[] args) {
        String email = "somchai@example.com";

        // ไม่ใช้ regex: ต้องเขียน logic ตรวจสอบด้วยมือ ยืดยาวและมี bug ได้ง่าย
        boolean hasAt = email.contains("@");
        boolean hasDot = email.contains(".");
        // ... ต้องเช็คอีกหลายเงื่อนไข ซับซ้อนมากถ้าต้องการตรวจสอบให้ถูกต้องจริง ๆ

        // ใช้ regex: กระชับกว่ามาก (แม้จะอ่านยากในตอนแรก)
        boolean isValidEmail = email.matches("^[\\w.+-]+@[\\w-]+\\.[a-zA-Z]{2,}$");
        System.out.println(isValidEmail); // true
    }
}
```

## 2. Syntax พื้นฐานของ Regex

| Syntax | ความหมาย | ตัวอย่าง |
|---|---|---|
| `.` | ตัวอักษรใดก็ได้ 1 ตัว | `a.c` matches "abc", "axc" |
| `*` | ตัวก่อนหน้า 0 ตัวขึ้นไป | `ab*` matches "a", "ab", "abbb" |
| `+` | ตัวก่อนหน้า 1 ตัวขึ้นไป | `ab+` matches "ab", "abbb" (ไม่ match "a") |
| `?` | ตัวก่อนหน้า 0 หรือ 1 ตัว | `colou?r` matches "color", "colour" |
| `\d` | ตัวเลข 0-9 | `\d+` matches "123" |
| `\w` | ตัวอักษร, ตัวเลข, หรือ underscore | `\w+` matches "hello_123" |
| `\s` | whitespace (space, tab, newline) | `\s+` matches "   " |
| `[abc]` | ตัวอักษรตัวใดตัวหนึ่งใน a, b, c | `[abc]` matches "a" หรือ "b" หรือ "c" |
| `[^abc]` | ตัวอักษรที่ไม่ใช่ a, b, c | `[^abc]` matches "d", "x" ฯลฯ |
| `[a-z]` | ตัวอักษรใน range a ถึง z | `[a-z]+` matches "hello" |
| `^` | จุดเริ่มต้นของข้อความ | `^Hello` matches ที่ขึ้นต้นด้วย "Hello" |
| `$` | จุดสิ้นสุดของข้อความ | `world$` matches ที่ลงท้ายด้วย "world" |
| `\|` | หรือ (OR) | `cat\|dog` matches "cat" หรือ "dog" |
| `{n,m}` | จำนวนซ้ำ ระหว่าง n ถึง m ครั้ง | `\d{2,4}` matches เลข 2-4 หลัก |

## 3. `Pattern` และ `Matcher`

Java ใช้คลาส `java.util.regex.Pattern` (แทนรูปแบบที่ compile แล้ว) และ
`java.util.regex.Matcher` (ใช้ทำการ match กับข้อความจริง)

```java
import java.util.regex.Matcher;
import java.util.regex.Pattern;

public class PatternMatcherDemo {
    public static void main(String[] args) {
        Pattern pattern = Pattern.compile("\\d+"); // compile pattern ครั้งเดียว (ควร cache ไว้ใช้ซ้ำ)
        Matcher matcher = pattern.matcher("มีสินค้า 25 ชิ้น ราคา 199 บาท");

        while (matcher.find()) { // find() หาทีละ match ถัดไป
            System.out.println("เจอ: " + matcher.group() + " ที่ตำแหน่ง " + matcher.start());
        }
        // เจอ: 25 ที่ตำแหน่ง 9
        // เจอ: 199 ที่ตำแหน่ง 20
    }
}
```

**คำแนะนำด้านประสิทธิภาพ**: ถ้าใช้ pattern เดียวกันซ้ำหลายครั้ง (เช่นในลูป)
ควร `Pattern.compile()` **ครั้งเดียว**เก็บไว้ใช้ซ้ำ (เช่นเป็น `static final`
field) เพราะการ compile pattern มี cost — ต่างจาก `String.matches()` ที่
compile pattern ใหม่ทุกครั้งที่เรียก (หัวข้อ 4)

## 4. เมธอด Regex ของ `String`

`String` มีเมธอดสะดวก ๆ ที่ใช้ regex ภายใน (ทบทวนจาก Part 9):

```java
public class StringRegexMethodsDemo {
    public static void main(String[] args) {
        String text = "Hello, World! 123";

        // matches(): ตรวจสอบว่าข้อความทั้งหมดตรงกับ pattern หรือไม่ (ต้อง match ทั้งสตริง)
        System.out.println("12345".matches("\\d+"));       // true
        System.out.println("12345abc".matches("\\d+"));      // false (ไม่ใช่ทั้งหมดเป็นตัวเลข)

        // replaceAll(): แทนที่ทุกจุดที่ตรงกับ pattern
        String noDigits = text.replaceAll("\\d+", "");
        System.out.println(noDigits); // "Hello, World! "

        String noPunctuation = text.replaceAll("[,!]", "");
        System.out.println(noPunctuation); // "Hello World 123"

        // replaceFirst(): แทนที่แค่จุดแรกที่เจอ
        String firstDigitReplaced = text.replaceFirst("\\d", "#");
        System.out.println(firstDigitReplaced); // "Hello, World! #23"

        // split(): แยกข้อความด้วย regex (ทบทวนจาก Part 9)
        String[] words = "apple, banana,  cherry".split(",\\s*"); // คั่นด้วย comma และ space (0 ตัวขึ้นไป)
        System.out.println(java.util.Arrays.toString(words)); // [apple, banana, cherry]
    }
}
```

## 5. Groups: การจับกลุ่มใน Regex

**Group** ใช้ **วงเล็บ `()`** จับส่วนหนึ่งของ pattern เพื่อดึงออกมาใช้แยกกัน
(เช่น แยกวันที่เป็น ปี/เดือน/วัน)

```java
import java.util.regex.Matcher;
import java.util.regex.Pattern;

public class RegexGroupsDemo {
    public static void main(String[] args) {
        String date = "2024-01-15";
        Pattern pattern = Pattern.compile("(\\d{4})-(\\d{2})-(\\d{2})");
        Matcher matcher = pattern.matcher(date);

        if (matcher.matches()) {
            System.out.println("Group 0 (ทั้งหมด): " + matcher.group(0)); // 2024-01-15
            System.out.println("Group 1 (ปี): " + matcher.group(1));       // 2024
            System.out.println("Group 2 (เดือน): " + matcher.group(2));      // 01
            System.out.println("Group 3 (วัน): " + matcher.group(3));         // 15
        }

        // ตัวอย่างการแยกชื่อ-นามสกุล
        String fullName = "Somchai Jaidee";
        Matcher nameMatcher = Pattern.compile("(\\w+)\\s(\\w+)").matcher(fullName);
        if (nameMatcher.matches()) {
            System.out.println("ชื่อ: " + nameMatcher.group(1) + ", นามสกุล: " + nameMatcher.group(2));
        }
    }
}
```

## 6. Named Groups

ใช้ `(?<name>...)` ตั้งชื่อ group เพื่อให้โค้ดอ่านง่ายขึ้น (ไม่ต้องนับตำแหน่ง
group เอง):

```java
import java.util.regex.Matcher;
import java.util.regex.Pattern;

public class NamedGroupsDemo {
    public static void main(String[] args) {
        String date = "2024-01-15";
        Pattern pattern = Pattern.compile("(?<year>\\d{4})-(?<month>\\d{2})-(?<day>\\d{2})");
        Matcher matcher = pattern.matcher(date);

        if (matcher.matches()) {
            System.out.println("ปี: " + matcher.group("year"));   // อ่านง่ายกว่า group(1) มาก
            System.out.println("เดือน: " + matcher.group("month"));
            System.out.println("วัน: " + matcher.group("day"));
        }
    }
}
```

## 7. Quantifiers และ Greedy vs Lazy Matching

**Greedy** (ค่าเริ่มต้นของ `*`, `+`, `{n,m}`) จะพยายาม match **ให้ได้มากที่สุด
เท่าที่เป็นไปได้** — **Lazy** (เติม `?` ต่อท้าย quantifier) จะ match
**ให้ได้น้อยที่สุด**

```java
import java.util.regex.Matcher;
import java.util.regex.Pattern;

public class GreedyLazyDemo {
    public static void main(String[] args) {
        String html = "<div>Hello</div><div>World</div>";

        // Greedy: .* พยายาม match ให้ยาวที่สุด -> จับตั้งแต่ <div> แรกถึง </div> สุดท้ายเลย!
        Matcher greedyMatcher = Pattern.compile("<div>.*</div>").matcher(html);
        if (greedyMatcher.find()) {
            System.out.println("Greedy: " + greedyMatcher.group());
            // "<div>Hello</div><div>World</div>" (!) จับทั้งหมดเพราะ .* โลภมาก
        }

        // Lazy: .*? พยายาม match ให้สั้นที่สุด -> จับแค่ <div> คู่แรกเท่านั้น
        Matcher lazyMatcher = Pattern.compile("<div>.*?</div>").matcher(html);
        if (lazyMatcher.find()) {
            System.out.println("Lazy: " + lazyMatcher.group());
            // "<div>Hello</div>" (ถูกต้องตามที่ต้องการมากกว่า)
        }
    }
}
```

**บทเรียนสำคัญ**: การ parse HTML/XML ด้วย regex เป็นเรื่องที่**เสี่ยงมาก**
(regex ไม่เหมาะกับโครงสร้างที่ nested ซับซ้อน) — ในทางปฏิบัติควรใช้ HTML/XML
parser เฉพาะทาง (เช่น Jsoup, JAXB) แทนการเขียน regex เอง

## 8. Character Classes และ Anchors ที่ใช้บ่อย

```java
import java.util.regex.Pattern;

public class CommonPatternsDemo {
    public static void main(String[] args) {
        // \b คือ word boundary (ขอบของคำ) - ใช้ป้องกันการ match บางส่วนของคำอื่น
        System.out.println("cat".matches(".*\\bcat\\b.*"));         // true
        System.out.println("category".matches(".*\\bcat\\b.*"));     // false (cat เป็นส่วนหนึ่งของ category)

        // (?i) เปิด case-insensitive mode ภายใน pattern
        System.out.println(Pattern.matches("(?i)hello", "HELLO"));   // true

        // Anchors ร่วมกับ multiline (^ และ $ จับแต่ละบรรทัดแยกกัน)
        String multiline = "line1\nline2\nline3";
        Pattern pattern = Pattern.compile("^line", Pattern.MULTILINE);
        java.util.regex.Matcher matcher = pattern.matcher(multiline);
        int count = 0;
        while (matcher.find()) count++;
        System.out.println("จำนวนบรรทัดที่ขึ้นต้นด้วย 'line': " + count); // 3
    }
}
```

## 9. ตัวอย่าง Regex ที่ใช้จริงบ่อยที่สุด

```java
public class CommonRegexPatternsDemo {
    public static void main(String[] args) {
        // Email (แบบง่าย เหมาะกับ validation เบื้องต้น ไม่ครอบคลุม edge case ทั้งหมดของ RFC 5322)
        String emailPattern = "^[\\w.+-]+@[\\w-]+\\.[a-zA-Z]{2,}$";
        System.out.println("test@example.com".matches(emailPattern)); // true

        // เบอร์โทรศัพท์ไทย (รูปแบบ 08X-XXX-XXXX หรือ 0XXXXXXXXX)
        String phonePattern = "^0\\d{1}-?\\d{3}-?\\d{4}$";
        System.out.println("081-234-5678".matches(phonePattern)); // true

        // รหัสผ่านที่แข็งแรง (มีตัวเล็ก, ตัวใหญ่, ตัวเลข, อย่างน้อย 8 ตัว)
        String passwordPattern = "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d).{8,}$";
        System.out.println("Password123".matches(passwordPattern)); // true
        System.out.println("weak".matches(passwordPattern));          // false

        // URL แบบง่าย
        String urlPattern = "^https?://[\\w.-]+(?:\\.[\\w.-]+)+[\\w\\-._~:/?#\\[\\]@!$&'()*+,;=]*$";
        System.out.println("https://example.com/page".matches(urlPattern)); // true

        // เลขบัตรประชาชนไทย (13 หลัก)
        String thaiIdPattern = "^\\d{13}$";
        System.out.println("1234567890123".matches(thaiIdPattern)); // true

        // ลบ whitespace ส่วนเกิน (มากกว่า 1 ตัวติดกัน) ให้เหลือแค่ 1 ตัว
        String messyText = "Hello    World   Java";
        System.out.println(messyText.replaceAll("\\s+", " ")); // "Hello World Java"
    }
}
```

**คำเตือนสำคัญเรื่อง Email Validation**: มาตรฐาน RFC 5322 สำหรับ email ซับซ้อน
มากจนแทบเป็นไปไม่ได้ที่จะเขียน regex ที่ครอบคลุมทุกกรณีอย่างสมบูรณ์ — ในทาง
ปฏิบัติจริง**การตรวจสอบด้วยการส่งอีเมลยืนยัน (verification email) สำคัญกว่า
regex ที่ซับซ้อน** — regex เพียงใช้กรองรูปแบบที่ผิดพลาดอย่างเห็นได้ชัดเท่านั้น

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียน regex ตรวจสอบว่า String เป็นเลขทศนิยมที่ถูกต้อง (เช่น "123",
"123.45", "-99.9")

**เฉลย:**

```java
public class Exercise1 {
    public static void main(String[] args) {
        String pattern = "^-?\\d+(\\.\\d+)?$";
        System.out.println("123".matches(pattern));    // true
        System.out.println("123.45".matches(pattern));  // true
        System.out.println("-99.9".matches(pattern));    // true
        System.out.println("12.34.56".matches(pattern));  // false
    }
}
```

**2)** ใช้ Named Groups แยกส่วนประกอบของ URL (protocol, domain, path)

**เฉลย:**

```java
import java.util.regex.Matcher;
import java.util.regex.Pattern;

public class Exercise2 {
    public static void main(String[] args) {
        String url = "https://example.com/products/123";
        Pattern pattern = Pattern.compile("(?<protocol>https?)://(?<domain>[^/]+)(?<path>/.*)?");
        Matcher matcher = pattern.matcher(url);

        if (matcher.matches()) {
            System.out.println("Protocol: " + matcher.group("protocol"));
            System.out.println("Domain: " + matcher.group("domain"));
            System.out.println("Path: " + matcher.group("path"));
        }
    }
}
```

**3)** เขียนโปรแกรมที่แทนที่ทุกคำที่เป็น "bad word" ในข้อความด้วย "***" โดยใช้
`\b` (word boundary) เพื่อไม่ให้ match ส่วนหนึ่งของคำอื่น

**เฉลย:**

```java
public class Exercise3 {
    public static void main(String[] args) {
        String text = "This is a badword and classic bad situation";
        String censored = text.replaceAll("\\bbad\\b", "***");
        System.out.println(censored); // "This is a badword and classic *** situation"
        // สังเกต: "badword" ไม่ถูกแทนที่ เพราะ \b ป้องกันการ match บางส่วนของคำอื่น
    }
}
```

### สรุปเนื้อหา Part 45

- Regex นิยามรูปแบบข้อความสำหรับ search, validate, และ replace
- `Pattern`/`Matcher` ใช้ทำงานกับ regex ใน Java, `String.matches/replaceAll/
  split` เป็นทางลัดที่สะดวก
- Groups (`()`) จับส่วนของ pattern, Named Groups (`(?<name>...)`) ทำให้อ่าน
  ง่ายกว่า
- Greedy quantifier match ให้มากที่สุด, Lazy (`?`) match ให้น้อยที่สุด — ระวัง
  เมื่อ parse โครงสร้างซ้อนกัน (ควรใช้ parser เฉพาะทางแทน regex สำหรับ HTML/XML)
- ควร `Pattern.compile()` เก็บไว้ใช้ซ้ำถ้าเรียกบ่อย เพื่อประสิทธิภาพที่ดีกว่า

**ต่อไป**: [Part 46 — Multithreading เบื้องต้น: Thread, Runnable](./part-046-multithreading-basics.md)
