# Part 24: Collections Framework: Map (HashMap, TreeMap, LinkedHashMap)

> ขั้นตอนที่ 231-240 ของหลักสูตร | ระดับ: กลาง

## สารบัญ

1. `Map` Interface คืออะไร
2. `HashMap` แบบละเอียด
3. `LinkedHashMap` และ `TreeMap`
4. เมธอดสำคัญของ Map (รวมเมธอดสมัยใหม่ Java 8+)
5. การวนซ้ำ Map ด้วยวิธีต่าง ๆ
6. Map เป็น Key ที่เป็น Object เอง (ทบทวน hashCode/equals)
7. `HashMap` ทำงานอย่างไรภายใน (โครงสร้างข้อมูล)
8. Nested Map และโครงสร้างข้อมูลซับซ้อน
9. `Map.Entry` และการประมวลผลแบบ Stream
10. แบบฝึกหัดและสรุป

---

## 1. `Map` Interface คืออะไร

**`Map<K, V>`** คือ collection ที่เก็บข้อมูลเป็น**คู่ key-value** โดย**key ต้อง
ไม่ซ้ำกัน** (เหมือน Set) แต่ **value ซ้ำกันได้** — เป็นโครงสร้างข้อมูลที่ใช้บ่อย
ที่สุดข้อหนึ่งในโปรแกรม Java จริง (แม้ `Map` จะ**ไม่ได้สืบทอดจาก `Collection`**
interface แต่ถือเป็นส่วนสำคัญของ Collections Framework)

```java
import java.util.HashMap;
import java.util.Map;

public class MapBasicDemo {
    public static void main(String[] args) {
        Map<String, Integer> ages = new HashMap<>();

        ages.put("Alice", 25);   // เพิ่มคู่ key-value
        ages.put("Bob", 30);
        ages.put("Alice", 26);    // key ซ้ำ -> ค่าเดิมถูกแทนที่ (ไม่ error)

        System.out.println(ages.get("Alice")); // 26 (ค่าล่าสุดที่ put ทับ)
        System.out.println(ages.get("Charlie")); // null (ไม่มี key นี้)
        System.out.println(ages.size());          // 2 (Alice, Bob)
        System.out.println(ages.containsKey("Bob"));    // true
        System.out.println(ages.containsValue(30));     // true
    }
}
```

**เปรียบเทียบ**: ถ้า `List` คือ "ชั้นวางหนังสือที่มีเลขลำดับ" `Map` คือ
"พจนานุกรม" — ค้นหาความหมาย (value) จากคำศัพท์ (key) ได้ทันทีโดยไม่ต้องไล่ดูทีละ
หน้า

## 2. `HashMap` แบบละเอียด

`HashMap` เป็น implementation ที่ใช้บ่อยที่สุด ทำงานเร็ว **O(1)** โดยเฉลี่ยสำหรับ
`get`/`put`/`remove` แต่**ไม่รักษาลำดับการเพิ่มข้อมูล**

```java
import java.util.HashMap;
import java.util.Map;

public class HashMapDemo {
    public static void main(String[] args) {
        Map<String, Double> prices = new HashMap<>();
        prices.put("แอปเปิ้ล", 30.0);
        prices.put("กล้วย", 15.0);
        prices.put("ส้ม", 25.0);

        // ดึงค่าอย่างปลอดภัยด้วยค่า default หาก key ไม่มีอยู่
        double mangoPrice = prices.getOrDefault("มะม่วง", 0.0);
        System.out.println("ราคามะม่วง: " + mangoPrice); // 0.0 (ไม่มีในระบบ)

        prices.remove("กล้วย");
        System.out.println(prices.containsKey("กล้วย")); // false

        prices.replace("แอปเปิ้ล", 35.0); // แก้ไขค่าถ้า key มีอยู่แล้วเท่านั้น
        System.out.println(prices.get("แอปเปิ้ล")); // 35.0
    }
}
```

## 3. `LinkedHashMap` และ `TreeMap`

เหมือนกับ Set ใน Part 23 — `LinkedHashMap` รักษาลำดับการเพิ่ม, `TreeMap` เรียง
ลำดับตาม key อัตโนมัติ (ใช้ Red-Black Tree ภายใน — `O(log n)`)

