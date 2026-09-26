# Part 62: Build Tools: Gradle เบื้องต้นถึงขั้นสูง

> ขั้นตอนที่ 611-620 ของหลักสูตร | ระดับ: สูง

## สารบัญ

1. Gradle คืออะไร ต่างจาก Maven อย่างไร
2. โครงสร้างโปรเจกต์ Gradle
3. `build.gradle`: Groovy DSL vs Kotlin DSL
4. Dependency Management ใน Gradle
5. Gradle Tasks
6. การเขียน Custom Task
7. Gradle Build Lifecycle
8. Multi-Project Builds
9. Gradle Wrapper: ทำไมสำคัญมาก
10. แบบฝึกหัดและสรุป

---

## 1. Gradle คืออะไร ต่างจาก Maven อย่างไร

**Gradle** (2012) เป็น build tool รุ่นใหม่กว่า Maven (Part 61) ที่ออกแบบมา
แก้ข้อจำกัดบางอย่างของ Maven — ใช้ **Groovy/Kotlin DSL** (Domain-Specific
Language) แทน XML ทำให้**เขียนโค้ดได้จริง** (ไม่ใช่แค่ declarative config)
และมี**performance ดีกว่า**ด้วย incremental build และ build cache

| ลักษณะ | Maven | Gradle |
|---|---|---|
| Configuration format | XML (`pom.xml`) | Groovy/Kotlin DSL (`build.gradle`) |
| ความยืดหยุ่น | จำกัด (ต้องใช้ plugin สำหรับ logic ซับซ้อน) | สูงมาก (เขียนโค้ดได้จริงใน build script) |
| Performance | ธรรมดา | เร็วกว่า (incremental build, build cache, parallel execution) |
| ความนิยม | ยังคงนิยมมาก โดยเฉพาะ enterprise/Spring | นิยมมากใน Android และโปรเจกต์ที่ต้องการความยืดหยุ่นสูง |

**Android Studio ใช้ Gradle เป็นค่าเริ่มต้น** ทำให้ Gradle แพร่หลายมากในวงการ
mobile development ด้วย

## 2. โครงสร้างโปรเจกต์ Gradle

```
my-gradle-app/
├── build.gradle          <- (หรือ build.gradle.kts สำหรับ Kotlin DSL)
├── settings.gradle        <- ระบุชื่อโปรเจกต์และ module (สำหรับ multi-project)
├── gradlew                 <- Gradle Wrapper script (Linux/macOS - หัวข้อ 9)
├── gradlew.bat              <- Gradle Wrapper script (Windows)
├── gradle/wrapper/           <- ไฟล์ config ของ wrapper
└── src/
    ├── main/java/            <- source code (โครงสร้างเดียวกับ Maven - ทบทวน Part 20)
    └── test/java/             <- test code
```

**ข้อสังเกต**: โครงสร้าง `src/main/java`, `src/test/java` **เหมือนกับ
Maven ทุกประการ** — ทั้งสอง build tool ใช้ convention เดียวกัน (Maven
Standard Directory Layout) ทำให้ย้ายจากเครื่องมือหนึ่งไปอีกตัวไม่ยากเกินไป

## 3. `build.gradle`: Groovy DSL vs Kotlin DSL

### Groovy DSL (ดั้งเดิม, ยังนิยมมาก)

```groovy
// build.gradle
plugins {
    id 'java'
}

group = 'com.example'
version = '1.0.0'

java {
    sourceCompatibility = JavaVersion.VERSION_21
    targetCompatibility = JavaVersion.VERSION_21
}

repositories {
    mavenCentral() // ดึง dependency จาก Maven Central (เหมือน Maven - ทบทวน Part 61)
}

dependencies {
    testImplementation 'org.junit.jupiter:junit-jupiter:5.10.0'
    testImplementation 'org.mockito:mockito-core:5.7.0'
}

test {
    useJUnitPlatform() // บอก Gradle ให้ใช้ JUnit 5 (Jupiter) ในการรัน test
}
```

