# Part 25: Queue, Deque, PriorityQueue, Stack Class

> ขั้นตอนที่ 241-250 ของหลักสูตร | ระดับ: กลาง

## สารบัญ

1. `Queue` Interface: หลักการ FIFO
2. `ArrayDeque`: Implementation แนะนำสำหรับ Queue และ Stack
3. `Deque` (Double-Ended Queue)
4. การใช้ `ArrayDeque` เป็น Stack (LIFO)
5. `PriorityQueue`: Queue ที่เรียงตามความสำคัญ
6. คลาส `Stack` แบบเก่า (และเหตุผลที่ไม่ควรใช้)
7. เปรียบเทียบ Implementation ทั้งหมดของ Queue/Deque
8. กรณีใช้งานจริง: BFS ด้วย Queue
9. กรณีใช้งานจริง: Task Scheduler ด้วย PriorityQueue
10. แบบฝึกหัดและสรุป

---

## 1. `Queue` Interface: หลักการ FIFO

**Queue** (คิว) ทำงานตามหลัก **FIFO (First-In-First-Out)** — สิ่งที่เข้าไปก่อน
จะถูกดึงออกมาก่อน เปรียบเสมือนการต่อแถวซื้อของ คนที่มาก่อนได้รับบริการก่อน

```java
import java.util.Queue;
import java.util.LinkedList;

public class QueueBasicDemo {
    public static void main(String[] args) {
        Queue<String> queue = new LinkedList<>(); // LinkedList implement Queue ได้ (ทบทวน Part 22)

        queue.offer("ลูกค้า A"); // เพิ่มเข้าคิว (ท้ายแถว) - แนะนำใช้ offer() มากกว่า add()
        queue.offer("ลูกค้า B");
        queue.offer("ลูกค้า C");

        System.out.println(queue); // [ลูกค้า A, ลูกค้า B, ลูกค้า C]

        System.out.println(queue.peek()); // ลูกค้า A (ดูตัวหน้าแถวโดยไม่เอาออก)
        System.out.println(queue.poll());  // ลูกค้า A (ดึงตัวหน้าแถวออกและคืนค่า)
        System.out.println(queue);          // [ลูกค้า B, ลูกค้า C]
    }
}
```

**เมธอดสำคัญของ Queue** — มี 2 รูปแบบ: แบบที่ throw exception เมื่อ queue ว่าง/เต็ม
และแบบที่คืนค่าพิเศษ (null/false) แทน — **แนะนำใช้แบบคืนค่าพิเศษเสมอ**เพราะปลอดภัย
กว่า:

| การดำเนินการ | Throw Exception | คืนค่าพิเศษ (แนะนำ) |
|---|---|---|
| เพิ่มเข้า queue | `add(e)` | `offer(e)` |
| ดึงออกและคืนค่า | `remove()` | `poll()` (คืน `null` ถ้าว่าง) |
| ดูโดยไม่เอาออก | `element()` | `peek()` (คืน `null` ถ้าว่าง) |

## 2. `ArrayDeque`: Implementation แนะนำสำหรับ Queue และ Stack

แม้ `LinkedList` จะ implement `Queue` ได้ แต่ **`ArrayDeque` เร็วกว่าและใช้
หน่วยความจำน้อยกว่า** ในการทำหน้าที่เป็น Queue หรือ Stack — Java Documentation
เองก็แนะนำให้ใช้ `ArrayDeque` แทน `LinkedList` และ `Stack` แบบเก่าในกรณีทั่วไป

```java
import java.util.ArrayDeque;
import java.util.Queue;

public class ArrayDequeQueueDemo {
    public static void main(String[] args) {
        Queue<String> queue = new ArrayDeque<>(); // แนะนำมากกว่า new LinkedList<>()

        queue.offer("งานที่ 1");
        queue.offer("งานที่ 2");
        queue.offer("งานที่ 3");

        while (!queue.isEmpty()) {
            System.out.println("กำลังทำ: " + queue.poll());
        }
        // ผลลัพธ์: ทำตามลำดับ 1, 2, 3 (FIFO)
    }
}
```

