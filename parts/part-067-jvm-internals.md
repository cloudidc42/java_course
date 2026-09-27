# Part 67: JVM Internals: Memory Model, Heap/Stack, Garbage Collection

> ขั้นตอนที่ 661-670 ของหลักสูตร | ระดับ: สูงมาก

## สารบัญ

1. ทบทวนภาพรวม JVM Architecture
2. Runtime Data Areas แบบละเอียด
3. Heap Memory: Young Generation และ Old Generation
4. Stack Memory และ StackOverflowError
5. Garbage Collection คืออะไร ทำงานอย่างไร
6. GC Algorithms: จาก Mark-and-Sweep สู่ Generational GC
7. GC Collectors ใน Java สมัยใหม่: G1, ZGC, Shenandoah
8. Memory Leak ใน Java (ทั้งที่มี GC)
9. การ Tuning JVM Memory ด้วย Flags
10. แบบฝึกหัดและสรุป

---

## 1. ทบทวนภาพรวม JVM Architecture

ทบทวนจาก Part 1: JVM ประกอบด้วย Class Loader, Runtime Data Area, และ
Execution Engine — Part นี้จะเจาะลึก **Runtime Data Area** (หน่วยความจำ)
และกลไก **Garbage Collection** ซึ่งเป็นสิ่งที่ทำให้ Java ไม่ต้อง `free()`
หน่วยความจำเองแบบ C/C++ (แต่ก็มีข้อควรรู้เพื่อเขียนโค้ดที่มีประสิทธิภาพดี)

## 2. Runtime Data Areas แบบละเอียด

```
┌───────────────────────────────────────────────────────┐
│                    JVM Memory Structure                  │
├───────────────────────────────────────────────────────┤
│  Method Area / Metaspace (แชร์ร่วมกันทุก thread)          │
│  - class metadata, static fields (ทบทวน Part 17)         │
│  - constant pool (ทบทวน String Pool จาก Part 9)            │
├───────────────────────────────────────────────────────┤
│  Heap (แชร์ร่วมกันทุก thread)                              │
│  - ทุก object ที่สร้างด้วย new (ทบทวน Part 11)               │
│  - แบ่งเป็น Young Generation และ Old Generation (หัวข้อ 3)   │
├───────────────────────────────────────────────────────┤
│  Stack (แยกต่อ thread แต่ละตัว - ทบทวน Part 46)             │
│  - local variables, method call frames                    │
│  - primitive values และ reference (ไม่ใช่ object เอง)         │
├───────────────────────────────────────────────────────┤
│  PC Register (แยกต่อ thread)                              │
│  - เก็บตำแหน่ง bytecode instruction ปัจจุบันที่กำลังรัน         │
├───────────────────────────────────────────────────────┤
│  Native Method Stack (แยกต่อ thread)                       │
│  - สำหรับเรียกโค้ด native (C/C++) ผ่าน JNI                     │
└───────────────────────────────────────────────────────┘
```

## 3. Heap Memory: Young Generation และ Old Generation

**Heap** แบ่งเป็น 2 ส่วนหลักตามสมมติฐาน **"Generational Hypothesis"**:
object ส่วนใหญ่มีอายุสั้น (สร้างแล้วใช้ไม่นานก็ถูกทิ้ง) มีเพียงส่วนน้อยที่
มีอายุยืน

```
┌─────────────────────────────────────────────────────┐
│                      Heap                              │
│  ┌───────────────────────────┐  ┌──────────────────┐  │
│  │    Young Generation         │  │  Old Generation   │  │
│  │  ┌────────┐ ┌────┐ ┌────┐  │  │  (Tenured)         │  │
│  │  │  Eden   │ │ S0 │ │ S1 │  │  │                    │  │
│  │  └────────┘ └────┘ └────┘  │  │  object ที่มีอายุยืน  │  │
│  │  object ใหม่ทั้งหมด          │  │  (รอดจาก Minor GC   │  │
│  │  ถูกสร้างที่นี่ก่อนเสมอ        │  │   หลายครั้ง)          │  │
│  └───────────────────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────┘
```

