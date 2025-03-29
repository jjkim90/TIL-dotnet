# SQL 기초 - DDL과 기본 DML

## 개요

SQL(Structured Query Language)은 관계형 데이터베이스를 관리하고 조작하기 위한 표준 언어입니다. 이 장에서는 데이터베이스 구조를 정의하는 DDL(Data Definition Language)과 데이터를 조작하는 DML(Data Manipulation Language)의 기본을 학습합니다.

## 1. DDL (Data Definition Language)

### 1.1 CREATE - 객체 생성

```sql
-- 데이터베이스 생성
CREATE DATABASE CompanyDB
COLLATE Korean_Wansung_CI_AS;  -- SQL Server 한글 설정

-- PostgreSQL
CREATE DATABASE companydb
WITH ENCODING = 'UTF8'
LC_COLLATE = 'ko_KR.UTF-8'
LC_CTYPE = 'ko_KR.UTF-8';

-- 데이터베이스 사용
USE CompanyDB;  -- SQL Server
\c companydb    -- PostgreSQL

-- 테이블 생성
CREATE TABLE Employees (
    EmployeeId INT PRIMARY KEY IDENTITY(1,1),  -- SQL Server
    -- EmployeeId SERIAL PRIMARY KEY,          -- PostgreSQL
    FirstName NVARCHAR(50) NOT NULL,
    LastName NVARCHAR(50) NOT NULL,
    Email VARCHAR(100) UNIQUE NOT NULL,
    HireDate DATE NOT NULL DEFAULT GETDATE(),   -- SQL Server
    -- HireDate DATE NOT NULL DEFAULT CURRENT_DATE,  -- PostgreSQL
    Salary DECIMAL(10, 2) CHECK (Salary > 0),
    DepartmentId INT,
    ManagerId INT,
    IsActive BIT DEFAULT 1,
    CreatedAt DATETIME2 DEFAULT SYSDATETIME(),
    UpdatedAt DATETIME2
);

-- 부서 테이블
CREATE TABLE Departments (
    DepartmentId INT PRIMARY KEY IDENTITY(1,1),
    DepartmentName NVARCHAR(100) NOT NULL,
    Location NVARCHAR(100),
    Budget DECIMAL(15, 2),
    CONSTRAINT UQ_DepartmentName UNIQUE (DepartmentName)
);

-- 프로젝트 테이블 (다대다 관계 예제)
CREATE TABLE Projects (
    ProjectId INT PRIMARY KEY IDENTITY(1,1),
    ProjectName NVARCHAR(200) NOT NULL,
    StartDate DATE NOT NULL,
    EndDate DATE,
    Status NVARCHAR(20) DEFAULT 'Planning',
    CONSTRAINT CHK_Dates CHECK (EndDate IS NULL OR EndDate >= StartDate),
    CONSTRAINT CHK_Status CHECK (Status IN ('Planning', 'Active', 'Completed', 'Cancelled'))
);

-- 직원-프로젝트 연결 테이블
CREATE TABLE EmployeeProjects (
    EmployeeId INT NOT NULL,
    ProjectId INT NOT NULL,
    Role NVARCHAR(50),
    AssignedDate DATE DEFAULT GETDATE(),
    PRIMARY KEY (EmployeeId, ProjectId)
);
```

### 1.2 데이터 타입 상세

