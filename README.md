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

### บันทึก ADR (Step-by-step Form)
- **Step 1** — ค้นหาผู้ป่วยจาก HN, เลือก Encounter (OPD/IPD), ระบุวันที่ / Cycle / Dose / รายการยา  
  ดึงรายชื่อยาจาก Drug Master (`GET /api/drugs`) มาแสดงเป็น dropdown ตอนโหลดหน้า  
  รองรับการเพิ่มยานอกบัญชีใหม่เข้า Drug Master ได้ทันที (`POST /api/drugs`)
- **Step 2** — เลือกอาการ ADR ตามมาตรฐาน CTCAE จัดกลุ่มตาม category พร้อมระบุ Grade 1–5  
  รองรับอาการ custom นอกเหนือจาก CTCAE มาตรฐาน

### Dashboard
- สรุปสถิติประจำเดือน: จำนวน Records, ADR Rate, Grade 3+ Events
- กราฟ Trend 6 เดือนย้อนหลัง และ Grade Distribution
- Top 5 อาการที่พบบ่อยในเดือนนั้น
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

---

## 🛠️ Stack เทคโนโลยี

| ส่วน        | เทคโนโลยี                                        |
|-------------|--------------------------------------------------|
| Frontend    | React 18, Vite, Tailwind CSS, IBM Plex Sans Thai |
| Backend     | Node.js, Express, CORS                           |
| Database    | MySQL (Local — เชื่อมต่อผ่าน mysql2)             |
| Auth        | JWT (`jsonwebtoken`), bcrypt                     |
| HTTP Client | Fetch API (custom wrapper ใน `services/api.js`)  |
| Charts      | Recharts                                         |

---

## 🚀 วิธีติดตั้งและรันระบบ

### 1. ติดตั้ง Dependencies

```bash
npm install
```

### 2. สร้าง Database และ Tables

```bash
mysql -u root -p < schema_mysql.sql
```

### 3. ตั้งค่าไฟล์ `.env`

สร้างไฟล์ `.env` ที่ root ของโปรเจกต์:

```env
DB_HOST=localhost
DB_PORT=3306
DB_NAME=adrt_db
DB_USER=root
DB_PASSWORD=your_password_here

JWT_SECRET=adr-secret-key-change-in-production
JWT_EXPIRES=8h
PORT=5000
```

### 4. สร้าง Password Hash สำหรับ Users

```bash
node -e "require('bcrypt').hash('your_password', 10).then(console.log)"
```

แล้ว UPDATE ลงตาราง `users`:

```sql
UPDATE users SET password_hash = '$2b$10$...' WHERE username = 'pharmacist1';
```

### 5. รันระบบ (Frontend + Backend พร้อมกัน)

```bash
npm start
```

| บริการ   | URL                   | คำอธิบาย        |
|----------|-----------------------|-----------------|
| Frontend | http://localhost:5173 | Vite dev server |
| Backend  | http://localhost:5000 | Express API     |

เมื่อพร้อมใช้งาน ควรเห็น log:

```
[0]   ➜  Local:   http://localhost:5173/
[1] ✅ Connected to MySQL Local
```

### หากต้องการรันแยก

```bash
npm run dev      # Frontend เท่านั้น
npm run server   # Backend เท่านั้น
```

---

## 🔍 DEV MODE (ดูหน้าต่างๆ โดยไม่ต้อง Login)

ใช้ในกรณีที่ยังไม่ได้เชื่อมต่อฐานข้อมูล หรือต้องการดู UI:

```
http://localhost:5173/?dev=1&page=dashboard
http://localhost:5173/?dev=1&page=records
http://localhost:5173/?dev=1&page=assessment
http://localhost:5173/?dev=1&page=reports
```

> ⚠️ ลบ `?dev=1` ออกจาก URL ก่อน deploy จริง  
> หน้าที่ดึงข้อมูลจาก API จะแสดง error banner เพราะ DB ยังไม่พร้อม — ดูได้เฉพาะ layout/UI

---

## 📁 โครงสร้างโปรเจกต์