- **Eden Space**: object ใหม่ทุกตัวถูกสร้างที่นี่ก่อนเสมอ (ทบทวน `new` จาก
  Part 11) — เต็มเร็วมากเพราะ object ส่วนใหญ่มีอายุสั้น
- **Survivor Spaces (S0, S1)**: object ที่รอดจาก **Minor GC** (การเก็บกวาด
  Young Generation) จะถูกย้ายมาที่นี่
- **Old Generation (Tenured)**: object ที่มีอายุยืนพอ (รอด Minor GC หลาย
  ครั้ง) จะถูก**promote**มาที่นี่ — การเก็บกวาด Old Generation เรียกว่า
  **Major GC** (หรือ Full GC) ซึ่งมี cost สูงกว่า Minor GC มาก

```java
public class HeapAllocationDemo {
    public static void main(String[] args) {
        for (int i = 0; i < 1_000_000; i++) {
            String temp = "object ชั่วคราว " + i; // สร้างใน Eden - อายุสั้นมาก ทิ้งทันทีหลัง loop รอบนี้จบ
        }
        // object ส่วนใหญ่ในลูปนี้ไม่รอดออกจาก Young Generation เลย - ถูกเก็บกวาดโดย Minor GC อย่างรวดเร็ว

        java.util.List<String> cache = new java.util.ArrayList<>(); // อาจถูก promote ไป Old Generation
        for (int i = 0; i < 1000; i++) {
            cache.add("ข้อมูลที่เก็บไว้นาน " + i); // ยังถูกอ้างอิงตลอดชีวิตโปรแกรม -> รอดหลาย GC cycle
        }
    }
}
```

## 4. Stack Memory และ StackOverflowError

ทบทวนจาก Part 8, 29: ทุกครั้งที่เรียกเมธอด จะสร้าง **stack frame** ใหม่บน
stack ของ thread นั้น เก็บ local variables, พารามิเตอร์, และ return address

```java
public class StackFrameDemo {
    static void methodA() {
        int x = 10; // x อยู่ใน stack frame ของ methodA
        methodB();
    }

    static void methodB() {
        int y = 20; // y อยู่ใน stack frame ของ methodB (แยกจาก methodA)
        System.out.println("Stack ปัจจุบัน: main -> methodA -> methodB");
    }

    public static void main(String[] args) {
        methodA();
        // เมื่อ methodA() คืนค่า stack frame ของมันถูกทำลายทันที (pop ออกจาก stack)
    }
}
```

**`StackOverflowError`** เกิดเมื่อ stack **เกินขนาดที่กำหนด** (ทบทวนจาก
Part 8, 29 — infinite recursion) — ต่างจาก `OutOfMemoryError` ของ heap
เพราะ stack มีขนาด**เล็กกว่ามาก**และแยกต่อ thread (ปรับขนาดได้ด้วย `-Xss`
flag — หัวข้อ 9)

## 5. Garbage Collection คืออะไร ทำงานอย่างไร

**Garbage Collection (GC)** คือกระบวนการ**หาและเก็บกวาด object ที่ไม่มีใคร
อ้างอิงถึงแล้ว**โดยอัตโนมัติ (ทบทวนแนวคิด reference จาก Part 11) — ทำให้
โปรแกรมเมอร์**ไม่ต้อง `free()` หน่วยความจำเอง**แบบ C/C++ (ลดปัญหา dangling
pointer และ double-free ได้เกือบทั้งหมด)

```java
public class GarbageCollectionDemo {
    public static void main(String[] args) {
        Object obj1 = new Object(); // สร้าง object A มี obj1 อ้างอิงอยู่
        Object obj2 = obj1;           // obj2 อ้างอิงถึง A ตัวเดียวกัน (ทบทวนจาก Part 11)

        obj1 = null; // A ยังมี obj2 อ้างอิงอยู่ -> ยังไม่ถูกเก็บกวาด

        obj2 = null; // ตอนนี้ไม่มีใครอ้างอิง A เลย -> "eligible for garbage collection"
                       // (แต่ไม่ได้ถูกเก็บกวาดทันที ต้องรอ GC cycle ถัดไป)

        System.gc(); // "คำขอ" ให้ JVM รัน GC (ไม่รับประกันว่าจะรันจริงทันที - เป็นแค่ hint)
    }
}
```

