# Part 48: Executor Framework, Thread Pool, Callable, Future

> ขั้นตอนที่ 471-480 ของหลักสูตร | ระดับ: สูง

## สารบัญ

1. ทำไมต้องใช้ Executor Framework
2. `ExecutorService` และ `Executors` Factory Methods
3. ชนิดของ Thread Pool
4. `Runnable` vs `Callable`
5. `Future`: รับผลลัพธ์จากงานที่รันแบบ asynchronous
6. `invokeAll()` และ `invokeAny()`
7. การจัดการ Exception ใน Executor
8. การปิด ExecutorService อย่างถูกต้อง (`shutdown` vs `shutdownNow`)
9. `ScheduledExecutorService`: งานที่ต้องทำตามกำหนดเวลา
10. แบบฝึกหัดและสรุป

---

## 1. ทำไมต้องใช้ Executor Framework

ทบทวนปัญหาจาก Part 46: การสร้าง `new Thread()` เองมีข้อเสียหลายข้อ —
**Executor Framework** (Java 5+) แก้ปัญหาเหล่านี้ด้วยการจัดการ **Thread Pool**
(กลุ่มของ thread ที่**สร้างไว้ล่วงหน้าและนำมาใช้ซ้ำ**) แทนการสร้าง/ทำลาย
thread ใหม่ทุกครั้ง

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ExecutorMotivationDemo {
    public static void main(String[] args) {
        // แบบเก่า: สร้าง thread ใหม่ทุกครั้ง (มี overhead สูง, ไม่จำกัดจำนวน)
        for (int i = 0; i < 5; i++) {
            new Thread(() -> System.out.println("งานที่ " + Thread.currentThread().getName())).start();
        }

        // แบบใหม่: ใช้ thread pool ที่มีอยู่แล้ว นำ thread กลับมาใช้ซ้ำ
        ExecutorService executor = Executors.newFixedThreadPool(3); // pool ที่มี thread คงที่ 3 ตัว
        for (int i = 0; i < 5; i++) {
            executor.submit(() -> System.out.println("งานที่ " + Thread.currentThread().getName()));
        }
        executor.shutdown(); // สำคัญมาก! ต้องปิด executor เสมอ (หัวข้อ 8)
    }
}
```

## 2. `ExecutorService` และ `Executors` Factory Methods

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ExecutorServiceDemo {
    public static void main(String[] args) throws InterruptedException {
        ExecutorService executor = Executors.newFixedThreadPool(4);

        for (int i = 1; i <= 8; i++) {
            final int taskId = i;
            executor.submit(() -> {
                System.out.println("กำลังทำงานที่ " + taskId + " บน " + Thread.currentThread().getName());
                try {
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
        }
        // มี 8 งาน แต่มีแค่ 4 thread ใน pool - งานที่เหลือจะรอในคิวจนกว่า thread จะว่าง

        executor.shutdown();
        executor.awaitTermination(5, java.util.concurrent.TimeUnit.SECONDS); // รอให้ทุกงานเสร็จ
        System.out.println("งานทั้งหมดเสร็จสิ้น");
    }
}
```