**เหตุผลที่ `ArrayDeque` เร็วกว่า `LinkedList`**: `ArrayDeque` ใช้ **circular
array** ภายใน (ไม่ต้องจัดสรร object node แยกทีละตัวเหมือน LinkedList) ทำให้
cache-friendly กว่าและมี overhead ต่อ element น้อยกว่ามาก

## 3. `Deque` (Double-Ended Queue)

**`Deque`** (อ่านว่า "deck") คือ queue ที่**เพิ่ม/ลบได้ทั้งสองด้าน** (หัวและท้าย)
`ArrayDeque` implement interface นี้โดยตรง ทำให้ใช้เป็นได้ทั้ง Queue, Stack,
หรือ Deque เต็มรูปแบบในตัวเดียว

```java
import java.util.ArrayDeque;
import java.util.Deque;

public class DequeDemo {
    public static void main(String[] args) {
        Deque<Integer> deque = new ArrayDeque<>();

        deque.addFirst(1);  // เพิ่มที่หัว: [1]
        deque.addLast(2);   // เพิ่มที่ท้าย: [1, 2]
        deque.addFirst(0);  // เพิ่มที่หัว: [0, 1, 2]
        deque.addLast(3);   // เพิ่มที่ท้าย: [0, 1, 2, 3]

        System.out.println(deque); // [0, 1, 2, 3]

        System.out.println(deque.peekFirst()); // 0
        System.out.println(deque.peekLast());   // 3

        deque.removeFirst(); // เอา 0 ออก
        deque.removeLast();   // เอา 3 ออก
        System.out.println(deque); // [1, 2]
    }
}
```

**กรณีใช้งาน**: **Sliding Window algorithms** (ปรับหน้าต่างข้อมูลได้ทั้งสองด้าน),
**Undo/Redo functionality** (เพิ่มที่ท้าย ลบจากหัวหรือท้ายตามต้องการ), **Palindrome
checking** (เทียบจากทั้งสองด้านเข้าหากัน)

## 4. การใช้ `ArrayDeque` เป็น Stack (LIFO)

**Stack** ทำงานตามหลัก **LIFO (Last-In-First-Out)** — สิ่งที่เข้าไปหลังสุดจะถูก
ดึงออกมาก่อน เปรียบเสมือนกองจาน ใครวางจานบนสุดก็หยิบออกก่อน

```java
import java.util.ArrayDeque;
import java.util.Deque;

public class StackViaDequeDemo {
    public static void main(String[] args) {
        Deque<String> stack = new ArrayDeque<>(); // ใช้ Deque เป็น Stack

        stack.push("A"); // เทียบเท่า addFirst()
        stack.push("B");
        stack.push("C");

        System.out.println(stack); // [C, B, A]

        System.out.println(stack.peek()); // C (ดูตัวบนสุดโดยไม่เอาออก)
        System.out.println(stack.pop());   // C (เอาตัวบนสุดออกและคืนค่า)
        System.out.println(stack);          // [B, A]
    }
}
```

**ตัวอย่างการใช้งานจริง**: ตรวจสอบวงเล็บที่ถูกต้อง (Balanced Parentheses) — โจทย์
สัมภาษณ์งานคลาสสิกที่ใช้ Stack:

```java
import java.util.ArrayDeque;
import java.util.Deque;

public class BalancedParenthesesDemo {
    static boolean isBalanced(String expression) {
        Deque<Character> stack = new ArrayDeque<>();

        for (char c : expression.toCharArray()) {
            if (c == '(' || c == '[' || c == '{') {
                stack.push(c);
            } else if (c == ')' || c == ']' || c == '}') {
                if (stack.isEmpty()) return false; // ปิดแต่ไม่มีตัวเปิดค้างอยู่
                char open = stack.pop();
                if ((c == ')' && open != '(') ||
                    (c == ']' && open != '[') ||
                    (c == '}' && open != '{')) {
                    return false; // ตัวเปิดกับตัวปิดไม่ตรงคู่กัน
                }
            }
        }
        return stack.isEmpty(); // ต้องไม่มีตัวเปิดหลงเหลืออยู่เลย
    }

    public static void main(String[] args) {
        System.out.println(isBalanced("{[()]}"));  // true
        System.out.println(isBalanced("{[(])}"));  // false
        System.out.println(isBalanced("((("));     // false (เปิดไม่ครบคู่)
    }
}
```

