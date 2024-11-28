# 고급 Entity Framework Core와 성능 최적화

## EF Core 성능 진단

### 로깅 설정
```csharp
// appsettings.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.EntityFrameworkCore.Database.Command": "Information"
    }
  }
}

// Program.cs
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString)
           .LogTo(Console.WriteLine, LogLevel.Information)
           .EnableSensitiveDataLogging()  // 개발 환경에서만 사용
           .EnableDetailedErrors());
```

### 쿼리 태그
```csharp
// 쿼리에 태그를 추가하여 로그에서 쉽게 식별
var products = await context.Products
    .TagWith("GetActiveProducts - Called from ProductService.GetActiveAsync()")
    .Where(p => p.IsActive)
    .ToListAsync();

// SQL 로그 출력
-- GetActiveProducts - Called from ProductService.GetActiveAsync()
SELECT [p].[Id], [p].[Name], [p].[IsActive]
FROM [Products] AS [p]
WHERE [p].[IsActive] = 1
```

## 쿼리 성능 최적화

### Select 프로젝션
```csharp
// 나쁜 예: 전체 엔티티 로드
var products = await context.Products
    .Include(p => p.Category)
    .Include(p => p.Reviews)
    .ToListAsync();

// 좋은 예: 필요한 필드만 선택
var productDtos = await context.Products
    .Select(p => new ProductDto
    {
        Id = p.Id,
        Name = p.Name,
        Price = p.Price,
        CategoryName = p.Category.Name,
        ReviewCount = p.Reviews.Count()
    })
    .ToListAsync();
```

### 분할 쿼리 (Split Queries)
```csharp
// 단일 쿼리 (카테시안 곱 문제 발생 가능)
var blogs = context.Blogs
    .Include(b => b.Posts)
        .ThenInclude(p => p.Comments)
        .ThenInclude(c => c.Author)
    .ToList();

// 분할 쿼리 (여러 개의 쿼리로 실행)
var blogs = context.Blogs
    .AsSplitQuery()
    .Include(b => b.Posts)
        .ThenInclude(p => p.Comments)
        .ThenInclude(c => c.Author)
    .ToList();

// 전역 설정
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder.UseSqlServer(connectionString, 
        o => o.UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery));
}
```

### 추적 비활성화
```csharp
// 읽기 전용 쿼리에서 변경 추적 비활성화
var products = await context.Products
    .AsNoTracking()
    .Where(p => p.Price > 100)
    .ToListAsync();

// 전역 설정
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder.UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
}

// 특정 쿼리에서만 추적 활성화
var product = await context.Products
    .AsTracking()
    .FirstOrDefaultAsync(p => p.Id == id);
```

## N+1 문제 해결

### 즉시 로딩 (Eager Loading)
```csharp
// Include를 사용한 즉시 로딩
var orders = await context.Orders
    .Include(o => o.Customer)
    .Include(o => o.OrderItems)
        .ThenInclude(oi => oi.Product)
    .Where(o => o.OrderDate >= startDate)
    .ToListAsync();
```

### 명시적 로딩 (Explicit Loading)
```csharp
// 필요한 경우에만 관련 데이터 로드
var order = await context.Orders.FirstAsync(o => o.Id == orderId);

// 나중에 필요할 때 로드
await context.Entry(order)
    .Collection(o => o.OrderItems)
    .LoadAsync();

// 조건부 로드
await context.Entry(order)
    .Collection(o => o.OrderItems)
    .Query()
    .Where(oi => oi.Quantity > 10)
    .LoadAsync();
```

### 프로젝션을 통한 해결
```csharp
// N+1 문제를 피하는 프로젝션
var orderSummaries = await context.Orders
    .Select(o => new OrderSummaryDto
    {
        OrderId = o.Id,
        CustomerName = o.Customer.Name,
        TotalAmount = o.OrderItems.Sum(oi => oi.Quantity * oi.UnitPrice),
        ItemCount = o.OrderItems.Count()
    })
    .ToListAsync();
```

