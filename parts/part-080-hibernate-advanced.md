# Part 80: Hibernate/JPA ขั้นสูง: Relationships, Lazy/Eager Loading, N+1

> ขั้นตอนที่ 791-800 ของหลักสูตร | ระดับ: สูงมาก

## สารบัญ

1. ทบทวน Relational Database Relationships
2. `@ManyToOne` และ `@OneToMany`
3. `@OneToOne`
4. `@ManyToMany`
5. Lazy Loading vs Eager Loading
6. `LazyInitializationException`
7. ปัญหา N+1 Query
8. การแก้ปัญหา N+1: `JOIN FETCH` และ `@EntityGraph`
9. Cascade Types
10. แบบฝึกหัดและสรุป

---

## 1. ทบทวน Relational Database Relationships

ฐานข้อมูลเชิงสัมพันธ์ (ทบทวนจาก Part 64-65) มีความสัมพันธ์ระหว่างตาราง 3
ประเภทหลัก: **One-to-Many**, **Many-to-One**, **Many-to-Many** — JPA
(Part 79) แปลงความสัมพันธ์เหล่านี้เป็น **object reference** ใน Java (ทบทวน
แนวคิด reference จาก Part 11) โดยอัตโนมัติ

## 2. `@ManyToOne` และ `@OneToMany`

**ตัวอย่าง**: สินค้าหลายตัวอยู่ในหมวดหมู่เดียว (Category 1 -> Products
หลายตัว)

```java
import jakarta.persistence.*;
import java.util.ArrayList;
import java.util.List;

@Entity
public class Category {
    @Id @GeneratedValue
    private Long id;
    private String name;

    // "หนึ่ง" Category มีสินค้าได้ "หลาย" ตัว - mappedBy ระบุว่า field ไหนใน Product เป็นเจ้าของความสัมพันธ์
    @OneToMany(mappedBy = "category")
    private List<Product> products = new ArrayList<>(); // ทบทวน ArrayList จาก Part 22
}

@Entity
public class Product {
    @Id @GeneratedValue
    private Long id;
    private String name;

    // "หลาย" Product เป็นของ Category เดียว - ฝั่งนี้คือเจ้าของความสัมพันธ์จริง (มี foreign key)
    @ManyToOne
    @JoinColumn(name = "category_id") // ชื่อ foreign key column ในตาราง products
    private Category category;
}
```

```sql
-- โครงสร้างตารางที่ Hibernate สร้างให้ (DDL):
CREATE TABLE categories (id BIGINT PRIMARY KEY, name VARCHAR(255));
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    name VARCHAR(255),
    category_id BIGINT REFERENCES categories(id) -- foreign key (ทบทวนจาก Part 65)
);
```

**หลักการสำคัญ**: **`@ManyToOne` คือฝั่งที่เป็นเจ้าของความสัมพันธ์เสมอ**
(มี foreign key column จริงในตาราง) — **`@OneToMany` (ฝั่ง `mappedBy`) เป็น
แค่มุมมองที่สร้างขึ้นเพื่อความสะดวก** ไม่มี column เพิ่มในตารางของตัวเอง

```java
public class RelationshipUsageDemo {
    void example(Category category, Product product) {
        product.setCategory(category);          // ต้อง set ฝั่งเจ้าของความสัมพันธ์ (ManyToOne) เสมอ
        category.getProducts().add(product);      // ควร set ทั้งสองฝั่งให้ตรงกัน (แนวปฏิบัติที่ดี)
        // ถ้า set แค่ฝั่งเดียว object ใน memory จะไม่ตรงกับฐานข้อมูลจริง (แต่ Hibernate จะ save ตาม
        // ฝั่ง @ManyToOne เป็นหลัก เพราะเป็นเจ้าของ foreign key)
    }
}
```

## 3. `@OneToOne`

**ตัวอย่าง**: หนึ่ง User มี Profile เดียว

```java
@Entity
public class User {
    @Id @GeneratedValue
    private Long id;
    private String username;

    @OneToOne(mappedBy = "user", cascade = CascadeType.ALL) // ทบทวน Cascade ในหัวข้อ 9
    private UserProfile profile;
}

@Entity
public class UserProfile {
    @Id @GeneratedValue
    private Long id;
    private String bio;

    @OneToOne
    @JoinColumn(name = "user_id", unique = true) // unique = true ทำให้เป็น "หนึ่งต่อหนึ่ง" จริง ๆ
    private User user;
}
```

## 4. `@ManyToMany`

**ตัวอย่าง**: นักเรียนหลายคนลงทะเบียนวิชาได้หลายวิชา (Student <-> Course)
ต้องมี**ตารางกลาง (junction table)**

