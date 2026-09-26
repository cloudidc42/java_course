# Part 65: SQL กับ Java: CRUD แบบเต็มรูปแบบ, Transaction

> ขั้นตอนที่ 641-650 ของหลักสูตร | ระดับ: สูง

## สารบัญ

1. CRUD คืออะไร
2. Create: INSERT พร้อมดึงค่า Auto-generated Key
3. Read: SELECT พร้อม Filter, Sort, Pagination
4. Update และ Delete
5. Batch Operations: ประสิทธิภาพสำหรับข้อมูลจำนวนมาก
6. Transaction: ACID Properties
7. Transaction ใน JDBC: `commit()`/`rollback()`
8. Isolation Levels
9. DAO Pattern (Data Access Object)
10. แบบฝึกหัดและสรุป

---

## 1. CRUD คืออะไร

**CRUD** คือตัวย่อของ 4 operation พื้นฐานที่แอปพลิเคชันเกือบทุกตัวต้องทำกับ
ข้อมูล: **Create** (สร้าง), **Read** (อ่าน), **Update** (แก้ไข), **Delete**
(ลบ) — ทบทวนพื้นฐาน JDBC จาก Part 64 มาสร้างเป็นระบบ CRUD ที่สมบูรณ์

## 2. Create: INSERT พร้อมดึงค่า Auto-generated Key

```java
import java.sql.*;

public class CreateDemo {
    static int insertUser(Connection conn, String name, String email) throws SQLException {
        String sql = "INSERT INTO users (name, email) VALUES (?, ?)";

        // ระบุ RETURN_GENERATED_KEYS เพื่อดึงค่า id ที่ฐานข้อมูลสร้างให้อัตโนมัติ (AUTO_INCREMENT)
        try (PreparedStatement pstmt = conn.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS)) {
            pstmt.setString(1, name);
            pstmt.setString(2, email);
            pstmt.executeUpdate();

            try (ResultSet keys = pstmt.getGeneratedKeys()) {
                if (keys.next()) {
                    return keys.getInt(1); // ได้ id ที่ database generate ให้กลับมา
                }
            }
        }
        throw new SQLException("ไม่สามารถสร้าง user ได้");
    }
}
```

## 3. Read: SELECT พร้อม Filter, Sort, Pagination

```java
import java.sql.*;
import java.util.ArrayList;
import java.util.List;

public class ReadDemo {
    record User(int id, String name, String email) { }

    static List<User> findUsers(Connection conn, String nameFilter, int page, int pageSize) throws SQLException {
        String sql = """
                SELECT id, name, email FROM users
                WHERE name LIKE ?
                ORDER BY name ASC
                LIMIT ? OFFSET ?
                """; // ทบทวน Text Block จาก Part 9

        List<User> users = new ArrayList<>();
        try (PreparedStatement pstmt = conn.prepareStatement(sql)) {
            pstmt.setString(1, "%" + nameFilter + "%"); // LIKE + % สำหรับค้นหาแบบ partial match
            pstmt.setInt(2, pageSize);
            pstmt.setInt(3, page * pageSize); // OFFSET สำหรับ pagination (ทบทวนแนวคิดจาก Part 85)

            try (ResultSet rs = pstmt.executeQuery()) {
                while (rs.next()) {
                    users.add(new User(rs.getInt("id"), rs.getString("name"), rs.getString("email")));
                }
            }
        }
        return users;
    }
}
```

## 4. Update และ Delete

