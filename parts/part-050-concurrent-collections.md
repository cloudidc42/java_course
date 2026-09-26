# Part 50: Concurrent Collections: ConcurrentHashMap, CopyOnWriteArrayList, Atomic

> ขั้นตอนที่ 491-500 ของหลักสูตร | ระดับ: สูง (จบหมวด Concurrency พื้นฐานถึงขั้นสูง)

## สารบัญ

1. ทำไม Collections ปกติไม่ Thread-safe
2. `ConcurrentHashMap`
3. `CopyOnWriteArrayList`
4. `BlockingQueue`: Producer-Consumer แบบสำเร็จรูป
5. Atomic Classes: `AtomicInteger`, `AtomicLong`, `AtomicReference`
6. `Collections.synchronizedXxx()` เทียบกับ Concurrent Collections
7. `ConcurrentSkipListMap`/`ConcurrentSkipListSet`
8. เปรียบเทียบ Concurrent Collections ทั้งหมด
9. หลักปฏิบัติในการเขียนโค้ด Concurrent ที่ปลอดภัย
10. แบบฝึกหัดและสรุปหมวด Concurrency

---

## 1. ทำไม Collections ปกติไม่ Thread-safe

ทบทวนจาก Part 22-24, 46: `ArrayList`, `HashMap`, `HashSet` **ไม่ thread-safe**
โดยการออกแบบ (เพื่อประสิทธิภาพสูงสุดในกรณี single-thread ซึ่งเป็นกรณีส่วนใหญ่)
— การใช้กับหลาย thread พร้อมกันโดยไม่มีการป้องกัน อาจทำให้เกิด
`ConcurrentModificationException` หรือข้อมูลเสียหาย (corrupted state)

**`java.util.concurrent`** package (Java 5+) มี collection ที่ออกแบบมาให้
**thread-safe และมีประสิทธิภาพสูงในสถานการณ์ concurrent** โดยเฉพาะ

## 2. `ConcurrentHashMap`

**`ConcurrentHashMap`** เป็นทางเลือก thread-safe ของ `HashMap` (ทบทวนจาก
Part 24) ที่มีประสิทธิภาพสูงกว่าการใช้ `synchronized` ครอบทั้ง `HashMap`
เพราะใช้เทคนิค**การแบ่ง lock เป็นส่วน ๆ (lock striping)** — หลาย thread
เขียนไปยัง**ส่วนต่างกัน**ของ map ได้พร้อมกันโดยไม่ต้องรอกัน

```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.Map;

public class ConcurrentHashMapDemo {
    public static void main(String[] args) throws InterruptedException {
        Map<String, Integer> counts = new ConcurrentHashMap<>();

        Runnable task = () -> {
            for (int i = 0; i < 1000; i++) {
                // merge() เป็น atomic operation ที่ปลอดภัยสำหรับการอัปเดตพร้อมกัน (ทบทวนจาก Part 24)
                counts.merge("counter", 1, Integer::sum);
            }
        };

        Thread t1 = new Thread(task);
        Thread t2 = new Thread(task);
        t1.start();
        t2.start();
        t1.join();
        t2.join();

        System.out.println(counts.get("counter")); // 2000 เสมอ - ถูกต้องและปลอดภัย
    }
}
```

**ข้อควรระวังสำคัญ**: แม้ `ConcurrentHashMap` thread-safe ในระดับ operation
เดี่ยว (เช่น `put`, `get`, `merge`) แต่**ไม่รับประกัน atomicity ข้ามหลาย
operation** — โค้ดแบบนี้ยังเสี่ยง race condition:

```java
import java.util.concurrent.ConcurrentHashMap;

public class ConcurrentHashMapPitfallDemo {
    public static void main(String[] args) {
        ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
        map.put("key", 10);

        // ผิด: get() และ put() เป็นคนละ operation - อาจมี thread อื่นแก้ไขค่าระหว่างสองขั้นตอนนี้!
        int value = map.get("key");
        map.put("key", value + 1); // race condition ยังเกิดได้!

        // ถูก: ใช้ atomic method ที่ทำ get+update+put ในขั้นตอนเดียว (atomic operation)
        map.compute("key", (k, v) -> v + 1); // ทบทวนจาก Part 24 - ปลอดภัยจริง
    }
}
```

