# Part 41: Stream API เบื้องต้น: filter, map, reduce, collect

> ขั้นตอนที่ 401-410 ของหลักสูตร | ระดับ: กลาง-สูง

## สารบัญ

1. Stream คืออะไร ต่างจาก Collection อย่างไร
2. การสร้าง Stream
3. Intermediate Operations: `filter`, `map`, `sorted`, `distinct`
4. Terminal Operations: `forEach`, `count`, `collect`
5. `reduce()`: รวมค่าทั้งหมดเป็นค่าเดียว
6. `Optional` จาก Stream: `findFirst`, `findAny`, `min`, `max`
7. Lazy Evaluation: Stream ทำงานตอนไหนจริง ๆ
8. Stream Pipeline: การเชื่อมต่อหลาย Operation
9. ข้อจำกัดของ Stream: ใช้ได้ครั้งเดียว
10. แบบฝึกหัดและสรุป

---

## 1. Stream คืออะไร ต่างจาก Collection อย่างไร

**Stream API** (Java 8+) คือวิธีประมวลผล collection ของข้อมูลแบบ **functional
style** — ประกาศ**"จะทำอะไร"** (declarative) มากกว่า**"ทำอย่างไรทีละขั้น"**
(imperative) — ต่อยอดจากทุกอย่างที่เรียนใน Part 39-40 (Lambda, Functional
Interfaces)

```java
import java.util.List;
import java.util.ArrayList;

public class ImperativeVsDeclarativeDemo {
    public static void main(String[] args) {
        List<Integer> numbers = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

        // แบบ Imperative (แบบเดิม): บอกทุกขั้นตอนว่าต้องทำอะไร
        List<Integer> evenSquaresOld = new ArrayList<>();
        for (int n : numbers) {
            if (n % 2 == 0) {
                evenSquaresOld.add(n * n);
            }
        }
        System.out.println(evenSquaresOld); // [4, 16, 36, 64, 100]

        // แบบ Declarative (Stream): บอกแค่ "ต้องการอะไร" อ่านเหมือนประโยคภาษาอังกฤษ
        List<Integer> evenSquaresNew = numbers.stream()
                .filter(n -> n % 2 == 0)     // เอาแค่เลขคู่
                .map(n -> n * n)                // ยกกำลังสองแต่ละตัว
                .collect(java.util.stream.Collectors.toList()); // รวมเป็น List

        System.out.println(evenSquaresNew); // [4, 16, 36, 64, 100]
    }
}
```

**ข้อสำคัญ**: **Stream ไม่ใช่โครงสร้างข้อมูล** (ไม่เก็บข้อมูลเอง) เป็นเพียง
**"สายท่อประมวลผล"** ที่ดึงข้อมูลจาก source (List, Set, Array, ไฟล์ — ทบทวน
`Files.lines()` จาก Part 37) แล้วส่งผ่าน operation ต่าง ๆ ที่กำหนดไว้

## 2. การสร้าง Stream

```java
import java.util.Arrays;
import java.util.List;
import java.util.stream.Stream;
import java.util.stream.IntStream;

public class StreamCreationDemo {
    public static void main(String[] args) {
        // จาก Collection
        List<String> list = List.of("a", "b", "c");
        Stream<String> streamFromList = list.stream();

        // จาก Array
        String[] array = {"x", "y", "z"};
        Stream<String> streamFromArray = Arrays.stream(array);

        // จากค่าตรง ๆ ด้วย Stream.of()
        Stream<Integer> streamOfValues = Stream.of(1, 2, 3, 4, 5);

        // Stream ตัวเลขแบบต่อเนื่อง (primitive specialization - ทบทวนแนวคิดจาก Part 40)
        IntStream range = IntStream.range(1, 5);       // 1, 2, 3, 4 (ไม่รวม 5)
        IntStream rangeClosed = IntStream.rangeClosed(1, 5); // 1, 2, 3, 4, 5 (รวม 5)

        // Stream ว่าง (ใช้บ่อยเป็นค่า default)
        Stream<String> emptyStream = Stream.empty();

        // Stream infinite (ไม่มีที่สิ้นสุด - ต้องจำกัดด้วย limit() เสมอ)
        Stream<Integer> infiniteStream = Stream.iterate(1, n -> n * 2).limit(5);
        System.out.println(infiniteStream.toList()); // [1, 2, 4, 8, 16]
    }
}
```

