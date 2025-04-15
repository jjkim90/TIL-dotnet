# 쿼리 최적화와 실행 계획

## 개요

효율적인 쿼리 작성과 실행 계획 분석은 데이터베이스 성능 최적화의 핵심입니다. 이 장에서는 쿼리 옵티마이저의 동작 원리, 실행 계획 분석 방법, 쿼리 최적화 기법, 그리고 성능 문제 진단 및 해결 방법을 학습합니다.

## 1. 쿼리 옵티마이저 이해

### 1.1 옵티마이저 기본 개념

```sql
-- 통계 정보 업데이트
UPDATE STATISTICS Orders WITH FULLSCAN;
UPDATE STATISTICS Customers WITH FULLSCAN;

-- 통계 정보 확인
DBCC SHOW_STATISTICS('Orders', 'IX_Orders_CustomerId');

-- 테이블 통계 정보 조회
SELECT 
    s.name AS StatisticsName,
    sp.last_updated,
    sp.rows,
    sp.rows_sampled,
    sp.modification_counter,
    STUFF((
        SELECT ', ' + c.name
        FROM sys.stats_columns sc
        INNER JOIN sys.columns c ON sc.object_id = c.object_id 
            AND sc.column_id = c.column_id
        WHERE sc.object_id = s.object_id 
            AND sc.stats_id = s.stats_id
        ORDER BY sc.stats_column_id
        FOR XML PATH('')
    ), 1, 2, '') AS Columns
FROM sys.stats s
CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) sp
WHERE s.object_id = OBJECT_ID('Orders');

-- 카디널리티 추정
SET SHOWPLAN_XML ON;
GO
SELECT 
    o.OrderId,
    o.OrderDate,
    c.CustomerName
FROM Orders o
INNER JOIN Customers c ON o.CustomerId = c.CustomerId
WHERE o.OrderDate >= '2024-01-01'
AND c.City = 'Seoul';
GO
SET SHOWPLAN_XML OFF;
```

### 1.2 옵티마이저 힌트

```sql
-- JOIN 힌트
-- LOOP JOIN (Nested Loop)
SELECT o.*, c.*
FROM Orders o
INNER LOOP JOIN Customers c ON o.CustomerId = c.CustomerId
WHERE o.OrderDate >= '2024-01-01';

-- MERGE JOIN
SELECT o.*, c.*
FROM Orders o
INNER MERGE JOIN Customers c ON o.CustomerId = c.CustomerId
WHERE o.OrderDate >= '2024-01-01';

-- HASH JOIN
SELECT o.*, c.*
FROM Orders o
INNER HASH JOIN Customers c ON o.CustomerId = c.CustomerId
WHERE o.OrderDate >= '2024-01-01';

-- 쿼리 힌트
SELECT * FROM Orders
WHERE CustomerId = 1234
OPTION (
    RECOMPILE,                    -- 재컴파일
    OPTIMIZE FOR (@CustomerId = 5000),  -- 특정 값으로 최적화
    MAXDOP 4,                     -- 병렬 처리 수준
    USE PLAN N'<ShowPlanXML>...' -- 특정 실행 계획 강제
);

-- 테이블 힌트
SELECT * FROM Orders WITH (NOLOCK)      -- 더티 리드 허용
WHERE OrderDate >= '2024-01-01';

SELECT * FROM Orders WITH (READPAST)    -- 잠긴 행 건너뛰기
WHERE Status = 'Pending';

SELECT * FROM Orders WITH (TABLOCK)     -- 테이블 수준 잠금
WHERE OrderDate < '2023-01-01';
```

## 2. 실행 계획 분석

### 2.1 실행 계획 읽기

