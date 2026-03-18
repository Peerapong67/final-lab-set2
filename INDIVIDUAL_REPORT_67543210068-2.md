# INDIVIDUAL_REPORT — สมาชิกที่ 2

**วิชา:** ENGSE207 Software Architecture  
**งาน:** Final Lab Set 1 — Task Board Microservices + HTTPS + Basic Logging

---

## 1. ข้อมูลผู้จัดทำ

| รายการ | ข้อมูล |
|---|---|
| ชื่อ-นามสกุล | นางณัฐพงศ์ จันทร์สิงห์|
| รหัสนักศึกษา | 67543210068-2|
| สาขาวิชา | วิศวกรรมซอฟต์แวร์ |

---

## 2. ส่วนที่รับผิดชอบ

รับผิดชอบหลัก 3 ส่วน ได้แก่ **Task Service**, **Log Service**, และ **Frontend (index.html + logs.html)**

---

## 3. สิ่งที่ลงมือพัฒนาด้วยตนเอง

### 3.1 Task Service (`task-service/`)

**JWT Middleware (`authMiddleware.js`)**

เขียน middleware `requireAuth` ที่ดึง token จาก `Authorization: Bearer <token>` header แล้วเรียก `verifyToken()` ถ้า token ไม่ถูกต้องหรือหมดอายุ → return 401 พร้อม log event `JWT_INVALID` ไปที่ Log Service แบบ fire-and-forget ด้วย `fetch().catch(() => {})` เพื่อไม่ให้ middleware block เมื่อ Log Service ไม่พร้อม

ถ้า token ถูกต้อง → แนบ decoded payload เป็น `req.user` ให้ route handlers ใช้งานต่อได้ทันที

**Task Routes (`tasks.js`)**

เขียน 4 endpoints ที่มีเงื่อนไข authorization ต่างกัน:

- `GET /api/tasks/` — ถ้า `req.user.role === 'admin'` ดึงทุก task พร้อม JOIN username, ถ้าเป็น member ดึงเฉพาะ `user_id = req.user.sub`
- `POST /api/tasks/` — สร้าง task ด้วย `user_id` จาก JWT payload (ไม่รับ user_id จาก request body เพื่อความปลอดภัย)
- `PUT /api/tasks/:id` — ตรวจสอบ ownership ก่อน update: `task.user_id !== req.user.sub && req.user.role !== 'admin'` → 403
- `DELETE /api/tasks/:id` — ตรวจสอบ ownership เช่นเดียวกัน

ใช้ `COALESCE` ใน UPDATE query เพื่อให้ส่ง partial update ได้ ไม่ต้องส่ง field ทุกตัว

### 3.2 Log Service (`log-service/src/index.js`)

**POST /api/logs/internal**

รับ log payload จาก services อื่น และ INSERT ลงตาราง `logs` ใน PostgreSQL ตรวจสอบ required fields (`service`, `level`, `event`) ถ้าขาดให้ return 400 ไม่มีการตรวจ JWT เพราะเรียกภายใน Docker network เท่านั้น — Nginx block path นี้จากภายนอกอยู่แล้ว

