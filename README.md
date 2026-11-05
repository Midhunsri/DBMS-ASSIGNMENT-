# DBMS-ASSIGNMENT-
queries 
CSA05 DBMS CO1 Assignment | Meridian Museum
-- Student: D. Midhun Sri | Register No: 1924260014 | Slot A
-- Target: MySQL 8.0+
-- Run this entire script in MySQL Workbench or mysql command-line.
-- It creates a fresh database named meridian_museum_slot_a.

DROP DATABASE IF EXISTS meridian_museum_slot_a;
CREATE DATABASE meridian_museum_slot_a;
USE meridian_museum_slot_a;

SET FOREIGN_KEY_CHECKS = 0;
DROP TABLE IF EXISTS condition_report, placement, display_slot, loan_object,
                     artifact_owner, loan, artifact, lender_phone, lender,
                     curator, registrar, employee, person, collection_unit,
                     exhibition, organization, notification_contact;
SET FOREIGN_KEY_CHECKS = 1;

CREATE TABLE person (
  person_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  first_name VARCHAR(60) NOT NULL,
  middle_name VARCHAR(60),
  last_name VARCHAR(60) NOT NULL,
  email VARCHAR(120) UNIQUE
) ENGINE=InnoDB;

CREATE TABLE employee (
  person_id BIGINT PRIMARY KEY,
  role ENUM('CURATOR','REGISTRAR','BOTH','OTHER') NOT NULL,
  joined_on DATE,
  FOREIGN KEY (person_id) REFERENCES person(person_id)
) ENGINE=InnoDB;

CREATE TABLE curator (
  person_id BIGINT PRIMARY KEY,
  specialty VARCHAR(100),
  FOREIGN KEY (person_id) REFERENCES employee(person_id)
) ENGINE=InnoDB;

CREATE TABLE registrar (
  person_id BIGINT PRIMARY KEY,
  office_code VARCHAR(20),
  FOREIGN KEY (person_id) REFERENCES employee(person_id)
) ENGINE=InnoDB;

CREATE TABLE lender (
  lender_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  person_id BIGINT UNIQUE,
  lender_name VARCHAR(150) NOT NULL,
  lender_type ENUM('PERSON','ORGANIZATION') NOT NULL,
  email VARCHAR(120),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (person_id) REFERENCES person(person_id)
) ENGINE=InnoDB;

CREATE TABLE lender_phone (
  lender_id BIGINT NOT NULL,
  phone_no VARCHAR(25) NOT NULL,
  phone_label VARCHAR(20) DEFAULT 'MAIN',
  PRIMARY KEY (lender_id, phone_no),
  FOREIGN KEY (lender_id) REFERENCES lender(lender_id)
) ENGINE=InnoDB;

CREATE TABLE organization (
  organization_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  organization_name VARCHAR(150) NOT NULL UNIQUE,
  address VARCHAR(250)
) ENGINE=InnoDB;

CREATE TABLE notification_contact (
  contact_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  person_id BIGINT,
  organization_id BIGINT,
  CHECK ((person_id IS NOT NULL AND organization_id IS NULL)
      OR (person_id IS NULL AND organization_id IS NOT NULL)),
  FOREIGN KEY (person_id) REFERENCES person(person_id),
  FOREIGN KEY (organization_id) REFERENCES organization(organization_id)
) ENGINE=InnoDB;

CREATE TABLE collection_unit (
  unit_id INT PRIMARY KEY,
  unit_name VARCHAR(80) NOT NULL UNIQUE
) ENGINE=InnoDB;

CREATE TABLE artifact (
  artifact_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  catalog_no VARCHAR(40) NOT NULL UNIQUE,
  title VARCHAR(180) NOT NULL,
  collection_type VARCHAR(60) NOT NULL,
  unit_id INT NOT NULL,
  estimated_value DECIMAL(14,2),
  FOREIGN KEY (unit_id) REFERENCES collection_unit(unit_id)
) ENGINE=InnoDB;

CREATE TABLE exhibition (
  exhibition_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  exhibition_name VARCHAR(150) NOT NULL,
  starts_on DATE NOT NULL,
  ends_on DATE NOT NULL,
  CHECK (ends_on >= starts_on)
) ENGINE=InnoDB;

-- DISPLAY_SLOT is weak: slot_no is unique only within its exhibition.
CREATE TABLE display_slot (
  exhibition_id BIGINT NOT NULL,
  slot_no INT NOT NULL,
  gallery_name VARCHAR(100),
  PRIMARY KEY (exhibition_id, slot_no),
  FOREIGN KEY (exhibition_id) REFERENCES exhibition(exhibition_id)
) ENGINE=InnoDB;

CREATE TABLE loan (
  loan_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  lender_id BIGINT NOT NULL,
  curator_id BIGINT NOT NULL,
  registrar_id BIGINT NOT NULL,
  loan_start DATE NOT NULL,
  loan_due DATE NOT NULL,
  returned_on DATE,
  loan_status ENUM('OPEN','CLOSED','CANCELLED') NOT NULL DEFAULT 'OPEN',
  CHECK (loan_due >= loan_start),
  CHECK (returned_on IS NULL OR returned_on >= loan_start),
  FOREIGN KEY (lender_id) REFERENCES lender(lender_id),
  FOREIGN KEY (curator_id) REFERENCES curator(person_id),
  FOREIGN KEY (registrar_id) REFERENCES registrar(person_id)
) ENGINE=InnoDB;

