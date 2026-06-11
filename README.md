# 🏥 ADR-T System — ระบบติดตามอาการไม่พึงประสงค์จากยา

> ระบบบันทึก ประเมิน และวิเคราะห์ Adverse Drug Reactions (ADR) ในผู้ป่วยมะเร็ง  
> ตามมาตรฐาน **CTCAE (Common Terminology Criteria for Adverse Events)**  
> พัฒนาสำหรับ **ฝ่ายเภสัชกรรม โรงพยาบาลกรุงเทพสิริโรจน์**

---

## 🎛️ ฟีเจอร์หลักของระบบ

### 🔐 ระบบ Authentication
- เข้าสู่ระบบด้วย Username / Password (JWT-based)
- แบ่ง Role: `pharmacist`, `nurse`, `doctor` — มีสิทธิ์การใช้งานต่างกัน
- Token มีอายุ 8 ชั่วโมง และแนบ Authorization header อัตโนมัติทุก request
=======
- `pharmacist`, `nurse`, `doctor` = pass 54321
>>>>>>> f9e79d2 (update backend and schema)

### บันทึก ADR (Step-by-step Form)
- **Step 1** — ค้นหาผู้ป่วยจาก HN, เลือก Encounter (OPD/IPD), ระบุวันที่ / Cycle / Dose / รายการยา  
  ดึงรายชื่อยาจาก Drug Master (`GET /api/drugs`) มาแสดงเป็น dropdown ตอนโหลดหน้า  
  รองรับการเพิ่มยานอกบัญชีใหม่เข้า Drug Master ได้ทันที (`POST /api/drugs`)
- **Step 2** — เลือกอาการ ADR ตามมาตรฐาน CTCAE จัดกลุ่มตาม category พร้อมระบุ Grade 1–5  
  รองรับอาการ custom นอกเหนือจาก CTCAE มาตรฐาน
- บันทึก Recommendation, หมายเหตุ และวันนัดติดตาม

### Dashboard
- สรุปสถิติประจำเดือน: จำนวน Records, ADR Rate, Grade 3+ Events
- กราฟ Trend 6 เดือนย้อนหลัง และ Grade Distribution
- Top  5 อาการที่พบบ่อยในเดือนนั้น
- เปรียบเทียบกับเดือนก่อนและสถิติทั้งหมด (All-time)

### Records — รายการบันทึก ADR
- ค้นหา / กรองตาม HN, ชื่อผู้ป่วย, เดือน, Grade
- ดูรายละเอียด ADR ทุก symptom พร้อม Grade และคำอธิบาย
- แก้ไข / ลบ record (เฉพาะ pharmacist)

### Report — วิเคราะห์รายปี
- ดึงข้อมูลทั้งปีจาก `GET /api/records?year=YYYY` มาคำนวณทั้งหมดฝั่ง client
- **Dashboard tab** — กราฟ Monthly Count, Monthly Rate, Grade Distribution
- **Symptom Matrix tab** — ตารางความถี่ของแต่ละอาการแยกตามเดือน กรองตาม Grade ได้
- **Monthly Summary tab** — ตารางสรุปรายเดือน: Visits, ADR Events, ADR/Visit, G3+
- **Compare Regimen tab** — เปรียบเทียบสูตรยาสูงสุด 3 กลุ่ม
- Export CSV 
>>>>>>> f9e79d2 (update backend and schema)

---

## 🛠️ Stack เทคโนโลยี

| ส่วน        | เทคโนโลยี                                     |
|-------------|-----------------------------------------------|
| Frontend    | React 18, Vite, Tailwind CSS, IBM Plex Sans Thai |
| Backend     | Node.js, Express, CORS                        |
| Database    | PostgreSQL (Neon Serverless — ap-southeast-1) |
| Auth        | JWT (`jsonwebtoken`), bcrypt                  |
| HTTP Client | Fetch API (custom wrapper ใน `services/api.js`) |
| Charts      | Recharts                                      |

---

## 🚀 วิธีติดตั้งและรันระบบ

### 1. ติดตั้ง Dependencies

```bash
npm install
```

### 2. ตั้งค่าไฟล์ `.env`

สร้างไฟล์ `.env` ที่ root ของโปรเจกต์:

```env
# Neon PostgreSQL
PGHOST=your-neon-host.neon.tech
PGPORT=5432
PGDATABASE=neondb
PGUSER=neondb_owner
PGPASSWORD=your_password

# JWT
JWT_SECRET=your_secret_key
JWT_EXPIRES=8h

# Server
PORT=5000
```

### 3. รันระบบ (Frontend + Backend พร้อมกัน)

```bash
npm start
```

| บริการ   | URL                   | คำอธิบาย              |
|----------|-----------------------|-----------------------|
| Frontend | http://localhost:5173 | Vite dev server       |
| Backend  | http://localhost:5000 | Express API + Neon DB |

เมื่อพร้อมใช้งาน ควรเห็น log:
```
[0]   ➜  Local:   http://localhost:5173/
[1] ✅ Connected to Neon PostgreSQL
```

### หากต้องการรันแยก

```bash
npm run dev      # Frontend เท่านั้น
npm run server   # Backend เท่านั้น
```

### 4. เปิดเบราว์เซอร์

ไปที่ **http://localhost:5173** และเข้าสู่ระบบด้วย username / password ที่กำหนดไว้ใน table `users`

---

## 📁 โครงสร้างโปรเจกต์

