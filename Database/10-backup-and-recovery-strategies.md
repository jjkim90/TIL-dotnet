# 백업과 복구 전략

## 개요

데이터는 기업의 가장 중요한 자산 중 하나입니다. 효과적인 백업과 복구 전략은 하드웨어 장애, 인적 오류, 자연재해, 사이버 공격 등으로부터 데이터를 보호하는 핵심입니다. 이 장에서는 다양한 백업 유형, 복구 모델, 자동화된 백업 전략, 그리고 재해 복구 계획을 학습합니다.

## 1. 백업 기본 개념

### 1.1 백업 유형과 복구 모델

```sql
-- 데이터베이스 복구 모델 확인
SELECT 
    name AS DatabaseName,
    recovery_model_desc AS RecoveryModel,
    log_reuse_wait_desc AS LogReuseWait,
    is_auto_shrink_on AS AutoShrink,
    is_auto_close_on AS AutoClose,
    page_verify_option_desc AS PageVerify
FROM sys.databases
WHERE database_id > 4;  -- 시스템 DB 제외

-- 복구 모델 변경
ALTER DATABASE YourDatabase SET RECOVERY FULL;
-- ALTER DATABASE YourDatabase SET RECOVERY BULK_LOGGED;
-- ALTER DATABASE YourDatabase SET RECOVERY SIMPLE;

-- 백업 히스토리 확인
SELECT 
    bs.database_name,
    bs.backup_start_date,
    bs.backup_finish_date,
    bs.type AS BackupType,
    CASE bs.type
        WHEN 'D' THEN 'Full'
        WHEN 'I' THEN 'Differential'
        WHEN 'L' THEN 'Log'
        WHEN 'F' THEN 'File/Filegroup'
        WHEN 'G' THEN 'Differential File'
        WHEN 'P' THEN 'Partial'
        WHEN 'Q' THEN 'Differential Partial'
    END AS BackupTypeDesc,
    bs.backup_size / 1024 / 1024 AS BackupSizeMB,
    bs.compressed_backup_size / 1024 / 1024 AS CompressedSizeMB,
    DATEDIFF(SECOND, bs.backup_start_date, bs.backup_finish_date) AS DurationSeconds,
    bmf.physical_device_name
FROM msdb.dbo.backupset bs
INNER JOIN msdb.dbo.backupmediafamily bmf 
    ON bs.media_set_id = bmf.media_set_id
WHERE bs.database_name = 'YourDatabase'
ORDER BY bs.backup_start_date DESC;
```

### 1.2 전체 백업 (Full Backup)

```sql
-- 기본 전체 백업
BACKUP DATABASE YourDatabase
TO DISK = 'C:\Backup\YourDatabase_Full_20240315.bak'
WITH FORMAT,  -- 미디어 덮어쓰기
     INIT,    -- 백업 세트 덮어쓰기
     NAME = 'YourDatabase Full Backup',
     SKIP,    -- 만료 확인 건너뛰기
     STATS = 10;  -- 진행률 표시 (10% 단위)

-- 압축 백업 (Enterprise Edition)
BACKUP DATABASE YourDatabase
TO DISK = 'C:\Backup\YourDatabase_Full_Compressed.bak'
WITH COMPRESSION,
     CHECKSUM,  -- 백업 체크섬 생성
     STATS = 5;

-- 여러 파일로 분할 백업 (병렬 처리)
BACKUP DATABASE YourDatabase
TO DISK = 'C:\Backup\YourDatabase_Full_1.bak',
   DISK = 'D:\Backup\YourDatabase_Full_2.bak',
   DISK = 'E:\Backup\YourDatabase_Full_3.bak',
   DISK = 'F:\Backup\YourDatabase_Full_4.bak'
WITH FORMAT, INIT, 
     MEDIANAME = 'YourDatabase_Striped',
     MEDIADESCRIPTION = 'Striped backup across 4 disks';

-- 백업 검증
RESTORE VERIFYONLY
FROM DISK = 'C:\Backup\YourDatabase_Full_20240315.bak'
WITH CHECKSUM;

-- 백업 정보 조회
RESTORE HEADERONLY
FROM DISK = 'C:\Backup\YourDatabase_Full_20240315.bak';

RESTORE FILELISTONLY
FROM DISK = 'C:\Backup\YourDatabase_Full_20240315.bak';
```

### 1.3 차등 백업 (Differential Backup)

