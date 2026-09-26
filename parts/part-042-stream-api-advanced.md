# Part 42: Stream API ขั้นสูง: Collectors, groupingBy, Parallel Streams

> ขั้นตอนที่ 411-420 ของหลักสูตร | ระดับ: สูง

## สารบัญ

1. `Collectors` Class: ภาพรวม
2. `Collectors.toMap()`
3. `Collectors.groupingBy()`: จัดกลุ่มข้อมูล
4. `Collectors.partitioningBy()`: แบ่งเป็นสองกลุ่ม
5. `Collectors.joining()`: รวม String
6. Collectors สำหรับสถิติ: `summarizingInt`, `averagingDouble`
7. `Collectors.mapping()` และ Downstream Collectors
8. `flatMap()`: แปลง Stream ซ้อน Stream ให้แบนราบ
9. Parallel Streams: ประมวลผลแบบขนาน
10. แบบฝึกหัดและสรุป

---

## 1. `Collectors` Class: ภาพรวม

`Collectors` เป็น utility class ที่รวม**เมธอด factory** สำหรับสร้าง
`Collector` ที่ใช้กับ `Stream.collect()` (ทบทวนจาก Part 41) — เป็นเครื่องมือ
ที่ทรงพลังที่สุดสำหรับการ**แปลง stream ให้เป็นโครงสร้างข้อมูลที่ซับซ้อน**

```java
import java.util.List;
import java.util.Set;
import java.util.stream.Collectors;

public class CollectorsOverviewDemo {
    public static void main(String[] args) {
        List<String> names = List.of("Alice", "Bob", "Charlie");

        List<String> asList = names.stream().collect(Collectors.toList());
        Set<String> asSet = names.stream().collect(Collectors.toSet());
        String asString = names.stream().collect(Collectors.joining(", "));

        System.out.println(asList);
        System.out.println(asSet);
        System.out.println(asString); // "Alice, Bob, Charlie"
    }
}
```

## 2. `Collectors.toMap()`

แปลง stream เป็น `Map<K, V>` โดยระบุวิธีดึง key และ value จากแต่ละ element

```java
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

public class ToMapDemo {
    record Employee(String name, double salary) { }

    public static void main(String[] args) {
        List<Employee> employees = List.of(
            new Employee("Alice", 35000),
            new Employee("Bob", 28000),
            new Employee("Charlie", 42000)
        );

        Map<String, Double> nameToSalary = employees.stream()
                .collect(Collectors.toMap(Employee::name, Employee::salary));
        System.out.println(nameToSalary); // {Bob=28000.0, Alice=35000.0, Charlie=42000.0}

        // ถ้ามี key ซ้ำกัน (เช่น สอง Employee ชื่อเดียวกัน) ต้องระบุ merge function ตัวที่ 3
        // ไม่งั้นจะเกิด IllegalStateException: Duplicate key
        List<Employee> withDuplicates = List.of(
            new Employee("Alice", 35000),
            new Employee("Alice", 40000) // ชื่อซ้ำ!
        );
        Map<String, Double> merged = withDuplicates.stream()
                .collect(Collectors.toMap(Employee::name, Employee::salary,
                                           (existing, replacement) -> existing + replacement)); // รวมค่าที่ซ้ำกัน
        System.out.println(merged); // {Alice=75000.0}
    }
}
```

## 3. `Collectors.groupingBy()`: จัดกลุ่มข้อมูล

**`groupingBy()`** คือ Collector ที่ใช้บ่อยที่สุดในโค้ดจริง — จัดกลุ่ม element
ตาม key ที่กำหนด ได้ผลลัพธ์เป็น `Map<K, List<T>>` (เทียบเท่ากับ `GROUP BY`
ใน SQL — ปูทางสู่ Part 64-65)

```java
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

public class GroupingByDemo {
    record Student(String name, String major, int score) { }

    public static void main(String[] args) {
        List<Student> students = List.of(
            new Student("Alice", "CS", 85),
            new Student("Bob", "Math", 90),
            new Student("Charlie", "CS", 78),
            new Student("David", "Math", 88),
            new Student("Eve", "CS", 92)
        );

        // จัดกลุ่มตามสาขา
        Map<String, List<Student>> byMajor = students.stream()
                .collect(Collectors.groupingBy(Student::major));
        byMajor.forEach((major, list) -> {
            System.out.println(major + ": " + list.size() + " คน");
        });

        // จัดกลุ่มแล้วนับจำนวน (downstream collector - หัวข้อ 7)
        Map<String, Long> countByMajor = students.stream()
                .collect(Collectors.groupingBy(Student::major, Collectors.counting()));
        System.out.println(countByMajor); // {CS=3, Math=2}

        // จัดกลุ่มแล้วหาคะแนนเฉลี่ยของแต่ละกลุ่ม
        Map<String, Double> avgScoreByMajor = students.stream()
                .collect(Collectors.groupingBy(Student::major, Collectors.averagingInt(Student::score)));
        System.out.println(avgScoreByMajor); // {CS=85.0, Math=89.0}

        // จัดกลุ่มหลายชั้น (nested grouping)
        Map<String, Map<Boolean, List<Student>>> byMajorThenPassing = students.stream()
                .collect(Collectors.groupingBy(Student::major,
                         Collectors.groupingBy(s -> s.score() >= 85)));
        System.out.println(byMajorThenPassing);
    }
}
```

