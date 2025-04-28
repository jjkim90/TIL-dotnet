# 복제와 샤딩

## 개요

데이터베이스의 확장성과 가용성을 높이기 위한 핵심 기술인 복제(Replication)와 샤딩(Sharding)을 학습합니다. 복제는 데이터의 사본을 여러 서버에 유지하여 읽기 성능과 가용성을 향상시키고, 샤딩은 데이터를 여러 서버에 분산하여 수평적 확장을 가능하게 합니다.

## 1. 복제 기본 개념

### 1.1 복제 유형과 토폴로지

```sql
-- SQL Server 복제 구성 확인
SELECT 
    name AS PublicationName,
    publication_type,
    CASE publication_type
        WHEN 0 THEN 'Transactional'
        WHEN 1 THEN 'Snapshot'
        WHEN 2 THEN 'Merge'
    END AS PublicationType,
    status,
    allow_push,
    allow_pull,
    allow_anonymous,
    immediate_sync
FROM distribution.dbo.MSpublications;

-- 복제 에이전트 상태
EXEC sp_replmonitorhelppublication;
EXEC sp_replmonitorhelpsubscription;

-- PostgreSQL 복제 설정 (postgresql.conf)
/*
wal_level = replica
max_wal_senders = 10
wal_keep_segments = 64
hot_standby = on
*/

-- PostgreSQL 주 서버에서 복제 사용자 생성
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'RepPass123!';

-- pg_hba.conf 설정
/*
host    replication     replicator      192.168.1.0/24      md5
*/

-- MySQL/MariaDB 복제 설정
-- 마스터 서버
CREATE USER 'repl_user'@'%' IDENTIFIED BY 'RepPass123!';
GRANT REPLICATION SLAVE ON *.* TO 'repl_user'@'%';

-- 바이너리 로그 위치 확인
SHOW MASTER STATUS;

-- 슬레이브 서버 설정
CHANGE MASTER TO
    MASTER_HOST='master_server_ip',
    MASTER_USER='repl_user',
    MASTER_PASSWORD='RepPass123!',
    MASTER_LOG_FILE='mysql-bin.000001',
    MASTER_LOG_POS=154;

START SLAVE;
SHOW SLAVE STATUS\G
```

### 1.2 트랜잭션 복제 설정

```sql
-- SQL Server 게시자(Publisher) 설정
USE master;
EXEC sp_adddistributor @distributor = N'DistributorServer';

-- 배포 데이터베이스 생성
EXEC sp_adddistributiondb 
    @database = N'distribution',
    @data_folder = N'C:\SQLData\',
    @log_folder = N'C:\SQLLog\',
    @log_file_size = 2,
    @min_distretention = 0,
    @max_distretention = 72,
    @history_retention = 48;

-- 게시자 등록
EXEC sp_adddistpublisher 
    @publisher = N'PublisherServer',
    @distribution_db = N'distribution',
    @security_mode = 1;

-- 게시 데이터베이스 활성화
USE YourDatabase;
EXEC sp_replicationdboption 
    @dbname = N'YourDatabase',
    @optname = N'publish',
    @value = N'true';

-- 트랜잭션 게시 생성
EXEC sp_addpublication 
    @publication = N'YourDB_Publication',
    @description = N'Transactional publication of YourDatabase',
    @sync_method = N'concurrent',
    @retention = 0,
    @allow_push = N'true',
    @allow_pull = N'true',
    @allow_anonymous = N'false',
    @snapshot_in_defaultfolder = N'true',
    @compress_snapshot = N'false',
    @repl_freq = N'continuous',
    @status = N'active';

-- 아티클 추가
EXEC sp_addarticle 
    @publication = N'YourDB_Publication',
    @article = N'Customers',
    @source_owner = N'dbo',
    @source_object = N'Customers',
    @type = N'logbased',
    @pre_creation_cmd = N'drop',
    @schema_option = 0x000000000803509F;

-- 구독 생성 (Push Subscription)
EXEC sp_addsubscription 
    @publication = N'YourDB_Publication',
    @subscriber = N'SubscriberServer',
    @destination_db = N'YourDatabase_Replica',
    @subscription_type = N'Push',
    @sync_type = N'automatic';

-- 구독 에이전트 추가
EXEC sp_addpushsubscription_agent 
    @publication = N'YourDB_Publication',
    @subscriber = N'SubscriberServer',
    @subscriber_db = N'YourDatabase_Replica',
    @job_login = NULL,
    @job_password = NULL,
    @subscriber_security_mode = 1;
```

### 1.3 복제 모니터링과 문제 해결

