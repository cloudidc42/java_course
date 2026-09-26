# Part 7: Arrays หนึ่งมิติและหลายมิติ

> ขั้นตอนที่ 61-70 ของหลักสูตร | ระดับ: พื้นฐาน

## สารบัญ

1. Array คืออะไร ทำไมต้องใช้
2. การประกาศและสร้าง Array หนึ่งมิติ
3. การเข้าถึงและแก้ไขสมาชิกใน Array
4. `ArrayIndexOutOfBoundsException`
5. Array ของ Object และค่าเริ่มต้น
6. Multidimensional Arrays (Array หลายมิติ)
7. Jagged Arrays (Array ที่แต่ละแถวความยาวไม่เท่ากัน)
8. คลาส `java.util.Arrays` (utility methods)
9. การส่ง Array เป็นพารามิเตอร์และคืนค่าจากเมธอด
10. แบบฝึกหัดและสรุป

---

## 1. Array คืออะไร ทำไมต้องใช้

**Array** คือโครงสร้างข้อมูลที่เก็บข้อมูล**ชนิดเดียวกัน**หลายค่าไว้ในตัวแปรเดียว
โดยแต่ละค่าถูกจัดเก็บต่อเนื่องกันในหน่วยความจำ และเข้าถึงผ่าน **index** (เริ่มที่ 0)

ทำไมไม่ประกาศตัวแปรทีละตัว? ลองเปรียบเทียบ:

```java
// ไม่ใช้ array: ต้องประกาศตัวแปรทีละตัว จัดการยากเมื่อข้อมูลเยอะ
int score1 = 80, score2 = 90, score3 = 75, score4 = 88, score5 = 92;

// ใช้ array: จัดการข้อมูลชุดใหญ่ได้อย่างเป็นระบบ วนซ้ำได้ด้วย loop
int[] scores = {80, 90, 75, 88, 92};
```

Array ใน Java เป็น **reference type** (ไม่ใช่ primitive) มีขนาด**คงที่**เมื่อสร้างแล้ว
(ถ้าต้องการขนาดที่ปรับได้ ต้องใช้ `ArrayList` ซึ่งจะเรียนใน Part 22)

## 2. การประกาศและสร้าง Array หนึ่งมิติ

มี 3 วิธีหลักในการสร้าง array:

```java
public class ArrayCreationDemo {
    public static void main(String[] args) {
        // วิธีที่ 1: ประกาศแล้วกำหนดขนาด (ค่าเริ่มต้นเป็น 0 สำหรับตัวเลข)
        int[] numbers1 = new int[5]; // array ขนาด 5 ช่อง: [0, 0, 0, 0, 0]

        // วิธีที่ 2: ประกาศพร้อมค่าเริ่มต้นทันที (array literal)
        int[] numbers2 = {10, 20, 30, 40, 50};

        // วิธีที่ 3: ใช้ new พร้อมกำหนดค่าเริ่มต้น (เทียบเท่าวิธีที่ 2)
        int[] numbers3 = new int[]{10, 20, 30, 40, 50};

        // รูปแบบการประกาศชนิดข้อมูล [] วางได้ทั้งหลังชนิดข้อมูลและหลังชื่อตัวแปร (แบบแรกนิยมกว่า)
        int[] style1 = {1, 2, 3};  // แนะนำ (Java style)
        int style2[] = {1, 2, 3};  // ใช้ได้ (C style) แต่ไม่นิยมใน Java

        System.out.println(numbers1.length); // 5
        System.out.println(numbers2.length); // 5
    }
}
```

**หมายเหตุสำคัญ**: `.length` เป็น**field** (ไม่ใช่เมธอด) ของ array ดังนั้นเขียน
`numbers.length` ไม่ใช่ `numbers.length()` (ต่างจาก `String.length()` ที่เป็นเมธอด
ต้องมีวงเล็บ — เป็นจุดที่มือใหม่สับสนบ่อยมาก)

## 3. การเข้าถึงและแก้ไขสมาชิกใน Array

Index ของ array เริ่มต้นที่ **0 เสมอ** และสมาชิกตัวสุดท้ายอยู่ที่ index `length - 1`

```java
public class ArrayAccessDemo {
    public static void main(String[] args) {
        int[] numbers = {10, 20, 30, 40, 50};
        //                0   1   2   3   4   <- index

        System.out.println(numbers[0]);  // 10 (ตัวแรก)
        System.out.println(numbers[4]);  // 50 (ตัวสุดท้าย)
        System.out.println(numbers[numbers.length - 1]); // 50 (วิธีเข้าถึงตัวสุดท้ายที่ปลอดภัย)

        numbers[2] = 999; // แก้ไขค่าที่ index 2
        System.out.println(numbers[2]); // 999

        // วนซ้ำแก้ไขทุกค่า (คูณ 2 ทุกตัว)
        for (int i = 0; i < numbers.length; i++) {
            numbers[i] = numbers[i] * 2;
        }

        for (int num : numbers) {
            System.out.print(num + " ");
        }
    }
}
```

