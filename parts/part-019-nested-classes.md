# Part 19: Nested Classes, Inner Classes, Local Classes, Anonymous Classes

> ขั้นตอนที่ 181-190 ของหลักสูตร | ระดับ: OOP ขั้นกลาง

## สารบัญ

1. ภาพรวม 4 ประเภทของ Nested Class
2. Static Nested Class
3. Inner Class (Non-static Nested Class)
4. Local Class (Class ในเมธอด)
5. Anonymous Class
6. Anonymous Class กับ Interface และ Abstract Class
7. Effectively Final Variables ใน Local/Anonymous Class
8. เปรียบเทียบ Anonymous Class กับ Lambda (ปูทางสู่ Part 39)
9. กรณีใช้งานจริงของแต่ละประเภท
10. แบบฝึกหัดและสรุป

---

## 1. ภาพรวม 4 ประเภทของ Nested Class

**Nested Class** คือ class ที่ประกาศอยู่ภายใน class อื่น แบ่งเป็น 4 ประเภท:

| ประเภท | ประกาศที่ไหน | เข้าถึง instance ของ outer class ได้ไหม | ใช้บ่อยแค่ไหน |
|---|---|---|---|
| **Static Nested Class** | ระดับคลาส + มี `static` | ไม่ได้ (ไม่ผูกกับ instance ใด) | บ่อยมาก |
| **Inner Class** | ระดับคลาส (ไม่มี `static`) | ได้ (ผูกกับ instance ของ outer เสมอ) | ปานกลาง |
| **Local Class** | ภายในเมธอด | ได้ (ถ้าไม่ใช่ static context) | น้อย |
| **Anonymous Class** | ภายในนิพจน์ ไม่มีชื่อ | ได้ (ถ้าไม่ใช่ static context) | บ่อยมาก (ก่อนมี Lambda) |

## 2. Static Nested Class

ทบทวนจาก Part 17: Static nested class ทำงาน**เหมือน class ปกติ**ที่บังเอิญ
ประกาศอยู่ภายใน class อื่นเพื่อจัดกลุ่มโค้ดที่เกี่ยวข้องกันให้เป็นระเบียบ
**ไม่ผูกกับ instance ของ outer class** จึงเข้าถึง instance field/method ของ
outer class โดยตรงไม่ได้

```java
public class LinkedListDemo {
    private Node head;

    // Static nested class: ใช้กันมากในการสร้างโครงสร้างข้อมูล (Part 32 จะลงลึก)
    static class Node {
        int value;
        Node next;

        Node(int value) {
            this.value = value;
        }
    }

    void addFirst(int value) {
        Node newNode = new Node(value); // สร้างได้โดยไม่ต้องพึ่ง instance ของ LinkedListDemo
        newNode.next = head;
        head = newNode;
    }

    void printAll() {
        Node current = head;
        while (current != null) {
            System.out.print(current.value + " -> ");
            current = current.next;
        }
        System.out.println("null");
    }
}
```

```java
public class StaticNestedDemo {
    public static void main(String[] args) {
        LinkedListDemo list = new LinkedListDemo();
        list.addFirst(3);
        list.addFirst(2);
        list.addFirst(1);
        list.printAll(); // 1 -> 2 -> 3 -> null

        // สร้าง Node จากภายนอกได้โดยตรง (ถ้า access modifier เปิด) โดยไม่ต้องมี LinkedListDemo instance
        LinkedListDemo.Node standalone = new LinkedListDemo.Node(99);
        System.out.println(standalone.value);
    }
}
```

**ประโยชน์**: จัดกลุ่มโค้ดที่เกี่ยวข้องกันแน่นแฟ้น (เช่น `Node` มีความหมายเฉพาะ
ภายใน `LinkedListDemo` เท่านั้น) โดยไม่ต้องแยกไฟล์ และยังคง namespace ที่ชัดเจน
(`LinkedListDemo.Node`)

## 3. Inner Class (Non-static Nested Class)

**Inner Class** คือ nested class ที่**ไม่มี `static`** — ต้องผูกกับ **instance
ของ outer class เสมอ** และสามารถเข้าถึง field/method (แม้เป็น private) ของ
instance นั้นได้โดยตรง

