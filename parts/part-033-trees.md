# Part 33: Tree: Binary Tree, Binary Search Tree, Tree Traversal

> ขั้นตอนที่ 321-330 ของหลักสูตร | ระดับ: กลาง-สูง

## สารบัญ

1. Tree คืออะไร ศัพท์พื้นฐานที่ต้องรู้
2. Binary Tree: การสร้างและโครงสร้างพื้นฐาน
3. Tree Traversal: DFS (Preorder, Inorder, Postorder)
4. Tree Traversal: BFS (Level-order)
5. Binary Search Tree (BST) คืออะไร
6. การ Insert และ Search ใน BST
7. การ Delete ใน BST
8. ความสูงของ Tree และปัญหา Unbalanced BST
9. เบื้องต้นสู่ Balanced Tree (AVL, Red-Black Tree)
10. แบบฝึกหัดและสรุป

---

## 1. Tree คืออะไร ศัพท์พื้นฐานที่ต้องรู้

**Tree** คือโครงสร้างข้อมูลแบบ**ลำดับชั้น (hierarchical)** ประกอบด้วย node
ที่เชื่อมต่อกันแบบไม่มี cycle — ต่างจาก Linked List (Part 32) ที่เป็นเส้นตรง
Tree แต่ละ node สามารถมี**ลูกหลายตัว**ได้

```
ศัพท์พื้นฐาน:                    ตัวอย่าง Tree:

- Root: node บนสุด                      A (root)
- Parent: node แม่                     / \
- Child: node ลูก                      B   C
- Leaf: node ที่ไม่มีลูก               / \   \
- Height: ความสูงของ tree            D   E   F (leaf nodes: D, E, F)
- Depth: ระดับความลึกของ node หนึ่งตัว

A คือ root, B และ C คือลูกของ A (children), D และ E คือลูกของ B
D, E, F คือ leaf nodes (ไม่มีลูกต่อ)
Height ของ tree นี้ = 2 (จำนวน edge จาก root ถึง leaf ที่ลึกสุด)
```

**Binary Tree** คือ tree ที่**แต่ละ node มีลูกได้สูงสุด 2 ตัวเท่านั้น** (ซ้าย
และขวา) — เป็นชนิด tree ที่ใช้บ่อยที่สุดและเป็นพื้นฐานของโครงสร้างข้อมูลสำคัญ
อื่น ๆ อีกมาก (BST, Heap, AVL Tree, Red-Black Tree)

## 2. Binary Tree: การสร้างและโครงสร้างพื้นฐาน

```java
public class TreeNode {
    int value;
    TreeNode left;
    TreeNode right;

    TreeNode(int value) {
        this.value = value;
    }
}
```

```java
public class BinaryTreeBasicDemo {
    public static void main(String[] args) {
        // สร้าง tree ด้วยมือ:
        //         1
        //        / \
        //       2   3
        //      / \
        //     4   5
        TreeNode root = new TreeNode(1);
        root.left = new TreeNode(2);
        root.right = new TreeNode(3);
        root.left.left = new TreeNode(4);
        root.left.right = new TreeNode(5);

        System.out.println(root.value);              // 1
        System.out.println(root.left.value);           // 2
        System.out.println(root.left.left.value);       // 4
    }
}
```

## 3. Tree Traversal: DFS (Preorder, Inorder, Postorder)

**Depth-First Search (DFS)** เดินลงลึกไปทางหนึ่งก่อนจนสุด แล้วค่อยถอยกลับมา
ทางอื่น — สำหรับ Binary Tree มี 3 รูปแบบหลัก ต่างกันที่**ลำดับการเยี่ยม root**
เทียบกับลูกซ้าย-ขวา (ทบทวนแนวคิด recursion จาก Part 8, 29)

