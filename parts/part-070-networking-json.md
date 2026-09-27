# Part 70: Networking: Socket Programming, JSON ด้วย Jackson/Gson

> ขั้นตอนที่ 691-700 ของหลักสูตร | ระดับ: สูง (จบหมวด Java ขั้นสูงและ Tooling)

## สารบัญ

1. Networking พื้นฐาน: Socket คืออะไร
2. TCP Socket: Server และ Client
3. การจัดการหลาย Client พร้อมกัน
4. `java.net.http.HttpClient`: เรียก HTTP API สมัยใหม่
5. JSON คืออะไร (ทบทวนจาก Part 38)
6. Jackson: การแปลง Object เป็น JSON (Serialization)
7. Jackson: การแปลง JSON เป็น Object (Deserialization)
8. Gson: ทางเลือกที่เรียบง่ายกว่า
9. การจัดการ JSON ที่ซับซ้อน: Nested Object, List, Custom Serializer
10. แบบฝึกหัดและสรุปหมวด Java ขั้นสูงและ Tooling

---

## 1. Networking พื้นฐาน: Socket คืออะไร

**Socket** คือ**จุดปลายทาง (endpoint)** ของการสื่อสารระหว่างสองโปรแกรมผ่าน
เครือข่าย ระบุด้วย **IP address + Port number** — เป็นพื้นฐานที่ทุก network
protocol (HTTP, FTP, database connection ที่เรียนใน Part 64) สร้างขึ้นมาบน
นี้ทั้งหมด

## 2. TCP Socket: Server และ Client

```java
// Server: รอรับการเชื่อมต่อจาก client
import java.io.*;
import java.net.*;

public class SimpleServer {
    public static void main(String[] args) throws IOException {
        try (ServerSocket serverSocket = new ServerSocket(8080)) { // เปิด port 8080 รอรับ connection
            System.out.println("Server กำลังรอการเชื่อมต่อที่ port 8080...");

            try (Socket clientSocket = serverSocket.accept(); // บล็อกรอจนกว่า client จะเชื่อมต่อเข้ามา
                 BufferedReader in = new BufferedReader(new InputStreamReader(clientSocket.getInputStream()));
                 PrintWriter out = new PrintWriter(clientSocket.getOutputStream(), true)) {

                String message = in.readLine(); // อ่านข้อความจาก client (ทบทวน BufferedReader จาก Part 36)
                System.out.println("ได้รับ: " + message);
                out.println("Server ตอบกลับ: " + message.toUpperCase());
            }
        }
    }
}
```

```java
// Client: เชื่อมต่อไปยัง server
import java.io.*;
import java.net.*;

public class SimpleClient {
    public static void main(String[] args) throws IOException {
        try (Socket socket = new Socket("localhost", 8080); // เชื่อมต่อไปยัง server ที่ port 8080
             PrintWriter out = new PrintWriter(socket.getOutputStream(), true);
             BufferedReader in = new BufferedReader(new InputStreamReader(socket.getInputStream()))) {

            out.println("สวัสดี server!");
            String response = in.readLine();
            System.out.println("ได้รับการตอบกลับ: " + response);
        }
    }
}
```

## 3. การจัดการหลาย Client พร้อมกัน

Server แบบข้างบนรับได้แค่ **client เดียว** — สำหรับ server จริงที่ต้องรับ
หลาย client พร้อมกัน ใช้ **Thread ต่อ connection** (ทบทวนจาก Part 46, 48):

```java
import java.io.*;
import java.net.*;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class MultiClientServer {
    public static void main(String[] args) throws IOException {
        ExecutorService executor = Executors.newFixedThreadPool(10); // thread pool (ทบทวน Part 48)

        try (ServerSocket serverSocket = new ServerSocket(8080)) {
            System.out.println("Server พร้อมรับหลาย client");
            while (true) {
                Socket clientSocket = serverSocket.accept(); // รอรับ connection ใหม่
                executor.submit(() -> handleClient(clientSocket)); // มอบหมายให้ thread pool จัดการแยกกัน
            }
        }
    }

    static void handleClient(Socket clientSocket) {
        try (Socket socket = clientSocket;
             BufferedReader in = new BufferedReader(new InputStreamReader(socket.getInputStream()));
             PrintWriter out = new PrintWriter(socket.getOutputStream(), true)) {

            String message = in.readLine();
            out.println("ได้รับ: " + message);
        } catch (IOException e) {
            System.out.println("เกิดข้อผิดพลาด: " + e.getMessage());
        }
    }
}
```

