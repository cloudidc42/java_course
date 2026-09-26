# Part 1: บทนำสู่ Java, JDK/JRE/JVM และการติดตั้งเครื่องมือพัฒนา

> ขั้นตอนที่ 1-10 ของหลักสูตร | ระดับ: พื้นฐานที่สุด (ไม่ต้องมีพื้นฐานมาก่อน)

## สารบัญ

1. Java คืออะไร ทำไมต้องเรียน
2. ประวัติและปรัชญาของภาษา Java ("Write Once, Run Anywhere")
3. Java ใช้ทำอะไรได้บ้างในโลกจริง
4. ทำความเข้าใจ JDK, JRE, JVM
5. การติดตั้ง JDK (Windows / macOS / Linux)
6. การติดตั้งและตั้งค่า IDE (IntelliJ IDEA / VS Code)
7. โปรแกรมแรก: Hello World
8. การคอมไพล์และรันโปรแกรมด้วยคำสั่ง command line
9. ทำความเข้าใจ Bytecode และการทำงานของ JVM
10. แบบฝึกหัดท้ายบทและสรุป

---

## 1. Java คืออะไร ทำไมต้องเรียน

**Java** เป็นภาษาโปรแกรมเชิงวัตถุ (Object-Oriented Programming: OOP) ที่ถูกออกแบบมาให้
"เขียนครั้งเดียว รันได้ทุกที่" (Write Once, Run Anywhere - WORA) สร้างโดยบริษัท Sun
Microsystems เมื่อปี ค.ศ. 1995 (ปัจจุบันเป็นของ Oracle) และยังคงเป็นหนึ่งในภาษาโปรแกรม
ที่มีคนใช้งานมากที่สุดในโลกจนถึงปัจจุบัน

เหตุผลที่ควรเรียน Java:

- **ตลาดงานกว้าง**: บริษัทระดับโลกอย่าง Google, Netflix, LinkedIn, Amazon, ธนาคาร
  และองค์กรขนาดใหญ่เกือบทั้งหมดใช้ Java เป็น backbone ของระบบ
- **Ecosystem แข็งแกร่ง**: มี framework ระดับโลกอย่าง Spring, Hibernate และเครื่องมือ
  สนับสนุนครบวงจร (Maven, Gradle, JUnit)
- **Performance ดี**: JVM สมัยใหม่มี JIT Compiler ที่ทำให้ Java เร็วใกล้เคียงภาษา native
- **Portable**: โปรแกรม Java compile เป็น bytecode ที่รันได้บนทุกแพลตฟอร์มที่มี JVM
- **Strongly Typed & Robust**: ภาษาที่เข้มงวดเรื่องชนิดข้อมูล ช่วยลด bug ตั้งแต่ compile-time
- **Backward Compatible**: โค้ด Java ที่เขียนเมื่อ 20 ปีก่อนส่วนใหญ่ยังรันได้บน JVM ปัจจุบัน

## 2. ประวัติและปรัชญาของภาษา Java

Java ถูกพัฒนาโดยทีมของ **James Gosling** ที่ Sun Microsystems เริ่มต้นในปี 1991
ภายใต้ชื่อโปรเจกต์ "Oak" มีเป้าหมายเดิมคือใช้ควบคุมอุปกรณ์อิเล็กทรอนิกส์ ก่อนจะถูกปรับมาใช้
กับอินเทอร์เน็ตที่กำลังเติบโตในขณะนั้น และเปิดตัวอย่างเป็นทางการในชื่อ "Java" ปี 1995

หลักการออกแบบ 5 ข้อของ Java:

1. **Simple** — เรียบง่าย ตัดความซับซ้อนที่ไม่จำเป็นออกจาก C++ (เช่น pointer arithmetic)
2. **Object-Oriented** — ทุกอย่างในโปรแกรมออกแบบเป็น object (ยกเว้น primitive types)
3. **Robust** — ตรวจสอบข้อผิดพลาดตั้งแต่ compile-time เข้มงวดเรื่อง type-checking
4. **Platform Independent** — compile เป็น bytecode รันบน JVM ได้ทุก OS
5. **Secure** — มีกลไกความปลอดภัยในตัว เช่น bytecode verifier, security manager

เวอร์ชันสำคัญที่ควรรู้:

| เวอร์ชัน | ปี | ฟีเจอร์เด่น |
|---|---|---|
| Java 8 | 2014 | Lambda Expressions, Stream API, Optional (จุดเปลี่ยนสำคัญ) |
| Java 11 (LTS) | 2018 | var (local variable type inference), HTTP Client ใหม่ |
| Java 17 (LTS) | 2021 | Records, Sealed Classes, Pattern Matching for switch |
| Java 21 (LTS) | 2023 | Virtual Threads, Record Patterns, Sequenced Collections |

