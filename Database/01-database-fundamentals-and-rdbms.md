# 데이터베이스 기초와 RDBMS 개념

## 개요

데이터베이스는 현대 소프트웨어 개발의 핵심 구성 요소입니다. 이 장에서는 데이터베이스의 기본 개념부터 RDBMS의 핵심 원리, 그리고 주요 데이터베이스 시스템들의 특징을 학습합니다.

## 1. 데이터베이스의 역사와 발전

### 1.1 데이터베이스의 탄생

```
1960년대: 파일 시스템
- 데이터를 개별 파일로 저장
- 중복 데이터 문제
- 데이터 불일치 문제

1970년대: 계층적/네트워크 데이터베이스
- IMS (Information Management System)
- CODASYL 모델
- 복잡한 포인터 기반 구조

1970년: 관계형 모델 제안
- Edgar F. Codd의 논문
- 테이블 기반 구조
- SQL의 탄생

1980년대: RDBMS 상용화
- Oracle, DB2, SQL Server
- ACID 특성 확립
- 트랜잭션 개념 정립

2000년대: NoSQL 등장
- 빅데이터 처리 요구
- 수평적 확장성
- 다양한 데이터 모델
```

### 1.2 데이터베이스의 정의

데이터베이스는 체계적으로 구성된 데이터의 집합으로, 다음과 같은 특징을 갖습니다:

```csharp
// 데이터베이스가 해결하는 문제들
public class DatabaseBenefits
{
    // 1. 데이터 중복 최소화
    public void AvoidDataRedundancy()
    {
        // 파일 시스템: 각 애플리케이션이 별도 파일 관리
        // 학생정보.txt, 수강정보.txt, 성적정보.txt
        // → 학생 이름이 각 파일에 중복 저장
        
        // 데이터베이스: 정규화된 테이블 구조
        // Students 테이블: StudentId, Name, Email
        // Enrollments 테이블: StudentId, CourseId, Grade
        // → 학생 정보는 한 곳에만 저장
    }
    
    // 2. 데이터 무결성 보장
    public void EnsureDataIntegrity()
    {
        // 제약 조건을 통한 데이터 검증
        // PRIMARY KEY, FOREIGN KEY, UNIQUE, CHECK, NOT NULL
    }
    
    // 3. 동시성 제어
    public void HandleConcurrency()
    {
        // 여러 사용자가 동시에 접근해도 일관성 유지
        // 트랜잭션, 락킹 메커니즘
    }
    
    // 4. 보안
    public void ProvideSecurity()
    {
        // 사용자별 권한 관리
        // 뷰를 통한 데이터 접근 제어
    }
}
```

## 2. RDBMS vs NoSQL

### 2.1 관계형 데이터베이스 (RDBMS)

```sql
-- RDBMS의 특징을 보여주는 예제
-- 1. 구조화된 스키마
CREATE TABLE Customers (
    CustomerId INT PRIMARY KEY,
    Name NVARCHAR(100) NOT NULL,
    Email NVARCHAR(255) UNIQUE,
    CreatedDate DATETIME DEFAULT GETDATE()
);

CREATE TABLE Orders (
    OrderId INT PRIMARY KEY,
    CustomerId INT NOT NULL,
    OrderDate DATETIME NOT NULL,
    TotalAmount DECIMAL(10, 2),
    FOREIGN KEY (CustomerId) REFERENCES Customers(CustomerId)
);

-- 2. 관계를 통한 데이터 연결
SELECT 
    c.Name,
    COUNT(o.OrderId) as OrderCount,
    SUM(o.TotalAmount) as TotalSpent
FROM Customers c
LEFT JOIN Orders o ON c.CustomerId = o.CustomerId
GROUP BY c.CustomerId, c.Name;

-- 3. ACID 트랜잭션
BEGIN TRANSACTION;
    INSERT INTO Orders (CustomerId, OrderDate, TotalAmount)
    VALUES (1, GETDATE(), 99.99);
    
    UPDATE Customers 
    SET LastOrderDate = GETDATE()
    WHERE CustomerId = 1;
COMMIT TRANSACTION;
```

### 2.2 NoSQL 데이터베이스

```javascript
// NoSQL (MongoDB) 예제
// 1. 유연한 스키마
{
    "_id": ObjectId("..."),
    "name": "John Doe",
    "email": "john@example.com",
    "orders": [
        {
            "orderId": "ORD001",
            "date": ISODate("2024-01-15"),
            "items": [
                {"product": "Laptop", "price": 999.99},
                {"product": "Mouse", "price": 29.99}
            ]
        }
    ],
    // 동적으로 필드 추가 가능
    "preferences": {
        "newsletter": true,
        "theme": "dark"
    }
}

// 2. 수평적 확장 (Sharding)
sh.enableSharding("ecommerce")
sh.shardCollection("ecommerce.customers", {"region": 1})
```