```java
import java.sql.*;

public class UpdateDeleteDemo {
    static int updateUserEmail(Connection conn, int userId, String newEmail) throws SQLException {
        String sql = "UPDATE users SET email = ? WHERE id = ?";
        try (PreparedStatement pstmt = conn.prepareStatement(sql)) {
            pstmt.setString(1, newEmail);
            pstmt.setInt(2, userId);
            return pstmt.executeUpdate(); // คืนค่าจำนวนแถวที่ได้รับผลกระทบ
        }
    }

    static boolean deleteUser(Connection conn, int userId) throws SQLException {
        String sql = "DELETE FROM users WHERE id = ?";
        try (PreparedStatement pstmt = conn.prepareStatement(sql)) {
            pstmt.setInt(1, userId);
            int rowsAffected = pstmt.executeUpdate();
            return rowsAffected > 0; // true ถ้าลบสำเร็จ (มีแถวที่ตรงกับ id นี้จริง)
        }
    }
}
```

**หลักปฏิบัติที่ดี**: ตรวจสอบค่าที่ `executeUpdate()` คืนมาเสมอ — ถ้าได้ `0`
หมายความว่าไม่มีแถวไหนตรงกับเงื่อนไข (เช่น `id` ที่ระบุไม่มีอยู่จริง) ควร
จัดการสถานการณ์นี้อย่างเหมาะสม (throw exception หรือแจ้งเตือนผู้เรียกใช้)

## 5. Batch Operations: ประสิทธิภาพสำหรับข้อมูลจำนวนมาก

การเรียก `executeUpdate()` **ทีละครั้ง**สำหรับข้อมูลจำนวนมากช้ามาก เพราะแต่
ละครั้งมีการติดต่อฐานข้อมูล (network round-trip) — **Batch** รวมหลาย
statement เป็น**การส่งครั้งเดียว**

```java
import java.sql.*;
import java.util.List;

public class BatchDemo {
    static void insertUsersBatch(Connection conn, List<String[]> usersData) throws SQLException {
        String sql = "INSERT INTO users (name, email) VALUES (?, ?)";

        conn.setAutoCommit(false); // ปิด auto-commit เพื่อควบคุม transaction เอง (หัวข้อ 7)
        try (PreparedStatement pstmt = conn.prepareStatement(sql)) {
            for (String[] userData : usersData) {
                pstmt.setString(1, userData[0]);
                pstmt.setString(2, userData[1]);
                pstmt.addBatch(); // เพิ่มเข้า batch ไม่ส่งไปฐานข้อมูลทันที

                // ส่ง batch ทุก 1000 รายการ เพื่อไม่ให้ batch ใหญ่เกินไปจนกิน memory มากเกินจำเป็น
                if (usersData.indexOf(userData) % 1000 == 0) {
                    pstmt.executeBatch();
                }
            }
            pstmt.executeBatch(); // ส่ง batch ที่เหลือทั้งหมด
            conn.commit(); // ยืนยัน transaction ทั้งหมด (หัวข้อ 7)
        } catch (SQLException e) {
            conn.rollback(); // ถ้าเกิด error ใด ๆ ยกเลิกทุกอย่างที่ทำไปในรอบนี้
            throw e;
        } finally {
            conn.setAutoCommit(true); // คืนค่า default เสมอ
        }
    }
}
```

**ผลลัพธ์ด้านประสิทธิภาพ**: การ insert 10,000 แถวแบบทีละครั้งอาจใช้เวลาเป็น
นาที ในขณะที่ batch operation ทำเสร็จได้ในไม่กี่วินาที — ความแตกต่างมาจาก
การลด network round-trip จาก 10,000 ครั้งเหลือเพียงไม่กี่ครั้ง

## 6. Transaction: ACID Properties

**Transaction** คือกลุ่มของ operation ที่ต้อง**ทำงานร่วมกันแบบ all-or-
nothing** — รับประกันด้วยคุณสมบัติ **ACID**:

| คุณสมบัติ | ความหมาย |
|---|---|
| **Atomicity** (เอกภาพ) | Transaction ทำสำเร็จทั้งหมด หรือล้มเหลวทั้งหมด (ไม่มีสถานะครึ่ง ๆ กลาง ๆ) |
| **Consistency** (ความสอดคล้อง) | ข้อมูลต้องอยู่ในสถานะที่ถูกต้องตามกฎเสมอ ทั้งก่อนและหลัง transaction |
| **Isolation** (การแยกตัว) | Transaction ที่ทำงานพร้อมกันไม่รบกวนกัน (ทบทวนแนวคิด concurrency จาก Part 46-50) |
| **Durability** (ความคงทน) | เมื่อ commit สำเร็จแล้ว ข้อมูลต้องไม่สูญหายแม้ระบบล่ม |

**ตัวอย่างคลาสสิก**: การโอนเงินระหว่าง 2 บัญชี ต้อง**หักเงินจากบัญชี A**และ
**เพิ่มเงินให้บัญชี B**สำเร็จ**พร้อมกันทั้งคู่** — ถ้าหักจาก A สำเร็จแต่เพิ่ม
ให้ B ล้มเหลว (เช่น server ล่มกลางทาง) เงินจะ**สูญหาย**ไปเลยถ้าไม่มี
Transaction ป้องกัน

## 7. Transaction ใน JDBC: `commit()`/`rollback()`

```java
import java.sql.*;

public class TransactionDemo {
    static void transferMoney(Connection conn, int fromAccountId, int toAccountId, double amount)
            throws SQLException {
        conn.setAutoCommit(false); // เริ่ม transaction (ปิด auto-commit ที่เป็นค่า default)

        try {
            // ขั้นตอนที่ 1: หักเงินจากบัญชีต้นทาง
            String withdrawSql = "UPDATE accounts SET balance = balance - ? WHERE id = ? AND balance >= ?";
            try (PreparedStatement pstmt = conn.prepareStatement(withdrawSql)) {
                pstmt.setDouble(1, amount);
                pstmt.setInt(2, fromAccountId);
                pstmt.setDouble(3, amount); // ป้องกันยอดติดลบ (ทบทวน validation จาก Part 12-13)
                int rows = pstmt.executeUpdate();
                if (rows == 0) {
                    throw new SQLException("ยอดเงินไม่พอ หรือไม่พบบัญชีต้นทาง");
                }
            }

            // ขั้นตอนที่ 2: เพิ่มเงินให้บัญชีปลายทาง
            String depositSql = "UPDATE accounts SET balance = balance + ? WHERE id = ?";
            try (PreparedStatement pstmt = conn.prepareStatement(depositSql)) {
                pstmt.setDouble(1, amount);
                pstmt.setInt(2, toAccountId);
                pstmt.executeUpdate();
            }

            conn.commit(); // ทั้งสองขั้นตอนสำเร็จ -> ยืนยันการเปลี่ยนแปลงทั้งหมดพร้อมกัน
            System.out.println("โอนเงินสำเร็จ");

        } catch (SQLException e) {
            conn.rollback(); // ขั้นตอนใดขั้นตอนหนึ่งล้มเหลว -> ยกเลิกทุกอย่างกลับไปสถานะเดิม
            System.out.println("โอนเงินล้มเหลว: " + e.getMessage() + " (ยกเลิกทุกการเปลี่ยนแปลง)");
            throw e;
        } finally {
            conn.setAutoCommit(true); // คืนค่า default เสมอ (ทบทวน try-finally จาก Part 10)
        }
    }
}
```

**Savepoint**: สำหรับ transaction ที่ซับซ้อน สามารถกำหนดจุด**"บันทึกไว้"**
เพื่อ rollback แค่บางส่วนได้ (ไม่ต้อง rollback ทั้ง transaction):

```java
Savepoint savepoint = conn.setSavepoint();
// ... ทำงานบางอย่าง ...
conn.rollback(savepoint); // ย้อนกลับไปแค่จุดนี้ ไม่ใช่ทั้ง transaction
```

## 8. Isolation Levels