```sql
-- 복제 지연 모니터링
CREATE PROCEDURE sp_MonitorReplicationLatency
AS
BEGIN
    -- 트랜잭션 복제 지연
    SELECT 
        p.publication,
        s.subscriber,
        s.subscriber_db,
        da.name AS distribution_agent,
        da.cur_latency AS current_latency_sec,
        da.avg_latency AS average_latency_sec,
        CASE 
            WHEN da.cur_latency > 300 THEN 'Critical'
            WHEN da.cur_latency > 60 THEN 'Warning'
            ELSE 'Normal'
        END AS status,
        da.last_sync
    FROM distribution.dbo.MSdistribution_agents da
    INNER JOIN distribution.dbo.MSsubscriptions s 
        ON da.publisher_id = s.publisher_id 
        AND da.publisher_db = s.publisher_db
    INNER JOIN distribution.dbo.MSpublications p 
        ON s.publication_id = p.publication_id
    ORDER BY da.cur_latency DESC;
    
    -- 복제 오류 확인
    SELECT TOP 100
        time,
        error_code,
        error_text,
        xact_seqno,
        command_id
    FROM distribution.dbo.MSrepl_errors
    ORDER BY time DESC;
END;

-- 복제 성능 추적
CREATE TABLE ReplicationMetrics (
    MetricId INT IDENTITY(1,1) PRIMARY KEY,
    CaptureTime DATETIME DEFAULT GETDATE(),
    PublicationName NVARCHAR(128),
    SubscriberName NVARCHAR(128),
    LatencySeconds INT,
    DeliveredTransactions INT,
    DeliveredCommands INT,
    AverageCommands INT,
    DeliveryRate INT,
    INDEX IX_ReplicationMetrics_Time (CaptureTime)
);

-- 복제 성능 수집 작업
CREATE PROCEDURE sp_CollectReplicationMetrics
AS
BEGIN
    INSERT INTO ReplicationMetrics 
        (PublicationName, SubscriberName, LatencySeconds, 
         DeliveredTransactions, DeliveredCommands, AverageCommands, DeliveryRate)
    SELECT 
        p.publication,
        s.subscriber,
        h.cur_latency,
        h.delivered_transactions,
        h.delivered_commands,
        h.average_commands,
        h.delivery_rate
    FROM distribution.dbo.MSdistribution_history h
    INNER JOIN distribution.dbo.MSdistribution_agents a 
        ON h.agent_id = a.id
    INNER JOIN distribution.dbo.MSsubscriptions s 
        ON a.publisher_id = s.publisher_id 
        AND a.publisher_db = s.publisher_db
    INNER JOIN distribution.dbo.MSpublications p 
        ON s.publication_id = p.publication_id
    WHERE h.time = (
        SELECT MAX(time) 
        FROM distribution.dbo.MSdistribution_history 
        WHERE agent_id = h.agent_id
    );
    
    -- 오래된 데이터 정리
    DELETE FROM ReplicationMetrics
    WHERE CaptureTime < DATEADD(DAY, -30, GETDATE());
END;
```

## 2. 고급 복제 전략

### 2.1 양방향 복제

```sql
-- 충돌 해결을 위한 트리거
CREATE TRIGGER trg_ConflictResolution
ON Customers
INSTEAD OF UPDATE
AS
BEGIN
    -- 타임스탬프 기반 충돌 해결
    UPDATE c
    SET 
        c.CustomerName = 
            CASE 
                WHEN i.LastModified > c.LastModified THEN i.CustomerName
                ELSE c.CustomerName
            END,
        c.Email = 
            CASE 
                WHEN i.LastModified > c.LastModified THEN i.Email
                ELSE c.Email
            END,
        c.LastModified = GREATEST(i.LastModified, c.LastModified),
        c.ModifiedBy = 
            CASE 
                WHEN i.LastModified > c.LastModified THEN i.ModifiedBy
                ELSE c.ModifiedBy
            END
    FROM Customers c
    INNER JOIN inserted i ON c.CustomerId = i.CustomerId;
    
    -- 충돌 로그 기록
    INSERT INTO ConflictLog (TableName, RecordId, ConflictTime, Resolution)
    SELECT 
        'Customers',
        i.CustomerId,
        GETDATE(),
        CASE 
            WHEN i.LastModified > c.LastModified THEN 'Remote Win'
            ELSE 'Local Win'
        END
    FROM inserted i
    INNER JOIN Customers c ON i.CustomerId = c.CustomerId
    WHERE i.LastModified <> c.LastModified;
END;

-- 병합 복제 충돌 해결 프로시저
CREATE PROCEDURE sp_MergeConflictResolver
    @tableowner SYSNAME,
    @tablename SYSNAME,
    @rowguid UNIQUEIDENTIFIER,
    @subscriber SYSNAME,
    @subscriber_db SYSNAME,
    @log_conflict INT OUTPUT,
    @conflict_message NVARCHAR(512) OUTPUT
AS
BEGIN
    -- 비즈니스 규칙에 따른 충돌 해결
    DECLARE @Priority INT;
    
    -- 우선순위 결정 (예: 특정 서버가 우선)
    SELECT @Priority = 
        CASE 
            WHEN @subscriber = 'PrimaryServer' THEN 100
            WHEN @subscriber = 'SecondaryServer' THEN 50
            ELSE 10
        END;
    
    -- 우선순위가 높은 변경사항 선택
    IF @Priority >= 50
    BEGIN
        SET @log_conflict = 0;  -- 충돌 해결됨
        SET @conflict_message = 'Resolved by priority rule';
    END
    ELSE
    BEGIN
        SET @log_conflict = 1;  -- 충돌 로그
        SET @conflict_message = 'Lower priority change rejected';
    END;
END;
```

### 2.2 체인 복제와 캐스케이딩