**ข้อสังเกต**: นี่คือ**หลักการเบื้องหลัง web server** (Tomcat ที่ Spring Boot
ใช้ — Part 75) — แต่ในทางปฏิบัติจริงเราไม่เขียน socket server เองแบบนี้
สำหรับ web application เพราะมี framework จัดการ HTTP protocol ที่ซับซ้อน
(headers, encoding, ฯลฯ) ให้แล้ว

## 4. `java.net.http.HttpClient`: เรียก HTTP API สมัยใหม่

Java 11 เพิ่ม `HttpClient` ที่ทันสมัย (แทน `HttpURLConnection` แบบเก่าที่ใช้
ยาก) — รองรับ HTTP/2 และ async request ผ่าน `CompletableFuture` (ทบทวนจาก
Part 49):

```java
import java.net.URI;
import java.net.http.*;
import java.time.Duration;

public class HttpClientDemo {
    public static void main(String[] args) throws Exception {
        HttpClient client = HttpClient.newBuilder()
                .connectTimeout(Duration.ofSeconds(10))
                .build();

        // GET request
        HttpRequest getRequest = HttpRequest.newBuilder()
                .uri(URI.create("https://api.example.com/users/1"))
                .header("Accept", "application/json")
                .GET()
                .build();

        HttpResponse<String> response = client.send(getRequest, HttpResponse.BodyHandlers.ofString());
        System.out.println("Status: " + response.statusCode());
        System.out.println("Body: " + response.body());

        // POST request พร้อม JSON body
        String jsonBody = """
                {"name": "Alice", "age": 25}
                """; // ทบทวน Text Block จาก Part 9
        HttpRequest postRequest = HttpRequest.newBuilder()
                .uri(URI.create("https://api.example.com/users"))
                .header("Content-Type", "application/json")
                .POST(HttpRequest.BodyPublishers.ofString(jsonBody))
                .build();

        client.send(postRequest, HttpResponse.BodyHandlers.ofString());

        // Async request (ไม่บล็อก thread - ทบทวนแนวคิดจาก Part 49)
        client.sendAsync(getRequest, HttpResponse.BodyHandlers.ofString())
              .thenApply(HttpResponse::body)
              .thenAccept(System.out::println);
    }
}
```

## 5. JSON คืออะไร (ทบทวนจาก Part 38)

**JSON (JavaScript Object Notation)** เป็น text format ที่นิยมที่สุดสำหรับ
การแลกเปลี่ยนข้อมูลระหว่างระบบ (ทบทวนเหตุผลจาก Part 38 ว่าดีกว่า Java
Serialization อย่างไร):

```json
{
  "name": "Alice",
  "age": 25,
  "isActive": true,
  "address": {
    "city": "Bangkok",
    "zipCode": "10110"
  },
  "hobbies": ["reading", "coding"]
}
```

## 6. Jackson: การแปลง Object เป็น JSON (Serialization)

**Jackson** เป็น JSON library ที่นิยมที่สุดในโลก Java (เป็น default ของ
Spring Boot — Part 78) ใช้ **Reflection** (ทบทวนจาก Part 53) อ่าน field/
getter ของ object แล้วแปลงเป็น JSON โดยอัตโนมัติ

```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.16.0</version>
</dependency>
```

```java
import com.fasterxml.jackson.databind.ObjectMapper;

public record User(String name, int age, boolean isActive) { } // ทบทวน record จาก Part 51

public class JacksonSerializationDemo {
    public static void main(String[] args) throws Exception {
        ObjectMapper mapper = new ObjectMapper(); // จุดเริ่มต้นของทุกอย่างใน Jackson

        User user = new User("Alice", 25, true);
        String json = mapper.writeValueAsString(user); // Object -> JSON String
        System.out.println(json);
        // {"name":"Alice","age":25,"isActive":true}

        // Pretty print (จัดรูปแบบให้อ่านง่าย)
        String prettyJson = mapper.writerWithDefaultPrettyPrinter().writeValueAsString(user);
        System.out.println(prettyJson);
    }
}
```

## 7. Jackson: การแปลง JSON เป็น Object (Deserialization)

