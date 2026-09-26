# Part 22: Collections Framework ภาพรวม, List: ArrayList, LinkedList

> ขั้นตอนที่ 211-220 ของหลักสูตร | ระดับ: กลาง

## สารบัญ

1. Collections Framework คืออะไร
2. Collection Hierarchy ภาพรวม
3. `List` Interface
4. `ArrayList` แบบละเอียด
5. `LinkedList` แบบละเอียด
6. เปรียบเทียบ ArrayList กับ LinkedList (Big O)
7. เมธอดสำคัญที่ทุก List มีร่วมกัน
8. การวนซ้ำ List และ `ConcurrentModificationException`
9. `List.of()` และ Immutable List
10. แบบฝึกหัดและสรุป

---

## 1. Collections Framework คืออะไร

**Collections Framework** คือชุดของ interface และ class ใน `java.util` ที่ใช้
จัดการกลุ่มข้อมูล (group of objects) — เป็นการยกระดับจาก array (Part 7) ที่มี
ขนาดคงที่ ไปสู่โครงสร้างข้อมูลที่**ขยายขนาดได้แบบไดนามิก**และมีเมธอดสำเร็จรูป
มากมาย

**ทำไมต้องใช้ Collections แทน Array?**

```java
public class ArrayLimitationDemo {
    public static void main(String[] args) {
        int[] numbers = new int[5]; // ขนาดคงที่ตายตัวตั้งแต่สร้าง แก้ไขไม่ได้อีก
        numbers[0] = 1;
        numbers[1] = 2;
        // ถ้าต้องการเพิ่มตัวที่ 6 ต้องสร้าง array ใหม่ทั้งหมดแล้วคัดลอกข้อมูลเก่ามา
        // ไม่มีเมธอดสำเร็จรูปสำหรับ "ลบ" หรือ "แทรก" ตรงกลางเลย ต้องเขียน logic เอง
    }
}
```

```java
import java.util.ArrayList;
import java.util.List;

public class CollectionAdvantageDemo {
    public static void main(String[] args) {
        List<Integer> numbers = new ArrayList<>(); // ขยายขนาดได้อัตโนมัติ
        numbers.add(1);
        numbers.add(2);
        numbers.add(3);
        numbers.remove(1);        // ลบ element ที่ index 1 ได้ทันที
        numbers.add(1, 99);        // แทรกที่ index 1 ได้ทันที
        System.out.println(numbers); // [1, 99, 3]
    }
}
```

## 2. Collection Hierarchy ภาพรวม

```
                        Iterable
                            |
                       Collection
              ┌─────────────┼─────────────┐
             List           Set          Queue
              |              |             |
        ArrayList       HashSet      LinkedList
        LinkedList      LinkedHashSet PriorityQueue
        Vector          TreeSet       ArrayDeque
        Stack

                Map (แยกสาย ไม่ได้สืบทอดจาก Collection)
              ┌────┴────┐
          HashMap    TreeMap
          LinkedHashMap
```

- **`List`**: เก็บข้อมูลแบบมีลำดับ (ordered) และ**อนุญาตค่าซ้ำได้** เข้าถึงด้วย index
- **`Set`**: เก็บข้อมูลที่**ไม่ซ้ำกัน** (unique) — Part 23
- **`Queue`**: เก็บข้อมูลแบบ FIFO (หรือ priority) — Part 25
- **`Map`**: เก็บคู่ key-value — Part 24 (ไม่ได้สืบทอดจาก `Collection` แต่ถือเป็น
  ส่วนหนึ่งของ framework)

Part นี้จะเจาะลึกเฉพาะ `List` เท่านั้น

## 3. `List` Interface

`List<E>` เป็น **generic interface** (จะเรียนลึกใน Part 26) — `E` คือชนิดข้อมูล
ที่ List นี้เก็บ กำหนดตอนสร้าง object

