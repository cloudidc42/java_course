# Part 49: CompletableFuture และ Asynchronous Programming

> ขั้นตอนที่ 481-490 ของหลักสูตร | ระดับ: สูง

## สารบัญ

1. ปัญหาของ `Future` แบบเดิม
2. `CompletableFuture` คืออะไร
3. การสร้าง `CompletableFuture`
4. Chaining: `thenApply`, `thenAccept`, `thenRun`
5. การรวม CompletableFuture หลายตัว: `thenCompose`, `thenCombine`
6. การรอหลาย Future: `allOf`, `anyOf`
7. การจัดการ Exception: `exceptionally`, `handle`, `whenComplete`
8. Async Variants: `*Async` methods และ Custom Executor
9. ตัวอย่างการใช้งานจริง: เรียก API หลายตัวพร้อมกัน
10. แบบฝึกหัดและสรุป

---

## 1. ปัญหาของ `Future` แบบเดิม

`Future` (Part 48) มีข้อจำกัดสำคัญ: **`get()` เป็น blocking call** (ต้องรอ
จนกว่าผลลัพธ์พร้อม) และ**ไม่มีทาง "ต่อ" การทำงานหลังจากได้ผลลัพธ์**โดยไม่บล็อก
thread — ทำให้เขียนโค้ด asynchronous แบบ non-blocking ทั้งกระบวนการยาก

```java
import java.util.concurrent.*;

public class FutureLimitationDemo {
    public static void main(String[] args) throws Exception {
        ExecutorService executor = Executors.newFixedThreadPool(2);

        Future<Integer> future = executor.submit(() -> {
            Thread.sleep(1000);
            return 10;
        });

        // ต้องการ "ต่อการทำงาน" หลังได้ผลลัพธ์ (เช่น คูณ 2) แต่ Future ไม่มีเมธอดให้ทำแบบนี้
        // ต้องเรียก get() ซึ่ง "บล็อก" thread ปัจจุบันจนกว่าผลลัพธ์จะพร้อม
        int result = future.get(); // บล็อก!
        int doubled = result * 2;
        System.out.println(doubled);

        executor.shutdown();
    }
}
```

## 2. `CompletableFuture` คืออะไร

**`CompletableFuture<T>`** (Java 8+) แก้ปัญหานี้ด้วย **functional style
chaining** (เหมือน Stream — Part 41-42) — สามารถ "ต่อ" การทำงานหลังจากได้
ผลลัพธ์ได้โดย**ไม่ต้องบล็อก thread** และรองรับการ**รวมผลลัพธ์จากหลาย future**
ได้อย่างสะดวก

```java
import java.util.concurrent.CompletableFuture;

public class CompletableFutureMotivationDemo {
    public static void main(String[] args) {
        CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
            try { Thread.sleep(1000); } catch (InterruptedException e) { }
            return 10;
        });

        // ต่อการทำงานได้เลย โดยไม่ต้องเรียก get() บล็อกก่อน
        future.thenApply(result -> result * 2)
              .thenAccept(doubled -> System.out.println("ผลลัพธ์: " + doubled));

        System.out.println("main thread ทำงานต่อได้ทันที ไม่ต้องรอ");

        try { Thread.sleep(1500); } catch (InterruptedException e) { } // แค่รอให้ demo นี้ทำงานจบ
    }
}
```

## 3. การสร้าง `CompletableFuture`

```java
import java.util.concurrent.CompletableFuture;

public class CreatingCompletableFutureDemo {
    public static void main(String[] args) throws Exception {
        // supplyAsync(): รันงานที่คืนค่า (เหมือน Callable) แบบ asynchronous บน thread pool ที่ใช้ร่วมกัน
        CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> {
            return "ผลลัพธ์จากงาน asynchronous";
        });

        // runAsync(): รันงานที่ไม่คืนค่า (เหมือน Runnable)
        CompletableFuture<Void> future2 = CompletableFuture.runAsync(() -> {
            System.out.println("ทำงานที่ไม่คืนค่า");
        });

        // completedFuture(): สร้าง future ที่มีค่าอยู่แล้ว (ใช้บ่อยตอนทดสอบหรือ default value)
        CompletableFuture<String> future3 = CompletableFuture.completedFuture("ค่าที่มีอยู่แล้ว");

        System.out.println(future1.get()); // ยังคงมี get() ให้ใช้ (แต่ปกติไม่จำเป็นถ้าเขียนแบบ chaining)
    }
}
```

