# Part 97: CI/CD Pipeline

> ขั้นตอนที่ 961-970 ของหลักสูตร | ระดับ: ระดับมืออาชีพ/โลก

## สารบัญ

1. ปัญหาของการ Deploy แบบ Manual
2. Continuous Integration (CI) คืออะไร
3. Continuous Delivery vs Continuous Deployment
4. GitHub Actions: เขียน CI Pipeline พื้นฐาน
5. Pipeline Stage: Build, Test, Package
6. Build Docker Image และ Push ไปที่ Registry
7. Deploy อัตโนมัติไปที่ Kubernetes
8. Environment Strategy: Dev, Staging, Production
9. Deployment Strategies: Blue-Green และ Canary
10. แบบฝึกหัดและสรุป

---

## 1. ปัญหาของการ Deploy แบบ Manual

ทบทวนตลอดหลักสูตร: เราเขียนโค้ด, รัน test (Part 58-60, 84), build JAR
(Part 61-62), สร้าง Docker image (Part 95), deploy ไปที่ Kubernetes
(Part 96) — ถ้าทำ**ทุกขั้นตอนด้วยมือทุกครั้ง**ที่มีการเปลี่ยนโค้ด จะเกิด
ปัญหา:

```java
public class ManualDeploymentProblems {
    /*
     * 1. ช้า: developer ต้องทำหลายขั้นตอนซ้ำ ๆ ทุกครั้ง (build, test, package, docker build, deploy)
     * 2. เสี่ยงต่อความผิดพลาดจากมนุษย์ (human error): ลืมรัน test ก่อน deploy, deploy ผิด environment
     * 3. ไม่สม่ำเสมอ: แต่ละคนอาจทำขั้นตอนต่างกันเล็กน้อย (inconsistent process)
     * 4. Feedback ช้า: ถ้ามี bug จะรู้ตัวช้าเพราะไม่มีการทดสอบอัตโนมัติทุกครั้งที่ commit
     */
}
```

**CI/CD (Continuous Integration/Continuous Delivery/Deployment)**
อัตโนมัติทุกขั้นตอนนี้ ทำให้กระบวนการ**เร็วขึ้น, สม่ำเสมอ, และปลอดภัย
มากขึ้น**

## 2. Continuous Integration (CI) คืออะไร

**CI** คือการ**รวม (integrate) โค้ดจากทุกคนบ่อย ๆ** (เช่น ทุกครั้งที่
push) พร้อม**รัน automated test ทันที** เพื่อจับปัญหาให้เร็วที่สุด

```java
public class ContinuousIntegrationDemo {
    /*
     * ไม่มี CI: developer 5 คนทำงานแยกกันหลายสัปดาห์ แล้วค่อย merge รวมกันครั้งใหญ่
     *   -> เกิด "merge conflict" จำนวนมาก, bug ที่เกิดจากโค้ดชนกันถูกพบช้ามาก (integration hell)
     *
     * มี CI: ทุกครั้งที่ push code (แม้แค่ commit เล็ก ๆ) -> CI server รัน build + test อัตโนมัติทันที
     *   -> ถ้ามีปัญหา (compile error, test fail) รู้ตัวภายในไม่กี่นาที ไม่ใช่หลังจากผ่านไปหลายสัปดาห์
     *   -> แก้ไขปัญหาได้เร็วเพราะ scope ของการเปลี่ยนแปลงยังเล็ก (แค่ commit ล่าสุด)
     */
}
```

## 3. Continuous Delivery vs Continuous Deployment

```java
public class DeliveryVsDeploymentDemo {
    /*
     * Continuous Delivery: ทุก commit ที่ผ่าน CI (build+test สำเร็จ) พร้อม deploy ได้ทันที
     *   แต่ "การ deploy จริงไปที่ production ยังต้องมีคนกดปุ่มอนุมัติ" (manual approval gate)
     *
     * Continuous Deployment: ทุก commit ที่ผ่าน CI จะถูก deploy ไปที่ production "โดยอัตโนมัติ"
     *   ไม่มีคนต้องกดปุ่มอนุมัติเลย - ต้องมั่นใจมากว่า automated test ครอบคลุมเพียงพอ (ทบทวน Part 58, 84)
     *
     * ส่วนใหญ่บริษัทเลือก Continuous Delivery สำหรับ production (มี manual gate)
     * และ Continuous Deployment สำหรับ environment ที่มีความเสี่ยงต่ำกว่า (dev, staging)
     */
}
```

