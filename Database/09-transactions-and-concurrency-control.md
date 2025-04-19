# 트랜잭션과 동시성 제어

## 개요

트랜잭션과 동시성 제어는 데이터베이스의 일관성과 무결성을 보장하는 핵심 메커니즘입니다. 이 장에서는 ACID 속성의 실제 구현, 격리 수준별 동작 방식, 락킹 메커니즘, 데드락 처리, 그리고 낙관적/비관적 동시성 제어 전략을 학습합니다.

## 1. 트랜잭션 기본 개념

### 1.1 ACID 속성 구현

```sql
-- 원자성(Atomicity) 예제
BEGIN TRANSACTION;
    DECLARE @ErrorCount INT = 0;
    
    -- 계좌 이체 시나리오
    UPDATE Accounts 
    SET Balance = Balance - 1000 
    WHERE AccountId = 1;
    
    IF @@ERROR != 0 SET @ErrorCount = @ErrorCount + 1;
    
    UPDATE Accounts 
    SET Balance = Balance + 1000 
    WHERE AccountId = 2;
    
    IF @@ERROR != 0 SET @ErrorCount = @ErrorCount + 1;
    
    -- 트랜잭션 로그 기록
    INSERT INTO TransactionLog (FromAccount, ToAccount, Amount, TransactionDate)
    VALUES (1, 2, 1000, GETDATE());
    
    IF @@ERROR != 0 SET @ErrorCount = @ErrorCount + 1;
    
    -- 원자성 보장
    IF @ErrorCount = 0
        COMMIT TRANSACTION;
    ELSE
        ROLLBACK TRANSACTION;

-- 일관성(Consistency) 보장
CREATE TABLE Accounts (
    AccountId INT PRIMARY KEY,
    AccountNumber VARCHAR(20) UNIQUE NOT NULL,
    Balance DECIMAL(15, 2) NOT NULL,
    AccountType VARCHAR(20) NOT NULL,
    CONSTRAINT CHK_Balance CHECK (Balance >= 0),
    CONSTRAINT CHK_AccountType CHECK (AccountType IN ('Checking', 'Savings', 'Credit'))
);

-- 트리거를 통한 비즈니스 규칙 강제
CREATE TRIGGER trg_EnforceMinimumBalance
ON Accounts
AFTER UPDATE
AS
BEGIN
    IF EXISTS (
        SELECT 1 FROM inserted i
        INNER JOIN deleted d ON i.AccountId = d.AccountId
        WHERE i.AccountType = 'Savings' 
        AND i.Balance < 100
    )
    BEGIN
        RAISERROR('Savings account must maintain minimum balance of 100', 16, 1);
        ROLLBACK TRANSACTION;
    END
END;

-- 격리성(Isolation) 테스트
-- Session 1
BEGIN TRANSACTION;
UPDATE Products SET Price = Price * 1.1 WHERE CategoryId = 1;
-- 아직 COMMIT하지 않음

-- Session 2
SELECT * FROM Products WHERE CategoryId = 1;  -- 격리 수준에 따라 다른 결과

-- 지속성(Durability) 확인
-- 체크포인트 강제 실행
CHECKPOINT;

-- 트랜잭션 로그 정보 확인
SELECT 
    [Current LSN],
    [Transaction ID],
    Operation,
    Context,
    [Transaction Name],
    [Begin Time]
FROM fn_dblog(NULL, NULL)
WHERE [Transaction ID] != '0000:00000000';
```

### 1.2 트랜잭션 상태와 관리

