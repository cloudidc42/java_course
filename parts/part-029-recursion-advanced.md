# Part 29: Recursion ขั้นสูง: Backtracking, Memoization

> ขั้นตอนที่ 281-290 ของหลักสูตร | ระดับ: กลาง-สูง

## สารบัญ

1. ทบทวน Recursion และ Call Stack
2. Head Recursion vs Tail Recursion
3. ปัญหา Exponential Time: Fibonacci แบบ Naive Recursion
4. Memoization: แคชผลลัพธ์เพื่อลดการคำนวณซ้ำ
5. Dynamic Programming เบื้องต้น (Top-down vs Bottom-up)
6. Backtracking คืออะไร
7. ตัวอย่าง Backtracking: N-Queens Problem
8. ตัวอย่าง Backtracking: Generate All Permutations
9. ตัวอย่าง Backtracking: Subset Sum
10. แบบฝึกหัดและสรุป

---

## 1. ทบทวน Recursion และ Call Stack

ทบทวนจาก Part 8: Recursion คือเมธอดที่เรียกตัวเอง ทุกครั้งที่เรียกจะมีการสร้าง
**stack frame** ใหม่บน **call stack** ของ thread เก็บตัวแปร local และ return
address — ถ้า recursion ลึกเกินไปจะเกิด `StackOverflowError`

```java
public class CallStackVisualizationDemo {
    static int factorial(int n) {
        System.out.println("เรียก factorial(" + n + ")"); // เมื่อลงไป (going down)
        if (n <= 1) return 1;
        int result = n * factorial(n - 1);
        System.out.println("คืนค่า factorial(" + n + ") = " + result); // เมื่อกลับขึ้นมา (coming back up)
        return result;
    }

    public static void main(String[] args) {
        factorial(4);
        // สังเกตลำดับ: เรียกลงไปจนถึง base case ก่อน แล้วค่อย "ม้วนกลับ" คำนวณจากล่างขึ้นบน
    }
}
```

```
Call Stack (แต่ละกล่องคือ stack frame หนึ่งอัน):

factorial(4)
  └─ factorial(3)
       └─ factorial(2)
            └─ factorial(1)  <- base case, return 1
       └─ คืนค่า 2 * 1 = 2
  └─ คืนค่า 3 * 2 = 6
คืนค่า 4 * 6 = 24
```

## 2. Head Recursion vs Tail Recursion

**Head Recursion**: การเรียกตัวเองอยู่**ต้น**เมธอด, มี logic ทำงานต่อ**หลัง**
การเรียกกลับมา (เช่น `factorial` ข้างบน ต้องรอผลจาก `factorial(n-1)` ก่อนคูณ)

**Tail Recursion**: การเรียกตัวเองเป็น**คำสั่งสุดท้าย**ของเมธอด ไม่มี logic
เหลือทำต่อหลังจากนั้น:

```java
public class TailRecursionDemo {
    // Tail recursion: การเรียกตัวเองเป็นคำสั่งสุดท้าย ไม่มีอะไรทำต่อหลังจากนั้น
    static int factorialTail(int n, int accumulator) {
        if (n <= 1) return accumulator;
        return factorialTail(n - 1, n * accumulator); // ส่งผลลัพธ์สะสมไปเรื่อย ๆ
    }

    public static void main(String[] args) {
        System.out.println(factorialTail(5, 1)); // 120
    }
}
```

**ข้อควรรู้สำคัญ**: หลายภาษา (เช่น Scala, Scheme) มี **Tail Call Optimization**
ที่แปลง tail recursion ให้กลายเป็น loop ภายใน ไม่กิน stack เพิ่ม แต่
**Java ไม่มี Tail Call Optimization** (JVM ยังไม่รองรับ ณ ปัจจุบัน) ดังนั้น
tail recursion ใน Java ก็ยังเสี่ยง `StackOverflowError` เหมือนเดิมถ้า n ใหญ่มาก
— ถ้าต้องการประสิทธิภาพสูงสุดในกรณีนี้ ควรแปลงเป็น loop ปกติแทน

## 3. ปัญหา Exponential Time: Fibonacci แบบ Naive Recursion

```java
public class NaiveFibonacciDemo {
    static long fibonacci(int n) {
        if (n <= 1) return n;
        return fibonacci(n - 1) + fibonacci(n - 2); // เรียกซ้ำ 2 ครั้งทุกระดับ!
    }

    public static void main(String[] args) {
        long start = System.currentTimeMillis();
        System.out.println(fibonacci(40)); // ใช้เวลานานมาก! (หลายวินาที)
        System.out.println("ใช้เวลา: " + (System.currentTimeMillis() - start) + " ms");
    }
}
```