```java
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.TreeMap;

public class LinkedAndTreeMapDemo {
    public static void main(String[] args) {
        Map<String, Integer> insertionOrder = new LinkedHashMap<>();
        insertionOrder.put("ซี", 3);
        insertionOrder.put("เอ", 1);
        insertionOrder.put("บี", 2);
        System.out.println(insertionOrder); // {ซี=3, เอ=1, บี=2} - ตามลำดับที่ put

        Map<String, Integer> sortedByKey = new TreeMap<>(insertionOrder);
        System.out.println(sortedByKey); // เรียงตาม key ตามลำดับตัวอักษร/unicode อัตโนมัติ

        TreeMap<String, Integer> treeMap = new TreeMap<>(sortedByKey);
        System.out.println(treeMap.firstKey()); // key แรกตามลำดับ
        System.out.println(treeMap.lastKey());  // key สุดท้ายตามลำดับ
    }
}
```

**LinkedHashMap มีประโยชน์พิเศษอีกอย่าง**: สามารถใช้ทำ **LRU Cache (Least
Recently Used)** ได้ด้วยการ override เมธอด `removeEldestEntry()`:

```java
import java.util.LinkedHashMap;
import java.util.Map;

public class LRUCacheDemo extends LinkedHashMap<Integer, String> {
    private final int capacity;

    public LRUCacheDemo(int capacity) {
        super(16, 0.75f, true); // accessOrder=true: จัดลำดับตามการเข้าถึงล่าสุด ไม่ใช่แค่การเพิ่ม
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<Integer, String> eldest) {
        return size() > capacity; // ลบตัวที่ใช้งานนานสุดออกอัตโนมัติเมื่อเกินความจุ
    }

    public static void main(String[] args) {
        LRUCacheDemo cache = new LRUCacheDemo(3);
        cache.put(1, "A");
        cache.put(2, "B");
        cache.put(3, "C");
        cache.get(1);          // เข้าถึง key 1 ทำให้มันกลายเป็น "ใหม่ล่าสุด"
        cache.put(4, "D");     // เกินความจุ -> ลบตัวที่ไม่ได้ใช้นานสุดออก (key 2)

        System.out.println(cache); // {3=C, 1=A, 4=D} (key 2 ถูกลบออกเพราะไม่ได้ใช้นานสุด)
    }
}
```

## 4. เมธอดสำคัญของ Map (รวมเมธอดสมัยใหม่ Java 8+)

```java
import java.util.HashMap;
import java.util.Map;

public class MapModernMethodsDemo {
    public static void main(String[] args) {
        Map<String, Integer> wordCount = new HashMap<>();
        String[] words = {"apple", "banana", "apple", "cherry", "banana", "apple"};

        // แบบเก่า: ต้องเช็คว่ามี key อยู่หรือไม่ก่อนเสมอ (ยืดยาว)
        for (String word : words) {
            if (wordCount.containsKey(word)) {
                wordCount.put(word, wordCount.get(word) + 1);
            } else {
                wordCount.put(word, 1);
            }
        }
        System.out.println(wordCount);

        // แบบใหม่ (Java 8+): merge() - กระชับกว่ามาก ทำสิ่งเดียวกันในบรรทัดเดียว
        Map<String, Integer> wordCount2 = new HashMap<>();
        for (String word : words) {
            wordCount2.merge(word, 1, Integer::sum); // ถ้ามี key อยู่แล้ว บวกค่าเข้าไป, ถ้าไม่มีใส่ 1
        }
        System.out.println(wordCount2); // {banana=2, apple=3, cherry=1}

        // computeIfAbsent: สร้างค่าเริ่มต้นถ้า key ยังไม่มี (มีประโยชน์มากกับ nested collection)
        Map<String, java.util.List<String>> groups = new HashMap<>();
        groups.computeIfAbsent("fruits", k -> new java.util.ArrayList<>()).add("apple");
        groups.computeIfAbsent("fruits", k -> new java.util.ArrayList<>()).add("banana");
        System.out.println(groups); // {fruits=[apple, banana]}

        // computeIfPresent: แก้ไขค่าเฉพาะถ้า key มีอยู่แล้ว
        wordCount2.computeIfPresent("apple", (k, v) -> v * 10);
        System.out.println(wordCount2.get("apple")); // 30

        // compute: ทำได้ทั้งสองกรณี (มีหรือไม่มี key) ในเมธอดเดียว
        wordCount2.compute("mango", (k, v) -> (v == null) ? 1 : v + 1);
        System.out.println(wordCount2.get("mango")); // 1

        // forEach: วนซ้ำแบบ functional style
        wordCount2.forEach((word, count) -> System.out.println(word + ": " + count));
    }
}
```

