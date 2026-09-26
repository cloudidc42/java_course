# Part 66: Connection Pooling (HikariCP), DataSource

> ขั้นตอนที่ 651-660 ของหลักสูตร | ระดับ: สูง

## สารบัญ

1. ปัญหาของการสร้าง Connection ใหม่ทุกครั้ง
2. Connection Pool คืออะไร
3. `DataSource` Interface
4. HikariCP: Connection Pool ที่เร็วที่สุด
5. การตั้งค่า HikariCP ที่สำคัญ
6. Pool Sizing: กำหนดขนาด Pool ที่เหมาะสม
7. Connection Leak Detection
8. Health Check และ Monitoring ของ Pool
9. เปรียบเทียบ HikariCP กับ Connection Pool อื่น
10. แบบฝึกหัดและสรุป

---

## 1. ปัญหาของการสร้าง Connection ใหม่ทุกครั้ง

ทบทวนจาก Part 64: `DriverManager.getConnection()` **สร้าง connection ใหม่
ทุกครั้ง** ซึ่งมี cost สูงมาก — การสร้าง connection เกี่ยวข้องกับ: TCP
handshake, authentication กับฐานข้อมูล, การจัดสรร resource ฝั่งฐานข้อมูล
(อาจใช้เวลาหลาย**สิบ**ถึง**หลายร้อย**มิลลิวินาที)

```java
public class ConnectionCreationCostDemo {
    public static void main(String[] args) throws Exception {
        long start = System.currentTimeMillis();
        for (int i = 0; i < 100; i++) {
            try (var conn = java.sql.DriverManager.getConnection(
                    "jdbc:mysql://localhost:3306/mydb", "root", "password")) {
                // ทำงานเล็ก ๆ กับ connection แล้วปิดทิ้งทันที
            }
        }
        System.out.println("สร้าง connection 100 ครั้ง ใช้เวลา: "
                          + (System.currentTimeMillis() - start) + " ms");
        // ในระบบ web application ที่รับ request หลายพันครั้งต่อวินาที
        // การสร้าง connection ใหม่ทุกครั้งจะทำให้ระบบช้าอย่างมหาศาลและ scale ไม่ได้เลย
    }
}
```

## 2. Connection Pool คืออะไร

**Connection Pool** คือกลไก**สร้าง connection ไว้ล่วงหน้าจำนวนหนึ่ง เก็บไว้
ใน "pool" (สระ) และนำมาใช้ซ้ำ** — เมื่อแอปพลิเคชันต้องการ connection จะ**ยืม**
จาก pool แทนสร้างใหม่ และเมื่อใช้เสร็จจะ**คืน**กลับเข้า pool (ไม่ปิดทิ้งจริง)
— แนวคิดเดียวกับ Thread Pool ที่เรียนใน Part 48

```
Application Code
      │
      ▼ ยืม connection
┌──────────────────────┐
│   Connection Pool      │   [Conn1] [Conn2] [Conn3] [Conn4] [Conn5]
│  (สร้างไว้ล่วงหน้าแล้ว)  │    ← พร้อมใช้      ← กำลังถูกใช้     ← พร้อมใช้
└──────────────────────┘
      │
      ▼ คืน connection กลับ pool (ไม่ได้ปิดจริง)
Application Code (ทำงานเสร็จแล้ว)
```

## 3. `DataSource` Interface

**`javax.sql.DataSource`** เป็น**interface มาตรฐาน**สำหรับได้มาซึ่ง
`Connection` (ทบทวนแนวคิด interface และ DIP จาก Part 16, 57) — Connection
Pool ทุกตัว (HikariCP, Apache DBCP, C3P0) implement interface นี้ ทำให้
สลับ pool implementation ได้โดยไม่แก้โค้ด business logic

```java
import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.SQLException;

public class DataSourceInterfaceDemo {
    void useDataSource(DataSource dataSource) throws SQLException {
        // โค้ดนี้ไม่รู้เลยว่า dataSource คือ HikariCP หรือ pool อื่น (ทบทวน polymorphism จาก Part 15)
        try (Connection conn = dataSource.getConnection()) { // getConnection() คืน connection จาก pool ทันที (เร็วมาก)
            // ใช้งาน connection ตามปกติ (เหมือนที่เรียนใน Part 64-65)
        }
        // close() ในที่นี้ไม่ได้ "ปิด" connection จริง แต่ "คืน" กลับเข้า pool ต่างหาก
    }
}
```

## 4. HikariCP: Connection Pool ที่เร็วที่สุด

