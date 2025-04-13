# 인덱스 설계와 최적화

## 개요

인덱스는 데이터베이스 성능 최적화의 핵심 요소입니다. 올바른 인덱스 설계는 쿼리 성능을 수십, 수백 배 향상시킬 수 있지만, 잘못된 인덱스는 오히려 성능을 저하시킬 수 있습니다. 이 장에서는 다양한 인덱스 유형, 설계 원칙, 최적화 전략, 그리고 인덱스 모니터링 방법을 학습합니다.

## 1. 인덱스 기본 개념

### 1.1 인덱스 구조와 동작 원리

```sql
-- 테스트 테이블 생성
CREATE TABLE Orders (
    OrderId INT PRIMARY KEY IDENTITY(1,1),
    CustomerId INT NOT NULL,
    OrderDate DATETIME NOT NULL,
    TotalAmount DECIMAL(10, 2) NOT NULL,
    Status VARCHAR(20) NOT NULL,
    ShippingCity NVARCHAR(100),
    ProductCategory VARCHAR(50),
    PaymentMethod VARCHAR(20)
);

-- 샘플 데이터 삽입 (100만 건)
WITH Numbers AS (
    SELECT TOP 1000000 ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS n
    FROM sys.all_columns a
    CROSS JOIN sys.all_columns b
)
INSERT INTO Orders (CustomerId, OrderDate, TotalAmount, Status, ShippingCity, ProductCategory, PaymentMethod)
SELECT 
    ABS(CHECKSUM(NEWID())) % 10000 + 1 AS CustomerId,
    DATEADD(DAY, -n % 730, GETDATE()) AS OrderDate,
    ROUND(RAND(CHECKSUM(NEWID())) * 9900 + 100, 2) AS TotalAmount,
    CASE n % 5 
        WHEN 0 THEN 'Pending'
        WHEN 1 THEN 'Processing'
        WHEN 2 THEN 'Shipped'
        WHEN 3 THEN 'Delivered'
        ELSE 'Cancelled'
    END AS Status,
    CASE n % 10
        WHEN 0 THEN 'Seoul'
        WHEN 1 THEN 'Busan'
        WHEN 2 THEN 'Incheon'
        WHEN 3 THEN 'Daegu'
        WHEN 4 THEN 'Daejeon'
        WHEN 5 THEN 'Gwangju'
        WHEN 6 THEN 'Ulsan'
        WHEN 7 THEN 'Sejong'
        WHEN 8 THEN 'Suwon'
        ELSE 'Jeju'
    END AS ShippingCity,
    CASE n % 5
        WHEN 0 THEN 'Electronics'
        WHEN 1 THEN 'Clothing'
        WHEN 2 THEN 'Books'
        WHEN 3 THEN 'Food'
        ELSE 'Others'
    END AS ProductCategory,
    CASE n % 4
        WHEN 0 THEN 'CreditCard'
        WHEN 1 THEN 'DebitCard'
        WHEN 2 THEN 'Cash'
        ELSE 'Transfer'
    END AS PaymentMethod
FROM Numbers;

-- 인덱스 없이 쿼리 실행
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

SELECT * FROM Orders 
WHERE CustomerId = 5000 
AND OrderDate >= '2023-01-01';

-- 실행 계획 확인
/*
Table Scan - 전체 테이블을 읽음
Logical reads: 수천 페이지
CPU time: 높음
*/
```

### 1.2 B-Tree 인덱스 구조

```sql
-- 단일 컬럼 인덱스 생성
CREATE INDEX IX_Orders_CustomerId 
ON Orders(CustomerId);

-- 복합 인덱스 생성
CREATE INDEX IX_Orders_CustomerDate 
ON Orders(CustomerId, OrderDate);

-- 인덱스 정보 조회
SELECT 
    i.name AS IndexName,
    i.type_desc AS IndexType,
    i.is_unique,
    i.is_primary_key,
    STUFF((
        SELECT ', ' + c.name + ' (' + 
            CASE ic.is_descending_key 
                WHEN 1 THEN 'DESC' 
                ELSE 'ASC' 
            END + ')'
        FROM sys.index_columns ic
        INNER JOIN sys.columns c ON ic.object_id = c.object_id 
            AND ic.column_id = c.column_id
        WHERE ic.object_id = i.object_id 
            AND ic.index_id = i.index_id
        ORDER BY ic.key_ordinal
        FOR XML PATH('')
    ), 1, 2, '') AS Columns
FROM sys.indexes i
WHERE i.object_id = OBJECT_ID('Orders')
AND i.type > 0;

-- 인덱스 사용 통계
SELECT 
    OBJECT_NAME(s.object_id) AS TableName,
    i.name AS IndexName,
    s.user_seeks,
    s.user_scans,
    s.user_lookups,
    s.user_updates,
    s.last_user_seek,
    s.last_user_scan
FROM sys.dm_db_index_usage_stats s
INNER JOIN sys.indexes i ON s.object_id = i.object_id 
    AND s.index_id = i.index_id
WHERE s.database_id = DB_ID()
AND OBJECT_NAME(s.object_id) = 'Orders';
```

