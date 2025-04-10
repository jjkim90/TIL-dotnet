# 데이터베이스 설계 기초

## 개요

효과적인 데이터베이스 설계는 성능, 확장성, 유지보수성을 좌우하는 핵심 요소입니다. 이 장에서는 ER 다이어그램을 통한 개념적 설계, 정규화를 통한 논리적 설계, 역정규화 전략, 그리고 실무에서 활용되는 명명 규칙을 학습합니다.

## 1. 개념적 설계와 ER 다이어그램

### 1.1 엔티티(Entity) 정의

```sql
-- 엔티티 예제: 온라인 쇼핑몰 시스템
/*
주요 엔티티:
1. Customer (고객)
2. Product (상품)
3. Order (주문)
4. Category (카테고리)
5. Supplier (공급업체)
6. Employee (직원)
7. Payment (결제)
8. Shipping (배송)
*/

-- 강한 엔티티 (독립적으로 존재)
CREATE TABLE Customers (
    CustomerId INT PRIMARY KEY IDENTITY(1,1),
    Email VARCHAR(255) UNIQUE NOT NULL,
    FirstName NVARCHAR(50) NOT NULL,
    LastName NVARCHAR(50) NOT NULL,
    PhoneNumber VARCHAR(20),
    DateOfBirth DATE,
    RegistrationDate DATETIME DEFAULT GETDATE(),
    IsActive BIT DEFAULT 1
);

-- 약한 엔티티 (다른 엔티티에 의존)
CREATE TABLE CustomerAddresses (
    CustomerId INT NOT NULL,
    AddressType VARCHAR(20) NOT NULL, -- 'Home', 'Work', 'Shipping'
    Street NVARCHAR(200) NOT NULL,
    City NVARCHAR(100) NOT NULL,
    State NVARCHAR(50),
    PostalCode VARCHAR(20),
    Country NVARCHAR(100) NOT NULL,
    IsDefault BIT DEFAULT 0,
    PRIMARY KEY (CustomerId, AddressType),
    FOREIGN KEY (CustomerId) REFERENCES Customers(CustomerId) ON DELETE CASCADE
);
```

### 1.2 관계(Relationship) 유형

```sql
-- 1:1 관계 (One-to-One)
-- 고객과 고객 상세 정보
CREATE TABLE CustomerDetails (
    CustomerId INT PRIMARY KEY,
    PreferredLanguage VARCHAR(10) DEFAULT 'ko',
    MarketingOptIn BIT DEFAULT 1,
    LoyaltyPoints INT DEFAULT 0,
    CustomerLevel VARCHAR(20) DEFAULT 'Bronze', -- Bronze, Silver, Gold, Platinum
    Notes NVARCHAR(MAX),
    FOREIGN KEY (CustomerId) REFERENCES Customers(CustomerId)
);

-- 1:N 관계 (One-to-Many)
-- 카테고리와 상품
CREATE TABLE Categories (
    CategoryId INT PRIMARY KEY IDENTITY(1,1),
    CategoryName NVARCHAR(100) NOT NULL,
    ParentCategoryId INT NULL,
    Description NVARCHAR(500),
    IsActive BIT DEFAULT 1,
    FOREIGN KEY (ParentCategoryId) REFERENCES Categories(CategoryId)
);

CREATE TABLE Products (
    ProductId INT PRIMARY KEY IDENTITY(1,1),
    ProductName NVARCHAR(200) NOT NULL,
    CategoryId INT NOT NULL,
    SupplierId INT NOT NULL,
    UnitPrice DECIMAL(10, 2) NOT NULL CHECK (UnitPrice > 0),
    UnitsInStock INT DEFAULT 0 CHECK (UnitsInStock >= 0),
    ReorderLevel INT DEFAULT 10,
    Discontinued BIT DEFAULT 0,
    FOREIGN KEY (CategoryId) REFERENCES Categories(CategoryId)
);

-- N:M 관계 (Many-to-Many)
-- 주문과 상품 (연결 테이블 필요)
CREATE TABLE Orders (
    OrderId INT PRIMARY KEY IDENTITY(1,1),
    CustomerId INT NOT NULL,
    OrderDate DATETIME DEFAULT GETDATE(),
    RequiredDate DATETIME,
    ShippedDate DATETIME,
    ShipToAddress NVARCHAR(500),
    OrderStatus VARCHAR(20) DEFAULT 'Pending',
    TotalAmount DECIMAL(10, 2),
    FOREIGN KEY (CustomerId) REFERENCES Customers(CustomerId)
);

CREATE TABLE OrderDetails (
    OrderId INT NOT NULL,
    ProductId INT NOT NULL,
    Quantity INT NOT NULL CHECK (Quantity > 0),
    UnitPrice DECIMAL(10, 2) NOT NULL,
    Discount DECIMAL(3, 2) DEFAULT 0 CHECK (Discount >= 0 AND Discount <= 1),
    LineTotal AS (Quantity * UnitPrice * (1 - Discount)),
    PRIMARY KEY (OrderId, ProductId),
    FOREIGN KEY (OrderId) REFERENCES Orders(OrderId),
    FOREIGN KEY (ProductId) REFERENCES Products(ProductId)
);
```