```sql
-- 예상 실행 계획
SET SHOWPLAN_ALL ON;
GO
SELECT 
    o.OrderId,
    o.OrderDate,
    o.TotalAmount,
    c.CustomerName,
    c.Email
FROM Orders o
INNER JOIN Customers c ON o.CustomerId = c.CustomerId
WHERE o.OrderDate >= '2024-01-01'
AND o.TotalAmount > 1000
ORDER BY o.OrderDate DESC;
GO
SET SHOWPLAN_ALL OFF;

-- 실제 실행 계획
SET STATISTICS XML ON;
GO
SELECT 
    o.OrderId,
    o.OrderDate,
    o.TotalAmount,
    c.CustomerName,
    c.Email
FROM Orders o
INNER JOIN Customers c ON o.CustomerId = c.CustomerId
WHERE o.OrderDate >= '2024-01-01'
AND o.TotalAmount > 1000
ORDER BY o.OrderDate DESC;
GO
SET STATISTICS XML OFF;

-- 실행 계획 캐시 조회
SELECT 
    cp.refcounts,
    cp.usecounts,
    cp.size_in_bytes,
    cp.cacheobjtype,
    cp.objtype,
    st.text,
    qp.query_plan
FROM sys.dm_exec_cached_plans cp
CROSS APPLY sys.dm_exec_sql_text(cp.plan_handle) st
CROSS APPLY sys.dm_exec_query_plan(cp.plan_handle) qp
WHERE st.text LIKE '%Orders%'
AND st.text NOT LIKE '%sys.dm_exec_cached_plans%'
ORDER BY cp.usecounts DESC;
```

### 2.2 주요 연산자 분석

```sql
-- Table Scan vs Index Seek 비교
-- 테이블 스캔 강제
SELECT * FROM Orders WITH (INDEX(0))
WHERE OrderId = 12345;

-- 인덱스 탐색
SELECT * FROM Orders
WHERE OrderId = 12345;

-- Key Lookup 발생 예제
CREATE INDEX IX_Orders_Date ON Orders(OrderDate);

-- Key Lookup 발생
SELECT OrderId, OrderDate, CustomerId, TotalAmount
FROM Orders
WHERE OrderDate = '2024-01-15';

-- Key Lookup 제거 (커버링 인덱스)
CREATE INDEX IX_Orders_DateCovering 
ON Orders(OrderDate)
INCLUDE (CustomerId, TotalAmount);

-- Sort 연산자
-- 명시적 정렬
SELECT * FROM Orders
ORDER BY TotalAmount DESC;

-- 인덱스를 활용한 정렬 회피
CREATE INDEX IX_Orders_TotalAmount ON Orders(TotalAmount DESC);

SELECT * FROM Orders WITH (INDEX(IX_Orders_TotalAmount))
ORDER BY TotalAmount DESC;

-- Parallelism 연산자
SELECT 
    ProductCategory,
    COUNT(*) AS OrderCount,
    SUM(TotalAmount) AS TotalRevenue
FROM Orders
GROUP BY ProductCategory
OPTION (MAXDOP 4);  -- 병렬 처리
```

### 2.3 실행 계획 경고와 문제점

```sql
-- 암시적 변환 경고
DECLARE @CustomerId NVARCHAR(10) = '1234';

-- 경고 발생 (CustomerId는 INT)
SELECT * FROM Orders
WHERE CustomerId = @CustomerId;

-- 올바른 방법
DECLARE @CustomerIdInt INT = 1234;
SELECT * FROM Orders
WHERE CustomerId = @CustomerIdInt;

-- 누락된 인덱스 경고
SELECT 
    o.OrderId,
    o.OrderDate,
    c.CustomerName
FROM Orders o
INNER JOIN Customers c ON o.CustomerId = c.CustomerId
WHERE o.Status = 'Pending'
AND o.ShippingCity = 'Seoul'
AND o.OrderDate >= '2024-01-01';

-- 카디널리티 추정 오류
-- 통계 정보가 오래된 경우
DBCC SHOW_STATISTICS('Orders', 'IX_Orders_CustomerId');

-- 통계 업데이트
UPDATE STATISTICS Orders IX_Orders_CustomerId WITH FULLSCAN;

-- 파라미터 스니핑 문제
CREATE PROCEDURE sp_GetOrdersByCustomer
    @CustomerId INT
AS
BEGIN
    SELECT * FROM Orders
    WHERE CustomerId = @CustomerId;
END;

-- 파라미터 스니핑 해결
CREATE PROCEDURE sp_GetOrdersByCustomer_Fixed
    @CustomerId INT
AS
BEGIN
    DECLARE @LocalCustomerId INT = @CustomerId;
    
    SELECT * FROM Orders
    WHERE CustomerId = @LocalCustomerId
    OPTION (RECOMPILE);
END;
```