```java
public class DFSTraversalDemo {
    // Preorder: Root -> Left -> Right (เยี่ยม root ก่อนเสมอ)
    static void preorder(TreeNode node) {
        if (node == null) return;
        System.out.print(node.value + " "); // เยี่ยม root ก่อน
        preorder(node.left);
        preorder(node.right);
    }

    // Inorder: Left -> Root -> Right (สำหรับ BST จะได้ค่าเรียงลำดับจากน้อยไปมาก!)
    static void inorder(TreeNode node) {
        if (node == null) return;
        inorder(node.left);
        System.out.print(node.value + " "); // เยี่ยม root ระหว่างซ้ายและขวา
        inorder(node.right);
    }

    // Postorder: Left -> Right -> Root (เยี่ยม root หลังสุด - มีประโยชน์เมื่อต้อง "ลบ" tree)
    static void postorder(TreeNode node) {
        if (node == null) return;
        postorder(node.left);
        postorder(node.right);
        System.out.print(node.value + " "); // เยี่ยม root หลังสุด
    }

    public static void main(String[] args) {
        TreeNode root = new TreeNode(1);
        root.left = new TreeNode(2);
        root.right = new TreeNode(3);
        root.left.left = new TreeNode(4);
        root.left.right = new TreeNode(5);

        System.out.print("Preorder: ");  preorder(root);  System.out.println(); // 1 2 4 5 3
        System.out.print("Inorder: ");   inorder(root);   System.out.println(); // 4 2 5 1 3
        System.out.print("Postorder: "); postorder(root); System.out.println(); // 4 5 2 3 1
    }
}
```

```
Preorder visualization:  1 -> 2 -> 4 -> 5 -> 3
เยี่ยม root ก่อนเสมอ แล้วค่อยลงซ้าย แล้วค่อยลงขวา (เหมาะสำหรับ "copy" tree)

         1(1st)
        /      \
     2(2nd)    3(5th)
    /      \
 4(3rd)   5(4th)
```

**กรณีใช้งานของแต่ละแบบ**:
- **Preorder**: ใช้ทำสำเนา (clone) tree, หรือแปลง tree เป็น expression แบบ
  prefix notation
- **Inorder**: ใช้กับ **Binary Search Tree** เพื่อให้ได้ค่าเรียงลำดับ (สำคัญ
  มากที่สุด — ใช้บ่อยที่สุด)
- **Postorder**: ใช้เมื่อต้อง**ประมวลผลลูกก่อนแม่** เช่น การลบ tree (ต้องลบ
  ลูกก่อนแม่เสมอ เพื่อไม่ให้สูญเสีย reference), หรือคำนวณขนาดของ subdirectory
  ในระบบไฟล์ (ต้องรู้ขนาดของทุกไฟล์ย่อยก่อนจะรวมเป็นขนาดของโฟลเดอร์แม่)

## 4. Tree Traversal: BFS (Level-order)

**Breadth-First Search (BFS)** เยี่ยม node **ทีละระดับ (level)** จากบนลงล่าง
ใช้ **Queue** เป็นแกนหลัก (ทบทวนจาก Part 25)

```java
import java.util.ArrayDeque;
import java.util.Queue;

public class BFSTraversalDemo {
    static void levelOrder(TreeNode root) {
        if (root == null) return;

        Queue<TreeNode> queue = new ArrayDeque<>();
        queue.offer(root);

        while (!queue.isEmpty()) {
            TreeNode current = queue.poll();
            System.out.print(current.value + " ");

            if (current.left != null) queue.offer(current.left);
            if (current.right != null) queue.offer(current.right);
        }
    }

    public static void main(String[] args) {
        TreeNode root = new TreeNode(1);
        root.left = new TreeNode(2);
        root.right = new TreeNode(3);
        root.left.left = new TreeNode(4);
        root.left.right = new TreeNode(5);

        levelOrder(root); // 1 2 3 4 5 (เยี่ยมทีละระดับ: ระดับ 0 -> ระดับ 1 -> ระดับ 2)
    }
}
```

