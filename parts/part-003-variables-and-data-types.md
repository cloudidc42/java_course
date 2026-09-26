# Part 3: ตัวแปร ชนิดข้อมูล และการแปลงชนิดข้อมูล

> ขั้นตอนที่ 21-30 ของหลักสูตร | ระดับ: พื้นฐาน

## สารบัญ

1. ตัวแปรคืออะไร และการประกาศตัวแปร
2. Primitive Data Types ทั้ง 8 ชนิด
3. ช่วงค่าและขนาดหน่วยความจำของแต่ละชนิด
4. Reference Types เบื้องต้น
5. `var` — Local Variable Type Inference (Java 10+)
6. ค่าเริ่มต้น (Default Values) และ Literal
7. การแปลงชนิดข้อมูล (Type Casting): Widening vs Narrowing
8. Constants ด้วย `final`
9. Scope ของตัวแปร
10. แบบฝึกหัดและสรุป

---

## 1. ตัวแปรคืออะไร และการประกาศตัวแปร

**ตัวแปร (Variable)** คือชื่อที่ใช้อ้างอิงถึงพื้นที่หน่วยความจำที่เก็บค่าข้อมูล
Java เป็นภาษาแบบ **Statically Typed** หมายความว่าทุกตัวแปรต้อง**ประกาศชนิดข้อมูล**
ตั้งแต่ตอนเขียนโค้ด และชนิดข้อมูลนั้นจะถูกตรวจสอบตอน compile-time

รูปแบบการประกาศตัวแปร:

```java
ชนิดข้อมูล ชื่อตัวแปร = ค่าเริ่มต้น;

int age = 25;
double price = 199.99;
String name = "Somchai";
boolean isActive = true;
```

สามารถประกาศหลายตัวแปรชนิดเดียวกันในบรรทัดเดียว:

```java
int x = 1, y = 2, z = 3;
```

หรือประกาศก่อน แล้วกำหนดค่าทีหลัง:

```java
int score;   // ประกาศ (declaration)
score = 100; // กำหนดค่า (assignment)
```

## 2. Primitive Data Types ทั้ง 8 ชนิด

Java มีชนิดข้อมูลพื้นฐาน (primitive types) ที่ built-in อยู่ในภาษาทั้งหมด **8 ชนิด**
แบ่งเป็น 4 กลุ่ม:

### กลุ่มจำนวนเต็ม (Integer Types)

```java
byte smallNumber = 100;        // 8-bit  : -128 ถึง 127
short mediumNumber = 30000;    // 16-bit : -32,768 ถึง 32,767
int normalNumber = 2000000000; // 32-bit : -2^31 ถึง 2^31-1 (ค่าเริ่มต้นของ integer)
long bigNumber = 9000000000L;  // 64-bit : -2^63 ถึง 2^63-1 (ต้องมี L ต่อท้าย)
```

### กลุ่มทศนิยม (Floating-Point Types)

```java
float pi = 3.14f;              // 32-bit ความแม่นยำ ~6-7 หลัก (ต้องมี f ต่อท้าย)
double preciseValue = 3.14159265358979; // 64-bit ความแม่นยำสูงกว่า (ค่าเริ่มต้นของทศนิยม)
```

### กลุ่มตัวอักษร (Character Type)

```java
char grade = 'A';              // 16-bit เก็บอักขระ 1 ตัว (ใช้ single quote เท่านั้น)
char unicodeChar = 'A';   // สามารถใช้ unicode escape ได้ (นี่คือ 'A')
```

### กลุ่มบูลีน (Boolean Type)

```java
boolean isJavaFun = true;      // เก็บได้แค่ true หรือ false เท่านั้น
```

## 3. ช่วงค่าและขนาดหน่วยความจำของแต่ละชนิด

