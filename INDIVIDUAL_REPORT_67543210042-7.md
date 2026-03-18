# INDIVIDUAL_REPORT — สมาชิกที่ 1

**วิชา:** ENGSE207 Software Architecture  
**งาน:** Final Lab Set 1 — Task Board Microservices + HTTPS + Basic Logging

---

## 1. ข้อมูลผู้จัดทำ

| รายการ | ข้อมูล |
|---|---|
| ชื่อ-นามสกุล | นายพีรพงศ์ ปัญญาสัน |
| รหัสนักศึกษา | 67543210042-7 |
| สาขาวิชา | วิศวกรรมซอฟต์แวร์ |

---

## 2. ส่วนที่รับผิดชอบ

รับผิดชอบหลัก 3 ส่วน ได้แก่ **Auth Service**, **Nginx + HTTPS**, และ **Database Schema + Seed Users**

---

## 3. สิ่งที่ลงมือพัฒนา
### 3.1 Auth Service (`auth-service/`)

**Login Route (`POST /api/auth/login`)**

เขียน logic ตรวจสอบผู้ใช้โดย query จาก PostgreSQL ด้วย email แล้วใช้ `bcrypt.compare()` เปรียบเทียบ password กับ hash ที่เก็บไว้ใน DB ส่วนที่ต้องระวังคือ Timing Attack — ถ้าไม่พบ user ในระบบ ต้องยัง run `bcrypt.compare()` กับ dummy hash เพื่อให้ใช้เวลาเท่ากัน ไม่เช่นนั้น attacker จะรู้ว่า email นี้มีในระบบหรือไม่จากเวลา response

```javascript
const DUMMY_HASH = '$2b$10$CwTycUXWue0Thq9StjUM0u...';
const user = result.rows[0] || null;
const hash = user ? user.password_hash : DUMMY_HASH;
const isValid = await bcrypt.compare(password, hash);
if (!user || !isValid) { /* return 401 */ }
```

**JWT Utils (`jwtUtils.js`)**

เขียน `generateToken(payload)` และ `verifyToken(token)` โดยใช้ library `jsonwebtoken` — `generateToken` รับ payload `{ sub, email, role, username }` แล้ว sign ด้วย `JWT_SECRET` จาก environment variable กำหนด expires เป็น `1h` ผ่าน `JWT_EXPIRES`

**logEvent() Helper**

เพิ่มฟังก์ชัน helper ที่เรียก `POST http://log-service:3003/api/logs/internal` แบบ fire-and-forget ใน try-catch เพื่อให้ระบบ login ไม่ล้มเหลวแม้ Log Service จะไม่ตอบสนอง

### 3.2 Nginx + HTTPS (`nginx/`)

**Self-Signed Certificate (`scripts/gen-certs.sh`)**

เขียน bash script ใช้ `openssl req` สร้าง self-signed certificate อายุ 365 วัน กำหนด Subject `/C=TH/ST=ChiangMai/O=RMUTL/CN=localhost` certificate ถูกสร้างเป็น `cert.pem` และ `key.pem` ใน `nginx/certs/` และถูก gitignore ไว้ไม่ให้ขึ้น repository

**nginx.conf**

ตั้งค่า 2 server block:
- HTTP (:80) — `return 301 https://$host$request_uri` redirect ทุก request ไป HTTPS
- HTTPS (:443) — `ssl_certificate` และ `ssl_certificate_key` ชี้ไปที่ cert ที่สร้างไว้, เปิดใช้ `TLSv1.2 TLSv1.3`, เพิ่ม security headers (`HSTS`, `X-Frame-Options`, `X-Content-Type-Options`)

กำหนด Rate Limit 2 zone: `login_limit` สำหรับ `/api/auth/login` (5 req/min) และ `api_limit` สำหรับ endpoint ทั่วไป (30 req/min)

Block `/api/logs/internal` ด้วย `location = /api/logs/internal { return 403; }` ก่อน location `/api/logs/` เพื่อให้ exact match มีความสำคัญสูงกว่า

### 3.3 Database Schema (`db/init.sql`)

ออกแบบและเขียน SQL สำหรับ 3 ตาราง:

- **users** — `id`, `username`, `email`, `password_hash`, `role`, `created_at`, `last_login`
- **tasks** — `id`, `user_id` (FK → users), `title`, `description`, `status` (CHECK constraint), `priority` (CHECK constraint), `created_at`, `updated_at`
- **logs** — `id`, `service`, `level` (CHECK), `event`, `user_id`, `ip_address`, `method`, `path`, `status_code`, `message`, `meta` (JSONB), `created_at`

สร้าง index บนตาราง logs 3 ตัว: `service`, `level`, `created_at DESC` เพื่อเพิ่มประสิทธิภาพการ query

เพิ่ม Seed Users 3 บัญชีพร้อม bcrypt hash ที่ generate ด้วย `bcryptjs.hashSync()` และใช้ `ON CONFLICT DO UPDATE` เพื่อให้ re-run init.sql ได้โดยไม่ error

---

## 4. ปัญหาที่พบและวิธีแก้ไข

### ปัญหาที่ 1 — รัน docker compose แล้ว service crash ทันที

`docker compose up` ครั้งแรก auth-service พังทุกครั้ง เพราะ service พยายาม connect ฐานข้อมูลตั้งแต่แรก แต่ PostgreSQL ยังไม่ทันพร้อม

แก้ด้วยการให้ service วนรอ ถ้า connect ไม่ได้ก็รอ 3 วินาทีแล้วลองใหม่ ทำแบบนี้สูงสุด 10 รอบ ซึ่งนานพอให้ฐานข้อมูลพร้อมได้ทันเสมอ

### ปัญหาที่ 2 — Browser แจ้งเตือนว่าเว็บไม่ปลอดภัย

เปิด `https://localhost` แล้ว Chrome ขึ้น "Your connection is not private" ตกใจคิดว่า HTTPS พัง

จริงๆ แล้วไม่ใช่ bug — เป็นเพราะใช้ self-signed certificate ที่สร้างเอง browser เลยไม่ยอมรับ กด Advanced แล้วกด Proceed to localhost ได้เลย ไม่มีปัญหาอะไร

---

## 5. สิ่งที่ได้เรียนรู้
* งานนี้ทำให้เข้าใจเรื่อง HTTPS มากขึ้น ก่อนหน้านี้รู้แค่ว่ามันทำให้เว็บปลอดภัย แต่ไม่รู้ว่า certificate คืออะไร พอได้ลองสร้าง self-signed certificate เองและ config Nginx ก็เริ่มเข้าใจมากขึ้นว่ามันทำงานยังไง
 
* เรื่อง JWT ก็เป็นอีกอย่างที่ได้เรียนรู้ใหม่ ตอนแรกคิดว่า server ต้องเก็บข้อมูล session ไว้เอง แต่จริงๆ แล้ว JWT เก็บข้อมูลไว้ใน token เลย server แค่ verify ว่า token ถูกต้องไหมก็พอ
 
* โดยรวมงานนี้ได้ลองทำ microservices จริงๆ ครั้งแรก รู้สึกว่าการที่แต่ละ service แยกกันทำงานมันซับซ้อนกว่าที่คิด แต่ก็เห็นว่ามันยืดหยุ่นกว่ามากถ้าต้องการแก้หรือเพิ่มส่วนใดส่วนหนึ่ง

---

## 6. แนวทางที่จะพัฒนาต่อใน Set 2

**ด้าน Authentication:**
- เพิ่ม Refresh Token เพื่อให้ขอ access token ใหม่ได้โดยไม่ต้อง login ซ้ำ
- เพิ่ม Token Blacklist หรือ Redis cache เพื่อรองรับ logout จริง (revoke token)
- เพิ่มระบบ Register + Email Verification สำหรับ production

**ด้าน Infrastructure:**
- เปลี่ยนจาก Self-Signed Certificate เป็น Let's Encrypt บน Cloud
- แยก Database per Service (Auth DB, Task DB, Log DB) ตาม Microservices principle
- เพิ่ม CORS policy ที่ strict กว่านี้

**ด้าน Security:**
- เพิ่ม HTTPS certificate rotation
- เพิ่ม IP Whitelist สำหรับ internal endpoints
- เพิ่ม Request ID header เพื่อ trace request ข้าม services