## 3. Intermediate Operations: `filter`, `map`, `sorted`, `distinct`

**Intermediate Operation** คือ operation ที่**คืนค่า Stream ใหม่เสมอ**
ทำให้ต่อ operation อื่นได้ (chaining) — **ไม่ประมวลผลทันที** (ทบทวน lazy
evaluation ในหัวข้อ 7)

```java
import java.util.List;
import java.util.stream.Collectors;

public class IntermediateOperationsDemo {
    public static void main(String[] args) {
        List<String> names = List.of("Charlie", "alice", "Bob", "alice", "David");

        // filter: เลือกเฉพาะที่ตรงเงื่อนไข (รับ Predicate - ทบทวนจาก Part 40)
        List<String> longNames = names.stream()
                .filter(name -> name.length() > 4)
                .collect(Collectors.toList());
        System.out.println(longNames); // [Charlie, alice, David]

        // map: แปลงข้อมูลแต่ละตัว (รับ Function)
        List<Integer> lengths = names.stream()
                .map(String::length)
                .collect(Collectors.toList());
        System.out.println(lengths); // [7, 5, 3, 5, 5]

        // sorted: เรียงลำดับ (natural ordering หรือ Comparator - ทบทวนจาก Part 27)
        List<String> sortedNames = names.stream()
                .sorted()
                .collect(Collectors.toList());
        System.out.println(sortedNames); // [Bob, Charlie, David, alice, alice]

        // distinct: กำจัดค่าซ้ำ (ใช้ equals() ภายใน - ทบทวนจาก Part 23)
        List<String> uniqueNames = names.stream()
                .distinct()
                .collect(Collectors.toList());
        System.out.println(uniqueNames); // [Charlie, alice, Bob, David]

        // ต่อกันหลาย operation (chaining) - นี่คือพลังของ Stream
        List<String> result = names.stream()
                .distinct()
                .filter(name -> name.length() >= 4)
                .map(String::toUpperCase)
                .sorted()
                .collect(Collectors.toList());
        System.out.println(result); // [ALICE, CHARLIE, DAVID]

        // limit / skip: จำกัดจำนวน / ข้าม (ใช้บ่อยกับ pagination)
        List<Integer> numbers = java.util.stream.IntStream.rangeClosed(1, 10).boxed().toList();
        System.out.println(numbers.stream().skip(3).limit(4).toList()); // [4, 5, 6, 7]
    }
}
```

## 4. Terminal Operations: `forEach`, `count`, `collect`

**Terminal Operation** คือ operation ที่**"จุดชนวน"**ให้ stream ประมวลผลจริง
และ**คืนค่าที่ไม่ใช่ Stream**อีกต่อไป (เช่น `List`, `long`, `void`) — หลังเรียก
terminal operation แล้ว **stream นั้นจะใช้งานต่อไม่ได้อีก** (หัวข้อ 9)

