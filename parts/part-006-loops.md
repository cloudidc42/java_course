# Part 6: คำสั่งวนซ้ำ for, while, do-while

> ขั้นตอนที่ 51-60 ของหลักสูตร | ระดับ: พื้นฐาน

## สารบัญ

1. คำสั่ง `while`
2. คำสั่ง `do-while`
3. คำสั่ง `for` (Traditional for loop)
4. Enhanced for-loop (for-each)
5. `break` และ `continue`
6. Labeled loops (การควบคุม loop ซ้อนกัน)
7. Infinite loops และการใช้อย่างมีเจตนา
8. การเลือกใช้ loop แบบไหนเหมาะกับสถานการณ์ใด
9. ตัวอย่างโปรแกรมเชิงปฏิบัติ
10. แบบฝึกหัดและสรุป

---

## 1. คำสั่ง `while`

`while` ทำงานซ้ำ**ตราบใดที่เงื่อนไขเป็นจริง** โดยตรวจสอบเงื่อนไข**ก่อน**ทำงานทุกครั้ง
(อาจไม่ทำงานเลยสักครั้งถ้าเงื่อนไขเป็นเท็จตั้งแต่แรก)

```java
public class WhileDemo {
    public static void main(String[] args) {
        int count = 1;
        while (count <= 5) {
            System.out.println("รอบที่ " + count);
            count++; // ต้องเพิ่มค่าเอง ไม่งั้นจะเป็น infinite loop
        }
        System.out.println("จบการทำงาน");
    }
}
```

ผลลัพธ์:

```
รอบที่ 1
รอบที่ 2
รอบที่ 3
รอบที่ 4
รอบที่ 5
จบการทำงาน
```

ตัวอย่างที่ไม่ทำงานเลยแม้แต่ครั้งเดียว:

```java
public class WhileNeverRuns {
    public static void main(String[] args) {
        int x = 10;
        while (x < 5) {
            System.out.println("จะไม่ถูกพิมพ์เลย เพราะ 10 ไม่น้อยกว่า 5");
        }
        System.out.println("ข้ามมาที่นี่ทันที");
    }
}
```

## 2. คำสั่ง `do-while`

`do-while` ตรวจสอบเงื่อนไข**หลัง**ทำงาน จึงรับประกันว่าจะทำงาน**อย่างน้อย 1 ครั้งเสมอ**

```java
public class DoWhileDemo {
    public static void main(String[] args) {
        int count = 1;
        do {
            System.out.println("รอบที่ " + count);
            count++;
        } while (count <= 5);

        // ตัวอย่างที่ทำงาน 1 ครั้งแม้เงื่อนไขเท็จตั้งแต่แรก
        int x = 10;
        do {
            System.out.println("ทำงานอย่างน้อย 1 ครั้งเสมอ แม้ x=" + x + " จะไม่น้อยกว่า 5");
        } while (x < 5);
    }
}
```

**กรณีใช้งานทั่วไปของ `do-while`**: เมนูโปรแกรมที่ต้องแสดงอย่างน้อยหนึ่งครั้งก่อนถาม
ว่าจะทำต่อหรือไม่ เช่น การรับ input จากผู้ใช้ที่ต้องแสดง prompt ก่อนตรวจสอบเงื่อนไข

```java
import java.util.Scanner;

public class MenuDemo {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);
        int choice;
        do {
            System.out.println("=== เมนู ===");
            System.out.println("1. เพิ่มข้อมูล");
            System.out.println("2. ลบข้อมูล");
            System.out.println("0. ออกจากโปรแกรม");
            System.out.print("เลือกเมนู: ");
            choice = scanner.nextInt();
            System.out.println("คุณเลือก: " + choice);
        } while (choice != 0);
        System.out.println("ออกจากโปรแกรมแล้ว");
    }
}
```

## 3. คำสั่ง `for` (Traditional for loop)

`for` เหมาะที่สุดเมื่อ**รู้จำนวนรอบที่แน่นอน**ล่วงหน้า มีโครงสร้าง 3 ส่วนคั่นด้วย `;`:

```java
for (การกำหนดค่าเริ่มต้น; เงื่อนไข; การเปลี่ยนแปลงค่า) {
    // คำสั่งที่ทำซ้ำ
}
```

```java
public class ForDemo {
    public static void main(String[] args) {
        for (int i = 1; i <= 5; i++) {
            System.out.println("รอบที่ " + i);
        }

        // นับถอยหลัง
        for (int i = 10; i >= 1; i--) {
            System.out.println(i);
        }
        System.out.println("ปล่อยจรวด!");

        // ก้าวกระโดดทีละ 2
        for (int i = 0; i <= 20; i += 2) {
            System.out.print(i + " ");
        }
        System.out.println();
    }
}
```

### รูปแบบขั้นสูงของ `for`: หลายตัวแปร, ว่างบางส่วน

```java
public class ForAdvancedDemo {
    public static void main(String[] args) {
        // มีหลายตัวแปรใน initialization และ update พร้อมกันด้วย comma
        for (int i = 0, j = 10; i < j; i++, j--) {
            System.out.println("i=" + i + ", j=" + j);
        }

        // สามารถเว้นว่างส่วนใดก็ได้ (แต่ต้องมี ; คั่นครบ 3 ส่วนเสมอ)
        int k = 0;
        for (; k < 3; ) {
            System.out.println("k=" + k);
            k++;
        }
    }
}
```

### Nested for loop (ลูปซ้อนลูป)

```java
public class NestedForDemo {
    public static void main(String[] args) {
        // สร้างตารางสูตรคูณ 1-3 x 1-3
        for (int i = 1; i <= 3; i++) {
            for (int j = 1; j <= 3; j++) {
                System.out.print(i * j + "\t");
            }
            System.out.println(); // ขึ้นบรรทัดใหม่หลังจบแต่ละแถว
        }

        // สร้างสามเหลี่ยมดาว
        int rows = 5;
        for (int i = 1; i <= rows; i++) {
            for (int j = 1; j <= i; j++) {
                System.out.print("*");
            }
            System.out.println();
        }
    }
}
```

ผลลัพธ์สามเหลี่ยมดาว:

```
*
**
***
****
*****
```

## 4. Enhanced for-loop (for-each)

ใช้สำหรับวนซ้ำ array หรือ collection โดยไม่ต้องจัดการ index เอง อ่านง่ายกว่ามาก
แต่**ไม่สามารถแก้ไข index หรือรู้ตำแหน่งปัจจุบันได้โดยตรง**

```java
public class ForEachDemo {
    public static void main(String[] args) {
        int[] numbers = {10, 20, 30, 40, 50};

        // for-each: อ่านง่าย เหมาะเมื่อไม่ต้องการ index
        for (int num : numbers) {
            System.out.println(num);
        }

        String[] fruits = {"แอปเปิ้ล", "กล้วย", "ส้ม"};
        for (String fruit : fruits) {
            System.out.println("ผลไม้: " + fruit);
        }

        // เทียบกับ traditional for ที่ทำแบบเดียวกันแต่ยาวกว่า
        for (int i = 0; i < numbers.length; i++) {
            System.out.println("index " + i + " = " + numbers[i]);
        }
    }
}
```

**ข้อควรระวัง**: for-each สร้าง copy ของค่า primitive ในแต่ละรอบ การแก้ไขตัวแปร loop
จะไม่ส่งผลกลับไปยัง array ต้นฉบับ:

```java
public class ForEachPitfall {
    public static void main(String[] args) {
        int[] numbers = {1, 2, 3};
        for (int num : numbers) {
            num = num * 10; // แก้แค่ copy ในตัวแปร num เท่านั้น
        }
        System.out.println(numbers[0]); // ยังคงเป็น 1 ไม่เปลี่ยนแปลง!

        // ถ้าต้องการแก้ไข array จริง ต้องใช้ traditional for พร้อม index
        for (int i = 0; i < numbers.length; i++) {
            numbers[i] = numbers[i] * 10;
        }
        System.out.println(numbers[0]); // ตอนนี้เป็น 10 แล้ว
    }
}
```

## 5. `break` และ `continue`