## 2. 인덱스 유형

### 2.1 클러스터드 인덱스

```sql
-- 클러스터드 인덱스 생성 (테이블당 하나만 가능)
-- Primary Key가 자동으로 클러스터드 인덱스가 됨
CREATE TABLE Products (
    ProductId INT NOT NULL,
    ProductName NVARCHAR(200) NOT NULL,
    CategoryId INT NOT NULL,
    Price DECIMAL(10, 2) NOT NULL,
    CreatedDate DATETIME NOT NULL DEFAULT GETDATE()
);

-- 명시적 클러스터드 인덱스 생성
CREATE CLUSTERED INDEX CIX_Products_ProductId 
ON Products(ProductId);

-- 복합 키로 클러스터드 인덱스
CREATE TABLE OrderItems (
    OrderId INT NOT NULL,
    ProductId INT NOT NULL,
    Quantity INT NOT NULL,
    UnitPrice DECIMAL(10, 2) NOT NULL
);

CREATE CLUSTERED INDEX CIX_OrderItems_OrderProduct 
ON OrderItems(OrderId, ProductId);

-- 클러스터드 인덱스 재구성
ALTER INDEX CIX_Products_ProductId ON Products REBUILD;

-- 온라인 인덱스 재구성 (Enterprise Edition)
ALTER INDEX CIX_Products_ProductId ON Products 
REBUILD WITH (ONLINE = ON);
```

### 2.2 넌클러스터드 인덱스

```sql
-- 다양한 넌클러스터드 인덱스
-- 1. 단일 컬럼 인덱스
CREATE NONCLUSTERED INDEX IX_Orders_OrderDate 
ON Orders(OrderDate);

-- 2. 복합 인덱스
CREATE NONCLUSTERED INDEX IX_Orders_StatusDate 
ON Orders(Status, OrderDate DESC);

-- 3. 포함 열이 있는 인덱스
CREATE NONCLUSTERED INDEX IX_Orders_CustomerAmount 
ON Orders(CustomerId, OrderDate)
INCLUDE (TotalAmount, Status);

-- 4. 필터링된 인덱스 (SQL Server 2008+)
CREATE NONCLUSTERED INDEX IX_Orders_PendingOrders 
ON Orders(OrderDate, CustomerId)
WHERE Status = 'Pending';

-- 5. 유니크 인덱스
CREATE UNIQUE NONCLUSTERED INDEX UIX_Customers_Email 
ON Customers(Email);

-- 인덱스 사용 예제
-- 포함 열 활용
SELECT CustomerId, OrderDate, TotalAmount, Status
FROM Orders
WHERE CustomerId = 1234
AND OrderDate >= '2024-01-01';
-- IX_Orders_CustomerAmount 인덱스만으로 처리 (Key Lookup 없음)
```

### 2.3 특수 인덱스

```sql
-- 1. Columnstore 인덱스 (SQL Server 2012+)
-- 분석 워크로드에 최적화
CREATE COLUMNSTORE INDEX CCI_Orders_Analytics
ON Orders(OrderId, CustomerId, OrderDate, TotalAmount, Status);

-- 클러스터드 컬럼스토어 인덱스
CREATE CLUSTERED COLUMNSTORE INDEX CCI_OrdersArchive
ON OrdersArchive;

-- 2. 전체 텍스트 인덱스
-- 전체 텍스트 카탈로그 생성
CREATE FULLTEXT CATALOG ProductCatalog;

-- 전체 텍스트 인덱스 생성
CREATE FULLTEXT INDEX ON Products(ProductName, Description)
KEY INDEX PK_Products
ON ProductCatalog;

-- 전체 텍스트 검색
SELECT ProductId, ProductName
FROM Products
WHERE CONTAINS(ProductName, 'laptop OR notebook');

-- 3. 공간 인덱스
CREATE TABLE Locations (
    LocationId INT PRIMARY KEY,
    LocationName NVARCHAR(100),
    GeoLocation GEOGRAPHY
);

CREATE SPATIAL INDEX SIX_Locations_GeoLocation
ON Locations(GeoLocation);

-- 4. XML 인덱스
CREATE TABLE Orders_XML (
    OrderId INT PRIMARY KEY,
    OrderDetails XML
);

-- Primary XML 인덱스
CREATE PRIMARY XML INDEX PXML_Orders_Details
ON Orders_XML(OrderDetails);

-- Secondary XML 인덱스
CREATE XML INDEX IXML_Orders_Path
ON Orders_XML(OrderDetails)
USING XML INDEX PXML_Orders_Details
FOR PATH;
```

