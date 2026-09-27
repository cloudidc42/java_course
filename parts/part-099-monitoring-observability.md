# Part 99: Monitoring และ Observability

> ขั้นตอนที่ 981-990 ของหลักสูตร | ระดับ: ระดับมืออาชีพ/โลก

## สารบัญ

1. ทำไม Microservices ต้องมี Observability
2. สามเสาหลักของ Observability: Logs, Metrics, Traces
3. Structured Logging และ Centralized Log Aggregation
4. Metrics ด้วย Micrometer และ Prometheus
5. Grafana: สร้าง Dashboard จาก Metrics
6. Distributed Tracing ด้วย OpenTelemetry
7. Correlation ID: เชื่อมโยง Log ข้าม Service
8. Alerting: แจ้งเตือนก่อนผู้ใช้รู้ตัว
9. SLI, SLO, SLA: การวัดความน่าเชื่อถือของระบบ
10. แบบฝึกหัดและสรุป

---

## 1. ทำไม Microservices ต้องมี Observability

ทบทวนจาก Part 91: ระบบ microservices มี**หลาย service กระจายอยู่หลาย
เครื่อง** — เมื่อเกิดปัญหา (เช่น response ช้าผิดปกติ) การ debug ด้วยการ
`docker logs` ทีละ container **ไม่พอ**อีกต่อไป

```java
public class ObservabilityNeedDemo {
    /*
     * Monolith เดียว: มี log file เดียว, debug ง่าย (ทบทวน Part 63 - Logging)
     *
     * Microservices 20 ตัว: request หนึ่งอาจผ่าน 5-6 service ก่อนตอบกลับ (ทบทวน API Composition จาก Part 91)
     *   ถ้า response ช้า -> "ช้าที่ service ไหน?" ต้องดู log ของทุก service แยกกันหรือ?
     *   ถ้า service ไหนมี error rate สูงขึ้น -> จะรู้ได้อย่างไรก่อนที่ผู้ใช้จะร้องเรียน?
     *
     * Observability คือความสามารถในการ "เข้าใจสถานะภายในของระบบ" จากข้อมูลที่ระบบส่งออกมา
     * (logs, metrics, traces) โดยไม่ต้องเดา - จำเป็นอย่างยิ่งสำหรับ distributed system
     */
}
```

## 2. สามเสาหลักของ Observability: Logs, Metrics, Traces

```java
public class ThreePillarsOfObservability {
    /*
     * Logs: บันทึกเหตุการณ์แบบละเอียด ณ จุดเวลาหนึ่ง (ทบทวน Part 63)
     *   "2026-01-15 10:23:45 ERROR OrderService - Failed to process order 42: Connection timeout"
     *   ดีสำหรับ: การ debug เจาะลึกปัญหาเฉพาะจุด
     *
     * Metrics: ตัวเลขที่วัดค่าตามเวลา (aggregatable) เช่น จำนวน request/วินาที, latency เฉลี่ย
     *   ดีสำหรับ: เห็นภาพรวม trend และ pattern ของระบบ, ตั้ง alert ได้ง่าย
     *
     * Traces: เส้นทางการเดินทางของ request เดียวข้ามหลาย service (ทบทวน Part 91 - หลาย service ต่อ 1 request)
     *   ดีสำหรับ: หาว่า "ช้าที่ service ไหน" ในการเดินทางของ request หนึ่ง ๆ
     *
     * สามอย่างนี้ทำงานเสริมกัน: Metrics แจ้งว่า "มีปัญหา" -> Traces บอกว่า "ปัญหาอยู่ที่ไหน"
     * -> Logs บอกรายละเอียดว่า "เกิดอะไรขึ้นจริง ๆ"
     */
}
```

## 3. Structured Logging และ Centralized Log Aggregation

ทบทวนจาก Part 63: SLF4J/Logback เขียน log เป็น**ข้อความธรรมดา** — ใน
ระบบที่มีหลาย service, log แบบข้อความอ่านยากเมื่อต้อง**ค้นหา/filter ข้าม
service จำนวนมาก**