หลักสูตรนี้จะใช้ **Java 21 (LTS)** เป็นหลัก เนื่องจากเป็นเวอร์ชัน Long-Term Support ล่าสุด
ที่นิยมใช้งานจริงในองค์กร และรองรับฟีเจอร์ทันสมัยครบถ้วน

## 3. Java ใช้ทำอะไรได้บ้างในโลกจริง

- **Enterprise Web Applications**: ระบบธนาคาร, ระบบ ERP, E-Commerce ด้วย Spring Boot
- **Android Development**: แอปมือถือ Android (ใช้ Java/Kotlin)
- **Big Data**: Hadoop, Spark เขียนด้วย Java/Scala บน JVM
- **Desktop Applications**: JavaFX, Swing
- **Embedded Systems & IoT**
- **Microservices & Cloud-Native Applications**
- **Game Development**: เช่น Minecraft เขียนด้วย Java (เวอร์ชันดั้งเดิม)

## 4. ทำความเข้าใจ JDK, JRE, JVM

นี่คือ 3 คำที่มือใหม่มักสับสนที่สุด มาดูความสัมพันธ์กัน:

```
┌─────────────────────────────────────────────┐
│                     JDK                      │  <- ใช้สำหรับ "พัฒนา" โปรแกรม
│  (Java Development Kit)                      │
│  ┌─────────────────────────────────────────┐ │
│  │                  JRE                     │ │  <- ใช้สำหรับ "รัน" โปรแกรม
│  │  (Java Runtime Environment)               │ │
│  │  ┌─────────────────────────────────────┐ │ │
│  │  │              JVM                     │ │ │  <- เครื่องเสมือนที่รัน bytecode
│  │  │  (Java Virtual Machine)              │ │ │
│  │  └─────────────────────────────────────┘ │ │
│  │   + Core Libraries (java.lang, java.util) │ │
│  └─────────────────────────────────────────┘ │
│   + Compiler (javac), Debugger, javadoc, jar  │
└─────────────────────────────────────────────┘
```

- **JVM (Java Virtual Machine)**: เครื่องเสมือนที่ทำหน้าที่รัน bytecode (.class files)
  JVM มีเวอร์ชันเฉพาะสำหรับแต่ละ OS (Windows, macOS, Linux) นี่คือสิ่งที่ทำให้ Java
  "Write Once, Run Anywhere" — เราไม่ต้อง compile ใหม่สำหรับแต่ละ OS เพราะ JVM
  จะแปล bytecode เป็นเครื่องคำสั่งของแต่ละเครื่องให้เอง
- **JRE (Java Runtime Environment)**: คือ JVM + Standard Library (เช่น `java.lang`,
  `java.util`) ที่จำเป็นสำหรับการ**รัน**โปรแกรม Java แต่ไม่มีเครื่องมือสำหรับพัฒนา
- **JDK (Java Development Kit)**: คือ JRE + เครื่องมือสำหรับ**พัฒนา**โปรแกรม เช่น
  - `javac` — ตัวคอมไพเลอร์ (แปลง `.java` เป็น `.class`)
  - `java` — ตัวรันโปรแกรม
  - `javadoc` — สร้างเอกสารจาก comment
  - `jar` — บีบอัดไฟล์ .class เป็น .jar
  - `jshell` — REPL สำหรับทดลองรันโค้ด Java แบบ interactive

> ในฐานะผู้พัฒนา (developer) เราต้องติดตั้ง **JDK** เสมอ เพราะรวมทุกอย่างไว้แล้ว

## 5. การติดตั้ง JDK

หลักสูตรนี้แนะนำให้ใช้ **Eclipse Temurin (Adoptium) JDK 21** ซึ่งเป็น OpenJDK
distribution ที่ฟรี เสถียร และนิยมใช้ในองค์กร

### Windows

1. ดาวน์โหลด installer จาก https://adoptium.net (เลือก JDK 21 LTS, Windows, x64, .msi)
2. รัน installer และเลือก "Set JAVA_HOME variable" และ "Add to PATH" ระหว่างติดตั้ง
3. เปิด Command Prompt ใหม่ แล้วตรวจสอบด้วยคำสั่ง:

```bash
java -version
javac -version
```

ควรได้ผลลัพธ์ประมาณ:

```
java version "21.0.x" 2024-xx-xx LTS
javac 21.0.x
```

### macOS