## 3. 인덱스 설계 전략

### 3.1 인덱스 선택 기준

```sql
-- 카디널리티 분석
SELECT 
    'CustomerId' AS ColumnName,
    COUNT(DISTINCT CustomerId) AS DistinctValues,
    COUNT(*) AS TotalRows,
    CAST(COUNT(DISTINCT CustomerId) AS FLOAT) / COUNT(*) * 100 AS Selectivity
FROM Orders
UNION ALL
SELECT 
    'Status',
    COUNT(DISTINCT Status),
    COUNT(*),
    CAST(COUNT(DISTINCT Status) AS FLOAT) / COUNT(*) * 100
FROM Orders
UNION ALL
SELECT 
    'OrderDate',
    COUNT(DISTINCT CAST(OrderDate AS DATE)),
    COUNT(*),
    CAST(COUNT(DISTINCT CAST(OrderDate AS DATE)) AS FLOAT) / COUNT(*) * 100
FROM Orders;

-- 쿼리 패턴 분석
-- 자주 사용되는 WHERE 조건 확인
SELECT 
    cp.objtype,
    cp.usecounts,
    cp.size_in_bytes,
    st.text,
    qp.query_plan
FROM sys.dm_exec_cached_plans cp
CROSS APPLY sys.dm_exec_sql_text(cp.plan_handle) st
CROSS APPLY sys.dm_exec_query_plan(cp.plan_handle) qp
WHERE st.text LIKE '%Orders%'
AND cp.objtype = 'Adhoc'
ORDER BY cp.usecounts DESC;

-- 누락된 인덱스 확인
SELECT 
    mig.index_group_handle,
    mid.index_handle,
    CONVERT(decimal(28,1), migs.avg_total_user_cost * migs.avg_user_impact * (migs.user_seeks + migs.user_scans)) AS improvement_measure,
    'CREATE INDEX missing_index_' + CONVERT(varchar, mig.index_group_handle) + '_' + CONVERT(varchar, mid.index_handle) 
    + ' ON ' + mid.statement 
    + ' (' + ISNULL(mid.equality_columns,'') 
    + CASE WHEN mid.equality_columns IS NOT NULL AND mid.inequality_columns IS NOT NULL THEN ',' ELSE '' END 
    + ISNULL(mid.inequality_columns, '')
    + ')' 
    + ISNULL(' INCLUDE (' + mid.included_columns + ')', '') AS create_index_statement,
    migs.*
FROM sys.dm_db_missing_index_groups mig
INNER JOIN sys.dm_db_missing_index_group_stats migs ON migs.group_handle = mig.index_group_handle
INNER JOIN sys.dm_db_missing_index_details mid ON mig.index_handle = mid.index_handle
WHERE mid.database_id = DB_ID()
ORDER BY improvement_measure DESC;
```

### 3.2 복합 인덱스 설계

```sql
-- 복합 인덱스 컬럼 순서의 중요성
-- 잘못된 순서
CREATE INDEX IX_Orders_Bad 
ON Orders(OrderDate, CustomerId);

-- 올바른 순서 (더 선택적인 컬럼이 먼저)
CREATE INDEX IX_Orders_Good 
ON Orders(CustomerId, OrderDate);

-- 쿼리별 최적 인덱스
-- Query 1: WHERE CustomerId = ? AND OrderDate >= ?
CREATE INDEX IX_Orders_Q1 
ON Orders(CustomerId, OrderDate);

-- Query 2: WHERE Status = ? ORDER BY OrderDate DESC
CREATE INDEX IX_Orders_Q2 
ON Orders(Status, OrderDate DESC);

-- Query 3: WHERE CustomerId = ? AND Status = ? ORDER BY OrderDate DESC
CREATE INDEX IX_Orders_Q3 
ON Orders(CustomerId, Status, OrderDate DESC);

-- 인덱스 조합 테스트
DECLARE @CustomerId INT = 5000;
DECLARE @StartDate DATE = '2023-01-01';

-- 인덱스 힌트로 특정 인덱스 강제 사용
SELECT * FROM Orders WITH (INDEX(IX_Orders_Q1))
WHERE CustomerId = @CustomerId
AND OrderDate >= @StartDate;

-- 인덱스 사용 여부 확인
SELECT * FROM Orders WITH (INDEX(0))  -- 테이블 스캔 강제
WHERE CustomerId = @CustomerId
AND OrderDate >= @StartDate;
```

