# Part 98: Cloud Deployment: AWS/GCP

> ขั้นตอนที่ 971-980 ของหลักสูตร | ระดับ: ระดับมืออาชีพ/โลก

## สารบัญ

1. ทำไมต้องใช้ Cloud Provider แทนเซิร์ฟเวอร์ของตัวเอง
2. Compute Options: VM, Container Service, Serverless
3. Deploy Spring Boot บน AWS Elastic Beanstalk
4. Managed Kubernetes: AWS EKS และ GCP GKE
5. Managed Database: AWS RDS
6. Object Storage: AWS S3 สำหรับไฟล์
7. Secret Management: AWS Secrets Manager
8. Serverless: AWS Lambda กับ Java
9. Infrastructure as Code: Terraform เบื้องต้น
10. แบบฝึกหัดและสรุป

---

## 1. ทำไมต้องใช้ Cloud Provider แทนเซิร์ฟเวอร์ของตัวเอง

ทบทวนจาก Part 95, 96: เราเรียนรู้วิธี containerize และ orchestrate แอป
ด้วย Docker/Kubernetes — แต่ **เครื่องจริงที่รัน Kubernetes cluster** มา
จากไหน? การซื้อและดูแลเซิร์ฟเวอร์เอง (on-premise) มีข้อจำกัดมาก:

```java
public class CloudProviderBenefitsDemo {
    /*
     * On-premise (ซื้อเซิร์ฟเวอร์เอง):
     *   - ต้องประเมิน capacity ล่วงหน้า (ซื้อเครื่องแรงเกินไป = เปลืองเงิน, น้อยเกินไป = รับโหลดไม่พอ)
     *   - ต้องดูแล hardware เอง (ไฟดับ, ฮาร์ดดิสก์เสีย, ต้อง maintenance เอง)
     *   - Scale ขึ้น/ลงทำได้ช้า (ต้องซื้อเครื่องใหม่ ใช้เวลาหลายวัน/สัปดาห์)
     *
     * Cloud Provider (AWS, GCP, Azure):
     *   - Pay-as-you-go: จ่ายตามที่ใช้จริง ไม่ต้องซื้อเครื่องล่วงหน้า
     *   - Scale ขึ้น/ลงได้ในนาทีเดียว (ทบทวน HPA จาก Part 96)
     *   - Managed Service: ไม่ต้องดูแล hardware/OS patching เอง (ปูทางสู่หัวข้อ 4, 5)
     *   - มี infrastructure กระจายทั่วโลก (region ต่าง ๆ) ทำให้ deploy ใกล้ผู้ใช้ได้ง่าย
     */
}
```

## 2. Compute Options: VM, Container Service, Serverless

```java
public class ComputeOptionsSpectrum {
    /*
     * ระดับการควบคุม vs ความสะดวก (trade-off):
     *
     * Virtual Machine (AWS EC2, GCP Compute Engine):
     *   ควบคุมเต็มที่ - เลือก OS, ติดตั้งอะไรก็ได้ - แต่ต้องดูแล patching, scaling เอง
     *
     * Container Service (AWS ECS/EKS, GCP GKE - ทบทวน Kubernetes จาก Part 96):
     *   ควบคุมน้อยกว่า VM แต่มากกว่า Serverless - จัดการ container orchestration ให้ส่วนใหญ่
     *
     * Serverless (AWS Lambda, GCP Cloud Functions - หัวข้อ 8):
     *   ควบคุมน้อยที่สุด - แค่เขียน function, cloud provider จัดการ infrastructure ทั้งหมด
     *   จ่ายเงินตาม "การเรียกจริง" ไม่ใช่ตามเวลาที่เครื่องรันอยู่ (ไม่มี idle cost)
     */
}
```

## 3. Deploy Spring Boot บน AWS Elastic Beanstalk

**Elastic Beanstalk** เป็นบริการที่ **จัดการ infrastructure ให้เกือบ
ทั้งหมด** — เหมาะกับทีมที่ไม่ต้องการจัดการ Kubernetes เต็มรูปแบบ

