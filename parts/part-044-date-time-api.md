# Part 44: Date and Time API (java.time)

> ขั้นตอนที่ 431-440 ของหลักสูตร | ระดับ: กลาง

## สารบัญ

1. ทำไม Java 8 ต้องออกแบบ Date/Time API ใหม่
2. `LocalDate`, `LocalTime`, `LocalDateTime`
3. การคำนวณวันที่และเวลา
4. `Period` และ `Duration`
5. `ZonedDateTime` และ `ZoneId`: จัดการ Time Zone
6. `Instant`: จุดเวลาแบบ machine-readable
7. การแปลงระหว่างชนิดข้อมูลเวลาต่าง ๆ
8. `DateTimeFormatter`: จัดรูปแบบวันที่
9. การแปลงจาก/ไปยัง Legacy `Date`/`Calendar`
10. แบบฝึกหัดและสรุป

---

## 1. ทำไม Java 8 ต้องออกแบบ Date/Time API ใหม่

ก่อน Java 8 การจัดการวันที่ใน Java ใช้ `java.util.Date` และ `java.util.Calendar`
ซึ่งมีปัญหาที่รู้จักกันดีในวงการ:

- **Mutable**: `Date` object แก้ไขค่าได้หลังสร้าง (ขัดกับหลัก immutability ที่ดี
  — ทบทวนจาก Part 17) ทำให้ไม่ thread-safe และเสี่ยง bug
- **Month เริ่มที่ 0**: `Calendar.JANUARY == 0` ทำให้สับสนและเป็นสาเหตุ bug บ่อย
- **Design ที่สับสน**: `Date` ทำหน้าที่ปนกันระหว่างจุดเวลาและ formatting
- **Thread-unsafe**: `SimpleDateFormat` ไม่ thread-safe ทำให้เกิดปัญหาในระบบ
  concurrent (Part 46-50)

**`java.time`** (Java 8+, พัฒนาโดย Stephen Colebourne ผู้สร้าง Joda-Time)
แก้ปัญหาทั้งหมดนี้ด้วยการออกแบบใหม่ทั้งหมด: **immutable, thread-safe, และ
เข้าใจง่าย**

## 2. `LocalDate`, `LocalTime`, `LocalDateTime`

```java
import java.time.LocalDate;
import java.time.LocalTime;
import java.time.LocalDateTime;
import java.time.Month;

public class LocalDateTimeDemo {
    public static void main(String[] args) {
        // LocalDate: วันที่อย่างเดียว (ไม่มีเวลา ไม่มี time zone)
        LocalDate today = LocalDate.now();
        LocalDate specificDate = LocalDate.of(2024, Month.JANUARY, 15); // ใช้ enum Month ชัดเจนกว่าตัวเลข
        LocalDate fromNumbers = LocalDate.of(2024, 1, 15); // หรือใช้ตัวเลขตรง ๆ (1=January เสมอ ไม่ใช่ 0!)

        System.out.println(today);
        System.out.println(specificDate);          // 2024-01-15
        System.out.println(specificDate.getYear());   // 2024
        System.out.println(specificDate.getMonth());   // JANUARY
        System.out.println(specificDate.getDayOfWeek()); // MONDAY
        System.out.println(specificDate.isLeapYear());   // true (2024 เป็นปีอธิกสุรทิน)

        // LocalTime: เวลาอย่างเดียว (ไม่มีวันที่)
        LocalTime now = LocalTime.now();
        LocalTime specificTime = LocalTime.of(14, 30, 0); // 14:30:00
        System.out.println(specificTime);

        // LocalDateTime: รวมทั้งวันที่และเวลา (ยังไม่มี time zone)
        LocalDateTime dateTime = LocalDateTime.of(specificDate, specificTime);
        System.out.println(dateTime); // 2024-01-15T14:30
        LocalDateTime now2 = LocalDateTime.now();
    }
}
```

**สำคัญมาก**: ทุก class ใน `java.time` เป็น **immutable** (ทบทวนจาก Part 17)
— เมธอดที่ดู "แก้ไข" ค่าจริง ๆ แล้ว**คืน object ใหม่เสมอ** ไม่แก้ไข object เดิม

## 3. การคำนวณวันที่และเวลา

