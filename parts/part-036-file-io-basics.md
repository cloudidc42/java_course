# Part 36: File I/O เบื้องต้น: File, FileReader/Writer, BufferedReader/Writer

> ขั้นตอนที่ 351-360 ของหลักสูตร | ระดับ: กลาง (เริ่มหมวด Java ระดับกลางถึงขั้นสูง)

## สารบัญ

1. ภาพรวม I/O Stream ใน Java
2. คลาส `File`: ตัวแทนของไฟล์และไดเรกทอรี
3. การเขียนไฟล์ด้วย `FileWriter`
4. การอ่านไฟล์ด้วย `FileReader`
5. `BufferedReader`/`BufferedWriter`: เพิ่มประสิทธิภาพ
6. Byte Stream vs Character Stream
7. การอ่าน/เขียนไฟล์ CSV เบื้องต้น
8. การจัดการไดเรกทอรี
9. Best Practices สำหรับ File I/O
10. แบบฝึกหัดและสรุป

---

## 1. ภาพรวม I/O Stream ใน Java

**I/O (Input/Output) Stream** คือกลไกของ Java สำหรับอ่าน/เขียนข้อมูลระหว่าง
โปรแกรมกับแหล่งข้อมูลภายนอก (ไฟล์, network, console) — ข้อมูลไหลเป็น**ลำดับ
(stream)** ทีละส่วน ไม่ใช่โหลดทั้งหมดเข้าหน่วยความจำพร้อมกันเสมอไป

```
Input Stream:  แหล่งข้อมูล (ไฟล์) ---> โปรแกรม   (อ่านเข้า)
Output Stream: โปรแกรม ---> ปลายทาง (ไฟล์)       (เขียนออก)
```

Java แบ่ง I/O เป็น 2 ประเภทหลัก:
- **Byte Stream** (`InputStream`/`OutputStream`): จัดการข้อมูลแบบ raw bytes
  เหมาะกับไฟล์ไบนารี (รูปภาพ, วิดีโอ, ไฟล์บีบอัด)
- **Character Stream** (`Reader`/`Writer`): จัดการข้อมูลแบบตัวอักษร (text) ที่
  แปลง encoding ให้อัตโนมัติ เหมาะกับไฟล์ text ทั่วไป

Part นี้เน้น Character Stream สำหรับไฟล์ text เป็นหลัก

## 2. คลาส `File`: ตัวแทนของไฟล์และไดเรกทอรี

`java.io.File` เป็น**ตัวแทนของ path** (เส้นทาง) ไปยังไฟล์หรือโฟลเดอร์ —
**ไม่ได้เปิดหรืออ่านไฟล์จริง** เป็นเพียง object ที่เก็บข้อมูล metadata

```java
import java.io.File;

public class FileClassDemo {
    public static void main(String[] args) {
        File file = new File("data.txt");

        System.out.println(file.exists());        // มีไฟล์นี้อยู่จริงหรือไม่
        System.out.println(file.getName());         // ชื่อไฟล์: "data.txt"
        System.out.println(file.getAbsolutePath()); // path แบบเต็ม
        System.out.println(file.isDirectory());      // เป็นโฟลเดอร์หรือไม่
        System.out.println(file.isFile());             // เป็นไฟล์หรือไม่

        try {
            boolean created = file.createNewFile(); // สร้างไฟล์เปล่าถ้ายังไม่มี
            System.out.println("สร้างไฟล์: " + created);
        } catch (java.io.IOException e) {
            System.out.println("สร้างไฟล์ไม่สำเร็จ: " + e.getMessage());
        }

        if (file.exists()) {
            System.out.println("ขนาดไฟล์: " + file.length() + " bytes");
            file.delete(); // ลบไฟล์
        }
    }
}
```

## 3. การเขียนไฟล์ด้วย `FileWriter`

```java
import java.io.FileWriter;
import java.io.IOException;

public class FileWriterDemo {
    public static void main(String[] args) {
        try (FileWriter writer = new FileWriter("greeting.txt")) { // try-with-resources (Part 21)
            writer.write("สวัสดี Java!\n");
            writer.write("นี่คือบรรทัดที่สอง\n");
            System.out.println("เขียนไฟล์สำเร็จ");
        } catch (IOException e) {
            System.out.println("เขียนไฟล์ไม่สำเร็จ: " + e.getMessage());
        }

        // เขียนแบบ append (ต่อท้ายไฟล์เดิม ไม่ลบของเก่า) - พารามิเตอร์ที่สอง = true
        try (FileWriter appendWriter = new FileWriter("greeting.txt", true)) {
            appendWriter.write("บรรทัดที่เพิ่มเข้ามาทีหลัง\n");
        } catch (IOException e) {
            System.out.println(e.getMessage());
        }
    }
}
```