- **`break`**: หยุดการทำงานของ loop ทันทีและออกจาก loop นั้นเลย
- **`continue`**: ข้ามรอบปัจจุบัน ไปเริ่มรอบถัดไปทันที (ไม่ออกจาก loop)

```java
public class BreakContinueDemo {
    public static void main(String[] args) {
        // break: หยุดทันทีเมื่อเจอเลข 5
        System.out.println("--- ตัวอย่าง break ---");
        for (int i = 1; i <= 10; i++) {
            if (i == 5) {
                break;
            }
            System.out.println(i);
        }
        // ผลลัพธ์: พิมพ์ 1, 2, 3, 4 แล้วหยุด

        // continue: ข้ามเลขคู่ พิมพ์เฉพาะเลขคี่
        System.out.println("--- ตัวอย่าง continue ---");
        for (int i = 1; i <= 10; i++) {
            if (i % 2 == 0) {
                continue;
            }
            System.out.println(i);
        }
        // ผลลัพธ์: พิมพ์ 1, 3, 5, 7, 9
    }
}
```

## 6. Labeled loops (การควบคุม loop ซ้อนกัน)

ปกติ `break`/`continue` จะมีผลกับ loop ที่**ใกล้ที่สุด**เท่านั้น หากต้องการควบคุม
loop ชั้นนอกจากภายใน loop ชั้นใน ต้องใช้ **label**:

```java
public class LabeledLoopDemo {
    public static void main(String[] args) {
        // ต้องการหยุดทั้งสอง loop ทันทีที่เจอคู่ที่ผลรวมเป็น 8
        outerLoop:
        for (int i = 1; i <= 5; i++) {
            for (int j = 1; j <= 5; j++) {
                if (i + j == 8) {
                    System.out.println("เจอคู่: i=" + i + ", j=" + j);
                    break outerLoop; // ออกจาก loop ชั้นนอกเลย ไม่ใช่แค่ loop ชั้นใน
                }
            }
        }

        // labeled continue: ข้ามไปทำ loop ชั้นนอกรอบถัดไปทันที
        System.out.println("--- labeled continue ---");
        outer:
        for (int i = 1; i <= 3; i++) {
            for (int j = 1; j <= 3; j++) {
                if (j == 2) {
                    continue outer; // ข้ามไปที่ i รอบถัดไปทันที ไม่ทำ j=3 ต่อ
                }
                System.out.println("i=" + i + ", j=" + j);
            }
        }
    }
}
```

## 7. Infinite loops และการใช้อย่างมีเจตนา

บางครั้งเราต้องการ loop ที่ทำงานตลอดไปจนกว่าจะมีเงื่อนไขบางอย่างสั่งหยุด (เช่น
server ที่รอรับ request ตลอดเวลา) ใช้ `break` ควบคุมการออกจาก loop:

```java
public class InfiniteLoopDemo {
    public static void main(String[] args) {
        int attempts = 0;
        while (true) { // infinite loop โดยเจตนา
            attempts++;
            System.out.println("พยายามครั้งที่ " + attempts);
            if (attempts >= 3) {
                System.out.println("ครบจำนวนครั้งที่กำหนด หยุดการทำงาน");
                break; // จุดเดียวที่จะออกจาก loop นี้ได้
            }
        }

        // เทียบเท่ากับ for (;;)
        int counter = 0;
        for (;;) {
            counter++;
            if (counter >= 3) break;
        }
        System.out.println("counter = " + counter);
    }
}
```

## 8. การเลือกใช้ loop แบบไหนเหมาะกับสถานการณ์ใด

| สถานการณ์ | Loop ที่เหมาะสม |
|---|---|
| รู้จำนวนรอบแน่นอน (เช่น 1-10) | `for` |
| วนซ้ำ array/collection ทั้งหมดโดยไม่ต้องใช้ index | Enhanced `for` (for-each) |
| ไม่รู้จำนวนรอบ ขึ้นกับเงื่อนไขที่เปลี่ยนแปลง | `while` |
| ต้องทำงานอย่างน้อย 1 ครั้งเสมอ (เช่น เมนู, การยืนยัน) | `do-while` |
| ทำงานตลอดไปจนกว่าจะมีสัญญาณหยุด (เช่น server loop) | `while (true)` + `break` |