**GET /api/logs/** พร้อม Filter

เขียน dynamic query builder ที่รับ query params `service`, `level`, `event` แล้วสร้าง WHERE clause แบบ parameterized เพื่อป้องกัน SQL Injection โดยเพิ่ม condition ทีละตัวพร้อม placeholder `$N`:

```javascript
const conditions = [];
const values = [];
let idx = 1;
if (service) { conditions.push(`service = $${idx++}`); values.push(service); }
// ...
const where = conditions.length ? 'WHERE ' + conditions.join(' AND ') : '';
```

**requireAdmin() Middleware**

ตรวจสอบ JWT ด้วย `jwt.verify()` แล้วเช็ค `user.role !== 'admin'` → return 403 เพื่อให้ admin เท่านั้นที่ดู log ได้ แม้จะมี JWT token ที่ถูกต้อง

**GET /api/logs/stats**

ใช้ `Promise.all()` รัน 4 query พร้อมกัน (by_level, by_service, top_events, last_24h) เพื่อลด latency

### 3.3 Frontend

**index.html — Task Board UI**

ปรับ code จาก Week 12 โดยลบส่วนที่ไม่ต้องการออก ได้แก่:
- ลบ Register tab และ `doRegister()` function ทั้งหมด
- ลบ Users nav และ `loadUsers()` function
- เปลี่ยน `apiFetch()` wrapper กลับเป็น `fetch()` ธรรมดา (Log Service จัดการแล้ว)
- แก้ `user.name` → `user.username` ทุกจุด
- แก้ `loadProfile()` ให้เรียก `/api/auth/me` แทน `/api/users/me`
- เพิ่ม link ไปหน้า `logs.html` ใน sidebar (แสดงเฉพาะ admin)
- เพิ่ม Seed Users hint ใน login form

เขียนฟังก์ชันหลัก: `doLogin()`, `initApp()`, `showPage()`, `loadTasks()`, `renderTasks()`, `openCreateModal()`, `submitTask()`, `confirmDelete()`, `loadProfile()`, `renderJwtInspector()`

**logs.html — Log Dashboard**

สร้างใหม่ทั้งหมด (ไม่ copy จาก Week 12 เพราะ Week 12 อ่านจาก localStorage) ประกอบด้วย:
- Login overlay สำหรับ admin (ตรวจ role ก่อน proceed)
- Stats cards 4 ใบ: Total, INFO, WARN, ERROR
- Filter bar: search text, dropdown service, dropdown level
- Log table แสดง: เวลา, service, level badge, event, message + meta
- Auto-refresh ทุก 5 วินาที พร้อม toggle on/off
- Client-side search filter บน `allLogs` array

---

## 4. ปัญหาที่พบและวิธีแก้ไข

### ปัญหาที่ 1 — เรียก API แล้วได้ 401 ทุกครั้ง ทั้งที่ login แล้ว
 
กด login แล้วก็ได้ token มา แต่พอเรียก `/api/tasks/` กลับได้ 401 กลับมาตลอด งงมากว่าเกิดอะไรขึ้น
 
เปิด DevTools ดูถึงเจอว่าลืมใส่ Authorization header ไปกับ request เลยเพิ่ม `-H "Authorization: Bearer $TOKEN"` เข้าไป ก็ใช้งานได้ปกติแล้ว
 
### ปัญหาที่ 2 — กด login แล้วหน้าเว็บไม่เปลี่ยน ค้างอยู่ที่หน้า login
 
กรอก email password แล้วกด login แต่ไม่มีอะไรเกิดขึ้นเลย ไม่มี error ไม่มีอะไร
 
เปิด console ดูถึงเห็นว่า fetch ส่ง request ไปผิด URL เพราะพิมพ์ `/api/auth/loginn` เกิน n มาตัวหนึ่ง แก้ spelling แล้วใช้งานได้ทันที
 


---

## 5. สิ่งที่ได้เรียนรู้

* งานนี้เป็นครั้งแรกที่ได้ทำ Frontend จริงๆ แบบต่อกับ Backend หลายตัวพร้อมกัน ก่อนหน้านี้คิดว่าแค่เรียก API แล้วแสดงผลก็จบ แต่จริงๆ มีเรื่องที่ต้องคิดอีกเยอะ เช่น token หมดอายุต้องทำยังไง หรือถ้า service ไม่ตอบจะ handle ยังไง

* เรื่อง JWT ได้เข้าใจมากขึ้นว่า token ที่ได้มาเก็บข้อมูลอยู่ข้างในด้วย ไม่ใช่แค่รหัสผ่านสุ่มๆ

* อีกเรื่องที่ไม่เคยรู้มาก่อนคือ CORS ตอนแรกงงมากว่าทำไม browser ถึงบล็อก ทั้งที่เรียก server ตัวเดียวกัน พอเข้าใจแล้วก็รู้สึกว่า Nginx ช่วยแก้ปัญหานี้ได้ดีมาก 

* โดยรวมชอบงานนี้ได้เห็นภาพรวมของระบบจริงๆ ว่า frontend กับ backend คุยกันยังไง และทำไมถึงต้องมี Nginx คั่นกลาง

---

## 6. แนวทางที่จะพัฒนาต่อใน Set 2

**ด้าน Task Service:**
- เพิ่ม pagination สำหรับ task list (limit/offset หรือ cursor-based)
- เพิ่ม due_date field และ filter ตาม status/priority
- เพิ่ม real-time update ด้วย WebSocket หรือ Server-Sent Events

**ด้าน Log Service:**
- เพิ่ม log retention policy — auto-delete log เก่ากว่า N วัน
- เพิ่ม alert system — ถ้า ERROR count สูงเกินค่า threshold ส่ง notification
- เพิ่ม log export เป็น CSV

**ด้าน Frontend:**
- เพิ่ม Drag-and-Drop สำหรับเปลี่ยน status (Kanban board style)
- เพิ่ม Pagination ใน log table
- ย้ายไปใช้ Framework (React/Vue) เพื่อจัดการ state ที่ซับซ้อนขึ้น

**ด้าน Infrastructure สำหรับ Cloud (Set 2):**
- แยก Database per Service (Task DB และ Log DB)
- ใช้ Docker Secrets แทน environment variables สำหรับ JWT_SECRET
- เพิ่ม CI/CD pipeline (GitHub Actions) build → test → deploy