## 3. 쿼리 최적화 기법

### 3.1 JOIN 최적화

```sql
-- JOIN 순서 최적화
-- 비효율적인 JOIN 순서
SELECT *
FROM OrderDetails od
INNER JOIN Orders o ON od.OrderId = o.OrderId
INNER JOIN Customers c ON o.CustomerId = c.CustomerId
WHERE c.Country = 'Korea';

-- 최적화된 JOIN 순서 (필터링 먼저)
SELECT *
FROM Customers c
INNER JOIN Orders o ON c.CustomerId = o.CustomerId
INNER JOIN OrderDetails od ON o.OrderId = od.OrderId
WHERE c.Country = 'Korea';

-- EXISTS vs JOIN
-- JOIN 사용
SELECT DISTINCT c.*
FROM Customers c
INNER JOIN Orders o ON c.CustomerId = o.CustomerId
WHERE o.OrderDate >= '2024-01-01';

-- EXISTS 사용 (더 효율적)
SELECT c.*
FROM Customers c
WHERE EXISTS (
    SELECT 1 FROM Orders o
    WHERE o.CustomerId = c.CustomerId
    AND o.OrderDate >= '2024-01-01'
);

-- NOT EXISTS vs LEFT JOIN
-- LEFT JOIN 사용
SELECT c.*
FROM Customers c
LEFT JOIN Orders o ON c.CustomerId = o.CustomerId
    AND o.OrderDate >= '2024-01-01'
WHERE o.OrderId IS NULL;

-- NOT EXISTS 사용 (더 효율적)
SELECT c.*
FROM Customers c
WHERE NOT EXISTS (
    SELECT 1 FROM Orders o
    WHERE o.CustomerId = c.CustomerId
    AND o.OrderDate >= '2024-01-01'
);
```

### 3.2 서브쿼리 최적화

```sql
-- 상관 서브쿼리를 JOIN으로 변환
-- 비효율적인 상관 서브쿼리
SELECT 
    c.CustomerId,
    c.CustomerName,
    (SELECT COUNT(*) FROM Orders o WHERE o.CustomerId = c.CustomerId) AS OrderCount,
    (SELECT SUM(TotalAmount) FROM Orders o WHERE o.CustomerId = c.CustomerId) AS TotalSpent
FROM Customers c;

-- JOIN으로 최적화
SELECT 
    c.CustomerId,
    c.CustomerName,
    COUNT(o.OrderId) AS OrderCount,
    SUM(o.TotalAmount) AS TotalSpent
FROM Customers c
LEFT JOIN Orders o ON c.CustomerId = o.CustomerId
GROUP BY c.CustomerId, c.CustomerName;

-- IN을 EXISTS로 변환
-- IN 사용
SELECT * FROM Orders
WHERE CustomerId IN (
    SELECT CustomerId FROM Customers
    WHERE Country = 'Korea'
);

-- EXISTS 사용 (대량 데이터에서 더 효율적)
SELECT o.* FROM Orders o
WHERE EXISTS (
    SELECT 1 FROM Customers c
    WHERE c.CustomerId = o.CustomerId
    AND c.Country = 'Korea'
);

-- 스칼라 서브쿼리 최적화
-- 비효율적 (각 행마다 서브쿼리 실행)
SELECT 
    o.OrderId,
    o.OrderDate,
    o.TotalAmount,
    (SELECT TOP 1 p.PaymentDate 
     FROM Payments p 
     WHERE p.OrderId = o.OrderId 
     ORDER BY p.PaymentDate DESC) AS LastPaymentDate
FROM Orders o;

-- OUTER APPLY로 최적화
SELECT 
    o.OrderId,
    o.OrderDate,
    o.TotalAmount,
    p.PaymentDate AS LastPaymentDate
FROM Orders o
OUTER APPLY (
    SELECT TOP 1 PaymentDate
    FROM Payments
    WHERE OrderId = o.OrderId
    ORDER BY PaymentDate DESC
) p;
```

