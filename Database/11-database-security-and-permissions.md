# 데이터베이스 보안과 권한 관리

## 개요

데이터베이스 보안은 민감한 데이터를 보호하고 규정 준수를 보장하는 필수 요소입니다. 이 장에서는 인증과 권한 부여, 데이터 암호화, 감사 및 모니터링, 보안 모범 사례, 그리고 규정 준수를 위한 구현 방법을 학습합니다.

## 1. 인증과 권한 부여

### 1.1 인증 모드와 로그인 관리

```sql
-- 인증 모드 확인 (Windows/Mixed)
SELECT 
    CASE SERVERPROPERTY('IsIntegratedSecurityOnly')
        WHEN 1 THEN 'Windows Authentication Only'
        WHEN 0 THEN 'Mixed Mode Authentication'
    END AS AuthenticationMode;

-- SQL Server 로그인 생성
CREATE LOGIN AppUser 
WITH PASSWORD = 'Str0ng!P@ssw0rd123',
     DEFAULT_DATABASE = YourDatabase,
     CHECK_EXPIRATION = ON,
     CHECK_POLICY = ON,
     CREDENTIAL = YourCredential;  -- 옵션

-- 암호 정책 설정
ALTER LOGIN AppUser 
WITH PASSWORD = 'N3w!Str0ng@P@ssw0rd' 
     OLD_PASSWORD = 'Str0ng!P@ssw0rd123',
     UNLOCK;  -- 잠금 해제

-- Windows 인증 로그인
CREATE LOGIN [DOMAIN\UserName] FROM WINDOWS
WITH DEFAULT_DATABASE = YourDatabase;

-- 로그인 정보 조회
SELECT 
    sp.name AS LoginName,
    sp.type_desc AS LoginType,
    sp.create_date,
    sp.modify_date,
    sp.default_database_name,
    LOGINPROPERTY(sp.name, 'PasswordLastSetTime') AS PasswordLastSetTime,
    LOGINPROPERTY(sp.name, 'DaysUntilExpiration') AS DaysUntilExpiration,
    LOGINPROPERTY(sp.name, 'IsLocked') AS IsLocked,
    LOGINPROPERTY(sp.name, 'BadPasswordCount') AS BadPasswordCount
FROM sys.server_principals sp
WHERE sp.type IN ('S', 'U', 'G')  -- SQL, Windows, Windows Group
ORDER BY sp.create_date DESC;

-- 로그인 감사 트리거
CREATE TRIGGER trg_LogonAudit
ON ALL SERVER WITH EXECUTE AS 'sa'
FOR LOGON
AS
BEGIN
    DECLARE @LoginName NVARCHAR(128) = ORIGINAL_LOGIN();
    DECLARE @ClientHost NVARCHAR(128) = HOST_NAME();
    DECLARE @AppName NVARCHAR(128) = APP_NAME();
    
    INSERT INTO master.dbo.LogonAuditLog 
        (LoginName, LogonTime, ClientHost, ApplicationName, SessionId)
    VALUES 
        (@LoginName, GETDATE(), @ClientHost, @AppName, @@SPID);
END;
```

### 1.2 데이터베이스 사용자와 역할

```sql
-- 데이터베이스 사용자 생성
USE YourDatabase;
GO

CREATE USER AppUser FOR LOGIN AppUser
WITH DEFAULT_SCHEMA = dbo;

-- 포함된 데이터베이스 사용자 (로그인 없이)
CREATE USER ContainedUser 
WITH PASSWORD = 'P@ssw0rd123',
     DEFAULT_SCHEMA = AppSchema;

-- 사용자 정의 데이터베이스 역할
CREATE ROLE DataReader;
CREATE ROLE DataWriter;
CREATE ROLE ReportUser;

-- 역할에 권한 부여
GRANT SELECT ON SCHEMA::dbo TO DataReader;
GRANT INSERT, UPDATE, DELETE ON SCHEMA::dbo TO DataWriter;
GRANT SELECT ON dbo.vw_Reports TO ReportUser;
GRANT EXECUTE ON dbo.sp_GenerateReport TO ReportUser;

-- 사용자를 역할에 추가
ALTER ROLE DataReader ADD MEMBER AppUser;
ALTER ROLE DataWriter ADD MEMBER AppUser;

-- 고정 데이터베이스 역할
ALTER ROLE db_datareader ADD MEMBER ReadOnlyUser;
ALTER ROLE db_datawriter ADD MEMBER DataEntryUser;
ALTER ROLE db_ddladmin ADD MEMBER DeveloperUser;

-- 역할 계층 구조
CREATE ROLE Manager;
ALTER ROLE DataReader ADD MEMBER Manager;
ALTER ROLE DataWriter ADD MEMBER Manager;
GRANT EXECUTE ON SCHEMA::dbo TO Manager;

-- 사용자와 권한 정보 조회
SELECT 
    dp.name AS PrincipalName,
    dp.type_desc AS PrincipalType,
    p.permission_name,
    p.state_desc AS PermissionState,
    p.class_desc AS ObjectClass,
    COALESCE(
        OBJECT_NAME(p.major_id),
        SCHEMA_NAME(p.major_id),
        DB_NAME()
    ) AS ObjectName
FROM sys.database_permissions p
LEFT JOIN sys.database_principals dp ON p.grantee_principal_id = dp.principal_id
WHERE dp.name NOT IN ('public', 'guest')
ORDER BY dp.name, p.permission_name;
```

