# Part 8: Methods และการส่งผ่านพารามิเตอร์

> ขั้นตอนที่ 71-80 ของหลักสูตร | ระดับ: พื้นฐาน

## สารบัญ

1. Method คืออะไร ทำไมต้องแบ่งโค้ดเป็นเมธอด
2. โครงสร้างของ Method (Method Signature)
3. Parameters และ Return Values
4. Pass by Value คือหัวใจสำคัญที่สุดของ Java
5. Method Overloading
6. Varargs (จำนวนพารามิเตอร์ไม่แน่นอน)
7. Recursion เบื้องต้น (การเรียกซ้ำ)
8. Static Method vs Instance Method (เบื้องต้น)
9. หลักการออกแบบเมธอดที่ดี
10. แบบฝึกหัดและสรุป

---

## 1. Method คืออะไร ทำไมต้องแบ่งโค้ดเป็นเมธอด

**Method (เมธอด)** คือกลุ่มคำสั่งที่ถูกตั้งชื่อและเรียกใช้ซ้ำได้ ช่วยให้โค้ด:

- **ไม่ซ้ำซ้อน (DRY - Don't Repeat Yourself)**: เขียนตรรกะครั้งเดียว เรียกใช้ได้หลายที่
- **อ่านง่าย**: แบ่งปัญหาใหญ่เป็นส่วนย่อยที่มีชื่อสื่อความหมาย
- **ทดสอบง่าย**: ทดสอบแต่ละเมธอดแยกกันได้ (unit testing — Part 58)
- **บำรุงรักษาง่าย**: แก้ไข logic ที่จุดเดียว มีผลทุกที่ที่เรียกใช้

```java
public class WithoutMethodDemo {
    public static void main(String[] args) {
        // ไม่มีเมธอด: ต้องเขียนตรรกะเดิมซ้ำ ๆ ทุกครั้งที่ต้องการ
        int a = 5, b = 10;
        int max1 = (a > b) ? a : b;

        int c = 20, d = 8;
        int max2 = (c > d) ? c : d;

        System.out.println(max1 + " " + max2);
    }
}
```

```java
public class WithMethodDemo {
    // มีเมธอด: เขียนตรรกะครั้งเดียว เรียกใช้ซ้ำได้ไม่จำกัด
    static int findMax(int x, int y) {
        return (x > y) ? x : y;
    }

    public static void main(String[] args) {
        int max1 = findMax(5, 10);
        int max2 = findMax(20, 8);
        System.out.println(max1 + " " + max2);
    }
}
```

## 2. โครงสร้างของ Method (Method Signature)

```java
[access modifier] [static] [final] ชนิดข้อมูลที่คืนค่า ชื่อเมธอด(พารามิเตอร์) [throws ...] {
    // เนื้อหาเมธอด
    return ค่าที่คืน; // ถ้าชนิดข้อมูลไม่ใช่ void
}
```

ตัวอย่างครบทุกส่วนประกอบ:

```java
public class MethodStructureDemo {
    //  ↓access  ↓return type  ↓ชื่อ    ↓พารามิเตอร์
    public      double        calculateArea(double width, double height) {
        double area = width * height; // เนื้อหาเมธอด (method body)
        return area;                  // return statement
    }

    // เมธอดที่ไม่คืนค่า ใช้ void
    public void printGreeting(String name) {
        System.out.println("สวัสดี, " + name);
        // ไม่มี return หรือมี "return;" เฉย ๆ ได้ (ออกจากเมธอดก่อนกำหนด)
    }

    public static void main(String[] args) {
        MethodStructureDemo demo = new MethodStructureDemo();
        double area = demo.calculateArea(5.0, 3.0);
        System.out.println("พื้นที่: " + area);
        demo.printGreeting("Somchai");
    }
}
```

**Method Signature** ประกอบด้วย **ชื่อเมธอด + ชนิดและลำดับของพารามิเตอร์** เท่านั้น
(ไม่รวม return type หรือชื่อพารามิเตอร์) — นี่คือกุญแจสำคัญของ Method Overloading
ที่จะอธิบายในหัวข้อ 5

## 3. Parameters และ Return Values

```java
public class ParametersDemo {
    // ไม่มีพารามิเตอร์ ไม่คืนค่า
    static void sayHello() {
        System.out.println("Hello!");
    }

    // มีพารามิเตอร์ 1 ตัว ไม่คืนค่า
    static void sayHelloTo(String name) {
        System.out.println("Hello, " + name + "!");
    }

    // มีหลายพารามิเตอร์ คืนค่า
    static double calculateBMI(double weightKg, double heightM) {
        return weightKg / (heightM * heightM);
    }

    // return ได้หลายจุด (แต่ต้องมี path ที่ return เสมอ ไม่งั้น compile error)
    static String classifyBMI(double bmi) {
        if (bmi < 18.5) {
            return "น้ำหนักน้อย";
        } else if (bmi < 25) {
            return "น้ำหนักปกติ";
        } else if (bmi < 30) {
            return "น้ำหนักเกิน";
        } else {
            return "โรคอ้วน";
        }
        // ไม่ต้องมี return หลังจากนี้ เพราะทุกทางเลือกข้างบน cover หมดแล้ว (exhaustive)
    }

    public static void main(String[] args) {
        sayHello();
        sayHelloTo("Alice");

        double bmi = calculateBMI(65, 1.70);
        System.out.printf("BMI: %.2f (%s)%n", bmi, classifyBMI(bmi));
    }
}
```

### Early Return (การคืนค่าก่อนกำหนด)

```java
public class EarlyReturnDemo {
    static String validateAge(int age) {
        if (age < 0) {
            return "อายุติดลบไม่ได้"; // ออกจากเมธอดทันที ไม่ทำโค้ดด้านล่างต่อ
        }
        if (age > 150) {
            return "อายุเกินขอบเขตที่เป็นไปได้";
        }
        return "อายุถูกต้อง: " + age;
    }

    public static void main(String[] args) {
        System.out.println(validateAge(-5));
        System.out.println(validateAge(200));
        System.out.println(validateAge(25));
    }
}
```

## 4. Pass by Value คือหัวใจสำคัญที่สุดของ Java

**Java ส่งพารามิเตอร์แบบ Pass by Value เสมอ ไม่มีข้อยกเว้น** — นี่คือแนวคิดที่มือใหม่
เข้าใจผิดบ่อยที่สุด เพราะพฤติกรรมของ primitive และ reference type ดู "เหมือน" ต่างกัน
แต่จริง ๆ แล้วกลไกเบื้องหลังเหมือนกันทุกประการ

### กรณี Primitive Type: คัดลอกค่าจริง ๆ

```java
public class PassByValuePrimitive {
    static void tryToModify(int number) {
        number = 999; // แก้ไขแค่ "สำเนา" ในพารามิเตอร์ ไม่กระทบตัวแปรต้นฉบับ
    }

    public static void main(String[] args) {
        int original = 10;
        tryToModify(original);
        System.out.println(original); // ยังคงเป็น 10 (ไม่เปลี่ยน)
    }
}
```

### กรณี Reference Type: คัดลอก "ที่อยู่อ้างอิง" (ไม่ใช่ตัว object)

```java
public class PassByValueReference {
    static void modifyArray(int[] arr) {
        arr[0] = 999; // แก้ไขผ่าน reference ที่ชี้ไปยัง object เดียวกับต้นฉบับ -> กระทบจริง
    }

    static void reassignArray(int[] arr) {
        arr = new int[]{1, 1, 1}; // แก้ไขแค่ "สำเนาของ reference" ให้ชี้ไป object ใหม่
                                   // ไม่กระทบตัวแปรต้นฉบับที่อยู่นอกเมธอด
    }

    public static void main(String[] args) {
        int[] numbers = {10, 20, 30};

        modifyArray(numbers);
        System.out.println(java.util.Arrays.toString(numbers)); // [999, 20, 30] เปลี่ยนจริง!

        reassignArray(numbers);
        System.out.println(java.util.Arrays.toString(numbers)); // [999, 20, 30] ไม่เปลี่ยน!
    }
}
```

**คำอธิบายเชิงลึก**: ตัวแปร reference type เก็บ "ที่อยู่" ของ object ใน heap ไม่ใช่
object เอง เมื่อส่งเข้าเมธอด Java จะ**คัดลอกที่อยู่นั้น** (ไม่ใช่คัดลอก object) ดังนั้น:

- ถ้าเมธอดแก้ไข**เนื้อหาภายใน**ของ object ผ่าน reference ที่ได้รับมา (เช่น
  `arr[0] = 999`) → **มีผลกระทบ**ต่อ object ต้นฉบับ เพราะทั้งสอง reference ชี้ไปยัง
  object เดียวกันในหน่วยความจำ
- ถ้าเมธอด**เปลี่ยน reference ให้ชี้ไปที่ object ใหม่**ทั้งหมด (เช่น `arr = new
  int[]{...}`) → **ไม่มีผลกระทบ**ต่อตัวแปรต้นฉบับ เพราะแก้ไขแค่ "สำเนาของที่อยู่"
  ที่อยู่ในพารามิเตอร์ ตัวแปรต้นฉบับข้างนอกยังคงชี้ไปที่ object เดิม

```
สถานะก่อนเรียกเมธอด:
  numbers (ตัวแปรใน main)  ---> [10, 20, 30]  (object A ใน heap)

ระหว่างเรียก modifyArray(numbers):
  numbers (ตัวแปรใน main)  ---> [10, 20, 30]  (object A)
  arr (พารามิเตอร์ในเมธอด)  ---↗   (arr คัดลอกที่อยู่ ชี้ไป object A ตัวเดียวกัน)
  arr[0] = 999  ->  แก้ไข object A โดยตรง -> numbers เห็นการเปลี่ยนแปลงด้วย!

ระหว่างเรียก reassignArray(numbers):
  numbers (ตัวแปรใน main)  ---> [999, 20, 30]  (object A - ไม่เปลี่ยน)
  arr (พารามิเตอร์ในเมธอด)  -----------> [1, 1, 1]  (object B - ใหม่)
  arr = new int[]{1,1,1}  ->  arr ชี้ไป object ใหม่ (B) แต่ numbers ยังชี้ไป A เหมือนเดิม
```

## 5. Method Overloading

**Method Overloading** คือการมีเมธอดชื่อเดียวกันหลายตัวในคลาสเดียวกัน โดยมี
**พารามิเตอร์แตกต่างกัน** (จำนวน, ชนิด, หรือลำดับ) — Java จะเลือกเวอร์ชันที่ตรงกับ
argument ที่ส่งเข้ามาให้อัตโนมัติตอน compile-time (เรียกว่า **compile-time
polymorphism** หรือ **static binding**)

```java
public class OverloadingDemo {
    // Overload กันด้วยจำนวนพารามิเตอร์
    static int add(int a, int b) {
        return a + b;
    }

    static int add(int a, int b, int c) {
        return a + b + c;
    }

    // Overload กันด้วยชนิดพารามิเตอร์
    static double add(double a, double b) {
        return a + b;
    }

    // Overload กันด้วยลำดับของชนิดพารามิเตอร์
    static String combine(String s, int n) {
        return s + n;
    }

    static String combine(int n, String s) {
        return n + s;
    }

    public static void main(String[] args) {
        System.out.println(add(1, 2));         // เรียก add(int, int) -> 3
        System.out.println(add(1, 2, 3));       // เรียก add(int, int, int) -> 6
        System.out.println(add(1.5, 2.5));      // เรียก add(double, double) -> 4.0
        System.out.println(combine("Score: ", 100)); // "Score: 100"
        System.out.println(combine(100, " points")); // "100 points"
    }
}
```

**ข้อควรระวัง**: การ overload ที่แตกต่างกันแค่**ชื่อพารามิเตอร์** หรือ**return type
เพียงอย่างเดียว** จะ compile error เพราะ signature ไม่ต่างกันจริง:

```java
// Error! signature ซ้ำกัน (ต่างกันแค่ return type ไม่นับเป็นการ overload)
// static int calculate(int a, int b) { return a + b; }
// static double calculate(int a, int b) { return a + b; }
```

## 6. Varargs (จำนวนพารามิเตอร์ไม่แน่นอน)

**Varargs** (Variable Arguments) ใช้เมื่อไม่ทราบจำนวนพารามิเตอร์ล่วงหน้า เขียนด้วย
`...` ต่อท้ายชนิดข้อมูล — ภายในเมธอด varargs จะถูกมองเป็น array ธรรมดา

```java
public class VarargsDemo {
    static int sum(int... numbers) { // รับได้ 0, 1, 2, ... ถึง N ตัว
        int total = 0;
        for (int n : numbers) {
            total += n;
        }
        return total;
    }

    // varargs ต้องเป็นพารามิเตอร์ตัวสุดท้ายเสมอ ถ้ามีพารามิเตอร์อื่นด้วย
    static void printWithPrefix(String prefix, String... items) {
        for (String item : items) {
            System.out.println(prefix + item);
        }
    }

    public static void main(String[] args) {
        System.out.println(sum());           // 0 (ไม่ส่งอะไรเลยก็ได้)
        System.out.println(sum(1));           // 1
        System.out.println(sum(1, 2, 3));     // 6
        System.out.println(sum(1, 2, 3, 4, 5)); // 15

        // ส่ง array ตรง ๆ ก็ได้เช่นกัน
        int[] arr = {10, 20, 30};
        System.out.println(sum(arr));         // 60

        printWithPrefix("- ", "แอปเปิ้ล", "กล้วย", "ส้ม");
    }
}
```

**ตัวอย่างที่คุ้นเคย**: `String.format()`, `System.out.printf()`, และ
`List.of()` ล้วนใช้ varargs ภายใน

## 7. Recursion เบื้องต้น (การเรียกซ้ำ)

**Recursion** คือเมธอดที่**เรียกตัวเองซ้ำ** ๆ จนกว่าจะถึงเงื่อนไขหยุด (**base case**)
เหมาะกับปัญหาที่แบ่งเป็นปัญหาย่อยที่มีลักษณะเดียวกันได้ (จะลงลึกอีกครั้งใน Part 29)

```java
public class RecursionDemo {
    // หาแฟกทอเรียล: n! = n * (n-1) * (n-2) * ... * 1
    static long factorial(int n) {
        if (n <= 1) {          // base case: จุดหยุดการเรียกซ้ำ (สำคัญที่สุด!)
            return 1;
        }
        return n * factorial(n - 1); // recursive case: เรียกตัวเองด้วยปัญหาที่เล็กลง
    }

    public static void main(String[] args) {
        System.out.println(factorial(5)); // 5*4*3*2*1 = 120

        // แกะรอยการทำงาน:
        // factorial(5) = 5 * factorial(4)
        //              = 5 * (4 * factorial(3))
        //              = 5 * (4 * (3 * factorial(2)))
        //              = 5 * (4 * (3 * (2 * factorial(1))))
        //              = 5 * (4 * (3 * (2 * 1)))
        //              = 120
    }
}
```

**ข้อควรระวังสำคัญ**: หากไม่มี base case หรือ base case ไปไม่ถึง จะเกิด
**infinite recursion** และท้ายที่สุดจะเกิด `StackOverflowError`:

```java
public class StackOverflowDemo {
    static void infiniteRecursion() {
        infiniteRecursion(); // ไม่มี base case เลย -> เรียกตัวเองไม่หยุด
    }

    public static void main(String[] args) {
        try {
            infiniteRecursion();
        } catch (StackOverflowError e) {
            System.out.println("เกิด StackOverflowError เพราะ call stack เต็ม!");
        }
    }
}
```

## 8. Static Method vs Instance Method (เบื้องต้น)

เนื้อหาลึกอยู่ใน Part 11-17 แต่ควรเข้าใจความแตกต่างพื้นฐานตั้งแต่ตอนนี้:

```java
public class StaticVsInstanceDemo {
    // Static method: เรียกผ่านชื่อคลาสได้เลย ไม่ต้องสร้าง object
    static void staticMethod() {
        System.out.println("นี่คือ static method");
    }

    // Instance method: ต้องสร้าง object ก่อนถึงจะเรียกได้
    void instanceMethod() {
        System.out.println("นี่คือ instance method");
    }

    public static void main(String[] args) {
        staticMethod(); // เรียกได้ตรง ๆ (main เองก็เป็น static)

        StaticVsInstanceDemo obj = new StaticVsInstanceDemo(); // ต้องสร้าง object ก่อน
        obj.instanceMethod();
    }
}
```

**กฎสำคัญ**: static method **เรียก instance method ตรง ๆ ไม่ได้** (เพราะยังไม่มี
object) แต่ instance method เรียก static method ได้เสมอ (เพราะ static มีอยู่แล้ว
ตั้งแต่คลาสถูกโหลด ไม่ต้องรอสร้าง object)

## 9. หลักการออกแบบเมธอดที่ดี

1. **Single Responsibility**: เมธอดหนึ่งควรทำหน้าที่เดียวให้ดี ไม่ทำหลายอย่างปนกัน
2. **ตั้งชื่อสื่อความหมาย**: `calculateTotalPrice()` ดีกว่า `calc()` หรือ `doStuff()`
3. **เมธอดควรสั้น**: ถ้ายาวเกิน 20-30 บรรทัด ควรพิจารณาแบ่งเป็นเมธอดย่อย
4. **จำนวนพารามิเตอร์ไม่ควรเยอะเกินไป**: ถ้าเกิน 3-4 ตัว ควรพิจารณาห่อเป็น object แทน
5. **หลีกเลี่ยง side effect ที่ไม่คาดคิด**: เมธอดควรทำในสิ่งที่ชื่อบอกไว้เท่านั้น

```java
// ไม่ดี: ชื่อบอกว่า "get" แต่กลับไปแก้ไขข้อมูลด้วย (side effect ที่ไม่คาดคิด)
static int getBalance(Account account) {
    account.balance -= 10; // ผิดหลักการ! ไม่ควรแก้ไขข้อมูลในเมธอดที่ชื่อว่า "get"
    return account.balance;
}

// ดี: แยกหน้าที่ให้ชัดเจน ชื่อตรงกับสิ่งที่ทำ
static int getBalance(Account account) {
    return account.balance;
}

static void deductFee(Account account, int fee) {
    account.balance -= fee;
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียนเมธอด overload ชื่อ `area` ที่คำนวณพื้นที่ได้ 3 แบบ: สี่เหลี่ยม
(กว้าง x ยาว), วงกลม (รัศมี), และสามเหลี่ยม (ฐาน x สูง / 2)

**เฉลย:**

```java
public class Exercise1 {
    static double area(double width, double length) {
        return width * length;
    }

    static double area(double radius) {
        return Math.PI * radius * radius;
    }

    static double area(double base, double height, boolean isTriangle) {
        return base * height / 2;
    }

    public static void main(String[] args) {
        System.out.println(area(4, 5));           // สี่เหลี่ยม: 20.0
        System.out.println(area(3));               // วงกลม: 28.27...
        System.out.println(area(6, 4, true));       // สามเหลี่ยม: 12.0
    }
}
```

**2)** เขียนเมธอด `findMax` แบบ varargs ที่หาค่ามากที่สุดจากจำนวนเท่าไรก็ได้

**เฉลย:**

```java
public class Exercise2 {
    static int findMax(int... numbers) {
        if (numbers.length == 0) {
            throw new IllegalArgumentException("ต้องมีอย่างน้อย 1 ค่า");
        }
        int max = numbers[0];
        for (int n : numbers) {
            if (n > max) max = n;
        }
        return max;
    }

    public static void main(String[] args) {
        System.out.println(findMax(3, 7, 2, 9, 4)); // 9
    }
}
```

**3)** เขียนเมธอด recursion หาผลรวมของ 1 ถึง n

**เฉลย:**

```java
public class Exercise3 {
    static int sumUpTo(int n) {
        if (n <= 0) {
            return 0;
        }
        return n + sumUpTo(n - 1);
    }

    public static void main(String[] args) {
        System.out.println(sumUpTo(10)); // 55
    }
}
```

### สรุปเนื้อหา Part 8

- Method ช่วยลดโค้ดซ้ำซ้อน เพิ่มความอ่านง่ายและบำรุงรักษาโปรแกรม
- **Java ส่งพารามิเตอร์แบบ Pass by Value เสมอ** — สำหรับ reference type คือคัดลอก
  "ที่อยู่อ้างอิง" ไม่ใช่ตัว object เอง (แต่แก้ไขเนื้อหาผ่าน reference นั้นมีผลจริง)
- Method Overloading คือมีชื่อเมธอดเดียวกันแต่พารามิเตอร์ต่างกัน เลือกใช้ตัวที่ตรงกัน
  ตอน compile-time
- Varargs (`...`) ใช้เมื่อไม่ทราบจำนวนพารามิเตอร์ล่วงหน้า
- Recursion ต้องมี base case เสมอ ไม่งั้นจะเกิด `StackOverflowError`
- ออกแบบเมธอดให้ทำหน้าที่เดียว ชื่อสื่อความหมาย และไม่มี side effect ที่ไม่คาดคิด

**ต่อไป**: [Part 9 — String, StringBuilder, StringBuffer](./part-009-strings.md)