## 4. Chaining: `thenApply`, `thenAccept`, `thenRun`

เทียบเคียงกับ `Function`, `Consumer`, `Runnable` (ทบทวนจาก Part 40):

```java
import java.util.concurrent.CompletableFuture;

public class ChainingDemo {
    public static void main(String[] args) throws Exception {
        CompletableFuture<Void> pipeline = CompletableFuture
            .supplyAsync(() -> 5)                        // เริ่มด้วยค่า 5
            .thenApply(n -> n * 2)                          // แปลงค่า (เหมือน Function) -> 10
            .thenApply(n -> n + 3)                            // แปลงต่อ -> 13
            .thenAccept(n -> System.out.println("ผลลัพธ์: " + n)) // ใช้ค่า ไม่คืนค่าต่อ (เหมือน Consumer)
            .thenRun(() -> System.out.println("จบ pipeline")); // ทำงานหลังจบ ไม่สนใจผลลัพธ์ก่อนหน้าเลย (เหมือน Runnable)

        pipeline.get(); // รอให้ pipeline ทั้งหมดเสร็จ (ใช้ตอน demo/testing)
    }
}
```

| เมธอด | รับ Function/Consumer/Runnable | คืนค่า |
|---|---|---|
| `thenApply(Function)` | แปลงค่า | `CompletableFuture<R>` (มีค่าต่อ) |
| `thenAccept(Consumer)` | ใช้ค่า ไม่คืนอะไร | `CompletableFuture<Void>` |
| `thenRun(Runnable)` | ไม่สนใจค่าก่อนหน้าเลย | `CompletableFuture<Void>` |

## 5. การรวม CompletableFuture หลายตัว: `thenCompose`, `thenCombine`

**`thenCompose()`** ใช้เมื่อ operation ถัดไป**คืนค่าเป็น CompletableFuture
เอง** (คล้าย `flatMap` ของ Stream — ทบทวนจาก Part 42) — ป้องกันปัญหา "future
ซ้อน future" (`CompletableFuture<CompletableFuture<T>>`)

```java
import java.util.concurrent.CompletableFuture;

public class ThenComposeDemo {
    static CompletableFuture<String> getUserName(int userId) {
        return CompletableFuture.supplyAsync(() -> "User" + userId);
    }

    static CompletableFuture<String> getUserEmail(String userName) {
        return CompletableFuture.supplyAsync(() -> userName + "@example.com");
    }

    public static void main(String[] args) throws Exception {
        // thenApply จะได้ CompletableFuture<CompletableFuture<String>> ซึ่งใช้งานยาก
        // thenCompose แก้ปัญหานี้ด้วยการ "แบนราบ" ให้เป็น CompletableFuture<String> เดียว
        CompletableFuture<String> result = getUserName(1)
            .thenCompose(ThenComposeDemo::getUserEmail);

        System.out.println(result.get()); // "User1@example.com"
    }
}
```

**`thenCombine()`** ใช้**รวมผลลัพธ์จาก 2 CompletableFuture ที่ทำงานอิสระกัน**
(ทำงานพร้อมกัน ไม่ใช่ทำต่อกันแบบ sequential):