### 3.3 집계 쿼리 최적화

```sql
-- 사전 집계를 통한 최적화
-- 비효율적 (전체 조인 후 집계)
SELECT 
    c.Country,
    COUNT(od.OrderDetailId) AS ItemCount,
    SUM(od.Quantity * od.UnitPrice) AS TotalRevenue
FROM Customers c
INNER JOIN Orders o ON c.CustomerId = o.CustomerId
INNER JOIN OrderDetails od ON o.OrderId = od.OrderId
WHERE o.OrderDate >= '2024-01-01'
GROUP BY c.Country;

-- 사전 집계로 최적화
WITH OrderAggregates AS (
    SELECT 
        o.CustomerId,
        COUNT(od.OrderDetailId) AS ItemCount,
        SUM(od.Quantity * od.UnitPrice) AS Revenue
    FROM Orders o
    INNER JOIN OrderDetails od ON o.OrderId = od.OrderId
    WHERE o.OrderDate >= '2024-01-01'
    GROUP BY o.CustomerId
)
SELECT 
    c.Country,
    SUM(oa.ItemCount) AS ItemCount,
    SUM(oa.Revenue) AS TotalRevenue
FROM Customers c
INNER JOIN OrderAggregates oa ON c.CustomerId = oa.CustomerId
GROUP BY c.Country;

-- 윈도우 함수 활용
-- 비효율적 (자기 조인)
SELECT 
    o1.OrderId,
    o1.CustomerId,
    o1.TotalAmount,
    (SELECT COUNT(*) FROM Orders o2 
     WHERE o2.CustomerId = o1.CustomerId 
     AND o2.TotalAmount <= o1.TotalAmount) AS RankInCustomer
FROM Orders o1;

-- 윈도우 함수로 최적화
SELECT 
    OrderId,
    CustomerId,
    TotalAmount,
    RANK() OVER (PARTITION BY CustomerId ORDER BY TotalAmount) AS RankInCustomer
FROM Orders;
```

## 4. 성능 모니터링과 진단

### 4.1 쿼리 성능 모니터링

```sql
-- 실행 시간이 긴 쿼리 찾기
SELECT TOP 20
    qs.total_elapsed_time / qs.execution_count AS avg_elapsed_time,
    qs.total_logical_reads / qs.execution_count AS avg_logical_reads,
    qs.total_worker_time / qs.execution_count AS avg_cpu_time,
    qs.execution_count,
    qs.creation_time,
    qs.last_execution_time,
    SUBSTRING(st.text, (qs.statement_start_offset/2) + 1,
        ((CASE qs.statement_end_offset
            WHEN -1 THEN DATALENGTH(st.text)
            ELSE qs.statement_end_offset
        END - qs.statement_start_offset)/2) + 1) AS query_text,
    qp.query_plan
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
WHERE qs.total_elapsed_time / qs.execution_count > 1000000  -- 1초 이상
ORDER BY avg_elapsed_time DESC;

-- CPU 사용량이 높은 쿼리
SELECT TOP 20
    qs.total_worker_time / qs.execution_count AS avg_cpu_time,
    qs.total_worker_time AS total_cpu_time,
    qs.execution_count,
    SUBSTRING(st.text, (qs.statement_start_offset/2) + 1,
        ((CASE qs.statement_end_offset
            WHEN -1 THEN DATALENGTH(st.text)
            ELSE qs.statement_end_offset
        END - qs.statement_start_offset)/2) + 1) AS query_text,
    DB_NAME(st.dbid) AS database_name
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY total_cpu_time DESC;

-- I/O가 많은 쿼리
SELECT TOP 20
    qs.total_logical_reads / qs.execution_count AS avg_logical_reads,
    qs.total_logical_writes / qs.execution_count AS avg_logical_writes,
    qs.total_physical_reads / qs.execution_count AS avg_physical_reads,
    qs.execution_count,
    SUBSTRING(st.text, (qs.statement_start_offset/2) + 1,
        ((CASE qs.statement_end_offset
            WHEN -1 THEN DATALENGTH(st.text)
            ELSE qs.statement_end_offset
        END - qs.statement_start_offset)/2) + 1) AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY avg_logical_reads DESC;
```

