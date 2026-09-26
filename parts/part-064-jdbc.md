# Part 64: JDBC และการเชื่อมต่อฐานข้อมูล

> ขั้นตอนที่ 631-640 ของหลักสูตร | ระดับ: สูง

## สารบัญ

1. JDBC คืออะไร
2. การเชื่อมต่อฐานข้อมูลด้วย `DriverManager`
3. `Statement` และการรันคำสั่ง SQL
4. `ResultSet`: อ่านผลลัพธ์จากฐานข้อมูล
5. `PreparedStatement`: ป้องกัน SQL Injection
6. ทำไม SQL Injection อันตราย (ตัวอย่างจริง)
7. Metadata: `ResultSetMetaData`
8. Mapping ResultSet เป็น Object (Manual ORM)
9. Try-with-resources กับ JDBC Resources
10. แบบฝึกหัดและสรุป

---

## 1. JDBC คืออะไร

**JDBC (Java Database Connectivity)** คือ API มาตรฐานของ Java สำหรับเชื่อม
ต่อและทำงานกับฐานข้อมูลเชิงสัมพันธ์ (relational database) — ทำงานผ่าน
**Driver** เฉพาะของแต่ละฐานข้อมูล (MySQL, PostgreSQL, Oracle) ที่ implement
interface มาตรฐานเดียวกัน (ทบทวนแนวคิด interface จาก Part 16 — เขียนโค้ดครั้ง
เดียว สลับฐานข้อมูลได้โดยเปลี่ยน driver)

```
โค้ดของเรา ──> JDBC API (interface มาตรฐาน) ──> JDBC Driver ──> ฐานข้อมูลจริง
                                                (MySQL/PostgreSQL/...)
```

## 2. การเชื่อมต่อฐานข้อมูลด้วย `DriverManager`

```xml
<!-- pom.xml: ต้องเพิ่ม driver ของฐานข้อมูลที่ใช้ (ทบทวน dependency จาก Part 61) -->
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <version>8.2.0</version>
</dependency>
```

```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;

public class ConnectionDemo {
    public static void main(String[] args) {
        String url = "jdbc:mysql://localhost:3306/mydb"; // JDBC URL: jdbc:<database>://<host>:<port>/<db>
        String user = "root";
        String password = "password";

        try (Connection conn = DriverManager.getConnection(url, user, password)) {
            System.out.println("เชื่อมต่อสำเร็จ!");
            System.out.println("ฐานข้อมูล: " + conn.getMetaData().getDatabaseProductName());
        } catch (SQLException e) {
            System.out.println("เชื่อมต่อล้มเหลว: " + e.getMessage());
        }
        // Connection ถูกปิดอัตโนมัติเมื่อออกจาก try block (try-with-resources - ทบทวน Part 21)
    }
}
```

**`SQLException`** เป็น **checked exception** (ทบทวนจาก Part 10, 21) — ต้อง
catch หรือประกาศ `throws` เสมอ เพราะปัญหาการเชื่อมต่อฐานข้อมูล (network,
credential ผิด) เป็นสถานการณ์ที่คาดการณ์ได้และควรมีทางจัดการ

## 3. `Statement` และการรันคำสั่ง SQL

```java
import java.sql.*;

public class StatementDemo {
    public static void main(String[] args) throws SQLException {
        String url = "jdbc:mysql://localhost:3306/mydb";

        try (Connection conn = DriverManager.getConnection(url, "root", "password");
             Statement stmt = conn.createStatement()) {

            // DDL/DML ที่ไม่คืนผลลัพธ์ (CREATE, INSERT, UPDATE, DELETE)
            stmt.executeUpdate("CREATE TABLE IF NOT EXISTS users (id INT PRIMARY KEY, name VARCHAR(100))");
            int rowsAffected = stmt.executeUpdate("INSERT INTO users VALUES (1, 'Alice')");
            System.out.println("แถวที่ได้รับผลกระทบ: " + rowsAffected);

            // Query ที่คืนผลลัพธ์ (SELECT)
            ResultSet rs = stmt.executeQuery("SELECT * FROM users");
            // ... ประมวลผล ResultSet (หัวข้อ 4) ...
        }
    }
}
```

## 4. `ResultSet`: อ่านผลลัพธ์จากฐานข้อมูล

**`ResultSet`** ทำงานเหมือน**cursor** ที่วนไปทีละแถว (ทบทวนแนวคิด Iterator
จาก Part 27):

