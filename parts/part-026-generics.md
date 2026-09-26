# Part 26: Generics: Generic Class, Method, Bounded Types, Wildcards

> ขั้นตอนที่ 251-260 ของหลักสูตร | ระดับ: กลาง-สูง

## สารบัญ

1. Generics คืออะไร แก้ปัญหาอะไร
2. Generic Class
3. Generic Method
4. Multiple Type Parameters
5. Bounded Type Parameters (`extends`)
6. Wildcards: `?`, `? extends T`, `? super T`
7. PECS Principle (Producer Extends, Consumer Super)
8. Type Erasure: สิ่งที่เกิดขึ้นจริงตอน Compile
9. ข้อจำกัดของ Generics ใน Java
10. แบบฝึกหัดและสรุป

---

## 1. Generics คืออะไร แก้ปัญหาอะไร

**Generics** ช่วยให้เขียน class, interface, และ method ที่**ทำงานกับชนิดข้อมูล
ได้หลายแบบ**โดยยังคง **type safety** (ตรวจสอบชนิดข้อมูลตอน compile-time) — เรา
ใช้ generics มาตลอดทั้งหลักสูตรแล้วโดยไม่รู้ตัว (`List<String>`, `Map<K, V>`)
Part นี้จะสอนวิธี**สร้าง generic ของตัวเอง**

ก่อนมี generics (Java 1.4 และก่อนหน้า) ต้องใช้ `Object` แทน ซึ่งไม่ปลอดภัยเลย:

```java
import java.util.ArrayList;

public class PreGenericsProblem {
    public static void main(String[] args) {
        ArrayList list = new ArrayList(); // ไม่ระบุชนิดข้อมูล (raw type - แบบเก่ามาก)
        list.add("Hello");
        list.add(123); // เพิ่มชนิดข้อมูลอะไรก็ได้ ไม่มีการตรวจสอบเลย!

        String text = (String) list.get(1); // ClassCastException! เพราะ index 1 คือ Integer ไม่ใช่ String
        // ปัญหานี้ compiler จับไม่ได้เลย ต้องรอไปพังตอน runtime เท่านั้น
    }
}
```

```java
import java.util.ArrayList;
import java.util.List;

public class WithGenericsSolution {
    public static void main(String[] args) {
        List<String> list = new ArrayList<>(); // ระบุชนิดข้อมูลชัดเจนด้วย Generics
        list.add("Hello");
        // list.add(123); // Error! compiler ปฏิเสธทันที เพราะ list นี้รับได้แค่ String

        String text = list.get(0); // ไม่ต้อง cast เองเลย - compiler รู้ชนิดข้อมูลแน่นอนแล้ว
        System.out.println(text);
    }
}
```

## 2. Generic Class

สร้าง class ที่รับ **type parameter** (มักเขียนเป็นตัวอักษรตัวเดียว เช่น `T`
ย่อจาก "Type") แทนที่จะระบุชนิดข้อมูลตายตัว:

```java
public class Box<T> { // T คือ type parameter - จะถูกแทนที่ด้วยชนิดข้อมูลจริงตอนใช้งาน
    private T content;

    public void set(T content) {
        this.content = content;
    }

    public T get() {
        return content;
    }

    public boolean isEmpty() {
        return content == null;
    }
}
```

```java
public class GenericClassDemo {
    public static void main(String[] args) {
        Box<String> stringBox = new Box<>(); // ระบุว่า T คือ String
        stringBox.set("Hello Generics");
        System.out.println(stringBox.get()); // "Hello Generics" - ไม่ต้อง cast

        Box<Integer> intBox = new Box<>(); // Box เดียวกัน แต่ใช้กับ Integer ได้ด้วย
        intBox.set(42);
        int value = intBox.get(); // auto-unboxing (ทบทวนจาก Part 28)
        System.out.println(value);

        // Box rawBox = new Box(); // ใช้ได้แต่ไม่แนะนำ (raw type - เสีย type safety ทั้งหมด)
    }
}
```

### Generic Class ที่มี Type Parameter หลายตัว (จะขยายในหัวข้อ 4)

```java
public class Pair<K, V> {
    private K key;
    private V value;

    public Pair(K key, V value) {
        this.key = key;
        this.value = value;
    }

    public K getKey() { return key; }
    public V getValue() { return value; }

    @Override
    public String toString() {
        return "(" + key + ", " + value + ")";
    }
}
```

```java
public class PairDemo {
    public static void main(String[] args) {
        Pair<String, Integer> nameAge = new Pair<>("Alice", 25);
        System.out.println(nameAge); // (Alice, 25)
    }
}
```

## 3. Generic Method