```sql
-- 트랜잭션 상태 확인
SELECT 
    session_id,
    transaction_id,
    database_id,
    database_transaction_begin_time,
    database_transaction_type,
    database_transaction_state,
    database_transaction_log_record_count,
    database_transaction_log_bytes_used
FROM sys.dm_tran_database_transactions
WHERE database_id = DB_ID();

-- 활성 트랜잭션 모니터링
SELECT 
    st.session_id,
    st.transaction_id,
    st.enlist_count,
    st.is_user_transaction,
    st.is_local,
    st.is_enlisted,
    st.is_bound,
    at.name AS transaction_name,
    at.transaction_begin_time,
    CASE at.transaction_type
        WHEN 1 THEN 'Read/Write'
        WHEN 2 THEN 'Read-Only'
        WHEN 3 THEN 'System'
        WHEN 4 THEN 'Distributed'
    END AS transaction_type,
    CASE at.transaction_state
        WHEN 0 THEN 'Not initialized'
        WHEN 1 THEN 'Initialized but not started'
        WHEN 2 THEN 'Active'
        WHEN 3 THEN 'Ended (read-only)'
        WHEN 4 THEN 'Commit initiated'
        WHEN 5 THEN 'Prepared'
        WHEN 6 THEN 'Committed'
        WHEN 7 THEN 'Rolling back'
        WHEN 8 THEN 'Rolled back'
    END AS transaction_state
FROM sys.dm_tran_session_transactions st
LEFT JOIN sys.dm_tran_active_transactions at 
    ON st.transaction_id = at.transaction_id
WHERE st.session_id > 50;  -- 사용자 세션만

-- 저장점(Savepoint) 사용
BEGIN TRANSACTION;
    INSERT INTO Orders (CustomerId, OrderDate, TotalAmount)
    VALUES (1, GETDATE(), 100);
    
    SAVE TRANSACTION SavePoint1;
    
    INSERT INTO OrderDetails (OrderId, ProductId, Quantity, UnitPrice)
    VALUES (SCOPE_IDENTITY(), 1, 2, 50);
    
    -- 부분 롤백
    IF @@ERROR != 0
    BEGIN
        ROLLBACK TRANSACTION SavePoint1;
        -- SavePoint1 이후의 변경사항만 롤백
    END
    
COMMIT TRANSACTION;
```

## 2. 격리 수준 (Isolation Levels)

### 2.1 격리 수준별 동작

```sql
-- READ UNCOMMITTED (Dirty Read 허용)
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
BEGIN TRANSACTION;
    -- 커밋되지 않은 데이터도 읽을 수 있음
    SELECT * FROM Orders WITH (NOLOCK);
COMMIT;

-- READ COMMITTED (기본값)
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
BEGIN TRANSACTION;
    -- 커밋된 데이터만 읽음
    SELECT * FROM Orders;
    WAITFOR DELAY '00:00:05';
    -- 같은 쿼리 재실행 시 다른 결과 가능 (Non-repeatable read)
    SELECT * FROM Orders;
COMMIT;

-- REPEATABLE READ
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN TRANSACTION;
    SELECT * FROM Orders WHERE CustomerId = 1;
    -- 다른 세션에서 UPDATE는 블록됨
    -- 하지만 INSERT는 가능 (Phantom read)
    WAITFOR DELAY '00:00:05';
    SELECT * FROM Orders WHERE CustomerId = 1;
COMMIT;

-- SERIALIZABLE
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN TRANSACTION;
    SELECT COUNT(*) FROM Orders WHERE OrderDate >= '2024-01-01';
    -- 범위에 대한 INSERT도 블록됨
    WAITFOR DELAY '00:00:05';
    SELECT COUNT(*) FROM Orders WHERE OrderDate >= '2024-01-01';
COMMIT;

-- SNAPSHOT (MVCC)
-- 먼저 데이터베이스 레벨에서 활성화 필요
ALTER DATABASE YourDatabase SET ALLOW_SNAPSHOT_ISOLATION ON;

SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
BEGIN TRANSACTION;
    -- 트랜잭션 시작 시점의 데이터 스냅샷을 읽음
    SELECT * FROM Orders;
    -- 다른 트랜잭션의 변경사항이 보이지 않음
    WAITFOR DELAY '00:00:10';
    SELECT * FROM Orders;  -- 같은 결과
COMMIT;
```

### 2.2 격리 수준 비교 테스트

```sql
-- 테스트 환경 설정
CREATE TABLE IsolationTest (
    Id INT PRIMARY KEY,
    Value INT,
    Description VARCHAR(100)
);

INSERT INTO IsolationTest VALUES 
(1, 100, 'Initial'), 
(2, 200, 'Initial'),
(3, 300, 'Initial');

-- Dirty Read 테스트
-- Session 1
BEGIN TRANSACTION;
UPDATE IsolationTest SET Value = 999 WHERE Id = 1;
WAITFOR DELAY '00:00:10';
ROLLBACK;

-- Session 2 (동시 실행)
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT * FROM IsolationTest WHERE Id = 1;  -- 999 보임 (Dirty Read)

SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
SELECT * FROM IsolationTest WHERE Id = 1;  -- 대기 후 100 보임

-- Non-repeatable Read 테스트
-- Session 1
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
BEGIN TRANSACTION;
SELECT * FROM IsolationTest WHERE Id = 2;
WAITFOR DELAY '00:00:05';
SELECT * FROM IsolationTest WHERE Id = 2;  -- 다른 값 가능
COMMIT;

-- Session 2 (Session 1 대기 중 실행)
UPDATE IsolationTest SET Value = 250 WHERE Id = 2;

-- Phantom Read 테스트
-- Session 1
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN TRANSACTION;
SELECT * FROM IsolationTest WHERE Value > 200;
WAITFOR DELAY '00:00:05';
SELECT * FROM IsolationTest WHERE Value > 200;  -- 새 행 보임
COMMIT;

-- Session 2 (Session 1 대기 중 실행)
INSERT INTO IsolationTest VALUES (4, 400, 'Phantom');
```