```java
import com.fasterxml.jackson.databind.ObjectMapper;

public class JacksonDeserializationDemo {
    public static void main(String[] args) throws Exception {
        ObjectMapper mapper = new ObjectMapper();

        String json = "{\"name\":\"Bob\",\"age\":30,\"isActive\":false}";
        User user = mapper.readValue(json, User.class); // JSON String -> Object

        System.out.println(user.name() + ", " + user.age()); // Bob, 30

        // แปลงเป็น List
        String jsonArray = "[{\"name\":\"A\",\"age\":1,\"isActive\":true}," +
                            "{\"name\":\"B\",\"age\":2,\"isActive\":false}]";
        java.util.List<User> users = mapper.readValue(jsonArray,
                mapper.getTypeFactory().constructCollectionType(java.util.List.class, User.class));
        System.out.println(users.size()); // 2
    }
}
```

**Jackson ทำงานได้ราบรื่นกับ record (Part 51)** เพราะ record มี canonical
constructor ที่ Jackson ใช้ในการสร้าง object ระหว่าง deserialize ได้โดย
อัตโนมัติ — สำหรับ class ธรรมดา Jackson ใช้ getter (สำหรับ serialize) และ
no-arg constructor + setter หรือ constructor annotation (สำหรับ
deserialize)

## 8. Gson: ทางเลือกที่เรียบง่ายกว่า

**Gson** (พัฒนาโดย Google) เป็นทางเลือกที่**API เรียบง่ายกว่า** Jackson
เล็กน้อย เหมาะกับโปรเจกต์ขนาดเล็กที่ไม่ต้องการ feature ซับซ้อน:

```xml
<dependency>
    <groupId>com.google.code.gson</groupId>
    <artifactId>gson</artifactId>
    <version>2.10.1</version>
</dependency>
```

```java
import com.google.gson.Gson;

public class GsonDemo {
    public static void main(String[] args) {
        Gson gson = new Gson();

        User user = new User("Charlie", 28, true);
        String json = gson.toJson(user); // Object -> JSON
        System.out.println(json);

        User parsed = gson.fromJson(json, User.class); // JSON -> Object
        System.out.println(parsed.name());
    }
}
```

**เปรียบเทียบ Jackson vs Gson**:

| ลักษณะ | Jackson | Gson |
|---|---|---|
| ความนิยม | สูงมาก (default ของ Spring) | นิยมพอสมควร แต่น้อยกว่า Jackson |
| Performance | เร็วกว่าเล็กน้อยในหลาย benchmark | เร็วพอสมควร |
| Feature | ครบครันมาก (custom (de)serializer, annotation หลากหลาย) | เรียบง่ายกว่า เหมาะกับงานพื้นฐาน |
| ใช้กับ Spring Boot | Default อยู่แล้ว (Part 78) | ต้องตั้งค่าเพิ่ม |

**หลักปฏิบัติ**: ใช้ **Jackson เป็นค่าเริ่มต้น** โดยเฉพาะถ้าใช้ Spring Boot
(เพราะเป็น default อยู่แล้ว — ทบทวนจาก Part 78)

## 9. การจัดการ JSON ที่ซับซ้อน: Nested Object, List, Custom Serializer

```java
import com.fasterxml.jackson.annotation.JsonIgnore;
import com.fasterxml.jackson.annotation.JsonProperty;
import java.util.List;

public class Address {
    public String city;
    public String zipCode;
}

public class Customer {
    @JsonProperty("full_name") // แปลงชื่อ field ระหว่าง Java (camelCase) และ JSON (snake_case)
    public String fullName;

    public Address address; // nested object - Jackson แปลงให้อัตโนมัติแบบ recursive

    public List<String> hobbies; // List ก็แปลงให้อัตโนมัติเป็น JSON array

    @JsonIgnore // field นี้จะไม่ถูกรวมใน JSON เลย (เช่น password, ข้อมูลภายใน)
    public String internalNotes;
}
```

```java
import com.fasterxml.jackson.databind.ObjectMapper;

public class ComplexJsonDemo {
    public static void main(String[] args) throws Exception {
        ObjectMapper mapper = new ObjectMapper();

        Customer customer = new Customer();
        customer.fullName = "Somchai Jaidee";
        customer.address = new Address();
        customer.address.city = "Bangkok";
        customer.address.zipCode = "10110";
        customer.hobbies = List.of("reading", "coding");
        customer.internalNotes = "ข้อมูลภายใน ไม่ควรเปิดเผย";

        String json = mapper.writeValueAsString(customer);
        System.out.println(json);
        // {"full_name":"Somchai Jaidee","address":{"city":"Bangkok","zipCode":"10110"},
        //  "hobbies":["reading","coding"]}
        // สังเกต: internalNotes ไม่ปรากฏใน JSON เลย เพราะ @JsonIgnore
    }
}
```

