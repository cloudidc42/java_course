# Part 100: System Design สำหรับระบบขนาดใหญ่

> ขั้นตอนที่ 991-1000 ของหลักสูตร | ระดับ: ระดับมืออาชีพ/โลก

## สารบัญ

1. กรอบคิดสำหรับ System Design Interview/งานจริง
2. Requirement Gathering: Functional และ Non-Functional
3. Capacity Estimation (Back-of-the-envelope calculation)
4. High-Level Architecture Design
5. Database Design: SQL vs NoSQL สำหรับแต่ละ Use Case
6. Scaling Strategy: Vertical, Horizontal, Sharding
7. Case Study: ออกแบบ URL Shortener
8. Case Study: ออกแบบ News Feed System
9. CAP Theorem และการเลือก Trade-off
10. แบบฝึกหัดและสรุป

---

## 1. กรอบคิดสำหรับ System Design Interview/งานจริง

ทบทวนทุกอย่างที่เรียนมาตลอด 99 Part — System Design คือการ**นำความรู้
ทั้งหมด (data structure, database, caching, microservices, scaling) มา
ประกอบกันแก้ปัญหาจริง** กรอบคิดที่ใช้ได้ทั้ง interview และงานจริง:

```java
public class SystemDesignFramework {
    /*
     * 1. Requirement Gathering (หัวข้อ 2): เข้าใจโจทย์ก่อนออกแบบ
     * 2. Capacity Estimation (หัวข้อ 3): ประเมินขนาดที่ต้องรองรับ
     * 3. High-Level Design (หัวข้อ 4): วาดภาพรวม architecture
     * 4. Deep Dive: เจาะรายละเอียดส่วนที่สำคัญที่สุด (database schema, API design, algorithm)
     * 5. Identify Bottlenecks: หาจุดที่จะมีปัญหาเมื่อ scale และแก้ไข (หัวข้อ 6)
     *
     * ข้อผิดพลาดที่พบบ่อยที่สุด: กระโดดไปออกแบบรายละเอียดทันทีโดยไม่ถามคำถามเพื่อเข้าใจโจทย์ก่อน
     */
}
```

## 2. Requirement Gathering: Functional และ Non-Functional

```java
public class RequirementGatheringDemo {
    /*
     * Functional Requirements: "ระบบต้องทำอะไรได้"
     *   เช่น สำหรับ URL Shortener: "ผู้ใช้ต้องย่อ URL ยาวเป็น URL สั้นได้", "URL สั้นต้อง redirect ไป URL เดิมได้"
     *
     * Non-Functional Requirements: "ระบบต้องทำงานดีแค่ไหน"
     *   - Availability: ระบบต้อง online กี่ % ของเวลา (ทบทวน SLO จาก Part 99)
     *   - Latency: ต้องตอบสนองเร็วแค่ไหน (เช่น redirect ต้องเร็วกว่า 100ms)
     *   - Consistency: ข้อมูลต้อง sync กันทันทีหรือ eventual consistency พอ (ทบทวนหัวข้อ 9)
     *   - Scale: ต้องรองรับผู้ใช้/request กี่คน/ครั้งต่อวัน
     *
     * คำถามสำคัญที่ต้องถามก่อนออกแบบเสมอ: "Read-heavy หรือ Write-heavy?" คำตอบนี้เปลี่ยนการออกแบบทั้งหมด
     */
}
```

## 3. Capacity Estimation (Back-of-the-envelope calculation)

```java
public class CapacityEstimationDemo {
    /*
     * ตัวอย่าง: ออกแบบ URL Shortener ที่มี 100 ล้าน URL ใหม่ต่อเดือน, อ่าน:เขียน = 100:1
     *
     * Write QPS (queries per second):
     *   100,000,000 / (30 วัน * 24 ชม. * 3600 วิ) ≈ 38 writes/second
     *
     * Read QPS: 38 * 100 = 3,800 reads/second
     *
     * Storage (5 ปี): 100M/เดือน * 12 * 5 = 6,000M URLs
     *   ถ้าแต่ละ record ใช้ ~500 bytes (URL + metadata): 6,000M * 500 bytes = 3TB
     *
     * ตัวเลขเหล่านี้บอกว่า: "read-heavy มาก (100:1)" -> ต้องมี caching เป็นหัวใจสำคัญ (ทบทวน Part 87)
     *                     "storage ระดับ TB" -> database เดียวอาจไม่พอ ต้องคิดเรื่อง sharding (หัวข้อ 6)
     */
}
```