```sql
-- 계층적 복제 토폴로지 설정
CREATE TABLE ReplicationTopology (
    NodeId INT PRIMARY KEY,
    NodeName NVARCHAR(128),
    NodeType VARCHAR(20), -- 'Publisher', 'Distributor', 'Subscriber'
    ParentNodeId INT,
    ReplicationPath NVARCHAR(500),
    Level INT,
    IsActive BIT DEFAULT 1,
    FOREIGN KEY (ParentNodeId) REFERENCES ReplicationTopology(NodeId)
);

-- 복제 체인 구성
INSERT INTO ReplicationTopology VALUES
(1, 'MasterServer', 'Publisher', NULL, '/1', 0, 1),
(2, 'RegionalHub1', 'Distributor', 1, '/1/2', 1, 1),
(3, 'RegionalHub2', 'Distributor', 1, '/1/3', 1, 1),
(4, 'LocalServer1', 'Subscriber', 2, '/1/2/4', 2, 1),
(5, 'LocalServer2', 'Subscriber', 2, '/1/2/5', 2, 1),
(6, 'LocalServer3', 'Subscriber', 3, '/1/3/6', 2, 1);

-- 캐스케이딩 복제 설정 프로시저
CREATE PROCEDURE sp_SetupCascadingReplication
    @SourceNode INT,
    @TargetNode INT
AS
BEGIN
    DECLARE @SourceServer NVARCHAR(128);
    DECLARE @TargetServer NVARCHAR(128);
    DECLARE @PublicationName NVARCHAR(128);
    
    -- 노드 정보 가져오기
    SELECT @SourceServer = NodeName
    FROM ReplicationTopology
    WHERE NodeId = @SourceNode;
    
    SELECT @TargetServer = NodeName
    FROM ReplicationTopology
    WHERE NodeId = @TargetNode;
    
    SET @PublicationName = 'Cascade_' + CAST(@SourceNode AS VARCHAR) + 
                          '_to_' + CAST(@TargetNode AS VARCHAR);
    
    -- 동적으로 복제 설정
    DECLARE @SQL NVARCHAR(MAX);
    
    -- 게시 생성
    SET @SQL = 'EXEC [' + @SourceServer + '].master.dbo.sp_addpublication 
        @publication = N''' + @PublicationName + ''',
        @description = N''Cascading replication'',
        @sync_method = N''concurrent'',
        @repl_freq = N''continuous'',
        @status = N''active''';
    
    EXEC sp_executesql @SQL;
    
    -- 구독 생성
    SET @SQL = 'EXEC [' + @SourceServer + '].master.dbo.sp_addsubscription 
        @publication = N''' + @PublicationName + ''',
        @subscriber = N''' + @TargetServer + ''',
        @destination_db = N''ReplicatedDB'',
        @subscription_type = N''Push''';
    
    EXEC sp_executesql @SQL;
END;
```

## 3. 샤딩 기본 개념

### 3.1 샤딩 전략

```sql
-- 범위 기반 샤딩
CREATE FUNCTION fn_GetShardByRange(@CustomerId INT)
RETURNS NVARCHAR(128)
AS
BEGIN
    RETURN CASE 
        WHEN @CustomerId BETWEEN 1 AND 1000000 THEN 'Shard1'
        WHEN @CustomerId BETWEEN 1000001 AND 2000000 THEN 'Shard2'
        WHEN @CustomerId BETWEEN 2000001 AND 3000000 THEN 'Shard3'
        WHEN @CustomerId BETWEEN 3000001 AND 4000000 THEN 'Shard4'
        ELSE 'Shard5'
    END;
END;

-- 해시 기반 샤딩
CREATE FUNCTION fn_GetShardByHash(@Key NVARCHAR(100))
RETURNS NVARCHAR(128)
AS
BEGIN
    DECLARE @Hash INT = ABS(CHECKSUM(@Key));
    DECLARE @ShardCount INT = 8;
    DECLARE @ShardIndex INT = @Hash % @ShardCount;
    
    RETURN 'Shard' + CAST(@ShardIndex + 1 AS VARCHAR);
END;

-- 지리적 샤딩
CREATE FUNCTION fn_GetShardByGeography(@Country NVARCHAR(50))
RETURNS NVARCHAR(128)
AS
BEGIN
    RETURN CASE 
        WHEN @Country IN ('USA', 'Canada', 'Mexico') THEN 'NorthAmericaShard'
        WHEN @Country IN ('UK', 'Germany', 'France', 'Italy') THEN 'EuropeShard'
        WHEN @Country IN ('China', 'Japan', 'Korea', 'India') THEN 'AsiaShard'
        WHEN @Country IN ('Brazil', 'Argentina', 'Chile') THEN 'SouthAmericaShard'
        ELSE 'GlobalShard'
    END;
END;

-- 샤드 매핑 테이블
CREATE TABLE ShardMap (
    ShardKey NVARCHAR(100) PRIMARY KEY,
    ShardName NVARCHAR(128) NOT NULL,
    ConnectionString NVARCHAR(500) NOT NULL,
    IsActive BIT DEFAULT 1,
    CreatedDate DATETIME DEFAULT GETDATE(),
    LastHealthCheck DATETIME,
    INDEX IX_ShardMap_Active (IsActive, ShardName)
);

-- 복합 샤딩 전략
CREATE FUNCTION fn_GetShardByComposite(
    @CustomerId INT,
    @Country NVARCHAR(50),
    @CreatedDate DATETIME
)
RETURNS NVARCHAR(128)
AS
BEGIN
    DECLARE @YearShard VARCHAR(4) = CAST(YEAR(@CreatedDate) AS VARCHAR);
    DECLARE @GeoShard VARCHAR(20) = 
        CASE 
            WHEN @Country IN ('USA', 'Canada') THEN 'NA'
            WHEN @Country IN ('UK', 'Germany', 'France') THEN 'EU'
            WHEN @Country IN ('China', 'Japan', 'Korea') THEN 'AS'
            ELSE 'GL'
        END;
    DECLARE @HashShard VARCHAR(2) = 
        RIGHT('0' + CAST((@CustomerId % 10) AS VARCHAR), 2);
    
    RETURN 'Shard_' + @GeoShard + '_' + @YearShard + '_' + @HashShard;
END;
```

### 3.2 샤드 라우팅

