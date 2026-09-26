# Part 2: โครงสร้างโปรแกรม Java, การคอมไพล์และรัน, กฎการตั้งชื่อ

> ขั้นตอนที่ 11-20 ของหลักสูตร | ระดับ: พื้นฐาน

## สารบัญ

1. โครงสร้างไฟล์ `.java` แบบสมบูรณ์
2. Package declaration และการจัดโฟลเดอร์
3. Import statements
4. Class declaration และกฎการตั้งชื่อไฟล์
5. คอมเมนต์ 3 แบบใน Java
6. กฎและธรรมเนียมการตั้งชื่อ (Naming Conventions)
7. Keywords ที่สงวนไว้ใน Java
8. Statement, Block, และ Whitespace
9. การรันโปรแกรมที่มีหลายคลาสและหลายไฟล์
10. แบบฝึกหัดและสรุป

---

## 1. โครงสร้างไฟล์ `.java` แบบสมบูรณ์

ไฟล์ Java หนึ่งไฟล์มีโครงสร้างเรียงตามลำดับดังนี้ (ข้อใดไม่มีก็ข้ามได้ แต่ลำดับต้องถูกต้อง):

```java
// 1. Package declaration (ถ้ามี ต้องอยู่บรรทัดแรกสุด ไม่นับคอมเมนต์)
package com.example.myapp;

// 2. Import statements (ถ้ามี)
import java.util.List;
import java.util.ArrayList;

// 3. Class / Interface / Enum declaration (อย่างน้อย 1 คลาสต่อไฟล์)
public class MyApp {

    // 3.1 Fields (ตัวแปรระดับคลาส)
    private String name;

    // 3.2 Constructors
    public MyApp(String name) {
        this.name = name;
    }

    // 3.3 Methods
    public void printName() {
        System.out.println(name);
    }

    // 3.4 main method (ถ้าต้องการให้ไฟล์นี้รันได้)
    public static void main(String[] args) {
        MyApp app = new MyApp("ทดสอบ");
        app.printName();
    }
}
```

## 2. Package declaration และการจัดโฟลเดอร์

**Package** คือกลไกจัดกลุ่มคลาสให้เป็นระบบ คล้ายโฟลเดอร์ในการจัดไฟล์ ช่วยป้องกันชื่อคลาส
ชนกัน (name collision) และทำให้โปรเจกต์ใหญ่ ๆ เป็นระเบียบ

```java
package com.company.project.module;
```

**กฎสำคัญ**: ชื่อ package ต้องตรงกับโครงสร้างโฟลเดอร์เป๊ะ ๆ เช่นถ้า package คือ
`com.example.myapp` ไฟล์ต้องอยู่ที่:

```
src/
└── com/
    └── example/
        └── myapp/
            └── MyApp.java
```

ธรรมเนียมการตั้งชื่อ package: ใช้ตัวพิมพ์เล็กทั้งหมด และมักขึ้นต้นด้วย domain name
กลับด้าน เช่นบริษัทเจ้าของ domain `example.com` จะตั้ง package เป็น `com.example.*`

หากไม่ประกาศ package จะถือว่าคลาสอยู่ใน **default package** (ไม่แนะนำสำหรับโปรเจกต์จริง
ใช้ได้เฉพาะตอนทดลองโค้ดสั้น ๆ เท่านั้น)

## 3. Import statements

`import` ใช้เพื่อเรียกใช้คลาสจาก package อื่นโดยไม่ต้องพิมพ์ full path ทุกครั้ง

```java
import java.util.List;        // import เฉพาะคลาส List
import java.util.*;           // import ทุกคลาสใน package java.util (ไม่แนะนำ)
import static java.lang.Math.PI; // static import - เรียก PI ได้ตรง ๆ ไม่ต้องพิมพ์ Math.PI
```

ตัวอย่างการใช้งาน:

```java
import java.util.ArrayList;
import java.util.List;

public class ImportDemo {
    public static void main(String[] args) {
        List<String> names = new ArrayList<>();
        names.add("Alice");
        names.add("Bob");
        System.out.println(names);
    }
}
```

> หมายเหตุ: คลาสใน package `java.lang` (เช่น `String`, `System`, `Math`) ถูก import
> ให้อัตโนมัติเสมอ ไม่ต้องเขียน `import java.lang.String;`

## 4. Class declaration และกฎการตั้งชื่อไฟล์

กฎเหล็กที่พลาดไม่ได้: **ถ้าคลาสถูกประกาศเป็น `public` ชื่อไฟล์ต้องตรงกับชื่อคลาสเป๊ะ ๆ
(รวมตัวพิมพ์เล็ก-ใหญ่)** และ**ในหนึ่งไฟล์มี public class ได้สูงสุด 1 คลาสเท่านั้น**

```java
// ไฟล์: Calculator.java  (ถูกต้อง)
public class Calculator {
    // ...
}
```

แต่สามารถมีคลาสอื่นที่ไม่ใช่ public ในไฟล์เดียวกันได้:

```java
// ไฟล์: Calculator.java
public class Calculator {
    public int add(int a, int b) {
        return a + b;
    }
}

class Helper {   // ไม่ public - อยู่ในไฟล์เดียวกันได้ แต่มองเห็นเฉพาะใน package เดียวกัน
    static void log(String msg) {
        System.out.println("[LOG] " + msg);
    }
}
```

## 5. คอมเมนต์ 3 แบบใน Java

```java
public class CommentDemo {
    // 1. Single-line comment: ใช้อธิบายบรรทัดเดียว

    /*
     * 2. Multi-line comment (Traditional comment):
     * ใช้อธิบายหลายบรรทัด เหมาะกับคำอธิบายยาว ๆ
     */

    /**
     * 3. Javadoc comment: ใช้สร้างเอกสารอัตโนมัติด้วยคำสั่ง javadoc
     * รองรับ tag พิเศษ เช่น @param, @return, @author, @throws
     *
     * @param x ตัวเลขจำนวนเต็มตัวแรก
     * @param y ตัวเลขจำนวนเต็มตัวที่สอง
     * @return ผลรวมของ x และ y
     */
    public int add(int x, int y) {
        return x + y;
    }
}
```

สามารถสร้างเอกสาร HTML จาก Javadoc comment ได้ด้วยคำสั่ง:

```bash
javadoc CommentDemo.java -d docs/
```

## 6. กฎและธรรมเนียมการตั้งชื่อ (Naming Conventions)

Java มี**กฎบังคับ**และ**ธรรมเนียมปฏิบัติ** (convention) ที่แตกต่างกัน — กฎบังคับถ้าฝ่าฝืน
จะ compile ไม่ผ่าน ส่วน convention ถ้าไม่ทำตามโค้ดยังรันได้แต่จะถือว่าเขียนโค้ดไม่ดี

### กฎบังคับสำหรับ identifier (ชื่อตัวแปร/เมธอด/คลาส)

- ต้องขึ้นต้นด้วยตัวอักษร, `_`, หรือ `$` เท่านั้น (ห้ามขึ้นต้นด้วยตัวเลข)
- ตัวถัดไปเป็นตัวอักษร, ตัวเลข, `_`, หรือ `$` ก็ได้
- ห้ามใช้ keyword ที่สงวนไว้ (เช่น `class`, `public`, `int`) เป็นชื่อ
- Case-sensitive: `age` กับ `Age` ถือเป็นคนละตัวแปร

### ธรรมเนียมปฏิบัติ (Convention) ที่ใช้กันทั่วโลก

| ประเภท | รูปแบบ | ตัวอย่าง |
|---|---|---|
| Class / Interface | `PascalCase` (ขึ้นต้นตัวใหญ่ทุกคำ) | `Calculator`, `BankAccount`, `Runnable` |
| Method / Variable | `camelCase` (ขึ้นต้นตัวเล็ก คำถัดไปตัวใหญ่) | `calculateTotal()`, `firstName` |
| Constant (`static final`) | `UPPER_SNAKE_CASE` | `MAX_SIZE`, `PI_VALUE` |
| Package | `lowercase` ทั้งหมด ไม่มีขีด | `com.example.myapp` |

```java
public class BankAccount {                    // Class: PascalCase
    private static final double MAX_LIMIT = 100000.0; // Constant: UPPER_SNAKE_CASE
    private double accountBalance;            // Variable: camelCase

    public void depositMoney(double amount) { // Method: camelCase
        accountBalance += amount;
    }
}
```

ตั้งชื่อให้สื่อความหมาย (meaningful names) เสมอ:

```java
// ไม่ดี
int d;
double c;

// ดี
int daysElapsed;
double totalCost;
```

## 7. Keywords ที่สงวนไว้ใน Java

Java มีคำสงวน (reserved keywords) ทั้งหมด 53 คำ (ไม่นับ `true`, `false`, `null` ซึ่งเป็น
literal ไม่ใช่ keyword) ที่ไม่สามารถนำมาใช้เป็นชื่อตัวแปรได้ ตัวอย่างที่พบบ่อย:

```
abstract   continue   for          new         switch
assert     default    goto         package     synchronized
boolean    do         if           private     this
break      double     implements   protected   throw
byte       else       import       public      throws
case       enum       instanceof   return      transient
catch      extends    int          short       try
char       final      interface    static      void
class      finally    long         strictfp    volatile
const      float      native       super       while
```

รวมถึงคำที่เพิ่มมาในเวอร์ชันหลัง ๆ ที่เป็น **contextual keyword** (สงวนเฉพาะบริบท)
เช่น `var`, `record`, `sealed`, `permits`, `yield` — ใช้เป็นชื่อตัวแปรได้ในบางกรณี
แต่ไม่แนะนำเพื่อความชัดเจน

## 8. Statement, Block, และ Whitespace