## 2. 세분화된 권한 관리

### 2.1 객체 수준 권한

```sql
-- 테이블 수준 권한
GRANT SELECT, INSERT ON dbo.Customers TO AppUser;
GRANT UPDATE (Email, Phone) ON dbo.Customers TO AppUser;  -- 특정 열만
DENY DELETE ON dbo.Customers TO AppUser;

-- 뷰를 통한 권한 제어
CREATE VIEW vw_CustomerPublicInfo
AS
SELECT 
    CustomerId,
    CustomerName,
    City,
    Country,
    -- 민감한 정보 제외 (Email, Phone, CreditCard)
    CASE 
        WHEN USER_NAME() IN ('Manager', 'Admin') THEN Email
        ELSE 'Hidden'
    END AS Email
FROM Customers;

GRANT SELECT ON vw_CustomerPublicInfo TO PUBLIC;

-- 저장 프로시저를 통한 권한 캡슐화
CREATE PROCEDURE sp_UpdateCustomerContact
    @CustomerId INT,
    @Email NVARCHAR(100),
    @Phone NVARCHAR(20)
WITH EXECUTE AS OWNER  -- 소유자 권한으로 실행
AS
BEGIN
    -- 감사 로그
    INSERT INTO AuditLog (UserName, Action, TableName, RecordId, Timestamp)
    VALUES (USER_NAME(), 'UPDATE', 'Customers', @CustomerId, GETDATE());
    
    -- 업데이트 수행
    UPDATE Customers
    SET Email = @Email,
        Phone = @Phone,
        ModifiedBy = USER_NAME(),
        ModifiedDate = GETDATE()
    WHERE CustomerId = @CustomerId;
END;

GRANT EXECUTE ON sp_UpdateCustomerContact TO DataEntryUser;

-- 스키마 수준 권한
CREATE SCHEMA Reports AUTHORIZATION ReportManager;
GRANT SELECT ON SCHEMA::Reports TO ReportUser;
GRANT CREATE TABLE, CREATE VIEW TO ReportDeveloper;
```

### 2.2 행 수준 보안 (Row-Level Security)

```sql
-- 보안 정책을 위한 술어 함수
CREATE FUNCTION fn_SecurityPredicate(@UserId INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN 
    SELECT 1 AS Result
    WHERE @UserId = CAST(SESSION_CONTEXT(N'UserId') AS INT)
       OR USER_NAME() = 'Admin';

-- 테이블에 보안 정책 적용
CREATE SECURITY POLICY CustomerSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(UserId) ON dbo.Orders,
ADD BLOCK PREDICATE dbo.fn_SecurityPredicate(UserId) ON dbo.Orders;

-- 세션 컨텍스트 설정
EXEC sp_set_session_context @key = N'UserId', @value = 123;

-- 다중 테넌트 환경의 RLS
CREATE FUNCTION fn_TenantSecurity(@TenantId INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN 
    SELECT 1 AS Result
    FROM dbo.UserTenants ut
    WHERE ut.UserId = USER_ID()
    AND ut.TenantId = @TenantId
    AND ut.IsActive = 1;

CREATE SECURITY POLICY TenantIsolation
ADD FILTER PREDICATE dbo.fn_TenantSecurity(TenantId) ON dbo.Customers,
ADD FILTER PREDICATE dbo.fn_TenantSecurity(TenantId) ON dbo.Orders,
ADD BLOCK PREDICATE dbo.fn_TenantSecurity(TenantId) ON dbo.Customers AFTER INSERT,
ADD BLOCK PREDICATE dbo.fn_TenantSecurity(TenantId) ON dbo.Orders AFTER INSERT;

-- 동적 데이터 마스킹
ALTER TABLE Customers
ALTER COLUMN Email ADD MASKED WITH (FUNCTION = 'email()');

ALTER TABLE Customers
ALTER COLUMN Phone ADD MASKED WITH (FUNCTION = 'partial(0,"XXX-XXX-",4)');

ALTER TABLE Customers
ALTER COLUMN CreditCardNumber ADD MASKED WITH (FUNCTION = 'partial(0,"XXXX-XXXX-XXXX-",4)');

-- 마스킹 권한 부여
GRANT UNMASK TO FinanceManager;
```

## 3. 데이터 암호화

### 3.1 투명한 데이터 암호화 (TDE)

