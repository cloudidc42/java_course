# Part 71: HTTP และ Web Fundamentals สำหรับ Java Developer

> ขั้นตอนที่ 701-710 ของหลักสูตร | ระดับ: สูง (เริ่มหมวดการพัฒนาเว็บแอปพลิเคชัน)

## สารบัญ

1. HTTP คืออะไร: Request-Response Model
2. HTTP Methods
3. HTTP Status Codes
4. HTTP Headers
5. HTTP Request/Response Anatomy แบบละเอียด
6. Statelessness ของ HTTP และ Session
7. Cookies
8. HTTPS และ TLS
9. RESTful Architecture เบื้องต้น (ปูทางสู่ Part 78, 85)
10. แบบฝึกหัดและสรุป

---

## 1. HTTP คืออะไร: Request-Response Model

**HTTP (HyperText Transfer Protocol)** เป็น protocol มาตรฐานสำหรับสื่อสาร
บนเว็บ — ทำงานบน**TCP Socket** (ทบทวนจาก Part 70) ตามรูปแบบ
**Request-Response**: client ส่ง request, server ประมวลผลแล้วส่ง response
กลับมา

```
Client (Browser/App)                    Server
      │                                    │
      │ ────── HTTP Request ─────────────>  │  (GET /users/1 HTTP/1.1)
      │                                    │  ประมวลผล...
      │ <───── HTTP Response ─────────────  │  (200 OK + ข้อมูล JSON)
```

ทุกอย่างที่เราเรียนใน Part 70 (`HttpClient`) และจะเรียนใน Spring Boot
(Part 73 เป็นต้นไป) สร้างขึ้นบนความเข้าใจพื้นฐานของ HTTP นี้ทั้งหมด

## 2. HTTP Methods

| Method | ใช้ทำอะไร | มี Body? | Idempotent? |
|---|---|---|---|
| `GET` | ดึงข้อมูล | ไม่มี | ✅ (เรียกซ้ำได้ผลเหมือนกัน) |
| `POST` | สร้างข้อมูลใหม่ | มี | ❌ (เรียกซ้ำอาจสร้างซ้ำ) |
| `PUT` | แทนที่ข้อมูลทั้งหมด | มี | ✅ |
| `PATCH` | แก้ไขข้อมูลบางส่วน | มี | ❌ (ขึ้นกับ implementation) |
| `DELETE` | ลบข้อมูล | ไม่มี (ปกติ) | ✅ |
| `HEAD` | เหมือน GET แต่ไม่มี body ในผลลัพธ์ (เอาแค่ headers) | ไม่มี | ✅ |
| `OPTIONS` | ถามว่า method ไหนใช้ได้บ้าง (ใช้กับ CORS) | ไม่มี | ✅ |

**Idempotent** หมายถึง**เรียกซ้ำหลายครั้งได้ผลลัพธ์เหมือนเรียกครั้งเดียว**
(ทบทวนแนวคิดคล้าย pure function จาก Part 40) — `DELETE /users/1` เรียกซ้ำ
สิบครั้ง ผลลัพธ์คือ user นั้นถูกลบแล้ว (ไม่ต่างจากเรียกครั้งเดียว) แต่
`POST /users` เรียกซ้ำสิบครั้งอาจสร้าง user ใหม่สิบคน (ไม่ idempotent)

## 3. HTTP Status Codes

```
1xx: Informational (แจ้งเตือน กำลังดำเนินการ)
2xx: Success (สำเร็จ)
3xx: Redirection (เปลี่ยนทาง)
4xx: Client Error (ผู้เรียกใช้ผิด)
5xx: Server Error (server มีปัญหา)
```

