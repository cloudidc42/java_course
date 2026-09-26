# Part 27: Iterator, Iterable, Comparable, Comparator

> ขั้นตอนที่ 261-270 ของหลักสูตร | ระดับ: กลาง-สูง

## สารบัญ

1. `Iterable` และ `Iterator` Interface
2. การสร้าง Custom Iterable Class
3. `ListIterator`: Iterator ที่วนสองทาง
4. `Comparable<T>`: Natural Ordering
5. `Comparator<T>`: Custom Ordering
6. Comparator Chaining (`thenComparing`)
7. `Comparator` แบบ Static/Default Methods (Java 8+)
8. `equals()`, `hashCode()`, `compareTo()`: ความสัมพันธ์ที่ต้อง Consistent
9. เมื่อไรใช้ Comparable เมื่อไรใช้ Comparator
10. แบบฝึกหัดและสรุป

---

## 1. `Iterable` และ `Iterator` Interface

ทบทวนจาก Part 6: **enhanced for-loop (for-each)** ทำงานได้กับ array และ
collection เพราะ collection ทุกตัว implement interface **`Iterable<T>`**
ซึ่งกำหนดให้ต้องมีเมธอด `iterator()` ที่คืนค่า **`Iterator<T>`**

```java
public interface Iterable<T> {
    Iterator<T> iterator();
}

public interface Iterator<T> {
    boolean hasNext();
    T next();
    default void remove() { throw new UnsupportedOperationException(); }
}
```

โค้ด for-each นี้:

```java
for (String item : someList) {
    System.out.println(item);
}
```

ถูก compiler แปลงเป็นโค้ดที่ใช้ `Iterator` ภายในโดยอัตโนมัติ (นี่คือสิ่งที่เกิด
ขึ้น "เบื้องหลัง"):

```java
java.util.Iterator<String> it = someList.iterator();
while (it.hasNext()) {
    String item = it.next();
    System.out.println(item);
}
```

## 2. การสร้าง Custom Iterable Class

เมื่อสร้างโครงสร้างข้อมูลของตัวเอง (เช่น Linked List ใน Part 32) การ implement
`Iterable` ทำให้ใช้กับ for-each ได้ทันที

```java
import java.util.Iterator;
import java.util.NoSuchElementException;

public class Range implements Iterable<Integer> {
    private final int start, end;

    public Range(int start, int end) {
        this.start = start;
        this.end = end;
    }

    @Override
    public Iterator<Integer> iterator() {
        return new Iterator<Integer>() {
            private int current = start;

            @Override
            public boolean hasNext() {
                return current < end;
            }

            @Override
            public Integer next() {
                if (!hasNext()) {
                    throw new NoSuchElementException();
                }
                return current++;
            }
        };
    }
}
```

```java
public class CustomIterableDemo {
    public static void main(String[] args) {
        Range range = new Range(1, 5);

        // ใช้กับ for-each ได้ทันที เพราะ implement Iterable<Integer>
        for (int i : range) {
            System.out.println(i); // 1, 2, 3, 4
        }
    }
}
```

**ทำไมสร้าง Iterator แยกจาก collection object เอง?** เพื่อให้**วน loop หลายตัว
พร้อมกันบน object เดียวกันได้อย่างอิสระ** — แต่ละ Iterator มี state (`current`)
ของตัวเอง ไม่ปนกับตัวอื่น:

```java
public class MultipleIteratorsDemo {
    public static void main(String[] args) {
        Range range = new Range(1, 4);

        Iterator<Integer> it1 = range.iterator();
        Iterator<Integer> it2 = range.iterator();

        System.out.println(it1.next()); // 1 (Iterator แรก)
        System.out.println(it1.next()); // 2 (Iterator แรกทำงานต่อ)
        System.out.println(it2.next()); // 1 (Iterator ตัวที่สองเริ่มใหม่จากต้น - ไม่ปนกับ it1)
    }
}
```

## 3. `ListIterator`: Iterator ที่วนสองทาง

`List` มี Iterator พิเศษชื่อ `ListIterator<T>` ที่**วนได้ทั้งสองทาง** (เดินหน้า
และถอยหลัง) และแก้ไข list ระหว่างวนได้ปลอดภัย (ต่างจาก Iterator ธรรมดาที่แก้ไข
ได้แค่ `remove()`)

