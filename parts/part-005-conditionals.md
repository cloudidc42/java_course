# Part 5: คำสั่งควบคุมเงื่อนไข if-else, switch

> ขั้นตอนที่ 41-50 ของหลักสูตร | ระดับ: พื้นฐาน

## สารบัญ

1. คำสั่ง `if`, `if-else`, `else if`
2. Nested if และการจัดรูปแบบให้อ่านง่าย
3. คำสั่ง `switch` แบบดั้งเดิม (Traditional switch statement)
4. Switch Expression (Java 14+) และ Arrow syntax
5. Pattern Matching for switch (Java 21)
6. การเลือกใช้ if-else vs switch
7. ข้อผิดพลาดที่พบบ่อย
8. แบบฝึกหัดและสรุป

---

## 1. คำสั่ง `if`, `if-else`, `else if`

```java
public class IfDemo {
    public static void main(String[] args) {
        int score = 82;

        // if เดี่ยว
        if (score >= 50) {
            System.out.println("สอบผ่าน");
        }

        // if-else
        if (score >= 50) {
            System.out.println("สอบผ่าน");
        } else {
            System.out.println("สอบตก");
        }

        // if-else if-else แบบลูกโซ่
        if (score >= 80) {
            System.out.println("เกรด A");
        } else if (score >= 70) {
            System.out.println("เกรด B");
        } else if (score >= 60) {
            System.out.println("เกรด C");
        } else if (score >= 50) {
            System.out.println("เกรด D");
        } else {
            System.out.println("เกรด F");
        }
    }
}
```

**ข้อสังเกตสำคัญ**: เงื่อนไขใน Java **ต้องเป็น `boolean` เท่านั้น** (ไม่เหมือน C/C++
ที่ยอมให้ใช้ตัวเลขแทนได้) ดังนั้น `if (score)` จะ compile error ทันทีถ้า `score`
เป็น `int` — นี่คือหนึ่งในกลไก type-safety ของ Java

```java
int flag = 1;
// if (flag) { ... }  // Error! incompatible types: int cannot be converted to boolean
if (flag == 1) { ... } // ถูกต้อง ต้องเขียนเงื่อนไขให้ได้ boolean เสมอ
```

## 2. Nested if และการจัดรูปแบบให้อ่านง่าย

```java
public class NestedIfDemo {
    public static void main(String[] args) {
        int age = 25;
        boolean hasLicense = true;

        if (age >= 18) {
            if (hasLicense) {
                System.out.println("สามารถขับรถได้");
            } else {
                System.out.println("อายุถึงเกณฑ์แต่ยังไม่มีใบขับขี่");
            }
        } else {
            System.out.println("อายุยังไม่ถึงเกณฑ์");
        }

        // มักเขียนรวมเงื่อนไขด้วย && เพื่อลดความซ้อนถ้าทำได้
        if (age >= 18 && hasLicense) {
            System.out.println("สามารถขับรถได้ (แบบรวมเงื่อนไข)");
        }
    }
}
```

**แนวปฏิบัติที่ดี**: หากมี nested if เกิน 2-3 ชั้น ให้พิจารณาใช้เทคนิค **Guard Clause**
(early return) เพื่อลดความซับซ้อนและทำให้โค้ดแบนราบ (flat) มากขึ้น:

```java
public class GuardClauseDemo {
    static String checkEligibility(int age, boolean hasLicense, boolean hasInsurance) {
        // Guard clauses: จบเงื่อนไขที่ไม่ผ่านทันที ไม่ต้อง nest ลึก
        if (age < 18) {
            return "อายุยังไม่ถึงเกณฑ์";
        }
        if (!hasLicense) {
            return "ยังไม่มีใบขับขี่";
        }
        if (!hasInsurance) {
            return "ยังไม่มีประกันภัย";
        }
        return "สามารถขับรถได้ทุกเงื่อนไข";
    }

    public static void main(String[] args) {
        System.out.println(checkEligibility(20, true, false));
    }
}
```

## 3. คำสั่ง `switch` แบบดั้งเดิม (Traditional switch statement)

```java
public class SwitchTraditionalDemo {
    public static void main(String[] args) {
        int dayNumber = 3;
        String dayName;

        switch (dayNumber) {
            case 1:
                dayName = "จันทร์";
                break;
            case 2:
                dayName = "อังคาร";
                break;
            case 3:
                dayName = "พุธ";
                break;
            case 4:
                dayName = "พฤหัสบดี";
                break;
            case 5:
                dayName = "ศุกร์";
                break;
            case 6:
            case 7:
                dayName = "วันหยุดสุดสัปดาห์"; // fall-through: case 6 ไม่มี break จะไหลไปทำ case 7
                break;
            default:
                dayName = "ไม่ใช่วันที่ถูกต้อง";
        }

        System.out.println(dayName);
    }
}
```