### 4.2 실시간 성능 모니터링

```sql
-- 현재 실행 중인 쿼리
SELECT 
    r.session_id,
    r.start_time,
    r.status,
    r.command,
    r.database_id,
    r.user_id,
    r.wait_type,
    r.wait_time,
    r.blocking_session_id,
    r.cpu_time,
    r.logical_reads,
    r.reads,
    r.writes,
    st.text AS query_text,
    qp.query_plan
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) st
CROSS APPLY sys.dm_exec_query_plan(r.plan_handle) qp
WHERE r.session_id > 50  -- 시스템 세션 제외
AND r.session_id <> @@SPID  -- 현재 세션 제외
ORDER BY r.cpu_time DESC;

-- 블로킹 체인 분석
WITH BlockingChain AS (
    SELECT 
        s1.session_id AS blocked_session,
        s1.blocking_session_id AS blocking_session,
        s1.wait_type,
        s1.wait_time,
        s1.wait_resource,
        st.text AS blocked_query,
        0 AS level
    FROM sys.dm_exec_requests s1
    CROSS APPLY sys.dm_exec_sql_text(s1.sql_handle) st
    WHERE s1.blocking_session_id > 0
    
    UNION ALL
    
    SELECT 
        bc.blocked_session,
        s2.blocking_session_id,
        s2.wait_type,
        s2.wait_time,
        s2.wait_resource,
        st.text,
        bc.level + 1
    FROM BlockingChain bc
    INNER JOIN sys.dm_exec_requests s2 ON bc.blocking_session = s2.session_id
    CROSS APPLY sys.dm_exec_sql_text(s2.sql_handle) st
    WHERE s2.blocking_session_id > 0
)
SELECT * FROM BlockingChain
ORDER BY level, blocked_session;
```

### 4.3 쿼리 저장소 (Query Store)

```sql
-- Query Store 활성화
ALTER DATABASE YourDatabase
SET QUERY_STORE = ON (
    OPERATION_MODE = READ_WRITE,
    CLEANUP_POLICY = (STALE_QUERY_THRESHOLD_DAYS = 30),
    DATA_FLUSH_INTERVAL_SECONDS = 900,
    MAX_STORAGE_SIZE_MB = 1000,
    INTERVAL_LENGTH_MINUTES = 60,
    SIZE_BASED_CLEANUP_MODE = AUTO,
    QUERY_CAPTURE_MODE = AUTO,
    MAX_PLANS_PER_QUERY = 200
);

-- 퇴행된 쿼리 찾기
SELECT 
    q.query_id,
    qt.query_sql_text,
    rs1.avg_duration AS recent_avg_duration,
    rs2.avg_duration AS history_avg_duration,
    rs1.avg_duration / rs2.avg_duration AS regression_ratio
FROM sys.query_store_query q
INNER JOIN sys.query_store_query_text qt ON q.query_text_id = qt.query_text_id
INNER JOIN sys.query_store_plan p1 ON q.query_id = p1.query_id
INNER JOIN sys.query_store_runtime_stats rs1 ON p1.plan_id = rs1.plan_id
INNER JOIN sys.query_store_runtime_stats_interval i1 ON rs1.runtime_stats_interval_id = i1.runtime_stats_interval_id
INNER JOIN sys.query_store_plan p2 ON q.query_id = p2.query_id
INNER JOIN sys.query_store_runtime_stats rs2 ON p2.plan_id = rs2.plan_id
INNER JOIN sys.query_store_runtime_stats_interval i2 ON rs2.runtime_stats_interval_id = i2.runtime_stats_interval_id
WHERE i1.start_time > DATEADD(hour, -1, GETUTCDATE())  -- 최근 1시간
AND i2.start_time < DATEADD(hour, -24, GETUTCDATE())   -- 24시간 전
AND i2.start_time > DATEADD(hour, -48, GETUTCDATE())   -- 48시간 전
AND rs1.avg_duration > rs2.avg_duration * 2            -- 2배 이상 느려진 경우
ORDER BY regression_ratio DESC;

-- 특정 쿼리의 실행 계획 강제
EXEC sp_query_store_force_plan @query_id = 10, @plan_id = 5;

-- 강제된 계획 해제
EXEC sp_query_store_unforce_plan @query_id = 10, @plan_id = 5;
```