```sql
-- 차등 백업 (마지막 전체 백업 이후 변경사항만)
BACKUP DATABASE YourDatabase
TO DISK = 'C:\Backup\YourDatabase_Diff_20240315_1200.bak'
WITH DIFFERENTIAL,
     FORMAT, INIT,
     NAME = 'YourDatabase Differential Backup',
     STATS = 10;

-- 차등 백업 체인 확인
SELECT 
    database_name,
    backup_start_date,
    type,
    is_copy_only,
    first_lsn,
    last_lsn,
    differential_base_lsn,
    differential_base_guid
FROM msdb.dbo.backupset
WHERE database_name = 'YourDatabase'
AND type IN ('D', 'I')  -- Full and Differential
ORDER BY backup_start_date DESC;

-- 파일그룹별 차등 백업
BACKUP DATABASE YourDatabase
FILEGROUP = 'PRIMARY'
TO DISK = 'C:\Backup\YourDatabase_FG_Primary_Diff.bak'
WITH DIFFERENTIAL, 
     FORMAT, INIT;
```

### 1.4 트랜잭션 로그 백업

```sql
-- 트랜잭션 로그 백업 (FULL 또는 BULK_LOGGED 복구 모델 필요)
BACKUP LOG YourDatabase
TO DISK = 'C:\Backup\YourDatabase_Log_20240315_1300.trn'
WITH FORMAT, INIT,
     NAME = 'YourDatabase Log Backup',
     STATS = 10;

-- 로그 체인 확인
SELECT 
    database_name,
    backup_start_date,
    backup_finish_date,
    type,
    first_lsn,
    last_lsn,
    checkpoint_lsn,
    database_backup_lsn
FROM msdb.dbo.backupset
WHERE database_name = 'YourDatabase'
AND type = 'L'  -- Log backups
ORDER BY backup_start_date DESC;

-- 로그 백업 with 마크
BEGIN TRANSACTION ImportantUpdate WITH MARK 'Before Critical Update';
    -- 중요한 업데이트 수행
    UPDATE CriticalTable SET Value = NewValue WHERE Condition = True;
COMMIT;

BACKUP LOG YourDatabase
TO DISK = 'C:\Backup\YourDatabase_Log_AfterUpdate.trn'
WITH FORMAT, INIT;

-- Tail-Log 백업 (재해 발생 시)
BACKUP LOG YourDatabase
TO DISK = 'C:\Backup\YourDatabase_TailLog.trn'
WITH NO_TRUNCATE,  -- 데이터베이스가 손상된 경우
     INIT,
     NORECOVERY;   -- 데이터베이스를 복원 대기 상태로
```

## 2. 백업 전략 설계

### 2.1 백업 일정 자동화