### กับดักสำคัญ: ลืม `break` (Fall-through)

```java
public class FallThroughPitfall {
    public static void main(String[] args) {
        int number = 2;

        switch (number) {
            case 1:
                System.out.println("หนึ่ง");
                // ลืม break!
            case 2:
                System.out.println("สอง");
                // ลืม break!
            case 3:
                System.out.println("สาม");
                break;
            default:
                System.out.println("อื่น ๆ");
        }
        // ผลลัพธ์: พิมพ์ทั้ง "สอง" และ "สาม" เพราะไม่มี break คั่น
        // (ตั้งใจใช้ fall-through ได้ในบางกรณี เช่น case 6/7 ด้านบน แต่ถ้าไม่ตั้งใจคือ bug)
    }
}
```

`switch` แบบดั้งเดิมรองรับชนิดข้อมูล: `byte`, `short`, `char`, `int`, wrapper class
ของทั้ง 4 ชนิดนี้, `String` (ตั้งแต่ Java 7), และ `enum`

## 4. Switch Expression (Java 14+) และ Arrow Syntax

Java 14 เปิดตัว **switch expression** ที่แก้ปัญหา fall-through และให้ syntax ที่กระชับ
และปลอดภัยกว่าเดิมมาก — **แนะนำให้ใช้แบบนี้เป็นหลักในโค้ดใหม่**

```java
public class SwitchExpressionDemo {
    public static void main(String[] args) {
        int dayNumber = 3;

        // Arrow syntax: ไม่มี fall-through, ไม่ต้องใช้ break
        String dayName = switch (dayNumber) {
            case 1 -> "จันทร์";
            case 2 -> "อังคาร";
            case 3 -> "พุธ";
            case 4 -> "พฤหัสบดี";
            case 5 -> "ศุกร์";
            case 6, 7 -> "วันหยุดสุดสัปดาห์"; // รวมหลาย case ด้วย comma ได้เลย
            default -> "ไม่ใช่วันที่ถูกต้อง";
        };

        System.out.println(dayName);

        // ถ้า logic ซับซ้อนกว่าค่าเดียว ใช้ block + yield
        int score = 75;
        String grade = switch (score / 10) {
            case 10, 9, 8 -> "A";
            case 7 -> "B";
            case 6 -> {
                System.out.println("เกือบผ่านแบบสวย ๆ");
                yield "C"; // yield ใช้คืนค่าจาก block ใน switch expression
            }
            default -> "F";
        };
        System.out.println("เกรด: " + grade);
    }
}
```

**ข้อดีของ switch expression**:
- ไม่มี fall-through โดยไม่ตั้งใจ (แต่ละ case จบในตัวเอง)
- เป็น **expression** ที่คืนค่าได้ตรง ๆ ไม่ต้องประกาศตัวแปรว่างไว้ก่อนแล้วมา assign
  ในแต่ละ case แบบเก่า
- Compiler บังคับให้ครอบคลุมทุกกรณี (exhaustiveness) โดยเฉพาะกับ `enum`

## 5. Pattern Matching for switch (Java 21)

Java 21 (LTS) ทำให้ `switch` ทำงานร่วมกับ pattern matching ได้ ตรวจสอบและ cast
ชนิดข้อมูลในขั้นตอนเดียว (จะใช้งานจริงจังใน Part 51 เรื่อง Records/Pattern Matching):

```java
public class PatternMatchingSwitchDemo {
    static String describe(Object obj) {
        return switch (obj) {
            case Integer i when i > 0 -> "จำนวนเต็มบวก: " + i;
            case Integer i -> "จำนวนเต็ม (ไม่บวก): " + i;
            case String s -> "ข้อความความยาว " + s.length();
            case null -> "ค่าว่าง (null)";
            default -> "ไม่ทราบชนิด";
        };
    }

    public static void main(String[] args) {
        System.out.println(describe(42));
        System.out.println(describe(-5));
        System.out.println(describe("Hello"));
        System.out.println(describe(null));
        System.out.println(describe(3.14));
    }
}
```

## 6. การเลือกใช้ if-else vs switch

| ใช้ `if-else` เมื่อ | ใช้ `switch` เมื่อ |
|---|---|
| เงื่อนไขเป็นช่วงค่า (range) เช่น `score >= 80` | เปรียบเทียบค่าคงที่แบบตรง ๆ (discrete values) |
| เงื่อนไขซับซ้อน ผสมหลายตัวแปร | ตรวจสอบตัวแปรเดียวกับหลายค่าที่เป็นไปได้ |
| เงื่อนไขเป็น boolean expression ทั่วไป | มีตัวเลือกจำนวนมาก (เช่น เมนู, วัน, เดือน) |