**Isolation Level** กำหนดว่า transaction ที่ทำงาน**พร้อมกัน**จะ**"เห็น"**
การเปลี่ยนแปลงของกันและกันมากแค่ไหน (ทบทวนแนวคิด concurrency issues จาก Part
46) — ยิ่ง isolation สูง ยิ่งปลอดภัยแต่ยิ่งช้า (มี lock มากขึ้น)

```java
import java.sql.Connection;

public class IsolationLevelDemo {
    void setLevels(Connection conn) throws java.sql.SQLException {
        // READ_UNCOMMITTED: เห็นข้อมูลที่ transaction อื่นยังไม่ commit (เสี่ยง "dirty read")
        conn.setTransactionIsolation(Connection.TRANSACTION_READ_UNCOMMITTED);

        // READ_COMMITTED: เห็นแค่ข้อมูลที่ commit แล้ว (ค่า default ของหลายฐานข้อมูล เช่น PostgreSQL)
        conn.setTransactionIsolation(Connection.TRANSACTION_READ_COMMITTED);

        // REPEATABLE_READ: อ่านค่าเดิมซ้ำได้ผลเหมือนกันเสมอตลอด transaction (ค่า default ของ MySQL InnoDB)
        conn.setTransactionIsolation(Connection.TRANSACTION_REPEATABLE_READ);

        // SERIALIZABLE: เข้มงวดที่สุด เสมือน transaction ทำงานทีละตัว (ปลอดภัยสุด แต่ช้าสุด)
        conn.setTransactionIsolation(Connection.TRANSACTION_SERIALIZABLE);
    }
}
```

| Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| READ_UNCOMMITTED | เกิดได้ | เกิดได้ | เกิดได้ |
| READ_COMMITTED | ป้องกัน | เกิดได้ | เกิดได้ |
| REPEATABLE_READ | ป้องกัน | ป้องกัน | เกิดได้ |
| SERIALIZABLE | ป้องกัน | ป้องกัน | ป้องกัน |

**หลักปฏิบัติ**: ใช้ค่า default ของฐานข้อมูลนั้น ๆ ไว้ก่อน (มักเพียงพอสำหรับ
กรณีทั่วไป) ปรับ isolation level เฉพาะเมื่อพบปัญหา concurrency ที่เจาะจงจริง ๆ

## 9. DAO Pattern (Data Access Object)

**DAO** แยก**logic การเข้าถึงข้อมูล**ออกจาก**business logic** (ทบทวน SRP
และ DIP จาก Part 57) — เป็น pattern มาตรฐานก่อนที่ ORM (Part 79-80) จะเข้ามา
แทนที่บางส่วน

```java
public interface UserDao {
    int create(String name, String email);
    Optional<User> findById(int id);
    List<User> findAll();
    boolean update(int id, String email);
    boolean delete(int id);
}

public class JdbcUserDao implements UserDao {
    private final Connection connection; // depend on abstraction ที่จำเป็น เท่านั้น (ทบทวน DIP)

    public JdbcUserDao(Connection connection) {
        this.connection = connection;
    }

    @Override
    public int create(String name, String email) {
        // implementation จริงตามที่แสดงในหัวข้อ 2
        return 0; // placeholder
    }

    // ... implement เมธออื่น ๆ ตาม interface ...

    record User(int id, String name, String email) { }
    Optional<User> findById(int id) { return Optional.empty(); } // placeholder
    List<User> findAll() { return List.of(); }
    boolean update(int id, String email) { return true; }
    boolean delete(int id) { return true; }
}
```

**ประโยชน์**: business logic layer เรียกผ่าน `UserDao` interface โดยไม่รู้
เลยว่าเบื้องหลังใช้ JDBC, JPA (Part 79), หรืออื่น ๆ — ทดสอบง่ายด้วย Mock
(ทบทวนจาก Part 59) โดยไม่ต้องต่อฐานข้อมูลจริงเลย

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียนเมธอด `transferMoney()` เวอร์ชันที่ใช้ `Savepoint` เพื่อบันทึก
log การโอนเงิน โดยที่ถ้า log ล้มเหลว ให้ rollback แค่ log ไม่กระทบการโอนเงิน
ที่สำเร็จแล้ว

