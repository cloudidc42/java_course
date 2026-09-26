# Part 37: NIO.2: Path, Files, การอ่านเขียนไฟล์สมัยใหม่

> ขั้นตอนที่ 361-370 ของหลักสูตร | ระดับ: กลาง

## สารบัญ

1. NIO.2 คืออะไร ทำไมมาแทน `java.io`
2. `Path` และ `Paths`/`Path.of()`
3. `Files`: เมธอด Utility ที่ทรงพลัง
4. การอ่านไฟล์ทั้งหมดในครั้งเดียว
5. การเขียนไฟล์แบบกระชับ
6. การเดินไฟล์ในไดเรกทอรี (`Files.walk`, `Files.list`)
7. การคัดลอก ย้าย ลบไฟล์
8. `Stream<String>` จากไฟล์ (เชื่อมกับ Part 41-42)
9. WatchService เบื้องต้น (ตรวจจับการเปลี่ยนแปลงไฟล์)
10. แบบฝึกหัดและสรุป

---

## 1. NIO.2 คืออะไร ทำไมมาแทน `java.io`

**NIO.2** (New I/O 2, เปิดตัวใน Java 7) คือ API ใหม่ใน package `java.nio.file`
ที่ออกแบบมาให้**ใช้งานง่ายกว่า เร็วกว่า และปลอดภัยกว่า** `java.io` แบบดั้งเดิม
(Part 36) — ในโค้ดสมัยใหม่ **ควรใช้ NIO.2 เป็นค่าเริ่มต้นเสมอ**

**ข้อดีหลักของ NIO.2**:
- เมธอด utility สำเร็จรูปมากมายใน `Files` (ลดโค้ด boilerplate)
- จัดการ symbolic link ได้ดีกว่า
- รองรับ `Stream` API (Part 41-42) โดยตรง
- Error message ชัดเจนกว่า (เช่น `NoSuchFileException` แทน `IOException` กว้าง ๆ)

## 2. `Path` และ `Paths`/`Path.of()`

`Path` แทนที่ `File` จาก `java.io` — เป็นตัวแทนของเส้นทางไฟล์ที่ทำงานร่วมกับ
`Files` utility class ได้อย่างลงตัว

```java
import java.nio.file.Path;

public class PathDemo {
    public static void main(String[] args) {
        Path path = Path.of("data", "reports", "2024.txt"); // Java 11+ วิธีที่แนะนำ
        // Path path2 = java.nio.file.Paths.get("data/reports/2024.txt"); // วิธีเก่า (ยังใช้ได้)

        System.out.println(path);                  // data/reports/2024.txt
        System.out.println(path.getFileName());      // 2024.txt
        System.out.println(path.getParent());          // data/reports
        System.out.println(path.toAbsolutePath());      // path แบบเต็มจาก root
        System.out.println(path.isAbsolute());           // false (เป็น relative path)

        Path absolute = Path.of("/home/user/data.txt");
        System.out.println(absolute.isAbsolute()); // true

        // resolve() ต่อ path เข้าด้วยกัน (คล้าย File constructor ที่รับ parent+child)
        Path base = Path.of("/home/user");
        Path resolved = base.resolve("documents/file.txt");
        System.out.println(resolved); // /home/user/documents/file.txt
    }
}
```

## 3. `Files`: เมธอด Utility ที่ทรงพลัง

```java
import java.nio.file.Files;
import java.nio.file.Path;
import java.io.IOException;

public class FilesUtilityDemo {
    public static void main(String[] args) throws IOException {
        Path path = Path.of("example.txt");

        System.out.println(Files.exists(path));       // ตรวจสอบว่ามีไฟล์อยู่จริงหรือไม่
        System.out.println(Files.isDirectory(path));    // เป็นโฟลเดอร์หรือไม่
        System.out.println(Files.isRegularFile(path));    // เป็นไฟล์ธรรมดาหรือไม่

        if (!Files.exists(path)) {
            Files.createFile(path); // สร้างไฟล์เปล่า
        }

        System.out.println(Files.size(path)); // ขนาดไฟล์เป็น byte

        Files.deleteIfExists(path); // ลบไฟล์แบบปลอดภัย (ไม่ throw exception ถ้าไม่มีไฟล์)
    }
}
```