```java
public class Car {
    private String model;
    private int fuelLevel = 100;

    public Car(String model) {
        this.model = model;
    }

    // Inner class: ไม่มี static -> ต้องผูกกับ instance ของ Car เสมอ
    class Engine {
        void start() {
            // เข้าถึง private field ของ outer class (Car) ได้โดยตรง!
            if (fuelLevel > 0) {
                System.out.println(model + " สตาร์ทเครื่องยนต์สำเร็จ (น้ำมัน: " + fuelLevel + "%)");
            } else {
                System.out.println(model + " สตาร์ทไม่ได้ น้ำมันหมด!");
            }
        }
    }

    Engine createEngine() {
        return new Engine(); // สร้าง inner class จากภายใน outer class ได้ตรง ๆ
    }
}
```

```java
public class InnerClassDemo {
    public static void main(String[] args) {
        Car myCar = new Car("Toyota Camry");

        // วิธีที่ 1: สร้างผ่านเมธอดของ outer class (แนะนำ อ่านง่ายกว่า)
        Car.Engine engine1 = myCar.createEngine();
        engine1.start();

        // วิธีที่ 2: สร้าง inner class โดยตรงจากภายนอก (syntax พิเศษที่ต้องระบุ outer instance)
        Car.Engine engine2 = myCar.new Engine();
        engine2.start();
    }
}
```

**ข้อควรระวัง**: Inner class ที่ผูกกับ instance มักทำให้เกิด **memory leak**
ได้ง่ายถ้า inner class object มีชีวิตอยู่นานกว่าที่ควร เพราะมันถือ reference
ไปยัง outer class instance อยู่ตลอดเวลา (แม้ outer instance นั้นจะไม่มีที่ใช้
แล้วก็ตาม garbage collector จะเก็บกวาดไม่ได้)

## 4. Local Class (Class ในเมธอด)

**Local Class** คือ class ที่ประกาศ**ภายในเมธอด** — มองเห็นได้เฉพาะภายในเมธอดนั้น
เท่านั้น ใช้เมื่อต้องการ class ช่วยเหลือที่ใช้แค่ในเมธอดเดียว ไม่จำเป็นต้องแชร์
กับส่วนอื่นของโปรแกรม

```java
public class OrderProcessor {
    void processOrders(java.util.List<Double> prices) {
        final double taxRate = 0.07;

        // Local class: มองเห็นได้แค่ภายในเมธอดนี้เท่านั้น
        class PriceCalculator {
            double calculateWithTax(double price) {
                return price * (1 + taxRate); // เข้าถึงตัวแปร local ของเมธอดที่ครอบมันได้
            }
        }

        PriceCalculator calculator = new PriceCalculator();
        for (double price : prices) {
            System.out.printf("ราคา %.2f รวมภาษี: %.2f%n", price, calculator.calculateWithTax(price));
        }
    }
}
```

```java
public class LocalClassDemo {
    public static void main(String[] args) {
        OrderProcessor processor = new OrderProcessor();
        processor.processOrders(java.util.List.of(100.0, 250.0, 500.0));
        // PriceCalculator calc = new PriceCalculator(); // Error! มองไม่เห็นจากนอกเมธอด processOrders
    }
}
```

## 5. Anonymous Class

**Anonymous Class** คือ class ที่**ไม่มีชื่อ** ประกาศและสร้าง object พร้อมกันใน
คำสั่งเดียว เหมาะสำหรับกรณีที่ต้องการ implementation แบบใช้ครั้งเดียว (one-off)
โดยไม่ต้องสร้าง class แยกไฟล์

```java
public abstract class Animal {
    abstract void makeSound();
}
```

```java
public class AnonymousClassDemo {
    public static void main(String[] args) {
        // สร้าง object จาก anonymous class ที่ extends Animal พร้อม implement ทันที
        Animal cat = new Animal() {
            @Override
            void makeSound() {
                System.out.println("เมี้ยว~");
            }
        };

        cat.makeSound();

        // ใช้ได้กับ interface เช่นกัน (พบบ่อยมากในโค้ดก่อน Java 8)
        Runnable task = new Runnable() {
            @Override
            public void run() {
                System.out.println("กำลังทำงานใน thread");
            }
        };
        task.run();
    }
}
```

