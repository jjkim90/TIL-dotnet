# 저장 프로시저와 함수

## 개요

저장 프로시저와 함수는 데이터베이스에 저장되는 프로그래밍 객체로, 복잡한 비즈니스 로직을 캡슐화하고 재사용성을 높이는 핵심 기능입니다. 이 장에서는 저장 프로시저, 사용자 정의 함수, 트리거의 작성과 활용, 그리고 동적 SQL의 안전한 사용법을 학습합니다.

## 1. 저장 프로시저 기초

### 1.1 기본 저장 프로시저

```sql
-- 기본 저장 프로시저 생성
CREATE PROCEDURE sp_GetEmployeesByDepartment
    @DepartmentId INT
AS
BEGIN
    SET NOCOUNT ON;  -- 영향받은 행 수 메시지 숨김
    
    SELECT 
        EmployeeId,
        FirstName + ' ' + LastName AS FullName,
        Email,
        HireDate,
        Salary
    FROM Employees
    WHERE DepartmentId = @DepartmentId
    ORDER BY LastName, FirstName;
END;

-- 저장 프로시저 실행
EXEC sp_GetEmployeesByDepartment @DepartmentId = 1;

-- 여러 매개변수를 가진 프로시저
CREATE PROCEDURE sp_GetEmployeesBySalaryRange
    @MinSalary DECIMAL(10, 2),
    @MaxSalary DECIMAL(10, 2),
    @DepartmentId INT = NULL  -- 선택적 매개변수
AS
BEGIN
    SET NOCOUNT ON;
    
    SELECT 
        e.EmployeeId,
        e.FirstName + ' ' + e.LastName AS FullName,
        e.Salary,
        d.DepartmentName
    FROM Employees e
    LEFT JOIN Departments d ON e.DepartmentId = d.DepartmentId
    WHERE e.Salary BETWEEN @MinSalary AND @MaxSalary
      AND (@DepartmentId IS NULL OR e.DepartmentId = @DepartmentId);
END;

-- 다양한 실행 방법
EXEC sp_GetEmployeesBySalaryRange 40000, 70000;
EXEC sp_GetEmployeesBySalaryRange @MinSalary = 40000, @MaxSalary = 70000, @DepartmentId = 1;
```

### 1.2 OUTPUT 매개변수

```sql
-- OUTPUT 매개변수를 사용하는 프로시저
CREATE PROCEDURE sp_InsertEmployee
    @FirstName NVARCHAR(50),
    @LastName NVARCHAR(50),
    @Email VARCHAR(100),
    @Salary DECIMAL(10, 2),
    @DepartmentId INT,
    @NewEmployeeId INT OUTPUT,
    @ErrorMessage NVARCHAR(500) OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    
    BEGIN TRY
        -- 이메일 중복 체크
        IF EXISTS (SELECT 1 FROM Employees WHERE Email = @Email)
        BEGIN
            SET @ErrorMessage = 'Email already exists';
            SET @NewEmployeeId = -1;
            RETURN;
        END
        
        -- 직원 삽입
        INSERT INTO Employees (FirstName, LastName, Email, Salary, DepartmentId, HireDate)
        VALUES (@FirstName, @LastName, @Email, @Salary, @DepartmentId, GETDATE());
        
        SET @NewEmployeeId = SCOPE_IDENTITY();
        SET @ErrorMessage = NULL;
        
    END TRY
    BEGIN CATCH
        SET @NewEmployeeId = -1;
        SET @ErrorMessage = ERROR_MESSAGE();
    END CATCH
END;

-- OUTPUT 매개변수 사용
DECLARE @EmpId INT, @ErrMsg NVARCHAR(500);

EXEC sp_InsertEmployee 
    @FirstName = 'John',
    @LastName = 'Doe',
    @Email = 'john.doe@company.com',
    @Salary = 55000,
    @DepartmentId = 1,
    @NewEmployeeId = @EmpId OUTPUT,
    @ErrorMessage = @ErrMsg OUTPUT;

IF @EmpId > 0
    PRINT 'Employee created with ID: ' + CAST(@EmpId AS VARCHAR(10));
ELSE
    PRINT 'Error: ' + @ErrMsg;
```

### 1.3 반환 값과 결과 집합

