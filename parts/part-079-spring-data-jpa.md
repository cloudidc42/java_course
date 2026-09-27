# Part 79: Spring Data JPA เบื้องต้น: Repository, Query Methods

> ขั้นตอนที่ 781-790 ของหลักสูตร | ระดับ: สูงมาก

## สารบัญ

1. ORM คืออะไร แก้ปัญหาอะไร (ทบทวนจาก JDBC — Part 64-65)
2. JPA vs Hibernate vs Spring Data JPA
3. `@Entity`: การนิยาม Entity Class
4. Primary Key: `@Id`, `@GeneratedValue`
5. `JpaRepository`: CRUD โดยไม่ต้องเขียนโค้ดเลย
6. Query Methods: สร้าง Query จากชื่อเมธอด
7. `@Query`: เขียน JPQL เอง
8. Pagination และ Sorting
9. Transaction ใน Spring Data JPA: `@Transactional`
10. แบบฝึกหัดและสรุป

---

## 1. ORM คืออะไร แก้ปัญหาอะไร (ทบทวนจาก JDBC — Part 64-65)

ทบทวนจาก Part 64-65: JDBC ทำงานได้แต่ต้องเขียน**boilerplate code จำนวนมาก**
(mapping ResultSet เป็น object ด้วยมือทุกครั้ง — ทบทวน "Manual ORM" จาก Part
64 หัวข้อ 8) — **ORM (Object-Relational Mapping)** คือเทคนิคที่**แปลง object
Java เป็นแถวในฐานข้อมูลโดยอัตโนมัติ** (และย้อนกลับ) ลดโค้ดซ้ำซ้อนอย่างมาก

## 2. JPA vs Hibernate vs Spring Data JPA

```
JPA (Java Persistence API)         <- specification/interface มาตรฐาน (ไม่มี implementation จริง)
      │
      ▼ implement โดย
Hibernate                            <- implementation ที่นิยมที่สุด (ทบทวนแนวคิด interface จาก Part 16)
      │
      ▼ ห่อด้วย
Spring Data JPA                       <- เพิ่มความสะดวก: Repository pattern, Query Methods (หัวข้อ 5-6)
```

**เปรียบเทียบกับ JDBC (Part 64) และ DAO Pattern (Part 65)**:
Spring Data JPA คือ**วิวัฒนาการขั้นสุดท้าย**ของ DAO Pattern ที่เราเขียนเอง —
แทนที่จะเขียน `JdbcUserDao implements UserDao` เองทั้งหมด เราแค่ประกาศ
**interface** แล้ว Spring สร้าง implementation ให้อัตโนมัติ (ทบทวนแนวคิด
Proxy Pattern จาก Part 55 — Spring Data ใช้ dynamic proxy สร้าง
implementation ตอน runtime)

## 3. `@Entity`: การนิยาม Entity Class

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

```java
import jakarta.persistence.*;

@Entity // บอกว่า class นี้ map กับตารางในฐานข้อมูล (ทบทวน annotation จาก Part 52)
@Table(name = "products") // ชื่อตาราง (ถ้าไม่ระบุ ใช้ชื่อ class เป็นค่า default)
public class Product {

    @Id // Primary Key (หัวข้อ 4)
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "product_name", nullable = false, length = 100)
    private String name;

    @Column(precision = 10, scale = 2)
    private double price;

    // JPA ต้องการ no-arg constructor เสมอ (ทบทวนจาก Part 12) - ใช้ Reflection สร้าง object (ทบทวน Part 53)
    protected Product() { }

    public Product(String name, double price) {
        this.name = name;
        this.price = price;
    }

    // getter/setter (ทบทวน JavaBeans convention จาก Part 13)
    public Long getId() { return id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public double getPrice() { return price; }
    public void setPrice(double price) { this.price = price; }
}
```

**ข้อสังเกตสำคัญ**: Entity ต้องมี **no-arg constructor** (แม้จะเป็น
`protected` ก็ได้ — Hibernate ใช้ reflection สร้าง object แล้วค่อย set
field ผ่าน reflection เช่นกัน) — นี่คือเหตุผลที่ **record (Part 51) ใช้เป็น
JPA Entity ไม่ได้โดยตรง** (record ไม่มี no-arg constructor และ field เป็น
`final` เสมอ ขัดกับที่ Hibernate ต้องแก้ไขค่าผ่าน setter/reflection)

## 4. Primary Key: `@Id`, `@GeneratedValue`