**หลักการพื้นฐาน**: JVM ใช้ **Reachability Analysis** — เริ่มจาก **GC Roots**
(local variables ใน stack, static field, ทบทวน Part 17) แล้วไล่ตาม
reference ทั้งหมด object ที่**ไล่ตามไปไม่ถึง**จาก GC Root จะถูกมองว่า
"garbage" และถูกเก็บกวาด

```
GC Roots (stack variables, static fields)
    │
    ├──> Object A ──> Object B  (reachable - ยังใช้งานได้)
    │
    └──> Object C

Object D  (ไม่มีใครอ้างอิงถึงเลย - unreachable = garbage, จะถูกเก็บกวาด)
```

## 6. GC Algorithms: จาก Mark-and-Sweep สู่ Generational GC

**Mark-and-Sweep** (algorithm พื้นฐาน):
1. **Mark**: ไล่ตาม reference จาก GC Root ทำเครื่องหมาย object ที่ยัง
   reachable
2. **Sweep**: เก็บกวาด object ที่ไม่ได้ถูกทำเครื่องหมาย (unreachable)

```
ก่อน GC:  [A✓][B✓][C][D✓][E]    (✓ = reachable, ตัวไม่มี✓ = unreachable)
หลัง Mark: [A✓][B✓][C ][D✓][E ]
หลัง Sweep: [A✓][B✓][____][D✓][____]  (C, E ถูกลบทิ้ง เหลือช่องว่าง - fragmentation)
```

**ปัญหา**: หน่วยความจำ**กระจัดกระจาย (fragmentation)** — มีที่ว่างแต่ไม่
ต่อเนื่องกัน ทำให้จองพื้นที่สำหรับ object ใหญ่ยากขึ้น

**Mark-Compact** แก้ปัญหานี้โดย**ย้าย object ที่รอดให้อยู่ติดกัน**หลัง sweep:

```
หลัง Compact: [A✓][B✓][D✓][__________]  (พื้นที่ว่างต่อเนื่องกันเป็นก้อนใหญ่)
```

**Generational GC** (ที่ Java ใช้จริง — ทบทวนจากหัวข้อ 3) ใช้กลยุทธ์ต่างกัน
สำหรับ Young/Old Generation: Young ใช้ **Copying algorithm** (เร็ว เหมาะกับ
object ที่ตายเร็ว), Old ใช้ **Mark-Compact** (เหมาะกับ object ที่อยู่นาน)

## 7. GC Collectors ใน Java สมัยใหม่: G1, ZGC, Shenandoah

| Collector | ลักษณะเด่น | เหมาะกับ |
|---|---|---|
| **Serial GC** | Single-thread, หยุดโปรแกรมทั้งหมดตอน GC (stop-the-world) | แอปพลิเคชันเล็กมาก, single-core |
| **Parallel GC** | หลาย thread ช่วย GC พร้อมกัน แต่ยัง stop-the-world | throughput สูง ไม่สนใจ latency มาก |
| **G1 (Garbage-First)** | แบ่ง heap เป็น region เล็ก ๆ, ลด pause time ได้ดี | **ค่า default ของ Java สมัยใหม่** (Java 9+) |
| **ZGC** | Pause time ต่ำมาก (< 1ms) แม้ heap ขนาดหลาย TB | แอปพลิเคชัน latency-critical, heap ขนาดใหญ่มาก |
| **Shenandoah** | คล้าย ZGC (low pause time) พัฒนาโดย Red Hat | ทางเลือกคู่แข่งของ ZGC |

```bash
# เลือก GC collector ผ่าน JVM flag (หัวข้อ 9 จะลงรายละเอียด flag อื่น ๆ)
java -XX:+UseG1GC -jar myapp.jar        # G1 (default อยู่แล้วใน Java สมัยใหม่)
java -XX:+UseZGC -jar myapp.jar          # ZGC (เหมาะกับ latency-critical application)
```