```sql
-- 반환 값을 사용하는 프로시저
CREATE PROCEDURE sp_UpdateEmployeeSalary
    @EmployeeId INT,
    @NewSalary DECIMAL(10, 2)
AS
BEGIN
    SET NOCOUNT ON;
    
    -- 직원 존재 확인
    IF NOT EXISTS (SELECT 1 FROM Employees WHERE EmployeeId = @EmployeeId)
        RETURN -1;  -- 직원 없음
    
    -- 급여 범위 확인
    IF @NewSalary < 30000 OR @NewSalary > 500000
        RETURN -2;  -- 급여 범위 오류
    
    -- 현재 급여 확인
    DECLARE @CurrentSalary DECIMAL(10, 2);
    SELECT @CurrentSalary = Salary FROM Employees WHERE EmployeeId = @EmployeeId;
    
    -- 50% 이상 인상 제한
    IF @NewSalary > @CurrentSalary * 1.5
        RETURN -3;  -- 과도한 인상
    
    -- 급여 업데이트
    UPDATE Employees
    SET Salary = @NewSalary,
        UpdatedAt = GETDATE()
    WHERE EmployeeId = @EmployeeId;
    
    RETURN 0;  -- 성공
END;

-- 반환 값 처리
DECLARE @ReturnValue INT;
EXEC @ReturnValue = sp_UpdateEmployeeSalary @EmployeeId = 1, @NewSalary = 65000;

SELECT CASE @ReturnValue
    WHEN 0 THEN 'Success'
    WHEN -1 THEN 'Employee not found'
    WHEN -2 THEN 'Salary out of range'
    WHEN -3 THEN 'Excessive raise'
    ELSE 'Unknown error'
END AS Result;
```

## 2. 고급 저장 프로시저 기법

### 2.1 트랜잭션 처리

```sql
CREATE PROCEDURE sp_TransferEmployee
    @EmployeeId INT,
    @NewDepartmentId INT,
    @NewManagerId INT = NULL
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;  -- 오류 시 자동 롤백
    
    BEGIN TRY
        BEGIN TRANSACTION;
        
        -- 현재 정보 저장 (감사 목적)
        DECLARE @OldDepartmentId INT, @OldManagerId INT;
        
        SELECT 
            @OldDepartmentId = DepartmentId,
            @OldManagerId = ManagerId
        FROM Employees
        WHERE EmployeeId = @EmployeeId;
        
        IF @OldDepartmentId IS NULL
        BEGIN
            RAISERROR('Employee not found', 16, 1);
        END
        
        -- 이력 테이블에 기록
        INSERT INTO EmployeeTransferHistory 
            (EmployeeId, FromDepartmentId, ToDepartmentId, 
             FromManagerId, ToManagerId, TransferDate)
        VALUES 
            (@EmployeeId, @OldDepartmentId, @NewDepartmentId,
             @OldManagerId, @NewManagerId, GETDATE());
        
        -- 직원 정보 업데이트
        UPDATE Employees
        SET DepartmentId = @NewDepartmentId,
            ManagerId = @NewManagerId,
            UpdatedAt = GETDATE()
        WHERE EmployeeId = @EmployeeId;
        
        -- 부서 인원 수 확인
        DECLARE @DeptCount INT;
        SELECT @DeptCount = COUNT(*) 
        FROM Employees 
        WHERE DepartmentId = @NewDepartmentId;
        
        IF @DeptCount > 50
        BEGIN
            RAISERROR('Department is full (max 50 employees)', 16, 1);
        END
        
        COMMIT TRANSACTION;
        
        -- 성공 시 결과 반환
        SELECT 
            e.EmployeeId,
            e.FirstName + ' ' + e.LastName AS EmployeeName,
            od.DepartmentName AS OldDepartment,
            nd.DepartmentName AS NewDepartment,
            m.FirstName + ' ' + m.LastName AS NewManager
        FROM Employees e
        LEFT JOIN Departments od ON od.DepartmentId = @OldDepartmentId
        LEFT JOIN Departments nd ON nd.DepartmentId = @NewDepartmentId
        LEFT JOIN Employees m ON m.EmployeeId = @NewManagerId
        WHERE e.EmployeeId = @EmployeeId;
        
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        
        DECLARE @ErrorMessage NVARCHAR(4000) = ERROR_MESSAGE();
        DECLARE @ErrorSeverity INT = ERROR_SEVERITY();
        DECLARE @ErrorState INT = ERROR_STATE();
        
        RAISERROR(@ErrorMessage, @ErrorSeverity, @ErrorState);
    END CATCH
END;
```

### 2.2 테이블 반환 프로시저

