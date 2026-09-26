# Part 55: Design Patterns: Structural (Adapter, Decorator, Facade, Proxy, Composite)

> ขั้นตอนที่ 541-550 ของหลักสูตร | ระดับ: สูง

## สารบัญ

1. Structural Pattern คืออะไร
2. Adapter Pattern
3. Decorator Pattern
4. Facade Pattern
5. Proxy Pattern
6. Composite Pattern
7. เปรียบเทียบ Adapter, Decorator, Proxy (ที่มักสับสนกัน)
8. ตัวอย่างจริงใน Java Standard Library
9. เมื่อไรควรใช้ Pattern ไหน
10. แบบฝึกหัดและสรุป

---

## 1. Structural Pattern คืออะไร

**Structural Pattern** เกี่ยวข้องกับ**การจัดโครงสร้างความสัมพันธ์ระหว่าง
class/object** เพื่อให้ระบบมีความยืดหยุ่นและบำรุงรักษาง่ายขึ้น — ต่างจาก
Creational Pattern (Part 54) ที่เน้นเรื่อง "จะสร้าง object อย่างไร" กลุ่มนี้
เน้นเรื่อง **"จะจัดวาง object ที่มีอยู่แล้วให้ทำงานร่วมกันอย่างไร"**

## 2. Adapter Pattern

**Adapter** ทำให้ **interface ที่เข้ากันไม่ได้ (incompatible)** สามารถทำงาน
ร่วมกันได้ — เปรียบเสมือนปลั๊กแปลงไฟฟ้าที่แปลงขั้วปลั๊กแบบหนึ่งให้ใช้กับเต้า
เสียบอีกแบบได้

```java
// Interface ที่ระบบเราต้องการใช้
public interface MediaPlayer {
    void play(String fileName);
}

// Library เก่าที่มี interface ต่างกัน (ไม่สามารถแก้ไข source code ได้ - สมมติเป็น 3rd-party library)
public class LegacyAudioPlayer {
    public void playAudioFile(String file) {
        System.out.println("เล่นไฟล์เสียง: " + file);
    }
}

// Adapter: แปลง LegacyAudioPlayer ให้ implement MediaPlayer ได้
public class AudioPlayerAdapter implements MediaPlayer {
    private LegacyAudioPlayer legacyPlayer;

    public AudioPlayerAdapter(LegacyAudioPlayer legacyPlayer) {
        this.legacyPlayer = legacyPlayer;
    }

    @Override
    public void play(String fileName) {
        legacyPlayer.playAudioFile(fileName); // แปลง method call ให้ตรงกับ interface เก่า
    }
}
```

```java
public class AdapterDemo {
    static void playMedia(MediaPlayer player, String file) { // โค้ดของเราทำงานกับ MediaPlayer เท่านั้น
        player.play(file);
    }

    public static void main(String[] args) {
        MediaPlayer adapter = new AudioPlayerAdapter(new LegacyAudioPlayer());
        playMedia(adapter, "song.mp3"); // ใช้ library เก่าผ่าน interface ใหม่ได้อย่างราบรื่น
    }
}
```

**กรณีใช้งานจริง**: การใช้ library ภายนอกที่มี API ไม่ตรงกับ interface ที่
ระบบเราต้องการ, การรวมระบบเก่า (legacy system) เข้ากับระบบใหม่

## 3. Decorator Pattern

**Decorator** เพิ่ม**พฤติกรรมใหม่**ให้กับ object **ตอน runtime** โดยไม่ต้อง
แก้ไข class เดิม หรือใช้ inheritance (ทบทวนข้อจำกัดของ single inheritance
จาก Part 14) — "ห่อ" object เดิมด้วย object ใหม่ที่มี interface เดียวกัน

