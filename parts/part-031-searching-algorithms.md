# Part 31: Searching Algorithms และการสร้าง Stack/Queue เอง

> ขั้นตอนที่ 301-310 ของหลักสูตร | ระดับ: กลาง-สูง

## สารบัญ

1. Linear Search
2. Binary Search
3. Binary Search แบบ Recursive
4. ข้อผิดพลาดคลาสสิกใน Binary Search
5. `Arrays.binarySearch()` และ `Collections.binarySearch()`
6. การสร้าง Stack เองด้วย Array
7. การสร้าง Queue เองด้วย Array (Circular Buffer)
8. เปรียบเทียบ Linear vs Binary Search
9. Interpolation Search (เทคนิคเสริม)
10. แบบฝึกหัดและสรุป

---

## 1. Linear Search

**Linear Search** คือการค้นหาแบบ**ไล่ตรวจทีละตัว**ตั้งแต่ต้นจนจบ — ใช้ได้กับ
ข้อมูล**ไม่จำเป็นต้องเรียงลำดับ** แต่ช้าที่สุดในกลุ่ม searching algorithm พื้นฐาน

```java
public class LinearSearchDemo {
    static int linearSearch(int[] arr, int target) {
        for (int i = 0; i < arr.length; i++) {
            if (arr[i] == target) {
                return i; // เจอแล้ว คืน index ทันที
            }
        }
        return -1; // ไม่เจอ
    }

    public static void main(String[] args) {
        int[] numbers = {64, 34, 25, 12, 22, 11, 90};
        System.out.println(linearSearch(numbers, 22)); // 4
        System.out.println(linearSearch(numbers, 99)); // -1 (ไม่พบ)
    }
}
```

**Time Complexity**: O(n) — ในกรณีเลวร้ายสุดต้องตรวจทุก element

## 2. Binary Search

**Binary Search** ค้นหาข้อมูลที่**เรียงลำดับแล้วเท่านั้น** โดยเปรียบเทียบกับ
ค่ากลาง (middle) แล้ว**ตัดครึ่งที่ไม่มีทางเป็นคำตอบทิ้งไป**ทุกครั้ง ทำให้เร็ว
กว่า Linear Search มาก

```java
public class BinarySearchDemo {
    static int binarySearch(int[] arr, int target) {
        int left = 0, right = arr.length - 1;

        while (left <= right) {
            int mid = left + (right - left) / 2; // หาจุดกึ่งกลาง

            if (arr[mid] == target) {
                return mid; // เจอแล้ว
            } else if (arr[mid] < target) {
                left = mid + 1; // เป้าหมายต้องอยู่ครึ่งขวา ตัดครึ่งซ้ายทิ้ง
            } else {
                right = mid - 1; // เป้าหมายต้องอยู่ครึ่งซ้าย ตัดครึ่งขวาทิ้ง
            }
        }
        return -1; // ไม่เจอ
    }

    public static void main(String[] args) {
        int[] sortedArray = {11, 12, 22, 25, 34, 64, 90}; // ต้องเรียงลำดับก่อนเสมอ!
        System.out.println(binarySearch(sortedArray, 25)); // 3
        System.out.println(binarySearch(sortedArray, 99)); // -1
    }
}
```

```
Binary Search visualization: ค้นหา 25 ใน [11, 12, 22, 25, 34, 64, 90]

รอบ 1: left=0, right=6, mid=3 -> arr[3]=25 == target -> เจอทันที! คืน index 3

ตัวอย่างที่ต้องค้นหลายรอบ: ค้นหา 90

รอบ 1: left=0, right=6, mid=3 -> arr[3]=25 < 90 -> ตัดครึ่งซ้ายทิ้ง, left=4
รอบ 2: left=4, right=6, mid=5 -> arr[5]=64 < 90 -> ตัดครึ่งซ้ายทิ้ง, left=6
รอบ 3: left=6, right=6, mid=6 -> arr[6]=90 == target -> เจอ! คืน index 6

จาก 7 elements เหลือให้ตรวจสอบแค่ 3 รอบ (log2(7) ≈ 2.8) แทนที่จะตรวจทีละตัว 7 รอบ
```

**Time Complexity**: **O(log n)** — เร็วกว่า Linear Search มากสำหรับข้อมูลขนาด
ใหญ่ (array 1 ล้านตัว: linear search ใช้สูงสุด 1,000,000 รอบ แต่ binary search
ใช้สูงสุดแค่ ~20 รอบเท่านั้น!)

## 3. Binary Search แบบ Recursive