```java
import net.logstash.logback.argument.StructuredArguments;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class StructuredLoggingDemo {
    private static final Logger log = LoggerFactory.getLogger(StructuredLoggingDemo.class);

    public void processOrder(Long orderId, String customerId) {
        // Structured Logging: log เป็น JSON พร้อม field ที่ query ได้ง่าย (ต่างจาก plain text)
        log.info("Order processed", StructuredArguments.kv("orderId", orderId),
                StructuredArguments.kv("customerId", customerId), StructuredArguments.kv("status", "SUCCESS"));
        /* Output เป็น JSON:
           {"message":"Order processed","orderId":42,"customerId":"cust-123","status":"SUCCESS"}
           ค้นหาได้ง่ายมาก เช่น "หา log ทั้งหมดที่ orderId=42" โดยไม่ต้อง parse ข้อความด้วย regex
        */
    }
}
```

```java
public class CentralizedLogAggregationDemo {
    /*
     * ELK Stack (Elasticsearch, Logstash, Kibana) หรือ Grafana Loki:
     *   1. ทุก service เขียน structured log (JSON) ไปที่ stdout (ทบทวน container logging convention จาก Part 95)
     *   2. Agent (Fluentd/Filebeat/Promtail) เก็บ log จากทุก container ส่งไปที่ระบบกลาง (Elasticsearch/Loki)
     *   3. ทีม dev/ops ค้นหา log จาก "ทุก service พร้อมกัน" ผ่านหน้าเว็บเดียว (Kibana/Grafana)
     *      เช่น ค้นหา "orderId:42" เจอ log จาก OrderService, InventoryService, PaymentService พร้อมกันทันที
     *
     * แก้ปัญหาการต้อง SSH เข้าไปที่ทุกเครื่อง/ทุก Pod เพื่อดู log แยกทีละตัว
     */
}
```

## 4. Metrics ด้วย Micrometer และ Prometheus

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```properties
management.endpoints.web.exposure.include=health,metrics,prometheus
management.metrics.tags.application=order-service
```

```java
public class SpringBootMetricsAutoDemo {
    /*
     * Spring Boot Actuator + Micrometer สร้าง metrics พื้นฐานให้อัตโนมัติ โดยไม่ต้องเขียนโค้ดเพิ่ม:
     *   - http_server_requests_seconds: latency ของทุก HTTP endpoint (ทบทวน Part 71, 78)
     *   - jvm_memory_used_bytes: memory usage ของ JVM (ทบทวน Part 67 - JVM Internals)
     *   - hikaricp_connections_active: จำนวน connection ที่ใช้งานอยู่ (ทบทวน Part 66)
     *
     * เข้าถึงได้ที่ GET /actuator/prometheus - Prometheus server จะ "scrape" (ดึงข้อมูล) endpoint นี้
     * เป็นระยะ (เช่น ทุก 15 วินาที) แล้วเก็บเป็น time-series database
     */
}
```

```java
import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.stereotype.Service;

@Service
public class OrderMetricsService {
    private final MeterRegistry meterRegistry;

    public OrderMetricsService(MeterRegistry meterRegistry) { this.meterRegistry = meterRegistry; }

    public void recordOrderCreated(double amount) {
        // Custom metric: นับจำนวน order ที่สร้างสำเร็จ (Counter)
        meterRegistry.counter("orders.created.total").increment();
        // Custom metric: บันทึกการกระจายตัวของยอดเงิน order (Distribution Summary)
        meterRegistry.summary("orders.amount").record(amount);
        // metric แบบ custom เหล่านี้เจาะจงกับ business logic ที่ metric อัตโนมัติของ framework ไม่มี
    }
}
```

## 5. Grafana: สร้าง Dashboard จาก Metrics

```java
public class GrafanaDashboardConceptDemo {
    /*
     * Grafana เชื่อมต่อกับ Prometheus (data source) แล้วสร้างกราฟจาก metrics ที่เก็บไว้
     *
     * Query ตัวอย่าง (PromQL):
     *   rate(http_server_requests_seconds_count{uri="/api/orders"}[5m])
     *   -> คำนวณ "request rate" ของ endpoint /api/orders ในช่วง 5 นาทีที่ผ่านมา
     *
     *   histogram_quantile(0.95, http_server_requests_seconds_bucket)
     *   -> คำนวณ P95 latency (95% ของ request เร็วกว่าค่านี้) - สำคัญกว่า average latency มาก
     *      เพราะ average อาจถูกบิดเบือนจาก request ส่วนน้อยที่เร็วมาก ในขณะที่ผู้ใช้ส่วนใหญ่รอนาน
     *
     * Dashboard รวม panel หลายตัว (request rate, error rate, latency percentile, JVM memory)
     * แสดงภาพรวมสุขภาพของระบบแบบ real-time ในหน้าเดียว
     */
}
```