```sql
-- 임시 테이블을 사용하는 프로시저
CREATE PROCEDURE sp_GetDepartmentSummary
    @IncludeInactive BIT = 0
AS
BEGIN
    SET NOCOUNT ON;
    
    -- 임시 테이블 생성
    CREATE TABLE #DeptSummary (
        DepartmentId INT,
        DepartmentName NVARCHAR(100),
        EmployeeCount INT,
        AvgSalary DECIMAL(10, 2),
        MinSalary DECIMAL(10, 2),
        MaxSalary DECIMAL(10, 2),
        TotalSalary DECIMAL(15, 2),
        RecentHires INT
    );
    
    -- 데이터 수집
    INSERT INTO #DeptSummary
    SELECT 
        d.DepartmentId,
        d.DepartmentName,
        COUNT(e.EmployeeId) AS EmployeeCount,
        AVG(e.Salary) AS AvgSalary,
        MIN(e.Salary) AS MinSalary,
        MAX(e.Salary) AS MaxSalary,
        SUM(e.Salary) AS TotalSalary,
        SUM(CASE WHEN e.HireDate >= DATEADD(MONTH, -6, GETDATE()) THEN 1 ELSE 0 END) AS RecentHires
    FROM Departments d
    LEFT JOIN Employees e ON d.DepartmentId = e.DepartmentId
        AND (@IncludeInactive = 1 OR e.IsActive = 1)
    GROUP BY d.DepartmentId, d.DepartmentName;
    
    -- 추가 계산
    ALTER TABLE #DeptSummary ADD 
        BudgetUtilization DECIMAL(5, 2),
        SalaryRank INT;
    
    UPDATE ds
    SET BudgetUtilization = 
        CASE WHEN d.Budget > 0 
             THEN (ds.TotalSalary * 12) / d.Budget * 100 
             ELSE NULL 
        END
    FROM #DeptSummary ds
    INNER JOIN Departments d ON ds.DepartmentId = d.DepartmentId;
    
    -- 순위 매기기
    UPDATE #DeptSummary
    SET SalaryRank = ranking.Rank
    FROM #DeptSummary ds
    INNER JOIN (
        SELECT 
            DepartmentId,
            RANK() OVER (ORDER BY AvgSalary DESC) AS Rank
        FROM #DeptSummary
    ) ranking ON ds.DepartmentId = ranking.DepartmentId;
    
    -- 결과 반환
    SELECT * FROM #DeptSummary
    ORDER BY SalaryRank;
    
    -- 정리
    DROP TABLE #DeptSummary;
END;
```

## 3. 사용자 정의 함수

### 3.1 스칼라 함수

```sql
-- 간단한 스칼라 함수
CREATE FUNCTION fn_GetFullName
(
    @FirstName NVARCHAR(50),
    @LastName NVARCHAR(50)
)
RETURNS NVARCHAR(101)
AS
BEGIN
    RETURN @FirstName + ' ' + @LastName;
END;

-- 사용 예
SELECT 
    EmployeeId,
    dbo.fn_GetFullName(FirstName, LastName) AS FullName
FROM Employees;

-- 복잡한 비즈니스 로직을 포함한 함수
CREATE FUNCTION fn_CalculateEmployeeBonus
(
    @EmployeeId INT,
    @Year INT
)
RETURNS DECIMAL(10, 2)
AS
BEGIN
    DECLARE @Bonus DECIMAL(10, 2) = 0;
    DECLARE @BaseSalary DECIMAL(10, 2);
    DECLARE @PerformanceScore INT;
    DECLARE @YearsOfService INT;
    
    -- 기본 급여 조회
    SELECT @BaseSalary = Salary
    FROM Employees
    WHERE EmployeeId = @EmployeeId;
    
    IF @BaseSalary IS NULL
        RETURN 0;
    
    -- 근속 연수 계산
    SELECT @YearsOfService = DATEDIFF(YEAR, HireDate, GETDATE())
    FROM Employees
    WHERE EmployeeId = @EmployeeId;
    
    -- 성과 점수 조회 (가상의 테이블)
    SELECT @PerformanceScore = Score
    FROM PerformanceReviews
    WHERE EmployeeId = @EmployeeId AND Year = @Year;
    
    -- 보너스 계산
    SET @Bonus = @BaseSalary * 
        CASE 
            WHEN @PerformanceScore >= 90 THEN 0.20
            WHEN @PerformanceScore >= 80 THEN 0.15
            WHEN @PerformanceScore >= 70 THEN 0.10
            WHEN @PerformanceScore >= 60 THEN 0.05
            ELSE 0
        END;
    
    -- 근속 보너스 추가
    SET @Bonus = @Bonus + (@YearsOfService * 1000);
    
    RETURN @Bonus;
END;

-- 날짜 관련 유틸리티 함수
CREATE FUNCTION fn_GetWorkingDays
(
    @StartDate DATE,
    @EndDate DATE
)
RETURNS INT
AS
BEGIN
    DECLARE @Days INT = 0;
    DECLARE @CurrentDate DATE = @StartDate;
    
    WHILE @CurrentDate <= @EndDate
    BEGIN
        IF DATEPART(WEEKDAY, @CurrentDate) NOT IN (1, 7)  -- 주말 제외
            SET @Days = @Days + 1;
        
        SET @CurrentDate = DATEADD(DAY, 1, @CurrentDate);
    END
    
    RETURN @Days;
END;
```

