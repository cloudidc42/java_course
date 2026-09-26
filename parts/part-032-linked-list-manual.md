# Part 32: Linked List แบบ Manual (Singly, Doubly, Circular)

> ขั้นตอนที่ 311-320 ของหลักสูตร | ระดับ: กลาง-สูง

## สารบัญ

1. Linked List คืออะไร ต่างจาก Array อย่างไร
2. Singly Linked List: การสร้างและเมธอดพื้นฐาน
3. การแทรกและลบใน Singly Linked List
4. Doubly Linked List
5. Circular Linked List
6. การ Reverse Linked List
7. การหา Cycle ด้วย Floyd's Algorithm (Two Pointers)
8. การหา Middle Element ด้วย Two Pointers
9. เปรียบเทียบ Linked List กับ ArrayList (ทบทวนเชิงลึก)
10. แบบฝึกหัดและสรุป

---

## 1. Linked List คืออะไร ต่างจาก Array อย่างไร

**Linked List** คือโครงสร้างข้อมูลที่ประกอบด้วย **node** หลายตัวเชื่อมต่อกัน
แต่ละ node เก็บ**ข้อมูล**และ**reference ไปยัง node ถัดไป** — ต่างจาก array ที่
เก็บข้อมูลต่อเนื่องกันในหน่วยความจำ (contiguous memory) node ของ linked list
**กระจายอยู่ในหน่วยความจำที่ไหนก็ได้**

```
Array:        [10][20][30][40]   <- ติดกันในหน่วยความจำ เข้าถึงด้วย index ได้ O(1)
              addr:1000 1004 1008 1012

Linked List:  [10|●]->[20|●]->[30|●]->[40|null]  <- กระจายอยู่ที่ไหนก็ได้ เชื่อมด้วย pointer
              addr:2000    addr:5000   addr:1200   addr:8800
```

## 2. Singly Linked List: การสร้างและเมธอดพื้นฐาน

**Singly Linked List** แต่ละ node ชี้ไปยัง node**ถัดไป**เพียงทิศทางเดียว
(ทบทวนแนวคิด static nested class จาก Part 17/19)

```java
public class SinglyLinkedList {
    static class Node {
        int value;
        Node next;

        Node(int value) {
            this.value = value;
            this.next = null;
        }
    }

    private Node head; // จุดเริ่มต้นของ list (null หมายถึง list ว่าง)
    private int size;

    public void addFirst(int value) {
        Node newNode = new Node(value);
        newNode.next = head; // node ใหม่ชี้ไปที่ head เดิม
        head = newNode;        // head กลายเป็น node ใหม่นี้
        size++;
    }

    public void addLast(int value) {
        Node newNode = new Node(value);
        if (head == null) {
            head = newNode;
        } else {
            Node current = head;
            while (current.next != null) { // ไล่หา node สุดท้าย
                current = current.next;
            }
            current.next = newNode;
        }
        size++;
    }

    public void printAll() {
        Node current = head;
        StringBuilder sb = new StringBuilder();
        while (current != null) {
            sb.append(current.value).append(" -> ");
            current = current.next;
        }
        sb.append("null");
        System.out.println(sb);
    }

    public int size() { return size; }
}
```

```java
public class SinglyLinkedListDemo {
    public static void main(String[] args) {
        SinglyLinkedList list = new SinglyLinkedList();
        list.addLast(1);
        list.addLast(2);
        list.addLast(3);
        list.addFirst(0);

        list.printAll(); // 0 -> 1 -> 2 -> 3 -> null
        System.out.println("ขนาด: " + list.size()); // 4
    }
}
```

## 3. การแทรกและลบใน Singly Linked List

