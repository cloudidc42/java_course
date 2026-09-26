# Part 4: ตัวดำเนินการ (Operators) ทั้งหมดใน Java

> ขั้นตอนที่ 31-40 ของหลักสูตร | ระดับ: พื้นฐาน

## สารบัญ

1. Arithmetic Operators (ตัวดำเนินการทางคณิตศาสตร์)
2. Unary Operators
3. Assignment Operators (รวม compound assignment)
4. Relational / Comparison Operators
5. Logical Operators และ Short-circuit Evaluation
6. Bitwise และ Shift Operators
7. Ternary Operator (Conditional Operator)
8. Operator Precedence (ลำดับความสำคัญ)
9. `instanceof` Operator
10. แบบฝึกหัดและสรุป

---

## 1. Arithmetic Operators (ตัวดำเนินการทางคณิตศาสตร์)

```java
public class ArithmeticDemo {
    public static void main(String[] args) {
        int a = 17;
        int b = 5;

        System.out.println("a + b = " + (a + b)); // บวก: 22
        System.out.println("a - b = " + (a - b)); // ลบ: 12
        System.out.println("a * b = " + (a * b)); // คูณ: 85
        System.out.println("a / b = " + (a / b)); // หาร (integer division!): 3
        System.out.println("a % b = " + (a % b)); // หารเอาเศษ (modulo): 2

        double x = 17.0;
        double y = 5.0;
        System.out.println("x / y = " + (x / y)); // หาร (floating-point): 3.4
    }
}
```

### กับดักสำคัญ: Integer Division

เมื่อหาร `int` ด้วย `int` ผลลัพธ์จะเป็น `int` เสมอ ทศนิยมจะถูกตัดทิ้งไปเลย
(ไม่ใช่การปัดเศษ) นี่คือข้อผิดพลาดที่มือใหม่พลาดบ่อยที่สุดข้อหนึ่ง:

```java
public class DivisionPitfall {
    public static void main(String[] args) {
        int total = 10;
        int count = 3;

        double wrongAverage = total / count;          // 3.0 (ผิด! หารแบบ int ก่อนแล้วค่อยแปลง)
        double correctAverage = (double) total / count; // 3.3333... (ถูก! แปลงก่อนหาร)

        System.out.println("ผิด: " + wrongAverage);
        System.out.println("ถูก: " + correctAverage);
    }
}
```

**หลักการ**: ถ้าต้องการผลลัพธ์เป็นทศนิยม ต้อง cast อย่างน้อยหนึ่งตัวดำเนินการ (operand)
ให้เป็น `double` หรือ `float` **ก่อน**ทำการหาร

### กับดักสำคัญ: หารด้วยศูนย์

```java
public class DivideByZero {
    public static void main(String[] args) {
        // int a = 10 / 0;         // ArithmeticException: / by zero (โปรแกรมพัง!)

        double b = 10.0 / 0;       // Infinity (ไม่ throw exception)
        double c = -10.0 / 0;      // -Infinity
        double d = 0.0 / 0;        // NaN (Not a Number)

        System.out.println(b);
        System.out.println(c);
        System.out.println(d);
        System.out.println(Double.isNaN(d)); // true - วิธีตรวจสอบ NaN ที่ถูกต้อง
    }
}
```

> การหาร `int` ด้วย 0 จะ throw `ArithmeticException` ทันที แต่การหาร `double` ด้วย 0
> จะได้ `Infinity`, `-Infinity`, หรือ `NaN` แทน (ตาม IEEE 754 floating-point standard)

## 2. Unary Operators

```java
public class UnaryDemo {
    public static void main(String[] args) {
        int x = 5;

        System.out.println(-x);   // Unary minus: -5
        System.out.println(+x);   // Unary plus: 5 (แทบไม่ได้ใช้ในทางปฏิบัติ)

        boolean flag = true;
        System.out.println(!flag); // Logical NOT: false

        // Increment/Decrement
        int a = 10;
        System.out.println(a++);  // Post-increment: พิมพ์ 10 ก่อน แล้วค่อยเพิ่มเป็น 11
        System.out.println(a);    // 11
        System.out.println(++a);  // Pre-increment: เพิ่มเป็น 12 ก่อน แล้วค่อยพิมพ์
        System.out.println(a);    // 12

        int b = 10;
        System.out.println(b--);  // Post-decrement: พิมพ์ 10 ก่อน แล้วค่อยลดเป็น 9
        System.out.println(--b);  // Pre-decrement: ลดเป็น 8 ก่อน แล้วค่อยพิมพ์
    }
}
```

