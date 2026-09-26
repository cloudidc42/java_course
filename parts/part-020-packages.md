# Part 20: Packages, การจัดระเบียบโปรเจกต์, Classpath พื้นฐาน

> ขั้นตอนที่ 191-200 ของหลักสูตร | ระดับ: OOP ขั้นกลาง (จบหมวดพื้นฐาน)

## สารบัญ

1. ทบทวนและขยายความ Package
2. โครงสร้างโปรเจกต์ Java มาตรฐาน (Maven/Gradle Layout)
3. Classpath คืออะไร
4. การคอมไพล์และรันโปรเจกต์ที่มีหลาย Package
5. การสร้างและใช้งาน JAR File เบื้องต้น
6. Package Naming Convention ระดับองค์กร
7. `package-info.java`
8. Circular Dependency ระหว่าง Package (ปัญหาที่ควรหลีกเลี่ยง)
9. หลักการจัดโครงสร้างโปรเจกต์ที่ดี
10. แบบฝึกหัดและสรุป

---

## 1. ทบทวนและขยายความ Package

จาก Part 2 เราเรียนรู้ว่า package คือกลไกจัดกลุ่มคลาส Part นี้จะขยายความเรื่อง
การจัดโครงสร้างโปรเจกต์ระดับใหญ่ที่ใช้งานจริง

```
com.example.ecommerce
├── controller     <- จัดการ HTTP request (จะเรียนใน Part 78)
├── service        <- business logic
├── repository     <- เข้าถึงฐานข้อมูล (Part 64-65, 79)
├── model / entity <- โครงสร้างข้อมูล
├── dto            <- Data Transfer Object
├── exception      <- custom exceptions
└── util           <- utility classes
```

การแบ่ง package ตาม**ชั้นสถาปัตยกรรม (layer)** แบบนี้เรียกว่า **Layered
Architecture** เป็นรูปแบบที่นิยมที่สุดในโปรเจกต์ Java/Spring Boot

## 2. โครงสร้างโปรเจกต์ Java มาตรฐาน (Maven/Gradle Layout)

เมื่อใช้ build tool อย่าง Maven หรือ Gradle (Part 61-62) โปรเจกต์จะมีโครงสร้าง
มาตรฐานที่เรียกว่า **Standard Directory Layout**:

```
my-project/
├── pom.xml (หรือ build.gradle)
├── src/
│   ├── main/
│   │   ├── java/                    <- source code หลัก
│   │   │   └── com/example/myapp/
│   │   │       ├── Application.java
│   │   │       ├── model/
│   │   │       ├── service/
│   │   │       └── repository/
│   │   └── resources/                <- ไฟล์ config, properties, static resources
│   │       └── application.properties
│   └── test/
│       ├── java/                     <- unit test code (โครงสร้าง package ตรงกับ main)
│       │   └── com/example/myapp/
│       │       └── service/
│       └── resources/                <- test resources
└── target/ (หรือ build/)             <- ไฟล์ที่ compile แล้ว (สร้างอัตโนมัติ ไม่ commit ลง git)
```

**ข้อดีของโครงสร้างมาตรฐาน**: ทุก IDE และ build tool รู้จักโครงสร้างนี้อยู่แล้ว
ทำให้ทีมงานใหม่เข้าใจโปรเจกต์ได้ทันทีโดยไม่ต้องอ่านเอกสารเพิ่ม (convention over
configuration)

## 3. Classpath คืออะไร

**Classpath** คือรายการ**ตำแหน่ง**ที่ JVM ใช้ค้นหาไฟล์ `.class` (หรือ `.jar`)
เมื่อต้องโหลดคลาสมาใช้งาน — ถ้า JVM หาคลาสที่ต้องการไม่เจอใน classpath จะเกิด
`ClassNotFoundException` หรือ `NoClassDefFoundError`

