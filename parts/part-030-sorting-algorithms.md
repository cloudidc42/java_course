# Part 30: Sorting Algorithms: Bubble, Selection, Insertion, Merge, Quick Sort

> ขั้นตอนที่ 291-300 ของหลักสูตร | ระดับ: กลาง-สูง

## สารบัญ

1. ทำไมต้องเรียนเขียน Sorting Algorithm เอง (ทั้งที่มี `Arrays.sort()`)
2. Bubble Sort
3. Selection Sort
4. Insertion Sort
5. Merge Sort (Divide and Conquer)
6. Quick Sort (Divide and Conquer)
7. เปรียบเทียบ Time/Space Complexity ทั้งหมด
8. Stability ของ Sorting Algorithm
9. `Arrays.sort()` ใช้อัลกอริทึมอะไรจริง ๆ
10. แบบฝึกหัดและสรุป

---

## 1. ทำไมต้องเรียนเขียน Sorting Algorithm เอง

ใน Part 22 เราใช้ `Collections.sort()` และ `Arrays.sort()` ซึ่งเพียงพอสำหรับงาน
จริงเกือบทั้งหมด **แต่การเข้าใจกลไกภายในของ sorting algorithm สำคัญมากสำหรับ**:

1. **การสัมภาษณ์งาน**: เป็นคำถามพื้นฐานที่สุดข้อหนึ่งในการสัมภาษณ์ Software
   Engineer ทั่วโลก
2. **เข้าใจ Time/Space Complexity**: ปูพื้นฐานสำคัญสำหรับวิเคราะห์อัลกอริทึม
   (Part 35 — Big O Notation)
3. **เลือกอัลกอริทึมให้เหมาะกับสถานการณ์**: บางครั้ง built-in sort ไม่เหมาะกับ
   ข้อจำกัดเฉพาะ (เช่น หน่วยความจำจำกัดมาก หรือข้อมูลเกือบเรียงอยู่แล้ว)

## 2. Bubble Sort

**หลักการ**: เปรียบเทียบคู่ element ที่อยู่ติดกัน สลับตำแหน่งถ้าลำดับผิด ทำซ้ำ
จนกว่าจะไม่มีการสลับเกิดขึ้นอีก — ชื่อ "Bubble" มาจากการที่ค่าที่มากที่สุด
"ลอยฟอง" ขึ้นไปอยู่ท้ายสุดทีละรอบ

```java
import java.util.Arrays;

public class BubbleSortDemo {
    static void bubbleSort(int[] arr) {
        int n = arr.length;
        for (int i = 0; i < n - 1; i++) {
            boolean swapped = false;
            for (int j = 0; j < n - 1 - i; j++) { // -i เพราะท้าย ๆ array เรียงเสร็จแล้วจากรอบก่อน
                if (arr[j] > arr[j + 1]) {
                    // สลับตำแหน่ง
                    int temp = arr[j];
                    arr[j] = arr[j + 1];
                    arr[j + 1] = temp;
                    swapped = true;
                }
            }
            if (!swapped) break; // Optimization: ถ้ารอบนี้ไม่มีการสลับเลย แสดงว่าเรียงเสร็จแล้ว หยุดได้
        }
    }

    public static void main(String[] args) {
        int[] arr = {64, 34, 25, 12, 22, 11, 90};
        bubbleSort(arr);
        System.out.println(Arrays.toString(arr)); // [11, 12, 22, 25, 34, 64, 90]
    }
}
```

**Time Complexity**: O(n²) worst/average case, O(n) best case (ข้อมูลเรียงอยู่
แล้ว — ด้วย optimization ข้างบน) | **Space**: O(1) — ไม่ต้องใช้หน่วยความจำเพิ่ม

## 3. Selection Sort

**หลักการ**: ค้นหาค่า**น้อยที่สุด**ในส่วนที่ยังไม่เรียง แล้วสลับไปไว้ตำแหน่งแรก
ของส่วนนั้น ทำซ้ำไปเรื่อย ๆ จนครบ

