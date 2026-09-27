# Part 103: Capstone Project: ระบบ E-Commerce แบบเต็มรูปแบบ (ตอนที่ 1)

> ขั้นตอนที่ 1021-1030 ของหลักสูตร | ระดับ: ระดับมืออาชีพ/โลก

## สารบัญ

1. ภาพรวม Capstone Project: นำทุกอย่างมาประกอบกัน
2. Requirement และ Domain Model
3. โครงสร้างโปรเจกต์แบบ Multi-Module (Maven)
4. Domain Entity: Product, Customer, Order, OrderItem
5. Repository Layer ด้วย Spring Data JPA
6. Service Layer: Business Logic ของ Product Catalog
7. Service Layer: Business Logic ของ Order Processing
8. REST API Layer: Controller ที่สมบูรณ์
9. Exception Handling และ Validation แบบครบวงจร
10. แบบฝึกหัดและสรุป

---

## 1. ภาพรวม Capstone Project: นำทุกอย่างมาประกอบกัน

ทบทวนทั้งหลักสูตร 102 Part ที่ผ่านมา — ตอนนี้เราจะสร้าง **ระบบ
E-Commerce จริง** ที่ใช้แนวคิดเกือบทั้งหมดที่เรียนมา:

```java
public class CapstoneProjectOverview {
    /*
     * ระบบ "SimpleShop" ประกอบด้วย:
     *   - Product Catalog: จัดการสินค้า (Part 79 - Spring Data JPA)
     *   - Order Processing: สร้าง/จัดการคำสั่งซื้อ (Part 65, 91 - Transaction, Saga)
     *   - Customer Management: จัดการข้อมูลลูกค้า (Part 82, 83 - Security, JWT)
     *   - Caching: cache สินค้าที่ query บ่อย (Part 87)
     *   - Async Notification: แจ้งเตือนผ่าน Message Queue (Part 88)
     *   - Testing: unit + integration test ครบ (Part 58-60, 84)
     *   - Deployment: Docker + Kubernetes (Part 95-96)
     *
     * Part 103 (นี้): ออกแบบ domain model, repository, service, controller หลัก
     * Part 104: เพิ่ม security, testing, caching, async, containerize และ deploy
     */
}
```

## 2. Requirement และ Domain Model

ทบทวนกรอบคิด System Design จาก Part 100:

```java
public class SimpleShopRequirements {
    /*
     * Functional Requirements:
     *   - ดูรายการสินค้า, ค้นหาสินค้าตามชื่อ/หมวดหมู่
     *   - เพิ่มสินค้าลงตะกร้า, สร้างคำสั่งซื้อ
     *   - ตรวจสอบสต็อกก่อนยืนยันคำสั่งซื้อ (ต้องไม่ให้สต็อกติดลบ)
     *   - ดูประวัติคำสั่งซื้อของตัวเอง
     *
     * Non-Functional Requirements:
     *   - Consistency สูง (ธุรกรรมทางการเงิน - ทบทวน Part 100 หัวข้อ 10)
     *   - รองรับผู้ใช้พร้อมกันหลายคนสั่งซื้อสินค้าตัวเดียวกันโดยไม่เกิด overselling (race condition)
     */
}
```

```
Domain Model:
┌─────────────┐       ┌───────────┐       ┌────────────┐
│  Customer   │──────▶│   Order   │──────▶│ OrderItem  │
│ id, email   │  1  * │id,status  │  1  * │qty,price   │
└─────────────┘       └───────────┘       └──────┬─────┘
                                                   │ *  1
                                            ┌──────▼─────┐
                                            │  Product   │
                                            │id,name,    │
                                            │price,stock │
                                            └────────────┘
```

## 3. โครงสร้างโปรเจกต์แบบ Multi-Module (Maven)

ทบทวนจาก Part 61: ใช้ Maven multi-module จัดระเบียบโค้ดขนาดใหญ่

```xml
<!-- pom.xml (parent) -->
<project>
    <groupId>com.simpleshop</groupId>
    <artifactId>simpleshop-parent</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>
    <modules>
        <module>simpleshop-domain</module>
        <module>simpleshop-api</module>
    </modules>
    <properties>
        <java.version>21</java.version>
        <spring-boot.version>3.2.0</spring-boot.version>
    </properties>
</project>
```