### 3.3 커버링 인덱스

```sql
-- 커버링 인덱스 설계
-- 쿼리
SELECT CustomerId, OrderDate, TotalAmount, Status
FROM Orders
WHERE CustomerId = 1234
AND OrderDate >= '2024-01-01'
ORDER BY OrderDate DESC;

-- 일반 인덱스 (Key Lookup 발생)
CREATE INDEX IX_Orders_Regular
ON Orders(CustomerId, OrderDate);

-- 커버링 인덱스 (Key Lookup 없음)
CREATE INDEX IX_Orders_Covering
ON Orders(CustomerId, OrderDate)
INCLUDE (TotalAmount, Status);

-- 성능 비교
SET STATISTICS IO ON;

-- Key Lookup 발생
SELECT CustomerId, OrderDate, TotalAmount, Status
FROM Orders WITH (INDEX(IX_Orders_Regular))
WHERE CustomerId = 1234
AND OrderDate >= '2024-01-01';

-- Key Lookup 없음
SELECT CustomerId, OrderDate, TotalAmount, Status
FROM Orders WITH (INDEX(IX_Orders_Covering))
WHERE CustomerId = 1234
AND OrderDate >= '2024-01-01';
```

## 4. 인덱스 모니터링과 유지보수

### 4.1 인덱스 단편화 관리

```sql
-- 인덱스 단편화 확인
SELECT 
    OBJECT_NAME(ips.object_id) AS TableName,
    i.name AS IndexName,
    ips.index_type_desc,
    ips.avg_fragmentation_in_percent,
    ips.page_count,
    ips.avg_page_space_used_in_percent,
    ips.record_count,
    ips.ghost_record_count,
    ips.fragment_count
FROM sys.dm_db_index_physical_stats(
    DB_ID(), 
    NULL, 
    NULL, 
    NULL, 
    'LIMITED'
) ips
INNER JOIN sys.indexes i ON ips.object_id = i.object_id 
    AND ips.index_id = i.index_id
WHERE ips.avg_fragmentation_in_percent > 10
AND ips.page_count > 1000
ORDER BY ips.avg_fragmentation_in_percent DESC;

-- 인덱스 유지보수 스크립트
DECLARE @TableName NVARCHAR(128);
DECLARE @IndexName NVARCHAR(128);
DECLARE @Fragmentation FLOAT;
DECLARE @SQL NVARCHAR(MAX);

DECLARE index_cursor CURSOR FOR
SELECT 
    OBJECT_NAME(ips.object_id),
    i.name,
    ips.avg_fragmentation_in_percent
FROM sys.dm_db_index_physical_stats(
    DB_ID(), 
    NULL, 
    NULL, 
    NULL, 
    'LIMITED'
) ips
INNER JOIN sys.indexes i ON ips.object_id = i.object_id 
    AND ips.index_id = i.index_id
WHERE ips.avg_fragmentation_in_percent > 5
AND ips.page_count > 1000
AND i.name IS NOT NULL;

OPEN index_cursor;
FETCH NEXT FROM index_cursor INTO @TableName, @IndexName, @Fragmentation;

WHILE @@FETCH_STATUS = 0
BEGIN
    IF @Fragmentation < 30
    BEGIN
        -- REORGANIZE for moderate fragmentation
        SET @SQL = 'ALTER INDEX ' + @IndexName + ' ON ' + @TableName + ' REORGANIZE';
        PRINT 'Reorganizing: ' + @SQL;
        EXEC sp_executesql @SQL;
    END
    ELSE
    BEGIN
        -- REBUILD for high fragmentation
        SET @SQL = 'ALTER INDEX ' + @IndexName + ' ON ' + @TableName + ' REBUILD';
        PRINT 'Rebuilding: ' + @SQL;
        EXEC sp_executesql @SQL;
    END
    
    FETCH NEXT FROM index_cursor INTO @TableName, @IndexName, @Fragmentation;
END;

CLOSE index_cursor;
DEALLOCATE index_cursor;
```

