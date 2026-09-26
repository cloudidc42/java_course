# Part 61: Build Tools: Maven เบื้องต้นถึงขั้นสูง

> ขั้นตอนที่ 601-610 ของหลักสูตร | ระดับ: สูง

## สารบัญ

1. Build Tool คืออะไร แก้ปัญหาอะไร
2. การติดตั้ง Maven และโครงสร้างโปรเจกต์
3. `pom.xml`: หัวใจของ Maven
4. Dependency Management
5. Maven Build Lifecycle
6. Dependency Scope
7. Maven Plugins
8. Multi-Module Projects
9. Dependency ขัดแย้งกัน (Dependency Conflict) และการแก้ไข
10. แบบฝึกหัดและสรุป

---

## 1. Build Tool คืออะไร แก้ปัญหาอะไร

ทบทวนจาก Part 20, 58: การ compile และรัน test ด้วยมือ (`javac`, `java -cp`)
ใช้ได้กับโปรเจกต์เล็ก ๆ แต่**ไม่ scale** เมื่อโปรเจกต์มี dependency จำนวนมาก
(เช่น JUnit, Mockito จาก Part 58-59) — **Build Tool** จัดการ:

1. **Dependency Management**: ดาวน์โหลด library ที่ต้องใช้อัตโนมัติ (ไม่ต้อง
   ไปหาไฟล์ `.jar` มาเก็บเอง)
2. **Build Automation**: compile, test, package เป็นขั้นตอนมาตรฐานเดียวกัน
   ทุกโปรเจกต์
3. **Convention over Configuration**: โครงสร้างโปรเจกต์มาตรฐาน (ทบทวนจาก
   Part 20) ทำให้ทีมงานเข้าใจโปรเจกต์ใหม่ได้ทันที

**Maven** (2004) และ **Gradle** (2012, Part 62) เป็น build tool หลักของโลก
Java — Part นี้เน้น Maven เพราะยังเป็นมาตรฐานที่ใช้แพร่หลายที่สุด (โดยเฉพาะ
ในโปรเจกต์ Spring — Part 73 เป็นต้นไป)

## 2. การติดตั้ง Maven และโครงสร้างโปรเจกต์

```bash
# ตรวจสอบว่าติดตั้งแล้ว (Maven มักมาพร้อม IDE เช่น IntelliJ อยู่แล้ว)
mvn -version

# สร้างโปรเจกต์ใหม่จาก archetype (template) มาตรฐาน
mvn archetype:generate -DgroupId=com.example -DartifactId=my-app \
    -DarchetypeArtifactId=maven-archetype-quickstart -DinteractiveMode=false
```

โครงสร้างที่ได้ (ทบทวนจาก Part 20 — Maven Standard Directory Layout):

```
my-app/
├── pom.xml                          <- ไฟล์ config หลักของ Maven
└── src/
    ├── main/
    │   ├── java/                     <- source code
    │   └── resources/                 <- ไฟล์ config, properties
    └── test/
        ├── java/                      <- test code
        └── resources/                  <- test resources
```

## 3. `pom.xml`: หัวใจของ Maven

**POM (Project Object Model)** คือไฟล์ XML ที่นิยาม**ทุกอย่างเกี่ยวกับ
โปรเจกต์**: identity, dependency, build configuration

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                              http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <!-- Coordinates: identity เฉพาะตัวของโปรเจกต์ (เหมือน package name จาก Part 20) -->
    <groupId>com.example</groupId>        <!-- ชื่อองค์กร/กลุ่ม (มักเป็น domain กลับด้าน) -->
    <artifactId>my-app</artifactId>         <!-- ชื่อโปรเจกต์ -->
    <version>1.0.0</version>                 <!-- เวอร์ชัน -->
    <packaging>jar</packaging>                <!-- ผลลัพธ์ที่ build ได้ (jar/war/pom) -->

    <properties>
        <maven.compiler.source>21</maven.compiler.source>
        <maven.compiler.target>21</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <!-- ระบุ library ที่โปรเจกต์ต้องใช้ (หัวข้อ 4) -->
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>5.10.0</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