```java
public class ProjectStructureDemo {
    /*
     * simpleshop-parent/
     * ├── simpleshop-domain/       (Entity, Repository - business core ที่ไม่ผูกกับ web layer)
     * │   └── src/main/java/com/simpleshop/domain/
     * │       ├── model/          (Product, Customer, Order, OrderItem)
     * │       └── repository/     (ProductRepository, OrderRepository)
     * └── simpleshop-api/          (Controller, Service, Configuration - web layer)
     *     └── src/main/java/com/simpleshop/api/
     *         ├── service/
     *         ├── controller/
     *         └── config/
     *
     * การแยก domain module ออกจาก api module ทำให้ business logic core ทดสอบได้ง่าย
     * โดยไม่ต้องพึ่งพา Spring Web (ทบทวนหลักการ Separation of Concerns จาก Part 57)
     */
}
```

## 4. Domain Entity: Product, Customer, Order, OrderItem

```java
package com.simpleshop.domain.model;

import jakarta.persistence.*;
import jakarta.validation.constraints.*;
import java.math.BigDecimal;

@Entity
@Table(name = "products")
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY) // ทบทวน Part 79
    private Long id;

    @NotBlank
    @Column(nullable = false)
    private String name;

    @NotNull
    @DecimalMin("0.0")
    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal price; // ใช้ BigDecimal สำหรับเงิน (ทบทวน Part 3 - ห้ามใช้ double กับเงินเด็ดขาด)

    @Min(0)
    @Column(nullable = false)
    private int stockQuantity;

    @Version // Optimistic Locking (ทบทวน Part 65) - ป้องกัน race condition ตอนตัดสต็อกพร้อมกัน (หัวข้อ 7)
    private Long version;

    protected Product() {} // JPA ต้องการ no-arg constructor (ทบทวน Part 79)

    public Product(String name, BigDecimal price, int stockQuantity) {
        this.name = name;
        this.price = price;
        this.stockQuantity = stockQuantity;
    }

    public void decreaseStock(int quantity) {
        if (stockQuantity < quantity) {
            throw new InsufficientStockException(id, quantity, stockQuantity);
            // ทบทวนหลักการ Encapsulation จาก Part 13 - business rule อยู่ใน entity เอง ไม่กระจายไปที่ Service
        }
        this.stockQuantity -= quantity;
    }

    public Long getId() { return id; }
    public String getName() { return name; }
    public BigDecimal getPrice() { return price; }
    public int getStockQuantity() { return stockQuantity; }
}
```

```java
package com.simpleshop.domain.model;

import jakarta.persistence.*;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "orders")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String customerEmail;

    @Enumerated(EnumType.STRING) // ทบทวน Part 18 - เก็บ enum เป็น String อ่านง่ายกว่า ordinal
    @Column(nullable = false)
    private OrderStatus status;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>(); // ทบทวน @OneToMany จาก Part 80

    protected Order() {}

    public Order(String customerEmail) {
        this.customerEmail = customerEmail;
        this.status = OrderStatus.PENDING;
    }

    public void addItem(Product product, int quantity) {
        items.add(new OrderItem(this, product, quantity, product.getPrice()));
        // เก็บราคา ณ ตอนสั่งซื้อไว้ใน OrderItem - ป้องกันปัญหาถ้าราคา product เปลี่ยนไปในอนาคต
        // (ทบทวนความสำคัญของ historical data integrity)
    }

    public java.math.BigDecimal getTotalAmount() {
        return items.stream() // ทบทวน Stream API จาก Part 41
                .map(OrderItem::getSubtotal)
                .reduce(java.math.BigDecimal.ZERO, java.math.BigDecimal::add);
    }

    public void markCompleted() { this.status = OrderStatus.COMPLETED; }
    public void cancel() { this.status = OrderStatus.CANCELLED; }

    public Long getId() { return id; }
    public String getCustomerEmail() { return customerEmail; }
    public OrderStatus getStatus() { return status; }
    public List<OrderItem> getItems() { return items; }
}

public enum OrderStatus { PENDING, COMPLETED, CANCELLED }
```