## 6. Distributed Tracing ด้วย OpenTelemetry

ทบทวนปัญหาจากหัวข้อ 1: request หนึ่งผ่านหลาย service — **Distributed
Tracing** ติดตามเส้นทางทั้งหมดของ request นั้น

```
Trace ID: abc-123 (request เดียวกันทั้งหมด)
├── Span: API Gateway (5ms)
│   └── Span: OrderService.createOrder (120ms)
│       ├── Span: InventoryService.checkStock (40ms)  ← ทบทวน Part 91-93
│       └── Span: PaymentService.charge (70ms)
│           └── Span: PostgreSQL query (60ms)  ← เห็นได้ว่าช้าที่ database query นี้!
```

```xml
<dependency>
    <groupId>io.opentelemetry.instrumentation</groupId>
    <artifactId>opentelemetry-spring-boot-starter</artifactId>
</dependency>
```

```java
public class DistributedTracingExplanation {
    /*
     * OpenTelemetry สร้าง "Trace ID" เดียวตอน request เข้ามาครั้งแรก (ที่ API Gateway - ทบทวน Part 93)
     * แล้ว "ส่งต่อ" Trace ID นี้ผ่าน HTTP header ไปยังทุก service ที่เรียกต่อกันเป็นทอด ๆ
     *
     * แต่ละ service สร้าง "Span" ของตัวเอง (การทำงานย่อยหนึ่งช่วง) ผูกกับ Trace ID เดียวกัน
     * เมื่อดูใน tool อย่าง Jaeger/Zipkin จะเห็น "แผนภาพต้นไม้" ของ request ทั้งหมด
     * พร้อมเวลาที่ใช้ในแต่ละ span - เห็นชัดเจนทันทีว่า "ช้าที่ span ไหน" (เช่นตัวอย่างข้างบน
     * ที่เห็นว่า PostgreSQL query ใช้เวลา 60ms จาก PaymentService ทั้งหมด 70ms)
     */
}
```

## 7. Correlation ID: เชื่อมโยง Log ข้าม Service

Distributed Tracing (หัวข้อ 6) ตอบว่า "ช้าที่ไหน" แต่การเชื่อม **log ข้าม
service** เข้าด้วยกันต้องใช้ **Correlation ID** ที่แนบไปกับทุก log
statement

```java
import org.slf4j.MDC;
import jakarta.servlet.*;
import jakarta.servlet.http.*;
import java.io.IOException;
import java.util.UUID;

public class CorrelationIdFilter extends HttpFilter {
    @Override
    protected void doFilter(HttpServletRequest req, HttpServletResponse resp, FilterChain chain)
            throws IOException, ServletException {
        String correlationId = req.getHeader("X-Correlation-ID");
        if (correlationId == null) correlationId = UUID.randomUUID().toString(); // ทบทวน UUID จาก Part 28

        MDC.put("correlationId", correlationId); // ทบทวน MDC (Mapped Diagnostic Context) จาก Part 63
        try {
            resp.setHeader("X-Correlation-ID", correlationId); // ส่งต่อไปให้ service ถัดไปผ่าน header
            chain.doFilter(req, resp);
        } finally {
            MDC.clear(); // สำคัญมาก - ต้อง clear ทุกครั้งเพื่อไม่ให้ thread pool (ทบทวน Part 48) ใช้ค่าเก่าซ้ำ
        }
    }
}
```

```java
public class CorrelationIdLogFormatDemo {
    /*
     * Logback pattern: "%d{ISO8601} [%X{correlationId}] %-5level %logger - %msg%n"
     * ทำให้ทุก log statement มี correlationId ติดไปด้วยอัตโนมัติ (ไม่ต้องเขียนซ้ำทุกที่)
     *
     * ตัวอย่าง log จาก 3 service ที่ประมวลผล request เดียวกัน:
     *   [abc-123] INFO OrderService - Order created
     *   [abc-123] INFO InventoryService - Stock reserved
     *   [abc-123] ERROR PaymentService - Payment failed: card declined
     *
     * ค้นหา "abc-123" ใน centralized log (หัวข้อ 3) เจอ log ที่เกี่ยวข้องทั้งหมดจากทุก service ทันที
     */
}
```

## 8. Alerting: แจ้งเตือนก่อนผู้ใช้รู้ตัว