```sql
-- 마스터 키 생성
USE master;
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'M@st3rK3y!P@ssw0rd';

-- 인증서 생성
CREATE CERTIFICATE TDECertificate
WITH SUBJECT = 'TDE Certificate for YourDatabase';

-- 인증서 백업 (중요!)
BACKUP CERTIFICATE TDECertificate
TO FILE = 'C:\Secure\TDECertificate.cer'
WITH PRIVATE KEY (
    FILE = 'C:\Secure\TDECertificate.pvk',
    ENCRYPTION BY PASSWORD = 'Pr1v@t3K3y!P@ssw0rd'
);

-- 데이터베이스 암호화 키 생성
USE YourDatabase;
CREATE DATABASE ENCRYPTION KEY
WITH ALGORITHM = AES_256
ENCRYPTION BY SERVER CERTIFICATE TDECertificate;

-- TDE 활성화
ALTER DATABASE YourDatabase SET ENCRYPTION ON;

-- TDE 상태 확인
SELECT 
    d.name AS DatabaseName,
    dek.encryption_state,
    CASE dek.encryption_state
        WHEN 0 THEN 'No encryption'
        WHEN 1 THEN 'Unencrypted'
        WHEN 2 THEN 'Encryption in progress'
        WHEN 3 THEN 'Encrypted'
        WHEN 4 THEN 'Key change in progress'
        WHEN 5 THEN 'Decryption in progress'
        WHEN 6 THEN 'Protection change in progress'
    END AS EncryptionStateDesc,
    dek.percent_complete,
    dek.key_algorithm,
    dek.key_length,
    c.name AS CertificateName
FROM sys.databases d
LEFT JOIN sys.dm_database_encryption_keys dek ON d.database_id = dek.database_id
LEFT JOIN sys.certificates c ON dek.encryptor_thumbprint = c.thumbprint
WHERE d.database_id > 4;
```

### 3.2 열 수준 암호화

```sql
-- 대칭 키 생성
CREATE SYMMETRIC KEY CreditCardKey
WITH ALGORITHM = AES_256
ENCRYPTION BY CERTIFICATE EncryptionCertificate;

-- 암호화된 열을 가진 테이블
CREATE TABLE CustomerPayments (
    PaymentId INT IDENTITY(1,1) PRIMARY KEY,
    CustomerId INT NOT NULL,
    CardNumber VARBINARY(256),  -- 암호화된 데이터
    CardNumberHash AS HASHBYTES('SHA2_256', CONVERT(NVARCHAR(50), PaymentId) + 'Salt'),  -- 검색용
    ExpiryDate VARBINARY(256),
    CVV VARBINARY(256),
    CreatedDate DATETIME DEFAULT GETDATE()
);

-- 데이터 암호화 프로시저
CREATE PROCEDURE sp_InsertPaymentSecure
    @CustomerId INT,
    @CardNumber NVARCHAR(20),
    @ExpiryDate NVARCHAR(5),
    @CVV NVARCHAR(4)
AS
BEGIN
    -- 키 열기
    OPEN SYMMETRIC KEY CreditCardKey
    DECRYPTION BY CERTIFICATE EncryptionCertificate;
    
    -- 암호화하여 삽입
    INSERT INTO CustomerPayments (CustomerId, CardNumber, ExpiryDate, CVV)
    VALUES (
        @CustomerId,
        ENCRYPTBYKEY(KEY_GUID('CreditCardKey'), @CardNumber),
        ENCRYPTBYKEY(KEY_GUID('CreditCardKey'), @ExpiryDate),
        ENCRYPTBYKEY(KEY_GUID('CreditCardKey'), @CVV)
    );
    
    -- 키 닫기
    CLOSE SYMMETRIC KEY CreditCardKey;
END;

-- 데이터 복호화 함수
CREATE FUNCTION fn_DecryptCardNumber(@EncryptedCard VARBINARY(256))
RETURNS NVARCHAR(20)
WITH EXECUTE AS OWNER
AS
BEGIN
    DECLARE @CardNumber NVARCHAR(20);
    
    -- 권한 확인
    IF IS_ROLEMEMBER('PaymentAdmin') = 0 AND IS_ROLEMEMBER('db_owner') = 0
        RETURN 'XXXX-XXXX-XXXX-XXXX';
    
    -- 복호화
    SET @CardNumber = CONVERT(NVARCHAR(20), 
        DECRYPTBYKEY(@EncryptedCard));
    
    RETURN @CardNumber;
END;

-- Always Encrypted 설정
CREATE COLUMN MASTER KEY CMK_AlwaysEncrypted
WITH (
    KEY_STORE_PROVIDER_NAME = 'MSSQL_CERTIFICATE_STORE',
    KEY_PATH = 'CurrentUser/My/3E5F2B4A1C0D9E8F7A6B5C4D3E2F1A0B'
);

CREATE COLUMN ENCRYPTION KEY CEK_AlwaysEncrypted
WITH VALUES (
    COLUMN_MASTER_KEY = CMK_AlwaysEncrypted,
    ALGORITHM = 'RSA_OAEP',
    ENCRYPTED_VALUE = 0x01234567...  -- 실제 암호화된 값
);

-- Always Encrypted 열 정의
CREATE TABLE SensitiveData (
    Id INT IDENTITY(1,1) PRIMARY KEY,
    SSN NVARCHAR(11) COLLATE Latin1_General_BIN2
        ENCRYPTED WITH (
            COLUMN_ENCRYPTION_KEY = CEK_AlwaysEncrypted,
            ENCRYPTION_TYPE = DETERMINISTIC,
            ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256'
        ),
    Salary DECIMAL(10,2)
        ENCRYPTED WITH (
            COLUMN_ENCRYPTION_KEY = CEK_AlwaysEncrypted,
            ENCRYPTION_TYPE = RANDOMIZED,
            ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256'
        )
);
```