```java
import java.sql.*;

public class ResultSetDemo {
    public static void main(String[] args) throws SQLException {
        String url = "jdbc:mysql://localhost:3306/mydb";

        try (Connection conn = DriverManager.getConnection(url, "root", "password");
             Statement stmt = conn.createStatement();
             ResultSet rs = stmt.executeQuery("SELECT id, name, email FROM users")) {

            while (rs.next()) { // เลื่อน cursor ไปแถวถัดไป คืน false เมื่อหมด (เหมือน Iterator.hasNext())
                int id = rs.getInt("id");           // อ่านค่าตามชื่อคอลัมน์ (แนะนำ - อ่านง่ายกว่า)
                String name = rs.getString("name");
                String email = rs.getString(3);       // หรืออ่านตาม index (เริ่มที่ 1 ไม่ใช่ 0! ต่างจาก array)

                System.out.println(id + ": " + name + " (" + email + ")");
            }
        }
    }
}
```

**ข้อควรระวัง**: index ของ `ResultSet` **เริ่มที่ 1** (ไม่ใช่ 0 แบบ array/
List ที่เรียนมาตลอดหลักสูตร — Part 7, 22) เป็นกับดักคลาสสิกของมือใหม่ JDBC

## 5. `PreparedStatement`: ป้องกัน SQL Injection

**`PreparedStatement`** เป็นเวอร์ชันที่ปลอดภัยกว่า `Statement` — ใช้
**placeholder (`?`)** แทนการต่อ string SQL ตรง ๆ

```java
import java.sql.*;

public class PreparedStatementDemo {
    public static void main(String[] args) throws SQLException {
        String url = "jdbc:mysql://localhost:3306/mydb";

        try (Connection conn = DriverManager.getConnection(url, "root", "password")) {

            // INSERT ด้วย PreparedStatement
            String insertSql = "INSERT INTO users (id, name, email) VALUES (?, ?, ?)";
            try (PreparedStatement pstmt = conn.prepareStatement(insertSql)) {
                pstmt.setInt(1, 2);                    // ตั้งค่า parameter ที่ 1 (เริ่มที่ 1 เช่นกัน)
                pstmt.setString(2, "Bob");
                pstmt.setString(3, "bob@example.com");
                pstmt.executeUpdate();
            }

            // SELECT ด้วย PreparedStatement (ค่าที่มาจาก user input ต้องผ่าน parameter เสมอ)
            String selectSql = "SELECT * FROM users WHERE name = ?";
            try (PreparedStatement pstmt = conn.prepareStatement(selectSql)) {
                pstmt.setString(1, "Bob");
                try (ResultSet rs = pstmt.executeQuery()) {
                    while (rs.next()) {
                        System.out.println(rs.getString("email"));
                    }
                }
            }
        }
    }
}
```

## 6. ทำไม SQL Injection อันตราย (ตัวอย่างจริง)

```java
public class SqlInjectionVulnerabilityDemo {
    // อันตรายมาก! ต่อ string SQL ตรง ๆ จาก user input
    static String buildDangerousQuery(String username) {
        return "SELECT * FROM users WHERE name = '" + username + "'";
    }

    public static void main(String[] args) {
        // ถ้า user input ปกติ:
        System.out.println(buildDangerousQuery("Alice"));
        // SELECT * FROM users WHERE name = 'Alice'  <- ทำงานตามที่คาดหวัง

        // แต่ถ้าผู้โจมตีส่ง input ที่เป็น SQL injection:
        String malicious = "' OR '1'='1";
        System.out.println(buildDangerousQuery(malicious));
        // SELECT * FROM users WHERE name = '' OR '1'='1'
        // ผลลัพธ์: ดึงข้อมูล "ทุกแถว" ในตาราง users! เพราะ '1'='1' เป็นจริงเสมอ

        String moreDangerous = "'; DROP TABLE users; --";
        System.out.println(buildDangerousQuery(moreDangerous));
        // SELECT * FROM users WHERE name = ''; DROP TABLE users; --'
        // อาจทำให้ตาราง users ถูกลบทั้งหมด! (ขึ้นกับว่า driver/database อนุญาตหลาย statement หรือไม่)
    }
}
```