## 5. `PriorityQueue`: Queue ที่เรียงตามความสำคัญ

**`PriorityQueue`** ไม่ทำงานตาม FIFO ธรรมดา แต่**ดึงค่าที่มีความสำคัญสูงสุด
(หรือน้อยสุดขึ้นกับการกำหนด) ออกมาก่อนเสมอ** — ใช้โครงสร้างข้อมูล **Binary Heap**
ภายใน (จะเรียนลึกเรื่อง Heap ใน Part 33-34) ทำให้ `poll()` ได้ค่าที่เหมาะสมที่สุด
ใน **O(log n)**

```java
import java.util.PriorityQueue;

public class PriorityQueueDemo {
    public static void main(String[] args) {
        // ค่าเริ่มต้น: min-heap (ดึงค่าน้อยที่สุดออกก่อนเสมอ)
        PriorityQueue<Integer> pq = new PriorityQueue<>();
        pq.offer(50);
        pq.offer(10);
        pq.offer(30);
        pq.offer(20);

        while (!pq.isEmpty()) {
            System.out.print(pq.poll() + " "); // 10 20 30 50 (เรียงจากน้อยไปมากเสมอ)
        }
        System.out.println();

        // ใช้ Comparator เพื่อทำ max-heap (ดึงค่ามากที่สุดออกก่อน)
        PriorityQueue<Integer> maxHeap = new PriorityQueue<>(java.util.Comparator.reverseOrder());
        maxHeap.offer(50);
        maxHeap.offer(10);
        maxHeap.offer(30);

        while (!maxHeap.isEmpty()) {
            System.out.print(maxHeap.poll() + " "); // 50 30 10
        }
    }
}
```

### PriorityQueue กับ Custom Object

```java
public class Task {
    String name;
    int priority; // ตัวเลขน้อย = ความสำคัญสูง

    public Task(String name, int priority) {
        this.name = name;
        this.priority = priority;
    }

    @Override
    public String toString() {
        return name + "(P" + priority + ")";
    }
}
```

```java
import java.util.PriorityQueue;
import java.util.Comparator;

public class TaskPriorityQueueDemo {
    public static void main(String[] args) {
        PriorityQueue<Task> tasks = new PriorityQueue<>(
            Comparator.comparingInt(t -> t.priority) // เรียงตาม priority จากน้อยไปมาก
        );

        tasks.offer(new Task("ล้างจาน", 3));
        tasks.offer(new Task("ดับไฟไหม้!", 1));
        tasks.offer(new Task("รดน้ำต้นไม้", 2));

        while (!tasks.isEmpty()) {
            System.out.println("ทำงาน: " + tasks.poll()); // ดับไฟไหม้! ก่อนเสมอ เพราะ priority ต่ำสุด
        }
    }
}
```

## 6. คลาส `Stack` แบบเก่า (และเหตุผลที่ไม่ควรใช้)

Java มีคลาส `java.util.Stack` มาตั้งแต่ Java 1.0 (**ก่อนมี Collections
Framework**) ยังใช้งานได้อยู่ แต่**ไม่แนะนำให้ใช้ในโค้ดใหม่**:

```java
import java.util.Stack;

public class OldStackDemo {
    public static void main(String[] args) {
        Stack<String> stack = new Stack<>(); // ใช้ได้ แต่ไม่แนะนำ
        stack.push("A");
        stack.push("B");
        System.out.println(stack.pop()); // B
    }
}
```

**เหตุผลที่ควรใช้ `ArrayDeque` แทน `Stack`**:
1. `Stack` สืบทอดจาก `Vector` ซึ่งเป็น **legacy class** ที่ทุกเมธอดเป็น
   `synchronized` (thread-safe แต่ช้ากว่ามากในกรณี single-thread ที่พบบ่อยที่สุด)