```sql
-- 숫자 타입
CREATE TABLE NumericTypes (
    -- 정수형
    TinyIntCol TINYINT,           -- 0 ~ 255
    SmallIntCol SMALLINT,         -- -32,768 ~ 32,767
    IntCol INT,                   -- -2,147,483,648 ~ 2,147,483,647
    BigIntCol BIGINT,             -- -9,223,372,036,854,775,808 ~ 9,223,372,036,854,775,807
    
    -- 실수형
    DecimalCol DECIMAL(10, 2),    -- 고정 소수점, 정확한 숫자
    NumericCol NUMERIC(10, 2),    -- DECIMAL과 동일
    FloatCol FLOAT,               -- 부동 소수점, 근사치
    RealCol REAL,                 -- FLOAT(24)와 동일
    
    -- 통화
    MoneyCol MONEY,               -- -922,337,203,685,477.5808 ~ 922,337,203,685,477.5807
    SmallMoneyCol SMALLMONEY      -- -214,748.3648 ~ 214,748.3647
);

-- 문자열 타입
CREATE TABLE StringTypes (
    -- 고정 길이
    CharCol CHAR(10),             -- 고정 길이, 패딩
    NCharCol NCHAR(10),           -- 유니코드 고정 길이
    
    -- 가변 길이
    VarCharCol VARCHAR(100),      -- 가변 길이
    NVarCharCol NVARCHAR(100),    -- 유니코드 가변 길이
    VarCharMaxCol VARCHAR(MAX),   -- 최대 2GB
    TextCol TEXT,                 -- 구버전 호환성 (권장하지 않음)
    
    -- PostgreSQL 특수 타입
    -- TextCol TEXT,              -- PostgreSQL에서는 제한 없는 가변 길이
);

-- 날짜/시간 타입
CREATE TABLE DateTimeTypes (
    DateCol DATE,                          -- 날짜만 (YYYY-MM-DD)
    TimeCol TIME,                          -- 시간만 (HH:MI:SS)
    DateTimeCol DATETIME,                  -- 날짜와 시간
    DateTime2Col DATETIME2(3),             -- 더 정밀한 날짜/시간
    SmallDateTimeCol SMALLDATETIME,        -- 작은 범위의 날짜/시간
    DateTimeOffsetCol DATETIMEOFFSET,      -- 시간대 포함
    
    -- PostgreSQL
    -- TimestampCol TIMESTAMP,             -- 날짜와 시간
    -- TimestampTzCol TIMESTAMPTZ,         -- 시간대 포함
    -- IntervalCol INTERVAL                -- 시간 간격
);

-- 기타 타입
CREATE TABLE OtherTypes (
    -- 이진 데이터
    BinaryCol BINARY(50),                  -- 고정 길이 이진
    VarBinaryCol VARBINARY(100),           -- 가변 길이 이진
    VarBinaryMaxCol VARBINARY(MAX),        -- 최대 2GB 이진
    
    -- 부울
    BitCol BIT,                            -- 0, 1, NULL
    
    -- GUID
    UniqueIdCol UNIQUEIDENTIFIER DEFAULT NEWID(),  -- SQL Server
    -- UuidCol UUID DEFAULT gen_random_uuid(),      -- PostgreSQL
    
    -- JSON (SQL Server 2016+, PostgreSQL)
    JsonCol NVARCHAR(MAX) CONSTRAINT CHK_JsonValid CHECK (ISJSON(JsonCol) = 1),  -- SQL Server
    -- JsonCol JSON,                                                               -- PostgreSQL
    -- JsonbCol JSONB,                                                            -- PostgreSQL (이진 JSON)
    
    -- XML
    XmlCol XML,
    
    -- 공간 데이터
    GeographyCol GEOGRAPHY,                -- 지리 데이터
    GeometryCol GEOMETRY                   -- 기하 데이터
);
```

### 1.3 제약 조건 (Constraints)

```sql
-- PRIMARY KEY 제약 조건
CREATE TABLE Products (
    ProductId INT NOT NULL,
    CategoryId INT NOT NULL,
    ProductName NVARCHAR(100) NOT NULL,
    CONSTRAINT PK_Products PRIMARY KEY (ProductId)
);

-- 복합 키
CREATE TABLE OrderDetails (
    OrderId INT NOT NULL,
    ProductId INT NOT NULL,
    Quantity INT NOT NULL,
    UnitPrice DECIMAL(10, 2) NOT NULL,
    CONSTRAINT PK_OrderDetails PRIMARY KEY (OrderId, ProductId)
);

-- FOREIGN KEY 제약 조건
ALTER TABLE Employees
ADD CONSTRAINT FK_Employees_Departments 
    FOREIGN KEY (DepartmentId) REFERENCES Departments(DepartmentId);

ALTER TABLE Employees
ADD CONSTRAINT FK_Employees_Manager 
    FOREIGN KEY (ManagerId) REFERENCES Employees(EmployeeId);

-- CASCADE 옵션
ALTER TABLE OrderDetails
ADD CONSTRAINT FK_OrderDetails_Orders
    FOREIGN KEY (OrderId) REFERENCES Orders(OrderId)
    ON DELETE CASCADE    -- 부모 삭제 시 자식도 삭제
    ON UPDATE CASCADE;   -- 부모 수정 시 자식도 수정

-- UNIQUE 제약 조건
ALTER TABLE Employees
ADD CONSTRAINT UQ_Employee_Email UNIQUE (Email);

-- 복합 유니크
ALTER TABLE Employees
ADD CONSTRAINT UQ_Employee_Name UNIQUE (FirstName, LastName);

-- CHECK 제약 조건
ALTER TABLE Employees
ADD CONSTRAINT CHK_Salary_Range 
    CHECK (Salary BETWEEN 30000 AND 500000);

ALTER TABLE Products
ADD CONSTRAINT CHK_Price_Discount 
    CHECK (DiscountPrice < RegularPrice OR DiscountPrice IS NULL);

-- DEFAULT 제약 조건
ALTER TABLE Orders
ADD CONSTRAINT DF_OrderDate DEFAULT GETDATE() FOR OrderDate;

ALTER TABLE Employees
ADD CONSTRAINT DF_IsActive DEFAULT 1 FOR IsActive;

-- 복잡한 CHECK 제약 조건
CREATE TABLE Meetings (
    MeetingId INT PRIMARY KEY,
    StartTime DATETIME NOT NULL,
    EndTime DATETIME NOT NULL,
    RoomNumber INT NOT NULL,
    CONSTRAINT CHK_Meeting_Times CHECK (
        EndTime > StartTime AND
        DATEPART(HOUR, StartTime) >= 8 AND
        DATEPART(HOUR, EndTime) <= 18
    )
);
```

