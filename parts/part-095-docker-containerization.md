# Part 95: Docker: Containerization

> ขั้นตอนที่ 941-950 ของหลักสูตร | ระดับ: ระดับมืออาชีพ/โลก

## สารบัญ

1. ปัญหา "It works on my machine": ทำไมต้องมี Containerization
2. Container vs Virtual Machine
3. Docker Concepts: Image, Container, Layer
4. เขียน Dockerfile สำหรับแอป Spring Boot
5. Multi-stage Build: ลดขนาด Image
6. Build และ Run Container
7. Docker Compose: รันหลาย Container พร้อมกัน
8. Environment Variables และ Secrets ใน Container
9. Docker Networking และ Volume
10. แบบฝึกหัดและสรุป

---

## 1. ปัญหา "It works on my machine": ทำไมต้องมี Containerization

ทบทวนปัญหาที่พบตลอดหลักสูตร: แอป Spring Boot ที่รันได้ดีบนเครื่อง
developer อาจ**รันไม่ได้**บนเครื่อง production เพราะ **เวอร์ชัน Java
ต่างกัน**, **environment variable ขาด**, หรือ **library ระดับ OS ขาด**

```java
public class ItWorksOnMyMachineProblem {
    /*
     * "แต่มันรันได้บนเครื่องผม!" - ปัญหาคลาสสิกที่เกิดจาก:
     *   - Developer ใช้ JDK 21, production server มี JDK 17 (ทบทวนความสำคัญของเวอร์ชันจาก Part 1)
     *   - Developer ตั้งค่า environment variable ไว้ในเครื่องตัวเอง แต่ลืมบอกทีม ops
     *   - Library บางตัวต้องพึ่ง native library ของ OS ที่ไม่มีบน production server
     *
     * Docker แก้ปัญหานี้โดย "แพ็ครวม" ทุกอย่างที่แอปต้องการ (code, runtime, library, config)
     * ไว้ใน "container image" เดียว ที่รันเหมือนกันทุกที่ไม่ว่าจะเป็นเครื่อง dev หรือ production
     */
}
```

## 2. Container vs Virtual Machine

```
Virtual Machine:                      Container:
┌─────────────────────┐               ┌─────────────────────┐
│   App A     App B    │               │   App A     App B    │
│  Guest OS  Guest OS   │               │  (bins/libs)(bins/libs)│
├──────────────────────┤               ├──────────────────────┤
│     Hypervisor        │               │   Container Runtime   │
├──────────────────────┤               │   (Docker Engine)      │
│      Host OS          │               ├──────────────────────┤
├──────────────────────┤               │      Host OS (แชร์ kernel)│
│      Hardware          │               ├──────────────────────┤
└─────────────────────┘               │      Hardware          │
                                       └─────────────────────┘
```

**ความแตกต่างสำคัญ**: VM แต่ละตัวมี **Guest OS สมบูรณ์ของตัวเอง**
(ใช้ resource มาก, boot ช้า) — Container **แชร์ Kernel ของ Host OS
ร่วมกัน** (มีแค่ binary/library ที่จำเป็นของแต่ละแอป) ทำให้ **เบากว่ามาก,
start เร็วกว่ามาก** (วินาทีเดียว เทียบกับ VM ที่ใช้เวลานาทีในการ boot)

## 3. Docker Concepts: Image, Container, Layer

```java
public class DockerConceptsDemo {
    /*
     * Image: "แบบพิมพ์เขียว" (template) ที่ไม่เปลี่ยนแปลง (immutable) มีทุกอย่างที่แอปต้องการ
     *        เก็บใน "layer" หลายชั้นซ้อนกัน (แต่ละ instruction ใน Dockerfile สร้าง 1 layer)
     *
     * Container: "instance ที่รันจริง" จาก Image หนึ่ง (คล้ายความสัมพันธ์ระหว่าง Class และ Object - ทบทวน Part 11!)
     *            สร้าง Container หลายตัวจาก Image เดียวกันได้ (แต่ละตัวมี process/state แยกกัน)
     *
     * Layer: Docker cache แต่ละ layer ไว้ - ถ้า layer ไม่เปลี่ยน ไม่ต้อง build ใหม่ (เร็วขึ้นมาก)
     */
}
```