**เฉลย (แนวคิด)**:

```java
conn.setAutoCommit(false);
try {
    // ... withdraw + deposit ...
    Savepoint sp = conn.setSavepoint();
    try {
        // ... insert log ...
    } catch (SQLException e) {
        conn.rollback(sp); // rollback แค่ log ไม่กระทบการโอนเงินที่ทำไปแล้ว
    }
    conn.commit();
} catch (SQLException e) {
    conn.rollback();
}
```

**2)** อธิบายว่าทำไม batch operation เร็วกว่าการเรียก `executeUpdate()`
ทีละครั้งสำหรับข้อมูลจำนวนมาก

**เฉลย**: การเรียก `executeUpdate()` แต่ละครั้งต้องมี**network round-trip**
ไปยังฐานข้อมูล (ส่ง SQL statement, รอผลลัพธ์กลับมา) ซึ่งมี **latency** คงที่
ต่อครั้งไม่ว่า statement จะเล็กแค่ไหน — ถ้า insert 10,000 แถวทีละครั้ง จะเกิด
round-trip 10,000 ครั้ง ในขณะที่ batch รวมหลาย statement**ส่งไปพร้อมกันใน
คำสั่งเดียว** ลด round-trip เหลือเพียงไม่กี่ครั้ง (ตามจำนวนรอบที่เรียก
`executeBatch()`) ทำให้เวลารวมลดลงอย่างมาก โดยเฉพาะเมื่อ latency ของ network
สูง (เช่น database อยู่คนละ data center)

**3)** อธิบายความแตกต่างระหว่าง `READ_COMMITTED` และ `REPEATABLE_READ`
isolation level

**เฉลย**: `READ_COMMITTED` รับประกันว่า transaction จะเห็นแค่ข้อมูลที่ถูก
commit แล้ว (ไม่เห็น dirty data ที่ transaction อื่นยังไม่ commit) แต่**ถ้า
อ่านค่าเดิมซ้ำสองครั้งภายใน transaction เดียวกัน อาจได้ค่าต่างกัน**ถ้ามี
transaction อื่น commit การเปลี่ยนแปลงระหว่างสองครั้งนั้น (เรียกว่า
"non-repeatable read") ในขณะที่ `REPEATABLE_READ` รับประกันว่า**ถ้าอ่านค่า
เดิมซ้ำภายใน transaction เดียวกัน จะได้ค่าเหมือนกันเสมอตลอด transaction**
(ล็อกข้อมูลที่อ่านไปแล้วไม่ให้ transaction อื่นแก้ไขจนกว่า transaction ปัจจุบัน
จะจบ) — ทำให้ `REPEATABLE_READ` ปลอดภัยกว่าแต่มี lock มากขึ้นและอาจช้ากว่า

### สรุปเนื้อหา Part 65

- CRUD ครอบคลุม INSERT (พร้อม auto-generated key), SELECT (พร้อม filter/
  sort/pagination), UPDATE, DELETE
- Batch operations ลด network round-trip อย่างมากสำหรับข้อมูลจำนวนมาก
- ACID properties (Atomicity, Consistency, Isolation, Durability) คือหลัก
  ประกันของ transaction
- `commit()`/`rollback()` ควบคุม transaction ด้วยมือ, `Savepoint` ให้
  rollback แค่บางส่วนได้
- Isolation levels แลกความปลอดภัยกับประสิทธิภาพ (ยิ่งสูงยิ่งปลอดภัยแต่ช้าลง)
- DAO Pattern แยก data access logic ออกจาก business logic ตามหลัก SRP/DIP

**ต่อไป**: [Part 66 — Connection Pooling (HikariCP)](./part-066-connection-pooling.md)