## 4. GitHub Actions: เขียน CI Pipeline พื้นฐาน

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: maven   # cache dependency (ทบทวนแนวคิด layer caching จาก Part 95)

      - name: Build with Maven
        run: mvn clean compile

      - name: Run tests
        run: mvn test   # รัน unit test + integration test (ทบทวน Part 58-60, 84)

      - name: Package application
        run: mvn package -DskipTests
```

```java
public class GitHubActionsTriggersExplanation {
    /*
     * "on: push/pull_request" กำหนดว่า pipeline นี้รันเมื่อไหร่:
     *   - push ไปที่ main/develop branch
     *   - เปิด/อัปเดต pull request ที่จะ merge เข้า main
     *
     * ทำให้ทุก pull request ถูกทดสอบอัตโนมัติก่อน merge - ป้องกันโค้ดที่มี bug เข้าสู่ main branch
     * (ทบทวนความสำคัญของ code review + automated check ร่วมกัน)
     */
}
```

## 5. Pipeline Stage: Build, Test, Package

```yaml
jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      - name: Run unit tests
        run: mvn test -Dtest='**/*Test' # ทบทวน naming convention จาก Part 58

      - name: Run integration tests with Testcontainers
        run: mvn verify -Dtest='**/*IntegrationTest'
        # CI server ต้องมี Docker daemon ให้ Testcontainers ใช้งานได้ (ทบทวน Part 84)

      - name: Generate test coverage report
        run: mvn jacoco:report

      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: target/site/jacoco/
```

**หลักการสำคัญ**: pipeline ควร**fail เร็วที่สุด** — รัน unit test (เร็ว)
ก่อน integration test (ช้ากว่า) ถ้า unit test fail ก็ไม่ต้องเสียเวลารัน
integration test ต่อ (ทบทวนแนวคิด Testing Pyramid จาก Part 58, 84)

## 6. Build Docker Image และ Push ไปที่ Registry

```yaml
  build-and-push-image:
    needs: build-and-test   # รันต่อจาก job นี้เท่านั้นถ้า build-and-test สำเร็จ
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Log in to Docker Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}   # secret ที่เก็บใน GitHub - ไม่ hardcode (ทบทวน Part 95)

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ghcr.io/mycompany/order-service:${{ github.sha }}
          # ใช้ commit SHA เป็น tag - ทำให้ image แต่ละตัวสืบย้อนกลับไปหา commit ที่แน่นอนได้
```

## 7. Deploy อัตโนมัติไปที่ Kubernetes

```yaml
  deploy-to-staging:
    needs: build-and-push-image
    runs-on: ubuntu-latest
    steps:
      - name: Set up kubectl
        uses: azure/setup-kubectl@v4

      - name: Deploy to Kubernetes staging
        run: |
          kubectl set image deployment/order-service \
            order-service=ghcr.io/mycompany/order-service:${{ github.sha }} \
            --namespace=staging
        env:
          KUBECONFIG: ${{ secrets.KUBE_CONFIG_STAGING }}
        # ทบทวน Rolling Update จาก Part 96 - Kubernetes อัปเดต Pod ทีละตัวแบบ zero-downtime
```

```java
public class PipelineFlowSummaryDemo {
    /*
     * Flow ทั้งหมด: developer push code
     *   -> job "build-and-test" รัน build + unit test + integration test (หัวข้อ 5)
     *   -> ถ้าสำเร็จ -> job "build-and-push-image" สร้าง Docker image push ไปที่ registry (หัวข้อ 6)
     *   -> ถ้าสำเร็จ -> job "deploy-to-staging" สั่ง Kubernetes อัปเดต Deployment (หัวข้อ 7)
     *   -> ทั้งหมดนี้เกิดขึ้นอัตโนมัติภายในไม่กี่นาที โดยไม่ต้องมีคนทำอะไรด้วยมือเลย (Continuous Deployment
     *      สำหรับ staging - ทบทวนหัวข้อ 3)
     */
}
```

## 8. Environment Strategy: Dev, Staging, Production

```java
public class EnvironmentStrategyDemo {
    /*
     * Dev: environment ที่ developer ทดสอบงานที่กำลังพัฒนา - deploy บ่อยมาก, ไม่ต้องมั่นคงมาก
     * Staging: environment ที่เหมือน production มากที่สุด (same config, similar data scale)
     *          ใช้ทดสอบครั้งสุดท้ายก่อน deploy จริง - ทีม QA/business ตรวจสอบที่นี่
     * Production: environment จริงที่ผู้ใช้จริงเข้าถึง - deploy ต้องระมัดระวังที่สุด
     *
     * Pipeline ทั่วไป: push ไปที่ dev -> auto-deploy dev ทันที
     *                 merge เข้า main -> auto-deploy staging + รัน automated test เพิ่มเติม
     *                 manual approval -> deploy production (Continuous Delivery - ทบทวนหัวข้อ 3)
     */
}
```

```yaml
  deploy-to-production:
    needs: deploy-to-staging
    runs-on: ubuntu-latest
    environment:
      name: production   # GitHub Environment ที่ตั้งค่า "required reviewers" ไว้
    steps:
      - name: Deploy to Kubernetes production
        run: |
          kubectl set image deployment/order-service \
            order-service=ghcr.io/mycompany/order-service:${{ github.sha }} \
            --namespace=production
        env:
          KUBECONFIG: ${{ secrets.KUBE_CONFIG_PRODUCTION }}
        # "environment: production" ทำให้ job นี้ "รอการอนุมัติจากคนที่กำหนดไว้" ก่อนรันจริง
        # (Manual Approval Gate - ทบทวนหัวข้อ 3)