### 4.2 인덱스 사용 분석

```sql
-- 사용되지 않는 인덱스 찾기
WITH IndexUsageStats AS (
    SELECT 
        OBJECT_NAME(s.object_id) AS TableName,
        i.name AS IndexName,
        i.type_desc AS IndexType,
        s.user_seeks + s.user_scans + s.user_lookups AS TotalReads,
        s.user_updates AS TotalWrites,
        s.last_user_seek,
        s.last_user_scan,
        s.last_user_lookup,
        s.last_user_update,
        i.is_primary_key,
        i.is_unique
    FROM sys.dm_db_index_usage_stats s
    INNER JOIN sys.indexes i ON s.object_id = i.object_id 
        AND s.index_id = i.index_id
    WHERE s.database_id = DB_ID()
    AND i.type > 0  -- 힙 제외
)
SELECT 
    TableName,
    IndexName,
    IndexType,
    TotalReads,
    TotalWrites,
    CASE 
        WHEN TotalReads = 0 AND TotalWrites > 0 THEN 'Never Used - Consider Dropping'
        WHEN TotalReads < TotalWrites / 10 THEN 'Rarely Used - Review'
        ELSE 'Active'
    END AS Recommendation,
    DATEDIFF(DAY, 
        ISNULL(ISNULL(last_user_seek, last_user_scan), last_user_lookup), 
        GETDATE()
    ) AS DaysSinceLastRead
FROM IndexUsageStats
WHERE is_primary_key = 0 
AND is_unique = 0
ORDER BY TotalReads ASC, TotalWrites DESC;

-- 중복 인덱스 찾기
WITH IndexColumns AS (
    SELECT 
        i.object_id,
        i.index_id,
        i.name AS IndexName,
        STUFF((
            SELECT ',' + c.name
            FROM sys.index_columns ic
            INNER JOIN sys.columns c ON ic.object_id = c.object_id 
                AND ic.column_id = c.column_id
            WHERE ic.object_id = i.object_id 
                AND ic.index_id = i.index_id
                AND ic.is_included_column = 0
            ORDER BY ic.key_ordinal
            FOR XML PATH('')
        ), 1, 1, '') AS KeyColumns,
        STUFF((
            SELECT ',' + c.name
            FROM sys.index_columns ic
            INNER JOIN sys.columns c ON ic.object_id = c.object_id 
                AND ic.column_id = c.column_id
            WHERE ic.object_id = i.object_id 
                AND ic.index_id = i.index_id
                AND ic.is_included_column = 1
            ORDER BY ic.column_id
            FOR XML PATH('')
        ), 1, 1, '') AS IncludedColumns
    FROM sys.indexes i
    WHERE i.type > 0
    AND i.is_primary_key = 0
)
SELECT 
    OBJECT_NAME(ic1.object_id) AS TableName,
    ic1.IndexName AS Index1,
    ic2.IndexName AS Index2,
    ic1.KeyColumns AS KeyColumns1,
    ic2.KeyColumns AS KeyColumns2,
    CASE 
        WHEN ic1.KeyColumns = ic2.KeyColumns THEN 'Duplicate'
        WHEN ic1.KeyColumns LIKE ic2.KeyColumns + '%' THEN 'Index2 is subset of Index1'
        WHEN ic2.KeyColumns LIKE ic1.KeyColumns + '%' THEN 'Index1 is subset of Index2'
        ELSE 'Overlapping'
    END AS Relationship
FROM IndexColumns ic1
INNER JOIN IndexColumns ic2 ON ic1.object_id = ic2.object_id 
    AND ic1.index_id < ic2.index_id
WHERE (ic1.KeyColumns = ic2.KeyColumns
    OR ic1.KeyColumns LIKE ic2.KeyColumns + '%'
    OR ic2.KeyColumns LIKE ic1.KeyColumns + '%');
```

## 5. 인덱스 최적화 기법

### 5.1 파티션 인덱스

