# แผนที่หลักสูตร (Curriculum Roadmap) — ขั้นตอนที่ 1 ถึง 1000+

เอกสารนี้คือ "แผนที่" ทั้งหมดของหลักสูตร แบ่งเป็น 105 Part (และจะขยายต่อได้ในอนาคต)
แต่ละ Part ถูกออกแบบให้ครอบคลุมประมาณ 9-10 "ขั้นตอนย่อย" (steps) ของการเรียนรู้
รวมแล้วทั้งหลักสูตรครอบคลุมขั้นตอนที่ 1 ถึง 1000+ ตามที่ตั้งเป้าไว้

สถานะ: ✅ = เขียนเสร็จแล้ว (มีไฟล์ใน `parts/`) | ⏳ = อยู่ในแผน ยังไม่เขียน

---

## ส่วนที่ 1: พื้นฐานภาษา Java (Part 1-20 | Steps 1-200)

| Part | ขั้นตอน | หัวข้อ | สถานะ |
|---|---|---|---|
| 1 | 1-10 | บทนำสู่ Java, ประวัติ, JDK/JRE/JVM, การติดตั้งเครื่องมือ, โปรแกรมแรก Hello World | ✅ |
| 2 | 11-20 | โครงสร้างโปรแกรม Java, การคอมไพล์และรัน, คอมเมนต์, กฎการตั้งชื่อ | ✅ |
| 3 | 21-30 | ตัวแปร ชนิดข้อมูล (Primitive types) และการแปลงชนิดข้อมูล (Casting) | ✅ |
| 4 | 31-40 | ตัวดำเนินการ (Arithmetic, Relational, Logical, Bitwise, Assignment) | ✅ |
| 5 | 41-50 | คำสั่งควบคุมเงื่อนไข if-else, switch-case, switch expression | ✅ |
| 6 | 51-60 | คำสั่งวนซ้ำ for, while, do-while, break, continue, labeled loop | ✅ |
| 7 | 61-70 | Arrays หนึ่งมิติ หลายมิติ และ Arrays utility class | ✅ |
| 8 | 71-80 | Methods, Parameter passing, Overloading, Varargs, Recursion เบื้องต้น | ✅ |
| 9 | 81-90 | String, StringBuilder, StringBuffer, String Pool, การจัดการข้อความ | ✅ |
| 10 | 91-100 | Exception Handling เบื้องต้น try-catch-finally, throw, throws | ✅ |
| 11 | 101-110 | Classes และ Objects, Fields, Methods, this keyword | ✅ |
| 12 | 111-120 | Constructors, Constructor Overloading, Constructor Chaining | ✅ |
| 13 | 121-130 | Encapsulation, Access Modifiers, Getter/Setter, JavaBeans | ✅ |
| 14 | 131-140 | Inheritance, super keyword, Method Overriding | ✅ |
| 15 | 141-150 | Polymorphism, Dynamic Binding, instanceof, Casting Objects | ✅ |
| 16 | 151-160 | Abstract Classes และ Interfaces (default/static methods) | ✅ |
| 17 | 161-170 | Static members, Final keyword, Immutability | ✅ |
| 18 | 171-180 | Enums แบบละเอียด (methods, fields, abstract methods ใน enum) | ✅ |
| 19 | 181-190 | Nested Classes, Inner Classes, Local Classes, Anonymous Classes | ✅ |
| 20 | 191-200 | Packages, การจัดระเบียบโปรเจกต์, classpath พื้นฐาน | ✅ |

## ส่วนที่ 2: โครงสร้างข้อมูลและอัลกอริทึม (Part 21-35 | Steps 201-350)

| Part | ขั้นตอน | หัวข้อ | สถานะ |
|---|---|---|---|
| 21 | 201-210 | Exception Handling ขั้นสูง: Custom Exception, try-with-resources, multi-catch | ✅ |
| 22 | 211-220 | Collections Framework ภาพรวม, List: ArrayList, LinkedList | ✅ |
| 23 | 221-230 | Set: HashSet, LinkedHashSet, TreeSet | ✅ |
| 24 | 231-240 | Map: HashMap, LinkedHashMap, TreeMap | ✅ |
| 25 | 241-250 | Queue, Deque, PriorityQueue, Stack class | ✅ |
| 26 | 251-260 | Generics: Generic class, method, bounded types, wildcards | ✅ |
| 27 | 261-270 | Iterator, Iterable, Comparable, Comparator | ✅ |
| 28 | 271-280 | Wrapper Classes, Autoboxing/Unboxing, Number formatting | ✅ |
| 29 | 281-290 | Recursion ขั้นสูง: backtracking, memoization เบื้องต้น | ✅ |
| 30 | 291-300 | Sorting Algorithms: Bubble, Selection, Insertion, Merge, Quick Sort | ✅ |
| 31 | 301-310 | Searching Algorithms: Linear, Binary Search, และการสร้าง Stack/Queue เอง | ✅ |
| 32 | 311-320 | Linked List แบบ manual (Singly, Doubly, Circular) | ✅ |
| 33 | 321-330 | Tree: Binary Tree, Binary Search Tree, Tree Traversal | ✅ |
| 34 | 331-340 | Graph เบื้องต้น: representation, BFS, DFS | ✅ |
| 35 | 341-350 | Big O Notation, Time/Space Complexity, การวิเคราะห์อัลกอริทึม | ✅ |