```sql
-- 샤드 라우터 구현
CREATE PROCEDURE sp_ExecuteOnShard
    @ShardKey NVARCHAR(100),
    @Query NVARCHAR(MAX),
    @Parameters NVARCHAR(MAX) = NULL
AS
BEGIN
    DECLARE @ShardName NVARCHAR(128);
    DECLARE @ConnectionString NVARCHAR(500);
    DECLARE @LinkedServer NVARCHAR(128);
    
    -- 샤드 결정
    SELECT @ShardName = ShardName,
           @ConnectionString = ConnectionString
    FROM ShardMap
    WHERE ShardKey = @ShardKey
    AND IsActive = 1;
    
    IF @ShardName IS NULL
    BEGIN
        -- 기본 샤드 할당
        SET @ShardName = dbo.fn_GetShardByHash(@ShardKey);
        
        -- 샤드 맵 업데이트
        INSERT INTO ShardMap (ShardKey, ShardName, ConnectionString)
        VALUES (@ShardKey, @ShardName, 
                'Server=' + @ShardName + ';Database=ShardDB;Trusted_Connection=yes;');
    END;
    
    -- 연결된 서버로 쿼리 실행
    SET @LinkedServer = '[' + @ShardName + ']';
    
    DECLARE @SQL NVARCHAR(MAX) = 
        'EXEC ' + @LinkedServer + '.ShardDB.dbo.sp_executesql @Query, @Parameters';
    
    EXEC sp_executesql @SQL, 
        N'@Query NVARCHAR(MAX), @Parameters NVARCHAR(MAX)',
        @Query = @Query,
        @Parameters = @Parameters;
END;

-- 분산 쿼리 실행기
CREATE PROCEDURE sp_ExecuteDistributedQuery
    @Query NVARCHAR(MAX),
    @ShardPattern NVARCHAR(100) = '%'
AS
BEGIN
    CREATE TABLE #Results (
        ShardName NVARCHAR(128),
        ResultSet NVARCHAR(MAX)
    );
    
    DECLARE @ShardName NVARCHAR(128);
    DECLARE @SQL NVARCHAR(MAX);
    
    DECLARE shard_cursor CURSOR FOR
        SELECT DISTINCT ShardName
        FROM ShardMap
        WHERE IsActive = 1
        AND ShardName LIKE @ShardPattern;
    
    OPEN shard_cursor;
    FETCH NEXT FROM shard_cursor INTO @ShardName;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        BEGIN TRY
            SET @SQL = 'SELECT @Result = (
                SELECT * FROM OPENQUERY([' + @ShardName + '], 
                ''' + REPLACE(@Query, '''', '''''') + ''') 
                FOR JSON AUTO
            )';
            
            DECLARE @Result NVARCHAR(MAX);
            EXEC sp_executesql @SQL, N'@Result NVARCHAR(MAX) OUTPUT', 
                @Result = @Result OUTPUT;
            
            INSERT INTO #Results VALUES (@ShardName, @Result);
        END TRY
        BEGIN CATCH
            INSERT INTO #Results VALUES (@ShardName, 
                'ERROR: ' + ERROR_MESSAGE());
        END CATCH;
        
        FETCH NEXT FROM shard_cursor INTO @ShardName;
    END;
    
    CLOSE shard_cursor;
    DEALLOCATE shard_cursor;
    
    SELECT * FROM #Results;
END;
```

## 4. 샤드 관리

### 4.1 샤드 재분배