### 1.3 속성(Attribute) 유형

```sql
-- 다양한 속성 유형 예제
CREATE TABLE Employees (
    -- 단순 속성
    EmployeeId INT PRIMARY KEY IDENTITY(1,1),
    FirstName NVARCHAR(50) NOT NULL,
    LastName NVARCHAR(50) NOT NULL,
    
    -- 복합 속성 (주소)
    StreetAddress NVARCHAR(200),
    City NVARCHAR(100),
    State NVARCHAR(50),
    PostalCode VARCHAR(20),
    
    -- 다중값 속성 (별도 테이블로 분리)
    -- Skills는 별도 테이블로 관리
    
    -- 유도 속성
    BirthDate DATE NOT NULL,
    Age AS (DATEDIFF(YEAR, BirthDate, GETDATE())),
    
    -- 키 속성
    EmployeeCode AS ('EMP' + RIGHT('0000' + CAST(EmployeeId AS VARCHAR(10)), 5)),
    
    HireDate DATE DEFAULT GETDATE(),
    Salary DECIMAL(10, 2),
    DepartmentId INT
);

-- 다중값 속성을 위한 별도 테이블
CREATE TABLE EmployeeSkills (
    EmployeeId INT NOT NULL,
    Skill NVARCHAR(100) NOT NULL,
    ProficiencyLevel INT CHECK (ProficiencyLevel BETWEEN 1 AND 5),
    CertificationDate DATE,
    PRIMARY KEY (EmployeeId, Skill),
    FOREIGN KEY (EmployeeId) REFERENCES Employees(EmployeeId)
);
```

## 2. 정규화 (Normalization)

### 2.1 제1정규형 (1NF)

```sql
-- 정규화 전 (1NF 위반 - 반복 그룹 존재)
CREATE TABLE OrdersUnnormalized (
    OrderId INT,
    CustomerName NVARCHAR(100),
    OrderDate DATE,
    -- 반복 그룹
    Product1 NVARCHAR(100),
    Quantity1 INT,
    Price1 DECIMAL(10, 2),
    Product2 NVARCHAR(100),
    Quantity2 INT,
    Price2 DECIMAL(10, 2),
    Product3 NVARCHAR(100),
    Quantity3 INT,
    Price3 DECIMAL(10, 2)
);

-- 1NF 적용 후
CREATE TABLE Orders_1NF (
    OrderId INT NOT NULL,
    CustomerName NVARCHAR(100) NOT NULL,
    OrderDate DATE NOT NULL,
    ProductName NVARCHAR(100) NOT NULL,
    Quantity INT NOT NULL,
    Price DECIMAL(10, 2) NOT NULL,
    PRIMARY KEY (OrderId, ProductName)
);

-- 다중값 속성 정규화 예제
-- 정규화 전
CREATE TABLE StudentsUnnormalized (
    StudentId INT PRIMARY KEY,
    StudentName NVARCHAR(100),
    Courses NVARCHAR(500) -- 'Math,Physics,Chemistry' 형태로 저장
);

-- 1NF 적용 후
CREATE TABLE Students (
    StudentId INT PRIMARY KEY,
    StudentName NVARCHAR(100)
);

CREATE TABLE StudentCourses (
    StudentId INT,
    CourseName NVARCHAR(100),
    PRIMARY KEY (StudentId, CourseName),
    FOREIGN KEY (StudentId) REFERENCES Students(StudentId)
);
```

### 2.2 제2정규형 (2NF)

