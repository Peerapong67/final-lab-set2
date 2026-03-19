# INDIVIDUAL_REPORT — สมาชิกที่ 1

**วิชา:** ENGSE207 Software Architecture
**งาน:** Final Lab Sec2 Set 2

---

## 1. ข้อมูลผู้จัดทำ

| รายการ | ข้อมูล |
|---|---|
| ชื่อ-นามสกุล | นายพีรพงศ์ ปัญญาสัน |
| รหัสนักศึกษา | 67543210042-7 |
| สาขาวิชา | วิศวกรรมซอฟต์แวร์ |

---

## 2. ส่วนที่รับผิดชอบ

รับผิดชอบ **Auth Service** ทั้งหมด และ **Deploy บน Railway**

---

## 3. สิ่งที่ลงมือทำจริง

เพิ่ม `/api/auth/register` ให้รับ username, email, password แล้ว hash password ด้วย bcrypt ก่อน insert ลง database ตรวจสอบ email และ username ซ้ำก่อนสร้าง user ใหม่

เพิ่ม `logToDB()` เก็บ log ลง auth-db ของตัวเองโดยไม่ต้องพึ่ง service อื่น

เพิ่ม `logActivity()` ส่ง event ไปหา activity-service แบบ fire-and-forget ทั้ง USER_REGISTERED และ USER_LOGIN

ปรับ `db.js` จาก Pool config เดิมมาใช้ `DATABASE_URL` เพื่อให้ deploy บน Railway ได้

Deploy auth-service และ auth-db บน Railway ตั้งค่า Environment Variables และ Generate Domain

---

## 4. ปัญหาที่พบและวิธีแก้ไข

### ปัญหาที่ 1 — Railway หา db.js ไม่เจอ

ทำงานได้ปกติบน local แต่พอ deploy บน Railway ได้ error `Cannot find module './db/db'` ทั้งที่ไฟล์มีอยู่บน GitHub

แก้ด้วยการเปลี่ยนจาก `require('../db/db')` มาใช้ inline Pool แทน

```javascript
const { Pool } = require('pg');
const pool = new Pool({ connectionString: process.env.DATABASE_URL });
```

### ปัญหาที่ 2 — DATABASE_URL reference ไม่ทำงาน

ตั้งค่า `DATABASE_URL = ${{auth-db.DATABASE_URL}}` แต่ service connect DB ไม่ได้ได้ ECONNREFUSED

แก้ด้วยการ copy DATABASE_URL จริงจาก auth-db มาใส่ใน auth-service Variables โดยตรง แทนที่จะใช้ reference variable

---

## 5. Denormalization ใน activities table คืออะไร และทำไมต้องทำ

Denormalization คือการเก็บข้อมูลซ้ำซ้อนโดยตั้งใจ เช่น เก็บ `username` ไว้ใน activities table ทั้งที่มี `user_id` อยู่แล้ว

ทำเพราะ activity-db ไม่มี users table ข้อมูล username อยู่ใน auth-db ถ้าไม่ denormalize ต้องการ username ต้อง query ข้าม 2 databases ซึ่งทำไม่ได้ใน Database-per-Service pattern จึงเก็บ username ไว้ตอนที่ event เกิดขึ้นเลย ยอมให้ข้อมูลซ้ำเพื่อแลกกับความสามารถในการ query ได้โดยไม่ต้องข้าม database

---

## 6. ทำไม logActivity() ต้องเป็น fire-and-forget

เพราะ activity เป็นข้อมูลเสริม ไม่ใช่ core business logic ถ้า login สำเร็จแต่ activity-service ล่ม ก็ไม่ควรทำให้ login ล้มเหลวด้วย

การใช้ `.catch(() => {})` ต่อท้ายทำให้ถ้า activity-service ไม่ตอบ โค้ดจะไม่ throw error และ auth-service ยังทำงานต่อได้ตามปกติ เพียงแต่ activity จะไม่ถูกบันทึกชั่วคราว

---