# SQL 고급 - 윈도우 함수와 CTE

## 개요

윈도우 함수와 CTE(Common Table Expression)는 복잡한 분석과 계층적 데이터 처리를 가능하게 하는 SQL의 고급 기능입니다. 이 장에서는 다양한 윈도우 함수의 활용법과 CTE를 통한 가독성 높은 쿼리 작성 방법을 학습합니다.

## 1. 윈도우 함수 기본

### 1.1 윈도우 함수의 구조

```sql
-- 윈도우 함수의 기본 문법
/*
함수() OVER (
    [PARTITION BY 컬럼]
    [ORDER BY 컬럼]
    [ROWS/RANGE 프레임]
)
*/

-- 실습 데이터 준비
CREATE TABLE Sales (
    SalesId INT PRIMARY KEY,
    EmployeeId INT,
    ProductId INT,
    SaleDate DATE,
    Quantity INT,
    Amount DECIMAL(10, 2),
    Region NVARCHAR(50),
    Category NVARCHAR(50)
);

INSERT INTO Sales VALUES
(1, 101, 1, '2024-01-05', 10, 1000, 'Seoul', 'Electronics'),
(2, 101, 2, '2024-01-10', 5, 500, 'Seoul', 'Electronics'),
(3, 102, 1, '2024-01-08', 8, 800, 'Seoul', 'Electronics'),
(4, 103, 3, '2024-01-15', 15, 1500, 'Busan', 'Clothing'),
(5, 101, 1, '2024-02-05', 12, 1200, 'Seoul', 'Electronics'),
(6, 102, 3, '2024-02-10', 20, 2000, 'Seoul', 'Clothing'),
(7, 103, 2, '2024-02-12', 7, 700, 'Busan', 'Electronics'),
(8, 104, 1, '2024-03-01', 25, 2500, 'Incheon', 'Electronics'),
(9, 101, 3, '2024-03-15', 18, 1800, 'Seoul', 'Clothing'),
(10, 102, 2, '2024-03-20', 10, 1000, 'Seoul', 'Electronics');
```

### 1.2 집계 윈도우 함수

```sql
-- SUM() OVER()
SELECT 
    SalesId,
    EmployeeId,
    SaleDate,
    Amount,
    SUM(Amount) OVER () AS TotalSales,
    SUM(Amount) OVER (PARTITION BY EmployeeId) AS EmployeeTotalSales,
    SUM(Amount) OVER (PARTITION BY YEAR(SaleDate), MONTH(SaleDate)) AS MonthlyTotal
FROM Sales
ORDER BY SaleDate;

-- 누적 합계
SELECT 
    SalesId,
    EmployeeId,
    SaleDate,
    Amount,
    SUM(Amount) OVER (ORDER BY SaleDate) AS RunningTotal,
    SUM(Amount) OVER (PARTITION BY EmployeeId ORDER BY SaleDate) AS EmployeeRunningTotal
FROM Sales
ORDER BY EmployeeId, SaleDate;

-- AVG(), COUNT(), MIN(), MAX()
SELECT 
    SalesId,
    Region,
    Category,
    Amount,
    AVG(Amount) OVER (PARTITION BY Region) AS RegionAverage,
    COUNT(*) OVER (PARTITION BY Category) AS CategoryCount,
    MIN(Amount) OVER (PARTITION BY Region) AS RegionMin,
    MAX(Amount) OVER (PARTITION BY Region) AS RegionMax
FROM Sales;

-- 이동 평균 (Moving Average)
SELECT 
    SaleDate,
    Amount,
    AVG(Amount) OVER (
        ORDER BY SaleDate 
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS ThreeDayMovingAvg,
    AVG(Amount) OVER (
        ORDER BY SaleDate 
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS SevenDayMovingAvg
FROM Sales
ORDER BY SaleDate;
```

## 2. 순위 함수

### 2.1 ROW_NUMBER, RANK, DENSE_RANK

