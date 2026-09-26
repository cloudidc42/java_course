# Part 35: Big O Notation, Time/Space Complexity

> ขั้นตอนที่ 341-350 ของหลักสูตร | ระดับ: กลาง-สูง (จบหมวดโครงสร้างข้อมูลและอัลกอริทึม)

## สารบัญ

1. ทำไมต้องวิเคราะห์ประสิทธิภาพของอัลกอริทึม
2. Big O Notation คืออะไร
3. Complexity Classes ที่พบบ่อยที่สุด (เรียงจากเร็วไปช้า)
4. วิธีวิเคราะห์ Time Complexity จากโค้ดจริง
5. Best, Average, Worst Case
6. Space Complexity
7. Big O ของ Collections และ Operations ที่ใช้บ่อย (ตารางสรุปทั้งหลักสูตร)
8. Amortized Complexity
9. ข้อควรระวัง: Big O ไม่ใช่ทุกอย่าง
10. แบบฝึกหัดและสรุปหมวดโครงสร้างข้อมูลและอัลกอริทึม

---

## 1. ทำไมต้องวิเคราะห์ประสิทธิภาพของอัลกอริทึม

ตลอด Part 22-34 เราใช้คำว่า "O(1)", "O(n)", "O(log n)" มาโดยตลอด Part นี้จะ
อธิบายแนวคิดนี้อย่างเป็นระบบ — **Big O Notation** คือภาษากลางที่ใช้**บอก
ประสิทธิภาพของอัลกอริทึม**โดยไม่ขึ้นกับความเร็วของเครื่อง CPU หรือภาษาโปรแกรม
ที่ใช้เขียน ทำให้เปรียบเทียบอัลกอริทึมสองตัวได้อย่างเป็นกลาง

```java
public class WhyAnalyzeDemo {
    // เมธอดทั้งสองนี้ทำงานเหมือนกัน (หาว่ามี duplicate หรือไม่) แต่ประสิทธิภาพต่างกันมาก

    static boolean hasDuplicateSlow(int[] arr) { // O(n²) - ช้า
        for (int i = 0; i < arr.length; i++) {
            for (int j = i + 1; j < arr.length; j++) {
                if (arr[i] == arr[j]) return true;
            }
        }
        return false;
    }

    static boolean hasDuplicateFast(int[] arr) { // O(n) - เร็วกว่ามาก
        java.util.Set<Integer> seen = new java.util.HashSet<>();
        for (int num : arr) {
            if (!seen.add(num)) return true; // add() คืน false ถ้ามีอยู่แล้ว (ทบทวนจาก Part 23)
        }
        return false;
    }

    public static void main(String[] args) {
        // สำหรับ array ขนาด 100 ตัว: ทั้งสองเร็วพอ ๆ กันในทางปฏิบัติ
        // สำหรับ array ขนาด 1,000,000 ตัว: O(n²) ใช้เวลาเป็นชั่วโมง, O(n) ใช้เวลาไม่ถึงวินาที!
    }
}
```

## 2. Big O Notation คืออะไร

**Big O** อธิบาย**อัตราการเติบโต**ของเวลา (หรือหน่วยความจำ) ที่อัลกอริทึมใช้
เมื่อขนาดข้อมูลนำเข้า (`n`) เพิ่มขึ้น — **ไม่ใช่เวลาที่ใช้จริงเป็นวินาที**
แต่เป็นแนวโน้มว่า "ถ้าข้อมูลเพิ่มเป็น 2 เท่า เวลาจะเพิ่มขึ้นแบบไหน"

**กฎการเขียน Big O**:
1. **ตัดค่าคงที่ทิ้ง**: O(3n) เขียนเป็น O(n), O(500) เขียนเป็น O(1)
2. **เก็บเฉพาะเทอมที่เติบโตเร็วที่สุด**: O(n² + n) เขียนเป็น O(n²) เท่านั้น
   (เมื่อ n มีค่ามาก เทอม n² จะครอบงำ n จนไม่มีผลกระทบอย่างมีนัยสำคัญ)