### 3.2 테이블 반환 함수 (TVF)

```sql
-- 인라인 테이블 반환 함수 (더 효율적)
CREATE FUNCTION fn_GetEmployeesByHireYear
(
    @Year INT
)
RETURNS TABLE
AS
RETURN
(
    SELECT 
        e.EmployeeId,
        e.FirstName + ' ' + e.LastName AS FullName,
        e.HireDate,
        e.Salary,
        d.DepartmentName
    FROM Employees e
    LEFT JOIN Departments d ON e.DepartmentId = d.DepartmentId
    WHERE YEAR(e.HireDate) = @Year
);

-- 사용 예
SELECT * FROM dbo.fn_GetEmployeesByHireYear(2023);

-- 다중 문장 테이블 반환 함수
CREATE FUNCTION fn_GetEmployeeHierarchy
(
    @ManagerId INT
)
RETURNS @Hierarchy TABLE
(
    EmployeeId INT,
    EmployeeName NVARCHAR(101),
    Level INT,
    ManagerId INT,
    Path NVARCHAR(MAX)
)
AS
BEGIN
    WITH EmployeeHierarchy AS (
        -- Anchor
        SELECT 
            EmployeeId,
            FirstName + ' ' + LastName AS EmployeeName,
            0 AS Level,
            ManagerId,
            CAST(FirstName + ' ' + LastName AS NVARCHAR(MAX)) AS Path
        FROM Employees
        WHERE EmployeeId = @ManagerId
        
        UNION ALL
        
        -- Recursive
        SELECT 
            e.EmployeeId,
            e.FirstName + ' ' + e.LastName,
            eh.Level + 1,
            e.ManagerId,
            eh.Path + ' > ' + e.FirstName + ' ' + e.LastName
        FROM Employees e
        INNER JOIN EmployeeHierarchy eh ON e.ManagerId = eh.EmployeeId
    )
    INSERT INTO @Hierarchy
    SELECT * FROM EmployeeHierarchy;
    
    RETURN;
END;

-- 복잡한 비즈니스 로직을 포함한 TVF
CREATE FUNCTION fn_GetSalesAnalysis
(
    @StartDate DATE,
    @EndDate DATE,
    @MinAmount DECIMAL(10, 2) = 0
)
RETURNS @Analysis TABLE
(
    Period NVARCHAR(20),
    Region NVARCHAR(50),
    TotalSales DECIMAL(15, 2),
    TransactionCount INT,
    AvgTransaction DECIMAL(10, 2),
    TopProduct NVARCHAR(100),
    GrowthRate DECIMAL(5, 2)
)
AS
BEGIN
    -- 현재 기간 데이터
    INSERT INTO @Analysis
    SELECT 
        'Current' AS Period,
        Region,
        SUM(Amount) AS TotalSales,
        COUNT(*) AS TransactionCount,
        AVG(Amount) AS AvgTransaction,
        NULL AS TopProduct,
        NULL AS GrowthRate
    FROM Sales
    WHERE SaleDate BETWEEN @StartDate AND @EndDate
      AND Amount >= @MinAmount
    GROUP BY Region;
    
    -- 이전 기간 데이터 (비교용)
    DECLARE @PreviousStart DATE = DATEADD(YEAR, -1, @StartDate);
    DECLARE @PreviousEnd DATE = DATEADD(YEAR, -1, @EndDate);
    
    -- 성장률 계산
    UPDATE a
    SET GrowthRate = 
        CASE WHEN p.TotalSales > 0 
             THEN ((a.TotalSales - p.TotalSales) / p.TotalSales) * 100
             ELSE NULL 
        END
    FROM @Analysis a
    LEFT JOIN (
        SELECT 
            Region,
            SUM(Amount) AS TotalSales
        FROM Sales
        WHERE SaleDate BETWEEN @PreviousStart AND @PreviousEnd
        GROUP BY Region
    ) p ON a.Region = p.Region
    WHERE a.Period = 'Current';
    
    -- 최고 판매 상품 업데이트
    UPDATE a
    SET TopProduct = tp.ProductName
    FROM @Analysis a
    INNER JOIN (
        SELECT 
            s.Region,
            p.ProductName,
            ROW_NUMBER() OVER (PARTITION BY s.Region ORDER BY SUM(s.Amount) DESC) AS rn
        FROM Sales s
        INNER JOIN Products p ON s.ProductId = p.ProductId
        WHERE s.SaleDate BETWEEN @StartDate AND @EndDate
        GROUP BY s.Region, p.ProductName
    ) tp ON a.Region = tp.Region AND tp.rn = 1;
    
    RETURN;
END;
```