```java
import java.util.ArrayList;
import java.util.List;
import java.util.ListIterator;

public class ListIteratorDemo {
    public static void main(String[] args) {
        List<Integer> numbers = new ArrayList<>(List.of(1, 2, 3, 4, 5));

        ListIterator<Integer> it = numbers.listIterator();
        while (it.hasNext()) {
            int value = it.next();
            if (value % 2 == 0) {
                it.set(value * 10); // แก้ไขค่าปัจจุบันได้ (ไม่มีใน Iterator ธรรมดา)
            }
        }
        System.out.println(numbers); // [1, 20, 3, 40, 5]

        // วนถอยหลังจากจุดปัจจุบัน (ตอนนี้อยู่ที่ปลาย list แล้ว)
        while (it.hasPrevious()) {
            System.out.print(it.previous() + " "); // 5 40 3 20 1
        }
    }
}
```

## 4. `Comparable<T>`: Natural Ordering

ทบทวนและขยายความจาก Part 23: `Comparable<T>` กำหนด**ลำดับธรรมชาติ (natural
ordering)** ของ class — มีเพียง `compareTo()` เดียวต่อ class ดังนั้นเหมาะกับ
"ลำดับที่สมเหตุสมผลที่สุดเพียงหนึ่งแบบ" เท่านั้น

```java
public class Product implements Comparable<Product> {
    String name;
    double price;

    public Product(String name, double price) {
        this.name = name;
        this.price = price;
    }

    @Override
    public int compareTo(Product other) {
        // ธรรมเนียม: คืนค่าลบถ้า this < other, 0 ถ้าเท่ากัน, บวกถ้า this > other
        return Double.compare(this.price, other.price); // เรียงตามราคาจากน้อยไปมาก (natural ordering)
    }

    @Override
    public String toString() {
        return name + "(" + price + ")";
    }
}
```

```java
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public class ComparableDemo {
    public static void main(String[] args) {
        List<Product> products = new ArrayList<>(List.of(
            new Product("Laptop", 25000),
            new Product("Mouse", 500),
            new Product("Keyboard", 1200)
        ));

        Collections.sort(products); // ใช้ compareTo() ที่ implement ไว้ (natural ordering)
        System.out.println(products); // [Mouse(500.0), Keyboard(1200.0), Laptop(25000.0)]
    }
}
```

**หลักการเขียน `compareTo()` ที่ถูกต้อง**:
- ใช้ `Integer.compare()`, `Double.compare()` แทนการลบตรง ๆ (`a - b`) เพราะ
  การลบอาจเกิด overflow ได้กับตัวเลขค่าสูงหรือค่าติดลบมาก
- ต้อง**consistent กับ `equals()`**: ถ้า `a.equals(b)` เป็น `true` แล้ว
  `a.compareTo(b)` ควรเป็น `0` เสมอ (ไม่ใช่กฎบังคับทางเทคนิค แต่เป็นข้อแนะนำ
  อย่างยิ่ง เพราะ `TreeSet`/`TreeMap` ใช้ `compareTo()` เพียงอย่างเดียวในการเช็ค
  ความซ้ำ ไม่ใช้ `equals()`)

## 5. `Comparator<T>`: Custom Ordering

**`Comparator<T>`** ให้กำหนด**ลำดับการเรียงแบบอื่น**ได้โดยไม่ต้องแก้ไข class
เดิม — สร้างได้หลายตัวสำหรับ class เดียวกัน (ต่างจาก `Comparable` ที่มีได้แค่
หนึ่งแบบต่อ class)

```java
import java.util.Comparator;

public class NameComparator implements Comparator<Product> {
    @Override
    public int compare(Product p1, Product p2) {
        return p1.name.compareTo(p2.name); // เรียงตามชื่อ (ไม่เกี่ยวกับ natural ordering ที่เป็นราคา)
    }
}
```

```java
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public class ComparatorDemo {
    public static void main(String[] args) {
        List<Product> products = new ArrayList<>(List.of(
            new Product("Laptop", 25000),
            new Product("Mouse", 500),
            new Product("Keyboard", 1200)
        ));

        Collections.sort(products, new NameComparator()); // เรียงตามชื่อแทน (ผ่าน Comparator แยก)
        System.out.println(products); // [Keyboard(1200.0), Laptop(25000.0), Mouse(500.0)]

        // ใช้ lambda expression แทน implement class แยก (กระชับกว่ามาก - Part 39)
        products.sort((p1, p2) -> Double.compare(p2.price, p1.price)); // เรียงราคาจากมากไปน้อย
        System.out.println(products);
    }
}
```

## 6. Comparator Chaining (`thenComparing`)

เมื่อต้องเรียงตามหลายเงื่อนไข (เช่น เรียงตามแผนกก่อน ถ้าแผนกเดียวกันค่อยเรียงตาม
ชื่อ) ใช้ `Comparator.comparing()` และ `thenComparing()` ต่อกันเป็น chain:

```java
public class Employee {
    String department;
    String name;
    double salary;

    public Employee(String department, String name, double salary) {
        this.department = department;
        this.name = name;
        this.salary = salary;
    }

    @Override
    public String toString() {
        return name + "(" + department + ", " + salary + ")";
    }
}
```