Method เดี่ยว ๆ ก็สามารถมี type parameter ของตัวเองได้ **โดยไม่จำเป็นต้องอยู่ใน
generic class** — type parameter ประกาศไว้ก่อน return type

```java
public class GenericMethodDemo {
    // <T> ประกาศ type parameter ของเมธอดนี้เอง (ไม่เกี่ยวกับ class)
    static <T> void printArray(T[] array) {
        for (T element : array) {
            System.out.print(element + " ");
        }
        System.out.println();
    }

    // Generic method ที่มี return type เป็น T ด้วย
    static <T> T getFirst(T[] array) {
        if (array.length == 0) {
            throw new IllegalArgumentException("array ห้ามว่างเปล่า");
        }
        return array[0];
    }

    // Generic method ที่ compiler อนุมานชนิดข้อมูลจาก argument โดยอัตโนมัติ
    static <T> boolean isEqual(T a, T b) {
        return a.equals(b);
    }

    public static void main(String[] args) {
        Integer[] intArray = {1, 2, 3};
        String[] stringArray = {"a", "b", "c"};

        printArray(intArray);   // T ถูกอนุมานเป็น Integer โดยอัตโนมัติ
        printArray(stringArray); // T ถูกอนุมานเป็น String โดยอัตโนมัติ

        System.out.println(getFirst(intArray)); // 1
        System.out.println(isEqual("hello", "hello")); // true
    }
}
```

## 4. Multiple Type Parameters

```java
public class Triple<A, B, C> {
    private A first;
    private B second;
    private C third;

    public Triple(A first, B second, C third) {
        this.first = first;
        this.second = second;
        this.third = third;
    }

    @Override
    public String toString() {
        return "(" + first + ", " + second + ", " + third + ")";
    }
}
```

```java
public class MultipleTypeParametersDemo {
    static <K, V> void printPair(K key, V value) {
        System.out.println(key + " -> " + value);
    }

    public static void main(String[] args) {
        Triple<String, Integer, Boolean> data = new Triple<>("Alice", 25, true);
        System.out.println(data); // (Alice, 25, true)

        printPair("username", "somchai");
        printPair(1, "first");
    }
}
```

## 5. Bounded Type Parameters (`extends`)

**Bounded Type Parameter** จำกัดว่า `T` ต้องเป็น subtype ของชนิดข้อมูลที่กำหนด
(ใช้ `extends` แม้จะเป็น interface ก็ตาม — Java ใช้ `extends` เสมอในบริบท
generics ไม่ใช้ `implements`)

```java
public class NumberBox<T extends Number> { // T ต้องเป็น Number หรือ subtype (Integer, Double, ...)
    private T value;

    public NumberBox(T value) {
        this.value = value;
    }

    public double doubleValue() {
        return value.doubleValue(); // เรียกเมธอดของ Number ได้เลย เพราะ compiler รู้ว่า T extends Number
    }
}
```

```java
public class BoundedTypeDemo {
    public static void main(String[] args) {
        NumberBox<Integer> intBox = new NumberBox<>(42);
        NumberBox<Double> doubleBox = new NumberBox<>(3.14);
        System.out.println(intBox.doubleValue());    // 42.0
        System.out.println(doubleBox.doubleValue());  // 3.14

        // NumberBox<String> stringBox = new NumberBox<>("text"); // Error! String ไม่ใช่ Number
    }
}
```

### Generic Method หา max ด้วย Bounded Type + `Comparable`

```java
import java.util.List;

public class MaxFinderDemo {
    // T ต้อง implement Comparable<T> จึงจะเปรียบเทียบด้วย compareTo() ได้ (ทบทวนจาก Part 23)
    static <T extends Comparable<T>> T findMax(List<T> list) {
        T max = list.get(0);
        for (T item : list) {
            if (item.compareTo(max) > 0) {
                max = item;
            }
        }
        return max;
    }

    public static void main(String[] args) {
        System.out.println(findMax(List.of(3, 7, 2, 9, 4)));           // 9
        System.out.println(findMax(List.of("banana", "apple", "cherry"))); // cherry
    }
}
```

## 6. Wildcards: `?`, `? extends T`, `? super T`

**Wildcard** ใช้เมื่อไม่ต้องการระบุ type parameter เฉพาะเจาะจง แต่ต้องการความ
ยืดหยุ่นในการรับ generic type ที่หลากหลาย มี 3 รูปแบบ:

### Unbounded Wildcard: `?`