```sql
-- 백업 유지보수 계획 프로시저
CREATE PROCEDURE sp_BackupMaintenancePlan
    @DatabaseName NVARCHAR(128),
    @BackupPath NVARCHAR(500),
    @RetentionDays INT = 7
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @FileName NVARCHAR(500);
    DECLARE @BackupName NVARCHAR(200);
    DECLARE @DateString NVARCHAR(20);
    
    -- 날짜 문자열 생성
    SET @DateString = CONVERT(NVARCHAR(20), GETDATE(), 112) + '_' + 
                      REPLACE(CONVERT(NVARCHAR(20), GETDATE(), 108), ':', '');
    
    -- 1. 전체 백업 (일요일)
    IF DATEPART(WEEKDAY, GETDATE()) = 1
    BEGIN
        SET @FileName = @BackupPath + '\' + @DatabaseName + '_Full_' + @DateString + '.bak';
        SET @BackupName = @DatabaseName + ' Full Backup ' + CONVERT(NVARCHAR(20), GETDATE(), 120);
        
        BACKUP DATABASE @DatabaseName
        TO DISK = @FileName
        WITH FORMAT, INIT, 
             NAME = @BackupName,
             COMPRESSION,
             CHECKSUM,
             STATS = 10;
             
        -- 백업 검증
        RESTORE VERIFYONLY FROM DISK = @FileName WITH CHECKSUM;
    END
    -- 2. 차등 백업 (화-토)
    ELSE IF DATEPART(WEEKDAY, GETDATE()) BETWEEN 2 AND 7
    BEGIN
        SET @FileName = @BackupPath + '\' + @DatabaseName + '_Diff_' + @DateString + '.bak';
        SET @BackupName = @DatabaseName + ' Differential Backup ' + CONVERT(NVARCHAR(20), GETDATE(), 120);
        
        BACKUP DATABASE @DatabaseName
        TO DISK = @FileName
        WITH DIFFERENTIAL,
             FORMAT, INIT,
             NAME = @BackupName,
             COMPRESSION,
             CHECKSUM,
             STATS = 10;
    END
    
    -- 3. 로그 백업 (매시간 - FULL 복구 모델인 경우)
    IF (SELECT recovery_model FROM sys.databases WHERE name = @DatabaseName) = 1
    BEGIN
        SET @FileName = @BackupPath + '\' + @DatabaseName + '_Log_' + @DateString + '.trn';
        SET @BackupName = @DatabaseName + ' Log Backup ' + CONVERT(NVARCHAR(20), GETDATE(), 120);
        
        BACKUP LOG @DatabaseName
        TO DISK = @FileName
        WITH FORMAT, INIT,
             NAME = @BackupName,
             COMPRESSION,
             CHECKSUM;
    END
    
    -- 4. 오래된 백업 파일 정리
    EXEC sp_CleanupOldBackups @BackupPath, @RetentionDays;
END;

-- SQL Agent 작업으로 스케줄링
USE msdb;
GO

-- 전체 백업 작업 (매주 일요일 오전 2시)
EXEC sp_add_job
    @job_name = 'Weekly Full Backup',
    @enabled = 1;

EXEC sp_add_jobstep
    @job_name = 'Weekly Full Backup',
    @step_name = 'Full Backup',
    @command = 'EXEC sp_BackupMaintenancePlan ''YourDatabase'', ''C:\Backup'', 30',
    @database_name = 'master';

EXEC sp_add_schedule
    @schedule_name = 'Weekly Sunday 2AM',
    @freq_type = 8,  -- Weekly
    @freq_interval = 1,  -- Sunday
    @freq_recurrence_factor = 1,
    @active_start_time = 020000;  -- 2:00 AM

EXEC sp_attach_schedule
    @job_name = 'Weekly Full Backup',
    @schedule_name = 'Weekly Sunday 2AM';
```

### 2.2 백업 파일 관리

```sql
-- 백업 파일 정리 프로시저
CREATE PROCEDURE sp_CleanupOldBackups
    @BackupPath NVARCHAR(500),
    @RetentionDays INT
AS
BEGIN
    DECLARE @Command NVARCHAR(4000);
    DECLARE @CutoffDate DATETIME = DATEADD(DAY, -@RetentionDays, GETDATE());
    
    -- xp_cmdshell 임시 활성화
    EXEC sp_configure 'show advanced options', 1;
    RECONFIGURE;
    EXEC sp_configure 'xp_cmdshell', 1;
    RECONFIGURE;
    
    -- PowerShell 스크립트로 오래된 파일 삭제
    SET @Command = 'powershell.exe -Command "Get-ChildItem -Path ''' + @BackupPath + 
                   ''' -Filter *.bak,*.trn | Where-Object {$_.LastWriteTime -lt ''' + 
                   CONVERT(NVARCHAR(20), @CutoffDate, 120) + '''} | Remove-Item -Force"';
    
    EXEC xp_cmdshell @Command, no_output;
    
    -- xp_cmdshell 비활성화
    EXEC sp_configure 'xp_cmdshell', 0;
    RECONFIGURE;
    EXEC sp_configure 'show advanced options', 0;
    RECONFIGURE;
    
    -- 백업 히스토리 정리
    EXEC msdb.dbo.sp_delete_backuphistory @oldest_date = @CutoffDate;
END;

-- 백업 크기 추세 분석
WITH BackupTrend AS (
    SELECT 
        database_name,
        CAST(backup_start_date AS DATE) AS BackupDate,
        type,
        SUM(backup_size) / 1024 / 1024 / 1024 AS TotalSizeGB,
        COUNT(*) AS BackupCount
    FROM msdb.dbo.backupset
    WHERE backup_start_date >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY database_name, CAST(backup_start_date AS DATE), type
)
SELECT 
    database_name,
    BackupDate,
    [D] AS FullBackupGB,
    [I] AS DiffBackupGB,
    [L] AS LogBackupGB
FROM BackupTrend
PIVOT (
    SUM(TotalSizeGB)
    FOR type IN ([D], [I], [L])
) AS BackupPivot
ORDER BY database_name, BackupDate;
```