```
ปัญหา: fibonacci(5) คำนวณ fibonacci(3) ซ้ำถึง 2 ครั้ง, fibonacci(2) ซ้ำถึง 3 ครั้ง!

                    fib(5)
                  /        \
              fib(4)         fib(3)
             /     \         /     \
         fib(3)   fib(2)  fib(2)   fib(1)
        /    \    /   \    /  \
    fib(2) fib(1) fib(1) fib(0) ...

Time Complexity: O(2^n) - เติบโตแบบทวีคูณ ช้ามากเมื่อ n สูงขึ้น
```

## 4. Memoization: แคชผลลัพธ์เพื่อลดการคำนวณซ้ำ

**Memoization** คือเทคนิค**เก็บผลลัพธ์ที่คำนวณไปแล้วไว้ใน cache** (มักใช้
`HashMap` — ทบทวนจาก Part 24) เพื่อไม่ต้องคำนวณซ้ำ ลด time complexity จาก
**O(2^n)** เหลือแค่ **O(n)**

```java
import java.util.HashMap;
import java.util.Map;

public class MemoizedFibonacciDemo {
    static Map<Integer, Long> cache = new HashMap<>();

    static long fibonacci(int n) {
        if (n <= 1) return n;

        if (cache.containsKey(n)) { // ถ้าคำนวณไปแล้ว ดึงจาก cache ทันที ไม่ต้องคำนวณซ้ำ
            return cache.get(n);
        }

        long result = fibonacci(n - 1) + fibonacci(n - 2);
        cache.put(n, result); // เก็บผลลัพธ์ไว้ใน cache สำหรับการเรียกครั้งต่อไป
        return result;
    }

    public static void main(String[] args) {
        long start = System.currentTimeMillis();
        System.out.println(fibonacci(40)); // เร็วมาก! (เสี้ยววินาที)
        System.out.println("ใช้เวลา: " + (System.currentTimeMillis() - start) + " ms");

        System.out.println(fibonacci(50)); // คำนวณได้ทันที แม้ naive recursion จะทำไม่ได้ในเวลาที่สมเหตุสมผล
    }
}
```

## 5. Dynamic Programming เบื้องต้น (Top-down vs Bottom-up)

Memoization คือ **Dynamic Programming แบบ Top-down** (เริ่มจากปัญหาใหญ่ แตกย่อย
ลงไป แคชผลลัพธ์) อีกแบบคือ **Bottom-up** (เริ่มจากปัญหาเล็กสุด ไล่คำนวณขึ้นไป
ทีละขั้น โดยไม่ใช้ recursion เลย)

```java
public class BottomUpFibonacciDemo {
    static long fibonacci(int n) {
        if (n <= 1) return n;

        long[] dp = new long[n + 1]; // เก็บผลลัพธ์ของทุกขั้นตอนย่อยไว้ใน array
        dp[0] = 0;
        dp[1] = 1;

        for (int i = 2; i <= n; i++) {
            dp[i] = dp[i - 1] + dp[i - 2]; // สร้างจากผลลัพธ์ที่คำนวณไปแล้วก่อนหน้า (ไม่มี recursion เลย)
        }

        return dp[n];
    }

    public static void main(String[] args) {
        System.out.println(fibonacci(40)); // เร็วเท่ากับ memoization และไม่เสี่ยง StackOverflowError เลย
    }
}
```

**เปรียบเทียบ Top-down (Memoization) vs Bottom-up (Tabulation)**:

| ลักษณะ | Top-down (Memoization) | Bottom-up (Tabulation) |
|---|---|---|
| ทิศทางการคำนวณ | ปัญหาใหญ่ → เล็ก (recursion) | ปัญหาเล็ก → ใหญ่ (loop) |
| ความเสี่ยง Stack Overflow | มี (ยังใช้ recursion) | ไม่มี (ไม่ใช้ recursion เลย) |
| เขียนง่ายกว่า | มักเขียนง่ายกว่า เข้าใจตรงไปตรงมา | ต้องคิดลำดับการคำนวณล่วงหน้า |
| Space (บางกรณี) | เก็บทุกค่าใน cache | สามารถ optimize ใช้แค่ 2 ตัวแปรได้ (ดูตัวอย่างด้านล่าง) |

```java
public class OptimizedFibonacciDemo {
    // Bottom-up ที่ optimize หน่วยความจำ: ไม่ต้องเก็บทั้ง array เพราะใช้แค่ 2 ค่าล่าสุด
    static long fibonacci(int n) {
        if (n <= 1) return n;
        long prev2 = 0, prev1 = 1;
        for (int i = 2; i <= n; i++) {
            long current = prev1 + prev2;
            prev2 = prev1;
            prev1 = current;
        }
        return prev1;
    }

    public static void main(String[] args) {
        System.out.println(fibonacci(50)); // O(n) time, O(1) space - เหมาะที่สุด
    }
}
```

## 6. Backtracking คืออะไร

