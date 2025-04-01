# SQL 중급 - 조인과 서브쿼리

## 개요

관계형 데이터베이스의 강력함은 여러 테이블의 데이터를 연결하여 의미 있는 정보를 추출할 수 있다는 점에 있습니다. 이 장에서는 JOIN을 통한 테이블 결합과 서브쿼리를 활용한 복잡한 데이터 조회 기법을 학습합니다.

## 1. JOIN의 기본 개념

### 1.1 JOIN의 종류와 동작 원리

```sql
-- 실습용 테이블 생성
CREATE TABLE Employees (
    EmployeeId INT PRIMARY KEY,
    Name NVARCHAR(100),
    DepartmentId INT,
    ManagerId INT,
    Salary DECIMAL(10, 2),
    HireDate DATE
);

CREATE TABLE Departments (
    DepartmentId INT PRIMARY KEY,
    DepartmentName NVARCHAR(100),
    LocationId INT
);

CREATE TABLE Locations (
    LocationId INT PRIMARY KEY,
    City NVARCHAR(100),
    Country NVARCHAR(100)
);

CREATE TABLE Projects (
    ProjectId INT PRIMARY KEY,
    ProjectName NVARCHAR(200),
    StartDate DATE,
    Budget DECIMAL(15, 2)
);

CREATE TABLE EmployeeProjects (
    EmployeeId INT,
    ProjectId INT,
    Role NVARCHAR(50),
    AssignedDate DATE,
    PRIMARY KEY (EmployeeId, ProjectId)
);

-- 샘플 데이터
INSERT INTO Departments VALUES 
    (1, 'IT', 1), (2, 'HR', 1), (3, 'Sales', 2), (4, 'Marketing', NULL);

INSERT INTO Locations VALUES 
    (1, 'Seoul', 'Korea'), (2, 'Busan', 'Korea'), (3, 'Tokyo', 'Japan');

INSERT INTO Employees VALUES 
    (1, 'Kim Minsu', 1, NULL, 70000, '2020-01-15'),
    (2, 'Lee Jiyoung', 1, 1, 60000, '2021-03-20'),
    (3, 'Park Junho', 2, 1, 55000, '2019-07-10'),
    (4, 'Choi Sera', 3, 1, 65000, '2022-02-01'),
    (5, 'Jung Hoon', NULL, 1, 50000, '2023-01-10'),  -- 부서 없음
    (6, 'Kim Yuna', 5, NULL, 45000, '2023-06-15');   -- 잘못된 부서

INSERT INTO Projects VALUES 
    (1, 'Website Redesign', '2024-01-01', 500000),
    (2, 'Mobile App', '2024-02-01', 800000),
    (3, 'Data Migration', '2024-03-01', 300000);

INSERT INTO EmployeeProjects VALUES 
    (1, 1, 'Lead', '2024-01-01'),
    (2, 1, 'Developer', '2024-01-05'),
    (2, 2, 'Developer', '2024-02-01'),
    (3, 3, 'Analyst', '2024-03-01');
```

## 2. INNER JOIN

### 2.1 기본 INNER JOIN

```sql
-- 두 테이블 조인
SELECT 
    e.EmployeeId,
    e.Name,
    e.Salary,
    d.DepartmentName
FROM Employees e
INNER JOIN Departments d ON e.DepartmentId = d.DepartmentId;

-- JOIN 조건에 여러 조건 사용
SELECT 
    e.Name,
    d.DepartmentName,
    e.Salary
FROM Employees e
INNER JOIN Departments d 
    ON e.DepartmentId = d.DepartmentId 
    AND e.Salary > 55000;

-- 여러 테이블 조인
SELECT 
    e.Name AS EmployeeName,
    d.DepartmentName,
    l.City,
    l.Country
FROM Employees e
INNER JOIN Departments d ON e.DepartmentId = d.DepartmentId
INNER JOIN Locations l ON d.LocationId = l.LocationId;

-- 테이블 별칭 없이 (권장하지 않음)
SELECT 
    Employees.Name,
    Departments.DepartmentName
FROM Employees
INNER JOIN Departments 
    ON Employees.DepartmentId = Departments.DepartmentId;
```

### 2.2 SELF JOIN

