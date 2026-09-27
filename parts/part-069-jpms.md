# Part 69: JAR Files, Classpath, Java Module System (JPMS)

> ขั้นตอนที่ 681-690 ของหลักสูตร | ระดับ: สูง

## สารบัญ

1. ทบทวน JAR Files และ Classpath
2. ปัญหาของ Classpath: "JAR Hell"
3. Java Module System (JPMS) คืออะไร
4. `module-info.java`: ไฟล์นิยาม Module
5. `requires` และ `exports`
6. Strong Encapsulation: ประโยชน์หลักของ Module
7. Module Types: Named, Automatic, Unnamed
8. `jlink`: สร้าง Custom Runtime Image
9. เหตุผลที่ JPMS ยังไม่แพร่หลายเท่าที่คาด
10. แบบฝึกหัดและสรุป

---

## 1. ทบทวน JAR Files และ Classpath

ทบทวนจาก Part 20: **JAR** รวม `.class` files เป็นไฟล์เดียว, **Classpath**
คือรายการที่ JVM ค้นหาคลาส — ระบบนี้ทำงานมาตั้งแต่ Java 1.0 แต่มีข้อจำกัด
สำคัญที่ Part นี้จะอธิบาย

## 2. ปัญหาของ Classpath: "JAR Hell"

**JAR Hell** (หรือ "Classpath Hell") คือกลุ่มปัญหาที่เกิดจาก classpath แบบ
เดิม:

```java
public class ClasspathProblemsDemo {
    public static void main(String[] args) {
        // ปัญหา 1: ไม่มี encapsulation ระดับ JAR
        // ทุก public class ใน JAR หนึ่งสามารถถูกเรียกใช้จาก JAR อื่นได้เสมอ
        // แม้ว่า class นั้นจะเป็น "internal implementation detail" ที่ไม่ควรถูกใช้จากภายนอก

        // ปัญหา 2: Version Conflict (ทบทวนจาก Part 61 - Dependency Conflict)
        // classpath เป็น "flat list" - ถ้ามี commons-lang 2 เวอร์ชันอยู่ในนั้น
        // JVM จะโหลดตัวแรกที่เจอเท่านั้น อาจไม่ใช่เวอร์ชันที่ถูกต้อง

        // ปัญหา 3: ไม่รู้ว่า dependency ที่จำเป็นครบถ้วนหรือไม่จนกว่าจะรันจริง
        // (ClassNotFoundException เกิดตอน runtime ไม่ใช่ compile-time)
    }
}
```

**ตัวอย่างปัญหา encapsulation**: แม้ class จะเป็น**internal implementation
detail**ของ library หนึ่ง (ไม่ได้ตั้งใจให้ใครใช้ภายนอก) แต่ถ้าประกาศเป็น
`public` (ทบทวน access modifier จาก Part 13) **ก็ยังถูกเรียกใช้จากที่ไหนก็ได้
บน classpath** — ไม่มีทางป้องกันในระดับ package/JAR ได้อย่างแท้จริง

## 3. Java Module System (JPMS) คืออะไร

**Java Platform Module System (JPMS)** เปิดตัวใน **Java 9** (Project Jigsaw)
เพิ่มแนวคิด **Module** ที่อยู่**เหนือ package** (ทบทวนจาก Part 20) —
ให้**ประกาศชัดเจน**ว่า module นี้**ต้องการ (requires)** module อื่นอะไรบ้าง
และ**เปิดเผย (exports)** package ไหนให้ใช้งานจากภายนอกได้

```
ก่อน JPMS:                          หลัง JPMS:
┌─────────────────────┐             ┌─────────────────────┐
│  JAR (ไม่มี structure) │             │  Module              │
│  package A (public)   │             │  ├── package A (exported - ใช้ได้จากภายนอก) │
│  package B (public)   │             │  └── package B (internal - ใช้ได้แค่ในตัว module เอง) │
│  package C (public)   │             │  requires: other.module │
└─────────────────────┘             └─────────────────────┘
ทุก public class เข้าถึงได้หมด         ควบคุมได้ว่าอะไรเปิดเผย อะไรซ่อนไว้จริง ๆ
```