```sql
-- 2NF 위반 예제 (부분 종속 존재)
CREATE TABLE OrderDetails_1NF (
    OrderId INT NOT NULL,
    ProductId INT NOT NULL,
    ProductName NVARCHAR(100), -- ProductId에만 종속 (부분 종속)
    CategoryName NVARCHAR(50), -- ProductId에만 종속 (부분 종속)
    Quantity INT,
    UnitPrice DECIMAL(10, 2),
    PRIMARY KEY (OrderId, ProductId)
);

-- 2NF 적용 후
-- 주문 상세 테이블 (복합키의 모든 속성에 완전 함수 종속)
CREATE TABLE OrderDetails_2NF (
    OrderId INT NOT NULL,
    ProductId INT NOT NULL,
    Quantity INT NOT NULL,
    UnitPrice DECIMAL(10, 2) NOT NULL,
    PRIMARY KEY (OrderId, ProductId),
    FOREIGN KEY (OrderId) REFERENCES Orders(OrderId),
    FOREIGN KEY (ProductId) REFERENCES Products(ProductId)
);

-- 상품 테이블 (부분 종속 속성 분리)
CREATE TABLE Products_2NF (
    ProductId INT PRIMARY KEY,
    ProductName NVARCHAR(100) NOT NULL,
    CategoryId INT NOT NULL
);

CREATE TABLE Categories_2NF (
    CategoryId INT PRIMARY KEY,
    CategoryName NVARCHAR(50) NOT NULL
);
```

### 2.3 제3정규형 (3NF)

```sql
-- 3NF 위반 예제 (이행적 종속 존재)
CREATE TABLE Employees_2NF (
    EmployeeId INT PRIMARY KEY,
    EmployeeName NVARCHAR(100),
    DepartmentId INT,
    DepartmentName NVARCHAR(100), -- DepartmentId에 종속 (이행적 종속)
    DepartmentLocation NVARCHAR(100), -- DepartmentId에 종속 (이행적 종속)
    Salary DECIMAL(10, 2)
);

-- 3NF 적용 후
CREATE TABLE Employees_3NF (
    EmployeeId INT PRIMARY KEY,
    EmployeeName NVARCHAR(100) NOT NULL,
    DepartmentId INT NOT NULL,
    Salary DECIMAL(10, 2),
    FOREIGN KEY (DepartmentId) REFERENCES Departments_3NF(DepartmentId)
);

CREATE TABLE Departments_3NF (
    DepartmentId INT PRIMARY KEY,
    DepartmentName NVARCHAR(100) NOT NULL,
    LocationId INT NOT NULL,
    FOREIGN KEY (LocationId) REFERENCES Locations_3NF(LocationId)
);

CREATE TABLE Locations_3NF (
    LocationId INT PRIMARY KEY,
    LocationName NVARCHAR(100) NOT NULL,
    Address NVARCHAR(200),
    City NVARCHAR(100),
    Country NVARCHAR(100)
);
```

### 2.4 BCNF (Boyce-Codd Normal Form)

```sql
-- BCNF 위반 예제
-- 교수가 특정 과목만 가르치는 경우
CREATE TABLE CourseEnrollment_3NF (
    StudentId INT,
    CourseId INT,
    ProfessorId INT,
    Grade CHAR(2),
    PRIMARY KEY (StudentId, CourseId, ProfessorId)
);
-- 문제: {StudentId, CourseId} -> ProfessorId (CourseId가 ProfessorId를 결정)

-- BCNF 적용 후
CREATE TABLE Courses_BCNF (
    CourseId INT PRIMARY KEY,
    CourseName NVARCHAR(100),
    ProfessorId INT NOT NULL,
    FOREIGN KEY (ProfessorId) REFERENCES Professors(ProfessorId)
);

CREATE TABLE Enrollments_BCNF (
    StudentId INT,
    CourseId INT,
    Grade CHAR(2),
    EnrollmentDate DATE,
    PRIMARY KEY (StudentId, CourseId),
    FOREIGN KEY (StudentId) REFERENCES Students(StudentId),
    FOREIGN KEY (CourseId) REFERENCES Courses_BCNF(CourseId)
);
```

### 2.5 제4정규형과 제5정규형

```sql
-- 4NF 위반 (다중값 종속)
CREATE TABLE EmployeeProjects_3NF (
    EmployeeId INT,
    ProjectId INT,
    SkillId INT,
    PRIMARY KEY (EmployeeId, ProjectId, SkillId)
);
-- 문제: Employee는 독립적으로 여러 Project와 여러 Skill을 가질 수 있음

-- 4NF 적용 후
CREATE TABLE EmployeeProjects_4NF (
    EmployeeId INT,
    ProjectId INT,
    AssignmentDate DATE,
    Role NVARCHAR(50),
    PRIMARY KEY (EmployeeId, ProjectId)
);

CREATE TABLE EmployeeSkills_4NF (
    EmployeeId INT,
    SkillId INT,
    ProficiencyLevel INT,
    PRIMARY KEY (EmployeeId, SkillId)
);

-- 5NF 예제 (조인 종속)
-- 복잡한 비즈니스 규칙이 있는 경우
CREATE TABLE SupplierProductRegion (
    SupplierId INT,
    ProductId INT,
    RegionId INT,
    CanSupply BIT,
    PRIMARY KEY (SupplierId, ProductId, RegionId)
);
-- 특정 공급업체가 특정 지역에 특정 제품을 공급할 수 있는지 여부
```

