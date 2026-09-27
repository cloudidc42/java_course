# Part 96: Kubernetes

> ขั้นตอนที่ 951-960 ของหลักสูตร | ระดับ: ระดับมืออาชีพ/โลก

## สารบัญ

1. ปัญหา: Docker Compose ไม่พอสำหรับ Production ขนาดใหญ่
2. Kubernetes คืออะไร: แนวคิดหลัก
3. Pod, Node, Cluster: หน่วยพื้นฐานของ Kubernetes
4. Deployment: การจัดการ Replica และ Self-healing
5. Service: การเข้าถึง Pod อย่างมั่นคง
6. ConfigMap และ Secret
7. Horizontal Pod Autoscaler (HPA)
8. Rolling Update และ Rollback
9. Liveness Probe และ Readiness Probe
10. แบบฝึกหัดและสรุป

---

## 1. ปัญหา: Docker Compose ไม่พอสำหรับ Production ขนาดใหญ่

ทบทวนจาก Part 95: Docker Compose ดีสำหรับรันหลาย container **บนเครื่อง
เดียว** — แต่ระบบ production จริงต้องการมากกว่านั้น:

```java
public class DockerComposeLimitationsDemo {
    /*
     * Docker Compose ทำไม่ได้ (หรือทำได้ยากมาก):
     *   1. รัน container กระจายไปหลายเครื่อง (server) พร้อมกัน - ทำงานแค่บนเครื่องเดียว
     *   2. Auto-scaling: เพิ่ม/ลด instance ของ service อัตโนมัติตามโหลด
     *   3. Self-healing: ถ้า container ตาย ต้องมีคนสั่ง restart เอง (docker compose ไม่ทำอัตโนมัติ)
     *   4. Rolling update แบบ zero-downtime ข้ามหลายเครื่องพร้อมกัน
     *   5. Load balancing ข้ามหลาย node/เครื่อง
     *
     * Kubernetes (K8s) คือ "Container Orchestration Platform" ที่แก้ปัญหาเหล่านี้ทั้งหมด
     * ในระดับ production ที่มีเครื่องจำนวนมาก (cluster)
     */
}
```

## 2. Kubernetes คืออะไร: แนวคิดหลัก

Kubernetes จัดการ container จำนวนมากข้ามหลายเครื่อง (nodes) โดยอัตโนมัติ
ตาม **"desired state"** ที่เราประกาศไว้ — เราไม่ต้องสั่งทีละขั้นตอน
(imperative) แค่บอกว่า "อยากได้อะไร" (declarative) แล้ว Kubernetes จัดการ
ให้เอง

```java
public class DeclarativeVsImperativeDemo {
    /*
     * Imperative (สั่งทีละขั้นตอน - เหมือน docker run):
     *   "รัน container นี้ที่เครื่อง A, ถ้าตายให้รันที่เครื่อง B, เช็คทุก 5 วินาทีว่ายังรันอยู่..."
     *
     * Declarative (ประกาศ desired state - แบบ Kubernetes):
     *   "ฉันต้องการให้ order-service รันอยู่ 3 instance เสมอ"
     *   Kubernetes จะคอยตรวจสอบและปรับให้ตรงกับ desired state นี้ตลอดเวลาโดยอัตโนมัติ
     *   ถ้า instance หนึ่งตาย -> Kubernetes สร้างตัวใหม่แทนทันที (self-healing - หัวข้อ 4)
     */
}
```

## 3. Pod, Node, Cluster: หน่วยพื้นฐานของ Kubernetes

```
Cluster
┌───────────────────────────────────────────────┐
│  Node 1                    Node 2               │
│  ┌─────────┐ ┌─────────┐   ┌─────────┐          │
│  │  Pod A  │ │  Pod B  │   │  Pod C  │          │
│  │(container)│(container)│   │(container)│          │
│  └─────────┘ └─────────┘   └─────────┘          │
└───────────────────────────────────────────────┘
```

- **Pod**: หน่วยที่เล็กที่สุดที่ Kubernetes จัดการ — ปกติมี **1 container**
  ต่อ 1 Pod (แม้จะรองรับหลาย container ต่อ pod ได้ในบางกรณี)
- **Node**: เครื่อง (physical หรือ virtual machine) หนึ่งเครื่องที่รัน
  Pod อยู่
- **Cluster**: กลุ่มของ Node ทั้งหมดที่ทำงานร่วมกันภายใต้การควบคุมของ
  Kubernetes

## 4. Deployment: การจัดการ Replica และ Self-healing

```yaml
# order-service-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3   # ต้องการ 3 instance เสมอ (desired state - ทบทวนหัวข้อ 2)
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
        - name: order-service
          image: mycompany/order-service:1.0.0   # image ที่ build จาก Dockerfile (ทบทวน Part 95)
          ports:
            - containerPort: 8080
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
```