```java
@Entity
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY) // ให้ฐานข้อมูล generate (AUTO_INCREMENT - ทบทวน Part 65)
    private Long id;

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "order_seq") // ใช้ SQL Sequence (PostgreSQL)
    @SequenceGenerator(name = "order_seq", sequenceName = "order_sequence")
    private Long id2; // ตัวอย่างที่สอง (ในโค้ดจริงมีแค่ @Id เดียวต่อ entity)

    @Id
    @GeneratedValue(strategy = GenerationType.UUID) // ใช้ UUID (เหมาะกับระบบ distributed - ปูทางสู่ Part 91)
    private String id3;
}
```

| Strategy | ใช้เมื่อ |
|---|---|
| `IDENTITY` | MySQL, ใช้ AUTO_INCREMENT ของฐานข้อมูล |
| `SEQUENCE` | PostgreSQL, Oracle ที่รองรับ SQL Sequence |
| `UUID` | ระบบ distributed ที่ต้องการ ID ที่ไม่ชนกันข้าม database instance |
| `AUTO` | ให้ Hibernate เลือก strategy ที่เหมาะกับฐานข้อมูลที่ใช้เอง |

## 5. `JpaRepository`: CRUD โดยไม่ต้องเขียนโค้ดเลย

```java
import org.springframework.data.jpa.repository.JpaRepository;

public interface ProductRepository extends JpaRepository<Product, Long> {
    // ไม่ต้องเขียนอะไรเพิ่ม! ได้เมธอด CRUD ครบทันที (ทบทวนจาก Part 65)
}
```

```java
import org.springframework.stereotype.Service;
import java.util.List;
import java.util.Optional;

@Service
public class ProductService {
    private final ProductRepository repository;

    public ProductService(ProductRepository repository) { // ทบทวน Constructor Injection จาก Part 74
        this.repository = repository;
    }

    public Product create(Product product) {
        return repository.save(product); // INSERT (หรือ UPDATE ถ้ามี id อยู่แล้ว)
    }

    public Optional<Product> findById(Long id) {
        return repository.findById(id); // SELECT ... WHERE id = ? (ทบทวน Optional จาก Part 43)
    }

    public List<Product> findAll() {
        return repository.findAll(); // SELECT * FROM products
    }

    public void delete(Long id) {
        repository.deleteById(id); // DELETE FROM products WHERE id = ?
    }

    public long count() {
        return repository.count(); // SELECT COUNT(*) FROM products
    }
}
```

**เมธอดสำเร็จรูปของ `JpaRepository`**: `save()`, `findById()`, `findAll()`,
`deleteById()`, `count()`, `existsById()` — ครอบคลุม CRUD พื้นฐานทั้งหมด
โดยที่**เราไม่ต้องเขียน implementation เองแม้แต่บรรทัดเดียว** (ต่างจาก DAO
Pattern ที่เขียนเองใน Part 65 อย่างสิ้นเชิง)

## 6. Query Methods: สร้าง Query จากชื่อเมธอด

**Query Methods** คือความสามารถพิเศษของ Spring Data ที่**สร้าง SQL query
จากชื่อเมธอดโดยอัตโนมัติ** (ทบทวนแนวคิด convention over configuration จาก
Part 75) — ไม่ต้องเขียน SQL หรือ implementation เอง

```java
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;
import java.util.Optional;

public interface ProductRepository extends JpaRepository<Product, Long> {

    // findBy + field name -> SELECT * FROM products WHERE name = ?
    Optional<Product> findByName(String name);

    // findBy + field + condition keyword -> SELECT * FROM products WHERE price > ?
    List<Product> findByPriceGreaterThan(double price);

    // รวมหลายเงื่อนไขด้วย And/Or
    List<Product> findByNameContainingAndPriceLessThan(String keyword, double maxPrice);
    // -> SELECT * FROM products WHERE name LIKE %keyword% AND price < maxPrice

    // เรียงลำดับผลลัพธ์ (ทบทวน Comparator/sorting จาก Part 27)
    List<Product> findByPriceGreaterThanOrderByPriceDesc(double price);

    // นับจำนวน
    long countByNameContaining(String keyword);

    // เช็คว่ามีอยู่หรือไม่
    boolean existsByName(String name);

    // ลบตามเงื่อนไข
    void deleteByPriceLessThan(double price);
}
```

| Keyword | ความหมาย | SQL ที่ได้ |
|---|---|---|
| `findBy` | ดึงข้อมูล | `SELECT ... WHERE ...` |
| `And`/`Or` | รวมเงื่อนไข | `AND`/`OR` |
| `GreaterThan`/`LessThan` | เปรียบเทียบ | `>`/`<` |
| `Containing` | ค้นหาแบบ partial match | `LIKE %...%` |
| `OrderBy...Asc/Desc` | เรียงลำดับ | `ORDER BY ... ASC/DESC` |
| `IgnoreCase` | ไม่สนตัวพิมพ์ใหญ่-เล็ก | ปรับ collation |