```java
public interface Coffee {
    double cost();
    String description();
}

public class SimpleCoffee implements Coffee {
    public double cost() { return 30; }
    public String description() { return "กาแฟธรรมดา"; }
}

// Decorator base class: implement interface เดียวกัน และ "ห่อ" Coffee อีกตัวไว้ภายใน
public abstract class CoffeeDecorator implements Coffee {
    protected Coffee decoratedCoffee;

    public CoffeeDecorator(Coffee coffee) {
        this.decoratedCoffee = coffee;
    }
}

public class MilkDecorator extends CoffeeDecorator {
    public MilkDecorator(Coffee coffee) { super(coffee); }

    @Override
    public double cost() { return decoratedCoffee.cost() + 10; } // เพิ่มราคานม
    @Override
    public String description() { return decoratedCoffee.description() + " + นม"; }
}

public class SugarDecorator extends CoffeeDecorator {
    public SugarDecorator(Coffee coffee) { super(coffee); }

    @Override
    public double cost() { return decoratedCoffee.cost() + 5; }
    @Override
    public String description() { return decoratedCoffee.description() + " + น้ำตาล"; }
}
```

```java
public class DecoratorDemo {
    public static void main(String[] args) {
        Coffee coffee = new SimpleCoffee();
        System.out.println(coffee.description() + " ราคา " + coffee.cost()); // กาแฟธรรมดา ราคา 30.0

        // ห่อด้วย decorator ทับกันได้เรื่อย ๆ (compose ได้อิสระ ต่างจาก inheritance ที่ตายตัว)
        Coffee coffeeWithMilk = new MilkDecorator(coffee);
        Coffee coffeeWithMilkAndSugar = new SugarDecorator(coffeeWithMilk);

        System.out.println(coffeeWithMilkAndSugar.description() + " ราคา " + coffeeWithMilkAndSugar.cost());
        // กาแฟธรรมดา + นม + น้ำตาล ราคา 45.0
    }
}
```

**ข้อดีสำคัญ**: เพิ่ม/ลบ feature ได้อย่างอิสระ**ตอน runtime** ไม่ต้องสร้าง
subclass สำหรับทุก combination (ถ้าใช้ inheritance จะต้องมี
`MilkSugarCoffee`, `MilkOnlyCoffee`, `SugarOnlyCoffee` ฯลฯ แยกกันหมด — ระเบิด
จำนวน class แบบ combinatorial)

## 4. Facade Pattern

**Facade** ให้ **interface ที่เรียบง่าย**สำหรับระบบที่ซับซ้อนภายใน — ซ่อน
รายละเอียดการทำงานร่วมกันของหลาย subsystem ไว้หลัง interface เดียว (ทบทวน
แนวคิด Abstraction จาก Part 16)

```java
public class DVDPlayer {
    void on() { System.out.println("DVD Player เปิด"); }
    void play(String movie) { System.out.println("เล่น: " + movie); }
}

public class Projector {
    void on() { System.out.println("โปรเจกเตอร์เปิด"); }
    void setInput(String source) { System.out.println("ตั้งค่า input: " + source); }
}

public class SoundSystem {
    void on() { System.out.println("ระบบเสียงเปิด"); }
    void setVolume(int level) { System.out.println("ตั้งระดับเสียง: " + level); }
}

// Facade: ซ่อนความซับซ้อนของการเปิดอุปกรณ์ทั้งหมดไว้หลัง method เดียว
public class HomeTheaterFacade {
    private DVDPlayer dvd = new DVDPlayer();
    private Projector projector = new Projector();
    private SoundSystem sound = new SoundSystem();

    public void watchMovie(String movie) { // ผู้ใช้เรียกแค่เมธอดนี้เมธอดเดียว
        System.out.println("=== เตรียมดูหนัง ===");
        projector.on();
        projector.setInput("DVD");
        sound.on();
        sound.setVolume(20);
        dvd.on();
        dvd.play(movie);
    }
}
```

```java
public class FacadeDemo {
    public static void main(String[] args) {
        HomeTheaterFacade homeTheater = new HomeTheaterFacade();
        homeTheater.watchMovie("The Matrix");
        // ผู้ใช้ไม่ต้องรู้เลยว่าต้องเปิดอุปกรณ์ 3 ตัวและตั้งค่าตามลำดับที่ถูกต้องอย่างไร
    }
}
```

## 5. Proxy Pattern