## 3. `CopyOnWriteArrayList`

**`CopyOnWriteArrayList`** ทำงานโดย**สร้าง array ใหม่ทั้งหมดทุกครั้งที่มีการ
แก้ไข** (add/remove/set) — การ**อ่าน (iterate)** ไม่ต้อง lock เลยและปลอดภัย
100% เพราะทำงานกับ snapshot ของ array ณ ตอนที่เริ่ม iterate

```java
import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

public class CopyOnWriteArrayListDemo {
    public static void main(String[] args) {
        List<String> list = new CopyOnWriteArrayList<>(List.of("A", "B", "C"));

        // ปลอดภัย! ไม่เกิด ConcurrentModificationException แม้แก้ไข list ระหว่างวน for-each
        // (ทบทวนปัญหาจาก Part 22) เพราะ for-each ทำงานกับ snapshot ของ array ตอนเริ่ม iterate
        for (String item : list) {
            System.out.println(item);
            if (item.equals("B")) {
                list.add("D"); // แก้ไขระหว่าง iterate ได้โดยไม่ throw exception
            }
        }
        System.out.println(list); // [A, B, C, D]
    }
}
```

**ข้อจำกัดสำคัญ**: การเขียน (write) มี **cost สูงมาก O(n)** เพราะต้องคัดลอก
array ทั้งหมดทุกครั้ง — **เหมาะกับสถานการณ์ที่อ่านบ่อยมาก แต่เขียนน้อยมาก**
(read-heavy, write-rare) เช่น **listener list** ที่เปลี่ยนแปลงไม่บ่อยแต่ถูก
วนอ่านบ่อยมาก — **ไม่เหมาะกับ list ที่เขียนบ่อย** (จะช้ากว่า
`Collections.synchronizedList()` มาก)

## 4. `BlockingQueue`: Producer-Consumer แบบสำเร็จรูป

ทบทวนจาก Part 47: เราเขียน Producer-Consumer ด้วย `wait()`/`notify()` เอง
ซึ่งซับซ้อนและเสี่ยง bug — **`BlockingQueue`** ทำให้ pattern นี้**ง่ายขึ้นมาก**
โดยจัดการ wait/notify ให้อัตโนมัติภายใน

```java
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;

public class BlockingQueueDemo {
    public static void main(String[] args) throws InterruptedException {
        BlockingQueue<Integer> queue = new LinkedBlockingQueue<>(5); // capacity=5

        Thread producer = new Thread(() -> {
            try {
                for (int i = 1; i <= 10; i++) {
                    queue.put(i); // ถ้า queue เต็ม จะ "รอ" อัตโนมัติ (ไม่ต้องเขียน wait() เอง!)
                    System.out.println("Produced: " + i);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        Thread consumer = new Thread(() -> {
            try {
                for (int i = 1; i <= 10; i++) {
                    int value = queue.take(); // ถ้า queue ว่าง จะ "รอ" อัตโนมัติ
                    System.out.println("Consumed: " + value);
                    Thread.sleep(200);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        producer.start();
        consumer.start();
        producer.join();
        consumer.join();
    }
}
```

**เปรียบเทียบกับ Part 47**: โค้ดนี้**สั้นกว่ามาก**และ**ปลอดภัยกว่า** เพราะไม่
ต้องจัดการ `synchronized`, `wait()`, `notify()` ด้วยตัวเอง — `put()`/`take()`
จัดการทุกอย่างให้อัตโนมัติ นี่คือเหตุผลที่ในโค้ด production **ไม่ควรเขียน
Producer-Consumer ด้วย `wait()`/`notify()` เอง** ควรใช้ `BlockingQueue` เสมอ

ชนิดของ `BlockingQueue` ที่ใช้บ่อย: `LinkedBlockingQueue` (ขนาดปรับได้หรือ
กำหนด capacity), `ArrayBlockingQueue` (ขนาดคงที่), `PriorityBlockingQueue`
(เรียงตามความสำคัญ — ทบทวน `PriorityQueue` จาก Part 25)