```java
public class BigOExampleDemo {
    static void example(int[] arr) {
        System.out.println(arr[0]);          // O(1) - ทำครั้งเดียว

        for (int x : arr) {                    // O(n) - loop ครั้งเดียวตามขนาด array
            System.out.println(x);
        }

        for (int x : arr) {                    // O(n) อีกครั้ง
            for (int y : arr) {                  // ซ้อนกัน -> O(n) * O(n) = O(n²)
                System.out.println(x + y);
            }
        }
        // Total: O(1) + O(n) + O(n²) = O(n²) (เก็บแค่เทอมที่โตเร็วสุด ตัดที่เหลือทิ้ง)
    }
}
```

## 3. Complexity Classes ที่พบบ่อยที่สุด (เรียงจากเร็วไปช้า)

| Big O | ชื่อ | ตัวอย่าง | n=10 | n=1,000 | n=1,000,000 |
|---|---|---|---|---|---|
| O(1) | Constant | เข้าถึง array ด้วย index | 1 | 1 | 1 |
| O(log n) | Logarithmic | Binary Search | ~3 | ~10 | ~20 |
| O(n) | Linear | Linear Search, วน loop ครั้งเดียว | 10 | 1,000 | 1,000,000 |
| O(n log n) | Linearithmic | Merge Sort, Quick Sort (average) | ~33 | ~10,000 | ~20,000,000 |
| O(n²) | Quadratic | Bubble/Selection/Insertion Sort, nested loop | 100 | 1,000,000 | 10^12 |
| O(2^n) | Exponential | Naive Fibonacci recursion, generate all subsets | 1,024 | 10^301 | (คำนวณไม่จบในชีวิตนี้) |
| O(n!) | Factorial | Generate all permutations, brute-force TSP | 3,628,800 | (มหาศาลเกินคำนวณ) | - |

```
กราฟเปรียบเทียบอัตราการเติบโต (ยิ่งชันมาก ยิ่งแย่เมื่อ n โต):

เวลา
 ^
 |                                          O(n²)
 |                                    /
 |                              /  O(n log n)
 |                        /  ⟋
 |                  /  ⟋  O(n)
 |            /  ⟋
 |      ⟋⟋  O(log n)
 |___⟋________________________________  O(1)
 └──────────────────────────────────────> n (ขนาดข้อมูล)
```

## 4. วิธีวิเคราะห์ Time Complexity จากโค้ดจริง

```java
public class AnalyzingCodeDemo {
    // O(1): ไม่มี loop เลย ทำงานคงที่ไม่ว่า n จะเป็นเท่าไร
    static int constantTime(int[] arr) {
        return arr[0] + arr[arr.length - 1];
    }

    // O(n): loop เดียวตามขนาด input
    static int linearTime(int[] arr) {
        int sum = 0;
        for (int x : arr) sum += x;
        return sum;
    }

    // O(n): แม้มี 2 loop แต่ "แยกกัน" ไม่ซ้อนกัน -> บวกกัน O(n) + O(n) = O(n)
    static void twoSeparateLoops(int[] arr) {
        for (int x : arr) System.out.print(x);
        for (int x : arr) System.out.print(x);
    }

    // O(n²): loop ซ้อนกัน โดยตัวในขึ้นอยู่กับขนาดเดียวกับตัวนอก
    static void nestedLoops(int[] arr) {
        for (int x : arr) {
            for (int y : arr) {
                System.out.print(x + y);
            }
        }
    }

    // O(n * m): loop ซ้อนกันแต่ขนาดต่างกัน (ไม่ใช่ n² เพราะ 2 loop คนละขนาด)
    static void differentSizedLoops(int[] arr1, int[] arr2) {
        for (int x : arr1) {          // O(n) โดย n = arr1.length
            for (int y : arr2) {        // O(m) โดย m = arr2.length
                System.out.print(x + y);
            }
        }
        // Total: O(n * m)
    }

    // O(log n): ขนาดปัญหาลดลงครึ่งหนึ่งทุกรอบ (เหมือน binary search)
    static void logarithmicTime(int n) {
        while (n > 1) {
            n = n / 2; // ลดลงครึ่งหนึ่งทุกรอบ
        }
    }
}
```