```java
package com.simpleshop.domain.model;

import jakarta.persistence.*;
import java.math.BigDecimal;

@Entity
@Table(name = "order_items")
public class OrderItem {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY) // ทบทวน N+1 problem จาก Part 80 - ใช้ LAZY เป็น default
    @JoinColumn(name = "order_id")
    private Order order;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id")
    private Product product;

    private int quantity;
    private BigDecimal priceAtPurchase;

    protected OrderItem() {}

    public OrderItem(Order order, Product product, int quantity, BigDecimal priceAtPurchase) {
        this.order = order;
        this.product = product;
        this.quantity = quantity;
        this.priceAtPurchase = priceAtPurchase;
    }

    public BigDecimal getSubtotal() { return priceAtPurchase.multiply(BigDecimal.valueOf(quantity)); }
    public Product getProduct() { return product; }
    public int getQuantity() { return quantity; }
}
```

## 5. Repository Layer ด้วย Spring Data JPA

```java
package com.simpleshop.domain.repository;

import com.simpleshop.domain.model.Product;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Lock;
import org.springframework.data.jpa.repository.Query;
import jakarta.persistence.LockModeType;
import java.util.List;
import java.util.Optional;

public interface ProductRepository extends JpaRepository<Product, Long> {
    // Query method (ทบทวน Part 79) - Spring Data JPA generate SQL ให้อัตโนมัติจากชื่อเมธอด
    List<Product> findByNameContainingIgnoreCase(String name);

    @Lock(LockModeType.PESSIMISTIC_WRITE) // Pessimistic Locking (ทบทวน Part 65) สำหรับกรณีที่ต้องล็อกแน่นอน
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdForUpdate(Long id);
    // ใช้ตอนต้อง "การันตี" ว่าไม่มี transaction อื่นแก้ไข product นี้พร้อมกันเด็ดขาด (หัวข้อ 7 จะเลือกวิธีที่เหมาะกว่า)
}
```

```java
package com.simpleshop.domain.repository;

import com.simpleshop.domain.model.Order;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;

public interface OrderRepository extends JpaRepository<Order, Long> {
    List<Order> findByCustomerEmailOrderByIdDesc(String customerEmail);
    // ทบทวน naming convention ของ Spring Data JPA query method จาก Part 79
}
```

## 6. Service Layer: Business Logic ของ Product Catalog

```java
package com.simpleshop.api.service;

import com.simpleshop.domain.model.Product;
import com.simpleshop.domain.repository.ProductRepository;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;
import java.util.List;

@Service
public class ProductCatalogService {
    private final ProductRepository productRepository;

    public ProductCatalogService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    @Cacheable(value = "products", key = "#id") // ทบทวน Part 87 - สินค้าถูกอ่านบ่อยกว่าที่ถูกแก้ไขมาก
    public ProductDto getProductById(Long id) {
        Product product = productRepository.findById(id)
                .orElseThrow(() -> new ProductNotFoundException(id));
        return toDto(product);
    }

    public List<ProductDto> searchByName(String keyword) {
        return productRepository.findByNameContainingIgnoreCase(keyword).stream()
                .map(this::toDto)
                .toList(); // ทบทวน Stream API จาก Part 41-42
    }

    private ProductDto toDto(Product product) {
        return new ProductDto(product.getId(), product.getName(), product.getPrice(), product.getStockQuantity());
    }

    public record ProductDto(Long id, String name, java.math.BigDecimal price, int stockQuantity) {}
}
```

## 7. Service Layer: Business Logic ของ Order Processing

นี่คือส่วนที่ซับซ้อนที่สุด — ต้องจัดการ **concurrency** (หลายคนสั่งสินค้า
ตัวเดียวกันพร้อมกัน) และ **transaction** (ทบทวน Part 65)

