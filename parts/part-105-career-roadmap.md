# Part 105: Interview Preparation และ Career Roadmap

> ขั้นตอนที่ 1041-1050+ ของหลักสูตร | ระดับ: ระดับมืออาชีพ/โลก

## สารบัญ

1. ภาพรวมเส้นทางที่ผ่านมา: จาก Part 1 ถึง Part 104
2. Technical Interview: Coding Round
3. Technical Interview: System Design Round
4. Behavioral Interview: STAR Method
5. การเตรียม Resume/Portfolio สำหรับ Java Developer
6. Junior → Mid-level → Senior: เส้นทางการเติบโต
7. Specialization Paths: เลือกทางที่ใช่สำหรับคุณ
8. การเรียนรู้ต่อเนื่อง: Reading List และ Community
9. Contributing to Open Source
10. บทสรุปหลักสูตรและก้าวต่อไป

---

## 1. ภาพรวมเส้นทางที่ผ่านมา: จาก Part 1 ถึง Part 104

ทบทวนเส้นทางทั้งหมดที่เราเดินผ่านมา:

```java
public class JourneyOverview {
    /*
     * Part 1-20:   พื้นฐานภาษา Java (syntax, control flow, OOP เบื้องต้น)
     * Part 21-35:  โครงสร้างข้อมูลและอัลกอริทึม (Collections, Big O, Sorting/Searching)
     * Part 36-55:  Java ขั้นกลาง-สูง (Functional Programming, Concurrency, Design Patterns)
     * Part 56-70:  Java ขั้นสูงและ Tooling (Testing, Build Tools, JVM, Networking)
     * Part 71-90:  Web Development (Spring Boot, REST API, Security, Testing, Messaging)
     * Part 91-105: Professional/World-class (Microservices, Cloud, DevOps, System Design, Capstone)
     *
     * นี่คือเส้นทางที่ตรงกับที่วิศวกรซอฟต์แวร์มืออาชีพต้องรู้จริง - ไม่ใช่แค่ทฤษฎี
     * แต่ผ่านการฝึกเขียนโค้ดที่ "ใช้งานได้จริง" ในทุก Part (ตามเป้าหมายเดิมของหลักสูตรนี้)
     */
}
```

## 2. Technical Interview: Coding Round

```java
public class CodingInterviewPreparation {
    /*
     * ทบทวนพื้นฐานที่ต้องแม่นสำหรับ coding interview:
     *   - Big O Notation (Part 35): วิเคราะห์ time/space complexity ของทุกคำตอบที่เขียน
     *   - Data Structures (Part 22-25, 32-34): Array, LinkedList, HashMap, Tree, Graph
     *   - Algorithms (Part 29-31): Recursion, Sorting, Searching, BFS/DFS
     *
     * กลยุทธ์การตอบคำถาม coding interview:
     *   1. Clarify requirement ก่อนเขียนโค้ด (ทบทวนหลักการ Requirement Gathering จาก Part 100)
     *   2. อธิบาย approach ด้วยคำพูดก่อนเขียนโค้ดจริง (แสดงกระบวนการคิด)
     *   3. เขียนโค้ดที่ compile ได้จริง ไม่ใช่ pseudocode (ทบทวนมาตรฐานทั้งหลักสูตรที่เน้นโค้ดใช้งานได้จริง)
     *   4. วิเคราะห์ time/space complexity ของคำตอบเสมอ
     *   5. เสนอวิธี optimize ถ้ามีเวลาเหลือ (trade-off ระหว่าง time/space)
     */
}
```

```java
public class SampleInterviewQuestion {
    // ตัวอย่างคำถามคลาสสิก: หา 2 ตัวเลขใน array ที่บวกกันได้เท่ากับ target (Two Sum)
    public int[] twoSum(int[] nums, int target) {
        java.util.Map<Integer, Integer> seen = new java.util.HashMap<>(); // ทบทวน HashMap จาก Part 24
        for (int i = 0; i < nums.length; i++) {
            int complement = target - nums[i];
            if (seen.containsKey(complement)) {
                return new int[]{seen.get(complement), i}; // O(n) time, O(n) space - ดีกว่า nested loop O(n²)
            }
            seen.put(nums[i], i);
        }
        throw new IllegalArgumentException("No solution found");
    }
    /*
     * การตอบที่ดี: อธิบายว่าวิธี nested loop (O(n²)) ทำงานอย่างไรก่อน แล้วค่อยเสนอวิธี HashMap
     * (O(n)) ที่ดีกว่า - แสดงให้เห็นว่าเข้าใจ trade-off และสามารถ optimize ได้เมื่อจำเป็น
     */
}
```

