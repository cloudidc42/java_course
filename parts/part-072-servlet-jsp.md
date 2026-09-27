# Part 72: Servlet และ JSP เบื้องต้น

> ขั้นตอนที่ 711-720 ของหลักสูตร | ระดับ: สูง

## สารบัญ

1. Servlet คืออะไร ทำไมต้องเรียน (แม้ Spring Boot จะซ่อนมันไว้)
2. Servlet Container
3. การเขียน Servlet แรก
4. Servlet Lifecycle
5. `HttpServletRequest` และ `HttpServletResponse`
6. Session Management ด้วย `HttpSession`
7. `web.xml` และ Annotation-based Configuration
8. Filter: การประมวลผลก่อน/หลัง Servlet
9. JSP (JavaServer Pages) เบื้องต้น
10. แบบฝึกหัดและสรุป

---

## 1. Servlet คืออะไร ทำไมต้องเรียน (แม้ Spring Boot จะซ่อนมันไว้)

**Servlet** เป็น API มาตรฐานของ Java สำหรับสร้าง**web application** — รับ
HTTP request แล้วสร้าง HTTP response กลับ (ทบทวน HTTP จาก Part 71) — เป็น
**พื้นฐานที่ Spring MVC (Part 77) และ Spring Boot (Part 75) สร้างขึ้นมาบน
นี้ทั้งหมด**

**เหตุผลที่ควรเข้าใจ Servlet แม้จะใช้ Spring**: เมื่อเจอปัญหาระดับลึก (เช่น
filter chain, session management ที่ผิดปกติ) ความเข้าใจ Servlet ช่วยให้
debug ได้ — Spring ไม่ได้ "แทนที่" Servlet แต่ "ห่อ" มันไว้ให้ใช้งานสะดวกขึ้น
(ทบทวนแนวคิด Facade Pattern จาก Part 55)

## 2. Servlet Container

**Servlet Container** (หรือ "Web Server" เช่น **Apache Tomcat**) คือ
โปรแกรมที่**จัดการ lifecycle ของ Servlet** และ**แปลง HTTP request/response
ดิบให้เป็น Java object** ที่ใช้งานง่าย

```
Client ──HTTP Request──> Servlet Container (Tomcat) ──> Servlet.service() ──> Response
                          (จัดการ socket, thread pool - ทบทวน Part 70, 48)
```

Spring Boot (Part 75) **ฝัง (embed) Tomcat ไว้ในตัวแอปพลิเคชันเอง** ทำให้
รัน `java -jar myapp.jar` แล้วมี web server ในตัวทันที ไม่ต้อง deploy ไปยัง
Tomcat แยกต่างหากแบบสมัยก่อน

## 3. การเขียน Servlet แรก

```xml
<dependency>
    <groupId>jakarta.servlet</groupId>
    <artifactId>jakarta.servlet-api</artifactId>
    <version>6.0.0</version>
    <scope>provided</scope> <!-- provided: Tomcat มีให้อยู่แล้วตอน runtime (ทบทวน scope จาก Part 61) -->
</dependency>
```

```java
import jakarta.servlet.ServletException;
import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.http.*;
import java.io.IOException;
import java.io.PrintWriter;

@WebServlet("/hello") // ทบทวน annotation จาก Part 52 - แมป URL path นี้เข้ากับ Servlet นี้
public class HelloServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        response.setContentType("text/html;charset=UTF-8");
        PrintWriter out = response.getWriter();
        out.println("<h1>สวัสดี Servlet!</h1>");
    }
}
```

เมื่อ deploy และเรียก `http://localhost:8080/hello` — Tomcat จะเรียก
`doGet()` ให้อัตโนมัติ

## 4. Servlet Lifecycle

```
1. init()     <- ทำงานครั้งเดียวตอน Servlet ถูกโหลดเข้า container (คล้าย constructor)
2. service()  <- ทำงานทุกครั้งที่มี request เข้ามา (เรียก doGet/doPost ตาม HTTP method)
3. destroy()  <- ทำงานครั้งเดียวตอน container จะปิด Servlet นี้ (cleanup resource)
```

```java
import jakarta.servlet.*;
import jakarta.servlet.http.*;

public class LifecycleServlet extends HttpServlet {
    @Override
    public void init() throws ServletException {
        System.out.println("Servlet ถูกโหลด - เตรียม resource (เช่น database connection pool)");
    }

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) {
        System.out.println("รับ request"); // ทำงานทุกครั้งที่มี request
    }

    @Override
    public void destroy() {
        System.out.println("Servlet ถูกปิด - คืน resource");
    }
}
```