```bash
# ระบุ classpath ด้วย -cp หรือ -classpath
java -cp bin com.example.Main

# ระบุหลายที่ คั่นด้วย : (Linux/macOS) หรือ ; (Windows)
java -cp "bin:libs/gson.jar:libs/junit.jar" com.example.Main   # Linux/macOS
java -cp "bin;libs/gson.jar;libs/junit.jar" com.example.Main   # Windows

# คอมไพล์โดยระบุ output directory และ classpath ของ dependency
javac -cp "libs/gson.jar" -d bin src/com/example/Main.java
```

```
ตัวอย่างการค้นหา: java -cp "bin:libs/gson.jar" com.example.Main

JVM มองหา com.example.Main.class ที่ไหนบ้าง:
1. bin/com/example/Main.class          <- หาเจอที่นี่ก่อน (compiled classes)
2. libs/gson.jar (ภายในไฟล์ jar)        <- หาต่อถ้ายังไม่เจอ (external library)
```

ในทางปฏิบัติปัจจุบัน แทบไม่มีใครจัดการ classpath ด้วยมือแล้ว เพราะ build tool
(Maven/Gradle) และ IDE จัดการให้อัตโนมัติทั้งหมด — แต่การเข้าใจแนวคิดนี้ยังสำคัญ
มากเมื่อต้อง debug ปัญหา `ClassNotFoundException` ในโปรดักชัน

## 4. การคอมไพล์และรันโปรเจกต์ที่มีหลาย Package

```
src/
└── com/
    └── example/
        ├── Main.java
        ├── model/
        │   └── Student.java
        └── service/
            └── StudentService.java
```

```java
// src/com/example/model/Student.java
package com.example.model;

public class Student {
    public String name;
    public Student(String name) { this.name = name; }
}
```

```java
// src/com/example/service/StudentService.java
package com.example.service;

import com.example.model.Student; // import จาก package อื่น

public class StudentService {
    public void printGreeting(Student student) {
        System.out.println("ยินดีต้อนรับ " + student.name);
    }
}
```

```java
// src/com/example/Main.java
package com.example;

import com.example.model.Student;
import com.example.service.StudentService;

public class Main {
    public static void main(String[] args) {
        Student student = new Student("Alice");
        StudentService service = new StudentService();
        service.printGreeting(student);
    }
}
```

คำสั่ง compile และรัน (จาก root ของ `src/`):

```bash
# คอมไพล์ทุกไฟล์ .java พร้อมกัน ให้ output ไปที่โฟลเดอร์ bin (-d = destination)
javac -d bin com/example/Main.java com/example/model/Student.java com/example/service/StudentService.java

# หรือใช้ wildcard แบบ recursive (find + javac ผสมกัน)
find . -name "*.java" | xargs javac -d bin

# รันโปรแกรม โดยระบุ full qualified name ของคลาสที่มี main()
java -cp bin com.example.Main
```

## 5. การสร้างและใช้งาน JAR File เบื้องต้น

**JAR (Java ARchive)** คือไฟล์บีบอัด (คล้าย ZIP) ที่รวม `.class` files, resources,
และ metadata เข้าด้วยกัน ใช้สำหรับแจกจ่ายโปรแกรมหรือ library

```bash
# สร้าง JAR ธรรมดา (ไม่ระบุ entry point)
jar cf myapp.jar -C bin .

# สร้าง Manifest file ที่ระบุ Main-Class เพื่อให้รันได้โดยตรง
# ไฟล์ manifest.txt:
# Main-Class: com.example.Main

jar cfm myapp.jar manifest.txt -C bin .

# รัน JAR ที่มี Main-Class ระบุไว้แล้ว
java -jar myapp.jar

# ดูเนื้อหาภายใน JAR
jar tf myapp.jar

# แตก JAR ออกมาดู
jar xf myapp.jar
```

ในโปรเจกต์ที่ใช้ Maven/Gradle การสร้าง JAR จะทำผ่านคำสั่ง `mvn package` หรือ
`gradle build` โดยอัตโนมัติ (Part 61-62)

## 6. Package Naming Convention ระดับองค์กร

