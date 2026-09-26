# Part 47: Multithreading ขั้นสูง: synchronized, wait/notify, Lock, Deadlock

> ขั้นตอนที่ 461-470 ของหลักสูตร | ระดับ: สูง

## สารบัญ

1. `synchronized` Keyword: แก้ปัญหา Race Condition
2. `synchronized` บน Method vs Block
3. Intrinsic Lock (Monitor Lock) ทำงานอย่างไร
4. `wait()`, `notify()`, `notifyAll()`
5. Producer-Consumer Pattern
6. `java.util.concurrent.locks.Lock` และ `ReentrantLock`
7. `volatile` Keyword
8. Deadlock: สาเหตุและการป้องกัน
9. Livelock และ Starvation
10. แบบฝึกหัดและสรุป

---

## 1. `synchronized` Keyword: แก้ปัญหา Race Condition

ทบทวนปัญหาจาก Part 46: **`synchronized`** ทำให้เฉพาะ**thread เดียวเท่านั้น**
เข้าถึง code block ที่กำกับไว้ได้ในเวลาเดียวกัน (mutual exclusion) แก้ปัญหา
race condition ได้

```java
public class SynchronizedFixDemo {
    static int counter = 0;

    static synchronized void increment() { // synchronized method: ล็อคทั้งเมธอด
        counter++;
    }

    public static void main(String[] args) throws InterruptedException {
        Thread[] threads = new Thread[10];
        for (int i = 0; i < 10; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 1000; j++) {
                    increment();
                }
            });
            threads[i].start();
        }
        for (Thread t : threads) {
            t.join();
        }
        System.out.println("ผลลัพธ์: " + counter); // 10000 เสมอ (ถูกต้องแน่นอนแล้ว!)
    }
}
```

**หลักการทำงาน**: `synchronized` ใช้ **monitor lock** (หรือ **intrinsic
lock**) ที่ทุก object ใน Java มีอยู่แล้วโดยกำเนิด — เมื่อ thread หนึ่งเข้าไป
ใน synchronized block/method จะ**ยึด lock** ไว้ thread อื่นที่ต้องการเข้า
synchronized block/method **เดียวกัน**ต้อง**รอ**จนกว่า lock จะถูกปล่อย

## 2. `synchronized` บน Method vs Block

```java
public class SynchronizedTypesDemo {
    private int balance = 1000;
    private final Object lock = new Object(); // object เปล่า ๆ ที่ใช้เป็น lock โดยเฉพาะ

    // synchronized method: ล็อคทั้งเมธอด ใช้ "this" เป็น lock โดยปริยาย
    public synchronized void withdraw(int amount) {
        if (balance >= amount) {
            balance -= amount;
        }
    }

    // synchronized block: ล็อคเฉพาะส่วนที่จำเป็น (แนะนำมากกว่า - ล็อคน้อยที่สุดที่จำเป็น)
    public void deposit(int amount) {
        System.out.println("กำลังตรวจสอบข้อมูล..."); // ส่วนนี้ไม่ต้องล็อค (ไม่แก้ไข shared state)
        synchronized (lock) { // ล็อคแค่ส่วนที่แก้ไขข้อมูลร่วมกันเท่านั้น
            balance += amount;
        }
        System.out.println("บันทึก log..."); // ส่วนนี้ก็ไม่ต้องล็อคเช่นกัน
    }

    // static synchronized method: ใช้ Class object (SynchronizedTypesDemo.class) เป็น lock
    public static synchronized void staticMethod() {
        System.out.println("static synchronized method");
    }
}
```

**หลักปฏิบัติที่ดี**: **ล็อคเฉพาะส่วนที่จำเป็นเท่านั้น (minimize critical
section)** — ยิ่งล็อคนานเท่าไร ยิ่งลดประสิทธิภาพของการทำงานแบบขนาน (thread
อื่นต้องรอนานขึ้น) ควรใช้ `synchronized` block แทน method ทั้งตัวเมื่อมีแค่
บางส่วนที่ต้องป้องกัน

## 3. Intrinsic Lock (Monitor Lock) ทำงานอย่างไร