```java
// เหมาะกับ if-else: เงื่อนไขเป็นช่วง
if (temperature > 35) { ... }

// เหมาะกับ switch: ค่าคงที่แบบตรง ๆ หลายตัวเลือก
switch (month) {
    case 12, 1, 2 -> System.out.println("ฤดูหนาว");
    case 3, 4, 5 -> System.out.println("ฤดูร้อน");
    default -> System.out.println("ฤดูฝน");
}
```

## 7. ข้อผิดพลาดที่พบบ่อย

### 7.1 Assignment (`=`) แทน Comparison (`==`) โดยไม่ตั้งใจ

```java
int x = 5;
// if (x = 10) { ... }  // Error! ใน Java คุมเข้มไม่ให้เกิดกับดักนี้ (ต่างจาก C)
                        // เพราะ x = 10 คืนค่า int ไม่ใช่ boolean ทำให้ compile ไม่ผ่าน
if (x == 10) { }        // ถูกต้อง - นี่คือข้อดีของ Java ที่ป้องกัน bug คลาสสิกนี้ไว้แล้ว
```

### 7.2 Dangling else (else ผูกกับ if ไหน)

```java
public class DanglingElseDemo {
    public static void main(String[] args) {
        int x = 5, y = 10;

        // else จะผูกกับ if ที่ใกล้ที่สุดเสมอ (if ที่ 2)
        if (x > 0)
            if (y > 20)
                System.out.println("A");
            else
                System.out.println("B"); // ผลลัพธ์คือบรรทัดนี้ (ผูกกับ if (y > 20))

        // ควรใช้ { } เสมอเพื่อไม่ให้เกิดความกำกวม
        if (x > 0) {
            if (y > 20) {
                System.out.println("A");
            }
        } else {
            System.out.println("B"); // ตอนนี้ else ผูกกับ if (x > 0) แทน ความหมายเปลี่ยนไปเลย
        }
    }
}
```

**แนวปฏิบัติที่ดี**: ใส่ `{ }` ทุกครั้งแม้จะมีแค่ 1 statement เพื่อป้องกันความกำกวมและ
ป้องกัน bug ตอนมีคนมาแก้โค้ดทีหลัง

## 8. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียนโปรแกรมตรวจสอบเกรดจากคะแนน (0-100) โดยใช้ switch expression:
90-100=A, 80-89=B, 70-79=C, 60-69=D, ต่ำกว่า 60=F

**เฉลย:**

```java
public class Exercise1 {
    public static void main(String[] args) {
        int score = 88;
        String grade = switch (score / 10) {
            case 10, 9 -> "A";
            case 8 -> "B";
            case 7 -> "C";
            case 6 -> "D";
            default -> "F";
        };
        System.out.println("เกรด: " + grade);
    }
}
```

**2)** เขียนโปรแกรมตรวจสอบว่าปี (year) เป็นปีอธิกสุรทิน (leap year) หรือไม่
(กฎ: หารด้วย 4 ลงตัว และ (หารด้วย 100 ไม่ลงตัว หรือ หารด้วย 400 ลงตัว))

**เฉลย:**

```java
public class Exercise2 {
    public static void main(String[] args) {
        int year = 2024;
        boolean isLeap = (year % 4 == 0) && (year % 100 != 0 || year % 400 == 0);
        System.out.println(year + (isLeap ? " เป็นปีอธิกสุรทิน" : " ไม่ใช่ปีอธิกสุรทิน"));
    }
}
```

**3)** ใช้ guard clause เขียนเมธอดตรวจสอบสิทธิ์เข้าถึงเนื้อหา: ต้องมีอายุ >= 18 ปี,
เป็นสมาชิก (isMember), และยังไม่ถูกระงับบัญชี (isBanned == false)

**เฉลย:**

```java
public class Exercise3 {
    static boolean canAccess(int age, boolean isMember, boolean isBanned) {
        if (isBanned) return false;
        if (age < 18) return false;
        if (!isMember) return false;
        return true;
    }

    public static void main(String[] args) {
        System.out.println(canAccess(20, true, false)); // true
        System.out.println(canAccess(15, true, false)); // false
    }
}
```

### สรุปเนื้อหา Part 5

- เงื่อนไขใน Java ต้องเป็น `boolean` เท่านั้น ป้องกัน bug การพิมพ์ `=` แทน `==`
- ใช้ `{ }` เสมอแม้มี statement เดียว เพื่อป้องกัน dangling else และเพิ่มความชัดเจน
- Switch แบบดั้งเดิมต้องระวังเรื่อง fall-through หากลืม `break`
- **Switch Expression** (Java 14+) เป็นวิธีที่แนะนำในโค้ดใหม่ ปลอดภัยกว่า กระชับกว่า
- ใช้ Guard Clause (early return) แทน nested if ที่ลึกเกินไป เพื่อความอ่านง่าย

**ต่อไป**: [Part 6 — คำสั่งวนซ้ำ for, while, do-while](./part-006-loops.md)