## 3. 락킹 메커니즘

### 3.1 락 유형과 호환성

```sql
-- 락 정보 조회
SELECT 
    tl.request_session_id,
    tl.resource_type,
    tl.resource_database_id,
    DB_NAME(tl.resource_database_id) AS database_name,
    CASE 
        WHEN tl.resource_type = 'OBJECT' THEN OBJECT_NAME(tl.resource_associated_entity_id)
        WHEN tl.resource_type = 'DATABASE' THEN 'DATABASE'
        ELSE CAST(tl.resource_associated_entity_id AS VARCHAR(20))
    END AS resource_name,
    tl.request_mode,
    tl.request_type,
    tl.request_status,
    tl.request_reference_count,
    tl.request_lifetime,
    tl.request_owner_type
FROM sys.dm_tran_locks tl
WHERE tl.resource_database_id = DB_ID()
ORDER BY tl.request_session_id;

-- 락 호환성 매트릭스 구현
CREATE TABLE #LockCompatibility (
    RequestedMode VARCHAR(10),
    HeldMode VARCHAR(10),
    Compatible BIT
);

INSERT INTO #LockCompatibility VALUES
-- IS (Intent Shared)
('IS', 'IS', 1), ('IS', 'S', 1), ('IS', 'U', 1), ('IS', 'IX', 1), ('IS', 'SIX', 1), ('IS', 'X', 0),
-- S (Shared)
('S', 'IS', 1), ('S', 'S', 1), ('S', 'U', 1), ('S', 'IX', 0), ('S', 'SIX', 0), ('S', 'X', 0),
-- U (Update)
('U', 'IS', 1), ('U', 'S', 1), ('U', 'U', 0), ('U', 'IX', 0), ('U', 'SIX', 0), ('U', 'X', 0),
-- IX (Intent Exclusive)
('IX', 'IS', 1), ('IX', 'S', 0), ('IX', 'U', 0), ('IX', 'IX', 1), ('IX', 'SIX', 0), ('IX', 'X', 0),
-- SIX (Shared Intent Exclusive)
('SIX', 'IS', 1), ('SIX', 'S', 0), ('SIX', 'U', 0), ('SIX', 'IX', 0), ('SIX', 'SIX', 0), ('SIX', 'X', 0),
-- X (Exclusive)
('X', 'IS', 0), ('X', 'S', 0), ('X', 'U', 0), ('X', 'IX', 0), ('X', 'SIX', 0), ('X', 'X', 0);

-- 락 에스컬레이션 모니터링
SELECT 
    OBJECT_NAME(object_id) AS TableName,
    index_id,
    partition_number,
    lock_escalation_attempt_count,
    lock_escalation_count,
    page_lock_count,
    row_lock_count,
    page_lock_wait_count,
    row_lock_wait_count,
    page_lock_wait_in_ms,
    row_lock_wait_in_ms
FROM sys.dm_db_index_operational_stats(DB_ID(), NULL, NULL, NULL)
WHERE lock_escalation_count > 0
ORDER BY lock_escalation_count DESC;
```

### 3.2 락 힌트와 제어