## 4. `module-info.java`: ไฟล์นิยาม Module

Module ถูกนิยามด้วยไฟล์พิเศษ `module-info.java` ที่วางไว้ที่**root ของ
source folder**:

```
my-app/
└── src/
    ├── module-info.java   <- นิยาม module (วางไว้ที่ root ของ source)
    └── com/
        └── example/
            ├── api/
            │   └── PublicService.java     <- ต้องการเปิดเผยให้ใช้ภายนอก
            └── internal/
                └── InternalHelper.java      <- ไม่ต้องการเปิดเผย (internal only)
```

```java
// module-info.java
module com.example.myapp {
    requires java.sql;        // module นี้ต้องการใช้ java.sql (ทบทวน JDBC จาก Part 64)
    requires java.logging;

    exports com.example.api;   // เปิดเผย package นี้ให้ module อื่นใช้ได้
    // com.example.internal ไม่ได้ exports -> ใช้ได้แค่ภายใน module นี้เท่านั้น
    // แม้ class ใน internal package จะเป็น public ก็ตาม! (นี่คือ "Strong Encapsulation" - หัวข้อ 6)
}
```

## 5. `requires` และ `exports`

```java
// module-info.java ของ module "com.example.api"
module com.example.api {
    exports com.example.api.service; // เปิดเผยให้ทุก module ที่ requires เห็นได้

    exports com.example.api.internal to com.example.trusted.module;
    // "qualified exports": เปิดเผยเฉพาะให้ module ที่ระบุชื่อเห็นเท่านั้น (ไม่ใช่ทุกคน)

    requires transitive java.sql;
    // transitive: module ที่ requires "com.example.api" จะได้ requires java.sql ต่ออัตโนมัติด้วย
    // (คล้ายแนวคิด "api" configuration ของ Gradle - ทบทวนจาก Part 62)
}
```

```java
// module-info.java ของ module ที่ต้องการใช้ com.example.api
module com.example.consumer {
    requires com.example.api; // ต้องประกาศก่อนถึงจะใช้ package ที่ com.example.api exports ได้
}
```

**การ compile และรันด้วย module path** (แทน classpath แบบเดิม):

```bash
javac -d out --module-path libs -m com.example.myapp/com.example.Main
java --module-path out:libs -m com.example.myapp/com.example.Main
```

## 6. Strong Encapsulation: ประโยชน์หลักของ Module

**ประโยชน์ที่สำคัญที่สุดของ JPMS**: ทำให้ **`public` แต่ไม่ `exports` = ใช้
ได้แค่ภายใน module เท่านั้น** — นี่คือ **"Strong Encapsulation"** ที่แก้
ปัญหาจาก Part 13 (Encapsulation แบบเดิมทำได้แค่ระดับ class ไม่ใช่ระดับ JAR)

```java
package com.example.internal;

public class InternalHelper { // เป็น public แต่อยู่ใน package ที่ไม่ได้ exports
    public static void doWork() {
        System.out.println("ทำงานภายใน");
    }
}
```

```java
// จาก module อื่นที่ requires com.example.myapp
public class ExternalCode {
    public static void main(String[] args) {
        // com.example.internal.InternalHelper.doWork();
        // Compile Error! IllegalAccessError ตอน runtime แม้ class จะเป็น public!
        // เพราะ package "com.example.internal" ไม่ได้ exports จาก module ต้นทาง
    }
}
```

**เปรียบเทียบกับ Reflection (ทบทวนจาก Part 53)**: ก่อน JPMS การใช้
`setAccessible(true)` สามารถเจาะทะลุ `private` ได้เสมอ — JPMS เพิ่มการ
ป้องกันอีกชั้นที่**Reflection ก็เจาะทะลุไม่ได้ด้วย**ถ้า module ไม่ได้
`opens` package นั้นให้ (มี directive `opens` แยกจาก `exports` สำหรับกรณีนี้
โดยเฉพาะ — framework อย่าง Spring/Hibernate ที่ใช้ reflection หนัก ต้องพึ่งพา
`opens` directive)

## 7. Module Types: Named, Automatic, Unnamed