## 4. การอ่านไฟล์ทั้งหมดในครั้งเดียว

NIO.2 มีเมธอดที่**อ่านไฟล์ทั้งหมดในคำสั่งเดียว** เหมาะสำหรับไฟล์ขนาดไม่ใหญ่มาก
(ลดโค้ด boilerplate ของ BufferedReader loop ใน Part 36 อย่างมาก):

```java
import java.nio.file.Files;
import java.nio.file.Path;
import java.io.IOException;
import java.util.List;

public class ReadAllDemo {
    public static void main(String[] args) throws IOException {
        Path path = Path.of("students.txt");
        Files.writeString(path, "Alice\nBob\nCharlie\n"); // เขียนไฟล์แบบง่ายที่สุด (หัวข้อ 5)

        // อ่านทั้งไฟล์เป็น String เดียว (รวมทุกบรรทัด)
        String content = Files.readString(path);
        System.out.println(content);

        // อ่านทีละบรรทัดเป็น List<String> (แต่ละบรรทัดเป็น 1 element)
        List<String> lines = Files.readAllLines(path);
        System.out.println(lines); // [Alice, Bob, Charlie]

        // อ่านเป็น byte array (เหมาะกับไฟล์ไบนารีขนาดเล็ก)
        byte[] bytes = Files.readAllBytes(path);
        System.out.println(bytes.length + " bytes");
    }
}
```

**ข้อควรระวัง**: เมธอดเหล่านี้**โหลดทั้งไฟล์เข้าหน่วยความจำพร้อมกัน** ไม่เหมาะกับ
ไฟล์ขนาดใหญ่มาก (หลาย GB) — สำหรับไฟล์ขนาดใหญ่ควรใช้ Stream (หัวข้อ 8) หรือ
`BufferedReader` แบบดั้งเดิม

## 5. การเขียนไฟล์แบบกระชับ

```java
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.StandardOpenOption;
import java.io.IOException;
import java.util.List;

public class WriteFilesDemo {
    public static void main(String[] args) throws IOException {
        Path path = Path.of("output.txt");

        // เขียน String เดียวลงไฟล์ (สร้างไฟล์ใหม่ทับของเดิม)
        Files.writeString(path, "Hello, NIO.2!\n");

        // เขียนแบบ append
        Files.writeString(path, "บรรทัดที่เพิ่มเข้ามา\n", StandardOpenOption.APPEND);

        // เขียนจาก List<String> ทีละบรรทัด
        List<String> lines = List.of("บรรทัด 1", "บรรทัด 2", "บรรทัด 3");
        Files.write(path, lines); // เขียนทับของเดิม (ไม่ใช่ append) โดยแต่ละ element คือ 1 บรรทัด

        System.out.println(Files.readString(path));
    }
}
```

## 6. การเดินไฟล์ในไดเรกทอรี (`Files.walk`, `Files.list`)

```java
import java.nio.file.Files;
import java.nio.file.Path;
import java.io.IOException;
import java.util.stream.Stream;

public class DirectoryWalkDemo {
    public static void main(String[] args) throws IOException {
        Path dir = Path.of(".");

        // Files.list(): แสดงไฟล์/โฟลเดอร์ "ระดับเดียว" ในโฟลเดอร์นี้ (ไม่ลงลึกเข้าไปในโฟลเดอร์ย่อย)
        try (Stream<Path> stream = Files.list(dir)) {
            stream.forEach(System.out::println);
        }

        // Files.walk(): เดินลงลึกเข้าไปในทุกโฟลเดอร์ย่อยแบบ recursive (คล้าย DFS จาก Part 33)
        try (Stream<Path> stream = Files.walk(dir, 2)) { // depth=2: ลงลึกได้แค่ 2 ชั้น
            stream.filter(Files::isRegularFile) // เอาแค่ไฟล์ ไม่เอาโฟลเดอร์
                  .filter(p -> p.toString().endsWith(".txt")) // เอาแค่ไฟล์ .txt
                  .forEach(System.out::println);
        }
    }
}
```