**Stop-the-World**: ช่วงเวลาที่ **application thread ทั้งหมดหยุดทำงาน** เพื่อ
ให้ GC ทำงานอย่างปลอดภัย (ป้องกัน race condition ระหว่าง GC กับโค้ดที่กำลัง
แก้ไข object — ทบทวนแนวคิดจาก Part 46-47) — collector รุ่นใหม่ (G1, ZGC)
พยายามลดช่วงเวลานี้ให้สั้นที่สุดเท่าที่เป็นไปได้

## 8. Memory Leak ใน Java (ทั้งที่มี GC)

**Java ยังเกิด Memory Leak ได้** แม้มี GC! เพราะ GC เก็บกวาดเฉพาะ object ที่
**unreachable** — ถ้าโค้ดยัง**อ้างอิงถึง object ที่ไม่ได้ใช้แล้ว**อยู่ (โดย
ไม่ตั้งใจ) GC จะไม่กล้าเก็บกวาดมันเลย

```java
import java.util.ArrayList;
import java.util.List;

public class MemoryLeakDemo {
    static List<byte[]> cache = new ArrayList<>(); // static field: มีชีวิตยืนตลอดโปรแกรม (ทบทวน Part 17)

    static void addToCache(byte[] data) {
        cache.add(data); // เพิ่มเข้า cache เรื่อย ๆ โดยไม่มีการลบออกเลย
        // ถึงแม้ไม่มีใครใช้ data นี้อีกแล้วในทางตรรกะ แต่ cache ยังอ้างอิงอยู่
        // -> GC มองว่า "reachable" เสมอ -> ไม่เก็บกวาด -> หน่วยความจำโตขึ้นเรื่อย ๆ จนกระทั่ง OutOfMemoryError
    }
}
```

**สาเหตุ Memory Leak ที่พบบ่อยใน Java**:
1. **Static collections ที่ไม่เคยลบข้อมูลออก** (ทบทวนคำเตือนจาก Part 17)
2. **Listener/Observer ที่ไม่ unsubscribe** (ทบทวน Observer Pattern จาก
   Part 56 — ถ้าไม่ `unsubscribe()` observer จะยังถูกอ้างอิงตลอดไป)
3. **Inner class ที่ผูกกับ outer instance** (ทบทวนคำเตือนจาก Part 19)
4. **ThreadLocal ที่ไม่ถูก remove** ใน thread pool ที่ใช้ thread ซ้ำ (ทบทวน
   Part 48, 63 — คล้ายปัญหา MDC ที่ต้อง clear())
5. **Resource ที่ไม่ปิด** (connection, stream — ทบทวน try-with-resources
   จาก Part 21)

## 9. การ Tuning JVM Memory ด้วย Flags

```bash
# กำหนดขนาด Heap
java -Xms512m -Xmx2g -jar myapp.jar
#     ↑ initial heap 512MB      ↑ maximum heap 2GB

# กำหนดขนาด Stack ต่อ thread
java -Xss1m -jar myapp.jar  # stack ขนาด 1MB ต่อ thread (ค่า default มักอยู่ที่ 512KB-1MB)

# ดู GC log เพื่อ debug/tuning (Part 99 จะลงลึกเรื่อง monitoring)
java -Xlog:gc*:file=gc.log -jar myapp.jar

# ตั้งค่า GC collector ที่ใช้ (ทบทวนจากหัวข้อ 7)
java -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -jar myapp.jar
```

**หลักปฏิบัติ**: **อย่า tune JVM flags โดยไม่มีข้อมูลจริง** — ควรวัดผล
(profiling, GC log analysis) ก่อนปรับค่าเสมอ การเดาค่าที่ "น่าจะดี" โดยไม่วัด
ผลอาจทำให้ระบบแย่ลงกว่าเดิม

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** อธิบายว่าทำไม object ส่วนใหญ่ที่สร้างในเมธอดสั้น ๆ (เช่นตัวแปร local
ใน loop) ไม่ทำให้ heap โตขึ้นอย่างมีนัยสำคัญ

