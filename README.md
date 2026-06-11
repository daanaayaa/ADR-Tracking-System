# 🏥 ADR-T System — ระบบติดตามอาการไม่พึงประสงค์จากยา

> ระบบบันทึก ประเมิน และวิเคราะห์ Adverse Drug Reactions (ADR) ในผู้ป่วยมะเร็ง  
> ตามมาตรฐาน **CTCAE (Common Terminology Criteria for Adverse Events)**  
> พัฒนาสำหรับ **ฝ่ายเภสัชกรรม โรงพยาบาลกรุงเทพสิริโรจน์**

---

## 🎛️ ฟีเจอร์หลักของระบบ

### 🔐 ระบบ Authentication
- เข้าสู่ระบบด้วย Username / Password (JWT-based)
- แบ่ง Role: `pharmacist`, `nurse` และ `doctor` — มีสิทธิ์การใช้งานต่างกัน
- Token มีอายุ 8 ชั่วโมง และแนบ Authorization header อัตโนมัติทุก request
- `pharmacist`, `nurse` และ `doctor01`  PASS = 54321


### 📋 บันทึก ADR (Step-by-step Form)
- **Step 1** — ค้นหาผู้ป่วยจาก HN, เลือก Encounter (OPD/IPD), ระบุวันที่ / Cycle / Dose / รายการยา  
  รองรับการเพิ่มยานอกบัญชีใหม่เข้า Drug Master ได้ทันที
- **Step 2** — เลือกอาการ ADR ตามมาตรฐาน CTCAE จัดกลุ่มตาม category พร้อมระบุ Grade 1–5  
  รองรับอาการ custom นอกเหนือจาก CTCAE มาตรฐาน
- บันทึก Recommendation, หมายเหตุ และวันนัดติดตาม

### 📊 Dashboard
- สรุปสถิติประจำเดือน: จำนวน Records, ADR Rate, Grade 3+ Events
- กราฟ Trend 6 เดือนย้อนหลัง และ Grade Distribution
- Top 5 อาการที่พบบ่อยในเดือนนั้น
- เปรียบเทียบกับเดือนก่อนและสถิติทั้งหมด (All-time)

### 🗂️ Records — รายการบันทึก ADR
- ค้นหา / กรองตาม HN, ชื่อผู้ป่วย, เดือน, Grade
- ดูรายละเอียด ADR ทุก symptom พร้อม Grade และคำอธิบาย
- แก้ไข / ลบ record (เฉพาะ pharmacist)

### 📈 Report — รายงานประจำปี
- Symptom Matrix แสดงความถี่ของแต่ละอาการแยกตามเดือน
- กรองตาม Grade และปีที่ต้องการ
- Export excel

---

## 🛠️ Stack เทคโนโลยี

| ส่วน       | เทคโนโลยี                                      |
|------------|------------------------------------------------|
| Frontend   | React 18, Vite, IBM Plex Sans Thai             |
| Backend    | Node.js, Express, CORS                         |
| Database   | PostgreSQL (Neon Serverless — ap-southeast-1)  |
| Auth       | JWT (`jsonwebtoken`), bcrypt                   |
| HTTP Client| Fetch API (custom wrapper ใน `services/api.js`) |

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

คำสั่งนี้จะรัน **Frontend และ Backend พร้อมกันในครั้งเดียว** ผ่าน `concurrently`:

| บริการ   | URL                       | คำอธิบาย                    |
|----------|---------------------------|-----------------------------|
| Frontend | http://localhost:5173     | Vite dev server             |
| Backend  | http://localhost:5000     | Express API + Neon DB       |

เมื่อพร้อมใช้งาน ควรเห็น log ดังนี้:
```
[0]   ➜  Local:   http://localhost:5173/
[1] ✅ Connected to Neon PostgreSQL
```

> ⚠️ ต้องมีไฟล์ `.env` ที่ถูกต้องก่อนรัน มิฉะนั้น backend จะเชื่อมต่อ Neon ไม่ได้

### หากต้องการรันแยก

```bash
npm run dev      # Frontend เท่านั้น
npm run server   # Backend เท่านั้น  (node src/services/server.js)
```

### 4. เปิดเบราว์เซอร์

ไปที่ **http://localhost:5173** และเข้าสู่ระบบด้วย username / password ที่กำหนดไว้ใน table `users`

---

## 📁 โครงสร้างโปรเจกต์