| ชนิดข้อมูล | ขนาด | ช่วงค่า | ค่าเริ่มต้น |
|---|---|---|---|
| `byte` | 8 bit (1 byte) | -128 ถึง 127 | 0 |
| `short` | 16 bit (2 byte) | -32,768 ถึง 32,767 | 0 |
| `int` | 32 bit (4 byte) | -2,147,483,648 ถึง 2,147,483,647 | 0 |
| `long` | 64 bit (8 byte) | -9.2×10^18 ถึง 9.2×10^18 | 0L |
| `float` | 32 bit (4 byte) | ~±3.4×10^38 (7 หลักความแม่นยำ) | 0.0f |
| `double` | 64 bit (8 byte) | ~±1.8×10^308 (15 หลักความแม่นยำ) | 0.0d |
| `char` | 16 bit (2 byte) | 0 ถึง 65,535 (Unicode) | '\u0000' |
| `boolean` | 1 bit (ตามทฤษฎี, JVM แต่ละตัวจัดการต่างกัน) | true / false | false |

ตัวอย่างการดูค่าขอบเขตสูงสุด/ต่ำสุดด้วย Wrapper Classes:

```java
public class RangeDemo {
    public static void main(String[] args) {
        System.out.println("int  max: " + Integer.MAX_VALUE);
        System.out.println("int  min: " + Integer.MIN_VALUE);
        System.out.println("long max: " + Long.MAX_VALUE);
        System.out.println("byte max: " + Byte.MAX_VALUE);
        System.out.println("double max: " + Double.MAX_VALUE);
    }
}
```

ผลลัพธ์:

```
int  max: 2147483647
int  min: -2147483648
long max: 9223372036854775807
byte max: 127
double max: 1.7976931348623157E308
```

### Integer Overflow — กับดักที่พบบ่อย

```java
public class OverflowDemo {
    public static void main(String[] args) {
        int maxInt = Integer.MAX_VALUE;
        System.out.println(maxInt);        // 2147483647
        System.out.println(maxInt + 1);    // -2147483648 (!!) เกิด overflow วนกลับไปค่าติดลบ
    }
}
```

นี่คือเหตุผลที่ต้องเลือกชนิดข้อมูลให้เหมาะสมกับขนาดของค่าที่จะใช้งานจริง หากคาดว่าค่า
จะเกิน `int` (เช่นการนับ population โลก หรือ timestamp เป็นมิลลิวินาที) ควรใช้ `long`

## 4. Reference Types เบื้องต้น

นอกจาก primitive types แล้ว Java ยังมี **Reference Types** ซึ่งเก็บ "ที่อยู่อ้างอิง"
ไปยัง object ในหน่วยความจำ (heap) ไม่ใช่ค่าตรง ๆ แบบ primitive

ตัวอย่างที่ใช้บ่อยที่สุดคือ `String`:

```java
String name = "Somchai";      // String เป็น reference type (ไม่ใช่ primitive)
int[] numbers = {1, 2, 3};    // Array ก็เป็น reference type
```

ความแตกต่างสำคัญระหว่าง primitive และ reference:

```java
public class PrimitiveVsReference {
    public static void main(String[] args) {
        // Primitive: คัดลอกค่าโดยตรง
        int a = 10;
        int b = a;  // b ได้ค่า 10 (คนละพื้นที่หน่วยความจำ)
        b = 20;
        System.out.println("a = " + a); // a = 10 (ไม่เปลี่ยนตาม b)

        // Reference: คัดลอกที่อยู่อ้างอิง
        int[] arr1 = {1, 2, 3};
        int[] arr2 = arr1;  // arr2 ชี้ไปที่ array เดียวกับ arr1
        arr2[0] = 99;
        System.out.println("arr1[0] = " + arr1[0]); // arr1[0] = 99 (!) เปลี่ยนตาม arr2
    }
}
```

เนื้อหาเรื่อง reference และ heap memory จะลงลึกอีกครั้งใน Part 11 (Classes and Objects)
และ Part 67 (JVM Internals)