**Backtracking** คือเทคนิคการแก้ปัญหาด้วยการ**ลองทุกความเป็นไปได้แบบมีระบบ**
(systematic trial and error) — เมื่อพบว่าทางที่เลือกไปไม่สามารถนำไปสู่คำตอบได้
จะ**ถอยกลับ (backtrack)** ไปลองทางเลือกอื่น เหมาะกับปัญหาประเภท **"หาคำตอบที่
เป็นไปได้ทั้งหมด"** หรือ **"หาว่ามีคำตอบหรือไม่"**

**รูปแบบทั่วไปของ Backtracking**:

```java
public class BacktrackingTemplate {
    static void backtrack(/* state ปัจจุบัน */) {
        if (/* พบคำตอบที่สมบูรณ์ */) {
            // บันทึกหรือประมวลผลคำตอบ
            return;
        }

        for (/* ทุกตัวเลือกที่เป็นไปได้ในขั้นตอนนี้ */) {
            if (/* ตัวเลือกนี้ถูกต้องตามเงื่อนไข */) {
                // เลือกตัวเลือกนี้ (make a choice)
                // backtrack(state ใหม่); // ลองต่อไปข้างหน้า
                // ยกเลิกตัวเลือกนี้ (undo the choice) <- นี่คือ "backtrack" ตัวจริง
            }
        }
    }
}
```

## 7. ตัวอย่าง Backtracking: N-Queens Problem

โจทย์คลาสสิก: วางราชินี (Queen) N ตัวบนกระดานหมากรุก N×N โดยไม่ให้ตัวใดกินกันได้
(ห้ามอยู่แถวเดียวกัน, คอลัมน์เดียวกัน, หรือแนวทะแยงเดียวกัน)

```java
import java.util.ArrayList;
import java.util.List;

public class NQueensDemo {
    static List<List<String>> solutions = new ArrayList<>();

    static boolean isSafe(int[] queens, int row, int col) {
        for (int r = 0; r < row; r++) {
            int c = queens[r];
            if (c == col) return false;                          // คอลัมน์ชนกัน
            if (Math.abs(c - col) == Math.abs(r - row)) return false; // แนวทะแยงชนกัน
        }
        return true;
    }

    static void backtrack(int[] queens, int row, int n) {
        if (row == n) { // วางครบทุกแถวแล้ว = พบคำตอบหนึ่งชุด
            solutions.add(buildBoard(queens, n));
            return;
        }

        for (int col = 0; col < n; col++) {
            if (isSafe(queens, row, col)) {
                queens[row] = col;          // เลือก: วางราชินีที่ (row, col)
                backtrack(queens, row + 1, n); // ลองต่อแถวถัดไป
                // ไม่ต้อง reset queens[row] เพราะ loop รอบถัดไปจะเขียนทับค่าใหม่ทันที (backtrack โดยธรรมชาติ)
            }
        }
    }

    static List<String> buildBoard(int[] queens, int n) {
        List<String> board = new ArrayList<>();
        for (int col : queens) {
            StringBuilder row = new StringBuilder();
            for (int c = 0; c < n; c++) {
                row.append(c == col ? "Q" : ".");
            }
            board.add(row.toString());
        }
        return board;
    }

    public static void main(String[] args) {
        int n = 4;
        backtrack(new int[n], 0, n);
        System.out.println("จำนวนคำตอบทั้งหมด: " + solutions.size()); // 2 คำตอบสำหรับ n=4
        for (List<String> solution : solutions) {
            solution.forEach(System.out::println);
            System.out.println("---");
        }
    }
}
```

## 8. ตัวอย่าง Backtracking: Generate All Permutations

```java
import java.util.ArrayList;
import java.util.List;

public class PermutationsDemo {
    static List<List<Integer>> result = new ArrayList<>();

    static void backtrack(List<Integer> current, List<Integer> remaining) {
        if (remaining.isEmpty()) { // ไม่มีตัวเลขเหลือให้เลือกแล้ว = จัดเรียงครบแล้ว
            result.add(new ArrayList<>(current)); // ต้อง copy list เพราะ current จะถูกแก้ไขต่อ
            return;
        }

        for (int i = 0; i < remaining.size(); i++) {
            int chosen = remaining.get(i);

            current.add(chosen);                        // เลือก
            remaining.remove(i);

            backtrack(current, remaining);                // ลองต่อ

            remaining.add(i, chosen);                      // ยกเลิกการเลือก (backtrack ตัวจริง)
            current.remove(current.size() - 1);
        }
    }

    public static void main(String[] args) {
        backtrack(new ArrayList<>(), new ArrayList<>(List.of(1, 2, 3)));
        System.out.println("จำนวนการจัดเรียงทั้งหมด: " + result.size()); // 3! = 6
        result.forEach(System.out::println);
    }
}
```

## 9. ตัวอย่าง Backtracking: Subset Sum