## 3. Technical Interview: System Design Round

ทบทวนกรอบคิดทั้งหมดจาก Part 100:

```java
public class SystemDesignInterviewStrategy {
    /*
     * ลำดับขั้นตอนที่ควรทำใน System Design Interview (45-60 นาที):
     *   1. (5 นาที) Requirement Gathering - ถามคำถามเพื่อเข้าใจ scope (ทบทวน Part 100 หัวข้อ 2)
     *   2. (5 นาที) Capacity Estimation - ประเมิน QPS, storage คร่าว ๆ (ทบทวนหัวข้อ 3)
     *   3. (15 นาที) High-Level Design - วาด architecture diagram บนกระดาน (ทบทวนหัวข้อ 4)
     *   4. (15 นาที) Deep Dive - เจาะรายละเอียดที่ interviewer สนใจ (database schema, algorithm)
     *   5. (10 นาที) Identify Bottleneck/Trade-off - พูดถึงข้อจำกัดและวิธีแก้ (ทบทวน CAP Theorem หัวข้อ 9)
     *
     * ข้อผิดพลาดที่พบบ่อย: ใช้เวลานานเกินไปกับขั้นตอนใดขั้นตอนหนึ่ง โดยเฉพาะการ deep dive
     * เข้ารายละเอียดเร็วเกินไปโดยไม่ทำ high-level design ให้ครบก่อน
     */
}
```

```java
public class CommonSystemDesignTopics {
    /*
     * หัวข้อที่ควรฝึกออกแบบให้คล่อง (ใช้ concept จากทั้งหลักสูตร):
     *   - URL Shortener (ทบทวน Part 100 หัวข้อ 7)
     *   - Rate Limiter (ทบทวน Part 85, 93)
     *   - Chat System (ทบทวน Part 90 - WebSocket)
     *   - News Feed (ทบทวน Part 100 หัวข้อ 8)
     *   - Distributed Cache (ทบทวน Part 87)
     *   - Notification System (ทบทวน Part 88, 89)
     *
     * ฝึกออกแบบหัวข้อเหล่านี้ด้วยตัวเองซ้ำ ๆ จนคล่อง - การฝึกซ้อมสำคัญกว่าการอ่านทฤษฎีเฉย ๆ มาก
     */
}
```

## 4. Behavioral Interview: STAR Method

```java
public class StarMethodDemo {
    /*
     * STAR Method สำหรับตอบคำถาม behavioral (เช่น "เล่าเหตุการณ์ที่คุณแก้ปัญหายาก ๆ ในทีม")
     *
     * Situation: อธิบายสถานการณ์/บริบทให้ชัดเจน (โครงการอะไร, ทีมขนาดไหน)
     * Task: หน้าที่ความรับผิดชอบของคุณในสถานการณ์นั้น
     * Action: สิ่งที่คุณทำจริง ๆ (เจาะจง ไม่พูดกว้าง ๆ)
     * Result: ผลลัพธ์ที่เกิดขึ้น (ควรมีตัวเลข/ผลลัพธ์ที่วัดได้ถ้าเป็นไปได้)
     *
     * ตัวอย่างการใช้เนื้อหาจากหลักสูตรนี้มาตอบ:
     *   S: "ระบบ e-commerce ของทีมเจอปัญหา overselling ตอนมีโปรโมชั่น"
     *   T: "ผมรับผิดชอบแก้ไข OrderService ที่จัดการสต็อกสินค้า"
     *   A: "ผมเปลี่ยนจาก naive update เป็น Optimistic Locking ด้วย @Version พร้อม @Retryable
     *       (ทบทวน Part 103-104) และเขียน integration test ด้วย Testcontainers เพื่อพิสูจน์
     *       ว่าแก้ปัญหา race condition ได้จริงภายใต้ concurrent load"
     *   R: "ลดอัตรา overselling จาก 5% เหลือ 0% ในช่วงโปรโมชั่นถัดไป"
     */
}
```

## 5. การเตรียม Resume/Portfolio สำหรับ Java Developer