```sql
-- 순위 함수 비교
SELECT 
    EmployeeId,
    Region,
    Amount,
    ROW_NUMBER() OVER (ORDER BY Amount DESC) AS RowNum,
    RANK() OVER (ORDER BY Amount DESC) AS Rank,
    DENSE_RANK() OVER (ORDER BY Amount DESC) AS DenseRank
FROM Sales
ORDER BY Amount DESC;

-- PARTITION BY와 함께 사용
SELECT 
    EmployeeId,
    Region,
    Category,
    Amount,
    ROW_NUMBER() OVER (PARTITION BY Region ORDER BY Amount DESC) AS RegionRowNum,
    RANK() OVER (PARTITION BY Category ORDER BY Amount DESC) AS CategoryRank,
    DENSE_RANK() OVER (PARTITION BY Region, Category ORDER BY Amount DESC) AS RegionCategoryDenseRank
FROM Sales;

-- 각 직원의 최고 판매 기록 찾기
WITH RankedSales AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY EmployeeId ORDER BY Amount DESC) AS rn
    FROM Sales
)
SELECT * FROM RankedSales WHERE rn = 1;

-- 상위 N개 추출 (각 지역별 상위 3개)
WITH RegionTopSales AS (
    SELECT 
        *,
        DENSE_RANK() OVER (PARTITION BY Region ORDER BY Amount DESC) AS dr
    FROM Sales
)
SELECT * FROM RegionTopSales WHERE dr <= 3;
```

### 2.2 NTILE

```sql
-- 사분위수 계산
SELECT 
    SalesId,
    EmployeeId,
    Amount,
    NTILE(4) OVER (ORDER BY Amount) AS Quartile,
    NTILE(10) OVER (ORDER BY Amount) AS Decile,
    NTILE(100) OVER (ORDER BY Amount) AS Percentile
FROM Sales;

-- 그룹별 분위수
SELECT 
    Category,
    Amount,
    NTILE(3) OVER (PARTITION BY Category ORDER BY Amount) AS CategoryTertile,
    CASE 
        WHEN NTILE(3) OVER (PARTITION BY Category ORDER BY Amount) = 1 THEN 'Low'
        WHEN NTILE(3) OVER (PARTITION BY Category ORDER BY Amount) = 2 THEN 'Medium'
        ELSE 'High'
    END AS PerformanceGroup
FROM Sales;
```

## 3. 값 접근 함수

### 3.1 LAG와 LEAD

```sql
-- 이전/다음 행 값 접근
SELECT 
    SaleDate,
    EmployeeId,
    Amount,
    LAG(Amount, 1) OVER (ORDER BY SaleDate) AS PreviousAmount,
    LEAD(Amount, 1) OVER (ORDER BY SaleDate) AS NextAmount,
    Amount - LAG(Amount, 1, 0) OVER (ORDER BY SaleDate) AS ChangeFromPrevious
FROM Sales
ORDER BY SaleDate;

-- 직원별 이전 판매와 비교
SELECT 
    EmployeeId,
    SaleDate,
    Amount,
    LAG(SaleDate) OVER (PARTITION BY EmployeeId ORDER BY SaleDate) AS PreviousSaleDate,
    DATEDIFF(DAY, 
        LAG(SaleDate) OVER (PARTITION BY EmployeeId ORDER BY SaleDate), 
        SaleDate
    ) AS DaysSincePreviousSale,
    LAG(Amount) OVER (PARTITION BY EmployeeId ORDER BY SaleDate) AS PreviousAmount,
    CAST(Amount AS FLOAT) / NULLIF(LAG(Amount) OVER (PARTITION BY EmployeeId ORDER BY SaleDate), 0) * 100 AS PercentChange
FROM Sales
ORDER BY EmployeeId, SaleDate;

-- 여러 행 전후 접근
SELECT 
    SaleDate,
    Amount,
    LAG(Amount, 1) OVER (ORDER BY SaleDate) AS Lag1,
    LAG(Amount, 2) OVER (ORDER BY SaleDate) AS Lag2,
    LAG(Amount, 3) OVER (ORDER BY SaleDate) AS Lag3,
    LEAD(Amount, 1) OVER (ORDER BY SaleDate) AS Lead1,
    LEAD(Amount, 2) OVER (ORDER BY SaleDate) AS Lead2
FROM Sales
ORDER BY SaleDate;
```