## 3. 역정규화 (Denormalization)

### 3.1 역정규화 전략

```sql
-- 계산된 필드 추가
CREATE TABLE Orders_Denormalized (
    OrderId INT PRIMARY KEY,
    CustomerId INT,
    OrderDate DATETIME,
    -- 정규화된 형태에서는 OrderDetails에서 계산
    TotalAmount DECIMAL(10, 2), -- 역정규화: 미리 계산된 값 저장
    TotalItems INT, -- 역정규화: 주문 항목 수
    LastUpdated DATETIME DEFAULT GETDATE()
);

-- 트리거로 계산된 필드 유지
CREATE TRIGGER trg_UpdateOrderTotals
ON OrderDetails
AFTER INSERT, UPDATE, DELETE
AS
BEGIN
    UPDATE o
    SET 
        TotalAmount = (
            SELECT SUM(Quantity * UnitPrice * (1 - Discount))
            FROM OrderDetails
            WHERE OrderId = o.OrderId
        ),
        TotalItems = (
            SELECT COUNT(*)
            FROM OrderDetails
            WHERE OrderId = o.OrderId
        ),
        LastUpdated = GETDATE()
    FROM Orders_Denormalized o
    WHERE o.OrderId IN (
        SELECT OrderId FROM inserted
        UNION
        SELECT OrderId FROM deleted
    );
END;

-- 중복 데이터 저장
CREATE TABLE ProductsWithCategory (
    ProductId INT PRIMARY KEY,
    ProductName NVARCHAR(200),
    CategoryId INT,
    CategoryName NVARCHAR(100), -- 역정규화: Categories 테이블의 데이터 중복
    SupplierId INT,
    SupplierName NVARCHAR(100), -- 역정규화: Suppliers 테이블의 데이터 중복
    UnitPrice DECIMAL(10, 2),
    UnitsInStock INT
);

-- 집계 테이블
CREATE TABLE DailySalesSummary (
    SaleDate DATE PRIMARY KEY,
    TotalOrders INT,
    TotalRevenue DECIMAL(15, 2),
    TotalItems INT,
    UniqueCustomers INT,
    TopSellingProduct NVARCHAR(200),
    LastCalculated DATETIME DEFAULT GETDATE()
);

-- 집계 데이터 생성 프로시저
CREATE PROCEDURE sp_CalculateDailySales
    @Date DATE
AS
BEGIN
    DELETE FROM DailySalesSummary WHERE SaleDate = @Date;
    
    INSERT INTO DailySalesSummary 
        (SaleDate, TotalOrders, TotalRevenue, TotalItems, UniqueCustomers, TopSellingProduct)
    SELECT 
        @Date,
        COUNT(DISTINCT o.OrderId),
        SUM(od.Quantity * od.UnitPrice * (1 - od.Discount)),
        SUM(od.Quantity),
        COUNT(DISTINCT o.CustomerId),
        (
            SELECT TOP 1 p.ProductName
            FROM OrderDetails od2
            INNER JOIN Products p ON od2.ProductId = p.ProductId
            INNER JOIN Orders o2 ON od2.OrderId = o2.OrderId
            WHERE CAST(o2.OrderDate AS DATE) = @Date
            GROUP BY p.ProductId, p.ProductName
            ORDER BY SUM(od2.Quantity) DESC
        )
    FROM Orders o
    INNER JOIN OrderDetails od ON o.OrderId = od.OrderId
    WHERE CAST(o.OrderDate AS DATE) = @Date;
END;
```

### 3.2 수직/수평 분할

```sql
-- 수직 분할 (Vertical Partitioning)
-- 자주 사용되는 컬럼과 그렇지 않은 컬럼 분리
CREATE TABLE Products_Core (
    ProductId INT PRIMARY KEY,
    ProductName NVARCHAR(200) NOT NULL,
    CategoryId INT NOT NULL,
    UnitPrice DECIMAL(10, 2) NOT NULL,
    UnitsInStock INT DEFAULT 0,
    IsActive BIT DEFAULT 1
);

CREATE TABLE Products_Details (
    ProductId INT PRIMARY KEY,
    Description NVARCHAR(MAX),
    TechnicalSpecs XML,
    Image VARBINARY(MAX),
    LastReviewDate DATE,
    FOREIGN KEY (ProductId) REFERENCES Products_Core(ProductId)
);

-- 수평 분할 (Horizontal Partitioning)
-- 연도별 주문 테이블 분할
CREATE TABLE Orders_2023 (
    CHECK (YEAR(OrderDate) = 2023)
) INHERITS (Orders);

CREATE TABLE Orders_2024 (
    CHECK (YEAR(OrderDate) = 2024)
) INHERITS (Orders);

-- 파티션 뷰
CREATE VIEW Orders_All AS
SELECT * FROM Orders_2023
UNION ALL
SELECT * FROM Orders_2024;
```