## ส่วนที่ 3: Java ระดับกลางถึงขั้นสูง (Part 36-55 | Steps 351-550)

| Part | ขั้นตอน | หัวข้อ | สถานะ |
|---|---|---|---|
| 36 | 351-360 | File I/O เบื้องต้น: File, FileReader/Writer, BufferedReader/Writer | ✅ |
| 37 | 361-370 | NIO.2: Path, Files, การอ่านเขียนไฟล์สมัยใหม่ | ✅ |
| 38 | 371-380 | Serialization, Deserialization, Serializable interface | ✅ |
| 39 | 381-390 | Lambda Expressions พื้นฐานถึงขั้นสูง | ✅ |
| 40 | 391-400 | Functional Interfaces: Function, Predicate, Consumer, Supplier | ✅ |
| 41 | 401-410 | Stream API เบื้องต้น: filter, map, reduce, collect | ✅ |
| 42 | 411-420 | Stream API ขั้นสูง: Collectors, groupingBy, parallel streams | ✅ |
| 43 | 421-430 | Optional Class และการเขียนโค้ด null-safe | ✅ |
| 44 | 431-440 | Date and Time API (java.time): LocalDate, LocalDateTime, Duration | ✅ |
| 45 | 441-450 | Regular Expressions (Regex) ใน Java | ✅ |
| 46 | 451-460 | Multithreading เบื้องต้น: Thread, Runnable, Thread lifecycle | ✅ |
| 47 | 461-470 | Multithreading ขั้นสูง: synchronized, wait/notify, Lock, Deadlock | ✅ |
| 48 | 471-480 | Executor Framework, Thread Pool, Callable, Future | ✅ |
| 49 | 481-490 | CompletableFuture และ Asynchronous Programming | ✅ |
| 50 | 491-500 | Concurrent Collections: ConcurrentHashMap, CopyOnWriteArrayList, Atomic | ✅ |
| 51 | 501-510 | Records, Sealed Classes, Pattern Matching (Java 17-21) | ⏳ |
| 52 | 511-520 | Annotations: การใช้งานและการสร้าง Custom Annotation | ⏳ |
| 53 | 521-530 | Reflection API | ⏳ |
| 54 | 531-540 | Design Patterns: Creational (Singleton, Factory, Abstract Factory, Builder) | ⏳ |
| 55 | 541-550 | Design Patterns: Structural (Adapter, Decorator, Facade, Proxy, Composite) | ⏳ |

## ส่วนที่ 4: Java ขั้นสูงและ Tooling (Part 56-70 | Steps 551-700)

| Part | ขั้นตอน | หัวข้อ | สถานะ |
|---|---|---|---|
| 56 | 551-560 | Design Patterns: Behavioral (Observer, Strategy, Command, Template Method, State) | ⏳ |
| 57 | 561-570 | SOLID Principles และ Clean Code ในทางปฏิบัติ | ⏳ |
| 58 | 571-580 | Unit Testing ด้วย JUnit 5 (Assertions, Lifecycle, Parameterized Tests) | ⏳ |
| 59 | 581-590 | Mocking ด้วย Mockito | ⏳ |
| 60 | 591-600 | Test-Driven Development (TDD) ในทางปฏิบัติ | ⏳ |
| 61 | 601-610 | Build Tools: Maven เบื้องต้นถึงขั้นสูง (POM, Lifecycle, Plugins) | ⏳ |
| 62 | 611-620 | Build Tools: Gradle เบื้องต้นถึงขั้นสูง | ⏳ |
| 63 | 621-630 | Logging ด้วย SLF4J และ Logback | ⏳ |
| 64 | 631-640 | JDBC และการเชื่อมต่อฐานข้อมูล (Connection, Statement, ResultSet) | ⏳ |
| 65 | 641-650 | SQL กับ Java: CRUD แบบเต็มรูปแบบ, PreparedStatement, Transaction | ⏳ |
| 66 | 651-660 | Connection Pooling (HikariCP), DataSource | ⏳ |
| 67 | 661-670 | JVM Internals: Memory Model, Heap/Stack, Garbage Collection | ⏳ |
| 68 | 671-680 | Performance Tuning และ Profiling เบื้องต้น | ⏳ |
| 69 | 681-690 | JAR Files, Classpath, Java Module System (JPMS) | ⏳ |
| 70 | 691-700 | Networking: Socket Programming, JSON ด้วย Jackson/Gson | ⏳ |

## ส่วนที่ 5: การพัฒนาเว็บแอปพลิเคชัน (Part 71-90 | Steps 701-900)