## 4. `ArrayIndexOutOfBoundsException`

การเข้าถึง index ที่ไม่มีอยู่จริง (ติดลบ หรือมากกว่าหรือเท่ากับ `length`) จะทำให้เกิด
`ArrayIndexOutOfBoundsException` ทันที (runtime exception — โปรแกรมจะพังถ้าไม่จัดการ)

```java
public class OutOfBoundsDemo {
    public static void main(String[] args) {
        int[] numbers = {10, 20, 30};

        System.out.println(numbers[2]); // OK: 30

        try {
            System.out.println(numbers[3]); // Error! index 3 ไม่มีอยู่ (มีแค่ 0,1,2)
        } catch (ArrayIndexOutOfBoundsException e) {
            System.out.println("เกิดข้อผิดพลาด: " + e.getMessage());
        }

        try {
            System.out.println(numbers[-1]); // Error! index ติดลบไม่มีอยู่จริง
        } catch (ArrayIndexOutOfBoundsException e) {
            System.out.println("เกิดข้อผิดพลาด: " + e.getMessage());
        }
    }
}
```

**หลักการป้องกัน**: เมื่อวน loop เข้าถึง array ด้วย index เสมอตรวจสอบเงื่อนไขให้ถูกต้อง
เช่น `i < array.length` (ไม่ใช่ `i <= array.length`) — ความผิดพลาดนี้เรียกว่า
**off-by-one error** พบบ่อยมากในหมู่มือใหม่

## 5. Array ของ Object และค่าเริ่มต้น

Array ของ reference type (เช่น `String[]`) จะมีค่าเริ่มต้นเป็น `null` ไม่ใช่ค่าว่าง
ต้องระวังการเรียกใช้เมธอดบนสมาชิกที่ยังเป็น `null`:

```java
public class ObjectArrayDemo {
    public static void main(String[] args) {
        String[] names = new String[3]; // สร้าง array 3 ช่อง แต่ละช่องเป็น null

        System.out.println(names[0]); // null

        try {
            System.out.println(names[0].length()); // NullPointerException! เรียกเมธอดบน null
        } catch (NullPointerException e) {
            System.out.println("เกิด NullPointerException เพราะช่องนี้ยังไม่ถูกกำหนดค่า");
        }

        names[0] = "Alice";
        names[1] = "Bob";
        // names[2] ยังคงเป็น null

        for (String name : names) {
            System.out.println(name != null ? name : "(ยังไม่มีค่า)");
        }
    }
}
```

ตารางค่าเริ่มต้นของ array แต่ละชนิด:

| ชนิดข้อมูลของ array | ค่าเริ่มต้นของแต่ละช่อง |
|---|---|
| `int[]`, `long[]`, `short[]`, `byte[]` | `0` |
| `double[]`, `float[]` | `0.0` |
| `boolean[]` | `false` |
| `char[]` | `'\u0000'` |
| `String[]`, object array อื่น ๆ | `null` |

## 6. Multidimensional Arrays (Array หลายมิติ)

Array 2 มิติ (เช่น ตาราง/เมทริกซ์) คือ "array ของ array":

```java
public class TwoDArrayDemo {
    public static void main(String[] args) {
        // สร้าง array 2 มิติขนาด 3 แถว 4 คอลัมน์
        int[][] matrix = new int[3][4];

        // กำหนดค่าทีละช่อง
        matrix[0][0] = 1;
        matrix[1][2] = 99;

        // ประกาศพร้อมค่าเริ่มต้น (นิยมมากกว่า)
        int[][] grid = {
            {1, 2, 3},
            {4, 5, 6},
            {7, 8, 9}
        };

        // วนซ้ำ 2 มิติด้วย nested for
        for (int row = 0; row < grid.length; row++) {
            for (int col = 0; col < grid[row].length; col++) {
                System.out.print(grid[row][col] + " ");
            }
            System.out.println();
        }

        // วนซ้ำด้วย enhanced for-each (สะดวกกว่า)
        System.out.println("--- ใช้ for-each ---");
        for (int[] row : grid) {
            for (int value : row) {
                System.out.print(value + " ");
            }
            System.out.println();
        }
    }
}
```

ผลลัพธ์:

```
1 2 3
4 5 6
7 8 9
--- ใช้ for-each ---
1 2 3
4 5 6
7 8 9
```