### ความแตกต่างระหว่าง Pre และ Post ในนิพจน์ซับซ้อน

```java
public class PrePostSubtlety {
    public static void main(String[] args) {
        int i = 5;
        int result = i++ + ++i; // i++ ใช้ 5 ก่อน (i กลายเป็น 6), ++i เพิ่มก่อนใช้ (i กลายเป็น 7)
        System.out.println("result = " + result); // 5 + 7 = 12
        System.out.println("i = " + i);            // 7
    }
}
```

> คำแนะนำ: หลีกเลี่ยงการเขียนนิพจน์ที่ซับซ้อนแบบนี้ในโค้ดจริง เพราะอ่านยากและเสี่ยง bug
> ควรแยกเป็นคนละบรรทัดจะชัดเจนกว่า

## 3. Assignment Operators

```java
public class AssignmentDemo {
    public static void main(String[] args) {
        int x = 10;

        x += 5;  // เทียบเท่า x = x + 5;   -> 15
        x -= 3;  // เทียบเท่า x = x - 3;   -> 12
        x *= 2;  // เทียบเท่า x = x * 2;   -> 24
        x /= 4;  // เทียบเท่า x = x / 4;   -> 6
        x %= 4;  // เทียบเท่า x = x % 4;   -> 2

        System.out.println(x); // 2

        // Bitwise compound assignment
        int y = 12;
        y &= 10; // y = y & 10
        y |= 5;  // y = y | 5
        y ^= 3;  // y = y ^ 3
        y <<= 2; // y = y << 2
        y >>= 1; // y = y >> 1

        System.out.println(y);
    }
}
```

### เกร็ดความรู้: Compound Assignment มี Implicit Cast ซ่อนอยู่

```java
public class CompoundAssignmentCast {
    public static void main(String[] args) {
        byte b = 10;
        // b = b + 5;  // Error! ผลลัพธ์ของ b + 5 คือ int ไม่สามารถ assign กลับเป็น byte ได้ตรง ๆ
        b += 5;        // OK! เพราะ compound assignment แอบใส่ (byte) cast ให้อัตโนมัติ
                       // เทียบเท่ากับ b = (byte) (b + 5);
        System.out.println(b); // 15
    }
}
```

## 4. Relational / Comparison Operators

ผลลัพธ์ของตัวดำเนินการเปรียบเทียบเป็น `boolean` เสมอ (`true`/`false`)

```java
public class RelationalDemo {
    public static void main(String[] args) {
        int a = 10, b = 20;

        System.out.println(a == b);  // เท่ากับ: false
        System.out.println(a != b);  // ไม่เท่ากับ: true
        System.out.println(a > b);   // มากกว่า: false
        System.out.println(a < b);   // น้อยกว่า: true
        System.out.println(a >= 10); // มากกว่าหรือเท่ากับ: true
        System.out.println(a <= 9);  // น้อยกว่าหรือเท่ากับ: false
    }
}
```

### กับดักสำคัญ: การเปรียบเทียบ String ด้วย `==`

```java
public class StringEqualsPitfall {
    public static void main(String[] args) {
        String s1 = "hello";
        String s2 = "hello";
        String s3 = new String("hello");

        System.out.println(s1 == s2);       // true (มาจาก String Pool เดียวกัน - จะอธิบายใน Part 9)
        System.out.println(s1 == s3);       // false (!) s3 ถูกสร้างใหม่บน heap คนละที่
        System.out.println(s1.equals(s3));  // true (เปรียบเทียบ "เนื้อหา" ถูกต้อง)
    }
}
```

> **กฎทอง**: สำหรับ reference types (เช่น `String`, object ทุกชนิด) ให้ใช้ `.equals()`
> เปรียบเทียบ**เนื้อหา** เสมอ ใช้ `==` เปรียบเทียบเฉพาะ primitive types หรือต้องการเช็คว่า
> เป็น object ตัวเดียวกันในหน่วยความจำจริง ๆ (reference equality) เท่านั้น

## 5. Logical Operators และ Short-circuit Evaluation

```java
public class LogicalDemo {
    public static void main(String[] args) {
        boolean p = true, q = false;

        System.out.println(p && q); // AND: false (ทั้งคู่ต้องจริง)
        System.out.println(p || q); // OR: true (ข้อใดข้อหนึ่งจริงพอ)
        System.out.println(!p);     // NOT: false

        // Non-short-circuit versions (ประเมินทุกด้านเสมอ ใช้น้อยกว่ามาก)
        System.out.println(p & q);  // AND แบบไม่ short-circuit
        System.out.println(p | q);  // OR แบบไม่ short-circuit
        System.out.println(p ^ q);  // XOR: true ถ้าค่าต่างกัน
    }
}
```