## 4. 명명 규칙

### 4.1 테이블 명명 규칙

```sql
-- 권장 명명 규칙 예제

-- 1. 테이블명: PascalCase, 복수형
CREATE TABLE Customers (...);
CREATE TABLE OrderDetails (...);
CREATE TABLE ProductCategories (...);

-- 2. 컬럼명: PascalCase
CREATE TABLE Products (
    ProductId INT PRIMARY KEY,
    ProductName NVARCHAR(200),
    UnitPrice DECIMAL(10, 2),
    CreatedDate DATETIME,
    ModifiedBy NVARCHAR(100)
);

-- 3. 관계 테이블: 관련 테이블명 조합
CREATE TABLE CustomerOrders (...); -- Customers + Orders
CREATE TABLE EmployeeProjects (...); -- Employees + Projects

-- 4. 뷰 명명: v_ 접두사 또는 View 접미사
CREATE VIEW v_ActiveCustomers AS ...;
CREATE VIEW CustomerOrdersView AS ...;

-- 5. 저장 프로시저: sp_ 또는 usp_ 접두사
CREATE PROCEDURE sp_GetCustomerOrders ...;
CREATE PROCEDURE usp_UpdateInventory ...;

-- 6. 함수: fn_ 또는 uf_ 접두사
CREATE FUNCTION fn_CalculateDiscount(...);
CREATE FUNCTION uf_GetAge(...);

-- 7. 트리거: trg_ 접두사
CREATE TRIGGER trg_Orders_AfterInsert ...;
CREATE TRIGGER trg_Customers_InsteadOfDelete ...;

-- 8. 인덱스: IX_ 접두사
CREATE INDEX IX_Customers_Email ON Customers(Email);
CREATE INDEX IX_Orders_CustomerDate ON Orders(CustomerId, OrderDate);

-- 9. 제약조건
-- Primary Key: PK_테이블명
ALTER TABLE Customers ADD CONSTRAINT PK_Customers PRIMARY KEY (CustomerId);

-- Foreign Key: FK_자식테이블_부모테이블
ALTER TABLE Orders ADD CONSTRAINT FK_Orders_Customers 
    FOREIGN KEY (CustomerId) REFERENCES Customers(CustomerId);

-- Unique: UQ_테이블명_컬럼명
ALTER TABLE Customers ADD CONSTRAINT UQ_Customers_Email UNIQUE (Email);

-- Check: CHK_테이블명_설명
ALTER TABLE Products ADD CONSTRAINT CHK_Products_Price 
    CHECK (UnitPrice > 0);

-- Default: DF_테이블명_컬럼명
ALTER TABLE Orders ADD CONSTRAINT DF_Orders_OrderDate 
    DEFAULT GETDATE() FOR OrderDate;
```

### 4.2 일관성 있는 명명 패턴

```sql
-- 날짜/시간 컬럼
CREATE TABLE AuditLog (
    CreatedDate DATETIME DEFAULT GETDATE(),     -- 생성일
    ModifiedDate DATETIME,                      -- 수정일
    DeletedDate DATETIME,                       -- 삭제일
    StartDate DATE,                             -- 시작일
    EndDate DATE,                               -- 종료일
    LastAccessedDateTime DATETIME               -- 마지막 접근 일시
);

-- 상태/플래그 컬럼
CREATE TABLE Users (
    IsActive BIT DEFAULT 1,                     -- 활성 여부
    IsDeleted BIT DEFAULT 0,                    -- 삭제 여부
    HasVerifiedEmail BIT DEFAULT 0,             -- 이메일 인증 여부
    Status VARCHAR(20),                         -- 상태 (Active, Inactive, Suspended)
    UserType VARCHAR(20)                        -- 유형 (Admin, User, Guest)
);

-- ID/코드 컬럼
CREATE TABLE Products (
    ProductId INT IDENTITY(1,1),                -- 내부 ID
    ProductCode VARCHAR(20),                    -- 비즈니스 코드
    SKU VARCHAR(50),                           -- Stock Keeping Unit
    BarCode VARCHAR(50),                       -- 바코드
    ExternalId VARCHAR(100)                    -- 외부 시스템 ID
);

-- 수량/금액 컬럼
CREATE TABLE Inventory (
    QuantityOnHand INT,                        -- 재고 수량
    QuantityAvailable INT,                     -- 가용 수량
    ReorderQuantity INT,                       -- 재주문 수량
    UnitPrice DECIMAL(10, 2),                  -- 단가
    TotalAmount DECIMAL(15, 2),                -- 총액
    DiscountPercent DECIMAL(5, 2),             -- 할인율
    TaxAmount DECIMAL(10, 2)                   -- 세금
);
```

