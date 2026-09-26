# Part 40: Functional Interfaces: Function, Predicate, Consumer, Supplier

> ขั้นตอนที่ 391-400 ของหลักสูตร | ระดับ: กลาง-สูง

## สารบัญ

1. ภาพรวม `java.util.function` Package
2. `Function<T, R>`: แปลงค่าจาก T เป็น R
3. `Predicate<T>`: ทดสอบเงื่อนไข คืนค่า boolean
4. `Consumer<T>`: รับค่าเข้าไปทำงาน ไม่คืนค่า
5. `Supplier<T>`: ไม่รับพารามิเตอร์ คืนค่ากลับมา
6. `BiFunction`, `BiConsumer`, `BiPredicate`: รับสองพารามิเตอร์
7. `UnaryOperator` และ `BinaryOperator`
8. Primitive Specialization (หลีกเลี่ยง Autoboxing)
9. Composing Functions: `andThen`, `compose`, `and`, `or`, `negate`
10. แบบฝึกหัดและสรุป

---

## 1. ภาพรวม `java.util.function` Package

Java 8 เพิ่ม package `java.util.function` ที่รวม **functional interface
สำเร็จรูป** ไว้ใช้ครอบคลุมกรณีทั่วไปเกือบทั้งหมด ทำให้ไม่ต้องสร้าง custom
functional interface เองบ่อย ๆ (ทบทวนจาก Part 39) — มี 4 ตัวหลักที่ใช้บ่อย
ที่สุดและเป็นพื้นฐานของ Stream API (Part 41-42)

| Interface | Abstract Method | รับพารามิเตอร์ | คืนค่า | ใช้เมื่อ |
|---|---|---|---|---|
| `Function<T, R>` | `R apply(T t)` | 1 | มี | แปลงค่า |
| `Predicate<T>` | `boolean test(T t)` | 1 | boolean | ทดสอบเงื่อนไข |
| `Consumer<T>` | `void accept(T t)` | 1 | ไม่มี | ทำงานกับค่า (side effect) |
| `Supplier<T>` | `T get()` | 0 | มี | สร้าง/จ่ายค่า |

## 2. `Function<T, R>`: แปลงค่าจาก T เป็น R

```java
import java.util.function.Function;

public class FunctionDemo {
    public static void main(String[] args) {
        Function<String, Integer> stringLength = String::length;
        System.out.println(stringLength.apply("Hello")); // 5

        Function<Integer, Integer> square = x -> x * x;
        System.out.println(square.apply(5)); // 25

        // ใช้เป็น parameter ของเมธอดที่รับ Function (higher-order function)
        System.out.println(applyTwice(square, 3)); // square(square(3)) = square(9) = 81
    }

    static int applyTwice(Function<Integer, Integer> f, int input) {
        return f.apply(f.apply(input));
    }
}
```

## 3. `Predicate<T>`: ทดสอบเงื่อนไข คืนค่า boolean

`Predicate` ใช้บ่อยที่สุดกับ `Stream.filter()` (Part 41) — เป็นตัวแทนของ
"เงื่อนไขในรูปแบบ object" ที่ส่งต่อได้เหมือนตัวแปรทั่วไป

```java
import java.util.function.Predicate;
import java.util.List;
import java.util.ArrayList;

public class PredicateDemo {
    public static void main(String[] args) {
        Predicate<Integer> isEven = n -> n % 2 == 0;
        Predicate<String> isEmpty = String::isEmpty;

        System.out.println(isEven.test(4));   // true
        System.out.println(isEmpty.test(""));  // true

        // ใช้กรองข้อมูลด้วยมือ (ก่อนจะไปใช้ Stream.filter ที่กระชับกว่าใน Part 41)
        List<Integer> numbers = List.of(1, 2, 3, 4, 5, 6);
        List<Integer> evens = filterList(numbers, isEven);
        System.out.println(evens); // [2, 4, 6]
    }

    static <T> List<T> filterList(List<T> list, Predicate<T> condition) {
        List<T> result = new ArrayList<>();
        for (T item : list) {
            if (condition.test(item)) {
                result.add(item);
            }
        }
        return result;
    }
}
```

## 4. `Consumer<T>`: รับค่าเข้าไปทำงาน ไม่คืนค่า

`Consumer` เหมาะกับการ**ทำ side effect** (เช่น print, บันทึกลง database, log)
— ใช้บ่อยกับ `List.forEach()` และ `Map.forEach()` (ทบทวนจาก Part 22, 24)