-- An artifact may occur in many successive loans; each loan contains many artifacts.
CREATE TABLE loan_object (
  loan_id BIGINT NOT NULL,
  artifact_id BIGINT NOT NULL,
  received_condition VARCHAR(500) NOT NULL,
  line_insured_value DECIMAL(14,2),
  received_at DATETIME NOT NULL,
  PRIMARY KEY (loan_id, artifact_id),
  FOREIGN KEY (loan_id) REFERENCES loan(loan_id),
  FOREIGN KEY (artifact_id) REFERENCES artifact(artifact_id)
) ENGINE=InnoDB;

-- Legal ownership is many-to-many and time-varying, independent of loan custody.
CREATE TABLE artifact_owner (
  artifact_id BIGINT NOT NULL,
  owner_lender_id BIGINT NOT NULL,
  valid_from DATE NOT NULL,
  valid_to DATE,
  ownership_share DECIMAL(5,2) NOT NULL,
  PRIMARY KEY (artifact_id, owner_lender_id, valid_from),
  CHECK (ownership_share > 0 AND ownership_share <= 100),
  CHECK (valid_to IS NULL OR valid_to >= valid_from),
  FOREIGN KEY (artifact_id) REFERENCES artifact(artifact_id),
  FOREIGN KEY (owner_lender_id) REFERENCES lender(lender_id)
) ENGINE=InnoDB;

CREATE TABLE placement (
  placement_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  loan_id BIGINT NOT NULL,
  artifact_id BIGINT NOT NULL,
  exhibition_id BIGINT NOT NULL,
  slot_no INT NOT NULL,
  placed_from DATE NOT NULL,
  placed_to DATE,
  FOREIGN KEY (loan_id, artifact_id) REFERENCES loan_object(loan_id, artifact_id),
  FOREIGN KEY (exhibition_id, slot_no) REFERENCES display_slot(exhibition_id, slot_no),
  CHECK (placed_to IS NULL OR placed_to >= placed_from)
) ENGINE=InnoDB;

CREATE TABLE condition_report (
  report_id BIGINT PRIMARY KEY AUTO_INCREMENT,
  loan_id BIGINT NOT NULL,
  artifact_id BIGINT NOT NULL,
  inspected_by BIGINT NOT NULL,
  inspected_at DATETIME NOT NULL,
  condition_stage ENUM('BEFORE_DISPATCH','ON_RECEIPT','DURING_DISPLAY','ON_RETURN') NOT NULL,
  condition_notes TEXT NOT NULL,
  image_reference VARCHAR(255),
  FOREIGN KEY (loan_id, artifact_id) REFERENCES loan_object(loan_id, artifact_id),
  FOREIGN KEY (inspected_by) REFERENCES registrar(person_id)
) ENGINE=InnoDB;

INSERT INTO collection_unit(unit_id,unit_name) VALUES
(1,'Sculpture'),(2,'Paintings'),(3,'Textiles'),(4,'Coins'),
(5,'Manuscripts'),(6,'Archaeology'),(7,'Decorative Arts'),(8,'Photography');

-- Sample people and employees
INSERT INTO person(person_id,first_name,last_name,email) VALUES
(1,'Anita','Rao','anita@example.org'),
(2,'Bala','Kumar','bala@example.org'),
(3,'Chitra','Das','chitra@example.org'),
(4,'Dev','Iyer','dev@example.org');

INSERT INTO employee(person_id,role,joined_on) VALUES
(1,'CURATOR','2024-06-01'),(2,'REGISTRAR','2023-02-15'),
(3,'BOTH','2022-01-10');

INSERT INTO curator(person_id,specialty) VALUES (1,'Sculpture'),(3,'Paintings');
INSERT INTO registrar(person_id,office_code) VALUES (2,'REG-01'),(3,'REG-02');

INSERT INTO lender(lender_id,person_id,lender_name,lender_type,email) VALUES
(1,4,'Dev Iyer','PERSON','dev@example.org'),
(2,NULL,'Heritage Trust','ORGANIZATION','loans@heritage.org'),
(3,NULL,'Art Patrons Group','ORGANIZATION','contact@artpatrons.org');

INSERT INTO lender_phone(lender_id,phone_no,phone_label) VALUES
(1,'9000000001','MOBILE'),(2,'9000000002','OFFICE');

INSERT INTO organization(organization_id,organization_name,address) VALUES
(1,'Heritage Trust','Chennai'),(2,'Art Patrons Group','Chennai');
INSERT INTO notification_contact(contact_id,organization_id) VALUES (1,1),(2,2);

INSERT INTO artifact(artifact_id,catalog_no,title,collection_type,unit_id,estimated_value) VALUES
(1,'ART-001','Bronze Dancer','Sculpture',1,250000.00),
(2,'ART-002','Temple Painting','Painting',2,180000.00),
(3,'ART-003','Ancient Textile','Textile',3,95000.00);

INSERT INTO exhibition(exhibition_id,exhibition_name,starts_on,ends_on) VALUES
(1,'Living Heritage','2026-10-01','2026-12-31'),
(2,'Art Through Time','2026-11-01','2027-01-31');

INSERT INTO display_slot(exhibition_id,slot_no,gallery_name) VALUES
(1,1,'Main Gallery'),(1,2,'East Gallery'),(2,1,'History Gallery');

INSERT INTO loan(loan_id,lender_id,curator_id,registrar_id,loan_start,loan_due,loan_status) VALUES
(1,1,1,2,'2026-09-20','2026-10-20','OPEN'),
(2,2,3,2,'2026-10-15','2026-12-15','OPEN'),
(3,1,1,3,'2025-01-01','2025-03-01','CLOSED');

INSERT INTO loan_object(loan_id,artifact_id,received_condition,line_insured_value,received_at) VALUES
(1,1,'Good; minor surface wear','260000.00','2026-09-20 10:00:00'),
(2,2,'Good; stable pigment','190000.00','2026-10-15 11:00:00'),
(2,3,'Fair; folds documented','100000.00','2026-10-15 11:30:00'),
(3,1,'Good at return','255000.00','2025-01-01 09:00:00');