ใช้ [Homebrew](https://brew.sh) ติดตั้งง่ายที่สุด:

```bash
brew install --cask temurin21
java -version
```

### Linux (Ubuntu/Debian)

```bash
sudo apt update
sudo apt install -y openjdk-21-jdk
java -version
javac -version
```

### การตั้งค่า JAVA_HOME (Linux/macOS)

เพิ่มบรรทัดนี้ในไฟล์ `~/.bashrc` หรือ `~/.zshrc`:

```bash
export JAVA_HOME=$(dirname $(dirname $(readlink -f $(which javac))))
export PATH=$JAVA_HOME/bin:$PATH
```

จากนั้นรัน `source ~/.bashrc` (หรือ `~/.zshrc`) แล้วตรวจสอบด้วย `echo $JAVA_HOME`

## 6. การติดตั้งและตั้งค่า IDE

แนะนำ 2 ตัวเลือก:

### ตัวเลือกที่ 1: IntelliJ IDEA Community Edition (แนะนำสำหรับหลักสูตรนี้)

- ดาวน์โหลดฟรีที่ https://www.jetbrains.com/idea/download/
- เป็น IDE ที่ออกแบบมาสำหรับ Java โดยเฉพาะ มี auto-complete, refactoring,
  debugger ที่ทรงพลังที่สุดในบรรดา IDE ทั้งหมด
- เมื่อเปิดครั้งแรก: New Project → Java → เลือก JDK 21 ที่ติดตั้งไว้

### ตัวเลือกที่ 2: Visual Studio Code + Extension Pack for Java

- ติดตั้ง VS Code แล้วเพิ่ม extension "Extension Pack for Java" (โดย Microsoft)
- เหมาะกับผู้ที่คุ้นเคยกับ VS Code อยู่แล้วและต้องการ IDE ที่เบากว่า

ตลอดหลักสูตรนี้ ตัวอย่างโค้ดสามารถใช้ได้กับทั้งสอง IDE หรือแม้แต่ text editor ธรรมดา
ร่วมกับ command line ก็ได้

## 7. โปรแกรมแรก: Hello World

สร้างไฟล์ชื่อ `HelloWorld.java` (**ชื่อไฟล์ต้องตรงกับชื่อ public class เป๊ะ ๆ**
รวมถึงตัวพิมพ์เล็ก-ใหญ่):

```java
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello, World!");
        System.out.println("ยินดีต้อนรับสู่โลกของ Java!");
    }
}
```

### อธิบายทีละส่วน

- `public class HelloWorld` — ประกาศคลาสชื่อ `HelloWorld` ที่เข้าถึงได้จากทุกที่ (public)
  ใน Java ทุกโปรแกรมต้องอยู่ภายใน class อย่างน้อยหนึ่งคลาส
- `public static void main(String[] args)` — นี่คือ **entry point** หรือจุดเริ่มต้น
  การทำงานของโปรแกรม Java ทุกโปรแกรม JVM จะมองหาเมธอดนี้เพื่อเริ่มรัน:
  - `public` — เข้าถึงได้จากภายนอก (JVM ต้องเรียกมันได้)
  - `static` — ไม่ต้องสร้าง object ของคลาสก่อนก็เรียกได้
  - `void` — ไม่มีการคืนค่ากลับ
  - `main` — ชื่อเมธอดที่ JVM กำหนดตายตัว ห้ามเปลี่ยน
  - `String[] args` — พารามิเตอร์รับค่า argument จาก command line เป็น array ของ String
- `System.out.println(...)` — คำสั่งพิมพ์ข้อความออกทาง console แล้วขึ้นบรรทัดใหม่
  (`System.out.print(...)` จะไม่ขึ้นบรรทัดใหม่)

## 8. การคอมไพล์และรันโปรแกรมด้วยคำสั่ง command line

เปิด terminal ไปยังโฟลเดอร์ที่มีไฟล์ `HelloWorld.java` แล้วรัน:

```bash
# ขั้นตอนที่ 1: คอมไพล์ .java -> .class (bytecode)
javac HelloWorld.java

# จะได้ไฟล์ HelloWorld.class ในโฟลเดอร์เดียวกัน

# ขั้นตอนที่ 2: รันโปรแกรมด้วย JVM (ไม่ต้องใส่ .class ต่อท้าย)
java HelloWorld
```

ผลลัพธ์:

```
Hello, World!
ยินดีต้อนรับสู่โลกของ Java!
```

### เกร็ดความรู้: Single-File Source-Code Program (Java 11+)

ตั้งแต่ Java 11 เป็นต้นไป เราสามารถรันไฟล์ `.java` ได้โดยตรงโดยไม่ต้อง compile ก่อน
(เหมาะสำหรับสคริปต์เล็ก ๆ หรือทดลองโค้ด):

```bash
java HelloWorld.java
```

คำสั่งนี้จะ compile ในหน่วยความจำแล้วรันทันที โดยไม่สร้างไฟล์ `.class` ทิ้งไว้

## 9. ทำความเข้าใจ Bytecode และการทำงานของ JVM

เมื่อรันคำสั่ง `javac HelloWorld.java` คอมไพเลอร์จะแปลงโค้ดต้นฉบับ (source code)
เป็น **bytecode** ซึ่งเก็บในไฟล์ `HelloWorld.class` — bytecode ไม่ใช่ machine code
ของ CPU โดยตรง แต่เป็นภาษากลางที่ JVM เข้าใจ

ขั้นตอนการทำงานทั้งหมด:

```
HelloWorld.java  --(javac)-->  HelloWorld.class (bytecode)  --(java/JVM)-->  ผลลัพธ์
     ↑ Source Code                    ↑ Platform-independent      ↑ Platform-specific
                                        รันบน JVM ได้ทุก OS          JVM แปลเป็นคำสั่งจริง
                                                                      ของ CPU เครื่องนั้น ๆ
```

ภายใน JVM มีองค์ประกอบสำคัญ 3 ส่วน:

1. **Class Loader** — โหลดไฟล์ `.class` เข้าสู่หน่วยความจำ
2. **Runtime Data Area** — พื้นที่หน่วยความจำ แบ่งเป็น Heap, Stack, Method Area ฯลฯ
   (จะอธิบายลึกใน Part 67 เรื่อง JVM Internals)
3. **Execution Engine** — ตัวประมวลผล bytecode มี 2 แบบทำงานร่วมกัน:
   - **Interpreter** — แปล bytecode ทีละบรรทัดแล้วรันทันที (เริ่มทำงานได้เร็ว)
   - **JIT Compiler (Just-In-Time)** — แปล bytecode ส่วนที่ถูกเรียกใช้บ่อย ๆ
     ("hot code") ให้เป็น native machine code เพื่อความเร็วสูงสุด

นี่คือเหตุผลที่ Java ได้ทั้ง "portability" (รันได้ทุกที่) และ "performance" (เร็ว)
ในเวลาเดียวกัน

คุณสามารถลองดู bytecode ที่ถูกสร้างได้ด้วยคำสั่ง:

```bash
javap -c HelloWorld.class
```

ซึ่งจะแสดง bytecode instructions ในรูปแบบที่มนุษย์อ่านได้ (เช่น `getstatic`,
`invokevirtual`) — ยังไม่ต้องเข้าใจตอนนี้ทั้งหมด แค่รู้ว่ามันมีอยู่จริงก็พอ

## 10. แบบฝึกหัดท้ายบทและสรุป

### แบบฝึกหัด

**1)** เขียนโปรแกรม `MyProfile.java` ที่พิมพ์ชื่อ, อายุ, และเป้าหมายในการเรียน Java
ของตัวเอง ออกทาง console คนละ 1 บรรทัด