```java
public class ElasticBeanstalkDeployDemo {
    /*
     * ขั้นตอน:
     *   1. Build JAR ด้วย Maven/Gradle (ทบทวน Part 61, 62)
     *   2. Upload JAR ผ่าน AWS Console หรือ EB CLI: eb deploy
     *   3. Elastic Beanstalk จัดการ:
     *      - สร้าง EC2 instance (VM) รัน JAR
     *      - ตั้งค่า Load Balancer (กระจาย traffic - ทบทวน Part 92 หัวข้อ 7)
     *      - Auto Scaling (เพิ่ม/ลด instance ตามโหลด - ทบทวน HPA จาก Part 96)
     *      - Health monitoring (คล้าย Liveness Probe จาก Part 96)
     *
     * เหมาะกับทีมเล็กหรือโปรเจกต์ที่ไม่ซับซ้อนมาก ที่ไม่อยากจัดการ Kubernetes เต็มรูปแบบเอง
     */
}
```

## 4. Managed Kubernetes: AWS EKS และ GCP GKE

ทบทวนจาก Part 96: การรัน Kubernetes cluster เอง (self-managed) ต้อง
ดูแล **control plane** (etcd, API server, scheduler) ซึ่งซับซ้อนมาก —
**Managed Kubernetes** ให้ cloud provider ดูแลส่วนนี้ให้

```java
public class ManagedKubernetesDemo {
    /*
     * AWS EKS (Elastic Kubernetes Service) / GCP GKE (Google Kubernetes Engine):
     *   - Cloud provider ดูแล control plane ให้เต็มรูปแบบ (high availability, upgrade, patching)
     *   - เราแค่ดูแล "worker node" และ deploy application ผ่าน kubectl ตามปกติ (YAML เดียวกับ Part 96!)
     *   - ทุก YAML (Deployment, Service, ConfigMap) ที่เขียนใน Part 96 ใช้ได้เหมือนกันทุกที่
     *     (เขียนครั้งเดียว รันได้ทั้ง local cluster และ managed cloud cluster - portability สูง)
     *
     * นี่คือข้อดีสำคัญของ Kubernetes: "vendor-agnostic" - ไม่ผูกติดกับ cloud provider ตัวใดตัวหนึ่ง
     * ย้ายจาก AWS ไป GCP ทำได้ง่ายกว่าระบบที่ผูกกับ service เฉพาะของ provider ใดตัวหนึ่งมาก
     */
}
```

## 5. Managed Database: AWS RDS

ทบทวนจาก Part 66: การรัน PostgreSQL เอง (self-managed) ต้องดูแล backup,
replication, patching, scaling เอง — **AWS RDS (Relational Database
Service)** จัดการเรื่องนี้ให้

```java
public class RdsManagedDatabaseDemo {
    /*
     * AWS RDS จัดการให้อัตโนมัติ:
     *   - Automated Backup: backup ทุกวันอัตโนมัติ พร้อม point-in-time recovery
     *   - Multi-AZ Deployment: มี replica สำรองใน availability zone อื่น - ถ้า AZ หนึ่งล่ม สลับไปใช้ทันที
     *   - Read Replica: สร้าง replica สำหรับ query อ่านอย่างเดียว แบ่งโหลดจาก primary database
     *   - Automated patching: อัปเดต security patch ของ database engine ให้อัตโนมัติ (ในช่วงเวลาที่กำหนด)
     *
     * แอป Spring Boot เชื่อมต่อ RDS เหมือน PostgreSQL ธรรมดา (ทบทวน spring.datasource.url จาก Part 64-66)
     * - ไม่ต้องเปลี่ยนโค้ดเลย แค่เปลี่ยน connection string ไปที่ RDS endpoint
     */
}
```

```properties
# application-production.properties
spring.datasource.url=jdbc:postgresql://mydb.xxxxx.ap-southeast-1.rds.amazonaws.com:5432/orders
spring.datasource.username=${DB_USERNAME}
spring.datasource.password=${DB_PASSWORD}
```

## 6. Object Storage: AWS S3 สำหรับไฟล์