INSERT INTO artifact_owner(artifact_id,owner_lender_id,valid_from,valid_to,ownership_share) VALUES
(1,1,'2020-01-01',NULL,100.00),
(2,2,'2019-01-01',NULL,100.00),
(3,2,'2020-01-01',NULL,60.00),
(3,3,'2020-01-01',NULL,40.00);

INSERT INTO placement(loan_id,artifact_id,exhibition_id,slot_no,placed_from,placed_to) VALUES
(1,1,1,1,'2026-10-01',NULL),
(2,2,1,2,'2026-10-16',NULL);

INSERT INTO condition_report(loan_id,artifact_id,inspected_by,inspected_at,condition_stage,condition_notes) VALUES
(1,1,2,'2026-09-20 10:15:00','ON_RECEIPT','No structural damage observed'),
(2,2,2,'2026-10-15 11:45:00','ON_RECEIPT','Pigment stable'),
(2,2,3,'2026-11-01 09:00:00','DURING_DISPLAY','No visible change');

-- Eight required operational queries
-- Q1. Overdue loans that are still open.
SELECT loan_id, lender_id, loan_start, loan_due
FROM loan
WHERE loan_status='OPEN' AND returned_on IS NULL AND loan_due < CURRENT_DATE();

-- Q2. Artifacts with no condition report in any loan.
SELECT a.artifact_id, a.catalog_no, a.title
FROM artifact a
WHERE NOT EXISTS (
  SELECT 1 FROM condition_report cr WHERE cr.artifact_id=a.artifact_id
);

-- Q3. Current custody list with lender, artifact and due date.
SELECT l.loan_id, le.lender_name, a.catalog_no, a.title, l.loan_due
FROM loan l
JOIN lender le ON le.lender_id=l.lender_id
JOIN loan_object lo ON lo.loan_id=l.loan_id
JOIN artifact a ON a.artifact_id=lo.artifact_id
WHERE l.loan_status='OPEN';

-- Q4. Loan objects and number of condition reports.
SELECT lo.loan_id, lo.artifact_id, COUNT(cr.report_id) AS report_count
FROM loan_object lo
LEFT JOIN condition_report cr
  ON cr.loan_id=lo.loan_id AND cr.artifact_id=lo.artifact_id
GROUP BY lo.loan_id, lo.artifact_id;

-- Q5. Exhibition inventory with owner-scoped display slot.
SELECT e.exhibition_name, ds.slot_no, p.placement_id, a.title,
       p.placed_from, p.placed_to
FROM placement p
JOIN exhibition e ON e.exhibition_id=p.exhibition_id
JOIN display_slot ds
  ON ds.exhibition_id=p.exhibition_id AND ds.slot_no=p.slot_no
JOIN artifact a ON a.artifact_id=p.artifact_id
ORDER BY e.exhibition_id, ds.slot_no;

-- Q6. Ownership shares currently active (not the same as custody).
SELECT a.catalog_no, a.title, le.lender_name, ao.ownership_share
FROM artifact_owner ao
JOIN artifact a ON a.artifact_id=ao.artifact_id
JOIN lender le ON le.lender_id=ao.owner_lender_id
WHERE ao.valid_from <= CURRENT_DATE()
  AND (ao.valid_to IS NULL OR ao.valid_to >= CURRENT_DATE());

-- Q7. Loans handled by a curator who is also a registrar.
SELECT l.loan_id, p.first_name, p.last_name
FROM loan l
JOIN curator c ON c.person_id=l.curator_id
JOIN registrar r ON r.person_id=l.curator_id
JOIN person p ON p.person_id=l.curator_id;

-- Q8. Condition history for each artifact in a particular loan.
SELECT cr.loan_id, a.catalog_no, cr.inspected_at, cr.condition_stage,
       cr.condition_notes
FROM condition_report cr
JOIN artifact a ON a.artifact_id=cr.artifact_id
WHERE cr.loan_id=2
ORDER BY cr.inspected_at;

-- Validation examples: each statement below is expected to fail.
-- Uncomment and execute separately to observe the MySQL error.
-- Invalid slot: duplicate owner-scoped key
-- INSERT INTO display_slot(exhibition_id,slot_no) VALUES (1,1);
-- Invalid loan period: due date before start date
-- INSERT INTO loan(lender_id,curator_id,registrar_id,loan_start,loan_due)
-- VALUES (1,1,2,'2026-12-01','2026-11-01');
-- Invalid contact category: both union branches empty
-- INSERT INTO notification_contact(person_id,organization_id) VALUES (NULL,NULL);
-- Invalid condition report parent: non-existent loan-object pair
-- INSERT INTO condition_report(loan_id,artifact_id,inspected_by,inspected_at,
--   condition_stage,condition_notes)
-- VALUES (999,999,2,NOW(),'ON_RECEIPT','Invalid parent');

-- Reproducible lender population generator. Run one size at a time.
-- Seed 4211 is used in deterministic generated labels; no random ordering is needed.
-- Example: CALL generate_lenders(100); then 1000, 10000, or 100000.
DELIMITER //
CREATE PROCEDURE generate_lenders(IN p_count INT)
BEGIN
  DECLARE v_start BIGINT DEFAULT 10000;
  IF p_count NOT IN (100,1000,10000,100000) THEN
    SIGNAL SQLSTATE '45000'
      SET MESSAGE_TEXT='Allowed sizes: 100, 1000, 10000, 100000';
  END IF;
  INSERT INTO lender(lender_id,person_id,lender_name,lender_type,email)
  SELECT v_start + n,
         NULL,
         CONCAT('Generated Lender 4211-', n),
         'ORGANIZATION',
         CONCAT('generated',n,'@example.org')
  FROM (
    WITH RECURSIVE seq(n) AS (
      SELECT 1
      UNION ALL
      SELECT n+1 FROM seq WHERE n < p_count
    )
    SELECT n FROM seq
  ) AS numbers;