**ข้อควรระวังสำคัญ**: `new FileWriter("path")` (ไม่มี `true`) จะ**ลบเนื้อหา
เดิมทั้งหมดทันที**ที่เปิดไฟล์ (overwrite) — ถ้าต้องการต่อท้ายต้องระบุ
`append=true` เสมอ

## 4. การอ่านไฟล์ด้วย `FileReader`

```java
import java.io.FileReader;
import java.io.IOException;

public class FileReaderDemo {
    public static void main(String[] args) {
        try (FileReader reader = new FileReader("greeting.txt")) {
            int character;
            StringBuilder content = new StringBuilder();

            while ((character = reader.read()) != -1) { // read() คืน -1 เมื่อถึงจุดสิ้นสุดไฟล์ (EOF)
                content.append((char) character); // read() คืนค่าเป็น int (unicode code point)
            }

            System.out.println(content);
        } catch (IOException e) {
            System.out.println("อ่านไฟล์ไม่สำเร็จ: " + e.getMessage());
        }
    }
}
```

**ปัญหา**: อ่านทีละตัวอักษร (`read()`) **ช้ามาก**สำหรับไฟล์ขนาดใหญ่ เพราะทุก
การเรียก `read()` อาจต้องติดต่อระบบปฏิบัติการ (system call) — นี่คือเหตุผลที่
ต้องใช้ **Buffered Stream** ในหัวข้อถัดไป

## 5. `BufferedReader`/`BufferedWriter`: เพิ่มประสิทธิภาพ

**Buffered Stream** เก็บข้อมูลไว้ใน**บัฟเฟอร์ในหน่วยความจำ**ก่อน แล้วอ่าน/
เขียนเป็นชิ้นใหญ่ (batch) ลดจำนวนการติดต่อระบบปฏิบัติการอย่างมาก — **ควรใช้
เสมอ**สำหรับไฟล์ text ในทางปฏิบัติจริง

```java
import java.io.BufferedReader;
import java.io.BufferedWriter;
import java.io.FileReader;
import java.io.FileWriter;
import java.io.IOException;

public class BufferedIODemo {
    public static void main(String[] args) {
        // เขียนไฟล์แบบ buffered (เร็วกว่า FileWriter เปล่า ๆ มากสำหรับข้อมูลจำนวนมาก)
        try (BufferedWriter writer = new BufferedWriter(new FileWriter("students.txt"))) {
            String[] students = {"Alice,25,Engineering", "Bob,22,Science", "Charlie,28,Arts"};
            for (String student : students) {
                writer.write(student);
                writer.newLine(); // เขียนบรรทัดใหม่ที่ถูกต้องตาม OS (ดีกว่าเขียน "\n" ตรง ๆ)
            }
        } catch (IOException e) {
            System.out.println("เขียนไฟล์ไม่สำเร็จ: " + e.getMessage());
        }

        // อ่านไฟล์แบบ buffered ทีละบรรทัด (readLine()) - สะดวกและเร็วกว่า read() ทีละตัวอักษรมาก
        try (BufferedReader reader = new BufferedReader(new FileReader("students.txt"))) {
            String line;
            while ((line = reader.readLine()) != null) { // readLine() คืน null เมื่อถึง EOF
                String[] parts = line.split(",");
                System.out.println("ชื่อ: " + parts[0] + ", อายุ: " + parts[1] + ", สาขา: " + parts[2]);
            }
        } catch (IOException e) {
            System.out.println("อ่านไฟล์ไม่สำเร็จ: " + e.getMessage());
        }
    }
}
```

**เปรียบเทียบประสิทธิภาพ**: การอ่านไฟล์ขนาด 10MB ด้วย `FileReader.read()` ทีละ
ตัวอักษรอาจใช้เวลาหลายวินาที ในขณะที่ `BufferedReader.readLine()` ใช้เวลาไม่ถึง
วินาที — ความแตกต่างเกิดจากจำนวนการติดต่อ disk I/O ที่ลดลงอย่างมาก

## 6. Byte Stream vs Character Stream