```java
@Entity
public class Student {
    @Id @GeneratedValue
    private Long id;
    private String name;

    @ManyToMany
    @JoinTable(
        name = "student_course", // ชื่อตารางกลาง
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private List<Course> courses = new java.util.ArrayList<>();
}

@Entity
public class Course {
    @Id @GeneratedValue
    private Long id;
    private String title;

    @ManyToMany(mappedBy = "courses") // ฝั่งนี้เป็น mappedBy (ไม่ใช่เจ้าของ join table)
    private List<Student> students = new java.util.ArrayList<>();
}
```

```sql
-- ตารางกลางที่ Hibernate สร้าง (คล้าย Adjacency structure ที่ทบทวนจาก Part 34 - Graph)
CREATE TABLE student_course (
    student_id BIGINT REFERENCES students(id),
    course_id BIGINT REFERENCES courses(id),
    PRIMARY KEY (student_id, course_id)
);
```

## 5. Lazy Loading vs Eager Loading

**ค่า default ของ JPA**: `@ManyToOne`/`@OneToOne` เป็น **EAGER** (โหลดทันที),
`@OneToMany`/`@ManyToMany` เป็น **LAZY** (โหลดเมื่อเรียกใช้จริง)

```java
@Entity
public class Product {
    @ManyToOne(fetch = FetchType.LAZY) // เปลี่ยนเป็น LAZY: โหลด Category ต่อเมื่อเรียก getCategory() จริง
    private Category category;

    @OneToMany(fetch = FetchType.EAGER) // เปลี่ยนเป็น EAGER: โหลด reviews มาพร้อมกับ Product ทันที
    private List<Review> reviews;
}
```

```
EAGER Loading:                          LAZY Loading:
SELECT * FROM products p                SELECT * FROM products  <- แค่นี้ก่อน
JOIN categories c ON ...                (ยังไม่ query categories เลย)
(โหลดทุกอย่างมาพร้อมกันทีเดียว)              │
                                          ▼ ถ้าเรียก product.getCategory()
                                        SELECT * FROM categories WHERE id = ?  <- query แยกตอนนี้
```

**หลักปฏิบัติ**: ใช้ **LAZY เป็นค่าเริ่มต้นเสมอ** (โดยเฉพาะ `@ManyToOne`
ที่ default เป็น EAGER ควรเปลี่ยนเป็น LAZY อย่างชัดเจน) เพื่อ**ควบคุมได้ว่า
จะโหลดข้อมูลเมื่อไร** — โหลดข้อมูลที่ไม่ได้ใช้จริงเป็นการเสียประสิทธิภาพ
โดยไม่จำเป็น (ทบทวนหลักการ "อย่า optimize ก่อนวัดผล" จาก Part 68 — แต่การ
เลือก fetch strategy ที่เหมาะสมเป็นการออกแบบพื้นฐานที่ควรทำแต่แรก ไม่ใช่
premature optimization)

## 6. `LazyInitializationException`

**ปัญหาคลาสสิกของ Lazy Loading**: ถ้าพยายามเข้าถึง lazy field **หลังจาก**
Hibernate Session ปิดไปแล้ว (เช่น หลังจากเมธอด `@Transactional` — ทบทวนจาก
Part 79 — จบไปแล้ว) จะเกิด exception

```java
@Service
public class ProductServiceProblem {
    private final ProductRepository repository;
    public ProductServiceProblem(ProductRepository repository) { this.repository = repository; }

    @Transactional
    public Product getProduct(Long id) {
        return repository.findById(id).orElseThrow(); // session ยังเปิดอยู่ตอนนี้
    } // <- @Transactional สิ้นสุด, session ปิด

    public void useProduct(Long id) {
        Product product = getProduct(id); // session ปิดไปแล้ว
        Category category = product.getCategory(); // LazyInitializationException!
        // เพราะ category ยังไม่ถูกโหลด (LAZY) และ session ที่จะไป query เพิ่มก็ปิดไปแล้ว
    }
}
```

**วิธีแก้**: (1) เข้าถึง lazy field **ภายใน** transaction ที่ session ยัง
เปิดอยู่, (2) ใช้ `JOIN FETCH` หรือ `@EntityGraph` โหลดข้อมูลที่ต้องใช้มา
ล่วงหน้า (หัวข้อ 8), หรือ (3) แปลงเป็น DTO (ทบทวนจาก Part 78) **ภายใน**
transaction ก่อนส่งออกไปนอก layer

## 7. ปัญหา N+1 Query