| ชนิด Module | มี `module-info.java`? | ลักษณะ |
|---|---|---|
| **Named Module** | ✅ มี | Module ที่นิยามอย่างสมบูรณ์ (ตามหัวข้อ 4) |
| **Automatic Module** | ❌ ไม่มี (แต่อยู่บน module path) | JAR เก่าที่ยังไม่ได้ปรับเป็น module — JPMS สร้าง module ให้อัตโนมัติ (exports ทุก package) |
| **Unnamed Module** | ❌ ไม่มี (อยู่บน classpath แบบเดิม) | โค้ดที่ยังใช้ classpath แบบเดิม (ทบทวนจาก Part 20) — เข้าถึงทุกอย่างได้ (ไม่มี encapsulation) |

**นี่คือกลไก backward compatibility ที่สำคัญมาก**: โปรแกรม Java ที่เขียนมา
ก่อน Java 9 (ส่วนใหญ่ในโลก) **ยังรันได้ตามปกติ**บน JVM ใหม่ผ่าน Unnamed
Module และ Automatic Module — ไม่ต้อง migrate ทั้งหมดทันที (การ migrate เป็น
กระบวนการที่ทำได้แบบค่อยเป็นค่อยไป)

## 8. `jlink`: สร้าง Custom Runtime Image

**`jlink`** ใช้สร้าง**runtime image ที่กำหนดเอง** — รวมเฉพาะ module ที่
แอปพลิเคชันต้องการจริง ๆ (ไม่ต้องมี JRE เต็มรูปแบบทั้งหมด) ทำให้ได้
**ขนาดเล็กกว่ามาก** เหมาะกับ container/cloud deployment (ปูทางสู่ Part 95 —
Docker)

```bash
jlink --module-path $JAVA_HOME/jmods:out \
      --add-modules com.example.myapp \
      --output myapp-runtime \
      --strip-debug --compress=2 --no-header-files --no-man-pages

# ผลลัพธ์: โฟลเดอร์ myapp-runtime/ ที่มี Java runtime แบบ minimal + แอปพลิเคชันของเรา
./myapp-runtime/bin/java -m com.example.myapp/com.example.Main
```

**ประโยชน์สำหรับ Docker image (Part 95)**: image ขนาดเล็กกว่า deploy ได้เร็ว
กว่า และมี**attack surface**เล็กกว่า (มี code น้อยกว่า = ช่องโหว่ด้านความ
ปลอดภัยน้อยกว่า — ทบทวนแนวคิดจาก Part 101)

## 9. เหตุผลที่ JPMS ยังไม่แพร่หลายเท่าที่คาด

แม้ JPMS จะแก้ปัญหาสำคัญหลายอย่าง แต่**ยังไม่ถูกใช้อย่างกว้างขวางในโปรเจกต์
จริงส่วนใหญ่** เพราะ:

1. **Migration cost สูง**: โปรเจกต์เก่าจำนวนมากใช้ library ที่ยังไม่รองรับ
   module (Automatic Module อาจมีปัญหาเรื่อง naming ที่ไม่แน่นอน)
2. **Framework สำคัญหลายตัวยังไม่บังคับใช้**: Spring Boot (Part 75) ทำงาน
   ได้ดีบน classpath แบบเดิม ไม่บังคับให้ใช้ module — ทำให้แรงจูงใจในการ
   migrate ลดลง
3. **ความซับซ้อนเพิ่มขึ้น**: ต้องเรียนรู้ syntax ใหม่ (`requires`, `exports`,
   `opens`) และแก้ปัญหา reflection ที่ framework บางตัวใช้หนัก

**สถานะปัจจุบัน**: JPMS ใช้มากใน**JDK เอง** (JDK ถูกแบ่งเป็น module ตั้งแต่
Java 9 — `jlink` ทำงานได้เพราะเหตุนี้) แต่ในระดับ application ส่วนใหญ่ยัง
**เลือกไม่ใช้** (อยู่บน Unnamed Module ผ่าน classpath) — ควรรู้จักแนวคิดนี้
ไว้ (สำคัญสำหรับเข้าใจ JDK เอง และมีประโยชน์กับโปรเจกต์ใหม่ที่ต้องการ strong
encapsulation จริงจัง) แต่ไม่ต้องรู้สึกว่า "ต้องใช้เสมอ" ในทุกโปรเจกต์

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียน `module-info.java` สำหรับ module "com.example.payment" ที่
`requires` `java.sql` และ exports เฉพาะ package `com.example.payment.api`
(ไม่ exports `com.example.payment.internal`)