## 4. 감사와 모니터링

### 4.1 SQL Server Audit

```sql
-- 서버 감사 생성
CREATE SERVER AUDIT SecurityAudit
TO FILE (
    FILEPATH = 'C:\AuditLogs\',
    MAXSIZE = 100 MB,
    MAX_ROLLOVER_FILES = 10,
    RESERVE_DISK_SPACE = OFF
)
WITH (
    QUEUE_DELAY = 1000,
    ON_FAILURE = CONTINUE
);

-- 서버 감사 사양
CREATE SERVER AUDIT SPECIFICATION ServerAuditSpec
FOR SERVER AUDIT SecurityAudit
ADD (FAILED_LOGIN_GROUP),
ADD (SUCCESSFUL_LOGIN_GROUP),
ADD (SERVER_ROLE_MEMBER_CHANGE_GROUP),
ADD (DATABASE_ROLE_MEMBER_CHANGE_GROUP),
ADD (SERVER_PERMISSION_CHANGE_GROUP);

-- 데이터베이스 감사 사양
USE YourDatabase;
CREATE DATABASE AUDIT SPECIFICATION DatabaseAuditSpec
FOR SERVER AUDIT SecurityAudit
ADD (SELECT ON dbo.Customers BY PUBLIC),
ADD (INSERT, UPDATE, DELETE ON dbo.Orders BY PUBLIC),
ADD (EXECUTE ON SCHEMA::dbo BY PUBLIC),
ADD (DATABASE_PERMISSION_CHANGE_GROUP),
ADD (SCHEMA_OBJECT_PERMISSION_CHANGE_GROUP);

-- 감사 활성화
ALTER SERVER AUDIT SecurityAudit WITH (STATE = ON);
ALTER SERVER AUDIT SPECIFICATION ServerAuditSpec WITH (STATE = ON);
ALTER DATABASE AUDIT SPECIFICATION DatabaseAuditSpec WITH (STATE = ON);

-- 감사 로그 조회
SELECT 
    event_time,
    action_id,
    succeeded,
    session_id,
    server_principal_name,
    database_principal_name,
    database_name,
    schema_name,
    object_name,
    statement,
    file_name,
    audit_file_offset
FROM sys.fn_get_audit_file('C:\AuditLogs\SecurityAudit*.sqlaudit', DEFAULT, DEFAULT)
WHERE event_time > DATEADD(HOUR, -24, GETDATE())
ORDER BY event_time DESC;

-- 사용자 정의 감사 이벤트
EXEC sp_audit_write 
    @user_defined_event_id = 101,
    @succeeded = 1,
    @user_defined_information = N'High-value transaction processed';
```

### 4.2 확장 이벤트 (Extended Events)

