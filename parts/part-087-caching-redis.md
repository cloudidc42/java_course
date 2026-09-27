# Part 87: Caching with Redis, Spring Cache

> ขั้นตอนที่ 861-870 ของหลักสูตร | ระดับ: สูงมาก

## สารบัญ

1. ทำไมต้อง Cache: ปัญหาที่ Caching แก้ไข
2. Redis คืออะไร ทำไมนิยมใช้เป็น Cache
3. Spring Cache Abstraction: `@Cacheable`, `@CacheEvict`, `@CachePut`
4. ตั้งค่า Redis เป็น Cache Provider
5. Cache Key Design และปัญหา Cache Key Collision
6. Cache Eviction Strategies: TTL, LRU
7. ปัญหา Cache: Stale Data และ Cache Invalidation
8. Cache-Aside Pattern vs Write-Through Pattern
9. ปัญหา Cache Stampede และ Distributed Lock
10. แบบฝึกหัดและสรุป

---

## 1. ทำไมต้อง Cache: ปัญหาที่ Caching แก้ไข

ทบทวนจาก Part 66, 84: การ query database **มี cost สูง** (network
round-trip, disk I/O, query execution) — ถ้า data เดิม**ถูกอ่านซ้ำ ๆ บ่อย
ๆ โดยไม่เปลี่ยนแปลง** (เช่น รายการสินค้าขายดี, ข้อมูล profile ผู้ใช้) การ
query database ทุกครั้งเป็นการ**สิ้นเปลือง resource โดยไม่จำเป็น**

**Caching** คือการ**เก็บผลลัพธ์ที่คำนวณ/query แล้วไว้ใน memory** เพื่อให้
request ถัดไปที่ต้องการข้อมูลเดิม**ไม่ต้อง query database ซ้ำ**

```
ไม่มี Cache:  Request 1 -> Database (100ms) -> Response
              Request 2 -> Database (100ms) -> Response  (query เดิมซ้ำ!)
              Request 3 -> Database (100ms) -> Response

มี Cache:     Request 1 -> Database (100ms) -> Cache -> Response
              Request 2 -> Cache (1ms) -> Response       (เร็วขึ้น 100 เท่า!)
              Request 3 -> Cache (1ms) -> Response
```

## 2. Redis คืออะไร ทำไมนิยมใช้เป็น Cache

**Redis** เป็น **in-memory data store** ที่เก็บข้อมูลแบบ key-value
(ทบทวนแนวคิด Map จาก Part 24) — ต่างจาก HashMap ธรรมดาในหัวข้อสำคัญ:

```java
public class WhyRedisOverHashMap {
    /*
     * HashMap ใน JVM:
     *   - อยู่ใน memory ของ instance เดียวเท่านั้น
     *   - ถ้ามี server 3 instance (load balanced) แต่ละตัวมี cache แยกกัน ไม่ sync กัน!
     *   - ถ้า restart server, cache หายทั้งหมด
     *
     * Redis:
     *   - เป็น server แยก ที่ทุก instance ของแอปเชื่อมต่อไปใช้งานร่วมกัน (shared cache)
     *   - รองรับ TTL (Time-To-Live) ในตัว - key หมดอายุอัตโนมัติ
     *   - รองรับ data structure หลากหลาย: String, List, Set, Hash, Sorted Set
     *   - เร็วมาก (sub-millisecond) เพราะเก็บข้อมูลใน memory ทั้งหมด
     */
}
```

**สำคัญมากสำหรับ microservices** (ปูทางสู่ Part 91): เมื่อมี server หลาย
instance รับ request พร้อมกัน ทุก instance ต้องเห็น cache**เดียวกัน**
Redis (external cache) แก้ปัญหานี้ที่ in-memory cache ของ JVM เดียวทำไม่ได้

## 3. Spring Cache Abstraction: `@Cacheable`, `@CacheEvict`, `@CachePut`

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

```java
import org.springframework.boot.autoconfigure.domain.EntityScan;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.context.annotation.Configuration;

@Configuration
@EnableCaching // เปิดใช้งาน Spring Cache Abstraction (ทบทวนแนวคิด @Enable* จาก Part 74, 82)
public class CacheConfig {}
```

```java
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.CachePut;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

@Service
public class ProductService {
    private final ProductRepository productRepository;

    public ProductService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    @Cacheable(value = "products", key = "#id") // ครั้งแรก query database, ครั้งต่อไปอ่านจาก cache
    public ProductDto getProductById(Long id) {
        System.out.println("Querying database for product " + id); // จะเห็น log นี้แค่ครั้งแรก
        Product product = productRepository.findById(id)
                .orElseThrow(() -> new ProductNotFoundException(id));
        return new ProductDto(product.getId(), product.getName(), product.getPrice());
    }

    @CachePut(value = "products", key = "#result.id") // อัปเดต cache ทุกครั้งที่แก้ไข product
    public ProductDto updateProduct(Long id, UpdateProductRequest request) {
        Product product = productRepository.findById(id)
                .orElseThrow(() -> new ProductNotFoundException(id));
        product.setName(request.name());
        product.setPrice(request.price());
        productRepository.save(product);
        return new ProductDto(product.getId(), product.getName(), product.getPrice());
    }

    @CacheEvict(value = "products", key = "#id") // ลบ cache เมื่อ product ถูกลบ - ป้องกัน stale data
    public void deleteProduct(Long id) {
        productRepository.deleteById(id);
    }
}
```

