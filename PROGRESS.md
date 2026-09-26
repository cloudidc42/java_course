# สถานะความคืบหน้าของหลักสูตร

หลักสูตรนี้เป็นเอกสารที่เติบโตต่อเนื่อง (living document) เขียนและ commit เป็นชุด ๆ
ไฟล์นี้บันทึกว่า Part ไหนเขียนเสร็จสมบูรณ์แล้วบ้าง เพื่อให้ผู้เรียนและผู้ร่วมพัฒนา
ติดตามความคืบหน้าได้ง่าย

## Part ที่เขียนเสร็จแล้ว

- [x] Part 1 — บทนำสู่ Java, JDK/JRE/JVM, การติดตั้งเครื่องมือ, Hello World
- [x] Part 2 — โครงสร้างโปรแกรม Java, การคอมไพล์และรัน, กฎการตั้งชื่อ
- [x] Part 3 — ตัวแปร ชนิดข้อมูล และการแปลงชนิดข้อมูล
- [x] Part 4 — ตัวดำเนินการ (Operators) ทั้งหมดใน Java
- [x] Part 5 — คำสั่งควบคุมเงื่อนไข if-else, switch
- [x] Part 6 — คำสั่งวนซ้ำ for, while, do-while
- [x] Part 7 — Arrays หนึ่งมิติและหลายมิติ
- [x] Part 8 — Methods และการส่งผ่านพารามิเตอร์
- [x] Part 9 — String, StringBuilder, StringBuffer
- [x] Part 10 — Exception Handling เบื้องต้น
- [x] Part 11 — Classes และ Objects เบื้องต้น
- [x] Part 12 — Constructors และ this keyword
- [x] Part 13 — Encapsulation และ Access Modifiers
- [x] Part 14 — Inheritance
- [x] Part 15 — Polymorphism
- [x] Part 16 — Abstract Classes และ Interfaces
- [x] Part 17 — Static, Final และ Immutability
- [x] Part 18 — Enums แบบละเอียด
- [x] Part 19 — Nested Classes, Inner Classes, Anonymous Classes
- [x] Part 20 — Packages, การจัดระเบียบโปรเจกต์
- [x] Part 21 — Exception Handling ขั้นสูง
- [x] Part 22 — Collections: List (ArrayList, LinkedList)
- [x] Part 23 — Collections: Set (HashSet, LinkedHashSet, TreeSet)
- [x] Part 24 — Collections: Map (HashMap, LinkedHashMap, TreeMap)
- [x] Part 25 — Queue, Deque, PriorityQueue
- [x] Part 26 — Generics
- [x] Part 27 — Iterator, Iterable, Comparable, Comparator
- [x] Part 28 — Wrapper Classes, Autoboxing/Unboxing
- [x] Part 29 — Recursion ขั้นสูง: Backtracking, Memoization
- [x] Part 30 — Sorting Algorithms
- [x] Part 31 — Searching Algorithms, manual Stack/Queue
- [x] Part 32 — Linked List แบบ Manual
- [x] Part 33 — Tree: Binary Tree, BST, Traversal
- [x] Part 34 — Graph เบื้องต้น: BFS, DFS
- [x] Part 35 — Big O Notation

**หมวดที่ 2 (โครงสร้างข้อมูลและอัลกอริทึม, Part 21-35) เสร็จสมบูรณ์แล้ว!**

## กำลังดำเนินการ / อยู่ในแผนถัดไป

ดูรายการทั้งหมด (Part 36-105+) ได้ที่ [`CURRICULUM.md`](./CURRICULUM.md)
Part ถัดไปที่จะเขียนคือ **Part 36 — File I/O เบื้องต้น** (เริ่มหมวดที่ 3:
Java ระดับกลางถึงขั้นสูง)

## แนวทางการเขียนเนื้อหาต่อ

หากต้องการช่วยเขียน Part ถัดไป ให้ยึดรูปแบบเดียวกับ Part ที่มีอยู่แล้วใน `parts/`:
1. เริ่มด้วยสารบัญของ Part นั้น ๆ
2. อธิบายทฤษฎีสั้น กระชับ เข้าใจง่าย มีตัวอย่างประกอบทุกหัวข้อ
3. โค้ดตัวอย่างต้อง **คอมไพล์และรันได้จริง** (ระบุ `public class` ให้ตรงกับชื่อไฟล์เสมอ)
4. ปิดท้ายด้วยแบบฝึกหัด (พร้อมเฉลย) และสรุปเนื้อหา
5. อัปเดตสถานะในไฟล์นี้และใน `CURRICULUM.md` ทุกครั้งที่เขียน Part ใหม่เสร็จ