```
├── src/services/server.js # Express backend (API + DB)
├── .env                   # Environment variables (อย่า commit ขึ้น Git)
├── src/
│   ├── services/
│   │   └── api.js         # HTTP client / API service layer
│   ├── pages/
│   │   ├── Login.jsx      # หน้าเข้าสู่ระบบ
│   │   ├── Dashboard.jsx  # หน้าสรุปสถิติ
│   │   ├── Records.jsx    # รายการ ADR ทั้งหมด
│   │   ├── Recorddetail.jsx # รายละเอียด ADR รายบุคคล
│   │   └── report.jsx     # รายงานประจำปี
│   ├── components/
│   │   ├── Step1.jsx      # ฟอร์ม Step 1 — ข้อมูลผู้ป่วย + ยา
│   │   ├── Step2.jsx      # ฟอร์ม Step 2 — CTCAE Symptoms
│   │   └── Reporttokens.jsx # Token/badge สำหรับ Report
│   └── ...
└── SCHEMA.md              # Database schema & API reference
```

---

## 🔌 API Overview

Base URL: `http://localhost:5000/api`  
ทุก endpoint ต้องส่ง `Authorization: Bearer <token>` ยกเว้น `/api/auth/login`

| Method | Endpoint              | คำอธิบาย                        |
|--------|-----------------------|---------------------------------|
| POST   | `/api/auth/login`     | เข้าสู่ระบบ                     |
| GET    | `/api/patients`       | ค้นหาผู้ป่วย                    |
| POST   | `/api/encounters`     | สร้าง OPD/IPD encounter         |
| GET    | `/api/records`        | ดึง ADR records (filter ได้)   |
| POST   | `/api/records`        | บันทึก ADR ใหม่                 |
| GET    | `/api/ctcae`          | ดึงข้อมูล CTCAE ตาม category    |
| GET    | `/api/stats`          | สถิติ Dashboard ประจำเดือน      |
| GET    | `/api/report/symptoms`| Symptom matrix สำหรับรายงาน    |

> ดูรายละเอียด request/response body, schema และ ER Diagram ทั้งหมดได้ที่ **[SCHEMA.md](./SCHEMA.md)**

---

## 👤 Default Users

> ตั้งค่า user เริ่มต้นได้ใน table `users` ผ่าน Neon SQL Editor

| Username      | Role        | คำอธิบาย                                            |
|---------------|-------------|-----------------------------------------------------|
| pharmacist01  | pharmacist  | เภสัชกร — สิทธิ์เต็ม (รวมถึงลบ และจัดการข้อมูลผู้ป่วย) |
| nurse01       | nurse       | พยาบาล — บันทึก ADR และดูรายงานได้                   |
| doctor01      | doctor      | แพทย์ — บันทึก ADR และดูรายงานได้ (สิทธิ์เทียบเท่าพยาบาล) |

> **หมายเหตุ**: `doctor` มีสิทธิ์เหมือน `nurse` ทุกประการ — ลบ Record และแก้ไขข้อมูลผู้ป่วยไม่ได้  
> Password จะถูก hash ด้วย bcrypt ก่อนบันทึก

**SQL สำหรับเพิ่ม doctor ใน Neon:**
```sql
INSERT INTO users (username, password_hash, role, name, title, is_active)
VALUES ('doctor01', crypt('your_password', gen_salt('bf')), 'doctor', 'นพ.ชื่อ นามสกุล', 'แพทย์', true);
```

---

## 📋 Database & Schema

ระบบใช้ **PostgreSQL บน Neon** ประกอบด้วย 10 ตารางหลัก:

`patients` · `users` · `encounters` · `adr_records` · `adr_record_drugs`  
`adr_symptoms` · `ctcae_categories` · `ctcae_terms` · `ctcae_grade_descriptions` · `drugs`

และ 1 View: `view_adr_summary` — ใช้ query ข้อมูลหน้า Records และ Dashboard

> `users.role` รองรับ 3 ค่า: `pharmacist` / `nurse` / `doctor`  
> ดูโครงสร้างตาราง, column types, relation และ ER Diagram ได้ที่ **[SCHEMA.md](./SCHEMA.md)**

---

> 📌 v2.0.0 · ADR-T System · ฝ่ายเภสัชกรรม โรงพยาบาลกรุงเทพสิริโรจน์  
> ข้อมูลผู้ป่วยเป็นความลับตาม พ.ร.บ. สุขภาพแห่งชาติ