```java
import java.util.List;
import java.util.ArrayList;

public class ListInterfaceDemo {
    public static void main(String[] args) {
        // ประกาศด้วย interface (List) แต่สร้าง object จริงด้วย implementation (ArrayList)
        // นี่คือแนวปฏิบัติที่ดี (Program to an interface, not an implementation)
        List<String> names = new ArrayList<>();

        names.add("Alice");
        names.add("Bob");
        names.add("Charlie");

        System.out.println(names);       // [Alice, Bob, Charlie]
        System.out.println(names.get(0)); // Alice (เข้าถึงผ่าน index เหมือน array)
        System.out.println(names.size()); // 3
    }
}
```

**ทำไมประกาศตัวแปรเป็น `List<String>` แทน `ArrayList<String>`?** เพราะทำให้
เปลี่ยน implementation ในอนาคตได้ง่าย (เช่น เปลี่ยนจาก `ArrayList` เป็น
`LinkedList`) โดยแก้แค่บรรทัดที่สร้าง object บรรทัดเดียว โค้ดที่เหลือทั้งหมด
ไม่ต้องแก้ไขเลย เพราะเรียกผ่าน interface เดียวกัน

## 4. `ArrayList` แบบละเอียด

`ArrayList` ใช้ **array ภายในที่ขยายขนาดอัตโนมัติ** เมื่อเต็ม (โดยปกติจะขยายเป็น
1.5 เท่าของขนาดเดิม) เหมาะกับการ**เข้าถึงข้อมูลด้วย index บ่อย ๆ** และ**เพิ่ม/ลบ
ที่ท้าย list**

```java
import java.util.ArrayList;
import java.util.List;

public class ArrayListDemo {
    public static void main(String[] args) {
        List<String> fruits = new ArrayList<>();

        // เพิ่มข้อมูล
        fruits.add("แอปเปิ้ล");           // เพิ่มที่ท้าย list
        fruits.add("กล้วย");
        fruits.add(1, "ส้ม");              // แทรกที่ index 1 (ตัวอื่นเลื่อนขวา)
        System.out.println(fruits); // [แอปเปิ้ล, ส้ม, กล้วย]

        // เข้าถึงและแก้ไข
        System.out.println(fruits.get(0));  // แอปเปิ้ล
        fruits.set(0, "มะม่วง");             // แทนที่ค่าที่ index 0
        System.out.println(fruits); // [มะม่วง, ส้ม, กล้วย]

        // ลบข้อมูล
        fruits.remove("ส้ม");                // ลบตาม "ค่า" (element)
        fruits.remove(0);                    // ลบตาม "index"
        System.out.println(fruits); // [กล้วย]

        // ตรวจสอบ
        System.out.println(fruits.contains("กล้วย")); // true
        System.out.println(fruits.indexOf("กล้วย"));    // 0
        System.out.println(fruits.isEmpty());          // false

        fruits.clear(); // ลบทุก element
        System.out.println(fruits.isEmpty()); // true
    }
}
```

### สร้าง ArrayList พร้อมค่าเริ่มต้น

```java
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;

public class ArrayListInitDemo {
    public static void main(String[] args) {
        // วิธีที่ 1: จาก List.of() (immutable) แล้วห่อด้วย ArrayList ใหม่ (mutable)
        List<Integer> list1 = new ArrayList<>(List.of(1, 2, 3));

        // วิธีที่ 2: จาก Arrays.asList() (list ขนาดคงที่ ห่อ array เดิม)
        List<Integer> list2 = new ArrayList<>(Arrays.asList(1, 2, 3));

        // วิธีที่ 3: กำหนดขนาดเริ่มต้น (initial capacity) เพื่อประสิทธิภาพ ถ้ารู้ขนาดล่วงหน้า
        List<Integer> list3 = new ArrayList<>(100); // จองพื้นที่ไว้ล่วงหน้า 100 ช่อง (ไม่ใช่ขนาดจริง)
        System.out.println(list3.size()); // 0 (ยังไม่มี element ใด ๆ แม้จองพื้นที่ไว้แล้ว)

        list1.add(4);
        System.out.println(list1); // [1, 2, 3, 4]
    }
}
```

## 5. `LinkedList` แบบละเอียด

`LinkedList` เก็บข้อมูลแบบ **doubly linked list** (แต่ละ node ชี้ไปทั้งตัวก่อน
หน้าและตัวถัดไป — จะเขียนเองใน Part 32) เหมาะกับการ**เพิ่ม/ลบที่หัวหรือท้าย list
บ่อย ๆ** และยัง implement `Deque` interface ทำให้ใช้เป็น stack/queue ได้ด้วย