```java
public class SinglyLinkedListExtended extends SinglyLinkedList {
    // เพิ่มเมธอดค้นหาและลบตาม value
    public boolean remove(int value) {
        if (head == null) return false;

        if (head.value == value) { // ลบ node แรก (กรณีพิเศษ)
            head = head.next;
            size--;
            return true;
        }

        Node current = head;
        while (current.next != null) {
            if (current.next.value == value) {
                current.next = current.next.next; // ข้าม node ที่ต้องการลบไปเลย
                size--;
                return true;
            }
            current = current.next;
        }
        return false; // ไม่เจอค่านี้
    }

    public boolean contains(int value) {
        Node current = head;
        while (current != null) {
            if (current.value == value) return true;
            current = current.next;
        }
        return false;
    }
}
```

```
การลบ node ที่มีค่า 20 จาก list: 10 -> 20 -> 30 -> null

ก่อนลบ:  [10|●]--->[20|●]--->[30|null]
                current    current.next (ต้องการลบตัวนี้)

หลังลบ:  [10|●]------------->[30|null]
         current.next = current.next.next (ข้าม node 20 ไปเลย ไม่มีใครชี้มาที่มันอีก
         -> garbage collector จะเก็บกวาดทิ้งในที่สุด)
```

**Time Complexity ของ Singly Linked List**:

| การดำเนินการ | Time Complexity |
|---|---|
| `addFirst()` | O(1) |
| `addLast()` | O(n) — ต้องไล่หา node สุดท้ายก่อน |
| `remove(value)` | O(n) — ต้องค้นหาก่อน |
| `contains(value)` | O(n) |

## 4. Doubly Linked List

**Doubly Linked List** แต่ละ node มี pointer ชี้ไปทั้ง **node ถัดไป (`next`)**
และ **node ก่อนหน้า (`prev`)** ทำให้เดินย้อนกลับได้ (`Java's LinkedList` ที่
เรียนใน Part 22 คือ doubly linked list จริง ๆ)

```java
public class DoublyLinkedList {
    static class Node {
        int value;
        Node next;
        Node prev;

        Node(int value) {
            this.value = value;
        }
    }

    private Node head;
    private Node tail;
    private int size;

    public void addLast(int value) {
        Node newNode = new Node(value);
        if (head == null) {
            head = tail = newNode;
        } else {
            newNode.prev = tail; // node ใหม่ชี้กลับไปที่ tail เดิม
            tail.next = newNode;  // tail เดิมชี้ไปที่ node ใหม่
            tail = newNode;        // tail กลายเป็น node ใหม่นี้
        }
        size++;
    }

    public void removeLast() {
        if (tail == null) throw new IllegalStateException("List ว่างเปล่า");

        if (head == tail) { // มีแค่ node เดียว
            head = tail = null;
        } else {
            tail = tail.prev;   // tail ใหม่คือ node ก่อนหน้า
            tail.next = null;    // ตัดการเชื่อมต่อไปยัง node เดิมที่ถูกลบ
        }
        size--;
    }

    public void printForward() {
        Node current = head;
        while (current != null) {
            System.out.print(current.value + " -> ");
            current = current.next;
        }
        System.out.println("null");
    }

    public void printBackward() { // ทำได้เพราะมี prev pointer (ทำไม่ได้ใน singly linked list)
        Node current = tail;
        while (current != null) {
            System.out.print(current.value + " -> ");
            current = current.prev;
        }
        System.out.println("null");
    }
}
```

```java
public class DoublyLinkedListDemo {
    public static void main(String[] args) {
        DoublyLinkedList list = new DoublyLinkedList();
        list.addLast(1);
        list.addLast(2);
        list.addLast(3);

        list.printForward();  // 1 -> 2 -> 3 -> null
        list.printBackward(); // 3 -> 2 -> 1 -> null

        list.removeLast();
        list.printForward();  // 1 -> 2 -> null
    }
}
```

**ข้อดีของ Doubly Linked List**: `removeLast()` ทำได้ O(1) (ต่างจาก singly ที่
ต้องไล่หา node ก่อนสุดท้ายก่อน O(n)) และเดินย้อนกลับได้ — **ข้อเสีย**: ใช้
หน่วยความจำมากกว่า (เก็บ pointer เพิ่มอีกตัวต่อ node)

## 5. Circular Linked List

