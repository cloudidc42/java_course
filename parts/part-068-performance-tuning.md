# Part 68: Performance Tuning และ Profiling เบื้องต้น

> ขั้นตอนที่ 671-680 ของหลักสูตร | ระดับ: สูงมาก

## สารบัญ

1. หลักการสำคัญ: "อย่า Optimize ก่อนวัดผล"
2. Micro-benchmark ที่ผิดพลาด: กับดักของ JIT Compiler
3. JMH: เครื่องมือ Benchmark ที่ถูกต้อง
4. Profiling คืออะไร
5. CPU Profiling: หา Hot Path
6. Memory Profiling: หา Memory Leak และ Allocation ที่มากเกินไป
7. เทคนิค Optimization ที่ใช้บ่อย
8. String และ Collection Performance Tips (ทบทวนสรุป)
9. JIT Compiler: C1 และ C2, Warm-up
10. แบบฝึกหัดและสรุป

---

## 1. หลักการสำคัญ: "อย่า Optimize ก่อนวัดผล"

**Donald Knuth** กล่าวไว้ว่า **"Premature optimization is the root of all
evil"** — การพยายาม optimize โค้ดโดยไม่มีข้อมูลจริงมักนำไปสู่:
- โค้ดที่ซับซ้อนขึ้นโดยไม่ได้ประโยชน์จริง (ทบทวน Clean Code จาก Part 57)
- Optimize จุดที่ไม่ใช่ bottleneck จริง (เสียเวลาโดยเปล่าประโยชน์)
- Bug ใหม่จากความซับซ้อนที่เพิ่มขึ้น

**กระบวนการที่ถูกต้อง**:
```
1. Measure (วัดผลปัจจุบัน) -> 2. Profile (หาจุดที่ช้าจริง) ->
3. Optimize (แก้เฉพาะจุดนั้น) -> 4. Measure again (วัดผลซ้ำ ยืนยันว่าดีขึ้นจริง)
```

## 2. Micro-benchmark ที่ผิดพลาด: กับดักของ JIT Compiler

การวัดผลด้วย `System.currentTimeMillis()` ตรง ๆ (ทบทวนที่เราใช้มาตลอด
หลักสูตรใน Part 30, 35 ฯลฯ) **ใช้ได้สำหรับตัวอย่างในหลักสูตร แต่ผิดพลาดง่าย
มากสำหรับการวัดผลจริงจัง**:

```java
public class BadBenchmarkDemo {
    public static void main(String[] args) {
        long start = System.currentTimeMillis();
        for (int i = 0; i < 1000; i++) {
            String s = "a" + "b"; // Compiler อาจ optimize เป็น constant "ab" ตั้งแต่ compile-time!
        }
        System.out.println("เวลา: " + (System.currentTimeMillis() - start) + " ms");
        // ผลลัพธ์นี้ไม่มีความหมายเลย เพราะ JIT compiler (ทบทวนจาก Part 1) อาจ:
        // 1. Optimize loop ทั้งหมดออกไปเลย เพราะผลลัพธ์ไม่ถูกใช้ที่ไหน (dead code elimination)
        // 2. ยังไม่ warm up (interpreter mode ช้ากว่า JIT-compiled code มาก - หัวข้อ 9)
    }
}
```

**ปัญหาของ micro-benchmark แบบง่าย**:
1. **JIT ยังไม่ warm up**: โค้ดช้าตอนแรกเสมอ (interpreter mode) ก่อนที่ JIT
   จะแปลงเป็น native code
2. **Dead Code Elimination**: compiler อาจลบโค้ดที่ผลลัพธ์ไม่ถูกใช้ทิ้งไปเลย
3. **Constant Folding**: คำนวณค่าคงที่ล่วงหน้าตอน compile-time (เหมือน
   ตัวอย่างข้างบน)

## 3. JMH: เครื่องมือ Benchmark ที่ถูกต้อง

**JMH (Java Microbenchmark Harness)** พัฒนาโดยทีม OpenJDK เอง — ออกแบบมา
เฉพาะเพื่อแก้ปัญหาทั้งหมดในหัวข้อ 2

```xml
<dependency>
    <groupId>org.openjdk.jmh</groupId>
    <artifactId>jmh-core</artifactId>
    <version>1.37</version>
</dependency>
```