```java
import java.util.LinkedList;

public class LinkedListDemo {
    public static void main(String[] args) {
        LinkedList<String> tasks = new LinkedList<>();

        tasks.add("งานที่ 1");
        tasks.add("งานที่ 2");
        tasks.addFirst("งานเร่งด่วน"); // เพิ่มที่หัว list - เร็วมาก O(1) สำหรับ LinkedList
        tasks.addLast("งานที่ 3");      // เพิ่มที่ท้าย list

        System.out.println(tasks); // [งานเร่งด่วน, งานที่ 1, งานที่ 2, งานที่ 3]

        System.out.println(tasks.getFirst()); // งานเร่งด่วน
        System.out.println(tasks.getLast());   // งานที่ 3

        tasks.removeFirst(); // เอางานเร่งด่วนออกหลังทำเสร็จ
        System.out.println(tasks);

        // LinkedList ใช้เป็น Queue (FIFO) ได้ (จะลงลึกใน Part 25)
        tasks.offer("งานใหม่");   // เพิ่มเข้า queue (เทียบเท่า addLast)
        String nextTask = tasks.poll(); // ดึงและลบตัวหน้าสุดออก (เทียบเท่า removeFirst)
        System.out.println("งานถัดไปที่ต้องทำ: " + nextTask);
    }
}
```

## 6. เปรียบเทียบ ArrayList กับ LinkedList (Big O)

| การดำเนินการ | `ArrayList` | `LinkedList` |
|---|---|---|
| เข้าถึงด้วย index `get(i)` | **O(1)** — เร็วมาก (คำนวณ address ตรง ๆ) | O(n) — ต้องไล่ node ทีละตัว |
| เพิ่ม/ลบที่**ท้าย** list | O(1) โดยเฉลี่ย (บางครั้ง O(n) เมื่อต้อง resize) | **O(1)** เสมอ |
| เพิ่ม/ลบที่**หัว** list | O(n) — ต้องเลื่อนทุก element | **O(1)** — แค่เปลี่ยน pointer |
| เพิ่ม/ลบที่**กลาง** list | O(n) | O(n) (แม้การแทรกเป็น O(1) แต่ต้องเสียเวลาไล่หา node ก่อน) |
| หน่วยความจำ | ประหยัดกว่า (เก็บแค่ข้อมูล) | ใช้มากกว่า (แต่ละ node ต้องเก็บ pointer 2 ตัวเพิ่ม) |

**หลักการเลือกใช้ในทางปฏิบัติ**:
- **ใช้ `ArrayList` เป็นค่าเริ่มต้นเสมอ** (95% ของกรณีใช้งานทั่วไป) เพราะเข้าถึง
  ด้วย index เร็วกว่ามาก และประหยัดหน่วยความจำกว่า
- ใช้ `LinkedList` เฉพาะเมื่อ**ต้องเพิ่ม/ลบที่หัว list บ่อยมาก** หรือต้องการใช้
  เป็น Queue/Deque โดยเฉพาะ (แต่ในทางปฏิบัติ `ArrayDeque` มักเร็วกว่า `LinkedList`
  สำหรับ Queue/Stack — จะเรียนใน Part 25)

```java
import java.util.ArrayList;
import java.util.LinkedList;
import java.util.List;

public class PerformanceComparisonDemo {
    public static void main(String[] args) {
        int n = 100_000;

        List<Integer> arrayList = new ArrayList<>();
        long start = System.nanoTime();
        for (int i = 0; i < n; i++) arrayList.add(0, i); // เพิ่มที่หัวเสมอ (worst case สำหรับ ArrayList)
        long arrayListTime = System.nanoTime() - start;

        List<Integer> linkedList = new LinkedList<>();
        start = System.nanoTime();
        for (int i = 0; i < n; i++) linkedList.add(0, i); // เพิ่มที่หัวเสมอ (best case สำหรับ LinkedList)
        long linkedListTime = System.nanoTime() - start;

        System.out.println("ArrayList เพิ่มที่หัว: " + arrayListTime / 1_000_000 + " ms");
        System.out.println("LinkedList เพิ่มที่หัว: " + linkedListTime / 1_000_000 + " ms");
        // LinkedList จะเร็วกว่ามากในกรณีนี้โดยเฉพาะ
    }
}
```