## 컴파일된 쿼리

### 기본 컴파일된 쿼리
```csharp
// 컴파일된 쿼리 정의
private static readonly Func<AppDbContext, int, Task<Product>> GetProductById =
    EF.CompileAsyncQuery((AppDbContext context, int id) =>
        context.Products.FirstOrDefault(p => p.Id == id));

// 사용
var product = await GetProductById(context, productId);
```

### 복잡한 컴파일된 쿼리
```csharp
private static readonly Func<AppDbContext, decimal, int, Task<List<ProductDto>>> 
    GetExpensiveProducts = EF.CompileAsyncQuery(
        (AppDbContext context, decimal minPrice, int categoryId) =>
            context.Products
                .Where(p => p.Price >= minPrice && p.CategoryId == categoryId)
                .Select(p => new ProductDto
                {
                    Id = p.Id,
                    Name = p.Name,
                    Price = p.Price
                })
                .ToList());

// 사용
var expensiveProducts = await GetExpensiveProducts(context, 100m, 5);
```

## 대량 작업 최적화

### 배치 삽입
```csharp
// 나쁜 예: 개별 삽입
foreach (var product in products)
{
    context.Products.Add(product);
    await context.SaveChangesAsync();  // 매번 데이터베이스 호출
}

// 좋은 예: 배치 삽입
context.Products.AddRange(products);
await context.SaveChangesAsync();  // 한 번의 데이터베이스 호출

// 더 나은 예: 청크 단위 처리
const int batchSize = 1000;
for (int i = 0; i < products.Count; i += batchSize)
{
    var batch = products.Skip(i).Take(batchSize);
    context.Products.AddRange(batch);
    await context.SaveChangesAsync();
}
```

### ExecuteUpdate와 ExecuteDelete
```csharp
// EF Core 7.0+: 대량 업데이트
await context.Products
    .Where(p => p.CategoryId == oldCategoryId)
    .ExecuteUpdateAsync(p => p
        .SetProperty(x => x.CategoryId, newCategoryId)
        .SetProperty(x => x.ModifiedDate, DateTime.UtcNow));

// 대량 삭제
await context.Products
    .Where(p => p.IsDeleted && p.DeletedDate < cutoffDate)
    .ExecuteDeleteAsync();

// 복잡한 업데이트
await context.Orders
    .Where(o => o.Status == OrderStatus.Pending && 
                o.OrderDate < DateTime.UtcNow.AddDays(-30))
    .ExecuteUpdateAsync(o => o
        .SetProperty(x => x.Status, OrderStatus.Cancelled)
        .SetProperty(x => x.CancelledDate, DateTime.UtcNow));
```

## 동시성 처리

### 낙관적 동시성 제어
```csharp
// 엔티티 설정
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
    
    [Timestamp]
    public byte[] RowVersion { get; set; }
}

// Fluent API 설정
modelBuilder.Entity<Product>()
    .Property(p => p.RowVersion)
    .IsRowVersion();

// 동시성 충돌 처리
try
{
    var product = await context.Products.FindAsync(id);
    product.Price = newPrice;
    await context.SaveChangesAsync();
}
catch (DbUpdateConcurrencyException ex)
{
    var entry = ex.Entries.Single();
    var databaseValues = await entry.GetDatabaseValuesAsync();
    
    if (databaseValues == null)
    {
        // 엔티티가 삭제됨
    }
    else
    {
        // 데이터베이스 값으로 업데이트
        entry.OriginalValues.SetValues(databaseValues);
        // 재시도 로직
    }
}
```

### 격리 수준 설정
```csharp
// 트랜잭션 격리 수준 설정
using var transaction = await context.Database
    .BeginTransactionAsync(IsolationLevel.ReadCommitted);

try
{
    // 작업 수행
    await context.SaveChangesAsync();
    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

## 인덱스 최적화

### 인덱스 설정
```csharp
// 단일 컬럼 인덱스
modelBuilder.Entity<Product>()
    .HasIndex(p => p.Name);