Array สามารถมีได้มากกว่า 2 มิติ (3 มิติ, 4 มิติ ฯลฯ) แต่ในทางปฏิบัติเกิน 2-3 มิติ
จะพบไม่บ่อยนัก เพราะเข้าใจยากและมักใช้โครงสร้างข้อมูลอื่นแทน:

```java
int[][][] threeD = new int[2][3][4]; // 2 ชั้น x 3 แถว x 4 คอลัมน์
threeD[0][1][2] = 100;
```

## 7. Jagged Arrays (Array ที่แต่ละแถวความยาวไม่เท่ากัน)

ต่างจากภาษาอื่น Java อนุญาตให้แต่ละแถวใน array 2 มิติมีความยาว**ไม่เท่ากันได้**
เรียกว่า **jagged array** (เพราะจริง ๆ แล้ว array 2 มิติของ Java คือ array ของ
array อ้างอิง ไม่ใช่ตารางสี่เหลี่ยมผืนผ้าที่แท้จริง):

```java
public class JaggedArrayDemo {
    public static void main(String[] args) {
        int[][] jagged = new int[3][]; // ประกาศแค่จำนวนแถว ยังไม่กำหนดความยาวคอลัมน์

        jagged[0] = new int[]{1};
        jagged[1] = new int[]{1, 2, 3};
        jagged[2] = new int[]{1, 2, 3, 4, 5};

        for (int[] row : jagged) {
            for (int value : row) {
                System.out.print(value + " ");
            }
            System.out.println(" (ความยาวแถวนี้: " + row.length + ")");
        }
    }
}
```

ผลลัพธ์:

```
1  (ความยาวแถวนี้: 1)
1 2 3  (ความยาวแถวนี้: 3)
1 2 3 4 5  (ความยาวแถวนี้: 5)
```

**กรณีใช้งานจริง**: ตารางสามเหลี่ยม Pascal's Triangle, ข้อมูลที่แต่ละกลุ่มมีจำนวน
สมาชิกไม่เท่ากัน (เช่น รายชื่อนักเรียนแต่ละห้องที่มีจำนวนคนไม่เท่ากัน)

## 8. คลาส `java.util.Arrays` (Utility Methods)

Java มีคลาส utility ชื่อ `Arrays` ที่รวมเมธอดสำเร็จรูปสำหรับจัดการ array
ช่วยลดโค้ดที่ต้องเขียนเองซ้ำ ๆ:

```java
import java.util.Arrays;

public class ArraysUtilDemo {
    public static void main(String[] args) {
        int[] numbers = {5, 3, 8, 1, 9, 2};

        // เรียงลำดับ (sort) - แก้ไข array ต้นฉบับโดยตรง (in-place)
        Arrays.sort(numbers);
        System.out.println(Arrays.toString(numbers)); // [1, 2, 3, 5, 8, 9]

        // แปลง array เป็น String ที่อ่านง่ายสำหรับ debug/print
        int[] simple = {1, 2, 3};
        System.out.println(Arrays.toString(simple)); // [1, 2, 3]

        // ค้นหาค่าใน array ที่ sort แล้วด้วย binary search (เร็วกว่า linear search มาก)
        int index = Arrays.binarySearch(numbers, 8);
        System.out.println("เจอเลข 8 ที่ index: " + index);

        // เติมค่าเดียวกันทุกช่อง
        int[] filled = new int[5];
        Arrays.fill(filled, 7);
        System.out.println(Arrays.toString(filled)); // [7, 7, 7, 7, 7]

        // เปรียบเทียบว่า 2 array เหมือนกันทุกตำแหน่งหรือไม่ (ต่างจาก == ที่เทียบ reference)
        int[] a = {1, 2, 3};
        int[] b = {1, 2, 3};
        System.out.println(a == b);            // false (คนละ object ใน heap)
        System.out.println(Arrays.equals(a, b)); // true (เปรียบเทียบเนื้อหาทุกตัว)

        // คัดลอก array
        int[] original = {1, 2, 3, 4, 5};
        int[] copy = Arrays.copyOf(original, 3);          // เอา 3 ตัวแรก: [1, 2, 3]
        int[] copyRange = Arrays.copyOfRange(original, 1, 4); // index 1 ถึงก่อน 4: [2, 3, 4]
        System.out.println(Arrays.toString(copy));
        System.out.println(Arrays.toString(copyRange));

        // สำหรับ array 2 มิติ ต้องใช้ deepToString ไม่งั้นจะได้ผลลัพธ์แปลก ๆ
        int[][] matrix = {{1, 2}, {3, 4}};
        System.out.println(Arrays.deepToString(matrix)); // [[1, 2], [3, 4]]
    }
}
```

## 9. การส่ง Array เป็นพารามิเตอร์และคืนค่าจากเมธอด