## 5. การวนซ้ำ Map ด้วยวิธีต่าง ๆ

```java
import java.util.HashMap;
import java.util.Map;

public class MapIterationDemo {
    public static void main(String[] args) {
        Map<String, Integer> scores = new HashMap<>();
        scores.put("Math", 85);
        scores.put("Science", 92);
        scores.put("English", 78);

        // วิธีที่ 1: วนผ่าน entrySet() - แนะนำที่สุด (ได้ทั้ง key และ value ในครั้งเดียว)
        for (Map.Entry<String, Integer> entry : scores.entrySet()) {
            System.out.println(entry.getKey() + ": " + entry.getValue());
        }

        // วิธีที่ 2: วนผ่าน keySet() แล้วค่อย get() value (ช้ากว่าเล็กน้อย เพราะ get ทุกครั้ง)
        for (String subject : scores.keySet()) {
            System.out.println(subject + " -> " + scores.get(subject));
        }

        // วิธีที่ 3: วนผ่าน values() เมื่อไม่สนใจ key เลย
        int total = 0;
        for (int score : scores.values()) {
            total += score;
        }
        System.out.println("คะแนนรวม: " + total);

        // วิธีที่ 4: forEach() แบบ functional (Java 8+)
        scores.forEach((subject, score) -> System.out.println(subject + " = " + score));
    }
}
```

## 6. Map เป็น Key ที่เป็น Object เอง (ทบทวน hashCode/equals)

เช่นเดียวกับ `HashSet` ใน Part 23 — ถ้าใช้ custom class เป็น **key** ของ `HashMap`
ต้อง override `hashCode()` และ `equals()` ให้ถูกต้องเสมอ ไม่งั้นจะหาข้อมูลกลับ
ไม่เจอ

```java
import java.util.HashMap;
import java.util.Map;
import java.util.Objects;

public class Coordinate {
    int x, y;
    public Coordinate(int x, int y) { this.x = x; this.y = y; }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (!(obj instanceof Coordinate)) return false;
        Coordinate other = (Coordinate) obj;
        return x == other.x && y == other.y;
    }

    @Override
    public int hashCode() {
        return Objects.hash(x, y);
    }
}
```

```java
public class MapKeyObjectDemo {
    public static void main(String[] args) {
        Map<Coordinate, String> grid = new HashMap<>();
        grid.put(new Coordinate(1, 1), "จุดเริ่มต้น");

        // สร้าง object ใหม่ที่มีค่าเหมือนกัน แต่คนละ reference
        String result = grid.get(new Coordinate(1, 1));
        System.out.println(result); // "จุดเริ่มต้น" - หาเจอเพราะ hashCode/equals ถูก override ถูกต้อง

        // ถ้าไม่ override สิ่งนี้จะได้ null เพราะ HashMap จะมองว่าเป็นคนละ key กันเลย
    }
}
```

**ข้อควรระวังสำคัญ**: **อย่าใช้ mutable object เป็น key ของ HashMap** ถ้าแก้ไข
field ที่ใช้คำนวณ `hashCode()` หลังจากใส่เป็น key ไปแล้ว จะทำให้**หาข้อมูลนั้น
ไม่เจอ**อีกต่อไป (เพราะ hash bucket ที่ควรอยู่เปลี่ยนไปแล้ว แต่ HashMap ไม่รู้จัก
ย้ายให้อัตโนมัติ) — ควรใช้ immutable object (Part 17) เป็น key เสมอ

## 7. `HashMap` ทำงานอย่างไรภายใน (โครงสร้างข้อมูล)