## 5. 실무 설계 예제

### 5.1 전자상거래 시스템 설계

```sql
-- 핵심 도메인 모델
-- 1. 사용자 관리
CREATE TABLE Users (
    UserId INT PRIMARY KEY IDENTITY(1,1),
    Username VARCHAR(50) UNIQUE NOT NULL,
    Email VARCHAR(255) UNIQUE NOT NULL,
    PasswordHash VARCHAR(255) NOT NULL,
    Salt VARCHAR(50) NOT NULL,
    CreatedDate DATETIME DEFAULT GETDATE(),
    LastLoginDate DATETIME,
    IsEmailVerified BIT DEFAULT 0,
    IsActive BIT DEFAULT 1,
    INDEX IX_Users_Email (Email),
    INDEX IX_Users_Username (Username)
);

-- 2. 상품 카탈로그
CREATE TABLE Categories (
    CategoryId INT PRIMARY KEY IDENTITY(1,1),
    ParentCategoryId INT NULL,
    CategoryName NVARCHAR(100) NOT NULL,
    CategoryPath NVARCHAR(500), -- /Electronics/Computers/Laptops
    DisplayOrder INT DEFAULT 0,
    IsActive BIT DEFAULT 1,
    FOREIGN KEY (ParentCategoryId) REFERENCES Categories(CategoryId),
    INDEX IX_Categories_Parent (ParentCategoryId),
    INDEX IX_Categories_Path (CategoryPath)
);

CREATE TABLE Products (
    ProductId INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName NVARCHAR(200) NOT NULL,
    Description NVARCHAR(MAX),
    CategoryId INT NOT NULL,
    BrandId INT,
    UnitPrice DECIMAL(10, 2) NOT NULL,
    CostPrice DECIMAL(10, 2),
    Weight DECIMAL(8, 3),
    Dimensions VARCHAR(50),
    StockQuantity INT DEFAULT 0,
    ReorderLevel INT DEFAULT 10,
    IsActive BIT DEFAULT 1,
    CreatedDate DATETIME DEFAULT GETDATE(),
    FOREIGN KEY (CategoryId) REFERENCES Categories(CategoryId),
    INDEX IX_Products_Category (CategoryId),
    INDEX IX_Products_SKU (SKU),
    FULLTEXT INDEX ON (ProductName, Description)
);

-- 3. 장바구니와 주문
CREATE TABLE ShoppingCarts (
    CartId UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    UserId INT NULL, -- NULL for anonymous users
    SessionId VARCHAR(100),
    CreatedDate DATETIME DEFAULT GETDATE(),
    UpdatedDate DATETIME DEFAULT GETDATE(),
    ExpiryDate DATETIME DEFAULT DATEADD(DAY, 7, GETDATE()),
    FOREIGN KEY (UserId) REFERENCES Users(UserId),
    INDEX IX_Carts_User (UserId),
    INDEX IX_Carts_Session (SessionId)
);

CREATE TABLE CartItems (
    CartId UNIQUEIDENTIFIER NOT NULL,
    ProductId INT NOT NULL,
    Quantity INT NOT NULL DEFAULT 1,
    AddedDate DATETIME DEFAULT GETDATE(),
    PRIMARY KEY (CartId, ProductId),
    FOREIGN KEY (CartId) REFERENCES ShoppingCarts(CartId) ON DELETE CASCADE,
    FOREIGN KEY (ProductId) REFERENCES Products(ProductId)
);

-- 4. 주문 처리
CREATE TABLE Orders (
    OrderId INT PRIMARY KEY IDENTITY(1,1),
    OrderNumber AS ('ORD' + RIGHT('000000' + CAST(OrderId AS VARCHAR(10)), 7)),
    UserId INT NOT NULL,
    OrderDate DATETIME DEFAULT GETDATE(),
    OrderStatus VARCHAR(20) DEFAULT 'Pending',
    PaymentStatus VARCHAR(20) DEFAULT 'Pending',
    ShippingAddressId INT,
    BillingAddressId INT,
    SubTotal DECIMAL(10, 2) NOT NULL,
    TaxAmount DECIMAL(10, 2) DEFAULT 0,
    ShippingAmount DECIMAL(10, 2) DEFAULT 0,
    DiscountAmount DECIMAL(10, 2) DEFAULT 0,
    TotalAmount AS (SubTotal + TaxAmount + ShippingAmount - DiscountAmount),
    Notes NVARCHAR(500),
    FOREIGN KEY (UserId) REFERENCES Users(UserId),
    INDEX IX_Orders_User (UserId),
    INDEX IX_Orders_Date (OrderDate),
    INDEX IX_Orders_Status (OrderStatus)
);

-- 5. 재고 추적
CREATE TABLE InventoryTransactions (
    TransactionId INT PRIMARY KEY IDENTITY(1,1),
    ProductId INT NOT NULL,
    TransactionType VARCHAR(20) NOT NULL, -- 'Purchase', 'Sale', 'Return', 'Adjustment'
    Quantity INT NOT NULL,
    ReferenceType VARCHAR(20), -- 'Order', 'Return', 'Manual'
    ReferenceId INT,
    TransactionDate DATETIME DEFAULT GETDATE(),
    Notes NVARCHAR(500),
    CreatedBy INT,
    FOREIGN KEY (ProductId) REFERENCES Products(ProductId),
    INDEX IX_Inventory_Product (ProductId),
    INDEX IX_Inventory_Date (TransactionDate)
);
```