## 4. 트리거

### 4.1 DML 트리거

```sql
-- INSERT 트리거
CREATE TRIGGER trg_Employees_Insert
ON Employees
AFTER INSERT
AS
BEGIN
    SET NOCOUNT ON;
    
    -- 감사 로그 기록
    INSERT INTO AuditLog (TableName, Action, UserId, ActionDate, Details)
    SELECT 
        'Employees',
        'INSERT',
        SYSTEM_USER,
        GETDATE(),
        'New employee: ' + i.FirstName + ' ' + i.LastName + ' (ID: ' + CAST(i.EmployeeId AS VARCHAR(10)) + ')'
    FROM inserted i;
    
    -- 부서 통계 업데이트
    UPDATE d
    SET d.EmployeeCount = (
        SELECT COUNT(*) 
        FROM Employees 
        WHERE DepartmentId = d.DepartmentId
    )
    FROM Departments d
    WHERE d.DepartmentId IN (SELECT DISTINCT DepartmentId FROM inserted);
END;

-- UPDATE 트리거 (변경 이력 추적)
CREATE TRIGGER trg_Employees_Update
ON Employees
AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;
    
    -- 급여 변경 이력 기록
    IF UPDATE(Salary)
    BEGIN
        INSERT INTO SalaryHistory (EmployeeId, OldSalary, NewSalary, ChangedBy, ChangedDate)
        SELECT 
            i.EmployeeId,
            d.Salary AS OldSalary,
            i.Salary AS NewSalary,
            SYSTEM_USER,
            GETDATE()
        FROM inserted i
        INNER JOIN deleted d ON i.EmployeeId = d.EmployeeId
        WHERE i.Salary != d.Salary;
    END
    
    -- 중요 필드 변경 알림
    IF UPDATE(DepartmentId) OR UPDATE(ManagerId)
    BEGIN
        INSERT INTO Notifications (Type, Message, CreatedDate, IsRead)
        SELECT 
            'EmployeeTransfer',
            'Employee ' + i.FirstName + ' ' + i.LastName + 
            CASE 
                WHEN i.DepartmentId != d.DepartmentId THEN ' transferred to new department'
                WHEN i.ManagerId != d.ManagerId THEN ' assigned to new manager'
                ELSE ' updated'
            END,
            GETDATE(),
            0
        FROM inserted i
        INNER JOIN deleted d ON i.EmployeeId = d.EmployeeId
        WHERE i.DepartmentId != d.DepartmentId OR i.ManagerId != d.ManagerId;
    END
END;

-- INSTEAD OF 트리거
CREATE TRIGGER trg_Employees_Delete
ON Employees
INSTEAD OF DELETE
AS
BEGIN
    SET NOCOUNT ON;
    
    -- 실제로 삭제하지 않고 비활성화
    UPDATE e
    SET 
        IsActive = 0,
        TerminationDate = GETDATE(),
        UpdatedAt = GETDATE()
    FROM Employees e
    INNER JOIN deleted d ON e.EmployeeId = d.EmployeeId;
    
    -- 삭제 시도 로그
    INSERT INTO AuditLog (TableName, Action, UserId, ActionDate, Details)
    SELECT 
        'Employees',
        'SOFT_DELETE',
        SYSTEM_USER,
        GETDATE(),
        'Deactivated employee: ' + d.FirstName + ' ' + d.LastName
    FROM deleted d;
END;
```

### 4.2 DDL 트리거