2. `Stack` สืบทอดจาก `Vector` ทำให้มีเมธอดของ `List` ปนเข้ามาด้วย (เช่น
   `get(index)`, `add(index, e)`) ซึ่ง**ขัดกับหลักการของ Stack ที่ควรเข้าถึงได้
   แค่ปลายบนสุดเท่านั้น** — ทำให้เผลอเขียนโค้ดที่ผิดหลักการได้ง่าย
3. Java Documentation เองก็ระบุชัดเจนว่า `Deque` (ผ่าน `ArrayDeque`) เป็นตัวเลือก
   ที่ดีกว่าสำหรับ Stack operations

## 7. เปรียบเทียบ Implementation ทั้งหมดของ Queue/Deque

| Implementation | ใช้เป็น Queue | ใช้เป็น Stack | Thread-safe | แนะนำ |
|---|---|---|---|---|
| `ArrayDeque` | ✅ เร็ว | ✅ เร็ว | ❌ | **ใช้เป็นค่าเริ่มต้นเสมอ** |
| `LinkedList` | ✅ | ✅ | ❌ | ใช้เมื่อต้องการ List operations ร่วมด้วย |
| `PriorityQueue` | ✅ (ตามความสำคัญ) | ❌ | ❌ | เมื่อต้องเรียงตามความสำคัญ |
| `Stack` (legacy) | ❌ | ✅ (แต่ไม่แนะนำ) | ✅ | หลีกเลี่ยงในโค้ดใหม่ |
| `ConcurrentLinkedQueue` | ✅ | ❌ | ✅ | multi-thread environment (Part 50) |

## 8. กรณีใช้งานจริง: BFS ด้วย Queue

**Breadth-First Search (BFS)** เป็นอัลกอริทึมค้นหาที่ใช้ Queue เป็นแกนหลัก
(จะลงรายละเอียดเต็มรูปแบบใน Part 34 เรื่อง Graph) ตัวอย่างนี้แสดงแนวคิดพื้นฐาน:

```java
import java.util.ArrayDeque;
import java.util.HashSet;
import java.util.Map;
import java.util.Queue;
import java.util.Set;
import java.util.List;

public class BFSPreviewDemo {
    public static void main(String[] args) {
        Map<String, List<String>> graph = Map.of(
            "A", List.of("B", "C"),
            "B", List.of("D"),
            "C", List.of("D"),
            "D", List.of()
        );

        Queue<String> queue = new ArrayDeque<>();
        Set<String> visited = new HashSet<>();

        queue.offer("A");
        visited.add("A");

        while (!queue.isEmpty()) {
            String current = queue.poll();
            System.out.println("เยี่ยม node: " + current);

            for (String neighbor : graph.get(current)) {
                if (!visited.contains(neighbor)) {
                    visited.add(neighbor);
                    queue.offer(neighbor);
                }
            }
        }
        // ผลลัพธ์: A, B, C, D (เยี่ยมทีละ "ระดับ" จากจุดเริ่มต้นออกไป)
    }
}
```

## 9. กรณีใช้งานจริง: Task Scheduler ด้วย PriorityQueue

```java
import java.util.PriorityQueue;
import java.util.Comparator;

public class TaskSchedulerDemo {
    record ScheduledTask(String name, int priority, long timestamp) { } // ทบทวนใน Part 51

    public static void main(String[] args) {
        PriorityQueue<ScheduledTask> scheduler = new PriorityQueue<>(
            Comparator.comparingInt(ScheduledTask::priority)
                      .thenComparingLong(ScheduledTask::timestamp) // ถ้า priority เท่ากัน ใช้เวลาตัดสิน
        );

        scheduler.offer(new ScheduledTask("ส่งอีเมล", 5, 100));
        scheduler.offer(new ScheduledTask("แจ้งเตือนฉุกเฉิน", 1, 200));
        scheduler.offer(new ScheduledTask("สำรองข้อมูล", 3, 50));

        while (!scheduler.isEmpty()) {
            System.out.println("ประมวลผล: " + scheduler.poll());
        }
        // ผลลัพธ์: แจ้งเตือนฉุกเฉิน (priority 1) ก่อนเสมอ ตามด้วยสำรองข้อมูล (3), ส่งอีเมล (5)
    }
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** ใช้ `ArrayDeque` เขียนโปรแกรมตรวจสอบว่าข้อความเป็น Palindrome หรือไม่
(เทียบกับ Part 9 ที่ใช้ StringBuilder) โดยใช้ deque เทียบจากหัวและท้ายเข้าหากัน

**เฉลย:**

```java
import java.util.ArrayDeque;
import java.util.Deque;