### 3.2 FIRST_VALUE와 LAST_VALUE

```sql
-- 첫 번째와 마지막 값
SELECT 
    SalesId,
    EmployeeId,
    SaleDate,
    Amount,
    FIRST_VALUE(Amount) OVER (
        PARTITION BY EmployeeId 
        ORDER BY SaleDate
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS FirstSale,
    LAST_VALUE(Amount) OVER (
        PARTITION BY EmployeeId 
        ORDER BY SaleDate
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS LastSale
FROM Sales;

-- 월별 첫 판매와 마지막 판매
SELECT DISTINCT
    YEAR(SaleDate) AS Year,
    MONTH(SaleDate) AS Month,
    FIRST_VALUE(Amount) OVER (
        PARTITION BY YEAR(SaleDate), MONTH(SaleDate) 
        ORDER BY SaleDate
    ) AS MonthFirstSale,
    LAST_VALUE(Amount) OVER (
        PARTITION BY YEAR(SaleDate), MONTH(SaleDate) 
        ORDER BY SaleDate
        RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS MonthLastSale
FROM Sales
ORDER BY Year, Month;
```

## 4. 고급 윈도우 프레임

### 4.1 ROWS vs RANGE

```sql
-- ROWS: 물리적 행 기준
SELECT 
    SaleDate,
    Amount,
    SUM(Amount) OVER (
        ORDER BY SaleDate
        ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
    ) AS RowsSum,
    AVG(Amount) OVER (
        ORDER BY SaleDate
        ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
    ) AS RowsAvg
FROM Sales
ORDER BY SaleDate;

-- RANGE: 논리적 값 기준
SELECT 
    SaleDate,
    Amount,
    SUM(Amount) OVER (
        ORDER BY Amount
        RANGE BETWEEN 100 PRECEDING AND 100 FOLLOWING
    ) AS RangeSum,
    COUNT(*) OVER (
        ORDER BY Amount
        RANGE BETWEEN 100 PRECEDING AND 100 FOLLOWING
    ) AS RangeCount
FROM Sales
ORDER BY Amount;

-- 복잡한 프레임 정의
SELECT 
    SaleDate,
    Amount,
    -- 현재 행부터 끝까지
    SUM(Amount) OVER (
        ORDER BY SaleDate
        ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING
    ) AS RemainingTotal,
    -- 시작부터 현재 행까지
    AVG(Amount) OVER (
        ORDER BY SaleDate
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS CumulativeAvg,
    -- 전체 범위
    MAX(Amount) OVER (
        ORDER BY SaleDate
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS OverallMax
FROM Sales;
```

## 5. Common Table Expression (CTE)

### 5.1 기본 CTE

```sql
-- 단순 CTE
WITH SalesSummary AS (
    SELECT 
        EmployeeId,
        COUNT(*) AS SalesCount,
        SUM(Amount) AS TotalAmount,
        AVG(Amount) AS AvgAmount
    FROM Sales
    GROUP BY EmployeeId
)
SELECT * FROM SalesSummary
WHERE TotalAmount > 3000;

-- 여러 CTE 정의
WITH 
RegionSales AS (
    SELECT 
        Region,
        SUM(Amount) AS RegionTotal
    FROM Sales
    GROUP BY Region
),
CategorySales AS (
    SELECT 
        Category,
        SUM(Amount) AS CategoryTotal
    FROM Sales
    GROUP BY Category
),
TotalSales AS (
    SELECT SUM(Amount) AS GrandTotal FROM Sales
)
SELECT 
    r.Region,
    r.RegionTotal,
    CAST(r.RegionTotal AS FLOAT) / t.GrandTotal * 100 AS RegionPercentage,
    c.Category,
    c.CategoryTotal,
    CAST(c.CategoryTotal AS FLOAT) / t.GrandTotal * 100 AS CategoryPercentage
FROM RegionSales r
CROSS JOIN CategorySales c
CROSS JOIN TotalSales t
WHERE r.Region = 'Seoul';
```