## 4. `Collectors.partitioningBy()`: แบ่งเป็นสองกลุ่ม

**`partitioningBy()`** คล้าย `groupingBy()` แต่ใช้เมื่อต้องการแบ่งเป็น**แค่
สองกลุ่มตาม boolean predicate** (`true`/`false`) — ผลลัพธ์เป็น
`Map<Boolean, List<T>>` เสมอ (ต่างจาก `groupingBy` ที่ key เป็นอะไรก็ได้)

```java
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

public class PartitioningByDemo {
    public static void main(String[] args) {
        List<Integer> numbers = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

        Map<Boolean, List<Integer>> partitioned = numbers.stream()
                .collect(Collectors.partitioningBy(n -> n % 2 == 0));

        System.out.println("เลขคู่: " + partitioned.get(true));   // [2, 4, 6, 8, 10]
        System.out.println("เลขคี่: " + partitioned.get(false));   // [1, 3, 5, 7, 9]
    }
}
```

## 5. `Collectors.joining()`: รวม String

```java
import java.util.List;
import java.util.stream.Collectors;

public class JoiningDemo {
    public static void main(String[] args) {
        List<String> names = List.of("Alice", "Bob", "Charlie");

        String simple = names.stream().collect(Collectors.joining());
        System.out.println(simple); // "AliceBobCharlie"

        String withDelimiter = names.stream().collect(Collectors.joining(", "));
        System.out.println(withDelimiter); // "Alice, Bob, Charlie"

        String withPrefixSuffix = names.stream()
                .collect(Collectors.joining(", ", "[", "]"));
        System.out.println(withPrefixSuffix); // "[Alice, Bob, Charlie]"
    }
}
```

## 6. Collectors สำหรับสถิติ: `summarizingInt`, `averagingDouble`

```java
import java.util.IntSummaryStatistics;
import java.util.List;
import java.util.stream.Collectors;

public class StatisticsCollectorsDemo {
    record Product(String name, int price) { }

    public static void main(String[] args) {
        List<Product> products = List.of(
            new Product("Laptop", 25000),
            new Product("Mouse", 500),
            new Product("Keyboard", 1200)
        );

        // summarizingInt: สถิติครบชุดในครั้งเดียว (count, sum, min, max, average)
        IntSummaryStatistics stats = products.stream()
                .collect(Collectors.summarizingInt(Product::price));

        System.out.println("จำนวน: " + stats.getCount());
        System.out.println("รวม: " + stats.getSum());
        System.out.println("เฉลี่ย: " + stats.getAverage());
        System.out.println("มากสุด: " + stats.getMax());
        System.out.println("น้อยสุด: " + stats.getMin());

        // หรือใช้ IntStream โดยตรง (เทียบเท่ากันแต่ไม่ต้องใช้ Collectors)
        IntSummaryStatistics stats2 = products.stream()
                .mapToInt(Product::price)
                .summaryStatistics();
        System.out.println(stats2.getSum());
    }
}
```

## 7. `Collectors.mapping()` และ Downstream Collectors

**Downstream Collector** คือ collector ที่ใช้**ต่อจาก `groupingBy()`** เพื่อ
ประมวลผลข้อมูลในแต่ละกลุ่มต่อ (เราเห็นตัวอย่างแล้วในหัวข้อ 3 — `counting()`,
`averagingInt()`) — `mapping()` ใช้แปลงข้อมูลก่อนรวมกลุ่ม