```java
package com.simpleshop.api.service;

import com.simpleshop.domain.model.*;
import com.simpleshop.domain.repository.OrderRepository;
import com.simpleshop.domain.repository.ProductRepository;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.dao.OptimisticLockingFailureException;
import org.springframework.retry.annotation.Retryable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.util.List;

@Service
public class OrderProcessingService {
    private final OrderRepository orderRepository;
    private final ProductRepository productRepository;

    public OrderProcessingService(OrderRepository orderRepository, ProductRepository productRepository) {
        this.orderRepository = orderRepository;
        this.productRepository = productRepository;
    }

    public record OrderItemRequest(Long productId, int quantity) {}
    public record CreateOrderRequest(String customerEmail, List<OrderItemRequest> items) {}

    @Transactional // ทบทวน Part 65 - การสร้าง order และตัดสต็อกต้องสำเร็จ/ล้มเหลวพร้อมกันทั้งหมด (atomicity)
    @Retryable(retryFor = OptimisticLockingFailureException.class, maxAttempts = 3)
    // ทบทวน Part 94 - ถ้าเกิด conflict จาก @Version (หัวข้อ 4) ให้ลองใหม่อัตโนมัติ (ปลอดภัยเพราะ operation นี้ idempotent
    // ในความหมายที่ว่า retry จะอ่านสต็อกล่าสุดใหม่ทุกครั้ง ไม่ใช่ใช้ค่าเก่าที่ stale)
    @CacheEvict(value = "products", allEntries = true) // สต็อกเปลี่ยน - ทบทวน Part 87 ต้อง evict cache เสมอ
    public Long createOrder(CreateOrderRequest request) {
        Order order = new Order(request.customerEmail());

        for (OrderItemRequest itemReq : request.items()) {
            Product product = productRepository.findById(itemReq.productId())
                    .orElseThrow(() -> new ProductNotFoundException(itemReq.productId()));

            product.decreaseStock(itemReq.quantity()); // business logic อยู่ใน entity เอง (ทบทวนหัวข้อ 4)
            productRepository.save(product); // @Version จะเช็คว่าไม่มีใครแก้ไข product นี้พร้อมกันระหว่างทาง

            order.addItem(product, itemReq.quantity());
        }

        orderRepository.save(order);
        return order.getId();
        // ถ้า decreaseStock() throw InsufficientStockException ที่ item ใดก็ตาม -> @Transactional
        // rollback ทุกอย่างที่ทำไปแล้วในเมธอดนี้ทั้งหมด (ทบทวน atomicity จาก Part 65) - ไม่มี order ค้างครึ่ง ๆ กลาง ๆ
    }

    @Transactional
    public void cancelOrder(Long orderId) {
        Order order = orderRepository.findById(orderId).orElseThrow(() -> new OrderNotFoundException(orderId));
        order.cancel();
        // ในระบบจริงต้องคืนสต็อกด้วย (compensating action - ทบทวน Saga Pattern จาก Part 91)
        for (OrderItem item : order.getItems()) {
            Product product = item.getProduct();
            productRepository.findById(product.getId()).ifPresent(p -> {
                // เพิ่มสต็อกกลับ (ไม่แสดง logic เต็มเพื่อความกระชับ - แนวคิดเดียวกับ decreaseStock ในหัวข้อ 4)
            });
        }
    }
}
```

**หลักการสำคัญ**: การใช้ **Optimistic Locking** (`@Version`) ร่วมกับ
**`@Retryable`** แก้ปัญหา **race condition** ได้อย่างสวยงาม — ถ้าสอง
คำสั่งซื้อพยายามตัดสต็อกสินค้าเดียวกันพร้อมกัน คำสั่งที่สองจะเจอ
`OptimisticLockingFailureException` (เพราะ version ไม่ตรงกันแล้ว) และถูก
retry อัตโนมัติด้วยข้อมูลสต็อกล่าสุด — ไม่มีทางเกิด **overselling**
(ขายสินค้าเกินสต็อกที่มีจริง) ได้เลย

## 8. REST API Layer: Controller ที่สมบูรณ์

```java
package com.simpleshop.api.controller;

import com.simpleshop.api.service.OrderProcessingService;
import com.simpleshop.api.service.OrderProcessingService.CreateOrderRequest;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import jakarta.validation.Valid;
import java.net.URI;

@RestController
@RequestMapping("/api/orders")
public class OrderController {
    private final OrderProcessingService orderProcessingService;

    public OrderController(OrderProcessingService orderProcessingService) {
        this.orderProcessingService = orderProcessingService;
    }

    @PostMapping
    public ResponseEntity<Void> createOrder(@Valid @RequestBody CreateOrderRequest request) {
        // ทบทวน Part 85 - POST คืน 201 Created พร้อม Location header
        Long orderId = orderProcessingService.createOrder(request);
        return ResponseEntity.created(URI.create("/api/orders/" + orderId)).build();
    }

    @PostMapping("/{id}/cancel")
    public ResponseEntity<Void> cancelOrder(@PathVariable Long id) {
        orderProcessingService.cancelOrder(id);
        return ResponseEntity.noContent().build();
    }
}
```

