# Part 46: Multithreading เบื้องต้น: Thread, Runnable

> ขั้นตอนที่ 451-460 ของหลักสูตร | ระดับ: สูง

## สารบัญ

1. Process vs Thread
2. การสร้าง Thread: 2 วิธีหลัก
3. Thread Lifecycle
4. `Thread` เมธอดสำคัญ
5. ปัญหา Race Condition (ทำไมต้องเรียน concurrency ให้เข้าใจจริง)
6. Thread Priority และ Daemon Thread
7. `join()`: รอ Thread ทำงานเสร็จ
8. ปัญหาคลาสสิก: การแชร์ข้อมูลระหว่าง Thread
9. ทำไมไม่ควรสร้าง Thread เองในโค้ดจริง (ปูทางสู่ Part 48 — Executor)
10. แบบฝึกหัดและสรุป

---

## 1. Process vs Thread

**Process** คือโปรแกรมที่กำลังรันอยู่ มีหน่วยความจำของตัวเองแยกจาก process
อื่นโดยสิ้นเชิง — **Thread** คือ**เส้นทางการทำงานย่อย**ภายใน process เดียวกัน
หลาย thread ใน process เดียวกัน**แชร์หน่วยความจำ (heap) ร่วมกัน** แต่มี
**stack ของตัวเอง**แยกกัน (ทบทวนแนวคิด stack/heap จาก Part 17, 35)

```
Process (JVM instance หนึ่งตัว)
┌─────────────────────────────────────────┐
│  Heap (แชร์ร่วมกันทุก thread)              │
│  ┌───────────────────────────────────┐  │
│  │ Object A, Object B, static fields  │  │
│  └───────────────────────────────────┘  │
│                                           │
│  Thread 1        Thread 2       Thread 3 │
│  ┌────────┐     ┌────────┐    ┌────────┐│
│  │ Stack 1 │     │ Stack 2 │    │ Stack 3 ││
│  └────────┘     └────────┘    └────────┘│
└─────────────────────────────────────────┘
```

**Multithreading** คือการรันหลาย thread**พร้อมกัน**ภายใน process เดียว
ประโยชน์คือใช้ประโยชน์จาก CPU หลาย core ได้เต็มที่ และทำงานหลายอย่างพร้อมกัน
โดยไม่บล็อกกัน (เช่น GUI ที่ต้องตอบสนองผู้ใช้ขณะดาวน์โหลดไฟล์ในพื้นหลัง)

## 2. การสร้าง Thread: 2 วิธีหลัก

### วิธีที่ 1: Extends `Thread`

```java
public class MyThread extends Thread {
    @Override
    public void run() { // override run() เพื่อกำหนดงานที่ thread นี้จะทำ
        for (int i = 1; i <= 5; i++) {
            System.out.println(Thread.currentThread().getName() + ": " + i);
        }
    }
}
```

```java
public class ExtendsThreadDemo {
    public static void main(String[] args) {
        MyThread thread = new MyThread();
        thread.start(); // start() เริ่ม thread ใหม่จริง ๆ (ไม่ใช่ run() ตรง ๆ - หัวข้อ 4 อธิบายเหตุผล)
        System.out.println("Main thread ทำงานต่อไปพร้อมกัน");
    }
}
```

### วิธีที่ 2: Implements `Runnable` (แนะนำมากกว่า)

```java
public class MyRunnable implements Runnable {
    @Override
    public void run() {
        for (int i = 1; i <= 5; i++) {
            System.out.println(Thread.currentThread().getName() + ": " + i);
        }
    }
}
```

```java
public class ImplementsRunnableDemo {
    public static void main(String[] args) {
        Thread thread = new Thread(new MyRunnable());
        thread.start();

        // หรือใช้ lambda เพราะ Runnable เป็น functional interface (ทบทวนจาก Part 39-40)
        Thread lambdaThread = new Thread(() -> {
            System.out.println("ทำงานผ่าน lambda");
        });
        lambdaThread.start();
    }
}
```

**ทำไม `Runnable` ดีกว่า extends `Thread`**: Java รองรับ single inheritance
(ทบทวนจาก Part 14) — ถ้า class ของเรา `extends Thread` แล้วจะ**extends class
อื่นไม่ได้อีก** แต่ถ้า `implements Runnable` ยัง extends class อื่นได้ตาม
ปกติ นอกจากนี้ `Runnable` ยังแยก "งานที่ต้องทำ" ออกจาก "กลไกการรัน thread"
อย่างชัดเจน (separation of concerns) ทำให้นำ logic เดียวกันไปใช้กับ
`ExecutorService` (Part 48) ได้ง่ายกว่ามาก