## 4. เขียน Dockerfile สำหรับแอป Spring Boot

```dockerfile
# Dockerfile (แบบพื้นฐาน - ยังไม่ optimize)
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY target/order-service-1.0.0.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

```java
public class DockerfileInstructionsExplanation {
    /*
     * FROM: ระบุ base image (ที่นี่คือ Java 21 JRE บน Alpine Linux - เวอร์ชันเล็กที่สุดของ Linux)
     * WORKDIR: กำหนด working directory ภายใน container
     * COPY: copy ไฟล์จากเครื่อง build เข้าไปใน image (JAR file ที่ build ด้วย Maven/Gradle - ทบทวน Part 61, 62)
     * EXPOSE: บอกว่า container นี้ฟัง port ไหน (เป็นแค่ documentation ไม่ได้ open port จริง)
     * ENTRYPOINT: คำสั่งที่รันเมื่อ container เริ่มทำงาน
     */
}
```

## 5. Multi-stage Build: ลดขนาด Image

```dockerfile
# Dockerfile (แบบ optimize ด้วย multi-stage build)

# Stage 1: Build - ใช้ image ที่มี Maven/JDK ครบ (ขนาดใหญ่ แต่ใช้แค่ตอน build)
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /build
COPY pom.xml .
RUN mvn dependency:go-offline   # ดาวน์โหลด dependency ก่อน - cache layer นี้ไว้ (ทบทวนหัวข้อ 3)
COPY src ./src
RUN mvn clean package -DskipTests

# Stage 2: Runtime - ใช้ image เล็ก ๆ ที่มีแค่ JRE (ไม่มี Maven/JDK build tools)
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=build /build/target/order-service-1.0.0.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

```java
public class MultiStageBuildBenefit {
    /*
     * Stage 1 (build): image ขนาด ~500MB+ (มี Maven, JDK เต็ม, source code, dependency cache)
     * Stage 2 (runtime): image ขนาด ~200MB (มีแค่ JRE + JAR file สุดท้าย)
     *
     * "COPY --from=build" ดึงแค่ไฟล์ JAR ที่ build เสร็จแล้วจาก stage 1 มาใส่ใน stage 2
     * ทำให้ image สุดท้ายที่ deploy จริงไม่มี source code, Maven cache, build tools ติดไปด้วย
     * (เล็กกว่า, ปลอดภัยกว่า - ลด attack surface, deploy เร็วกว่าเพราะโหลด image เร็วกว่า)
     */
}
```

## 6. Build และ Run Container

```java
public class DockerCommandsDemo {
    /*
     * Build image จาก Dockerfile:
     *   docker build -t order-service:1.0.0 .
     *
     * Run container จาก image:
     *   docker run -p 8080:8080 --name order-service-container order-service:1.0.0
     *   (-p 8080:8080 = map port 8080 ของ host ไปที่ port 8080 ของ container)
     *
     * ดู container ที่กำลังรัน:
     *   docker ps
     *
     * ดู log ของ container:
     *   docker logs order-service-container
     *
     * หยุด container:
     *   docker stop order-service-container
     */
}
```

## 7. Docker Compose: รันหลาย Container พร้อมกัน

ทบทวนจาก Part 91: ระบบ microservices มีหลาย service + database + Redis +
RabbitMQ ที่ต้อง**รันพร้อมกัน** — **Docker Compose** จัดการทั้งหมดนี้ด้วย
ไฟล์เดียว

```yaml
# docker-compose.yml
version: '3.8'
services:
  order-service:
    build: ./order-service
    ports:
      - "8081:8080"
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://order-db:5432/orders
      - SPRING_REDIS_HOST=redis
    depends_on:
      - order-db
      - redis

  order-db:
    image: postgres:16-alpine
    environment:
      - POSTGRES_DB=orders
      - POSTGRES_PASSWORD=secret
    volumes:
      - order-db-data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  rabbitmq:
    image: rabbitmq:3-management-alpine
    ports:
      - "5672:5672"
      - "15672:15672"

volumes:
  order-db-data:
```