## 3. 복구 시나리오

### 3.1 전체 복구

```sql
-- 1. 단순 복구 (전체 백업만)
RESTORE DATABASE YourDatabase
FROM DISK = 'C:\Backup\YourDatabase_Full_20240315.bak'
WITH REPLACE,  -- 기존 데이터베이스 덮어쓰기
     RECOVERY,  -- 복구 완료
     STATS = 10;

-- 2. 차등 백업 포함 복구
-- Step 1: 전체 백업 복원 (NORECOVERY)
RESTORE DATABASE YourDatabase
FROM DISK = 'C:\Backup\YourDatabase_Full_20240310.bak'
WITH NORECOVERY,  -- 추가 백업 대기
     REPLACE,
     STATS = 10;

-- Step 2: 차등 백업 복원
RESTORE DATABASE YourDatabase
FROM DISK = 'C:\Backup\YourDatabase_Diff_20240314.bak'
WITH RECOVERY,  -- 복구 완료
     STATS = 10;

-- 3. 로그 백업 포함 전체 복구
-- Step 1: 전체 백업
RESTORE DATABASE YourDatabase
FROM DISK = 'C:\Backup\YourDatabase_Full_20240310.bak'
WITH NORECOVERY, REPLACE;

-- Step 2: 차등 백업 (있는 경우)
RESTORE DATABASE YourDatabase
FROM DISK = 'C:\Backup\YourDatabase_Diff_20240314.bak'
WITH NORECOVERY;

-- Step 3: 로그 백업들 순차 적용
RESTORE LOG YourDatabase
FROM DISK = 'C:\Backup\YourDatabase_Log_20240314_1200.trn'
WITH NORECOVERY;

RESTORE LOG YourDatabase
FROM DISK = 'C:\Backup\YourDatabase_Log_20240314_1300.trn'
WITH NORECOVERY;

-- Step 4: 마지막 로그 백업과 함께 복구 완료
RESTORE LOG YourDatabase
FROM DISK = 'C:\Backup\YourDatabase_Log_20240314_1400.trn'
WITH RECOVERY;
```

### 3.2 시점 복구 (Point-in-Time Recovery)

```sql
-- 특정 시점으로 복구
DECLARE @RecoveryPoint DATETIME = '2024-03-14 13:45:00';

-- 전체 백업 복원
RESTORE DATABASE YourDatabase
FROM DISK = 'C:\Backup\YourDatabase_Full_20240310.bak'
WITH NORECOVERY, REPLACE;

-- 로그 백업 적용 (시점 이전까지)
RESTORE LOG YourDatabase
FROM DISK = 'C:\Backup\YourDatabase_Log_20240314_1200.trn'
WITH NORECOVERY;

-- 목표 시점이 포함된 로그 백업에서 특정 시점까지만 복원
RESTORE LOG YourDatabase
FROM DISK = 'C:\Backup\YourDatabase_Log_20240314_1400.trn'
WITH RECOVERY,
     STOPAT = @RecoveryPoint;

-- 마크된 트랜잭션으로 복구
RESTORE LOG YourDatabase
FROM DISK = 'C:\Backup\YourDatabase_Log_20240314.trn'
WITH RECOVERY,
     STOPATMARK = 'BeforeCriticalUpdate';

-- LSN 기반 복구
RESTORE LOG YourDatabase
FROM DISK = 'C:\Backup\YourDatabase_Log_20240314.trn'
WITH RECOVERY,
     STOPBEFOREMARK = 'lsn:123000000456700001';
```

### 3.3 페이지 수준 복구

```sql
-- 손상된 페이지 확인
SELECT * FROM msdb.dbo.suspect_pages;

-- DBCC CHECKDB로 손상 확인
DBCC CHECKDB('YourDatabase') WITH NO_INFOMSGS;

-- 페이지 수준 복원
-- Step 1: 손상된 페이지 복원
RESTORE DATABASE YourDatabase
PAGE = '1:153, 1:202, 1:545'  -- FileID:PageID
FROM DISK = 'C:\Backup\YourDatabase_Full_20240310.bak'
WITH NORECOVERY;

-- Step 2: 차등 백업 적용 (있는 경우)
RESTORE DATABASE YourDatabase
FROM DISK = 'C:\Backup\YourDatabase_Diff_20240314.bak'
WITH NORECOVERY;

-- Step 3: 로그 백업 적용
RESTORE LOG YourDatabase
FROM DISK = 'C:\Backup\YourDatabase_Log_20240314_1400.trn'
WITH NORECOVERY;

-- Step 4: Tail-log 백업 및 적용
BACKUP LOG YourDatabase
TO DISK = 'C:\Backup\YourDatabase_TailLog_Page.trn'
WITH NORECOVERY;

RESTORE LOG YourDatabase
FROM DISK = 'C:\Backup\YourDatabase_TailLog_Page.trn'
WITH RECOVERY;
```