```
├── src/services/server.js     # Express backend (API + DB)
├── .env                       # Environment variables (อย่า commit ขึ้น Git)
├── src/
│   ├── services/
│   │   └── api.js             # HTTP client / API service layer
│   ├── pages/
│   │   ├── Login.jsx          # หน้าเข้าสู่ระบบ
│   │   ├── Dashboard.jsx      # หน้าสรุปสถิติประจำเดือน
│   │   ├── Records.jsx        # รายการ ADR ทั้งหมด
│   │   ├── Recorddetail.jsx   # รายละเอียด ADR รายบุคคล
│   │   └── report.jsx         # รายงานวิเคราะห์รายปี
│   └── components/
│       ├── Step1.jsx          # ฟอร์ม Step 1 — ข้อมูลผู้ป่วย + ยา
│       ├── Step2.jsx          # ฟอร์ม Step 2 — CTCAE Symptoms
│       └── Reporttokens.jsx   # Components และ helpers สำหรับ Report
└── SCHEMA.md                  # Database schema & API reference
```

---

## 🔌 API Overview

Base URL: `http://localhost:5000/api`  
ทุก endpoint ต้องส่ง `Authorization: Bearer <token>` ยกเว้น `POST /api/auth/login`

| Method | Endpoint                           | Role       | คำอธิบาย                             |
|--------|------------------------------------|------------|--------------------------------------|
| POST   | `/api/auth/login`                  | —          | เข้าสู่ระบบ                           |
| GET    | `/api/auth/me`                     | ทุก role   | ดึงข้อมูล user ปัจจุบัน               |
| GET    | `/api/patients`                    | ทุก role   | ค้นหาผู้ป่วย                          |
| GET    | `/api/patients/:hn`                | ทุก role   | ดึงข้อมูลผู้ป่วยรายบุคคล              |
| POST   | `/api/patients`                    | pharmacist | เพิ่มผู้ป่วยใหม่                       |
| PUT    | `/api/patients/:hn`                | pharmacist | แก้ไขข้อมูลผู้ป่วย                    |
| GET    | `/api/drugs`                       | ทุก role   | ดึงรายการยาทั้งหมด (Drug Master)       |
| POST   | `/api/drugs`                       | ทุก role   | เพิ่มยานอกบัญชีใหม่                    |
| GET    | `/api/ctcae`                       | ทุก role   | ดึง CTCAE จัดกลุ่มตาม category        |
| GET    | `/api/ctcae/terms`                 | ทุก role   | ดึง CTCAE term list แบบ flat          |
| POST   | `/api/encounters`                  | ทุก role   | สร้าง OPD/IPD encounter               |
| GET    | `/api/encounters`                  | ทุก role   | ดึง encounter ของผู้ป่วย              |
| GET    | `/api/records`                     | ทุก role   | ดึง ADR records (filter ได้)          |
| GET    | `/api/records/:id`                 | ทุก role   | ดึง ADR record รายบุคคล               |
| POST   | `/api/records`                     | ทุก role   | บันทึก ADR ใหม่                        |
| PUT    | `/api/records/:id`                 | pharmacist | แก้ไข ADR record                      |
| DELETE | `/api/records/:id`                 | pharmacist | ลบ ADR record                         |
| GET    | `/api/stats`                       | ทุก role   | สถิติ Dashboard ประจำเดือน             |
| POST   | `/api/records/backfill-encounters` | pharmacist | แก้ record เก่าที่ encounter_id = NULL |

> ดูรายละเอียด request/response body, schema และ ER Diagram ทั้งหมดได้ที่ **[SCHEMA.md](./SCHEMA.md)**

---

## 👤 Default Users

> ตั้งค่า user เริ่มต้นได้ใน table `users` ผ่าน Neon SQL Editor

| Username     | Role       | คำอธิบาย                                             |
|--------------|------------|------------------------------------------------------|
| pharmacist01 | pharmacist | เภสัชกร — สิทธิ์เต็ม รวมถึงลบและจัดการข้อมูลผู้ป่วย  |
| nurse01      | nurse      | พยาบาล — บันทึก ADR และดูรายงานได้                    |
| doctor01     | doctor     | แพทย์ — สิทธิ์เทียบเท่าพยาบาลทุกประการ                |

Password จะถูก hash ด้วย bcrypt ก่อนบันทึก

**SQL สำหรับเพิ่ม user ใหม่ใน Neon:**
```sql
INSERT INTO users (username, password_hash, role, name, title, is_active)
VALUES (
  'doctor01',
  crypt('your_password', gen_salt('bf')),
  'doctor',
  'นพ.ชื่อ นามสกุล',
  'แพทย์',
  true
);
```

---

## 📋 Database & Schema

ระบบใช้ **PostgreSQL บน Neon** ประกอบด้วย 10 ตารางหลัก:

`patients` · `users` · `encounters` · `adr_records` · `adr_record_drugs`  
`adr_symptoms` · `ctcae_categories` · `ctcae_terms` · `ctcae_grade_descriptions` · `drugs`

และ 1 View: `view_adr_summary` — ใช้ query ข้อมูลหน้า Records, Dashboard และ Report

> `users.role` รองรับ 3 ค่า: `pharmacist` / `nurse` / `doctor`  
> ดูโครงสร้างตาราง, column types, relation และ ER Diagram ได้ที่ **[SCHEMA.md](./SCHEMA.md)**
---
<<<<<<< HEAD
> 📌 v2.0.0 · ADR-T System · ฝ่ายเภสัชกรรม โรงพยาบาลกรุงเทพสิริโรจน์  
> ข้อมูลผู้ป่วยเป็นความลับตาม พ.ร.บ. สุขภาพแห่งชาติ