**Circular Linked List** node สุดท้ายชี้**กลับไปที่ node แรก**แทนที่จะชี้ไป
`null` — เหมาะกับสถานการณ์ที่ต้อง**วนซ้ำไปเรื่อย ๆ ไม่มีจุดสิ้นสุด** เช่น
round-robin scheduling, เกมที่ผู้เล่นเวียนตาเล่นวนไปเรื่อย ๆ

```java
public class CircularLinkedList {
    static class Node {
        int value;
        Node next;
        Node(int value) { this.value = value; }
    }

    private Node head;
    private Node tail;

    public void add(int value) {
        Node newNode = new Node(value);
        if (head == null) {
            head = tail = newNode;
            newNode.next = head; // ชี้กลับมาที่ตัวเอง (list ที่มีแค่ 1 node)
        } else {
            tail.next = newNode;
            tail = newNode;
            tail.next = head; // node ล่าสุดชี้กลับไปที่ head เสมอ (นี่คือ "circular")
        }
    }

    public void printNTimes(int n) { // พิมพ์วนไป n รอบ (เพราะไม่มีจุดสิ้นสุดธรรมชาติ)
        if (head == null) return;
        Node current = head;
        for (int i = 0; i < n; i++) {
            System.out.print(current.value + " -> ");
            current = current.next;
        }
        System.out.println("...(วนต่อไปเรื่อย ๆ)");
    }
}
```

```java
public class CircularLinkedListDemo {
    public static void main(String[] args) {
        CircularLinkedList list = new CircularLinkedList();
        list.add(1);
        list.add(2);
        list.add(3);

        list.printNTimes(8); // 1 -> 2 -> 3 -> 1 -> 2 -> 3 -> 1 -> 2 -> ...(วนต่อไปเรื่อย ๆ)
    }
}
```

**ตัวอย่างการใช้งานจริง**: **Round-Robin CPU Scheduling** ในระบบปฏิบัติการ,
เกม "Musical Chairs" หรือ Josephus Problem (โจทย์คลาสสิกที่ผู้เล่นถูกคัดออกวน
ไปรอบ ๆ วงกลม)

## 6. การ Reverse Linked List

โจทย์สัมภาษณ์งานคลาสสิกที่ทดสอบความเข้าใจ pointer manipulation:

```java
public class ReverseLinkedListDemo {
    static class Node {
        int value;
        Node next;
        Node(int value) { this.value = value; }
    }

    static Node reverse(Node head) {
        Node prev = null;
        Node current = head;

        while (current != null) {
            Node next = current.next; // เก็บ next ไว้ก่อน (เดี๋ยวจะถูกเขียนทับ)
            current.next = prev;        // กลับทิศทาง pointer
            prev = current;              // เลื่อน prev ไปข้างหน้า
            current = next;               // เลื่อน current ไปข้างหน้า (ใช้ค่าที่เก็บไว้)
        }
        return prev; // prev คือ head ใหม่ของ list ที่กลับด้านแล้ว
    }

    public static void main(String[] args) {
        Node head = new Node(1);
        head.next = new Node(2);
        head.next.next = new Node(3);
        head.next.next.next = new Node(4);

        Node reversed = reverse(head);

        Node current = reversed;
        while (current != null) {
            System.out.print(current.value + " -> ");
            current = current.next;
        }
        System.out.println("null"); // 4 -> 3 -> 2 -> 1 -> null
    }
}
```

```
Reverse visualization: 1 -> 2 -> 3 -> null

รอบ 1: prev=null, current=1
  next = 2 (เก็บไว้ก่อน)
  1.next = null (กลับทิศทาง: null <- 1)
  prev = 1, current = 2

รอบ 2: prev=1, current=2
  next = 3
  2.next = 1 (กลับทิศทาง: null <- 1 <- 2)
  prev = 2, current = 3

รอบ 3: prev=2, current=3
  next = null
  3.next = 2 (กลับทิศทาง: null <- 1 <- 2 <- 3)
  prev = 3, current = null (loop จบ)

ผลลัพธ์: prev = 3 คือ head ใหม่ -> 3 -> 2 -> 1 -> null
```