```java
import java.util.Arrays;

public class SelectionSortDemo {
    static void selectionSort(int[] arr) {
        int n = arr.length;
        for (int i = 0; i < n - 1; i++) {
            int minIndex = i;
            for (int j = i + 1; j < n; j++) { // หาค่าน้อยที่สุดในส่วนที่เหลือ
                if (arr[j] < arr[minIndex]) {
                    minIndex = j;
                }
            }
            // สลับค่าน้อยที่สุดที่เจอไปไว้ตำแหน่ง i
            int temp = arr[minIndex];
            arr[minIndex] = arr[i];
            arr[i] = temp;
        }
    }

    public static void main(String[] args) {
        int[] arr = {64, 25, 12, 22, 11};
        selectionSort(arr);
        System.out.println(Arrays.toString(arr)); // [11, 12, 22, 25, 64]
    }
}
```

**Time Complexity**: O(n²) ในทุกกรณี (ไม่มี best case ที่ดีกว่า เพราะต้องค้นหา
ค่าน้อยที่สุดครบทุกรอบเสมอ) | **Space**: O(1) | **ข้อดี**: จำนวนการ**สลับ**
(swap) น้อยที่สุด (แค่ O(n) ครั้ง) เหมาะเมื่อการเขียนข้อมูลมี cost สูงมาก

## 4. Insertion Sort

**หลักการ**: สร้างส่วนที่เรียงแล้วทีละตัว โดยหยิบตัวถัดไปมา"แทรก"เข้าไปใน
ตำแหน่งที่ถูกต้องของส่วนที่เรียงแล้ว — คล้ายวิธีที่คนเราจัดเรียงไพ่ในมือ

```java
import java.util.Arrays;

public class InsertionSortDemo {
    static void insertionSort(int[] arr) {
        int n = arr.length;
        for (int i = 1; i < n; i++) {
            int key = arr[i];       // ตัวที่จะแทรกเข้าไปในส่วนที่เรียงแล้ว
            int j = i - 1;

            // เลื่อนค่าที่มากกว่า key ไปทางขวาทีละตัว เพื่อเปิดช่องให้ key แทรกเข้าไป
            while (j >= 0 && arr[j] > key) {
                arr[j + 1] = arr[j];
                j--;
            }
            arr[j + 1] = key; // แทรก key ในตำแหน่งที่ถูกต้อง
        }
    }

    public static void main(String[] args) {
        int[] arr = {12, 11, 13, 5, 6};
        insertionSort(arr);
        System.out.println(Arrays.toString(arr)); // [5, 6, 11, 12, 13]
    }
}
```

**Time Complexity**: O(n²) worst/average case, **O(n) best case** (ข้อมูลเรียง
อยู่แล้วหรือใกล้เคียง) | **Space**: O(1) | **ข้อดี**: เร็วมากสำหรับ**ข้อมูล
ขนาดเล็ก**หรือ**ข้อมูลที่เกือบเรียงอยู่แล้ว** — Java ใช้ insertion sort เป็นส่วน
หนึ่งของ hybrid algorithm สำหรับ array ขนาดเล็ก (หัวข้อ 9)

## 5. Merge Sort (Divide and Conquer)

**หลักการ Divide and Conquer**: **แบ่ง** array เป็นครึ่ง ๆ ซ้ำ ๆ จนเหลือ 1
element (เรียงอยู่แล้วโดยธรรมชาติ) แล้ว**รวม (merge)** กลับขึ้นมาโดยเรียงลำดับ
ระหว่างการรวมแต่ละคู่

```java
import java.util.Arrays;

public class MergeSortDemo {
    static void mergeSort(int[] arr, int left, int right) {
        if (left >= right) return; // base case: มี 1 element หรือน้อยกว่า ถือว่าเรียงแล้ว

        int mid = left + (right - left) / 2;
        mergeSort(arr, left, mid);       // แบ่งครึ่งซ้าย
        mergeSort(arr, mid + 1, right);   // แบ่งครึ่งขวา
        merge(arr, left, mid, right);      // รวมสองครึ่งที่เรียงแล้วเข้าด้วยกัน
    }

    static void merge(int[] arr, int left, int mid, int right) {
        int[] temp = new int[right - left + 1]; // buffer ชั่วคราวสำหรับรวมค่า
        int i = left, j = mid + 1, k = 0;

        while (i <= mid && j <= right) { // เทียบทีละคู่ เลือกตัวที่น้อยกว่าใส่ก่อน
            temp[k++] = (arr[i] <= arr[j]) ? arr[i++] : arr[j++];
        }
        while (i <= mid) temp[k++] = arr[i++];   // เก็บส่วนที่เหลือของครึ่งซ้าย (ถ้ามี)
        while (j <= right) temp[k++] = arr[j++]; // เก็บส่วนที่เหลือของครึ่งขวา (ถ้ามี)

        System.arraycopy(temp, 0, arr, left, temp.length); // คัดลอกกลับไปยัง array ต้นฉบับ
    }

    public static void main(String[] args) {
        int[] arr = {38, 27, 43, 3, 9, 82, 10};
        mergeSort(arr, 0, arr.length - 1);
        System.out.println(Arrays.toString(arr)); // [3, 9, 10, 27, 38, 43, 82]
    }
}
```