```java
import java.time.LocalDate;
import java.time.DayOfWeek;

public class DateCalculationDemo {
    public static void main(String[] args) {
        LocalDate date = LocalDate.of(2024, 1, 15);

        LocalDate nextWeek = date.plusDays(7);       // เพิ่ม 7 วัน
        LocalDate lastMonth = date.minusMonths(1);      // ลด 1 เดือน
        LocalDate nextYear = date.plusYears(1);           // เพิ่ม 1 ปี

        System.out.println(date);       // 2024-01-15 (ต้นฉบับไม่เปลี่ยน - immutable!)
        System.out.println(nextWeek);   // 2024-01-22
        System.out.println(lastMonth);   // 2023-12-15
        System.out.println(nextYear);     // 2025-01-15

        // with(): กำหนดค่าเฉพาะบางส่วน คืน object ใหม่
        LocalDate firstOfMonth = date.withDayOfMonth(1);
        System.out.println(firstOfMonth); // 2024-01-01

        // เปรียบเทียบวันที่
        LocalDate other = LocalDate.of(2024, 6, 1);
        System.out.println(date.isBefore(other)); // true
        System.out.println(date.isAfter(other));   // false
        System.out.println(date.equals(other));      // false

        // หาวันจันทร์ถัดไปจากวันนี้ (ใช้ TemporalAdjusters)
        LocalDate nextMonday = date.with(java.time.temporal.TemporalAdjusters.next(DayOfWeek.MONDAY));
        System.out.println(nextMonday);

        // หาวันสุดท้ายของเดือน
        LocalDate lastDayOfMonth = date.with(java.time.temporal.TemporalAdjusters.lastDayOfMonth());
        System.out.println(lastDayOfMonth); // 2024-01-31
    }
}
```

## 4. `Period` และ `Duration`

**`Period`** วัดระยะเวลาแบบ "ปี-เดือน-วัน" (เหมาะกับ `LocalDate`), **`Duration`**
วัดระยะเวลาแบบ "ชั่วโมง-นาที-วินาที" (เหมาะกับเวลาที่แม่นยำระดับวินาที)

```java
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.Period;
import java.time.Duration;

public class PeriodDurationDemo {
    public static void main(String[] args) {
        LocalDate birthDate = LocalDate.of(2000, 5, 15);
        LocalDate today = LocalDate.of(2024, 1, 15);

        Period age = Period.between(birthDate, today);
        System.out.println("อายุ: " + age.getYears() + " ปี " + age.getMonths() + " เดือน");

        Period twoWeeks = Period.ofWeeks(2);
        LocalDate futureDate = today.plus(twoWeeks);
        System.out.println(futureDate);

        // Duration: สำหรับเวลาที่แม่นยำ (ทำงานกับ LocalDateTime, Instant)
        LocalDateTime start = LocalDateTime.of(2024, 1, 15, 9, 0);
        LocalDateTime end = LocalDateTime.of(2024, 1, 15, 17, 30);

        Duration workDuration = Duration.between(start, end);
        System.out.println("ทำงาน: " + workDuration.toHours() + " ชั่วโมง " +
                            (workDuration.toMinutes() % 60) + " นาที");

        Duration oneHour = Duration.ofHours(1);
        Duration ninetyMinutes = Duration.ofMinutes(90);
        System.out.println(oneHour.plus(ninetyMinutes).toMinutes()); // 150
    }
}
```

## 5. `ZonedDateTime` และ `ZoneId`: จัดการ Time Zone

```java
import java.time.ZonedDateTime;
import java.time.ZoneId;
import java.time.LocalDateTime;

public class TimeZoneDemo {
    public static void main(String[] args) {
        ZoneId bangkok = ZoneId.of("Asia/Bangkok");
        ZoneId newYork = ZoneId.of("America/New_York");

        ZonedDateTime bangkokTime = ZonedDateTime.now(bangkok);
        System.out.println("เวลาที่กรุงเทพ: " + bangkokTime);

        // แปลงเวลาข้าม time zone (ยังคงเป็นจุดเวลาเดียวกัน แต่แสดงผลต่างกัน)
        ZonedDateTime newYorkTime = bangkokTime.withZoneSameInstant(newYork);
        System.out.println("เวลาที่นิวยอร์ก (จุดเวลาเดียวกัน): " + newYorkTime);

        // สร้าง ZonedDateTime จาก LocalDateTime + zone ที่ระบุ
        LocalDateTime localDateTime = LocalDateTime.of(2024, 6, 15, 10, 0);
        ZonedDateTime zoned = localDateTime.atZone(bangkok);
        System.out.println(zoned);

        // ดูรายการ zone ID ทั้งหมดที่ Java รู้จัก
        System.out.println("จำนวน time zone ที่รองรับ: " + ZoneId.getAvailableZoneIds().size());
    }
}
```