```sql
-- 샤드 재분배 관리
CREATE TABLE ShardRebalanceLog (
    RebalanceId INT IDENTITY(1,1) PRIMARY KEY,
    SourceShard NVARCHAR(128),
    TargetShard NVARCHAR(128),
    KeyRangeStart NVARCHAR(100),
    KeyRangeEnd NVARCHAR(100),
    RecordCount INT,
    StartTime DATETIME,
    EndTime DATETIME,
    Status VARCHAR(20),
    ErrorMessage NVARCHAR(MAX)
);

-- 샤드 재분배 프로시저
CREATE PROCEDURE sp_RebalanceShards
    @SourceShard NVARCHAR(128),
    @TargetShard NVARCHAR(128),
    @KeyRangeStart NVARCHAR(100),
    @KeyRangeEnd NVARCHAR(100)
AS
BEGIN
    SET XACT_ABORT ON;
    
    DECLARE @RebalanceId INT;
    DECLARE @RecordCount INT;
    
    -- 재분배 시작 로그
    INSERT INTO ShardRebalanceLog 
        (SourceShard, TargetShard, KeyRangeStart, KeyRangeEnd, StartTime, Status)
    VALUES 
        (@SourceShard, @TargetShard, @KeyRangeStart, @KeyRangeEnd, GETDATE(), 'Started');
    
    SET @RebalanceId = SCOPE_IDENTITY();
    
    BEGIN TRY
        BEGIN DISTRIBUTED TRANSACTION;
            
            -- 1. 대상 샤드로 데이터 복사
            DECLARE @SQL NVARCHAR(MAX);
            SET @SQL = '
                INSERT INTO [' + @TargetShard + '].ShardDB.dbo.Customers
                SELECT * FROM [' + @SourceShard + '].ShardDB.dbo.Customers
                WHERE CustomerId BETWEEN @Start AND @End';
            
            EXEC sp_executesql @SQL,
                N'@Start NVARCHAR(100), @End NVARCHAR(100)',
                @Start = @KeyRangeStart,
                @End = @KeyRangeEnd;
            
            -- 레코드 수 확인
            SET @SQL = 'SELECT @Count = COUNT(*) 
                        FROM [' + @SourceShard + '].ShardDB.dbo.Customers
                        WHERE CustomerId BETWEEN @Start AND @End';
            
            EXEC sp_executesql @SQL,
                N'@Start NVARCHAR(100), @End NVARCHAR(100), @Count INT OUTPUT',
                @Start = @KeyRangeStart,
                @End = @KeyRangeEnd,
                @Count = @RecordCount OUTPUT;
            
            -- 2. 샤드 맵 업데이트
            UPDATE ShardMap
            SET ShardName = @TargetShard
            WHERE ShardKey BETWEEN @KeyRangeStart AND @KeyRangeEnd
            AND ShardName = @SourceShard;
            
            -- 3. 원본에서 데이터 삭제
            SET @SQL = '
                DELETE FROM [' + @SourceShard + '].ShardDB.dbo.Customers
                WHERE CustomerId BETWEEN @Start AND @End';
            
            EXEC sp_executesql @SQL,
                N'@Start NVARCHAR(100), @End NVARCHAR(100)',
                @Start = @KeyRangeStart,
                @End = @KeyRangeEnd;
            
        COMMIT;
        
        -- 성공 로그
        UPDATE ShardRebalanceLog
        SET EndTime = GETDATE(),
            Status = 'Completed',
            RecordCount = @RecordCount
        WHERE RebalanceId = @RebalanceId;
        
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK;
        
        -- 실패 로그
        UPDATE ShardRebalanceLog
        SET EndTime = GETDATE(),
            Status = 'Failed',
            ErrorMessage = ERROR_MESSAGE()
        WHERE RebalanceId = @RebalanceId;
        
        THROW;
    END CATCH;
END;

-- 자동 샤드 분할
CREATE PROCEDURE sp_AutoSplitShard
    @ShardName NVARCHAR(128),
    @Threshold INT = 10000000  -- 1천만 레코드
AS
BEGIN
    DECLARE @RecordCount INT;
    DECLARE @NewShardName NVARCHAR(128);
    
    -- 샤드 크기 확인
    DECLARE @SQL NVARCHAR(MAX);
    SET @SQL = 'SELECT @Count = COUNT(*) FROM [' + @ShardName + '].ShardDB.dbo.Customers';
    EXEC sp_executesql @SQL, N'@Count INT OUTPUT', @Count = @RecordCount OUTPUT;
    
    IF @RecordCount > @Threshold
    BEGIN
        -- 새 샤드 생성
        SET @NewShardName = @ShardName + '_Split_' + 
                           CONVERT(VARCHAR(8), GETDATE(), 112);
        
        -- 새 샤드 데이터베이스 생성
        EXEC sp_CreateNewShard @NewShardName;
        
        -- 데이터 절반 이동
        DECLARE @MidPoint INT = @RecordCount / 2;
        
        SET @SQL = '
            WITH NumberedRows AS (
                SELECT *, ROW_NUMBER() OVER (ORDER BY CustomerId) AS RowNum
                FROM [' + @ShardName + '].ShardDB.dbo.Customers
            )
            INSERT INTO [' + @NewShardName + '].ShardDB.dbo.Customers
            SELECT * FROM NumberedRows
            WHERE RowNum > @Mid';
        
        EXEC sp_executesql @SQL, N'@Mid INT', @Mid = @MidPoint;
        
        -- 샤드 맵 업데이트
        -- ... (샤드 키 재매핑 로직)
    END;
END;
```

### 4.2 크로스 샤드 쿼리