```
Divide and Conquer visualization:

[38, 27, 43, 3, 9, 82, 10]
         /              \
[38, 27, 43, 3]       [9, 82, 10]
    /       \            /      \
[38, 27]  [43, 3]     [9, 82]  [10]
  /   \     /  \        /  \
[38] [27] [43] [3]   [9] [82]
  \   /     \  /        \  /
[27, 38]  [3, 43]      [9, 82]
    \       /            |
[3, 27, 38, 43]      [9, 10, 82]
         \              /
    [3, 9, 10, 27, 38, 43, 82]   <- merge สุดท้าย
```

**Time Complexity**: **O(n log n)** ในทุกกรณี (คงที่เสมอ ไม่ขึ้นกับข้อมูลนำเข้า)
**Space**: O(n) — ต้องใช้ buffer เพิ่ม (ไม่ใช่ in-place algorithm)

## 6. Quick Sort (Divide and Conquer)

**หลักการ**: เลือก **pivot** (ค่าอ้างอิง) แล้วแบ่ง array เป็น**ค่าน้อยกว่า
pivot**และ**ค่ามากกว่า pivot** ทำซ้ำแบบ recursive กับแต่ละส่วน — ต่างจาก Merge
Sort ที่ "แบ่งก่อนแล้วรวมทีหลัง" Quick Sort "จัดเรียงระหว่างแบ่ง" (partition)

```java
import java.util.Arrays;

public class QuickSortDemo {
    static void quickSort(int[] arr, int low, int high) {
        if (low < high) {
            int pivotIndex = partition(arr, low, high);
            quickSort(arr, low, pivotIndex - 1);  // เรียงส่วนที่น้อยกว่า pivot
            quickSort(arr, pivotIndex + 1, high);  // เรียงส่วนที่มากกว่า pivot
        }
    }

    static int partition(int[] arr, int low, int high) {
        int pivot = arr[high]; // เลือก element สุดท้ายเป็น pivot (วิธีที่นิยมที่สุด)
        int i = low - 1;        // ตำแหน่งของค่าที่น้อยกว่า pivot ล่าสุด

        for (int j = low; j < high; j++) {
            if (arr[j] < pivot) {
                i++;
                // สลับให้ค่าที่น้อยกว่า pivot อยู่ทางซ้ายของ array
                int temp = arr[i]; arr[i] = arr[j]; arr[j] = temp;
            }
        }

        // ย้าย pivot ไปอยู่ตำแหน่งที่ถูกต้อง (ระหว่างค่าน้อยกว่าและมากกว่า)
        int temp = arr[i + 1]; arr[i + 1] = arr[high]; arr[high] = temp;
        return i + 1; // คืนตำแหน่งสุดท้ายของ pivot
    }

    public static void main(String[] args) {
        int[] arr = {10, 7, 8, 9, 1, 5};
        quickSort(arr, 0, arr.length - 1);
        System.out.println(Arrays.toString(arr)); // [1, 5, 7, 8, 9, 10]
    }
}
```

**Time Complexity**: **O(n log n)** average case, **O(n²) worst case** (เกิด
เมื่อเลือก pivot แย่ที่สุดซ้ำ ๆ เช่น array ที่เรียงอยู่แล้วและเลือก pivot ท้าย
สุดเสมอ) | **Space**: O(log n) สำหรับ recursion stack (in-place algorithm — ไม่
ต้องใช้ buffer เพิ่มแบบ Merge Sort)

**การลดความเสี่ยง worst case**: เลือก pivot แบบสุ่ม (random pivot) หรือใช้
median-of-three (เลือกค่ากลางจาก 3 ตัวอย่าง) ช่วยลดโอกาสเกิด worst case ได้มาก
ในทางปฏิบัติ