**HikariCP** เป็น connection pool ที่**เร็วที่สุด**ในบรรดา Java connection
pool ทั้งหมด (benchmark เอาชนะ Apache DBCP, C3P0, Tomcat JDBC Pool อย่าง
ชัดเจน) — เป็น**ค่า default ของ Spring Boot** (Part 75) ตั้งแต่ Spring Boot 2

```xml
<dependency>
    <groupId>com.zaxxer</groupId>
    <artifactId>HikariCP</artifactId>
    <version>5.1.0</version>
</dependency>
```

```java
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import javax.sql.DataSource;
import java.sql.Connection;

public class HikariCPDemo {
    public static void main(String[] args) throws Exception {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://localhost:3306/mydb");
        config.setUsername("root");
        config.setPassword("password");
        config.setMaximumPoolSize(10); // จำนวน connection สูงสุดใน pool (หัวข้อ 6)

        DataSource dataSource = new HikariDataSource(config); // implement DataSource interface

        // ใช้งานได้ทันที — ครั้งแรกจะสร้าง connection จริง แต่ครั้งต่อ ๆ ไปนำกลับมาใช้ซ้ำ (เร็วมาก)
        try (Connection conn = dataSource.getConnection()) {
            System.out.println("ได้ connection จาก pool: " + conn);
        }
    }
}
```

**ทำไม HikariCP เร็ว**: ออกแบบโค้ดให้**ลด overhead ในทุกจุดที่เป็นไปได้**
(bytecode ที่ compile เฉพาะ, ใช้ `ConcurrentBag` ที่ optimize เองแทน
`ConcurrentHashMap` มาตรฐาน — ทบทวนแนวคิด Concurrent Collections จาก Part
50), ทีมพัฒนาเน้นเรื่อง performance เป็นหลักการออกแบบหลักตั้งแต่ต้น

## 5. การตั้งค่า HikariCP ที่สำคัญ

```java
import com.zaxxer.hikari.HikariConfig;

public class HikariConfigDemo {
    HikariConfig createConfig() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://localhost:3306/mydb");
        config.setUsername("root");
        config.setPassword("password");

        config.setMaximumPoolSize(10);      // จำนวน connection สูงสุด (หัวข้อ 6)
        config.setMinimumIdle(5);             // จำนวน connection ขั้นต่ำที่เตรียมไว้เสมอ (idle)
        config.setConnectionTimeout(30000);    // รอ connection จาก pool สูงสุด 30 วินาที (ถ้าไม่มีว่าง)
        config.setIdleTimeout(600000);          // connection ที่ idle นานกว่า 10 นาที จะถูกปิดทิ้ง
        config.setMaxLifetime(1800000);          // connection แต่ละตัวมีอายุสูงสุด 30 นาที (แล้วสร้างใหม่)
        config.setLeakDetectionThreshold(60000);  // แจ้งเตือนถ้า connection ถูกยืมไปนานกว่า 60 วินาที (หัวข้อ 7)

        return config;
    }
}
```

| Setting | ความหมาย |
|---|---|
| `maximumPoolSize` | จำนวน connection สูงสุดที่ pool จะสร้าง (หัวข้อ 6 อธิบายวิธีคำนวณ) |
| `minimumIdle` | จำนวน connection ขั้นต่ำที่เตรียมพร้อมไว้เสมอ (ลด latency ตอนมี request เข้ามาใหม่) |
| `connectionTimeout` | เวลารอสูงสุดถ้า pool เต็มหมด (ไม่มี connection ว่างให้ยืม) |
| `maxLifetime` | อายุสูงสุดของ connection แต่ละตัว (ป้องกันปัญหาจาก stale connection) |

## 6. Pool Sizing: กำหนดขนาด Pool ที่เหมาะสม

**ความเข้าใจผิดที่พบบ่อย**: "pool ใหญ่ = เร็วกว่าเสมอ" — **ผิด!** HikariCP
เอกสารแนะนำสูตรคำนวณจาก **PostgreSQL wiki** ที่ยอมรับกันอย่างกว้างขวาง:

```
connections = ((core_count * 2) + effective_spindle_count)
```

สำหรับเครื่องที่มี CPU 4 core และใช้ SSD (spindle count มักตั้งเป็น 1):

```
connections = (4 * 2) + 1 = 9 (ปัดเป็น 10)
```