```java
public class ResumePortfolioTips {
    /*
     * Portfolio ที่แข็งแรง (สำหรับ Junior/Mid-level ที่ยังไม่มี production experience มาก):
     *   - GitHub repository ที่มีโครงการที่ใช้แนวคิดจากหลักสูตรนี้ เช่น Capstone Project (Part 103-104)
     *   - README ที่อธิบาย architecture decision ชัดเจน (ทำไมเลือก Optimistic Locking, ทำไมแยก module)
     *   - README ที่มี diagram (ทบทวนการวาด architecture จาก Part 91-100)
     *   - Test coverage ที่ครบทั้ง unit + integration test (แสดงว่าเข้าใจ Testing Pyramid จาก Part 58, 84)
     *
     * Resume bullet point ที่ดี (ใช้ตัวเลขเสมอเมื่อเป็นไปได้):
     *   ไม่ดี: "พัฒนาระบบ e-commerce ด้วย Spring Boot"
     *   ดี: "ออกแบบและพัฒนา Order Processing Service ด้วย Spring Boot + PostgreSQL ที่รองรับ
     *       concurrent order 100+ ต่อวินาทีโดยไม่เกิด overselling ผ่าน Optimistic Locking"
     */
}
```

## 6. Junior → Mid-level → Senior: เส้นทางการเติบโต

```java
public class CareerLevelExpectations {
    /*
     * Junior Developer (0-2 ปี):
     *   - เขียนโค้ดตาม spec ที่ชัดเจนได้ถูกต้อง (ทบทวน Part 1-55 - พื้นฐานภาษาและ algorithm)
     *   - เข้าใจและใช้ framework (Spring Boot) ตาม pattern ที่มีอยู่แล้วในทีม
     *   - เขียน unit test ได้ (ทบทวน Part 58-60)
     *
     * Mid-level Developer (2-5 ปี):
     *   - ออกแบบ feature ขนาดกลางได้เองตั้งแต่ requirement ถึง deployment (ทบทวน Part 71-99)
     *   - เข้าใจ trade-off ของการออกแบบ (เช่น เมื่อไหร่ใช้ SQL vs NoSQL - ทบทวน Part 100)
     *   - Debug ปัญหาที่ซับซ้อนข้าม layer ได้ (ทบทวน Observability จาก Part 99)
     *   - Code review และให้ feedback ที่มีประโยชน์กับทีมได้
     *
     * Senior Developer (5+ ปี):
     *   - ออกแบบ system architecture ระดับใหญ่ได้ (ทบทวน Part 91-100 ทั้งหมด)
     *   - ตัดสินใจ trade-off ทางเทคนิคที่กระทบทั้งทีม/องค์กร (เช่น Monolith vs Microservices - Part 91)
     *   - Mentor รุ่นน้อง, มองเห็นปัญหาก่อนที่จะเกิดขึ้นจริง (ทบทวน Part 99 - Alerting ก่อนผู้ใช้รู้ตัว)
     *   - สื่อสารกับ stakeholder ที่ไม่ใช่ technical ได้อย่างมีประสิทธิภาพ
     *
     * กุญแจสำคัญของการเติบโต: ไม่ใช่แค่ "รู้เทคโนโลยีมากขึ้น" แต่คือ "เข้าใจ trade-off และผลกระทบ
     * ของการตัดสินใจทางเทคนิคที่กว้างขึ้นเรื่อย ๆ" (จาก code เดียว -> feature -> system -> organization)
     */
}
```

## 7. Specialization Paths: เลือกทางที่ใช่สำหรับคุณ

```java
public class SpecializationPathsDemo {
    /*
     * จากพื้นฐานที่แน่นในหลักสูตรนี้ สามารถแยกไปทางที่ชอบได้หลายทาง:
     *
     * Backend/Distributed Systems: เจาะลึก Part 91-100 ต่อ (Kafka, Kubernetes, System Design)
     *   เหมาะกับคนที่ชอบแก้ปัญหา scale และ reliability ของระบบขนาดใหญ่
     *
     * DevOps/Platform Engineering: เจาะลึก Part 95-99 (Docker, Kubernetes, CI/CD, Observability)
     *   เหมาะกับคนที่ชอบ automation และ infrastructure
     *
     * Security Engineering: เจาะลึก Part 82-83, 101 (Authentication, OWASP)
     *   เหมาะกับคนที่ชอบคิดในมุมผู้โจมตีและป้องกันระบบ
     *
     * Data Engineering: ต่อยอดจาก Part 64-66 (JDBC, SQL) และ Part 88-89 (Kafka)
     *   เหมาะกับคนที่ชอบทำงานกับข้อมูลปริมาณมากและ pipeline
     *
     * ไม่มีทางที่ "ดีที่สุด" - เลือกตามความสนใจของตัวเอง แต่พื้นฐานจาก Part 1-90 จำเป็นสำหรับทุกทาง
     */
}
```

