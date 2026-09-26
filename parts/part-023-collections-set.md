# Part 23: Collections Framework: Set (HashSet, LinkedHashSet, TreeSet)

> ขั้นตอนที่ 221-230 ของหลักสูตร | ระดับ: กลาง

## สารบัญ

1. `Set` Interface คืออะไร
2. `HashSet`: เร็วที่สุด ไม่มีลำดับ
3. `LinkedHashSet`: รักษาลำดับการเพิ่ม
4. `TreeSet`: เรียงลำดับอัตโนมัติ
5. เปรียบเทียบ HashSet, LinkedHashSet, TreeSet (Big O)
6. `hashCode()` และ `equals()`: หัวใจของ HashSet
7. `Comparable` และการเรียงลำดับใน TreeSet
8. เมธอด Set Operations (Union, Intersection, Difference)
9. กรณีใช้งานจริงของ Set
10. แบบฝึกหัดและสรุป

---

## 1. `Set` Interface คืออะไร

**`Set<E>`** คือ collection ที่**ไม่อนุญาตให้มีค่าซ้ำกัน** (no duplicate elements)
— ถ้าเพิ่มค่าที่มีอยู่แล้ว `add()` จะคืนค่า `false` และไม่มีอะไรเกิดขึ้น (ต่างจาก
`List` ที่เพิ่มค่าซ้ำได้ตามปกติ)

```java
import java.util.HashSet;
import java.util.Set;

public class SetBasicDemo {
    public static void main(String[] args) {
        Set<String> uniqueNames = new HashSet<>();

        System.out.println(uniqueNames.add("Alice")); // true - เพิ่มสำเร็จ
        System.out.println(uniqueNames.add("Bob"));    // true
        System.out.println(uniqueNames.add("Alice"));  // false - ซ้ำ ไม่ถูกเพิ่มเข้าไปอีก

        System.out.println(uniqueNames);      // [Alice, Bob] (ลำดับไม่การันตี ขึ้นกับ implementation)
        System.out.println(uniqueNames.size()); // 2 (ไม่ใช่ 3 ทั้งที่ add ไป 3 ครั้ง)
    }
}
```

Java มี 3 implementation หลักของ `Set` ที่มีคุณสมบัติต่างกันชัดเจน

## 2. `HashSet`: เร็วที่สุด ไม่มีลำดับ

`HashSet` ใช้ **hash table** ภายใน ทำให้การเพิ่ม/ลบ/ค้นหาเร็วที่สุด (**O(1)**
โดยเฉลี่ย) แต่**ไม่รักษาลำดับการเพิ่ม**ข้อมูลเลย (ลำดับที่แสดงอาจดูสุ่ม)

```java
import java.util.HashSet;
import java.util.Set;

public class HashSetDemo {
    public static void main(String[] args) {
        Set<Integer> numbers = new HashSet<>();
        numbers.add(50);
        numbers.add(10);
        numbers.add(30);
        numbers.add(20);

        System.out.println(numbers); // ลำดับไม่แน่นอน อาจได้ [50, 20, 10, 30] หรือลำดับอื่น
                                       // ไม่ควรเขียนโค้ดที่พึ่งพาลำดับของ HashSet เด็ดขาด

        System.out.println(numbers.contains(30)); // true - ค้นหาเร็วมาก O(1)
        numbers.remove(30);
        System.out.println(numbers.contains(30)); // false
    }
}
```

**กรณีใช้งาน**: เมื่อต้องการแค่**ตรวจสอบว่ามีค่านี้อยู่หรือไม่**และ**ไม่สนใจลำดับ**
เลย (เช่น เก็บรายการ ID ที่เคยประมวลผลแล้ว เพื่อไม่ให้ประมวลผลซ้ำ)

## 3. `LinkedHashSet`: รักษาลำดับการเพิ่ม

`LinkedHashSet` ทำงานเหมือน `HashSet` (ไม่มีค่าซ้ำ, เร็ว O(1)) แต่**รักษาลำดับ
การเพิ่มข้อมูลไว้** (insertion order) โดยใช้ linked list ภายในเพิ่มเติมจาก hash
table — เสียประสิทธิภาพเล็กน้อยแลกกับการมีลำดับที่คาดเดาได้

```java
import java.util.LinkedHashSet;
import java.util.Set;

public class LinkedHashSetDemo {
    public static void main(String[] args) {
        Set<Integer> numbers = new LinkedHashSet<>();
        numbers.add(50);
        numbers.add(10);
        numbers.add(30);
        numbers.add(20);

        System.out.println(numbers); // [50, 10, 30, 20] - รักษาลำดับที่ add เข้าไปเสมอ
    }
}
```