```sql
-- 다양한 락 힌트
-- NOLOCK (READ UNCOMMITTED와 동일)
SELECT * FROM Orders WITH (NOLOCK)
WHERE OrderDate >= '2024-01-01';

-- READPAST (잠긴 행 건너뛰기)
SELECT * FROM Orders WITH (READPAST)
WHERE Status = 'Pending';

-- UPDLOCK (업데이트 락)
BEGIN TRANSACTION;
SELECT * FROM Orders WITH (UPDLOCK)
WHERE OrderId = 1234;
-- 다른 세션의 UPDATE 방지
UPDATE Orders SET Status = 'Processing'
WHERE OrderId = 1234;
COMMIT;

-- HOLDLOCK (SERIALIZABLE과 동일)
SELECT * FROM Orders WITH (HOLDLOCK)
WHERE CustomerId = 1;

-- ROWLOCK (행 수준 락 강제)
UPDATE Orders WITH (ROWLOCK)
SET Status = 'Shipped'
WHERE OrderId = 1234;

-- TABLOCK (테이블 수준 락)
SELECT COUNT(*) FROM Orders WITH (TABLOCK)
WHERE OrderDate >= '2024-01-01';

-- TABLOCKX (배타적 테이블 락)
DELETE FROM OrdersArchive WITH (TABLOCKX)
WHERE OrderDate < '2020-01-01';

-- 락 타임아웃 설정
SET LOCK_TIMEOUT 5000;  -- 5초

BEGIN TRY
    SELECT * FROM Orders WITH (UPDLOCK)
    WHERE OrderId = 1234;
END TRY
BEGIN CATCH
    IF ERROR_NUMBER() = 1222  -- Lock timeout
    BEGIN
        PRINT 'Lock timeout occurred';
    END
END CATCH;

-- 락 타임아웃 해제
SET LOCK_TIMEOUT -1;
```

## 4. 데드락 처리

### 4.1 데드락 감지와 분석

```sql
-- 데드락 추적 활성화
DBCC TRACEON(1222, -1);  -- 상세 데드락 정보를 에러 로그에 기록
DBCC TRACEON(1204, -1);  -- 데드락 그래프 정보

-- 데드락 시뮬레이션
-- Session 1
BEGIN TRANSACTION;
UPDATE Orders SET Status = 'Processing' WHERE OrderId = 1;
WAITFOR DELAY '00:00:05';
UPDATE Customers SET LastOrderDate = GETDATE() WHERE CustomerId = 1;
COMMIT;

-- Session 2 (동시 실행)
BEGIN TRANSACTION;
UPDATE Customers SET LastOrderDate = GETDATE() WHERE CustomerId = 1;
WAITFOR DELAY '00:00:05';
UPDATE Orders SET Status = 'Processing' WHERE OrderId = 1;
COMMIT;

-- 확장 이벤트로 데드락 모니터링
CREATE EVENT SESSION DeadlockMonitor
ON SERVER
ADD EVENT sqlserver.xml_deadlock_report(
    ACTION(
        sqlserver.client_app_name,
        sqlserver.client_hostname,
        sqlserver.database_id,
        sqlserver.database_name,
        sqlserver.plan_handle,
        sqlserver.session_id,
        sqlserver.sql_text,
        sqlserver.username
    )
)
ADD TARGET package0.event_file(
    SET filename = N'C:\Temp\DeadlockMonitor.xel',
    max_file_size = 5,
    max_rollover_files = 4
)
WITH (
    MAX_MEMORY = 4096 KB,
    EVENT_RETENTION_MODE = ALLOW_SINGLE_EVENT_LOSS,
    MAX_DISPATCH_LATENCY = 30 SECONDS,
    MAX_EVENT_SIZE = 0 KB,
    MEMORY_PARTITION_MODE = NONE,
    TRACK_CAUSALITY = OFF,
    STARTUP_STATE = ON
);

-- 세션 시작
ALTER EVENT SESSION DeadlockMonitor ON SERVER STATE = START;

-- 데드락 정보 조회
SELECT 
    xml_data.value('(event/@timestamp)[1]', 'datetime') AS DeadlockTime,
    xml_data.value('(event/data[@name="database_name"]/value)[1]', 'varchar(100)') AS DatabaseName,
    xml_data.query('(event/data[@name="xml_report"]/value/deadlock)[1]') AS DeadlockGraph
FROM (
    SELECT CAST(event_data AS XML) AS xml_data
    FROM sys.fn_xe_file_target_read_file('C:\Temp\DeadlockMonitor*.xel', NULL, NULL, NULL)
) AS event_table;
```

### 4.2 데드락 방지 전략