### Short-circuit Evaluation คืออะไร และทำไมสำคัญมาก

`&&` และ `||` จะ**หยุดประเมินทันที**ถ้ารู้ผลลัพธ์แน่นอนแล้ว โดยไม่ต้องประเมินฝั่งขวา
ทั้งหมด — นี่เป็นเทคนิคสำคัญที่ใช้ป้องกัน `NullPointerException` ได้:

```java
public class ShortCircuitDemo {
    public static void main(String[] args) {
        String text = null;

        // ถ้าใช้ & แทน && บรรทัดนี้จะ throw NullPointerException ทันที
        if (text != null && text.length() > 0) {
            System.out.println("มีข้อความ: " + text);
        } else {
            System.out.println("ไม่มีข้อความ หรือเป็นค่าว่าง");
        }
        // เพราะ text != null เป็น false, && จึงหยุดทันทีโดยไม่ไปเรียก text.length()
        // ซึ่งจะพังถ้า text เป็น null
    }
}
```

## 6. Bitwise และ Shift Operators

ใช้ทำงานกับข้อมูลระดับ bit โดยตรง มักใช้ในงาน low-level เช่น flags, การเข้ารหัส,
หรือการ optimize performance

```java
public class BitwiseDemo {
    public static void main(String[] args) {
        int a = 12; // 0000 1100
        int b = 10; // 0000 1010

        System.out.println(a & b);  // AND ทีละ bit: 0000 1000 = 8
        System.out.println(a | b);  // OR ทีละ bit:  0000 1110 = 14
        System.out.println(a ^ b);  // XOR ทีละ bit: 0000 0110 = 6
        System.out.println(~a);     // NOT (bitwise complement): -13

        System.out.println(a << 2); // Left shift 2 ตำแหน่ง: 48 (คูณด้วย 2^2)
        System.out.println(a >> 2); // Right shift 2 ตำแหน่ง (arithmetic): 3 (หารด้วย 2^2)

        int negative = -8;
        System.out.println(negative >> 1);  // -4 (คงเครื่องหมายลบไว้)
        System.out.println(negative >>> 1); // 2147483644 (unsigned shift - เติม 0 นำหน้าเสมอ)
    }
}
```

**ความแตกต่างระหว่าง `>>` และ `>>>`**:
- `>>` (signed right shift): เติมบิตซ้ายด้วยบิตเครื่องหมายเดิม (คงค่าบวก/ลบไว้)
- `>>>` (unsigned right shift): เติมบิตซ้ายด้วย 0 เสมอ ไม่สนเครื่องหมาย

## 7. Ternary Operator (Conditional Operator)

`?:` เป็นตัวดำเนินการเดียวใน Java ที่รับ 3 operand (ternary) ใช้แทน if-else แบบสั้น ๆ

```java
public class TernaryDemo {
    public static void main(String[] args) {
        int score = 75;

        // เทียบเท่ากับ if-else 5 บรรทัด
        String result = (score >= 50) ? "สอบผ่าน" : "สอบตก";
        System.out.println(result);

        // ซ้อนกันได้ (แต่ไม่ควรซ้อนเกิน 1-2 ชั้น เพราะจะอ่านยาก)
        int age = 20;
        String category = (age < 13) ? "เด็ก" : (age < 20) ? "วัยรุ่น" : "ผู้ใหญ่";
        System.out.println(category);
    }
}
```

รูปแบบ: `เงื่อนไข ? ค่าถ้าจริง : ค่าถ้าเท็จ`

## 8. Operator Precedence (ลำดับความสำคัญ)

เมื่อมีตัวดำเนินการหลายตัวในนิพจน์เดียว Java จะประมวลผลตามลำดับความสำคัญ (จากสูงไปต่ำ):

| ลำดับ | ตัวดำเนินการ | ทิศทางการประมวลผล |
|---|---|---|
| 1 | `()` `[]` `.` | ซ้ายไปขวา |
| 2 | `++` `--` (unary) `!` `~` | ขวาไปซ้าย |
| 3 | `*` `/` `%` | ซ้ายไปขวา |
| 4 | `+` `-` (binary) | ซ้ายไปขวา |
| 5 | `<<` `>>` `>>>` | ซ้ายไปขวา |
| 6 | `<` `<=` `>` `>=` `instanceof` | ซ้ายไปขวา |
| 7 | `==` `!=` | ซ้ายไปขวา |
| 8 | `&` | ซ้ายไปขวา |
| 9 | `^` | ซ้ายไปขวา |
| 10 | `\|` | ซ้ายไปขวา |
| 11 | `&&` | ซ้ายไปขวา |
| 12 | `\|\|` | ซ้ายไปขวา |
| 13 | `?:` | ขวาไปซ้าย |
| 14 | `=` `+=` `-=` ฯลฯ | ขวาไปซ้าย |