## 8. การเรียนรู้ต่อเนื่อง: Reading List และ Community

```java
public class ContinuousLearningResources {
    /*
     * หนังสือคลาสสิกที่ควรอ่านต่อจากหลักสูตรนี้:
     *   - "Effective Java" โดย Joshua Bloch - best practice ของภาษา Java โดยละเอียด
     *   - "Designing Data-Intensive Applications" โดย Martin Kleppmann - ต่อยอดจาก Part 91-100
     *   - "Clean Code" / "Clean Architecture" โดย Robert C. Martin - ต่อยอดจาก Part 57
     *
     * ติดตามความเคลื่อนไหวของภาษา Java:
     *   - JEP (JDK Enhancement Proposal) - ดูฟีเจอร์ใหม่ที่กำลังจะมาในแต่ละเวอร์ชัน
     *   - Spring Blog / Release Notes - ติดตามการเปลี่ยนแปลงของ framework ที่ใช้งานจริง
     *
     * Community: เข้าร่วม Java User Group ในพื้นที่ของคุณ, ตอบคำถามใน Stack Overflow,
     * เข้าร่วม conference (JavaOne, SpringOne) หรือดู recording ย้อนหลังฟรีบน YouTube
     */
}
```

## 9. Contributing to Open Source

```java
public class OpenSourceContributionDemo {
    /*
     * การ contribute open source เป็นวิธีเรียนรู้และสร้าง portfolio ที่ทรงพลังมาก:
     *
     * ขั้นตอนเริ่มต้นที่แนะนำ:
     *   1. เลือกโครงการที่คุณใช้งานอยู่แล้ว (เช่น library ที่ใช้ใน Capstone Project ของคุณเอง)
     *   2. เริ่มจาก "good first issue" ที่ maintainer มักติด label ไว้สำหรับผู้เริ่มต้น
     *   3. อ่าน CONTRIBUTING.md ของโครงการก่อนเสมอ (แต่ละโครงการมี convention ต่างกัน)
     *   4. เขียน test ให้ครบสำหรับทุก pull request (ทบทวนมาตรฐานจาก Part 58-60, 84)
     *   5. ตอบ code review ด้วยใจเปิดรับ - นี่คือโอกาสเรียนรู้จาก senior engineer ทั่วโลกฟรี ๆ
     *
     * ประโยชน์: ได้เห็นโค้ดคุณภาพสูงจากทีมมืออาชีพจริง, สร้างเครือข่ายกับนักพัฒนาทั่วโลก,
     * และเป็นหลักฐานที่แสดงถึงความสามารถได้ดีกว่า resume เพียงอย่างเดียวมาก
     */
}
```

## 10. บทสรุปหลักสูตรและก้าวต่อไป

```java
public class CourseSummaryAndNextSteps {
    /*
     * หลักสูตรนี้ครอบคลุมตั้งแต่:
     *   "System.out.println("Hello, World!");" (Part 1)
     * ถึง:
     *   การออกแบบและ deploy ระบบ e-commerce แบบ production-grade บน Kubernetes
     *   พร้อม CI/CD, monitoring, security, และ concurrency control ที่ถูกต้อง (Part 104)
     *
     * สิ่งที่สำคัญที่สุดที่ควรจดจำ:
     *   1. พื้นฐานที่แน่น (data structure, algorithm, OOP) คือรากฐานของทุกอย่างที่ซับซ้อนกว่า
     *   2. ทุก design decision มี trade-off - ไม่มี "คำตอบที่ถูกเสมอ" มีแต่ "คำตอบที่เหมาะกับบริบท"
     *   3. Testing ไม่ใช่ทางเลือก แต่เป็นส่วนหนึ่งของการเขียนโค้ดที่ดี
     *   4. ระบบจริงมีมากกว่าโค้ด - ต้องคิดถึง deployment, monitoring, security ตั้งแต่ต้น
     *   5. การเรียนรู้ไม่มีวันจบ - เทคโนโลยีเปลี่ยนแปลงตลอดเวลา แต่หลักการพื้นฐานยังคงอยู่
     */
}
```

## แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียนคำตอบแบบ STAR Method สำหรับคำถาม "เล่าเหตุการณ์ที่คุณต้อง
ตัดสินใจเลือกระหว่าง trade-off ทางเทคนิคสองแบบ"

**เฉลย (ตัวอย่างแนวทาง):**
```
S: ทีมต้องเลือกระหว่าง Monolith และ Microservices สำหรับโครงการใหม่ (ทบทวน Part 91)
T: ในฐานะผู้ที่เสนอ architecture ผมต้องนำเสนอทางเลือกที่เหมาะสมให้ทีม
A: ผมประเมิน team size (ทีมเล็กมี 4 คน), ความซับซ้อนของ domain, และ timeline ที่จำกัด
   แล้วเสนอให้เริ่มจาก well-structured Monolith ก่อน (แยก package ชัดเจนตาม domain)
   พร้อมออกแบบให้ "แยกเป็น microservices ได้ง่ายในอนาคต" ถ้าจำเป็น (Strangler Fig - Part 91 หัวข้อ 9)
R: ทีมส่งมอบ MVP ได้เร็วกว่าประมาณ 30% เทียบกับแผนเดิมที่จะทำ microservices ตั้งแต่ต้น
   และไม่มีปัญหาด้าน operational complexity ที่ทีมเล็กจะรับมือไม่ไหว
```

**2)** อธิบายว่าทำไมการมี Capstone Project ของตัวเอง (เช่น SimpleShop
จาก Part 103-104) สำคัญกว่าการทำแค่ tutorial ตามที่คนอื่นสอนสำหรับการ
สัมภาษณ์งาน

**เฉลย**: การทำ tutorial ตามคนอื่นสอน**พิสูจน์ได้แค่ว่าคุณทำตามคำสั่งได้**
— แต่**ไม่ได้แสดงให้เห็นว่าคุณเข้าใจ "ทำไม" ต้องตัดสินใจแบบนั้น** เมื่อ
interviewer ถามคำถามเจาะลึก (เช่น "ทำไมเลือก Optimistic Locking แทน
Pessimistic Locking" — ทบทวน Part 103 หัวข้อ 7) คนที่**ทำ tutorial ตาม
อย่างเดียว**มักตอบไม่ได้เพราะไม่ได้ผ่านกระบวนการคิดตัดสินใจด้วยตัวเองจริง
ๆ ในขณะที่คนที่**สร้าง Capstone Project ของตัวเอง**จะต้อง**เผชิญกับ
ปัญหาจริงและตัดสินใจเลือกทางแก้ด้วยตัวเอง** (เช่น เจอปัญหา overselling
จริงตอนทดสอบ concurrent load แล้วต้องหาทางแก้เอง) ทำให้**สามารถอธิบาย
เหตุผลเบื้องหลังการตัดสินใจทุกอย่างได้อย่างลึกซึ้ง**และตอบคำถาม follow-up
ได้อย่างมั่นใจ ซึ่งเป็นสิ่งที่ interviewer มองหาจริง ๆ เพื่อประเมินว่า
ผู้สมัครจะสามารถตัดสินใจทางเทคนิคที่ดีได้เองในงานจริงหรือไม่ ไม่ใช่แค่ทำ
ตามคำสั่งที่มีคนบอกไว้แล้วเท่านั้น

**3)** สะท้อนย้อนกลับไปที่ Part 1 ของหลักสูตรนี้ อธิบายว่าความรู้พื้นฐาน
เรื่อง "ตัวแปรและชนิดข้อมูล" (Part 3) เชื่อมโยงไปถึงการตัดสินใจใช้
`BigDecimal` แทน `double` สำหรับราคาสินค้าใน Capstone Project (Part 103)
ได้อย่างไร