## 4. 고가용성 백업 전략

### 4.1 Always On 가용성 그룹에서의 백업

```sql
-- 백업 우선순위 설정
ALTER AVAILABILITY GROUP [AG_Name]
MODIFY REPLICA ON 'Server1' WITH (BACKUP_PRIORITY = 50);

ALTER AVAILABILITY GROUP [AG_Name]
MODIFY REPLICA ON 'Server2' WITH (BACKUP_PRIORITY = 40);

-- 백업 기본 설정
ALTER AVAILABILITY GROUP [AG_Name]
SET (AUTOMATED_BACKUP_PREFERENCE = SECONDARY);

-- 백업 위치 결정 함수
IF (sys.fn_hadr_backup_is_preferred_replica(@DatabaseName) = 1)
BEGIN
    -- 이 복제본에서 백업 수행
    BACKUP DATABASE @DatabaseName
    TO DISK = @BackupPath
    WITH COMPRESSION, CHECKSUM;
END;

-- 가용성 그룹 백업 정보
SELECT 
    ag.name AS AvailabilityGroup,
    ar.replica_server_name,
    ar.backup_priority,
    ar.secondary_role_allow_connections_desc,
    ars.role_desc,
    ars.operational_state_desc,
    ars.synchronization_health_desc
FROM sys.availability_groups ag
INNER JOIN sys.availability_replicas ar ON ag.group_id = ar.group_id
INNER JOIN sys.dm_hadr_availability_replica_states ars ON ar.replica_id = ars.replica_id
ORDER BY ag.name, ar.backup_priority DESC;
```

### 4.2 미러링과 로그 전달

```sql
-- 로그 전달 구성 모니터링
-- 주 서버
SELECT 
    primary_id,
    primary_server,
    primary_database,
    backup_threshold,
    threshold_alert_enabled,
    last_backup_file,
    last_backup_date,
    history_retention_period
FROM msdb.dbo.log_shipping_monitor_primary;

-- 보조 서버
SELECT 
    secondary_id,
    secondary_server,
    secondary_database,
    restore_threshold,
    threshold_alert_enabled,
    last_copied_file,
    last_copied_date,
    last_restored_file,
    last_restored_date,
    restore_delay
FROM msdb.dbo.log_shipping_monitor_secondary;

-- 로그 전달 작업 상태
EXEC sp_help_log_shipping_monitor;

-- 미러링 상태 확인
SELECT 
    DB_NAME(database_id) AS DatabaseName,
    mirroring_state_desc,
    mirroring_role_desc,
    mirroring_safety_level_desc,
    mirroring_witness_state_desc,
    mirroring_failover_lsn
FROM sys.database_mirroring
WHERE mirroring_guid IS NOT NULL;
```

## 5. 재해 복구 계획

### 5.1 RTO/RPO 기반 전략