```java
import java.util.concurrent.CompletableFuture;

public class ThenCombineDemo {
    public static void main(String[] args) throws Exception {
        CompletableFuture<Integer> price = CompletableFuture.supplyAsync(() -> {
            try { Thread.sleep(500); } catch (InterruptedException e) { }
            return 100;
        });

        CompletableFuture<Double> discountRate = CompletableFuture.supplyAsync(() -> {
            try { Thread.sleep(300); } catch (InterruptedException e) { }
            return 0.1;
        });

        // ทั้งสอง future ทำงาน "พร้อมกัน" (ไม่ใช่รอตัวแรกเสร็จก่อนเริ่มตัวสอง)
        CompletableFuture<Double> finalPrice = price.thenCombine(discountRate,
            (p, rate) -> p * (1 - rate)); // รวมผลลัพธ์ทั้งสองเข้าด้วยกันเมื่อทั้งคู่เสร็จแล้ว

        System.out.println(finalPrice.get()); // 90.0
    }
}
```

## 6. การรอหลาย Future: `allOf`, `anyOf`

```java
import java.util.List;
import java.util.concurrent.CompletableFuture;

public class AllOfAnyOfDemo {
    public static void main(String[] args) throws Exception {
        CompletableFuture<String> task1 = CompletableFuture.supplyAsync(() -> "ผลลัพธ์ 1");
        CompletableFuture<String> task2 = CompletableFuture.supplyAsync(() -> "ผลลัพธ์ 2");
        CompletableFuture<String> task3 = CompletableFuture.supplyAsync(() -> "ผลลัพธ์ 3");

        // allOf(): รอให้ "ทุกตัว" เสร็จ (คืนค่าเป็น CompletableFuture<Void> - ต้องดึงผลลัพธ์เองทีหลัง)
        CompletableFuture<Void> all = CompletableFuture.allOf(task1, task2, task3);
        all.get(); // รอให้ทั้ง 3 เสร็จ

        List<String> results = List.of(task1.get(), task2.get(), task3.get());
        System.out.println(results); // ตอนนี้ทุก future เสร็จแล้ว get() จะไม่บล็อกอีกต่อไป

        // anyOf(): คืนค่าทันทีที่ "ตัวใดตัวหนึ่ง" เสร็จก่อน (ไม่สนใจตัวอื่นที่ยังทำอยู่)
        CompletableFuture<Object> any = CompletableFuture.anyOf(task1, task2, task3);
        System.out.println("ตัวแรกที่เสร็จ: " + any.get());
    }
}
```

## 7. การจัดการ Exception: `exceptionally`, `handle`, `whenComplete`

```java
import java.util.concurrent.CompletableFuture;

public class ExceptionHandlingDemo {
    public static void main(String[] args) throws Exception {
        // exceptionally(): จัดการเฉพาะกรณี error เท่านั้น (คล้าย catch)
        CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(() -> {
            if (true) throw new RuntimeException("เกิดข้อผิดพลาด!");
            return 10;
        }).exceptionally(ex -> {
            System.out.println("จับ error ได้: " + ex.getMessage());
            return -1; // ค่า fallback
        });
        System.out.println(future1.get()); // -1

        // handle(): จัดการทั้งกรณีสำเร็จและ error ในที่เดียว (คล้าย try-catch ที่คืนค่าเสมอ)
        CompletableFuture<Integer> future2 = CompletableFuture.<Integer>supplyAsync(() -> {
            throw new RuntimeException("error อีกครั้ง");
        }).handle((result, ex) -> {
            if (ex != null) {
                System.out.println("มี error: " + ex.getMessage());
                return 0;
            }
            return result;
        });
        System.out.println(future2.get()); // 0

        // whenComplete(): ทำงานทั้งสองกรณี แต่ "ไม่เปลี่ยนผลลัพธ์" (คล้าย finally)
        CompletableFuture<Integer> future3 = CompletableFuture.supplyAsync(() -> 42)
            .whenComplete((result, ex) -> {
                if (ex == null) {
                    System.out.println("สำเร็จด้วยผลลัพธ์: " + result);
                } else {
                    System.out.println("ล้มเหลว: " + ex.getMessage());
                }
            });
        System.out.println(future3.get()); // 42 (ค่าไม่เปลี่ยนจาก whenComplete)
    }
}
```