```sql
-- 직원과 매니저 정보 조회
SELECT 
    e.Name AS EmployeeName,
    e.Salary AS EmployeeSalary,
    m.Name AS ManagerName,
    m.Salary AS ManagerSalary
FROM Employees e
INNER JOIN Employees m ON e.ManagerId = m.EmployeeId;

-- 같은 부서 직원들 찾기
SELECT 
    e1.Name AS Employee1,
    e2.Name AS Employee2,
    d.DepartmentName
FROM Employees e1
INNER JOIN Employees e2 
    ON e1.DepartmentId = e2.DepartmentId 
    AND e1.EmployeeId < e2.EmployeeId  -- 중복 제거
INNER JOIN Departments d ON e1.DepartmentId = d.DepartmentId
ORDER BY d.DepartmentName, e1.Name;

-- 계층 구조 조회 (3단계)
SELECT 
    e1.Name AS Level1,
    e2.Name AS Level2,
    e3.Name AS Level3
FROM Employees e1
LEFT JOIN Employees e2 ON e2.ManagerId = e1.EmployeeId
LEFT JOIN Employees e3 ON e3.ManagerId = e2.EmployeeId
WHERE e1.ManagerId IS NULL;  -- 최상위 직원부터 시작
```

## 3. OUTER JOIN

### 3.1 LEFT OUTER JOIN

```sql
-- 부서가 없는 직원도 포함
SELECT 
    e.Name,
    e.Salary,
    d.DepartmentName
FROM Employees e
LEFT OUTER JOIN Departments d ON e.DepartmentId = d.DepartmentId;

-- LEFT JOIN으로 NULL 값 찾기
SELECT 
    e.Name AS EmployeeWithoutDepartment
FROM Employees e
LEFT JOIN Departments d ON e.DepartmentId = d.DepartmentId
WHERE d.DepartmentId IS NULL;

-- 여러 LEFT JOIN
SELECT 
    e.Name,
    ISNULL(d.DepartmentName, 'No Department') AS Department,
    ISNULL(l.City, 'No Location') AS City,
    COUNT(ep.ProjectId) AS ProjectCount
FROM Employees e
LEFT JOIN Departments d ON e.DepartmentId = d.DepartmentId
LEFT JOIN Locations l ON d.LocationId = l.LocationId
LEFT JOIN EmployeeProjects ep ON e.EmployeeId = ep.EmployeeId
GROUP BY e.EmployeeId, e.Name, d.DepartmentName, l.City;
```

### 3.2 RIGHT OUTER JOIN

```sql
-- 직원이 없는 부서도 표시
SELECT 
    e.Name,
    d.DepartmentName
FROM Employees e
RIGHT OUTER JOIN Departments d ON e.DepartmentId = d.DepartmentId;

-- RIGHT JOIN을 LEFT JOIN으로 변환 (권장)
SELECT 
    e.Name,
    d.DepartmentName
FROM Departments d
LEFT JOIN Employees e ON d.DepartmentId = e.DepartmentId;

-- 프로젝트에 할당되지 않은 직원 찾기
SELECT 
    p.ProjectName,
    COUNT(ep.EmployeeId) AS AssignedEmployees
FROM Projects p
LEFT JOIN EmployeeProjects ep ON p.ProjectId = ep.ProjectId
GROUP BY p.ProjectId, p.ProjectName;
```

### 3.3 FULL OUTER JOIN

```sql
-- 모든 직원과 모든 부서 (매칭 여부 무관)
SELECT 
    ISNULL(e.Name, 'No Employee') AS EmployeeName,
    ISNULL(d.DepartmentName, 'No Department') AS DepartmentName
FROM Employees e
FULL OUTER JOIN Departments d ON e.DepartmentId = d.DepartmentId;

-- FULL JOIN으로 데이터 불일치 찾기
SELECT 
    CASE 
        WHEN e.EmployeeId IS NULL THEN 'Department without employees'
        WHEN d.DepartmentId IS NULL THEN 'Employee without valid department'
        ELSE 'Matched'
    END AS Status,
    e.Name,
    e.DepartmentId AS EmpDeptId,
    d.DepartmentId,
    d.DepartmentName
FROM Employees e
FULL OUTER JOIN Departments d ON e.DepartmentId = d.DepartmentId
WHERE e.EmployeeId IS NULL OR d.DepartmentId IS NULL;
```

## 4. CROSS JOIN