```java
import java.util.function.Consumer;
import java.util.List;

public class ConsumerDemo {
    public static void main(String[] args) {
        Consumer<String> printer = System.out::println;
        Consumer<String> logger = s -> System.out.println("[LOG] " + s);

        List<String> messages = List.of("สวัสดี", "ลาก่อน");
        messages.forEach(printer);
        messages.forEach(logger);

        // Consumer.andThen(): เรียงลำดับการทำงานหลาย Consumer ต่อกัน (ทบทวนหัวข้อ 9)
        Consumer<String> combined = printer.andThen(logger);
        combined.accept("ทดสอบ"); // พิมพ์ปกติก่อน แล้วค่อย log ตามหลัง
    }
}
```

## 5. `Supplier<T>`: ไม่รับพารามิเตอร์ คืนค่ากลับมา

`Supplier` ใช้เมื่อต้องการ**"เลื่อนการคำนวณค่า"ออกไปจนกว่าจะจำเป็นจริง ๆ**
(lazy evaluation) — มีประโยชน์มากเมื่อการคำนวณค่ามี cost สูงและอาจไม่ถูกใช้เลย

```java
import java.util.function.Supplier;

public class SupplierDemo {
    static String expensiveOperation() {
        System.out.println("กำลังคำนวณค่าที่ใช้เวลานาน...");
        return "ผลลัพธ์ที่ได้";
    }

    public static void main(String[] args) {
        Supplier<String> lazySupplier = SupplierDemo::expensiveOperation;

        System.out.println("ก่อนเรียก get()"); // ยังไม่มีการคำนวณเกิดขึ้นตรงนี้
        String result = lazySupplier.get();      // การคำนวณเกิดขึ้น "ตอนนี้" เท่านั้น
        System.out.println(result);

        // ตัวอย่างการใช้ประโยชน์จริง: Optional.orElseGet() (จะเรียนใน Part 43)
        java.util.Optional<String> maybeEmpty = java.util.Optional.empty();
        String value = maybeEmpty.orElseGet(SupplierDemo::expensiveOperation);
        // expensiveOperation() ถูกเรียกเฉพาะเมื่อ Optional ว่างจริง ๆ (lazy)
        // ต่างจาก orElse(expensiveOperation()) ที่จะเรียกเสมอแม้ Optional มีค่าอยู่แล้ว!
    }
}
```

## 6. `BiFunction`, `BiConsumer`, `BiPredicate`: รับสองพารามิเตอร์

เมื่อต้องรับ**สองพารามิเตอร์** ใช้ prefix "Bi" (ทบทวนตัวอย่างจาก Part 24
`merge()` ที่ใช้ `BiFunction`):

```java
import java.util.function.BiFunction;
import java.util.function.BiConsumer;
import java.util.function.BiPredicate;
import java.util.Map;
import java.util.HashMap;

public class BiFunctionDemo {
    public static void main(String[] args) {
        BiFunction<Integer, Integer, Integer> add = (a, b) -> a + b;
        System.out.println(add.apply(3, 4)); // 7

        BiConsumer<String, Integer> printPair = (name, age) ->
            System.out.println(name + " is " + age + " years old");
        printPair.accept("Alice", 25);

        BiPredicate<String, Integer> isValidAge = (name, age) -> age >= 0 && age <= 150;
        System.out.println(isValidAge.test("Bob", 30)); // true

        // Map.forEach() ใช้ BiConsumer (ทบทวนจาก Part 24)
        Map<String, Integer> ages = new HashMap<>();
        ages.put("Alice", 25);
        ages.put("Bob", 30);
        ages.forEach((name, age) -> System.out.println(name + ": " + age));
    }
}
```

## 7. `UnaryOperator` และ `BinaryOperator`

**`UnaryOperator<T>`** คือ `Function<T, T>` ชนิดพิเศษที่**input และ output
เป็นชนิดเดียวกัน** เช่นเดียวกับ **`BinaryOperator<T>`** ที่เป็น
`BiFunction<T, T, T>` — ใช้เพื่อความชัดเจนของ intent ในโค้ด:

```java
import java.util.function.UnaryOperator;
import java.util.function.BinaryOperator;

public class OperatorDemo {
    public static void main(String[] args) {
        UnaryOperator<Integer> increment = x -> x + 1; // input/output เป็น Integer เหมือนกัน
        System.out.println(increment.apply(5)); // 6

        BinaryOperator<Integer> max = (a, b) -> (a > b) ? a : b;
        System.out.println(max.apply(3, 7)); // 7

        // ใช้บ่อยกับ List.replaceAll() (รับ UnaryOperator)
        java.util.List<Integer> numbers = new java.util.ArrayList<>(java.util.List.of(1, 2, 3));
        numbers.replaceAll(increment); // แก้ไขทุก element ในตัวเดิม
        System.out.println(numbers); // [2, 3, 4]

        // BinaryOperator มี static helper methods
        BinaryOperator<Integer> minOp = BinaryOperator.minBy(Integer::compareTo);
        System.out.println(minOp.apply(5, 3)); // 3
    }
}
```

## 8. Primitive Specialization (หลีกเลี่ยง Autoboxing)

ทบทวนจาก Part 28: autoboxing ใน loop จำนวนมากทำให้ performance แย่ — Java
มี functional interface เวอร์ชัน**primitive specialization** เพื่อหลีกเลี่ยง
ปัญหานี้ (เช่น `IntFunction`, `IntPredicate`, `IntConsumer`, `IntSupplier`,
`ToIntFunction`, `IntBinaryOperator`):

```java
import java.util.function.IntPredicate;
import java.util.function.IntUnaryOperator;
import java.util.function.ToIntFunction;

public class PrimitiveSpecializationDemo {
    public static void main(String[] args) {
        // IntPredicate: ไม่มี autoboxing เพราะรับ int ตรง ๆ ไม่ต้องแปลงเป็น Integer
        IntPredicate isEven = n -> n % 2 == 0;
        System.out.println(isEven.test(4)); // true (ไม่มี autoboxing เกิดขึ้นเลย)

        IntUnaryOperator doubleIt = n -> n * 2;
        System.out.println(doubleIt.applyAsInt(5)); // 10

        ToIntFunction<String> stringToLength = String::length; // รับ T คืน int (ไม่ box ผลลัพธ์)
        System.out.println(stringToLength.applyAsInt("Hello")); // 5

        // เทียบกับ Predicate<Integer> ธรรมดาที่ต้อง autobox ทุกครั้งที่เรียก
        java.util.function.Predicate<Integer> isEvenBoxed = n -> n % 2 == 0; // มี autoboxing แอบแฝง
    }
}
```

มี specialization ครบทั้ง `int`, `long`, `double` — ใช้เมื่อทำงานกับข้อมูล
จำนวนมากในลูปที่ performance-critical

## 9. Composing Functions: `andThen`, `compose`, `and`, `or`, `negate`

Functional interfaces มี **default method** (ทบทวนจาก Part 16) ที่ใช้
**รวม (compose) function หลายตัวเข้าด้วยกัน** — เทคนิคสำคัญของ functional
programming

```java
import java.util.function.Function;
import java.util.function.Predicate;

public class ComposingFunctionsDemo {
    public static void main(String[] args) {
        Function<Integer, Integer> addTwo = x -> x + 2;
        Function<Integer, Integer> multiplyByThree = x -> x * 3;

        // andThen: ทำ f ก่อน แล้วส่งผลลัพธ์ไปทำ g ต่อ -> g(f(x))
        Function<Integer, Integer> addThenMultiply = addTwo.andThen(multiplyByThree);
        System.out.println(addThenMultiply.apply(5)); // (5+2)*3 = 21

        // compose: ทำ g ก่อน แล้วส่งผลลัพธ์ไปทำ f ต่อ -> f(g(x)) (ทิศทางตรงข้ามกับ andThen)
        Function<Integer, Integer> multiplyThenAdd = addTwo.compose(multiplyByThree);
        System.out.println(multiplyThenAdd.apply(5)); // (5*3)+2 = 17

        // Predicate composition: and, or, negate
        Predicate<Integer> isPositive = n -> n > 0;
        Predicate<Integer> isEven = n -> n % 2 == 0;

        Predicate<Integer> isPositiveAndEven = isPositive.and(isEven);
        Predicate<Integer> isPositiveOrEven = isPositive.or(isEven);
        Predicate<Integer> isNotPositive = isPositive.negate();

        System.out.println(isPositiveAndEven.test(4));  // true (4 > 0 และ 4 % 2 == 0)
        System.out.println(isPositiveAndEven.test(-4));  // false (ไม่ > 0)
        System.out.println(isNotPositive.test(-5));        // true
    }
}
```