### Kotlin DSL (สมัยใหม่กว่า, type-safe)

```kotlin
// build.gradle.kts
plugins {
    java
}

group = "com.example"
version = "1.0.0"

java {
    sourceCompatibility = JavaVersion.VERSION_21
}

repositories {
    mavenCentral()
}

dependencies {
    testImplementation("org.junit.jupiter:junit-jupiter:5.10.0")
    testImplementation("org.mockito:mockito-core:5.7.0")
}

tasks.test {
    useJUnitPlatform()
}
```

**ข้อดีของ Kotlin DSL**: **type-safe** (IDE ช่วย autocomplete และตรวจสอบ
error ได้ดีกว่า Groovy ที่เป็น dynamic language) — โปรเจกต์ใหม่นิยม Kotlin
DSL มากขึ้นเรื่อย ๆ

## 4. Dependency Management ใน Gradle

```groovy
dependencies {
    implementation 'org.apache.commons:commons-lang3:3.14.0' // compile + runtime (เทียบเท่า Maven scope=compile)
    testImplementation 'org.junit.jupiter:junit-jupiter:5.10.0' // เทียบเท่า Maven scope=test
    compileOnly 'org.projectlombok:lombok:1.18.30'                // เทียบเท่า Maven scope=provided
    runtimeOnly 'com.mysql:mysql-connector-j:8.2.0'                 // เทียบเท่า Maven scope=runtime
    api 'com.google.guava:guava:32.1.3-jre'                          // เฉพาะ library module: expose ให้ผู้ใช้เห็นด้วย
}
```

| Gradle Configuration | เทียบเท่า Maven Scope |
|---|---|
| `implementation` | `compile` (แต่ไม่ expose ให้ dependent module เห็น — เร็วกว่า `api`) |
| `api` | `compile` (expose ให้ dependent module เห็นด้วย transitive) |
| `testImplementation` | `test` |
| `compileOnly` | `provided` |
| `runtimeOnly` | `runtime` |

**ความแตกต่างสำคัญ `implementation` vs `api`**: ถ้า module A ใช้ B ผ่าน
`implementation`, module C ที่ depend on A **จะไม่เห็น B** โดยตรง (ซ่อนไว้
เป็น implementation detail) — ถ้าใช้ `api`, C จะเห็น B ด้วย (transitive
exposure) — `implementation` ช่วยให้ build เร็วขึ้นเพราะ Gradle รู้ว่าไม่
ต้อง rebuild module ที่ depend on A ทุกครั้งที่ B เปลี่ยน (ถ้า B เป็นแค่
internal detail ของ A)

```bash
./gradlew dependencies # แสดง dependency tree (เทียบเท่า mvn dependency:tree)
```

## 5. Gradle Tasks

**Task** คือหน่วยงานพื้นฐานของ Gradle (เทียบเท่า "phase" ของ Maven แต่
ยืดหยุ่นกว่ามาก — สร้าง task ใหม่ได้เองอย่างอิสระ)

```bash
./gradlew tasks          # แสดง task ทั้งหมดที่มี
./gradlew build           # compile + test + package (คล้าย mvn package)
./gradlew test              # รันแค่ test
./gradlew clean              # ลบไฟล์ build เดิม (โฟลเดอร์ build/)
./gradlew run                  # รันแอปพลิเคชัน (ถ้าใช้ application plugin)
```

**Task Dependency**: task หนึ่งสามารถ**ขึ้นอยู่กับ**task อื่น (คล้าย
Topological Sort จาก Part 34) — Gradle จะรัน task ที่จำเป็นก่อนโดยอัตโนมัติ
เช่น `build` ขึ้นกับ `test` ซึ่งขึ้นกับ `compileJava` เป็นลำดับ

## 6. การเขียน Custom Task

