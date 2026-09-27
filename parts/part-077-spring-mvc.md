# Part 77: Spring MVC: Controller, Model, View, Thymeleaf

> ขั้นตอนที่ 761-770 ของหลักสูตร | ระดับ: สูงมาก

## สารบัญ

1. MVC Pattern คืออะไร
2. `DispatcherServlet`: หัวใจของ Spring MVC
3. การเขียน Controller แรก
4. `@RequestMapping` และ Shortcut Annotations
5. รับพารามิเตอร์: `@PathVariable`, `@RequestParam`, `@RequestBody`
6. Model และการส่งข้อมูลไปยัง View
7. Thymeleaf: Template Engine สำหรับ Server-side Rendering
8. Form Handling
9. Redirect และ Forward
10. แบบฝึกหัดและสรุป

---

## 1. MVC Pattern คืออะไร

**MVC (Model-View-Controller)** เป็น pattern สถาปัตยกรรมที่แยก 3 ส่วน:

- **Model**: ข้อมูลและ business logic (ทบทวน Entity/Service จาก Part 65)
- **View**: การแสดงผล (HTML ที่ผู้ใช้เห็น)
- **Controller**: รับ request, ประสานงาน Model กับ View (ทบทวนแนวคิด
  Facade Pattern จาก Part 55 — Controller ทำหน้าที่คล้าย facade ระหว่าง
  HTTP layer กับ business logic)

```
Client Request
     │
     ▼
Controller (รับ request, เรียก Service) ──> Model (Business Logic + Data)
     │                                            │
     ▼                                            │
View (Thymeleaf template) <───── ส่งข้อมูลกลับมา ───┘
     │
     ▼
HTML Response กลับไปยัง Client
```

## 2. `DispatcherServlet`: หัวใจของ Spring MVC

**`DispatcherServlet`** เป็น**Servlet ตัวเดียว**ที่รับทุก HTTP request เข้า
มา (ทบทวน Servlet จาก Part 72) แล้ว**ค้นหา Controller ที่เหมาะสม**และส่งต่อ
request ไปให้ — Spring Boot ตั้งค่านี้ให้อัตโนมัติทั้งหมด (ทบทวน
auto-configuration จาก Part 75) ไม่ต้อง config `web.xml` เองแบบเก่า

```
Client Request ──> DispatcherServlet ──> ค้นหา Controller ที่ตรงกับ URL
                                              │
                                              ▼
                                       เรียก method ของ Controller
                                              │
                                              ▼
                            ได้ View name หรือ JSON กลับมา ──> ส่ง Response
```

## 3. การเขียน Controller แรก

```java
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;

@Controller // ทบทวนจาก Part 73 - เหมือน @Component แต่สื่อความหมายว่าเป็น web layer
public class HomeController {

    @GetMapping("/") // แมป HTTP GET ที่ path "/" เข้ากับเมธอดนี้ (ทบทวน HTTP Method จาก Part 71)
    public String home(Model model) { // Model: ส่งข้อมูลไปยัง View
        model.addAttribute("message", "สวัสดีจาก Spring MVC!");
        return "home"; // ชื่อ View (Spring จะไปหาไฟล์ templates/home.html - หัวข้อ 7)
    }
}
```

```html
<!-- src/main/resources/templates/home.html -->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<body>
    <h1 th:text="${message}">ข้อความ default</h1>
</body>
</html>
```

## 4. `@RequestMapping` และ Shortcut Annotations

**`@RequestMapping`** เป็น annotation พื้นฐาน มี **shortcut** สำหรับ HTTP
method ต่าง ๆ (ทบทวน HTTP Methods จาก Part 71):

```java
import org.springframework.web.bind.annotation.*;

@Controller
@RequestMapping("/products") // path prefix สำหรับทุกเมธอดใน controller นี้
public class ProductController {

    @GetMapping         // เทียบเท่า @RequestMapping(method = RequestMethod.GET)
    public String listProducts() { return "products/list"; }

    @GetMapping("/{id}")  // -> GET /products/{id}
    public String viewProduct() { return "products/detail"; }

    @PostMapping           // -> POST /products
    public String createProduct() { return "redirect:/products"; }

    @PutMapping("/{id}")     // -> PUT /products/{id}
    public String updateProduct() { return "redirect:/products"; }

    @DeleteMapping("/{id}")   // -> DELETE /products/{id}
    public String deleteProduct() { return "redirect:/products"; }
}
```

## 5. รับพารามิเตอร์: `@PathVariable`, `@RequestParam`, `@RequestBody`

