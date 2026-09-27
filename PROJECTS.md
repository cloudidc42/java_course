# 100 โปรเจคที่ใช้งานได้จริงด้วย Java

เอกสารนี้รวบรวม **100 โปรเจค** ที่สามารถเขียนและพัฒนาได้จริงด้วยความรู้จาก
หลักสูตร Java ทั้ง 105 Part (ดู [`CURRICULUM.md`](./CURRICULUM.md)) —
จัดเรียงจากระดับพื้นฐานไปถึงระดับมืออาชีพ แบ่งเป็น **10 หมวดหมู่ x 10
โปรเจค** แต่ละโปรเจคระบุ: คำอธิบายสั้น ๆ, เทคโนโลยี/แนวคิดหลักที่ใช้,
ระดับความยาก, และ Part ในหลักสูตรที่เกี่ยวข้อง

สถานะ: ✅ = มีคู่มือ/โค้ดแบบละเอียดแล้วใน `projects/` | ⏳ = อยู่ในแผน
ยังไม่เขียนคู่มือแบบละเอียด (แต่เขียนได้จริงด้วยความรู้ในหลักสูตร)

---

## หมวดที่ 1: Console & CLI Applications (ระดับเริ่มต้น)

ใช้ความรู้จาก Part 1-20 (พื้นฐานภาษา Java)

| # | โปรเจค | คำอธิบาย | เทคโนโลยีหลัก | สถานะ |
|---|---|---|---|---|
| 1 | เครื่องคิดเลขคอนโซล | รับ input ทางคณิตศาสตร์ คำนวณและแสดงผล พร้อมจัดการ error | Scanner, if-else, Exception | ⏳ |
| 2 | ระบบจัดการรายชื่อติดต่อ (Contact Book) | เพิ่ม/ลบ/ค้นหา/แก้ไขรายชื่อ เก็บใน memory | Array/ArrayList, Class | ⏳ |
| 3 | เกมทายตัวเลข (Number Guessing Game) | สุ่มเลขให้ผู้เล่นทาย พร้อม hint มากกว่า/น้อยกว่า | Random, Loop | ⏳ |
| 4 | ตัวแปลงหน่วย (Unit Converter) | แปลงความยาว น้ำหนัก อุณหภูมิ สกุลเงิน | Method overloading, enum | ⏳ |
| 5 | ระบบจัดการงาน To-Do List (คอนโซล) | เพิ่ม/ทำเครื่องหมายเสร็จ/ลบงาน บันทึกลงไฟล์ | ArrayList, File I/O | ⏳ |
| 6 | เกมทอยเต๋า/ไพ่จำลอง | จำลองการทอยเต๋า สับไพ่ แจกไพ่ | Random, Collections | ⏳ |
| 7 | ระบบคำนวณเกรดนักเรียน | รับคะแนน คำนวณเกรดและ GPA | Array 2D, Method | ⏳ |
| 8 | ตัวแยกวิเคราะห์ข้อความ (Word Counter) | นับคำ, ตัวอักษร, ความถี่คำในข้อความ | String, HashMap | ⏳ |
| 9 | เครื่องสุ่มรหัสผ่าน (Password Generator) | สุ่มรหัสผ่านตามเงื่อนไขความยาว/ความซับซ้อน | StringBuilder, Random | ⏳ |
| 10 | ระบบจองที่นั่งโรงหนังแบบง่าย (คอนโซล) | แสดงผังที่นั่ง จอง/ยกเลิกที่นั่ง | Array 2D, OOP เบื้องต้น | ⏳ |

## หมวดที่ 2: Data Structures & Algorithms Projects

ใช้ความรู้จาก Part 21-35 (โครงสร้างข้อมูลและอัลกอริทึม)