**เฉลย:**

```java
module com.example.payment {
    requires java.sql;
    exports com.example.payment.api;
}
```

**2)** อธิบายว่าทำไม `public class` ใน package ที่ไม่ได้ `exports` ยังคง
เข้าถึงไม่ได้จากภายนอก module ทั้งที่ก่อน JPMS `public` เข้าถึงได้จากทุกที่

**เฉลย**: ก่อน JPMS (ทบทวน access modifier จาก Part 13) `public` หมายถึง
"เข้าถึงได้จากทุกที่ในโปรแกรม" อย่างแท้จริง เพราะไม่มีขอบเขตที่ใหญ่กว่า
package ให้ควบคุม — JPMS เพิ่มขอบเขตใหม่ (module) ที่**อยู่เหนือ access
modifier ระดับ class** โดย module สามารถเลือก**ไม่ exports** package ใด
package หนึ่ง ทำให้แม้ class ใน package นั้นจะเป็น `public` (เข้าถึงได้ภายใน
module เดียวกันตามปกติ) แต่**module อื่นที่ requires module นี้จะไม่เห็น
package นั้นเลย** — เป็นการเพิ่มชั้นการควบคุมการเข้าถึงที่ granular กว่า
access modifier แบบเดิม (Strong Encapsulation ตามหัวข้อ 6)

**3)** อธิบายว่าทำไมโปรเจกต์เก่าที่เขียนก่อน Java 9 ยังรันได้ปกติบน Java
เวอร์ชันใหม่ ทั้งที่ไม่มี `module-info.java`

**เฉลย**: เพราะ JPMS ออกแบบมาให้**backward compatible**ผ่านแนวคิด **Unnamed
Module** — โค้ดที่วางอยู่บน classpath แบบเดิม (ไม่ใช่ module path) จะถูก
รวมเข้าเป็น Unnamed Module โดยอัตโนมัติ ซึ่ง**ไม่มีการบังคับ encapsulation
ใด ๆ** (ทำงานเหมือนระบบ classpath ดั้งเดิมก่อน Java 9 ทุกประการ) ทำให้
โปรเจกต์เก่าที่ compile ด้วย `javac` แบบเดิมและรันด้วย `-cp` (classpath)
ยังคงทำงานได้ปกติโดยไม่ต้องแก้ไขอะไรเลย — นี่คือการตัดสินใจออกแบบที่สำคัญของ
ทีม OpenJDK เพื่อไม่ทำให้ ecosystem ของ Java ที่มีมานานพังทลายไปในทันที

### สรุปเนื้อหา Part 69

- Classpath แบบเดิมไม่มี encapsulation ระดับ JAR และเสี่ยง version conflict
  (JAR Hell)
- JPMS (Java 9+) เพิ่มแนวคิด Module ที่นิยามด้วย `module-info.java`
- `requires` ประกาศ dependency, `exports` เปิดเผย package ให้ module อื่นใช้
- Strong Encapsulation: `public` แต่ไม่ `exports` = ใช้ได้แค่ภายใน module
  เท่านั้น (เจาะทะลุด้วย reflection ก็ไม่ได้ถ้าไม่ `opens`)
- Named/Automatic/Unnamed Module ทำให้ backward compatibility สมบูรณ์
- `jlink` สร้าง custom runtime image ขนาดเล็ก เหมาะกับ container deployment
- JPMS ใช้มากใน JDK เอง แต่ยังไม่แพร่หลายในระดับ application เพราะ migration
  cost และ framework ส่วนใหญ่ไม่บังคับใช้

**ต่อไป**: [Part 70 — Networking: Socket Programming, JSON ด้วย Jackson/Gson](./part-070-networking-json.md)