**เปรียบเทียบ DFS กับ BFS**: DFS ใช้ recursion (หรือ explicit stack) และเหมาะ
กับการหาเส้นทางลึก ๆ, BFS ใช้ queue และเหมาะกับการหาเส้นทางที่**สั้นที่สุด**
หรือประมวลผลทีละระดับ (ทบทวนตัวอย่าง BFS บน graph จาก Part 25)

## 5. Binary Search Tree (BST) คืออะไร

**BST** คือ Binary Tree ที่มี**คุณสมบัติพิเศษ**: สำหรับทุก node —
**ค่าทุกตัวใน subtree ซ้ายต้องน้อยกว่า node นั้น** และ **ค่าทุกตัวใน subtree
ขวาต้องมากกว่า node นั้น** — คุณสมบัตินี้ทำให้ค้นหา, แทรก, ลบข้อมูลได้เร็ว

```
BST ที่ถูกต้อง:              ไม่ใช่ BST (ผิดกฎ):

       8                            8
      / \                          / \
     3   10                       3   10
    / \    \                     / \    \
   1   6    14                  1   9    14   <- 9 อยู่ใน subtree ซ้ายของ 8
                                                   แต่ 9 > 8 ผิดกฎ BST!
```

## 6. การ Insert และ Search ใน BST

```java
public class BST {
    TreeNode root;

    public void insert(int value) {
        root = insertRecursive(root, value);
    }

    private TreeNode insertRecursive(TreeNode node, int value) {
        if (node == null) {
            return new TreeNode(value); // base case: เจอตำแหน่งที่ควรแทรกแล้ว
        }

        if (value < node.value) {
            node.left = insertRecursive(node.left, value); // ค่าน้อยกว่า -> ไปทางซ้าย
        } else if (value > node.value) {
            node.right = insertRecursive(node.right, value); // ค่ามากกว่า -> ไปทางขวา
        }
        // ถ้า value == node.value ไม่ทำอะไร (ไม่อนุญาตค่าซ้ำใน BST นี้)

        return node;
    }

    public boolean search(int value) {
        return searchRecursive(root, value);
    }

    private boolean searchRecursive(TreeNode node, int value) {
        if (node == null) return false;
        if (value == node.value) return true;
        return (value < node.value)
               ? searchRecursive(node.left, value)
               : searchRecursive(node.right, value);
    }

    public void printInorder() {
        inorderHelper(root);
        System.out.println();
    }

    private void inorderHelper(TreeNode node) {
        if (node == null) return;
        inorderHelper(node.left);
        System.out.print(node.value + " ");
        inorderHelper(node.right);
    }
}
```

```java
public class BSTDemo {
    public static void main(String[] args) {
        BST bst = new BST();
        int[] values = {8, 3, 10, 1, 6, 14, 4, 7};
        for (int v : values) {
            bst.insert(v);
        }

        bst.printInorder(); // 1 3 4 6 7 8 10 14 (เรียงลำดับอัตโนมัติ! - นี่คือคุณสมบัติสำคัญของ BST)

        System.out.println(bst.search(6));  // true
        System.out.println(bst.search(99)); // false
    }
}
```

**Time Complexity ของ BST**: `insert()`, `search()`, `delete()` ทั้งหมด
**O(h)** โดย h คือความสูงของ tree — ถ้า tree **balanced** (สมดุล) h ≈ log(n)
ทำให้เร็วมาก แต่ถ้า **unbalanced** (ไม่สมดุล) h อาจเท่ากับ n ทำให้ช้าเท่า
Linked List (ดูหัวข้อ 8)

## 7. การ Delete ใน BST

การลบ node ใน BST มี 3 กรณี ซับซ้อนกว่า insert/search มาก:

```java
public class BSTDelete extends BST {
    public void delete(int value) {
        root = deleteRecursive(root, value);
    }

    private TreeNode deleteRecursive(TreeNode node, int value) {
        if (node == null) return null;

        if (value < node.value) {
            node.left = deleteRecursive(node.left, value);
        } else if (value > node.value) {
            node.right = deleteRecursive(node.right, value);
        } else {
            // เจอ node ที่ต้องการลบแล้ว - มี 3 กรณี

            // กรณี 1: ไม่มีลูก (leaf node) - ลบตรง ๆ ได้เลย
            if (node.left == null && node.right == null) {
                return null;
            }

            // กรณี 2: มีลูกข้างเดียว - แทนที่ node นี้ด้วยลูกที่มี
            if (node.left == null) return node.right;
            if (node.right == null) return node.left;

            // กรณี 3: มีลูกทั้งสองข้าง - หา "successor" (ค่าน้อยที่สุดใน subtree ขวา)
            // มาแทนที่ node นี้ แล้วลบ successor ตัวเดิมออกจาก subtree ขวา
            TreeNode successor = findMin(node.right);
            node.value = successor.value;
            node.right = deleteRecursive(node.right, successor.value);
        }
        return node;
    }

    private TreeNode findMin(TreeNode node) {
        while (node.left != null) {
            node = node.left; // ค่าน้อยที่สุดอยู่ทางซ้ายสุดเสมอ
        }
        return node;
    }
}
```

```
การลบกรณีที่ 3 (มีลูกทั้งสองข้าง): ลบ node ค่า 8 จาก BST นี้

       8                    หา successor (ค่าน้อยสุดใน subtree ขวาของ 8) = 10
      / \                   แทนที่ 8 ด้วย 10 แล้วลบ 10 ตัวเดิมออก
     3   14
        /  \
      10    16          ผลลัพธ์:      10
                                       / \
                                      3   14
                                         /  \
                                       null  16
```

## 8. ความสูงของ Tree และปัญหา Unbalanced BST

**ปัญหาสำคัญ**: ถ้าแทรกข้อมูลที่**เรียงลำดับอยู่แล้ว** ลงใน BST ทีละตัว จะได้
tree ที่**เอียงไปทางเดียว**จนกลายเป็นเหมือน Linked List — ทำให้ Time
Complexity แย่ลงจาก O(log n) เป็น **O(n)**

```java
public class UnbalancedBSTDemo {
    public static void main(String[] args) {
        BST bst = new BST();
        // แทรกข้อมูลที่เรียงลำดับแล้ว -> ได้ tree ที่เอียงไปทางขวาทั้งหมด
        for (int i = 1; i <= 7; i++) {
            bst.insert(i);
        }

        // โครงสร้างที่ได้:
        //  1
        //   \
        //    2
        //     \
        //      3
        //       \
        //        4
        //         \
        //          5
        //           \
        //            6
        //             \
        //              7
        // นี่คือ "worst case" ของ BST - กลายเป็น linked list ที่ search เป็น O(n) ไม่ใช่ O(log n)!
    }
}
```

## 9. เบื้องต้นสู่ Balanced Tree (AVL, Red-Black Tree)

เพื่อแก้ปัญหา unbalanced BST มีโครงสร้างข้อมูลที่**ปรับสมดุลอัตโนมัติ**ทุกครั้ง
ที่ insert/delete:

- **AVL Tree**: รักษาสมดุลโดยเช็คว่าความสูงของ subtree ซ้าย-ขวาต่างกันไม่เกิน 1
  เสมอ ถ้าไม่สมดุลจะทำ **rotation** (หมุนโครงสร้าง) ปรับให้สมดุลใหม่
- **Red-Black Tree**: ใช้ "สี" (แดง/ดำ) กำกับแต่ละ node ตามกฎเฉพาะที่การันตี
  ความสมดุลแบบคร่าว ๆ — **นี่คือโครงสร้างที่ Java's `TreeMap` และ `TreeSet`
  ใช้ภายในจริง** (ทบทวนจาก Part 23-24) ทำให้ operation ต่าง ๆ คงที่ที่ O(log n)
  เสมอไม่ว่าจะแทรกข้อมูลแบบไหนก็ตาม