### 5.2 재귀 CTE

```sql
-- 계층 구조 데이터 생성
CREATE TABLE EmployeeHierarchy (
    EmployeeId INT PRIMARY KEY,
    EmployeeName NVARCHAR(100),
    ManagerId INT
);

INSERT INTO EmployeeHierarchy VALUES
(1, 'CEO Kim', NULL),
(2, 'CTO Lee', 1),
(3, 'CFO Park', 1),
(4, 'Dev Manager Choi', 2),
(5, 'Finance Manager Jung', 3),
(6, 'Senior Dev A', 4),
(7, 'Senior Dev B', 4),
(8, 'Junior Dev C', 6),
(9, 'Accountant D', 5);

-- 재귀 CTE로 조직도 생성
WITH OrgChart AS (
    -- Anchor member: 최상위 직원
    SELECT 
        EmployeeId,
        EmployeeName,
        ManagerId,
        0 AS Level,
        CAST(EmployeeName AS NVARCHAR(1000)) AS Path
    FROM EmployeeHierarchy
    WHERE ManagerId IS NULL
    
    UNION ALL
    
    -- Recursive member: 하위 직원들
    SELECT 
        e.EmployeeId,
        e.EmployeeName,
        e.ManagerId,
        o.Level + 1,
        CAST(o.Path + ' > ' + e.EmployeeName AS NVARCHAR(1000))
    FROM EmployeeHierarchy e
    INNER JOIN OrgChart o ON e.ManagerId = o.EmployeeId
)
SELECT 
    EmployeeId,
    REPLICATE('  ', Level) + EmployeeName AS IndentedName,
    Level,
    Path
FROM OrgChart
ORDER BY Path;

-- 특정 직원의 모든 부하직원 찾기
WITH Subordinates AS (
    SELECT EmployeeId, EmployeeName, ManagerId
    FROM EmployeeHierarchy
    WHERE EmployeeId = 2  -- CTO Lee
    
    UNION ALL
    
    SELECT e.EmployeeId, e.EmployeeName, e.ManagerId
    FROM EmployeeHierarchy e
    INNER JOIN Subordinates s ON e.ManagerId = s.EmployeeId
)
SELECT * FROM Subordinates;

-- 날짜 시퀀스 생성
WITH DateSequence AS (
    SELECT CAST('2024-01-01' AS DATE) AS DateValue
    
    UNION ALL
    
    SELECT DATEADD(DAY, 1, DateValue)
    FROM DateSequence
    WHERE DateValue < '2024-01-31'
)
SELECT 
    DateValue,
    DATENAME(WEEKDAY, DateValue) AS DayName,
    CASE 
        WHEN DATEPART(WEEKDAY, DateValue) IN (1, 7) THEN 'Weekend'
        ELSE 'Weekday'
    END AS DayType
FROM DateSequence
OPTION (MAXRECURSION 31);
```

### 5.3 CTE와 윈도우 함수 결합