**Proxy** เป็น**"ตัวแทน"**ที่ควบคุมการเข้าถึง object จริง — เพิ่ม logic เสริม
ก่อน/หลังการเรียกใช้งานจริง (เช่น lazy loading, access control, logging,
caching) โดยที่ผู้เรียกใช้ไม่รู้ตัวว่ากำลังทำงานผ่าน proxy

```java
public interface Image {
    void display();
}

public class RealImage implements Image {
    private String filename;

    public RealImage(String filename) {
        this.filename = filename;
        loadFromDisk(); // การโหลดไฟล์มี cost สูง (จำลอง)
    }

    private void loadFromDisk() {
        System.out.println("กำลังโหลดไฟล์: " + filename + " (ใช้เวลานาน)");
    }

    @Override
    public void display() {
        System.out.println("แสดงภาพ: " + filename);
    }
}

// Proxy: หน่วงการสร้าง RealImage ไว้จนกว่าจะจำเป็นจริง ๆ (Lazy Loading / Virtual Proxy)
public class ImageProxy implements Image {
    private RealImage realImage;
    private String filename;

    public ImageProxy(String filename) {
        this.filename = filename; // ยังไม่โหลดไฟล์จริง แค่เก็บชื่อไว้ก่อน
    }

    @Override
    public void display() {
        if (realImage == null) {
            realImage = new RealImage(filename); // สร้าง (และโหลด) เฉพาะตอนที่เรียก display() ครั้งแรก
        }
        realImage.display();
    }
}
```

```java
public class ProxyDemo {
    public static void main(String[] args) {
        Image image = new ImageProxy("photo.jpg");
        System.out.println("สร้าง proxy แล้ว แต่ยังไม่โหลดไฟล์เลย");

        image.display(); // โหลดไฟล์จริงตอนนี้เท่านั้น (ครั้งแรกที่เรียก display)
        image.display(); // ครั้งที่สอง: ไม่โหลดซ้ำ เพราะ realImage ถูกสร้างไปแล้ว
    }
}
```

**ชนิดของ Proxy ที่พบบ่อย**: **Virtual Proxy** (lazy loading — ตัวอย่างข้างบน),
**Protection Proxy** (ตรวจสอบสิทธิ์ก่อนอนุญาตเข้าถึง), **Remote Proxy**
(ตัวแทนของ object ที่อยู่บนเครื่องอื่น — คล้ายหลักการของ RMI/gRPC),
**Logging Proxy** (บันทึก log ทุกครั้งที่เรียกใช้งาน)

## 6. Composite Pattern

**Composite** ให้ **object เดี่ยว (leaf)** และ **กลุ่มของ object (composite)**
ถูกจัดการผ่าน**interface เดียวกัน** — เหมาะกับโครงสร้างแบบ**tree** (ทบทวนจาก
Part 33) เช่น ระบบไฟล์ (ไฟล์เดี่ยว vs โฟลเดอร์ที่มีไฟล์ย่อย)

```java
import java.util.ArrayList;
import java.util.List;

public interface FileSystemComponent {
    void showDetails(String indent);
    long getSize();
}

// Leaf: ไฟล์เดี่ยว ไม่มีลูก
public class FileComponent implements FileSystemComponent {
    private String name;
    private long size;

    public FileComponent(String name, long size) {
        this.name = name;
        this.size = size;
    }

    @Override
    public void showDetails(String indent) {
        System.out.println(indent + "📄 " + name + " (" + size + " KB)");
    }

    @Override
    public long getSize() { return size; }
}

// Composite: โฟลเดอร์ที่มีทั้งไฟล์และโฟลเดอร์ย่อยเป็นลูกได้ (ใช้ interface เดียวกัน)
public class FolderComponent implements FileSystemComponent {
    private String name;
    private List<FileSystemComponent> children = new ArrayList<>();

    public FolderComponent(String name) {
        this.name = name;
    }

    public void add(FileSystemComponent component) {
        children.add(component);
    }

    @Override
    public void showDetails(String indent) {
        System.out.println(indent + "📁 " + name + "/");
        for (FileSystemComponent child : children) {
            child.showDetails(indent + "  "); // recursion (ทบทวนจาก Part 8, 29, 33)
        }
    }

    @Override
    public long getSize() {
        long total = 0;
        for (FileSystemComponent child : children) {
            total += child.getSize(); // รวมขนาดของทุกลูก (ไม่สนว่าเป็นไฟล์หรือโฟลเดอร์)
        }
        return total;
    }
}
```