```java
public class DeploymentSelfHealingDemo {
    /*
     * Kubernetes ตรวจสอบตลอดเวลาว่ามี Pod ที่รัน order-service อยู่ครบ 3 instance ตาม "replicas: 3"
     *
     * ถ้า Pod หนึ่งตาย (crash, node ที่มัน host อยู่ล่ม):
     *   Kubernetes ตรวจพบทันทีว่าจำนวน Pod ที่ทำงานจริง (2) ไม่ตรงกับ desired state (3)
     *   -> สร้าง Pod ใหม่แทนที่ทันทีโดยอัตโนมัติ (self-healing) โดยไม่ต้องมีคนเข้ามาสั่งเอง
     *
     * นี่คือความแตกต่างสำคัญจาก Docker Compose ที่ไม่มี self-healing แบบนี้
     */
}
```

```java
public class ResourceRequestsLimitsExplanation {
    /*
     * requests: ทรัพยากรขั้นต่ำที่ Pod ต้องการ - Kubernetes ใช้ค่านี้ตัดสินใจว่าจะวาง Pod บน node ไหน
     * limits: ทรัพยากรสูงสุดที่ Pod ใช้ได้ - ถ้าใช้เกิน CPU limit จะถูกจำกัด (throttle)
     *         ถ้าใช้เกิน memory limit จะถูก kill (OOMKilled) และ Kubernetes จะสร้าง Pod ใหม่แทน
     * การตั้งค่านี้ป้องกัน Pod ตัวเดียวใช้ทรัพยากรทั้ง node จนกระทบ Pod อื่น (ทบทวนแนวคิด Bulkhead จาก Part 94)
     */
}
```

## 5. Service: การเข้าถึง Pod อย่างมั่นคง

ปัญหา: Pod แต่ละตัวมี **IP ที่เปลี่ยนไปทุกครั้งที่ถูกสร้างใหม่**
(ทบทวนปัญหาเดียวกันจาก Part 92) — **Service** ให้ **ที่อยู่คงที่** สำหรับ
เข้าถึงกลุ่ม Pod

```yaml
# order-service-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service   # จับคู่กับ label ของ Pod ใน Deployment (หัวข้อ 4)
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP   # เข้าถึงได้แค่ภายใน cluster (ไม่เปิดสู่ internet)
```

```java
public class ServiceLoadBalancingDemo {
    /*
     * Service "order-service" มี DNS name คงที่ภายใน cluster (คล้าย Eureka - ทบทวน Part 92)
     * แต่ทำงานที่ระดับ network ของ Kubernetes เอง (ไม่ต้องมี client library พิเศษเหมือน Eureka Client)
     *
     * InventoryService เรียก OrderService ผ่าน: http://order-service/api/orders
     * Kubernetes จะ load balance request ไปยัง Pod ตัวใดตัวหนึ่งจาก 3 instance ที่ label ตรงกับ selector
     * โดยอัตโนมัติ แม้ Pod จะถูกสร้าง/ทำลายไปกี่ครั้งก็ตาม ที่อยู่ Service ("order-service") ไม่เปลี่ยน
     */
}
```

## 6. ConfigMap และ Secret

ทบทวนจาก Part 94: Config Server เก็บ configuration แบบ centralized —
Kubernetes มีกลไกในตัวที่คล้ายกัน

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-config
data:
  application.properties: |
    spring.jpa.hibernate.ddl-auto=validate
    logging.level.com.mycompany=INFO
---
apiVersion: v1
kind: Secret
metadata:
  name: order-service-secret
type: Opaque
data:
  db-password: bXlzZWNyZXRwYXNzd29yZA==   # เก็บแบบ base64 encode (ไม่ใช่เข้ารหัสจริง - ทบทวน Part 83 หัวข้อ 9)
```

```yaml
# ใน Deployment - เชื่อม ConfigMap/Secret เข้ากับ Pod ผ่าน environment variable
spec:
  containers:
    - name: order-service
      envFrom:
        - configMapRef:
            name: order-service-config
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: order-service-secret
              key: db-password
```

**ข้อควรระวัง**: `Secret` เก็บค่าแบบ **base64 encode เท่านั้น ไม่ใช่การ
เข้ารหัส** (ทบทวนความแตกต่าง encoding/encryption จาก Part 83) — ต้องใช้
ร่วมกับ RBAC (Role-Based Access Control) ของ Kubernetes เพื่อจำกัดว่าใคร
เข้าถึง Secret ได้บ้าง หรือใช้ external secret manager สำหรับข้อมูลที่
sensitive มาก (ปูทางสู่ Part 98, 101)

## 7. Horizontal Pod Autoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70   # ถ้า CPU เฉลี่ยเกิน 70% -> เพิ่ม Pod อัตโนมัติ
```