```java
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;

public class ByteStreamDemo {
    public static void main(String[] args) {
        // Byte Stream: สำหรับไฟล์ไบนารี (เช่น รูปภาพ, ไฟล์บีบอัด) - ไม่แปลง encoding ใด ๆ
        try (FileOutputStream out = new FileOutputStream("binary.dat")) {
            byte[] data = {72, 101, 108, 108, 111}; // "Hello" ในรูปแบบ byte array
            out.write(data);
        } catch (IOException e) {
            System.out.println(e.getMessage());
        }

        try (FileInputStream in = new FileInputStream("binary.dat")) {
            byte[] buffer = new byte[5];
            int bytesRead = in.read(buffer);
            System.out.println("อ่านได้ " + bytesRead + " bytes: " + new String(buffer));
        } catch (IOException e) {
            System.out.println(e.getMessage());
        }
    }
}
```

| ลักษณะ | Byte Stream (`InputStream`/`OutputStream`) | Character Stream (`Reader`/`Writer`) |
|---|---|---|
| หน่วยข้อมูล | 8-bit byte | 16-bit character (Unicode) |
| ใช้กับ | ไฟล์ไบนารี (รูปภาพ, เสียง, PDF) | ไฟล์ text (แปลง encoding อัตโนมัติ) |
| ตัวอย่าง class | `FileInputStream`, `FileOutputStream` | `FileReader`, `FileWriter` |

## 7. การอ่าน/เขียนไฟล์ CSV เบื้องต้น

```java
import java.io.BufferedReader;
import java.io.BufferedWriter;
import java.io.FileReader;
import java.io.FileWriter;
import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

public class CsvDemo {
    record Student(String name, int age, String major) { }

    static void writeStudentsToCsv(List<Student> students, String filename) throws IOException {
        try (BufferedWriter writer = new BufferedWriter(new FileWriter(filename))) {
            writer.write("name,age,major"); // header row
            writer.newLine();
            for (Student s : students) {
                writer.write(s.name() + "," + s.age() + "," + s.major());
                writer.newLine();
            }
        }
    }

    static List<Student> readStudentsFromCsv(String filename) throws IOException {
        List<Student> students = new ArrayList<>();
        try (BufferedReader reader = new BufferedReader(new FileReader(filename))) {
            reader.readLine(); // ข้าม header row
            String line;
            while ((line = reader.readLine()) != null) {
                String[] parts = line.split(",");
                students.add(new Student(parts[0], Integer.parseInt(parts[1]), parts[2]));
            }
        }
        return students;
    }

    public static void main(String[] args) throws IOException {
        List<Student> students = List.of(
            new Student("Alice", 25, "Engineering"),
            new Student("Bob", 22, "Science")
        );

        writeStudentsToCsv(students, "students.csv");
        List<Student> loaded = readStudentsFromCsv("students.csv");
        loaded.forEach(System.out::println);
    }
}
```

**ข้อควรระวัง**: การ `split(",")` แบบง่าย ๆ นี้**ใช้ไม่ได้**กับ CSV ที่มีค่า
ที่มี comma อยู่ในตัวเอง (เช่น `"Bangkok, Thailand"`) — สำหรับ CSV ระดับ
production ควรใช้ library เฉพาะทาง เช่น **Apache Commons CSV** หรือ **OpenCSV**
แทนการ parse ด้วยมือ

## 8. การจัดการไดเรกทอรี

```java
import java.io.File;

public class DirectoryDemo {
    public static void main(String[] args) {
        File dir = new File("output");

        if (!dir.exists()) {
            boolean created = dir.mkdir(); // สร้างโฟลเดอร์เดียว
            System.out.println("สร้างโฟลเดอร์: " + created);
        }

        File nestedDir = new File("output/reports/2024");
        nestedDir.mkdirs(); // mkdirs() สร้างโฟลเดอร์ซ้อนกันหลายชั้นพร้อมกัน (ต่างจาก mkdir())

        File file = new File(dir, "report.txt"); // สร้าง File object ที่อยู่ในโฟลเดอร์ dir
        System.out.println(file.getPath()); // "output/report.txt"

        // แสดงรายการไฟล์ทั้งหมดในโฟลเดอร์
        File currentDir = new File(".");
        String[] files = currentDir.list();
        if (files != null) {
            for (String name : files) {
                System.out.println(name);
            }
        }
    }
}
```

## 9. Best Practices สำหรับ File I/O

1. **ใช้ `try-with-resources` เสมอ** (ทบทวนจาก Part 21) เพื่อปิด stream อัตโนมัติ
   แม้เกิด exception
2. **ใช้ Buffered version เสมอ** สำหรับไฟล์ text (BufferedReader/BufferedWriter)
   เพื่อประสิทธิภาพที่ดีกว่ามาก