**สังเกต**: annotation เหล่านี้ทำงานผ่าน **AOP Proxy** (ทบทวน Part 74) —
Spring สร้าง proxy ที่**ดักการเรียกเมธอด**ก่อนรัน code จริง เพื่อเช็ค cache
ก่อน (คล้ายกับ `@Transactional` ที่เคยเรียนใน Part 65)

## 4. ตั้งค่า Redis เป็น Cache Provider

```properties
# application.properties
spring.data.redis.host=localhost
spring.data.redis.port=6379
spring.cache.type=redis
spring.cache.redis.time-to-live=600000
```

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext;
import java.time.Duration;

@Configuration
public class RedisCacheConfig {
    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(10)) // ทบทวนความสำคัญของ TTL ในหัวข้อ 6
                .serializeValuesWith(RedisSerializationContext.SerializationPair
                        .fromSerializer(new GenericJackson2JsonRedisSerializer()));
                // เก็บ object เป็น JSON ใน Redis (ทบทวน Jackson จาก Part 70) - อ่านง่ายเวลา debug

        return RedisCacheManager.builder(connectionFactory).cacheDefaults(config).build();
    }
}
```

## 5. Cache Key Design และปัญหา Cache Key Collision

```java
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

@Service
public class OrderSearchService {
    private final OrderRepository orderRepository;

    public OrderSearchService(OrderRepository orderRepository) { this.orderRepository = orderRepository; }

    // อันตราย: ถ้า key ไม่รวมพารามิเตอร์ทั้งหมด อาจได้ผลลัพธ์ผิดจาก cache!
    @Cacheable(value = "orderSearch", key = "#status + '_' + #minAmount")
    public java.util.List<OrderDto> searchOrders(String status, double minAmount) {
        // key ต้องรวมทุกพารามิเตอร์ที่มีผลต่อผลลัพธ์ - มิเช่นนั้น search("PENDING", 100)
        // และ search("PENDING", 200) จะชนกันที่ key เดียวกัน (ถ้าลืมรวม minAmount ใน key)
        return orderRepository.findByStatusAndAmountGreaterThan(status, minAmount).stream()
                .map(o -> new OrderDto(o.getId(), o.getStatus(), o.getAmount()))
                .toList();
    }
}
```

**หลักการสำคัญ**: **Cache Key ต้องระบุอินพุตที่มีผลต่อผลลัพธ์ให้ครบถ้วน** —
ถ้าลืมรวมพารามิเตอร์ใดไปในการสร้าง key จะเกิด **key collision** ทำให้
ผู้ใช้เห็นผลลัพธ์ของ request อื่นที่ไม่ตรงกับพารามิเตอร์ที่ตัวเองส่งมา (บั๊ก
ที่ตรวจจับยากมากเพราะเกิดเป็นบางครั้ง — ทบทวนความสำคัญของการทดสอบ edge
case จาก Part 58)

## 6. Cache Eviction Strategies: TTL, LRU

```java
public class CacheEvictionStrategies {
    /*
     * TTL (Time-To-Live): key หมดอายุอัตโนมัติหลังเวลาที่กำหนด (หัวข้อ 4 - entryTtl)
     *   เหมาะกับข้อมูลที่ "เก่าได้ในระดับหนึ่ง" เช่น ราคาสินค้า, สถิติรายวัน
     *
     * LRU (Least Recently Used): เมื่อ cache เต็ม (memory จำกัด) ลบ key ที่ "ไม่ถูกใช้นานที่สุด" ก่อน
     *   Redis config: maxmemory-policy allkeys-lru
     *   เหมาะเมื่อต้องการควบคุม memory usage ของ Redis ไม่ให้เกินขนาดที่กำหนด
     *
     * ทั้งสองแบบทำงานร่วมกันได้: TTL กำหนดว่า "เก่าแค่ไหนถึงต้องทิ้ง"
     * LRU กำหนดว่า "ถ้า memory เต็มก่อนหมดอายุ จะทิ้งตัวไหนก่อน"
     */
}
```

## 7. ปัญหา Cache: Stale Data และ Cache Invalidation

**ปัญหาที่ยากที่สุดของ Caching** (มีคำกล่าวในวงการที่ว่า "there are only
two hard things in Computer Science: cache invalidation and naming
things"): เมื่อข้อมูลใน database เปลี่ยน แต่ cache ยังเก็บ**ค่าเก่า**
(stale data) อยู่ ผู้ใช้จะเห็นข้อมูลที่ผิด

```java
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.stereotype.Service;