```java
public class HorizontalPodAutoscalerDemo {
    /*
     * HPA ตรวจสอบ CPU utilization ของ Pod ทั้งหมดใน order-service Deployment ตลอดเวลา
     *
     * ช่วงเวลาปกติ (โหลดต่ำ): รันแค่ 3 Pod (minReplicas)
     * ช่วงโปรโมชั่น/เทศกาลลดราคา (โหลดสูง): CPU เฉลี่ยเกิน 70% -> HPA เพิ่ม Pod อัตโนมัติจนถึง 10 Pod (maxReplicas)
     * หลังโหลดลดลง: HPA ลด Pod กลับไปที่ 3 อัตโนมัติ - ประหยัดค่า infrastructure
     *
     * นี่คือ "Horizontal Scaling" (เพิ่มจำนวน instance) ต่างจาก "Vertical Scaling" (เพิ่มขนาดเครื่องเดิม)
     */
}
```

## 8. Rolling Update และ Rollback

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1   # ยอมให้ Pod ไม่พร้อมใช้งานได้สูงสุด 1 ตัวระหว่าง update
      maxSurge: 1         # สร้าง Pod ใหม่เกินจำนวน desired ได้สูงสุด 1 ตัวระหว่าง update
```

```java
public class RollingUpdateDemo {
    /*
     * Deploy เวอร์ชันใหม่: kubectl set image deployment/order-service order-service=mycompany/order-service:2.0.0
     *
     * Kubernetes จะ:
     *   1. สร้าง Pod ใหม่ (เวอร์ชัน 2.0.0) 1 ตัว (maxSurge=1) - ตอนนี้มี 4 Pod (3 เก่า + 1 ใหม่)
     *   2. รอ Pod ใหม่ผ่าน Readiness Probe (หัวข้อ 9) แล้วค่อยลบ Pod เก่า 1 ตัว
     *   3. ทำซ้ำจนกว่า Pod ทั้งหมดเป็นเวอร์ชัน 2.0.0 - ตลอดกระบวนการมี Pod พร้อมใช้งานเสมอ (zero-downtime!)
     *
     * ถ้าพบปัญหาหลัง deploy: kubectl rollout undo deployment/order-service
     *   -> Kubernetes ย้อนกลับไปที่เวอร์ชันก่อนหน้าทันทีด้วยกระบวนการ rolling update เดียวกัน
     */
}
```

## 9. Liveness Probe และ Readiness Probe

```yaml
spec:
  containers:
    - name: order-service
      livenessProbe:
        httpGet:
          path: /actuator/health/liveness   # ทบทวน Spring Boot Actuator จาก Part 94
          port: 8080
        initialDelaySeconds: 30
        periodSeconds: 10
      readinessProbe:
        httpGet:
          path: /actuator/health/readiness
          port: 8080
        initialDelaySeconds: 10
        periodSeconds: 5
