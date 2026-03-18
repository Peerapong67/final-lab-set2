# 📋 Final Lab Set 1 — Task Board Microservices

**วิชา:** ENGSE207 Software Architecture  
**มหาวิทยาลัย:** มหาวิทยาลัยเทคโนโลยีราชมงคลล้านนา  
**ผู้สอน:** อาจารย์ธนิต เกตุแก้ว

## 👥 รายชื่อสมาชิก

| รหัสนักศึกษา | ชื่อ-นามสกุล | บทบาท |
|---|---|---|
| 67543210042-7 | นายพีรพงศ์ ปัญญาสัน | Auth Service, Nginx, Database |
| 67543210068-2 | นางสาวณัฐพงศ์ จันทร์สิงห์ | Task Service, Log Service, Frontend |

> ⚠️ **แก้ไขรหัสนักศึกษาและชื่อ-นามสกุลให้ตรงกับข้อมูลจริงก่อนส่งงาน**

---

## 📌 ภาพรวมและวัตถุประสงค์

Final Lab Set 1 สร้างระบบ **Task Board Microservices** พร้อมฟีเจอร์หลักดังนี้

- **ไม่มี Register** — ใช้ Seed Users ที่กำหนดไว้ล่วงหน้าเท่านั้น
- **HTTPS** — Nginx ใช้ Self-Signed Certificate (port 443) พร้อม redirect HTTP → HTTPS
- **JWT Authentication** — ทุก API ที่ต้องการสิทธิ์ใช้ JWT token
- **Lightweight Logging** — เก็บ log ลง PostgreSQL ผ่าน Log Service REST API
- **Frontend 2 หน้า** — Task Board (`index.html`) และ Log Dashboard (`logs.html`)

---

## 🏗️ Architecture Diagram

```
Browser / Postman
       │
       │ HTTPS :443  (HTTP :80 → redirect HTTPS)
       ▼
┌──────────────────────────────────────────────────────────────┐
│  🛡️ Nginx (API Gateway + TLS Termination + Rate Limiter)     │
│                                                              │
│  /api/auth/*         → auth-service:3001  (ไม่ต้องมี JWT)   │
│  /api/tasks/*        → task-service:3002  [JWT required]     │
│  /api/logs/internal  → BLOCKED (403 จาก Nginx)               │
│  /api/logs/*         → log-service:3003   [JWT + admin only] │
│  /                   → frontend:80        (Static HTML)      │
└──────┬──────────────┬─────────────────┬──────────────────────┘
       │              │                 │
       ▼              ▼                 ▼
┌──────────────┐ ┌───────────────┐ ┌──────────────────┐
│ Auth Service │ │ Task Service  │ │  Log Service     │
│   :3001      │ │   :3002       │ │   :3003          │
│              │ │               │ │                  │
│ POST /login  │ │ GET/POST/PUT  │ │ POST /internal   │
│ GET /verify  │ │ DELETE /tasks │ │ GET /            │
│ GET /me      │ │ JWT Guard     │ │ GET /stats       │
│ logEvent()→  │ │ logEvent()→   │ │ เก็บลง DB        │
└──────┬───────┘ └───────┬───────┘ └──────────────────┘
       │                 │                    │
       └─────────────────┴────────────────────┘
                         │
               ┌─────────────────────┐
               │  PostgreSQL :5432   │
               │  (1 shared DB)      │
               │  • users   table    │
               │  • tasks   table    │
               │  • logs    table    │
               └─────────────────────┘
```

### Services

| Service | Port | หน้าที่ |
|---|---|---|
| nginx | 80, 443 | TLS termination, reverse proxy, rate limit |
| frontend | 80 (internal) | Static HTML/CSS/JS |
| auth-service | 3001 | Login, ออก JWT, Seed Users |
| task-service | 3002 | CRUD Tasks + JWT middleware |
| log-service | 3003 | รับ log ภายใน, เก็บ DB, API ดึง log (admin) |
| postgres | 5432 | ฐานข้อมูลร่วม |

---

## 📁 โครงสร้าง Repository