## 8. Async Variants: `*Async` Methods และ Custom Executor

เมธอดหลายตัวมี**เวอร์ชัน `Async`** (เช่น `thenApplyAsync` เทียบกับ
`thenApply`) — ความแตกต่างคือ**thread ที่ใช้รัน callback**:

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class AsyncVariantsDemo {
    public static void main(String[] args) throws Exception {
        ExecutorService customExecutor = Executors.newFixedThreadPool(4);

        CompletableFuture.supplyAsync(() -> {
            System.out.println("Step 1 บน: " + Thread.currentThread().getName());
            return 10;
        })
        .thenApply(n -> { // ไม่มี Async: รันบน thread เดียวกับ step ก่อนหน้า (หรือ caller thread ถ้าเสร็จแล้ว)
            System.out.println("Step 2 (thenApply) บน: " + Thread.currentThread().getName());
            return n * 2;
        })
        .thenApplyAsync(n -> { // มี Async: รันบน thread pool ที่ระบุ (หรือ default pool ถ้าไม่ระบุ)
            System.out.println("Step 3 (thenApplyAsync) บน: " + Thread.currentThread().getName());
            return n + 1;
        }, customExecutor) // ระบุ executor ของเราเอง แทนใช้ default ForkJoinPool.commonPool()
        .join(); // join() เหมือน get() แต่ throw unchecked exception แทน checked

        customExecutor.shutdown();
    }
}
```

**คำแนะนำสำคัญ**: ควร**ระบุ custom `Executor` เสมอ**ในโค้ด production
(โดยเฉพาะใน web application) เพราะค่า default (`ForkJoinPool.commonPool()`)
เป็น**thread pool ที่แชร์ร่วมกันทั้ง JVM** — ถ้าโค้ดส่วนหนึ่งของแอปพลิเคชันใช้
งานหนักเกินไปจะกระทบส่วนอื่นที่ใช้ common pool เดียวกันด้วย

## 9. ตัวอย่างการใช้งานจริง: เรียก API หลายตัวพร้อมกัน

```java
import java.util.concurrent.CompletableFuture;

public class RealWorldExampleDemo {
    record UserProfile(String name, String email) { }
    record OrderHistory(int totalOrders) { }
    record Recommendation(String product) { }

    static CompletableFuture<UserProfile> fetchUserProfile(int userId) {
        return CompletableFuture.supplyAsync(() -> {
            simulateNetworkDelay(300);
            return new UserProfile("Alice", "alice@example.com");
        });
    }

    static CompletableFuture<OrderHistory> fetchOrderHistory(int userId) {
        return CompletableFuture.supplyAsync(() -> {
            simulateNetworkDelay(500);
            return new OrderHistory(15);
        });
    }

    static CompletableFuture<Recommendation> fetchRecommendation(int userId) {
        return CompletableFuture.supplyAsync(() -> {
            simulateNetworkDelay(400);
            return new Recommendation("Laptop Stand");
        });
    }

    static void simulateNetworkDelay(long ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) { }
    }

    public static void main(String[] args) throws Exception {
        long start = System.currentTimeMillis();

        // เรียก 3 "API" พร้อมกัน (ไม่ใช่ทีละตัวตามลำดับ) - ประหยัดเวลารวมได้มาก
        CompletableFuture<UserProfile> profileFuture = fetchUserProfile(1);
        CompletableFuture<OrderHistory> historyFuture = fetchOrderHistory(1);
        CompletableFuture<Recommendation> recommendationFuture = fetchRecommendation(1);

        CompletableFuture.allOf(profileFuture, historyFuture, recommendationFuture).join();

        UserProfile profile = profileFuture.get();
        OrderHistory history = historyFuture.get();
        Recommendation rec = recommendationFuture.get();

        System.out.println(profile + ", " + history + ", " + rec);
        System.out.println("ใช้เวลารวม: " + (System.currentTimeMillis() - start) + " ms");
        // ใช้เวลาประมาณ 500ms (เวลาของ API ที่ช้าที่สุด) ไม่ใช่ 300+500+400=1200ms
        // เพราะทั้ง 3 เรียกพร้อมกัน ไม่ใช่ทีละตัวตามลำดับ (sequential)
    }
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียน pipeline ที่ดึงตัวเลข, บวก 10, คูณ 2, แล้วแปลงเป็น String ทั้งหมด
ผ่าน `thenApply` แบบ chaining