**Time Complexity**: O(n), **Space Complexity**: O(1) — ไม่ต้องสร้าง node ใหม่
เลย แค่เปลี่ยนทิศทาง pointer ที่มีอยู่แล้ว

## 7. การหา Cycle ด้วย Floyd's Algorithm (Two Pointers)

**Floyd's Cycle Detection Algorithm** (หรือ "Tortoise and Hare") ใช้ pointer
สองตัวที่เดินด้วยความเร็วต่างกัน (ตัวช้า 1 ก้าว, ตัวเร็ว 2 ก้าว) — ถ้ามี cycle
(node วนกลับมาเชื่อมกันเอง) ทั้งสอง pointer จะมาเจอกันในที่สุด

```java
public class FloydCycleDetectionDemo {
    static class Node {
        int value;
        Node next;
        Node(int value) { this.value = value; }
    }

    static boolean hasCycle(Node head) {
        if (head == null) return false;

        Node slow = head; // เดินทีละ 1 ก้าว ("tortoise")
        Node fast = head;  // เดินทีละ 2 ก้าว ("hare")

        while (fast != null && fast.next != null) {
            slow = slow.next;
            fast = fast.next.next;

            if (slow == fast) { // ถ้าเจอกัน แสดงว่ามี cycle แน่นอน
                return true;
            }
        }
        return false; // fast ไปถึง null ได้ แสดงว่าไม่มี cycle
    }

    public static void main(String[] args) {
        Node head = new Node(1);
        head.next = new Node(2);
        head.next.next = new Node(3);
        head.next.next.next = head.next; // สร้าง cycle: node 3 ชี้กลับไปที่ node 2

        System.out.println(hasCycle(head)); // true

        Node noCycleHead = new Node(1);
        noCycleHead.next = new Node(2);
        System.out.println(hasCycle(noCycleHead)); // false
    }
}
```

**Time Complexity**: O(n), **Space Complexity**: O(1) — ไม่ต้องใช้ `HashSet`
เก็บ node ที่เคยเยี่ยมแล้ว (ซึ่งจะใช้ O(n) space) ทำให้เป็นวิธีที่มีประสิทธิภาพ
ด้านหน่วยความจำที่ดีที่สุด

## 8. การหา Middle Element ด้วย Two Pointers

เทคนิค two pointers ยังใช้หา**จุดกึ่งกลาง**ของ linked list ได้ใน**การผ่านครั้ง
เดียว (single pass)** โดยไม่ต้องนับความยาวก่อน:

```java
public class FindMiddleDemo {
    static class Node {
        int value;
        Node next;
        Node(int value) { this.value = value; }
    }

    static Node findMiddle(Node head) {
        Node slow = head, fast = head;
        while (fast != null && fast.next != null) {
            slow = slow.next;       // เดินทีละ 1 ก้าว
            fast = fast.next.next;   // เดินทีละ 2 ก้าว
        }
        return slow; // เมื่อ fast ถึงปลาย slow จะอยู่ตรงกลางพอดี
    }

    public static void main(String[] args) {
        Node head = new Node(1);
        head.next = new Node(2);
        head.next.next = new Node(3);
        head.next.next.next = new Node(4);
        head.next.next.next.next = new Node(5);

        System.out.println(findMiddle(head).value); // 3 (จุดกึ่งกลางของ 1-2-3-4-5)
    }
}
```

## 9. เปรียบเทียบ Linked List กับ ArrayList (ทบทวนเชิงลึก)

ตอนนี้เราเข้าใจกลไกภายในของ Linked List แล้ว มาทวนความเข้าใจจาก Part 22 อีก
ครั้งด้วยเหตุผลเชิงลึก:

- **ArrayList เข้าถึง index เร็ว O(1)** เพราะคำนวณ memory address ได้ตรง ๆ จาก
  `baseAddress + index * elementSize`