END//
DELIMITER ;

-- Set recursion depth once before generating up to 100,000 records.
SET SESSION cte_max_recursion_depth = 100001;
-- Run ONE at a time; remove existing generated rows before repeating a size:
-- CALL generate_lenders(100);
-- SELECT COUNT(*) AS lender_count FROM lender;
-- CALL generate_lenders(1000);
-- CALL generate_lenders(10000);
-- CALL generate_lenders(100000);

-- Timing template: execute each query repeatedly (e.g. 5 times), record elapsed
-- milliseconds from Workbench/your SQL client, then report median.
-- Do not invent timing values; client/server hardware affects the result.




ASSIGNMENT 2 



-- =====================================================================
-- SwiftSpan Parcels - Courier Handover Completion   (CSA05 Assignment CO2)
-- Student: D. Midhun Sri   Reg. No: 1924260014   Slot A
-- MySQL 8.0.31+ (CHECK enforced from 8.0.16; WITH/CTE; EXCEPT needs 8.0.31).  InnoDB.
-- STATUS: written for MySQL 8; the same logic was executed and verified in the
--         SQLite harness (swiftspan_experiment.py).  This MySQL script itself
--         was NOT executed in the authoring environment (no MySQL server there).
-- RUN ORDER: 0 -> 1 -> 2 -> 3 -> 4 -> 5 -> 6 -> 7 -> 8 -> 9 (set @actors in section 2)
-- =====================================================================

-- ---------------------------------------------------------------------
-- 0. DATABASES
-- ---------------------------------------------------------------------
DROP DATABASE IF EXISTS swiftspan_a;  DROP DATABASE IF EXISTS swiftspan_b;  DROP DATABASE IF EXISTS swiftspan_c;
CREATE DATABASE swiftspan_a;   -- Design A : flat baseline
CREATE DATABASE swiftspan_b;   -- Design B : partial normalisation
CREATE DATABASE swiftspan_c;   -- Design C : BCNF (recommended)

-- ---------------------------------------------------------------------
-- 1. DDL
-- ---------------------------------------------------------------------
-- ===== DESIGN A : FLAT (14 attributes, logical key (SenderID,ServiceID,DispatchWave)) =====
CREATE TABLE swiftspan_a.flat (
  SenderID        INT UNSIGNED  NOT NULL,
  SenderName      VARCHAR(20)   NOT NULL,
  RouteGroup      TINYINT UNSIGNED NOT NULL,
  ServiceID       TINYINT UNSIGNED NOT NULL,
  ServiceCode     CHAR(6)       NOT NULL,
  ServiceName     VARCHAR(30)   NOT NULL,
  VolumeWeight    DECIMAL(4,2)  NOT NULL,
  CoordinatorID   INT UNSIGNED  NOT NULL,
  CoordinatorName VARCHAR(20)   NOT NULL,
  Hub             VARCHAR(10)   NOT NULL,
  DispatchWave    TINYINT UNSIGNED NOT NULL,
  Completion      TINYINT UNSIGNED NOT NULL,
  ServiceBand     VARCHAR(4)    NOT NULL,
  RouteFactor     DECIMAL(3,2)  NOT NULL,
  PRIMARY KEY (SenderID, ServiceID, DispatchWave),
  CONSTRAINT uq_flat_code UNIQUE (SenderID, ServiceCode, DispatchWave),   -- alternate key
  CONSTRAINT ck_flat_completion CHECK (Completion BETWEEN 0 AND 100),
  KEY ix_flat_service (ServiceID),
  KEY ix_flat_coord   (CoordinatorID),
  KEY ix_flat_wave    (DispatchWave, SenderID)
) ENGINE=InnoDB;

-- ===== DESIGN B : PARTIAL (identity/catalog removed; operator + policy facts still repeated) =====
CREATE TABLE swiftspan_b.sender (
  SenderID INT UNSIGNED NOT NULL PRIMARY KEY, SenderName VARCHAR(20) NOT NULL, RouteGroup TINYINT UNSIGNED NOT NULL
) ENGINE=InnoDB;
CREATE TABLE swiftspan_b.service (
  ServiceID TINYINT UNSIGNED NOT NULL PRIMARY KEY, ServiceCode CHAR(6) NOT NULL, ServiceName VARCHAR(30) NOT NULL,
  VolumeWeight DECIMAL(4,2) NOT NULL, CONSTRAINT uq_service_code UNIQUE (ServiceCode)
) ENGINE=InnoDB;
CREATE TABLE swiftspan_b.assignment (            -- CoordinatorName, Hub repeated: K -> Kn,H is transitive here
  SenderID INT UNSIGNED NOT NULL, ServiceID TINYINT UNSIGNED NOT NULL, CoordinatorID INT UNSIGNED NOT NULL,
  CoordinatorName VARCHAR(20) NOT NULL, Hub VARCHAR(10) NOT NULL,
  PRIMARY KEY (SenderID, ServiceID), KEY ix_asg_coord (CoordinatorID),
  CONSTRAINT fk_b_asg_sender  FOREIGN KEY (SenderID)  REFERENCES swiftspan_b.sender(SenderID)   ON DELETE RESTRICT,
  CONSTRAINT fk_b_asg_service FOREIGN KEY (ServiceID) REFERENCES swiftspan_b.service(ServiceID) ON DELETE RESTRICT
) ENGINE=InnoDB;
CREATE TABLE swiftspan_b.observation (           -- ServiceBand, RouteFactor repeated: P -> B -> F is transitive here
  SenderID INT UNSIGNED NOT NULL, ServiceID TINYINT UNSIGNED NOT NULL, DispatchWave TINYINT UNSIGNED NOT NULL,
  Completion TINYINT UNSIGNED NOT NULL, ServiceBand VARCHAR(4) NOT NULL, RouteFactor DECIMAL(3,2) NOT NULL,
  PRIMARY KEY (SenderID, ServiceID, DispatchWave), KEY ix_obs_wave (DispatchWave, SenderID),
  CONSTRAINT ck_obs_completion CHECK (Completion BETWEEN 0 AND 100),
  CONSTRAINT fk_b_obs_asg FOREIGN KEY (SenderID, ServiceID) REFERENCES swiftspan_b.assignment(SenderID, ServiceID) ON DELETE RESTRICT
) ENGINE=InnoDB;