```java
public class RecursiveBinarySearchDemo {
    static int binarySearch(int[] arr, int target, int left, int right) {
        if (left > right) return -1; // base case: ค้นหมดแล้วไม่เจอ

        int mid = left + (right - left) / 2;

        if (arr[mid] == target) {
            return mid;
        } else if (arr[mid] < target) {
            return binarySearch(arr, target, mid + 1, right); // ค้นครึ่งขวา
        } else {
            return binarySearch(arr, target, left, mid - 1); // ค้นครึ่งซ้าย
        }
    }

    public static void main(String[] args) {
        int[] sortedArray = {11, 12, 22, 25, 34, 64, 90};
        System.out.println(binarySearch(sortedArray, 34, 0, sortedArray.length - 1)); // 4
    }
}
```

**หมายเหตุ**: recursive version มี recursion depth เป็น O(log n) เท่านั้น จึง
ไม่เสี่ยง `StackOverflowError` แม้กับข้อมูลขนาดใหญ่มาก (ทบทวนแนวคิด recursion
จาก Part 29)

## 4. ข้อผิดพลาดคลาสสิกใน Binary Search

**"Nearly All Binary Searches are Broken"** เป็นบทความชื่อดังของ Joshua Bloch
(ผู้เขียน Effective Java) ที่ชี้ให้เห็นข้อผิดพลาดที่พบบ่อยมาก:

```java
public class BinarySearchBugDemo {
    // เวอร์ชันที่มี bug: overflow เมื่อ array มีขนาดใหญ่มาก (ใกล้ Integer.MAX_VALUE)
    static int buggyMid(int left, int right) {
        return (left + right) / 2; // ถ้า left + right เกิน Integer.MAX_VALUE จะเกิด overflow!
    }

    // เวอร์ชันที่ถูกต้อง: ป้องกัน overflow
    static int correctMid(int left, int right) {
        return left + (right - left) / 2; // ไม่มีทาง overflow เพราะไม่บวกค่าที่มากทั้งคู่เข้าด้วยกัน
    }

    public static void main(String[] args) {
        int left = 1_500_000_000, right = 2_000_000_000;
        System.out.println(buggyMid(left, right));   // ค่าติดลบ! (overflow เกิดขึ้นจริง)
        System.out.println(correctMid(left, right)); // ค่าถูกต้อง
    }
}
```

**บทเรียนสำคัญ**: แม้อัลกอริทึมพื้นฐานที่ดูง่าย ก็อาจมี bug ที่ซ่อนอยู่ในกรณี
ขอบ (edge case) ที่ไม่คาดคิด — เป็นเหตุผลสำคัญที่ควรเขียน unit test ครอบคลุม
(Part 58) แทนพึ่งพาความรอบคอบของโปรแกรมเมอร์อย่างเดียว

## 5. `Arrays.binarySearch()` และ `Collections.binarySearch()`

ในทางปฏิบัติ ไม่ต้องเขียน binary search เองเสมอไป Java มีให้ใช้แล้ว:

```java
import java.util.Arrays;
import java.util.Collections;
import java.util.List;

public class BuiltInBinarySearchDemo {
    public static void main(String[] args) {
        int[] sortedArray = {11, 12, 22, 25, 34, 64, 90};

        int index = Arrays.binarySearch(sortedArray, 25);
        System.out.println(index); // 3

        // ถ้าไม่เจอ จะคืนค่าติดลบที่บอกตำแหน่งที่ "ควรจะแทรก" (insertion point)
        int notFoundIndex = Arrays.binarySearch(sortedArray, 20);
        System.out.println(notFoundIndex); // ค่าติดลบ เช่น -3 หมายถึงควรแทรกที่ index 2 (คำนวณจาก -(insertionPoint)-1)

        List<Integer> sortedList = List.of(11, 12, 22, 25, 34, 64, 90);
        int listIndex = Collections.binarySearch(sortedList, 64);
        System.out.println(listIndex); // 5
    }
}
```

**คำเตือนสำคัญ**: `binarySearch()` ให้ผลลัพธ์ที่**ไม่นิยาม (undefined)** ถ้า
array/list **ไม่ได้เรียงลำดับก่อน** — ต้องมั่นใจว่าข้อมูลเรียงแล้วเสมอก่อนเรียก

## 6. การสร้าง Stack เองด้วย Array

การเข้าใจว่า Stack (LIFO — ทบทวนจาก Part 25) implement ด้วย array ได้อย่างไร
ช่วยเสริมความเข้าใจพื้นฐานของโครงสร้างข้อมูล:

```java
public class ArrayStack {
    private int[] data;
    private int top; // index ของ element บนสุด (-1 หมายถึง stack ว่าง)
    private int capacity;

    public ArrayStack(int capacity) {
        this.capacity = capacity;
        this.data = new int[capacity];
        this.top = -1;
    }

    public void push(int value) {
        if (isFull()) {
            throw new IllegalStateException("Stack เต็มแล้ว (capacity=" + capacity + ")");
        }
        data[++top] = value; // เพิ่ม top ก่อน แล้วเก็บค่า
    }

    public int pop() {
        if (isEmpty()) {
            throw new IllegalStateException("Stack ว่างเปล่า");
        }
        return data[top--]; // คืนค่าปัจจุบันก่อน แล้วลด top
    }

    public int peek() {
        if (isEmpty()) {
            throw new IllegalStateException("Stack ว่างเปล่า");
        }
        return data[top];
    }

    public boolean isEmpty() { return top == -1; }
    public boolean isFull() { return top == capacity - 1; }
    public int size() { return top + 1; }
}
```

```java
public class ArrayStackDemo {
    public static void main(String[] args) {
        ArrayStack stack = new ArrayStack(5);
        stack.push(10);
        stack.push(20);
        stack.push(30);

        System.out.println(stack.peek()); // 30
        System.out.println(stack.pop());   // 30
        System.out.println(stack.size());   // 2
    }
}
```

## 7. การสร้าง Queue เองด้วย Array (Circular Buffer)

การสร้าง Queue (FIFO) ด้วย array **ตรงไปตรงมา** (naive) จะเสีย performance
เพราะต้องเลื่อน element ทุกตัวเมื่อ dequeue — วิธีที่มีประสิทธิภาพคือ
**Circular Buffer** (วนกลับมาใช้พื้นที่ว่างที่หัว array อีกครั้ง)

```java
public class CircularQueue {
    private int[] data;
    private int front, rear, size, capacity;

    public CircularQueue(int capacity) {
        this.capacity = capacity;
        this.data = new int[capacity];
        this.front = 0;
        this.rear = -1;
        this.size = 0;
    }

    public void enqueue(int value) {
        if (isFull()) {
            throw new IllegalStateException("Queue เต็มแล้ว");
        }
        rear = (rear + 1) % capacity; // วนกลับไปที่ index 0 เมื่อถึงปลาย array (นี่คือ "circular")
        data[rear] = value;
        size++;
    }

    public int dequeue() {
        if (isEmpty()) {
            throw new IllegalStateException("Queue ว่างเปล่า");
        }
        int value = data[front];
        front = (front + 1) % capacity; // วนกลับเช่นกัน
        size--;
        return value;
    }

    public boolean isEmpty() { return size == 0; }
    public boolean isFull() { return size == capacity; }
}
```

```java
public class CircularQueueDemo {
    public static void main(String[] args) {
        CircularQueue queue = new CircularQueue(3);
        queue.enqueue(1);
        queue.enqueue(2);
        queue.enqueue(3);

        System.out.println(queue.dequeue()); // 1
        queue.enqueue(4); // ใส่ตัวใหม่ได้ เพราะ dequeue ปล่อยพื้นที่ให้แล้ว (วนกลับมาใช้ index 0)

        System.out.println(queue.dequeue()); // 2
        System.out.println(queue.dequeue()); // 3
        System.out.println(queue.dequeue()); // 4
    }
}
```

```
Circular Buffer visualization (capacity=3):

เริ่มต้น:        [_, _, _]  front=0, rear=-1
enqueue(1):     [1, _, _]  front=0, rear=0
enqueue(2):     [1, 2, _]  front=0, rear=1
enqueue(3):     [1, 2, 3]  front=0, rear=2 (เต็มแล้ว)
dequeue() -> 1: [_, 2, 3]  front=1, rear=2
enqueue(4):     [4, 2, 3]  front=1, rear=0 (!) rear วนกลับมาที่ index 0
                            เพราะ (2+1) % 3 = 0 - นี่คือหัวใจของ circular buffer
```

## 8. เปรียบเทียบ Linear vs Binary Search

| ลักษณะ | Linear Search | Binary Search |
|---|---|---|
| ต้องเรียงลำดับก่อน | ❌ ไม่จำเป็น | ✅ จำเป็นเสมอ |
| Time Complexity | O(n) | O(log n) |
| ใช้กับ Linked List | ✅ ใช้ได้ดี | ❌ ไม่มีประโยชน์ (ต้อง random access) |
| Implementation | ง่ายมาก | ต้องระวัง edge case (overflow, off-by-one) |
| เหมาะกับ | ข้อมูลที่ไม่เรียง หรือขนาดเล็ก | ข้อมูลขนาดใหญ่ที่เรียงลำดับแล้ว |

## 9. Interpolation Search (เทคนิคเสริม)

**Interpolation Search** เป็นการปรับปรุง Binary Search โดย**ประมาณตำแหน่ง**
ของเป้าหมายจากค่าจริง (เหมือนการเดาว่าคำใน dictionary น่าจะอยู่ตรงไหนจากตัวอักษร
ตัวแรก) — เร็วกว่า Binary Search มากสำหรับข้อมูลที่กระจายตัวอย่างสม่ำเสมอ
(uniformly distributed)