```yaml
# Prometheus Alert Rule
groups:
  - name: order-service-alerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_server_requests_seconds_count{status="500"}[5m]) > 0.05
        for: 2m
        annotations:
          summary: "Order Service error rate สูงกว่า 5% เป็นเวลา 2 นาที"
      - alert: HighLatencyP95
        expr: histogram_quantile(0.95, http_server_requests_seconds_bucket) > 2
        for: 5m
        annotations:
          summary: "P95 latency สูงกว่า 2 วินาที"
```

```java
public class AlertingPhilosophyDemo {
    /*
     * หลักการสำคัญ: Alert ควร "แจ้งเตือนก่อนที่ผู้ใช้จะร้องเรียน" ไม่ใช่หลังจากนั้น
     *
     * ถ้ารอให้ผู้ใช้ร้องเรียนก่อนถึงจะรู้ว่ามีปัญหา -> แสดงว่า monitoring ไม่เพียงพอ
     * Alert Rule ที่ดี (เช่นตัวอย่างข้างบน) ตรวจจับ pattern ที่บ่งบอกปัญหา (error rate สูง, latency สูง)
     * แล้วส่งแจ้งเตือนไปที่ทีม ops/on-call (ผ่าน Slack, PagerDuty, email) ทันทีที่เกิดขึ้น
     *
     * "for: 2m" สำคัญมาก - ป้องกัน alert fatigue จาก spike ชั่วขณะที่ไม่ใช่ปัญหาจริง
     * (ต้องเกิด pattern นี้ต่อเนื่อง 2 นาทีก่อนถึงจะแจ้งเตือนจริง)
     */
}
```

## 9. SLI, SLO, SLA: การวัดความน่าเชื่อถือของระบบ

```java
public class SliSloSlaDemo {
    /*
     * SLI (Service Level Indicator): "ตัวเลขที่วัดได้จริง" เช่น
     *   - Availability: % ของ request ที่สำเร็จ (ไม่ error)
     *   - Latency: P95/P99 response time
     *
     * SLO (Service Level Objective): "เป้าหมายภายใน" ที่ทีมตั้งไว้เอง
     *   เช่น "Availability ต้อง >= 99.9% ต่อเดือน" หรือ "P95 latency ต้อง < 500ms"
     *   ใช้เป็นเกณฑ์ตัดสินว่า "ระบบทำงานดีพอหรือยัง" และตั้ง Alert (หัวข้อ 8) ตาม SLO นี้
     *
     * SLA (Service Level Agreement): "สัญญาต่อลูกค้า" ที่มีผลทางกฎหมาย/ธุรกิจ
     *   เช่น "ถ้า Availability ต่ำกว่า 99.5% ต่อเดือน ลูกค้าได้รับเงินคืนส่วนหนึ่ง"
     *   SLA มักตั้งเกณฑ์ "หลวมกว่า" SLO ภายใน เพื่อให้มี "buffer" ก่อนผิดสัญญาจริง
     *
     * ความสัมพันธ์: SLI (วัดจริง) <-> SLO (เป้าหมายภายในทีม, เข้มกว่า) <-> SLA (สัญญาลูกค้า, หลวมกว่า SLO)
     */
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียน custom metric ด้วย Micrometer ที่นับจำนวนครั้งที่ Circuit
Breaker (ทบทวน Part 94) เปลี่ยนไปสถานะ OPEN

**เฉลย:**
```java
@Service
public class CircuitBreakerMetricsListener {
    private final MeterRegistry meterRegistry;