```sql
-- 1. 일관된 접근 순서
-- 좋은 예: 항상 같은 순서로 테이블 접근
CREATE PROCEDURE sp_UpdateOrderAndCustomer
    @OrderId INT,
    @CustomerId INT
AS
BEGIN
    BEGIN TRANSACTION;
        -- 항상 Orders -> Customers 순서
        UPDATE Orders SET LastModified = GETDATE() WHERE OrderId = @OrderId;
        UPDATE Customers SET LastOrderDate = GETDATE() WHERE CustomerId = @CustomerId;
    COMMIT;
END;

-- 2. 짧은 트랜잭션
-- 나쁜 예
BEGIN TRANSACTION;
    UPDATE Orders SET Status = 'Processing' WHERE OrderId = @OrderId;
    
    -- 긴 처리 작업
    EXEC sp_SendEmailNotification @OrderId;
    
    UPDATE Inventory SET Quantity = Quantity - 1 WHERE ProductId = @ProductId;
COMMIT;

-- 좋은 예
UPDATE Orders SET Status = 'Processing' WHERE OrderId = @OrderId;

-- 트랜잭션 밖에서 처리
EXEC sp_SendEmailNotification @OrderId;

UPDATE Inventory SET Quantity = Quantity - 1 WHERE ProductId = @ProductId;

-- 3. 인덱스 활용
-- 적절한 인덱스로 락 범위 최소화
CREATE INDEX IX_Orders_CustomerId_Status 
ON Orders(CustomerId, Status)
INCLUDE (OrderDate, TotalAmount);

-- 4. SNAPSHOT 격리 수준 사용
ALTER DATABASE YourDatabase SET READ_COMMITTED_SNAPSHOT ON;

-- 5. 재시도 로직
CREATE PROCEDURE sp_ProcessOrderWithRetry
    @OrderId INT
AS
BEGIN
    DECLARE @RetryCount INT = 0;
    DECLARE @MaxRetries INT = 3;
    
    WHILE @RetryCount < @MaxRetries
    BEGIN
        BEGIN TRY
            BEGIN TRANSACTION;
                UPDATE Orders SET Status = 'Processing' 
                WHERE OrderId = @OrderId;
                
                -- 다른 처리...
                
            COMMIT;
            BREAK;  -- 성공 시 종료
        END TRY
        BEGIN CATCH
            IF @@TRANCOUNT > 0 ROLLBACK;
            
            IF ERROR_NUMBER() = 1205  -- 데드락
            BEGIN
                SET @RetryCount = @RetryCount + 1;
                IF @RetryCount < @MaxRetries
                BEGIN
                    WAITFOR DELAY '00:00:00.100';  -- 100ms 대기
                    CONTINUE;
                END
            END
            
            -- 데드락이 아니거나 재시도 초과
            THROW;
        END CATCH;
    END;
END;
```

## 5. 낙관적 vs 비관적 동시성 제어

### 5.1 비관적 동시성 제어

```sql
-- 비관적 락킹 구현
CREATE PROCEDURE sp_UpdateProductPessimistic
    @ProductId INT,
    @NewPrice DECIMAL(10, 2)
AS
BEGIN
    BEGIN TRANSACTION;
        -- 즉시 배타적 락 획득
        SELECT * FROM Products WITH (UPDLOCK, HOLDLOCK)
        WHERE ProductId = @ProductId;
        
        -- 비즈니스 로직 수행
        WAITFOR DELAY '00:00:02';  -- 시뮬레이션
        
        UPDATE Products 
        SET Price = @NewPrice,
            LastModified = GETDATE()
        WHERE ProductId = @ProductId;
        
    COMMIT;
END;

-- 응용 프로그램 수준 락
CREATE TABLE ApplicationLocks (
    ResourceName NVARCHAR(255) PRIMARY KEY,
    LockedBy NVARCHAR(128),
    LockedAt DATETIME,
    ExpiresAt DATETIME
);

CREATE PROCEDURE sp_AcquireApplicationLock
    @ResourceName NVARCHAR(255),
    @LockedBy NVARCHAR(128),
    @TimeoutSeconds INT = 30
AS
BEGIN
    DECLARE @Now DATETIME = GETDATE();
    DECLARE @ExpiresAt DATETIME = DATEADD(SECOND, @TimeoutSeconds, @Now);
    
    -- 만료된 락 정리
    DELETE FROM ApplicationLocks
    WHERE ExpiresAt < @Now;
    
    -- 락 획득 시도
    BEGIN TRY
        INSERT INTO ApplicationLocks (ResourceName, LockedBy, LockedAt, ExpiresAt)
        VALUES (@ResourceName, @LockedBy, @Now, @ExpiresAt);
        
        RETURN 0;  -- 성공
    END TRY
    BEGIN CATCH
        IF ERROR_NUMBER() = 2627  -- Primary key violation
        BEGIN
            RETURN -1;  -- 락 획득 실패
        END
        THROW;
    END CATCH;
END;
```

### 5.2 낙관적 동시성 제어