**ข้อสำคัญ**: **Servlet เป็น Singleton โดย default** (ทบทวน Singleton
Pattern จาก Part 54) — มี instance เดียวรับ request หลายตัวพร้อมกันผ่าน
**หลาย thread** (ทบทวน Part 46-48) ดังนั้น**instance field ของ Servlet ต้อง
thread-safe เสมอ** (ทบทวนปัญหา race condition จาก Part 46-47) หรือหลีกเลี่ยง
การใช้ instance field ที่ mutable ไปเลย

## 5. `HttpServletRequest` และ `HttpServletResponse`

```java
import jakarta.servlet.http.*;
import java.io.IOException;

public class RequestResponseServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws IOException {
        // อ่านข้อมูลจาก Request
        String name = req.getParameter("name");          // query parameter: ?name=Alice
        String userAgent = req.getHeader("User-Agent");    // ทบทวน HTTP Headers จาก Part 71
        String method = req.getMethod();                    // GET, POST, ...
        String path = req.getRequestURI();

        // เขียน Response
        resp.setStatus(HttpServletResponse.SC_OK);          // 200 (ทบทวน status code จาก Part 71)
        resp.setContentType("application/json");
        resp.getWriter().write("{\"greeting\": \"สวัสดี " + name + "\"}");
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws IOException {
        // อ่าน body ของ POST request
        StringBuilder body = new StringBuilder();
        try (var reader = req.getReader()) {
            String line;
            while ((line = reader.readLine()) != null) {
                body.append(line);
            }
        }
        System.out.println("ได้รับ body: " + body);
        resp.setStatus(HttpServletResponse.SC_CREATED); // 201
    }
}
```

## 6. Session Management ด้วย `HttpSession`

ทบทวนแนวคิด Session จาก Part 71: Servlet API มี `HttpSession` ให้ใช้งานได้
ทันที (Servlet Container จัดการ session ID + cookie ให้อัตโนมัติ)

```java
import jakarta.servlet.http.*;
import java.io.IOException;

public class SessionServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws IOException {
        HttpSession session = req.getSession(); // สร้าง session ใหม่ถ้ายังไม่มี หรือดึงของเดิม

        Integer visitCount = (Integer) session.getAttribute("visitCount");
        if (visitCount == null) {
            visitCount = 0;
        }
        visitCount++;
        session.setAttribute("visitCount", visitCount); // เก็บข้อมูลไว้ใน session (ฝั่ง server)

        resp.getWriter().write("คุณเข้าชมหน้านี้ " + visitCount + " ครั้งแล้ว");
        // ครั้งถัดไปที่ browser เดียวกัน request เข้ามา (มี cookie session ID เดิม)
        // จะได้ visitCount ที่เพิ่มขึ้นต่อเนื่อง เพราะ session ยังจำสถานะไว้อยู่
    }
}
```

## 7. `web.xml` และ Annotation-based Configuration

ก่อน Servlet 3.0 ต้อง config ผ่าน `web.xml` (deployment descriptor):

```xml
<!-- web.xml (แบบเก่า) -->
<web-app>
    <servlet>
        <servlet-name>HelloServlet</servlet-name>
        <servlet-class>com.example.HelloServlet</servlet-class>
    </servlet>
    <servlet-mapping>
        <servlet-name>HelloServlet</servlet-name>
        <url-pattern>/hello</url-pattern>
    </servlet-mapping>
</web-app>
```

ตั้งแต่ Servlet 3.0 (Java EE 6) ใช้ **`@WebServlet`** annotation แทนได้
(ทบทวนตัวอย่างจากหัวข้อ 3) — กระชับกว่ามากและเป็นที่นิยมในโค้ดสมัยใหม่
(Spring Boot ไปไกลกว่านี้อีกขั้นด้วย auto-configuration — Part 75)

## 8. Filter: การประมวลผลก่อน/หลัง Servlet

**Filter** ทำงานคล้าย **Chain of Responsibility Pattern** (ทบทวนจาก Part
56) — ประมวลผล request**ก่อน**ถึง Servlet และ response**หลัง**จาก Servlet
เหมาะกับ cross-cutting concern เช่น logging, authentication check

```java
import jakarta.servlet.*;
import jakarta.servlet.annotation.WebFilter;
import jakarta.servlet.http.*;
import java.io.IOException;

@WebFilter("/*") // ใช้กับทุก URL path
public class LoggingFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {

        long start = System.currentTimeMillis();
        HttpServletRequest httpReq = (HttpServletRequest) request;
        System.out.println("Request เข้ามา: " + httpReq.getRequestURI());

        chain.doFilter(request, response); // ส่งต่อไปยัง filter ถัดไป (หรือ Servlet ถ้าเป็นตัวสุดท้าย)

        long duration = System.currentTimeMillis() - start;
        System.out.println("Request เสร็จสิ้นใน " + duration + " ms");
    }
}
```