**คำแนะนำสำคัญ**: ในระบบที่ต้องรองรับผู้ใช้หลาย time zone (เช่น แอปพลิเคชัน
ระดับโลก) ควร**เก็บเวลาเป็น UTC เสมอในฐานข้อมูล** (ใช้ `Instant` หรือ
`ZonedDateTime` กับ zone UTC) แล้วค่อยแปลงเป็น local time zone ของผู้ใช้
**ตอนแสดงผลเท่านั้น**

## 6. `Instant`: จุดเวลาแบบ Machine-readable

**`Instant`** แทน**จุดเวลาบนเส้นเวลา (timeline) แบบสากล** — เป็นจำนวนวินาที
(และ nanosecond) นับจาก **Unix Epoch** (1 มกราคม 1970 UTC) ไม่ขึ้นกับ time
zone ใด ๆ เหมาะสำหรับ**timestamp ในระบบ**และ**การบันทึก log**

```java
import java.time.Instant;
import java.time.Duration;

public class InstantDemo {
    public static void main(String[] args) throws InterruptedException {
        Instant start = Instant.now();
        System.out.println(start); // 2024-01-15T10:30:00.123456Z (Z = UTC)

        Thread.sleep(100); // จำลองการทำงานที่ใช้เวลา (ทบทวนแนวคิด Thread ใน Part 46)

        Instant end = Instant.now();
        Duration elapsed = Duration.between(start, end);
        System.out.println("ใช้เวลา: " + elapsed.toMillis() + " ms");

        // แปลงจาก/ไปยัง epoch millis (เหมาะสำหรับเก็บใน database หรือส่งผ่าน API)
        long epochMillis = end.toEpochMilli();
        Instant fromMillis = Instant.ofEpochMilli(epochMillis);
        System.out.println(fromMillis);
    }
}
```

## 7. การแปลงระหว่างชนิดข้อมูลเวลาต่าง ๆ

```
LocalDate  <----->  LocalDateTime  <----->  ZonedDateTime  <----->  Instant
(วันที่)          (วันที่+เวลา)         (วันที่+เวลา+zone)      (จุดเวลาสากล UTC)
```

```java
import java.time.*;

public class ConversionDemo {
    public static void main(String[] args) {
        LocalDate date = LocalDate.of(2024, 1, 15);
        LocalTime time = LocalTime.of(10, 30);

        LocalDateTime dateTime = date.atTime(time); // LocalDate -> LocalDateTime
        System.out.println(dateTime);

        ZonedDateTime zoned = dateTime.atZone(ZoneId.of("Asia/Bangkok")); // LocalDateTime -> ZonedDateTime
        System.out.println(zoned);

        Instant instant = zoned.toInstant(); // ZonedDateTime -> Instant
        System.out.println(instant);

        LocalDate backToDate = dateTime.toLocalDate(); // LocalDateTime -> LocalDate (ตัดเวลาทิ้ง)
        System.out.println(backToDate);
    }
}
```

## 8. `DateTimeFormatter`: จัดรูปแบบวันที่

```java
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.Locale;

public class DateTimeFormatterDemo {
    public static void main(String[] args) {
        LocalDateTime dateTime = LocalDateTime.of(2024, 1, 15, 14, 30, 0);

        // ISO format มาตรฐาน (ค่า default ตอน toString())
        System.out.println(dateTime); // 2024-01-15T14:30

        // Format แบบกำหนดเอง (pattern letters ตามมาตรฐานของ java.time)
        DateTimeFormatter formatter1 = DateTimeFormatter.ofPattern("dd/MM/yyyy HH:mm:ss");
        System.out.println(dateTime.format(formatter1)); // 15/01/2024 14:30:00

        DateTimeFormatter formatter2 = DateTimeFormatter.ofPattern("EEEE, MMMM d, yyyy", Locale.ENGLISH);
        System.out.println(dateTime.format(formatter2)); // Monday, January 15, 2024

        // Parse: แปลงจาก String กลับเป็น LocalDateTime (ต้องรูปแบบตรงกับ formatter)
        String dateString = "15/01/2024 14:30:00";
        LocalDateTime parsed = LocalDateTime.parse(dateString, formatter1);
        System.out.println(parsed);

        // Format มาตรฐานที่ built-in มาให้แล้ว
        System.out.println(dateTime.format(DateTimeFormatter.ISO_LOCAL_DATE_TIME));
    }
}
```