// 복합 인덱스
modelBuilder.Entity<Order>()
    .HasIndex(o => new { o.CustomerId, o.OrderDate })
    .HasDatabaseName("IX_Order_Customer_Date");

// 고유 인덱스
modelBuilder.Entity<User>()
    .HasIndex(u => u.Email)
    .IsUnique();

// 필터된 인덱스
modelBuilder.Entity<Product>()
    .HasIndex(p => p.Price)
    .HasFilter("[IsActive] = 1");

// Include 인덱스
modelBuilder.Entity<Product>()
    .HasIndex(p => p.CategoryId)
    .IncludeProperties(p => new { p.Name, p.Price });
```

### 인덱스 사용 확인
```csharp
// 실행 계획 확인을 위한 로깅
var products = await context.Products
    .TagWith("Check index usage")
    .Where(p => p.CategoryId == categoryId && p.Price > minPrice)
    .ToListAsync();

// SQL Server에서 실행 계획 확인
// SET STATISTICS IO ON
// SET STATISTICS TIME ON
```

## 연결 복원력

### 재시도 정책 설정
```csharp
// SQL Server 연결 복원력
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString,
        sqlServerOptionsAction: sqlOptions =>
        {
            sqlOptions.EnableRetryOnFailure(
                maxRetryCount: 5,
                maxRetryDelay: TimeSpan.FromSeconds(30),
                errorNumbersToAdd: null);
        }));

// 사용자 정의 실행 전략
public class CustomExecutionStrategy : ExecutionStrategy
{
    public CustomExecutionStrategy(DbContext context, int maxRetryCount, TimeSpan maxRetryDelay)
        : base(context, maxRetryCount, maxRetryDelay)
    {
    }

    protected override bool ShouldRetryOn(Exception exception)
    {
        // 재시도할 예외 조건 정의
        return exception is SqlException sqlException &&
               (sqlException.Number == 49918 || // 리소스 제한
                sqlException.Number == 49919 || // 리소스 제한
                sqlException.Number == 49920);  // 리소스 제한
    }
}
```

## 메모리 최적화

### DbContext 풀링
```csharp
// DbContext 풀 사용
builder.Services.AddDbContextPool<AppDbContext>(options =>
    options.UseSqlServer(connectionString), 
    poolSize: 128);

// 또는 Factory 패턴
builder.Services.AddPooledDbContextFactory<AppDbContext>(options =>
    options.UseSqlServer(connectionString), 
    poolSize: 128);

// Factory 사용
public class ProductService
{
    private readonly IDbContextFactory<AppDbContext> _contextFactory;

    public ProductService(IDbContextFactory<AppDbContext> contextFactory)
    {
        _contextFactory = contextFactory;
    }

    public async Task<List<Product>> GetProductsAsync()
    {
        using var context = await _contextFactory.CreateDbContextAsync();
        return await context.Products.ToListAsync();
    }
}
```

### 메모리 효율적인 스트리밍
```csharp
// 대량 데이터 스트리밍
public async IAsyncEnumerable<ProductDto> GetAllProductsStreamAsync()
{
    await using var context = await _contextFactory.CreateDbContextAsync();
    
    await foreach (var product in context.Products
        .AsNoTracking()
        .AsAsyncEnumerable())
    {
        yield return new ProductDto
        {
            Id = product.Id,
            Name = product.Name,
            Price = product.Price
        };
    }
}
```

## 고급 매핑 기법

### 테이블 분할
```csharp
// 하나의 테이블을 여러 엔티티로 분할
modelBuilder.Entity<Order>()
    .ToTable("Orders");

modelBuilder.Entity<OrderDetail>()
    .ToTable("Orders");