```java
import org.openjdk.jmh.annotations.*;
import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.AverageTime) // วัดเวลาเฉลี่ยต่อ operation
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@Warmup(iterations = 5) // รัน warm-up 5 รอบก่อนวัดผลจริง (แก้ปัญหา JIT warm-up จากหัวข้อ 2)
@Measurement(iterations = 10) // วัดผลจริง 10 รอบ
public class StringConcatBenchmark {

    @Benchmark
    public String testPlusOperator() {
        return "hello" + "world"; // JMH ป้องกัน constant folding ด้วยกลไกภายใน
    }

    @Benchmark
    public String testStringBuilder() {
        return new StringBuilder("hello").append("world").toString(); // ทบทวนจาก Part 9
    }
}
```

รันด้วย JMH runner (ผ่าน Maven plugin หรือ main class ที่ generate ให้) จะ
ได้ผลลัพธ์ที่**น่าเชื่อถือทางสถิติ** (ค่าเฉลี่ย, ค่าคลาดเคลื่อน, throughput)

**หลักปฏิบัติ**: ใช้ JMH สำหรับการวัดผลระดับ**เมธอดเดียว** (micro-benchmark)
ที่ต้องแม่นยำจริงจัง สำหรับการวัดผล**ระดับระบบทั้งหมด** (เช่น response time
ของ API) ให้ใช้เครื่องมือ APM (Application Performance Monitoring — Part
99) แทน

## 4. Profiling คืออะไร

**Profiler** คือเครื่องมือที่**สังเกตการทำงานของโปรแกรมขณะรันจริง** เพื่อ
หาว่า**เวลา/หน่วยความจำถูกใช้ไปที่ไหนบ้าง** — ต่างจาก benchmark ที่วัดโค้ด
ส่วนเล็ก ๆ ที่รู้อยู่แล้ว profiler ช่วย**ค้นหา**ว่าปัญหาอยู่ที่ไหนในระบบ
ขนาดใหญ่ที่ซับซ้อน

เครื่องมือที่นิยม: **JProfiler**, **VisualVM** (มาพร้อม JDK บางเวอร์ชัน,
ฟรี), **async-profiler**, **JDK Flight Recorder (JFR)** — ตัวหลังนี้มีมา
พร้อม JDK โดยไม่ต้องติดตั้งเพิ่ม

```bash
# JDK Flight Recorder: บันทึกข้อมูล profiling โดย overhead ต่ำมาก (เหมาะกับ production)
java -XX:StartFlightRecording=duration=60s,filename=recording.jfr -jar myapp.jar

# เปิดไฟล์ .jfr ด้วย JDK Mission Control (JMC) เพื่อวิเคราะห์ผล
```

## 5. CPU Profiling: หา Hot Path

**CPU Profiling** แสดงว่า**เมธอดไหนใช้เวลา CPU มากที่สุด** — มักแสดงผลเป็น
**Flame Graph** (กราฟรูปเปลวไฟ) ที่เห็นภาพรวมของ call stack ทั้งระบบได้ในทันที

```
Flame Graph (แนวคิด - ยิ่งกว้าง ยิ่งใช้เวลามาก):

main()
└── processOrders()                              [กว้างมาก - ใช้เวลานาน]
    ├── validateOrder()          [แคบ - เร็ว]
    └── calculateTotalPrice()                     [กว้าง - ช้า, นี่คือ "hot path"]
        └── queryDatabaseForPrice() (N+1 query!)   [กว้างที่สุด - ต้นเหตุตัวจริง]
```

**หลักการอ่าน Flame Graph**: มองหา**แถบที่กว้างที่สุด**ในระดับลึก ๆ — นั่น
คือจุดที่ควร optimize ก่อน (ทบทวนหลักการ 80/20 — มักมีแค่ไม่กี่จุดที่เป็น
bottleneck จริง ๆ ในระบบขนาดใหญ่)

## 6. Memory Profiling: หา Memory Leak และ Allocation ที่มากเกินไป

ทบทวนจาก Part 67: Memory Profiler ช่วยดู**Heap Dump** (สถานะของ heap ณ
เวลาหนึ่ง) เพื่อหา object ที่**ค้างอยู่มากผิดปกติ**

```bash
# สร้าง heap dump ด้วยมือ (สำหรับ debug ปัญหาที่เกิดขึ้นแล้ว)
jmap -dump:live,format=b,file=heapdump.hprof <PID>

# หรือให้ JVM สร้างอัตโนมัติเมื่อเกิด OutOfMemoryError (ทบทวนจาก Part 67)
java -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=./heapdump.hprof -jar myapp.jar
```

เปิดไฟล์ `.hprof` ด้วย **Eclipse MAT (Memory Analyzer Tool)** หรือ
**VisualVM** เพื่อดู:
- **Object ที่มีจำนวนมากที่สุด** (เช่น `HashMap$Node` เยอะผิดปกติ อาจบ่งบอก
  memory leak ใน cache — ทบทวนตัวอย่างจาก Part 67)