### 1.4 ALTER - 객체 수정

```sql
-- 열 추가
ALTER TABLE Employees
ADD PhoneNumber VARCHAR(20);

ALTER TABLE Employees
ADD 
    Address NVARCHAR(200),
    City NVARCHAR(50),
    PostalCode VARCHAR(10);

-- 열 수정
ALTER TABLE Employees
ALTER COLUMN PhoneNumber VARCHAR(30) NOT NULL;  -- SQL Server

-- PostgreSQL
-- ALTER TABLE Employees
-- ALTER COLUMN PhoneNumber TYPE VARCHAR(30),
-- ALTER COLUMN PhoneNumber SET NOT NULL;

-- 열 삭제
ALTER TABLE Employees
DROP COLUMN Address, City, PostalCode;

-- 제약 조건 추가/삭제
ALTER TABLE Employees
ADD CONSTRAINT CHK_Email_Format 
    CHECK (Email LIKE '%@%.%');

ALTER TABLE Employees
DROP CONSTRAINT CHK_Email_Format;

-- 테이블 이름 변경
EXEC sp_rename 'Employees', 'Staff';  -- SQL Server
-- ALTER TABLE Employees RENAME TO Staff;  -- PostgreSQL

-- 열 이름 변경
EXEC sp_rename 'Staff.FirstName', 'GivenName', 'COLUMN';  -- SQL Server
-- ALTER TABLE Staff RENAME COLUMN FirstName TO GivenName;  -- PostgreSQL

-- 인덱스 생성
CREATE INDEX IX_Employees_LastName ON Employees(LastName);
CREATE INDEX IX_Employees_DeptSalary ON Employees(DepartmentId, Salary DESC);
CREATE UNIQUE INDEX UIX_Employees_Email ON Employees(Email);

-- 인덱스 포함 열
CREATE INDEX IX_Employees_Name 
ON Employees(LastName, FirstName)
INCLUDE (Email, DepartmentId);  -- 포함 열은 검색에만 사용
```

### 1.5 DROP과 TRUNCATE

```sql
-- DROP: 객체 완전 삭제
DROP TABLE IF EXISTS EmployeeProjects;  -- SQL Server 2016+, PostgreSQL
DROP TABLE Employees;
DROP DATABASE TestDB;

-- TRUNCATE: 테이블 데이터만 삭제 (구조는 유지)
TRUNCATE TABLE Logs;  -- WHERE 절 사용 불가, 빠름, 롤백 불가(SQL Server)

-- DELETE와 비교
DELETE FROM Logs;  -- WHERE 절 사용 가능, 느림, 롤백 가능
DELETE FROM Logs WHERE LogDate < '2024-01-01';

-- CASCADE 옵션 (PostgreSQL)
-- TRUNCATE TABLE Orders CASCADE;  -- 참조하는 테이블도 함께 비움
```

## 2. DML (Data Manipulation Language)

### 2.1 INSERT - 데이터 삽입