```java
import java.util.List;
import java.util.stream.Collectors;

public class TerminalOperationsDemo {
    public static void main(String[] args) {
        List<Integer> numbers = List.of(1, 2, 3, 4, 5);

        // forEach: ทำงานกับแต่ละตัว ไม่คืนค่า (รับ Consumer - ทบทวนจาก Part 40)
        numbers.stream().forEach(System.out::println);

        // count: นับจำนวน element
        long evenCount = numbers.stream().filter(n -> n % 2 == 0).count();
        System.out.println("จำนวนเลขคู่: " + evenCount); // 2

        // collect: รวมผลลัพธ์เป็น collection (List, Set, Map - จะลงลึกใน Part 42)
        List<Integer> squared = numbers.stream()
                .map(n -> n * n)
                .collect(Collectors.toList());
        System.out.println(squared);

        // toList(): ทางลัดของ collect(Collectors.toList()) (Java 16+)
        List<Integer> squaredShortcut = numbers.stream().map(n -> n * n).toList();
        System.out.println(squaredShortcut);

        // anyMatch / allMatch / noneMatch: ตรวจสอบเงื่อนไข คืนค่า boolean
        System.out.println(numbers.stream().anyMatch(n -> n > 4));  // true (มีตัวใดตัวหนึ่ง > 4)
        System.out.println(numbers.stream().allMatch(n -> n > 0));   // true (ทุกตัว > 0)
        System.out.println(numbers.stream().noneMatch(n -> n > 10)); // true (ไม่มีตัวไหน > 10)
    }
}
```

## 5. `reduce()`: รวมค่าทั้งหมดเป็นค่าเดียว

**`reduce()`** ใช้**รวมค่าทุกตัวใน stream เป็นผลลัพธ์เดียว** โดยกำหนด**ค่า
เริ่มต้น (identity)** และ**วิธีรวม (BinaryOperator — ทบทวนจาก Part 40)**

```java
import java.util.List;

public class ReduceDemo {
    public static void main(String[] args) {
        List<Integer> numbers = List.of(1, 2, 3, 4, 5);

        // reduce แบบมี identity: ผลรวมทั้งหมด
        int sum = numbers.stream().reduce(0, (a, b) -> a + b);
        System.out.println(sum); // 15

        // ใช้ method reference แทน lambda ก็ได้ (Integer::sum)
        int sum2 = numbers.stream().reduce(0, Integer::sum);
        System.out.println(sum2); // 15

        // หาผลคูณทั้งหมด (identity = 1 เพราะ x * 1 = x)
        int product = numbers.stream().reduce(1, (a, b) -> a * b);
        System.out.println(product); // 120

        // reduce แบบไม่มี identity: คืนค่าเป็น Optional (เพราะ stream อาจว่างเปล่า - ทบทวน Part 43)
        java.util.Optional<Integer> max = numbers.stream().reduce((a, b) -> a > b ? a : b);
        System.out.println(max.orElse(0)); // 5

        // หาข้อความที่ยาวที่สุดด้วย reduce
        List<String> words = List.of("cat", "elephant", "dog", "butterfly");
        String longest = words.stream()
                .reduce("", (a, b) -> a.length() >= b.length() ? a : b);
        System.out.println(longest); // "butterfly"
    }
}
```

**ทำไมต้องมี identity**: ทำหน้าที่เป็นค่าเริ่มต้นของการรวม และเป็นค่าที่คืนกลับ
ถ้า stream ว่างเปล่า (เช่น `sum` ของ stream ว่างควรเป็น `0` ไม่ใช่ error)

## 6. `Optional` จาก Stream: `findFirst`, `findAny`, `min`, `max`

จะลงลึกเรื่อง `Optional` เต็มรูปแบบใน Part 43 แต่ต้องรู้เบื้องต้นตอนนี้ เพราะ
หลาย terminal operation ของ Stream คืนค่าเป็น `Optional`:

```java
import java.util.List;
import java.util.Optional;
import java.util.Comparator;

public class StreamOptionalDemo {
    public static void main(String[] args) {
        List<Integer> numbers = List.of(5, 3, 8, 1, 9);

        Optional<Integer> first = numbers.stream().filter(n -> n > 5).findFirst();
        System.out.println(first.orElse(-1)); // 8 (ตัวแรกที่มากกว่า 5)

        Optional<Integer> max = numbers.stream().max(Comparator.naturalOrder());
        System.out.println(max.orElse(-1)); // 9

        Optional<Integer> min = numbers.stream().min(Comparator.naturalOrder());
        System.out.println(min.orElse(-1)); // 1

        Optional<Integer> notFound = numbers.stream().filter(n -> n > 100).findFirst();
        System.out.println(notFound.isPresent()); // false (Optional ว่างเปล่า - ปลอดภัยกว่า null มาก)
    }
}
```