จุดแข็งที่สุดของ Gradle คือ**เขียน task เองได้ด้วยโค้ดจริง** (ต่างจาก Maven
ที่ต้องพึ่งพา plugin สำเร็จรูปเท่านั้น):

```groovy
// เพิ่มใน build.gradle
tasks.register('printProjectInfo') {
    doLast { // action ที่จะรันเมื่อ task นี้ถูก execute
        println "โปรเจกต์: ${project.name}"
        println "เวอร์ชัน: ${project.version}"
        println "Java version: ${JavaVersion.current()}"
    }
}

tasks.register('generateReport') {
    dependsOn 'test' // task นี้ต้องรัน test ก่อนเสมอ
    doLast {
        println "สร้างรายงานผลการทดสอบ..."
        // สามารถเขียนโค้ด Groovy/Java จริง ๆ ได้เต็มรูปแบบตรงนี้
        def testResultsDir = file("$buildDir/test-results")
        if (testResultsDir.exists()) {
            println "พบผลการทดสอบที่: ${testResultsDir}"
        }
    }
}
```

```bash
./gradlew printProjectInfo
./gradlew generateReport # จะรัน test ก่อนโดยอัตโนมัติ เพราะ dependsOn 'test'
```

## 7. Gradle Build Lifecycle

Gradle มี 3 phase หลัก (ต่างจาก Maven ที่มีหลาย phase เรียงลำดับตายตัว):

```
1. Initialization: อ่าน settings.gradle กำหนดว่าโปรเจกต์ไหนจะถูก build (สำหรับ multi-project)
2. Configuration: รัน build.gradle ของทุกโปรเจกต์ สร้าง task graph (DAG - ทบทวน Part 34)
3. Execution: รัน task ที่ถูกเรียก (และ dependency ของมัน) ตามลำดับใน task graph
```

**ข้อดีของ task graph แบบ DAG**: Gradle รู้ทั้งหมดว่า task ไหนขึ้นกับ task
ไหนก่อนเริ่ม execution จริง ทำให้**รัน task ที่ไม่ขึ้นกับกันแบบ parallel ได้**
(ทบทวนแนวคิด concurrency จาก Part 46-50) และ**cache ผลลัพธ์ของ task ที่ไม่
เปลี่ยนแปลง**ได้ (incremental build — ถ้า source code ไม่เปลี่ยน ไม่ต้อง
compile ใหม่)

## 8. Multi-Project Builds

```
// settings.gradle
rootProject.name = 'ecommerce-platform'
include 'common-lib', 'order-service', 'payment-service'
```

```groovy
// build.gradle (root - ตั้งค่าที่ใช้ร่วมกันทุก module)
subprojects {
    apply plugin: 'java'

    repositories {
        mavenCentral()
    }

    dependencies {
        testImplementation 'org.junit.jupiter:junit-jupiter:5.10.0'
    }

    test {
        useJUnitPlatform()
    }
}
```

```groovy
// order-service/build.gradle
dependencies {
    implementation project(':common-lib') // depend on module อื่นในโปรเจกต์เดียวกัน
}
```

```bash
./gradlew build # build ทุก module ตามลำดับ dependency ที่ถูกต้องโดยอัตโนมัติ
./gradlew :order-service:test # รัน test เฉพาะ module order-service
```

## 9. Gradle Wrapper: ทำไมสำคัญมาก

**Gradle Wrapper** (`gradlew`/`gradlew.bat`) คือ script ที่ **ดาวน์โหลดและ
ใช้ Gradle เวอร์ชันที่ระบุไว้เฉพาะสำหรับโปรเจกต์นี้เท่านั้น** โดยผู้พัฒนา
**ไม่ต้องติดตั้ง Gradle เองเลย**

```properties
# gradle/wrapper/gradle-wrapper.properties
distributionUrl=https\://services.gradle.org/distributions/gradle-8.5-bin.zip
```

```bash
./gradlew build # ใช้ Gradle 8.5 เสมอ ไม่ว่าเครื่องนี้จะมี Gradle version อื่นติดตั้งอยู่หรือไม่
```