```sql
-- 파티션 함수 생성
CREATE PARTITION FUNCTION PF_OrderDate (DATETIME)
AS RANGE RIGHT FOR VALUES (
    '2022-01-01', '2022-07-01',
    '2023-01-01', '2023-07-01',
    '2024-01-01', '2024-07-01'
);

-- 파티션 스키마 생성
CREATE PARTITION SCHEME PS_OrderDate
AS PARTITION PF_OrderDate
ALL TO ([PRIMARY]);

-- 파티션된 테이블 생성
CREATE TABLE Orders_Partitioned (
    OrderId INT NOT NULL,
    CustomerId INT NOT NULL,
    OrderDate DATETIME NOT NULL,
    TotalAmount DECIMAL(10, 2) NOT NULL,
    Status VARCHAR(20) NOT NULL
) ON PS_OrderDate(OrderDate);

-- 파티션된 인덱스
CREATE CLUSTERED INDEX CIX_Orders_Part_OrderDate
ON Orders_Partitioned(OrderDate)
ON PS_OrderDate(OrderDate);

CREATE NONCLUSTERED INDEX IX_Orders_Part_Customer
ON Orders_Partitioned(CustomerId, OrderDate)
ON PS_OrderDate(OrderDate);

-- 파티션 정보 조회
SELECT 
    p.partition_number,
    p.rows,
    prv.value AS BoundaryValue,
    ps.name AS PartitionScheme,
    pf.name AS PartitionFunction
FROM sys.partitions p
INNER JOIN sys.indexes i ON p.object_id = i.object_id 
    AND p.index_id = i.index_id
INNER JOIN sys.partition_schemes ps ON i.data_space_id = ps.data_space_id
INNER JOIN sys.partition_functions pf ON ps.function_id = pf.function_id
LEFT JOIN sys.partition_range_values prv ON pf.function_id = prv.function_id 
    AND p.partition_number = prv.boundary_id + 1
WHERE p.object_id = OBJECT_ID('Orders_Partitioned')
ORDER BY p.partition_number;
```

### 5.2 인덱스 압축

```sql
-- 페이지 압축을 사용한 인덱스
CREATE INDEX IX_Orders_Compressed
ON Orders(CustomerId, OrderDate)
WITH (DATA_COMPRESSION = PAGE);

-- 행 압축
CREATE INDEX IX_Orders_RowCompressed
ON Orders(Status, OrderDate)
WITH (DATA_COMPRESSION = ROW);

-- 압축 효과 예측
EXEC sp_estimate_data_compression_savings
    @schema_name = 'dbo',
    @object_name = 'Orders',
    @index_id = NULL,
    @partition_number = NULL,
    @data_compression = 'PAGE';

-- 기존 인덱스 압축
ALTER INDEX IX_Orders_CustomerId ON Orders
REBUILD WITH (DATA_COMPRESSION = PAGE);

-- 압축 상태 확인
SELECT 
    OBJECT_NAME(p.object_id) AS TableName,
    i.name AS IndexName,
    p.data_compression_desc
FROM sys.partitions p
INNER JOIN sys.indexes i ON p.object_id = i.object_id 
    AND p.index_id = i.index_id
WHERE p.object_id = OBJECT_ID('Orders')
AND i.type > 0;
```

### 5.3 인덱스 힌트와 강제 사용

```sql
-- 인덱스 힌트 사용
-- 특정 인덱스 강제 사용
SELECT * FROM Orders WITH (INDEX(IX_Orders_CustomerDate))
WHERE CustomerId = 1234
AND OrderDate >= '2024-01-01';

-- 인덱스 스캔 강제
SELECT * FROM Orders WITH (INDEX(IX_Orders_CustomerDate), FORCESCAN)
WHERE CustomerId = 1234;

-- 인덱스 탐색 강제
SELECT * FROM Orders WITH (INDEX(IX_Orders_CustomerDate), FORCESEEK)
WHERE CustomerId = 1234
AND OrderDate >= '2024-01-01';

-- 여러 인덱스 중 선택
SELECT * FROM Orders WITH (INDEX(IX_Orders_CustomerDate, IX_Orders_Status))
WHERE CustomerId = 1234
AND Status = 'Pending';

-- 테이블 스캔 강제 (인덱스 무시)
SELECT * FROM Orders WITH (INDEX(0))
WHERE CustomerId = 1234;

-- 쿼리 옵션으로 최적화 제어
SELECT * FROM Orders
WHERE CustomerId = 1234
OPTION (
    MAXDOP 4,  -- 병렬 처리 수준
    RECOMPILE, -- 재컴파일
    USE HINT('FORCE_DEFAULT_CARDINALITY_ESTIMATION')
);
```

## 6. 고급 인덱스 전략

### 6.1 인덱스 뷰