| # | โปรเจค | คำอธิบาย | เทคโนโลยีหลัก | สถานะ |
|---|---|---|---|---|
| 11 | ระบบจัดคิว Priority Queue สำหรับห้องฉุกเฉิน | จัดลำดับผู้ป่วยตามความรุนแรง | PriorityQueue, Comparator | ⏳ |
| 12 | เครื่องมือค้นหาเส้นทางที่สั้นที่สุด | หาเส้นทางในแผนที่ (Dijkstra's Algorithm) | Graph, PriorityQueue | ⏳ |
| 13 | ระบบ Autocomplete คำค้นหา | เสนอคำที่พิมพ์ต่อจากคำนำหน้า | Trie (Tree structure) | ⏳ |
| 14 | เครื่องมือเรียงลำดับไฟล์ขนาดใหญ่ (External Sort) | เรียงข้อมูลที่ใหญ่กว่า memory | Merge Sort, File I/O | ⏳ |
| 15 | ระบบตรวจสอบวงเล็บที่ถูกต้อง (Balanced Parentheses) | ตรวจสอบ syntax ของโค้ด/สมการ | Stack | ⏳ |
| 16 | เครื่องคำนวณสมการ (Expression Evaluator) | คำนวณสมการคณิตศาสตร์จาก String | Stack, Recursion | ⏳ |
| 17 | ระบบจัดการแผนผังองค์กร (Org Chart) | แสดง/ค้นหาโครงสร้างองค์กรแบบ tree | Tree, DFS/BFS | ⏳ |
| 18 | เกม Sudoku Solver | แก้ปริศนา Sudoku อัตโนมัติ | Backtracking | ⏳ |
| 19 | ระบบแนะนำเพื่อน (Friend Recommendation) | แนะนำเพื่อนจาก mutual connections | Graph, BFS | ⏳ |
| 20 | เครื่องมือบันทึกและย้อนกลับ (Undo/Redo System) | จัดการประวัติการแก้ไขเอกสาร | Stack, Deque | ⏳ |

## หมวดที่ 3: OOP Design & Design Patterns Projects

ใช้ความรู้จาก Part 11-20, 54-57 (OOP และ Design Patterns)

| # | โปรเจค | คำอธิบาย | เทคโนโลยีหลัก | สถานะ |
|---|---|---|---|---|
| 21 | ระบบจำลองร้านกาแฟ (Coffee Shop Simulator) | ออกแบบเมนู, ตัวเลือกเสริม ด้วย Builder Pattern | Builder Pattern | ⏳ |
| 22 | ระบบแจ้งเตือนหลายช่องทาง (Multi-channel Notifier) | ส่ง Email/SMS/Push ผ่าน interface เดียว | Strategy Pattern | ⏳ |
| 23 | เกมจำลองสัตว์ในสวนสัตว์ (Zoo Simulation) | สัตว์แต่ละชนิดมีพฤติกรรมต่างกัน | Inheritance, Polymorphism | ⏳ |
| 24 | ระบบจัดการ Plugin แบบขยายได้ | โหลดและรัน plugin โดยไม่แก้ core code | Factory + Open/Closed Principle | ⏳ |
| 25 | ระบบ Logging แบบ Singleton | จัดการ log ทั้งแอปผ่าน instance เดียว | Singleton Pattern | ⏳ |
| 26 | เครื่องมือแปลงไฟล์อัตโนมัติ (Document Converter) | แปลง PDF/DOCX/TXT ผ่าน interface เดียวกัน | Adapter Pattern | ⏳ |
| 27 | ระบบสั่งอาหารที่ปรับแต่งได้ (Pizza Builder) | เลือก topping, ขนาด, ราคาคำนวณอัตโนมัติ | Builder + Decorator Pattern | ⏳ |
| 28 | ระบบสมาชิกและสิทธิ์แบบ Role-based | จัดการสิทธิ์ผู้ใช้ตาม Role ต่างกัน | Composite/Strategy Pattern | ⏳ |
| 29 | Event Bus ภายในแอป (In-app Event System) | ส่ง event ระหว่าง component โดยไม่ผูกกันตรง | Observer Pattern | ⏳ |
| 30 | เกม RPG เบื้องต้น (ตัวละคร, อาวุธ, มอนสเตอร์) | ระบบต่อสู้ inventory และ skill tree | Full OOP + Design Patterns | ⏳ |

## หมวดที่ 4: File Handling & Automation Tools

ใช้ความรู้จาก Part 36-38, 45 (File I/O, Regex, Serialization)

| # | โปรเจค | คำอธิบาย | เทคโนโลยีหลัก | สถานะ |
|---|---|---|---|---|
| 31 | เครื่องมือค้นหาและแทนที่ข้อความในไฟล์จำนวนมาก | Bulk find-replace ข้ามหลายไฟล์/โฟลเดอร์ | NIO.2, Regex | ⏳ |
| 32 | ระบบสำรองข้อมูลอัตโนมัติ (Backup Tool) | คัดลอกไฟล์ที่เปลี่ยนแปลงไปยังปลายทาง | File I/O, Scheduled Task | ⏳ |
| 33 | เครื่องมือวิเคราะห์ Log File | ค้นหา error pattern และสรุปสถิติจาก log | Regex, Stream API | ⏳ |
| 34 | ระบบแปลงไฟล์ CSV เป็น JSON และกลับกัน | แปลงข้อมูลระหว่างฟอร์แมต | Jackson/Gson, File I/O | ⏳ |
| 35 | เครื่องมือเปรียบเทียบไฟล์ (Diff Tool) | หาความแตกต่างระหว่างไฟล์ข้อความสองไฟล์ | String algorithm, File I/O | ⏳ |
| 36 | ระบบจัดระเบียบไฟล์อัตโนมัติ (File Organizer) | ย้ายไฟล์ตามประเภท/วันที่โดยอัตโนมัติ | NIO.2, WatchService | ⏳ |
| 37 | เครื่องมือ Merge/Split ไฟล์ PDF อัตโนมัติ | รวม/แยกไฟล์ PDF ตามเงื่อนไข | External library, File I/O | ⏳ |
| 38 | ระบบบันทึก/โหลด Save Game (Serialization) | เก็บสถานะเกมและโหลดกลับมาได้ | Serializable, ObjectStream | ⏳ |
| 39 | เครื่องมือสร้างรายงานอัตโนมัติจากข้อมูล | อ่านข้อมูล สร้างรายงานสรุปเป็นไฟล์ | Stream API, File I/O | ⏳ |
| 40 | ระบบเข้ารหัส/ถอดรหัสไฟล์ส่วนตัว | เข้ารหัสไฟล์ด้วย AES ก่อนเก็บ | javax.crypto | ⏳ |

## หมวดที่ 5: Desktop GUI Applications (JavaFX/Swing)

ต่อยอดจาก Part 11-28 (OOP) ด้วย JavaFX/Swing (เนื้อหาเพิ่มเติมนอกหลักสูตรหลัก)

| # | โปรเจค | คำอธิบาย | เทคโนโลยีหลัก | สถานะ |
|---|---|---|---|---|
| 41 | โปรแกรมจดโน้ต (Notepad Clone) | เปิด/บันทึกไฟล์ข้อความ, find-replace | JavaFX/Swing, File I/O | ⏳ |
| 42 | เครื่องคิดเลข GUI | คำนวณพื้นฐาน-วิทยาศาสตร์ พร้อมหน้าจอกราฟิก | JavaFX, Event Handling | ⏳ |
| 43 | โปรแกรมวาดภาพเบื้องต้น (Paint Clone) | วาดเส้น/รูปทรง เลือกสี บันทึกเป็นรูปภาพ | JavaFX Canvas | ⏳ |
| 44 | ระบบจัดการสต็อกสินค้า (Desktop) | เพิ่ม/แก้ไข/ลบสินค้า เชื่อมต่อฐานข้อมูล | JavaFX + JDBC | ⏳ |
| 45 | เกม Tic-Tac-Toe แบบ GUI | เล่นกับเพื่อนหรือ AI พื้นฐาน | JavaFX, Minimax Algorithm | ⏳ |
| 46 | โปรแกรมจัดการรายรับ-รายจ่ายส่วนตัว | บันทึกธุรกรรม แสดงกราฟสรุป | JavaFX Charts, JDBC | ⏳ |
| 47 | เครื่องเล่นเพลง MP3 เบื้องต้น | เล่น/หยุด/ข้ามเพลง จัดการ playlist | JavaFX Media | ⏳ |
| 48 | ระบบจองห้องประชุม (Desktop) | แสดงตารางเวลา จอง/ยกเลิกห้อง | JavaFX, JDBC | ⏳ |
| 49 | โปรแกรมแปลงไฟล์รูปภาพ (Batch Image Converter) | แปลงฟอร์แมต/ปรับขนาดรูปหลายไฟล์พร้อมกัน | JavaFX, ImageIO | ⏳ |
| 50 | Kanban Board แบบ Desktop (Trello Clone) | ลาก-วางการ์ดงานระหว่างคอลัมน์ | JavaFX Drag & Drop | ⏳ |

## หมวดที่ 6: Database & Persistence Projects

ใช้ความรู้จาก Part 64-66, 79-80 (JDBC, Spring Data JPA, Hibernate)

| # | โปรเจค | คำอธิบาย | เทคโนโลยีหลัก | สถานะ |
|---|---|---|---|---|
| 51 | ระบบห้องสมุด (Library Management System) | จัดการหนังสือ, สมาชิก, การยืม-คืน | JPA, Relational Mapping | ⏳ |
| 52 | ระบบจัดการโรงพยาบาล (คนไข้, แพทย์, นัดหมาย) | ความสัมพันธ์ complex ระหว่าง entity | JPA, Transaction | ⏳ |
| 53 | ระบบบันทึกเวลาเข้า-ออกงาน (Attendance System) | บันทึกและรายงานเวลาทำงานพนักงาน | JDBC/JPA, Reporting | ⏳ |
| 54 | ระบบจัดการหลักสูตรมหาวิทยาลัย (Student-Course) | ความสัมพันธ์ many-to-many ระหว่างนักเรียน/วิชา | JPA @ManyToMany | ⏳ |
| 55 | ระบบจัดการคลังสินค้าหลายสาขา (Multi-warehouse) | ติดตามสต็อกข้ามหลายสถานที่ | JPA, Aggregation Query | ⏳ |
| 56 | ระบบจองตั๋วรถทัวร์/รถไฟ | จัดการที่นั่ง ป้องกันการจองซ้ำ | Transaction, Locking | ⏳ |
| 57 | ระบบบัญชีธนาคารเบื้องต้น (Bank Account System) | โอนเงิน, ประวัติธุรกรรม, ACID | Transaction, ACID | ⏳ |
| 58 | ระบบจัดการฟาร์มปศุสัตว์ (Livestock Tracker) | บันทึกข้อมูลสัตว์, สุขภาพ, การผลิต | JPA, Audit Trail | ⏳ |
| 59 | ระบบจัดการสัญญาเช่า (Lease Management) | ติดตามสัญญา, การแจ้งเตือนวันหมดอายุ | JPA, Scheduled Job | ⏳ |
| 60 | Data Migration Tool ระหว่างฐานข้อมูล | ย้ายข้อมูลจาก database เก่าไปใหม่ | JDBC, Batch Processing | ⏳ |

## หมวดที่ 7: REST API & Backend Projects (Spring Boot)

ใช้ความรู้จาก Part 73-90 (Spring Boot, REST API, Security)

| # | โปรเจค | คำอธิบาย | เทคโนโลยีหลัก | สถานะ |
|---|---|---|---|---|
| 61 | REST API ระบบ Blog (CRUD บทความ+คอมเมนต์) | API มาตรฐานพร้อม pagination, validation | Spring Boot, JPA, Validation | ⏳ |
| 62 | API ระบบ URL Shortener | ย่อ URL พร้อม analytics การคลิก | Spring Boot, Caching (Redis) | ⏳ |
| 63 | API ระบบจัดการงาน Task Manager แบบทีม | มอบหมายงาน, ติดตามสถานะ, แจ้งเตือน | Spring Boot, Security, JWT | ⏳ |
| 64 | API E-Wallet เบื้องต้น (โอนเงินระหว่างบัญชี) | ธุรกรรมทางการเงินที่ต้อง consistency สูง | Transaction, Optimistic Lock | ⏳ |
| 65 | API ระบบจองโรงแรม (ห้องพัก, ราคาตามช่วงเวลา) | ตรวจสอบห้องว่าง ป้องกัน double booking | Spring Boot, Concurrency | ⏳ |
| 66 | API ระบบสอบออนไลน์ (Quiz/Exam System) | สร้างข้อสอบ, ตรวจคะแนนอัตโนมัติ | Spring Boot, JPA | ⏳ |
| 67 | API ระบบให้คะแนนรีวิว (Rating & Review) | รีวิวสินค้า/ร้านค้า พร้อมคำนวณคะแนนเฉลี่ย | Spring Boot, Aggregation | ⏳ |
| 68 | API ระบบแชทเบื้องต้น (REST + WebSocket) | ส่งข้อความ real-time ระหว่างผู้ใช้ | Spring WebFlux/WebSocket | ⏳ |
| 69 | API ระบบสมัครสมาชิกและยืนยันอีเมล | Register, verify email, reset password | Spring Security, Mail | ⏳ |
| 70 | API ระบบแจ้งเตือนแบบ Push Notification | ส่ง notification ผ่าน queue เมื่อมี event | Spring Boot, RabbitMQ/Kafka | ⏳ |

## หมวดที่ 8: Full-Stack Web Applications

ต่อยอดจาก Part 71-90 รวมกับ frontend (HTML/JS หรือ Thymeleaf)

| # | โปรเจค | คำอธิบาย | เทคโนโลยีหลัก | สถานะ |
|---|---|---|---|---|
| 71 | เว็บระบบจัดการค่าใช้จ่ายส่วนตัว (Expense Tracker) | บันทึกรายรับ-จ่าย พร้อมกราฟสรุป | Spring Boot + Thymeleaf/React | ⏳ |
| 72 | เว็บระบบจองคิวร้านตัดผม/คลินิก | ลูกค้าจองเวลา, เจ้าของร้านจัดการตาราง | Spring Boot, Full CRUD | ⏳ |
| 73 | เว็บ Forum/กระดานสนทนา | โพสต์, ตอบกลับ, upvote/downvote | Spring Boot, Nested Comments | ⏳ |
| 74 | เว็บระบบสั่งอาหารออนไลน์ (Food Delivery) | เมนู, ตะกร้า, สถานะคำสั่งซื้อ real-time | Spring Boot, WebSocket | ⏳ |
| 75 | เว็บ Portfolio/CV Builder | สร้างและแก้ไข portfolio ออนไลน์ | Spring Boot, File Upload (S3) | ⏳ |
| 76 | เว็บระบบจัดการอีเวนต์ (Event Management) | สร้างอีเวนต์, ลงทะเบียน, ออกตั๋ว QR Code | Spring Boot, QR Generation | ⏳ |
| 77 | เว็บ E-Commerce ขนาดเล็ก (สินค้า+ตะกร้า+ชำระเงิน) | ระบบขายสินค้าครบวงจร (ทบทวน Capstone Part 103-104) | Full Spring Boot Stack | ⏳ |
| 78 | เว็บระบบติดตามพัสดุ (Package Tracking) | อัปเดตสถานะพัสดุ real-time | Spring Boot, WebSocket | ⏳ |
| 79 | เว็บระบบสำรวจความคิดเห็น (Survey/Poll System) | สร้างแบบสอบถาม, เก็บผล, แสดงกราฟ | Spring Boot, Charting | ⏳ |
| 80 | เว็บระบบจัดการห้องเช่า/คอนโด (Property Management) | จัดการผู้เช่า, ค่าเช่า, การแจ้งซ่อม | Spring Boot, Multi-role Auth | ⏳ |

## หมวดที่ 9: Microservices, Cloud & DevOps Projects

ใช้ความรู้จาก Part 91-99 (Microservices, Docker, Kubernetes, CI/CD)

| # | โปรเจค | คำอธิบาย | เทคโนโลยีหลัก | สถานะ |
|---|---|---|---|---|
| 81 | ระบบ E-Commerce แบบ Microservices เต็มรูปแบบ | แยก Order/Inventory/Payment/User Service | Spring Cloud, Eureka, Gateway | ⏳ |
| 82 | ระบบ Notification Service กลาง (Multi-tenant) | รับ event จากหลายระบบ ส่ง email/SMS/push | Kafka, Spring Boot | ⏳ |
| 83 | Log Aggregation Pipeline (รวม log จากหลาย service) | เก็บ, ค้นหา, วิเคราะห์ log แบบ centralized | ELK Stack, Docker Compose | ⏳ |
| 84 | ระบบ Rate Limiter แบบ Distributed | จำกัด request rate ข้ามหลาย instance | Redis, API Gateway | ⏳ |
| 85 | ระบบ CI/CD Pipeline สำหรับทีมพัฒนา | Build-test-deploy อัตโนมัติทุก push | GitHub Actions, Docker, K8s | ⏳ |
| 86 | ระบบ Monitoring Dashboard สำหรับ Microservices | แสดง metric/health ของทุก service แบบ real-time | Prometheus, Grafana | ⏳ |
| 87 | ระบบ File Storage Service (คล้าย S3 ขนาดเล็ก) | อัปโหลด/ดาวน์โหลดไฟล์ผ่าน API เดียว | Spring Boot, Object Storage | ⏳ |
| 88 | ระบบ Job Scheduler แบบ Distributed | รันงานตามเวลาข้ามหลาย instance โดยไม่ซ้ำกัน | Quartz, Distributed Lock | ⏳ |
| 89 | ระบบ Feature Flag Service | เปิด/ปิดฟีเจอร์แบบ real-time โดยไม่ redeploy | Config Server, Redis | ⏳ |
| 90 | Chat Application แบบ Scalable (Microservices) | รองรับผู้ใช้จำนวนมากพร้อมกันผ่าน WebSocket+Kafka | Kubernetes, Kafka, WebSocket | ⏳ |

## หมวดที่ 10: Games, Simulations & Enterprise Systems

ระดับสูง ผสมผสานความรู้จากทุกหมวดหมู่ + Part 100-105

| # | โปรเจค | คำอธิบาย | เทคโนโลยีหลัก | สถานะ |
|---|---|---|---|---|
| 91 | เกม Snake/Tetris แบบ Real-time | เกมคลาสสิกพร้อม game loop และ collision detection | Game Loop, 2D Graphics | ⏳ |
| 92 | ระบบจำลองการจราจร (Traffic Simulation) | จำลองรถวิ่งบนถนน, สัญญาณไฟจราจร | Multithreading, Simulation | ⏳ |
| 93 | ระบบเทรดหุ้นจำลอง (Stock Trading Simulator) | ซื้อ-ขายหุ้นจำลอง, คำนวณกำไร-ขาดทุน | Real-time Data, Concurrency | ⏳ |
| 94 | ระบบ Matchmaking สำหรับเกมออนไลน์ | จับคู่ผู้เล่นตามระดับความสามารถ | Queue, Algorithm Design | ⏳ |
| 95 | AI Chatbot เบื้องต้น (Rule-based + NLP พื้นฐาน) | ตอบคำถามอัตโนมัติตาม pattern | String Matching, State Machine | ⏳ |
| 96 | ระบบบริหารทรัพยากรบุคคล (HRM) แบบเต็มรูปแบบ | เงินเดือน, ลา, ประเมินผล, องค์กร multi-level | Enterprise Architecture | ⏳ |
| 97 | ระบบ ERP ขนาดเล็กสำหรับ SME | บัญชี, สต็อก, ขาย, จัดซื้อ ในระบบเดียว | Microservices, Full Stack | ⏳ |
| 98 | ระบบ Fraud Detection เบื้องต้น | ตรวจจับธุรกรรมผิดปกติด้วย rule-based analysis | Stream Processing, Kafka | ⏳ |
| 99 | ระบบจองและบริหารเที่ยวบิน (Flight Booking System) | ที่นั่ง, ราคาตามช่วงเวลา, multi-leg booking | Concurrency, Complex Domain | ⏳ |
| 100 | Capstone ส่วนตัว: ระบบตามความสนใจของคุณเอง | เลือกโดเมนที่คุณสนใจ ออกแบบและพัฒนาเองแบบเต็มรูปแบบ | ทุกแนวคิดจากหลักสูตร | ⏳ |

---

## หลักการเลือกโปรเจคเพื่อฝึกฝน

```
ระดับพื้นฐาน (1-20, 31-40):    ฝึก syntax, logic, file handling - ทำเสร็จได้ใน 1-3 วัน
ระดับกลาง (11-30, 41-60):      ฝึก OOP, design pattern, database - ทำเสร็จได้ใน 3-7 วัน
ระดับสูง (61-80):              ฝึก REST API, full-stack, security - ทำเสร็จได้ใน 1-2 สัปดาห์
ระดับมืออาชีพ (81-100):        ฝึก microservices, cloud, ระบบซับซ้อน - ทำเสร็จได้ใน 2-4 สัปดาห์
```

**คำแนะนำการฝึกฝน**:
1. เลือกโปรเจคที่ตรงกับ Part ที่กำลังเรียนอยู่ในหลักสูตร (ทำควบคู่กันจะ
   เข้าใจลึกกว่าเรียนทฤษฎีอย่างเดียว)
2. ทำโปรเจคให้ "เสร็จสมบูรณ์" ก่อนเริ่มโปรเจคถัดไป (มี README, มี test,
   commit ขึ้น GitHub) — portfolio ที่มีโปรเจคเสร็จสมบูรณ์ 10 ตัว ดีกว่า
   โปรเจคที่เริ่มไว้ 50 ตัวแต่ไม่เสร็จสักตัว
3. ทุกโปรเจคควรมี: unit test (ทบทวน Part 58-60), README ที่อธิบาย
   architecture decision, และถ้าเป็นไปได้ควร deploy ให้ใช้งานได้จริง
   (ทบทวน Part 95-98)

## ขั้นตอนถัดไป

โปรเจคทั้งหมดในตารางข้างบนนี้ **เขียนได้จริงด้วยความรู้จากหลักสูตร Java
105 Part** ที่มีอยู่แล้ว — ไฟล์นี้เป็น **แผนที่/รายการไอเดีย** ให้เลือกฝึก
ฝนได้เอง หากต้องการ **คู่มือแบบละเอียดพร้อมโค้ดที่ใช้งานได้จริง** (แบบ
เดียวกับ Capstone Project ใน Part 103-104) สำหรับโปรเจคใดโปรเจคหนึ่งหรือ
หลายโปรเจคพร้อมกัน สามารถแจ้งได้ว่าต้องการเริ่มจากโปรเจคไหนก่อน