## 4. High-Level Architecture Design

```
                     ┌─────────────┐
Client ────────────▶│ API Gateway  │  (ทบทวน Part 93 - auth, rate limit ที่นี่)
                     └──────┬───────┘
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
      ┌──────────────┐┌──────────┐┌──────────────┐
      │ URL Shortener││  Cache   ││   Database    │
      │   Service    ││ (Redis)  ││ (Sharded SQL) │
      │(ทบทวน Part 91)││(Part 87) ││  (หัวข้อ 6)   │
      └──────────────┘└──────────┘└──────────────┘
```

```java
public class HighLevelDesignApproach {
    /*
     * เริ่มจากภาพกว้าง ๆ ก่อน (ไม่ต้องเจาะรายละเอียดทุกจุดในตอนแรก):
     *   Client -> Load Balancer/API Gateway -> Application Service -> Cache -> Database
     *
     * แล้วค่อย "deep dive" ในส่วนที่สำคัญที่สุดตามโจทย์ (เช่น algorithm การสร้าง short URL,
     * database schema, การจัดการ cache invalidation) - ไม่ต้อง deep dive ทุกส่วนเท่ากัน
     * ให้เวลากับส่วนที่เป็นหัวใจของปัญหาตามที่ non-functional requirement บอก (หัวข้อ 2)
     */
}
```

## 5. Database Design: SQL vs NoSQL สำหรับแต่ละ Use Case

```java
public class SqlVsNoSqlDecisionDemo {
    /*
     * เลือก SQL (PostgreSQL/MySQL - ทบทวน Part 64-66) เมื่อ:
     *   - ข้อมูลมีความสัมพันธ์ซับซ้อน ต้อง JOIN บ่อย (เช่น Order-Customer-Product)
     *   - ต้องการ ACID transaction เข้มงวด (เช่น ระบบการเงิน - ทบทวน Part 65)
     *   - Schema ค่อนข้างชัดเจนและไม่เปลี่ยนบ่อย
     *
     * เลือก NoSQL เมื่อ:
     *   - Key-Value (Redis - Part 87): ต้องการความเร็วสูงสุด ข้อมูลเรียบง่าย (cache, session)
     *   - Document (MongoDB): schema ยืดหยุ่น เปลี่ยนบ่อย ข้อมูลเป็น nested object ธรรมชาติ
     *   - Wide-Column (Cassandra): เขียนข้อมูลปริมาณมหาศาลต่อวินาที (time-series, log data)
     *     รองรับ horizontal scale ได้ดีกว่า SQL แบบดั้งเดิมมาก (partition ข้อมูลข้ามเครื่องได้ธรรมชาติ)
     *
     * สำหรับ URL Shortener: ข้อมูลเรียบง่าย (short_code -> long_url) ไม่มีความสัมพันธ์ซับซ้อน
     * เลือก Key-Value store หรือ SQL แบบง่าย ๆ ก็เพียงพอ ไม่จำเป็นต้องใช้ระบบซับซ้อน
     */
}
```

## 6. Scaling Strategy: Vertical, Horizontal, Sharding

```java
public class ScalingStrategiesDemo {
    /*
     * Vertical Scaling: เพิ่มขนาดเครื่องเดิม (CPU/RAM มากขึ้น)
     *   ง่ายที่สุด แต่มีขีดจำกัดทางกายภาพ (เครื่องแรงสุดก็มีเพดาน) และ single point of failure
     *
     * Horizontal Scaling: เพิ่มจำนวนเครื่อง (ทบทวน HPA จาก Part 96)
     *   ต้องออกแบบให้ stateless (ทบทวน Part 83 - JWT stateless authentication) เพื่อ scale ได้ง่าย
     *
     * Database Sharding: แบ่งข้อมูลออกเป็นหลาย database ตาม "shard key"
     *   เช่น แบ่งตาม hash(user_id) % จำนวน shard -> ข้อมูลของ user แต่ละคนอยู่ที่ shard ที่คำนวณได้แน่นอน
     *   ปัญหา: query ที่ต้องรวมข้อมูลข้าม shard ทำได้ยากขึ้นมาก (ทบทวนปัญหา API Composition จาก Part 91)
     *
     * สำหรับ URL Shortener: Horizontal Scaling ของ Application Service ทำได้ง่าย (stateless)
     * ส่วน Database อาจ shard ตาม hash(short_code) เมื่อข้อมูลใหญ่เกินเครื่องเดียวรับได้ (ทบทวนหัวข้อ 3 - 3TB)
     */
}
```