ธรรมเนียมสากลคือใช้ **domain name กลับด้าน** เป็นฐาน:

```
บริษัทเจ้าของ domain "example.com" แผนก engineering ทำโปรเจกต์ "shop-service":
→ com.example.shop

หน่วยงานราชการ/การศึกษาไทย domain "ac.th" หรือ "go.th":
→ th.ac.university.projectname
→ th.go.ministry.systemname

Open source project บน GitHub (ไม่มี domain องค์กร):
→ io.github.username.projectname
```

**กฎการตั้งชื่อ package**:
- ตัวพิมพ์เล็กทั้งหมดเสมอ (ไม่ผสมตัวใหญ่)
- ห้ามใช้ keyword สงวนของ Java (เช่น `com.example.class` ผิด เพราะ `class`
  เป็น keyword)
- แต่ละ "." คือระดับโฟลเดอร์หนึ่งชั้น

## 7. `package-info.java`

ไฟล์พิเศษที่ใช้เพิ่ม **Javadoc ระดับ package** และ **package-level annotations**
(เช่น กำหนดว่าทุกคลาสใน package นี้ควรมี `@NonNull` เป็นค่าเริ่มต้น) — ไม่มี class
declaration ปกติ มีแค่ package statement และ Javadoc comment

```java
/**
 * แพ็กเกจนี้รวมคลาสทั้งหมดที่เกี่ยวกับการจัดการคำสั่งซื้อ (Order)
 * รวมถึง entity, service, และ repository ของระบบคำสั่งซื้อ
 *
 * @since 1.0
 */
package com.example.order;
```

เมื่อรัน `javadoc` เอกสารระดับ package นี้จะปรากฏเป็นหน้าแรกของแต่ละ package
ในเอกสาร HTML ที่สร้างขึ้น

## 8. Circular Dependency ระหว่าง Package (ปัญหาที่ควรหลีกเลี่ยง)

**Circular Dependency** เกิดเมื่อ package A พึ่งพา package B และ package B ก็
พึ่งพา package A กลับมาด้วย — Java **อนุญาตให้ compile ได้** (ไม่เหมือนบางภาษา
ที่ปฏิเสธทันที) แต่เป็น**สัญญาณของการออกแบบสถาปัตยกรรมที่ไม่ดี**:

```
com.example.order  ──depends on──>  com.example.customer
com.example.customer  ──depends on──>  com.example.order   (วนกลับมา! ปัญหา)
```

**ผลเสียของ circular dependency**:
- ทดสอบแยกส่วน (unit test) ยากขึ้น เพราะทั้งสอง package ผูกติดกันแน่น
- เปลี่ยนแปลงโค้ดใน package หนึ่งมีผลกระทบลูกโซ่ไปยังอีก package เสมอ
- บ่งบอกว่าอาจต้องแยก responsibility ให้ชัดเจนกว่านี้ (อาจต้องดึง logic ที่ใช้
  ร่วมกันออกมาเป็น package ที่สาม)

**แนวทางแก้ไข**: ใช้หลัก **Dependency Inversion** (จะเรียนใน Part 56 เรื่อง
SOLID) หรือดึง interface ร่วมออกมาไว้ใน package กลาง แล้วให้ทั้งสอง package
พึ่งพา interface นั้นแทนที่จะพึ่งพากันโดยตรง

## 9. หลักการจัดโครงสร้างโปรเจกต์ที่ดี

### แบบที่ 1: Package by Layer (จัดตามชั้นสถาปัตยกรรม) — นิยมในโปรเจกต์ขนาดเล็ก-กลาง

```
com.example.app
├── controller
├── service
├── repository
└── model
```

### แบบที่ 2: Package by Feature (จัดตามฟีเจอร์) — นิยมในโปรเจกต์ขนาดใหญ่

```
com.example.app
├── order
│   ├── OrderController.java
│   ├── OrderService.java
│   └── OrderRepository.java
├── customer
│   ├── CustomerController.java
│   ├── CustomerService.java
│   └── CustomerRepository.java
└── payment
    ├── PaymentController.java
    └── PaymentService.java
```