@Service
public class ProductPriceService {
    private final ProductRepository productRepository;

    public ProductPriceService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    // ผิด: ลืม evict cache เมื่อแก้ไขราคา -> ผู้ใช้ยังเห็นราคาเก่าจาก cache!
    public void updatePriceWrong(Long productId, double newPrice) {
        Product p = productRepository.findById(productId).orElseThrow();
        p.setPrice(newPrice);
        productRepository.save(p);
        // ไม่มีการ evict cache ที่ key "products::productId" - บั๊กร้ายแรง!
    }

    // ถูก: evict cache ทุกครั้งที่มีการเปลี่ยนแปลงข้อมูลที่เกี่ยวข้อง
    @CacheEvict(value = "products", key = "#productId")
    public void updatePriceCorrect(Long productId, double newPrice) {
        Product p = productRepository.findById(productId).orElseThrow();
        p.setPrice(newPrice);
        productRepository.save(p);
    }
}
```

**หลักการ**: **ทุกจุดที่แก้ไขข้อมูล ต้อง evict/update cache ที่เกี่ยวข้อง
เสมอ** — นี่คือเหตุผลที่การใช้ TTL แบบสั้น (แม้จะลดประสิทธิภาพ cache
บ้าง) มักปลอดภัยกว่าการพึ่งพา manual eviction ทั้งหมดในระบบที่ซับซ้อน

## 8. Cache-Aside Pattern vs Write-Through Pattern

```java
public class CachePatterns {
    /*
     * Cache-Aside (Lazy Loading) - ที่ใช้ใน @Cacheable ข้างบน:
     *   Read:  เช็ค cache ก่อน -> ถ้าไม่มี query database -> เก็บผลลง cache -> คืนค่า
     *   Write: เขียนลง database ตรง ๆ แล้ว evict/update cache
     *   ข้อดี: เรียบง่าย, cache เก็บแค่ข้อมูลที่ถูกอ่านจริง (ไม่เปลืองพื้นที่)
     *   ข้อเสีย: request แรกหลัง cache miss จะช้า (ต้อง query database)
     *
     * Write-Through:
     *   Write: เขียนลง cache และ database พร้อมกันในทุกครั้งที่มีการแก้ไข (atomic)
     *   ข้อดี: cache ไม่มี stale data เลย (sync กับ database เสมอ)
     *   ข้อเสีย: ทุก write ช้าลง (ต้องเขียน 2 ที่เสมอ) แม้ข้อมูลนั้นจะไม่ถูกอ่านบ่อยก็ตาม
     */
}
```

**คำแนะนำ**: **Cache-Aside** (ที่ Spring `@Cacheable` ใช้) เหมาะกับ
ส่วนใหญ่ในโลกจริงเพราะเรียบง่ายและมีประสิทธิภาพดีสำหรับ **read-heavy
workload** (อ่านบ่อยกว่าเขียนมาก) ซึ่งเป็นกรณีทั่วไปของเว็บแอปพลิเคชัน

## 9. ปัญหา Cache Stampede และ Distributed Lock

**Cache Stampede**: เมื่อ cache key ที่ถูกใช้บ่อยมาก**หมดอายุพร้อมกัน**
(TTL ตรงกัน) request จำนวนมากจะ**query database พร้อมกันในเวลาเดียวกัน**
ทำให้ database ล่ม

```java
import java.util.concurrent.locks.ReentrantLock;

public class CacheStampedePreventionDemo {
    private final ReentrantLock lock = new ReentrantLock(); // ทบทวน ReentrantLock จาก Part 47

    public ProductDto getProductWithLockProtection(Long id, CacheService cache, ProductRepository repo) {
        ProductDto cached = cache.get(id);
        if (cached != null) return cached;

        lock.lock(); // ให้แค่ 1 thread query database ตอน cache miss พร้อมกันหลาย thread
        try {
            cached = cache.get(id); // เช็คอีกครั้งหลังได้ lock (double-checked locking - ทบทวน Part 47)
            if (cached != null) return cached;

            ProductDto fresh = repo.findDtoById(id); // แค่ thread เดียวที่ query database จริง
            cache.put(id, fresh);
            return fresh;
        } finally {
            lock.unlock();
        }
    }
    // ในระบบ distributed (หลาย server instance) ต้องใช้ Redis distributed lock (SETNX)
    // แทน ReentrantLock ธรรมดา เพราะ ReentrantLock ทำงานแค่ใน JVM เดียว
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เพิ่ม `@Cacheable` ให้เมธอด `getUserProfile(Long userId)` โดยกำหนด
TTL เฉพาะ cache นี้เป็น 5 นาที (ต่างจาก default 10 นาทีของระบบ)

**เฉลย:**
```java
@Bean
public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
    RedisCacheConfiguration defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10));
    RedisCacheConfiguration userProfileConfig = defaultConfig.entryTtl(Duration.ofMinutes(5));

    return RedisCacheManager.builder(factory)
            .cacheDefaults(defaultConfig)
            .withCacheConfiguration("userProfiles", userProfileConfig) // override เฉพาะ cache นี้
            .build();
}
// จากนั้นใช้ @Cacheable(value = "userProfiles", key = "#userId")
```

**2)** อธิบายว่าทำไม HashMap ธรรมดาใน JVM ใช้เป็น cache ไม่ได้ผลดีใน
ระบบที่มี server หลาย instance