### 2.3 비교 표

```csharp
public class DatabaseComparison
{
    public class Feature
    {
        public string Aspect { get; set; }
        public string RDBMS { get; set; }
        public string NoSQL { get; set; }
    }
    
    public List<Feature> ComparisonTable = new List<Feature>
    {
        new Feature 
        { 
            Aspect = "데이터 모델",
            RDBMS = "테이블, 행, 열",
            NoSQL = "문서, 키-값, 그래프, 컬럼"
        },
        new Feature 
        { 
            Aspect = "스키마",
            RDBMS = "엄격한 스키마",
            NoSQL = "유연한/동적 스키마"
        },
        new Feature 
        { 
            Aspect = "확장성",
            RDBMS = "수직적 확장 (Scale-up)",
            NoSQL = "수평적 확장 (Scale-out)"
        },
        new Feature 
        { 
            Aspect = "ACID",
            RDBMS = "완전한 ACID 지원",
            NoSQL = "최종 일관성 (Eventually Consistent)"
        },
        new Feature 
        { 
            Aspect = "쿼리 언어",
            RDBMS = "표준화된 SQL",
            NoSQL = "데이터베이스별 고유 API"
        }
    };
}
```

## 3. ACID 특성

### 3.1 Atomicity (원자성)

```sql
-- 원자성: 트랜잭션의 모든 작업이 성공하거나 모두 실패
CREATE PROCEDURE TransferMoney
    @FromAccount INT,
    @ToAccount INT,
    @Amount DECIMAL(10, 2)
AS
BEGIN
    BEGIN TRY
        BEGIN TRANSACTION;
            -- 출금
            UPDATE Accounts 
            SET Balance = Balance - @Amount
            WHERE AccountId = @FromAccount;
            
            -- 잔액 부족 체크
            IF (SELECT Balance FROM Accounts WHERE AccountId = @FromAccount) < 0
                THROW 50001, 'Insufficient funds', 1;
            
            -- 입금
            UPDATE Accounts 
            SET Balance = Balance + @Amount
            WHERE AccountId = @ToAccount;
            
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        -- 오류 발생 시 모든 변경사항 롤백
        ROLLBACK TRANSACTION;
        THROW;
    END CATCH
END;
```

### 3.2 Consistency (일관성)

```sql
-- 일관성: 트랜잭션 전후로 데이터베이스가 일관된 상태 유지
-- 제약 조건을 통한 일관성 보장
CREATE TABLE Products (
    ProductId INT PRIMARY KEY,
    Name NVARCHAR(100) NOT NULL,
    Price DECIMAL(10, 2) CHECK (Price > 0),
    Stock INT CHECK (Stock >= 0),
    CategoryId INT NOT NULL,
    FOREIGN KEY (CategoryId) REFERENCES Categories(CategoryId)
);

-- 트리거를 통한 비즈니스 규칙 적용
CREATE TRIGGER trg_UpdateProductStock
ON OrderItems
AFTER INSERT
AS
BEGIN
    UPDATE p
    SET p.Stock = p.Stock - i.Quantity
    FROM Products p
    INNER JOIN inserted i ON p.ProductId = i.ProductId;
    
    -- 재고가 음수가 되면 롤백
    IF EXISTS (SELECT 1 FROM Products WHERE Stock < 0)
    BEGIN
        RAISERROR('Insufficient stock', 16, 1);
        ROLLBACK TRANSACTION;
    END
END;
```

### 3.3 Isolation (격리성)

```sql
-- 격리성: 동시 실행되는 트랜잭션들이 서로 영향을 주지 않음
-- 격리 수준 설정
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- 격리 수준별 동작
/*
READ UNCOMMITTED: Dirty Read 허용
READ COMMITTED: Dirty Read 방지 (기본값)
REPEATABLE READ: Non-repeatable Read 방지
SERIALIZABLE: Phantom Read 방지
*/

-- 예제: 격리 수준에 따른 동작 차이
-- Session 1
BEGIN TRANSACTION;
UPDATE Products SET Price = 29.99 WHERE ProductId = 1;
-- 아직 커밋하지 않음

-- Session 2
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT * FROM Products WHERE ProductId = 1; -- Dirty Read 발생

SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
SELECT * FROM Products WHERE ProductId = 1; -- 대기 또는 이전 값 반환
```