## 7. เมธอดสำคัญที่ทุก List มีร่วมกัน

```java
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public class ListCommonMethodsDemo {
    public static void main(String[] args) {
        List<Integer> numbers = new ArrayList<>(List.of(5, 3, 8, 1, 9, 3));

        System.out.println(numbers.size());              // 6
        System.out.println(numbers.contains(8));           // true
        System.out.println(numbers.indexOf(3));            // 1 (ตำแหน่งแรกที่เจอ)
        System.out.println(numbers.lastIndexOf(3));         // 5 (ตำแหน่งสุดท้ายที่เจอ)

        List<Integer> sub = numbers.subList(1, 4);          // sub-list จาก index 1 ถึงก่อน 4
        System.out.println(sub); // [3, 8, 1]

        Collections.sort(numbers);                            // เรียงลำดับ (ใช้ utility class Collections)
        System.out.println(numbers); // [1, 3, 3, 5, 8, 9]

        Collections.reverse(numbers);
        System.out.println(numbers); // [9, 8, 5, 3, 3, 1]

        System.out.println(Collections.max(numbers));         // 9
        System.out.println(Collections.min(numbers));         // 1

        Integer[] array = numbers.toArray(new Integer[0]);    // แปลง List เป็น array
        System.out.println(array.length);
    }
}
```

## 8. การวนซ้ำ List และ `ConcurrentModificationException`

```java
import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;

public class ConcurrentModificationDemo {
    public static void main(String[] args) {
        List<Integer> numbers = new ArrayList<>(List.of(1, 2, 3, 4, 5));

        try {
            for (int num : numbers) { // for-each ใช้ Iterator ภายในโดยอัตโนมัติ
                if (num == 3) {
                    numbers.remove(Integer.valueOf(3)); // แก้ไข list ระหว่างวน loop -> เกิดปัญหา!
                }
            }
        } catch (java.util.ConcurrentModificationException e) {
            System.out.println("Error: แก้ไข list ระหว่างวน for-each ไม่ได้!");
        }

        // วิธีแก้ที่ถูกต้อง: ใช้ Iterator.remove() โดยตรง
        List<Integer> numbers2 = new ArrayList<>(List.of(1, 2, 3, 4, 5));
        Iterator<Integer> it = numbers2.iterator();
        while (it.hasNext()) {
            int num = it.next();
            if (num == 3) {
                it.remove(); // ปลอดภัย! Iterator รู้ว่าตัวเองเพิ่งลบ ไม่เกิดความขัดแย้งของ state
            }
        }
        System.out.println(numbers2); // [1, 2, 4, 5]

        // หรือใช้ removeIf() (Java 8+) ซึ่งกระชับและปลอดภัยที่สุด
        List<Integer> numbers3 = new ArrayList<>(List.of(1, 2, 3, 4, 5));
        numbers3.removeIf(n -> n == 3);
        System.out.println(numbers3); // [1, 2, 4, 5]
    }
}
```

## 9. `List.of()` และ Immutable List

`List.of()` (Java 9+) สร้าง **immutable list** (แก้ไขไม่ได้เลยหลังสร้าง) —
เหมาะกับข้อมูลที่ไม่ต้องการให้เปลี่ยนแปลง (ทบทวนแนวคิด immutability จาก Part 17)