```sql
-- 인덱스 뷰 생성
CREATE VIEW vw_OrderSummary
WITH SCHEMABINDING
AS
SELECT 
    o.CustomerId,
    c.CustomerName,
    COUNT_BIG(*) AS OrderCount,
    SUM(o.TotalAmount) AS TotalRevenue,
    MIN(o.OrderDate) AS FirstOrderDate,
    MAX(o.OrderDate) AS LastOrderDate
FROM dbo.Orders o
INNER JOIN dbo.Customers c ON o.CustomerId = c.CustomerId
GROUP BY o.CustomerId, c.CustomerName;

-- 인덱스 뷰에 클러스터드 인덱스 생성
CREATE UNIQUE CLUSTERED INDEX CIX_vw_OrderSummary
ON vw_OrderSummary(CustomerId);

-- 추가 인덱스
CREATE NONCLUSTERED INDEX IX_vw_OrderSummary_Revenue
ON vw_OrderSummary(TotalRevenue DESC)
INCLUDE (CustomerName, OrderCount);

-- 인덱스 뷰 사용
SELECT * FROM vw_OrderSummary WITH (NOEXPAND)
WHERE TotalRevenue > 100000;
```

### 6.2 동적 인덱스 관리

```sql
-- 동적 인덱스 생성 프로시저
CREATE PROCEDURE sp_CreateDynamicIndex
    @TableName NVARCHAR(128),
    @Columns NVARCHAR(500),
    @IncludeColumns NVARCHAR(500) = NULL,
    @Filter NVARCHAR(1000) = NULL
AS
BEGIN
    DECLARE @SQL NVARCHAR(MAX);
    DECLARE @IndexName NVARCHAR(128);
    
    -- 인덱스 이름 생성
    SET @IndexName = 'IX_' + @TableName + '_' + 
        REPLACE(REPLACE(@Columns, ',', '_'), ' ', '');
    
    -- 동적 SQL 구성
    SET @SQL = 'CREATE NONCLUSTERED INDEX ' + @IndexName +
        ' ON ' + @TableName + '(' + @Columns + ')';
    
    IF @IncludeColumns IS NOT NULL
        SET @SQL = @SQL + ' INCLUDE (' + @IncludeColumns + ')';
    
    IF @Filter IS NOT NULL
        SET @SQL = @SQL + ' WHERE ' + @Filter;
    
    -- 인덱스 생성
    EXEC sp_executesql @SQL;
    
    PRINT 'Index created: ' + @IndexName;
END;

-- 사용 예
EXEC sp_CreateDynamicIndex 
    @TableName = 'Orders',
    @Columns = 'Status, OrderDate DESC',
    @IncludeColumns = 'CustomerId, TotalAmount',
    @Filter = 'Status IN (''Pending'', ''Processing'')';
```

### 6.3 인덱스 성능 비교

```sql
-- 인덱스 성능 테스트 프레임워크
CREATE PROCEDURE sp_CompareIndexPerformance
    @Query NVARCHAR(MAX)
AS
BEGIN
    DECLARE @StartTime DATETIME;
    DECLARE @EndTime DATETIME;
    DECLARE @Duration INT;
    DECLARE @IOStats TABLE (
        IndexName NVARCHAR(128),
        LogicalReads BIGINT,
        PhysicalReads BIGINT,
        Duration_ms INT
    );
    
    -- 캐시 정리
    DBCC DROPCLEANBUFFERS;
    DBCC FREEPROCCACHE;
    
    -- 원본 쿼리 (인덱스 자동 선택)
    SET STATISTICS IO ON;
    SET @StartTime = GETDATE();
    EXEC sp_executesql @Query;
    SET @EndTime = GETDATE();
    SET @Duration = DATEDIFF(MILLISECOND, @StartTime, @EndTime);
    
    -- 결과 저장
    INSERT INTO @IOStats VALUES ('Auto-selected', 
        @@TOTAL_READ, @@TOTAL_WRITE, @Duration);
    
    -- 각 인덱스별 테스트
    DECLARE @IndexName NVARCHAR(128);
    DECLARE index_cursor CURSOR FOR
        SELECT name FROM sys.indexes
        WHERE object_id = OBJECT_ID('Orders')
        AND type > 0;
    
    OPEN index_cursor;
    FETCH NEXT FROM index_cursor INTO @IndexName;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        -- 캐시 정리
        DBCC DROPCLEANBUFFERS;
        
        -- 특정 인덱스로 쿼리 실행
        DECLARE @QueryWithHint NVARCHAR(MAX);
        SET @QueryWithHint = REPLACE(@Query, 'FROM Orders', 
            'FROM Orders WITH (INDEX(' + @IndexName + '))');
        
        SET @StartTime = GETDATE();
        EXEC sp_executesql @QueryWithHint;
        SET @EndTime = GETDATE();
        SET @Duration = DATEDIFF(MILLISECOND, @StartTime, @EndTime);
        
        INSERT INTO @IOStats VALUES (@IndexName, 
            @@TOTAL_READ, @@TOTAL_WRITE, @Duration);
        
        FETCH NEXT FROM index_cursor INTO @IndexName;
    END;
    
    CLOSE index_cursor;
    DEALLOCATE index_cursor;
    
    -- 결과 출력
    SELECT * FROM @IOStats
    ORDER BY Duration_ms;
END;
```