- **Dominator Tree**: ดูว่า object ใดถือ reference ไปยัง object อื่นจำนวน
  มาก (ตัวการหลักที่ทำให้หน่วยความจำไม่ถูกเก็บกวาด)

## 7. เทคนิค Optimization ที่ใช้บ่อย

**ทบทวนจากทั้งหลักสูตร — นี่คือรายการเทคนิคที่ได้ผลจริงและวัดผลได้ชัดเจน**:

```java
public class CommonOptimizationsDemo {
    // 1. ใช้ StringBuilder แทนการต่อ String ใน loop (ทบทวนจาก Part 9)
    static String buildString(int n) {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < n; i++) sb.append(i);
        return sb.toString();
    }

    // 2. หลีกเลี่ยง autoboxing ใน loop จำนวนมาก (ทบทวนจาก Part 28, 40)
    static long sumPrimitive(int[] numbers) {
        long sum = 0;
        for (int n : numbers) sum += n; // ใช้ primitive ตรง ๆ ไม่ผ่าน Integer
        return sum;
    }

    // 3. กำหนด initial capacity ของ collection ถ้ารู้ขนาดล่วงหน้า (ทบทวนจาก Part 22, 24)
    static java.util.List<Integer> createList(int expectedSize) {
        return new java.util.ArrayList<>(expectedSize); // ลดการ resize ภายในซ้ำ ๆ
    }

    // 4. ใช้ HashMap/HashSet แทน List เมื่อต้องค้นหาบ่อย (ทบทวน Big O จาก Part 35)
    static boolean containsEfficient(java.util.Set<String> set, String target) {
        return set.contains(target); // O(1) แทน O(n) ของ List.contains()
    }

    // 5. Lazy initialization สำหรับ object ที่มี cost สูงและอาจไม่ถูกใช้ (ทบทวน Supplier จาก Part 40)
    static class ExpensiveResource {
        private static ExpensiveResource instance;
        static ExpensiveResource getInstance() { // คล้าย Singleton lazy จาก Part 54
            if (instance == null) instance = new ExpensiveResource();
            return instance;
        }
    }
}
```

## 8. String และ Collection Performance Tips (ทบทวนสรุป)

| สถานการณ์ | ควรใช้ | เหตุผล (Part ที่เกี่ยวข้อง) |
|---|---|---|
| ต่อ String ใน loop มาก ๆ | `StringBuilder` | หลีกเลี่ยงสร้าง object ใหม่ทุกครั้ง (Part 9) |
| เข้าถึงด้วย index บ่อย | `ArrayList` | O(1) เทียบกับ `LinkedList` ที่ O(n) (Part 22) |
| ค้นหา/ตรวจสอบสมาชิกภาพบ่อย | `HashSet`/`HashMap` | O(1) เทียบกับ `List` ที่ O(n) (Part 23-24) |
| ข้อมูลจำนวนมากที่ทราบขนาดล่วงหน้า | กำหนด initial capacity | ลดการ resize/copy ภายใน (Part 22, 24) |
| Concurrent access | `ConcurrentHashMap` แทน synchronized wrapper | Lock striping เร็วกว่า (Part 50) |

## 9. JIT Compiler: C1 และ C2, Warm-up

ทบทวนจาก Part 1: JVM มี**สอง JIT compiler** ทำงานร่วมกันแบบ **tiered
compilation**:

- **C1 (Client Compiler)**: compile เร็ว แต่ optimize น้อย ใช้ตอนโปรแกรม
  เพิ่งเริ่มทำงาน (ให้ผลลัพธ์เร็วโดยไม่ต้องรอ optimize นาน)
- **C2 (Server Compiler)**: compile ช้ากว่า แต่ optimize ลึกกว่ามาก ใช้กับ
  โค้ดที่ถูกเรียกบ่อยมาก ("hot code" — เหมือนที่ profiler หาเจอในหัวข้อ 5)

```
โค้ดถูกเรียกครั้งแรก
    │
    ▼
Interpreter (ช้าที่สุด แต่เริ่มทำงานได้ทันที)
    │  (ถูกเรียกซ้ำหลายพันครั้ง)
    ▼
C1 JIT-compiled (เร็วขึ้น, compile แบบง่าย)
    │  (ยังถูกเรียกบ่อยต่อไป - "hot" method)
    ▼
C2 JIT-compiled (เร็วที่สุด, optimize ลึก เช่น inlining, loop unrolling)
```