modelBuilder.Entity<Order>()
    .HasOne(o => o.Detail)
    .WithOne()
    .HasForeignKey<OrderDetail>(d => d.Id);
```

### 엔티티 분할
```csharp
// 하나의 엔티티를 여러 테이블로 분할
modelBuilder.Entity<Customer>()
    .ToTable("Customers")
    .SplitToTable("CustomerDetails", tableBuilder =>
    {
        tableBuilder.Property(c => c.Phone);
        tableBuilder.Property(c => c.Address);
    });
```

### 소유 타입
```csharp
// 값 객체 매핑
modelBuilder.Entity<Order>()
    .OwnsOne(o => o.ShippingAddress, sa =>
    {
        sa.Property(a => a.Street).HasColumnName("ShippingStreet");
        sa.Property(a => a.City).HasColumnName("ShippingCity");
        sa.Property(a => a.ZipCode).HasColumnName("ShippingZipCode");
    });

// 소유 타입 컬렉션
modelBuilder.Entity<Customer>()
    .OwnsMany(c => c.Addresses, a =>
    {
        a.ToTable("CustomerAddresses");
        a.WithOwner().HasForeignKey("CustomerId");
        a.Property<int>("Id");
        a.HasKey("Id");
    });
```

## 프로파일링과 모니터링

### Application Insights 통합
```csharp
// Application Insights 설정
builder.Services.AddApplicationInsightsTelemetry();
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString)
           .UseApplicationServiceProvider(serviceProvider));
```

### 사용자 정의 인터셉터
```csharp
public class PerformanceInterceptor : DbCommandInterceptor
{
    private readonly ILogger<PerformanceInterceptor> _logger;
    private readonly DiagnosticSource _diagnosticSource;

    public PerformanceInterceptor(
        ILogger<PerformanceInterceptor> logger,
        DiagnosticSource diagnosticSource)
    {
        _logger = logger;
        _diagnosticSource = diagnosticSource;
    }

    public override async ValueTask<DbDataReader> ReaderExecutedAsync(
        DbCommand command,
        CommandExecutedEventData eventData,
        DbDataReader result,
        CancellationToken cancellationToken = default)
    {
        if (eventData.Duration.TotalMilliseconds > 100)
        {
            _logger.LogWarning(
                "Slow query detected ({Duration}ms): {CommandText}",
                eventData.Duration.TotalMilliseconds,
                command.CommandText);
        }

        return result;
    }
}

// 인터셉터 등록
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString)
           .AddInterceptors(new PerformanceInterceptor(logger, diagnosticSource)));
```

## 모범 사례

### DbContext 수명 관리
```csharp
// 짧은 수명 유지
public async Task<Product> GetProductAsync(int id)
{
    using var context = await _contextFactory.CreateDbContextAsync();
    return await context.Products.FindAsync(id);
}

// 작업 단위별 컨텍스트
public async Task ProcessOrderAsync(OrderDto orderDto)
{
    using var context = await _contextFactory.CreateDbContextAsync();
    using var transaction = await context.Database.BeginTransactionAsync();
    
    try
    {
        // 모든 작업을 하나의 컨텍스트에서 수행
        var order = CreateOrder(orderDto);
        context.Orders.Add(order);
        
        await UpdateInventoryAsync(context, order);
        await SendNotificationAsync(order);
        
        await context.SaveChangesAsync();
        await transaction.CommitAsync();
    }
    catch
    {
        await transaction.RollbackAsync();
        throw;
    }
}
```

### 쿼리 필터
```csharp
// 전역 쿼리 필터
modelBuilder.Entity<Product>()
    .HasQueryFilter(p => !p.IsDeleted);

modelBuilder.Entity<TenantEntity>()
    .HasQueryFilter(e => e.TenantId == _currentTenantId);

// 필터 무시
var allProducts = await context.Products
    .IgnoreQueryFilters()
    .ToListAsync();
```