## 5. Atomic Classes: `AtomicInteger`, `AtomicLong`, `AtomicReference`

**Atomic classes** ใน `java.util.concurrent.atomic` ใช้เทคนิค **CAS
(Compare-And-Swap)** ระดับ hardware ทำให้ operation พื้นฐาน (increment,
update) เป็น **atomic โดยไม่ต้องใช้ lock เลย** — เร็วกว่า `synchronized`
สำหรับ operation ง่าย ๆ บนตัวแปรเดี่ยว

```java
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicReference;

public class AtomicDemo {
    static AtomicInteger counter = new AtomicInteger(0);

    public static void main(String[] args) throws InterruptedException {
        Thread[] threads = new Thread[10];
        for (int i = 0; i < 10; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 1000; j++) {
                    counter.incrementAndGet(); // atomic - ไม่มี race condition แม้ไม่มี lock ใด ๆ
                }
            });
            threads[i].start();
        }
        for (Thread t : threads) t.join();

        System.out.println(counter.get()); // 10000 เสมอ

        // เมธอดสำคัญอื่น ๆ ของ AtomicInteger
        AtomicInteger num = new AtomicInteger(10);
        System.out.println(num.getAndAdd(5));     // 10 (คืนค่าเดิมก่อนบวก) -> ตอนนี้ num=15
        System.out.println(num.addAndGet(5));       // 20 (คืนค่าหลังบวก)
        System.out.println(num.compareAndSet(20, 100)); // true (ถ้าค่าปัจจุบัน==20 ให้เปลี่ยนเป็น 100)
        System.out.println(num.get());                // 100

        // AtomicReference: ใช้กับ object type ใดก็ได้
        AtomicReference<String> ref = new AtomicReference<>("initial");
        ref.compareAndSet("initial", "updated");
        System.out.println(ref.get()); // "updated"
    }
}
```

**หลักการทำงานของ CAS (Compare-And-Swap)**: เป็นคำสั่งระดับ CPU ที่ทำ**"เทียบ
และแทนที่"ในขั้นตอนเดียวแบบ atomic** — ถ้าค่าปัจจุบันตรงกับที่คาดไว้ ให้
เปลี่ยนเป็นค่าใหม่ ถ้าไม่ตรง (เพราะ thread อื่นเปลี่ยนไปแล้ว) ให้**ลองใหม่**
(retry) — ไม่ต้อง block thread เลย (เรียกว่า **lock-free algorithm**)

## 6. `Collections.synchronizedXxx()` เทียบกับ Concurrent Collections

Java มี utility เก่าที่**ห่อ collection ปกติด้วย synchronized wrapper**:

```java
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

public class SynchronizedWrapperDemo {
    public static void main(String[] args) {
        // แบบเก่า: ห่อด้วย synchronized wrapper (ล็อคทั้ง object ทุก operation)
        Map<String, Integer> syncMap = Collections.synchronizedMap(new HashMap<>());
        List<String> syncList = Collections.synchronizedList(new ArrayList<>());

        // แบบใหม่ (แนะนำ): ใช้ concurrent collection โดยตรง (lock striping - เร็วกว่ามาก)
        Map<String, Integer> concurrentMap = new ConcurrentHashMap<>();

        // ข้อควรระวัง: การ iterate synchronizedXxx ต้อง synchronized เองด้วยตนเอง!
        synchronized (syncList) { // ต้องล็อคด้วยมือตอน iterate ไม่งั้นเสี่ยง ConcurrentModificationException
            for (String item : syncList) {
                System.out.println(item);
            }
        }
    }
}
```

| ลักษณะ | `Collections.synchronizedXxx()` | Concurrent Collections |
|---|---|---|
| กลไก | ล็อคทั้ง object (coarse-grained lock) | Lock striping / lock-free (fine-grained) |
| ประสิทธิภาพ | ต่ำกว่าเมื่อมีหลาย thread แข่งกัน | สูงกว่ามากในสถานการณ์ concurrent จริง |
| Iteration ปลอดภัย | ต้อง synchronized ด้วยมือเอง | ปลอดภัยในตัว (weakly consistent iterator) |
| แนะนำในโค้ดใหม่ | ไม่ค่อยแนะนำ (legacy) | **แนะนำ** |