```sql
-- 판매 추세 분석
WITH MonthlySales AS (
    SELECT 
        YEAR(SaleDate) AS Year,
        MONTH(SaleDate) AS Month,
        SUM(Amount) AS MonthlyTotal,
        COUNT(*) AS TransactionCount
    FROM Sales
    GROUP BY YEAR(SaleDate), MONTH(SaleDate)
),
SalesTrend AS (
    SELECT 
        Year,
        Month,
        MonthlyTotal,
        TransactionCount,
        LAG(MonthlyTotal) OVER (ORDER BY Year, Month) AS PreviousMonth,
        AVG(MonthlyTotal) OVER (
            ORDER BY Year, Month 
            ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
        ) AS ThreeMonthAvg,
        SUM(MonthlyTotal) OVER (
            PARTITION BY Year 
            ORDER BY Month
        ) AS YearToDate
    FROM MonthlySales
)
SELECT 
    Year,
    Month,
    MonthlyTotal,
    ISNULL(MonthlyTotal - PreviousMonth, 0) AS MonthOverMonth,
    CASE 
        WHEN PreviousMonth > 0 
        THEN CAST((MonthlyTotal - PreviousMonth) AS FLOAT) / PreviousMonth * 100
        ELSE NULL
    END AS GrowthPercent,
    ThreeMonthAvg,
    YearToDate
FROM SalesTrend
ORDER BY Year, Month;

-- 퍼널 분석
WITH SalesFunnel AS (
    SELECT 
        EmployeeId,
        COUNT(CASE WHEN Amount < 1000 THEN 1 END) AS SmallSales,
        COUNT(CASE WHEN Amount >= 1000 AND Amount < 2000 THEN 1 END) AS MediumSales,
        COUNT(CASE WHEN Amount >= 2000 THEN 1 END) AS LargeSales,
        COUNT(*) AS TotalSales
    FROM Sales
    GROUP BY EmployeeId
),
RankedEmployees AS (
    SELECT 
        *,
        RANK() OVER (ORDER BY TotalSales DESC) AS SalesRank,
        CAST(LargeSales AS FLOAT) / NULLIF(TotalSales, 0) * 100 AS LargeSalesRatio
    FROM SalesFunnel
)
SELECT * FROM RankedEmployees
WHERE SalesRank <= 5;
```

## 6. PIVOT과 UNPIVOT

### 6.1 PIVOT

```sql
-- 월별 지역별 판매 피벗
SELECT *
FROM (
    SELECT 
        Region,
        MONTH(SaleDate) AS Month,
        Amount
    FROM Sales
) AS SourceTable
PIVOT (
    SUM(Amount)
    FOR Month IN ([1], [2], [3])
) AS PivotTable;

-- 동적 PIVOT
DECLARE @columns NVARCHAR(MAX), @sql NVARCHAR(MAX);

SELECT @columns = STRING_AGG(QUOTENAME(Category), ', ')
FROM (SELECT DISTINCT Category FROM Sales) AS Categories;

SET @sql = '
SELECT Region, ' + @columns + '
FROM (
    SELECT Region, Category, Amount
    FROM Sales
) AS SourceTable
PIVOT (
    SUM(Amount)
    FOR Category IN (' + @columns + ')
) AS PivotTable';

EXEC sp_executesql @sql;

-- 다중 집계 PIVOT (수동)
WITH PreparedData AS (
    SELECT 
        Region,
        Category,
        SUM(Amount) AS TotalAmount,
        COUNT(*) AS TransactionCount,
        AVG(Amount) AS AvgAmount
    FROM Sales
    GROUP BY Region, Category
)
SELECT 
    Region,
    MAX(CASE WHEN Category = 'Electronics' THEN TotalAmount END) AS Electronics_Total,
    MAX(CASE WHEN Category = 'Electronics' THEN TransactionCount END) AS Electronics_Count,
    MAX(CASE WHEN Category = 'Electronics' THEN AvgAmount END) AS Electronics_Avg,
    MAX(CASE WHEN Category = 'Clothing' THEN TotalAmount END) AS Clothing_Total,
    MAX(CASE WHEN Category = 'Clothing' THEN TransactionCount END) AS Clothing_Count,
    MAX(CASE WHEN Category = 'Clothing' THEN AvgAmount END) AS Clothing_Avg
FROM PreparedData
GROUP BY Region;
```

### 6.2 UNPIVOT