**หลักการทำงาน**: Spring Data**parse ชื่อเมธอด**ตอน startup (ทบทวน
Reflection จาก Part 53) แยกเป็นส่วน ๆ ตาม keyword ที่รู้จัก แล้วสร้าง JPQL
query ให้อัตโนมัติ — ถ้าตั้งชื่อผิด (เช่นสะกด field name ผิด) จะ **fail
ตั้งแต่ startup** (ทบทวนหลักการ fail fast จาก Part 76)

## 7. `@Query`: เขียน JPQL เอง

เมื่อ query ซับซ้อนเกินกว่าที่ Query Method (หัวข้อ 6) จะสร้างชื่อเมธอดที่
อ่านง่ายได้ ใช้ **`@Query`** เขียน **JPQL** (Java Persistence Query
Language — คล้าย SQL แต่ทำงานกับ Entity/field name ไม่ใช่ table/column
name ตรง ๆ) เอง:

```java
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import java.util.List;

public interface ProductRepository extends JpaRepository<Product, Long> {

    @Query("SELECT p FROM Product p WHERE p.price BETWEEN :min AND :max")
    List<Product> findByPriceRange(@Param("min") double min, @Param("max") double max);

    // Native SQL (ถ้าต้องใช้ feature เฉพาะของฐานข้อมูลที่ JPQL ทำไม่ได้)
    @Query(value = "SELECT * FROM products WHERE name REGEXP :pattern", nativeQuery = true)
    List<Product> findByNamePattern(@Param("pattern") String pattern);

    @Query("SELECT AVG(p.price) FROM Product p")
    Double findAveragePrice();
}
```

**JPQL vs Native SQL**: JPQL ทำงานกับ **Entity class name** และ **field
name** (`Product`, `p.price`) ไม่ใช่ table/column name จริง — Hibernate
แปลงเป็น SQL จริงให้อัตโนมัติ ทำให้ query**ไม่ผูกติดกับฐานข้อมูลเฉพาะเจาะจง**
(portable ข้ามฐานข้อมูลต่างชนิดได้ดีกว่า native SQL)

## 8. Pagination และ Sorting

ทบทวนแนวคิด pagination จาก Part 65:

```java
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;
import org.springframework.data.jpa.repository.JpaRepository;

public interface ProductRepository extends JpaRepository<Product, Long> {
    Page<Product> findByPriceGreaterThan(double price, Pageable pageable); // รับ Pageable เพิ่มเข้ามา
}
```

```java
@Service
public class ProductPaginationService {
    private final ProductRepository repository;
    public ProductPaginationService(ProductRepository repository) { this.repository = repository; }

    public Page<Product> getPage(int pageNumber, int pageSize) {
        Pageable pageable = PageRequest.of(pageNumber, pageSize, Sort.by("price").descending());
        Page<Product> page = repository.findAll(pageable); // JpaRepository รองรับ pagination ในตัว

        System.out.println("หน้าปัจจุบัน: " + page.getNumber());
        System.out.println("จำนวนทั้งหมด: " + page.getTotalElements());
        System.out.println("จำนวนหน้าทั้งหมด: " + page.getTotalPages());

        return page;
    }
}
```

**`Page<T>`** มีทั้งข้อมูล (content) และ **metadata** (total elements,
total pages) ในตัวเดียว — สะดวกมากสำหรับสร้าง UI ที่ต้องแสดงเลขหน้า (ทบทวน
แนวคิด pagination ที่จะลงลึกเรื่อง API design ใน Part 85)

## 9. Transaction ใน Spring Data JPA: `@Transactional`

ทบทวน ACID และ transaction จาก Part 65: Spring จัดการ transaction ให้
อัตโนมัติผ่าน annotation เดียว (ทบทวนแนวคิด AOP/cross-cutting concern จาก
Part 72 — Filter):