**N+1 Problem** เป็นปัญหาประสิทธิภาพที่พบบ่อยที่สุดของ ORM — เกิดเมื่อดึง
list ของ entity (1 query) แล้ว**วน loop เข้าถึง lazy relationship ของแต่ละ
ตัว** (N query เพิ่ม) รวมเป็น **1 + N queries**

```java
@Service
public class NPlusOneProblemDemo {
    private final ProductRepository repository;
    public NPlusOneProblemDemo(ProductRepository repository) { this.repository = repository; }

    @Transactional
    public void printAllProductsWithCategory() {
        List<Product> products = repository.findAll(); // Query 1: SELECT * FROM products (สมมติได้ 100 แถว)

        for (Product product : products) {
            // ทุกครั้งที่เรียก getCategory() บน lazy relationship ที่ยังไม่โหลด
            // Hibernate จะยิง query แยกไปดึง Category ทีละตัว!
            System.out.println(product.getCategory().getName()); // Query 2, 3, 4, ..., 101!
        }
        // รวม: 1 (products) + 100 (category ของแต่ละ product) = 101 queries!
        // ถ้าใช้ JOIN เดียวตอนแรก จะใช้แค่ 1 query เท่านั้น - ต่างกันมหาศาล
    }
}
```

**ผลกระทบ**: สำหรับข้อมูล 100 แถว การมี 101 queries (แต่ละ query มี network
round-trip — ทบทวนจาก Part 65-66) ช้ากว่า 1 query ที่มี JOIN อย่างมาก —
เป็นสาเหตุอันดับหนึ่งของปัญหา performance ใน production application ที่ใช้
ORM

## 8. การแก้ปัญหา N+1: `JOIN FETCH` และ `@EntityGraph`

```java
import org.springframework.data.jpa.repository.EntityGraph;
import org.springframework.data.jpa.repository.Query;
import java.util.List;

public interface ProductRepository extends JpaRepository<Product, Long> {

    // วิธีที่ 1: JOIN FETCH ใน JPQL (ทบทวน @Query จาก Part 79) - โหลด category มาพร้อมกันใน query เดียว
    @Query("SELECT p FROM Product p JOIN FETCH p.category")
    List<Product> findAllWithCategory();

    // วิธีที่ 2: @EntityGraph - ระบุว่า relationship ไหนควรโหลดมาพร้อมกัน โดยไม่ต้องเขียน JPQL เอง
    @EntityGraph(attributePaths = {"category"})
    List<Product> findAll(); // override findAll() เดิมให้ fetch category มาด้วยเสมอ
}
```

```sql
-- ผลลัพธ์: query เดียวที่ join ตารางทั้งสอง (แทน 101 queries จากหัวข้อ 7)
SELECT p.*, c.* FROM products p JOIN categories c ON p.category_id = c.id;
```

**หลักปฏิบัติในการตรวจจับ N+1**: เปิด log SQL ของ Hibernate ดูตอน
development (ทบทวนแนวคิด logging จาก Part 63) — ถ้าเห็น query แบบเดิมซ้ำ ๆ
กันหลายสิบ/หลายร้อยครั้งในการทำงานเดียว นั่นคือสัญญาณของ N+1 Problem

```yaml
# application.yml
spring:
  jpa:
    show-sql: true # แสดง SQL ที่ Hibernate สร้างจริงใน console (มีประโยชน์มากตอน debug N+1)
    properties:
      hibernate:
        format_sql: true
```

## 9. Cascade Types

**Cascade** กำหนดว่า operation บน entity แม่ (parent) จะ**ส่งต่อ**ไปยัง
entity ลูก (child) อย่างไร (ทบทวนแนวคิด Composite Pattern จาก Part 55 —
การทำงานกับ "กลุ่ม" ของ object ผ่าน parent เดียว)

```java
@Entity
public class Order {
    @Id @GeneratedValue
    private Long id;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new java.util.ArrayList<>();
}
```

| Cascade Type | ความหมาย |
|---|---|
| `PERSIST` | บันทึก parent -> บันทึก child ตามไปด้วยอัตโนมัติ |
| `REMOVE` | ลบ parent -> ลบ child ตามไปด้วย |
| `MERGE` | อัปเดต parent -> อัปเดต child ตามไปด้วย |
| `ALL` | รวมทุก cascade type ข้างบน |
| `orphanRemoval=true` | ถ้าลบ child ออกจาก collection ของ parent -> ลบ child นั้นออกจากฐานข้อมูลด้วย |