## 7. 실무 최적화 사례

### 7.1 전자상거래 주문 시스템 최적화

```sql
-- 실제 쿼리 패턴 분석
-- 1. 고객별 최근 주문 조회
CREATE INDEX IX_Orders_CustomerRecent
ON Orders(CustomerId, OrderDate DESC)
INCLUDE (TotalAmount, Status);

-- 2. 상태별 처리 대기 주문
CREATE INDEX IX_Orders_StatusProcessing
ON Orders(Status, OrderDate)
WHERE Status IN ('Pending', 'Processing');

-- 3. 일별 매출 집계
CREATE INDEX IX_Orders_DateAmount
ON Orders(OrderDate, TotalAmount)
INCLUDE (CustomerId, Status);

-- 4. 고객 구매 분석
CREATE INDEX IX_Orders_CustomerAnalysis
ON Orders(CustomerId, ProductCategory, OrderDate)
INCLUDE (TotalAmount);

-- 복합 쿼리 최적화
WITH CustomerStats AS (
    SELECT 
        CustomerId,
        COUNT(*) AS OrderCount,
        SUM(TotalAmount) AS TotalSpent,
        MAX(OrderDate) AS LastOrderDate
    FROM Orders WITH (INDEX(IX_Orders_CustomerAnalysis))
    WHERE OrderDate >= DATEADD(YEAR, -1, GETDATE())
    GROUP BY CustomerId
),
HighValueCustomers AS (
    SELECT CustomerId
    FROM CustomerStats
    WHERE TotalSpent > 10000
    OR OrderCount > 50
)
SELECT 
    o.OrderId,
    o.OrderDate,
    o.TotalAmount,
    o.Status
FROM Orders o
INNER JOIN HighValueCustomers h ON o.CustomerId = h.CustomerId
WHERE o.Status = 'Pending'
ORDER BY o.OrderDate;
```

### 7.2 보고서 쿼리 최적화

```sql
-- 월별 판매 보고서 최적화
-- 기존 느린 쿼리
SELECT 
    YEAR(OrderDate) AS Year,
    MONTH(OrderDate) AS Month,
    ProductCategory,
    COUNT(*) AS OrderCount,
    SUM(TotalAmount) AS Revenue,
    AVG(TotalAmount) AS AvgOrderValue
FROM Orders
WHERE OrderDate >= '2023-01-01'
GROUP BY YEAR(OrderDate), MONTH(OrderDate), ProductCategory
ORDER BY Year, Month, Revenue DESC;

-- 최적화된 인덱스
CREATE INDEX IX_Orders_Reporting
ON Orders(OrderDate, ProductCategory)
INCLUDE (TotalAmount);

-- 계산 컬럼을 활용한 최적화
ALTER TABLE Orders
ADD YearMonth AS CAST(YEAR(OrderDate) * 100 + MONTH(OrderDate) AS INT) PERSISTED;

CREATE INDEX IX_Orders_YearMonth
ON Orders(YearMonth, ProductCategory)
INCLUDE (TotalAmount);

-- 최적화된 쿼리
SELECT 
    YearMonth / 100 AS Year,
    YearMonth % 100 AS Month,
    ProductCategory,
    COUNT(*) AS OrderCount,
    SUM(TotalAmount) AS Revenue,
    AVG(TotalAmount) AS AvgOrderValue
FROM Orders
WHERE YearMonth >= 202301
GROUP BY YearMonth, ProductCategory
ORDER BY YearMonth, Revenue DESC;
```

## 마무리

인덱스는 데이터베이스 성능 최적화의 가장 강력한 도구입니다. 적절한 인덱스 설계는 쿼리 성능을 극적으로 향상시킬 수 있지만, 과도한 인덱스는 오히려 성능을 저하시킬 수 있습니다. 지속적인 모니터링과 최적화를 통해 균형 잡힌 인덱스 전략을 유지하는 것이 중요합니다. 다음 장에서는 쿼리 최적화와 실행 계획 분석을 통한 성능 튜닝 기법을 학습하겠습니다.