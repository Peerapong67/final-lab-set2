# TEAM_SPLIT.md — Final Lab Set 1

## ข้อมูลกลุ่ม

**วิชา:** ENGSE207 Software Architecture  
**งาน:** Final Lab Set 1 — Task Board Microservices + HTTPS + Basic Logging

## รายชื่อสมาชิก

| รหัสนักศึกษา | ชื่อ-นามสกุล |
|---|---|
| 67543210042-7 | นายพีรพงศ์ ปัญญาสัน |
| 67543210068-2 | นายณัฐพงศ์ จันทร์สิงห์ |

---

## การแบ่งงาน

### สมาชิกที่ 1 — นายพีรพงศ์ ปัญญาสัน (67543210042-7)

**รับผิดชอบหลัก:**

- **Auth Service** (`auth-service/`)
  - เขียน `POST /api/auth/login` ตรวจสอบ password ด้วย bcrypt
  - เขียน `GET /api/auth/verify` และ `GET /api/auth/me`
  - เขียน `jwtUtils.js` — `generateToken()` และ `verifyToken()`
  - เพิ่ม `logEvent()` ส่ง log ไปที่ Log Service

- **Nginx + HTTPS** (`nginx/`)
  - เขียน `nginx.conf` — HTTPS server block, HTTP redirect, proxy rules
  - ตั้งค่า Rate Limit (`login_limit`, `api_limit`)
  - Block `/api/logs/internal` ด้วย `return 403`
  - เขียน `scripts/gen-certs.sh` สร้าง self-signed certificate

- **Database** (`db/`)
  - เขียน `db/init.sql` — schema ตาราง `users`, `tasks`, `logs`
  - เพิ่ม Seed Users (alice, bob, admin) พร้อม bcrypt hash
  - เพิ่ม Seed Tasks ตั้งต้น
  - สร้าง index บนตาราง logs

- **Docker Compose** (ร่วมกับสมาชิกที่ 2)
  - ตั้งค่า `postgres` service + healthcheck
  - ตั้งค่า `auth-service` และ `nginx` service
  - เขียน `.env.example`

---

### สมาชิกที่ 2 — นายณัฐพงศ์ จันทร์สิงห์ (67543210068-2)

**รับผิดชอบหลัก:**

- **Task Service** (`task-service/`)
  - เขียน `GET /api/tasks/` — admin เห็นทั้งหมด, member เห็นเฉพาะของตัวเอง
  - เขียน `POST /api/tasks/` สร้าง task ใหม่
  - เขียน `PUT /api/tasks/:id` แก้ไข task (เจ้าของหรือ admin)
  - เขียน `DELETE /api/tasks/:id` ลบ task (เจ้าของหรือ admin)
  - เขียน `authMiddleware.js` — `requireAuth()` ตรวจสอบ JWT
  - เพิ่ม `logEvent()` ส่ง log ไปที่ Log Service

- **Log Service** (`log-service/`)
  - เขียน `POST /api/logs/internal` รับ log จาก services อื่น
  - เขียน `GET /api/logs/` พร้อม query filter (service, level, event)
  - เขียน `GET /api/logs/stats` สรุปสถิติ log
  - เขียน `requireAdmin()` middleware ตรวจ JWT + role=admin

- **Frontend** (`frontend/`)
  - เขียน `index.html` — Task Board UI (Login, Task list, Create/Edit modal, JWT Inspector)
  - เขียน `logs.html` — Log Dashboard (stats, filter, auto-refresh)
  - ปรับ code จาก Week 12 (ลบ Register tab, แก้ `user.name` → `user.username`)
  - ปรับ API calls ให้ใช้ relative URL ผ่าน Nginx

- **Docker Compose** (ร่วมกับสมาชิกที่ 1)
  - ตั้งค่า `task-service`, `log-service`, `frontend` service
  - ตรวจสอบ `depends_on` และ restart policy

---

## งานที่ทำร่วมกัน

| งาน | ผู้รับผิดชอบ |
|---|---|
| Architecture design และ วางโครงสร้าง repo | ทั้งคู่ |
| End-to-end testing ด้วย Postman/curl | ทั้งคู่ |
| จัดทำ screenshots 12 รูป | ทั้งคู่ |
| README.md | ทั้งคู่ |
| แก้ bug ข้ามส่วน | ทั้งคู่ |

---

## Integration Notes

### จุดเชื่อมต่อสำคัญระหว่างงานทั้งสองส่วน

**1. logEvent() — Auth/Task → Log Service**

สมาชิกที่ 1 (Auth) และสมาชิกที่ 2 (Task) ต่างเรียก `logEvent()` ไปที่ Log Service ของสมาชิกที่ 2 ผ่าน URL `http://log-service:3003/api/logs/internal` ภายใน Docker network ตกลง schema payload ร่วมกันตั้งแต่แรก ได้แก่ `{ service, level, event, user_id, ip_address, method, path, status_code, message, meta }`

**2. JWT Secret ร่วมกัน**

Auth Service (สมาชิกที่ 1) ออก JWT ด้วย `JWT_SECRET` เดียวกับที่ Task Service และ Log Service (สมาชิกที่ 2) ใช้ verify — ทั้งสองส่วนอ่านจาก `process.env.JWT_SECRET` ซึ่งกำหนดใน `.env` และส่งผ่าน `docker-compose.yml`

**3. Nginx Proxy Rules (สมาชิกที่ 1) → Services (สมาชิกที่ 2)**

สมาชิกที่ 1 เขียน nginx.conf ให้ proxy `/api/tasks/*` ไปที่ `task-service:3002` และ `/api/logs/*` ไปที่ `log-service:3003` โดยใช้ชื่อ service ตรงกับที่ระบุใน `docker-compose.yml`

**4. Database Schema ร่วมกัน**

สมาชิกที่ 1 เขียน `db/init.sql` ที่สร้างตาราง `users`, `tasks`, `logs` ทั้งสมาชิกที่ 2 ใช้ตารางนี้ตรงๆ — ตกลง column names ร่วมกันก่อนพัฒนา

---

## Git Branching Strategy

```
main
├── feat/auth-service        (สมาชิกที่ 1)
├── feat/nginx-https         (สมาชิกที่ 1)
├── feat/db-schema           (สมาชิกที่ 1)
├── feat/task-service        (สมาชิกที่ 2)
├── feat/log-service         (สมาชิกที่ 2)
└── feat/frontend            (สมาชิกที่ 2)
```