```sql
-- 타임스탬프 기반 낙관적 동시성
ALTER TABLE Products
ADD RowVersion ROWVERSION NOT NULL;

CREATE PROCEDURE sp_UpdateProductOptimistic
    @ProductId INT,
    @NewPrice DECIMAL(10, 2),
    @OriginalRowVersion BINARY(8)
AS
BEGIN
    UPDATE Products
    SET Price = @NewPrice,
        LastModified = GETDATE()
    WHERE ProductId = @ProductId
    AND RowVersion = @OriginalRowVersion;
    
    IF @@ROWCOUNT = 0
    BEGIN
        -- 동시성 충돌
        RAISERROR('The product has been modified by another user', 16, 1);
    END
END;

-- 버전 번호 기반 낙관적 동시성
ALTER TABLE Orders
ADD VersionNumber INT NOT NULL DEFAULT 1;

CREATE TRIGGER trg_Orders_Version
ON Orders
AFTER UPDATE
AS
BEGIN
    UPDATE o
    SET VersionNumber = o.VersionNumber + 1
    FROM Orders o
    INNER JOIN inserted i ON o.OrderId = i.OrderId;
END;

CREATE PROCEDURE sp_UpdateOrderOptimistic
    @OrderId INT,
    @NewStatus VARCHAR(20),
    @ExpectedVersion INT
AS
BEGIN
    UPDATE Orders
    SET Status = @NewStatus
    WHERE OrderId = @OrderId
    AND VersionNumber = @ExpectedVersion;
    
    IF @@ROWCOUNT = 0
    BEGIN
        -- 현재 버전 확인
        DECLARE @CurrentVersion INT;
        SELECT @CurrentVersion = VersionNumber
        FROM Orders
        WHERE OrderId = @OrderId;
        
        IF @CurrentVersion IS NULL
            RAISERROR('Order not found', 16, 1);
        ELSE
            RAISERROR('Order version mismatch. Expected: %d, Current: %d', 
                      16, 1, @ExpectedVersion, @CurrentVersion);
    END
END;

-- 체크섬 기반 낙관적 동시성
CREATE FUNCTION fn_CalculateOrderChecksum(@OrderId INT)
RETURNS INT
AS
BEGIN
    DECLARE @Checksum INT;
    
    SELECT @Checksum = CHECKSUM_AGG(CHECKSUM(*))
    FROM (
        SELECT OrderId, CustomerId, OrderDate, Status, TotalAmount
        FROM Orders
        WHERE OrderId = @OrderId
        
        UNION ALL
        
        SELECT OrderId, ProductId, Quantity, UnitPrice, NULL
        FROM OrderDetails
        WHERE OrderId = @OrderId
    ) AS OrderData;
    
    RETURN @Checksum;
END;
```

## 6. 고급 동시성 패턴

### 6.1 MVCC (Multi-Version Concurrency Control)

```sql
-- Read Committed Snapshot Isolation 활성화
ALTER DATABASE YourDatabase SET READ_COMMITTED_SNAPSHOT ON;

-- Snapshot Isolation 활성화
ALTER DATABASE YourDatabase SET ALLOW_SNAPSHOT_ISOLATION ON;

-- 버전 저장소 모니터링
SELECT 
    DB_NAME(database_id) AS DatabaseName,
    SUM(version_store_reserved_page_count) * 8 / 1024.0 AS VersionStoreSizeMB,
    SUM(version_store_space_used_page_count) * 8 / 1024.0 AS VersionStoreUsedMB
FROM sys.dm_db_file_space_usage
GROUP BY database_id;

-- 장기 실행 트랜잭션 확인
SELECT 
    s.session_id,
    s.login_name,
    s.host_name,
    s.program_name,
    st.transaction_id,
    at.transaction_begin_time,
    DATEDIFF(SECOND, at.transaction_begin_time, GETDATE()) AS duration_seconds,
    at.transaction_type,
    at.transaction_state
FROM sys.dm_exec_sessions s
INNER JOIN sys.dm_tran_session_transactions st ON s.session_id = st.session_id
INNER JOIN sys.dm_tran_active_transactions at ON st.transaction_id = at.transaction_id
WHERE at.transaction_begin_time < DATEADD(MINUTE, -5, GETDATE())  -- 5분 이상
ORDER BY at.transaction_begin_time;

-- Snapshot 충돌 처리
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
BEGIN TRY
    BEGIN TRANSACTION;
        DECLARE @CurrentValue INT;
        
        SELECT @CurrentValue = Quantity
        FROM Inventory
        WHERE ProductId = 1;
        
        -- 다른 처리...
        WAITFOR DELAY '00:00:05';
        
        UPDATE Inventory
        SET Quantity = @CurrentValue - 10
        WHERE ProductId = 1;
        
    COMMIT;
END TRY
BEGIN CATCH
    IF ERROR_NUMBER() = 3960  -- Snapshot isolation conflict
    BEGIN
        -- 재시도 또는 다른 처리
        PRINT 'Snapshot conflict detected';
    END
    
    IF @@TRANCOUNT > 0 ROLLBACK;
END CATCH;
```