```java
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class TransferService {
    private final AccountRepository accountRepository;

    public TransferService(AccountRepository accountRepository) {
        this.accountRepository = accountRepository;
    }

    @Transactional // ทบทวนแนวคิด commit/rollback จาก Part 65 - Spring จัดการให้อัตโนมัติทั้งหมด
    public void transferMoney(Long fromId, Long toId, double amount) {
        Account from = accountRepository.findById(fromId).orElseThrow();
        Account to = accountRepository.findById(toId).orElseThrow();

        from.withdraw(amount); // ถ้า throw exception ที่นี่ Spring จะ rollback ทั้งหมดอัตโนมัติ
        to.deposit(amount);

        accountRepository.save(from);
        accountRepository.save(to);
        // ไม่ต้องเรียก commit()/rollback() เองเลย! (เทียบกับที่เขียนด้วยมือใน Part 65)
        // ถ้าเมธอดนี้ทำงานจนจบโดยไม่มี exception -> Spring commit ให้อัตโนมัติ
        // ถ้าเกิด unchecked exception ระหว่างทาง -> Spring rollback ให้อัตโนมัติ
    }

    static class Account {
        void withdraw(double amount) { }
        void deposit(double amount) { }
    }
    interface AccountRepository extends JpaRepository<Account, Long> { }
}
```

**ข้อควรระวังสำคัญ**: `@Transactional` **rollback อัตโนมัติเฉพาะ unchecked
exception** (`RuntimeException` และ subclass — ทบทวนจาก Part 10, 21) ถ้า
throw checked exception จะ**ไม่ rollback**โดย default (ต้องระบุ
`@Transactional(rollbackFor = SomeCheckedException.class)` เพิ่มเติมถ้า
ต้องการ) — นี่คือเหตุผลเชิงปฏิบัติอีกข้อที่สนับสนุนแนวโน้มการใช้ unchecked
exception ที่กล่าวถึงใน Part 21

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียน `interface UserRepository extends JpaRepository<User, Long>`
พร้อม Query Method หา user ตามอีเมล และหา user ที่อายุมากกว่าค่าที่กำหนด
เรียงจากอายุน้อยไปมาก

**เฉลย:**

```java
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
    List<User> findByAgeGreaterThanOrderByAgeAsc(int age);
}
```

**2)** เขียน `@Query` ที่หาสินค้าที่ชื่อขึ้นต้นด้วยคำที่กำหนด (ใช้ JPQL
`LIKE`)

**เฉลย:**

```java
@Query("SELECT p FROM Product p WHERE p.name LIKE :prefix%")
List<Product> findByNameStartingWith(@Param("prefix") String prefix);
```

**3)** อธิบายว่าทำไม JPA Entity ต้องมี no-arg constructor แม้จะเป็น
`protected`

**เฉลย**: Hibernate (ทบทวนจาก Part 2) สร้าง object ของ Entity ผ่าน
**Reflection** (ทบทวนจาก Part 53) โดยเรียก **no-arg constructor** ก่อน
แล้วค่อยใช้ reflection **set ค่าให้แต่ละ field ทีหลัง** (ไม่ผ่าน
constructor ที่มีพารามิเตอร์ที่เราเขียนเอง) — ถ้าไม่มี no-arg constructor
เลย Hibernate จะไม่มีทางสร้าง object ได้ตอน**ดึงข้อมูลจากฐานข้อมูลกลับมา
เป็น Java object** (deserialize จาก ResultSet — ทบทวนแนวคิดคล้าย Part 64
หัวข้อ 8) — การใช้ `protected` (แทน `public`) ยังคงป้องกันไม่ให้โค้ด
ธุรกิจทั่วไปสร้าง object แบบ "เปล่า ๆ ไม่มีข้อมูล" โดยไม่ตั้งใจ (ทบทวน
access modifier จาก Part 13) แต่ Hibernate (ซึ่งใช้ reflection ที่เจาะทะลุ
access modifier ได้ — ทบทวนจาก Part 53) ยังเรียกใช้ได้อยู่ดี

### สรุปเนื้อหา Part 79

- ORM แปลง object Java เป็นแถวในฐานข้อมูลอัตโนมัติ ลด boilerplate เทียบกับ
  JDBC/DAO ที่เขียนเองใน Part 64-65
- JPA เป็น specification, Hibernate เป็น implementation ที่นิยม, Spring
  Data JPA เพิ่ม Repository pattern
- `@Entity` ต้องมี `@Id` และ no-arg constructor (ใช้ reflection สร้าง
  object — ทบทวน Part 53) — record ใช้เป็น Entity ตรง ๆ ไม่ได้
- `JpaRepository` ให้ CRUD สำเร็จรูปโดยไม่ต้องเขียน implementation
- Query Methods สร้าง SQL จากชื่อเมธอด, `@Query` ใช้เมื่อ query ซับซ้อนกว่า
- `@Transactional` จัดการ commit/rollback อัตโนมัติ — rollback เฉพาะ
  unchecked exception โดย default

**ต่อไป**: [Part 80 — Hibernate/JPA ขั้นสูง: Relationships, Lazy/Eager Loading](./part-080-hibernate-advanced.md)