ทบทวนจาก Part 36-37: File I/O ที่เราเรียนใช้ **local filesystem** — แต่
ใน production, container/Pod อาจถูกทำลายและสร้างใหม่ตลอดเวลา (ทบทวนจาก
Part 96) ทำให้ **ไฟล์ที่เก็บใน local filesystem ของ container หายไปได้**

```java
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.s3.model.PutObjectRequest;
import java.nio.file.Path;

public class S3FileUploadService {
    private final S3Client s3Client;
    private static final String BUCKET_NAME = "mycompany-order-attachments";

    public S3FileUploadService(S3Client s3Client) { this.s3Client = s3Client; }

    public void uploadFile(String key, Path filePath) {
        PutObjectRequest request = PutObjectRequest.builder()
                .bucket(BUCKET_NAME)
                .key(key) // เช่น "orders/42/invoice.pdf"
                .build();
        s3Client.putObject(request, filePath);
        // ไฟล์นี้จะอยู่ถาวรใน S3 ไม่ว่า container ของแอปจะถูกสร้าง/ทำลายไปกี่ครั้งก็ตาม (ทบทวน Volume จาก Part 95)
        // เข้าถึงได้จากทุก Pod/instance ของแอป ไม่ผูกกับ container ตัวใดตัวหนึ่ง (ต่างจาก local filesystem)
    }
}
```

## 7. Secret Management: AWS Secrets Manager

ทบทวนคำเตือนจาก Part 95, 96: การเก็บ secret ใน environment variable
ธรรมดา หรือ Kubernetes Secret (แค่ base64 encode) ยังมีความเสี่ยง — **AWS
Secrets Manager** เก็บ secret แบบเข้ารหัสจริงและมี **automatic
rotation**

```java
import software.amazon.awssdk.services.secretsmanager.SecretsManagerClient;
import software.amazon.awssdk.services.secretsmanager.model.GetSecretValueRequest;
import org.springframework.stereotype.Service;

@Service
public class SecretRetrievalService {
    private final SecretsManagerClient secretsManagerClient;

    public SecretRetrievalService(SecretsManagerClient secretsManagerClient) {
        this.secretsManagerClient = secretsManagerClient;
    }

    public String getDatabasePassword() {
        var request = GetSecretValueRequest.builder().secretId("prod/order-service/db-password").build();
        return secretsManagerClient.getSecretValue(request).secretString();
        // secret ถูกเข้ารหัสจริงด้วย AWS KMS (Key Management Service) - ต่างจาก base64 encode ธรรมดา
        // รองรับ "automatic rotation" - เปลี่ยน password อัตโนมัติเป็นระยะโดยไม่ต้อง redeploy แอป
    }
}
```

## 8. Serverless: AWS Lambda กับ Java

ทบทวนหัวข้อ 2: **Serverless** เหมาะกับงานที่**ไม่ต้องรันตลอดเวลา**
(event-driven) — จ่ายเงินแค่ตอนที่ function ถูกเรียกจริง

```java
import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;

public class OrderNotificationHandler implements RequestHandler<OrderCreatedEvent, String> {
    // ทบทวน OrderCreatedEvent จาก Part 88, 89

    @Override
    public String handleRequest(OrderCreatedEvent event, Context context) {
        System.out.println("Processing notification for order " + event.orderId());
        // ส่ง email/SMS แจ้งเตือน - เหมาะกับงาน asynchronous ที่เกิดไม่บ่อยมาก (ทบทวน Part 88 หัวข้อ 1)
        return "Notification sent for order " + event.orderId();
    }
    /*
     * ต่างจาก Spring Boot ที่รันตลอดเวลาบน server/Pod (ทบทวน Part 96):
     * Lambda function "ไม่มีเครื่องรันอยู่เลย" จนกว่าจะมี event เรียกเข้ามา (เช่น จาก SQS, S3, API Gateway)
     * AWS สร้าง instance ชั่วคราวรัน function นี้ ประมวลผลเสร็จแล้วปิดทันที - จ่ายเงินตามเวลาที่ใช้จริงเป็นวินาที
     *
     * ข้อจำกัด: มี "cold start" (ครั้งแรกที่เรียกช้ากว่าปกติเพราะต้องสร้าง instance ใหม่)
     * ไม่เหมาะกับงานที่ต้องการ response time ต่ำมากตลอดเวลา หรือ workload ที่สูงต่อเนื่องมาก
     */
}
```