**ทำไมสำคัญมาก**: รับประกันว่า**ทุกคนในทีม (และ CI/CD server — Part 97) ใช้
Gradle version เดียวกันเป๊ะ ๆ** ป้องกันปัญหา "ทำงานบนเครื่องฉันแต่ไม่ทำงาน
บนเครื่องเธอ" ที่เกิดจาก build tool version ไม่ตรงกัน — **ควร commit ไฟล์
`gradlew`, `gradlew.bat`, และ `gradle/wrapper/` ลง git เสมอ** (ไม่ใช่แค่
`build.gradle`)

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียน `build.gradle` (Groovy DSL) สำหรับโปรเจกต์ Java 21 ที่ใช้ JUnit
5 และ Mockito

**เฉลย:**

```groovy
plugins {
    id 'java'
}

group = 'com.example'
version = '1.0.0'

java {
    sourceCompatibility = JavaVersion.VERSION_21
}

repositories {
    mavenCentral()
}

dependencies {
    testImplementation 'org.junit.jupiter:junit-jupiter:5.10.0'
    testImplementation 'org.mockito:mockito-core:5.7.0'
}

test {
    useJUnitPlatform()
}
```

**2)** เขียน custom Gradle task ชื่อ `countJavaFiles` ที่นับจำนวนไฟล์
`.java` ใน `src/main/java`

**เฉลย:**

```groovy
tasks.register('countJavaFiles') {
    doLast {
        def count = fileTree('src/main/java').matching { include '**/*.java' }.files.size()
        println "จำนวนไฟล์ .java: ${count}"
    }
}
```

**3)** อธิบายว่าทำไมควร commit ไฟล์ `gradlew` ลง version control เสมอ

**เฉลย**: `gradlew` (Gradle Wrapper) รับประกันว่าทุกคนที่ clone โปรเจกต์นี้
(รวมถึง CI/CD server) จะใช้ **Gradle version เดียวกันเป๊ะ ๆ** ตามที่ระบุไว้ใน
`gradle-wrapper.properties` โดยไม่ต้องติดตั้ง Gradle เองล่วงหน้า — ถ้าไม่
commit ไฟล์นี้ ผู้พัฒนาแต่ละคนอาจใช้ Gradle version ต่างกัน (ที่ติดตั้งไว้
บนเครื่องตัวเอง) ซึ่งอาจทำให้ behavior การ build ต่างกันเล็กน้อยจนเกิดปัญหา
"ทำงานบนเครื่องฉันแต่ไม่ทำงานบนเครื่องอื่น" ได้ — การมี wrapper ทำให้
"เครื่องมือที่ใช้ build" กลายเป็นส่วนหนึ่งของโปรเจกต์เอง ไม่ใช่สิ่งที่ต้อง
ติดตั้งแยกและหวังว่าจะตรงกันโดยบังเอิญ

### สรุปเนื้อหา Part 62

- Gradle ใช้ Groovy/Kotlin DSL (เขียนโค้ดได้จริง) แทน XML ของ Maven มี
  performance ดีกว่าด้วย incremental build และ caching
- `implementation` ซ่อน dependency จาก module ที่ depend on เรา, `api`
  expose ให้เห็นด้วย
- Task เป็นหน่วยงานพื้นฐาน มี dependency ระหว่างกันได้ (task graph แบบ DAG)
- เขียน custom task ได้ด้วยโค้ดจริง — จุดแข็งที่สุดของ Gradle เทียบกับ Maven
- Multi-project build ใช้ `settings.gradle` ประกาศ module และ
  `subprojects{}` ตั้งค่าร่วมกัน
- Gradle Wrapper (`gradlew`) รับประกัน version เดียวกันทุกเครื่อง ต้อง commit
  ลง git เสมอ

**ต่อไป**: [Part 63 — Logging ด้วย SLF4J และ Logback](./part-063-logging.md)