ตัวอักษร pattern ที่ใช้บ่อย: `yyyy`(ปี 4 หลัก), `MM`(เดือน 2 หลัก), `dd`(วัน 2
หลัก), `HH`(ชั่วโมง 24hr), `mm`(นาที), `ss`(วินาที), `EEEE`(ชื่อวันแบบเต็ม)

## 9. การแปลงจาก/ไปยัง Legacy `Date`/`Calendar`

เมื่อทำงานกับ library เก่าที่ยังใช้ `java.util.Date` ต้องแปลงข้าม API:

```java
import java.time.Instant;
import java.time.LocalDateTime;
import java.time.ZoneId;
import java.util.Date;

public class LegacyConversionDemo {
    public static void main(String[] args) {
        // java.util.Date -> java.time
        Date legacyDate = new Date();
        Instant instant = legacyDate.toInstant();
        LocalDateTime modern = LocalDateTime.ofInstant(instant, ZoneId.systemDefault());
        System.out.println(modern);

        // java.time -> java.util.Date
        Date backToLegacy = Date.from(instant);
        System.out.println(backToLegacy);
    }
}
```

## 10. แบบฝึกหัดและสรุป

### แบบฝึกหัด

**1)** เขียนโปรแกรมคำนวณอายุที่แน่นอน (ปี, เดือน, วัน) จากวันเกิดที่กำหนด

**เฉลย:**

```java
import java.time.LocalDate;
import java.time.Period;

public class Exercise1 {
    public static void main(String[] args) {
        LocalDate birthDate = LocalDate.of(1998, 3, 20);
        LocalDate today = LocalDate.now();
        Period age = Period.between(birthDate, today);
        System.out.printf("อายุ: %d ปี %d เดือน %d วัน%n",
                          age.getYears(), age.getMonths(), age.getDays());
    }
}
```

**2)** เขียนโปรแกรมที่รับเวลาเริ่มและเวลาสิ้นสุดของการประชุม แล้วคำนวณว่าประชุม
กี่นาที

**เฉลย:**

```java
import java.time.LocalTime;
import java.time.Duration;

public class Exercise2 {
    public static void main(String[] args) {
        LocalTime start = LocalTime.of(9, 15);
        LocalTime end = LocalTime.of(10, 45);
        Duration meetingLength = Duration.between(start, end);
        System.out.println("ประชุมทั้งหมด: " + meetingLength.toMinutes() + " นาที");
    }
}
```

**3)** เขียนโปรแกรมแสดงเวลาปัจจุบันใน 3 time zone: Bangkok, Tokyo, London
พร้อม format ที่อ่านง่าย

**เฉลย:**

```java
import java.time.ZonedDateTime;
import java.time.ZoneId;
import java.time.format.DateTimeFormatter;

public class Exercise3 {
    public static void main(String[] args) {
        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("dd/MM/yyyy HH:mm:ss (zzz)");
        String[] zones = {"Asia/Bangkok", "Asia/Tokyo", "Europe/London"};

        for (String zone : zones) {
            ZonedDateTime time = ZonedDateTime.now(ZoneId.of(zone));
            System.out.println(zone + ": " + time.format(formatter));
        }
    }
}
```

### สรุปเนื้อหา Part 44

- `java.time` (Java 8+) แก้ปัญหาของ `Date`/`Calendar` เดิม: เป็น immutable,
  thread-safe, และเข้าใจง่ายกว่ามาก
- `LocalDate`/`LocalTime`/`LocalDateTime` ไม่มี time zone, `ZonedDateTime` มี
  time zone, `Instant` แทนจุดเวลาสากล (UTC)
- `Period` วัดระยะเวลาแบบปี-เดือน-วัน, `Duration` วัดระยะเวลาแบบชั่วโมง-นาที-
  วินาที
- เก็บเวลาเป็น UTC ในฐานข้อมูลเสมอ แปลงเป็น local time zone ตอนแสดงผลเท่านั้น
- `DateTimeFormatter` ใช้จัดรูปแบบและ parse วันที่/เวลา

**ต่อไป**: [Part 45 — Regular Expressions (Regex)](./part-045-regex.md)