**เฉลย:**

```java
import java.util.concurrent.CompletableFuture;

public class Exercise1 {
    public static void main(String[] args) throws Exception {
        CompletableFuture<String> result = CompletableFuture.supplyAsync(() -> 5)
            .thenApply(n -> n + 10)
            .thenApply(n -> n * 2)
            .thenApply(n -> "ผลลัพธ์สุดท้าย: " + n);
        System.out.println(result.get()); // ผลลัพธ์สุดท้าย: 30
    }
}
```

**2)** ใช้ `thenCombine` รวมผลลัพธ์จาก 2 future ที่คำนวณราคาสินค้าและอัตราแลก
เปลี่ยนสกุลเงิน แล้วคำนวณราคาสุดท้ายเป็นเงินอีกสกุล

**เฉลย:**

```java
import java.util.concurrent.CompletableFuture;

public class Exercise2 {
    public static void main(String[] args) throws Exception {
        CompletableFuture<Double> priceInThb = CompletableFuture.supplyAsync(() -> 1000.0);
        CompletableFuture<Double> exchangeRate = CompletableFuture.supplyAsync(() -> 0.028); // THB->USD

        CompletableFuture<Double> priceInUsd = priceInThb.thenCombine(exchangeRate,
            (price, rate) -> price * rate);

        System.out.println(priceInUsd.get()); // 28.0
    }
}
```

**3)** อธิบายว่าทำไมควรระบุ custom `Executor` เองในโค้ด production แทนใช้
default thread pool

**เฉลย**: ค่า default ของ `CompletableFuture` (เมื่อไม่ระบุ executor) คือ
`ForkJoinPool.commonPool()` ซึ่งเป็น**thread pool ที่แชร์ร่วมกันทั้ง JVM**
ถ้าส่วนหนึ่งของแอปพลิเคชัน (เช่น งานคำนวณหนัก หรือ blocking I/O) ใช้ thread
จาก common pool นานเกินไป จะทำให้ส่วนอื่นของแอปพลิเคชันที่ใช้ common pool
เดียวกัน**ขาด thread ไปใช้งาน**ด้วย (thread starvation) การระบุ custom
executor ทำให้แต่ละส่วนของระบบมี thread pool ของตัวเอง แยกจากกันอย่างชัดเจน
ป้องกันปัญหานี้ และยังปรับขนาด pool ให้เหมาะกับลักษณะงานแต่ละประเภทได้ด้วย

### สรุปเนื้อหา Part 49

- `CompletableFuture` แก้ปัญหา blocking ของ `Future` เดิมด้วย functional
  chaining แบบ non-blocking
- `thenApply`/`thenAccept`/`thenRun` ต่อการทำงานหลังผลลัพธ์พร้อม (เทียบกับ
  Function/Consumer/Runnable)
- `thenCompose` แบนราบ future ซ้อน future, `thenCombine` รวมผลลัพธ์จาก 2
  future ที่ทำงานอิสระกัน
- `allOf`/`anyOf` รอหลาย future พร้อมกัน (ทุกตัว หรือตัวแรกที่เสร็จ)
- `exceptionally`/`handle`/`whenComplete` จัดการ error แบบต่าง ๆ
- ควรระบุ custom `Executor` เองในโค้ด production เพื่อไม่ให้กระทบ common pool
  ที่แชร์กันทั้ง JVM

**ต่อไป**: [Part 50 — Concurrent Collections: ConcurrentHashMap, CopyOnWriteArrayList, Atomic](./part-050-concurrent-collections.md)