## 7. เปรียบเทียบ Time/Space Complexity ทั้งหมด

| อัลกอริทึม | Best Case | Average Case | Worst Case | Space | Stable? | In-place? |
|---|---|---|---|---|---|---|
| Bubble Sort | O(n) | O(n²) | O(n²) | O(1) | ✅ | ✅ |
| Selection Sort | O(n²) | O(n²) | O(n²) | O(1) | ❌ | ✅ |
| Insertion Sort | O(n) | O(n²) | O(n²) | O(1) | ✅ | ✅ |
| Merge Sort | O(n log n) | O(n log n) | O(n log n) | O(n) | ✅ | ❌ |
| Quick Sort | O(n log n) | O(n log n) | O(n²) | O(log n) | ❌ | ✅ |

## 8. Stability ของ Sorting Algorithm

**Stable Sort** คือ sorting algorithm ที่**รักษาลำดับสัมพัทธ์**ของ element
ที่มีค่าเท่ากัน (เช่น ถ้า A มาก่อน B ในข้อมูลเดิม และ A.key == B.key แล้ว A ต้อง
ยังมาก่อน B ในผลลัพธ์ที่เรียงแล้วด้วย)

```java
public class StabilityDemo {
    record Person(String name, int age) { }

    public static void main(String[] args) {
        // ก่อนเรียง: Alice(25) มาก่อน Bob(25) แม้อายุเท่ากัน
        java.util.List<Person> people = new java.util.ArrayList<>(java.util.List.of(
            new Person("Alice", 25),
            new Person("Bob", 25),
            new Person("Charlie", 20)
        ));

        // Java's Collections.sort() เป็น stable sort (ใช้ TimSort - หัวข้อ 9)
        people.sort(java.util.Comparator.comparingInt(Person::age));

        people.forEach(System.out::println);
        // Charlie(20), Alice(25), Bob(25) - Alice ยังมาก่อน Bob เหมือนข้อมูลต้นฉบับ (stable!)
    }
}
```

**ความสำคัญของ stability**: มีความสำคัญมากเมื่อเรียงข้อมูลหลายชั้น (multi-key
sort) เช่น เรียงตามแผนกก่อน แล้วค่อยเรียงตามชื่อ (ทบทวนจาก Part 27) — ถ้า sort
ไม่ stable ผลลัพธ์การเรียงชั้นที่สองอาจสลับลำดับข้อมูลที่เรียงไว้แล้วในชั้นแรก
โดยไม่ได้ตั้งใจ

## 9. `Arrays.sort()` ใช้อัลกอริทึมอะไรจริง ๆ

Java's built-in sort **ไม่ได้ใช้อัลกอริทึมเดียวตลอด** แต่เป็น **hybrid
algorithm**:

- **สำหรับ primitive type** (`int[]`, `double[]`, ฯล�ึ): ใช้ **Dual-Pivot
  Quicksort** (ปรับปรุงจาก Quick Sort ธรรมดา ใช้ 2 pivot แทน 1) — เร็วมาก แต่
  **ไม่ stable** (ซึ่งไม่เป็นปัญหา เพราะ primitive value ที่เท่ากันไม่มี "identity"
  ให้ต้องรักษาลำดับ)
- **สำหรับ Object type** (`Integer[]`, `String[]`, `List<T>`): ใช้ **TimSort**
  (ผสมระหว่าง Merge Sort และ Insertion Sort พัฒนาโดย Tim Peters สำหรับ Python
  แล้ว Java เอามาปรับใช้) — **stable** และปรับให้เร็วมากสำหรับข้อมูลที่มี
  ส่วนที่เรียงอยู่แล้วบางส่วน (real-world data มักมีลักษณะนี้)