```java
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.stream.Collectors;

public class DownstreamCollectorsDemo {
    record Student(String name, String major) { }

    public static void main(String[] args) {
        List<Student> students = List.of(
            new Student("Alice", "CS"),
            new Student("Bob", "Math"),
            new Student("Charlie", "CS"),
            new Student("David", "Math")
        );

        // จัดกลุ่มตามสาขา แล้วเอาแค่ "ชื่อ" ของแต่ละคนในกลุ่ม (ไม่เอา Student object ทั้งตัว)
        Map<String, List<String>> namesByMajor = students.stream()
                .collect(Collectors.groupingBy(Student::major,
                         Collectors.mapping(Student::name, Collectors.toList())));
        System.out.println(namesByMajor); // {CS=[Alice, Charlie], Math=[Bob, David]}

        // จัดกลุ่มแล้วรวมเป็น Set แทน List (กำจัดค่าซ้ำในแต่ละกลุ่ม)
        Map<String, Set<String>> namesSetByMajor = students.stream()
                .collect(Collectors.groupingBy(Student::major,
                         Collectors.mapping(Student::name, Collectors.toSet())));
        System.out.println(namesSetByMajor);

        // joining เป็น downstream collector
        Map<String, String> joinedNamesByMajor = students.stream()
                .collect(Collectors.groupingBy(Student::major,
                         Collectors.mapping(Student::name, Collectors.joining(", "))));
        System.out.println(joinedNamesByMajor); // {CS=Alice, Charlie, Math=Bob, David}
    }
}
```

## 8. `flatMap()`: แปลง Stream ซ้อน Stream ให้แบนราบ

**`flatMap()`** ใช้เมื่อ `map()` จะได้ **"stream ของ stream"** (nested
structure) — `flatMap()` **รวมทุก sub-stream เข้าเป็น stream เดียว (flatten)**

```java
import java.util.List;
import java.util.stream.Collectors;

public class FlatMapDemo {
    public static void main(String[] args) {
        List<List<Integer>> nestedList = List.of(
            List.of(1, 2, 3),
            List.of(4, 5),
            List.of(6, 7, 8, 9)
        );

        // map() ธรรมดา: ได้ Stream<Stream<Integer>> ซึ่งใช้งานยาก
        // flatMap(): แปลงเป็น Stream<Integer> เดียวที่แบนราบ
        List<Integer> flatList = nestedList.stream()
                .flatMap(List::stream) // แปลงแต่ละ List<Integer> เป็น Stream<Integer> แล้วรวมกัน
                .collect(Collectors.toList());
        System.out.println(flatList); // [1, 2, 3, 4, 5, 6, 7, 8, 9]

        // ตัวอย่างการใช้งานจริง: แยกคำจากหลายประโยคออกมาเป็น word เดียว ๆ
        List<String> sentences = List.of("Java is fun", "Streams are powerful");
        List<String> allWords = sentences.stream()
                .flatMap(sentence -> java.util.Arrays.stream(sentence.split(" ")))
                .collect(Collectors.toList());
        System.out.println(allWords); // [Java, is, fun, Streams, are, powerful]

        // flatMap กับ Optional (ทบทวนแนวคิดจาก Part 43 ที่จะเรียนต่อไป)
        record Person(String name, java.util.Optional<String> nickname) { }
    }
}
```

## 9. Parallel Streams: ประมวลผลแบบขนาน

**Parallel Stream** ใช้**หลาย thread ประมวลผล stream พร้อมกัน** เพื่อเพิ่ม
ความเร็วสำหรับข้อมูลขนาดใหญ่บน CPU หลาย core (ปูทางสู่ Part 46-48 เรื่อง
Multithreading เต็มรูปแบบ)

```java
import java.util.List;
import java.util.stream.IntStream;

public class ParallelStreamDemo {
    public static void main(String[] args) {
        // Sequential stream: ใช้ thread เดียว
        long startSeq = System.currentTimeMillis();
        long sumSeq = IntStream.rangeClosed(1, 100_000_000).sum();
        System.out.println("Sequential: " + (System.currentTimeMillis() - startSeq) + " ms");

        // Parallel stream: แบ่งงานให้หลาย thread ทำพร้อมกัน (ใช้ ForkJoinPool ภายใน)
        long startPar = System.currentTimeMillis();
        long sumPar = IntStream.rangeClosed(1, 100_000_000).parallel().sum();
        System.out.println("Parallel: " + (System.currentTimeMillis() - startPar) + " ms");

        System.out.println(sumSeq == sumPar); // true (ผลลัพธ์เหมือนกัน)
    }
}
```

**ข้อควรระวังสำคัญของ Parallel Stream**:
1. **มี overhead ในการแบ่งงานและรวมผลลัพธ์** — เหมาะกับข้อมูล**ขนาดใหญ่มาก**
   และการคำนวณที่**ซับซ้อนต่อ element** เท่านั้น ถ้าข้อมูลน้อยหรือ operation
   ง่ายมาก parallel อาจ**ช้ากว่า** sequential เพราะ overhead
2. **ต้องระมัดระวัง shared mutable state** — ถ้า lambda ใน parallel stream
   แก้ไขตัวแปรร่วมกัน (เช่น `ArrayList` ที่ไม่ thread-safe) จะเกิดปัญหา race
   condition (จะลงลึกเรื่อง thread-safety ใน Part 46-50)