**เฉลย**: object เหล่านี้ถูกสร้างใน **Eden Space** ของ Young Generation และ
มีอายุสั้นมาก (ถูกใช้แค่ระหว่างการทำงานของเมธอดนั้น แล้วก็ไม่มีใครอ้างอิงถึง
อีกเมื่อออกจาก scope — ทบทวนแนวคิด scope จาก Part 3) **Minor GC** ทำงานบ่อย
และเร็ว (เพราะ Young Generation มีขนาดเล็ก) ทำให้ object เหล่านี้ถูกเก็บกวาด
อย่างรวดเร็วโดยไม่กระทบ performance โดยรวมมากนัก ต่างจาก object ที่มีอายุยืน
(ถูกอ้างอิงต่อเนื่องนาน) ที่จะถูก promote ไป Old Generation ซึ่งการเก็บกวาด
มี cost สูงกว่ามาก

**2)** ระบุว่าโค้ดนี้เสี่ยง memory leak หรือไม่ และเพราะเหตุใด:

```java
static Map<String, Connection> connectionCache = new HashMap<>();
void cacheConnection(String key, Connection conn) {
    connectionCache.put(key, conn);
}
```

**เฉลย**: **เสี่ยง memory leak สูง** เพราะ `connectionCache` เป็น `static`
field ที่มีชีวิตยืนตลอดโปรแกรม (ทบทวนจาก Part 17) — ถ้าไม่มีกลไก**ลบ
connection ที่ไม่ใช้แล้วออก** (เช่น expiration policy, LRU eviction —
ทบทวนตัวอย่าง `LinkedHashMap` LRU cache จาก Part 24) `Connection` object
(ซึ่งมักถือ resource หนักอย่าง socket — ทบทวนจาก Part 64-66) จะสะสมอยู่ใน
`Map` เรื่อย ๆ โดยไม่ถูกเก็บกวาดเลย เพราะ GC มองว่า `connectionCache` ยัง
"reachable" อยู่ตลอดเวลา ทำให้ทั้งหน่วยความจำ Java heap และ resource ของ
ระบบปฏิบัติการ (file descriptor สำหรับ socket) ค่อย ๆ หมดลง

**3)** อธิบายความแตกต่างระหว่าง `StackOverflowError` และ `OutOfMemoryError`

**เฉลย**: `StackOverflowError` เกิดจาก **stack** (หน่วยความจำแยกต่อ thread
ที่มีขนาดค่อนข้างเล็ก) เต็มจากการเรียกเมธอดซ้อนกันลึกเกินไป (มักเกิดจาก
infinite recursion ที่ไม่มี base case — ทบทวนจาก Part 8, 29) ส่วน
`OutOfMemoryError` เกิดจาก **heap** (หน่วยความจำที่ใหญ่กว่ามากและแชร์ร่วมกัน
ทุก thread) เต็มจากการมี object ที่ยัง reachable อยู่มากเกินไป (ไม่ว่าจะจาก
การสร้าง object จำนวนมากเกินไปในเวลาสั้น ๆ หรือจาก memory leak ที่สะสมมานาน
— ทบทวนหัวข้อ 8) ทั้งสองเป็น `Error` (ไม่ใช่ `Exception` — ทบทวนลำดับชั้น
จาก Part 10) เพราะเป็นปัญหาระดับ JVM ที่โปรแกรมทั่วไปไม่ควรพยายาม catch
และกู้คืนสถานะเอง

### สรุปเนื้อหา Part 67

- Heap แบ่งเป็น Young Generation (Eden + Survivor) และ Old Generation ตาม
  Generational Hypothesis
- Stack เก็บ local variables/method frames แยกต่อ thread, เต็มแล้วเกิด
  StackOverflowError
- GC ใช้ Reachability Analysis จาก GC Roots หา object ที่ unreachable แล้ว
  เก็บกวาด
- G1 คือ default GC ของ Java สมัยใหม่, ZGC/Shenandoah เหมาะกับ low-latency
  application
- Java ยังเกิด Memory Leak ได้จาก object ที่ถูกอ้างอิงโดยไม่ตั้งใจ (static
  collection, listener ที่ไม่ unsubscribe, resource ที่ไม่ปิด)
- ควร tune JVM flags จากข้อมูลการวัดผลจริง ไม่ใช่การเดา

**ต่อไป**: [Part 68 — Performance Tuning และ Profiling เบื้องต้น](./part-068-performance-tuning.md)