## 7. `ConcurrentSkipListMap`/`ConcurrentSkipListSet`

เทียบเท่า `TreeMap`/`TreeSet` (ทบทวนจาก Part 23-24) แต่**thread-safe** — ใช้
โครงสร้างข้อมูล **Skip List** (ทางเลือกที่มีประสิทธิภาพเทียบเคียง balanced
tree แต่ implement ง่ายกว่าในสถานการณ์ concurrent)

```java
import java.util.concurrent.ConcurrentSkipListMap;

public class ConcurrentSkipListDemo {
    public static void main(String[] args) {
        ConcurrentSkipListMap<Integer, String> map = new ConcurrentSkipListMap<>();
        map.put(5, "five");
        map.put(1, "one");
        map.put(3, "three");

        System.out.println(map); // {1=one, 3=three, 5=five} - เรียงลำดับอัตโนมัติเหมือน TreeMap
        System.out.println(map.firstKey()); // 1
        System.out.println(map.lastKey());   // 5
        // ทั้งหมดนี้ thread-safe โดยไม่ต้อง synchronized เอง
    }
}
```

## 8. เปรียบเทียบ Concurrent Collections ทั้งหมด

| Collection | เทียบเท่ากับ | เหมาะกับ |
|---|---|---|
| `ConcurrentHashMap` | `HashMap` | Map ที่อ่าน-เขียนบ่อยพร้อมกันหลาย thread |
| `CopyOnWriteArrayList` | `ArrayList` | List ที่อ่านบ่อยมาก เขียนน้อยมาก |
| `ConcurrentSkipListMap/Set` | `TreeMap`/`TreeSet` | ต้องการข้อมูลเรียงลำดับ + thread-safe |
| `LinkedBlockingQueue` ฯลฯ | `Queue`/`Deque` | Producer-Consumer pattern |
| `AtomicInteger` ฯลฯ | `int`/`long` แบบ shared | ตัวแปรเดี่ยวที่หลาย thread แก้ไขบ่อย |

## 9. หลักปฏิบัติในการเขียนโค้ด Concurrent ที่ปลอดภัย

1. **หลีกเลี่ยง shared mutable state ถ้าเป็นไปได้** — วิธีที่ปลอดภัยที่สุดคือ
   ไม่มี state ที่แชร์กันเลย (immutable objects — ทบทวนจาก Part 17)
2. **ใช้ concurrent collection ที่เหมาะสมกับ pattern การใช้งาน** ตามตารางใน
   หัวข้อ 8 แทนการ synchronized เอง
3. **ใช้ high-level abstraction เสมอ** (`ExecutorService`, `CompletableFuture`,
   `BlockingQueue`) แทนการจัดการ `Thread`/`synchronized`/`wait`/`notify` ด้วยมือ
   ถ้าไม่มีเหตุผลจำเป็นจริง ๆ
4. **ทดสอบด้วยเครื่องมือเฉพาะทาง**: race condition มักไม่แสดงตัวใน unit test
   ปกติ (Part 58) — ควรใช้เครื่องมือเช่น stress testing หรือ static analysis
   tool ที่ตรวจจับปัญหา concurrency ได้

## 10. แบบฝึกหัดและสรุปหมวด Concurrency

### แบบฝึกหัด

**1)** เขียนโปรแกรมนับจำนวนคำในหลายไฟล์พร้อมกัน (ใช้ `ExecutorService`) แล้ว
รวมผลลัพธ์ด้วย `ConcurrentHashMap` และ `AtomicInteger`

**เฉลย (แนวคิด):**

```java
import java.util.List;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class Exercise1 {
    public static void main(String[] args) throws Exception {
        List<String> texts = List.of("hello world", "hello java", "world of java");
        ConcurrentHashMap<String, Integer> wordCounts = new ConcurrentHashMap<>();
        AtomicInteger totalWords = new AtomicInteger(0);

        ExecutorService executor = Executors.newFixedThreadPool(3);
        List<Future<?>> futures = texts.stream().map(text -> executor.submit(() -> {
            for (String word : text.split(" ")) {
                wordCounts.merge(word, 1, Integer::sum);
                totalWords.incrementAndGet();
            }
        })).toList();

        for (Future<?> f : futures) f.get();
        executor.shutdown();

        System.out.println(wordCounts);       // {hello=2, world=2, java=2, of=1}
        System.out.println(totalWords.get()); // 7
    }
}
```