```java
public class IntrinsicLockDemo {
    public static void main(String[] args) throws InterruptedException {
        Object sharedLock = new Object();

        Runnable task = () -> {
            synchronized (sharedLock) { // ต้องยึด lock ของ sharedLock ก่อนเข้ามาในนี้ได้
                System.out.println(Thread.currentThread().getName() + " เข้าครอบครอง lock");
                try {
                    Thread.sleep(1000); // จำลองการทำงานที่ใช้เวลา (ยัง"ถือ"lock ไว้ตลอดช่วงนี้)
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                System.out.println(Thread.currentThread().getName() + " ปล่อย lock");
            } // lock ถูกปล่อยอัตโนมัติเมื่อออกจาก block (แม้เกิด exception ก็ปล่อยเสมอ - เหมือน try-finally)
        };

        Thread t1 = new Thread(task, "Thread-A");
        Thread t2 = new Thread(task, "Thread-B");
        t1.start();
        t2.start();
        // ผลลัพธ์: Thread-A เข้าครอบครอง lock ก่อน (หรือ B ก็ได้ ไม่แน่นอน) แล้วอีก thread ต้องรอ
        // จนกว่า thread แรกจะปล่อย lock ก่อนถึงจะเข้าไปทำงานต่อได้
    }
}
```

**ข้อสำคัญ**: lock ถูก**ปล่อยอัตโนมัติ**เมื่อออกจาก synchronized block **ไม่
ว่าจะสำเร็จหรือเกิด exception ก็ตาม** (คล้ายกับการทำงานของ try-with-resources
จาก Part 21) — นี่คือข้อดีสำคัญที่ทำให้ `synchronized` ปลอดภัยกว่าการจัดการ
lock ด้วยมือ

## 4. `wait()`, `notify()`, `notifyAll()`

**`wait()`/`notify()`/`notifyAll()`** เป็นเมธอดของ `Object` (ทุก object มี
เมธอดนี้อยู่แล้ว — ทบทวนจาก Part 14) ใช้ให้ thread**"รอ"อย่างมีประสิทธิภาพ**
จนกว่าจะมีสัญญาณบอกให้ทำงานต่อ (ต่างจากการวน loop เช็คเงื่อนไขไปเรื่อย ๆ ซึ่ง
สิ้นเปลือง CPU มาก — เรียกว่า "busy waiting")

```java
public class WaitNotifyDemo {
    private static boolean dataReady = false;
    private static final Object lock = new Object();

    static void producer() {
        synchronized (lock) {
            System.out.println("Producer: กำลังเตรียมข้อมูล...");
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) { }
            dataReady = true;
            System.out.println("Producer: ข้อมูลพร้อมแล้ว! แจ้ง consumer");
            lock.notify(); // ปลุก thread ที่กำลัง wait() อยู่บน lock นี้ให้ทำงานต่อ
        }
    }

    static void consumer() {
        synchronized (lock) {
            while (!dataReady) {
                try {
                    System.out.println("Consumer: ข้อมูลยังไม่พร้อม รอก่อน...");
                    lock.wait(); // ปล่อย lock ชั่วคราวและ "หลับ" จนกว่าจะถูก notify()
                                  // (สำคัญมาก: wait() ปล่อย lock ให้ thread อื่นใช้ได้ ต่างจาก sleep())
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
            System.out.println("Consumer: ได้รับข้อมูลแล้ว ทำงานต่อได้!");
        }
    }

    public static void main(String[] args) {
        new Thread(WaitNotifyDemo::consumer).start();
        new Thread(WaitNotifyDemo::producer).start();
    }
}
```

**ข้อสำคัญที่ต้องจำ**:
- `wait()`, `notify()`, `notifyAll()` **ต้องเรียกภายใน synchronized block ของ
  lock เดียวกันเท่านั้น** (ไม่งั้น throw `IllegalMonitorStateException`)
- ใช้ `while` (ไม่ใช่ `if`) เช็คเงื่อนไขก่อน `wait()` เสมอ เพื่อป้องกัน
  **"spurious wakeup"** (thread ถูกปลุกโดยไม่มีเหตุผลชัดเจน — เป็นพฤติกรรมที่
  JVM อนุญาตได้ตามสเปก)
- `notify()` ปลุก thread ที่ wait() อยู่**เพียง 1 ตัว**, `notifyAll()` ปลุก
  **ทุกตัว** — ควรใช้ `notifyAll()` เป็นค่าเริ่มต้นเว้นแต่มั่นใจจริง ๆ ว่า
  `notify()` ปลอดภัยในสถานการณ์นั้น

## 5. Producer-Consumer Pattern