## 3. Thread Lifecycle

```
        new Thread()
             |
             v
          NEW ──start()──> RUNNABLE ◄──────┐
                               |            │ (ได้ CPU กลับมา)
                    (JVM scheduler          │
                     เลือกให้ทำงาน)          │
                               |            │
                               v            │
                          RUNNING ──────────┘
                          /    \
              (รอ I/O,     (เสร็จงาน
               lock,        run() จบ)
               sleep)          |
                  |             v
                  v         TERMINATED
              BLOCKED /
              WAITING /
              TIMED_WAITING
```

```java
public class ThreadLifecycleDemo {
    public static void main(String[] args) throws InterruptedException {
        Thread thread = new Thread(() -> {
            try {
                Thread.sleep(1000); // เข้าสู่สถานะ TIMED_WAITING
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        System.out.println(thread.getState()); // NEW
        thread.start();
        System.out.println(thread.getState()); // RUNNABLE (หรือบางครั้ง TIMED_WAITING ถ้าเริ่ม sleep แล้ว)

        Thread.sleep(100);
        System.out.println(thread.getState()); // TIMED_WAITING (กำลัง sleep อยู่)

        thread.join(); // รอจนกว่า thread จะเสร็จงาน (หัวข้อ 7)
        System.out.println(thread.getState()); // TERMINATED
    }
}
```

## 4. `Thread` เมธอดสำคัญ

```java
public class ThreadMethodsDemo {
    public static void main(String[] args) {
        Thread thread = new Thread(() -> System.out.println("กำลังทำงาน"));

        System.out.println(thread.getName());  // Thread-0 (ชื่อ default)
        thread.setName("MyWorkerThread");
        System.out.println(thread.getName());   // MyWorkerThread

        System.out.println(thread.isAlive());    // false (ยังไม่ start)
        thread.start();
        System.out.println(thread.isAlive());     // อาจ true หรือ false ขึ้นกับว่าทำงานเสร็จหรือยัง

        System.out.println(Thread.currentThread().getName()); // "main" (thread หลักของโปรแกรม)
    }
}
```

**ข้อผิดพลาดคลาสสิก**: **การเรียก `run()` ตรง ๆ ไม่ได้สร้าง thread ใหม่!**
มันแค่เรียกเมธอดธรรมดาบน thread ปัจจุบัน (เหมือนเรียกเมธอดทั่วไป) — **ต้อง
เรียก `start()`** เพื่อให้ JVM สร้าง thread ใหม่จริง ๆ และเรียก `run()` บน
thread นั้นให้อัตโนมัติ

```java
public class RunVsStartDemo {
    public static void main(String[] args) {
        Thread thread = new Thread(() -> {
            System.out.println("ทำงานบน thread: " + Thread.currentThread().getName());
        });

        thread.run();   // ผิด! รันบน "main" thread ตรง ๆ ไม่สร้าง thread ใหม่เลย
        thread.start(); // ถูก! สร้าง thread ใหม่จริง ๆ (แต่ thread เดียวกันเรียก start() ซ้ำไม่ได้ — throw IllegalThreadStateException)
    }
}
```

## 5. ปัญหา Race Condition (ทำไมต้องเรียน Concurrency ให้เข้าใจจริง)

**Race Condition** เกิดเมื่อหลาย thread**เข้าถึงและแก้ไขข้อมูลร่วมกัน**โดยไม่
มีการควบคุมที่เหมาะสม ทำให้ผลลัพธ์**ไม่แน่นอน (non-deterministic)** และผิดพลาด

```java
public class RaceConditionDemo {
    static int counter = 0; // ข้อมูลที่แชร์ร่วมกันระหว่าง thread

    static void increment() {
        counter++; // ดูเหมือน 1 คำสั่ง แต่จริง ๆ แล้วประกอบด้วย 3 ขั้นตอน:
                    // 1. อ่านค่า counter ปัจจุบัน  2. บวก 1  3. เขียนค่ากลับ
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
            t.join(); // รอทุก thread เสร็จก่อน
        }

        System.out.println("ผลลัพธ์: " + counter);
        // คาดว่าจะได้ 10,000 (10 threads x 1000 ครั้ง) แต่มักได้ค่าน้อยกว่านั้น!
        // เพราะ counter++ ไม่ใช่ atomic operation - หลาย thread อาจอ่านค่าเดียวกัน
        // ก่อนที่อีก thread จะเขียนค่าใหม่ทัน ทำให้การ increment บางครั้ง "หายไป"
    }
}
```