```java
import java.util.List;

public class UnboundedWildcardDemo {
    // รับ List ของชนิดข้อมูลอะไรก็ได้ (ไม่รู้และไม่สนใจว่าเป็นชนิดไหน)
    static void printListSize(List<?> list) {
        System.out.println("จำนวนสมาชิก: " + list.size());
        // list.add("x"); // Error! ไม่รู้ชนิดข้อมูลจริง จึงเพิ่มอะไรเข้าไปไม่ได้เลย (ปลอดภัยไว้ก่อน)
    }

    public static void main(String[] args) {
        printListSize(List.of(1, 2, 3));
        printListSize(List.of("a", "b"));
    }
}
```

### Upper Bounded Wildcard: `? extends T`

```java
import java.util.List;

public class UpperBoundedWildcardDemo {
    // รับ List ของ Number หรือ subtype ใดก็ได้ (Integer, Double, ...)
    static double sumAll(List<? extends Number> list) {
        double sum = 0;
        for (Number n : list) { // อ่านค่าออกมาได้ (as Number) แต่เพิ่มค่าใหม่เข้าไปไม่ได้
            sum += n.doubleValue();
        }
        return sum;
    }

    public static void main(String[] args) {
        System.out.println(sumAll(List.of(1, 2, 3)));       // ใช้กับ List<Integer> ได้
        System.out.println(sumAll(List.of(1.5, 2.5, 3.5)));  // ใช้กับ List<Double> ได้ด้วย
    }
}
```

### Lower Bounded Wildcard: `? super T`

```java
import java.util.List;
import java.util.ArrayList;

public class LowerBoundedWildcardDemo {
    // รับ List ที่เก็บ Integer หรือ superclass ของ Integer ได้ (Integer, Number, Object)
    static void addNumbers(List<? super Integer> list) {
        list.add(1); // เพิ่มค่าเข้าไปได้ เพราะ list รับ Integer หรือ superclass ของมันได้แน่นอน
        list.add(2);
        list.add(3);
    }

    public static void main(String[] args) {
        List<Number> numberList = new ArrayList<>();
        addNumbers(numberList); // ใช้ได้ เพราะ Number เป็น superclass ของ Integer
        System.out.println(numberList); // [1, 2, 3]

        List<Object> objectList = new ArrayList<>();
        addNumbers(objectList); // ใช้ได้เช่นกัน เพราะ Object เป็น superclass ของทุกอย่าง
    }
}
```

## 7. PECS Principle (Producer Extends, Consumer Super)

**PECS** เป็นหลักช่วยจำในการเลือก wildcard ที่เหมาะสม:

- **"Producer Extends"**: ถ้า generic structure ทำหน้าที่**ผลิต/คืนค่า**ออกมา
  (อ่านค่าจากมัน) ให้ใช้ `? extends T`
- **"Consumer Super"**: ถ้า generic structure ทำหน้าที่**รับ/บริโภคค่า**เข้าไป
  (เขียนค่าใส่มัน) ให้ใช้ `? super T`

```java
import java.util.List;

public class PECSDemo {
    // copy จาก source (producer - อ่านค่าออกมา) ไปยัง destination (consumer - เขียนค่าเข้าไป)
    static <T> void copy(List<? extends T> source, List<? super T> destination) {
        for (T item : source) {
            destination.add(item);
        }
    }

    public static void main(String[] args) {
        List<Integer> integers = List.of(1, 2, 3);
        List<Number> numbers = new java.util.ArrayList<>();

        copy(integers, numbers); // source ผลิต Integer (extends), destination รับ Number (super)
        System.out.println(numbers); // [1, 2, 3]
    }
}
```

`Collections.copy()` ใน JDK จริงก็ใช้หลักการนี้เป๊ะ ๆ:

```java
// signature จริงจาก java.util.Collections
// public static <T> void copy(List<? super T> dest, List<? extends T> src)
```

## 8. Type Erasure: สิ่งที่เกิดขึ้นจริงตอน Compile

**Type Erasure** คือกลไกที่ Java ใช้ implement generics — **type parameter จะถูก
ลบออกไปตอน compile-time** และแทนที่ด้วย `Object` (หรือ upper bound ที่กำหนด)
นี่คือเหตุผลที่ generics ของ Java ทำงานต่างจาก C++ templates หรือ generics ของ
บางภาษา:

```java
public class TypeErasureDemo {
    public static void main(String[] args) {
        java.util.List<String> stringList = new java.util.ArrayList<>();
        java.util.List<Integer> intList = new java.util.ArrayList<>();

        // ตอน runtime ทั้งสองมี Class object เดียวกัน! เพราะ type parameter ถูก "erase" ไปแล้ว
        System.out.println(stringList.getClass() == intList.getClass()); // true (!)

        // ก่อน compile:  List<String> list = new ArrayList<String>();
        // หลัง compile:  List list = new ArrayList(); (แทรก cast อัตโนมัติตรงจุดที่ get() ค่า)
    }
}
```