-- ===== DESIGN C : BCNF (7 relations) =====
CREATE TABLE swiftspan_c.sender (
  SenderID INT UNSIGNED NOT NULL PRIMARY KEY, SenderName VARCHAR(20) NOT NULL, RouteGroup TINYINT UNSIGNED NOT NULL
) ENGINE=InnoDB;
CREATE TABLE swiftspan_c.service (                -- candidate keys: ServiceID (PK) and ServiceCode (alternate)
  ServiceID TINYINT UNSIGNED NOT NULL PRIMARY KEY, ServiceCode CHAR(6) NOT NULL, ServiceName VARCHAR(30) NOT NULL,
  VolumeWeight DECIMAL(4,2) NOT NULL, CONSTRAINT uq_service_code UNIQUE (ServiceCode)
) ENGINE=InnoDB;
CREATE TABLE swiftspan_c.coordinator (
  CoordinatorID INT UNSIGNED NOT NULL PRIMARY KEY, CoordinatorName VARCHAR(20) NOT NULL, Hub VARCHAR(10) NOT NULL
) ENGINE=InnoDB;
CREATE TABLE swiftspan_c.band_policy (
  ServiceBand VARCHAR(4) NOT NULL PRIMARY KEY, RouteFactor DECIMAL(3,2) NOT NULL
) ENGINE=InnoDB;
CREATE TABLE swiftspan_c.completion_policy (      -- fixed fictional policy: Completion -> ServiceBand
  Completion TINYINT UNSIGNED NOT NULL PRIMARY KEY, ServiceBand VARCHAR(4) NOT NULL,
  CONSTRAINT ck_cp_completion CHECK (Completion BETWEEN 0 AND 100),
  CONSTRAINT fk_cp_band FOREIGN KEY (ServiceBand) REFERENCES swiftspan_c.band_policy(ServiceBand) ON DELETE RESTRICT
) ENGINE=InnoDB;
CREATE TABLE swiftspan_c.assignment (             -- scope {SenderID,ServiceID} -> CoordinatorID
  SenderID INT UNSIGNED NOT NULL, ServiceID TINYINT UNSIGNED NOT NULL, CoordinatorID INT UNSIGNED NOT NULL,
  PRIMARY KEY (SenderID, ServiceID), KEY ix_asg_coord (CoordinatorID),
  CONSTRAINT fk_asg_sender  FOREIGN KEY (SenderID)      REFERENCES swiftspan_c.sender(SenderID)           ON DELETE RESTRICT,
  CONSTRAINT fk_asg_service FOREIGN KEY (ServiceID)     REFERENCES swiftspan_c.service(ServiceID)         ON DELETE RESTRICT,
  CONSTRAINT fk_asg_coord   FOREIGN KEY (CoordinatorID) REFERENCES swiftspan_c.coordinator(CoordinatorID) ON DELETE RESTRICT
) ENGINE=InnoDB;
CREATE TABLE swiftspan_c.observation (            -- key (SenderID,ServiceID,DispatchWave) -> Completion
  SenderID INT UNSIGNED NOT NULL, ServiceID TINYINT UNSIGNED NOT NULL, DispatchWave TINYINT UNSIGNED NOT NULL,
  Completion TINYINT UNSIGNED NOT NULL,
  PRIMARY KEY (SenderID, ServiceID, DispatchWave),
  KEY ix_obs_wave (DispatchWave, SenderID), KEY ix_obs_comp (Completion),
  CONSTRAINT ck_obs_completion CHECK (Completion BETWEEN 0 AND 100),
  CONSTRAINT fk_obs_asg  FOREIGN KEY (SenderID, ServiceID) REFERENCES swiftspan_c.assignment(SenderID, ServiceID) ON DELETE RESTRICT,
  CONSTRAINT fk_obs_comp FOREIGN KEY (Completion) REFERENCES swiftspan_c.completion_policy(Completion) ON DELETE RESTRICT
) ENGINE=InnoDB;

-- ---------------------------------------------------------------------
-- 2. SEEDED, DETERMINISTIC DATA GENERATION  (identical logical state in A, B, C)
--    Choose ONE population: 500 actors = 10,000 obs | 5000 = 100,000 | 30000 = 600,000
--    Same arithmetic as the Python harness (verified):  expected SUM(Completion) =
--       500 actors -> 591041     5000 actors -> 5907094     30000 actors -> 35442636
-- ---------------------------------------------------------------------
SET @actors := 500;                              -- <<< change to 5000 or 30000
SET SESSION cte_max_recursion_depth = 100000;
DROP TABLE IF EXISTS swiftspan_a.nums;
CREATE TABLE swiftspan_a.nums (i BIGINT NOT NULL PRIMARY KEY) ENGINE=InnoDB;
INSERT INTO swiftspan_a.nums
WITH RECURSIVE n(i) AS (SELECT CAST(1 AS SIGNED) UNION ALL SELECT i+1 FROM n WHERE i < 30000)
SELECT i FROM n;