**`PreparedStatement` ป้องกันปัญหานี้ได้อย่างสมบูรณ์** เพราะ**ค่าที่ผ่าน
`setString()`/`setInt()` จะถูกส่งไปยังฐานข้อมูลแยกจาก SQL command** (ฐาน
ข้อมูล compile SQL statement ไว้ล่วงหน้าก่อนใส่ค่าเข้าไป — ค่าที่ใส่จะถูก
มองเป็น**data เท่านั้น ไม่ใช่ SQL syntax**) — แม้ผู้ใช้ส่ง `"' OR '1'='1"`
เข้ามา มันจะถูกมองเป็น**ชื่อผู้ใช้ที่แปลกประหลาด**เท่านั้น ไม่ใช่ SQL command

**กฎทอง**: **ห้ามต่อ SQL string จาก user input โดยตรงเด็ดขาด — ใช้
`PreparedStatement` เสมอ** (นี่คือช่องโหว่ยอดนิยมอันดับต้น ๆ ใน OWASP Top 10
ที่จะเรียนเชิงลึกใน Part 101)

## 7. Metadata: `ResultSetMetaData`

ใช้ตรวจสอบ**โครงสร้าง**ของผลลัพธ์โดยไม่ต้องรู้ล่วงหน้า (มีประโยชน์เมื่อเขียน
เครื่องมือทั่วไปที่ทำงานกับ query ใดก็ได้):

```java
import java.sql.*;

public class MetadataDemo {
    public static void main(String[] args) throws SQLException {
        try (Connection conn = DriverManager.getConnection("jdbc:mysql://localhost:3306/mydb", "root", "");
             Statement stmt = conn.createStatement();
             ResultSet rs = stmt.executeQuery("SELECT * FROM users")) {

            ResultSetMetaData metadata = rs.getMetaData();
            int columnCount = metadata.getColumnCount();

            for (int i = 1; i <= columnCount; i++) { // metadata index เริ่มที่ 1 เช่นกัน
                System.out.println("คอลัมน์ " + i + ": " + metadata.getColumnName(i)
                                  + " (" + metadata.getColumnTypeName(i) + ")");
            }
        }
    }
}
```

## 8. Mapping ResultSet เป็น Object (Manual ORM)

รูปแบบที่พบบ่อยมากในทางปฏิบัติ: แปลง `ResultSet` เป็น object (ปูทางสู่แนวคิด
ORM เต็มรูปแบบใน Part 79-80 — Hibernate/JPA ทำสิ่งนี้ให้อัตโนมัติ):

```java
import java.sql.*;
import java.util.ArrayList;
import java.util.List;

public class ManualORMDemo {
    record User(int id, String name, String email) { } // ทบทวน record จาก Part 51

    static List<User> findAllUsers(Connection conn) throws SQLException {
        List<User> users = new ArrayList<>();
        String sql = "SELECT id, name, email FROM users";

        try (PreparedStatement pstmt = conn.prepareStatement(sql);
             ResultSet rs = pstmt.executeQuery()) {
            while (rs.next()) {
                users.add(mapRowToUser(rs)); // แยก logic การ map ออกเป็นเมธอดของตัวเอง (ทบทวน SRP จาก Part 57)
            }
        }
        return users;
    }

    static User mapRowToUser(ResultSet rs) throws SQLException {
        return new User(
            rs.getInt("id"),
            rs.getString("name"),
            rs.getString("email")
        );
    }
}
```

## 9. Try-with-resources กับ JDBC Resources

JDBC มี resource หลายชั้นที่ต้องปิด: `Connection`, `Statement`,
`ResultSet` — ทบทวนจาก Part 21 ว่า **try-with-resources ปิดตามลำดับย้อนกลับ
โดยอัตโนมัติ**:

```java
import java.sql.*;

public class ProperResourceManagementDemo {
    public static void main(String[] args) {
        String sql = "SELECT * FROM users WHERE id = ?";

        // ซ้อน try-with-resources 3 ชั้น: ปิดตามลำดับ ResultSet -> PreparedStatement -> Connection
        try (Connection conn = DriverManager.getConnection("jdbc:mysql://localhost:3306/mydb", "root", "");
             PreparedStatement pstmt = conn.prepareStatement(sql)) {

            pstmt.setInt(1, 1);

            try (ResultSet rs = pstmt.executeQuery()) {
                if (rs.next()) {
                    System.out.println(rs.getString("name"));
                }
            }
        } catch (SQLException e) {
            System.out.println("เกิดข้อผิดพลาด: " + e.getMessage());
        }
    }
}
```