**ผลกระทบที่สำคัญของ Type Erasure**:

```java
public class TypeErasureLimitationsDemo {
    // static <T> T createInstance() {
    //     return new T(); // Error! สร้าง instance ของ T ตรง ๆ ไม่ได้ เพราะตอน runtime ไม่รู้ว่า T คืออะไร
    // }

    // static <T> void checkType(Object obj) {
    //     if (obj instanceof T) { } // Error! ตรวจสอบ instanceof กับ type parameter ตรง ๆ ไม่ได้
    // }

    public static void main(String[] args) {
        // List<String>[] arrays = new List<String>[10]; // Error! สร้าง generic array ตรง ๆ ไม่ได้
        // (เกี่ยวข้องกับความปลอดภัยของ type erasure ร่วมกับ array covariance)
    }
}
```

## 9. ข้อจำกัดของ Generics ใน Java

1. **ใช้กับ primitive type ตรง ๆ ไม่ได้**: ต้องใช้ Wrapper class เสมอ
   (`List<int>` ผิด ต้องเป็น `List<Integer>`)
2. **สร้าง instance ของ type parameter ตรง ๆ ไม่ได้** (`new T()`)
3. **สร้าง generic array ตรง ๆ ไม่ได้** (`new T[10]`)
4. **`instanceof` กับ type parameter ตรง ๆ ไม่ได้** (เพราะ type erasure)
5. **Static context เข้าถึง type parameter ของ class ไม่ได้** (static field/
   method ไม่ผูกกับ instance ใด ๆ แต่ type parameter ผูกกับ instance)

```java
public class StaticLimitationDemo<T> {
    // static T value; // Error! static field ใช้ type parameter ของ class ไม่ได้
    T value; // instance field ใช้ได้ปกติ

    static <U> void staticGenericMethod(U value) { // แต่ static METHOD มี type parameter ของตัวเองได้
        System.out.println(value);
    }
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** สร้าง generic class `Stack<T>` ของตัวเอง (ใช้ `ArrayDeque` ภายใน) พร้อม
เมธอด `push()`, `pop()`, `peek()`, `isEmpty()`

**เฉลย:**

```java
import java.util.ArrayDeque;
import java.util.Deque;

public class MyStack<T> {
    private final Deque<T> items = new ArrayDeque<>();

    public void push(T item) { items.push(item); }
    public T pop() { return items.pop(); }
    public T peek() { return items.peek(); }
    public boolean isEmpty() { return items.isEmpty(); }
}
```

**2)** เขียน generic method `swap(T[] array, int i, int j)` ที่สลับตำแหน่งสอง
element ใน array

**เฉลย:**

```java
public class Exercise2 {
    static <T> void swap(T[] array, int i, int j) {
        T temp = array[i];
        array[i] = array[j];
        array[j] = temp;
    }

    public static void main(String[] args) {
        Integer[] numbers = {1, 2, 3};
        swap(numbers, 0, 2);
        System.out.println(java.util.Arrays.toString(numbers)); // [3, 2, 1]
    }
}
```

**3)** เขียนเมธอด `printAll(Collection<? extends Animal> animals)` ที่รับ
collection ของ Animal หรือ subtype ใดก็ได้ แล้ววนพิมพ์ทุกตัว (ใช้ PECS principle
อธิบายว่าทำไมต้องใช้ `extends`)

**เฉลย:**

```java
import java.util.Collection;

public class Exercise3 {
    static void printAll(Collection<? extends Animal> animals) {
        for (Animal a : animals) { // อ่านค่าออกมาอย่างเดียว (producer) -> ใช้ extends ถูกต้องตาม PECS
            System.out.println(a);
        }
    }
}
```

### สรุปเนื้อหา Part 26

- Generics ให้ type safety ตอน compile-time ลดการต้อง cast เอง และป้องกัน
  `ClassCastException`
- Generic class/method ประกาศ type parameter ด้วย `<T>` ก่อนใช้งาน
- Bounded type parameter (`T extends Number`) จำกัดว่า T ต้องเป็น subtype ที่
  กำหนด
- Wildcard: `?` (ไม่ระบุ), `? extends T` (อ่านค่า - producer), `? super T`
  (เขียนค่า - consumer) ตามหลัก PECS
- Type Erasure ทำให้ type parameter ถูกลบออกตอน compile กลายเป็น `Object` หรือ
  upper bound — เป็นที่มาของข้อจำกัดหลายอย่างของ generics

**ต่อไป**: [Part 27 — Iterator, Iterable, Comparable, Comparator](./part-027-iterator-comparable.md)