**นี่คือเหตุผลที่ JMH ต้องมี warm-up iterations (ทบทวนจากหัวข้อ 3)** — ถ้า
วัดผลตั้งแต่ตอนที่โค้ดยังรันด้วย interpreter หรือ C1 เท่านั้น ผลลัพธ์จะ
**ไม่สะท้อนประสิทธิภาพจริงตอน production** ที่โค้ด hot path ถูก C2 compile
ไปแล้ว

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** อธิบายว่าทำไมการวัดผลด้วย `System.currentTimeMillis()` แบบง่าย ๆ
(ไม่ใช้ JMH) อาจให้ผลลัพธ์ที่เข้าใจผิดได้

**เฉลย**: เพราะ (1) โค้ดที่วัดอาจยังไม่ผ่าน JIT warm-up (ทบทวนหัวข้อ 9) ทำให้
ผลลัพธ์สะท้อนความช้าของ interpreter mode ไม่ใช่ประสิทธิภาพจริงตอน production,
(2) compiler อาจทำ dead code elimination หรือ constant folding ถ้าผลลัพธ์
ของโค้ดที่วัดไม่ถูกใช้งานจริง ทำให้เวลาที่วัดได้ไม่ตรงกับที่ควรจะเป็น, และ
(3) การวัดครั้งเดียวไม่ได้คำนึงถึงความแปรปรวน (variance) จาก GC pause หรือ
ปัจจัยอื่นของระบบปฏิบัติการ — JMH ถูกออกแบบมาแก้ปัญหาทั้งหมดนี้อย่างเป็นระบบ

**2)** อธิบายว่า Flame Graph ช่วยหา performance bottleneck อย่างไร

**เฉลย**: Flame Graph แสดง call stack ของโปรแกรมในรูปแบบกราฟ โดย**ความกว้าง
ของแต่ละแถบแทนเวลา CPU ที่ใช้ไปในเมธอดนั้น** (รวมเมธอดที่มันเรียกต่อด้วย) —
การมองหาแถบที่**กว้างที่สุดในระดับลึก ๆ** ของกราฟ ช่วยระบุได้อย่างรวดเร็วว่า
เมธอดใดเป็น "hot path" ที่ใช้เวลา CPU มากที่สุดจริง ๆ โดยไม่ต้องเดาหรือไล่
อ่านโค้ดทั้งระบบ — ทำให้มุ่งความพยายาม optimize ไปที่จุดที่ให้ผลตอบแทนสูง
สุดตามหลักการ "measure before optimize" ในหัวข้อ 1

**3)** ระบุเทคนิคจากหัวข้อ 7 ที่เหมาะสมกับสถานการณ์นี้: ระบบต้องค้นหาว่า
username มีอยู่ในระบบแล้วหรือไม่ จากผู้ใช้ 1 ล้านคน โดยเรียกบ่อยมาก

**เฉลย**: ควรใช้ **`HashSet<String>`** (หรือ `HashMap` ถ้าต้องเก็บข้อมูล
เพิ่มเติมของ user) แทน `List<String>` เพราะการค้นหาด้วย `contains()` บน
`HashSet` มี Time Complexity **O(1)** โดยเฉลี่ย ในขณะที่ `List.contains()`
มี Time Complexity **O(n)** — สำหรับผู้ใช้ 1 ล้านคนที่เรียกค้นหาบ่อยมาก
ความแตกต่างนี้มีผลกระทบต่อประสิทธิภาพอย่างมหาศาล (ทบทวน Big O จาก Part 35
และ HashSet จาก Part 23)

### สรุปเนื้อหา Part 68

- "อย่า optimize ก่อนวัดผล" — measure, profile, optimize, measure again
- Micro-benchmark แบบง่ายเสี่ยงผลลัพธ์ผิดพลาดจาก JIT warm-up และ dead code
  elimination — ใช้ JMH สำหรับการวัดผลที่แม่นยำ
- Profiler (VisualVM, JFR) ช่วยหา hot path (CPU) และ memory leak (heap dump)
  ในระบบขนาดใหญ่
- Flame Graph แสดง call stack แบบเห็นภาพรวม ช่วยหา bottleneck ได้เร็ว
- เทคนิค optimization ที่ได้ผลจริง: StringBuilder, หลีกเลี่ยง autoboxing,
  เลือก collection ที่เหมาะกับ pattern การใช้งาน, lazy initialization
- JIT ใช้ tiered compilation (interpreter -> C1 -> C2) — โค้ดต้อง "warm up"
  ก่อนถึงประสิทธิภาพสูงสุด

**ต่อไป**: [Part 69 — JAR Files, Classpath, Java Module System (JPMS)](./part-069-jpms.md)