เนื่องจาก array เป็น reference type การส่ง array ไปยังเมธอดคือการส่ง**ที่อยู่อ้างอิง**
ไป การแก้ไขค่าภายใน array ในเมธอดจะส่งผลกลับไปยัง array ต้นฉบับ:

```java
public class ArrayMethodDemo {
    // รับ array เป็นพารามิเตอร์ และแก้ไขค่าจริงในนั้น
    static void doubleAllValues(int[] arr) {
        for (int i = 0; i < arr.length; i++) {
            arr[i] *= 2;
        }
    }

    // คืนค่าเป็น array ใหม่
    static int[] createSquares(int n) {
        int[] result = new int[n];
        for (int i = 0; i < n; i++) {
            result[i] = i * i;
        }
        return result;
    }

    // หาค่ามากที่สุดใน array
    static int findMax(int[] arr) {
        int max = arr[0];
        for (int value : arr) {
            if (value > max) {
                max = value;
            }
        }
        return max;
    }

    public static void main(String[] args) {
        int[] numbers = {1, 2, 3, 4, 5};
        doubleAllValues(numbers);
        System.out.println(java.util.Arrays.toString(numbers)); // [2, 4, 6, 8, 10]

        int[] squares = createSquares(5);
        System.out.println(java.util.Arrays.toString(squares)); // [0, 1, 4, 9, 16]

        System.out.println("ค่ามากที่สุด: " + findMax(numbers)); // 10
    }
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียนเมธอด `reverseArray(int[] arr)` ที่คืนค่า array ใหม่ที่เรียงลำดับกลับด้าน
โดยไม่แก้ไข array ต้นฉบับ

**เฉลย:**

```java
import java.util.Arrays;

public class Exercise1 {
    static int[] reverseArray(int[] arr) {
        int[] result = new int[arr.length];
        for (int i = 0; i < arr.length; i++) {
            result[i] = arr[arr.length - 1 - i];
        }
        return result;
    }

    public static void main(String[] args) {
        int[] original = {1, 2, 3, 4, 5};
        int[] reversed = reverseArray(original);
        System.out.println(Arrays.toString(original)); // [1, 2, 3, 4, 5] ไม่เปลี่ยน
        System.out.println(Arrays.toString(reversed));  // [5, 4, 3, 2, 1]
    }
}
```

**2)** เขียนโปรแกรมสร้าง array 2 มิติขนาด 5x5 ที่เก็บสูตรคูณแม่ 1-5 (matrix[i][j] =
(i+1) * (j+1)) แล้วพิมพ์ออกมาเป็นตาราง

**เฉลย:**

```java
public class Exercise2 {
    public static void main(String[] args) {
        int[][] multiplicationTable = new int[5][5];
        for (int i = 0; i < 5; i++) {
            for (int j = 0; j < 5; j++) {
                multiplicationTable[i][j] = (i + 1) * (j + 1);
            }
        }

        for (int[] row : multiplicationTable) {
            for (int value : row) {
                System.out.printf("%4d", value);
            }
            System.out.println();
        }
    }
}
```

**3)** เขียนเมธอดหาผลรวมของ array 2 มิติ (jagged array ก็ต้องรองรับได้)

**เฉลย:**

```java
public class Exercise3 {
    static int sum2D(int[][] arr) {
        int total = 0;
        for (int[] row : arr) {
            for (int value : row) {
                total += value;
            }
        }
        return total;
    }

    public static void main(String[] args) {
        int[][] jagged = {{1, 2}, {3, 4, 5}, {6}};
        System.out.println("ผลรวมทั้งหมด: " + sum2D(jagged)); // 21
    }
}
```

### สรุปเนื้อหา Part 7

- Array เก็บข้อมูลชนิดเดียวกันหลายค่า มีขนาดคงที่ตั้งแต่สร้าง เข้าถึงผ่าน index
  เริ่มที่ 0
- `.length` เป็น field ไม่ใช่เมธอด (ต่างจาก `String.length()`)
- เข้าถึง index นอกขอบเขตจะเกิด `ArrayIndexOutOfBoundsException`
- Array ของ object มีค่าเริ่มต้นเป็น `null` ต้องระวัง `NullPointerException`
- Array 2 มิติของ Java เป็น jagged array ได้ (แต่ละแถวความยาวต่างกันได้)
- คลาส `java.util.Arrays` มีเมธอดสำเร็จรูปมากมาย: `sort`, `toString`, `equals`,
  `copyOf`, `binarySearch` ฯลฯ
- Array เป็น reference type การส่งเข้าเมธอดคือส่ง reference แก้ไขในเมธอดจะกระทบ
  ต้นฉบับ

**ต่อไป**: [Part 8 — Methods และการส่งผ่านพารามิเตอร์](./part-008-methods.md)