## 9. Infrastructure as Code: Terraform เบื้องต้น

ทบทวนปัญหาจาก Part 94: การตั้งค่า infrastructure ด้วยมือผ่าน AWS
Console **ไม่มี version control, ทำซ้ำไม่ได้แน่นอน** — **Terraform**
ให้เขียน infrastructure เป็นโค้ด (เหมือนที่เราเขียน Kubernetes YAML
declarative ใน Part 96)

```hcl
# main.tf
resource "aws_db_instance" "orders_db" {
  identifier        = "orders-db"
  engine            = "postgres"
  engine_version    = "16"
  instance_class    = "db.t3.medium"
  allocated_storage = 50
  db_name           = "orders"
  username          = "admin"
  password          = var.db_password  # อ้างถึง variable แยก - ไม่ hardcode (ทบทวนคำเตือนจาก Part 95)
  multi_az          = true             # High Availability (ทบทวนหัวข้อ 5)
}

resource "aws_s3_bucket" "order_attachments" {
  bucket = "mycompany-order-attachments"
}
```

```java
public class TerraformBenefitsDemo {
    /*
     * terraform plan  -> แสดงว่าจะเปลี่ยนแปลง infrastructure อะไรบ้าง (ก่อนสั่งจริง)
     * terraform apply -> สร้าง/แก้ไข infrastructure ตามที่ประกาศไว้ใน .tf file
     *
     * ข้อดี (เหมือน declarative ของ Kubernetes ในหัวข้อ 4):
     *   - Infrastructure ทั้งหมดถูกเก็บเป็นโค้ดใน Git - มี version history, review ผ่าน pull request ได้
     *   - สร้าง environment ใหม่ (เช่น staging environment ที่เหมือน production เป๊ะ) ได้ซ้ำแบบแน่นอน
     *   - ลบความเสี่ยงจาก "การตั้งค่าด้วยมือที่ไม่มีใครจำได้ว่าทำอะไรไปบ้าง" (configuration drift)
     */
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียนโค้ด Java ที่ดึง database password จาก AWS Secrets Manager
แล้วใช้สร้าง `DataSource` connection string แบบ dynamic

**เฉลย:**
```java
@Configuration
public class DataSourceConfig {
    private final SecretRetrievalService secretService;

    public DataSourceConfig(SecretRetrievalService secretService) { this.secretService = secretService; }