```

## 9. Deployment Strategies: Blue-Green และ Canary

```java
public class BlueGreenDeploymentDemo {
    /*
     * Blue-Green Deployment:
     *   มี environment 2 ชุดคู่กัน: "Blue" (เวอร์ชันปัจจุบันที่ผู้ใช้เข้าอยู่) และ "Green" (เวอร์ชันใหม่)
     *   1. Deploy เวอร์ชันใหม่ไปที่ Green (ยังไม่มีผู้ใช้เข้าถึง)
     *   2. ทดสอบ Green อย่างละเอียดโดยไม่กระทบผู้ใช้จริง (ที่ยังใช้ Blue อยู่)
     *   3. สลับ traffic ทั้งหมดจาก Blue ไป Green ทันที (เปลี่ยนที่ Load Balancer/Service)
     *   4. ถ้าพบปัญหา -> สลับกลับไป Blue ได้ทันที (rollback เร็วมาก เพราะ Blue ยังรันอยู่)
     *   ข้อเสีย: ต้องมี infrastructure สำรองเต็มรูปแบบ 2 ชุด (cost สูงกว่า)
     */
}
```

```java
public class CanaryDeploymentDemo {
    /*
     * Canary Deployment (ทบทวนชื่อจาก "canary in a coal mine" - นกที่ใช้เตือนอันตรายในเหมืองสมัยก่อน):
     *   1. Deploy เวอร์ชันใหม่ไปแค่ "ส่วนน้อย" ของ Pod (เช่น 1 ใน 10 instance = 10% ของ traffic)
     *   2. Monitor metrics ของ Pod ใหม่อย่างใกล้ชิด (error rate, latency - ปูทางสู่ Part 99)
     *   3. ถ้าปกติดี -> เพิ่มสัดส่วน traffic ไปเวอร์ชันใหม่ทีละน้อย (10% -> 50% -> 100%)
     *   4. ถ้าพบปัญหาตั้งแต่ช่วง 10% -> "ตัด" Pod เวอร์ชันใหม่ทิ้ง กระทบผู้ใช้แค่ 10% เท่านั้น
     *   ข้อดี: ความเสี่ยงต่ำกว่า Blue-Green มาก เพราะจำกัดผลกระทบตั้งแต่ต้น ไม่ต้องมี infra สำรองเต็มชุด
     */
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เพิ่ม step ใน GitHub Actions workflow ที่รัน static code analysis
(เช่น Checkstyle) ก่อนขั้นตอน test เพื่อ fail เร็วที่สุดถ้าโค้ดไม่ตรงตาม
มาตรฐาน

**เฉลย:**
```yaml
      - name: Run Checkstyle
        run: mvn checkstyle:check
      - name: Run tests
        run: mvn test
```
วาง Checkstyle **ก่อน** test เพราะเป็นการตรวจสอบที่**เร็วกว่า**การรัน
test ทั้งชุด (ทบทวนหลักการ fail fast จากหัวข้อ 5) — ถ้าโค้ดไม่ผ่าน
มาตรฐานพื้นฐาน ก็ไม่ต้องเสียเวลารัน test ที่ใช้เวลานานกว่าเลย

**2)** อธิบายความแตกต่างระหว่าง Continuous Delivery และ Continuous
Deployment พร้อมยกตัวอย่างสถานการณ์ที่ควรเลือกแต่ละแบบ

