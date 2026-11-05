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