## 10. แบบฝึกหัดและสรุปหมวด Java ขั้นสูงและ Tooling

### แบบฝึกหัด

**1)** เขียนโปรแกรมที่ใช้ `HttpClient` เรียก GET request ไปยัง API สาธารณะ
แล้วใช้ Jackson แปลงผลลัพธ์ JSON เป็น record

**เฉลย (แนวคิด):**

```java
record Post(int id, String title, String body) { }

HttpClient client = HttpClient.newHttpClient();
HttpRequest request = HttpRequest.newBuilder()
        .uri(URI.create("https://jsonplaceholder.typicode.com/posts/1"))
        .build();
HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());

ObjectMapper mapper = new ObjectMapper();
Post post = mapper.readValue(response.body(), Post.class);
System.out.println(post.title());
```

**2)** เขียน server socket ที่รับตัวเลขจาก client แล้วส่งกลับผลลัพธ์ของการ
คำนวณ factorial (ทบทวนจาก Part 8, 29)

**เฉลย (แนวคิด)**: ดัดแปลงจาก `SimpleServer` ในหัวข้อ 2 โดยเปลี่ยน logic
เป็น `int n = Integer.parseInt(message); long result = factorial(n);
out.println(result);`

**3)** อธิบายว่าทำไม JSON เหมาะกับการสื่อสารระหว่าง microservices มากกว่า
Java Serialization ที่เรียนใน Part 38

**เฉลย**: (ทบทวนจาก Part 38 หัวข้อ 8) JSON เป็น text format ที่
**human-readable** และ**language-independent** — microservice ที่เขียนด้วย
Java สามารถสื่อสารกับ service ที่เขียนด้วย Python, Node.js, Go ได้อย่าง
ราบรื่นผ่าน JSON เพราะทุกภาษามี JSON library รองรับ ในขณะที่ Java
Serialization ผูกติดกับ JVM เท่านั้นและมีความเสี่ยงด้านความปลอดภัย (การ
deserialize ข้อมูลที่ไม่น่าเชื่อถือ) — ในสถาปัตยกรรม microservices (Part 91)
ที่ service ต่าง ๆ อาจเขียนด้วยภาษาต่างกันและสื่อสารผ่าน network เป็นหลัก
JSON (หรือ Protocol Buffers สำหรับกรณีที่ต้องการ performance สูงกว่า) จึง
เป็นตัวเลือกที่เหมาะสมกว่ามาก

### สรุปเนื้อหา Part 70 และหมวด Java ขั้นสูงและ Tooling (Part 56-70)

- Socket คือจุดปลายทางของการสื่อสารเครือข่าย ระบุด้วย IP + Port — เป็น
  พื้นฐานของทุก network protocol
- `java.net.http.HttpClient` (Java 11+) เรียก HTTP API ได้ทั้งแบบ sync และ
  async (ผ่าน CompletableFuture)
- JSON เป็น format มาตรฐานสำหรับแลกเปลี่ยนข้อมูล human-readable และ
  language-independent
- Jackson (default ของ Spring Boot) และ Gson แปลง Object <-> JSON โดยใช้
  Reflection ภายใน
- `@JsonProperty`, `@JsonIgnore` ควบคุมการแปลงชื่อ field และซ่อนข้อมูลที่
  ไม่ต้องการเปิดเผย

**จบหมวดที่ 4: Java ขั้นสูงและ Tooling (Part 56-70) อย่างสมบูรณ์!**
ครอบคลุม Design Patterns ครบ 3 กลุ่ม, SOLID, Testing (JUnit/Mockito/TDD),
Build Tools (Maven/Gradle), Logging, JDBC/Database, JVM Internals,
Performance Tuning, JPMS, และ Networking/JSON

**ต่อไป**: [Part 71 — HTTP และ Web Fundamentals สำหรับ Java Developer](./part-071-http-fundamentals.md)
(เริ่มหมวดที่ 5: การพัฒนาเว็บแอปพลิเคชัน)