**Producer-Consumer** เป็น pattern คลาสสิกที่ใช้ `wait()`/`notify()` (หรือ
`BlockingQueue` ที่ Part 48-50 จะแนะนำซึ่งง่ายกว่ามาก) — producer สร้างข้อมูล
ใส่ buffer, consumer ดึงข้อมูลออกมาใช้ ทั้งสองทำงานพร้อมกันโดยไม่บล็อกกันแบบ
ไม่จำเป็น

```java
import java.util.LinkedList;
import java.util.Queue;

public class ProducerConsumerDemo {
    static final int CAPACITY = 5;
    static Queue<Integer> buffer = new LinkedList<>();
    static final Object lock = new Object();

    static void produce() throws InterruptedException {
        int value = 0;
        while (true) {
            synchronized (lock) {
                while (buffer.size() == CAPACITY) { // buffer เต็ม - รอให้ consumer เอาออกก่อน
                    lock.wait();
                }
                buffer.add(value);
                System.out.println("Produced: " + value);
                value++;
                lock.notifyAll(); // แจ้ง consumer ว่ามีข้อมูลใหม่แล้ว
            }
            Thread.sleep(200);
        }
    }

    static void consume() throws InterruptedException {
        while (true) {
            synchronized (lock) {
                while (buffer.isEmpty()) { // buffer ว่าง - รอให้ producer เติมก่อน
                    lock.wait();
                }
                int value = buffer.poll();
                System.out.println("Consumed: " + value);
                lock.notifyAll(); // แจ้ง producer ว่ามีที่ว่างแล้ว
            }
            Thread.sleep(500);
        }
    }
}
```

## 6. `java.util.concurrent.locks.Lock` และ `ReentrantLock`

**`Lock`** (ตั้งแต่ Java 5) เป็นทางเลือกที่**ยืดหยุ่นกว่า** `synchronized`:
รองรับการ**พยายามยึด lock แบบมี timeout**, ตรวจสอบว่ามี thread รออยู่หรือไม่,
และปลด lock ได้จากคนละเมธอดกับที่ยึด (ต่างจาก `synchronized` ที่ต้องยึด-ปลด
ใน scope เดียวกันเสมอ)

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class ReentrantLockDemo {
    private static final Lock lock = new ReentrantLock();
    private static int counter = 0;

    static void increment() {
        lock.lock(); // ยึด lock (บล็อกจนกว่าจะได้ ถ้าถูกยึดอยู่)
        try {
            counter++;
        } finally {
            lock.unlock(); // **ต้อง**ปลด lock ใน finally เสมอ (ไม่อัตโนมัติแบบ synchronized!)
        }
    }

    static boolean tryIncrement() {
        if (lock.tryLock()) { // พยายามยึด lock ทันที ไม่รอถ้าไม่ได้ (คืนค่า boolean)
            try {
                counter++;
                return true;
            } finally {
                lock.unlock();
            }
        }
        return false; // ยึด lock ไม่ได้ (มีคนอื่นถือไว้อยู่) - ไม่ต้องรอ ทำงานอื่นต่อได้ทันที
    }

    public static void main(String[] args) throws InterruptedException {
        Thread[] threads = new Thread[10];
        for (int i = 0; i < 10; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 1000; j++) increment();
            });
            threads[i].start();
        }
        for (Thread t : threads) t.join();
        System.out.println(counter); // 10000
    }
}
```

**ชื่อ "Reentrant" (เข้าซ้ำได้)**: thread เดียวกันสามารถยึด lock ตัวเดียวกัน
**ซ้ำได้หลายชั้น**โดยไม่ deadlock กับตัวเอง (เช่นเมธอด A ที่ยึด lock เรียก
เมธอด B ที่ยึด lock เดียวกันต่อ) — `synchronized` ก็เป็น reentrant lock
เช่นกันโดยธรรมชาติ

## 7. `volatile` Keyword

**`volatile`** รับประกันว่า**การอ่าน/เขียน field นั้นมองเห็นได้ทันทีข้าม
thread** — ป้องกันปัญหาที่ thread หนึ่งอาจเห็น**ค่าเก่าที่ถูก cache ไว้**ใน
CPU register/cache ของตัวเอง (ไม่ใช่ค่าล่าสุดจาก main memory)

```java
public class VolatileDemo {
    private static volatile boolean running = true; // volatile: บังคับอ่าน/เขียนจาก main memory เสมอ