```sql
-- 크로스 샤드 조인
CREATE PROCEDURE sp_CrossShardJoin
    @CustomerId INT,
    @StartDate DATETIME,
    @EndDate DATETIME
AS
BEGIN
    -- 임시 테이블 생성
    CREATE TABLE #CustomerOrders (
        CustomerId INT,
        CustomerName NVARCHAR(100),
        OrderId INT,
        OrderDate DATETIME,
        TotalAmount DECIMAL(10, 2),
        ShardName NVARCHAR(128)
    );
    
    -- 고객 정보 가져오기
    DECLARE @CustomerShard NVARCHAR(128);
    SELECT @CustomerShard = dbo.fn_GetShardByHash(@CustomerId);
    
    DECLARE @SQL NVARCHAR(MAX);
    SET @SQL = '
        INSERT INTO #CustomerOrders (CustomerId, CustomerName, ShardName)
        SELECT CustomerId, CustomerName, @Shard
        FROM [' + @CustomerShard + '].ShardDB.dbo.Customers
        WHERE CustomerId = @CustId';
    
    EXEC sp_executesql @SQL,
        N'@CustId INT, @Shard NVARCHAR(128)',
        @CustId = @CustomerId,
        @Shard = @CustomerShard;
    
    -- 모든 샤드에서 주문 정보 수집
    DECLARE @ShardName NVARCHAR(128);
    DECLARE shard_cursor CURSOR FOR
        SELECT DISTINCT ShardName FROM ShardMap WHERE IsActive = 1;
    
    OPEN shard_cursor;
    FETCH NEXT FROM shard_cursor INTO @ShardName;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        SET @SQL = '
            UPDATE co
            SET co.OrderId = o.OrderId,
                co.OrderDate = o.OrderDate,
                co.TotalAmount = o.TotalAmount
            FROM #CustomerOrders co
            INNER JOIN [' + @ShardName + '].ShardDB.dbo.Orders o
                ON co.CustomerId = o.CustomerId
            WHERE o.OrderDate BETWEEN @Start AND @End';
        
        EXEC sp_executesql @SQL,
            N'@Start DATETIME, @End DATETIME',
            @Start = @StartDate,
            @End = @EndDate;
        
        FETCH NEXT FROM shard_cursor INTO @ShardName;
    END;
    
    CLOSE shard_cursor;
    DEALLOCATE shard_cursor;
    
    -- 결과 반환
    SELECT * FROM #CustomerOrders
    WHERE OrderId IS NOT NULL
    ORDER BY OrderDate DESC;
END;

-- 분산 집계 쿼리
CREATE PROCEDURE sp_DistributedAggregation
    @AggregateFunction NVARCHAR(50),  -- 'SUM', 'COUNT', 'AVG', etc.
    @Column NVARCHAR(128),
    @Table NVARCHAR(128),
    @WhereClause NVARCHAR(MAX) = ''
AS
BEGIN
    CREATE TABLE #ShardResults (
        ShardName NVARCHAR(128),
        AggregateValue DECIMAL(20, 4),
        RecordCount INT
    );
    
    DECLARE @ShardName NVARCHAR(128);
    DECLARE @SQL NVARCHAR(MAX);
    
    DECLARE shard_cursor CURSOR FOR
        SELECT DISTINCT ShardName FROM ShardMap WHERE IsActive = 1;
    
    OPEN shard_cursor;
    FETCH NEXT FROM shard_cursor INTO @ShardName;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        IF @AggregateFunction = 'AVG'
        BEGIN
            -- AVG는 특별 처리 필요 (SUM과 COUNT 수집)
            SET @SQL = '
                INSERT INTO #ShardResults (ShardName, AggregateValue, RecordCount)
                SELECT @Shard, SUM(' + @Column + '), COUNT(*)
                FROM [' + @ShardName + '].ShardDB.dbo.' + @Table;
        END
        ELSE
        BEGIN
            SET @SQL = '
                INSERT INTO #ShardResults (ShardName, AggregateValue, RecordCount)
                SELECT @Shard, ' + @AggregateFunction + '(' + @Column + '), COUNT(*)
                FROM [' + @ShardName + '].ShardDB.dbo.' + @Table;
        END;
        
        IF @WhereClause != ''
            SET @SQL = @SQL + ' WHERE ' + @WhereClause;
        
        EXEC sp_executesql @SQL, N'@Shard NVARCHAR(128)', @Shard = @ShardName;
        
        FETCH NEXT FROM shard_cursor INTO @ShardName;
    END;
    
    CLOSE shard_cursor;
    DEALLOCATE shard_cursor;
    
    -- 최종 집계
    IF @AggregateFunction = 'SUM'
        SELECT SUM(AggregateValue) AS Result FROM #ShardResults;
    ELSE IF @AggregateFunction = 'COUNT'
        SELECT SUM(RecordCount) AS Result FROM #ShardResults;
    ELSE IF @AggregateFunction = 'AVG'
        SELECT SUM(AggregateValue) / SUM(RecordCount) AS Result FROM #ShardResults;
    ELSE IF @AggregateFunction = 'MAX'
        SELECT MAX(AggregateValue) AS Result FROM #ShardResults;
    ELSE IF @AggregateFunction = 'MIN'
        SELECT MIN(AggregateValue) AS Result FROM #ShardResults;
END;
```

## 5. 복제와 샤딩 통합

### 5.1 샤드 복제

```sql
-- 샤드별 복제 구성
CREATE TABLE ShardReplication (
    ShardId INT IDENTITY(1,1) PRIMARY KEY,
    ShardName NVARCHAR(128),
    PrimaryServer NVARCHAR(128),
    ReplicaServers NVARCHAR(MAX),  -- JSON array
    ReplicationType VARCHAR(20),    -- 'Sync', 'Async', 'SemiSync'
    ReplicationLag INT,             -- seconds
    LastHealthCheck DATETIME,
    Status VARCHAR(20)
);

-- 샤드 복제 상태 모니터링
CREATE PROCEDURE sp_MonitorShardReplication
AS
BEGIN
    CREATE TABLE #ReplicationStatus (
        ShardName NVARCHAR(128),
        PrimaryServer NVARCHAR(128),
        ReplicaServer NVARCHAR(128),
        LagSeconds INT,
        Status VARCHAR(20)
    );
    
    DECLARE @ShardName NVARCHAR(128);
    DECLARE @PrimaryServer NVARCHAR(128);
    DECLARE @ReplicaServers NVARCHAR(MAX);
    
    DECLARE shard_cursor CURSOR FOR
        SELECT ShardName, PrimaryServer, ReplicaServers
        FROM ShardReplication
        WHERE Status = 'Active';
    
    OPEN shard_cursor;
    FETCH NEXT FROM shard_cursor INTO @ShardName, @PrimaryServer, @ReplicaServers;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        -- 각 복제본 상태 확인
        DECLARE @SQL NVARCHAR(MAX);
        SET @SQL = '
            SELECT 
                @Shard AS ShardName,
                @Primary AS PrimaryServer,
                secondary_replica AS ReplicaServer,
                DATEDIFF(SECOND, last_commit_time, GETDATE()) AS LagSeconds,
                synchronization_state_desc AS Status
            FROM [' + @PrimaryServer + '].master.sys.dm_hadr_database_replica_states
            WHERE database_id = DB_ID(''ShardDB'')';
        
        INSERT INTO #ReplicationStatus
        EXEC sp_executesql @SQL,
            N'@Shard NVARCHAR(128), @Primary NVARCHAR(128)',
            @Shard = @ShardName,
            @Primary = @PrimaryServer;
        
        FETCH NEXT FROM shard_cursor INTO @ShardName, @PrimaryServer, @ReplicaServers;
    END;
    
    CLOSE shard_cursor;
    DEALLOCATE shard_cursor;
    
    -- 결과 반환 및 알림
    SELECT * FROM #ReplicationStatus;
    
    -- 지연 임계값 초과 시 알림
    IF EXISTS (SELECT 1 FROM #ReplicationStatus WHERE LagSeconds > 60)
    BEGIN
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'DBA_Profile',
            @recipients = 'dba@company.com',
            @subject = 'Shard Replication Lag Alert',
            @body = 'One or more shard replicas are lagging behind.';
    END;
END;

-- 읽기 부하 분산
CREATE FUNCTION fn_GetReadReplica(@ShardName NVARCHAR(128))
RETURNS NVARCHAR(128)
AS
BEGIN
    DECLARE @ReplicaServer NVARCHAR(128);
    
    -- 라운드 로빈 또는 최소 부하 서버 선택
    SELECT TOP 1 @ReplicaServer = ReplicaServer
    FROM (
        SELECT 
            value AS ReplicaServer,
            ROW_NUMBER() OVER (ORDER BY NEWID()) AS RandomOrder
        FROM ShardReplication sr
        CROSS APPLY STRING_SPLIT(sr.ReplicaServers, ',')
        WHERE sr.ShardName = @ShardName
        AND sr.Status = 'Active'
    ) AS Replicas
    ORDER BY RandomOrder;
    
    RETURN ISNULL(@ReplicaServer, @ShardName);  -- 복제본이 없으면 주 샤드 반환
END;
```