```java
public class DockerComposeDemo {
    /*
     * รันทั้งระบบด้วยคำสั่งเดียว: docker compose up
     *
     * Docker Compose จะ:
     *   1. Build image ของ order-service จาก Dockerfile ใน ./order-service
     *   2. ดึง image postgres, redis, rabbitmq จาก Docker Hub
     *   3. สร้าง network เดียวกันให้ทุก container คุยกันได้ (ทบทวนหัวข้อ 9)
     *   4. รันตามลำดับ depends_on (order-db, redis ก่อน order-service)
     *
     * แทนที่จะต้อง docker run แยกทีละตัว พร้อมจำ config การเชื่อมต่อทั้งหมดเอง
     */
}
```

## 8. Environment Variables และ Secrets ใน Container

ทบทวนจาก Part 76: `application.properties` ใน Spring Boot อ่านค่าจาก
environment variable ได้อัตโนมัติ

```dockerfile
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY target/order-service-1.0.0.jar app.jar
ENV SPRING_PROFILES_ACTIVE=production
ENTRYPOINT ["java", "-jar", "app.jar"]
```

```java
public class ContainerSecretsWarning {
    /*
     * อันตราย: ห้าม hardcode password/API key ลงใน Dockerfile หรือ docker-compose.yml ที่ commit เข้า git!
     *   environment:
     *     - DB_PASSWORD=mysecretpassword   # ผิด! รั่วไหลผ่าน git history ทันที
     *
     * วิธีที่ปลอดภัยกว่า: ใช้ไฟล์ .env (ที่ .gitignore ไว้ - ทบทวนความสำคัญของ .gitignore)
     *   docker-compose.yml:  environment: - DB_PASSWORD=${DB_PASSWORD}
     *   .env (ไม่ commit): DB_PASSWORD=realSecretValue
     *
     * ใน production จริง ควรใช้ Secret Manager ของ cloud provider (AWS Secrets Manager, HashiCorp Vault)
     * แทนการเก็บ secret เป็น plain text แม้ใน .env ก็ตาม (ปูทางสู่ Part 98)
     */
}
```

## 9. Docker Networking และ Volume

```java
public class DockerNetworkingDemo {
    /*
     * ทบทวนจาก docker-compose.yml หัวข้อ 7: "SPRING_DATASOURCE_URL=jdbc:postgresql://order-db:5432/orders"
     * ใช้ "order-db" เป็น hostname ตรง ๆ ได้เลย! Docker Compose สร้าง internal DNS
     * ที่ resolve ชื่อ service (ตามที่กำหนดใน docker-compose.yml) เป็น IP ของ container นั้นให้อัตโนมัติ
     * (คล้ายกับแนวคิด Service Discovery ที่เรียนใน Part 92 แต่ทำงานในระดับ Docker network)
     *
     * Volume: container โดย default เป็น "ephemeral" (ข้อมูลหายเมื่อ container ถูกลบ)
     * "volumes: order-db-data:/var/lib/postgresql/data" ทำให้ข้อมูล PostgreSQL ถูกเก็บไว้ที่ host
     * แม้ container ของ database ถูกลบและสร้างใหม่ ข้อมูลก็ยังอยู่ครบ (persistent storage)
     */
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียน Dockerfile แบบ multi-stage build สำหรับแอป Spring Boot ที่
ใช้ Gradle (แทน Maven)

**เฉลย:**
```dockerfile
FROM gradle:8-jdk21 AS build
WORKDIR /build
COPY build.gradle settings.gradle ./
COPY src ./src
RUN gradle clean bootJar -x test

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=build /build/build/libs/order-service-1.0.0.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**2)** อธิบายว่าทำไม Container เบากว่าและ start เร็วกว่า Virtual Machine
มาก