**เฉลย**: Part 3 สอนว่า **`double`/`float` เก็บค่าทศนิยมแบบ
floating-point ซึ่งมีความคลาดเคลื่อนที่หลีกเลี่ยงไม่ได้** (เช่น `0.1 +
0.2` ไม่เท่ากับ `0.3` เป๊ะในการคำนวณแบบ binary floating-point) — ความรู้
พื้นฐานนี้ดูเหมือนเป็นรายละเอียดเล็กน้อยตอนเรียน Part 3 แต่**ส่งผลกระทบ
โดยตรงต่อการออกแบบ `Product` entity ใน Part 103** ที่เลือกใช้
**`BigDecimal`** สำหรับเก็บราคาสินค้าแทน `double` เพราะ **`BigDecimal`
คำนวณทศนิยมได้แม่นยำ 100% ไม่มีความคลาดเคลื่อนสะสม** ซึ่ง**จำเป็นอย่างยิ่ง
สำหรับระบบที่เกี่ยวข้องกับเงิน** — ถ้าใช้ `double` ในระบบ e-commerce จริง
ความคลาดเคลื่อนเล็ก ๆ ที่สะสมจากการคำนวณหลายครั้ง (เช่น คูณราคากับจำนวน
สินค้า แล้วรวมยอดหลาย order) อาจทำให้ยอดเงินรวมสุดท้ายผิดพลาดไปหลาย
สตางค์หรือมากกว่านั้น ซึ่งเป็นเรื่องที่ยอมรับไม่ได้ในระบบการเงินจริง นี่
คือตัวอย่างที่ชัดเจนว่า**ความรู้พื้นฐานที่ดูเรียบง่ายที่สุดในหลักสูตร
(Part 3) ยังคงมีผลกระทบสำคัญต่อการออกแบบระบบระดับ production ที่ซับซ้อน
ที่สุด (Part 103-104)** — นี่คือเหตุผลที่หลักสูตรนี้เน้นสร้างพื้นฐานที่
แน่นตั้งแต่ Part แรก ๆ

---

## บทสรุปหลักสูตร: จาก Part 1 ถึง Part 105

หลักสูตร **"เขียนและพัฒนาโปรแกรมและเว็บแอปพลิเคชันด้วย Java"** ฉบับสมบูรณ์
จบลงที่ Part 105 นี้ — เดินทางผ่าน **105 Part, ครอบคลุมขั้นตอนที่ 1 ถึง
1050+** ตั้งแต่ระดับพื้นฐานที่สุด (`Hello World`) จนถึงระดับมืออาชีพ/โลก
(ออกแบบและ deploy ระบบ distributed ที่ scale ได้จริง)

**สิ่งที่ผู้เรียนได้รับจากหลักสูตรนี้**:
- พื้นฐานภาษา Java ที่แน่นและครบถ้วน (Part 1-35)
- ความสามารถเขียนโปรแกรมระดับกลาง-สูงด้วย Java สมัยใหม่ (Part 36-70)
- ความสามารถพัฒนาเว็บแอปพลิเคชันแบบ production-ready ด้วย Spring Boot
  (Part 71-90)
- ความเข้าใจสถาปัตยกรรมระดับองค์กรและแนวทาง DevOps ที่ทันสมัย (Part 91-102)
- ประสบการณ์สร้างระบบจริงแบบ end-to-end ผ่าน Capstone Project (Part 103-104)
- ความพร้อมสำหรับการสัมภาษณ์งานและเส้นทางอาชีพในระยะยาว (Part 105)

**หลักสูตรนี้เป็น living document** — เทคโนโลยีเปลี่ยนแปลงตลอดเวลา
(เวอร์ชัน Java ใหม่, framework ใหม่, แนวทางปฏิบัติที่ดีที่สุดที่พัฒนาต่อ
ไป) แต่**หลักการพื้นฐานที่เรียนในหลักสูตรนี้** (OOP, data structure,
algorithm, testing, security, system design) **จะยังคงมีค่าไปอีกนาน**
ไม่ว่าเทคโนโลยีเฉพาะจะเปลี่ยนไปอย่างไรก็ตาม

ขอให้ทุกคนที่เรียนจบหลักสูตรนี้ประสบความสำเร็จในเส้นทางการเป็นนักพัฒนา
ซอฟต์แวร์มืออาชีพ — และจำไว้ว่า **การเขียนโค้ดที่ดีที่สุดคือการเขียนโค้ด
ที่แก้ปัญหาจริงได้ โดยไม่ลืมว่าใครคือคนที่จะอ่านและดูแลโค้ดนั้นต่อในวันข้างหน้า**
(ซึ่งบางครั้งก็คือตัวคุณเองในอีก 6 เดือนข้างหน้านั่นเอง)

**จบหลักสูตร 105/105 Part — 100% สมบูรณ์**