```java
import java.util.List;

public class ImmutableListDemo {
    public static void main(String[] args) {
        List<String> colors = List.of("แดง", "เขียว", "น้ำเงิน");

        System.out.println(colors); // [แดง, เขียว, น้ำเงิน]

        try {
            colors.add("เหลือง"); // Error!
        } catch (UnsupportedOperationException e) {
            System.out.println("แก้ไข immutable list ไม่ได้");
        }

        // ถ้าต้องการ list ที่แก้ไขได้ ให้สร้าง ArrayList ใหม่ครอบ List.of() อีกที
        List<String> mutableColors = new java.util.ArrayList<>(colors);
        mutableColors.add("เหลือง"); // ได้แล้ว เพราะเป็น ArrayList จริง ๆ
        System.out.println(mutableColors);
    }
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียนโปรแกรมที่มี `List<String>` ของชื่อนักเรียน แล้วเขียนเมธอด
`removeDuplicates(List<String> list)` ที่คืนค่า list ใหม่ที่ไม่มีชื่อซ้ำ

**เฉลย:**

```java
import java.util.ArrayList;
import java.util.LinkedHashSet;
import java.util.List;

public class Exercise1 {
    static List<String> removeDuplicates(List<String> list) {
        return new ArrayList<>(new LinkedHashSet<>(list)); // ใช้ Set กำจัดค่าซ้ำ (Part 23)
    }

    public static void main(String[] args) {
        List<String> names = List.of("Alice", "Bob", "Alice", "Charlie", "Bob");
        System.out.println(removeDuplicates(names)); // [Alice, Bob, Charlie]
    }
}
```

**2)** เขียนเมธอด `findSecondLargest(List<Integer> numbers)` ที่หาค่ามากที่สุด
อันดับ 2 โดยไม่ใช้ `Collections.sort()`

**เฉลย:**

```java
import java.util.List;

public class Exercise2 {
    static int findSecondLargest(List<Integer> numbers) {
        int largest = Integer.MIN_VALUE, secondLargest = Integer.MIN_VALUE;
        for (int num : numbers) {
            if (num > largest) {
                secondLargest = largest;
                largest = num;
            } else if (num > secondLargest && num != largest) {
                secondLargest = num;
            }
        }
        return secondLargest;
    }

    public static void main(String[] args) {
        System.out.println(findSecondLargest(List.of(5, 3, 9, 1, 9, 7))); // 7
    }
}
```

**3)** ใช้ `LinkedList` เขียนโปรแกรมจำลองคิว "ผู้ป่วยรอตรวจ" ที่รองรับการเพิ่ม
ผู้ป่วยฉุกเฉินไว้หัวคิว และผู้ป่วยทั่วไปไปต่อท้ายคิว

**เฉลย:**

```java
import java.util.LinkedList;

public class Exercise3 {
    public static void main(String[] args) {
        LinkedList<String> queue = new LinkedList<>();
        queue.addLast("ผู้ป่วย A (ทั่วไป)");
        queue.addLast("ผู้ป่วย B (ทั่วไป)");
        queue.addFirst("ผู้ป่วย C (ฉุกเฉิน!)");

        System.out.println(queue); // [ผู้ป่วย C (ฉุกเฉิน!), ผู้ป่วย A (ทั่วไป), ผู้ป่วย B (ทั่วไป)]

        while (!queue.isEmpty()) {
            System.out.println("เรียกตรวจ: " + queue.removeFirst());
        }
    }
}
```

### สรุปเนื้อหา Part 22

- Collections Framework ให้โครงสร้างข้อมูลที่ขยายขนาดได้แบบไดนามิก พร้อมเมธอด
  สำเร็จรูปมากมาย เหนือกว่า array แบบพื้นฐาน
- `List` เก็บข้อมูลแบบมีลำดับ อนุญาตค่าซ้ำ เข้าถึงด้วย index
- `ArrayList` เร็วสำหรับเข้าถึงด้วย index (O(1)), `LinkedList` เร็วสำหรับเพิ่ม/ลบ
  ที่หัว-ท้าย list (O(1))
- ใช้ `ArrayList` เป็นค่าเริ่มต้นเสมอ เว้นแต่มีเหตุผลเฉพาะให้ใช้ `LinkedList`
- ห้ามแก้ไข list ระหว่างวน for-each โดยตรง ให้ใช้ `Iterator.remove()` หรือ
  `removeIf()` แทน เพื่อป้องกัน `ConcurrentModificationException`
- `List.of()` สร้าง immutable list ที่แก้ไขไม่ได้หลังสร้าง

**ต่อไป**: [Part 23 — Collections Framework: Set](./part-023-collections-set.md)