```sql
-- 보안 모니터링을 위한 확장 이벤트 세션
CREATE EVENT SESSION SecurityMonitoring
ON SERVER
ADD EVENT sqlserver.login_event(
    ACTION (
        sqlserver.client_hostname,
        sqlserver.client_app_name,
        sqlserver.username,
        sqlserver.session_id
    )
    WHERE sqlserver.username NOT LIKE 'NT SERVICE%'
),
ADD EVENT sqlserver.error_reported(
    ACTION (
        sqlserver.session_id,
        sqlserver.username,
        sqlserver.sql_text
    )
    WHERE error_number IN (18456, 18470, 18452)  -- 로그인 실패 오류
),
ADD EVENT sqlserver.sql_statement_completed(
    ACTION (
        sqlserver.session_id,
        sqlserver.username,
        sqlserver.database_name,
        sqlserver.sql_text
    )
    WHERE sqlserver.database_name = 'YourDatabase'
    AND (
        sqlserver.sql_text LIKE '%DELETE%'
        OR sqlserver.sql_text LIKE '%TRUNCATE%'
        OR sqlserver.sql_text LIKE '%DROP%'
    )
)
ADD TARGET package0.event_file(
    SET filename = 'C:\XEvents\SecurityMonitoring.xel',
    max_file_size = 100,
    max_rollover_files = 5
)
WITH (
    MAX_MEMORY = 4096 KB,
    EVENT_RETENTION_MODE = ALLOW_SINGLE_EVENT_LOSS,
    MAX_DISPATCH_LATENCY = 30 SECONDS,
    STARTUP_STATE = ON
);

-- 세션 시작
ALTER EVENT SESSION SecurityMonitoring ON SERVER STATE = START;

-- 이벤트 데이터 조회
SELECT 
    event_data.value('(event/@timestamp)[1]', 'datetime2') AS event_time,
    event_data.value('(event/@name)[1]', 'varchar(50)') AS event_name,
    event_data.value('(event/action[@name="username"]/value)[1]', 'varchar(100)') AS username,
    event_data.value('(event/action[@name="database_name"]/value)[1]', 'varchar(100)') AS database_name,
    event_data.value('(event/action[@name="sql_text"]/value)[1]', 'varchar(max)') AS sql_text
FROM (
    SELECT CAST(event_data AS XML) AS event_data
    FROM sys.fn_xe_file_target_read_file('C:\XEvents\SecurityMonitoring*.xel', NULL, NULL, NULL)
) AS EventData
ORDER BY event_time DESC;
```

## 5. 보안 모범 사례

### 5.1 최소 권한 원칙

```sql
-- 역할 기반 접근 제어 구현
-- 1. 애플리케이션 역할 생성
CREATE APPLICATION ROLE AppRole
WITH PASSWORD = 'R0l3!P@ssw0rd',
     DEFAULT_SCHEMA = dbo;

-- 2. 최소 권한 부여
GRANT SELECT ON dbo.Products TO AppRole;
GRANT SELECT, INSERT ON dbo.Orders TO AppRole;
GRANT EXECUTE ON dbo.sp_ProcessOrder TO AppRole;

-- 3. 애플리케이션에서 역할 활성화
CREATE PROCEDURE sp_ActivateAppRole
    @Cookie VARBINARY(8000) OUTPUT
AS
BEGIN
    DECLARE @Result INT;
    EXEC @Result = sp_setapprole 
        @rolename = 'AppRole',
        @password = 'R0l3!P@ssw0rd',
        @fCreateCookie = true,
        @cookie = @Cookie OUTPUT;
    
    RETURN @Result;
END;

-- 서비스 계정 권한 제한
CREATE USER ServiceAccount FOR LOGIN ServiceLogin;

-- 특정 작업만 허용
GRANT SELECT ON dbo.fn_GetOrderStatus TO ServiceAccount;
GRANT EXECUTE ON dbo.sp_UpdateOrderStatus TO ServiceAccount;
DENY SELECT ON dbo.Customers TO ServiceAccount;

-- 권한 검증 뷰
CREATE VIEW vw_UserPermissions
AS
SELECT 
    p.state_desc + ' ' + p.permission_name AS Permission,
    p.class_desc AS ObjectClass,
    CASE p.class
        WHEN 0 THEN 'Database'
        WHEN 1 THEN OBJECT_NAME(p.major_id)
        WHEN 3 THEN SCHEMA_NAME(p.major_id)
        ELSE 'Unknown'
    END AS ObjectName,
    pr.name AS PrincipalName,
    pr.type_desc AS PrincipalType
FROM sys.database_permissions p
INNER JOIN sys.database_principals pr ON p.grantee_principal_id = pr.principal_id
WHERE pr.name = USER_NAME()
   OR pr.principal_id IN (
       SELECT role_principal_id
       FROM sys.database_role_members
       WHERE member_principal_id = USER_ID()
   );
```

### 5.2 SQL Injection 방지

