# Final Lab Sec2 Set 2 — Microservices + Activity Tracking + Cloud

**วิชา:** ENGSE207 Software Architecture
**มหาวิทยาลัย:** มหาวิทยาลัยเทคโนโลยีราชมงคลล้านนา
**ผู้สอน:** อาจารย์ธนิต เกตุแก้ว

## 👥 รายชื่อสมาชิก

| รหัสนักศึกษา | ชื่อ-นามสกุล | บทบาท |
|---|---|---|
| 67543210042-7 | นายพีรพงศ์ ปัญญาสัน | Backend Developer (Auth Service) |
| 67543210068-2 | นายณัฐพงศ์ จันทร์สิงห์ | Backend Developer (Task + Activity) + Frontend |



---

## 🌐 Railway Service URLs

| Service | URL |
|---|---|
| Auth Service | https://auth-service-production-3780.up.railway.app |
| Task Service | https://task-service-production-168f.up.railway.app |
| Activity Service | https://activity-service-production-6e05.up.railway.app |

---

## 📌 ภาพรวม

Final Lab Set 2 ต่อยอดจาก Set 1 โดยเพิ่ม 3 สิ่งหลัก:
- **Register API** — ผู้ใช้สมัครสมาชิกใหม่ได้
- **Activity Service** — บันทึก events ที่มีความหมายต่อผู้ใช้
- **Deploy บน Railway** — ทุก service ขึ้น Cloud พร้อม HTTPS อัตโนมัติ

---

## 🏗️ Architecture Diagram

```
Browser / Postman
        │
        │ HTTPS (Railway จัดการให้อัตโนมัติ)
        ▼
┌─────────────────────────────────────────────────────────┐
│                    Railway Project                       │
│                                                         │
│  Auth Service          Task Service    Activity Service  │
│  :3001                 :3002           :3003             │
│                                                         │
│  POST /register        CRUD Tasks      POST /internal   │
│  POST /login    ──────────────────────▶ GET /me         │
│  GET  /me       logActivity()          GET /all (admin) │
│  logActivity()  logActivity()                           │
│       │               │                    ▲            │
│       └───────────────┴────────────────────┘            │
│       │               │                    │            │
│       ▼               ▼                    ▼            │
│   auth-db         task-db           activity-db         │
│   users, logs     tasks, logs       activities          │
└─────────────────────────────────────────────────────────┘

Frontend (config.js เรียกแต่ละ service โดยตรง)
```

---

## 🗄️ Database-per-Service Pattern

Set 2 แยก Database ตาม service ทำให้แต่ละ service เป็นอิสระจากกัน

| Service | Database | Tables |
|---|---|---|
| auth-service | auth-db | users, logs |
| task-service | task-db | tasks, logs |
| activity-service | activity-db | activities |

---

## 📝 Denormalization ใน activities table

activities table เก็บ `username` ไว้ด้วย ทั้งที่รู้ `user_id` อยู่แล้ว เพราะ activity-db ไม่มี users table ถ้าไม่ denormalize ต้อง query ข้าม 2 databases ซึ่งทำไม่ได้ใน Database-per-Service pattern จึงเก็บ username ไว้ ณ เวลาที่ event เกิดขึ้นเลย

---

## 🔥 Fire-and-Forget Pattern

fire-and-forget คือการส่ง request แบบไม่รอ response กลับมา

```javascript
logActivity({ ... }).catch(() => {})
```

ใช้ `.catch(() => {})` ต่อท้ายเสมอ เพื่อให้ถ้า Activity Service ล่ม Auth/Task Service ยังทำงานได้ปกติ เพียงแต่ activity จะไม่ถูกบันทึกชั่วคราว เลือก pattern นี้เพราะ activity เป็นข้อมูลเสริม ไม่ใช่ core business logic

---

## 🌐 Gateway Strategy

เลือก **Option A: Frontend เรียก URL ของแต่ละ service โดยตรง** ผ่าน `config.js`

```javascript
window.APP_CONFIG = {
  AUTH_URL:     'https://auth-service-production-3780.up.railway.app',
  TASK_URL:     'https://task-service-production-168f.up.railway.app',
  ACTIVITY_URL: 'https://activity-service-production-6e05.up.railway.app'
};
```

เหตุผล: ไม่ต้องตั้งค่า Nginx หรือ Gateway เพิ่ม Railway จัดการ HTTPS ให้ทุก service อัตโนมัติ แก้ URL ได้ที่ config.js ไฟล์เดียว

---

## 🚀 วิธีรัน Local

```bash
# 1. Clone repo
git clone https://github.com/Peerapong67/final-lab-set2.git
cd final-lab-set2

# 2. สร้าง .env
cp .env.example .env

# 3. รัน
docker compose up --build

# 4. ทดสอบ
curl http://localhost:3001/api/auth/health
curl http://localhost:3002/api/tasks/health
curl http://localhost:3003/api/activity/health
```

---

## ⚙️ Environment Variables

### auth-service
```
DATABASE_URL         = postgresql://...
JWT_SECRET           = your-sec2-shared-secret-here
JWT_EXPIRES          = 1h
PORT                 = 3001
NODE_ENV             = production
ACTIVITY_SERVICE_URL = https://activity-service-production-6e05.up.railway.app
```

### task-service
```
DATABASE_URL         = postgresql://...
JWT_SECRET           = your-sec2-shared-secret-here
PORT                 = 3002
NODE_ENV             = production
ACTIVITY_SERVICE_URL = https://activity-service-production-6e05.up.railway.app
```

### activity-service
```
DATABASE_URL = postgresql://...
JWT_SECRET   = your-sec2-shared-secret-here
PORT         = 3003
NODE_ENV     = production
```

---

## 🧪 วิธีทดสอบ (Cloud URLs)

```bash
AUTH_URL="https://auth-service-production-3780.up.railway.app"
TASK_URL="https://task-service-production-168f.up.railway.app"
ACTIVITY_URL="https://activity-service-production-6e05.up.railway.app"

# Register
curl -X POST $AUTH_URL/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"testuser","email":"test@test.com","password":"123456"}'

# Login
TOKEN=$(curl -s -X POST $AUTH_URL/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@test.com","password":"123456"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")

# ตรวจ activity
curl $ACTIVITY_URL/api/activity/me -H "Authorization: Bearer $TOKEN"

# Create task
curl -X POST $TASK_URL/api/tasks \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title":"test task","priority":"high"}'
```

---

## ⚠️ Known Limitations

- Frontend เป็น static files ต้องเปิดผ่าน Live Server หรือ deploy แยก
- JWT หมดอายุใน 1 ชั่วโมง ต้อง login ใหม่
- ไม่มี Refresh Token
- Activity Service ล่ม → activity ไม่ถูกบันทึก แต่ระบบยังทำงานได้ (fire-and-forget)
- Database-per-Service ทำให้ไม่สามารถ JOIN ข้าม DB ได้ต้องใช้ Denormalization แทน