## 7. Lazy Evaluation: Stream ทำงานตอนไหนจริง ๆ

**Intermediate operations เป็น lazy** — ไม่ประมวลผลจนกว่าจะมี **terminal
operation** เรียกใช้ นี่คือแนวคิดสำคัญที่ทำให้ Stream มีประสิทธิภาพ

```java
import java.util.List;

public class LazyEvaluationDemo {
    public static void main(String[] args) {
        List<Integer> numbers = List.of(1, 2, 3, 4, 5);

        System.out.println("สร้าง stream pipeline...");
        var stream = numbers.stream()
                .filter(n -> {
                    System.out.println("กรอง: " + n);
                    return n % 2 == 0;
                })
                .map(n -> {
                    System.out.println("แปลง: " + n);
                    return n * n;
                });

        System.out.println("ยังไม่มีอะไรพิมพ์ออกมาเลย! (lazy)");
        System.out.println("เรียก terminal operation...");

        List<Integer> result = stream.collect(java.util.stream.Collectors.toList());
        // ตอนนี้เท่านั้นที่ "กรอง:" และ "แปลง:" จะถูกพิมพ์ออกมา

        System.out.println(result);
    }
}
```

**ประโยชน์ของ lazy evaluation**: Stream ประมวลผล**ทีละ element แบบครบวงจร**
(ผ่านทุก operation ก่อนไปตัวถัดไป) ไม่ใช่ทำ filter ทั้งหมดก่อนแล้วค่อยทำ map
ทั้งหมด — ทำให้สามารถ**หยุดกลางทางได้** (เช่นด้วย `findFirst()`, `limit()`)
โดยไม่ต้องประมวลผลข้อมูลที่ไม่จำเป็นเลย ซึ่งสำคัญมากกับ infinite stream

## 8. Stream Pipeline: การเชื่อมต่อหลาย Operation

```
Source (List/Set/Array)
    |
    v
stream()
    |
    v
Intermediate Operation 1 (filter)  ─┐
    |                                │
    v                                │  Lazy - ยังไม่ทำงานจริง
Intermediate Operation 2 (map)     ─┤  จนกว่าจะเจอ terminal operation
    |                                │
    v                                │
Intermediate Operation 3 (sorted)  ─┘
    |
    v
Terminal Operation (collect/forEach/reduce) <- จุดที่ทุกอย่างเริ่มทำงานจริง
    |
    v
ผลลัพธ์ (List, long, void, ...)
```

## 9. ข้อจำกัดของ Stream: ใช้ได้ครั้งเดียว

**Stream ใช้ซ้ำไม่ได้!** — เมื่อเรียก terminal operation ไปแล้ว stream นั้นจะ
"ถูกใช้ไปแล้ว" (consumed) เรียกใช้ซ้ำจะได้ `IllegalStateException`

```java
import java.util.List;
import java.util.stream.Stream;

public class StreamReuseDemo {
    public static void main(String[] args) {
        List<Integer> numbers = List.of(1, 2, 3);
        Stream<Integer> stream = numbers.stream();

        long count = stream.count(); // terminal operation ครั้งแรก - ใช้ได้

        try {
            long sum = stream.mapToInt(Integer::intValue).sum(); // Error! stream ถูกใช้ไปแล้ว
        } catch (IllegalStateException e) {
            System.out.println("Error: stream has already been operated upon or closed");
        }

        // วิธีที่ถูกต้อง: สร้าง stream ใหม่ทุกครั้งที่ต้องการประมวลผลใหม่
        long sumCorrect = numbers.stream().mapToInt(Integer::intValue).sum();
        System.out.println(sumCorrect); // 6
    }
}
```