| Code | ความหมาย | ใช้เมื่อ |
|---|---|---|
| `200 OK` | สำเร็จ | GET/PUT/PATCH สำเร็จ |
| `201 Created` | สร้างสำเร็จ | POST ที่สร้าง resource ใหม่ |
| `204 No Content` | สำเร็จ แต่ไม่มีข้อมูลคืนกลับ | DELETE สำเร็จ |
| `301 Moved Permanently` | ย้ายไปที่ URL ใหม่ถาวร | |
| `304 Not Modified` | ข้อมูลไม่เปลี่ยน (ใช้ cache ได้) | |
| `400 Bad Request` | request ผิดรูปแบบ | ข้อมูลที่ส่งมาไม่ถูกต้อง (ทบทวน validation จาก Part 81) |
| `401 Unauthorized` | ยังไม่ยืนยันตัวตน | ไม่ได้ login หรือ token ไม่ถูกต้อง (ทบทวน Part 82-83) |
| `403 Forbidden` | ยืนยันตัวตนแล้วแต่ไม่มีสิทธิ์ | login แล้วแต่ไม่มี permission |
| `404 Not Found` | ไม่พบ resource | URL ไม่มีอยู่จริง |
| `409 Conflict` | ข้อมูลขัดแย้งกัน | เช่น email ซ้ำตอนสมัครสมาชิก |
| `500 Internal Server Error` | server มี bug | exception ที่ไม่ได้ handle |
| `503 Service Unavailable` | server ไม่พร้อมให้บริการชั่วคราว | maintenance, overload |

**ข้อผิดพลาดที่พบบ่อย**: สับสนระหว่าง `401` กับ `403` — `401` คือ **"คุณ
ยังไม่ได้ยืนยันว่าคุณเป็นใคร"** (ยังไม่ login), `403` คือ **"เรารู้ว่าคุณ
เป็นใครแล้ว แต่คุณไม่มีสิทธิ์ทำสิ่งนี้"** (login แล้วแต่ permission ไม่พอ)

## 4. HTTP Headers

**Headers** คือ metadata ที่แนบไปกับ request/response (ทบทวนแนวคิด
metadata จาก Part 52 — Annotations ก็เป็น metadata ในระดับโค้ด, headers
เป็น metadata ในระดับ network):

```
GET /users/1 HTTP/1.1
Host: api.example.com
Accept: application/json          <- client บอกว่ารับ response แบบไหนได้
Authorization: Bearer eyJhbGc...  <- token สำหรับยืนยันตัวตน (Part 83)
User-Agent: Mozilla/5.0...

HTTP/1.1 200 OK
Content-Type: application/json     <- server บอกว่า response เป็นชนิดไหน
Content-Length: 234
Cache-Control: max-age=3600         <- บอกว่า cache ได้นานแค่ไหน
Set-Cookie: sessionId=abc123        <- server สั่งให้ client เก็บ cookie (หัวข้อ 7)
```

## 5. HTTP Request/Response Anatomy แบบละเอียด

```java
import java.net.URI;
import java.net.http.*;

public class HttpAnatomyDemo {
    public static void main(String[] args) throws Exception {
        // สร้าง request ที่มีทุกส่วนประกอบ (ทบทวน HttpClient จาก Part 70)
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create("https://api.example.com/users"))       // URL
                .header("Accept", "application/json")                    // Headers
                .header("Authorization", "Bearer token123")
                .POST(HttpRequest.BodyPublishers.ofString("""
                        {"name": "Alice"}
                        """))                                              // Method + Body
                .build();

        HttpClient client = HttpClient.newHttpClient();
        HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());

        System.out.println("Status: " + response.statusCode());          // Status Code
        System.out.println("Headers: " + response.headers().map());       // Response Headers
        System.out.println("Body: " + response.body());                    // Response Body
    }
}
```

## 6. Statelessness ของ HTTP และ Session

**HTTP เป็น Stateless Protocol**: **ทุก request เป็นอิสระจากกันโดยสิ้นเชิง**
— server **ไม่จำ**ว่า request ก่อนหน้ามาจากใคร ทุก request ต้องมีข้อมูลครบ
ในตัวเองว่า "ฉันคือใคร ต้องการทำอะไร"

```
Request 1: GET /profile  -> Server: "ฉันไม่รู้ว่าคุณคือใคร ต้องแจ้งตัวตนมาด้วย"
Request 2: GET /profile  -> Server: (ยังจำ Request 1 ไม่ได้เลย - เป็น request ใหม่โดยสิ้นเชิง)
```