## 5. 고급 최적화 기법

### 5.1 파티션 정렬 제거

```sql
-- 파티션 함수와 스키마 생성
CREATE PARTITION FUNCTION PF_OrderDate (DATETIME)
AS RANGE RIGHT FOR VALUES ('2023-01-01', '2023-07-01', '2024-01-01', '2024-07-01');

CREATE PARTITION SCHEME PS_OrderDate
AS PARTITION PF_OrderDate
ALL TO ([PRIMARY]);

-- 파티션된 테이블
CREATE TABLE Orders_Partitioned (
    OrderId INT NOT NULL,
    OrderDate DATETIME NOT NULL,
    CustomerId INT NOT NULL,
    TotalAmount DECIMAL(10, 2)
) ON PS_OrderDate(OrderDate);

-- 파티션 정렬 제거
CREATE CLUSTERED INDEX CIX_Orders_Part 
ON Orders_Partitioned(OrderDate, OrderId)
ON PS_OrderDate(OrderDate);

-- 파티션 제거 확인
SELECT * FROM Orders_Partitioned
WHERE OrderDate >= '2024-01-01' AND OrderDate < '2024-02-01'
ORDER BY OrderDate, OrderId;  -- 정렬 연산 없음
```

### 5.2 배치 모드 처리

```sql
-- 배치 모드 활성화를 위한 columnstore 인덱스
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_Orders_Batch
ON Orders(OrderId, CustomerId, OrderDate, TotalAmount)
WHERE OrderDate >= '2023-01-01';

-- 배치 모드 실행 확인
SELECT 
    YEAR(OrderDate) AS OrderYear,
    MONTH(OrderDate) AS OrderMonth,
    COUNT(*) AS OrderCount,
    SUM(TotalAmount) AS TotalRevenue,
    AVG(TotalAmount) AS AvgOrderValue
FROM Orders
WHERE OrderDate >= '2023-01-01'
GROUP BY YEAR(OrderDate), MONTH(OrderDate)
OPTION (USE HINT('FORCE_DEFAULT_CARDINALITY_ESTIMATION'));

-- 실행 계획에서 Batch Mode 확인
SET STATISTICS XML ON;
SELECT 
    CustomerId,
    COUNT(*) AS OrderCount,
    SUM(TotalAmount) AS TotalSpent
FROM Orders
WHERE OrderDate >= '2023-01-01'
GROUP BY CustomerId
HAVING COUNT(*) > 10;
SET STATISTICS XML OFF;
```

### 5.3 인라인 테이블 반환 함수