## 3. ชนิดของ Thread Pool

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ThreadPoolTypesDemo {
    public static void main(String[] args) {
        // FixedThreadPool: จำนวน thread คงที่ตลอด เหมาะกับงานที่มีปริมาณคาดเดาได้
        ExecutorService fixed = Executors.newFixedThreadPool(4);

        // CachedThreadPool: สร้าง thread ใหม่ตามต้องการ, นำ thread ที่ไม่ใช้กลับมาใช้ซ้ำ
        // เหมาะกับงานจำนวนมากที่ใช้เวลาสั้น แต่เสี่ยงสร้าง thread ไม่จำกัดถ้าใช้ผิด (ควรระวัง)
        ExecutorService cached = Executors.newCachedThreadPool();

        // SingleThreadExecutor: มีแค่ 1 thread เท่านั้น รับประกันงานทำตามลำดับ (FIFO)
        ExecutorService single = Executors.newSingleThreadExecutor();

        // ScheduledExecutorService: สำหรับงานที่ต้องทำตามกำหนดเวลา (หัวข้อ 9)
        var scheduled = Executors.newScheduledThreadPool(2);

        // ปิดทุก executor ให้เรียบร้อย
        fixed.shutdown();
        cached.shutdown();
        single.shutdown();
        scheduled.shutdown();
    }
}
```

| ชนิด | จำนวน Thread | เหมาะกับ |
|---|---|---|
| `newFixedThreadPool(n)` | คงที่ n ตัว | งานที่มีปริมาณคาดเดาได้ ต้องการควบคุม resource |
| `newCachedThreadPool()` | ปรับตามความต้องการ (ไม่จำกัด) | งานจำนวนมาก สั้น ๆ (ระวังการใช้งานผิด — resource exhaustion) |
| `newSingleThreadExecutor()` | 1 ตัว | งานที่ต้องทำตามลำดับ ไม่ต้องการ concurrency |
| `newVirtualThreadPerTaskExecutor()` (Java 21+) | Virtual Thread (Part 102) | งานจำนวนมหาศาลที่ I/O-bound |

**คำเตือน**: `newCachedThreadPool()` และ `newFixedThreadPool()`/
`newSingleThreadExecutor()` ที่สร้างผ่าน `Executors` มีข้อจำกัดเรื่อง queue
ที่ไม่จำกัดขนาด (unbounded) ซึ่งอาจทำให้เกิด `OutOfMemoryError` ได้ถ้างานเข้า
มาเร็วกว่าที่ประมวลผลได้ — ในโค้ด production ระดับสูง มักสร้าง
`ThreadPoolExecutor` เองโดยตรงเพื่อควบคุม queue size และ rejection policy
อย่างละเอียด

## 4. `Runnable` vs `Callable`

**`Runnable`** (ทบทวนจาก Part 46) **ไม่คืนค่าและไม่ throw checked exception**
— **`Callable<V>`** (Java 5+) **คืนค่าได้และ throw checked exception ได้**
เหมาะกับงานที่ต้องการผลลัพธ์กลับมา

```java
import java.util.concurrent.Callable;

public class CallableDemo {
    public static void main(String[] args) throws Exception {
        Callable<Integer> task = () -> { // functional interface (ทบทวนจาก Part 39-40)
            Thread.sleep(1000);
            return 42; // Callable คืนค่าได้ (Runnable.run() คืน void เท่านั้น)
        };

        System.out.println(task.call()); // เรียกตรง ๆ ได้ (แต่ปกติใช้ผ่าน ExecutorService — หัวข้อ 5)
    }
}
```

## 5. `Future`: รับผลลัพธ์จากงานที่รันแบบ Asynchronous

**`Future<V>`** เป็นตัวแทนของ**ผลลัพธ์ที่ยังไม่พร้อม** (จะพร้อมในอนาคต) —
คืนค่าจาก `ExecutorService.submit(Callable)` ใช้ดึงผลลัพธ์ออกมาเมื่องานเสร็จ

```java
import java.util.concurrent.*;

public class FutureDemo {
    public static void main(String[] args) throws InterruptedException, ExecutionException {
        ExecutorService executor = Executors.newFixedThreadPool(2);

        Future<Integer> future = executor.submit(() -> {
            Thread.sleep(2000); // จำลองงานที่ใช้เวลานาน
            return 42;
        });

        System.out.println("งานถูกส่งไปแล้ว ทำงานอื่นต่อได้ระหว่างรอ...");
        System.out.println(future.isDone()); // false (ยังทำงานไม่เสร็จ)

        Integer result = future.get(); // บล็อกรอจนกว่าผลลัพธ์จะพร้อม (get() คือจุดที่รอจริง ๆ)
        System.out.println("ผลลัพธ์: " + result);
        System.out.println(future.isDone()); // true

        // get() พร้อม timeout: ไม่รอตลอดไป
        Future<Integer> future2 = executor.submit(() -> {
            Thread.sleep(5000);
            return 100;
        });
        try {
            Integer result2 = future2.get(1, TimeUnit.SECONDS); // รอสูงสุด 1 วินาที
        } catch (TimeoutException e) {
            System.out.println("รอนานเกินไป ยกเลิกงาน");
            future2.cancel(true); // ยกเลิกงาน (interrupt thread ที่กำลังทำงาน)
        }

        executor.shutdown();
    }
}
```

## 6. `invokeAll()` และ `invokeAny()`

```java
import java.util.List;
import java.util.concurrent.*;