```sql
-- 단일 행 삽입
INSERT INTO Departments (DepartmentName, Location, Budget)
VALUES ('IT', 'Seoul', 1000000);

-- 여러 행 삽입
INSERT INTO Departments (DepartmentName, Location, Budget)
VALUES 
    ('HR', 'Seoul', 500000),
    ('Sales', 'Busan', 750000),
    ('Marketing', 'Seoul', 600000);

-- 특정 열만 지정
INSERT INTO Employees (FirstName, LastName, Email, HireDate, Salary, DepartmentId)
VALUES ('John', 'Doe', 'john.doe@company.com', '2024-01-15', 50000, 1);

-- DEFAULT 값 사용
INSERT INTO Employees (FirstName, LastName, Email, Salary, DepartmentId)
VALUES ('Jane', 'Smith', 'jane.smith@company.com', 55000, 2);
-- HireDate는 DEFAULT인 GETDATE() 사용됨

-- 서브쿼리를 사용한 삽입
INSERT INTO EmployeeArchive (EmployeeId, FirstName, LastName, Email)
SELECT EmployeeId, FirstName, LastName, Email
FROM Employees
WHERE IsActive = 0;

-- OUTPUT 절 사용 (SQL Server)
DECLARE @NewEmployees TABLE (EmployeeId INT, Email VARCHAR(100));

INSERT INTO Employees (FirstName, LastName, Email, Salary, DepartmentId)
OUTPUT INSERTED.EmployeeId, INSERTED.Email INTO @NewEmployees
VALUES 
    ('Alice', 'Johnson', 'alice.j@company.com', 60000, 1),
    ('Bob', 'Williams', 'bob.w@company.com', 58000, 3);

SELECT * FROM @NewEmployees;

-- MERGE 문 (UPSERT)
MERGE Employees AS target
USING (VALUES 
    (100, 'Updated', 'Name', 'updated@email.com', 70000),
    (999, 'New', 'Employee', 'new@email.com', 65000)
) AS source (EmployeeId, FirstName, LastName, Email, Salary)
ON target.EmployeeId = source.EmployeeId
WHEN MATCHED THEN
    UPDATE SET 
        FirstName = source.FirstName,
        LastName = source.LastName,
        Email = source.Email,
        Salary = source.Salary,
        UpdatedAt = SYSDATETIME()
WHEN NOT MATCHED THEN
    INSERT (EmployeeId, FirstName, LastName, Email, Salary)
    VALUES (source.EmployeeId, source.FirstName, source.LastName, source.Email, source.Salary);
```

### 2.2 SELECT - 데이터 조회

```sql
-- 기본 SELECT
SELECT * FROM Employees;

-- 특정 열 선택
SELECT FirstName, LastName, Email, Salary
FROM Employees;

-- 별칭 사용
SELECT 
    FirstName AS 이름,
    LastName AS 성,
    Salary AS 연봉,
    Salary * 0.1 AS 보너스
FROM Employees;

-- DISTINCT - 중복 제거
SELECT DISTINCT DepartmentId FROM Employees;
SELECT DISTINCT DepartmentId, ManagerId FROM Employees;

-- WHERE 절
SELECT FirstName, LastName, Salary
FROM Employees
WHERE Salary > 50000;

-- 다양한 WHERE 조건
SELECT * FROM Employees
WHERE DepartmentId = 1
  AND Salary BETWEEN 40000 AND 70000
  AND Email LIKE '%@company.com'
  AND ManagerId IS NOT NULL
  AND HireDate >= '2023-01-01';

-- IN 연산자
SELECT * FROM Employees
WHERE DepartmentId IN (1, 2, 3);

SELECT * FROM Employees
WHERE DepartmentId IN (
    SELECT DepartmentId 
    FROM Departments 
    WHERE Location = 'Seoul'
);

-- ORDER BY
SELECT FirstName, LastName, Salary
FROM Employees
ORDER BY Salary DESC, LastName ASC;

-- TOP / LIMIT
SELECT TOP 10 * FROM Employees ORDER BY Salary DESC;  -- SQL Server
-- SELECT * FROM Employees ORDER BY Salary DESC LIMIT 10;  -- PostgreSQL

-- OFFSET FETCH (SQL Server 2012+)
SELECT FirstName, LastName, Salary
FROM Employees
ORDER BY Salary DESC
OFFSET 10 ROWS
FETCH NEXT 5 ROWS ONLY;

-- GROUP BY와 집계 함수
SELECT 
    DepartmentId,
    COUNT(*) AS EmployeeCount,
    AVG(Salary) AS AverageSalary,
    MIN(Salary) AS MinSalary,
    MAX(Salary) AS MaxSalary,
    SUM(Salary) AS TotalSalary
FROM Employees
GROUP BY DepartmentId;

-- HAVING 절
SELECT 
    DepartmentId,
    COUNT(*) AS EmployeeCount,
    AVG(Salary) AS AverageSalary
FROM Employees
GROUP BY DepartmentId
HAVING COUNT(*) > 5 AND AVG(Salary) > 50000;

-- 계산된 열
SELECT 
    FirstName + ' ' + LastName AS FullName,
    Salary,
    CASE 
        WHEN Salary < 40000 THEN 'Junior'
        WHEN Salary < 60000 THEN 'Mid-level'
        WHEN Salary < 80000 THEN 'Senior'
        ELSE 'Executive'
    END AS Level,
    DATEDIFF(YEAR, HireDate, GETDATE()) AS YearsOfService
FROM Employees;
```