**Coordinates (`groupId:artifactId:version`)** คือ**ที่อยู่เฉพาะตัว**ของทุก
library บนโลก — Maven ใช้สามค่านี้ค้นหา library ที่ต้องการจาก **Maven
Central Repository** (คลังกลางที่เก็บ library นับล้านตัว)

## 4. Dependency Management

```xml
<dependencies>
    <!-- Spring Boot (Part 75) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <version>3.2.0</version>
    </dependency>

    <!-- Mockito (Part 59) -->
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-core</artifactId>
        <version>5.7.0</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

เมื่อรัน `mvn compile` ครั้งแรก Maven จะ:
1. อ่าน `pom.xml` หา dependency ทั้งหมด
2. ดาวน์โหลดจาก **Maven Central** (หรือ repository อื่นที่กำหนด) ลงเก็บใน
   **local repository** (`~/.m2/repository/`)
3. **Transitive Dependency Resolution**: ดาวน์โหลด dependency ของ
   dependency ด้วย (เช่น `spring-boot-starter-web` ต้องการ Tomcat, Jackson
   ฯลฯ อีกหลายสิบตัว — Maven จัดการให้อัตโนมัติทั้งหมด)

```bash
mvn dependency:tree # แสดง dependency tree ทั้งหมด (รวม transitive dependencies)
```

## 5. Maven Build Lifecycle

Maven มี **lifecycle** ที่ประกอบด้วยหลาย **phase** เรียงลำดับตายตัว — รัน
phase ใดจะรัน**ทุก phase ก่อนหน้าโดยอัตโนมัติ**ด้วย:

```
validate -> compile -> test -> package -> verify -> install -> deploy
```

| Phase | ทำอะไร |
|---|---|
| `validate` | ตรวจสอบว่าโครงสร้างโปรเจกต์ถูกต้อง |
| `compile` | คอมไพล์ source code (`src/main/java`) |
| `test` | รัน unit test (`src/test/java`) ด้วย JUnit (ทบทวนจาก Part 58) |
| `package` | บีบอัดเป็น `.jar`/`.war` (ทบทวนจาก Part 20) |
| `verify` | ตรวจสอบผลลัพธ์ผ่านเกณฑ์คุณภาพ (integration test เป็นต้น) |
| `install` | ติดตั้งลง local repository (ให้โปรเจกต์อื่นในเครื่องใช้ได้) |
| `deploy` | อัปโหลดไปยัง remote repository (ให้ทีมอื่นใช้ได้) |

```bash
mvn compile   # แค่คอมไพล์
mvn test      # คอมไพล์ + รัน test
mvn package   # คอมไพล์ + test + สร้าง jar/war
mvn clean     # ลบไฟล์ที่ build ไว้ก่อนหน้า (โฟลเดอร์ target/)
mvn clean install # ลบของเก่า + build ใหม่ทั้งหมด + install ลง local repo
```

## 6. Dependency Scope

**Scope** กำหนดว่า dependency นั้นใช้ได้ใน**ขั้นตอนไหน**ของ build:

| Scope | ใช้ได้ที่ไหน | ตัวอย่าง |
|---|---|---|
| `compile` (default) | ทุกที่ (compile, test, runtime) | library หลักที่โค้ดใช้จริง |
| `test` | แค่ตอน compile/run test เท่านั้น ไม่รวมใน production jar | JUnit, Mockito (Part 58-59) |
| `provided` | compile-time เท่านั้น คาดว่า runtime environment มีให้แล้ว | Servlet API (Part 72 — Tomcat มีให้แล้ว) |
| `runtime` | ไม่ต้องใช้ตอน compile แต่ต้องมีตอนรัน | JDBC Driver (Part 64) |

```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.10.0</version>
    <scope>test</scope> <!-- ไม่รวมอยู่ใน production jar ที่ deploy จริง -->