### 5.2 설계 검증 쿼리

```sql
-- 정규화 검증
-- 함수 종속성 확인
WITH FunctionalDependencies AS (
    SELECT 
        'Orders->Customers' AS Dependency,
        COUNT(DISTINCT CustomerId) AS UniqueParents,
        COUNT(*) AS TotalRows,
        CASE 
            WHEN COUNT(DISTINCT CustomerId) = COUNT(*) THEN 'No Redundancy'
            ELSE 'Redundancy Detected'
        END AS Status
    FROM Orders
)
SELECT * FROM FunctionalDependencies;

-- 참조 무결성 확인
-- 고아 레코드 찾기
SELECT 'Orders without valid Customer' AS Issue, COUNT(*) AS Count
FROM Orders o
WHERE NOT EXISTS (SELECT 1 FROM Customers c WHERE c.CustomerId = o.CustomerId)
UNION ALL
SELECT 'OrderDetails without valid Order', COUNT(*)
FROM OrderDetails od
WHERE NOT EXISTS (SELECT 1 FROM Orders o WHERE o.OrderId = od.OrderId)
UNION ALL
SELECT 'Products without valid Category', COUNT(*)
FROM Products p
WHERE NOT EXISTS (SELECT 1 FROM Categories c WHERE c.CategoryId = p.CategoryId);

-- 역정규화 동기화 확인
-- 계산된 필드 검증
SELECT 
    o.OrderId,
    o.TotalAmount AS StoredTotal,
    SUM(od.Quantity * od.UnitPrice * (1 - od.Discount)) AS CalculatedTotal,
    o.TotalAmount - SUM(od.Quantity * od.UnitPrice * (1 - od.Discount)) AS Difference
FROM Orders o
INNER JOIN OrderDetails od ON o.OrderId = od.OrderId
GROUP BY o.OrderId, o.TotalAmount
HAVING ABS(o.TotalAmount - SUM(od.Quantity * od.UnitPrice * (1 - od.Discount))) > 0.01;
```

## 6. 설계 도구와 문서화

### 6.1 메타데이터 쿼리