## 5. `var` — Local Variable Type Inference (Java 10+)

ตั้งแต่ Java 10 เราสามารถใช้ `var` แทนการระบุชนิดข้อมูลตรง ๆ ได้ โดย compiler
จะ**อนุมานชนิดข้อมูล**จากค่าทางขวาให้อัตโนมัติ (แต่ยังคง Statically Typed อยู่ ไม่ใช่
dynamic typing แบบ Python/JavaScript)

```java
public class VarDemo {
    public static void main(String[] args) {
        var age = 25;              // อนุมานเป็น int
        var price = 199.99;        // อนุมานเป็น double
        var name = "Somchai";      // อนุมานเป็น String
        var isActive = true;       // อนุมานเป็น boolean

        // age = "text";   // Error! ชนิดข้อมูลถูกล็อคเป็น int แล้วตั้งแต่ compile-time
        System.out.println(age + ", " + price + ", " + name + ", " + isActive);
    }
}
```

**ข้อจำกัดของ `var`:**
- ใช้ได้กับตัวแปร local เท่านั้น (ในเมธอด) ใช้กับ field ระดับคลาสหรือ parameter ไม่ได้
- ต้องกำหนดค่าเริ่มต้นทันทีตอนประกาศ (compiler ต้องมีค่าไปอนุมานชนิด)
- ใช้ไม่ได้กับตัวแปรที่ยังไม่กำหนดค่า เช่น `var x;` (ผิด)

**เมื่อไรควรใช้ `var`**: เมื่อชนิดข้อมูลเห็นชัดเจนจากบริบทอยู่แล้วและช่วยให้โค้ดกระชับขึ้น
เช่น `var list = new ArrayList<String>();` แต่ไม่ควรใช้ถ้าทำให้โค้ดอ่านยากขึ้น

## 6. ค่าเริ่มต้น (Default Values) และ Literal

**Field** ของคลาส (ตัวแปรระดับคลาส) จะมีค่าเริ่มต้นอัตโนมัติถ้าไม่ได้กำหนดค่า
(ตามตารางในหัวข้อ 3) แต่**ตัวแปร local** (ในเมธอด) **ไม่มีค่าเริ่มต้น** ต้องกำหนดค่า
ก่อนใช้งานเสมอ มิฉะนั้น compiler จะ error

```java
public class DefaultValueDemo {
    static int classLevelVar; // field: ได้ค่าเริ่มต้น 0 อัตโนมัติ

    public static void main(String[] args) {
        System.out.println(classLevelVar); // พิมพ์ 0 ได้ปกติ

        int localVar;
        // System.out.println(localVar); // Error: variable might not have been initialized
        localVar = 5;
        System.out.println(localVar); // ต้องกำหนดค่าก่อนถึงจะใช้ได้
    }
}
```

### Literal พิเศษที่ควรรู้

```java
public class LiteralDemo {
    public static void main(String[] args) {
        int decimal = 100;          // เลขฐาน 10 (ปกติ)
        int binary = 0b1100100;     // เลขฐาน 2 (ขึ้นต้น 0b) = 100
        int octal = 0144;           // เลขฐาน 8 (ขึ้นต้น 0) = 100
        int hex = 0x64;             // เลขฐาน 16 (ขึ้นต้น 0x) = 100

        int million = 1_000_000;    // underscore คั่นหลักเพื่อความอ่านง่าย (Java 7+)

        System.out.println(decimal + " " + binary + " " + octal + " " + hex);
        System.out.println(million);
    }
}
```

## 7. การแปลงชนิดข้อมูล (Type Casting)

### Widening Conversion (แปลงขยาย) — ทำอัตโนมัติ ปลอดภัย ไม่เสียข้อมูล

เมื่อแปลงจากชนิดข้อมูลที่มีขนาดเล็กไปใหญ่กว่า Java จะทำให้อัตโนมัติ:

```
byte -> short -> int -> long -> float -> double
                  ↑
                 char
```

```java
public class WideningDemo {
    public static void main(String[] args) {
        int myInt = 100;
        long myLong = myInt;      // int -> long อัตโนมัติ ไม่ต้อง cast
        double myDouble = myLong; // long -> double อัตโนมัติ

        System.out.println(myInt + " " + myLong + " " + myDouble);
    }
}
```

### Narrowing Conversion (แปลงย่อ) — ต้อง cast เอง เสี่ยงข้อมูลหาย

เมื่อแปลงจากชนิดข้อมูลใหญ่ไปเล็กกว่า ต้อง**ระบุการ cast อย่างชัดเจน** (explicit cast)
เพราะอาจทำให้ข้อมูลสูญหายหรือผิดเพี้ยน:

```java
public class NarrowingDemo {
    public static void main(String[] args) {
        double myDouble = 9.99;
        int myInt = (int) myDouble;  // ต้องใส่ (int) ไม่งั้น compile error
        System.out.println(myInt);   // 9 (ทศนิยมถูกตัดทิ้ง ไม่ใช่ปัดเศษ!)

        int bigValue = 300;
        byte myByte = (byte) bigValue; // byte รับได้แค่ -128 ถึง 127
        System.out.println(myByte);    // 44 (!) เกิด overflow เพราะ 300 เกินขอบเขตของ byte
    }
}
```

> **ข้อควรระวัง**: การ cast จาก `double` เป็น `int` คือการ**ตัดทิ้ง (truncate)** ไม่ใช่
> การปัดเศษ (round) ถ้าต้องการปัดเศษให้ใช้ `Math.round()` แทน:

```java
double value = 9.99;
int rounded = (int) Math.round(value); // 10
```

### การแปลง String ↔ ตัวเลข

```java
public class StringConversionDemo {
    public static void main(String[] args) {
        // String -> ตัวเลข
        String numText = "123";
        int parsedInt = Integer.parseInt(numText);
        double parsedDouble = Double.parseDouble("45.67");

        // ตัวเลข -> String
        int number = 999;
        String numberAsText = String.valueOf(number);
        String concatenated = number + "";  // trick ที่ใช้บ่อยแต่ String.valueOf ชัดเจนกว่า

        System.out.println(parsedInt + 1);       // 124
        System.out.println(parsedDouble * 2);    // 91.34
        System.out.println(numberAsText.length()); // 3
    }
}
```

## 8. Constants ด้วย `final`

ถ้าต้องการตัวแปรที่ค่าไม่สามารถเปลี่ยนแปลงได้หลังจากกำหนดค่าครั้งแรก ให้ใช้ keyword
`final` (รายละเอียดเชิงลึกอยู่ใน Part 17):

```java
public class ConstantDemo {
    static final double TAX_RATE = 0.07; // ตามธรรมเนียม constant ใช้ UPPER_SNAKE_CASE

    public static void main(String[] args) {
        final int MAX_ATTEMPTS = 3;
        System.out.println("อัตราภาษี: " + TAX_RATE);
        System.out.println("จำนวนครั้งสูงสุด: " + MAX_ATTEMPTS);

        // MAX_ATTEMPTS = 5; // Error! cannot assign a value to final variable
    }
}
```

## 9. Scope ของตัวแปร

**Scope** คือขอบเขตที่ตัวแปรนั้นสามารถถูกมองเห็นและใช้งานได้ กำหนดโดยตำแหน่งของ
เครื่องหมายปีกกา `{ }` ที่ตัวแปรถูกประกาศอยู่ภายใน