```java
public class CascadeDemo {
    void example(Order order, OrderItem item) {
        order.getItems().add(item);
        // save(order) จะ save item ทั้งหมดใน order.getItems() ตามไปด้วยอัตโนมัติ
        // เพราะ cascade = CascadeType.ALL (ไม่ต้อง save item แยกเอง)
    }
}
```

**ข้อควรระวัง**: `CascadeType.REMOVE`/`ALL` ที่ใช้ผิดที่อาจ**ลบข้อมูลที่ไม่
ควรลบตามไปด้วยโดยไม่ตั้งใจ** — ควรใช้เฉพาะกับความสัมพันธ์แบบ **"ownership"
ที่แท้จริง** (เช่น Order เป็นเจ้าของ OrderItem อย่างสมบูรณ์ ถ้า Order ถูกลบ
OrderItem ก็ไม่มีความหมายอีกต่อไป) ไม่ใช่กับความสัมพันธ์แบบ reference ทั่วไป
(เช่น Product กับ Category — ลบ Category ไม่ควรลบ Product ตามไปด้วย)

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียน Entity `Author` และ `Book` ที่มีความสัมพันธ์ One-to-Many
(Author เขียนได้หลาย Book) โดยใช้ LAZY loading

**เฉลย:**

```java
@Entity
public class Author {
    @Id @GeneratedValue private Long id;
    private String name;

    @OneToMany(mappedBy = "author", fetch = FetchType.LAZY)
    private List<Book> books = new ArrayList<>();
}

@Entity
public class Book {
    @Id @GeneratedValue private Long id;
    private String title;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id")
    private Author author;
}
```

**2)** เขียน `@Query` ที่ใช้ `JOIN FETCH` แก้ปัญหา N+1 สำหรับการดึง Author
ทั้งหมดพร้อม Book ของแต่ละคน

**เฉลย:**

```java
@Query("SELECT DISTINCT a FROM Author a JOIN FETCH a.books")
List<Author> findAllWithBooks(); // DISTINCT ป้องกัน Author ซ้ำถ้ามีหลาย Book
```

**3)** อธิบายว่าทำไม N+1 Problem เกิดขึ้นได้ง่ายมากในโค้ดที่ใช้ ORM โดยไม่
ระมัดระวัง

**เฉลย**: เพราะ Lazy Loading (ทบทวนหัวข้อ 5) ทำให้การเข้าถึง relationship
(`product.getCategory()`) **ดูเหมือนการเข้าถึง field ปกติ** (ไม่มีสัญญาณ
เตือนใด ๆ ในโค้ดว่ากำลังยิง query ใหม่ไปยังฐานข้อมูล) — นักพัฒนาที่ไม่ระวัง
อาจเขียน loop วน list ของ entity แล้วเข้าถึง lazy field ของแต่ละตัวโดยไม่รู้
ตัวว่าแต่ละครั้งที่เข้าถึงคือการยิง SQL query แยกไปยังฐานข้อมูล (ทบทวน cost
ของ network round-trip จาก Part 65-66) — ต่างจาก JDBC ดั้งเดิม (Part 64) ที่
ทุก query ต้องเขียนอย่างชัดเจน ทำให้เห็น "จำนวน query" ในโค้ดได้ตรง ๆ ORM
ซ่อนความซับซ้อนนี้ไว้เบื้องหลัง ทำให้ปัญหานี้เกิดขึ้นได้ง่ายและตรวจจับยากถ้า
ไม่เปิด SQL logging (ทบทวนหัวข้อ 8) ตรวจสอบเป็นประจำ

### สรุปเนื้อหา Part 80

- `@ManyToOne` เป็นเจ้าของความสัมพันธ์ (มี foreign key), `@OneToMany` เป็น
  มุมมองผ่าน `mappedBy`
- `@OneToOne` ใช้ `unique=true` บน join column, `@ManyToMany` ต้องมีตารางกลาง
- ใช้ LAZY loading เป็นค่าเริ่มต้นเสมอ ควบคุมการโหลดข้อมูลให้ชัดเจน
- `LazyInitializationException` เกิดเมื่อเข้าถึง lazy field หลัง session
  ปิดไปแล้ว
- N+1 Problem คือปัญหาประสิทธิภาพที่พบบ่อยที่สุดของ ORM — แก้ด้วย `JOIN
  FETCH`/`@EntityGraph`
- Cascade กำหนดว่า operation บน parent ส่งต่อไปยัง child อย่างไร — ใช้กับ
  ความสัมพันธ์แบบ ownership ที่แท้จริงเท่านั้น

**ต่อไป**: [Part 81 — Spring Boot: Validation และ Global Exception Handling](./part-081-validation.md)