```sql
-- 모든 조합 생성
SELECT 
    e.Name,
    p.ProjectName
FROM Employees e
CROSS JOIN Projects p
ORDER BY e.Name, p.ProjectName;

-- WHERE 조건과 함께 사용
SELECT 
    e.Name,
    d.DepartmentName
FROM Employees e
CROSS JOIN Departments d
WHERE e.DepartmentId != d.DepartmentId;  -- 자기 부서 제외

-- 날짜 범위 생성에 활용
WITH DateRange AS (
    SELECT CAST('2024-01-01' AS DATE) AS DateValue
    UNION ALL
    SELECT DATEADD(DAY, 1, DateValue)
    FROM DateRange
    WHERE DateValue < '2024-01-31'
)
SELECT 
    d.DateValue,
    e.Name
FROM DateRange d
CROSS JOIN Employees e
WHERE e.DepartmentId = 1
OPTION (MAXRECURSION 31);
```

## 5. 고급 JOIN 기법

### 5.1 다중 조건 JOIN

```sql
-- 복잡한 JOIN 조건
SELECT 
    e.Name,
    p.ProjectName,
    ep.Role,
    DATEDIFF(DAY, p.StartDate, ep.AssignedDate) AS DaysAfterStart
FROM Employees e
INNER JOIN EmployeeProjects ep 
    ON e.EmployeeId = ep.EmployeeId
INNER JOIN Projects p 
    ON ep.ProjectId = p.ProjectId 
    AND ep.AssignedDate >= p.StartDate  -- 프로젝트 시작 후 할당된 경우만
    AND DATEDIFF(DAY, p.StartDate, ep.AssignedDate) <= 30;  -- 30일 이내

-- 비등가 조인
SELECT 
    e1.Name AS Employee,
    e1.Salary,
    e2.Name AS HigherPaidColleague,
    e2.Salary AS ColleagueSalary
FROM Employees e1
INNER JOIN Employees e2 
    ON e1.DepartmentId = e2.DepartmentId 
    AND e1.Salary < e2.Salary  -- 비등가 조건
ORDER BY e1.Name, e2.Salary;
```

### 5.2 JOIN과 집계 함수

```sql
-- 부서별 통계와 직원 정보
SELECT 
    e.Name,
    e.Salary,
    d.DepartmentName,
    dept_stats.AvgSalary,
    dept_stats.TotalEmployees,
    e.Salary - dept_stats.AvgSalary AS SalaryDifference
FROM Employees e
INNER JOIN Departments d ON e.DepartmentId = d.DepartmentId
INNER JOIN (
    SELECT 
        DepartmentId,
        AVG(Salary) AS AvgSalary,
        COUNT(*) AS TotalEmployees
    FROM Employees
    GROUP BY DepartmentId
) dept_stats ON e.DepartmentId = dept_stats.DepartmentId;

-- 프로젝트별 인력 현황
SELECT 
    p.ProjectName,
    p.Budget,
    COUNT(DISTINCT ep.EmployeeId) AS TeamSize,
    STRING_AGG(e.Name, ', ') AS TeamMembers,  -- SQL Server 2017+
    SUM(e.Salary) / 12 AS MonthlySalaryCost
FROM Projects p
LEFT JOIN EmployeeProjects ep ON p.ProjectId = ep.ProjectId
LEFT JOIN Employees e ON ep.EmployeeId = e.EmployeeId
GROUP BY p.ProjectId, p.ProjectName, p.Budget;
```

## 6. 서브쿼리

### 6.1 스칼라 서브쿼리

```sql
-- SELECT 절의 서브쿼리
SELECT 
    e.Name,
    e.Salary,
    (SELECT AVG(Salary) FROM Employees) AS CompanyAverage,
    e.Salary - (SELECT AVG(Salary) FROM Employees) AS Difference,
    (SELECT DepartmentName 
     FROM Departments d 
     WHERE d.DepartmentId = e.DepartmentId) AS Department
FROM Employees e;

-- 계산된 값 사용
SELECT 
    e.Name,
    e.Salary,
    (SELECT COUNT(*) 
     FROM Employees e2 
     WHERE e2.DepartmentId = e.DepartmentId 
       AND e2.Salary > e.Salary) + 1 AS SalaryRankInDept
FROM Employees e
WHERE e.DepartmentId IS NOT NULL
ORDER BY e.DepartmentId, SalaryRankInDept;
```

### 6.2 인라인 뷰 (FROM 절 서브쿼리)