**เหตุผลที่ pool ใหญ่เกินไปไม่ช่วยหรือแม้แต่ทำให้แย่ลง**: ฐานข้อมูลมี**CPU
core จำนวนจำกัด**ในการประมวลผล query — การมี connection มากเกินไป**ทำให้
เกิดการแข่งกันของ CPU/lock ฝั่งฐานข้อมูล**มากขึ้น (ทบทวนแนวคิด context
switching และ thread contention จาก Part 46-47) ซึ่งช้ากว่าการมี pool ขนาด
เหมาะสมที่ประมวลผลคิวได้อย่างมีระเบียบ

```java
public class PoolSizingDemo {
    public static void main(String[] args) {
        int coreCount = Runtime.getRuntime().availableProcessors();
        int recommendedPoolSize = (coreCount * 2) + 1;
        System.out.println("จำนวน CPU core: " + coreCount);
        System.out.println("ขนาด pool ที่แนะนำเริ่มต้น: " + recommendedPoolSize);
        System.out.println("ควรวัดผลจริง (benchmark) และปรับตามภาระงานจริงของระบบ");
    }
}
```

## 7. Connection Leak Detection

**Connection Leak** เกิดเมื่อโค้ด**ยืม connection จาก pool แต่ไม่คืน**
(ลืมเรียก `close()`, ทบทวนความสำคัญของ try-with-resources จาก Part 21, 64)
— ถ้าเกิดซ้ำ ๆ **pool จะหมด connection ว่าง** และ request ใหม่ทั้งหมดจะ
ต้องรอ (`connectionTimeout`) จนกว่าจะ timeout — ระบบล่มในที่สุด

```java
public class ConnectionLeakDemo {
    // อันตราย! ไม่ปิด connection (ไม่มี try-with-resources หรือ finally)
    void leakyMethod(javax.sql.DataSource dataSource) throws java.sql.SQLException {
        java.sql.Connection conn = dataSource.getConnection();
        // ทำงานกับ conn ...
        // ลืม conn.close()! connection นี้จะไม่ถูกคืนกลับ pool เลยตลอดไป
    }

    // ถูกต้อง: ใช้ try-with-resources เสมอ (ทบทวนจาก Part 21, 64)
    void safeMethod(javax.sql.DataSource dataSource) throws java.sql.SQLException {
        try (java.sql.Connection conn = dataSource.getConnection()) {
            // ทำงานกับ conn ...
        } // conn.close() ถูกเรียกอัตโนมัติเสมอ ไม่ว่าจะสำเร็จหรือเกิด exception
    }
}
```

`leakDetectionThreshold` (ทบทวนจากหัวข้อ 5) ช่วย**แจ้งเตือน**ในตอน
development/staging ว่ามี connection ที่ถูกยืมไปนานผิดปกติ (มักบ่งบอกว่ามี
leak) — ควรเปิดใช้ในสภาพแวดล้อมที่ไม่ใช่ production high-traffic (มี overhead
เล็กน้อยจากการติดตาม)

## 8. Health Check และ Monitoring ของ Pool

```java
import com.zaxxer.hikari.HikariDataSource;
import com.zaxxer.hikari.HikariPoolMXBean;

public class PoolMonitoringDemo {
    void checkPoolHealth(HikariDataSource dataSource) {
        HikariPoolMXBean poolMXBean = dataSource.getHikariPoolMXBean();

        System.out.println("Connection ที่ใช้งานอยู่: " + poolMXBean.getActiveConnections());
        System.out.println("Connection ที่ว่าง (idle): " + poolMXBean.getIdleConnections());
        System.out.println("Connection ทั้งหมดใน pool: " + poolMXBean.getTotalConnections());
        System.out.println("Thread ที่กำลังรอ connection: " + poolMXBean.getThreadsAwaitingConnection());

        // ถ้า threadsAwaitingConnection สูงอย่างต่อเนื่อง แสดงว่า pool อาจเล็กเกินไป
        // หรือมี connection leak (ทบทวนหัวข้อ 7) ที่ทำให้ pool ไม่พอใช้งาน
    }
}
```

การ monitor ค่านี้อย่างต่อเนื่อง (ปูทางสู่ Part 99 — Monitoring และ
Observability) ช่วยตรวจจับปัญหาได้ก่อนที่ระบบจะล่มจริง

## 9. เปรียบเทียบ HikariCP กับ Connection Pool อื่น