```sql
-- UNPIVOT 예제를 위한 테이블
CREATE TABLE QuarterlySales (
    EmployeeId INT,
    Q1 DECIMAL(10, 2),
    Q2 DECIMAL(10, 2),
    Q3 DECIMAL(10, 2),
    Q4 DECIMAL(10, 2)
);

INSERT INTO QuarterlySales VALUES
(101, 10000, 12000, 11000, 15000),
(102, 8000, 9000, 9500, 10000),
(103, 15000, 14000, 16000, 18000);

-- UNPIVOT 사용
SELECT EmployeeId, Quarter, Sales
FROM QuarterlySales
UNPIVOT (
    Sales FOR Quarter IN (Q1, Q2, Q3, Q4)
) AS UnpivotTable;

-- CROSS APPLY를 사용한 UNPIVOT (더 유연함)
SELECT 
    qs.EmployeeId,
    q.Quarter,
    q.Sales
FROM QuarterlySales qs
CROSS APPLY (
    VALUES 
        ('Q1', Q1),
        ('Q2', Q2),
        ('Q3', Q3),
        ('Q4', Q4)
) q(Quarter, Sales)
WHERE q.Sales IS NOT NULL;
```

## 7. 실무 활용 예제

### 7.1 판매 실적 대시보드

```sql
-- 종합 판매 대시보드 쿼리
WITH SalesMetrics AS (
    SELECT 
        EmployeeId,
        Region,
        Category,
        YEAR(SaleDate) AS Year,
        MONTH(SaleDate) AS Month,
        SUM(Amount) AS MonthlyAmount,
        COUNT(*) AS TransactionCount,
        AVG(Amount) AS AvgTransactionValue
    FROM Sales
    GROUP BY EmployeeId, Region, Category, YEAR(SaleDate), MONTH(SaleDate)
),
RankedMetrics AS (
    SELECT 
        *,
        -- 전체 순위
        RANK() OVER (ORDER BY MonthlyAmount DESC) AS OverallRank,
        -- 지역별 순위
        RANK() OVER (PARTITION BY Region ORDER BY MonthlyAmount DESC) AS RegionRank,
        -- 전월 대비
        LAG(MonthlyAmount) OVER (
            PARTITION BY EmployeeId 
            ORDER BY Year, Month
        ) AS PreviousMonth,
        -- 3개월 이동평균
        AVG(MonthlyAmount) OVER (
            PARTITION BY EmployeeId 
            ORDER BY Year, Month 
            ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
        ) AS ThreeMonthMA,
        -- 누적 판매
        SUM(MonthlyAmount) OVER (
            PARTITION BY EmployeeId, Year 
            ORDER BY Month
        ) AS YTD,
        -- 카테고리별 비중
        CAST(MonthlyAmount AS FLOAT) / SUM(MonthlyAmount) OVER (
            PARTITION BY Year, Month
        ) * 100 AS MonthlyShare
    FROM SalesMetrics
)
SELECT 
    EmployeeId,
    Region,
    Category,
    Year,
    Month,
    MonthlyAmount,
    OverallRank,
    RegionRank,
    ISNULL(MonthlyAmount - PreviousMonth, 0) AS MoM_Change,
    CASE 
        WHEN PreviousMonth > 0 
        THEN CAST((MonthlyAmount - PreviousMonth) AS FLOAT) / PreviousMonth * 100
        ELSE NULL 
    END AS MoM_GrowthRate,
    ThreeMonthMA,
    YTD,
    MonthlyShare
FROM RankedMetrics
WHERE Year = 2024
ORDER BY Year, Month, OverallRank;
```

### 7.2 코호트 분석