### 6.2 큐 기반 동시성 제어

```sql
-- 작업 큐 테이블
CREATE TABLE JobQueue (
    JobId INT IDENTITY(1,1) PRIMARY KEY,
    JobType VARCHAR(50) NOT NULL,
    JobData NVARCHAR(MAX),
    Status VARCHAR(20) NOT NULL DEFAULT 'Pending',
    CreatedAt DATETIME NOT NULL DEFAULT GETDATE(),
    ProcessingStartedAt DATETIME,
    ProcessingCompletedAt DATETIME,
    ProcessedBy NVARCHAR(128),
    ErrorMessage NVARCHAR(MAX),
    RetryCount INT NOT NULL DEFAULT 0,
    INDEX IX_JobQueue_Status_Created (Status, CreatedAt)
);

-- 작업 획득 프로시저 (경쟁 조건 방지)
CREATE PROCEDURE sp_DequeueJob
    @WorkerId NVARCHAR(128),
    @JobType VARCHAR(50) = NULL
AS
BEGIN
    DECLARE @JobId INT;
    
    -- 한 번에 하나의 작업만 획득
    UPDATE TOP(1) JobQueue
    SET @JobId = JobId,
        Status = 'Processing',
        ProcessingStartedAt = GETDATE(),
        ProcessedBy = @WorkerId
    OUTPUT INSERTED.*
    WHERE Status = 'Pending'
    AND (@JobType IS NULL OR JobType = @JobType)
    AND RetryCount < 3;
    
    RETURN @JobId;
END;

-- 배치 처리를 위한 파티션 락
CREATE PROCEDURE sp_ProcessBatchWithPartitionLock
    @BatchSize INT = 1000
AS
BEGIN
    DECLARE @PartitionKey INT = DATEPART(MINUTE, GETDATE()) % 10;
    
    -- 응용 프로그램 수준 락 사용
    EXEC sp_getapplock 
        @Resource = @PartitionKey,
        @LockMode = 'Exclusive',
        @LockTimeout = 10000;
    
    BEGIN TRY
        -- 배치 처리
        UPDATE TOP(@BatchSize) Orders
        SET ProcessedFlag = 1,
            ProcessedDate = GETDATE()
        WHERE ProcessedFlag = 0
        AND PartitionKey = @PartitionKey;
        
    END TRY
    BEGIN FINALLY
        EXEC sp_releaseapplock @Resource = @PartitionKey;
    END FINALLY;
END;
```

## 7. 실무 시나리오

### 7.1 전자상거래 재고 관리