**เฉลย**: HashMap อยู่ใน **memory ของ JVM instance เดียว** — เมื่อระบบมี
server หลาย instance (เพื่อรองรับ load ที่มาก, ทบทวนแนวคิด horizontal
scaling ที่ปูทางสู่ Part 91) แต่ละ instance จะมี HashMap แยกกันเป็น
**อิสระ** ไม่ sync กัน ทำให้เกิดปัญหา: (1) ถ้า request สองครั้งจาก user
เดียวกันไปตกที่ instance ต่างกัน (load balancer สุ่มเลือก) instance ที่สอง
จะไม่เห็น cache ที่ instance แรกเก็บไว้ (cache miss ทุกครั้งแม้ query
ซ้ำ) (2) เมื่อข้อมูลถูกแก้ไขและต้อง evict cache การ evict จะเกิดที่
instance เดียวเท่านั้น ทำให้ instance อื่นยังมี stale data อยู่ Redis
แก้ปัญหานี้เพราะเป็น**cache ภายนอกที่ทุก instance เชื่อมต่อไปใช้ร่วมกัน**
(shared state) ทำให้ทุก instance เห็นข้อมูล cache เดียวกันเสมอ

**3)** อธิบายว่า Cache Stampede คืออะไร และวิธีป้องกันเบื้องต้น

**เฉลย**: Cache Stampede คือสถานการณ์ที่ cache key ที่ถูกอ่านบ่อยมาก
**หมดอายุ (TTL expire) ในเวลาเดียวกัน** ทำให้ request จำนวนมากที่เข้ามา
พร้อมกันในช่วงนั้น**ทุกตัวพบ cache miss พร้อมกัน** และ**ยิง query ไปที่
database พร้อมกันทั้งหมด** ซึ่งอาจทำให้ database ทำงานหนักเกินจนล่ม (ทบทวน
แนวคิดโหลดสูงพร้อมกันจาก Part 46) วิธีป้องกันเบื้องต้นคือใช้ **lock**
(หัวข้อ 9) เพื่อให้**เฉพาะ thread/request แรกเท่านั้น**ที่ query database
จริง ส่วน request อื่นที่มาพร้อมกันจะรอผลลัพธ์จาก thread แรกแทนที่จะยิง
query ซ้ำซ้อนกันทั้งหมด อีกวิธีคือการทำให้ TTL ของแต่ละ key มี **jitter**
(สุ่มเวลาหมดอายุเล็กน้อย) เพื่อไม่ให้ key จำนวนมากหมดอายุพร้อมกันเป๊ะ

### สรุปเนื้อหา Part 87

- Caching ลด load ของ database โดยเก็บผลลัพธ์ที่ query/คำนวณแล้วไว้ใน
  memory
- Redis เป็น external in-memory data store ที่ทุก server instance ใช้
  ร่วมกันได้ (ต่างจาก HashMap ใน JVM เดียว)
- Spring Cache Abstraction (`@Cacheable`/`@CachePut`/`@CacheEvict`) ทำงาน
  ผ่าน AOP Proxy คล้าย `@Transactional`
- Cache Key ต้องรวมพารามิเตอร์ที่มีผลต่อผลลัพธ์ให้ครบ ไม่งั้นเกิด key
  collision
- TTL และ LRU เป็นกลไกหลักในการจัดการ eviction ของ cache
- Cache invalidation (ทำให้ cache ไม่ stale) เป็นปัญหาที่ยากที่สุดของ
  caching — ทุกจุดที่เขียนข้อมูลต้อง evict cache ที่เกี่ยวข้องเสมอ
- Cache-Aside Pattern เหมาะกับ read-heavy workload ทั่วไป
- Cache Stampede แก้ได้ด้วย lock ป้องกันการ query database พร้อมกันจำนวน
  มากตอน cache miss

**ต่อไป**: [Part 88 — Messaging: RabbitMQ, Spring AMQP](./part-088-messaging-rabbitmq.md)