## 7. การคัดลอก ย้าย ลบไฟล์

```java
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.StandardCopyOption;
import java.io.IOException;

public class CopyMoveDeleteDemo {
    public static void main(String[] args) throws IOException {
        Path source = Path.of("original.txt");
        Files.writeString(source, "เนื้อหาต้นฉบับ");

        Path copy = Path.of("copy.txt");
        Files.copy(source, copy, StandardCopyOption.REPLACE_EXISTING); // คัดลอก (ทับถ้ามีอยู่แล้ว)

        Path moved = Path.of("moved.txt");
        Files.move(copy, moved, StandardCopyOption.REPLACE_EXISTING); // ย้าย/เปลี่ยนชื่อไฟล์

        System.out.println(Files.exists(copy)); // false (ถูกย้ายไปแล้ว)
        System.out.println(Files.exists(moved)); // true

        Files.delete(source); // ลบไฟล์ (throw NoSuchFileException ถ้าไม่มีไฟล์)
        Files.deleteIfExists(moved); // ลบแบบปลอดภัย ไม่ throw exception ถ้าไม่มีไฟล์
    }
}
```

## 8. `Stream<String>` จากไฟล์ (เชื่อมกับ Part 41-42)

`Files.lines()` คืนค่าเป็น `Stream<String>` ที่**อ่านทีละบรรทัดแบบ lazy**
(ไม่โหลดทั้งไฟล์เข้าหน่วยความจำพร้อมกัน) เหมาะกับไฟล์ขนาดใหญ่และเชื่อมกับ
Stream API operations ได้โดยตรง (ปูพื้นฐานสำหรับ Part 41-42):

```java
import java.nio.file.Files;
import java.nio.file.Path;
import java.io.IOException;
import java.util.List;
import java.util.stream.Stream;

public class FileStreamDemo {
    public static void main(String[] args) throws IOException {
        Path path = Path.of("large_file.txt");
        Files.write(path, List.of("apple:5", "banana:3", "cherry:8", "date:2"));

        // ใช้งานร่วมกับ Stream API ได้โดยตรง (try-with-resources สำคัญมาก เพราะ Stream นี้ผูกกับไฟล์)
        try (Stream<String> lines = Files.lines(path)) {
            long totalCount = lines
                .map(line -> line.split(":"))
                .mapToInt(parts -> Integer.parseInt(parts[1]))
                .sum();
            System.out.println("ผลรวม: " + totalCount); // 18
        }
    }
}
```

**ข้อควรจำสำคัญ**: `Files.lines()` **ต้องปิดด้วย try-with-resources เสมอ**
เพราะมันเปิด file handle ค้างไว้จนกว่า stream operation จะเสร็จ (ต่างจาก
`Files.readAllLines()` ที่อ่านเสร็จและปิดไฟล์ทันที)

## 9. WatchService เบื้องต้น (ตรวจจับการเปลี่ยนแปลงไฟล์)

**WatchService** ใช้**เฝ้าดูการเปลี่ยนแปลง**ในไดเรกทอรี (ไฟล์ถูกสร้าง แก้ไข
หรือลบ) — มีประโยชน์มากสำหรับเครื่องมือที่ต้อง auto-reload เมื่อไฟล์เปลี่ยน
(เช่น dev server ที่ reload อัตโนมัติเมื่อบันทึกโค้ด):