    public CircuitBreakerMetricsListener(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    public void onCircuitOpen(String serviceName) {
        meterRegistry.counter("circuit_breaker.opened", "service", serviceName).increment();
        // Prometheus/Grafana query metric นี้ได้ทันที เพื่อตั้ง alert ว่า circuit breaker เปิดบ่อยผิดปกติ
    }
}
```

**2)** อธิบายว่าทำไม Correlation ID จำเป็นแม้ระบบจะมี Distributed Tracing
อยู่แล้ว

**เฉลย**: **Distributed Tracing** (หัวข้อ 6) ดีมากในการแสดง**เส้นทางและ
เวลาที่ใช้**ของ request ผ่านแต่ละ service (เห็นว่า "ช้าที่ span ไหน")
แต่**ไม่ได้แสดงรายละเอียดของสิ่งที่เกิดขึ้นจริงในแต่ละขั้นตอน** (เช่น
ค่าพารามิเตอร์ที่ใช้, ข้อความ error ที่แน่นอน, business logic ที่ตัดสินใจ
ไปทางใดทางหนึ่ง) รายละเอียดเหล่านี้อยู่ใน **log statement** ของแต่ละ
service (ทบทวน Part 63) — ถ้าไม่มี **Correlation ID** (หัวข้อ 7) ที่แนบ
ไปกับทุก log statement การ**เชื่อมโยง log จากหลาย service ที่เกี่ยวข้อง
กับ request เดียวกัน**เข้าด้วยกันจะทำไม่ได้เลย (ต้องเดาจากเวลาที่ใกล้เคียง
กันซึ่งไม่แม่นยำเมื่อมี request จำนวนมากเกิดขึ้นพร้อมกัน) Correlation ID
ทำให้สามารถ**ค้นหา log ทั้งหมดที่เกี่ยวข้องกับ request หนึ่งจากทุก
service ได้ในคำสั่งค้นหาเดียว** ซึ่ง Distributed Tracing เพียงอย่างเดียว
ทำไม่ได้ — ทั้งสองเครื่องมือทำงาน**เสริมกัน** ไม่ใช่แทนกัน

**3)** ทีมหนึ่งตั้ง SLO ว่า "Availability ต้อง >= 99.9% ต่อเดือน" อธิบาย
ว่าทำไมค่านี้มักตั้งให้ "เข้มกว่า" SLA ที่สัญญากับลูกค้า (เช่น SLA
กำหนดไว้ที่ 99.5%)

**เฉลย**: SLO (หัวข้อ 9) เป็น**เป้าหมายภายในที่ทีม engineering ใช้ควบคุม
คุณภาพของตัวเอง** ในขณะที่ SLA เป็น**สัญญาที่มีผลทางธุรกิจ/กฎหมายกับ
ลูกค้า** การตั้ง SLO ให้**เข้มกว่า** SLA สร้าง **"buffer" หรือ "error
budget"** — ถ้าระบบเริ่มมีปัญหาและ Availability ลดลงมาที่ระดับ 99.9%
(ผ่าน SLO ภายใน) ทีมจะ**รู้ตัวและเริ่มแก้ไขทันที**ตั้งแต่ตอนที่ยัง**ห่าง
จากขีดจำกัดของ SLA (99.5%) พอสมควร** ทำให้มีเวลาแก้ไขปัญหาได้ก่อนที่จะ
กระทบ SLA จริงและต้องจ่ายค่าชดเชยให้ลูกค้า (ทบทวนแนวคิด Alert ในหัวข้อ 8
ที่ควรแจ้งเตือนก่อนปัญหาลุกลาม) ถ้าตั้ง SLO เท่ากับ SLA เป๊ะ ๆ ทีมจะรู้ตัว
ว่ามีปัญหา**พร้อมกับ**ตอนที่ผิดสัญญากับลูกค้าไปแล้ว ซึ่งสายเกินไปที่จะ
ป้องกันความเสียหายทางธุรกิจ

### สรุปเนื้อหา Part 99

- Observability จำเป็นมากสำหรับ microservices ที่ request ผ่านหลาย
  service กระจายอยู่หลายเครื่อง
- สามเสาหลัก: Logs (รายละเอียดเชิงลึก), Metrics (ภาพรวมตามเวลา), Traces
  (เส้นทางของ request ข้าม service)
- Structured Logging (JSON) + Centralized Aggregation (ELK/Loki) ทำให้
  ค้นหา log ข้าม service ได้ง่าย
- Micrometer + Prometheus เก็บ metrics ทั้งแบบอัตโนมัติจาก framework และ
  custom metric ทางธุรกิจ
- Grafana สร้าง dashboard และใช้ P95/P99 latency แทน average เพื่อสะท้อน
  experience ของผู้ใช้จริงมากกว่า
- Distributed Tracing (OpenTelemetry) แสดงเส้นทางและเวลาที่ใช้ของ request
  ข้ามหลาย service
- Correlation ID เชื่อมโยง log จากหลาย service ที่เกี่ยวข้องกับ request
  เดียวกัน ทำงานเสริมกับ Distributed Tracing
- Alerting ควรแจ้งเตือนก่อนผู้ใช้ร้องเรียน; SLI/SLO/SLA กำหนดมาตรฐานความ
  น่าเชื่อถือโดย SLO ควรเข้มกว่า SLA เพื่อมี buffer

**ต่อไป**: [Part 100 — System Design สำหรับระบบขนาดใหญ่](./part-100-system-design.md)