    public static void main(String[] args) throws InterruptedException {
        Thread worker = new Thread(() -> {
            int count = 0;
            while (running) { // ถ้าไม่มี volatile บาง thread อาจไม่เห็นการเปลี่ยนแปลงของ running เลย!
                count++;
            }
            System.out.println("Worker หยุดทำงานแล้ว, count=" + count);
        });

        worker.start();
        Thread.sleep(100);
        running = false; // สั่งให้ worker หยุด - ด้วย volatile การเปลี่ยนแปลงนี้จะเห็นทันทีจาก worker thread
        worker.join();
    }
}
```

**ข้อจำกัดสำคัญของ `volatile`**: **รับประกันแค่ visibility ไม่ได้รับประกัน
atomicity** — `volatile int counter` แล้วทำ `counter++` **ยังเกิด race
condition ได้เหมือนเดิม** เพราะ `++` ไม่ใช่ operation เดียว (ทบทวนจาก Part
46) `volatile` เหมาะกับตัวแปรที่เป็น**flag แบบอ่าน-เขียนล้วน ๆ** (เช่น
`running` ในตัวอย่างข้างบน) ไม่เหมาะกับตัวแปรที่ต้องคำนวณจากค่าเดิม

## 8. Deadlock: สาเหตุและการป้องกัน

**Deadlock** เกิดเมื่อ**สอง thread (หรือมากกว่า) ต่างรอ lock ที่อีกฝ่ายถืออยู่
พร้อมกัน** ทำให้ทั้งคู่**ค้างตลอดไป** ไม่มีใครทำงานต่อได้

```java
public class DeadlockDemo {
    static final Object lockA = new Object();
    static final Object lockB = new Object();

    static void method1() {
        synchronized (lockA) {
            System.out.println("Thread 1: ยึด lockA");
            try { Thread.sleep(100); } catch (InterruptedException e) { }
            System.out.println("Thread 1: กำลังพยายามยึด lockB...");
            synchronized (lockB) { // รอ lockB ที่ Thread 2 ถืออยู่
                System.out.println("Thread 1: ยึด lockB สำเร็จ");
            }
        }
    }

    static void method2() {
        synchronized (lockB) {
            System.out.println("Thread 2: ยึด lockB");
            try { Thread.sleep(100); } catch (InterruptedException e) { }
            System.out.println("Thread 2: กำลังพยายามยึด lockA...");
            synchronized (lockA) { // รอ lockA ที่ Thread 1 ถืออยู่ -> DEADLOCK!
                System.out.println("Thread 2: ยึด lockA สำเร็จ");
            }
        }
    }

    public static void main(String[] args) {
        new Thread(DeadlockDemo::method1).start();
        new Thread(DeadlockDemo::method2).start();
        // ทั้งสอง thread จะค้างตลอดไป: Thread 1 รอ lockB (ที่ Thread 2 ถือ)
        // ในขณะที่ Thread 2 รอ lockA (ที่ Thread 1 ถือ) - ไม่มีใครยอมปล่อยก่อน
    }
}
```

```
Deadlock visualization:

Thread 1 ──ถือ──> lockA        Thread 2 ──ถือ──> lockB
   │                              │
   └──ต้องการ (รอ)──> lockB       └──ต้องการ (รอ)──> lockA
                          (วนกลับมาที่กัน = deadlock)
```

**วิธีป้องกัน Deadlock**:
1. **กำหนดลำดับการยึด lock ให้เหมือนกันเสมอทุก thread** (เช่น ยึด lockA ก่อน
   lockB เสมอ ไม่มี thread ไหนยึดสลับลำดับ) — วิธีที่ได้ผลที่สุด
2. **ใช้ `tryLock()` แทน `lock()`** เพื่อไม่ให้รอตลอดไป (มี timeout) แล้ว
   ถอยกลับ (backoff) และลองใหม่
3. **หลีกเลี่ยงการยึด lock หลายตัวพร้อมกันถ้าเป็นไปได้**

```java
public class DeadlockPreventionDemo {
    static final Object lockA = new Object();
    static final Object lockB = new Object();

    // แก้ไข: ทั้งสองเมธอดยึด lock ตามลำดับเดียวกันเสมอ (lockA ก่อน lockB เสมอ)
    static void method1() {
        synchronized (lockA) {
            synchronized (lockB) {
                System.out.println("Thread 1: ทำงานสำเร็จ");
            }
        }
    }