## 7. Case Study: ออกแบบ URL Shortener

```java
import java.util.Base64;

public class UrlShortenerDesign {
    /*
     * Algorithm หลัก: แปลง auto-increment ID (จาก database sequence) เป็น base62 string
     * (a-z, A-Z, 0-9 = 62 ตัวอักษร) เพื่อให้ short code สั้นและไม่ซ้ำกันแน่นอน
     */
    private static final String BASE62_CHARS = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";

    public String encode(long id) {
        StringBuilder sb = new StringBuilder(); // ทบทวน StringBuilder จาก Part 9
        while (id > 0) {
            sb.append(BASE62_CHARS.charAt((int) (id % 62)));
            id /= 62;
        }
        return sb.reverse().toString();
        // id=1000000 -> "4c92" (สั้นกว่า decimal string มาก, ไม่ชนกันเพราะมาจาก unique ID)
    }
}
```

```java
public class UrlShortenerArchitectureFlow {
    /*
     * Write Flow (สร้าง short URL ใหม่):
     *   1. Client ส่ง long URL มาที่ API
     *   2. Application ขอ unique ID ใหม่จาก database sequence (หรือ distributed ID generator)
     *   3. แปลง ID เป็น short code ด้วย base62 encoding (ข้างบน)
     *   4. บันทึก (short_code, long_url) ลง database
     *   5. คืน short URL กลับไปให้ client
     *
     * Read Flow (redirect - เกิดบ่อยกว่ามาก ทบทวนหัวข้อ 3 อัตรา 100:1):
     *   1. Client เข้า short URL -> Application เช็ค Cache (Redis) ก่อน (ทบทวน Part 87)
     *   2. Cache hit -> redirect ทันที (เร็วมาก, ไม่แตะ database เลย)
     *   3. Cache miss -> query database, เก็บผลลง cache, แล้ว redirect
     *   เพราะ read:write = 100:1 (หัวข้อ 3) การมี cache ที่ hit rate สูงมีผลกระทบต่อ performance สูงมาก
     */
}
```

## 8. Case Study: ออกแบบ News Feed System

```java
public class NewsFeedSystemDesign {
    /*
     * Requirement: ผู้ใช้ follow คนอื่น เห็น post ของคนที่ follow เรียงตามเวลาใน feed ของตัวเอง
     *
     * แนวทางที่ 1 - Pull Model (Fan-out on Read):
     *   ตอนเปิด feed -> query post ล่าสุดจากทุกคนที่ user follow -> รวมกัน sort ตามเวลา
     *   ข้อดี: เขียน post เร็ว (insert record เดียว), ข้อเสีย: อ่าน feed ช้า (query กระจาย + merge เยอะ)
     *   เหมาะกับ: ผู้ใช้ที่มี follower น้อย (ไม่ต้องกระจายงานมาก)
     *
     * แนวทางที่ 2 - Push Model (Fan-out on Write):
     *   ตอน post ใหม่ -> เขียนไปที่ "feed cache" ของ follower ทุกคนทันที (ทบทวน Message Queue จาก Part 88, 89)
     *   ข้อดี: อ่าน feed เร็วมาก (แค่อ่าน cache ของตัวเอง), ข้อเสีย: เขียนช้าลงถ้ามี follower เยอะมาก
     *   ปัญหา "Celebrity Problem": คนที่มี follower หลักล้าน -> fan-out เขียนไปหลักล้าน cache ต่อ 1 post!
     *
     * แนวทางที่ 3 - Hybrid: คนทั่วไปใช้ Push Model, คนที่มี follower มาก (celebrity) ใช้ Pull Model
     *   ผสมข้อดีทั้งสองแบบ แก้ปัญหา Celebrity Problem ได้ - นี่คือแนวทางที่ระบบใหญ่จริงส่วนมากใช้
     */
}
```