## 6. Anonymous Class กับ Interface และ Abstract Class

Anonymous class สามารถ implement interface หรือ extends abstract class ได้
(**แต่ทำได้แค่อย่างใดอย่างหนึ่งเท่านั้น** และ **implement ได้แค่ 1 interface**
เพราะ anonymous class ไม่มีชื่อให้เขียน `implements A, B` หลายตัวได้)

```java
import java.util.Comparator;
import java.util.ArrayList;
import java.util.List;

public class AnonymousComparatorDemo {
    public static void main(String[] args) {
        List<String> names = new ArrayList<>(List.of("Charlie", "Alice", "Bob"));

        // ใช้ anonymous class implement Comparator เพื่อกำหนดวิธีเรียงลำดับเอง
        names.sort(new Comparator<String>() {
            @Override
            public int compare(String a, String b) {
                return b.compareTo(a); // เรียงจาก Z ไป A (reverse order)
            }
        });

        System.out.println(names); // [Charlie, Bob, Alice]
    }
}
```

## 7. Effectively Final Variables ใน Local/Anonymous Class

Local class และ anonymous class สามารถเข้าถึงตัวแปร local ของเมธอดที่ครอบมันได้
แต่**ตัวแปรนั้นต้องเป็น `final` หรือ "effectively final"** (คือค่าไม่เปลี่ยนแปลง
หลังกำหนดครั้งแรก แม้จะไม่ได้เขียน keyword `final` ก็ตาม)

```java
public class EffectivelyFinalDemo {
    static Runnable createTask(String message) {
        int counter = 10; // effectively final: ไม่มีการเปลี่ยนแปลงค่าหลังจากนี้เลย

        // counter = 20; // ถ้าเพิ่มบรรทัดนี้ counter จะไม่ effectively final อีกต่อไป
        //                // และจะทำให้ anonymous class ด้านล่างนี้ compile error ทันที

        return new Runnable() {
            @Override
            public void run() {
                System.out.println(message + " (counter=" + counter + ")");
                // เข้าถึง message และ counter ได้เพราะทั้งคู่เป็น effectively final
            }
        };
    }

    public static void main(String[] args) {
        Runnable task = createTask("สวัสดี");
        task.run();
    }
}
```

**เหตุผลเชิงเทคนิค**: local/anonymous class จะ**คัดลอกค่า**ของตัวแปร local ที่ใช้
เก็บไว้เป็นของตัวเอง (เพราะเมธอดที่ครอบมันอาจ return และตัวแปร local ถูกทำลายไป
จาก stack แล้ว แต่ object ของ local/anonymous class ยังมีชีวิตอยู่ต่อ) ถ้ายอมให้
แก้ไขค่าได้ จะเกิดความไม่สอดคล้องกันระหว่างค่าที่ local class เห็นกับค่าจริงใน
เมธอด

## 8. เปรียบเทียบ Anonymous Class กับ Lambda (ปูทางสู่ Part 39)

ตั้งแต่ Java 8 **Lambda Expression** (Part 39) เข้ามาแทนที่ anonymous class ในกรณี
ที่ implement **functional interface** (interface ที่มี abstract method เดียว –
ทบทวนจาก Part 16) ทำให้โค้ดกระชับขึ้นมาก:

```java
import java.util.Comparator;
import java.util.ArrayList;
import java.util.List;

public class AnonymousVsLambdaDemo {
    public static void main(String[] args) {
        List<String> names = new ArrayList<>(List.of("Charlie", "Alice", "Bob"));

        // แบบ Anonymous Class (ยาว)
        names.sort(new Comparator<String>() {
            @Override
            public int compare(String a, String b) {
                return a.compareTo(b);
            }
        });

        // แบบ Lambda Expression (กระชับกว่ามาก - ทำสิ่งเดียวกันทุกประการ)
        names.sort((a, b) -> a.compareTo(b));

        System.out.println(names);
    }
}
```

**ข้อควรรู้**: Lambda ใช้แทน anonymous class ได้เฉพาะกรณี **functional interface**
เท่านั้น (abstract method เดียว) ถ้าเป็น **abstract class** หรือ interface ที่มี
abstract method หลายตัว ยังต้องใช้ anonymous class เหมือนเดิม (ดูตัวอย่าง `Animal`
ในหัวข้อ 5 ซึ่งเป็น abstract class ใช้ lambda แทนไม่ได้)