</dependency>
```

## 7. Maven Plugins

**Plugin** คือ**หน่วยการทำงานที่ปลั๊กอินเข้ากับ lifecycle** — Maven core เอง
ทำอะไรได้น้อยมาก งานส่วนใหญ่ (รวมถึง compile) จริง ๆ ทำผ่าน plugin ทั้งหมด

```xml
<build>
    <plugins>
        <!-- กำหนด Java version สำหรับ compile (ทางเลือกอื่นจาก properties ในหัวข้อ 3) -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.11.0</version>
            <configuration>
                <release>21</release>
            </configuration>
        </plugin>

        <!-- สร้าง executable JAR ที่รันได้ด้วย java -jar (ทบทวน Manifest จาก Part 20) -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-jar-plugin</artifactId>
            <configuration>
                <archive>
                    <manifest>
                        <mainClass>com.example.Main</mainClass>
                    </manifest>
                </archive>
            </configuration>
        </plugin>

        <!-- Spring Boot plugin: สร้าง "fat jar" ที่รวม dependency ทั้งหมดไว้ในไฟล์เดียว (Part 75) -->
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

## 8. Multi-Module Projects

โปรเจกต์ขนาดใหญ่ (โดยเฉพาะ microservices — Part 91) มักแบ่งเป็นหลาย
**module** ที่จัดการร่วมกันผ่าน **parent POM**:

```
ecommerce-platform/         <- parent project
├── pom.xml                  <- parent pom (aggregator)
├── common-lib/               <- module: code ที่ใช้ร่วมกัน
│   └── pom.xml
├── order-service/             <- module: จัดการคำสั่งซื้อ
│   └── pom.xml
└── payment-service/            <- module: จัดการการเงิน
    └── pom.xml
```

```xml
<!-- ecommerce-platform/pom.xml (parent) -->
<project>
    <groupId>com.example</groupId>
    <artifactId>ecommerce-platform</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging> <!-- packaging=pom สำหรับ parent (ไม่มี code ของตัวเอง) -->

    <modules>
        <module>common-lib</module>
        <module>order-service</module>
        <module>payment-service</module>
    </modules>

    <!-- dependencyManagement: กำหนด version กลาง ให้ทุก module ใช้เวอร์ชันเดียวกัน -->
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.junit.jupiter</groupId>
                <artifactId>junit-jupiter</artifactId>
                <version>5.10.0</version>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

```xml
<!-- order-service/pom.xml -->
<project>
    <parent>
        <groupId>com.example</groupId>
        <artifactId>ecommerce-platform</artifactId>
        <version>1.0.0</version>
    </parent>

    <artifactId>order-service</artifactId>

    <dependencies>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <!-- ไม่ต้องระบุ version! ดึงมาจาก parent's dependencyManagement อัตโนมัติ -->
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

**ประโยชน์**: `mvn install` จาก parent จะ build ทุก module ตามลำดับ
dependency ที่ถูกต้องโดยอัตโนมัติ (ทบทวน Topological Sort จาก Part 34!) และ
`dependencyManagement` ป้องกันปัญหา version ไม่ตรงกันระหว่าง module

## 9. Dependency ขัดแย้งกัน (Dependency Conflict) และการแก้ไข

**Dependency Conflict** เกิดเมื่อ dependency สองตัวต้องการ**คนละ version**
ของ library เดียวกัน (transitive dependency ที่ทับซ้อนกัน)

```
โปรเจกต์ของเรา
├── library-A --requires--> commons-lang v2.6
└── library-B --requires--> commons-lang v3.0  <- ขัดแย้งกัน! ต้องเลือกตัวเดียว
```

Maven ใช้กฎ **"Nearest Wins"**: dependency ที่**ใกล้ที่สุด**ใน dependency
tree (ประกาศตรงใน `pom.xml` ของเราเอง) จะถูกเลือกก่อนเสมอ