```
HashMap ภายในคือ array ของ "bucket" โดยแต่ละ bucket เก็บ linked list (หรือ
red-black tree ถ้า bucket มีข้อมูลจำนวนมากตั้งแต่ Java 8) ของ entry ที่มี
hashCode() ตกอยู่ใน bucket เดียวกัน (hash collision)

put("apple", 1):
  1. คำนวณ hashCode("apple") -> ได้ hash value
  2. แปลง hash value เป็น index ของ bucket (hash % จำนวน bucket ทั้งหมด)
  3. เก็บ entry (key="apple", value=1) ไว้ที่ bucket นั้น

Bucket Array:
┌──────┐
│  [0] │ -> (key="cherry", value=3)
│  [1] │ -> (key="apple", value=1) -> (key="grape", value=5)  <- hash collision!
│  [2] │ -> null
│  [3] │ -> (key="banana", value=2)
└──────┘

get("apple"):
  1. คำนวณ hashCode("apple") -> ได้ index เดียวกับตอน put (index 1)
  2. ไปที่ bucket[1] แล้วไล่หาด้วย equals() จนกว่าจะเจอ key ที่ตรงกัน
  3. คืนค่า value ที่ตรงกับ key นั้น
```

**ทำไมต้อง override ทั้ง hashCode และ equals**: `hashCode()` ใช้หาว่า
**"ควรมองหาใน bucket ไหน"** (ขั้นตอน 1-2) ส่วน `equals()` ใช้**ยืนยันว่าใช่ key
ตัวจริงหรือไม่**เมื่อไปถึง bucket นั้นแล้ว (ขั้นตอน 3) — ถ้ามีแค่ตัวหนึ่งถูก
override การค้นหาจะผิดพลาดทันที

**Load Factor และ Resizing**: HashMap มี **load factor** เริ่มต้นที่ 0.75 —
เมื่อจำนวน entry เกิน 75% ของขนาด bucket array ปัจจุบัน จะเกิดการ **resize**
(ขยาย bucket array เป็น 2 เท่า แล้วย้ายทุก entry ใหม่ทั้งหมด) ซึ่งมี cost สูง
ถ้ารู้ขนาดข้อมูลล่วงหน้า ควรระบุ initial capacity ตอนสร้างเพื่อลดการ resize
ที่ไม่จำเป็น:

```java
Map<String, Integer> map = new HashMap<>(1000); // จองพื้นที่ล่วงหน้า ลด resize ในอนาคต
```

## 8. Nested Map และโครงสร้างข้อมูลซับซ้อน

```java
import java.util.HashMap;
import java.util.Map;

public class NestedMapDemo {
    public static void main(String[] args) {
        // จำลองข้อมูลนักเรียนแต่ละห้อง: Map<ห้อง, Map<ชื่อนักเรียน, คะแนน>>
        Map<String, Map<String, Integer>> classScores = new HashMap<>();

        classScores.computeIfAbsent("ห้อง A", k -> new HashMap<>()).put("Alice", 85);
        classScores.computeIfAbsent("ห้อง A", k -> new HashMap<>()).put("Bob", 90);
        classScores.computeIfAbsent("ห้อง B", k -> new HashMap<>()).put("Charlie", 78);

        for (Map.Entry<String, Map<String, Integer>> classEntry : classScores.entrySet()) {
            System.out.println(classEntry.getKey() + ":");
            for (Map.Entry<String, Integer> studentEntry : classEntry.getValue().entrySet()) {
                System.out.println("  " + studentEntry.getKey() + " = " + studentEntry.getValue());
            }
        }
    }
}
```

## 9. `Map.Entry` และการประมวลผลแบบ Stream

จะลงลึกเรื่อง Stream ใน Part 41-42 แต่ควรเห็นตัวอย่างการผสม Map กับ Stream
ตั้งแต่ตอนนี้ เพราะเป็นรูปแบบที่ใช้บ่อยมากในโค้ดสมัยใหม่:

```java
import java.util.HashMap;
import java.util.Map;

public class MapStreamPreviewDemo {
    public static void main(String[] args) {
        Map<String, Integer> scores = new HashMap<>();
        scores.put("Alice", 85);
        scores.put("Bob", 92);
        scores.put("Charlie", 78);

        // หาคนที่คะแนนสูงสุดด้วย Stream (ปูพื้นฐานสำหรับ Part 41-42)
        scores.entrySet().stream()
              .max(Map.Entry.comparingByValue())
              .ifPresent(entry -> System.out.println("คะแนนสูงสุด: "
                                  + entry.getKey() + " (" + entry.getValue() + ")"));

        // เรียงลำดับตาม value แล้วพิมพ์ออกมา
        scores.entrySet().stream()
              .sorted(Map.Entry.comparingByValue(Comparator.reverseOrder()))
              .forEach(e -> System.out.println(e.getKey() + ": " + e.getValue()));
    }
}
```
(หมายเหตุ: ต้อง `import java.util.Comparator;` เพิ่มเติมสำหรับตัวอย่างข้างบน)

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียนโปรแกรมนับความถี่ของตัวอักษรในข้อความ (character frequency counter)

**เฉลย:**

```java
import java.util.HashMap;
import java.util.Map;

public class Exercise1 {
    public static void main(String[] args) {
        String text = "hello world";
        Map<Character, Integer> frequency = new HashMap<>();
        for (char c : text.toCharArray()) {
            if (c != ' ') {
                frequency.merge(c, 1, Integer::sum);
            }
        }
        System.out.println(frequency); // {r=1, o=2, l=3, h=1, w=1, e=1, d=1}
    }
}
```

**2)** เขียนโปรแกรมกลุ่มคำตามความยาวของคำ โดยใช้
`Map<Integer, List<String>>`

**เฉลย:**

```java
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class Exercise2 {
    public static void main(String[] args) {
        String[] words = {"cat", "dog", "elephant", "ant", "bee", "tiger"};
        Map<Integer, List<String>> byLength = new HashMap<>();
        for (String word : words) {
            byLength.computeIfAbsent(word.length(), k -> new ArrayList<>()).add(word);
        }
        System.out.println(byLength); // {3=[cat, dog, ant, bee], 8=[elephant], 5=[tiger]}
    }
}
```

**3)** เขียนโปรแกรมที่มี `Map<String, Double>` ราคาสินค้า แล้วหาสินค้าที่ราคา
แพงที่สุดโดยใช้ `entrySet()` (ไม่ใช้ Stream)

**เฉลย:**

```java
import java.util.HashMap;
import java.util.Map;

public class Exercise3 {
    public static void main(String[] args) {
        Map<String, Double> prices = new HashMap<>();
        prices.put("Laptop", 25000.0);
        prices.put("Mouse", 500.0);
        prices.put("Keyboard", 1200.0);

        String mostExpensive = null;
        double maxPrice = Double.MIN_VALUE;
        for (Map.Entry<String, Double> entry : prices.entrySet()) {
            if (entry.getValue() > maxPrice) {
                maxPrice = entry.getValue();
                mostExpensive = entry.getKey();
            }
        }
        System.out.println("สินค้าแพงสุด: " + mostExpensive + " (" + maxPrice + " บาท)");
    }
}
```

### สรุปเนื้อหา Part 24

- `Map<K, V>` เก็บคู่ key-value โดย key ไม่ซ้ำกัน, value ซ้ำกันได้
- `HashMap` เร็วที่สุด (O(1)) ไม่รักษาลำดับ, `LinkedHashMap` รักษาลำดับการเพิ่ม,
  `TreeMap` เรียงลำดับตาม key
- เมธอดสมัยใหม่ `merge()`, `computeIfAbsent()`, `computeIfPresent()`, `compute()`
  ช่วยลดโค้ด boilerplate ได้มาก
- ใช้ `entrySet()` วนซ้ำ Map เป็นวิธีที่มีประสิทธิภาพที่สุด (ได้ key และ value
  พร้อมกัน)
- HashMap key ต้อง override `hashCode()`/`equals()` ถูกต้อง และควรเป็น
  immutable object เท่านั้น
- HashMap ภายในใช้ bucket array + hash collision resolution, มี load factor
  0.75 ที่ทำให้ resize อัตโนมัติ

**ต่อไป**: [Part 25 — Queue, Deque, PriorityQueue](./part-025-queue-deque.md)