```sql
-- 안전하지 않은 동적 SQL (취약점)
CREATE PROCEDURE sp_SearchCustomers_Unsafe
    @SearchTerm NVARCHAR(100)
AS
BEGIN
    DECLARE @SQL NVARCHAR(MAX);
    SET @SQL = 'SELECT * FROM Customers WHERE CustomerName LIKE ''%' + 
               @SearchTerm + '%''';
    EXEC(@SQL);  -- SQL Injection 취약점!
END;

-- 안전한 매개변수화된 쿼리
CREATE PROCEDURE sp_SearchCustomers_Safe
    @SearchTerm NVARCHAR(100)
AS
BEGIN
    SELECT * FROM Customers 
    WHERE CustomerName LIKE '%' + @SearchTerm + '%';
END;

-- sp_executesql을 사용한 안전한 동적 SQL
CREATE PROCEDURE sp_DynamicSearch_Safe
    @TableName NVARCHAR(128),
    @ColumnName NVARCHAR(128),
    @SearchValue NVARCHAR(100)
AS
BEGIN
    -- 테이블과 열 이름 검증
    IF NOT EXISTS (
        SELECT 1 FROM INFORMATION_SCHEMA.COLUMNS
        WHERE TABLE_NAME = @TableName
        AND COLUMN_NAME = @ColumnName
        AND TABLE_SCHEMA = 'dbo'
    )
    BEGIN
        RAISERROR('Invalid table or column name', 16, 1);
        RETURN;
    END
    
    DECLARE @SQL NVARCHAR(MAX);
    SET @SQL = N'SELECT * FROM dbo.' + QUOTENAME(@TableName) + 
               N' WHERE ' + QUOTENAME(@ColumnName) + N' = @Value';
    
    EXEC sp_executesql @SQL, N'@Value NVARCHAR(100)', @Value = @SearchValue;
END;

-- 입력 검증 함수
CREATE FUNCTION fn_ValidateInput(@Input NVARCHAR(MAX))
RETURNS BIT
AS
BEGIN
    -- SQL Injection 패턴 검사
    IF @Input LIKE '%[-';]%' 
       OR @Input LIKE '%[/*]%'
       OR @Input LIKE '%xp_%'
       OR @Input LIKE '%sp_%'
       OR UPPER(@Input) LIKE '%UNION%'
       OR UPPER(@Input) LIKE '%SELECT%'
       OR UPPER(@Input) LIKE '%INSERT%'
       OR UPPER(@Input) LIKE '%UPDATE%'
       OR UPPER(@Input) LIKE '%DELETE%'
       OR UPPER(@Input) LIKE '%DROP%'
       OR UPPER(@Input) LIKE '%CREATE%'
       OR UPPER(@Input) LIKE '%ALTER%'
       OR UPPER(@Input) LIKE '%EXEC%'
    BEGIN
        RETURN 0;  -- 유효하지 않음
    END
    
    RETURN 1;  -- 유효함
END;
```

## 6. 규정 준수와 데이터 보호

### 6.1 GDPR 준수

```sql
-- 개인정보 식별 및 분류
-- 민감도 분류 추가
ADD SENSITIVITY CLASSIFICATION TO dbo.Customers.Email
WITH (LABEL = 'Confidential - GDPR', INFORMATION_TYPE = 'Contact Info');

ADD SENSITIVITY CLASSIFICATION TO dbo.Customers.Phone
WITH (LABEL = 'Confidential - GDPR', INFORMATION_TYPE = 'Contact Info');

ADD SENSITIVITY CLASSIFICATION TO dbo.Customers.DateOfBirth
WITH (LABEL = 'Highly Confidential', INFORMATION_TYPE = 'Personal');

-- 분류된 데이터 조회
SELECT 
    SCHEMA_NAME(o.schema_id) AS SchemaName,
    o.name AS TableName,
    c.name AS ColumnName,
    sc.information_type,
    sc.label,
    sc.rank
FROM sys.sensitivity_classifications sc
INNER JOIN sys.objects o ON sc.major_id = o.object_id
INNER JOIN sys.columns c ON sc.major_id = c.object_id 
    AND sc.minor_id = c.column_id
ORDER BY sc.rank DESC, o.name, c.column_id;

-- 개인정보 삭제 권리 (Right to be Forgotten)
CREATE PROCEDURE sp_ErasePersonalData
    @CustomerId INT,
    @ConfirmationCode NVARCHAR(50)
AS
BEGIN
    SET XACT_ABORT ON;
    
    BEGIN TRANSACTION;
        -- 감사 로그
        INSERT INTO GDPRAuditLog 
            (Action, CustomerId, RequestDate, ConfirmationCode)
        VALUES 
            ('Right to Erasure', @CustomerId, GETDATE(), @ConfirmationCode);
        
        -- 관련 데이터 익명화
        UPDATE Customers
        SET FirstName = 'DELETED',
            LastName = 'DELETED',
            Email = CONCAT('deleted_', CustomerId, '@deleted.com'),
            Phone = 'DELETED',
            Address = 'DELETED',
            DateOfBirth = '1900-01-01',
            IsDeleted = 1,
            DeletionDate = GETDATE()
        WHERE CustomerId = @CustomerId;
        
        -- 주문 기록은 법적 요구사항으로 보존하되 익명화
        UPDATE Orders
        SET CustomerName = 'DELETED CUSTOMER',
            ShippingAddress = 'DELETED',
            ContactInfo = 'DELETED'
        WHERE CustomerId = @CustomerId;
        
    COMMIT;
END;

-- 데이터 이동성 (Right to Portability)
CREATE PROCEDURE sp_ExportPersonalData
    @CustomerId INT
AS
BEGIN
    -- JSON 형식으로 개인 데이터 내보내기
    SELECT 
        'CustomerData' AS DataCategory,
        (
            SELECT 
                c.CustomerId,
                c.FirstName,
                c.LastName,
                c.Email,
                c.Phone,
                c.DateOfBirth,
                c.RegistrationDate,
                (
                    SELECT 
                        OrderId,
                        OrderDate,
                        TotalAmount,
                        Status
                    FROM Orders o
                    WHERE o.CustomerId = c.CustomerId
                    FOR JSON PATH
                ) AS Orders
            FROM Customers c
            WHERE c.CustomerId = @CustomerId
            FOR JSON PATH, WITHOUT_ARRAY_WRAPPER
        ) AS PersonalData
    FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
END;
```