public class InvokeAllAnyDemo {
    public static void main(String[] args) throws InterruptedException, ExecutionException {
        ExecutorService executor = Executors.newFixedThreadPool(3);

        List<Callable<Integer>> tasks = List.of(
            () -> { Thread.sleep(1000); return 1; },
            () -> { Thread.sleep(500); return 2; },
            () -> { Thread.sleep(1500); return 3; }
        );

        // invokeAll(): รันทุก task พร้อมกัน รอให้ "ทุกตัว" เสร็จก่อนคืนค่า (List<Future<T>>)
        List<Future<Integer>> results = executor.invokeAll(tasks);
        for (Future<Integer> f : results) {
            System.out.println("ผลลัพธ์: " + f.get());
        }

        // invokeAny(): รันทุก task พร้อมกัน คืนค่า "ตัวแรกที่เสร็จ" เท่านั้น (ยกเลิกตัวที่เหลือ)
        Integer firstResult = executor.invokeAny(tasks);
        System.out.println("ตัวแรกที่เสร็จ: " + firstResult); // มักจะเป็น 2 (เพราะใช้เวลาน้อยสุด)

        executor.shutdown();
    }
}
```

## 7. การจัดการ Exception ใน Executor

```java
import java.util.concurrent.*;

public class ExecutorExceptionDemo {
    public static void main(String[] args) throws InterruptedException {
        ExecutorService executor = Executors.newFixedThreadPool(2);

        // ปัญหา: exception จาก Runnable ที่ส่งผ่าน execute() จะ "หายไปเงียบ ๆ"
        // (แสดงใน stack trace ของ thread แต่ไม่ propagate ไปที่ main thread)
        executor.execute(() -> {
            throw new RuntimeException("เกิดข้อผิดพลาดใน task!");
        });

        // ถ้าใช้ submit() กับ Callable/Runnable: exception จะถูกเก็บใน Future
        // และ throw ออกมาเมื่อเรียก future.get() (wrapped เป็น ExecutionException)
        Future<?> future = executor.submit(() -> {
            throw new RuntimeException("เกิดข้อผิดพลาดใน task ที่สอง!");
        });

        try {
            future.get();
        } catch (ExecutionException e) {
            System.out.println("จับ exception ได้: " + e.getCause().getMessage());
            // e.getCause() คือ exception ตัวจริงที่เกิดขึ้นใน task (ทบทวน exception chaining จาก Part 21)
        }

        Thread.sleep(500);
        executor.shutdown();
    }
}
```

**บทเรียนสำคัญ**: ใช้ `submit()` แทน `execute()` เมื่อต้องการจับ exception
จาก task ได้อย่างถูกต้อง — `execute()` (จาก `Executor` interface พื้นฐาน)
ไม่ให้ทางจับ exception กลับมาที่ผู้เรียกได้เลย

## 8. การปิด ExecutorService อย่างถูกต้อง (`shutdown` vs `shutdownNow`)

```java
import java.util.concurrent.*;

public class ShutdownDemo {
    public static void main(String[] args) throws InterruptedException {
        ExecutorService executor = Executors.newFixedThreadPool(2);

        for (int i = 0; i < 5; i++) {
            executor.submit(() -> {
                try { Thread.sleep(1000); } catch (InterruptedException e) { }
            });
        }

        // shutdown(): ไม่รับงานใหม่ แต่ "ทำงานที่มีอยู่ในคิวให้เสร็จก่อน" (graceful shutdown)
        executor.shutdown();

        try {
            if (!executor.awaitTermination(3, TimeUnit.SECONDS)) {
                System.out.println("รอไม่ทันเวลา บังคับปิดทันที");
                executor.shutdownNow(); // shutdownNow(): พยายาม interrupt งานที่ทำอยู่และยกเลิกงานในคิวทั้งหมด
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
        }

        System.out.println("ExecutorService ปิดเรียบร้อยแล้ว");
    }
}
```

**รูปแบบมาตรฐานที่แนะนำ** (จัดการ shutdown แบบสมบูรณ์):

```java
public class GracefulShutdownPattern {
    static void shutdownExecutor(ExecutorService executor) {
        executor.shutdown();
        try {
            if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
                executor.shutdownNow();
                if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
                    System.err.println("ExecutorService ไม่ยอมปิดตัว");
                }
            }
        } catch (InterruptedException ie) {
            executor.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }
}
```

## 9. `ScheduledExecutorService`: งานที่ต้องทำตามกำหนดเวลา

```java
import java.util.concurrent.*;

public class ScheduledExecutorDemo {
    public static void main(String[] args) throws InterruptedException {
        ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);

        // schedule(): รันครั้งเดียวหลังจาก delay ที่กำหนด
        scheduler.schedule(() -> System.out.println("รันหลังจาก 2 วินาที"), 2, TimeUnit.SECONDS);

        // scheduleAtFixedRate(): รันซ้ำทุก ๆ ช่วงเวลาที่กำหนด (นับจากเวลาเริ่มของแต่ละครั้ง)
        ScheduledFuture<?> repeatingTask = scheduler.scheduleAtFixedRate(() -> {
            System.out.println("ทำงานซ้ำ: " + java.time.LocalTime.now());
        }, 0, 1, TimeUnit.SECONDS); // เริ่มทันที (delay=0) ทำซ้ำทุก 1 วินาที