```
ตัวอย่างการเกิด Race Condition:

Thread A: อ่าน counter = 5
Thread B: อ่าน counter = 5        <- ทั้งคู่อ่านค่าเดียวกัน!
Thread A: คำนวณ 5 + 1 = 6
Thread B: คำนวณ 5 + 1 = 6
Thread A: เขียน counter = 6
Thread B: เขียน counter = 6        <- ควรจะเป็น 7 แต่กลายเป็น 6! (การ increment ของ B "หายไป")
```

**นี่คือปัญหาพื้นฐานที่สุดของ concurrent programming** — Part 47 จะสอนวิธี
แก้ปัญหานี้ด้วย `synchronized` และ `Lock`

## 6. Thread Priority และ Daemon Thread

```java
public class ThreadPriorityDemo {
    public static void main(String[] args) {
        Thread highPriority = new Thread(() -> System.out.println("high priority"));
        highPriority.setPriority(Thread.MAX_PRIORITY); // 10 - "คำแนะนำ" ให้ scheduler ไม่ใช่การันตี

        Thread lowPriority = new Thread(() -> System.out.println("low priority"));
        lowPriority.setPriority(Thread.MIN_PRIORITY); // 1

        // Daemon Thread: thread ที่ทำงานเบื้องหลัง JVM จะปิดตัวเองได้แม้ daemon thread ยังทำงานไม่เสร็จ
        Thread daemonThread = new Thread(() -> {
            while (true) {
                // งานเบื้องหลัง เช่น garbage collection, monitoring
            }
        });
        daemonThread.setDaemon(true); // ต้องเรียกก่อน start() เท่านั้น
        daemonThread.start();

        System.out.println("main thread จบการทำงาน - JVM จะปิดตัวได้ทันที แม้ daemon thread ยังรันอยู่");
    }
}
```

**ข้อควรรู้**: Thread priority เป็นเพียง**คำแนะนำ**ให้ OS scheduler ไม่ใช่การ
การันตีลำดับการทำงานที่แน่นอน (ขึ้นกับ OS และ JVM implementation) — ไม่ควร
พึ่งพา priority เพื่อควบคุม correctness ของโปรแกรม

## 7. `join()`: รอ Thread ทำงานเสร็จ

```java
public class JoinDemo {
    public static void main(String[] args) throws InterruptedException {
        Thread worker = new Thread(() -> {
            System.out.println("เริ่มทำงาน...");
            try {
                Thread.sleep(2000); // จำลองงานที่ใช้เวลา 2 วินาที
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            System.out.println("ทำงานเสร็จแล้ว!");
        });

        worker.start();
        System.out.println("รอ worker thread ทำงานให้เสร็จ...");
        worker.join(); // main thread หยุดรอที่นี่ จนกว่า worker จะจบ (run() คืนค่า)
        System.out.println("worker เสร็จแล้ว ทำงานต่อได้"); // รันหลัง worker เสร็จเสมอ

        // join(timeout): รอสูงสุดเท่าที่กำหนด ไม่รอตลอดไป
        Thread longTask = new Thread(() -> {
            try { Thread.sleep(5000); } catch (InterruptedException e) { }
        });
        longTask.start();
        longTask.join(1000); // รอสูงสุด 1 วินาที แล้วทำงานต่อไม่ว่า longTask จะเสร็จหรือไม่
        System.out.println("ไปทำงานอื่นต่อ ไม่รอ longTask จนจบ");
    }
}
```

## 8. ปัญหาคลาสสิก: การแชร์ข้อมูลระหว่าง Thread

```java
import java.util.ArrayList;
import java.util.List;

public class SharedDataProblemDemo {
    public static void main(String[] args) throws InterruptedException {
        List<Integer> list = new ArrayList<>(); // ArrayList ไม่ thread-safe! (ทบทวนจาก Part 42)

        Runnable addTask = () -> {
            for (int i = 0; i < 1000; i++) {
                list.add(i); // หลาย thread เขียนพร้อมกัน อาจเกิด ConcurrentModificationException
                              // หรือข้อมูลเสียหาย (corrupted internal state)
            }
        };

        Thread t1 = new Thread(addTask);
        Thread t2 = new Thread(addTask);
        t1.start();
        t2.start();
        t1.join();
        t2.join();

        System.out.println("ขนาด list: " + list.size());
        // คาดว่า 2000 แต่อาจได้ผลลัพธ์ผิดพลาด หรือ throw exception ระหว่างทาง!
        // (จะเรียนวิธีแก้ด้วย synchronized collections และ concurrent collections ใน Part 47, 50)
    }
}
```

## 9. ทำไมไม่ควรสร้าง Thread เองในโค้ดจริง