```sql
-- RTO/RPO 측정 프로시저
CREATE PROCEDURE sp_MeasureRTORPO
    @DatabaseName NVARCHAR(128)
AS
BEGIN
    -- 최근 백업 정보
    WITH BackupInfo AS (
        SELECT 
            backup_start_date,
            backup_finish_date,
            type,
            backup_size / 1024.0 / 1024.0 / 1024.0 AS BackupSizeGB,
            compressed_backup_size / 1024.0 / 1024.0 / 1024.0 AS CompressedSizeGB,
            DATEDIFF(SECOND, backup_start_date, backup_finish_date) AS BackupDurationSec
        FROM msdb.dbo.backupset
        WHERE database_name = @DatabaseName
        AND backup_start_date >= DATEADD(DAY, -7, GETDATE())
    )
    SELECT 
        'Backup Metrics' AS Metric,
        AVG(BackupDurationSec) AS AvgBackupTimeSec,
        MAX(BackupDurationSec) AS MaxBackupTimeSec,
        AVG(BackupSizeGB) AS AvgBackupSizeGB,
        AVG(CompressedSizeGB) AS AvgCompressedSizeGB
    FROM BackupInfo
    WHERE type = 'D'
    
    UNION ALL
    
    -- 복원 시간 예측
    SELECT 
        'Restore Estimate' AS Metric,
        AVG(BackupSizeGB) * 60 AS EstimatedRestoreTimeSec,  -- 1GB당 60초 가정
        NULL AS MaxBackupTimeSec,
        AVG(BackupSizeGB) AS AvgBackupSizeGB,
        NULL AS AvgCompressedSizeGB
    FROM BackupInfo
    WHERE type = 'D'
    
    UNION ALL
    
    -- RPO (최대 데이터 손실)
    SELECT 
        'RPO (Data Loss)' AS Metric,
        DATEDIFF(MINUTE, MAX(backup_start_date), GETDATE()) AS CurrentRPOMinutes,
        NULL,
        NULL,
        NULL
    FROM BackupInfo
    WHERE type IN ('D', 'I', 'L');
END;

-- 재해 복구 테스트 자동화
CREATE PROCEDURE sp_DisasterRecoveryTest
    @SourceDatabase NVARCHAR(128),
    @TestDatabase NVARCHAR(128),
    @BackupPath NVARCHAR(500)
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @StartTime DATETIME = GETDATE();
    DECLARE @RestoreTime DATETIME;
    DECLARE @ValidationTime DATETIME;
    
    BEGIN TRY
        -- 1. 최신 백업 파일 찾기
        DECLARE @FullBackup NVARCHAR(500);
        DECLARE @DiffBackup NVARCHAR(500);
        
        SELECT TOP 1 @FullBackup = bmf.physical_device_name
        FROM msdb.dbo.backupset bs
        INNER JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
        WHERE bs.database_name = @SourceDatabase
        AND bs.type = 'D'
        ORDER BY bs.backup_start_date DESC;
        
        -- 2. 테스트 데이터베이스 복원
        RESTORE DATABASE @TestDatabase
        FROM DISK = @FullBackup
        WITH REPLACE,
             MOVE 'YourDatabase' TO 'C:\TestRestore\TestDB.mdf',
             MOVE 'YourDatabase_log' TO 'C:\TestRestore\TestDB_log.ldf',
             RECOVERY,
             STATS = 10;
             
        SET @RestoreTime = GETDATE();
        
        -- 3. 무결성 검사
        DBCC CHECKDB(@TestDatabase) WITH NO_INFOMSGS;
        
        SET @ValidationTime = GETDATE();
        
        -- 4. 결과 기록
        INSERT INTO DisasterRecoveryTests 
            (TestDate, SourceDB, TestDB, RestoreDuration, ValidationDuration, Result)
        VALUES 
            (@StartTime, @SourceDatabase, @TestDatabase,
             DATEDIFF(SECOND, @StartTime, @RestoreTime),
             DATEDIFF(SECOND, @RestoreTime, @ValidationTime),
             'Success');
             
        -- 5. 테스트 데이터베이스 정리
        DROP DATABASE @TestDatabase;
        
    END TRY
    BEGIN CATCH
        -- 오류 기록
        INSERT INTO DisasterRecoveryTests 
            (TestDate, SourceDB, TestDB, Result, ErrorMessage)
        VALUES 
            (@StartTime, @SourceDatabase, @TestDatabase,
             'Failed', ERROR_MESSAGE());
             
        -- 정리
        IF EXISTS (SELECT 1 FROM sys.databases WHERE name = @TestDatabase)
            DROP DATABASE @TestDatabase;
            
        THROW;
    END CATCH;
END;
```

### 5.2 자동화된 재해 복구