```
final-lab-set1/
├── README.md
├── TEAM_SPLIT.md
├── INDIVIDUAL_REPORT_67543210042-7.md
├── INDIVIDUAL_REPORT_67543210068-2.md
├── docker-compose.yml
├── .env.example
├── .gitignore
├── nginx/
│   ├── nginx.conf
│   ├── Dockerfile
│   └── certs/          ← สร้างด้วย gen-certs.sh (ไม่ commit ลง Git)
├── frontend/
│   ├── Dockerfile
│   ├── index.html      ← Task Board UI
│   └── logs.html       ← Log Dashboard (admin only)
├── auth-service/
│   ├── Dockerfile
│   ├── package.json
│   └── src/
│       ├── index.js
│       ├── routes/auth.js
│       ├── middleware/jwtUtils.js
│       └── db/db.js
├── task-service/
│   ├── Dockerfile
│   ├── package.json
│   └── src/
│       ├── index.js
│       ├── routes/tasks.js
│       ├── middleware/
│       │   ├── authMiddleware.js
│       │   └── jwtUtils.js
│       └── db/db.js
├── log-service/
│   ├── Dockerfile
│   ├── package.json
│   └── src/index.js
├── db/
│   └── init.sql        ← Schema + Seed Users
├── scripts/
│   └── gen-certs.sh    ← สร้าง self-signed cert
└── screenshots/        ← หลักฐานการทดสอบ 12 รูป
```

---

## 🔑 Seed Users

| Username | Email | Password | Role |
|---|---|---|---|
| alice | alice@lab.local | alice123 | member |
| bob | bob@lab.local | bob456 | member |
| admin | admin@lab.local | adminpass | admin |

> ไม่มีการ Register ใหม่ในระบบนี้ — ใช้เฉพาะบัญชีข้างต้นเท่านั้น

---

## 🚀 วิธีรันระบบ

### ขั้นตอนที่ 1 — สร้าง SSL Certificate

```bash
chmod +x scripts/gen-certs.sh
./scripts/gen-certs.sh
# ✅ Certificate จะถูกสร้างที่ nginx/certs/cert.pem และ nginx/certs/key.pem
```

### ขั้นตอนที่ 2 — สร้างไฟล์ .env

```bash
cp .env.example .env
# แก้ไขค่าตามต้องการ (โดยเฉพาะ JWT_SECRET ในงานจริง)
```

### ขั้นตอนที่ 3 — Build และรัน

```bash
docker compose up --build
```

### ขั้นตอนที่ 4 — เปิดใช้งาน

| URL | รายละเอียด |
|---|---|
| `https://localhost` | Task Board (มี cert warning → กด Advanced → Proceed) |
| `https://localhost/logs.html` | Log Dashboard (เฉพาะ admin) |
| `http://localhost` | จะ redirect ไป HTTPS อัตโนมัติ |

### รีเซ็ตฐานข้อมูล

```bash
docker compose down -v
docker compose up --build
```

---

## 🧪 วิธีทดสอบ API

ใช้ `-k` หรือ `--insecure` เพราะเป็น self-signed certificate

```bash
BASE="https://localhost"

# Login (เก็บ token)
TOKEN=$(curl -sk -X POST $BASE/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@lab.local","password":"alice123"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")

# สร้าง Task
curl -sk -X POST $BASE/api/tasks/ \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title":"ทดสอบระบบ","priority":"high"}'

# ดู Tasks
curl -sk $BASE/api/tasks/ -H "Authorization: Bearer $TOKEN"

# เรียกโดยไม่มี JWT → 401
curl -sk $BASE/api/tasks/

# Admin ดู Logs
ADMIN_TOKEN=$(curl -sk -X POST $BASE/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@lab.local","password":"adminpass"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")
curl -sk $BASE/api/logs/ -H "Authorization: Bearer $ADMIN_TOKEN"
```

---

## 🔐 HTTPS Flow

```
1. Browser ส่ง HTTP request มาที่ port 80
2. Nginx return 301 redirect ไป https://localhost
3. Browser ส่ง HTTPS request มาที่ port 443
4. Nginx ทำ TLS Handshake โดยใช้ self-signed certificate
   (Browser แสดง warning "Your connection is not private" — กด Advanced → Proceed)
5. Nginx decrypt traffic แล้ว proxy ต่อไปยัง service ปลายทาง
   ผ่าน HTTP ภายใน Docker network (services ไม่ต้องจัดการ TLS เอง)
6. Response ส่งกลับผ่าน Nginx → encrypt → ส่งกลับ Browser
```