### 2.3 UPDATE - 데이터 수정

```sql
-- 단일 행 수정
UPDATE Employees
SET Salary = 55000
WHERE EmployeeId = 1;

-- 여러 열 수정
UPDATE Employees
SET 
    Salary = Salary * 1.1,  -- 10% 인상
    UpdatedAt = SYSDATETIME()
WHERE DepartmentId = 1;

-- 조건부 수정
UPDATE Employees
SET Salary = 
    CASE 
        WHEN YearsOfService >= 10 THEN Salary * 1.15
        WHEN YearsOfService >= 5 THEN Salary * 1.10
        WHEN YearsOfService >= 2 THEN Salary * 1.05
        ELSE Salary
    END,
    UpdatedAt = SYSDATETIME()
FROM Employees e
CROSS APPLY (
    SELECT DATEDIFF(YEAR, HireDate, GETDATE()) AS YearsOfService
) AS calc;

-- JOIN을 사용한 UPDATE
UPDATE e
SET 
    e.Salary = e.Salary * 1.08,
    e.UpdatedAt = SYSDATETIME()
FROM Employees e
INNER JOIN Departments d ON e.DepartmentId = d.DepartmentId
WHERE d.DepartmentName = 'IT';

-- 서브쿼리를 사용한 UPDATE
UPDATE Employees
SET ManagerId = (
    SELECT EmployeeId 
    FROM Employees 
    WHERE Email = 'ceo@company.com'
)
WHERE DepartmentId = (
    SELECT DepartmentId 
    FROM Departments 
    WHERE DepartmentName = 'Executive'
);

-- OUTPUT 절 사용 (SQL Server)
DECLARE @UpdatedEmployees TABLE (
    EmployeeId INT,
    OldSalary DECIMAL(10, 2),
    NewSalary DECIMAL(10, 2)
);

UPDATE Employees
SET Salary = Salary * 1.05
OUTPUT 
    INSERTED.EmployeeId,
    DELETED.Salary AS OldSalary,
    INSERTED.Salary AS NewSalary
INTO @UpdatedEmployees
WHERE DepartmentId = 2;

SELECT * FROM @UpdatedEmployees;
```

### 2.4 DELETE - 데이터 삭제

```sql
-- 단일 행 삭제
DELETE FROM Employees
WHERE EmployeeId = 100;

-- 조건부 삭제
DELETE FROM Employees
WHERE IsActive = 0 
  AND DATEDIFF(YEAR, UpdatedAt, GETDATE()) > 2;

-- JOIN을 사용한 DELETE
DELETE e
FROM Employees e
INNER JOIN Departments d ON e.DepartmentId = d.DepartmentId
WHERE d.DepartmentName = 'Closed Department';

-- 서브쿼리를 사용한 DELETE
DELETE FROM EmployeeProjects
WHERE EmployeeId IN (
    SELECT EmployeeId
    FROM Employees
    WHERE IsActive = 0
);

-- OUTPUT 절 사용 (SQL Server)
DECLARE @DeletedEmployees TABLE (
    EmployeeId INT,
    FirstName NVARCHAR(50),
    LastName NVARCHAR(50),
    Email VARCHAR(100)
);

DELETE FROM Employees
OUTPUT 
    DELETED.EmployeeId,
    DELETED.FirstName,
    DELETED.LastName,
    DELETED.Email
INTO @DeletedEmployees
WHERE TerminationDate < DATEADD(YEAR, -7, GETDATE());

-- 삭제된 데이터를 아카이브 테이블로 이동
INSERT INTO EmployeesArchive
SELECT * FROM @DeletedEmployees;

-- 소프트 삭제 (실제로 삭제하지 않고 플래그만 변경)
UPDATE Employees
SET 
    IsActive = 0,
    TerminationDate = GETDATE(),
    UpdatedAt = SYSDATETIME()
WHERE EmployeeId = 50;
```

## 3. 트랜잭션 처리