**กรณีใช้งาน**: เมื่อต้องการกำจัดค่าซ้ำ**แต่ยังต้องการรักษาลำดับเดิม** เช่น
ตัวอย่างการกำจัดชื่อซ้ำจาก Part 22 ที่ใช้ `LinkedHashSet` เพื่อคง**ลำดับที่ปรากฏ
ครั้งแรก**ของแต่ละชื่อ

## 4. `TreeSet`: เรียงลำดับอัตโนมัติ

`TreeSet` ใช้ **Red-Black Tree** (โครงสร้างข้อมูลแบบ balanced binary search tree
— จะเรียนพื้นฐาน BST ใน Part 33) ภายใน ทำให้**ข้อมูลถูกเรียงลำดับอัตโนมัติเสมอ**
ตามค่าธรรมชาติ (natural ordering) หรือ `Comparator` ที่กำหนดเอง

```java
import java.util.Set;
import java.util.TreeSet;

public class TreeSetDemo {
    public static void main(String[] args) {
        Set<Integer> numbers = new TreeSet<>();
        numbers.add(50);
        numbers.add(10);
        numbers.add(30);
        numbers.add(20);

        System.out.println(numbers); // [10, 20, 30, 50] - เรียงจากน้อยไปมากเสมอโดยอัตโนมัติ!

        TreeSet<Integer> treeSet = new TreeSet<>(numbers);
        System.out.println(treeSet.first());  // 10 (ค่าน้อยที่สุด)
        System.out.println(treeSet.last());    // 50 (ค่ามากที่สุด)
        System.out.println(treeSet.higher(20)); // 30 (ค่าถัดไปที่มากกว่า 20)
        System.out.println(treeSet.lower(20));  // 10 (ค่าก่อนหน้าที่น้อยกว่า 20)
        System.out.println(treeSet.headSet(30)); // [10, 20] (ทุกค่าที่น้อยกว่า 30)
        System.out.println(treeSet.tailSet(20)); // [20, 30, 50] (ทุกค่าที่มากกว่าหรือเท่ากับ 20)
    }
}
```

**กรณีใช้งาน**: เมื่อต้องการให้ข้อมูล**เรียงลำดับอยู่เสมอ**โดยไม่ต้องเรียก
`Collections.sort()` เอง (เช่น leaderboard, ตารางเวลาที่ต้องเรียงตามเวลา)

## 5. เปรียบเทียบ HashSet, LinkedHashSet, TreeSet (Big O)

| การดำเนินการ | `HashSet` | `LinkedHashSet` | `TreeSet` |
|---|---|---|---|
| `add()` / `remove()` / `contains()` | **O(1)** โดยเฉลี่ย | O(1) โดยเฉลี่ย (ช้ากว่าเล็กน้อย) | **O(log n)** |
| รักษาลำดับ | ❌ ไม่มีการันตี | ✅ ลำดับการเพิ่ม (insertion order) | ✅ เรียงลำดับตามค่า (sorted order) |
| หน่วยความจำ | น้อยที่สุด | มากกว่า HashSet เล็กน้อย | มากที่สุด (ต้องเก็บโครงสร้าง tree) |
| ใช้เมื่อ | เร็วที่สุด ไม่สนใจลำดับ | ต้องการลำดับการเพิ่ม | ต้องการข้อมูลเรียงลำดับเสมอ |

## 6. `hashCode()` และ `equals()`: หัวใจของ HashSet

`HashSet` (และ `HashMap` ใน Part 24) ทำงานโดยใช้ **`hashCode()`** เพื่อหาว่าข้อมูล
ควรอยู่ที่ "bucket" ไหนในหน่วยความจำ แล้วใช้ **`equals()`** เพื่อเช็คว่าเป็นค่า
เดียวกันจริงหรือไม่ (กรณีมี hash ชนกัน) — ถ้า class ของเราไม่ override ทั้งสอง
เมธอดนี้อย่างถูกต้อง `HashSet` จะทำงานผิดพลาด

```java
public class Point {
    int x, y;
    public Point(int x, int y) { this.x = x; this.y = y; }

    // ไม่ override equals() และ hashCode() -> ใช้ค่า default จาก Object
    // (เปรียบเทียบแค่ reference เท่านั้น ไม่เปรียบเทียบเนื้อหา)
}
```

```java
import java.util.HashSet;
import java.util.Set;

public class MissingEqualsHashCodeDemo {
    public static void main(String[] args) {
        Set<Point> points = new HashSet<>();
        points.add(new Point(1, 2));
        points.add(new Point(1, 2)); // เนื้อหาเหมือนกันเป๊ะ แต่เป็นคนละ object

        System.out.println(points.size()); // 2 (!!) ควรจะเป็น 1 ถ้ามองว่าจุดเดียวกันคือค่าเดียวกัน
                                              // เพราะไม่ได้ override equals()/hashCode() จึงถือว่าต่างกัน
    }
}
```