```sql
-- 자동 페일오버 스크립트
CREATE PROCEDURE sp_AutomatedFailover
    @PrimaryServer NVARCHAR(128),
    @SecondaryServer NVARCHAR(128),
    @DatabaseName NVARCHAR(128),
    @AGName NVARCHAR(128) = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Always On 가용성 그룹인 경우
    IF @AGName IS NOT NULL
    BEGIN
        -- 강제 페일오버
        ALTER AVAILABILITY GROUP @AGName FORCE_FAILOVER_ALLOW_DATA_LOSS;
        
        -- 새 주 복제본에서 실행
        ALTER AVAILABILITY GROUP @AGName SET (ROLE = PRIMARY);
    END
    -- 데이터베이스 미러링인 경우
    ELSE
    BEGIN
        -- 강제 서비스 시작
        ALTER DATABASE @DatabaseName SET PARTNER FORCE_SERVICE_ALLOW_DATA_LOSS;
    END
    
    -- 페일오버 후 작업
    -- 1. 애플리케이션 연결 문자열 업데이트
    EXEC sp_UpdateConnectionStrings @NewPrimary = @SecondaryServer;
    
    -- 2. DNS 업데이트
    EXEC xp_cmdshell 'powershell.exe -Command "Update-DnsRecord..."';
    
    -- 3. 알림 발송
    EXEC msdb.dbo.sp_send_dbmail
        @profile_name = 'DBA_Profile',
        @recipients = 'dba-team@company.com',
        @subject = 'Failover Completed',
        @body = 'Automatic failover has been completed successfully.';
END;

-- 복구 자동화 마스터 프로시저
CREATE PROCEDURE sp_DisasterRecoveryOrchestrator
    @RecoveryType NVARCHAR(20),  -- 'Full', 'PointInTime', 'PageLevel'
    @TargetDatabase NVARCHAR(128),
    @RecoveryPoint DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @SQL NVARCHAR(MAX);
    DECLARE @BackupFiles TABLE (
        BackupFile NVARCHAR(500),
        BackupType CHAR(1),
        BackupStartDate DATETIME,
        FirstLSN NUMERIC(25,0),
        LastLSN NUMERIC(25,0)
    );
    
    -- 1. 필요한 백업 파일 식별
    INSERT INTO @BackupFiles
    SELECT 
        bmf.physical_device_name,
        bs.type,
        bs.backup_start_date,
        bs.first_lsn,
        bs.last_lsn
    FROM msdb.dbo.backupset bs
    INNER JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
    WHERE bs.database_name = @TargetDatabase
    AND (@RecoveryPoint IS NULL OR bs.backup_start_date <= @RecoveryPoint)
    ORDER BY bs.backup_start_date;
    
    -- 2. 복구 스크립트 생성
    SET @SQL = '-- Disaster Recovery Script for ' + @TargetDatabase + CHAR(13) + CHAR(10);
    SET @SQL += '-- Generated: ' + CONVERT(NVARCHAR(30), GETDATE(), 120) + CHAR(13) + CHAR(10);
    SET @SQL += '-- Recovery Type: ' + @RecoveryType + CHAR(13) + CHAR(10);
    
    -- 전체 백업
    SELECT TOP 1 @SQL += 'RESTORE DATABASE ' + @TargetDatabase + 
                        ' FROM DISK = ''' + BackupFile + '''' +
                        ' WITH NORECOVERY, REPLACE;' + CHAR(13) + CHAR(10)
    FROM @BackupFiles
    WHERE BackupType = 'D'
    ORDER BY BackupStartDate DESC;
    
    -- 차등 백업
    SELECT TOP 1 @SQL += 'RESTORE DATABASE ' + @TargetDatabase + 
                        ' FROM DISK = ''' + BackupFile + '''' +
                        ' WITH NORECOVERY;' + CHAR(13) + CHAR(10)
    FROM @BackupFiles
    WHERE BackupType = 'I'
    ORDER BY BackupStartDate DESC;
    
    -- 로그 백업
    SELECT @SQL += 'RESTORE LOG ' + @TargetDatabase + 
                   ' FROM DISK = ''' + BackupFile + '''' +
                   ' WITH NORECOVERY;' + CHAR(13) + CHAR(10)
    FROM @BackupFiles
    WHERE BackupType = 'L'
    ORDER BY BackupStartDate;
    
    -- 최종 복구
    IF @RecoveryType = 'PointInTime'
        SET @SQL += 'RESTORE LOG ' + @TargetDatabase + 
                    ' WITH RECOVERY, STOPAT = ''' + 
                    CONVERT(NVARCHAR(30), @RecoveryPoint, 120) + ''';';
    ELSE
        SET @SQL += 'RESTORE DATABASE ' + @TargetDatabase + ' WITH RECOVERY;';
    
    -- 3. 스크립트 실행 또는 반환
    PRINT @SQL;
    -- EXEC sp_executesql @SQL;
END;
```

## 6. 클라우드 백업 전략

### 6.1 Azure Blob Storage 백업