```sql
-- 재고 차감 시나리오
CREATE PROCEDURE sp_DeductInventory
    @OrderId INT
AS
BEGIN
    SET XACT_ABORT ON;
    
    BEGIN TRY
        BEGIN TRANSACTION;
            -- 주문 상태 확인 및 락
            DECLARE @OrderStatus VARCHAR(20);
            
            SELECT @OrderStatus = Status
            FROM Orders WITH (UPDLOCK)
            WHERE OrderId = @OrderId;
            
            IF @OrderStatus != 'Pending'
            BEGIN
                THROW 50001, 'Order is not in pending status', 1;
            END
            
            -- 재고 차감
            UPDATE i
            SET i.Quantity = i.Quantity - od.Quantity
            FROM Inventory i WITH (ROWLOCK)
            INNER JOIN OrderDetails od ON i.ProductId = od.ProductId
            WHERE od.OrderId = @OrderId
            AND i.Quantity >= od.Quantity;
            
            IF @@ROWCOUNT < (SELECT COUNT(*) FROM OrderDetails WHERE OrderId = @OrderId)
            BEGIN
                THROW 50002, 'Insufficient inventory', 1;
            END
            
            -- 주문 상태 업데이트
            UPDATE Orders
            SET Status = 'Confirmed'
            WHERE OrderId = @OrderId;
            
        COMMIT;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK;
        THROW;
    END CATCH;
END;

-- 재고 예약 시스템
CREATE TABLE InventoryReservations (
    ReservationId INT IDENTITY(1,1) PRIMARY KEY,
    ProductId INT NOT NULL,
    Quantity INT NOT NULL,
    ReservedFor VARCHAR(50) NOT NULL,  -- OrderId or CartId
    ReservedAt DATETIME NOT NULL DEFAULT GETDATE(),
    ExpiresAt DATETIME NOT NULL,
    Status VARCHAR(20) NOT NULL DEFAULT 'Active',
    INDEX IX_Reservations_Product_Status (ProductId, Status, ExpiresAt)
);

CREATE PROCEDURE sp_ReserveInventory
    @ProductId INT,
    @Quantity INT,
    @ReservedFor VARCHAR(50),
    @ReservationMinutes INT = 15
AS
BEGIN
    BEGIN TRANSACTION;
        -- 만료된 예약 해제
        UPDATE InventoryReservations
        SET Status = 'Expired'
        WHERE ProductId = @ProductId
        AND Status = 'Active'
        AND ExpiresAt < GETDATE();
        
        -- 가용 재고 확인
        DECLARE @AvailableQuantity INT;
        
        SELECT @AvailableQuantity = i.Quantity - ISNULL(SUM(r.Quantity), 0)
        FROM Inventory i
        LEFT JOIN InventoryReservations r 
            ON i.ProductId = r.ProductId 
            AND r.Status = 'Active'
        WHERE i.ProductId = @ProductId
        GROUP BY i.Quantity;
        
        IF @AvailableQuantity >= @Quantity
        BEGIN
            INSERT INTO InventoryReservations 
                (ProductId, Quantity, ReservedFor, ExpiresAt)
            VALUES 
                (@ProductId, @Quantity, @ReservedFor, 
                 DATEADD(MINUTE, @ReservationMinutes, GETDATE()));
        END
        ELSE
        BEGIN
            THROW 50003, 'Insufficient available inventory', 1;
        END
    COMMIT;
END;
```

### 7.2 금융 트랜잭션 처리

```sql
-- 이체 트랜잭션 with 2PC
CREATE PROCEDURE sp_TransferMoney
    @FromAccountId INT,
    @ToAccountId INT,
    @Amount DECIMAL(15, 2),
    @TransactionId UNIQUEIDENTIFIER = NULL
AS
BEGIN
    SET XACT_ABORT ON;
    
    IF @TransactionId IS NULL
        SET @TransactionId = NEWID();
    
    -- 분산 트랜잭션 시작
    BEGIN DISTRIBUTED TRANSACTION;
        
        -- 중복 처리 방지
        IF EXISTS (
            SELECT 1 FROM TransactionLog 
            WHERE TransactionId = @TransactionId
        )
        BEGIN
            ROLLBACK;
            RETURN;
        END
        
        -- 출금 계좌 처리
        UPDATE Accounts
        SET Balance = Balance - @Amount,
            LastTransactionDate = GETDATE()
        WHERE AccountId = @FromAccountId
        AND Balance >= @Amount
        AND IsActive = 1;
        
        IF @@ROWCOUNT = 0
        BEGIN
            ROLLBACK;
            THROW 50004, 'Insufficient funds or account inactive', 1;
        END
        
        -- 입금 계좌 처리
        UPDATE Accounts
        SET Balance = Balance + @Amount,
            LastTransactionDate = GETDATE()
        WHERE AccountId = @ToAccountId
        AND IsActive = 1;
        
        IF @@ROWCOUNT = 0
        BEGIN
            ROLLBACK;
            THROW 50005, 'Target account not found or inactive', 1;
        END
        
        -- 트랜잭션 로그
        INSERT INTO TransactionLog 
            (TransactionId, FromAccountId, ToAccountId, 
             Amount, TransactionType, TransactionDate)
        VALUES 
            (@TransactionId, @FromAccountId, @ToAccountId, 
             @Amount, 'Transfer', GETDATE());
        
    COMMIT;
END;
```

## 마무리

트랜잭션과 동시성 제어는 데이터베이스 시스템의 핵심 기능입니다. 적절한 격리 수준 선택, 효율적인 락킹 전략, 데드락 방지, 그리고 상황에 맞는 낙관적/비관적 동시성 제어의 활용이 중요합니다. MVCC와 같은 현대적인 기법을 활용하면 높은 동시성과 성능을 달성할 수 있습니다. 다음 장에서는 백업과 복구 전략을 통한 데이터 보호 방법을 학습하겠습니다.