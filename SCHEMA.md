-- ═══════════════════════════════════════════════════════════════
--  ADR-T System — MySQL Local Schema
--  โรงพยาบาลกรุงเทพสิริโรจน์ · ฝ่ายเภสัชกรรม
--  เทียบเท่า Neon PostgreSQL schema ทุก table + view
-- ═══════════════════════════════════════════════════════════════

CREATE DATABASE IF NOT EXISTS adrt_db
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

USE adrt_db;

-- ─────────────────────────────────────────────
--  USERS
-- ─────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS users (
  id            INT           NOT NULL AUTO_INCREMENT PRIMARY KEY,
  username      VARCHAR(100)  NOT NULL UNIQUE,
  password_hash VARCHAR(255)  NOT NULL,
  role          ENUM('pharmacist','doctor','nurse') NOT NULL DEFAULT 'nurse',
  name          VARCHAR(200)  NOT NULL,
  title         VARCHAR(100),
  is_active     TINYINT(1)    NOT NULL DEFAULT 1,
  created_at    DATETIME      NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at    DATETIME      NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- ─────────────────────────────────────────────
--  PATIENTS
-- ─────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS patients (
  id            INT           NOT NULL AUTO_INCREMENT PRIMARY KEY,
  hn            VARCHAR(50)   NOT NULL UNIQUE,
  patient_name  VARCHAR(300)  NOT NULL,
  age           INT,
  gender        ENUM('M','F','Other'),
  weight        DECIMAL(6,2),
  height        DECIMAL(6,2),
  diagnosis     TEXT,
  created_at    DATETIME      NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at    DATETIME      NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  INDEX idx_patients_hn (hn),
  INDEX idx_patients_name (patient_name)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- ─────────────────────────────────────────────
--  DRUGS
-- ─────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS drugs (
  id         INT          NOT NULL AUTO_INCREMENT PRIMARY KEY,
  name       VARCHAR(300) NOT NULL UNIQUE,
  is_active  TINYINT(1)   NOT NULL DEFAULT 1,
  created_at DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- ─────────────────────────────────────────────
--  CTCAE CATEGORIES
-- ─────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS ctcae_categories (
  id         INT          NOT NULL AUTO_INCREMENT PRIMARY KEY,
  category   VARCHAR(200) NOT NULL UNIQUE,
  sort_order INT          NOT NULL DEFAULT 0
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- ─────────────────────────────────────────────
--  CTCAE TERMS
-- ─────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS ctcae_terms (
  id          INT          NOT NULL AUTO_INCREMENT PRIMARY KEY,
  category_id INT          NOT NULL,
  `key`       VARCHAR(200) NOT NULL UNIQUE,
  label       VARCHAR(300) NOT NULL,
  is_custom   TINYINT(1)   NOT NULL DEFAULT 0,
  created_at  DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (category_id) REFERENCES ctcae_categories(id),
  INDEX idx_ctcae_key (`key`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- ─────────────────────────────────────────────
--  CTCAE GRADE DESCRIPTIONS
-- ─────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS ctcae_grade_descriptions (
  id          INT   NOT NULL AUTO_INCREMENT PRIMARY KEY,
  term_id     INT   NOT NULL,
  grade       TINYINT NOT NULL,
  description TEXT,
  FOREIGN KEY (term_id) REFERENCES ctcae_terms(id) ON DELETE CASCADE,
  UNIQUE KEY uq_term_grade (term_id, grade)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- ─────────────────────────────────────────────
--  ENCOUNTERS
-- ─────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS encounters (
  id          INT         NOT NULL AUTO_INCREMENT PRIMARY KEY,
  hn          VARCHAR(50) NOT NULL,
  type        ENUM('OPD','IPD') NOT NULL,
  vn          VARCHAR(100),
  an          VARCHAR(100),
  visit_date  DATE        NOT NULL,
  created_by  INT,
  created_at  DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (hn) REFERENCES patients(hn),
  FOREIGN KEY (created_by) REFERENCES users(id),
  INDEX idx_enc_hn (hn),
  INDEX idx_enc_date (visit_date)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- ─────────────────────────────────────────────
--  ADR RECORDS
-- ─────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS adr_records (
  id              INT         NOT NULL AUTO_INCREMENT PRIMARY KEY,
  encounter_id    INT,
  hn              VARCHAR(50) NOT NULL,
  record_date     DATE        NOT NULL,
  cycle           INT,
  dose            DECIMAL(10,4),
  dose_unit       VARCHAR(50),
  note            TEXT,
  recommendation  TEXT,
  follow_up_date  DATE,
  created_by      INT,
  updated_by      INT,
  created_at      DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at      DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  FOREIGN KEY (encounter_id) REFERENCES encounters(id) ON DELETE SET NULL,
  FOREIGN KEY (hn) REFERENCES patients(hn),
  FOREIGN KEY (created_by) REFERENCES users(id),
  INDEX idx_rec_hn (hn),
  INDEX idx_rec_date (record_date)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- ─────────────────────────────────────────────
--  ADR RECORD DRUGS
-- ─────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS adr_record_drugs (
  id         INT          NOT NULL AUTO_INCREMENT PRIMARY KEY,
  record_id  INT          NOT NULL,
  drug_name  VARCHAR(300) NOT NULL,
  FOREIGN KEY (record_id) REFERENCES adr_records(id) ON DELETE CASCADE,
  INDEX idx_recdrug_rec (record_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- ─────────────────────────────────────────────
--  ADR SYMPTOMS
-- ─────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS adr_symptoms (
  id                INT      NOT NULL AUTO_INCREMENT PRIMARY KEY,
  record_id         INT      NOT NULL,
  term_id           INT,
  custom_key        VARCHAR(200),
  custom_label      VARCHAR(300),
  grade             TINYINT  NOT NULL,
  description       TEXT,
  note              TEXT,
  additional_detail TEXT,
  FOREIGN KEY (record_id) REFERENCES adr_records(id) ON DELETE CASCADE,
  FOREIGN KEY (term_id)   REFERENCES ctcae_terms(id) ON DELETE SET NULL,
  INDEX idx_sym_record (record_id),
  INDEX idx_sym_grade  (grade)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- ═══════════════════════════════════════════════════════════════
--  VIEW: view_adr_summary
--  เทียบเท่า PostgreSQL view เดิม — MySQL ไม่รองรับ FILTER(WHERE)
--  จึงใช้ SUM(CASE WHEN ... END) แทน
-- ═══════════════════════════════════════════════════════════════
DROP VIEW IF EXISTS view_adr_summary;

CREATE VIEW view_adr_summary AS
SELECT
  r.id,
  r.hn,
  p.patient_name,
  r.record_date,
  r.cycle,
  r.dose,
  r.dose_unit,
  r.note,
  r.recommendation,
  r.follow_up_date,
  r.created_by,
  r.created_at,
  r.updated_at,
  r.encounter_id,
  e.type                                                    AS encounter_type,
  p.diagnosis,
  -- drugs เป็น JSON array string  (แทน PostgreSQL array_agg)
  CONCAT('[',
    GROUP_CONCAT(
      DISTINCT CONCAT('"', REPLACE(d.drug_name, '"', '\\"'), '"')
      ORDER BY d.drug_name
      SEPARATOR ','
    ),
  ']')                                                       AS drugs,
  -- symptom aggregates
  MAX(s.grade)                                               AS max_grade,
  COUNT(s.id)                                                AS symptom_count,
  SUM(CASE WHEN s.grade >= 3 THEN 1 ELSE 0 END)             AS grade3_plus_count,
  -- symptoms JSON summary (key → grade) เพื่อความเข้ากันได้กับ frontend
  JSON_OBJECTAGG(
    COALESCE(ct.`key`, s.custom_key, CONCAT('custom_', s.id)),
    s.grade
  )                                                          AS symptoms
FROM adr_records r
JOIN patients p ON p.hn = r.hn
LEFT JOIN encounters  e  ON e.id  = r.encounter_id
LEFT JOIN adr_record_drugs d ON d.record_id = r.id
LEFT JOIN adr_symptoms  s  ON s.record_id  = r.id
LEFT JOIN ctcae_terms  ct  ON ct.id = s.term_id
GROUP BY
  r.id, r.hn, p.patient_name, r.record_date, r.cycle,
  r.dose, r.dose_unit, r.note, r.recommendation, r.follow_up_date,
  r.created_by, r.created_at, r.updated_at,
  r.encounter_id, e.type, p.diagnosis;

-- ═══════════════════════════════════════════════════════════════
--  SEED: ตัวอย่าง users (bcrypt hash ของ "password123")
--  สร้าง hash จริงด้วย: node -e "const b=require('bcrypt'); b.hash('password123',10).then(console.log)"
-- ═══════════════════════════════════════════════════════════════
INSERT IGNORE INTO users (username, password_hash, role, name, title) VALUES
  ('pharmacist1', '$2b$10$YourHashHere', 'pharmacist', 'ภก. สมชาย ใจดี',   'เภสัชกร'),
  ('nurse1',      '$2b$10$YourHashHere', 'nurse',      'พยาบาล สมหญิง ดีงาม', 'พยาบาล'),
  ('doctor1',     '$2b$10$YourHashHere', 'doctor',     'นพ. วิชัย รักษาดี',  'แพทย์');