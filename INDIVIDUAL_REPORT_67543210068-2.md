# INDIVIDUAL_REPORT — สมาชิกที่ 2

**วิชา:** ENGSE207 Software Architecture
**งาน:** Final Lab Sec2 Set 2

---

## 1. ข้อมูลผู้จัดทำ

| รายการ | ข้อมูล |
|---|---|
| ชื่อ-นามสกุล | นายณัฐพงศ์ จันทร์สิงห์ |
| รหัสนักศึกษา | 67543210068-2 |
| สาขาวิชา | วิศวกรรมซอฟต์แวร์ |

---

## 2. ส่วนที่รับผิดชอบ

รับผิดชอบ **Task Service**, **Activity Service** (สร้างใหม่ทั้งหมด) และ **Frontend**

---

## 3. สิ่งที่ลงมือทำจริง

เพิ่ม `logActivity()` ใน task-service ทุก CRUD route โดย TASK_STATUS_CHANGED จะ log เฉพาะเมื่อ status เปลี่ยนจริงๆ ไม่ใช่ทุกครั้งที่ PUT

สร้าง activity-service ใหม่ทั้งหมด ประกอบด้วย POST /internal รับ event, GET /me ดู activities ของตัวเอง, GET /all สำหรับ admin เท่านั้น

สร้าง `config.js` เก็บ Railway URLs แยกออกมา ทำให้เปลี่ยน URL ได้ง่ายโดยไม่ต้องแก้ไฟล์ HTML

ปรับ `index.html` เพิ่ม Register tab กลับมา ลบ Profile tab และ Log Dashboard ออก เพิ่มลิงก์ไปหน้า activity.html

สร้าง `activity.html` หน้าใหม่แสดง Activity Timeline พร้อม stats cards และ filter by event type

Deploy activity-service และ task-service บน Railway

---

## 4. ปัญหาที่พบและวิธีแก้ไข

### ปัญหาที่ 1 — TASK_STATUS_CHANGED log ทุกครั้งที่ PUT แม้ status ไม่เปลี่ยน

ตอนแรกใส่ logActivity() ใน PUT route โดยตรง ทำให้ log TASK_STATUS_CHANGED ทุกครั้งที่แก้ไข task แม้จะแก้แค่ title ไม่ได้เปลี่ยน status

แก้ด้วยการเช็ค old_status ก่อน

```javascript
if (status && status !== check.rows[0].status) {
  logActivity({ eventType: 'TASK_STATUS_CHANGED', ... });
}
```

### ปัญหาที่ 2 — activity.html เปิดแล้ว login overlay ไม่หาย

กด login แล้ว overlay ยังค้างอยู่ ไม่เปลี่ยนไปหน้า timeline

เจอว่าลืมเรียก `initPage()` หลัง login สำเร็จ แก้ด้วยการเพิ่ม function call กลับเข้าไป

---

## 5. Denormalization ใน activities table คืออะไร และทำไมต้องทำ

Denormalization คือการเก็บข้อมูลซ้ำซ้อนโดยตั้งใจ ในที่นี้คือเก็บ `username` ไว้ใน activities table ทั้งที่มี `user_id` อยู่แล้ว

เหตุผลคือ Set 2 ใช้ Database-per-Service ทำให้ activity-db ไม่มี users table อยู่ ถ้าต้องการแสดง username บน activity timeline ต้องไป query auth-db แต่ทำไม่ได้เพราะ database แยกกัน จึงแก้ด้วยการเก็บ username ไว้ตอนที่ event เกิดขึ้นเลย ทำให้ query ได้จาก activity-db ได้เลย

---

## 6. ทำไม logActivity() ต้องเป็น fire-and-forget

ถ้าเรียก logActivity() แบบรอ response การ login หรือสร้าง task จะช้าขึ้น และถ้า activity-service ล่มจะทำให้ login ล้มเหลวด้วยทั้งที่ไม่ควรเกี่ยวกัน

การใช้ fire-and-forget ทำให้ service หลักทำงานเสร็จก่อน แล้วค่อยส่ง event ไปแบบไม่รอ ถ้า activity-service ไม่ตอบก็แค่ log warning แล้วทำงานต่อไปได้เลย ผู้ใช้ไม่รู้สึกถึงความล่าช้าหรือ error ใดๆ

---