```sql
-- 다중 문 테이블 반환 함수 (느림)
CREATE FUNCTION fn_GetCustomerOrders_Slow(@CustomerId INT)
RETURNS @Orders TABLE (
    OrderId INT,
    OrderDate DATETIME,
    TotalAmount DECIMAL(10, 2)
)
AS
BEGIN
    INSERT INTO @Orders
    SELECT OrderId, OrderDate, TotalAmount
    FROM Orders
    WHERE CustomerId = @CustomerId;
    
    RETURN;
END;

-- 인라인 테이블 반환 함수 (빠름)
CREATE FUNCTION fn_GetCustomerOrders_Fast(@CustomerId INT)
RETURNS TABLE
AS
RETURN (
    SELECT OrderId, OrderDate, TotalAmount
    FROM Orders
    WHERE CustomerId = @CustomerId
);

-- 성능 비교
-- 느린 버전
SELECT c.CustomerName, o.*
FROM Customers c
CROSS APPLY fn_GetCustomerOrders_Slow(c.CustomerId) o;

-- 빠른 버전 (인라인 확장됨)
SELECT c.CustomerName, o.*
FROM Customers c
CROSS APPLY fn_GetCustomerOrders_Fast(c.CustomerId) o;
```

## 6. 실무 최적화 사례

### 6.1 페이징 쿼리 최적화

```sql
-- 비효율적인 페이징 (OFFSET-FETCH)
DECLARE @PageSize INT = 20;
DECLARE @PageNumber INT = 1000;

SELECT *
FROM Orders
ORDER BY OrderDate DESC
OFFSET (@PageNumber - 1) * @PageSize ROWS
FETCH NEXT @PageSize ROWS ONLY;

-- 키셋 페이징 (효율적)
DECLARE @LastOrderDate DATETIME = '2024-01-15';
DECLARE @LastOrderId INT = 123456;

SELECT TOP (@PageSize) *
FROM Orders
WHERE OrderDate < @LastOrderDate
   OR (OrderDate = @LastOrderDate AND OrderId < @LastOrderId)
ORDER BY OrderDate DESC, OrderId DESC;

-- 페이징을 위한 인덱스
CREATE INDEX IX_Orders_Paging
ON Orders(OrderDate DESC, OrderId DESC)
INCLUDE (CustomerId, TotalAmount);

-- CTE를 사용한 페이징 최적화
WITH OrderedOrders AS (
    SELECT 
        OrderId,
        OrderDate,
        CustomerId,
        TotalAmount,
        ROW_NUMBER() OVER (ORDER BY OrderDate DESC, OrderId DESC) AS RowNum
    FROM Orders
    WHERE OrderDate >= '2024-01-01'
)
SELECT *
FROM OrderedOrders
WHERE RowNum BETWEEN (@PageNumber - 1) * @PageSize + 1 
                 AND @PageNumber * @PageSize;
```

### 6.2 복잡한 보고서 쿼리 최적화