**เฉลยตัวอย่าง:**

```java
public class MyProfile {
    public static void main(String[] args) {
        System.out.println("ชื่อ: สมชาย ใจดี");
        System.out.println("อายุ: 25");
        System.out.println("เป้าหมาย: อยากเป็น Java Backend Developer มืออาชีพ");
    }
}
```

**2)** ทดลองเปลี่ยนชื่อไฟล์เป็น `Hello.java` โดยที่ยังคงชื่อคลาสเป็น `HelloWorld`
แล้วลอง compile ดูว่าเกิดอะไรขึ้น (คำใบ้: compiler จะแจ้ง error เพราะชื่อไฟล์ต้องตรงกับ
public class เสมอ)

**3)** ลองรันคำสั่ง `javap -c HelloWorld.class` แล้วสังเกตว่ามี bytecode instruction
กี่บรรทัด ลองเปรียบเทียบกับตอนที่เพิ่มบรรทัด `println` เข้าไปอีกบรรทัดหนึ่ง

### สรุปเนื้อหา Part 1

- Java เป็นภาษา OOP ที่ออกแบบให้ "Write Once, Run Anywhere" ผ่าน JVM
- **JDK** = เครื่องมือพัฒนา (มี compiler), **JRE** = สภาพแวดล้อมการรัน, **JVM** =
  เครื่องเสมือนที่รัน bytecode
- โปรแกรม Java ทุกตัวต้องมี entry point คือ `public static void main(String[] args)`
- ขั้นตอนคือ เขียนโค้ด (`.java`) → คอมไพล์ด้วย `javac` (`.class` bytecode) →
  รันด้วย `java`
- หลักสูตรนี้ใช้ **Java 21 (LTS)** เป็นหลัก

**ต่อไป**: [Part 2 — โครงสร้างโปรแกรม Java, การคอมไพล์และรัน, กฎการตั้งชื่อ](./part-002-program-structure.md)