| Pool | ความเร็ว | ความนิยม | หมายเหตุ |
|---|---|---|---|
| **HikariCP** | เร็วที่สุด | สูงมาก (default ของ Spring Boot) | แนะนำสำหรับโปรเจกต์ใหม่เสมอ |
| Apache DBCP2 | ช้ากว่า HikariCP | ปานกลาง | ยังใช้ในระบบเก่าบางระบบ |
| C3P0 | ช้าที่สุดในกลุ่มนี้ | ลดลง (legacy) | มักพบในระบบเก่ามาก ๆ |
| Tomcat JDBC Pool | เร็วปานกลาง | ใช้เมื่อ deploy บน Tomcat โดยเฉพาะ | ผูกกับ Tomcat ecosystem |

**ในโค้ดใหม่ ให้ใช้ HikariCP เป็นค่าเริ่มต้นเสมอ** — ไม่มีเหตุผลที่ดีพอที่
จะเลือกตัวอื่นในโปรเจกต์ Java สมัยใหม่

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียนโค้ดตั้งค่า HikariCP สำหรับเครื่องที่มี 8 CPU core โดยใช้สูตร
คำนวณ pool size จากหัวข้อ 6

**เฉลย:**

```java
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;

public class Exercise1 {
    public static void main(String[] args) {
        int coreCount = 8;
        int poolSize = (coreCount * 2) + 1; // = 17

        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://localhost:3306/mydb");
        config.setUsername("root");
        config.setPassword("password");
        config.setMaximumPoolSize(poolSize);
        config.setMinimumIdle(poolSize / 2);

        HikariDataSource dataSource = new HikariDataSource(config);
        System.out.println("Pool size: " + poolSize);
    }
}
```

**2)** อธิบายว่าทำไม pool ที่ใหญ่เกินไปอาจทำให้ระบบช้าลง ไม่ใช่เร็วขึ้น

**เฉลย**: ฐานข้อมูลมี CPU core และหน่วยความจำจำกัดในการประมวลผล query แต่
ละ query ต้องใช้ resource เหล่านี้ — ถ้ามี connection จำนวนมากพยายามรัน
query พร้อมกันเกินกว่าที่ CPU core ของฐานข้อมูลจะรองรับได้อย่างมี
ประสิทธิภาพ จะเกิด**การแข่งกันแย่ง CPU/lock** (context switching overhead
มากขึ้น — ทบทวนแนวคิดจาก Part 46-47) ทำให้แต่ละ query ใช้เวลานานขึ้นโดยรวม
แทนที่จะเร็วขึ้น — เหมือนกับการให้คนงาน 100 คนทำงานในห้องที่มีพื้นที่แค่
10 คน ยิ่งเยอะยิ่งชนกันวุ่นวาย ไม่ได้ทำงานเสร็จเร็วขึ้นเลย

**3)** อธิบายว่า `leakDetectionThreshold` ช่วยตรวจจับปัญหาอะไร และทำงาน
อย่างไร

**เฉลย**: `leakDetectionThreshold` กำหนด**เวลาสูงสุด**ที่ connection ควรถูก
"ยืม" ออกจาก pool ไปใช้งาน — ถ้า connection ตัวใดถูกยืมไปนานเกินค่านี้โดยยัง
ไม่ถูกคืน (`close()`) HikariCP จะพิมพ์ warning log พร้อม stack trace ของจุด
ที่ยืม connection นั้นไป ช่วยให้นักพัฒนา**ระบุตำแหน่งของ connection leak**
(ทบทวนจากหัวข้อ 7) ได้อย่างรวดเร็ว โดยไม่ต้องรอให้ปัญหาสะสมจนระบบล่มจริงก่อน
จึงจะรู้ว่ามีปัญหาเกิดขึ้น

### สรุปเนื้อหา Part 66

- Connection Pool สร้าง connection ไว้ล่วงหน้าและนำมาใช้ซ้ำ แก้ปัญหา cost
  สูงของการสร้าง connection ใหม่ทุกครั้ง
- `DataSource` เป็น interface มาตรฐานที่ connection pool ทุกตัว implement
  ทำให้สลับ implementation ได้โดยไม่แก้โค้ด
- HikariCP เป็น connection pool ที่เร็วที่สุดและเป็น default ของ Spring Boot
- Pool size ที่เหมาะสมคำนวณจาก CPU core (`(core*2)+1`) — ไม่ใช่ "ใหญ่กว่า
  ดีกว่าเสมอ"
- Connection Leak (ลืม close) ทำให้ pool หมด connection ว่าง —
  `leakDetectionThreshold` ช่วยตรวจจับปัญหานี้
- Monitor pool metrics (active/idle connections, threads awaiting) ช่วย
  ตรวจจับปัญหาก่อนระบบล่ม

**ต่อไป**: [Part 67 — JVM Internals: Memory Model, Garbage Collection](./part-067-jvm-internals.md)