## 9. กรณีใช้งานจริงของแต่ละประเภท

| ประเภท | กรณีใช้งานจริง |
|---|---|
| Static Nested Class | Helper class ที่เกี่ยวข้องแน่นแฟ้นกับ outer class เช่น `Node` ใน LinkedList, `Builder` ใน Builder Pattern (Part 12) |
| Inner Class | เมื่อ nested class ต้องเข้าถึง state ของ outer instance เยอะมาก เช่น Iterator ที่เขียนเองสำหรับ collection ที่กำหนด |
| Local Class | Logic ช่วยเหลือที่ใช้แค่ภายในเมธอดเดียว ซับซ้อนกว่าจะใช้ lambda ได้ |
| Anonymous Class | Event listener, callback แบบ one-off, หรือ implement interface ที่มี method มากกว่า 1 ตัว |

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** สร้าง class `House` ที่มี inner class `Room` ซึ่งเข้าถึง field
`houseAddress` ของ outer class ได้ พร้อมเมธอด `printRoomInfo()`

**เฉลย:**

```java
public class House {
    private String houseAddress;

    public House(String houseAddress) {
        this.houseAddress = houseAddress;
    }

    class Room {
        String roomName;

        Room(String roomName) {
            this.roomName = roomName;
        }

        void printRoomInfo() {
            System.out.println(roomName + " อยู่ที่บ้านเลขที่ " + houseAddress);
        }
    }

    Room createRoom(String name) {
        return new Room(name);
    }
}
```

**2)** ใช้ anonymous class implement interface `Comparator<Integer>` เพื่อเรียง
list ตัวเลขจากมากไปน้อย

**เฉลย:**

```java
import java.util.ArrayList;
import java.util.Comparator;
import java.util.List;

public class Exercise2 {
    public static void main(String[] args) {
        List<Integer> numbers = new ArrayList<>(List.of(5, 2, 8, 1, 9));
        numbers.sort(new Comparator<Integer>() {
            @Override
            public int compare(Integer a, Integer b) {
                return b - a;
            }
        });
        System.out.println(numbers); // [9, 8, 5, 2, 1]
    }
}
```

**3)** อธิบายว่าทำไมโค้ดนี้ compile error:

```java
Runnable createTask() {
    int x = 0;
    Runnable r = () -> System.out.println(x);
    x = 5; // บรรทัดนี้ทำให้เกิดปัญหา
    return r;
}
```

**เฉลย**: เพราะการเปลี่ยนค่า `x = 5` หลังจากที่ lambda (ซึ่งทำงานเหมือน anonymous
class ภายใน) ได้อ้างอิงถึง `x` ไปแล้ว ทำให้ `x` ไม่ใช่ "effectively final" อีก
ต่อไป (มีการเปลี่ยนค่าหลังกำหนดครั้งแรก) — lambda/anonymous class/local class
กำหนดให้ตัวแปร local ที่ใช้ต้องเป็น final หรือ effectively final เท่านั้น
เพื่อป้องกันความไม่สอดคล้องกันของค่าระหว่าง object กับตัวแปรใน stack frame เดิม

### สรุปเนื้อหา Part 19

- Nested class มี 4 ประเภท: Static Nested, Inner, Local, Anonymous — แต่ละแบบ
  มีขอบเขตการเข้าถึงและกรณีใช้งานต่างกัน
- Static Nested Class ไม่ผูกกับ instance ของ outer class, Inner Class ผูกกับ
  instance เสมอและเข้าถึง private field ได้โดยตรง
- Local Class มองเห็นได้แค่ในเมธอดที่ประกาศ, Anonymous Class สร้าง object พร้อม
  implement ในคำสั่งเดียวโดยไม่มีชื่อ
- ตัวแปร local ที่ local/anonymous class ใช้ ต้องเป็น final หรือ effectively final
- Lambda Expression (Java 8+) แทนที่ anonymous class ได้เฉพาะกรณี functional
  interface เท่านั้น

**ต่อไป**: [Part 20 — Packages, การจัดระเบียบโปรเจกต์](./part-020-packages.md)