```java
import java.util.TreeMap;

public class BalancedTreeInPracticeDemo {
    public static void main(String[] args) {
        // TreeMap ใช้ Red-Black Tree ภายใน - รับประกัน O(log n) เสมอ
        // แม้ใส่ข้อมูลที่เรียงลำดับแล้วทีละตัว (ไม่มีปัญหา unbalanced เหมือน BST ที่เขียนเอง)
        TreeMap<Integer, String> map = new TreeMap<>();
        for (int i = 1; i <= 100_000; i++) {
            map.put(i, "value" + i); // ยังคงเร็ว O(log n) ต่อการ insert เสมอ
        }
        System.out.println(map.size());
        System.out.println(map.get(50000)); // O(log n) เสมอ ไม่ว่าจะใส่ข้อมูลแบบไหน
    }
}
```

การเขียน AVL Tree หรือ Red-Black Tree เองอย่างสมบูรณ์มีความซับซ้อนสูงมาก
(rotation logic ที่ละเอียดอ่อน) ในทางปฏิบัติจึงใช้ `TreeMap`/`TreeSet` ที่ Java
เตรียมไว้ให้แล้วเสมอ — หลักสูตรนี้เน้นให้**เข้าใจแนวคิด**มากกว่าการ implement
เองอย่างสมบูรณ์ ซึ่งเพียงพอสำหรับใช้งานจริงและตอบคำถามสัมภาษณ์งานระดับกลาง

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียนเมธอด `height(TreeNode root)` ที่คำนวณความสูงของ binary tree

**เฉลย:**

```java
public class Exercise1 {
    static int height(TreeNode node) {
        if (node == null) return -1; // ความสูงของ tree ว่างคือ -1 (หรือ 0 ขึ้นกับนิยาม)
        return 1 + Math.max(height(node.left), height(node.right));
    }
}
```

**2)** เขียนเมธอด `isValidBST(TreeNode root)` ที่ตรวจสอบว่า tree เป็น BST ที่
ถูกต้องหรือไม่ (ใช้เทคนิค inorder traversal ต้องได้ค่าเรียงลำดับเสมอ)

**เฉลย:**

```java
import java.util.ArrayList;
import java.util.List;

public class Exercise2 {
    static boolean isValidBST(TreeNode root) {
        List<Integer> values = new ArrayList<>();
        inorder(root, values);
        for (int i = 1; i < values.size(); i++) {
            if (values.get(i) <= values.get(i - 1)) return false; // ต้องเรียงเพิ่มขึ้นเสมอ
        }
        return true;
    }

    static void inorder(TreeNode node, List<Integer> values) {
        if (node == null) return;
        inorder(node.left, values);
        values.add(node.value);
        inorder(node.right, values);
    }
}
```

**3)** เขียนเมธอด `countNodes(TreeNode root)` ที่นับจำนวน node ทั้งหมดใน tree
โดยใช้ recursion

**เฉลย:**

```java
public class Exercise3 {
    static int countNodes(TreeNode node) {
        if (node == null) return 0;
        return 1 + countNodes(node.left) + countNodes(node.right);
    }
}
```

### สรุปเนื้อหา Part 33

- Tree เป็นโครงสร้างข้อมูลแบบลำดับชั้น, Binary Tree จำกัดลูกไม่เกิน 2 ต่อ node
- DFS มี 3 แบบ: Preorder (root ก่อน), Inorder (root กลาง - ให้ค่าเรียงลำดับใน
  BST), Postorder (root หลังสุด)
- BFS (Level-order) เยี่ยมทีละระดับด้วย Queue
- BST มีคุณสมบัติ: ซ้ายน้อยกว่า, ขวามากกว่า node เสมอ — insert/search/delete
  เป็น O(h) โดย h คือความสูง
- BST ที่ไม่สมดุล (unbalanced) อาจแย่ลงเป็น O(n) — แก้ด้วย AVL/Red-Black Tree
  ที่ปรับสมดุลอัตโนมัติ (Java's `TreeMap`/`TreeSet` ใช้ Red-Black Tree จริง)

**ต่อไป**: [Part 34 — Graph เบื้องต้น: BFS, DFS](./part-034-graphs.md)