```sql
-- 테이블 구조 문서화
SELECT 
    t.TABLE_SCHEMA,
    t.TABLE_NAME,
    t.TABLE_TYPE,
    ISNULL(ep.value, '') AS TableDescription
FROM INFORMATION_SCHEMA.TABLES t
LEFT JOIN sys.extended_properties ep 
    ON ep.major_id = OBJECT_ID(t.TABLE_SCHEMA + '.' + t.TABLE_NAME)
    AND ep.minor_id = 0
    AND ep.name = 'MS_Description'
WHERE t.TABLE_TYPE = 'BASE TABLE'
ORDER BY t.TABLE_SCHEMA, t.TABLE_NAME;

-- 컬럼 정보 추출
SELECT 
    c.TABLE_NAME,
    c.COLUMN_NAME,
    c.DATA_TYPE,
    c.CHARACTER_MAXIMUM_LENGTH,
    c.NUMERIC_PRECISION,
    c.NUMERIC_SCALE,
    c.IS_NULLABLE,
    c.COLUMN_DEFAULT,
    ISNULL(ep.value, '') AS ColumnDescription
FROM INFORMATION_SCHEMA.COLUMNS c
LEFT JOIN sys.extended_properties ep 
    ON ep.major_id = OBJECT_ID(c.TABLE_SCHEMA + '.' + c.TABLE_NAME)
    AND ep.minor_id = c.ORDINAL_POSITION
    AND ep.name = 'MS_Description'
WHERE c.TABLE_SCHEMA = 'dbo'
ORDER BY c.TABLE_NAME, c.ORDINAL_POSITION;

-- 관계 매핑
SELECT 
    fk.name AS ForeignKeyName,
    tp.name AS ParentTable,
    cp.name AS ParentColumn,
    tr.name AS ReferencedTable,
    cr.name AS ReferencedColumn,
    fk.delete_referential_action_desc AS DeleteAction,
    fk.update_referential_action_desc AS UpdateAction
FROM sys.foreign_keys fk
INNER JOIN sys.tables tp ON fk.parent_object_id = tp.object_id
INNER JOIN sys.tables tr ON fk.referenced_object_id = tr.object_id
INNER JOIN sys.foreign_key_columns fkc ON fkc.constraint_object_id = fk.object_id
INNER JOIN sys.columns cp ON fkc.parent_column_id = cp.column_id 
    AND fkc.parent_object_id = cp.object_id
INNER JOIN sys.columns cr ON fkc.referenced_column_id = cr.column_id 
    AND fkc.referenced_object_id = cr.object_id
ORDER BY tp.name, fk.name;
```

### 6.2 설계 문서 자동 생성

```sql
-- 데이터베이스 설계 문서 생성 프로시저
CREATE PROCEDURE sp_GenerateDesignDocument
AS
BEGIN
    -- ER 다이어그램용 정보
    SELECT 
        'Entity Relationship Diagram' AS Section,
        t.TABLE_NAME AS Entity,
        (
            SELECT COUNT(*) 
            FROM INFORMATION_SCHEMA.COLUMNS 
            WHERE TABLE_NAME = t.TABLE_NAME
        ) AS AttributeCount,
        (
            SELECT COUNT(*) 
            FROM INFORMATION_SCHEMA.TABLE_CONSTRAINTS 
            WHERE TABLE_NAME = t.TABLE_NAME 
            AND CONSTRAINT_TYPE = 'PRIMARY KEY'
        ) AS PrimaryKeys,
        (
            SELECT COUNT(*) 
            FROM INFORMATION_SCHEMA.TABLE_CONSTRAINTS 
            WHERE TABLE_NAME = t.TABLE_NAME 
            AND CONSTRAINT_TYPE = 'FOREIGN KEY'
        ) AS ForeignKeys
    FROM INFORMATION_SCHEMA.TABLES t
    WHERE t.TABLE_TYPE = 'BASE TABLE';
    
    -- 정규화 레벨 체크
    SELECT 
        'Normalization Check' AS Section,
        'Tables with potential redundancy' AS CheckType,
        t.TABLE_NAME,
        COUNT(*) AS ColumnCount,
        SUM(CASE WHEN c.DATA_TYPE IN ('nvarchar', 'varchar', 'text') THEN 1 ELSE 0 END) AS TextColumns,
        CASE 
            WHEN COUNT(*) > 15 THEN 'Consider vertical partitioning'
            WHEN SUM(CASE WHEN c.DATA_TYPE IN ('nvarchar', 'varchar', 'text') THEN 1 ELSE 0 END) > 10 
                THEN 'Many text columns - check for redundancy'
            ELSE 'OK'
        END AS Recommendation
    FROM INFORMATION_SCHEMA.TABLES t
    INNER JOIN INFORMATION_SCHEMA.COLUMNS c ON t.TABLE_NAME = c.TABLE_NAME
    WHERE t.TABLE_TYPE = 'BASE TABLE'
    GROUP BY t.TABLE_NAME
    HAVING COUNT(*) > 15 OR SUM(CASE WHEN c.DATA_TYPE IN ('nvarchar', 'varchar', 'text') THEN 1 ELSE 0 END) > 10;
END;
```

## 마무리

효과적인 데이터베이스 설계는 시스템의 성능, 확장성, 유지보수성을 결정하는 핵심 요소입니다. ER 모델링을 통한 개념적 설계, 정규화를 통한 데이터 무결성 확보, 그리고 성능을 위한 전략적 역정규화가 균형을 이루어야 합니다. 일관된 명명 규칙과 철저한 문서화는 장기적인 시스템 관리에 필수적입니다.