**บทเรียนสำคัญ**: เขียน stream pipeline ให้จบในคำสั่งเดียว (method chaining)
หรือสร้าง stream ใหม่จาก source เดิม (`list.stream()`) ทุกครั้งที่ต้องการ
ประมวลผลซ้ำ — Collection (เช่น `List`) ใช้ซ้ำได้ตามปกติ แต่ Stream object
ที่สร้างจากมันใช้ได้ครั้งเดียวเท่านั้น

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** ใช้ Stream หาผลรวมของเลขคู่ทั้งหมดใน `List<Integer>`

**เฉลย:**

```java
import java.util.List;

public class Exercise1 {
    public static void main(String[] args) {
        List<Integer> numbers = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
        int sumOfEvens = numbers.stream()
                .filter(n -> n % 2 == 0)
                .reduce(0, Integer::sum);
        System.out.println(sumOfEvens); // 30
    }
}
```

**2)** ใช้ Stream แปลง `List<String>` ของชื่อให้เป็นตัวพิมพ์ใหญ่ เรียงลำดับ
ตามความยาว (สั้นไปยาว) แล้วเก็บใส่ `List<String>` ใหม่

**เฉลย:**

```java
import java.util.List;
import java.util.Comparator;

public class Exercise2 {
    public static void main(String[] args) {
        List<String> names = List.of("Bob", "Alice", "Jo", "Charlie");
        List<String> result = names.stream()
                .map(String::toUpperCase)
                .sorted(Comparator.comparingInt(String::length))
                .toList();
        System.out.println(result); // [JO, BOB, ALICE, CHARLIE]
    }
}
```

**3)** อธิบายว่าทำไม lazy evaluation ทำให้โค้ดนี้มีประสิทธิภาพดี แม้ list มี
ข้อมูลเป็นล้านตัว:

```java
Optional<Integer> result = hugeList.stream()
    .filter(n -> n > 1000)
    .map(n -> n * n)
    .findFirst();
```

**เฉลย**: เพราะ lazy evaluation ทำให้ stream ประมวลผล**ทีละ element แบบครบ
วงจร** (filter แล้ว map ทันทีสำหรับ element นั้น) และ `findFirst()` จะ**หยุด
ทันทีที่เจอผลลัพธ์แรก** — ไม่ต้องประมวลผล element ที่เหลือทั้งหมดในลิสต์เลย
ถ้าเจอคำตอบตั้งแต่ตัวที่ 100 จาก 1 ล้านตัว โปรแกรมจะหยุดทำงานทันทีโดยไม่ไป
แตะ element ที่เหลืออีก 999,900 ตัวเลย ต่างจากการเขียนแบบ imperative ที่มักจะ
ประมวลผลทุกตัวก่อนจึงค่อยกรองหาคำตอบ

### สรุปเนื้อหา Part 41

- Stream ประมวลผลข้อมูลแบบ declarative (บอกว่าทำอะไร ไม่ใช่ทำอย่างไร) ไม่ใช่
  โครงสร้างข้อมูลที่เก็บค่าเอง
- Intermediate operations (`filter`, `map`, `sorted`, `distinct`) คืนค่า
  Stream ใหม่ ต่อกันได้ (chaining) และเป็น lazy
- Terminal operations (`forEach`, `count`, `collect`, `reduce`) จุดชนวนให้
  ประมวลผลจริง และทำให้ stream นั้นใช้ต่อไม่ได้อีก
- `reduce()` รวมค่าทั้งหมดเป็นค่าเดียวด้วย identity + BinaryOperator
- Lazy evaluation ทำให้ stream หยุดกลางทางได้ (เช่น `findFirst`) โดยไม่ต้อง
  ประมวลผลข้อมูลทั้งหมด

**ต่อไป**: [Part 42 — Stream API ขั้นสูง: Collectors, groupingBy, parallel streams](./part-042-stream-api-advanced.md)