-- reference rows (catalogue + fixed policy)
INSERT INTO swiftspan_c.service VALUES (1,'SVC-01','Standard Parcel',1.00),(2,'SVC-02','Express Parcel',1.25),
  (3,'SVC-03','Cold Chain',2.00),(4,'SVC-04','Fragile Handling',1.50),(5,'SVC-05','Bulk Freight',2.50);
INSERT INTO swiftspan_c.band_policy VALUES ('LOW',1.50),('MID',1.00),('HIGH',0.80);
INSERT INTO swiftspan_c.completion_policy
SELECT i-1, CASE WHEN i-1 < 39 THEN 'LOW' WHEN i-1 < 70 THEN 'MID' ELSE 'HIGH' END FROM swiftspan_a.nums WHERE i <= 101;

-- 2a. build the flat baseline (the single source of generated facts)
INSERT INTO swiftspan_a.flat
SELECT t.sid, CONCAT('Sender-',LPAD(t.sid,5,'0')), MOD(t.sid-1,5)+1,
       sv.ServiceID, sv.ServiceCode, sv.ServiceName, sv.VolumeWeight,
       t.cid, CONCAT('Coord-',LPAD(t.cid,5,'0')), CONCAT('Hub-',MOD(t.cid-1,8)+1),
       t.wid, t.comp,
       CASE WHEN t.comp < 39 THEN 'LOW' WHEN t.comp < 70 THEN 'MID' ELSE 'HIGH' END,
       CASE WHEN t.comp < 39 THEN 1.50 WHEN t.comp < 70 THEN 1.00 ELSE 0.80 END
FROM (
  SELECT x.sid, x.vid, x.wid, x.cid, LEAST(100, MOD(x.u,61) + MOD(x.u DIV 61,61)) AS comp
  FROM (
    SELECT y.sid, y.vid, y.wid, y.cid, (MOD(y.h1*1103515245 + 12345, 2147483648) DIV 65536) AS u
    FROM (
      SELECT s.i AS sid, v.i AS vid, w.i AS wid,
             MOD((s.i-1)*5 + (v.i-1), GREATEST(1, @actors DIV 5)) + 1 AS cid,
             MOD(s.i*1103515245 + v.i*134775813 + w.i*69069 + 12345, 2147483648) AS h1
      FROM swiftspan_a.nums s JOIN swiftspan_a.nums v ON v.i <= 5 JOIN swiftspan_a.nums w ON w.i <= 4
      WHERE s.i <= @actors
    ) y
  ) x
) t
JOIN swiftspan_c.service sv ON sv.ServiceID = t.vid;

-- 2b. load Design B from the flat baseline
INSERT INTO swiftspan_b.sender  SELECT DISTINCT SenderID, SenderName, RouteGroup FROM swiftspan_a.flat;
INSERT INTO swiftspan_b.service SELECT DISTINCT ServiceID, ServiceCode, ServiceName, VolumeWeight FROM swiftspan_a.flat;
INSERT INTO swiftspan_b.assignment SELECT DISTINCT SenderID, ServiceID, CoordinatorID, CoordinatorName, Hub FROM swiftspan_a.flat;
INSERT INTO swiftspan_b.observation SELECT SenderID, ServiceID, DispatchWave, Completion, ServiceBand, RouteFactor FROM swiftspan_a.flat;

-- 2c. load Design C from the flat baseline
INSERT INTO swiftspan_c.sender      SELECT DISTINCT SenderID, SenderName, RouteGroup FROM swiftspan_a.flat;
INSERT INTO swiftspan_c.coordinator SELECT DISTINCT CoordinatorID, CoordinatorName, Hub FROM swiftspan_a.flat;
INSERT INTO swiftspan_c.assignment  SELECT DISTINCT SenderID, ServiceID, CoordinatorID FROM swiftspan_a.flat;
INSERT INTO swiftspan_c.observation SELECT SenderID, ServiceID, DispatchWave, Completion FROM swiftspan_a.flat;

-- 2d. verification: counts, checksum, and lossless round-trip (both EXCEPT results must be 0 rows)
SELECT COUNT(*) AS flat_rows, SUM(Completion) AS checksum FROM swiftspan_a.flat;
SELECT (SELECT COUNT(*) FROM swiftspan_b.observation) AS b_obs, (SELECT COUNT(*) FROM swiftspan_c.observation) AS c_obs;


-- ---------------------------------------------------------------------
-- 3. VIEWS (Design C).  Views expose columns; they do NOT by themselves restrict which
--    SenderID a caller may read (MySQL has no row-level security).
-- ---------------------------------------------------------------------
USE swiftspan_c;
CREATE VIEW v_actor_observation AS            -- universal relation reconstructed losslessly from the 7 tables
SELECT o.SenderID, s.SenderName, s.RouteGroup, o.ServiceID, sv.ServiceCode, sv.ServiceName, sv.VolumeWeight,
       a.CoordinatorID, co.CoordinatorName, co.Hub, o.DispatchWave, o.Completion, cp.ServiceBand, bp.RouteFactor
FROM observation o
JOIN assignment a ON a.SenderID=o.SenderID AND a.ServiceID=o.ServiceID
JOIN sender s ON s.SenderID=o.SenderID JOIN service sv ON sv.ServiceID=o.ServiceID
JOIN coordinator co ON co.CoordinatorID=a.CoordinatorID
JOIN completion_policy cp ON cp.Completion=o.Completion JOIN band_policy bp ON bp.ServiceBand=cp.ServiceBand;
CREATE VIEW v_actor_latest_status AS
SELECT o.SenderID, sv.ServiceCode, sv.ServiceName, o.DispatchWave AS LatestWave, o.Completion, cp.ServiceBand, bp.RouteFactor,
       ROUND(o.Completion*sv.VolumeWeight*bp.RouteFactor,2) AS AdjIndex