public class Exercise1 {
    static boolean isPalindrome(String text) {
        Deque<Character> deque = new ArrayDeque<>();
        for (char c : text.toLowerCase().toCharArray()) {
            if (Character.isLetterOrDigit(c)) {
                deque.addLast(c);
            }
        }
        while (deque.size() > 1) {
            if (!deque.pollFirst().equals(deque.pollLast())) {
                return false;
            }
        }
        return true;
    }

    public static void main(String[] args) {
        System.out.println(isPalindrome("level"));  // true
        System.out.println(isPalindrome("hello"));   // false
    }
}
```

**2)** เขียนโปรแกรมจำลอง "ห้องฉุกเฉินโรงพยาบาล" ด้วย `PriorityQueue` ที่รับผู้ป่วย
พร้อมระดับความรุนแรง (1=วิกฤต, 5=เล็กน้อย) แล้วเรียกตรวจตามความรุนแรง

**เฉลย:**

```java
import java.util.PriorityQueue;
import java.util.Comparator;

public class Exercise2 {
    record Patient(String name, int severity) { }

    public static void main(String[] args) {
        PriorityQueue<Patient> er = new PriorityQueue<>(Comparator.comparingInt(Patient::severity));
        er.offer(new Patient("คนไข้ A", 4));
        er.offer(new Patient("คนไข้ B", 1));
        er.offer(new Patient("คนไข้ C", 3));

        while (!er.isEmpty()) {
            System.out.println("เรียกตรวจ: " + er.poll());
        }
    }
}
```

**3)** อธิบายว่าทำไมควรใช้ `ArrayDeque` แทน `java.util.Stack` ในโค้ดใหม่

**เฉลย**: `Stack` สืบทอดจาก `Vector` ซึ่งเป็น legacy class ที่ทุกเมธอดเป็น
`synchronized` ทำให้ช้ากว่าในสถานการณ์ single-thread ที่พบบ่อยที่สุด นอกจากนี้
`Stack` ยังมีเมธอดของ `List` ปนเข้ามา (เช่น `get(index)`) ซึ่งขัดกับหลักการ
Stack ที่ควรเข้าถึงได้แค่ด้านบนสุดเท่านั้น `ArrayDeque` แก้ปัญหาทั้งสองข้อนี้
และมีประสิทธิภาพดีกว่าโดยรวม

### สรุปเนื้อหา Part 25

- Queue ทำงานแบบ FIFO, Stack ทำงานแบบ LIFO
- `ArrayDeque` เป็นตัวเลือกที่ดีที่สุดสำหรับทั้ง Queue และ Stack ในโค้ดใหม่ (เร็ว
  กว่า LinkedList และปลอดภัยกว่า Stack แบบเก่า)
- `Deque` เพิ่ม/ลบได้ทั้งสองด้าน (หัวและท้าย) เหมาะกับ sliding window, undo/redo
- `PriorityQueue` ดึงค่าที่มีความสำคัญสูงสุดออกก่อนเสมอ ใช้ Binary Heap ภายใน
  (O(log n))
- ใช้เมธอดแบบคืนค่าพิเศษ (`offer`, `poll`, `peek`) แทนแบบ throw exception เสมอ
  เพื่อความปลอดภัย
- BFS ใช้ Queue เป็นแกนหลัก, Task Scheduling ใช้ PriorityQueue

**ต่อไป**: [Part 26 — Generics: Generic Class, Method, Bounded Types, Wildcards](./part-026-generics.md)