**ข้อดีของ Package by Feature**: ทุกอย่างที่เกี่ยวข้องกับฟีเจอร์เดียวกันอยู่ใกล้
กัน ง่ายต่อการทำความเข้าใจและแก้ไข ลดโอกาสเกิด circular dependency ระหว่าง
feature ต่างกัน (เพราะแต่ละ feature ค่อนข้างอิสระจากกัน) — เป็นแนวทางที่แนะนำ
สำหรับโปรเจกต์ Spring Boot ขนาดใหญ่ในหลักสูตรช่วงหลัง (Part 71 เป็นต้นไป)

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** ออกแบบโครงสร้าง package สำหรับระบบ "ห้องสมุดออนไลน์" ที่มีฟีเจอร์: จัดการ
หนังสือ, จัดการสมาชิก, จัดการการยืม-คืน โดยใช้แนวทาง Package by Feature

**เฉลย:**

```
com.example.library
├── book
│   ├── Book.java
│   ├── BookService.java
│   └── BookRepository.java
├── member
│   ├── Member.java
│   ├── MemberService.java
│   └── MemberRepository.java
└── borrowing
    ├── BorrowRecord.java
    ├── BorrowingService.java
    └── BorrowingRepository.java
```

**2)** เขียนคำสั่ง command line สำหรับคอมไพล์โปรเจกต์ที่มี source อยู่ใน `src/`
และต้องการ output ไปที่ `out/` โดยใช้ dependency jar ชื่อ `lib/utils.jar`

**เฉลย:**

```bash
javac -cp "lib/utils.jar" -d out $(find src -name "*.java")
java -cp "out:lib/utils.jar" com.example.Main
```

**3)** อธิบายว่าทำไม circular dependency ระหว่าง package ถึงเป็นปัญหา และเสนอ
วิธีแก้เบื้องต้น

**เฉลย**: Circular dependency ทำให้สอง package ไม่สามารถแยกทดสอบหรือ deploy
อิสระจากกันได้ การแก้ไขโค้ดฝั่งใดฝั่งหนึ่งเสี่ยงกระทบอีกฝั่งเสมอ วิธีแก้คือดึง
ส่วนที่ใช้ร่วมกัน (shared interface หรือ common model) ออกมาไว้ใน package ที่สาม
แล้วให้ทั้งสอง package เดิมพึ่งพา package กลางนั้นแทนที่จะพึ่งพากันโดยตรง
(หลักการ Dependency Inversion)

### สรุปเนื้อหา Part 20

- Package จัดกลุ่มคลาสให้เป็นระเบียบ ป้องกันชื่อชนกัน โครงสร้างโฟลเดอร์ต้องตรงกับ
  package name เป๊ะ
- โครงสร้างโปรเจกต์มาตรฐาน (Maven/Gradle): `src/main/java`, `src/main/resources`,
  `src/test/java`
- Classpath คือรายการตำแหน่งที่ JVM ค้นหาคลาส กำหนดด้วย `-cp`
- JAR file รวม `.class` และ resources เป็นไฟล์เดียว พร้อม Manifest ระบุ Main-Class
  เพื่อรันได้โดยตรง
- ตั้งชื่อ package ด้วย domain กลับด้านตามธรรมเนียมสากล
- หลีกเลี่ยง circular dependency ระหว่าง package ด้วยการออกแบบที่ดี (Dependency
  Inversion)
- Package by Feature เหมาะกับโปรเจกต์ขนาดใหญ่มากกว่า Package by Layer

**จบหมวดพื้นฐานภาษา Java (Part 1-20) อย่างสมบูรณ์! ต่อไปจะเข้าสู่หมวดโครงสร้าง
ข้อมูลและอัลกอริทึม**

**ต่อไป**: [Part 21 — Exception Handling ขั้นสูง](./part-021-exception-handling-advanced.md)