การสร้าง `new Thread()` เองมีปัญหาหลายข้อในทางปฏิบัติ:
1. **ไม่มีการจำกัดจำนวน**: สร้าง thread ไม่จำกัดอาจทำให้ระบบล่มจาก resource
   exhaustion (แต่ละ thread ใช้หน่วยความจำ stack ~1MB โดยประมาณ)
2. **ไม่มี reuse**: สร้าง thread ใหม่ทุกครั้งมี overhead สูง (การสร้าง/ทำลาย
   thread ไม่ใช่ของฟรี)
3. **จัดการยาก**: ไม่มีกลไกจัดการ error, ผลลัพธ์ที่คืนค่า, หรือการยกเลิกงาน
   ที่เป็นมาตรฐาน

**Part 48 จะแนะนำ `ExecutorService`** ซึ่งเป็นวิธีที่แนะนำในโค้ด production
จริง — จัดการ thread pool, reuse thread, และมี API ที่ปลอดภัยกว่ามาก

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** สร้าง 3 thread ที่พิมพ์ชื่อตัวเองพร้อมตัวเลข 1-5 แล้วใช้ `join()` รอ
ทุก thread เสร็จก่อนพิมพ์ "จบการทำงานทั้งหมด"

**เฉลย:**

```java
public class Exercise1 {
    public static void main(String[] args) throws InterruptedException {
        Thread[] threads = new Thread[3];
        for (int i = 0; i < 3; i++) {
            final int id = i;
            threads[i] = new Thread(() -> {
                for (int j = 1; j <= 5; j++) {
                    System.out.println("Thread-" + id + ": " + j);
                }
            });
            threads[i].start();
        }
        for (Thread t : threads) {
            t.join();
        }
        System.out.println("จบการทำงานทั้งหมด");
    }
}
```

**2)** ทดลองรัน `RaceConditionDemo` จากหัวข้อ 5 หลายครั้ง สังเกตว่าผลลัพธ์
ต่างกันในแต่ละครั้งหรือไม่ อธิบายเหตุผล

**เฉลย**: ผลลัพธ์มักจะ**ไม่คงที่ (ไม่ deterministic)** ในแต่ละครั้งที่รัน
เพราะการ interleaving (การสลับกันทำงาน) ของ thread ขึ้นกับ OS scheduler ซึ่ง
ไม่สามารถควบคุมหรือคาดเดาได้แน่นอน — นี่คือธรรมชาติของ race condition ที่ทำให้
bug ประเภทนี้**ตรวจจับและ debug ได้ยากมาก** (บางครั้งอาจรันถูกหลายสิบครั้งก่อน
จะเจอผลลัพธ์ที่ผิดพลาดสักครั้ง)

**3)** อธิบายว่าทำไมการเรียก `thread.run()` ตรง ๆ ไม่ได้สร้าง thread ใหม่

**เฉลย**: `run()` เป็นเพียง**เมธอดธรรมดา**ของ object `Thread`/`Runnable` — การ
เรียกมันตรง ๆ ก็เหมือนเรียกเมธอดทั่วไปบน thread ปัจจุบัน (ไม่มีอะไรพิเศษ)
ในขณะที่ `start()` เป็นเมธอดพิเศษที่**สั่งให้ JVM สร้าง native thread ใหม่**
ในระดับ operating system แล้วเมื่อ thread ใหม่นั้นเริ่มทำงาน JVM จะเรียก
`run()` ให้อัตโนมัติบน thread ใหม่นั้น — ถ้าไม่เรียก `start()` โค้ดใน `run()`
จะทำงานแบบ synchronous บน thread ที่เรียกมันเท่านั้น ไม่มี concurrency เกิดขึ้น
เลย

### สรุปเนื้อหา Part 46

- Process มีหน่วยความจำแยกกัน, Thread แชร์ heap ร่วมกันแต่มี stack แยกกัน
- สร้าง Thread ได้ 2 วิธี: extends `Thread` หรือ implements `Runnable`
  (แนะนำมากกว่า เพราะไม่เสีย single inheritance)
- ต้องเรียก `start()` เพื่อสร้าง thread ใหม่จริง ๆ ไม่ใช่เรียก `run()` ตรง ๆ
- Race Condition เกิดเมื่อหลาย thread แก้ไขข้อมูลร่วมกันโดยไม่มีการควบคุม
  ทำให้ผลลัพธ์ไม่แน่นอน
- `join()` ใช้รอให้ thread อื่นทำงานเสร็จก่อนทำงานต่อ
- ไม่ควรสร้าง `Thread` เองในโค้ด production จริง ควรใช้ `ExecutorService`
  (Part 48) แทน

**ต่อไป**: [Part 47 — Multithreading ขั้นสูง: synchronized, wait/notify, Lock](./part-047-multithreading-advanced.md)