```java
public class CompositeDemo {
    public static void main(String[] args) {
        FolderComponent root = new FolderComponent("project");
        FolderComponent src = new FolderComponent("src");

        src.add(new FileComponent("Main.java", 5));
        src.add(new FileComponent("Utils.java", 3));

        root.add(src);
        root.add(new FileComponent("README.md", 2));

        root.showDetails("");
        System.out.println("ขนาดรวมทั้งหมด: " + root.getSize() + " KB");
        // 📁 project/
        //   📁 src/
        //     📄 Main.java (5 KB)
        //     📄 Utils.java (3 KB)
        //   📄 README.md (2 KB)
        // ขนาดรวมทั้งหมด: 10 KB
    }
}
```

## 7. เปรียบเทียบ Adapter, Decorator, Proxy (ที่มักสับสนกัน)

ทั้งสาม pattern นี้**"ห่อ" object อื่นและ implement interface** คล้ายกันมาก
แต่มี**เจตนา (intent) ต่างกันโดยสิ้นเชิง**:

| Pattern | เจตนา | Interface ของ wrapper กับ wrapped |
|---|---|---|
| **Adapter** | แปลง interface ที่เข้ากันไม่ได้ให้ใช้ร่วมกันได้ | **ต่างกัน** |
| **Decorator** | เพิ่มพฤติกรรมใหม่ | **เหมือนกัน** (เพิ่ม ไม่เปลี่ยน) |
| **Proxy** | ควบคุมการเข้าถึง (access control, lazy loading) | **เหมือนกัน** (โปร่งใสต่อผู้ใช้) |

**หลักจำง่าย ๆ**: Adapter ถามว่า **"ทำให้มันคุยกันได้ไหม"**, Decorator ถามว่า
**"เพิ่มความสามารถได้ไหม"**, Proxy ถามว่า **"ควบคุมการเข้าถึงได้ไหม"**

## 8. ตัวอย่างจริงใน Java Standard Library

```java
import java.io.*;
import java.util.*;

public class RealWorldPatternsDemo {
    public static void main(String[] args) {
        // Decorator Pattern: BufferedReader "ห่อ" FileReader เพิ่มความสามารถ buffer (ทบทวนจาก Part 36)
        // new BufferedReader(new FileReader("file.txt"));

        // Adapter Pattern: Arrays.asList() แปลง array ให้ implement interface List ได้ (ทบทวนจาก Part 22)
        String[] array = {"a", "b", "c"};
        List<String> list = Arrays.asList(array);

        // Proxy Pattern: Collections.unmodifiableList() คืน proxy ที่ป้องกันการแก้ไข (ทบทวนจาก Part 17)
        List<String> unmodifiable = Collections.unmodifiableList(list);

        // Composite Pattern: java.awt.Component/Container ใน GUI programming
        // (Container คือ composite ที่มี Component ย่อยได้ ทั้งคู่ implement interface เดียวกัน)
    }
}
```

## 9. เมื่อไรควรใช้ Pattern ไหน

| Pattern | ใช้เมื่อ |
|---|---|
| **Adapter** | ต้องใช้ library/class ที่มี interface ไม่ตรงกับที่ระบบต้องการ |
| **Decorator** | ต้องเพิ่ม feature หลายแบบผสมกันได้อย่างอิสระ (ไม่ระเบิด class ด้วย inheritance) |
| **Facade** | ระบบซับซ้อนมาก ต้องการ interface ง่าย ๆ ให้ผู้ใช้ทั่วไปเรียกใช้ |
| **Proxy** | ต้องควบคุมการเข้าถึง object (lazy loading, security, logging, caching) |
| **Composite** | โครงสร้างข้อมูลแบบ tree ที่ต้องจัดการ leaf และ group ผ่าน interface เดียวกัน |

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** สร้าง Decorator สำหรับ `Coffee` เพิ่มเติม: `WhipCreamDecorator` (เพิ่ม
15 บาท) แล้วห่อกาแฟด้วยทั้ง milk, sugar, และ whip cream