**หลักการจำง่าย ๆ**: 
- **Loop เดี่ยว** ตามขนาด input = O(n)
- **Loop ซ้อนกัน 2 ชั้น** (ขนาดเดียวกัน) = O(n²)
- **ลดขนาดปัญหาลงครึ่งหนึ่งทุกรอบ** = O(log n)
- **Recursion ที่แตกเป็น 2 กิ่งทุกครั้ง** (ไม่มี memoization) = O(2^n)

## 5. Best, Average, Worst Case

Big O มักหมายถึง **worst case** เป็นค่าเริ่มต้น แต่บางครั้งต้องแยกวิเคราะห์ทั้ง
3 กรณี (ทบทวนจาก Quick Sort ใน Part 30):

```java
public class CaseAnalysisDemo {
    static int linearSearch(int[] arr, int target) {
        for (int i = 0; i < arr.length; i++) {
            if (arr[i] == target) return i;
        }
        return -1;
    }

    public static void main(String[] args) {
        int[] arr = {1, 2, 3, 4, 5};

        // Best case: O(1) - เจอตัวแรกเลย
        linearSearch(arr, 1);

        // Worst case: O(n) - ไม่เจอเลย ต้องตรวจทุกตัว หรือเจอตัวสุดท้าย
        linearSearch(arr, 99);
        linearSearch(arr, 5);

        // Average case: O(n) - โดยเฉลี่ยต้องตรวจครึ่งหนึ่งของ array
    }
}
```

## 6. Space Complexity

**Space Complexity** วัด**หน่วยความจำเพิ่มเติม**ที่อัลกอริทึมต้องใช้ (ไม่นับ
input เดิม) ตามขนาดของ input — สำคัญพอ ๆ กับ Time Complexity โดยเฉพาะเมื่อทำงาน
กับข้อมูลขนาดใหญ่มากหรือระบบที่มีหน่วยความจำจำกัด

```java
public class SpaceComplexityDemo {
    // O(1) space: ใช้ตัวแปรจำนวนคงที่ ไม่ขึ้นกับขนาด input
    static int sumInPlace(int[] arr) {
        int sum = 0; // ตัวแปรเดียว ไม่ว่า array จะใหญ่แค่ไหน
        for (int x : arr) sum += x;
        return sum;
    }

    // O(n) space: สร้าง array/collection ใหม่ตามขนาด input
    static int[] doubleAll(int[] arr) {
        int[] result = new int[arr.length]; // ขนาดโตตาม arr.length
        for (int i = 0; i < arr.length; i++) result[i] = arr[i] * 2;
        return result;
    }

    // O(n) space จาก recursion stack (ทบทวนจาก Part 29)
    static int factorial(int n) {
        if (n <= 1) return 1;
        return n * factorial(n - 1); // แต่ละ call ใช้ stack frame เพิ่ม -> O(n) space
    }
}
```

**Time-Space Tradeoff**: บ่อยครั้งเราแลกหน่วยความจำเพิ่มเพื่อความเร็วที่ดีขึ้น
(เช่น Memoization ใน Part 29 ใช้ O(n) space เพิ่มเพื่อลด time จาก O(2^n) เป็น
O(n)) — การเลือกว่าจะแลกแบบไหนขึ้นกับข้อจำกัดของระบบจริง

## 7. Big O ของ Collections และ Operations ที่ใช้บ่อย (ตารางสรุปทั้งหลักสูตร)

| โครงสร้างข้อมูล | Access | Search | Insert | Delete |
|---|---|---|---|---|
| Array | O(1) | O(n) | O(n)* | O(n)* |
| `ArrayList` | O(1) | O(n) | O(1) ท้าย / O(n) กลาง | O(1) ท้าย / O(n) กลาง |
| `LinkedList` | O(n) | O(n) | O(1) หัว-ท้าย | O(1) หัว-ท้าย |
| `HashMap`/`HashSet` | - | O(1) เฉลี่ย | O(1) เฉลี่ย | O(1) เฉลี่ย |
| `TreeMap`/`TreeSet` | - | O(log n) | O(log n) | O(log n) |
| `ArrayDeque` (Stack/Queue) | - | O(n) | O(1) | O(1) |
| `PriorityQueue` | - | O(n) | O(log n) | O(log n) (poll) |
| Binary Search Tree (balanced) | - | O(log n) | O(log n) | O(log n) |
| Binary Search Tree (unbalanced) | - | O(n) worst | O(n) worst | O(n) worst |