```java
public class InterpolationSearchDemo {
    static int interpolationSearch(int[] arr, int target) {
        int low = 0, high = arr.length - 1;

        while (low <= high && target >= arr[low] && target <= arr[high]) {
            if (low == high) {
                return (arr[low] == target) ? low : -1;
            }

            // ประมาณตำแหน่งจากสัดส่วนของค่า (ต่างจาก binary search ที่หาจุดกึ่งกลางตรง ๆ)
            int pos = low + (int) ((double) (high - low) * (target - arr[low]) / (arr[high] - arr[low]));

            if (arr[pos] == target) return pos;
            if (arr[pos] < target) low = pos + 1;
            else high = pos - 1;
        }
        return -1;
    }

    public static void main(String[] args) {
        int[] uniformArray = {10, 20, 30, 40, 50, 60, 70, 80, 90};
        System.out.println(interpolationSearch(uniformArray, 70)); // 6
    }
}
```

**Time Complexity**: O(log log n) average case สำหรับข้อมูลที่กระจายสม่ำเสมอ
(ดีกว่า binary search) แต่ **O(n) worst case** ถ้าข้อมูลกระจายไม่สม่ำเสมอมาก
(เช่นมีค่าผิดปกติ/outlier)

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียนเมธอด `findFirstOccurrence(int[] arr, int target)` ที่หา**ตำแหน่ง
แรกสุด**ของ target ใน array ที่เรียงแล้วและมีค่าซ้ำได้ (โดยใช้ binary search
แบบปรับปรุง)

**เฉลย:**

```java
public class Exercise1 {
    static int findFirstOccurrence(int[] arr, int target) {
        int left = 0, right = arr.length - 1, result = -1;
        while (left <= right) {
            int mid = left + (right - left) / 2;
            if (arr[mid] == target) {
                result = mid;
                right = mid - 1; // ยังคงค้นต่อทางซ้าย เผื่อมีตัวที่มาก่อนอีก
            } else if (arr[mid] < target) {
                left = mid + 1;
            } else {
                right = mid - 1;
            }
        }
        return result;
    }

    public static void main(String[] args) {
        int[] arr = {1, 2, 2, 2, 3, 4, 5};
        System.out.println(findFirstOccurrence(arr, 2)); // 1
    }
}
```

**2)** ใช้ `ArrayStack` (จากหัวข้อ 6) เขียนโปรแกรมตรวจสอบวงเล็บที่ถูกต้อง
(เหมือนตัวอย่างใน Part 25 แต่ใช้ stack ที่เขียนเอง)

**เฉลย**: ใช้โครงสร้างเดียวกับตัวอย่าง `BalancedParenthesesDemo` ใน Part 25 แต่
เปลี่ยนจาก `Deque<Character>` เป็นการเขียน `ArrayStack` เวอร์ชันที่รับ `char`
แทน `int` (ปรับ type ให้ตรงกับความต้องการ)

**3)** อธิบายว่าทำไม Circular Buffer มีประสิทธิภาพดีกว่าการสร้าง Queue แบบ
"เลื่อน element ทุกตัวเมื่อ dequeue"

**เฉลย**: การเลื่อน element ทุกตัวเมื่อ dequeue มี Time Complexity O(n) ต่อครั้ง
(ต้องย้ายทุก element ที่เหลือไปข้างหน้าหนึ่งตำแหน่ง) ในขณะที่ Circular Buffer
ใช้ตัวแปร `front`/`rear` วนตำแหน่งกลับมาใช้พื้นที่ว่างที่ปลาย array แทน ทำให้
`enqueue()`/`dequeue()` เป็น O(1) เสมอ ไม่ต้องย้ายข้อมูลใด ๆ เลย

### สรุปเนื้อหา Part 31

- Linear Search O(n) ใช้ได้กับข้อมูลไม่เรียง, Binary Search O(log n) ต้องการ
  ข้อมูลเรียงลำดับก่อนเสมอ
- ระวัง integer overflow ในการคำนวณ `mid` — ใช้ `left + (right - left) / 2`
  แทน `(left + right) / 2`
- `Arrays.binarySearch()`/`Collections.binarySearch()` ใช้ได้ทันทีโดยไม่ต้อง
  เขียนเอง แต่ต้องมั่นใจว่าข้อมูลเรียงแล้ว
- การสร้าง Stack ด้วย array ใช้ตัวแปร `top` ติดตามตำแหน่งบนสุด
- Circular Buffer ทำให้ Queue ที่สร้างจาก array มี enqueue/dequeue เป็น O(1)
  โดยวนตำแหน่งกลับมาใช้พื้นที่ว่าง

**ต่อไป**: [Part 32 — Linked List แบบ Manual](./part-032-linked-list-manual.md)