```

```java
public class ProbesExplanationDemo {
    /*
     * Liveness Probe: เช็คว่า container "ยังมีชีวิตอยู่" หรือไม่ (ไม่ deadlock, ไม่ hang)
     *   ถ้า liveness probe ล้มเหลวซ้ำ ๆ -> Kubernetes "restart" container นั้นทันที
     *
     * Readiness Probe: เช็คว่า container "พร้อมรับ traffic" หรือยัง (เช่น ยังเชื่อมต่อ database ไม่สำเร็จ)
     *   ถ้า readiness probe ล้มเหลว -> Kubernetes "หยุดส่ง traffic" ไปที่ Pod นั้นชั่วคราว (ผ่าน Service)
     *   แต่ "ไม่ restart" - รอจนกว่า Pod พร้อมแล้วค่อยส่ง traffic กลับไปให้
     *
     * ความแตกต่างสำคัญ: Liveness = "ต้อง restart ไหม", Readiness = "ควรรับ traffic ไหม (ชั่วคราว)"
     * ใช้ผิดกันจะทำให้ระบบมีปัญหา เช่น restart container ที่แค่กำลัง warm-up (ควรใช้ readiness ไม่ใช่ liveness)
     */
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียน Deployment YAML สำหรับ `payment-service` ที่ต้องการ 2
replica เสมอ ใช้ image `mycompany/payment-service:1.0.0`

**เฉลย:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: payment-service
  template:
    metadata:
      labels:
        app: payment-service
    spec:
      containers:
        - name: payment-service
          image: mycompany/payment-service:1.0.0
          ports:
            - containerPort: 8080
```

**2)** อธิบายว่าทำไม Kubernetes Service จำเป็นแม้จะมี Deployment ที่
จัดการ Pod อยู่แล้ว

**เฉลย**: **Deployment** (หัวข้อ 4) มีหน้าที่**ดูแลว่ามี Pod จำนวนตามที่
กำหนดทำงานอยู่เสมอ**และ**สร้าง Pod ใหม่แทนตัวที่ตาย** แต่**ไม่ได้ให้
ที่อยู่คงที่**สำหรับเข้าถึง Pod เหล่านั้นเลย — **ทุกครั้งที่ Pod ถูกสร้าง
ใหม่ (แทนตัวที่ตายหรือระหว่าง rolling update) มันจะได้ IP address ใหม่
เสมอ** (ทบทวนปัญหาเดียวกันจาก Part 92 หัวข้อ 1) ถ้า service อื่นเก็บ IP
ของ Pod ไว้ตรง ๆ จะใช้งานไม่ได้ทันทีที่ Pod นั้นถูกแทนที่ **Service**
(หัวข้อ 5) แก้ปัญหานี้โดยให้ **DNS name และ IP ที่คงที่**สำหรับเข้าถึง
**กลุ่ม Pod ทั้งหมด**ที่ label ตรงกับ selector — ไม่ว่า Pod ข้างหลังจะ
ถูกสร้าง/ทำลายไปกี่ครั้ง ที่อยู่ของ Service ก็ไม่เปลี่ยน ทำให้ service
อื่นเรียกได้อย่างมั่นคงเสมอ

**3)** ทีมหนึ่ง deploy เวอร์ชันใหม่ของ `OrderService` แล้ว liveness probe
เริ่ม fail ทันทีเพราะแอปใช้เวลา 45 วินาทีในการเชื่อมต่อ database ตอน
startup (แต่ `initialDelaySeconds` ตั้งไว้แค่ 30 วินาที) จะเกิดอะไรขึ้น
และควรแก้ไขอย่างไร

**เฉลย**: เมื่อ `initialDelaySeconds` (30 วินาที) **น้อยกว่าเวลาที่แอป
ต้องการจริงในการพร้อมทำงาน** (45 วินาที) Kubernetes จะเริ่มเรียก
liveness probe **ก่อน**ที่แอปจะพร้อมเชื่อมต่อ database เสร็จ — probe จะ
**fail** เพราะแอปยังไม่ตอบสนองที่ endpoint นั้นได้ตามปกติ เมื่อ liveness
probe fail ซ้ำเกินเกณฑ์ (ทบทวนหัวข้อ 9) **Kubernetes จะสั่ง restart
container ทันที** — และเมื่อ container restart ใหม่ มันก็ต้องใช้เวลา 45
วินาทีเชื่อมต่อ database อีกครั้ง ซึ่ง liveness probe ก็จะ fail ที่ 30
วินาทีอีกเช่นเดิม **เกิดเป็นวงจร restart ไม่จบสิ้น (crash loop)** ทำให้
แอปไม่สามารถ start สำเร็จได้เลย วิธีแก้คือ**เพิ่ม `initialDelaySeconds`
ให้มากกว่าเวลา startup จริง**ของแอป (เช่น ตั้งเป็น 60 วินาทีเป็นอย่าง
น้อย) เพื่อให้ Kubernetes รอให้แอป startup เสร็จสมบูรณ์ก่อนเริ่มเรียก
liveness probe ครั้งแรก

### สรุปเนื้อหา Part 96

- Kubernetes เป็น Container Orchestration Platform ที่จัดการ container
  จำนวนมากข้ามหลายเครื่องตาม declarative desired state
- Pod เป็นหน่วยที่เล็กที่สุด, Node คือเครื่องที่รัน Pod, Cluster คือกลุ่ม
  Node ทั้งหมด
- Deployment จัดการจำนวน replica และ self-healing (สร้าง Pod ใหม่แทน
  ตัวที่ตายอัตโนมัติ)
- Service ให้ที่อยู่คงที่สำหรับเข้าถึงกลุ่ม Pod แม้ Pod จะถูกสร้าง/ทำลาย
  ไปกี่ครั้งก็ตาม
- ConfigMap/Secret จัดการ configuration/ข้อมูลลับแบบ centralized คล้าย
  Config Server (Part 94)
- HPA ปรับจำนวน Pod อัตโนมัติตามโหลด (horizontal scaling)
- Rolling Update deploy เวอร์ชันใหม่แบบ zero-downtime; Rollback ย้อนกลับ
  ได้ทันทีถ้าพบปัญหา
- Liveness Probe ตัดสินว่าต้อง restart container ไหม, Readiness Probe
  ตัดสินว่าควรรับ traffic ไหม — ตั้งค่าผิดทำให้เกิด crash loop ได้

**ต่อไป**: [Part 97 — CI/CD Pipeline](./part-097-cicd-pipeline.md)