FROM observation o
JOIN service sv ON sv.ServiceID=o.ServiceID
JOIN completion_policy cp ON cp.Completion=o.Completion
JOIN band_policy bp ON bp.ServiceBand=cp.ServiceBand
WHERE o.DispatchWave=(SELECT MAX(x.DispatchWave) FROM observation x WHERE x.SenderID=o.SenderID AND x.ServiceID=o.ServiceID);
CREATE VIEW v_coordinator_obs AS
SELECT a.CoordinatorID, co.Hub, o.Completion
FROM observation o JOIN assignment a ON a.SenderID=o.SenderID AND a.ServiceID=o.ServiceID
JOIN coordinator co ON co.CoordinatorID=a.CoordinatorID;
CREATE VIEW v_coordinator_below_hub_mean AS
SELECT cm.CoordinatorID, cm.Hub, ROUND(cm.m,4) AS CoordinatorMean, ROUND(hm.m,4) AS HubMean
FROM (SELECT CoordinatorID,Hub,AVG(Completion*1.0) AS m FROM v_coordinator_obs GROUP BY CoordinatorID,Hub) cm
JOIN (SELECT Hub,SUM(Completion)*1.0/COUNT(*) AS m FROM v_coordinator_obs GROUP BY Hub) hm ON hm.Hub=cm.Hub
WHERE cm.m < hm.m;
CREATE VIEW v_sender_subthreshold_waves AS
SELECT SenderID, COUNT(DISTINCT DispatchWave) AS SubThresholdWaves FROM observation WHERE Completion<39 GROUP BY SenderID;

-- lossless check (must return 0 rows for the first query and 0 rows for the second)
SELECT SenderID,SenderName,RouteGroup,ServiceID,ServiceCode,ServiceName,VolumeWeight,CoordinatorID,CoordinatorName,Hub,DispatchWave,Completion,ServiceBand,RouteFactor
FROM swiftspan_c.v_actor_observation
EXCEPT SELECT SenderID,SenderName,RouteGroup,ServiceID,ServiceCode,ServiceName,VolumeWeight,CoordinatorID,CoordinatorName,Hub,DispatchWave,Completion,ServiceBand,RouteFactor FROM swiftspan_a.flat;
SELECT SenderID,SenderName,RouteGroup,ServiceID,ServiceCode,ServiceName,VolumeWeight,CoordinatorID,CoordinatorName,Hub,DispatchWave,Completion,ServiceBand,RouteFactor
FROM swiftspan_a.flat
EXCEPT SELECT SenderID,SenderName,RouteGroup,ServiceID,ServiceCode,ServiceName,VolumeWeight,CoordinatorID,CoordinatorName,Hub,DispatchWave,Completion,ServiceBand,RouteFactor FROM swiftspan_c.v_actor_observation;


-- ---------------------------------------------------------------------
-- 4. THE FIVE CASE QUERIES  (A = flat, B = partial, C = BCNF).  Results must be identical.
--    Interpretation notes: "actor 1" = SenderID 1.  Weighted adjusted index = Completion x VolumeWeight x RouteFactor.
--    Q2 = every assigned service of the sender has >=1 observation and none below 39 (anti-join, no vacuous truth).
--    Q3 = sender has >=1 wave-4 observation and none of its wave-4 observations is below 39.
-- ---------------------------------------------------------------------
-- Q1  actor 1, DispatchWave=4, weighted adjusted index  [design A]
USE swiftspan_a;
SELECT SenderID,ServiceID,DispatchWave,Completion,ROUND(Completion*VolumeWeight*RouteFactor,2) FROM flat WHERE SenderID=1 AND DispatchWave=4 ORDER BY ServiceID;

-- Q1  actor 1, DispatchWave=4, weighted adjusted index  [design B]
USE swiftspan_b;
SELECT o.SenderID,o.ServiceID,o.DispatchWave,o.Completion,ROUND(o.Completion*sv.VolumeWeight*o.RouteFactor,2) FROM observation o JOIN service sv ON sv.ServiceID=o.ServiceID WHERE o.SenderID=1 AND o.DispatchWave=4 ORDER BY o.ServiceID;

-- Q1  actor 1, DispatchWave=4, weighted adjusted index  [design C]
USE swiftspan_c;
SELECT o.SenderID,o.ServiceID,o.DispatchWave,o.Completion,ROUND(o.Completion*sv.VolumeWeight*bp.RouteFactor,2) FROM observation o JOIN service sv ON sv.ServiceID=o.ServiceID JOIN completion_policy cp ON cp.Completion=o.Completion JOIN band_policy bp ON bp.ServiceBand=cp.ServiceBand WHERE o.SenderID=1 AND o.DispatchWave=4 ORDER BY o.ServiceID;

-- Q2  senders meeting threshold 39 for every required service  [design A]
USE swiftspan_a;
SELECT f.SenderID FROM flat f GROUP BY f.SenderID HAVING MIN(f.Completion)>=39 AND COUNT(DISTINCT f.ServiceID)=5 ORDER BY 1;

-- Q2  senders meeting threshold 39 for every required service  [design B]
USE swiftspan_b;
SELECT s.SenderID FROM sender s WHERE NOT EXISTS (SELECT 1 FROM assignment a WHERE a.SenderID=s.SenderID AND (NOT EXISTS (SELECT 1 FROM observation o WHERE o.SenderID=a.SenderID AND o.ServiceID=a.ServiceID) OR EXISTS (SELECT 1 FROM observation x WHERE x.SenderID=a.SenderID AND x.ServiceID=a.ServiceID AND x.Completion<39))) ORDER BY 1;