```
├── server.js                  # Express backend (API + MySQL)
├── schema_mysql.sql           # SQL สำหรับสร้าง Database + Tables + View
├── .env                       # Environment variables (อย่า commit ขึ้น Git)
└── src/
    ├── services/
    │   └── api.js             # HTTP client / API service layer
    ├── pages/
    │   ├── Login.jsx          # หน้าเข้าสู่ระบบ
    │   ├── Dashboard.jsx      # หน้าสรุปสถิติประจำเดือน
    │   ├── Records.jsx        # รายการ ADR ทั้งหมด
    │   ├── Recorddetail.jsx   # รายละเอียด ADR รายบุคคล (popup)
    │   ├── Step1.jsx          # ฟอร์ม Step 1 — ข้อมูลผู้ป่วย + ยา
    │   ├── Step2.jsx          # ฟอร์ม Step 2 — CTCAE Symptoms
    │   └── report.jsx         # รายงานวิเคราะห์รายปี
    ├── components/
    │   ├── Sidebar.jsx
    │   ├── Topbar.jsx
    │   └── ReportTokens.jsx   # Components และ helpers สำหรับ Report
    └── App.jsx                # Root component + routing
```

---

## 🔌 API Overview

Base URL: `http://localhost:5000/api`  
ทุก endpoint ต้องส่ง `Authorization: Bearer <token>` ยกเว้น `POST /api/auth/login`

| Method | Endpoint                           | Role       | คำอธิบาย                              |
|--------|------------------------------------|------------|---------------------------------------|
| POST   | `/api/auth/login`                  | —          | เข้าสู่ระบบ                            |
| GET    | `/api/auth/me`                     | ทุก role   | ดึงข้อมูล user ปัจจุบัน                |
| GET    | `/api/patients`                    | ทุก role   | ค้นหาผู้ป่วย                           |
| GET    | `/api/patients/:hn`                | ทุก role   | ดึงข้อมูลผู้ป่วยรายบุคคล               |
| POST   | `/api/patients`                    | pharmacist | เพิ่มผู้ป่วยใหม่                        |
| PUT    | `/api/patients/:hn`                | pharmacist | แก้ไขข้อมูลผู้ป่วย                     |
| GET    | `/api/drugs`                       | ทุก role   | ดึงรายการยาทั้งหมด (Drug Master)        |
| POST   | `/api/drugs`                       | pharmacist | เพิ่มยานอกบัญชีใหม่                     |
| GET    | `/api/ctcae`                       | ทุก role   | ดึง CTCAE จัดกลุ่มตาม category         |
| GET    | `/api/ctcae/terms`                 | ทุก role   | ดึง CTCAE term list แบบ flat           |
| POST   | `/api/encounters`                  | ทุก role   | สร้าง OPD/IPD encounter                |
| GET    | `/api/encounters`                  | ทุก role   | ดึง encounter ของผู้ป่วย               |
| GET    | `/api/records`                     | ทุก role   | ดึง ADR records (filter ได้)           |
| GET    | `/api/records/:id`                 | ทุก role   | ดึง ADR record รายบุคคล                |
| POST   | `/api/records`                     | ทุก role   | บันทึก ADR ใหม่                         |
| PUT    | `/api/records/:id`                 | pharmacist | แก้ไข ADR record                       |
| DELETE | `/api/records/:id`                 | pharmacist | ลบ ADR record                          |
| GET    | `/api/stats`                       | ทุก role   | สถิติ Dashboard ประจำเดือน              |
| POST   | `/api/records/backfill-encounters` | pharmacist | แก้ record เก่าที่ encounter_id = NULL  |

---

## 👤 Default Users

| Username     | Role       | คำอธิบาย                                              |
|--------------|------------|-------------------------------------------------------|
| pharmacist1  | pharmacist | เภสัชกร — สิทธิ์เต็ม รวมถึงลบและจัดการข้อมูลผู้ป่วย   |
| nurse1       | nurse      | พยาบาล — บันทึก ADR และดูรายงานได้                     |
| doctor1      | doctor     | แพทย์ — สิทธิ์เทียบเท่าพยาบาลทุกประการ                 |

Password hash ด้วย bcrypt ก่อนบันทึกลง DB เสมอ

---

## 📋 Database Schema

ระบบใช้ **MySQL Local** ประกอบด้วย 10 ตารางหลัก:

`patients` · `users` · `encounters` · `adr_records` · `adr_record_drugs`  
`adr_symptoms` · `ctcae_categories` · `ctcae_terms` · `ctcae_grade_descriptions` · `drugs`

และ 1 View: `view_adr_summary` — ใช้ query ข้อมูลหน้า Records, Dashboard และ Report

> `users.role` รองรับ 3 ค่า: `pharmacist` / `nurse` / `doctor`  
> ดูโครงสร้างตาราง, column types, relation ทั้งหมดได้ที่ **`schema_mysql.sql`**

---

> 📌 v2.0.0 · ADR-T System · ฝ่ายเภสัชกรรม โรงพยาบาลกรุงเทพสิริโรจน์  
> ข้อมูลผู้ป่วยเป็นความลับตาม พ.ร.บ. สุขภาพแห่งชาติ