**วิธีแก้ที่ถูกต้อง**: override ทั้ง `equals()` และ `hashCode()` ให้สอดคล้องกัน
(**กฎเหล็ก**: ถ้า override ตัวหนึ่ง**ต้อง**override อีกตัวด้วยเสมอ — จะอธิบาย
เหตุผลเชิงลึกใน Part 27):

```java
import java.util.Objects;

public class GoodPoint {
    int x, y;
    public GoodPoint(int x, int y) { this.x = x; this.y = y; }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (!(obj instanceof GoodPoint)) return false;
        GoodPoint other = (GoodPoint) obj;
        return this.x == other.x && this.y == other.y;
    }

    @Override
    public int hashCode() {
        return Objects.hash(x, y); // สร้าง hash code จาก field ที่ใช้ใน equals() เสมอ
    }
}
```

```java
import java.util.HashSet;
import java.util.Set;

public class CorrectEqualsHashCodeDemo {
    public static void main(String[] args) {
        Set<GoodPoint> points = new HashSet<>();
        points.add(new GoodPoint(1, 2));
        points.add(new GoodPoint(1, 2)); // ตอนนี้ HashSet รู้ว่าเนื้อหาเหมือนกัน

        System.out.println(points.size()); // 1 (ถูกต้องแล้ว!)
    }
}
```

## 7. `Comparable` และการเรียงลำดับใน `TreeSet`

`TreeSet` ต้องรู้วิธี**เปรียบเทียบ**ค่าเพื่อเรียงลำดับ — สำหรับชนิดข้อมูลพื้นฐาน
(`Integer`, `String`) มี natural ordering ให้อยู่แล้ว แต่สำหรับ class ของเราเอง
ต้อง implement `Comparable<T>` (จะลงลึกเต็มรูปแบบใน Part 27):

```java
public class Employee implements Comparable<Employee> {
    String name;
    double salary;

    public Employee(String name, double salary) {
        this.name = name;
        this.salary = salary;
    }

    @Override
    public int compareTo(Employee other) {
        return Double.compare(this.salary, other.salary); // เรียงตามเงินเดือนจากน้อยไปมาก
    }

    @Override
    public String toString() {
        return name + "(" + salary + ")";
    }
}
```

```java
import java.util.TreeSet;

public class ComparableTreeSetDemo {
    public static void main(String[] args) {
        TreeSet<Employee> employees = new TreeSet<>();
        employees.add(new Employee("Alice", 35000));
        employees.add(new Employee("Bob", 28000));
        employees.add(new Employee("Charlie", 42000));

        System.out.println(employees); // [Bob(28000.0), Alice(35000.0), Charlie(42000.0)]
                                         // เรียงตามเงินเดือนอัตโนมัติเพราะ implement Comparable แล้ว
    }
}
```

ถ้าไม่ต้องการแก้ไข class เดิม หรือต้องการเรียงลำดับแบบอื่นในบางสถานการณ์ ใช้
`Comparator` แทนได้ (Part 27 จะลงรายละเอียดความแตกต่าง):

```java
import java.util.Comparator;
import java.util.TreeSet;

public class ComparatorTreeSetDemo {
    public static void main(String[] args) {
        TreeSet<Employee> byNameDesc = new TreeSet<>(
            Comparator.comparing((Employee e) -> e.name).reversed()
        );
        byNameDesc.add(new Employee("Alice", 35000));
        byNameDesc.add(new Employee("Bob", 28000));
        byNameDesc.add(new Employee("Charlie", 42000));

        System.out.println(byNameDesc); // เรียงตามชื่อจาก Z ไป A
    }
}
```

## 8. เมธอด Set Operations (Union, Intersection, Difference)

`Set` ใน Java รองรับการดำเนินการทางคณิตศาสตร์เซตพื้นฐานผ่านเมธอดที่สืบทอดมาจาก
`Collection`:

```java
import java.util.HashSet;
import java.util.Set;

public class SetOperationsDemo {
    public static void main(String[] args) {
        Set<Integer> setA = new HashSet<>(Set.of(1, 2, 3, 4, 5));
        Set<Integer> setB = new HashSet<>(Set.of(4, 5, 6, 7, 8));

        // Union (ยูเนียน): รวมสมาชิกทั้งหมดจากทั้งสองเซต
        Set<Integer> union = new HashSet<>(setA);
        union.addAll(setB);
        System.out.println("Union: " + union); // [1, 2, 3, 4, 5, 6, 7, 8]

        // Intersection (อินเตอร์เซกชัน): สมาชิกที่มีอยู่ในทั้งสองเซต
        Set<Integer> intersection = new HashSet<>(setA);
        intersection.retainAll(setB);
        System.out.println("Intersection: " + intersection); // [4, 5]

        // Difference (ผลต่าง): สมาชิกที่อยู่ใน setA แต่ไม่อยู่ใน setB
        Set<Integer> difference = new HashSet<>(setA);
        difference.removeAll(setB);
        System.out.println("Difference (A-B): " + difference); // [1, 2, 3]
    }
}
```

