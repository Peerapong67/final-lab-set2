# TEAM_SPLIT.md — Final Lab Sec2 Set 2

## ข้อมูลกลุ่ม

**วิชา:** ENGSE207 Software Architecture
**งาน:** Final Lab Sec2 Set 2 — Microservices + Activity Tracking + Cloud

## รายชื่อสมาชิก

| รหัสนักศึกษา | ชื่อ-นามสกุล | บทบาท |
|---|---|---|
| 67543210042-7 | นายพีรพงศ์ ปัญญาสัน | Backend Developer |
| 67543210068-2 | นายณัฐพงศ์ จันทร์สิงห์ | Backend Developer + Frontend |

---

## การแบ่งงาน

### สมาชิกที่ 1 — นายพีรพงศ์ ปัญญาสัน (67543210042-7) — Backend (Auth)

**auth-service/**
- เพิ่ม `POST /api/auth/register`
- เพิ่ม `logToDB()` บันทึก log ลง auth-db
- เพิ่ม `logActivity()` ส่ง event ไป activity-service
- ปรับ `POST /api/auth/login` เพิ่ม logActivity()
- ปรับ `db.js` ใช้ DATABASE_URL
- `init.sql` — auth-db schema + seed users

**Deploy บน Railway**
- Deploy auth-service + auth-db
- ตั้งค่า Environment Variables
- อัปเดต ACTIVITY_SERVICE_URL

---

### สมาชิกที่ 2 — นายณัฐพงศ์ จันทร์สิงห์ (67543210068-2) — Backend (Task + Activity) + Frontend

**task-service/**
- เพิ่ม `logToDB()` บันทึก log ลง task-db
- เพิ่ม `logActivity()` ใน POST (TASK_CREATED)
- เพิ่ม `logActivity()` ใน PUT (TASK_STATUS_CHANGED)
- เพิ่ม `logActivity()` ใน DELETE (TASK_DELETED)
- `init.sql` — task-db schema

**activity-service/** (สร้างใหม่ทั้งหมด)
- `POST /api/activity/internal` — รับ event จาก services
- `GET /api/activity/me` — ดู activities ของตัวเอง
- `GET /api/activity/all` — ดู activities ทั้งหมด (admin)
- `init.sql` — activities table

**frontend/**
- `config.js` — Railway Service URLs
- `index.html` — เพิ่ม Register tab + Activity link
- `activity.html` — Activity Timeline หน้าใหม่

**Deploy บน Railway**
- Deploy activity-service + activity-db (ก่อน)
- Deploy task-service + task-db

---

## งานที่ทำร่วมกัน

| งาน | รับผิดชอบ |
|---|---|
| `docker-compose.yml` | ทั้งคู่ |
| `.env.example` | ทั้งคู่ |
| `README.md` | ทั้งคู่ |
| End-to-end testing บน Cloud | ทั้งคู่ |
| Screenshots 14 รูป | ทั้งคู่ |

---

## Integration Notes

**JWT_SECRET** ต้องเหมือนกันทุก service — ใช้ค่าเดียวกันใน auth, task, activity

**ACTIVITY_SERVICE_URL** ต้องตั้งค่าใน auth-service และ task-service ให้ชี้ไปที่ activity-service URL จริงบน Railway

**Fire-and-Forget** — logActivity() ใช้ `.catch(() => {})` ต่อท้ายเสมอ ถ้า activity-service ล่ม service อื่นยังทำงานได้

**ลำดับ Deploy** — deploy activity-service ก่อนเสมอ เพราะ auth และ task ต้องการ URL ของ activity

---

## Git Branching

```
main
├── feat/auth-service      (สมาชิกที่ 1)
├── feat/task-service      (สมาชิกที่ 2)
├── feat/activity-service  (สมาชิกที่ 2)
└── feat/frontend          (สมาชิกที่ 2)
```