```sql
-- 데이터베이스 레벨 DDL 트리거
CREATE TRIGGER trg_DDL_TableChanges
ON DATABASE
FOR CREATE_TABLE, ALTER_TABLE, DROP_TABLE
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @EventData XML = EVENTDATA();
    
    INSERT INTO DDLAuditLog (
        EventType,
        ObjectName,
        ObjectType,
        SqlCommand,
        LoginName,
        EventDate
    )
    SELECT
        @EventData.value('(/EVENT_INSTANCE/EventType)[1]', 'NVARCHAR(100)'),
        @EventData.value('(/EVENT_INSTANCE/ObjectName)[1]', 'NVARCHAR(100)'),
        @EventData.value('(/EVENT_INSTANCE/ObjectType)[1]', 'NVARCHAR(100)'),
        @EventData.value('(/EVENT_INSTANCE/TSQLCommand)[1]', 'NVARCHAR(MAX)'),
        @EventData.value('(/EVENT_INSTANCE/LoginName)[1]', 'NVARCHAR(100)'),
        GETDATE();
    
    -- 특정 테이블 삭제 방지
    IF @EventData.value('(/EVENT_INSTANCE/EventType)[1]', 'NVARCHAR(100)') = 'DROP_TABLE'
       AND @EventData.value('(/EVENT_INSTANCE/ObjectName)[1]', 'NVARCHAR(100)') IN ('Employees', 'Departments')
    BEGIN
        RAISERROR('Cannot drop critical tables', 16, 1);
        ROLLBACK;
    END
END;
```

## 5. 동적 SQL

### 5.1 기본 동적 SQL

```sql
-- 동적 SQL 실행 프로시저
CREATE PROCEDURE sp_DynamicSearch
    @TableName NVARCHAR(128),
    @SearchColumn NVARCHAR(128),
    @SearchValue NVARCHAR(100)
AS
BEGIN
    SET NOCOUNT ON;
    
    -- 테이블과 컬럼 존재 확인
    IF NOT EXISTS (
        SELECT 1 
        FROM INFORMATION_SCHEMA.COLUMNS 
        WHERE TABLE_NAME = @TableName 
          AND COLUMN_NAME = @SearchColumn
    )
    BEGIN
        RAISERROR('Invalid table or column name', 16, 1);
        RETURN;
    END
    
    -- 동적 SQL 구성
    DECLARE @SQL NVARCHAR(MAX);
    
    SET @SQL = N'SELECT * FROM ' + QUOTENAME(@TableName) + 
               N' WHERE ' + QUOTENAME(@SearchColumn) + N' LIKE @Value';
    
    -- 매개변수화된 실행 (SQL Injection 방지)
    EXEC sp_executesql 
        @SQL, 
        N'@Value NVARCHAR(100)', 
        @Value = '%' + @SearchValue + '%';
END;

-- 복잡한 동적 쿼리 생성
CREATE PROCEDURE sp_GenerateReport
    @Columns NVARCHAR(MAX),
    @TableName NVARCHAR(128),
    @WhereClause NVARCHAR(MAX) = NULL,
    @OrderBy NVARCHAR(256) = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @SQL NVARCHAR(MAX);
    DECLARE @SafeColumns NVARCHAR(MAX) = '';
    
    -- 컬럼 검증 및 안전한 컬럼 목록 생성
    DECLARE @ColumnTable TABLE (ColumnName NVARCHAR(128));
    
    INSERT INTO @ColumnTable
    SELECT value 
    FROM STRING_SPLIT(@Columns, ',');
    
    SELECT @SafeColumns = @SafeColumns + QUOTENAME(TRIM(c.ColumnName)) + ','
    FROM @ColumnTable c
    WHERE EXISTS (
        SELECT 1 
        FROM INFORMATION_SCHEMA.COLUMNS 
        WHERE TABLE_NAME = @TableName 
          AND COLUMN_NAME = TRIM(c.ColumnName)
    );
    
    -- 마지막 쉼표 제거
    SET @SafeColumns = LEFT(@SafeColumns, LEN(@SafeColumns) - 1);
    
    -- 동적 SQL 구성
    SET @SQL = N'SELECT ' + @SafeColumns + N' FROM ' + QUOTENAME(@TableName);
    
    -- WHERE 절 추가
    IF @WhereClause IS NOT NULL
    BEGIN
        -- 위험한 키워드 체크
        IF @WhereClause LIKE '%DELETE%' OR 
           @WhereClause LIKE '%DROP%' OR 
           @WhereClause LIKE '%EXEC%' OR
           @WhereClause LIKE '%INSERT%' OR
           @WhereClause LIKE '%UPDATE%'
        BEGIN
            RAISERROR('Dangerous keywords detected in WHERE clause', 16, 1);
            RETURN;
        END
        
        SET @SQL = @SQL + N' WHERE ' + @WhereClause;
    END
    
    -- ORDER BY 추가
    IF @OrderBy IS NOT NULL
    BEGIN
        SET @SQL = @SQL + N' ORDER BY ' + QUOTENAME(@OrderBy);
    END
    
    -- 실행
    EXEC sp_executesql @SQL;
END;
```