**Session** คือกลไกที่**สร้างสถานะจำลอง (state)** บน stateless protocol —
server สร้าง session ID เก็บข้อมูลผู้ใช้ไว้ฝั่ง server แล้วส่ง session ID
กลับไปให้ client เก็บไว้ (ผ่าน cookie — หัวข้อ 7) เพื่อใช้อ้างอิงใน request
ถัดไป

```
Request 1 (login): POST /login -> Server: สร้าง session, ส่ง Set-Cookie: sessionId=abc123
Request 2: GET /profile + Cookie: sessionId=abc123 -> Server: หา session abc123 เจอ รู้ว่าเป็นใคร
```

## 7. Cookies

**Cookie** คือข้อมูลขนาดเล็กที่ **server สั่งให้ browser เก็บไว้** แล้ว
**browser จะส่งกลับไปให้ server ทุกครั้ง**ที่ request ไปยัง domain เดียวกัน

```
Response จาก server:
Set-Cookie: sessionId=abc123; Max-Age=3600; HttpOnly; Secure; SameSite=Strict

Request ถัดไปจาก browser (อัตโนมัติ):
Cookie: sessionId=abc123
```

| Attribute | ความหมาย |
|---|---|
| `Max-Age` | อายุของ cookie (วินาที) |
| `HttpOnly` | JavaScript อ่าน cookie นี้ไม่ได้ (ป้องกัน XSS attack — ทบทวนความสำคัญด้านความปลอดภัยจะลงลึกใน Part 101) |
| `Secure` | ส่งได้ผ่าน HTTPS เท่านั้น |
| `SameSite` | ป้องกัน CSRF attack (ควบคุมว่า cookie ส่งข้าม site ได้หรือไม่) |

## 8. HTTPS และ TLS

**HTTPS** คือ HTTP ที่ทำงานผ่าน **TLS (Transport Layer Security)** —
เข้ารหัสข้อมูลระหว่าง client และ server ป้องกันการดักฟัง (eavesdropping)
และการปลอมแปลง (tampering) ระหว่างทาง

```
HTTP (ไม่เข้ารหัส):   Client ──[ข้อความชัดเจน อ่านได้]──> Server  (อันตราย! ใครดักฟังก็อ่านได้)
HTTPS (เข้ารหัส):     Client ──[ข้อมูลเข้ารหัส TLS]──> Server     (ปลอดภัย - อ่านไม่ได้แม้ดักฟังได้)
```

**ในโค้ดปฏิบัติ**: production application **ต้องใช้ HTTPS เสมอ** ไม่มี
เหตุผลที่ยอมรับได้ให้ใช้ HTTP ธรรมดาสำหรับข้อมูลที่มีความสำคัญ (ทบทวน
ความสำคัญของความปลอดภัยที่จะลงลึกเต็มรูปแบบใน Part 82-83, 101)

## 9. RESTful Architecture เบื้องต้น (ปูทางสู่ Part 78, 85)

**REST (Representational State Transfer)** เป็นสถาปัตยกรรมที่ใช้ HTTP
Methods และ Status Codes ตามความหมายที่ถูกต้อง (semantic) เพื่อออกแบบ API
ที่**เข้าใจง่ายและสอดคล้องกัน**

```
GET    /users          -> ดึงรายชื่อผู้ใช้ทั้งหมด
GET    /users/1         -> ดึงข้อมูลผู้ใช้ id=1
POST   /users            -> สร้างผู้ใช้ใหม่
PUT    /users/1           -> แทนที่ข้อมูลผู้ใช้ id=1 ทั้งหมด
PATCH  /users/1            -> แก้ไขข้อมูลผู้ใช้ id=1 บางส่วน
DELETE /users/1              -> ลบผู้ใช้ id=1
```