- **LinkedList เข้าถึง index ช้า O(n)** เพราะต้อง**ไล่ node ทีละตัว**จาก head
  (หรือ tail) จนกว่าจะถึงตำแหน่งที่ต้องการ ไม่มีทางคำนวณ address ล่วงหน้าได้
- **LinkedList เพิ่ม/ลบที่หัวเร็ว O(1)** เพราะแค่เปลี่ยน pointer ไม่ต้องเลื่อน
  ข้อมูลใด ๆ เลย ต่างจาก ArrayList ที่ต้องเลื่อนทุก element ไปหนึ่งตำแหน่ง (O(n))

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียนเมธอด `getNthFromEnd(Node head, int n)` ที่หา element ที่ n จาก
ท้าย list โดยผ่าน list **ครั้งเดียว**เท่านั้น (ใช้เทคนิค two pointers)

**เฉลย:**

```java
public class Exercise1 {
    static class Node {
        int value;
        Node next;
        Node(int value) { this.value = value; }
    }

    static Node getNthFromEnd(Node head, int n) {
        Node slow = head, fast = head;
        for (int i = 0; i < n; i++) { // เดิน fast ไปก่อน n ก้าว
            if (fast == null) return null;
            fast = fast.next;
        }
        while (fast != null) { // เดินทั้งคู่พร้อมกัน จนกว่า fast ถึงปลาย
            slow = slow.next;
            fast = fast.next;
        }
        return slow; // slow จะอยู่ที่ตำแหน่ง n จากท้ายพอดี
    }
}
```

**2)** เขียนเมธอด `removeDuplicates(Node head)` ที่ลบค่าซ้ำใน sorted singly
linked list

**เฉลย:**

```java
public class Exercise2 {
    static class Node {
        int value;
        Node next;
        Node(int value) { this.value = value; }
    }

    static void removeDuplicates(Node head) {
        Node current = head;
        while (current != null && current.next != null) {
            if (current.value == current.next.value) {
                current.next = current.next.next; // ข้าม node ที่ซ้ำไปเลย
            } else {
                current = current.next;
            }
        }
    }
}
```

**3)** อธิบายว่าทำไม Floyd's Cycle Detection ใช้หน่วยความจำน้อยกว่าการใช้
`HashSet` เก็บ node ที่เคยเยี่ยมแล้ว

**เฉลย**: Floyd's Algorithm ใช้เพียง 2 ตัวแปร (pointer `slow` และ `fast`)
ไม่ว่า list จะมีขนาดใหญ่แค่ไหน จึงมี Space Complexity O(1) คงที่ ในขณะที่วิธี
ใช้ `HashSet` ต้องเก็บ reference ของทุก node ที่เยี่ยมแล้วไว้ใน set เพื่อเช็ค
ว่าเคยเยี่ยมหรือไม่ ทำให้ใช้หน่วยความจำเพิ่มขึ้นตามจำนวน node (O(n)) — Floyd's
Algorithm จึงเป็นวิธีที่มีประสิทธิภาพด้านหน่วยความจำเหนือกว่ามาก โดยเฉพาะกับ
list ขนาดใหญ่มาก

### สรุปเนื้อหา Part 32

- Linked List เก็บ node กระจายในหน่วยความจำ เชื่อมด้วย pointer ต่างจาก array
  ที่เก็บต่อเนื่องกัน
- Singly Linked List ชี้ทางเดียว (`next`), Doubly Linked List ชี้สองทาง
  (`next`+`prev`), Circular Linked List วนกลับมาที่ head
- Reverse linked list ทำได้ใน O(n) time, O(1) space โดยกลับทิศทาง pointer
  ทีละตัว
- Floyd's Cycle Detection (two pointers ความเร็วต่างกัน) หา cycle ได้ใน O(n)
  time, O(1) space
- Two pointers technique ใช้หาจุดกึ่งกลาง, ตำแหน่งที่ n จากท้าย, และตรวจ cycle
  ได้ทั้งหมด

**ต่อไป**: [Part 33 — Tree: Binary Tree, Binary Search Tree, Tree Traversal](./part-033-trees.md)