**เฉลย:**

```java
public class WhipCreamDecorator extends CoffeeDecorator {
    public WhipCreamDecorator(Coffee coffee) { super(coffee); }

    @Override
    public double cost() { return decoratedCoffee.cost() + 15; }
    @Override
    public String description() { return decoratedCoffee.description() + " + วิปครีม"; }
}
```

```java
public class Exercise1 {
    public static void main(String[] args) {
        Coffee coffee = new WhipCreamDecorator(
                         new SugarDecorator(
                         new MilkDecorator(
                         new SimpleCoffee())));
        System.out.println(coffee.description() + " = " + coffee.cost());
        // กาแฟธรรมดา + นม + น้ำตาล + วิปครีม = 60.0
    }
}
```

**2)** สร้าง Proxy ที่ทำ **access control**: อนุญาตให้เรียก `deleteFile()`
เฉพาะถ้า user เป็น "admin" เท่านั้น

**เฉลย:**

```java
public interface FileManager {
    void deleteFile(String name);
}

public class RealFileManager implements FileManager {
    public void deleteFile(String name) { System.out.println("ลบไฟล์: " + name); }
}

public class ProtectedFileManagerProxy implements FileManager {
    private RealFileManager realManager = new RealFileManager();
    private String userRole;

    public ProtectedFileManagerProxy(String userRole) {
        this.userRole = userRole;
    }

    @Override
    public void deleteFile(String name) {
        if (!"admin".equals(userRole)) {
            throw new SecurityException("ไม่มีสิทธิ์ลบไฟล์");
        }
        realManager.deleteFile(name);
    }
}
```

**3)** อธิบายความแตกต่างระหว่าง Adapter และ Decorator ด้วยคำพูดของตัวเอง

**เฉลย**: Adapter ใช้เมื่อ interface ของสอง class **เข้ากันไม่ได้** และต้อง
การ "แปลง" ให้คุยกันได้ (interface ของ wrapper ต่างจาก wrapped object) ในขณะ
ที่ Decorator ใช้เมื่อ interface **เหมือนกันอยู่แล้ว** แต่ต้องการ "เพิ่ม"
พฤติกรรมใหม่เข้าไปโดยไม่แก้ไข class เดิม (ทั้ง wrapper และ wrapped ใช้
interface เดียวกัน ผู้เรียกใช้มองไม่เห็นความแตกต่าง) — พูดง่าย ๆ Adapter
แก้ปัญหา "เข้ากันไม่ได้" ส่วน Decorator แก้ปัญหา "อยากเพิ่มความสามารถแบบ
ยืดหยุ่น"

### สรุปเนื้อหา Part 55

- Structural Pattern จัดโครงสร้างความสัมพันธ์ระหว่าง object เพื่อความยืดหยุ่น
- Adapter แปลง interface ที่เข้ากันไม่ได้ให้ทำงานร่วมกันได้
- Decorator เพิ่มพฤติกรรมตอน runtime โดยไม่แก้ไข class เดิม, compose ได้อิสระ
  กว่า inheritance
- Facade ซ่อนความซับซ้อนของระบบหลัง interface ง่าย ๆ
- Proxy ควบคุมการเข้าถึง object (lazy loading, security, logging)
- Composite จัดการ object เดี่ยวและกลุ่มผ่าน interface เดียวกัน เหมาะกับ
  โครงสร้างแบบ tree

**จบหมวด Design Patterns เบื้องต้น (Creational + Structural, Part 54-55)!
Behavioral Patterns จะอยู่ใน Part 56**

**ต่อไป**: [Part 56 — Design Patterns: Behavioral (Observer, Strategy, Command, Template Method, State)](./part-056-design-patterns-behavioral.md)