```sql
-- 기본 트랜잭션
BEGIN TRANSACTION;
    UPDATE Accounts SET Balance = Balance - 1000 WHERE AccountId = 1;
    UPDATE Accounts SET Balance = Balance + 1000 WHERE AccountId = 2;
    
    -- 검증
    IF (SELECT Balance FROM Accounts WHERE AccountId = 1) < 0
    BEGIN
        ROLLBACK TRANSACTION;
        RAISERROR('Insufficient funds', 16, 1);
    END
    ELSE
    BEGIN
        COMMIT TRANSACTION;
    END

-- TRY-CATCH를 사용한 트랜잭션
BEGIN TRY
    BEGIN TRANSACTION;
        
        -- 부서 생성
        INSERT INTO Departments (DepartmentName, Location, Budget)
        VALUES ('New Department', 'Seoul', 500000);
        
        DECLARE @DeptId INT = SCOPE_IDENTITY();
        
        -- 직원들을 새 부서로 이동
        UPDATE Employees
        SET DepartmentId = @DeptId
        WHERE DepartmentId IS NULL;
        
        -- 예산 검증
        IF (SELECT SUM(Salary) FROM Employees WHERE DepartmentId = @DeptId) > 500000
            THROW 50001, 'Department budget exceeded', 1;
        
    COMMIT TRANSACTION;
    PRINT 'Transaction completed successfully';
END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0
        ROLLBACK TRANSACTION;
    
    PRINT 'Error: ' + ERROR_MESSAGE();
    THROW;
END CATCH;

-- 저장점 (SAVEPOINT) 사용
BEGIN TRANSACTION;
    INSERT INTO Orders (CustomerId, OrderDate) VALUES (1, GETDATE());
    DECLARE @OrderId INT = SCOPE_IDENTITY();
    
    SAVE TRANSACTION SavePoint1;
    
    INSERT INTO OrderDetails (OrderId, ProductId, Quantity, UnitPrice)
    VALUES (@OrderId, 1, 5, 29.99);
    
    -- 재고 확인 후 문제가 있으면 SavePoint1로 롤백
    IF (SELECT Stock FROM Products WHERE ProductId = 1) < 5
    BEGIN
        ROLLBACK TRANSACTION SavePoint1;
        -- 주문은 유지하고 주문 상세만 취소
    END
    
COMMIT TRANSACTION;
```

## 4. 실무 팁과 모범 사례

```sql
-- 1. 명명 규칙 일관성
/*
테이블: PascalCase 복수형 (Employees, Orders)
열: PascalCase (FirstName, OrderDate)
제약조건: PK_테이블명, FK_자식테이블_부모테이블, CHK_테이블_설명
인덱스: IX_테이블_열이름
*/

-- 2. NULL 처리
-- ISNULL (SQL Server) / COALESCE (표준 SQL)
SELECT 
    FirstName,
    ISNULL(MiddleName, '') AS MiddleName,
    COALESCE(PhoneNumber, MobileNumber, 'No contact') AS ContactNumber
FROM Employees;

-- 3. 날짜 처리
-- 안전한 날짜 형식 사용
INSERT INTO Events (EventDate) VALUES ('2024-12-25');  -- YYYY-MM-DD

-- 날짜 범위 쿼리
SELECT * FROM Orders
WHERE OrderDate >= '2024-01-01' 
  AND OrderDate < '2024-02-01';  -- BETWEEN 대신 사용

-- 4. 성능 고려사항
-- 인덱스가 있는 열로 검색
SELECT * FROM Employees WHERE Email = 'john@example.com';  -- Good
SELECT * FROM Employees WHERE LOWER(Email) = 'john@example.com';  -- Bad (인덱스 사용 불가)

-- 5. 보안 고려사항
-- 파라미터화된 쿼리 사용 (SQL Injection 방지)
DECLARE @Email NVARCHAR(100) = 'user@example.com';
SELECT * FROM Employees WHERE Email = @Email;  -- Good
-- EXEC('SELECT * FROM Employees WHERE Email = ''' + @Email + '''');  -- Bad
```

## 마무리

DDL과 DML은 SQL의 핵심 구성 요소입니다. DDL로 데이터베이스 구조를 설계하고, DML로 데이터를 조작합니다. 이러한 기본기를 탄탄히 다지는 것이 효율적인 데이터베이스 활용의 첫걸음입니다. 다음 장에서는 JOIN과 서브쿼리를 통한 고급 데이터 조회 기법을 학습하겠습니다.