3. **ระบุ character encoding อย่างชัดเจน** เมื่อทำงานกับไฟล์ที่มีภาษาอื่นที่ไม่ใช่
   ภาษาอังกฤษ (โดยเฉพาะภาษาไทย) เพื่อป้องกันปัญหาตัวอักษรเพี้ยน:

```java
import java.io.FileReader;
import java.io.InputStreamReader;
import java.io.FileInputStream;
import java.nio.charset.StandardCharsets;
import java.io.IOException;

public class EncodingDemo {
    public static void main(String[] args) throws IOException {
        // ระบุ UTF-8 อย่างชัดเจน (สำคัญมากสำหรับข้อความภาษาไทย)
        try (var reader = new InputStreamReader(
                new FileInputStream("thai_text.txt"), StandardCharsets.UTF_8)) {
            // อ่านไฟล์...
        }
    }
}
```

4. **สำหรับโค้ดใหม่ ควรใช้ NIO.2 (`java.nio.file`) แทน `java.io`** — Part 37
   จะสอนวิธีที่ทันสมัยและสะดวกกว่ามาก (ครอบคลุมข้อ 3 เรื่อง encoding ให้อัตโนมัติ)

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียนโปรแกรมที่นับจำนวนบรรทัดในไฟล์ text โดยใช้ `BufferedReader`

**เฉลย:**

```java
import java.io.BufferedReader;
import java.io.FileReader;
import java.io.IOException;

public class Exercise1 {
    public static void main(String[] args) {
        int lineCount = 0;
        try (BufferedReader reader = new BufferedReader(new FileReader("students.txt"))) {
            while (reader.readLine() != null) {
                lineCount++;
            }
            System.out.println("จำนวนบรรทัด: " + lineCount);
        } catch (IOException e) {
            System.out.println(e.getMessage());
        }
    }
}
```

**2)** เขียนโปรแกรมคัดลอกไฟล์ text จากไฟล์หนึ่งไปยังอีกไฟล์หนึ่งโดยใช้
`BufferedReader`/`BufferedWriter`

**เฉลย:**

```java
import java.io.*;

public class Exercise2 {
    public static void main(String[] args) {
        try (BufferedReader reader = new BufferedReader(new FileReader("students.txt"));
             BufferedWriter writer = new BufferedWriter(new FileWriter("students_copy.txt"))) {
            String line;
            while ((line = reader.readLine()) != null) {
                writer.write(line);
                writer.newLine();
            }
            System.out.println("คัดลอกไฟล์สำเร็จ");
        } catch (IOException e) {
            System.out.println(e.getMessage());
        }
    }
}
```

**3)** เขียนโปรแกรมที่อ่านไฟล์ CSV ของคะแนนนักเรียน แล้วคำนวณและพิมพ์คะแนนเฉลี่ย

**เฉลย:**

```java
import java.io.BufferedReader;
import java.io.FileReader;
import java.io.IOException;

public class Exercise3 {
    public static void main(String[] args) throws IOException {
        try (BufferedReader reader = new BufferedReader(new FileReader("scores.csv"))) {
            reader.readLine(); // ข้าม header
            String line;
            int sum = 0, count = 0;
            while ((line = reader.readLine()) != null) {
                String[] parts = line.split(",");
                sum += Integer.parseInt(parts[1]);
                count++;
            }
            System.out.println("คะแนนเฉลี่ย: " + (double) sum / count);
        }
    }
}
```

### สรุปเนื้อหา Part 36

- I/O Stream แบ่งเป็น Byte Stream (ไฟล์ไบนารี) และ Character Stream (ไฟล์ text)
- `File` เป็นตัวแทนของ path ไม่ใช่การเปิดไฟล์จริง
- `FileWriter`/`FileReader` อ่าน/เขียนไฟล์ text พื้นฐาน — `new FileWriter(path)`
  ลบเนื้อหาเดิมทันที ต้องใช้ `append=true` เพื่อต่อท้าย
- `BufferedReader`/`BufferedWriter` เพิ่มประสิทธิภาพอย่างมาก ควรใช้เสมอในทาง
  ปฏิบัติ (มี `readLine()` สะดวกมาก)
- ต้องระบุ character encoding (UTF-8) อย่างชัดเจนเมื่อทำงานกับข้อความภาษาไทย
- โค้ดสมัยใหม่ควรใช้ NIO.2 (`java.nio.file`) มากกว่า `java.io` แบบดั้งเดิม

**ต่อไป**: [Part 37 — NIO.2: Path, Files, การอ่านเขียนไฟล์สมัยใหม่](./part-037-nio2.md)