```java
public class PrecedenceDemo {
    public static void main(String[] args) {
        int result = 2 + 3 * 4;      // คูณก่อนบวก: 2 + 12 = 14
        System.out.println(result);

        int result2 = (2 + 3) * 4;   // วงเล็บมี precedence สูงสุด: 5 * 4 = 20
        System.out.println(result2);

        boolean check = 5 > 3 && 2 < 4; // เปรียบเทียบก่อน แล้วค่อย &&: true && true = true
        System.out.println(check);
    }
}
```

> **คำแนะนำในทางปฏิบัติ**: แม้จะจำลำดับได้ ก็ควรใช้วงเล็บ `()` เพื่อความชัดเจนเสมอ
> เมื่อนิพจน์ซับซ้อน — โค้ดที่ต้องคิดนานว่า "อันไหนทำก่อน" คือโค้ดที่ไม่ดี

## 9. `instanceof` Operator

ใช้ตรวจสอบว่า object เป็นของชนิด (type) ใดหรือ subtype ของมันหรือไม่
(รายละเอียดเชิงลึกอยู่ใน Part 15 เรื่อง Polymorphism)

```java
public class InstanceofDemo {
    public static void main(String[] args) {
        Object obj = "Hello Java";

        if (obj instanceof String) {
            System.out.println("obj เป็น String");
        }

        // Pattern Matching for instanceof (Java 16+) - ลดโค้ดการ cast
        if (obj instanceof String str) {
            System.out.println("ความยาว: " + str.length()); // ใช้ str ได้ทันทีโดยไม่ต้อง cast เอง
        }
    }
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียนโปรแกรมคำนวณค่าเฉลี่ยของคะแนน 3 วิชา (int) ให้ได้ผลลัพธ์เป็นทศนิยมที่ถูกต้อง

**เฉลย:**

```java
public class Exercise1 {
    public static void main(String[] args) {
        int math = 85, science = 90, english = 78;
        double average = (math + science + english) / 3.0; // ใช้ 3.0 ไม่ใช่ 3
        System.out.println("คะแนนเฉลี่ย: " + average);
    }
}
```

**2)** ทำนายผลลัพธ์ของโค้ดนี้ก่อนรันจริง แล้วอธิบายเหตุผล:

```java
int x = 5;
int y = (x++ * 2) + (++x);
```

**เฉลย**: `x++` ใช้ค่า 5 ก่อน (x กลายเป็น 6) → `5 * 2 = 10`, จากนั้น `++x` เพิ่มก่อนใช้
(x กลายเป็น 7) → ผลลัพธ์คือ `10 + 7 = 17`, และค่า `x` สุดท้ายคือ `7`

**3)** ใช้ ternary operator เขียนโปรแกรมตรวจสอบว่าตัวเลขเป็นจำนวนคู่หรือคี่

**เฉลย:**

```java
public class Exercise3 {
    public static void main(String[] args) {
        int number = 17;
        String result = (number % 2 == 0) ? "จำนวนคู่" : "จำนวนคี่";
        System.out.println(number + " เป็น " + result);
    }
}
```

### สรุปเนื้อหา Part 4

- Integer division ตัดทศนิยมทิ้งเสมอ ต้อง cast เป็น `double`/`float` ก่อนหารถ้าต้องการ
  ผลลัพธ์แม่นยำ
- `==` เปรียบเทียบ reference ไม่ใช่เนื้อหา สำหรับ object ควรใช้ `.equals()`
- `&&`/`||` ทำ short-circuit evaluation ช่วยป้องกัน NullPointerException ได้
- Bitwise operators (`&`, `|`, `^`, `~`, `<<`, `>>`, `>>>`) ทำงานระดับ bit
- Ternary operator `?:` ใช้แทน if-else แบบสั้น
- ใช้วงเล็บเสมอเมื่อนิพจน์ซับซ้อน แทนที่จะพึ่งพา operator precedence ล้วน ๆ

**ต่อไป**: [Part 5 — คำสั่งควบคุมเงื่อนไข if-else, switch](./part-005-conditionals.md)