```sql
-- 부서별 평균 이상 급여 받는 직원
SELECT 
    e.Name,
    e.Salary,
    dept_avg.DepartmentName,
    dept_avg.AvgSalary
FROM Employees e
INNER JOIN (
    SELECT 
        d.DepartmentId,
        d.DepartmentName,
        AVG(e.Salary) AS AvgSalary
    FROM Departments d
    INNER JOIN Employees e ON d.DepartmentId = e.DepartmentId
    GROUP BY d.DepartmentId, d.DepartmentName
) dept_avg ON e.DepartmentId = dept_avg.DepartmentId
WHERE e.Salary > dept_avg.AvgSalary;

-- 복잡한 집계를 위한 인라인 뷰
SELECT 
    DepartmentName,
    TotalEmployees,
    TotalSalary,
    AvgSalary,
    CASE 
        WHEN AvgSalary > 60000 THEN 'High'
        WHEN AvgSalary > 50000 THEN 'Medium'
        ELSE 'Low'
    END AS SalaryLevel
FROM (
    SELECT 
        d.DepartmentName,
        COUNT(e.EmployeeId) AS TotalEmployees,
        SUM(e.Salary) AS TotalSalary,
        AVG(e.Salary) AS AvgSalary
    FROM Departments d
    LEFT JOIN Employees e ON d.DepartmentId = e.DepartmentId
    GROUP BY d.DepartmentId, d.DepartmentName
) dept_summary;
```

### 6.3 WHERE 절 서브쿼리

```sql
-- 단일 행 서브쿼리
SELECT Name, Salary
FROM Employees
WHERE Salary = (SELECT MAX(Salary) FROM Employees);

-- 다중 행 서브쿼리 - IN
SELECT Name, DepartmentId
FROM Employees
WHERE DepartmentId IN (
    SELECT DepartmentId
    FROM Departments
    WHERE LocationId = 1
);

-- ANY/SOME 연산자
SELECT Name, Salary
FROM Employees
WHERE Salary > ANY (
    SELECT Salary
    FROM Employees
    WHERE DepartmentId = 2
);

-- ALL 연산자
SELECT Name, Salary
FROM Employees
WHERE Salary > ALL (
    SELECT Salary
    FROM Employees
    WHERE DepartmentId = 2
);
```

### 6.4 상관 서브쿼리

```sql
-- EXISTS 연산자
SELECT e.Name, e.DepartmentId
FROM Employees e
WHERE EXISTS (
    SELECT 1
    FROM EmployeeProjects ep
    WHERE ep.EmployeeId = e.EmployeeId
);

-- NOT EXISTS
SELECT d.DepartmentName
FROM Departments d
WHERE NOT EXISTS (
    SELECT 1
    FROM Employees e
    WHERE e.DepartmentId = d.DepartmentId
);

-- 상관 서브쿼리로 순위 계산
SELECT 
    e1.Name,
    e1.Salary,
    (SELECT COUNT(*) + 1
     FROM Employees e2
     WHERE e2.Salary > e1.Salary) AS OverallRank,
    (SELECT COUNT(*) + 1
     FROM Employees e2
     WHERE e2.DepartmentId = e1.DepartmentId
       AND e2.Salary > e1.Salary) AS DeptRank
FROM Employees e1
ORDER BY OverallRank;
```

## 7. 집합 연산자

### 7.1 UNION과 UNION ALL

```sql
-- UNION - 중복 제거
SELECT Name, 'Employee' AS Type
FROM Employees
WHERE DepartmentId = 1
UNION
SELECT DepartmentName, 'Department' AS Type
FROM Departments
WHERE LocationId = 1;

-- UNION ALL - 중복 포함
SELECT EmployeeId, Name, 'Current' AS Status
FROM Employees
WHERE IsActive = 1
UNION ALL
SELECT EmployeeId, Name, 'Former' AS Status
FROM EmployeesArchive;

-- 여러 테이블 통합
SELECT 'Employee' AS Category, Name, Email
FROM Employees
UNION ALL
SELECT 'Customer' AS Category, Name, Email
FROM Customers
UNION ALL
SELECT 'Vendor' AS Category, CompanyName, ContactEmail
FROM Vendors
ORDER BY Category, Name;
```

### 7.2 INTERSECT와 EXCEPT