```java
public class BuiltInSortDemo {
    public static void main(String[] args) {
        int[] primitives = {5, 2, 8, 1, 9};
        Arrays.sort(primitives); // Dual-Pivot Quicksort - เร็วมาก ไม่ stable (ไม่เป็นปัญหาสำหรับ primitive)

        Integer[] objects = {5, 2, 8, 1, 9};
        Arrays.sort(objects); // TimSort - stable, เหมาะกับ object ที่อาจมี "identity" ต่างกันแม้ค่าเท่ากัน

        System.out.println(java.util.Arrays.toString(primitives));
        System.out.println(java.util.Arrays.toString(objects));
    }
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** ทำนายว่า algorithm ใดในบรรดา Bubble, Selection, Insertion Sort จะเรียง
array ที่**เรียงอยู่แล้ว 99%** (มีแค่ 1 ตัวผิดที่) ได้เร็วที่สุด พร้อมอธิบายเหตุผล

**เฉลย**: **Insertion Sort** เร็วที่สุด เพราะ best case ของมันคือ O(n) เมื่อ
ข้อมูลเกือบเรียงอยู่แล้ว (แต่ละ element ที่ถูกแทรกจะเจอตำแหน่งที่ถูกต้องอย่าง
รวดเร็วโดยเลื่อนแค่ไม่กี่ครั้ง) ส่วน Selection Sort ไม่มี best case ที่ดีขึ้น
เลย (ต้องหาค่าน้อยที่สุดครบทุกรอบเสมอ ไม่ว่าข้อมูลจะเรียงแล้วแค่ไหน) และ Bubble
Sort ที่มี optimization (early exit) ก็ทำได้ดีในกรณีนี้เช่นกัน แต่โดยทั่วไป
insertion sort ยังคงมีค่าคงที่ (constant factor) ที่ดีกว่า

**2)** เขียน Quick Sort เวอร์ชันที่เลือก pivot แบบสุ่ม (random pivot) เพื่อลด
โอกาสเกิด worst case

**เฉลย:**

```java
import java.util.Random;

public class Exercise2 {
    static Random random = new Random();

    static void quickSort(int[] arr, int low, int high) {
        if (low < high) {
            int randomIndex = low + random.nextInt(high - low + 1);
            int temp = arr[randomIndex]; arr[randomIndex] = arr[high]; arr[high] = temp; // สุ่ม pivot ไปไว้ท้าย

            int pivotIndex = partition(arr, low, high);
            quickSort(arr, low, pivotIndex - 1);
            quickSort(arr, pivotIndex + 1, high);
        }
    }

    static int partition(int[] arr, int low, int high) {
        int pivot = arr[high];
        int i = low - 1;
        for (int j = low; j < high; j++) {
            if (arr[j] < pivot) {
                i++;
                int temp = arr[i]; arr[i] = arr[j]; arr[j] = temp;
            }
        }
        int temp = arr[i + 1]; arr[i + 1] = arr[high]; arr[high] = temp;
        return i + 1;
    }
}
```

**3)** อธิบายว่าทำไม Merge Sort เหมาะกับการเรียง **linked list** มากกว่า Quick
Sort

**เฉลย**: Merge Sort เข้าถึงข้อมูลแบบ**ลำดับ (sequential access)** เป็นหลัก
(อ่านทีละตัวจากหัวไปหาง) ซึ่งเหมาะกับ linked list ที่ไม่มี random access
(เข้าถึง index ตรง ๆ ไม่ได้ O(1) แบบ array — ทบทวนจาก Part 22) ในขณะที่ Quick
Sort ต้องการ**random access**บ่อยมากเพื่อสลับตำแหน่ง (swap) ระหว่างการ
partition ซึ่งบน linked list การเข้าถึงแบบสุ่มมี cost สูง (O(n) ต่อครั้ง)
ทำให้ Quick Sort ไม่มีประสิทธิภาพเมื่อใช้กับ linked list

### สรุปเนื้อหา Part 30

- Bubble/Selection/Insertion Sort เข้าใจง่าย แต่ช้า O(n²) — เหมาะกับข้อมูลขนาด
  เล็กหรือการศึกษาเท่านั้น
- Merge Sort ใช้ Divide and Conquer ได้ O(n log n) เสมอ แต่ต้องใช้ O(n) space
  เพิ่ม
- Quick Sort เร็วในทางปฏิบัติ O(n log n) average แต่มี worst case O(n²) — แก้ได้
  ด้วย random pivot
- Stable sort รักษาลำดับสัมพัทธ์ของค่าที่เท่ากัน สำคัญมากสำหรับ multi-key sorting
- Java ใช้ Dual-Pivot Quicksort สำหรับ primitive array และ TimSort (stable)
  สำหรับ object array/List

**ต่อไป**: [Part 31 — Searching Algorithms และ Stack/Queue แบบ Manual](./part-031-searching-algorithms.md)