### 6.2 PCI DSS 준수

```sql
-- PCI DSS 요구사항 구현
-- 카드 데이터 보안
CREATE TABLE PCIAuditLog (
    LogId INT IDENTITY(1,1) PRIMARY KEY,
    EventTime DATETIME DEFAULT GETDATE(),
    UserName NVARCHAR(128),
    IPAddress NVARCHAR(45),
    Action NVARCHAR(50),
    TableName NVARCHAR(128),
    RecordId INT,
    OldValue NVARCHAR(MAX),
    NewValue NVARCHAR(MAX),
    INDEX IX_PCIAudit_Time (EventTime)
);

-- 카드 데이터 접근 트리거
CREATE TRIGGER trg_CardDataAccess
ON CustomerPayments
AFTER SELECT
AS
BEGIN
    INSERT INTO PCIAuditLog 
        (UserName, IPAddress, Action, TableName)
    VALUES 
        (USER_NAME(), CONNECTIONPROPERTY('client_net_address'), 
         'SELECT', 'CustomerPayments');
END;

-- 암호화 키 순환
CREATE PROCEDURE sp_RotateEncryptionKeys
AS
BEGIN
    SET XACT_ABORT ON;
    
    BEGIN TRANSACTION;
        -- 새 키 생성
        CREATE SYMMETRIC KEY CreditCardKey_New
        WITH ALGORITHM = AES_256
        ENCRYPTION BY CERTIFICATE EncryptionCertificate;
        
        -- 데이터 재암호화
        OPEN SYMMETRIC KEY CreditCardKey
        DECRYPTION BY CERTIFICATE EncryptionCertificate;
        
        OPEN SYMMETRIC KEY CreditCardKey_New
        DECRYPTION BY CERTIFICATE EncryptionCertificate;
        
        UPDATE CustomerPayments
        SET CardNumber = ENCRYPTBYKEY(KEY_GUID('CreditCardKey_New'),
                         DECRYPTBYKEY(CardNumber));
        
        CLOSE SYMMETRIC KEY CreditCardKey;
        CLOSE SYMMETRIC KEY CreditCardKey_New;
        
        -- 이전 키 삭제
        DROP SYMMETRIC KEY CreditCardKey;
        
        -- 새 키 이름 변경
        -- SQL Server doesn't support key rename, 
        -- so this would be handled differently in practice
        
    COMMIT;
END;

-- 접근 제어 리스트
CREATE TABLE PCIAccessControlList (
    UserId INT,
    ResourceType NVARCHAR(50),
    ResourceId INT,
    Permission NVARCHAR(20),
    GrantedBy NVARCHAR(128),
    GrantedDate DATETIME,
    ExpiryDate DATETIME,
    PRIMARY KEY (UserId, ResourceType, ResourceId, Permission)
);

-- 보안 설정 검증
CREATE PROCEDURE sp_ValidatePCICompliance
AS
BEGIN
    -- 결과 테이블
    DECLARE @Results TABLE (
        CheckName NVARCHAR(100),
        Status NVARCHAR(20),
        Details NVARCHAR(500)
    );
    
    -- 1. 기본 sa 계정 확인
    INSERT INTO @Results
    SELECT 'SA Account Status',
           CASE WHEN is_disabled = 1 THEN 'PASS' ELSE 'FAIL' END,
           'SA account should be disabled'
    FROM sys.server_principals
    WHERE name = 'sa';
    
    -- 2. 암호 정책 확인
    INSERT INTO @Results
    SELECT 'Password Policy',
           CASE WHEN COUNT(*) = 0 THEN 'PASS' ELSE 'FAIL' END,
           'All SQL logins should have password policy enforced'
    FROM sys.sql_logins
    WHERE is_policy_checked = 0;
    
    -- 3. 암호화 상태 확인
    INSERT INTO @Results
    SELECT 'Database Encryption',
           CASE WHEN encryption_state = 3 THEN 'PASS' ELSE 'FAIL' END,
           'Database should be encrypted with TDE'
    FROM sys.dm_database_encryption_keys
    WHERE database_id = DB_ID();
    
    -- 4. 감사 활성화 확인
    INSERT INTO @Results
    SELECT 'Audit Status',
           CASE WHEN COUNT(*) > 0 THEN 'PASS' ELSE 'FAIL' END,
           'Security audit should be enabled'
    FROM sys.server_audits
    WHERE is_state_enabled = 1;
    
    SELECT * FROM @Results;
END;
```

## 7. 고급 보안 기능