## 9. CAP Theorem และการเลือก Trade-off

```java
public class CapTheoremDemo {
    /*
     * CAP Theorem: ในระบบ distributed ที่มี Network Partition (การเชื่อมต่อระหว่างเครื่องขาด)
     * เลือกได้แค่ 2 จาก 3 อย่างนี้เท่านั้น:
     *
     *   C (Consistency): ทุก node เห็นข้อมูลเดียวกันเสมอ (อ่านค่าล่าสุดที่ถูกเขียนแน่นอน)
     *   A (Availability): ระบบตอบสนอง request ได้เสมอ (ไม่ error แม้บาง node จะขาดการเชื่อมต่อ)
     *   P (Partition Tolerance): ระบบทำงานต่อได้แม้เครือข่ายบางส่วนขาดการเชื่อมต่อ
     *
     * ในทางปฏิบัติ P เป็นสิ่งที่เลือกไม่ได้ (network partition เกิดขึ้นได้เสมอในระบบจริง)
     * คำถามจริงคือ: เมื่อเกิด partition แล้ว จะเลือก "C" (ปฏิเสธ request ถ้าไม่แน่ใจว่าข้อมูลล่าสุด)
     * หรือ "A" (ตอบ request ต่อไปแม้ข้อมูลอาจไม่ล่าสุดที่สุด - eventual consistency)
     *
     * ตัวอย่าง: ระบบธนาคาร (โอนเงิน) เลือก Consistency (ทบทวน ACID จาก Part 65 - ห้ามข้อมูลผิดพลาดเด็ดขาด)
     *          News Feed/Social Media เลือก Availability (เห็น post ช้าไปนิดหน่อยยอมรับได้ ดีกว่า error)
     */
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** ประเมิน capacity แบบ back-of-the-envelope สำหรับระบบแชท ที่มีผู้ใช้
active 10 ล้านคนต่อวัน ส่งข้อความเฉลี่ยคนละ 20 ข้อความต่อวัน

**เฉลย:**
```java
public class ChatCapacityEstimation {
    /*
     * Total messages/day = 10,000,000 * 20 = 200,000,000 messages/day
     * Average QPS = 200,000,000 / 86,400 seconds ≈ 2,315 messages/second
     * Peak QPS (สมมติ peak = 3x average) ≈ 7,000 messages/second
     *
     * Storage/day (สมมติ 200 bytes/message): 200,000,000 * 200 bytes = 40GB/day
     * Storage/year: 40GB * 365 ≈ 14.6TB/year
     *
     * ผลลัพธ์: QPS ระดับพันต่อวินาทีบอกว่าต้องมี horizontal scaling (หัวข้อ 6)
     * ขนาด storage ระดับ TB/ปีบอกว่าต้องคิดเรื่อง sharding หรือ archiving ข้อมูลเก่า
     */
}
```

**2)** อธิบายว่าทำไม URL Shortener ควรให้ความสำคัญกับ Caching มากกว่า
Database Optimization เพียงอย่างเดียว

**เฉลย**: จากการประเมิน capacity ในหัวข้อ 3 อัตราส่วน**read:write = 100:1**
หมายความว่า**การอ่าน (redirect) เกิดขึ้นบ่อยกว่าการเขียน (สร้าง short
URL) มากถึง 100 เท่า** — แม้จะ optimize database ให้ query เร็วที่สุด
เท่าที่เป็นไปได้ (index ที่ดี, schema ที่เหมาะสม) **database ก็ยังต้อง
รับภาระ query จำนวนมหาศาลจากการอ่านซ้ำ ๆ** ที่ข้อมูลเดิมไม่เปลี่ยนแปลง
เลย (short_code หนึ่งตัวชี้ไปที่ long_url เดิมตลอดไป) ซึ่งเป็นลักษณะ
ข้อมูลที่**เหมาะกับ caching อย่างยิ่ง** (ทบทวน Part 87 - Cache-Aside
Pattern เหมาะกับ read-heavy workload) การเพิ่ม **Cache (Redis)** ที่มี
hit rate สูง (เพราะข้อมูลไม่เปลี่ยน ไม่มีปัญหา stale data ที่ต้องกังวล
มากนัก) ทำให้**การอ่านส่วนใหญ่ไม่ต้องแตะ database เลย** ลดภาระของ
database ลงอย่างมหาศาล และทำให้ latency ของการ redirect เร็วกว่าการ
optimize query เพียงอย่างเดียวมาก

**3)** อธิบายว่าทำไมระบบธนาคาร (โอนเงิน) ควรเลือก Consistency มากกว่า
Availability ตาม CAP Theorem ในขณะที่ระบบ Social Media News Feed เลือก
ตรงกันข้าม

**เฉลย**: ในระบบธนาคาร**ความถูกต้องของข้อมูลสำคัญกว่าความพร้อมใช้งาน
เสมอ** — ถ้าเกิด network partition และระบบยังคง**ตอบ request โอนเงินต่อไป
โดยไม่แน่ใจว่าข้อมูลยอดเงินล่าสุดถูกต้องหรือไม่** (เลือก Availability)
อาจเกิดสถานการณ์ที่**ผู้ใช้โอนเงินที่ตัวเองไม่มีจริง**หรือ**เกิด double
spending** (ทบทวนความสำคัญของ ACID transaction จาก Part 65) ซึ่งเป็น
ความเสียหายทางการเงินที่ร้ายแรงและแก้ไขยาก ดังนั้นระบบธนาคารจึงยอม**ปฏิเสธ
request ชั่วคราว (เลือก Consistency)** ในช่วงที่เกิด partition ดีกว่าเสี่ยง
ข้อมูลผิดพลาด ในทางกลับกัน **News Feed** ถ้าผู้ใช้เห็น post ใหม่**ช้าไป
สองสามวินาที**หรือเห็นจำนวน like ที่ไม่ตรงกับความเป็นจริงเป๊ะ ๆ ชั่วขณะ
(eventual consistency) **ไม่ก่อความเสียหายร้ายแรงใด ๆ** แต่ถ้าระบบ
**ปฏิเสธการให้บริการ (เลือก Consistency)** ทุกครั้งที่มีข้อสงสัยเรื่อง
ความสอดคล้องของข้อมูล ผู้ใช้จะรู้สึกว่าแอปใช้งานไม่ได้บ่อยครั้ง ซึ่งกระทบ
ประสบการณ์ผู้ใช้มากกว่าการเห็นข้อมูลที่ไม่ update ล่าสุดเสียอีก ดังนั้น
Social Media จึงเลือก**Availability**เป็นหลัก

### สรุปเนื้อหา Part 100

- System Design ใช้กรอบคิด: Requirement -> Capacity Estimation ->
  High-Level Design -> Deep Dive -> Identify Bottleneck
- Functional Requirements บอกว่าระบบทำอะไรได้, Non-Functional บอกว่าทำงาน
  ดีแค่ไหน (availability, latency, consistency, scale)
- Capacity Estimation (back-of-the-envelope) ช่วยตัดสินใจว่าต้องใช้
  caching, sharding มากแค่ไหน
- เลือก SQL/NoSQL ตาม use case: ความสัมพันธ์ของข้อมูล, ความต้องการ ACID,
  ความยืดหยุ่นของ schema
- Vertical Scaling ง่ายแต่มีเพดาน, Horizontal Scaling ต้อง stateless,
  Sharding แบ่งข้อมูลข้ามหลาย database
- Case study URL Shortener: base62 encoding + caching สำหรับ read-heavy
  workload
- Case study News Feed: Pull/Push/Hybrid Model แก้ปัญหา Celebrity Problem
- CAP Theorem: เมื่อเกิด network partition ต้องเลือกระหว่าง Consistency
  กับ Availability ตามความต้องการทางธุรกิจของแต่ละระบบ

**หมวดที่ 6 (Professional/World-class) ผ่านครึ่งทางแล้ว! รวม 100/105
Part — เหลือ Security, Reactive Programming, Capstone Project, และ
Career Roadmap**

**ต่อไป**: [Part 101 — Security Best Practices และ OWASP Top 10](./part-101-security-owasp.md)