หาว่ามี subset ของ array ที่บวกกันได้ผลรวมเท่ากับ target หรือไม่

```java
import java.util.ArrayList;
import java.util.List;

public class SubsetSumDemo {
    static boolean backtrack(int[] nums, int index, int remainingTarget) {
        if (remainingTarget == 0) return true;             // พบคำตอบ! ผลรวมครบตามต้องการแล้ว
        if (index >= nums.length || remainingTarget < 0) return false; // หมดตัวเลือกหรือเกินเป้า

        // ตัวเลือกที่ 1: ใช้ nums[index] ในชุดคำตอบ
        if (backtrack(nums, index + 1, remainingTarget - nums[index])) {
            return true;
        }

        // ตัวเลือกที่ 2: ไม่ใช้ nums[index] (backtrack ไปลองตัวเลือกอื่น)
        return backtrack(nums, index + 1, remainingTarget);
    }

    public static void main(String[] args) {
        int[] nums = {3, 7, 5, 1, 9};
        int target = 12;
        System.out.println(backtrack(nums, 0, target)); // true (3 + 9 = 12, หรือ 7 + 5 = 12)
    }
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียน memoized recursive function สำหรับหาค่า `binomial(n, k)`
(สัมประสิทธิ์ทวินาม) โดยใช้สูตร `C(n,k) = C(n-1,k-1) + C(n-1,k)`

**เฉลย:**

```java
import java.util.HashMap;
import java.util.Map;

public class Exercise1 {
    static Map<String, Long> cache = new HashMap<>();

    static long binomial(int n, int k) {
        if (k == 0 || k == n) return 1;
        String key = n + "," + k;
        if (cache.containsKey(key)) return cache.get(key);

        long result = binomial(n - 1, k - 1) + binomial(n - 1, k);
        cache.put(key, result);
        return result;
    }

    public static void main(String[] args) {
        System.out.println(binomial(10, 3)); // 120
    }
}
```

**2)** เขียน backtracking function ที่สร้าง**ทุก subset**ที่เป็นไปได้ของ array
(power set)

**เฉลย:**

```java
import java.util.ArrayList;
import java.util.List;

public class Exercise2 {
    static List<List<Integer>> result = new ArrayList<>();

    static void backtrack(int[] nums, int index, List<Integer> current) {
        if (index == nums.length) {
            result.add(new ArrayList<>(current));
            return;
        }
        // ไม่รวม nums[index]
        backtrack(nums, index + 1, current);
        // รวม nums[index]
        current.add(nums[index]);
        backtrack(nums, index + 1, current);
        current.remove(current.size() - 1);
    }

    public static void main(String[] args) {
        backtrack(new int[]{1, 2, 3}, 0, new ArrayList<>());
        System.out.println(result); // 2^3 = 8 subsets รวมทั้ง [] และ [1,2,3]
    }
}
```

**3)** อธิบายว่าทำไม naive recursive Fibonacci มี time complexity O(2^n) แต่
memoized version มี O(n)

**เฉลย**: Naive recursion คำนวณ `fibonacci(k)` ซ้ำหลายครั้งสำหรับค่า k เดียวกัน
(เช่น `fibonacci(3)` ถูกเรียกซ้ำหลายครั้งเมื่อคำนวณ `fibonacci(5)`) ทำให้จำนวน
การเรียกเติบโตแบบทวีคูณตามความลึกของ recursion tree ส่วน memoization เก็บผล
ลัพธ์ของแต่ละค่า n ไว้ใน cache หลังคำนวณครั้งแรก ทำให้แต่ละค่า n ถูกคำนวณจริง
**เพียงครั้งเดียว**เท่านั้น จำนวนการคำนวณทั้งหมดจึงเป็นเชิงเส้นตามค่า n (O(n))

### สรุปเนื้อหา Part 29

- Head recursion รอผลจากการเรียกซ้ำก่อนทำงานต่อ, tail recursion เรียกตัวเองเป็น
  คำสั่งสุดท้าย (Java ไม่มี tail call optimization)
- Memoization (top-down DP) แคชผลลัพธ์ลดการคำนวณซ้ำ จาก O(2^n) เป็น O(n) สำหรับ
  Fibonacci
- Bottom-up DP (tabulation) คำนวณจากปัญหาเล็กไปใหญ่โดยไม่ใช้ recursion ปลอดภัย
  จาก StackOverflowError กว่า
- Backtracking ลองทุกความเป็นไปได้อย่างมีระบบ ถอยกลับเมื่อทางที่เลือกไม่นำไปสู่
  คำตอบ
- ใช้ backtracking กับปัญหา N-Queens, Permutations, Subset Sum และปัญหา
  combinatorial อื่น ๆ

**ต่อไป**: [Part 30 — Sorting Algorithms](./part-030-sorting-algorithms.md)