```java
public class ScopeDemo {
    static int classField = 100; // Class-level scope: มองเห็นได้ทั้งคลาส

    public static void main(String[] args) {
        int methodVar = 10; // Method-level scope: มองเห็นได้เฉพาะใน main()

        if (methodVar > 5) {
            int blockVar = 20; // Block-level scope: มองเห็นได้เฉพาะใน if block นี้
            System.out.println(blockVar);      // OK
            System.out.println(methodVar);     // OK - มองเห็นตัวแปรจาก scope นอกได้
            System.out.println(classField);    // OK
        }

        // System.out.println(blockVar); // Error! blockVar อยู่นอก scope แล้ว

        for (int i = 0; i < 3; i++) { // i มี scope แค่ใน for loop นี้
            System.out.println("i = " + i);
        }
        // System.out.println(i); // Error! i อยู่นอก scope ของ for loop แล้ว
    }
}
```

**หลักการจำง่าย ๆ**: ตัวแปรจะมีชีวิตอยู่ตั้งแต่จุดที่ประกาศ จนถึงปีกกาปิด `}` ที่ตรงกัน
ของ block ที่มันถูกประกาศอยู่ภายใน เมื่อออกจาก block นั้น ตัวแปรจะถูกทำลายทันที

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียนโปรแกรมที่ประกาศตัวแปรทั้ง 8 primitive types พร้อมค่าตัวอย่าง แล้วพิมพ์
ค่าและชนิดข้อมูลออกมาให้ครบทุกตัว

**เฉลย:**

```java
public class AllPrimitivesDemo {
    public static void main(String[] args) {
        byte b = 10;
        short s = 1000;
        int i = 100000;
        long l = 10000000000L;
        float f = 3.14f;
        double d = 3.14159;
        char c = 'J';
        boolean bool = true;

        System.out.println("byte: " + b);
        System.out.println("short: " + s);
        System.out.println("int: " + i);
        System.out.println("long: " + l);
        System.out.println("float: " + f);
        System.out.println("double: " + d);
        System.out.println("char: " + c);
        System.out.println("boolean: " + bool);
    }
}
```

**2)** ทำนายผลลัพธ์ของโค้ดต่อไปนี้ก่อนรันจริง:

```java
public class Predict {
    public static void main(String[] args) {
        int a = 127;
        byte b = (byte) (a + 1);
        System.out.println(b);
    }
}
```

**เฉลย**: ผลลัพธ์คือ `-128` เพราะ `a + 1` = 128 ซึ่งเกินขอบเขตของ `byte` (สูงสุด 127)
จึง overflow วนกลับไปที่ค่าต่ำสุด (-128)

**3)** เขียนโปรแกรมรับสตริง `"3.14"` แปลงเป็น `double` แล้วบวกกับ `int` ค่า `10`
พิมพ์ผลลัพธ์

**เฉลย:**

```java
public class Exercise3 {
    public static void main(String[] args) {
        String text = "3.14";
        double value = Double.parseDouble(text);
        int number = 10;
        double result = value + number;
        System.out.println("ผลลัพธ์: " + result); // 13.14
    }
}
```

### สรุปเนื้อหา Part 3

- Java มี primitive types 8 ชนิด: `byte`, `short`, `int`, `long`, `float`, `double`,
  `char`, `boolean` แต่ละชนิดมีขนาดและช่วงค่าต่างกัน
- Reference types (เช่น `String`, array) เก็บที่อยู่อ้างอิง ไม่ใช่ค่าตรง ๆ
- `var` ช่วยลดความยืดยาวของโค้ด แต่ยังคง static typing เหมือนเดิม
- Widening conversion ทำอัตโนมัติ, Narrowing conversion ต้อง cast เอง และเสี่ยงข้อมูลหาย
- `final` ใช้สร้างค่าคงที่ที่เปลี่ยนแปลงไม่ได้
- Scope กำหนดโดยตำแหน่งของ `{ }` — ตัวแปรมีชีวิตอยู่แค่ใน block ที่ประกาศเท่านั้น

**ต่อไป**: [Part 4 — ตัวดำเนินการ (Operators) ทั้งหมดใน Java](./part-004-operators.md)