### 5.2 안전한 동적 SQL 패턴

```sql
-- 매개변수화된 동적 SQL
CREATE PROCEDURE sp_SafeDynamicUpdate
    @TableName NVARCHAR(128),
    @IdColumn NVARCHAR(128),
    @IdValue INT,
    @UpdateColumn NVARCHAR(128),
    @UpdateValue NVARCHAR(MAX)
AS
BEGIN
    SET NOCOUNT ON;
    
    -- 화이트리스트 방식의 테이블 검증
    IF @TableName NOT IN ('Employees', 'Departments', 'Projects')
    BEGIN
        RAISERROR('Table not allowed for updates', 16, 1);
        RETURN;
    END
    
    -- 컬럼 존재 확인
    IF NOT EXISTS (
        SELECT 1 
        FROM INFORMATION_SCHEMA.COLUMNS 
        WHERE TABLE_NAME = @TableName 
          AND COLUMN_NAME = @UpdateColumn
    )
    BEGIN
        RAISERROR('Invalid column name', 16, 1);
        RETURN;
    END
    
    DECLARE @SQL NVARCHAR(MAX);
    
    -- 안전한 동적 SQL 구성
    SET @SQL = N'UPDATE ' + QUOTENAME(@TableName) + 
               N' SET ' + QUOTENAME(@UpdateColumn) + N' = @Value, UpdatedAt = GETDATE()' +
               N' WHERE ' + QUOTENAME(@IdColumn) + N' = @Id';
    
    -- 매개변수화된 실행
    EXEC sp_executesql 
        @SQL, 
        N'@Value NVARCHAR(MAX), @Id INT', 
        @Value = @UpdateValue,
        @Id = @IdValue;
    
    -- 영향받은 행 수 반환
    SELECT @@ROWCOUNT AS RowsAffected;
END;

-- 메타데이터 기반 동적 쿼리
CREATE PROCEDURE sp_GetTableInfo
    @SchemaName NVARCHAR(128) = 'dbo',
    @TablePattern NVARCHAR(128) = '%'
AS
BEGIN
    SET NOCOUNT ON;
    
    -- 스키마별 테이블 정보
    SELECT 
        t.TABLE_SCHEMA,
        t.TABLE_NAME,
        t.TABLE_TYPE,
        (
            SELECT COUNT(*) 
            FROM INFORMATION_SCHEMA.COLUMNS c 
            WHERE c.TABLE_SCHEMA = t.TABLE_SCHEMA 
              AND c.TABLE_NAME = t.TABLE_NAME
        ) AS ColumnCount,
        (
            SELECT COUNT(*) 
            FROM INFORMATION_SCHEMA.TABLE_CONSTRAINTS tc 
            WHERE tc.TABLE_SCHEMA = t.TABLE_SCHEMA 
              AND tc.TABLE_NAME = t.TABLE_NAME 
              AND tc.CONSTRAINT_TYPE = 'PRIMARY KEY'
        ) AS HasPrimaryKey
    FROM INFORMATION_SCHEMA.TABLES t
    WHERE t.TABLE_SCHEMA = @SchemaName
      AND t.TABLE_NAME LIKE @TablePattern
    ORDER BY t.TABLE_NAME;
    
    -- 각 테이블의 행 수 (동적 SQL 필요)
    DECLARE @TableName NVARCHAR(256);
    DECLARE @SQL NVARCHAR(MAX);
    DECLARE @Results TABLE (TableName NVARCHAR(256), RowCount BIGINT);
    
    DECLARE table_cursor CURSOR FOR
    SELECT TABLE_SCHEMA + '.' + TABLE_NAME
    FROM INFORMATION_SCHEMA.TABLES
    WHERE TABLE_SCHEMA = @SchemaName
      AND TABLE_NAME LIKE @TablePattern
      AND TABLE_TYPE = 'BASE TABLE';
    
    OPEN table_cursor;
    FETCH NEXT FROM table_cursor INTO @TableName;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        SET @SQL = N'SELECT @Count = COUNT(*) FROM ' + QUOTENAME(@TableName);
        
        DECLARE @Count BIGINT;
        EXEC sp_executesql @SQL, N'@Count BIGINT OUTPUT', @Count OUTPUT;
        
        INSERT INTO @Results VALUES (@TableName, @Count);
        
        FETCH NEXT FROM table_cursor INTO @TableName;
    END
    
    CLOSE table_cursor;
    DEALLOCATE table_cursor;
    
    SELECT * FROM @Results ORDER BY RowCount DESC;
END;
```