```sql
-- Azure 자격 증명 생성
CREATE CREDENTIAL [https://yourstorageaccount.blob.core.windows.net/backups]
WITH IDENTITY = 'SHARED ACCESS SIGNATURE',
SECRET = 'sv=2021-06-08&ss=b&srt=co&sp=rwdlacyx&se=2025-12-31T23:59:59Z&...';

-- Azure Blob으로 백업
BACKUP DATABASE YourDatabase
TO URL = 'https://yourstorageaccount.blob.core.windows.net/backups/YourDatabase_Full.bak'
WITH COMPRESSION, CHECKSUM, FORMAT, INIT,
     MAXTRANSFERSIZE = 4194304,
     BLOCKSIZE = 65536,
     BACKUP_OPTIONS = 'ENABLE_COMPRESSION';

-- 스트라이프 백업 (성능 향상)
BACKUP DATABASE YourDatabase
TO URL = 'https://yourstorageaccount.blob.core.windows.net/backups/YourDatabase_1.bak',
   URL = 'https://yourstorageaccount.blob.core.windows.net/backups/YourDatabase_2.bak',
   URL = 'https://yourstorageaccount.blob.core.windows.net/backups/YourDatabase_3.bak',
   URL = 'https://yourstorageaccount.blob.core.windows.net/backups/YourDatabase_4.bak'
WITH COMPRESSION, FORMAT, MAXTRANSFERSIZE = 4194304;

-- 자동화된 클라우드 백업
CREATE PROCEDURE sp_CloudBackup
    @DatabaseName NVARCHAR(128),
    @StorageURL NVARCHAR(500),
    @RetentionDays INT = 30
AS
BEGIN
    DECLARE @BackupURL NVARCHAR(500);
    DECLARE @Timestamp NVARCHAR(20);
    
    SET @Timestamp = CONVERT(NVARCHAR(20), GETDATE(), 112) + 
                     REPLACE(CONVERT(NVARCHAR(20), GETDATE(), 108), ':', '');
    
    SET @BackupURL = @StorageURL + '/' + @DatabaseName + '_' + @Timestamp + '.bak';
    
    -- 백업 수행
    BACKUP DATABASE @DatabaseName
    TO URL = @BackupURL
    WITH COMPRESSION, CHECKSUM, FORMAT, INIT,
         STATS = 10;
    
    -- 메타데이터 기록
    INSERT INTO CloudBackupHistory 
        (DatabaseName, BackupURL, BackupDate, ExpirationDate)
    VALUES 
        (@DatabaseName, @BackupURL, GETDATE(), 
         DATEADD(DAY, @RetentionDays, GETDATE()));
END;
```

### 6.2 하이브리드 백업 전략

```sql
-- 로컬과 클라우드 동시 백업
CREATE PROCEDURE sp_HybridBackup
    @DatabaseName NVARCHAR(128),
    @LocalPath NVARCHAR(500),
    @CloudURL NVARCHAR(500)
AS
BEGIN
    DECLARE @Timestamp NVARCHAR(20);
    DECLARE @LocalFile NVARCHAR(500);
    DECLARE @CloudFile NVARCHAR(500);
    
    SET @Timestamp = CONVERT(NVARCHAR(20), GETDATE(), 112);
    SET @LocalFile = @LocalPath + '\' + @DatabaseName + '_' + @Timestamp + '.bak';
    SET @CloudFile = @CloudURL + '/' + @DatabaseName + '_' + @Timestamp + '.bak';
    
    -- 로컬과 클라우드에 동시 백업
    BACKUP DATABASE @DatabaseName
    TO DISK = @LocalFile,
       URL = @CloudFile
    WITH COMPRESSION, CHECKSUM, FORMAT, INIT,
         MIRROR TO DISK = @LocalPath + '\Mirror\' + @DatabaseName + '_' + @Timestamp + '.bak';
    
    -- 백업 검증
    RESTORE VERIFYONLY FROM DISK = @LocalFile WITH CHECKSUM;
    RESTORE VERIFYONLY FROM URL = @CloudFile WITH CHECKSUM;
END;
```

## 마무리

효과적인 백업과 복구 전략은 데이터 보호의 핵심입니다. 정기적인 백업, 복구 테스트, 그리고 자동화된 프로세스를 통해 RTO와 RPO 목표를 달성할 수 있습니다. 클라우드 서비스를 활용한 하이브리드 전략은 추가적인 보호 계층을 제공합니다. 다음 장에서는 데이터베이스 보안과 권한 관리를 통한 데이터 보호 방법을 학습하겠습니다.