**หลักการ**: ใช้ **noun (คำนาม)** สำหรับ URL เสมอ (`/users` ไม่ใช่
`/getUsers`) แล้วให้ **HTTP Method บอกว่าจะทำอะไร** กับ resource นั้น —
เนื้อหาเต็มรูปแบบเรื่องการออกแบบ RESTful API จะอยู่ใน Part 78, 85

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** ระบุ HTTP status code ที่เหมาะสมสำหรับสถานการณ์เหล่านี้: (a) สมัคร
สมาชิกด้วยอีเมลที่มีอยู่แล้ว (b) ขอดูข้อมูล user ที่ id ไม่มีอยู่จริง (c)
ลบ post สำเร็จ (d) ส่ง request ที่ JSON body ผิดรูปแบบ

**เฉลย**: (a) `409 Conflict` (ข้อมูลขัดแย้งกับที่มีอยู่แล้ว) (b) `404 Not
Found` (c) `204 No Content` (สำเร็จ ไม่มีข้อมูลคืนกลับ) (d) `400 Bad
Request` (client ส่งข้อมูลผิดรูปแบบ)

**2)** อธิบายว่าทำไม `GET` ควรเป็น idempotent แต่ `POST` ไม่จำเป็นต้องเป็น

**เฉลย**: `GET` มีเจตนาแค่**ดึงข้อมูล**โดยไม่เปลี่ยนแปลงสถานะของระบบ (ทบทวน
แนวคิด pure function/side-effect-free จาก Part 40) ดังนั้นเรียกซ้ำกี่ครั้ง
ก็ควรได้ผลลัพธ์เหมือนกัน (ข้อมูลเดิม ไม่มีผลข้างเคียง) ส่วน `POST` มีเจตนา
**สร้างข้อมูลใหม่** ซึ่งโดยธรรมชาติแล้วการเรียกซ้ำ**ควร**สร้างข้อมูลใหม่อีก
ชุด (เช่น การส่ง order สองครั้งควรได้สอง order ไม่ใช่ order เดียว) — ถ้า
ต้องการ operation ที่สร้าง/แทนที่ข้อมูลแต่ยัง idempotent ได้ ควรใช้ `PUT`
แทน (ทบทวนตาราง Idempotent ในหัวข้อ 2)

**3)** อธิบายว่า Cookie แก้ปัญหา "HTTP เป็น Stateless" ได้อย่างไร

**เฉลย**: HTTP ไม่มีกลไก "จำ" request ก่อนหน้าในตัวเอง (ทบทวนหัวข้อ 6) —
Cookie แก้ปัญหานี้โดยให้ server**ส่งข้อมูลระบุตัวตนกลับไปเก็บที่ client**
(ผ่าน `Set-Cookie` header) แล้ว**client (browser) ส่งข้อมูลนั้นกลับมาให้
server อัตโนมัติทุกครั้ง**ที่ request ไปยัง domain เดียวกัน (ผ่าน `Cookie`
header) — ทำให้ server สามารถ**เชื่อมโยง request หลายครั้งเข้าด้วยกัน**ได้
ว่ามาจาก "ผู้ใช้เดียวกัน" แม้ว่า HTTP protocol เองจะไม่มีแนวคิดเรื่อง state
ในตัวมันเองเลยก็ตาม — นี่คือกลไกพื้นฐานของ Session-based Authentication ที่
จะเรียนเชิงลึกใน Part 82

### สรุปเนื้อหา Part 71

- HTTP ทำงานตามรูปแบบ Request-Response บน TCP Socket (ทบทวนจาก Part 70)
- HTTP Methods (GET, POST, PUT, PATCH, DELETE) มีความหมายเฉพาะ, บางตัว
  idempotent บางตัวไม่
- Status codes แบ่งเป็น 5 กลุ่ม (1xx-5xx) แต่ละ code มีความหมายเฉพาะ —
  401 vs 403 ต้องแยกให้ถูก
- HTTP เป็น stateless — Session + Cookie สร้างสถานะจำลองเพื่อจดจำผู้ใช้
  ข้าม request
- HTTPS เข้ารหัสข้อมูลด้วย TLS ป้องกันการดักฟังและปลอมแปลง
- REST ใช้ noun สำหรับ URL และให้ HTTP Method กำหนดการกระทำ

**ต่อไป**: [Part 72 — Servlet และ JSP เบื้องต้น](./part-072-servlet-jsp.md)