### 3.4 Durability (지속성)

```sql
-- 지속성: 커밋된 트랜잭션은 시스템 장애에도 유지됨
-- 트랜잭션 로그와 체크포인트

-- 복구 모델 설정
ALTER DATABASE MyDatabase SET RECOVERY FULL;

-- 트랜잭션 로그 백업
BACKUP LOG MyDatabase 
TO DISK = 'C:\Backups\MyDatabase_Log.trn'
WITH COMPRESSION, STATS = 10;

-- Write-Ahead Logging (WAL)
-- 1. 변경사항을 먼저 로그에 기록
-- 2. 로그가 디스크에 쓰여진 후 데이터 페이지 수정
-- 3. 시스템 장애 시 로그를 통해 복구
```

## 4. 주요 RDBMS 비교

### 4.1 SQL Server

```sql
-- SQL Server 특징적인 기능들
-- 1. T-SQL 확장
WITH CustomerOrders AS (
    SELECT 
        CustomerId,
        OrderDate,
        TotalAmount,
        ROW_NUMBER() OVER (PARTITION BY CustomerId ORDER BY OrderDate DESC) as rn
    FROM Orders
)
SELECT * FROM CustomerOrders WHERE rn = 1;

-- 2. 인메모리 OLTP
CREATE TABLE CustomerSessions
(
    SessionId UNIQUEIDENTIFIER NOT NULL PRIMARY KEY NONCLUSTERED,
    CustomerId INT NOT NULL,
    LoginTime DATETIME2 NOT NULL,
    LastActivity DATETIME2 NOT NULL
) WITH (MEMORY_OPTIMIZED = ON, DURABILITY = SCHEMA_AND_DATA);

-- 3. Columnstore 인덱스
CREATE COLUMNSTORE INDEX CCI_FactSales
ON FactSales (DateKey, ProductKey, StoreKey, Quantity, Amount);
```

### 4.2 PostgreSQL

```sql
-- PostgreSQL 특징적인 기능들
-- 1. 고급 데이터 타입
CREATE TABLE Events (
    EventId SERIAL PRIMARY KEY,
    EventData JSONB,
    Location POINT,
    IpRange INET,
    Tags TEXT[]
);

-- JSONB 쿼리
SELECT EventData->>'user_name' as UserName
FROM Events
WHERE EventData @> '{"event_type": "login"}';

-- 2. 파티셔닝
CREATE TABLE Sales (
    SaleId BIGSERIAL,
    SaleDate DATE NOT NULL,
    Amount DECIMAL(10, 2)
) PARTITION BY RANGE (SaleDate);

CREATE TABLE Sales_2024_Q1 PARTITION OF Sales
FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');

-- 3. Full Text Search
CREATE INDEX idx_fts ON Articles USING gin(to_tsvector('english', content));

SELECT * FROM Articles
WHERE to_tsvector('english', content) @@ to_tsquery('english', 'database & performance');
```

### 4.3 MySQL

```sql
-- MySQL 특징적인 기능들
-- 1. 다양한 스토리지 엔진
CREATE TABLE Transactions (
    TransactionId BIGINT PRIMARY KEY,
    Amount DECIMAL(10, 2)
) ENGINE=InnoDB; -- 트랜잭션 지원

CREATE TABLE Logs (
    LogId BIGINT PRIMARY KEY,
    Message TEXT
) ENGINE=MyISAM; -- 빠른 읽기, 트랜잭션 미지원

-- 2. 복제 설정
CHANGE MASTER TO
    MASTER_HOST='master.example.com',
    MASTER_USER='replication',
    MASTER_PASSWORD='password',
    MASTER_LOG_FILE='mysql-bin.000001',
    MASTER_LOG_POS=154;

-- 3. 파티셔닝
CREATE TABLE AccessLogs (
    LogId BIGINT AUTO_INCREMENT,
    AccessTime DATETIME,
    UserId INT,
    PRIMARY KEY (LogId, AccessTime)
) PARTITION BY RANGE (YEAR(AccessTime)) (
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

### 4.4 비교 매트릭스

```csharp
public class DatabaseSystemComparison
{
    public enum Feature
    {
        OpenSource,
        JSONSupport,
        FullTextSearch,
        Replication,
        Sharding,
        InMemoryTables,
        WindowFunctions,
        CTEs,
        MaterializedViews,
        StoredProcedures
    }
    