*Array มีขนาดคงที่ ไม่สามารถ insert/delete แบบขยายขนาดได้จริง (ต้องสร้าง array
ใหม่)

## 8. Amortized Complexity

**Amortized Analysis** คือการวิเคราะห์**ค่าเฉลี่ยในระยะยาว**ของ operation ที่
บางครั้งช้า (worst case) แต่ส่วนใหญ่เร็ว — ตัวอย่างคลาสสิกคือ `ArrayList.add()`

```java
public class AmortizedAnalysisDemo {
    public static void main(String[] args) {
        java.util.List<Integer> list = new java.util.ArrayList<>();

        // ส่วนใหญ่ add() คือ O(1) เพราะมีที่ว่างเหลืออยู่ใน internal array
        // แต่บางครั้ง (เมื่อ array เต็ม) ต้อง resize (สร้าง array ใหม่ใหญ่กว่า + คัดลอกทุกตัว) -> O(n)
        // Amortized (เฉลี่ยในระยะยาว): ยังคงเป็น O(1) ต่อ operation

        for (int i = 0; i < 1000; i++) {
            list.add(i); // ดูจากภายนอกเหมือน O(1) เสมอ แม้บางครั้งจะมี resize เกิดขึ้นแอบแฝง
        }
    }
}
```

**คำอธิบายทางคณิตศาสตร์แบบง่าย**: เมื่อ `ArrayList` resize จะขยายเป็น**1.5-2
เท่า**ของขนาดเดิม (ไม่ใช่ +1) ทำให้จำนวนครั้งที่ resize เกิดขึ้นทั้งหมดเป็น
O(log n) เท่านั้น (ไม่ใช่ O(n)) เมื่อเฉลี่ย cost ของการ resize ทั้งหมดไปตลอด
การ `add()` ทุกครั้ง จะได้ amortized cost เพียง O(1) ต่อครั้ง

## 9. ข้อควรระวัง: Big O ไม่ใช่ทุกอย่าง

Big O บอก**แนวโน้ม**เมื่อ n มีค่ามาก ๆ แต่**ไม่ได้บอกความเร็วจริงเสมอไปสำหรับ n
ค่าน้อย** — บางครั้งอัลกอริทึมที่ Big O แย่กว่าอาจเร็วกว่าจริงในทางปฏิบัติ
เพราะมี **constant factor** ที่เล็กกว่า

```java
public class ConstantFactorDemo {
    // O(n) แต่มี constant factor สูง (ทำงานซับซ้อนต่อรอบ)
    static void complexLinear(int[] arr) {
        for (int x : arr) {
            // สมมติว่ามีการคำนวณที่ซับซ้อนมาก ใช้เวลานานต่อรอบ
        }
    }

    // O(n log n) แต่ constant factor ต่ำมาก (เช่น TimSort ที่ optimize มาอย่างดี)
    // สำหรับ n เล็ก ๆ (เช่น n < 50) อาจเร็วกว่า O(n) ที่มี constant factor สูงได้จริง
}
```

**นี่คือเหตุผลที่ Java's `Arrays.sort()` (ทบทวนจาก Part 30) ใช้ Insertion Sort
สำหรับ array ขนาดเล็ก** (แม้ Insertion Sort เป็น O(n²) ในทางทฤษฎี) เพราะสำหรับ
ข้อมูลจำนวนน้อย constant factor ที่ต่ำของ Insertion Sort ทำให้เร็วกว่า Merge/
Quick Sort ในทางปฏิบัติจริง — Big O เป็นเครื่องมือวิเคราะห์ที่ทรงพลัง แต่ต้อง
ใช้ร่วมกับการวัดผลจริง (benchmarking) เสมอในการตัดสินใจเชิงวิศวกรรมที่สำคัญ

## 10. แบบฝึกหัดและสรุปหมวดโครงสร้างข้อมูลและอัลกอริทึม

### แบบฝึกหัด

**1)** วิเคราะห์ Time Complexity ของโค้ดนี้:

```java
static void mystery(int[] arr) {
    for (int i = 0; i < arr.length; i++) {
        for (int j = i; j < arr.length; j++) {
            System.out.println(arr[i] + arr[j]);
        }
    }
}
```