```java
import java.util.ArrayList;
import java.util.Comparator;
import java.util.List;

public class ComparatorChainingDemo {
    public static void main(String[] args) {
        List<Employee> employees = new ArrayList<>(List.of(
            new Employee("IT", "Bob", 35000),
            new Employee("HR", "Alice", 30000),
            new Employee("IT", "Charlie", 32000),
            new Employee("HR", "David", 30000)
        ));

        // เรียงตามแผนกก่อน ถ้าแผนกเดียวกันเรียงตามเงินเดือนจากมากไปน้อย
        employees.sort(
            Comparator.comparing((Employee e) -> e.department)
                      .thenComparing(e -> e.salary, Comparator.reverseOrder())
        );

        employees.forEach(System.out::println);
        // HR แผนกแรก: David(30000) หรือ Alice(30000) - เงินเดือนเท่ากัน คงลำดับเดิม (stable sort)
        // IT แผนกถัดไป: Bob(35000), Charlie(32000) - เรียงเงินเดือนมากไปน้อย
    }
}
```

## 7. `Comparator` แบบ Static/Default Methods (Java 8+)

Java 8 เพิ่ม static/default method ให้ `Comparator` ที่ทำให้เขียน comparator
ซับซ้อนได้กระชับมาก (ทบทวนแนวคิด default/static method ใน interface จาก Part 16):

```java
import java.util.Comparator;
import java.util.List;

public class ModernComparatorDemo {
    public static void main(String[] args) {
        List<Employee> employees = new java.util.ArrayList<>(List.of(
            new Employee("IT", "Bob", 35000),
            new Employee("HR", "Alice", 30000)
        ));

        // naturalOrder / reverseOrder: สำหรับชนิดข้อมูลที่มี natural ordering อยู่แล้ว
        List<Integer> numbers = new java.util.ArrayList<>(List.of(3, 1, 4, 1, 5));
        numbers.sort(Comparator.naturalOrder());
        System.out.println(numbers); // [1, 1, 3, 4, 5]
        numbers.sort(Comparator.reverseOrder());
        System.out.println(numbers); // [5, 4, 3, 1, 1]

        // comparing + reversed(): กระชับกว่าการเขียน Comparator.reverseOrder() เอง
        employees.sort(Comparator.comparing((Employee e) -> e.salary).reversed());
        employees.forEach(System.out::println); // Bob ก่อน (เงินเดือนมากกว่า)

        // nullsFirst / nullsLast: จัดการ null ใน comparator อย่างปลอดภัย
        List<String> names = new java.util.ArrayList<>(List.of("Bob", null, "Alice"));
        names.sort(Comparator.nullsFirst(Comparator.naturalOrder()));
        System.out.println(names); // [null, Alice, Bob]
    }
}
```

## 8. `equals()`, `hashCode()`, `compareTo()`: ความสัมพันธ์ที่ต้อง Consistent

สามเมธอดนี้ทำงานร่วมกันในหลาย collection และต้อง**สอดคล้องกัน**เพื่อป้องกัน
พฤติกรรมที่ผิดเพี้ยน:

| กฎ | คำอธิบาย |
|---|---|
| **`equals()` consistent กับ `hashCode()`** | ถ้า `a.equals(b)` เป็น `true` ต้อง `a.hashCode() == b.hashCode()` เสมอ (บังคับ - ไม่งั้น HashMap/HashSet พังตามที่เห็นใน Part 23-24) |
| **`compareTo()` ควร consistent กับ `equals()`** | ถ้า `a.equals(b)` เป็น `true` ควรได้ `a.compareTo(b) == 0` (แนะนำอย่างยิ่ง ไม่ใช่บังคับทางเทคนิค แต่ถ้าไม่ทำตาม `TreeSet`/`TreeMap` จะมองว่า `a` และ `b` เป็นตัวเดียวกัน แม้ `equals()` บอกว่าต่างกัน) |