```xml
<!-- แก้ปัญหาด้วยการระบุ version ที่ต้องการอย่างชัดเจน (จะ "ชนะ" เพราะใกล้ที่สุด) -->
<dependency>
    <groupId>commons-lang</groupId>
    <artifactId>commons-lang</artifactId>
    <version>3.0</version>
</dependency>

<!-- หรือ exclude version ที่ไม่ต้องการออกจาก dependency ตัวใดตัวหนึ่ง -->
<dependency>
    <groupId>com.example</groupId>
    <artifactId>library-A</artifactId>
    <version>1.0</version>
    <exclusions>
        <exclusion>
            <groupId>commons-lang</groupId>
            <artifactId>commons-lang</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

```bash
mvn dependency:tree -Dverbose # แสดง dependency ที่ถูก conflict และเวอร์ชันที่ถูกเลือกจริง
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียน `pom.xml` สำหรับโปรเจกต์ที่ใช้ JUnit 5 และ Mockito (scope test)
พร้อมกำหนด Java version เป็น 21

**เฉลย:**

```xml
<project>
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>test-project</artifactId>
    <version>1.0.0</version>

    <properties>
        <maven.compiler.source>21</maven.compiler.source>
        <maven.compiler.target>21</maven.compiler.target>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>5.10.0</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.mockito</groupId>
            <artifactId>mockito-core</artifactId>
            <version>5.7.0</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

**2)** อธิบายว่าทำไม `mvn package` จะรัน test ด้วยเสมอ (แม้ไม่ได้เรียก `mvn
test` ตรง ๆ)

**เฉลย**: เพราะ Maven Lifecycle เรียงลำดับ phase ตายตัว (`validate ->
compile -> test -> package -> ...`) — การรัน phase ใด ๆ จะรัน**ทุก phase
ก่อนหน้าโดยอัตโนมัติ**เสมอ เมื่อเรียก `mvn package` Maven ต้องผ่าน phase
`test` ก่อนถึงจะไปถึง `package` ได้ ดังนั้น test ทั้งหมดจะถูกรันโดยอัตโนมัติ
เสมอ (ถ้า test ตัวใด fail, build จะหยุดทันทีไม่ไปถึง package — นี่คือกลไก
ป้องกันไม่ให้ deploy โค้ดที่มี test fail)

**3)** อธิบายกฎ "Nearest Wins" ของ Maven ในการแก้ dependency conflict

**เฉลย**: เมื่อ dependency tree มี library เดียวกันหลาย version (จาก
transitive dependency ที่ต่างกัน) Maven จะเลือก version ของ dependency
**ที่อยู่ใกล้ตัวโปรเจกต์หลักที่สุด**ใน dependency tree (นับจำนวน "ระดับ" จาก
root) — ถ้า dependency ที่เราประกาศตรงใน `pom.xml` ของโปรเจกต์เอง (ระดับ 1)
กับ dependency ที่มาจาก transitive dependency ของ library อื่น (ระดับ 2
หรือลึกกว่า) ขัดแย้งกัน ตัวที่ประกาศตรงจะ "ชนะ" เสมอเพราะอยู่ใกล้กว่า — นี่คือ
เหตุผลที่การประกาศ version ที่ต้องการอย่างชัดเจนใน `pom.xml` ของเราเองสามารถ
แก้ conflict ได้อย่างมีประสิทธิภาพ

### สรุปเนื้อหา Part 61

- Build tool จัดการ dependency, automation การ build, และ convention ของ
  โครงสร้างโปรเจกต์
- `pom.xml` นิยาม coordinates (groupId:artifactId:version), dependencies,
  และ build configuration
- Maven Lifecycle เรียงลำดับ phase ตายตัว: validate -> compile -> test ->
  package -> verify -> install -> deploy
- Dependency scope (`compile`, `test`, `provided`, `runtime`) กำหนดขั้นตอน
  ที่ dependency นั้นใช้ได้
- Multi-module project ใช้ parent POM + `dependencyManagement` จัดการ version
  ให้สอดคล้องกันทุก module
- Dependency conflict แก้ได้ด้วยการประกาศ version ชัดเจน หรือ `exclusions`

**ต่อไป**: [Part 62 — Build Tools: Gradle เบื้องต้นถึงขั้นสูง](./part-062-gradle.md)