```java
package com.simpleshop.api.controller;

import com.simpleshop.api.service.ProductCatalogService;
import org.springframework.web.bind.annotation.*;
import java.util.List;

@RestController
@RequestMapping("/api/products")
public class ProductController {
    private final ProductCatalogService productCatalogService;

    public ProductController(ProductCatalogService productCatalogService) {
        this.productCatalogService = productCatalogService;
    }

    @GetMapping("/{id}")
    public ProductCatalogService.ProductDto getProduct(@PathVariable Long id) {
        return productCatalogService.getProductById(id);
    }

    @GetMapping("/search")
    public List<ProductCatalogService.ProductDto> search(@RequestParam String keyword) {
        return productCatalogService.searchByName(keyword);
    }
}
```

## 9. Exception Handling และ Validation แบบครบวงจร

ทบทวนจาก Part 78, 81: `@RestControllerAdvice` รวบรวม exception handling
ไว้ที่เดียว

```java
package com.simpleshop.api.exception;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import java.time.Instant;

@RestControllerAdvice
public class GlobalExceptionHandler {

    public record ErrorResponse(String code, String message, Instant timestamp) {}

    @ExceptionHandler(ProductNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleProductNotFound(ProductNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(new ErrorResponse("PRODUCT_NOT_FOUND", ex.getMessage(), Instant.now()));
    }

    @ExceptionHandler(InsufficientStockException.class)
    public ResponseEntity<ErrorResponse> handleInsufficientStock(InsufficientStockException ex) {
        return ResponseEntity.status(HttpStatus.CONFLICT) // 409 - conflict กับสถานะปัจจุบันของสต็อก
                .body(new ErrorResponse("INSUFFICIENT_STOCK", ex.getMessage(), Instant.now()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class) // ทบทวน Part 81 - Bean Validation ล้มเหลว
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        String message = ex.getBindingResult().getFieldErrors().stream()
                .map(err -> err.getField() + ": " + err.getDefaultMessage())
                .findFirst().orElse("Validation failed");
        return ResponseEntity.badRequest().body(new ErrorResponse("VALIDATION_ERROR", message, Instant.now()));
    }

    @ExceptionHandler(Exception.class) // ทบทวน Part 101 - ไม่เผย stack trace ให้ client เห็นเด็ดขาด
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
        return ResponseEntity.internalServerError()
                .body(new ErrorResponse("INTERNAL_ERROR", "An unexpected error occurred", Instant.now()));
    }
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เพิ่มเมธอด `getOrderHistory(String customerEmail)` ใน
`OrderProcessingService` ที่คืนรายการ order ของลูกค้าเรียงจากล่าสุดไปเก่า
สุด พร้อม DTO ที่เหมาะสม

**เฉลย:**
```java
public record OrderSummaryDto(Long id, String status, java.math.BigDecimal totalAmount) {}