    public Dictionary<string, Dictionary<Feature, string>> CompareFeatures()
    {
        return new Dictionary<string, Dictionary<Feature, string>>
        {
            ["SQL Server"] = new Dictionary<Feature, string>
            {
                [Feature.OpenSource] = "No (Express는 무료)",
                [Feature.JSONSupport] = "Yes (JSON 함수)",
                [Feature.FullTextSearch] = "Yes (Full-Text Index)",
                [Feature.Replication] = "Yes (다양한 옵션)",
                [Feature.InMemoryTables] = "Yes (In-Memory OLTP)",
                [Feature.WindowFunctions] = "Yes (완전 지원)"
            },
            ["PostgreSQL"] = new Dictionary<Feature, string>
            {
                [Feature.OpenSource] = "Yes",
                [Feature.JSONSupport] = "Yes (JSONB)",
                [Feature.FullTextSearch] = "Yes (내장)",
                [Feature.Replication] = "Yes (Streaming)",
                [Feature.MaterializedViews] = "Yes",
                [Feature.WindowFunctions] = "Yes (완전 지원)"
            },
            ["MySQL"] = new Dictionary<Feature, string>
            {
                [Feature.OpenSource] = "Yes",
                [Feature.JSONSupport] = "Yes (JSON 타입)",
                [Feature.FullTextSearch] = "Yes (FULLTEXT)",
                [Feature.Replication] = "Yes (Master-Slave)",
                [Feature.StoredProcedures] = "Yes",
                [Feature.WindowFunctions] = "Yes (8.0+)"
            }
        };
    }
}
```

## 5. 데이터베이스 선택 가이드

### 5.1 선택 기준

```csharp
public class DatabaseSelectionCriteria
{
    public class UseCase
    {
        public string Scenario { get; set; }
        public string RecommendedDB { get; set; }
        public string Reason { get; set; }
    }
    
    public List<UseCase> GetRecommendations()
    {
        return new List<UseCase>
        {
            new UseCase
            {
                Scenario = "엔터프라이즈 .NET 애플리케이션",
                RecommendedDB = "SQL Server",
                Reason = ".NET과의 완벽한 통합, Visual Studio 도구 지원"
            },
            new UseCase
            {
                Scenario = "복잡한 쿼리와 분석",
                RecommendedDB = "PostgreSQL",
                Reason = "고급 SQL 기능, 확장성, 무료"
            },
            new UseCase
            {
                Scenario = "웹 애플리케이션 (읽기 중심)",
                RecommendedDB = "MySQL",
                Reason = "빠른 읽기 성능, 널리 사용됨, 호스팅 지원"
            },
            new UseCase
            {
                Scenario = "실시간 분석",
                RecommendedDB = "PostgreSQL + TimescaleDB",
                Reason = "시계열 데이터 최적화"
            },
            new UseCase
            {
                Scenario = "마이크로서비스",
                RecommendedDB = "PostgreSQL 또는 MongoDB",
                Reason = "스키마 유연성, 수평 확장"
            }
        };
    }
}
```

### 5.2 마이그레이션 고려사항

```sql
-- SQL Server에서 PostgreSQL로 마이그레이션 예제
-- 1. 데이터 타입 매핑
/*
SQL Server          PostgreSQL
NVARCHAR     →     VARCHAR
DATETIME     →     TIMESTAMP
MONEY        →     DECIMAL
UNIQUEIDENTIFIER → UUID
*/

-- 2. 함수 변환
-- SQL Server
SELECT TOP 10 * FROM Customers ORDER BY CreatedDate DESC;
-- PostgreSQL
SELECT * FROM Customers ORDER BY CreatedDate DESC LIMIT 10;

-- 3. 저장 프로시저 변환
-- SQL Server T-SQL을 PostgreSQL PL/pgSQL로 변환
CREATE OR REPLACE FUNCTION GetCustomerOrders(p_customer_id INT)
RETURNS TABLE(OrderId INT, OrderDate TIMESTAMP, TotalAmount DECIMAL)
AS $$
BEGIN
    RETURN QUERY
    SELECT o.OrderId, o.OrderDate, o.TotalAmount
    FROM Orders o
    WHERE o.CustomerId = p_customer_id;
END;
$$ LANGUAGE plpgsql;
```

## 마무리

데이터베이스는 현대 애플리케이션의 핵심이며, RDBMS는 여전히 가장 널리 사용되는 데이터 저장소입니다. ACID 특성을 이해하고 각 데이터베이스 시스템의 특징을 파악하면, 프로젝트에 적합한 데이터베이스를 선택하고 효과적으로 활용할 수 있습니다. 다음 장에서는 SQL의 기초인 DDL과 DML을 자세히 다루겠습니다.