**ความสำคัญ**: ถ้าไม่ปิด resource เหล่านี้อย่างถูกต้อง จะเกิด **connection
leak** — connection ที่ค้างอยู่จะใช้ resource ของฐานข้อมูลไปเรื่อย ๆ จนหมด
(database มี connection limit จำกัด) ทำให้ระบบล่มในที่สุด (ปูทางสู่ปัญหา
Connection Pooling ใน Part 66)

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียนเมธอด `findUserByEmail(Connection conn, String email)` ที่ใช้
`PreparedStatement` ค้นหาผู้ใช้ตามอีเมล คืนค่าเป็น `Optional<User>` (ทบทวน
Part 43)

**เฉลย:**

```java
import java.sql.*;
import java.util.Optional;

public class Exercise1 {
    record User(int id, String name, String email) { }

    static Optional<User> findUserByEmail(Connection conn, String email) throws SQLException {
        String sql = "SELECT id, name, email FROM users WHERE email = ?";
        try (PreparedStatement pstmt = conn.prepareStatement(sql)) {
            pstmt.setString(1, email);
            try (ResultSet rs = pstmt.executeQuery()) {
                if (rs.next()) {
                    return Optional.of(new User(rs.getInt("id"), rs.getString("name"), rs.getString("email")));
                }
                return Optional.empty();
            }
        }
    }
}
```

**2)** อธิบายว่าทำไมโค้ดนี้เสี่ยง SQL injection และแก้ไขให้ปลอดภัย:

```java
String sql = "SELECT * FROM products WHERE category = '" + userInput + "'";
stmt.executeQuery(sql);
```

**เฉลย**: เสี่ยงเพราะ `userInput` ถูกต่อเข้าไปใน SQL string โดยตรง — ถ้า
ผู้โจมตีส่ง input เช่น `"' OR '1'='1"` จะทำให้ query ดึงข้อมูลทุกแถวออกมา
ทั้งที่ตั้งใจกรองตาม category (ทบทวนตัวอย่างในหัวข้อ 6) วิธีแก้:

```java
String sql = "SELECT * FROM products WHERE category = ?";
try (PreparedStatement pstmt = conn.prepareStatement(sql)) {
    pstmt.setString(1, userInput);
    ResultSet rs = pstmt.executeQuery();
}
```

**3)** อธิบายว่าทำไม index ของ `ResultSet.getString(index)` เริ่มที่ 1 แทน 0
ต่างจาก array/List ที่เรียนมาก่อนหน้า

**เฉลย**: นี่เป็นเหตุผลเชิงประวัติศาสตร์ — JDBC ออกแบบตามธรรมเนียมของ SQL
standard และหลายภาษาฐานข้อมูลรุ่นเก่า (รวมถึง ODBC ที่ JDBC ได้รับอิทธิพลมา)
ที่นับคอลัมน์เริ่มที่ 1 มาตั้งแต่ต้น (ต่างจาก Java array/List ที่นับ index
เริ่มที่ 0 ตามธรรมเนียมของภาษาที่สืบเชื้อสายจาก C) — นี่คือความไม่สอดคล้องกัน
ที่ฝังอยู่ใน JDBC API และเป็นกับดักที่มือใหม่พลาดบ่อยมาก จึงแนะนำให้**ใช้
ชื่อคอลัมน์แทน index เสมอ** (`rs.getString("name")` แทน `rs.getString(2)`)
เพื่อความชัดเจนและป้องกันความสับสนนี้

### สรุปเนื้อหา Part 64

- JDBC เป็น API มาตรฐานเชื่อมต่อฐานข้อมูล ทำงานผ่าน driver เฉพาะของแต่ละ
  ฐานข้อมูล
- `Connection`, `Statement`, `ResultSet` ทำงานร่วมกันตามลำดับ ต้องปิดด้วย
  try-with-resources เสมอ
- `ResultSet` index เริ่มที่ 1 (ต่างจาก array/List) — ใช้ชื่อคอลัมน์แทนดีกว่า
- **`PreparedStatement` ป้องกัน SQL Injection** — ห้ามต่อ SQL string จาก user
  input โดยตรงเด็ดขาด
- SQL Injection เป็นช่องโหว่ยอดนิยม เกิดจากการต่อ string SQL ที่ไม่ผ่าน
  parameter binding

**ต่อไป**: [Part 65 — SQL กับ Java: การทำ CRUD แบบเต็มรูปแบบ](./part-065-sql-crud.md)