public List<OrderSummaryDto> getOrderHistory(String customerEmail) {
    return orderRepository.findByCustomerEmailOrderByIdDesc(customerEmail).stream()
            .map(order -> new OrderSummaryDto(order.getId(), order.getStatus().name(), order.getTotalAmount()))
            .toList();
}
```

**2)** อธิบายว่าทำไม `OrderItem` ต้องเก็บ `priceAtPurchase` แยกจากการอ่าน
ราคาจาก `Product` โดยตรงทุกครั้งที่แสดงผล

**เฉลย**: ราคาสินค้า (`Product.price`) **เปลี่ยนแปลงได้ตามเวลา** (ร้านค้า
อาจปรับราคาขึ้น-ลงในอนาคต) — ถ้า `OrderItem` **อ่านราคาจาก `Product`
โดยตรงทุกครั้ง**ที่แสดงประวัติคำสั่งซื้อ ราคาที่แสดงในใบเสร็จเก่าจะ
**เปลี่ยนไปตามราคาปัจจุบันของสินค้า** ซึ่งผิดหลักการทางบัญชีและสร้างความ
สับสนให้ลูกค้าอย่างมาก (เช่น ลูกค้าซื้อสินค้าราคา 100 บาท แต่ต่อมาร้าน
ขึ้นราคาเป็น 150 บาท ใบเสร็จเก่าของลูกค้าควรยังแสดง 100 บาทเสมอ ไม่ใช่
150 บาทที่เป็นราคาปัจจุบัน) การเก็บ `priceAtPurchase` แยกไว้ใน
`OrderItem` ทำให้**ข้อมูลประวัติคำสั่งซื้อคงที่ตลอดไป (historical
integrity)** ไม่ว่าราคาสินค้าปัจจุบันจะเปลี่ยนไปอย่างไรก็ตาม

**3)** อธิบายว่าทำไมการใช้ Optimistic Locking (`@Version`) ร่วมกับ
`@Retryable` ในเมธอด `createOrder` ป้องกัน overselling ได้ โดยไม่ต้องใช้
Pessimistic Locking ที่ block transaction อื่นไว้รอ

**เฉลย**: **Optimistic Locking** (ทบทวน Part 65, หัวข้อ 4) ทำงานโดยให้
แต่ละ record มี **`version` number** — เมื่อ transaction หนึ่งจะบันทึก
การเปลี่ยนแปลง (เช่น ลด `stockQuantity`) มันจะ**เช็คว่า version ที่อ่าน
มาตอนแรกยังตรงกับ version ปัจจุบันใน database หรือไม่** ถ้ามี transaction
อื่นแก้ไข record นั้นไปก่อนแล้ว (version ถูก increment ไปแล้ว) การบันทึก
ครั้งที่สองจะ**ล้มเหลวด้วย `OptimisticLockingFailureException` ทันที**
แทนที่จะเขียนทับข้อมูลที่ไม่ถูกต้องทับไป — **`@Retryable`** จับ
exception นี้แล้ว**สั่งให้เมธอดทำงานใหม่ทั้งหมด**ซึ่งจะ**อ่านค่า
`stockQuantity` ล่าสุดจาก database ใหม่**ทำให้การตรวจสอบสต็อกครั้งที่สอง
ใช้ข้อมูลที่ถูกต้องเสมอ (ไม่มีทาง "มองไม่เห็น" การเปลี่ยนแปลงของ
transaction อื่นได้เลย) ข้อดีเทียบกับ **Pessimistic Locking**
(`SELECT ... FOR UPDATE`) คือ Optimistic Locking **ไม่ block transaction
อื่นให้ต้องรอ**เลยในช่วงเวลาปกติที่ไม่มี conflict เกิดขึ้นจริง (ซึ่งเป็น
กรณีส่วนใหญ่) — เพิ่ม throughput ของระบบได้มากกว่า Pessimistic Locking
ที่ต้อง serialize ทุก transaction ที่แข่งกันเข้าถึง record เดียวกัน แม้
ในกรณีที่ไม่จำเป็นก็ตาม (ทบทวนการเปรียบเทียบ Locking strategy จาก
Part 65)

### สรุปเนื้อหา Part 103

- Capstone Project "SimpleShop" นำแนวคิดจากทั้งหลักสูตรมาประกอบกันเป็น
  ระบบ E-Commerce จริง
- Multi-Module Maven project แยก domain module (business core) จาก api
  module (web layer) ตามหลัก Separation of Concerns
- Entity เก็บ business logic ของตัวเอง (เช่น `decreaseStock`) ไม่กระจาย
  ไปที่ Service ทั้งหมด (Encapsulation)
- `priceAtPurchase` ใน `OrderItem` รักษา historical data integrity ไม่ให้
  ประวัติคำสั่งซื้อเปลี่ยนตามราคาปัจจุบัน
- Optimistic Locking (`@Version`) + `@Retryable` ป้องกัน overselling
  จาก race condition โดยไม่ block transaction อื่นเหมือน Pessimistic
  Locking
- `@Transactional` การันตี atomicity ของการสร้าง order และตัดสต็อกพร้อมกัน
- `@RestControllerAdvice` รวบรวม exception handling ที่ครอบคลุมทุกกรณี
  (not found, business rule violation, validation, unexpected error)

**ต่อไป**: [Part 104 — Capstone Project: ระบบ E-Commerce แบบเต็มรูปแบบ (ตอนที่ 2)](./part-104-capstone-ecommerce-part2.md)