- **Statement** คือคำสั่งหนึ่งคำสั่ง จบด้วย semicolon `;` เสมอ
- **Block** คือกลุ่มของ statement ที่ครอบด้วยปีกกา `{ }`
- **Whitespace** (ช่องว่าง, tab, การขึ้นบรรทัดใหม่) Java ไม่สนใจว่าคุณจัดรูปแบบยังไง
  แต่ควรจัดให้อ่านง่ายตาม convention

```java
public class StatementDemo {
    public static void main(String[] args) {   // เปิด block ของ main
        int x = 5;                              // statement 1
        int y = 10;                              // statement 2
        {                                        // block ซ้อนใน block (ยังไงก็รันได้)
            int sum = x + y;
            System.out.println(sum);
        }
    }                                            // ปิด block ของ main
}
```

โค้ดสองแบบนี้ compile ได้เหมือนกันทุกประการ เพราะ whitespace ไม่มีผลต่อความหมาย:

```java
public class A{public static void main(String[]a){System.out.println("hi");}}
```

```java
public class A {
    public static void main(String[] a) {
        System.out.println("hi");
    }
}
```

แต่แน่นอนว่าแบบที่สองอ่านง่ายกว่ามาก — **ในหลักสูตรนี้จะยึดการจัดรูปแบบตาม
Google Java Style Guide** เป็นมาตรฐาน

## 9. การรันโปรแกรมที่มีหลายคลาสและหลายไฟล์

ตัวอย่างโปรเจกต์ที่มี 2 ไฟล์ทำงานร่วมกัน:

```java
// ไฟล์: Greeter.java
public class Greeter {
    public String greet(String name) {
        return "สวัสดี, " + name + "!";
    }
}
```

```java
// ไฟล์: Main.java
public class Main {
    public static void main(String[] args) {
        Greeter greeter = new Greeter();
        System.out.println(greeter.greet("โลก"));
    }
}
```

คอมไพล์ทั้งสองไฟล์พร้อมกัน แล้วรัน entry point:

```bash
javac Greeter.java Main.java
# หรือใช้ wildcard คอมไพล์ทุกไฟล์ .java ในโฟลเดอร์
javac *.java

java Main
```

ผลลัพธ์:

```
สวัสดี, โลก!
```

Java compiler ฉลาดพอที่จะไล่หา dependency ให้เองด้วย ถ้าคุณสั่ง `javac Main.java`
เพียงไฟล์เดียว และ `Main.java` อ้างถึง `Greeter` มันจะไปคอมไพล์ `Greeter.java` ให้อัตโนมัติ
(ถ้าหาไฟล์เจอใน classpath เดียวกัน)

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** สร้างโปรเจกต์ที่มี package ชื่อ `com.myschool.students` และไฟล์ `Student.java`
ที่มี field `name` และ `grade` พร้อมเมธอด `printInfo()` ที่พิมพ์ข้อมูลออกมา
จัดโฟลเดอร์ให้ตรงกับ package name

**เฉลย:**

```
src/
└── com/
    └── myschool/
        └── students/
            └── Student.java
```

```java
package com.myschool.students;

public class Student {
    private String name;
    private String grade;

    public Student(String name, String grade) {
        this.name = name;
        this.grade = grade;
    }

    public void printInfo() {
        System.out.println("ชื่อ: " + name + ", ระดับชั้น: " + grade);
    }
}
```

คอมไพล์และรันจาก root ของ `src/`:

```bash
javac com/myschool/students/Student.java
```

**2)** ตัวแปรชื่อใดต่อไปนี้ **ผิดกฎ** ของ Java และเพราะเหตุใด: `2ndValue`, `_temp`,
`total-cost`, `$price`, `class`

**เฉลย:**
- `2ndValue` ผิด — ห้ามขึ้นต้นด้วยตัวเลข
- `_temp` ถูกต้อง — ขึ้นต้นด้วย underscore ได้
- `total-cost` ผิด — ห้ามมีเครื่องหมาย `-` (จะถูกตีความเป็นการลบ)
- `$price` ถูกต้อง — ขึ้นต้นด้วย `$` ได้ (แม้ไม่นิยมใช้)
- `class` ผิด — เป็น keyword สงวนไว้

### สรุปเนื้อหา Part 2

- ไฟล์ Java มีโครงสร้างตายตัว: package → import → class declaration
- ชื่อไฟล์ต้องตรงกับชื่อ public class เป๊ะ ๆ และมี public class ได้แค่ 1 ต่อไฟล์
- คอมเมนต์มี 3 แบบ: `//`, `/* */`, และ `/** */` (Javadoc)
- ใช้ PascalCase สำหรับคลาส, camelCase สำหรับตัวแปร/เมธอด, UPPER_SNAKE_CASE
  สำหรับค่าคงที่
- Whitespace ไม่มีผลต่อการทำงานของโปรแกรม แต่มีผลต่อความอ่านง่าย

**ต่อไป**: [Part 3 — ตัวแปร ชนิดข้อมูล และการแปลงชนิดข้อมูล](./part-003-variables-and-data-types.md)