-- Q2  senders meeting threshold 39 for every required service  [design C]
USE swiftspan_c;
SELECT s.SenderID FROM sender s WHERE NOT EXISTS (SELECT 1 FROM assignment a WHERE a.SenderID=s.SenderID AND (NOT EXISTS (SELECT 1 FROM observation o WHERE o.SenderID=a.SenderID AND o.ServiceID=a.ServiceID) OR EXISTS (SELECT 1 FROM observation x WHERE x.SenderID=a.SenderID AND x.ServiceID=a.ServiceID AND x.Completion<39))) ORDER BY 1;

-- Q3  observed in period 4 and no value below 39  [design A]
USE swiftspan_a;
SELECT DISTINCT o.SenderID FROM flat o WHERE o.DispatchWave=4 AND NOT EXISTS (SELECT 1 FROM flat x WHERE x.SenderID=o.SenderID AND x.DispatchWave=4 AND x.Completion<39) ORDER BY 1;

-- Q3  observed in period 4 and no value below 39  [design B]
USE swiftspan_b;
SELECT DISTINCT o.SenderID FROM observation o WHERE o.DispatchWave=4 AND NOT EXISTS (SELECT 1 FROM observation x WHERE x.SenderID=o.SenderID AND x.DispatchWave=4 AND x.Completion<39) ORDER BY 1;

-- Q3  observed in period 4 and no value below 39  [design C]
USE swiftspan_c;
SELECT DISTINCT o.SenderID FROM observation o WHERE o.DispatchWave=4 AND NOT EXISTS (SELECT 1 FROM observation x WHERE x.SenderID=o.SenderID AND x.DispatchWave=4 AND x.Completion<39) ORDER BY 1;

-- Q4  coordinator means below the observation-weighted hub mean  [design A]
USE swiftspan_a;
WITH cm AS (SELECT CoordinatorID,Hub,AVG(Completion*1.0) m FROM flat GROUP BY CoordinatorID,Hub), hm AS (SELECT Hub,SUM(Completion)*1.0/COUNT(*) m FROM flat GROUP BY Hub) SELECT cm.CoordinatorID,ROUND(cm.m,4) FROM cm JOIN hm ON hm.Hub=cm.Hub WHERE cm.m<hm.m ORDER BY 1;

-- Q4  coordinator means below the observation-weighted hub mean  [design B]
USE swiftspan_b;
WITH j AS (SELECT a.CoordinatorID,a.Hub,o.Completion FROM observation o JOIN assignment a ON a.SenderID=o.SenderID AND a.ServiceID=o.ServiceID), cm AS (SELECT CoordinatorID,Hub,AVG(Completion*1.0) m FROM j GROUP BY CoordinatorID,Hub), hm AS (SELECT Hub,SUM(Completion)*1.0/COUNT(*) m FROM j GROUP BY Hub) SELECT cm.CoordinatorID,ROUND(cm.m,4) FROM cm JOIN hm ON hm.Hub=cm.Hub WHERE cm.m<hm.m ORDER BY 1;

-- Q4  coordinator means below the observation-weighted hub mean  [design C]
USE swiftspan_c;
WITH j AS (SELECT a.CoordinatorID,co.Hub,o.Completion FROM observation o JOIN assignment a ON a.SenderID=o.SenderID AND a.ServiceID=o.ServiceID JOIN coordinator co ON co.CoordinatorID=a.CoordinatorID), cm AS (SELECT CoordinatorID,Hub,AVG(Completion*1.0) m FROM j GROUP BY CoordinatorID,Hub), hm AS (SELECT Hub,SUM(Completion)*1.0/COUNT(*) m FROM j GROUP BY Hub) SELECT cm.CoordinatorID,ROUND(cm.m,4) FROM cm JOIN hm ON hm.Hub=cm.Hub WHERE cm.m<hm.m ORDER BY 1;

-- Q5  sub-threshold observations in at least 3 distinct waves  [design A]
USE swiftspan_a;
SELECT SenderID FROM (SELECT SenderID,COUNT(DISTINCT DispatchWave) n FROM flat WHERE Completion<39 GROUP BY SenderID) AS t WHERE n>=3 ORDER BY 1;

-- Q5  sub-threshold observations in at least 3 distinct waves  [design B]
USE swiftspan_b;
SELECT SenderID FROM (SELECT SenderID,COUNT(DISTINCT DispatchWave) n FROM observation WHERE Completion<39 GROUP BY SenderID) AS t WHERE n>=3 ORDER BY 1;

-- Q5  sub-threshold observations in at least 3 distinct waves  [design C]
USE swiftspan_c;
SELECT SenderID FROM (SELECT SenderID,COUNT(DISTINCT DispatchWave) n FROM observation WHERE Completion<39 GROUP BY SenderID) AS t WHERE n>=3 ORDER BY 1;


-- ---------------------------------------------------------------------
-- 5. ANOMALY TRIALS  (each inside a transaction that is rolled back; ROW_COUNT() = rows touched)
-- ---------------------------------------------------------------------
-- 5.1 catalogue-name change for ServiceID=1  (expected: N/5 rows in A, 1 in B, 1 in C)
START TRANSACTION; UPDATE swiftspan_a.flat SET ServiceName='Standard Parcel PLUS' WHERE ServiceID=1; SELECT ROW_COUNT() AS rows_A; ROLLBACK;
START TRANSACTION; UPDATE swiftspan_b.service SET ServiceName='Standard Parcel PLUS' WHERE