**เหตุผลที่มี cert warning:** เป็น Self-Signed Certificate ที่สร้างเองด้วย OpenSSL ไม่ได้ผ่าน Certificate Authority (CA) สาธารณะ ในระบบ production จริงต้องใช้ cert จาก CA เช่น Let's Encrypt

---

## 🎫 JWT Flow

```
1. Client ส่ง POST /api/auth/login พร้อม email + password
2. Auth Service ตรวจสอบกับ DB โดยใช้ bcrypt.compare()
3. ถ้าถูกต้อง → สร้าง JWT token ด้วย jsonwebtoken.sign()
   Payload: { sub: userId, email, role, username }
   Secret: JWT_SECRET (จาก .env)
   หมดอายุ: 1 ชั่วโมง (JWT_EXPIRES=1h)
4. Client เก็บ token ใน localStorage
5. ทุก request ที่ต้องการสิทธิ์ → ส่ง Header: Authorization: Bearer <token>
6. Service ตรวจสอบด้วย jwt.verify(token, JWT_SECRET)
7. ถ้า token ไม่ถูกต้องหรือหมดอายุ → 401 Unauthorized
```

---

## 📝 Logging Flow

```
1. Auth Service / Task Service เกิด event (เช่น login สำเร็จ, สร้าง task)
2. เรียก logEvent() แบบ fire-and-forget (async, ไม่รอ response)
3. logEvent() ส่ง POST http://log-service:3003/api/logs/internal
   (เรียกภายใน Docker network — ไม่ผ่าน Nginx)
4. Log Service เก็บลง PostgreSQL ตาราง logs
5. Admin สามารถดู log ผ่าน GET /api/logs/ (ต้องมี JWT + role=admin)
6. Nginx block /api/logs/internal จากภายนอก → return 403
   (ป้องกันไม่ให้ภายนอกยิง log ปลอมเข้าระบบ)
```

**Log Events ที่บันทึก:**

| Event | Service | Level | เมื่อไหร่ |
|---|---|---|---|
| LOGIN_SUCCESS | auth-service | INFO | login ถูกต้อง |
| LOGIN_FAILED | auth-service | WARN | password ผิด |
| JWT_INVALID | task-service | ERROR | token ผิดหรือหมดอายุ |
| TASK_CREATED | task-service | INFO | สร้าง task สำเร็จ |
| TASK_DELETED | task-service | INFO | ลบ task |

---

## ⚠️ Known Limitations

- Self-Signed Certificate จะแสดง browser warning ในทุก browser — เป็นเรื่องปกติสำหรับ development
- ไม่มีระบบ Register — ใช้เฉพาะ Seed Users 3 บัญชีที่กำหนดไว้
- ใช้ Shared Database (1 PostgreSQL instance) — ในระบบ production จริงควรแยก DB ต่อ service
- JWT Secret เก็บใน .env — ต้องเปลี่ยนค่าและไม่ commit .env ลง Git ในงานจริง
- Log Service ไม่มีระบบ pagination ที่ frontend — แสดงสูงสุด 200 รายการ
- ไม่มี refresh token — เมื่อ JWT หมดอายุ (1h) ต้อง login ใหม่

---

## 📸 Screenshots

| ไฟล์ | รายการ |
|---|---|
| `01_docker_running.png` | docker compose up ทุก container healthy |
| `02_https_browser.png` | เข้า https://localhost ได้ (cert warning) |
| `03_login_success.png` | POST /api/auth/login → 200 + JWT |
| `04_login_fail.png` | POST /api/auth/login (ผิด password) → 401 |
| `05_create_task.png` | POST /api/tasks/ → 201 Created |
| `06_get_tasks.png` | GET /api/tasks/ → 200 + task list |
| `07_update_task.png` | PUT /api/tasks/:id → 200 Updated |
| `08_delete_task.png` | DELETE /api/tasks/:id → 200 Deleted |
| `09_no_jwt_401.png` | GET /api/tasks/ (ไม่มี JWT) → 401 |
| `10_logs_api.png` | GET /api/logs/ (admin) → 200 + logs |
| `11_rate_limit.png` | login ผิด > 5 ครั้ง/นาที → ส่ง 429 |
| `12_frontend_screenshot.png` | หน้า Task Board และ Log Dashboard |