### 7.1 Azure AD 통합

```sql
-- Azure AD 관리자 설정 (Azure SQL Database)
-- Azure Portal 또는 PowerShell로 설정

-- Azure AD 사용자 생성
CREATE USER [user@company.com] FROM EXTERNAL PROVIDER;

-- Azure AD 그룹 생성
CREATE USER [AAD-DBAdmins] FROM EXTERNAL PROVIDER;

-- 권한 부여
ALTER ROLE db_datareader ADD MEMBER [user@company.com];
ALTER ROLE db_owner ADD MEMBER [AAD-DBAdmins];

-- 조건부 접근 정책을 위한 연결 정보
SELECT 
    session_id,
    login_name,
    client_net_address,
    client_interface_name,
    auth_scheme,
    protocol_type,
    local_net_address,
    local_tcp_port
FROM sys.dm_exec_connections
WHERE session_id = @@SPID;
```

### 7.2 위협 탐지와 취약점 평가

```sql
-- 의심스러운 활동 모니터링
CREATE TABLE SecurityAlerts (
    AlertId INT IDENTITY(1,1) PRIMARY KEY,
    AlertTime DATETIME DEFAULT GETDATE(),
    AlertType NVARCHAR(50),
    Severity INT,
    Description NVARCHAR(MAX),
    AffectedObject NVARCHAR(256),
    UserName NVARCHAR(128),
    ClientIP NVARCHAR(45),
    IsResolved BIT DEFAULT 0
);

-- 비정상 접근 패턴 감지
CREATE PROCEDURE sp_DetectAnomalousAccess
AS
BEGIN
    -- 비정상적인 시간대 접근
    INSERT INTO SecurityAlerts (AlertType, Severity, Description, UserName)
    SELECT 'After Hours Access', 3,
           'Login detected outside business hours',
           login_name
    FROM sys.dm_exec_sessions
    WHERE is_user_process = 1
    AND (DATEPART(HOUR, login_time) < 6 OR DATEPART(HOUR, login_time) > 22)
    AND login_name NOT IN (SELECT UserName FROM AuthorizedAfterHoursUsers);
    
    -- 대량 데이터 추출 시도
    INSERT INTO SecurityAlerts (AlertType, Severity, Description, UserName)
    SELECT 'Mass Data Export', 4,
           'Large data export detected: ' + CAST(row_count AS NVARCHAR(20)) + ' rows',
           USER_NAME()
    FROM sys.dm_exec_requests r
    CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) st
    WHERE r.row_count > 100000
    AND st.text LIKE '%SELECT%'
    AND USER_NAME() NOT IN (SELECT UserName FROM AuthorizedDataExportUsers);
    
    -- 반복된 로그인 실패
    WITH LoginFailures AS (
        SELECT 
            login_name,
            COUNT(*) AS FailureCount
        FROM sys.fn_get_audit_file('C:\AuditLogs\*.sqlaudit', DEFAULT, DEFAULT)
        WHERE action_id = 'LGIF'  -- Login Failed
        AND event_time > DATEADD(MINUTE, -10, GETDATE())
        GROUP BY login_name
    )
    INSERT INTO SecurityAlerts (AlertType, Severity, Description, UserName)
    SELECT 'Brute Force Attempt', 5,
           'Multiple login failures: ' + CAST(FailureCount AS NVARCHAR(10)) + ' attempts',
           login_name
    FROM LoginFailures
    WHERE FailureCount > 5;
END;

-- 자동 위협 대응
CREATE PROCEDURE sp_RespondToThreat
    @AlertId INT
AS
BEGIN
    DECLARE @AlertType NVARCHAR(50);
    DECLARE @UserName NVARCHAR(128);
    
    SELECT @AlertType = AlertType, @UserName = UserName
    FROM SecurityAlerts
    WHERE AlertId = @AlertId;
    
    IF @AlertType = 'Brute Force Attempt'
    BEGIN
        -- 계정 잠금
        ALTER LOGIN [@UserName] DISABLE;
        
        -- 알림 발송
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'Security_Profile',
            @recipients = 'security@company.com',
            @subject = 'Security Alert: Account Locked',
            @body = 'Account has been locked due to multiple failed login attempts.';
    END
    
    -- 알림 해결 표시
    UPDATE SecurityAlerts
    SET IsResolved = 1
    WHERE AlertId = @AlertId;
END;
```

## 마무리

데이터베이스 보안은 다층적 접근이 필요한 복잡한 영역입니다. 강력한 인증과 권한 부여, 데이터 암호화, 지속적인 모니터링과 감사, 그리고 규정 준수는 모두 포괄적인 보안 전략의 필수 구성 요소입니다. 정기적인 보안 평가와 업데이트를 통해 진화하는 위협으로부터 데이터를 보호해야 합니다. 다음 장에서는 복제와 샤딩을 통한 확장성과 가용성 향상 방법을 학습하겠습니다.