    static void method2() {
        synchronized (lockA) { // เปลี่ยนจาก lockB เป็น lockA ให้ตรงลำดับกับ method1
            synchronized (lockB) {
                System.out.println("Thread 2: ทำงานสำเร็จ");
            }
        }
    }
}
```

## 9. Livelock และ Starvation

- **Livelock**: thread ไม่ได้ "ค้าง" แบบ deadlock แต่**ยุ่งอยู่ตลอดโดยไม่มี
  ความก้าวหน้า** (เช่น สอง thread คอยหลีกทางให้กันไปมาไม่จบสิ้น เหมือนคนสอง
  คนเดินสวนกันในทางเดินแคบแล้วต่างขยับหลีกไปทางเดียวกันซ้ำ ๆ)
- **Starvation**: thread หนึ่ง**ไม่ได้รับโอกาสทำงานเลย**เพราะ thread อื่นที่มี
  priority สูงกว่าหรือยึด resource ไว้ตลอดเวลา

```java
public class StarvationExampleDemo {
    // ถ้า thread ที่มี priority ต่ำ ไม่ได้ CPU time เลยเพราะ thread priority สูงยึดไปตลอด
    // นี่คือตัวอย่างแนวคิด starvation (ไม่ได้เขียนโค้ดจำลองเพราะขึ้นกับ OS scheduler มาก)
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** แก้ไข `RaceConditionDemo` จาก Part 46 ให้ถูกต้องโดยใช้ `ReentrantLock`
แทน `synchronized`

**เฉลย:**

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class Exercise1 {
    static int counter = 0;
    static Lock lock = new ReentrantLock();

    static void increment() {
        lock.lock();
        try {
            counter++;
        } finally {
            lock.unlock();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Thread[] threads = new Thread[10];
        for (int i = 0; i < 10; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 1000; j++) increment();
            });
            threads[i].start();
        }
        for (Thread t : threads) t.join();
        System.out.println(counter); // 10000
    }
}
```

**2)** อธิบายความแตกต่างระหว่าง `wait()` กับ `sleep()`

**เฉลย**: `sleep()` เป็น static method ของ `Thread` ที่**ไม่ปล่อย lock ที่ถือ
อยู่** (thread ยังคงยึด lock ไว้ต่อไปแม้จะ sleep) และ sleep เป็นเวลาที่กำหนด
แน่นอน ในขณะที่ `wait()` เป็น instance method ของ `Object` ที่**ปล่อย lock
ชั่วคราว**ให้ thread อื่นใช้ได้ และรอจนกว่าจะถูก `notify()`/`notifyAll()` (หรือ
timeout ถ้าระบุ) `wait()` ต้องเรียกภายใน synchronized block เท่านั้น ส่วน
`sleep()` เรียกที่ไหนก็ได้

**3)** อธิบายว่าทำไม deadlock ใน `DeadlockDemo` แก้ได้ด้วยการเรียงลำดับ lock
ให้เหมือนกัน

**เฉลย**: Deadlock เกิดเมื่อ thread สองตัวยึด lock คนละตัวก่อน แล้วต่างพยายาม
ยึด lock ของอีกฝ่าย (สร้าง cycle ของการรอกัน) — ถ้าทุก thread ยึด lock ตาม
**ลำดับเดียวกันเสมอ** (เช่น lockA ก่อน lockB เสมอ ไม่มีทางสลับ) จะไม่มีทางเกิด
cycle ของการรอได้เลย เพราะ thread ที่ยึด lockA ได้ก่อนจะยึด lockB ต่อได้เสมอ
(ไม่มี thread อื่นที่ยึด lockB ไว้แล้วรอ lockA อยู่)

### สรุปเนื้อหา Part 47

- `synchronized` (method หรือ block) แก้ race condition ด้วย mutual exclusion
  ผ่าน monitor lock ของ object
- `wait()`/`notify()`/`notifyAll()` ให้ thread รออย่างมีประสิทธิภาพ ต้องเรียก
  ใน synchronized block เท่านั้น
- `ReentrantLock` ยืดหยุ่นกว่า `synchronized`: มี `tryLock()`, timeout แต่
  ต้อง `unlock()` ใน `finally` เอง
- `volatile` รับประกัน visibility ข้าม thread แต่ไม่รับประกัน atomicity
- Deadlock เกิดจาก circular wait ของ lock — ป้องกันด้วยการเรียงลำดับการยึด
  lock ให้เหมือนกันเสมอทุก thread

**ต่อไป**: [Part 48 — Executor Framework, Thread Pool, Callable, Future](./part-048-executor-framework.md)