## 9. ตัวอย่างโปรแกรมเชิงปฏิบัติ

### หาผลรวมและค่าเฉลี่ยจากชุดตัวเลข

```java
public class SumAverageDemo {
    public static void main(String[] args) {
        int[] scores = {85, 92, 78, 90, 88};
        int sum = 0;

        for (int score : scores) {
            sum += score;
        }

        double average = (double) sum / scores.length;
        System.out.println("ผลรวม: " + sum);
        System.out.println("ค่าเฉลี่ย: " + average);
    }
}
```

### ตรวจสอบจำนวนเฉพาะ (Prime Number)

```java
public class PrimeCheckDemo {
    static boolean isPrime(int number) {
        if (number < 2) return false;
        for (int i = 2; i * i <= number; i++) { // เช็คแค่ถึง sqrt(number) พอ ไม่ต้องเช็คทุกตัว
            if (number % i == 0) {
                return false;
            }
        }
        return true;
    }

    public static void main(String[] args) {
        System.out.println("จำนวนเฉพาะระหว่าง 2-30:");
        for (int i = 2; i <= 30; i++) {
            if (isPrime(i)) {
                System.out.print(i + " ");
            }
        }
        System.out.println();
    }
}
```

### FizzBuzz (โจทย์สัมภาษณ์งานคลาสสิก)

```java
public class FizzBuzzDemo {
    public static void main(String[] args) {
        for (int i = 1; i <= 20; i++) {
            if (i % 15 == 0) {
                System.out.println("FizzBuzz");
            } else if (i % 3 == 0) {
                System.out.println("Fizz");
            } else if (i % 5 == 0) {
                System.out.println("Buzz");
            } else {
                System.out.println(i);
            }
        }
    }
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียนโปรแกรมหาผลรวมของเลข 1 ถึง 100 ด้วย `while` loop

**เฉลย:**

```java
public class Exercise1 {
    public static void main(String[] args) {
        int sum = 0, i = 1;
        while (i <= 100) {
            sum += i;
            i++;
        }
        System.out.println("ผลรวม 1-100 = " + sum); // 5050
    }
}
```

**2)** เขียนโปรแกรมพิมพ์ตัวเลข Fibonacci 10 ตัวแรก (0, 1, 1, 2, 3, 5, 8, ...)

**เฉลย:**

```java
public class Exercise2 {
    public static void main(String[] args) {
        int a = 0, b = 1;
        for (int i = 0; i < 10; i++) {
            System.out.print(a + " ");
            int next = a + b;
            a = b;
            b = next;
        }
    }
}
```

**3)** ใช้ labeled loop เขียนโปรแกรมค้นหาตำแหน่งแรกที่ผลคูณของ i*j มีค่าเท่ากับ 24
ในตาราง 1-10 x 1-10 แล้วหยุดทันทีที่เจอ

**เฉลย:**

```java
public class Exercise3 {
    public static void main(String[] args) {
        search:
        for (int i = 1; i <= 10; i++) {
            for (int j = 1; j <= 10; j++) {
                if (i * j == 24) {
                    System.out.println("เจอที่ i=" + i + ", j=" + j);
                    break search;
                }
            }
        }
    }
}
```

### สรุปเนื้อหา Part 6

- `while` ตรวจเงื่อนไขก่อนทำงาน อาจไม่ทำงานเลย, `do-while` ทำงานอย่างน้อย 1 ครั้งเสมอ
- `for` เหมาะกับกรณีรู้จำนวนรอบแน่นอน มีโครงสร้าง 3 ส่วน (init; condition; update)
- Enhanced for-loop (for-each) อ่านง่ายแต่แก้ไข array ต้นฉบับไม่ได้โดยตรง
- `break` ออกจาก loop ทันที, `continue` ข้ามไปรอบถัดไป
- Labeled loop ใช้ควบคุม loop ซ้อนกันจากภายในได้

**ต่อไป**: [Part 7 — Arrays หนึ่งมิติและหลายมิติ](./part-007-arrays.md)