**เฉลย**: **Virtual Machine** ต้องรัน **Guest Operating System เต็ม
รูปแบบ**ของตัวเอง (kernel, driver, system service ทั้งหมด) แยกจาก Host OS
โดยสมบูรณ์ผ่าน hypervisor — การ boot VM คือการ**boot ระบบปฏิบัติการใหม่
ทั้งเครื่องจำลอง**ซึ่งใช้เวลาระดับนาทีและใช้ CPU/memory จำนวนมาก
(ทบทวนหัวข้อ 2) ในขณะที่ **Container แชร์ Kernel ของ Host OS ร่วมกัน**
— container มีแค่**process ของแอปพลิเคชันเอง**พร้อม binary/library ที่
จำเป็นเท่านั้น ไม่ต้อง boot ระบบปฏิบัติการใหม่เลย การ "start container"
จึงเป็นแค่**การเริ่ม process ใหม่**ในระบบที่ Kernel ทำงานอยู่แล้ว
(คล้ายกับการเริ่ม process ธรรมดา) ทำให้เร็วกว่ามาก (วินาทีเดียว) และใช้
resource น้อยกว่า VM มาก

**3)** อธิบายว่าทำไมการเก็บ database password ไว้ตรง ๆ ใน
`docker-compose.yml` ที่ commit เข้า Git repository เป็นความเสี่ยงด้าน
ความปลอดภัยที่ร้ายแรง

**เฉลย**: `docker-compose.yml` ที่ commit เข้า Git repository จะถูก
**เก็บไว้ใน commit history ตลอดไป** แม้จะลบค่านั้นออกในคอมมิตถัดไปก็ตาม
(ทบทวนคุณสมบัติของ version control ที่เก็บ history ทุกอย่าง) — ใครก็ตาม
ที่มีสิทธิ์เข้าถึง repository (รวมถึงถ้า repository เผลอถูกตั้งเป็น
public หรือรั่วไหลออกไป) จะสามารถ**ย้อนดู commit history และเห็น
password ที่เคย commit ไว้ได้ทันที** แม้จะไม่ได้เห็นในไฟล์เวอร์ชัน
ปัจจุบันแล้วก็ตาม (ทบทวนความสำคัญของการไม่ commit secret ที่เกี่ยวข้อง
กับหลักการ .gitignore) นี่คือเหตุผลที่ต้องใช้**environment variable
substitution** (`${DB_PASSWORD}`) ร่วมกับไฟล์ `.env` ที่**ไม่ถูก commit**
(อยู่ใน `.gitignore`) หรือใช้ **Secret Manager ของ cloud provider** ใน
ระบบ production จริง เพื่อไม่ให้ข้อมูลลับปรากฏใน source code หรือ
version control เลย

### สรุปเนื้อหา Part 95

- Docker แก้ปัญหา "it works on my machine" โดยแพ็ครวมทุกอย่างที่แอป
  ต้องการไว้ใน container image เดียว
- Container เบากว่า VM มากเพราะแชร์ Kernel ของ Host OS ไม่ต้อง boot OS
  ใหม่ทุกครั้ง
- Dockerfile กำหนดขั้นตอนสร้าง image; Multi-stage build ลดขนาด image
  สุดท้ายอย่างมากด้วยการแยก build stage จาก runtime stage
- Docker Compose จัดการหลาย container (service + database + cache +
  message queue) พร้อมกันด้วยไฟล์เดียว
- Docker Network สร้าง internal DNS ให้ container คุยกันผ่านชื่อ service
  ได้ (คล้าย Service Discovery)
- Volume ทำให้ข้อมูลอยู่ถาวรแม้ container ถูกลบและสร้างใหม่
- ห้าม hardcode secret ใน Dockerfile/docker-compose.yml ที่ commit เข้า
  Git — ใช้ .env ที่ไม่ commit หรือ Secret Manager แทน

**ต่อไป**: [Part 96 — Kubernetes](./part-096-kubernetes.md)