## 6. 에러 처리와 디버깅

### 6.1 구조화된 에러 처리

```sql
CREATE PROCEDURE sp_RobustProcedure
    @Parameter1 INT,
    @Parameter2 NVARCHAR(100)
AS
BEGIN
    SET NOCOUNT ON;
    
    -- 로컬 변수
    DECLARE @ErrorMessage NVARCHAR(4000);
    DECLARE @ErrorSeverity INT;
    DECLARE @ErrorState INT;
    DECLARE @ErrorProcedure NVARCHAR(200);
    DECLARE @ErrorLine INT;
    DECLARE @ErrorNumber INT;
    
    BEGIN TRY
        -- 입력 검증
        IF @Parameter1 <= 0
        BEGIN
            RAISERROR('Parameter1 must be positive', 16, 1);
        END
        
        IF LEN(@Parameter2) < 3
        BEGIN
            RAISERROR('Parameter2 must be at least 3 characters', 16, 2);
        END
        
        BEGIN TRANSACTION;
        
        -- 비즈니스 로직
        -- ...
        
        COMMIT TRANSACTION;
        
    END TRY
    BEGIN CATCH
        -- 트랜잭션 롤백
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        
        -- 에러 정보 수집
        SELECT 
            @ErrorNumber = ERROR_NUMBER(),
            @ErrorMessage = ERROR_MESSAGE(),
            @ErrorSeverity = ERROR_SEVERITY(),
            @ErrorState = ERROR_STATE(),
            @ErrorProcedure = ERROR_PROCEDURE(),
            @ErrorLine = ERROR_LINE();
        
        -- 에러 로깅
        INSERT INTO ErrorLog 
            (ErrorNumber, ErrorMessage, ErrorSeverity, ErrorState, 
             ErrorProcedure, ErrorLine, ErrorDate, UserName)
        VALUES 
            (@ErrorNumber, @ErrorMessage, @ErrorSeverity, @ErrorState,
             @ErrorProcedure, @ErrorLine, GETDATE(), SYSTEM_USER);
        
        -- 사용자 정의 에러 메시지 생성
        SET @ErrorMessage = 'Error in ' + ISNULL(@ErrorProcedure, 'Unknown') + 
                           ' at line ' + CAST(@ErrorLine AS VARCHAR(10)) + 
                           ': ' + @ErrorMessage;
        
        -- 에러 재발생
        RAISERROR(@ErrorMessage, @ErrorSeverity, @ErrorState);
    END CATCH
END;

-- 디버깅을 위한 프로시저
CREATE PROCEDURE sp_DebugProcedure
    @DebugMode BIT = 0,
    @Parameter INT
AS
BEGIN
    SET NOCOUNT ON;
    
    IF @DebugMode = 1
        PRINT 'Starting procedure with Parameter: ' + CAST(@Parameter AS VARCHAR(10));
    
    DECLARE @StepCount INT = 0;
    DECLARE @IntermediateResult INT;
    
    -- Step 1
    SET @StepCount = @StepCount + 1;
    SET @IntermediateResult = @Parameter * 2;
    
    IF @DebugMode = 1
    BEGIN
        PRINT 'Step ' + CAST(@StepCount AS VARCHAR(10)) + ': Intermediate result = ' + 
              CAST(@IntermediateResult AS VARCHAR(10));
    END
    
    -- Step 2
    SET @StepCount = @StepCount + 1;
    SET @IntermediateResult = @IntermediateResult + 10;
    
    IF @DebugMode = 1
    BEGIN
        PRINT 'Step ' + CAST(@StepCount AS VARCHAR(10)) + ': Intermediate result = ' + 
              CAST(@IntermediateResult AS VARCHAR(10));
    END
    
    -- 결과 반환
    SELECT @IntermediateResult AS FinalResult;
END;
```

## 마무리

저장 프로시저, 함수, 트리거는 데이터베이스에서 비즈니스 로직을 구현하는 강력한 도구입니다. 적절한 에러 처리, 트랜잭션 관리, 그리고 보안을 고려한 동적 SQL 사용이 중요합니다. 이러한 데이터베이스 프로그래밍 객체들을 효과적으로 활용하면 애플리케이션의 성능과 유지보수성을 크게 향상시킬 수 있습니다.