## 9. กรณีใช้งานจริงของ Set

1. **กำจัดค่าซ้ำ**: แปลง List เป็น Set แล้วกลับเป็น List (ทบทวนจาก Part 22)
2. **ตรวจสอบสมาชิกภาพเร็ว**: เช็คว่า user อยู่ใน blacklist หรือไม่ (`HashSet`)
3. **Tag/Category System**: เก็บ tag ของบทความที่ไม่ซ้ำกัน
4. **Leaderboard ที่เรียงลำดับอัตโนมัติ**: ใช้ `TreeSet` กับ `Comparator`
5. **การหาความแตกต่างระหว่างชุดข้อมูล**: เปรียบเทียบรายชื่อผู้ใช้ก่อน-หลังอัปเดต

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียนโปรแกรมหาคำที่ไม่ซ้ำกัน (unique words) จากประโยค โดยไม่สนใจตัวพิมพ์
ใหญ่-เล็ก

**เฉลย:**

```java
import java.util.LinkedHashSet;
import java.util.Set;

public class Exercise1 {
    public static void main(String[] args) {
        String sentence = "Java is fun and Java is powerful";
        Set<String> uniqueWords = new LinkedHashSet<>();
        for (String word : sentence.toLowerCase().split("\\s+")) {
            uniqueWords.add(word);
        }
        System.out.println(uniqueWords); // [java, is, fun, and, powerful]
    }
}
```

**2)** สร้าง class `Student` ที่ override `equals()`/`hashCode()` ตาม field
`studentId` แล้วทดสอบว่า `HashSet` มองว่านักเรียนที่มี ID เดียวกันคือคนเดียวกัน

**เฉลย:**

```java
import java.util.HashSet;
import java.util.Objects;
import java.util.Set;

public class Student {
    String studentId;
    String name;

    public Student(String studentId, String name) {
        this.studentId = studentId;
        this.name = name;
    }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (!(obj instanceof Student)) return false;
        return this.studentId.equals(((Student) obj).studentId);
    }

    @Override
    public int hashCode() {
        return Objects.hash(studentId);
    }
}

public class Exercise2 {
    public static void main(String[] args) {
        Set<Student> students = new HashSet<>();
        students.add(new Student("S001", "Alice"));
        students.add(new Student("S001", "Alice (คนละ object)"));
        System.out.println(students.size()); // 1
    }
}
```

**3)** ใช้ Set operations หาว่านักเรียนคนไหนสมัครทั้งชมรมดนตรีและชมรมกีฬา
(intersection) จาก 2 set ของชื่อนักเรียน

**เฉลย:**

```java
import java.util.HashSet;
import java.util.Set;

public class Exercise3 {
    public static void main(String[] args) {
        Set<String> musicClub = new HashSet<>(Set.of("Alice", "Bob", "Charlie"));
        Set<String> sportsClub = new HashSet<>(Set.of("Bob", "Charlie", "David"));

        Set<String> both = new HashSet<>(musicClub);
        both.retainAll(sportsClub);
        System.out.println("สมัครทั้งสองชมรม: " + both); // [Bob, Charlie]
    }
}
```

### สรุปเนื้อหา Part 23

- `Set` ไม่อนุญาตค่าซ้ำ — `add()` คืนค่า false ถ้าเพิ่มค่าที่มีอยู่แล้ว
- `HashSet` เร็วที่สุด (O(1)) แต่ไม่รักษาลำดับ, `LinkedHashSet` รักษาลำดับการเพิ่ม,
  `TreeSet` เรียงลำดับอัตโนมัติ (O(log n))
- `HashSet`/`HashMap` ต้องพึ่งพา `hashCode()` และ `equals()` ที่ override ถูกต้อง
  คู่กันเสมอ ไม่งั้นจะทำงานผิดพลาด
- `TreeSet` ต้องการ natural ordering (`Comparable`) หรือ `Comparator` เพื่อเรียง
  ลำดับ
- Set operations (union, intersection, difference) ทำผ่าน `addAll()`,
  `retainAll()`, `removeAll()`

**ต่อไป**: [Part 24 — Collections Framework: Map](./part-024-collections-map.md)