```sql
-- 고객 코호트 분석 (첫 구매 기준)
WITH CustomerFirstPurchase AS (
    SELECT 
        EmployeeId,
        MIN(SaleDate) AS FirstPurchaseDate,
        YEAR(MIN(SaleDate)) AS CohortYear,
        MONTH(MIN(SaleDate)) AS CohortMonth
    FROM Sales
    GROUP BY EmployeeId
),
CustomerActivity AS (
    SELECT 
        s.EmployeeId,
        cfp.CohortYear,
        cfp.CohortMonth,
        YEAR(s.SaleDate) AS ActivityYear,
        MONTH(s.SaleDate) AS ActivityMonth,
        DATEDIFF(MONTH, cfp.FirstPurchaseDate, s.SaleDate) AS MonthsSinceFirst,
        SUM(s.Amount) AS Revenue
    FROM Sales s
    INNER JOIN CustomerFirstPurchase cfp ON s.EmployeeId = cfp.EmployeeId
    GROUP BY s.EmployeeId, cfp.CohortYear, cfp.CohortMonth, 
             cfp.FirstPurchaseDate, YEAR(s.SaleDate), MONTH(s.SaleDate)
),
CohortRetention AS (
    SELECT 
        CohortYear,
        CohortMonth,
        MonthsSinceFirst,
        COUNT(DISTINCT EmployeeId) AS ActiveCustomers,
        SUM(Revenue) AS TotalRevenue,
        AVG(Revenue) AS AvgRevenue
    FROM CustomerActivity
    GROUP BY CohortYear, CohortMonth, MonthsSinceFirst
)
SELECT 
    CohortYear,
    CohortMonth,
    MonthsSinceFirst,
    ActiveCustomers,
    FIRST_VALUE(ActiveCustomers) OVER (
        PARTITION BY CohortYear, CohortMonth 
        ORDER BY MonthsSinceFirst
    ) AS CohortSize,
    CAST(ActiveCustomers AS FLOAT) / FIRST_VALUE(ActiveCustomers) OVER (
        PARTITION BY CohortYear, CohortMonth 
        ORDER BY MonthsSinceFirst
    ) * 100 AS RetentionRate,
    TotalRevenue,
    AvgRevenue
FROM CohortRetention
ORDER BY CohortYear, CohortMonth, MonthsSinceFirst;
```

### 7.3 이상치 탐지

```sql
-- 통계적 이상치 탐지
WITH SalesStats AS (
    SELECT 
        *,
        AVG(Amount) OVER () AS OverallAvg,
        STDEV(Amount) OVER () AS OverallStdev,
        AVG(Amount) OVER (PARTITION BY Category) AS CategoryAvg,
        STDEV(Amount) OVER (PARTITION BY Category) AS CategoryStdev,
        PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY Amount) 
            OVER (PARTITION BY Category) AS Q1,
        PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY Amount) 
            OVER (PARTITION BY Category) AS Q3
    FROM Sales
),
Outliers AS (
    SELECT 
        *,
        -- Z-score 방법
        ABS((Amount - OverallAvg) / NULLIF(OverallStdev, 0)) AS ZScore,
        -- IQR 방법
        Q3 - Q1 AS IQR,
        Q1 - 1.5 * (Q3 - Q1) AS LowerBound,
        Q3 + 1.5 * (Q3 - Q1) AS UpperBound,
        -- 이상치 플래그
        CASE 
            WHEN ABS((Amount - OverallAvg) / NULLIF(OverallStdev, 0)) > 3 
            THEN 'Z-Score Outlier'
            WHEN Amount < Q1 - 1.5 * (Q3 - Q1) OR Amount > Q3 + 1.5 * (Q3 - Q1)
            THEN 'IQR Outlier'
            ELSE 'Normal'
        END AS OutlierFlag
    FROM SalesStats
)
SELECT 
    SalesId,
    EmployeeId,
    Category,
    Amount,
    CategoryAvg,
    ZScore,
    OutlierFlag
FROM Outliers
WHERE OutlierFlag != 'Normal'
ORDER BY ZScore DESC;
```

## 마무리

윈도우 함수와 CTE는 SQL에서 복잡한 분석과 보고서를 작성하는 데 필수적인 도구입니다. 특히 시계열 분석, 순위 계산, 누적 집계 등에서 강력한 성능을 발휘합니다. 재귀 CTE를 활용하면 계층 구조 데이터도 효과적으로 처리할 수 있습니다. 다음 장에서는 저장 프로시저와 함수를 통한 재사용 가능한 데이터베이스 로직 구현을 학습하겠습니다.