```java
import java.util.ArrayList;
import java.util.List;
import java.util.stream.IntStream;

public class ParallelStreamPitfallDemo {
    public static void main(String[] args) {
        List<Integer> unsafeList = new ArrayList<>(); // ไม่ thread-safe!

        // อันตราย: หลาย thread เขียนเข้า ArrayList เดียวกันพร้อมกัน อาจเกิดข้อมูลเสียหาย
        // IntStream.range(0, 10000).parallel().forEach(unsafeList::add); // ห้ามทำแบบนี้!

        // วิธีที่ถูกต้อง: ใช้ collect() ที่ Stream API จัดการ thread-safety ให้เอง
        List<Integer> safeList = IntStream.range(0, 10000)
                .parallel()
                .boxed()
                .collect(java.util.stream.Collectors.toList()); // ปลอดภัย - Stream API จัดการให้
        System.out.println(safeList.size()); // 10000
    }
}
```

**หลักปฏิบัติ**: ใช้ **sequential stream เป็นค่าเริ่มต้นเสมอ** เปลี่ยนเป็น
parallel เฉพาะเมื่อวัดผล (benchmark) แล้วพบว่าช่วยจริง และมั่นใจว่าไม่มี
shared mutable state ที่ไม่ thread-safe

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** ใช้ `groupingBy` + `counting` นับจำนวนคำแต่ละคำในประโยค (word frequency)

**เฉลย:**

```java
import java.util.Arrays;
import java.util.Map;
import java.util.stream.Collectors;

public class Exercise1 {
    public static void main(String[] args) {
        String text = "the quick brown fox the lazy dog the fox";
        Map<String, Long> wordCount = Arrays.stream(text.split(" "))
                .collect(Collectors.groupingBy(w -> w, Collectors.counting()));
        System.out.println(wordCount); // {the=3, quick=1, fox=2, brown=1, lazy=1, dog=1}
    }
}
```

**2)** ใช้ `flatMap` รวมรายชื่อนักเรียนจากหลายห้องเรียน (List ของ List) เป็น
รายชื่อเดียว แล้วเรียงตามตัวอักษร

**เฉลย:**

```java
import java.util.List;

public class Exercise2 {
    public static void main(String[] args) {
        List<List<String>> classrooms = List.of(
            List.of("Charlie", "Alice"),
            List.of("Bob", "David")
        );
        List<String> allSorted = classrooms.stream()
                .flatMap(List::stream)
                .sorted()
                .toList();
        System.out.println(allSorted); // [Alice, Bob, Charlie, David]
    }
}
```

**3)** อธิบายว่าเมื่อไรควรใช้ parallel stream และเมื่อไรไม่ควร

**เฉลย**: ควรใช้ parallel stream เมื่อ (1) มีข้อมูลจำนวนมาก (หลักหมื่นถึงล้าน
element ขึ้นไป) (2) การคำนวณต่อ element มี cost สูงพอที่จะคุ้มกับ overhead
การแบ่งงาน (3) เครื่องมี CPU หลาย core จริง และ (4) operation ไม่มี shared
mutable state ที่ไม่ thread-safe — ไม่ควรใช้เมื่อข้อมูลมีขนาดเล็ก (overhead
มากกว่าประโยชน์), operation ง่ายมาก (เช่นแค่บวกเลข), หรือมีการแก้ไขตัวแปรร่วม
ที่ไม่ปลอดภัยต่อ concurrent access — ควรวัดผลจริง (benchmark) เสมอก่อนตัดสินใจ
เปลี่ยนเป็น parallel ในโค้ด production

### สรุปเนื้อหา Part 42

- `Collectors.toMap()` แปลง stream เป็น Map, ต้องระบุ merge function ถ้ามี key
  ซ้ำ
- `groupingBy()` จัดกลุ่มข้อมูลตาม key (เทียบเท่า SQL GROUP BY), ใช้ร่วมกับ
  downstream collector (`counting`, `averagingInt`, `mapping`) เพื่อประมวลผล
  ในแต่ละกลุ่มต่อ
- `partitioningBy()` แบ่งเป็น 2 กลุ่มตาม boolean predicate เท่านั้น
- `flatMap()` แปลง nested stream ให้แบนราบเป็น stream เดียว
- Parallel stream ใช้หลาย thread ประมวลผล แต่มี overhead — ใช้เมื่อข้อมูลใหญ่
  มากและไม่มี shared mutable state ที่ไม่ปลอดภัย

**ต่อไป**: [Part 43 — Optional Class และการเขียนโค้ด Null-safe](./part-043-optional.md)