**เฉลย**: **Continuous Delivery** หมายถึงทุก commit ที่ผ่าน automated
test **พร้อมสำหรับการ deploy** แต่**ยังต้องมีคนอนุมัติ (manual gate)**
ก่อนที่จะ deploy จริงไปยัง environment นั้น ส่วน **Continuous Deployment**
คือทุก commit ที่ผ่าน test จะถูก**deploy โดยอัตโนมัติทันทีโดยไม่มีคนต้อง
อนุมัติเลย** (ทบทวนหัวข้อ 3) ควรเลือก **Continuous Deployment** สำหรับ
environment ที่มีความเสี่ยงต่ำ เช่น **dev หรือ staging** ที่ผลกระทบจาก
ความผิดพลาดจำกัดอยู่แค่ทีมพัฒนาหรือทีมทดสอบภายใน ในขณะที่ **Continuous
Delivery** (มี manual approval) เหมาะกับ **production** ที่ผู้ใช้จริง
ได้รับผลกระทบโดยตรง — การมีคนตรวจสอบครั้งสุดท้ายก่อน deploy จริงช่วยลด
ความเสี่ยงจากกรณีที่ automated test ยังไม่ครอบคลุมทุกสถานการณ์

**3)** ทีมหนึ่งกำลังจะ deploy ฟีเจอร์ใหม่ที่มีความเสี่ยงสูง (เปลี่ยน
business logic การคำนวณราคาสินค้า) ควรเลือก Blue-Green Deployment หรือ
Canary Deployment และทำไม

**เฉลย**: ควรเลือก **Canary Deployment** เพราะฟีเจอร์นี้**มีความเสี่ยงสูง
และกระทบ business logic สำคัญ** (การคำนวณราคา) — ถ้าใช้ **Blue-Green**
(หัวข้อ 9) การสลับ traffic ทั้งหมดจาก Blue ไป Green **เกิดขึ้นทันทีทั้ง
100% ของผู้ใช้พร้อมกัน** ถ้ามี bug ที่ automated test ตรวจไม่พบ (เช่น
edge case ที่เกิดขึ้นเฉพาะกับข้อมูลจริงบางแบบ) **ผู้ใช้ทุกคนจะได้รับ
ผลกระทบพร้อมกันทันที** ก่อนที่ทีมจะรู้ตัวและ rollback ได้ ในขณะที่
**Canary Deployment** ปล่อยเวอร์ชันใหม่ให้รับ traffic แค่**ส่วนน้อย
ก่อน** (เช่น 10%) ทำให้ทีมสามารถ**สังเกต metrics และผลกระทบจริงในสภาพ
แวดล้อม production**ด้วยผู้ใช้จำนวนจำกัดก่อน ถ้าพบปัญหาการคำนวณราคาผิด
พลาด ผลกระทบจะจำกัดอยู่แค่ 10% ของผู้ใช้เท่านั้น และสามารถตัดเวอร์ชันใหม่
ทิ้งได้ทันทีโดยไม่กระทบผู้ใช้ส่วนใหญ่ ซึ่งเหมาะกับความเสี่ยงสูงของ
ฟีเจอร์นี้มากกว่า

### สรุปเนื้อหา Part 97

- CI/CD อัตโนมัติกระบวนการ build, test, package, deploy ลดความผิดพลาด
  จากมนุษย์และเพิ่มความเร็วในการส่งมอบ
- Continuous Integration รวมโค้ดและทดสอบบ่อย ๆ เพื่อจับปัญหาให้เร็ว
- Continuous Delivery มี manual approval gate ก่อน deploy จริง;
  Continuous Deployment deploy อัตโนมัติทั้งหมดไม่มี gate
- GitHub Actions ใช้ YAML กำหนด job/step ของ pipeline พร้อม trigger ตาม
  event (push, pull_request)
- Pipeline ควร fail fast: รันการตรวจสอบที่เร็วที่สุดก่อน (lint → unit
  test → integration test)
- Docker image ที่ build ใน CI ควร tag ด้วย commit SHA เพื่อสืบย้อนกลับ
  ไปหา commit ที่แน่นอนได้
- Environment Strategy (Dev/Staging/Production) แยกระดับความเสี่ยงและ
  ความเข้มงวดของการ deploy
- Blue-Green Deployment สลับ traffic ทั้งหมดทันที; Canary Deployment
  ปล่อยทีละน้อยเพื่อลดความเสี่ยง

**ต่อไป**: [Part 98 — Cloud Deployment: AWS/GCP](./part-098-cloud-deployment.md)