| Part | ขั้นตอน | หัวข้อ | สถานะ |
|---|---|---|---|
| 71 | 701-710 | HTTP และ Web Fundamentals สำหรับ Java Developer | ⏳ |
| 72 | 711-720 | Servlet และ JSP เบื้องต้น | ⏳ |
| 73 | 721-730 | Introduction to Spring Framework, IoC Container | ⏳ |
| 74 | 731-740 | Spring Core: Dependency Injection, Bean Scopes, Configuration | ⏳ |
| 75 | 741-750 | Spring Boot เบื้องต้น: Auto-configuration, Starter | ⏳ |
| 76 | 751-760 | Spring Boot: Configuration Properties, Profiles, Environment | ⏳ |
| 77 | 761-770 | Spring MVC: Controller, Model, View, Thymeleaf | ⏳ |
| 78 | 771-780 | Building REST API ด้วย Spring Boot (@RestController, ResponseEntity) | ⏳ |
| 79 | 781-790 | Spring Data JPA เบื้องต้น: Repository, Query Methods | ⏳ |
| 80 | 791-800 | Hibernate/JPA ขั้นสูง: Relationships, Lazy/Eager Loading, N+1 | ⏳ |
| 81 | 801-810 | Spring Boot: Validation (Bean Validation) และ Global Exception Handling | ⏳ |
| 82 | 811-820 | Spring Security เบื้องต้น: Authentication, Authorization | ⏳ |
| 83 | 821-830 | Spring Security: JWT Authentication แบบเต็มระบบ | ⏳ |
| 84 | 831-840 | Spring Boot Testing: MockMvc, @SpringBootTest, Testcontainers | ⏳ |
| 85 | 841-850 | RESTful API Design Best Practices, Versioning, Pagination | ⏳ |
| 86 | 851-860 | API Documentation ด้วย Swagger/OpenAPI | ⏳ |
| 87 | 861-870 | Caching ด้วย Redis และ Spring Cache Abstraction | ⏳ |
| 88 | 871-880 | Messaging: RabbitMQ กับ Spring AMQP | ⏳ |
| 89 | 881-890 | Messaging: Apache Kafka กับ Spring Kafka | ⏳ |
| 90 | 891-900 | WebSocket และ Real-time Applications (STOMP) | ⏳ |

## ส่วนที่ 6: ระดับมืออาชีพและระดับโลก (Part 91-105 | Steps 901-1000+)

| Part | ขั้นตอน | หัวข้อ | สถานะ |
|---|---|---|---|
| 91 | 901-910 | Microservices Architecture เบื้องต้น: หลักการ, ข้อดี-ข้อเสีย | ⏳ |
| 92 | 911-920 | Spring Cloud: Service Discovery (Eureka), Load Balancing | ⏳ |
| 93 | 921-930 | Spring Cloud: API Gateway (Spring Cloud Gateway) | ⏳ |
| 94 | 931-940 | Spring Cloud: Config Server, Circuit Breaker (Resilience4j) | ⏳ |
| 95 | 941-950 | Docker สำหรับ Java Developers: Dockerfile, Docker Compose | ⏳ |
| 96 | 951-960 | Kubernetes เบื้องต้นสำหรับ Java Applications | ⏳ |
| 97 | 961-970 | CI/CD Pipeline ด้วย GitHub Actions/Jenkins | ⏳ |
| 98 | 971-980 | Cloud Deployment (AWS/GCP) สำหรับ Java Applications | ⏳ |
| 99 | 981-990 | Monitoring และ Observability (Actuator, Prometheus, Grafana, ELK) | ⏳ |
| 100 | 991-1000 | System Design สำหรับ Java Developers: Scalability, HA, Caching Strategy | ⏳ |
| 101 | 1001-1010 | Security Best Practices ระดับมืออาชีพ (OWASP Top 10 สำหรับ Java) | ⏳ |
| 102 | 1011-1020 | Advanced Concurrency Patterns และ Reactive Programming (Project Reactor) | ⏳ |
| 103 | 1021-1030 | โปรเจกต์จบหลักสูตร: E-Commerce Platform แบบ Full Stack (ตอนที่ 1) | ⏳ |
| 104 | 1031-1040 | โปรเจกต์จบหลักสูตร: E-Commerce Platform แบบ Full Stack (ตอนที่ 2) | ⏳ |
| 105 | 1041-1050+ | การเตรียมตัวสัมภาษณ์งานและเส้นทางอาชีพ Java Developer ระดับโลก | ⏳ |

---

> หมายเหตุ: หลักสูตรนี้ยังสามารถขยายต่อได้อีกหลาย Part ตามความต้องการ เช่น
> Reactive Spring (WebFlux), GraphQL กับ Java, gRPC, Event-Driven Architecture,
> Domain-Driven Design (DDD), Kotlin สำหรับ Java Developer ฯลฯ
> ดูสถานะความคืบหน้าล่าสุดได้ที่ [`PROGRESS.md`](./PROGRESS.md)