    @Bean
    public DataSource dataSource() {
        String password = secretService.getDatabasePassword();
        HikariConfig config = new HikariConfig(); // ทบทวน HikariCP จาก Part 66
        config.setJdbcUrl("jdbc:postgresql://mydb.rds.amazonaws.com:5432/orders");
        config.setUsername("admin");
        config.setPassword(password);
        return new HikariDataSource(config);
    }
}
```

**2)** อธิบายว่าทำไมการเก็บไฟล์ที่ผู้ใช้อัปโหลด (เช่น รูปโปรไฟล์) ไว้ใน
local filesystem ของ container เป็นแนวทางที่มีความเสี่ยงในระบบที่ deploy
บน Kubernetes

**เฉลย**: ใน Kubernetes (ทบทวน Part 96) **Pod ถูกสร้างและทำลายอยู่ตลอด
เวลา** — เมื่อ Pod ถูก restart (เพราะ crash, rolling update, หรือ
scheduler ย้าย Pod ไปยัง node อื่น) **local filesystem ของ container
จะถูกล้างและสร้างใหม่ตามค่าใน image เสมอ** (container filesystem เป็น
ephemeral โดยธรรมชาติ ทบทวนแนวคิดนี้จาก Part 95 หัวข้อ 9) ทำให้**ไฟล์ที่
ผู้ใช้อัปโหลดไว้ก่อนหน้าจะหายไปทันที**เมื่อ Pod นั้นถูกแทนที่ด้วย Pod ใหม่
นอกจากนี้ถ้ามี Pod หลาย instance (replicas) พร้อมกัน (ทบทวน Deployment
จาก Part 96) **ไฟล์ที่อัปโหลดผ่าน Pod ตัวหนึ่งจะไม่ปรากฏใน Pod ตัวอื่น
เลย** เพราะแต่ละ container มี filesystem แยกกันเป็นอิสระ วิธีแก้คือใช้
**Object Storage (S3)** (หัวข้อ 6) ที่เป็น**บริการเก็บไฟล์แยกจาก
container โดยสมบูรณ์** ทุก Pod (ไม่ว่าจะสร้างใหม่กี่ครั้งหรือมีกี่
instance) เข้าถึงไฟล์เดียวกันใน S3 ได้เสมอ

**3)** อธิบายข้อดีของการใช้ Terraform (Infrastructure as Code) เทียบกับ
การตั้งค่า infrastructure ผ่าน Cloud Console ด้วยมือ

**เฉลย**: การตั้งค่าผ่าน **Cloud Console ด้วยมือ**ไม่มี**record ที่ตรวจ
สอบได้ว่าใครเปลี่ยนแปลงอะไรเมื่อไหร่** (ไม่มี version control) และเมื่อ
ต้องการสร้าง environment ใหม่ที่เหมือนกัน (เช่น staging ที่ต้องเหมือน
production) **ต้องจำและทำตามขั้นตอนเดิมด้วยมือทุกครั้ง** ซึ่งเสี่ยงต่อการ
ลืมขั้นตอนหรือตั้งค่าไม่ตรงกัน (เรียกว่า **configuration drift** — สอง
environment ที่ควรจะเหมือนกันแต่ค่อย ๆ ต่างกันไปเพราะมีคนแก้ไขด้วยมือไม่
สม่ำเสมอ) **Terraform** (หัวข้อ 9) แก้ปัญหานี้โดยให้เขียน infrastructure
เป็น**โค้ดที่เก็บใน Git** (ทบทวนแนวคิด declarative ที่คล้ายกับ Kubernetes
YAML จาก Part 96) ทำให้**การเปลี่ยนแปลงทุกครั้งผ่าน pull request ที่
review ได้**, มี**version history ที่ตรวจสอบย้อนหลังได้ว่าใครเปลี่ยนอะไร
เมื่อไหร่**, และสามารถ**สร้าง environment ใหม่ที่เหมือนกันเป๊ะได้ซ้ำ ๆ
อย่างแน่นอน**ด้วยคำสั่งเดียว (`terraform apply`) โดยไม่มีความเสี่ยงจาก
ความผิดพลาดของมนุษย์ที่ทำตามขั้นตอนด้วยมือ

### สรุปเนื้อหา Part 98

- Cloud Provider ให้ pay-as-you-go, scale ได้เร็ว, และมี managed service
  ที่ลดงาน operations ลงมาก เทียบกับ on-premise
- Compute Options มีสเปกตรัมจาก VM (ควบคุมเต็มที่) ถึง Serverless
  (ควบคุมน้อยที่สุดแต่สะดวกที่สุด)
- Managed Kubernetes (EKS/GKE) ดูแล control plane ให้ ทำให้ YAML ที่เขียน
  ใน Part 96 ใช้งานได้เหมือนกันทุกที่ (vendor-agnostic)
- Managed Database (RDS) จัดการ backup, replication, patching อัตโนมัติ
- Object Storage (S3) แก้ปัญหาไฟล์หายจาก ephemeral container filesystem
  ใน Kubernetes
- Secrets Manager เข้ารหัส secret จริงพร้อม automatic rotation ปลอดภัย
  กว่า environment variable/Kubernetes Secret ธรรมดา
- Serverless (Lambda) เหมาะกับงาน event-driven ที่ไม่ต้องรันตลอดเวลา
- Terraform (Infrastructure as Code) ทำให้ infrastructure มี version
  control, review ได้, และสร้างซ้ำได้แน่นอน

**ต่อไป**: [Part 99 — Monitoring และ Observability](./part-099-monitoring-observability.md)