**เฉลย**: **O(n²)** — แม้ loop ชั้นในเริ่มที่ `i` (ไม่ใช่ 0) ทำให้จำนวนรอบทั้งหมด
ลดลงประมาณครึ่งหนึ่งเมื่อเทียบกับ nested loop ปกติ แต่ Big O ตัดค่าคงที่ทิ้ง
(ทบทวนกฎข้อ 1 ในหัวข้อ 2) ดังนั้นยังคงเป็น O(n²) เหมือนเดิม

**2)** เปรียบเทียบ Space Complexity ระหว่าง Bottom-up Fibonacci ที่เก็บทั้ง
array (Part 29 หัวข้อ 5) กับเวอร์ชันที่ optimize ใช้แค่ 2 ตัวแปร (หัวข้อ 5
ตัวอย่างสุดท้าย)

**เฉลย**: เวอร์ชันที่เก็บทั้ง array มี Space Complexity O(n) เพราะขนาด array
โตตาม n ส่วนเวอร์ชันที่ใช้แค่ 2 ตัวแปร (`prev1`, `prev2`) มี Space Complexity
O(1) เพราะใช้หน่วยความจำคงที่ไม่ว่า n จะมีค่าเท่าไร — ทั้งสองมี Time Complexity
เท่ากัน (O(n)) แต่ Space Complexity ต่างกันอย่างมีนัยสำคัญ

**3)** อธิบายว่าทำไม `HashMap.get()` เป็น O(1) "โดยเฉลี่ย" แต่ไม่ใช่ O(1)
เสมอไปในทุกกรณี

**เฉลย**: `HashMap.get()` เป็น O(1) โดยเฉลี่ยเพราะ hash function กระจายข้อมูล
ไปยัง bucket ต่าง ๆ อย่างสม่ำเสมอ ทำให้แต่ละ bucket มีจำนวน entry น้อย (ทบทวน
จาก Part 24) แต่ในกรณีที่เลวร้ายที่สุด (worst case) ถ้า `hashCode()` ของทุก key
ชนกันหมด (hash collision) ทุก entry จะกระจุกอยู่ใน bucket เดียว ทำให้การค้นหา
กลายเป็น O(n) (หรือ O(log n) ถ้า Java ใช้ red-black tree แทน linked list เมื่อ
bucket มีขนาดใหญ่มาก ซึ่งเป็น optimization ที่เพิ่มมาตั้งแต่ Java 8)

### สรุปเนื้อหา Part 35 และหมวดโครงสร้างข้อมูลและอัลกอริทึม (Part 21-35)

- Big O อธิบายอัตราการเติบโตของเวลา/หน่วยความจำเมื่อขนาดข้อมูลเพิ่มขึ้น ไม่ใช่
  เวลาที่ใช้จริง
- Complexity classes เรียงจากเร็วไปช้า: O(1) < O(log n) < O(n) < O(n log n) <
  O(n²) < O(2^n) < O(n!)
- Loop เดี่ยว = O(n), loop ซ้อน (ขนาดเดียวกัน) = O(n²), ลดขนาดครึ่งหนึ่งทุกรอบ
  = O(log n)
- Space Complexity สำคัญเท่า Time Complexity โดยเฉพาะกับข้อมูลขนาดใหญ่
- Amortized complexity คือค่าเฉลี่ยระยะยาว (เช่น ArrayList.add() amortized O(1)
  แม้บางครั้งต้อง resize เป็น O(n))
- Big O เป็นเครื่องมือวิเคราะห์ ไม่ใช่คำตอบสุดท้าย — ต้องพิจารณา constant
  factor และวัดผลจริงประกอบด้วยเสมอ

**จบหมวดที่ 2: โครงสร้างข้อมูลและอัลกอริทึม (Part 21-35) อย่างสมบูรณ์!**
ครอบคลุม Collections Framework ครบทุกส่วน, Generics, Recursion ขั้นสูง,
Sorting/Searching, Linked List, Tree, Graph, และ Big O Notation

**ต่อไป**: [Part 36 — File I/O เบื้องต้น: File, FileReader/Writer, BufferedReader/Writer](./part-036-file-io-basics.md)
(เริ่มหมวดที่ 3: Java ระดับกลางถึงขั้นสูง)