**2)** อธิบายว่าทำไม `CopyOnWriteArrayList` ไม่เหมาะกับสถานการณ์ที่เขียนบ่อย

**เฉลย**: `CopyOnWriteArrayList` **คัดลอก array ทั้งหมดใหม่ทุกครั้ง**ที่มีการ
แก้ไข (add, remove, set) ทำให้แต่ละ write มี Time Complexity O(n) (ทบทวนแนวคิด
Big O จาก Part 35) — ถ้าเขียนบ่อยมากบน list ขนาดใหญ่ จะเสีย performance และ
หน่วยความจำอย่างมาก (สร้าง garbage จำนวนมากให้ GC ต้องเก็บกวาด) เหมาะกับกรณีที่
**อ่านบ่อยมากแต่เขียนน้อยมาก**เท่านั้น เช่น listener list ที่ subscribe/
unsubscribe ไม่บ่อย แต่ต้อง notify (อ่าน+วน loop) ทุกครั้งที่มี event เกิดขึ้น

**3)** อธิบายว่า `AtomicInteger.incrementAndGet()` ทำงานอย่างไรโดยไม่ต้องใช้
`synchronized`

**เฉลย**: `AtomicInteger` ใช้เทคนิค **CAS (Compare-And-Swap)** ซึ่งเป็นคำสั่ง
ระดับ CPU hardware ที่ทำงานแบบ atomic ในตัวเอง — เมื่อเรียก
`incrementAndGet()` มันจะ: (1) อ่านค่าปัจจุบัน (2) คำนวณค่าใหม่ (บวก 1) (3)
พยายามเขียนค่าใหม่กลับไปด้วยคำสั่ง CAS ที่เช็คว่าค่าปัจจุบันยังตรงกับที่อ่านไว้
หรือไม่ ถ้าตรง (ไม่มี thread อื่นมาแก้ไขระหว่างทาง) การเขียนจะสำเร็จทันที ถ้า
ไม่ตรง (มี thread อื่นแก้ไขไปแล้ว) จะ**วนลูปลองใหม่**โดยอ่านค่าล่าสุดแล้วคำนวณ
ใหม่ — ทั้งกระบวนการนี้ไม่ต้อง block thread ใด ๆ เลย (lock-free) ทำให้เร็วกว่า
`synchronized` มากในสถานการณ์ที่การแข่งกันของ thread ไม่รุนแรงเกินไป

### สรุปเนื้อหา Part 50 และหมวด Concurrency (Part 46-50)

- `ConcurrentHashMap` ใช้ lock striping ให้ประสิทธิภาพสูงกว่า synchronized
  HashMap มาก
- `CopyOnWriteArrayList` ปลอดภัยสำหรับอ่านบ่อย-เขียนน้อย, เขียนมี cost O(n)
- `BlockingQueue` ทำให้ Producer-Consumer pattern ง่ายและปลอดภัยกว่าการเขียน
  wait/notify เอง
- Atomic classes ใช้ CAS ทำ operation แบบ lock-free เร็วกว่า synchronized
  สำหรับตัวแปรเดี่ยว
- ควรใช้ concurrent collection และ high-level abstraction (Executor,
  CompletableFuture, BlockingQueue) แทนการจัดการ thread ด้วยมือเสมอ

**จบหมวด Concurrency (Part 46-50) อย่างสมบูรณ์!** ครอบคลุมตั้งแต่ Thread
พื้นฐาน, synchronized/Lock, Executor Framework, CompletableFuture, ไปจนถึง
Concurrent Collections — เนื้อหาระดับสูงที่สำคัญมากสำหรับงาน backend/
enterprise Java

**ต่อไป**: [Part 51 — Records, Sealed Classes, Pattern Matching (Java 17-21)](./part-051-records-sealed-classes.md)