```sql
-- 원본 느린 쿼리
SELECT 
    c.CustomerName,
    c.Country,
    COUNT(DISTINCT o.OrderId) AS TotalOrders,
    COUNT(DISTINCT CASE WHEN o.Status = 'Delivered' THEN o.OrderId END) AS DeliveredOrders,
    SUM(od.Quantity * od.UnitPrice) AS TotalRevenue,
    AVG(od.Quantity * od.UnitPrice) AS AvgOrderValue,
    MIN(o.OrderDate) AS FirstOrderDate,
    MAX(o.OrderDate) AS LastOrderDate
FROM Customers c
LEFT JOIN Orders o ON c.CustomerId = o.CustomerId
LEFT JOIN OrderDetails od ON o.OrderId = od.OrderId
WHERE o.OrderDate >= '2023-01-01'
GROUP BY c.CustomerId, c.CustomerName, c.Country
HAVING SUM(od.Quantity * od.UnitPrice) > 10000
ORDER BY TotalRevenue DESC;

-- 최적화된 쿼리
WITH OrderMetrics AS (
    SELECT 
        o.CustomerId,
        o.OrderId,
        o.OrderDate,
        o.Status,
        SUM(od.Quantity * od.UnitPrice) AS OrderTotal
    FROM Orders o
    INNER JOIN OrderDetails od ON o.OrderId = od.OrderId
    WHERE o.OrderDate >= '2023-01-01'
    GROUP BY o.CustomerId, o.OrderId, o.OrderDate, o.Status
),
CustomerMetrics AS (
    SELECT 
        CustomerId,
        COUNT(DISTINCT OrderId) AS TotalOrders,
        COUNT(DISTINCT CASE WHEN Status = 'Delivered' THEN OrderId END) AS DeliveredOrders,
        SUM(OrderTotal) AS TotalRevenue,
        AVG(OrderTotal) AS AvgOrderValue,
        MIN(OrderDate) AS FirstOrderDate,
        MAX(OrderDate) AS LastOrderDate
    FROM OrderMetrics
    GROUP BY CustomerId
    HAVING SUM(OrderTotal) > 10000
)
SELECT 
    c.CustomerName,
    c.Country,
    cm.TotalOrders,
    cm.DeliveredOrders,
    cm.TotalRevenue,
    cm.AvgOrderValue,
    cm.FirstOrderDate,
    cm.LastOrderDate
FROM CustomerMetrics cm
INNER JOIN Customers c ON cm.CustomerId = c.CustomerId
ORDER BY cm.TotalRevenue DESC;

-- 인덱스 생성
CREATE INDEX IX_Orders_Reporting
ON Orders(OrderDate, CustomerId, OrderId, Status);

CREATE INDEX IX_OrderDetails_Order
ON OrderDetails(OrderId)
INCLUDE (Quantity, UnitPrice);
```

### 6.3 실시간 대시보드 최적화

```sql
-- 실시간 대시보드를 위한 인덱스 뷰
CREATE VIEW vw_RealtimeDashboard
WITH SCHEMABINDING
AS
SELECT 
    CAST(o.OrderDate AS DATE) AS OrderDate,
    o.Status,
    o.ProductCategory,
    COUNT_BIG(*) AS OrderCount,
    SUM(o.TotalAmount) AS TotalRevenue,
    SUM(CASE WHEN o.Status = 'Delivered' THEN o.TotalAmount ELSE 0 END) AS DeliveredRevenue
FROM dbo.Orders o
WHERE o.OrderDate >= DATEADD(DAY, -30, GETDATE())
GROUP BY CAST(o.OrderDate AS DATE), o.Status, o.ProductCategory;

-- 인덱스 뷰에 클러스터드 인덱스
CREATE UNIQUE CLUSTERED INDEX CIX_vw_RealtimeDashboard
ON vw_RealtimeDashboard(OrderDate, Status, ProductCategory);

-- 대시보드 쿼리 (인덱스 뷰 활용)
SELECT 
    OrderDate,
    SUM(OrderCount) AS TotalOrders,
    SUM(TotalRevenue) AS TotalRevenue,
    SUM(DeliveredRevenue) AS DeliveredRevenue,
    CAST(SUM(DeliveredRevenue) AS FLOAT) / NULLIF(SUM(TotalRevenue), 0) * 100 AS DeliveryRate
FROM vw_RealtimeDashboard WITH (NOEXPAND)
WHERE OrderDate >= DATEADD(DAY, -7, GETDATE())
GROUP BY OrderDate
ORDER BY OrderDate DESC;
```

## 마무리

쿼리 최적화는 지속적인 모니터링과 개선이 필요한 과정입니다. 실행 계획을 정확히 읽고 분석하는 능력, 다양한 최적화 기법의 적절한 활용, 그리고 성능 메트릭의 지속적인 추적이 중요합니다. Query Store와 같은 도구를 활용하여 성능 퇴행을 방지하고, 항상 변화하는 데이터 패턴에 맞춰 최적화 전략을 조정해야 합니다. 다음 장에서는 트랜잭션과 동시성 제어를 통한 데이터 무결성 보장 방법을 학습하겠습니다.