```sql
-- INTERSECT - 교집합
SELECT DepartmentId
FROM Employees
WHERE Salary > 60000
INTERSECT
SELECT DepartmentId
FROM Departments
WHERE LocationId = 1;

-- EXCEPT (MINUS in Oracle) - 차집합
SELECT EmployeeId
FROM Employees
EXCEPT
SELECT EmployeeId
FROM EmployeeProjects;

-- 복잡한 집합 연산
-- IT 부서에만 있고 프로젝트에 참여하지 않는 직원
SELECT e.EmployeeId, e.Name
FROM Employees e
WHERE e.DepartmentId = 1
EXCEPT
SELECT e.EmployeeId, e.Name
FROM Employees e
INNER JOIN EmployeeProjects ep ON e.EmployeeId = ep.EmployeeId;
```

## 8. 실무 활용 예제

### 8.1 복잡한 보고서 쿼리

```sql
-- 종합 직원 현황 보고서
WITH EmployeeStats AS (
    SELECT 
        e.EmployeeId,
        e.Name,
        e.Salary,
        d.DepartmentName,
        l.City,
        m.Name AS ManagerName,
        COUNT(DISTINCT ep.ProjectId) AS ProjectCount,
        RANK() OVER (PARTITION BY e.DepartmentId ORDER BY e.Salary DESC) AS SalaryRank
    FROM Employees e
    LEFT JOIN Departments d ON e.DepartmentId = d.DepartmentId
    LEFT JOIN Locations l ON d.LocationId = l.LocationId
    LEFT JOIN Employees m ON e.ManagerId = m.EmployeeId
    LEFT JOIN EmployeeProjects ep ON e.EmployeeId = ep.EmployeeId
    GROUP BY e.EmployeeId, e.Name, e.Salary, e.DepartmentId, 
             d.DepartmentName, l.City, m.Name
)
SELECT 
    *,
    CASE 
        WHEN ProjectCount = 0 THEN 'Available'
        WHEN ProjectCount = 1 THEN 'Assigned'
        ELSE 'Overloaded'
    END AS WorkloadStatus
FROM EmployeeStats
ORDER BY DepartmentName, SalaryRank;
```

### 8.2 데이터 품질 검증

```sql
-- 데이터 무결성 체크
SELECT 'Employees without department' AS Issue, COUNT(*) AS Count
FROM Employees WHERE DepartmentId IS NULL
UNION ALL
SELECT 'Employees with invalid department', COUNT(*)
FROM Employees e
WHERE NOT EXISTS (SELECT 1 FROM Departments d WHERE d.DepartmentId = e.DepartmentId)
UNION ALL
SELECT 'Departments without employees', COUNT(*)
FROM Departments d
WHERE NOT EXISTS (SELECT 1 FROM Employees e WHERE e.DepartmentId = d.DepartmentId)
UNION ALL
SELECT 'Circular manager reference', COUNT(*)
FROM Employees e1
WHERE EXISTS (
    SELECT 1 FROM Employees e2 
    WHERE e2.EmployeeId = e1.ManagerId 
    AND e2.ManagerId = e1.EmployeeId
);
```

### 8.3 성능 최적화된 쿼리

```sql
-- EXISTS vs IN (EXISTS가 일반적으로 더 빠름)
-- 느린 버전
SELECT * FROM Employees
WHERE DepartmentId IN (
    SELECT DepartmentId FROM Departments WHERE LocationId = 1
);

-- 빠른 버전
SELECT e.* FROM Employees e
WHERE EXISTS (
    SELECT 1 FROM Departments d 
    WHERE d.DepartmentId = e.DepartmentId 
    AND d.LocationId = 1
);

-- JOIN vs 서브쿼리 (JOIN이 일반적으로 더 빠름)
-- 느린 버전
SELECT 
    e.Name,
    (SELECT DepartmentName FROM Departments d WHERE d.DepartmentId = e.DepartmentId) AS Dept
FROM Employees e;

-- 빠른 버전
SELECT e.Name, d.DepartmentName
FROM Employees e
LEFT JOIN Departments d ON e.DepartmentId = d.DepartmentId;
```

## 마무리

JOIN과 서브쿼리는 관계형 데이터베이스에서 데이터를 효과적으로 조회하는 핵심 기술입니다. 적절한 JOIN 타입 선택, 효율적인 서브쿼리 작성, 그리고 성능을 고려한 쿼리 설계가 중요합니다. 다음 장에서는 윈도우 함수와 CTE를 활용한 고급 SQL 기법을 학습하겠습니다.