```java
import java.util.HashSet;
import java.util.Set;
import java.util.TreeSet;

public class ConsistencyProblemDemo {
    public static void main(String[] args) {
        // ตัวอย่างที่ compareTo() ไม่ consistent กับ equals() -> ผลลัพธ์ผิดเพี้ยน
        class BadPoint implements Comparable<BadPoint> {
            int x, y;
            BadPoint(int x, int y) { this.x = x; this.y = y; }

            @Override
            public boolean equals(Object obj) { // เทียบทั้ง x และ y
                if (!(obj instanceof BadPoint)) return false;
                BadPoint p = (BadPoint) obj;
                return x == p.x && y == p.y;
            }

            @Override
            public int compareTo(BadPoint other) { // แต่ compareTo() เทียบแค่ x!
                return Integer.compare(this.x, other.x);
            }
        }

        Set<BadPoint> treeSet = new TreeSet<>();
        treeSet.add(new BadPoint(1, 1));
        treeSet.add(new BadPoint(1, 2)); // x เท่ากัน (compareTo()=0) แต่ equals() ควรเป็น false!

        System.out.println(treeSet.size()); // 1 (!!) TreeSet มองว่าซ้ำกัน เพราะดูแค่ compareTo()
                                              // ทั้งที่ equals() บอกว่าเป็นคนละจุดกัน - นี่คือ bug ร้ายแรง
    }
}
```

## 9. เมื่อไรใช้ Comparable เมื่อไรใช้ Comparator

| ใช้ `Comparable` เมื่อ | ใช้ `Comparator` เมื่อ |
|---|---|
| Class นั้นมีลำดับ "ธรรมชาติ" ที่ชัดเจนเพียงแบบเดียว (เช่น ตัวเลขเรียงจากน้อยไปมาก) | ต้องการเรียงหลายแบบสำหรับ class เดียวกัน |
| เราเป็นเจ้าของและแก้ไข source code ของ class ได้ | ไม่สามารถแก้ไข source code ของ class ได้ (เช่น class จาก library ภายนอก) |
| ต้องการให้ใช้กับ `Collections.sort()`, `TreeSet`, `TreeMap` โดยไม่ต้องระบุอะไรเพิ่ม | ต้องการเรียงแบบชั่วคราว เฉพาะบางสถานการณ์ |

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** สร้าง class `Book` ที่ implement `Comparable<Book>` เรียงตามชื่อหนังสือ
(natural ordering) แล้วสร้าง `Comparator` แยกสำหรับเรียงตามราคา

**เฉลย:**

```java
import java.util.Comparator;

public class Book implements Comparable<Book> {
    String title;
    double price;

    public Book(String title, double price) {
        this.title = title;
        this.price = price;
    }

    @Override
    public int compareTo(Book other) {
        return this.title.compareTo(other.title);
    }

    static final Comparator<Book> BY_PRICE = Comparator.comparingDouble(b -> b.price);

    @Override
    public String toString() {
        return title + "(" + price + ")";
    }
}
```

**2)** สร้าง custom class `NumberRange` ที่ implement `Iterable<Integer>` วนค่า
ตั้งแต่ `start` ถึง `end` โดยก้าวกระโดดทีละ `step` ที่กำหนดเอง

**เฉลย:**

```java
import java.util.Iterator;

public class NumberRange implements Iterable<Integer> {
    int start, end, step;
    public NumberRange(int start, int end, int step) {
        this.start = start; this.end = end; this.step = step;
    }

    @Override
    public Iterator<Integer> iterator() {
        return new Iterator<Integer>() {
            int current = start;
            @Override public boolean hasNext() { return current < end; }
            @Override public Integer next() {
                int value = current;
                current += step;
                return value;
            }
        };
    }
}
```

**3)** เรียง `List<Employee>` ตามแผนก (a-z) แล้วตามด้วยชื่อ (a-z) โดยใช้
`thenComparing`

**เฉลย:**

```java
import java.util.Comparator;
import java.util.List;

public class Exercise3 {
    public static void main(String[] args) {
        List<Employee> employees = new java.util.ArrayList<>(List.of(
            new Employee("IT", "Bob", 35000),
            new Employee("HR", "Alice", 30000)
        ));
        employees.sort(
            Comparator.comparing((Employee e) -> e.department)
                      .thenComparing(e -> e.name)
        );
        employees.forEach(System.out::println);
    }
}
```

### สรุปเนื้อหา Part 27

- `Iterable`/`Iterator` คือกลไกเบื้องหลัง for-each; implement `Iterable` เอง
  เพื่อให้ custom class ใช้กับ for-each ได้
- `ListIterator` วนได้สองทางและแก้ไข list ระหว่างวนได้ปลอดภัยกว่า Iterator ธรรมดา
- `Comparable<T>` กำหนด natural ordering (มีได้แบบเดียวต่อ class), `Comparator<T>`
  กำหนด custom ordering (มีได้หลายแบบ)
- `thenComparing()` ใช้ chain เงื่อนไขการเรียงหลายชั้น
- `equals()`, `hashCode()`, `compareTo()` ต้อง consistent กัน ไม่งั้น
  HashSet/HashMap/TreeSet/TreeMap จะทำงานผิดพลาด

**ต่อไป**: [Part 28 — Wrapper Classes, Autoboxing/Unboxing](./part-028-wrapper-classes.md)