        Thread.sleep(5000);
        repeatingTask.cancel(false); // หยุดงานที่ทำซ้ำ

        scheduler.shutdown();
    }
}
```

**ตัวอย่างการใช้งานจริง**: health check เป็นระยะ, การล้าง cache ที่หมดอายุ,
การส่งรายงานสรุปทุกวัน (เนื้อหาเชิงลึกกว่านี้จะอยู่ใน Part 99 เรื่อง Monitoring)

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียนโปรแกรมที่ใช้ `ExecutorService` (FixedThreadPool ขนาด 3) รัน 10
งานที่คำนวณ factorial ของตัวเลข 1-10 แล้วรวบรวมผลลัพธ์ทั้งหมดด้วย `Future`

**เฉลย:**

```java
import java.util.*;
import java.util.concurrent.*;

public class Exercise1 {
    static long factorial(int n) {
        long result = 1;
        for (int i = 2; i <= n; i++) result *= i;
        return result;
    }

    public static void main(String[] args) throws Exception {
        ExecutorService executor = Executors.newFixedThreadPool(3);
        List<Future<Long>> futures = new ArrayList<>();

        for (int i = 1; i <= 10; i++) {
            final int n = i;
            futures.add(executor.submit(() -> factorial(n)));
        }

        for (int i = 0; i < futures.size(); i++) {
            System.out.println((i + 1) + "! = " + futures.get(i).get());
        }

        executor.shutdown();
    }
}
```

**2)** ใช้ `invokeAny()` จำลองการเรียก 3 server (สมมติด้วย delay ต่างกัน) แล้ว
ใช้ผลลัพธ์จาก server ที่ตอบเร็วที่สุด

**เฉลย:**

```java
import java.util.List;
import java.util.concurrent.*;

public class Exercise2 {
    public static void main(String[] args) throws Exception {
        ExecutorService executor = Executors.newFixedThreadPool(3);
        List<Callable<String>> servers = List.of(
            () -> { Thread.sleep(1000); return "Server A"; },
            () -> { Thread.sleep(300); return "Server B"; },
            () -> { Thread.sleep(2000); return "Server C"; }
        );
        String fastest = executor.invokeAny(servers);
        System.out.println("เร็วที่สุด: " + fastest); // Server B
        executor.shutdown();
    }
}
```

**3)** อธิบายว่าทำไม `execute()` ไม่เหมาะกับการจัดการ exception เท่า `submit()`

**เฉลย**: `execute()` มาจาก interface `Executor` พื้นฐานซึ่งไม่คืนค่าใด ๆ
กลับมา (`void`) — ถ้า task ที่ส่งเข้าไป throw exception, exception นั้นจะถูก
จัดการโดย default `UncaughtExceptionHandler` ของ thread (ปกติแค่ print stack
trace ออก console) โดยที่ผู้เรียก `execute()` ไม่มีทางรู้หรือจัดการมันได้เลย
ในขณะที่ `submit()` คืนค่า `Future` ที่**เก็บ exception ไว้ภายใน** และจะ
throw ออกมาเป็น `ExecutionException` (พร้อม `getCause()` เป็น exception ตัว
จริง) เมื่อเรียก `future.get()` ทำให้ผู้เรียกจัดการ error ได้อย่างเหมาะสม

### สรุปเนื้อหา Part 48

- Executor Framework ใช้ Thread Pool (สร้าง thread ไว้ล่วงหน้า นำมาใช้ซ้ำ)
  แทนการสร้าง `Thread` เองทุกครั้ง
- `Callable<V>` คืนค่าได้และ throw checked exception ได้ ต่างจาก `Runnable`
- `Future<V>` เป็นตัวแทนผลลัพธ์ในอนาคต, `get()` บล็อกรอผลลัพธ์ (มี timeout
  version ด้วย)
- ใช้ `submit()` แทน `execute()` เพื่อจับ exception จาก task ได้อย่างถูกต้อง
- ต้องปิด `ExecutorService` เสมอด้วย `shutdown()`/`shutdownNow()` เพื่อไม่ให้
  thread ค้างอยู่หลังโปรแกรมควรจบ
- `ScheduledExecutorService` ใช้กับงานที่ต้องทำตามกำหนดเวลาหรือทำซ้ำเป็นระยะ

**ต่อไป**: [Part 49 — CompletableFuture และ Asynchronous Programming](./part-049-completablefuture.md)