```java
import java.nio.file.*;

public class WatchServiceDemo {
    public static void main(String[] args) throws Exception {
        Path dir = Path.of(".");
        WatchService watchService = FileSystems.getDefault().newWatchService();
        dir.register(watchService,
                      StandardWatchEventKinds.ENTRY_CREATE,
                      StandardWatchEventKinds.ENTRY_MODIFY,
                      StandardWatchEventKinds.ENTRY_DELETE);

        System.out.println("กำลังเฝ้าดูการเปลี่ยนแปลงในโฟลเดอร์... (ตัวอย่างนี้จะรอ event 1 ครั้งแล้วหยุด)");

        // WatchKey key = watchService.take(); // บล็อกรอจนกว่าจะมี event เกิดขึ้นจริง
        // for (WatchEvent<?> event : key.pollEvents()) {
        //     System.out.println(event.kind() + ": " + event.context());
        // }
        // (คอมเมนต์ไว้เพื่อไม่ให้ตัวอย่างนี้ค้างรอ input ตลอดไปตอนรันจริงในหลักสูตร)
    }
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียนโปรแกรมที่ใช้ NIO.2 นับจำนวนไฟล์ `.java` ทั้งหมดในโฟลเดอร์ปัจจุบัน
(รวมโฟลเดอร์ย่อย)

**เฉลย:**

```java
import java.nio.file.Files;
import java.nio.file.Path;
import java.io.IOException;
import java.util.stream.Stream;

public class Exercise1 {
    public static void main(String[] args) throws IOException {
        try (Stream<Path> stream = Files.walk(Path.of("."))) {
            long count = stream.filter(Files::isRegularFile)
                                .filter(p -> p.toString().endsWith(".java"))
                                .count();
            System.out.println("จำนวนไฟล์ .java: " + count);
        }
    }
}
```

**2)** เขียนโปรแกรมที่ใช้ `Files.readAllLines()` อ่านไฟล์ตัวเลข (บรรทัดละหนึ่ง
ตัว) แล้วหาผลรวมและค่าเฉลี่ย

**เฉลย:**

```java
import java.nio.file.Files;
import java.nio.file.Path;
import java.io.IOException;
import java.util.List;

public class Exercise2 {
    public static void main(String[] args) throws IOException {
        Path path = Path.of("numbers.txt");
        Files.write(path, List.of("10", "20", "30", "40"));

        List<String> lines = Files.readAllLines(path);
        int sum = lines.stream().mapToInt(Integer::parseInt).sum();
        double average = (double) sum / lines.size();

        System.out.println("ผลรวม: " + sum + ", เฉลี่ย: " + average);
    }
}
```

**3)** เขียนโปรแกรมสำรองไฟล์ (backup) โดยคัดลอกไฟล์ไปยังชื่อใหม่ที่มี timestamp

**เฉลย:**

```java
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.StandardCopyOption;
import java.io.IOException;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

public class Exercise3 {
    public static void main(String[] args) throws IOException {
        Path source = Path.of("important.txt");
        Files.writeString(source, "ข้อมูลสำคัญ");

        String timestamp = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyyMMdd_HHmmss"));
        Path backup = Path.of("important_backup_" + timestamp + ".txt");

        Files.copy(source, backup, StandardCopyOption.REPLACE_EXISTING);
        System.out.println("สำรองไฟล์ไปที่: " + backup);
    }
}
```

### สรุปเนื้อหา Part 37

- NIO.2 (`java.nio.file`) เป็น API ที่แนะนำสำหรับโค้ดใหม่ มี `Path` แทน `File`
  และ `Files` เป็น utility class หลัก
- `Files.readString()`, `Files.readAllLines()` อ่านไฟล์ได้ในคำสั่งเดียว
  เหมาะกับไฟล์ขนาดไม่ใหญ่มาก
- `Files.walk()` เดินไฟล์ในไดเรกทอรีแบบ recursive, `Files.list()` แสดงระดับเดียว
- `Files.copy()`, `Files.move()`, `Files.delete()` จัดการไฟล์ได้กระชับกว่า
  `java.io` มาก
- `Files.lines()` คืน `Stream<String>` แบบ lazy เหมาะกับไฟล์ขนาดใหญ่ ต้องปิด
  ด้วย try-with-resources เสมอ

**ต่อไป**: [Part 38 — Serialization, Deserialization](./part-038-serialization.md)