**ตัวอย่างการใช้งานจริง**: การประกอบ validation logic หลายชั้นเข้าด้วยกัน:

```java
import java.util.function.Predicate;

public class ValidationCompositionDemo {
    public static void main(String[] args) {
        Predicate<String> notNull = s -> s != null;
        Predicate<String> notEmpty = s -> !s.isEmpty();
        Predicate<String> maxLength = s -> s.length() <= 20;

        Predicate<String> isValidUsername = notNull.and(notEmpty).and(maxLength);

        System.out.println(isValidUsername.test("somchai"));  // true
        System.out.println(isValidUsername.test(""));           // false
        System.out.println(isValidUsername.test(null));           // false (notNull เช็คก่อน short-circuit)
    }
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียนเมธอด `processAll(List<T> list, Consumer<T> action)` ทั่วไป แล้ว
ใช้ทดสอบพิมพ์ข้อมูลด้วยรูปแบบต่าง ๆ

**เฉลย:**

```java
import java.util.List;
import java.util.function.Consumer;

public class Exercise1 {
    static <T> void processAll(List<T> list, Consumer<T> action) {
        for (T item : list) {
            action.accept(item);
        }
    }

    public static void main(String[] args) {
        processAll(List.of("a", "b", "c"), s -> System.out.println("Item: " + s));
    }
}
```

**2)** ประกอบ `Predicate<Integer>` สามตัว: เป็นบวก, เป็นเลขคู่, น้อยกว่า 100
เข้าด้วยกันด้วย `and()` แล้วทดสอบด้วยหลายค่า

**เฉลย:**

```java
import java.util.function.Predicate;

public class Exercise2 {
    public static void main(String[] args) {
        Predicate<Integer> isPositive = n -> n > 0;
        Predicate<Integer> isEven = n -> n % 2 == 0;
        Predicate<Integer> lessThan100 = n -> n < 100;

        Predicate<Integer> combined = isPositive.and(isEven).and(lessThan100);

        System.out.println(combined.test(50));  // true
        System.out.println(combined.test(150));  // false (ไม่น้อยกว่า 100)
        System.out.println(combined.test(-4));    // false (ไม่เป็นบวก)
    }
}
```

**3)** อธิบายว่าทำไมควรใช้ `IntPredicate` แทน `Predicate<Integer>` เมื่อทำงาน
ในลูปที่ต้องเรียกใช้หลายล้านครั้ง

**เฉลย**: `Predicate<Integer>` ต้อง**autobox** ค่า `int` เป็น `Integer` object
ทุกครั้งที่เรียก `test()` (ทบทวนปัญหา autoboxing จาก Part 28) ทำให้เกิดการ
สร้าง object จำนวนมากโดยไม่จำเป็นเมื่อเรียกซ้ำหลายล้านครั้ง ส่งผลเสียต่อ
ประสิทธิภาพและเพิ่มภาระให้ garbage collector ในขณะที่ `IntPredicate` รับและ
ทำงานกับ `int` (primitive) โดยตรงตลอดทั้งกระบวนการ ไม่มี autoboxing เกิดขึ้น
เลย ทำให้เร็วกว่าอย่างมีนัยสำคัญในสถานการณ์ที่ต้องเรียกใช้บ่อยมาก

### สรุปเนื้อหา Part 40

- `Function<T,R>` แปลงค่า, `Predicate<T>` ทดสอบเงื่อนไข, `Consumer<T>` ทำ side
  effect, `Supplier<T>` จ่ายค่าแบบ lazy — คือ 4 functional interface หลักของ
  `java.util.function`
- `BiFunction`/`BiConsumer`/`BiPredicate` รับสองพารามิเตอร์
- `UnaryOperator`/`BinaryOperator` คือกรณีพิเศษที่ input/output ชนิดเดียวกัน
- Primitive specialization (`IntPredicate`, `ToIntFunction`, ฯลฯ) หลีกเลี่ยง
  autoboxing เพื่อประสิทธิภาพ
- `andThen`, `compose` ประกอบ Function ต่อกัน, `and`/`or`/`negate` ประกอบ
  Predicate ต่อกัน — เป็นรากฐานสำคัญของ Stream API ที่จะเรียนต่อไปใน Part 41-42

**ต่อไป**: [Part 41 — Stream API เบื้องต้น](./part-041-stream-api-basics.md)