**นี่คือหลักการเบื้องหลัง Spring's `@ControllerAdvice`, Interceptor, และ
Spring Security Filter Chain** (Part 82) — ทุกอย่างที่ Spring ทำใน layer
"cross-cutting" ล้วนสร้างขึ้นบนแนวคิด Filter นี้

## 9. JSP (JavaServer Pages) เบื้องต้น

**JSP** ผสม HTML กับ Java code ในไฟล์เดียว — คอมไพล์เป็น Servlet โดย
อัตโนมัติตอน request แรกที่เรียกใช้

```jsp
<%-- hello.jsp --%>
<html>
<body>
    <h1>สวัสดี <%= request.getParameter("name") %></h1>
    <%-- Scriptlet: เขียนโค้ด Java ได้ตรง ๆ ในหน้า HTML --%>
    <% for (int i = 1; i <= 5; i++) { %>
        <p>รอบที่ <%= i %></p>
    <% } %>
</body>
</html>
```

**ข้อควรรู้สำคัญ**: JSP **ไม่แนะนำให้ใช้ในโค้ดใหม่** เพราะการผสม
presentation (HTML) กับ business logic (Java) ในไฟล์เดียวกันขัดกับหลักการ
**Separation of Concerns** (ทบทวน SRP จาก Part 57) — ทำให้ทดสอบยาก
(ทบทวนจาก Part 58) และดูแลรักษายาก โค้ดสมัยใหม่ใช้ **Thymeleaf** (Part 77)
หรือแยก frontend/backend อย่างสมบูรณ์ผ่าน REST API (Part 78) แทน

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียน Servlet ที่รับ query parameter `a` และ `b` แล้วคืนค่าผลรวมเป็น
JSON

**เฉลย:**

```java
@WebServlet("/add")
public class AddServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws IOException {
        int a = Integer.parseInt(req.getParameter("a"));
        int b = Integer.parseInt(req.getParameter("b"));
        resp.setContentType("application/json");
        resp.getWriter().write("{\"result\": " + (a + b) + "}");
    }
}
```

**2)** เขียน Filter ที่บล็อก request ทั้งหมดที่ไม่มี header
`Authorization` (คืน `401`)

**เฉลย:**

```java
@WebFilter("/api/*")
public class AuthFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest httpReq = (HttpServletRequest) request;
        HttpServletResponse httpResp = (HttpServletResponse) response;

        if (httpReq.getHeader("Authorization") == null) {
            httpResp.setStatus(HttpServletResponse.SC_UNAUTHORIZED); // 401
            return; // ไม่เรียก chain.doFilter() -> หยุดไม่ให้ไปถึง Servlet
        }
        chain.doFilter(request, response);
    }
}
```

**3)** อธิบายว่าทำไม instance field ของ Servlet ต้อง thread-safe

**เฉลย**: Servlet Container สร้าง**instance ของ Servlet เพียงตัวเดียว**
(Singleton — ทบทวนจาก Part 54) แต่ใช้**หลาย thread จาก thread pool**
(ทบทวนจาก Part 48) รับ request หลายตัวพร้อมกันผ่าน instance เดียวกันนี้ —
ถ้า Servlet มี instance field ที่ mutable (เช่น counter, cache) และหลาย
thread แก้ไขพร้อมกันโดยไม่มีการป้องกัน (`synchronized`, `AtomicInteger` —
ทบทวนจาก Part 47, 50) จะเกิด**race condition**เหมือนตัวอย่างใน Part 46-47
ทำให้ข้อมูลเสียหายหรือผลลัพธ์ไม่ถูกต้อง — วิธีที่ปลอดภัยที่สุดคือหลีกเลี่ยง
instance field ที่ mutable ไปเลย ใช้ local variable ภายในเมธอด `doGet`/
`doPost` แทน (ซึ่งแต่ละ thread มี stack ของตัวเอง — ทบทวนจาก Part 46, 67)

### สรุปเนื้อหา Part 72

- Servlet รับ HTTP request สร้าง response เป็นพื้นฐานของ Spring MVC/Boot
- Servlet Container (Tomcat) จัดการ lifecycle: init() -> service() ->
  destroy()
- Servlet เป็น Singleton รับหลาย request ผ่านหลาย thread — instance field
  ต้อง thread-safe
- `HttpSession` เก็บสถานะข้าม request ผ่าน session ID + cookie (ทบทวน
  Part 71)
- Filter ประมวลผลก่อน/หลัง Servlet เหมาะกับ cross-cutting concern (logging,
  auth check) — เป็นพื้นฐานของ Spring Security Filter Chain
- JSP ผสม HTML+Java ในไฟล์เดียว ไม่แนะนำในโค้ดใหม่ (ใช้ Thymeleaf หรือ REST
  API แทน)

**ต่อไป**: [Part 73 — Introduction to Spring Framework](./part-073-spring-intro.md)