### 5.2 글로벌 분산 아키텍처

```sql
-- 지역별 샤드 및 복제 구성
CREATE TABLE GlobalShardTopology (
    RegionId INT IDENTITY(1,1) PRIMARY KEY,
    RegionName NVARCHAR(50),
    PrimaryDatacenter NVARCHAR(100),
    SecondaryDatacenters NVARCHAR(MAX),  -- JSON
    ShardRange NVARCHAR(100),
    ReplicationMode VARCHAR(20),
    NetworkLatency INT,  -- milliseconds
    Timezone VARCHAR(50)
);

-- 지역 간 데이터 동기화
CREATE PROCEDURE sp_SyncCrossRegionData
    @SourceRegion NVARCHAR(50),
    @TargetRegion NVARCHAR(50),
    @DataType VARCHAR(50)  -- 'Full', 'Incremental', 'Changes'
AS
BEGIN
    DECLARE @SourceDC NVARCHAR(100);
    DECLARE @TargetDC NVARCHAR(100);
    
    SELECT @SourceDC = PrimaryDatacenter
    FROM GlobalShardTopology
    WHERE RegionName = @SourceRegion;
    
    SELECT @TargetDC = PrimaryDatacenter
    FROM GlobalShardTopology
    WHERE RegionName = @TargetRegion;
    
    IF @DataType = 'Changes'
    BEGIN
        -- CDC (Change Data Capture) 기반 동기화
        DECLARE @LastSyncLSN BINARY(10);
        
        SELECT @LastSyncLSN = LastSyncLSN
        FROM CrossRegionSyncStatus
        WHERE SourceRegion = @SourceRegion
        AND TargetRegion = @TargetRegion;
        
        -- 변경 데이터 추출 및 전송
        DECLARE @SQL NVARCHAR(MAX);
        SET @SQL = '
            INSERT INTO [' + @TargetDC + '].ShardDB.dbo.Customers
            SELECT *
            FROM [' + @SourceDC + '].ShardDB.cdc.fn_cdc_get_all_changes_dbo_Customers(
                @LastLSN, 
                sys.fn_cdc_get_max_lsn(), 
                ''all''
            )
            WHERE __$operation IN (2, 4)';  -- Insert, Update
        
        EXEC sp_executesql @SQL, N'@LastLSN BINARY(10)', @LastLSN = @LastSyncLSN;
        
        -- 동기화 상태 업데이트
        UPDATE CrossRegionSyncStatus
        SET LastSyncLSN = sys.fn_cdc_get_max_lsn(),
            LastSyncTime = GETDATE()
        WHERE SourceRegion = @SourceRegion
        AND TargetRegion = @TargetRegion;
    END;
END;

-- 글로벌 쿼리 라우터
CREATE PROCEDURE sp_ExecuteGlobalQuery
    @Query NVARCHAR(MAX),
    @PreferredRegion NVARCHAR(50) = NULL
AS
BEGIN
    DECLARE @TargetDC NVARCHAR(100);
    DECLARE @NetworkLatency INT;
    
    IF @PreferredRegion IS NOT NULL
    BEGIN
        -- 지정된 지역 사용
        SELECT @TargetDC = PrimaryDatacenter
        FROM GlobalShardTopology
        WHERE RegionName = @PreferredRegion;
    END
    ELSE
    BEGIN
        -- 가장 가까운 지역 선택 (네트워크 지연 기준)
        SELECT TOP 1 
            @TargetDC = PrimaryDatacenter,
            @NetworkLatency = NetworkLatency
        FROM GlobalShardTopology
        ORDER BY NetworkLatency;
    END;
    
    -- 쿼리 실행
    DECLARE @SQL NVARCHAR(MAX);
    SET @SQL = 'EXEC [' + @TargetDC + '].ShardDB.dbo.sp_executesql @Query';
    
    -- 실행 시간 측정
    DECLARE @StartTime DATETIME = GETDATE();
    
    EXEC sp_executesql @SQL, N'@Query NVARCHAR(MAX)', @Query = @Query;
    
    DECLARE @ExecutionTime INT = DATEDIFF(MILLISECOND, @StartTime, GETDATE());
    
    -- 성능 로그
    INSERT INTO GlobalQueryLog 
        (Query, TargetDatacenter, NetworkLatency, ExecutionTime, Timestamp)
    VALUES 
        (@Query, @TargetDC, @NetworkLatency, @ExecutionTime, GETDATE());
END;
```

## 6. 모니터링과 유지보수

### 6.1 통합 모니터링 대시보드

