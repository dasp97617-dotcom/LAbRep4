-- ================================
-- DATABASE & TABLES (Kept as is for structure)
-- ================================
DROP DATABASE IF EXISTS faculty_trigger_lab;
CREATE DATABASE faculty_trigger_lab;
USE faculty_trigger_lab;

CREATE TABLE EMPLOYEE (
    EmpID INT PRIMARY KEY,
    EmpName VARCHAR(100) NOT NULL,
    BasicSalary DECIMAL(10,2) NOT NULL,
    StartDate DATE NOT NULL,
    NoOfPub INT NOT NULL CHECK(NoOfPub >= 0),
    IncrementRate DECIMAL(5,2) DEFAULT 0,
    UpdatedSalary DECIMAL(10,2)
);

CREATE TABLE SALARY_LOG (
    LogID INT AUTO_INCREMENT PRIMARY KEY,
    EmpID INT,
    OldSalary DECIMAL(10,2),
    NewSalary DECIMAL(10,2),
    ChangedAt DATETIME DEFAULT CURRENT_TIMESTAMP,
    Note VARCHAR(100),
    FOREIGN KEY (EmpID) REFERENCES EMPLOYEE(EmpID)
);

-- ================================
-- CONSOLIDATED CALCULATION LOGIC
-- ================================
-- We'll use a BEFORE INSERT and BEFORE UPDATE trigger 
-- to ensure the math is always correct.
-- ================================

DELIMITER $$

-- Logic optimization: Use a single logic flow for both Insert and Update
CREATE TRIGGER trg_salary_calc_insert
BEFORE INSERT ON EMPLOYEE
FOR EACH ROW
BEGIN
    DECLARE years INT;
    SET years = TIMESTAMPDIFF(YEAR, NEW.StartDate, CURDATE());
    
    IF years > 1 THEN
        SET NEW.IncrementRate = CASE 
            WHEN NEW.NoOfPub > 4 THEN 20
            WHEN NEW.NoOfPub BETWEEN 2 AND 4 THEN 10 -- Improved range logic
            WHEN NEW.NoOfPub = 1 THEN 5
            ELSE 0 
        END;
    ELSE
        SET NEW.IncrementRate = 0;
    END IF;
    
    SET NEW.UpdatedSalary = NEW.BasicSalary * (1 + (NEW.IncrementRate / 100));
END$$

CREATE TRIGGER trg_salary_calc_update
BEFORE UPDATE ON EMPLOYEE
FOR EACH ROW
BEGIN
    DECLARE years INT;
    SET years = TIMESTAMPDIFF(YEAR, NEW.StartDate, CURDATE());
    
    IF years > 1 THEN
        SET NEW.IncrementRate = CASE 
            WHEN NEW.NoOfPub > 4 THEN 20
            WHEN NEW.NoOfPub BETWEEN 2 AND 4 THEN 10
            WHEN NEW.NoOfPub = 1 THEN 5
            ELSE 0 
        END;
    ELSE
        SET NEW.IncrementRate = 0;
    END IF;
    
    SET NEW.UpdatedSalary = NEW.BasicSalary * (1 + (NEW.IncrementRate / 100));
END$$

-- ================================
-- LOGGING TRIGGER (Fixed Comparison)
-- ================================
CREATE TRIGGER trg_salary_log
AFTER UPDATE ON EMPLOYEE
FOR EACH ROW
BEGIN
    -- Using NULL-safe comparison or checking specific change
    IF OLD.UpdatedSalary <> NEW.UpdatedSalary THEN
        INSERT INTO SALARY_LOG(EmpID, OldSalary, NewSalary, Note)
        VALUES (NEW.EmpID, OLD.UpdatedSalary, NEW.UpdatedSalary, 
                CONCAT('Rate: ', NEW.IncrementRate, '% Applied'));
    END IF;
END$$

DELIMITER ;

-- ================================
-- DATA INSERTION (Triggers will handle the math automatically)
-- ================================
INSERT INTO EMPLOYEE (EmpID, EmpName, BasicSalary, StartDate, NoOfPub) VALUES
(1, 'Rahim', 30000, '2022-01-01', 5),
(2, 'Karim', 25000, '2023-01-01', 3),
(3, 'Jamal', 20000, '2024-06-01', 1),
(4, 'Hasan', 18000, '2025-01-01', 0);

-- ================================
-- TEST UPDATES
-- ================================
-- Jamal (ID 3) was hired in 2024. If today is April 2026, he has > 1 year.
UPDATE EMPLOYEE SET NoOfPub = 6 WHERE EmpID = 3; 

-- ================================
-- FINAL OUTPUT
-- ================================
SELECT * FROM EMPLOYEE;
SELECT * FROM SALARY_LOG;