```java
import org.springframework.web.bind.annotation.*;

@Controller
@RequestMapping("/products")
public class ProductParamController {

    // @PathVariable: ดึงค่าจากส่วนของ URL path (/products/5 -> id=5)
    @GetMapping("/{id}")
    public String viewProduct(@PathVariable Long id, org.springframework.ui.Model model) {
        model.addAttribute("productId", id);
        return "products/detail";
    }

    // @RequestParam: ดึงค่าจาก query parameter (?keyword=laptop&page=1)
    @GetMapping("/search")
    public String search(@RequestParam String keyword,
                          @RequestParam(defaultValue = "0") int page, // ค่า default ถ้าไม่ระบุ
                          @RequestParam(required = false) String category) { // optional parameter
        System.out.println("ค้นหา: " + keyword + ", หน้า: " + page);
        return "products/search-results";
    }

    // @RequestBody: ดึงค่าจาก JSON body (ทบทวน Jackson จาก Part 70)
    @PostMapping
    @ResponseBody // บอกว่า return value คือ response body ตรง ๆ ไม่ใช่ view name (ปูทางสู่ REST API - Part 78)
    public String createProduct(@RequestBody ProductRequest request) {
        System.out.println("สร้างสินค้า: " + request.name());
        return "สร้างสำเร็จ";
    }

    record ProductRequest(String name, double price) {}
}
```

## 6. Model และการส่งข้อมูลไปยัง View

```java
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import java.util.List;

@Controller
public class ProductListController {

    @GetMapping("/products")
    public String listProducts(Model model) {
        record Product(String name, double price) {} // ทบทวน record จาก Part 51

        List<Product> products = List.of(
            new Product("Laptop", 25000),
            new Product("Mouse", 500)
        );

        model.addAttribute("products", products);      // ส่ง List ไปยัง View
        model.addAttribute("totalCount", products.size());
        model.addAttribute("pageTitle", "รายการสินค้า");

        return "products/list"; // Spring หา templates/products/list.html
    }
}
```

```html
<!-- templates/products/list.html -->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title th:text="${pageTitle}">รายการสินค้า</title>
</head>
<body>
    <h1 th:text="${pageTitle}">Title</h1>
    <p>จำนวนสินค้า: <span th:text="${totalCount}">0</span></p>

    <ul>
        <li th:each="product : ${products}"> <!-- th:each = วน loop (ทบทวนแนวคิดจาก Part 6, 41) -->
            <span th:text="${product.name}">ชื่อสินค้า</span> -
            <span th:text="${product.price}">0</span> บาท
        </li>
    </ul>
</body>
</html>
```

## 7. Thymeleaf: Template Engine สำหรับ Server-side Rendering

**Thymeleaf** เป็น template engine ที่ Spring Boot แนะนำ (แทน JSP —
ทบทวนเหตุผลจาก Part 72) — ทำงานโดย**เติมข้อมูลลงใน HTML ฝั่ง server ก่อนส่ง
กลับไปยัง browser** (Server-side Rendering)

```html
<!-- ตัวอย่าง syntax ที่ใช้บ่อย -->
<p th:text="${user.name}">Placeholder</p>                <!-- แสดงค่าตัวแปร -->
<a th:href="@{/products/{id}(id=${product.id})}">ดูรายละเอียด</a> <!-- สร้าง URL แบบ dynamic -->
<div th:if="${products.size() > 0}">มีสินค้า</div>        <!-- เงื่อนไข (ทบทวน if จาก Part 5) -->
<div th:unless="${products.isEmpty()}">มีสินค้า</div>       <!-- ตรงข้ามกับ th:if -->
<select th:field="*{category}">                              <!-- ผูกกับ form object (หัวข้อ 8) -->
    <option th:each="c : ${categories}" th:value="${c}" th:text="${c}"></option>
</select>
```

**ข้อดีของ Thymeleaf เทียบกับ JSP**: ไฟล์ Thymeleaf **เป็น HTML ที่ถูกต้อง
สมบูรณ์** (สามารถเปิดดูใน browser ตรง ๆ ได้แม้ไม่ผ่าน server — เห็น
placeholder text แทนข้อมูลจริง) ต่างจาก JSP ที่มี syntax แปลกปลอมปนอยู่
(scriptlet `<% %>`) ทำให้นักออกแบบ (designer) ที่ไม่รู้ Java ทำงานร่วมกับ
นักพัฒนาได้ง่ายกว่า

## 8. Form Handling

```java
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.*;

@Controller
@RequestMapping("/register")
public class RegistrationController {

    record RegistrationForm(String username, String email) {} // แทน form data (ทบทวน record จาก Part 51)

    @GetMapping
    public String showForm(Model model) {
        model.addAttribute("form", new RegistrationForm("", "")); // object เปล่าให้ form ผูกไว้
        return "register";
    }

    @PostMapping
    public String submitForm(@ModelAttribute RegistrationForm form) { // ผูกข้อมูลจาก form เข้า object
        System.out.println("ลงทะเบียน: " + form.username() + ", " + form.email());
        return "redirect:/register/success"; // ทบทวนแนวคิด redirect ในหัวข้อ 9
    }

    @GetMapping("/success")
    public String success() {
        return "register-success";
    }
}
```

```html
<!-- templates/register.html -->
<form th:action="@{/register}" th:object="${form}" method="post">
    <input type="text" th:field="*{username}" placeholder="ชื่อผู้ใช้"/>
    <input type="email" th:field="*{email}" placeholder="อีเมล"/>
    <button type="submit">ลงทะเบียน</button>
</form>
```