```sql
-- 복제 및 샤딩 상태 뷰
CREATE VIEW vw_ReplicationShardingStatus
AS
SELECT 
    'Replication' AS Component,
    p.publication AS Name,
    s.subscriber AS Target,
    da.cur_latency AS LatencySeconds,
    CASE 
        WHEN da.cur_latency > 300 THEN 'Critical'
        WHEN da.cur_latency > 60 THEN 'Warning'
        ELSE 'Normal'
    END AS Status,
    da.last_sync AS LastSync
FROM distribution.dbo.MSdistribution_agents da
INNER JOIN distribution.dbo.MSsubscriptions s 
    ON da.publisher_id = s.publisher_id
INNER JOIN distribution.dbo.MSpublications p 
    ON s.publication_id = p.publication_id

UNION ALL

SELECT 
    'Sharding' AS Component,
    ShardName AS Name,
    ConnectionString AS Target,
    DATEDIFF(SECOND, LastHealthCheck, GETDATE()) AS LatencySeconds,
    CASE 
        WHEN DATEDIFF(MINUTE, LastHealthCheck, GETDATE()) > 5 THEN 'Critical'
        WHEN DATEDIFF(MINUTE, LastHealthCheck, GETDATE()) > 2 THEN 'Warning'
        ELSE 'Normal'
    END AS Status,
    LastHealthCheck AS LastSync
FROM ShardMap
WHERE IsActive = 1;

-- 성능 메트릭 수집
CREATE PROCEDURE sp_CollectPerformanceMetrics
AS
BEGIN
    -- 샤드별 성능
    INSERT INTO PerformanceMetrics 
        (MetricType, MetricName, MetricValue, Timestamp)
    SELECT 
        'Shard' AS MetricType,
        ShardName AS MetricName,
        (
            SELECT COUNT(*) 
            FROM sys.dm_exec_requests 
            WHERE database_id = DB_ID('ShardDB')
        ) AS MetricValue,
        GETDATE() AS Timestamp
    FROM ShardMap
    WHERE IsActive = 1;
    
    -- 복제 성능
    INSERT INTO PerformanceMetrics 
        (MetricType, MetricName, MetricValue, Timestamp)
    SELECT 
        'Replication' AS MetricType,
        publication AS MetricName,
        AVG(cur_latency) AS MetricValue,
        GETDATE() AS Timestamp
    FROM distribution.dbo.MSdistribution_agents da
    INNER JOIN distribution.dbo.MSpublications p 
        ON da.publisher_database = p.publisher_db
    GROUP BY publication;
END;
```

### 6.2 자동 장애 복구

```sql
-- 샤드 장애 감지 및 복구
CREATE PROCEDURE sp_HandleShardFailure
    @FailedShard NVARCHAR(128)
AS
BEGIN
    DECLARE @BackupShard NVARCHAR(128);
    
    -- 백업 샤드 찾기
    SELECT @BackupShard = ReplicaServers
    FROM ShardReplication
    WHERE ShardName = @FailedShard
    AND Status = 'Active';
    
    IF @BackupShard IS NOT NULL
    BEGIN
        -- 1. 샤드 맵 업데이트
        UPDATE ShardMap
        SET ConnectionString = REPLACE(ConnectionString, @FailedShard, @BackupShard),
            ShardName = @BackupShard + '_Promoted'
        WHERE ShardName = @FailedShard;
        
        -- 2. 복제 토폴로지 업데이트
        UPDATE ShardReplication
        SET PrimaryServer = @BackupShard,
            Status = 'Failover'
        WHERE ShardName = @FailedShard;
        
        -- 3. 애플리케이션 알림
        EXEC sp_NotifyApplications 
            @Event = 'ShardFailover',
            @Details = @FailedShard + ' -> ' + @BackupShard;
        
        -- 4. 로그 기록
        INSERT INTO FailoverLog 
            (FailedComponent, BackupComponent, FailoverTime, Status)
        VALUES 
            (@FailedShard, @BackupShard, GETDATE(), 'Success');
    END
    ELSE
    BEGIN
        -- 백업이 없는 경우 긴급 알림
        EXEC sp_SendEmergencyAlert 
            @Message = 'Critical: No backup available for shard ' + @FailedShard;
    END;
END;

-- 자동 복구 스케줄러
CREATE PROCEDURE sp_AutoRecoveryScheduler
AS
BEGIN
    -- 장애 샤드 확인
    DECLARE @FailedShard NVARCHAR(128);
    
    DECLARE failed_cursor CURSOR FOR
        SELECT ShardName
        FROM ShardMap
        WHERE IsActive = 1
        AND DATEDIFF(MINUTE, LastHealthCheck, GETDATE()) > 5;
    
    OPEN failed_cursor;
    FETCH NEXT FROM failed_cursor INTO @FailedShard;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        EXEC sp_HandleShardFailure @FailedShard;
        
        FETCH NEXT FROM failed_cursor INTO @FailedShard;
    END;
    
    CLOSE failed_cursor;
    DEALLOCATE failed_cursor;
    
    -- 복제 지연 해결
    EXEC sp_ResolveReplicationLag;
    
    -- 샤드 재분배 필요성 확인
    EXEC sp_CheckShardBalance;
END;
```

## 마무리

복제와 샤딩은 대규모 데이터베이스 시스템의 확장성과 가용성을 보장하는 핵심 기술입니다. 적절한 복제 전략으로 읽기 성능과 가용성을 향상시키고, 효과적인 샤딩으로 수평적 확장을 달성할 수 있습니다. 두 기술을 통합하여 글로벌 규모의 분산 데이터베이스 시스템을 구축할 수 있습니다. 다음 장에서는 NoSQL 데이터베이스의 개념과 활용 방법을 학습하겠습니다.