## 9. Redirect และ Forward

```java
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.PostMapping;

@Controller
public class RedirectDemoController {

    @PostMapping("/submit-order")
    public String submitOrder() {
        // ... บันทึกออร์เดอร์ ...

        // Redirect: ส่ง HTTP 302 กลับไปให้ browser -> browser ทำ request ใหม่ไปยัง URL ที่ระบุ
        // (URL bar ของ browser เปลี่ยน, refresh หน้าไม่ submit form ซ้ำ - ป้องกัน double-submit)
        return "redirect:/orders/confirmation";
    }

    @PostMapping("/preview-order")
    public String previewOrder() {
        // Forward: server ส่งต่อ request ไปยัง view อื่นภายในเซิร์ฟเวอร์เดียวกัน
        // (URL bar ของ browser ไม่เปลี่ยน, เป็น request เดียวกันตลอด)
        return "forward:/orders/preview-view";
    }
}
```

**หลักการเลือก**: ใช้ **redirect หลังจาก POST ที่แก้ไขข้อมูลสำเร็จ**เสมอ
(เรียกว่า pattern **"POST-Redirect-GET"**) เพื่อป้องกันปัญหา**double-submit**
ที่เกิดจากผู้ใช้กด refresh หน้าเว็บหลัง POST (browser จะถาม "ต้องการส่งข้อมูล
ซ้ำหรือไม่" ถ้าไม่ redirect) — ทบทวนแนวคิด idempotent จาก Part 71

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียน Controller ที่รับ `@PathVariable` เป็นชื่อเมือง และ
`@RequestParam` เป็นวันที่ (optional) แล้วแสดงผลผ่าน Thymeleaf template

**เฉลย:**

```java
@Controller
@RequestMapping("/weather")
public class WeatherController {
    @GetMapping("/{city}")
    public String showWeather(@PathVariable String city,
                               @RequestParam(required = false) String date,
                               Model model) {
        model.addAttribute("city", city);
        model.addAttribute("date", date != null ? date : "วันนี้");
        return "weather";
    }
}
```

**2)** เขียน form ที่รับข้อมูล login (username, password) แล้ว redirect ไป
หน้า dashboard หลังจาก submit

**เฉลย:**

```java
@Controller
public class LoginController {
    record LoginForm(String username, String password) {}

    @GetMapping("/login")
    public String showLoginForm(Model model) {
        model.addAttribute("form", new LoginForm("", ""));
        return "login";
    }

    @PostMapping("/login")
    public String processLogin(@ModelAttribute LoginForm form) {
        // ... ตรวจสอบ credential (Part 82 จะสอนวิธีที่ถูกต้องด้วย Spring Security) ...
        return "redirect:/dashboard";
    }
}
```

**3)** อธิบายว่าทำไม pattern "POST-Redirect-GET" ป้องกันปัญหา double-submit
ได้

**เฉลย**: ถ้า controller คืนค่า view ตรงจาก POST request (ไม่ redirect)
browser จะจำว่า "หน้านี้มาจากการ POST" — ถ้าผู้ใช้กด refresh (F5) browser
จะ**ส่ง POST request เดิมซ้ำอีกครั้ง** (อาจสร้างออร์เดอร์ซ้ำ ทบทวนแนวคิด
idempotent จาก Part 71 — POST ไม่ idempotent) ในขณะที่ pattern
POST-Redirect-GET ให้ controller ตอบกลับด้วย **HTTP 302 redirect** ไปยัง
URL ใหม่หลังจาก POST สำเร็จ — browser จะทำ **GET request** ไปยัง URL ใหม่
นั้นทันที (ตาม HTTP spec) ทำให้ URL bar และ "หน้าล่าสุดที่ browser จำไว้"
กลายเป็น GET request แทน ถ้าผู้ใช้กด refresh หลังจากนั้น จะเป็นการทำ GET
ซ้ำ (idempotent, ปลอดภัย) ไม่ใช่ POST ซ้ำอีกต่อไป

### สรุปเนื้อหา Part 77

- MVC แยก Model (ข้อมูล/logic), View (การแสดงผล), Controller (ประสานงาน)
- `DispatcherServlet` เป็น entry point เดียวที่ส่ง request ไปยัง Controller
  ที่ถูกต้อง
- `@GetMapping`/`@PostMapping`/ฯลฯ เป็น shortcut ของ `@RequestMapping`
- `@PathVariable` ดึงจาก URL path, `@RequestParam` ดึงจาก query string,
  `@RequestBody` ดึงจาก JSON body
- Thymeleaf render HTML ฝั่ง server ด้วย syntax ที่เป็น valid HTML (ต่างจาก
  JSP)
- ใช้ pattern POST-Redirect-GET เสมอหลัง POST ที่แก้ไขข้อมูล ป้องกัน
  double-submit

**ต่อไป**: [Part 78 — Building REST API ด้วย Spring Boot](./part-078-rest-api.md)
