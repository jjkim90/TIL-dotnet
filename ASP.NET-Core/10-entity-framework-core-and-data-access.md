# Entity Framework Core와 데이터 액세스

## Entity Framework Core 개요

Entity Framework Core (EF Core)는 .NET용 현대적인 객체-관계 매핑(ORM) 프레임워크입니다. 데이터베이스와의 상호작용을 추상화하여 개발자가 .NET 객체를 사용해 데이터를 다룰 수 있게 합니다.

### 주요 특징
- **크로스 플랫폼**: Windows, Linux, macOS에서 실행
- **다양한 데이터베이스 지원**: SQL Server, PostgreSQL, MySQL, SQLite 등
- **LINQ 쿼리**: 강력한 타입 안전 쿼리
- **마이그레이션**: 스키마 버전 관리
- **성능 최적화**: 쿼리 최적화와 캐싱

## EF Core 설정

### 패키지 설치
```bash
# SQL Server
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Design

# PostgreSQL
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL

# MySQL
dotnet add package Pomelo.EntityFrameworkCore.MySql

# 도구
dotnet tool install --global dotnet-ef
```

### DbContext 구성
```csharp
using Microsoft.EntityFrameworkCore;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options)
        : base(options)
    {
    }
    
    public DbSet<Product> Products { get; set; }
    public DbSet<Category> Categories { get; set; }
    public DbSet<Order> Orders { get; set; }
    public DbSet<OrderItem> OrderItems { get; set; }
    public DbSet<Customer> Customers { get; set; }
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        
        // Fluent API 구성
        modelBuilder.Entity<Product>(entity =>
        {
            entity.ToTable("Products");
            entity.HasKey(p => p.Id);
            entity.Property(p => p.Name).IsRequired().HasMaxLength(100);
            entity.Property(p => p.Price).HasPrecision(18, 2);
            entity.HasIndex(p => p.SKU).IsUnique();
        });
        
        // 관계 구성
        modelBuilder.Entity<OrderItem>()
            .HasOne(oi => oi.Order)
            .WithMany(o => o.OrderItems)
            .HasForeignKey(oi => oi.OrderId)
            .OnDelete(DeleteBehavior.Cascade);
    }
}

// Program.cs에서 등록
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));
```

## 엔티티 모델링

### 기본 엔티티
```csharp
public class Product
{
    public int Id { get; set; }
    
    [Required]
    [StringLength(100)]
    public string Name { get; set; }
    
    [StringLength(500)]
    public string Description { get; set; }
    
    [Column(TypeName = "decimal(18,2)")]
    public decimal Price { get; set; }
    
    [Required]
    [StringLength(50)]
    public string SKU { get; set; }
    
    public int Stock { get; set; }
    
    // 네비게이션 속성
    public int CategoryId { get; set; }
    public Category Category { get; set; }
    
    // 컬렉션 네비게이션
    public ICollection<OrderItem> OrderItems { get; set; }
    
    // 감사 필드
    public DateTime CreatedAt { get; set; }
    public DateTime? UpdatedAt { get; set; }
}

public class Category
{
    public int Id { get; set; }
    
    [Required]
    [StringLength(50)]
    public string Name { get; set; }
    
    public string Description { get; set; }
    
    // 네비게이션 속성
    public ICollection<Product> Products { get; set; }
}
```

### 복잡한 관계
```csharp
// 일대다 관계
public class Order
{
    public int Id { get; set; }
    public string OrderNumber { get; set; }
    public DateTime OrderDate { get; set; }
    public decimal TotalAmount { get; set; }
    public OrderStatus Status { get; set; }
    
    // 외래 키
    public int CustomerId { get; set; }
    public Customer Customer { get; set; }
    
    // 컬렉션 네비게이션
    public ICollection<OrderItem> OrderItems { get; set; }
}

public class OrderItem
{
    public int Id { get; set; }
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
    public decimal Discount { get; set; }
    
    // 복합 관계
    public int OrderId { get; set; }
    public Order Order { get; set; }
    
    public int ProductId { get; set; }
    public Product Product { get; set; }
}

// 다대다 관계 (명시적 조인 테이블)
public class Student
{
    public int Id { get; set; }
    public string Name { get; set; }
    
    public ICollection<StudentCourse> StudentCourses { get; set; }
}

public class Course
{
    public int Id { get; set; }
    public string Title { get; set; }
    
    public ICollection<StudentCourse> StudentCourses { get; set; }
}

public class StudentCourse
{
    public int StudentId { get; set; }
    public Student Student { get; set; }
    
    public int CourseId { get; set; }
    public Course Course { get; set; }
    
    public DateTime EnrollmentDate { get; set; }
    public Grade? Grade { get; set; }
}
```

## 리포지토리 패턴

### 제네릭 리포지토리
```csharp
public interface IRepository<T> where T : class
{
    Task<T> GetByIdAsync(int id);
    Task<IEnumerable<T>> GetAllAsync();
    Task<IEnumerable<T>> FindAsync(Expression<Func<T, bool>> predicate);
    Task<T> SingleOrDefaultAsync(Expression<Func<T, bool>> predicate);
    Task AddAsync(T entity);
    Task AddRangeAsync(IEnumerable<T> entities);
    void Update(T entity);
    void Remove(T entity);
    void RemoveRange(IEnumerable<T> entities);
}

public class Repository<T> : IRepository<T> where T : class
{
    protected readonly AppDbContext _context;
    protected readonly DbSet<T> _dbSet;
    
    public Repository(AppDbContext context)
    {
        _context = context;
        _dbSet = context.Set<T>();
    }
    
    public async Task<T> GetByIdAsync(int id)
    {
        return await _dbSet.FindAsync(id);
    }
    
    public async Task<IEnumerable<T>> GetAllAsync()
    {
        return await _dbSet.ToListAsync();
    }
    
    public async Task<IEnumerable<T>> FindAsync(Expression<Func<T, bool>> predicate)
    {
        return await _dbSet.Where(predicate).ToListAsync();
    }
    
    public async Task<T> SingleOrDefaultAsync(Expression<Func<T, bool>> predicate)
    {
        return await _dbSet.SingleOrDefaultAsync(predicate);
    }
    
    public async Task AddAsync(T entity)
    {
        await _dbSet.AddAsync(entity);
    }
    
    public async Task AddRangeAsync(IEnumerable<T> entities)
    {
        await _dbSet.AddRangeAsync(entities);
    }
    
    public void Update(T entity)
    {
        _dbSet.Update(entity);
    }
    
    public void Remove(T entity)
    {
        _dbSet.Remove(entity);
    }
    
    public void RemoveRange(IEnumerable<T> entities)
    {
        _dbSet.RemoveRange(entities);
    }
}
```

### 특화된 리포지토리
```csharp
public interface IProductRepository : IRepository<Product>
{
    Task<IEnumerable<Product>> GetProductsByCategoryAsync(int categoryId);
    Task<IEnumerable<Product>> GetTopSellingProductsAsync(int count);
    Task<Product> GetProductWithDetailsAsync(int id);
    Task<bool> IsSkuUniqueAsync(string sku, int? excludeProductId = null);
}

public class ProductRepository : Repository<Product>, IProductRepository
{
    public ProductRepository(AppDbContext context) : base(context)
    {
    }
    
    public async Task<IEnumerable<Product>> GetProductsByCategoryAsync(int categoryId)
    {
        return await _context.Products
            .Include(p => p.Category)
            .Where(p => p.CategoryId == categoryId)
            .OrderBy(p => p.Name)
            .ToListAsync();
    }
    
    public async Task<IEnumerable<Product>> GetTopSellingProductsAsync(int count)
    {
        return await _context.Products
            .Include(p => p.OrderItems)
            .OrderByDescending(p => p.OrderItems.Sum(oi => oi.Quantity))
            .Take(count)
            .ToListAsync();
    }
    
    public async Task<Product> GetProductWithDetailsAsync(int id)
    {
        return await _context.Products
            .Include(p => p.Category)
            .Include(p => p.OrderItems)
                .ThenInclude(oi => oi.Order)
            .FirstOrDefaultAsync(p => p.Id == id);
    }
    
    public async Task<bool> IsSkuUniqueAsync(string sku, int? excludeProductId = null)
    {
        var query = _context.Products.Where(p => p.SKU == sku);
        
        if (excludeProductId.HasValue)
        {
            query = query.Where(p => p.Id != excludeProductId.Value);
        }
        
        return !await query.AnyAsync();
    }
}
```

## Unit of Work 패턴

### Unit of Work 구현
```csharp
public interface IUnitOfWork : IDisposable
{
    IProductRepository Products { get; }
    ICategoryRepository Categories { get; }
    IOrderRepository Orders { get; }
    ICustomerRepository Customers { get; }
    
    Task<int> SaveChangesAsync();
    Task BeginTransactionAsync();
    Task CommitTransactionAsync();
    Task RollbackTransactionAsync();
}

public class UnitOfWork : IUnitOfWork
{
    private readonly AppDbContext _context;
    private IDbContextTransaction _transaction;
    
    // 리포지토리 인스턴스
    private IProductRepository _products;
    private ICategoryRepository _categories;
    private IOrderRepository _orders;
    private ICustomerRepository _customers;
    
    public UnitOfWork(AppDbContext context)
    {
        _context = context;
    }
    
    public IProductRepository Products => 
        _products ??= new ProductRepository(_context);
    
    public ICategoryRepository Categories => 
        _categories ??= new CategoryRepository(_context);
    
    public IOrderRepository Orders => 
        _orders ??= new OrderRepository(_context);
    
    public ICustomerRepository Customers => 
        _customers ??= new CustomerRepository(_context);
    
    public async Task<int> SaveChangesAsync()
    {
        return await _context.SaveChangesAsync();
    }
    
    public async Task BeginTransactionAsync()
    {
        _transaction = await _context.Database.BeginTransactionAsync();
    }
    
    public async Task CommitTransactionAsync()
    {
        try
        {
            await SaveChangesAsync();
            await _transaction?.CommitAsync();
        }
        catch
        {
            await RollbackTransactionAsync();
            throw;
        }
        finally
        {
            _transaction?.Dispose();
            _transaction = null;
        }
    }
    
    public async Task RollbackTransactionAsync()
    {
        await _transaction?.RollbackAsync();
        _transaction?.Dispose();
        _transaction = null;
    }
    
    public void Dispose()
    {
        _transaction?.Dispose();
        _context?.Dispose();
    }
}

// Program.cs에서 등록
builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();
```

## LINQ 쿼리

### 기본 쿼리 작업
```csharp
public class ProductService
{
    private readonly AppDbContext _context;
    
    public ProductService(AppDbContext context)
    {
        _context = context;
    }
    
    // 필터링과 정렬
    public async Task<List<Product>> GetProductsAsync(
        string searchTerm, 
        decimal? minPrice, 
        decimal? maxPrice,
        string sortBy = "name")
    {
        var query = _context.Products.AsQueryable();
        
        // 필터링
        if (!string.IsNullOrEmpty(searchTerm))
        {
            query = query.Where(p => p.Name.Contains(searchTerm) || 
                                   p.Description.Contains(searchTerm));
        }
        
        if (minPrice.HasValue)
        {
            query = query.Where(p => p.Price >= minPrice.Value);
        }
        
        if (maxPrice.HasValue)
        {
            query = query.Where(p => p.Price <= maxPrice.Value);
        }
        
        // 정렬
        query = sortBy?.ToLower() switch
        {
            "price" => query.OrderBy(p => p.Price),
            "price_desc" => query.OrderByDescending(p => p.Price),
            "name_desc" => query.OrderByDescending(p => p.Name),
            _ => query.OrderBy(p => p.Name)
        };
        
        return await query.ToListAsync();
    }
    
    // 그룹화와 집계
    public async Task<List<CategorySummary>> GetCategorySummaryAsync()
    {
        return await _context.Categories
            .Select(c => new CategorySummary
            {
                CategoryId = c.Id,
                CategoryName = c.Name,
                ProductCount = c.Products.Count(),
                TotalValue = c.Products.Sum(p => p.Price * p.Stock),
                AveragePrice = c.Products.Average(p => p.Price)
            })
            .ToListAsync();
    }
    
    // 조인 쿼리
    public async Task<List<OrderDetailDto>> GetOrderDetailsAsync(int orderId)
    {
        return await _context.OrderItems
            .Where(oi => oi.OrderId == orderId)
            .Select(oi => new OrderDetailDto
            {
                ProductName = oi.Product.Name,
                Quantity = oi.Quantity,
                UnitPrice = oi.UnitPrice,
                Discount = oi.Discount,
                Total = oi.Quantity * oi.UnitPrice * (1 - oi.Discount)
            })
            .ToListAsync();
    }
}
```

### 고급 쿼리 기법
```csharp
public class AdvancedQueryService
{
    private readonly AppDbContext _context;
    
    // 페이징
    public async Task<PagedResult<Product>> GetPagedProductsAsync(
        int page, 
        int pageSize)
    {
        var query = _context.Products.AsQueryable();
        
        var totalCount = await query.CountAsync();
        var items = await query
            .Skip((page - 1) * pageSize)
            .Take(pageSize)
            .ToListAsync();
        
        return new PagedResult<Product>
        {
            Items = items,
            TotalCount = totalCount,
            Page = page,
            PageSize = pageSize,
            TotalPages = (int)Math.Ceiling(totalCount / (double)pageSize)
        };
    }
    
    // 프로젝션
    public async Task<List<ProductListDto>> GetProductListAsync()
    {
        return await _context.Products
            .Select(p => new ProductListDto
            {
                Id = p.Id,
                Name = p.Name,
                Price = p.Price,
                CategoryName = p.Category.Name,
                InStock = p.Stock > 0
            })
            .ToListAsync();
    }
    
    // Include와 ThenInclude
    public async Task<Order> GetOrderWithDetailsAsync(int orderId)
    {
        return await _context.Orders
            .Include(o => o.Customer)
            .Include(o => o.OrderItems)
                .ThenInclude(oi => oi.Product)
                    .ThenInclude(p => p.Category)
            .FirstOrDefaultAsync(o => o.Id == orderId);
    }
    
    // Raw SQL
    public async Task<List<Product>> GetProductsByRawSqlAsync(int categoryId)
    {
        return await _context.Products
            .FromSqlRaw("SELECT * FROM Products WHERE CategoryId = {0}", categoryId)
            .ToListAsync();
    }
    
    // 저장 프로시저 호출
    public async Task<List<SalesReport>> GetSalesReportAsync(
        DateTime startDate, 
        DateTime endDate)
    {
        return await _context.Set<SalesReport>()
            .FromSqlRaw("EXEC GetSalesReport @StartDate = {0}, @EndDate = {1}", 
                startDate, endDate)
            .ToListAsync();
    }
}
```

## 마이그레이션

### 마이그레이션 명령어
```bash
# 마이그레이션 추가
dotnet ef migrations add InitialCreate

# 데이터베이스 업데이트
dotnet ef database update

# 마이그레이션 제거
dotnet ef migrations remove

# SQL 스크립트 생성
dotnet ef migrations script

# 특정 마이그레이션으로 롤백
dotnet ef database update PreviousMigration
```

### 프로그래밍 방식 마이그레이션
```csharp
// Program.cs
var app = builder.Build();

// 자동 마이그레이션 적용
using (var scope = app.Services.CreateScope())
{
    var dbContext = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    await dbContext.Database.MigrateAsync();
}

// 커스텀 마이그레이션 코드
public partial class AddProductIndex : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.CreateIndex(
            name: "IX_Products_SKU",
            table: "Products",
            column: "SKU",
            unique: true);
        
        // 데이터 시드
        migrationBuilder.InsertData(
            table: "Categories",
            columns: new[] { "Id", "Name", "Description" },
            values: new object[] { 1, "Electronics", "Electronic products" });
    }
    
    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropIndex(
            name: "IX_Products_SKU",
            table: "Products");
        
        migrationBuilder.DeleteData(
            table: "Categories",
            keyColumn: "Id",
            keyValue: 1);
    }
}
```

## 성능 최적화

### 쿼리 최적화
```csharp
public class OptimizedDataService
{
    private readonly AppDbContext _context;
    
    // 비추적 쿼리
    public async Task<List<ProductReadDto>> GetProductsForDisplayAsync()
    {
        return await _context.Products
            .AsNoTracking()
            .Select(p => new ProductReadDto
            {
                Id = p.Id,
                Name = p.Name,
                Price = p.Price
            })
            .ToListAsync();
    }
    
    // 분할 쿼리
    public async Task<List<Order>> GetOrdersWithSplitQueryAsync()
    {
        return await _context.Orders
            .AsSplitQuery()
            .Include(o => o.OrderItems)
            .ThenInclude(oi => oi.Product)
            .ToListAsync();
    }
    
    // 컴파일된 쿼리
    private static readonly Func<AppDbContext, int, Task<Product>> GetProductByIdQuery =
        EF.CompileAsyncQuery((AppDbContext context, int id) =>
            context.Products.FirstOrDefault(p => p.Id == id));
    
    public async Task<Product> GetProductByIdCompiledAsync(int id)
    {
        return await GetProductByIdQuery(_context, id);
    }
    
    // 배치 작업
    public async Task BulkUpdatePricesAsync(decimal percentage)
    {
        await _context.Database.ExecuteSqlRawAsync(
            "UPDATE Products SET Price = Price * {0} WHERE CategoryId = {1}",
            1 + percentage / 100, 1);
    }
    
    // 글로벌 쿼리 필터
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // 소프트 삭제 필터
        modelBuilder.Entity<Product>()
            .HasQueryFilter(p => !p.IsDeleted);
        
        // 다중 테넌트 필터
        modelBuilder.Entity<Order>()
            .HasQueryFilter(o => o.TenantId == _currentTenantId);
    }
}
```

### 캐싱 전략
```csharp
public class CachedProductService
{
    private readonly IMemoryCache _cache;
    private readonly AppDbContext _context;
    
    public CachedProductService(IMemoryCache cache, AppDbContext context)
    {
        _cache = cache;
        _context = context;
    }
    
    public async Task<Product> GetProductAsync(int id)
    {
        var cacheKey = $"product_{id}";
        
        if (!_cache.TryGetValue(cacheKey, out Product product))
        {
            product = await _context.Products
                .Include(p => p.Category)
                .FirstOrDefaultAsync(p => p.Id == id);
            
            if (product != null)
            {
                var cacheOptions = new MemoryCacheEntryOptions()
                    .SetSlidingExpiration(TimeSpan.FromMinutes(5))
                    .SetAbsoluteExpiration(TimeSpan.FromHours(1));
                
                _cache.Set(cacheKey, product, cacheOptions);
            }
        }
        
        return product;
    }
    
    public async Task InvalidateProductCacheAsync(int id)
    {
        _cache.Remove($"product_{id}");
    }
}
```

## 실전 예제: 완전한 데이터 액세스 레이어

```csharp
// 엔티티
public abstract class BaseEntity
{
    public int Id { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? UpdatedAt { get; set; }
    public bool IsDeleted { get; set; }
}

public class Product : BaseEntity
{
    public string Name { get; set; }
    public string Description { get; set; }
    public decimal Price { get; set; }
    public string SKU { get; set; }
    public int Stock { get; set; }
    public int CategoryId { get; set; }
    
    public virtual Category Category { get; set; }
    public virtual ICollection<OrderItem> OrderItems { get; set; }
}

// 서비스 계층
public interface IProductService
{
    Task<PagedResult<ProductDto>> GetProductsAsync(ProductFilterDto filter);
    Task<ProductDto> GetProductByIdAsync(int id);
    Task<ProductDto> CreateProductAsync(CreateProductDto dto);
    Task<ProductDto> UpdateProductAsync(int id, UpdateProductDto dto);
    Task<bool> DeleteProductAsync(int id);
}

public class ProductService : IProductService
{
    private readonly IUnitOfWork _unitOfWork;
    private readonly IMapper _mapper;
    private readonly ILogger<ProductService> _logger;
    
    public ProductService(
        IUnitOfWork unitOfWork, 
        IMapper mapper,
        ILogger<ProductService> logger)
    {
        _unitOfWork = unitOfWork;
        _mapper = mapper;
        _logger = logger;
    }
    
    public async Task<PagedResult<ProductDto>> GetProductsAsync(ProductFilterDto filter)
    {
        var query = _unitOfWork.Products.Query();
        
        // 필터 적용
        if (!string.IsNullOrEmpty(filter.SearchTerm))
        {
            query = query.Where(p => p.Name.Contains(filter.SearchTerm) ||
                                   p.Description.Contains(filter.SearchTerm));
        }
        
        if (filter.CategoryId.HasValue)
        {
            query = query.Where(p => p.CategoryId == filter.CategoryId.Value);
        }
        
        if (filter.MinPrice.HasValue)
        {
            query = query.Where(p => p.Price >= filter.MinPrice.Value);
        }
        
        if (filter.MaxPrice.HasValue)
        {
            query = query.Where(p => p.Price <= filter.MaxPrice.Value);
        }
        
        // 정렬
        query = filter.SortBy?.ToLower() switch
        {
            "price" => filter.SortDescending 
                ? query.OrderByDescending(p => p.Price) 
                : query.OrderBy(p => p.Price),
            "name" => filter.SortDescending 
                ? query.OrderByDescending(p => p.Name) 
                : query.OrderBy(p => p.Name),
            _ => query.OrderBy(p => p.Id)
        };
        
        // 페이징
        var pagedResult = await _unitOfWork.Products
            .GetPagedAsync(query, filter.Page, filter.PageSize);
        
        return new PagedResult<ProductDto>
        {
            Items = _mapper.Map<IEnumerable<ProductDto>>(pagedResult.Items),
            TotalCount = pagedResult.TotalCount,
            Page = pagedResult.Page,
            PageSize = pagedResult.PageSize,
            TotalPages = pagedResult.TotalPages
        };
    }
    
    public async Task<ProductDto> CreateProductAsync(CreateProductDto dto)
    {
        // 검증
        if (!await _unitOfWork.Products.IsSkuUniqueAsync(dto.SKU))
        {
            throw new BusinessException("SKU already exists");
        }
        
        var product = _mapper.Map<Product>(dto);
        product.CreatedAt = DateTime.UtcNow;
        
        await _unitOfWork.Products.AddAsync(product);
        await _unitOfWork.SaveChangesAsync();
        
        _logger.LogInformation("Product created with ID: {ProductId}", product.Id);
        
        return _mapper.Map<ProductDto>(product);
    }
    
    public async Task<ProductDto> UpdateProductAsync(int id, UpdateProductDto dto)
    {
        var product = await _unitOfWork.Products.GetByIdAsync(id);
        
        if (product == null)
        {
            throw new NotFoundException($"Product with ID {id} not found");
        }
        
        // SKU 중복 확인
        if (dto.SKU != product.SKU && 
            !await _unitOfWork.Products.IsSkuUniqueAsync(dto.SKU, id))
        {
            throw new BusinessException("SKU already exists");
        }
        
        _mapper.Map(dto, product);
        product.UpdatedAt = DateTime.UtcNow;
        
        _unitOfWork.Products.Update(product);
        await _unitOfWork.SaveChangesAsync();
        
        _logger.LogInformation("Product updated with ID: {ProductId}", id);
        
        return _mapper.Map<ProductDto>(product);
    }
    
    public async Task<bool> DeleteProductAsync(int id)
    {
        var product = await _unitOfWork.Products.GetByIdAsync(id);
        
        if (product == null)
        {
            return false;
        }
        
        // 소프트 삭제
        product.IsDeleted = true;
        product.UpdatedAt = DateTime.UtcNow;
        
        _unitOfWork.Products.Update(product);
        await _unitOfWork.SaveChangesAsync();
        
        _logger.LogInformation("Product soft deleted with ID: {ProductId}", id);
        
        return true;
    }
}

// 컨트롤러
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _productService;
    
    public ProductsController(IProductService productService)
    {
        _productService = productService;
    }
    
    [HttpGet]
    public async Task<ActionResult<PagedResult<ProductDto>>> GetProducts(
        [FromQuery] ProductFilterDto filter)
    {
        var result = await _productService.GetProductsAsync(filter);
        return Ok(result);
    }
    
    [HttpGet("{id}")]
    public async Task<ActionResult<ProductDto>> GetProduct(int id)
    {
        var product = await _productService.GetProductByIdAsync(id);
        return product != null ? Ok(product) : NotFound();
    }
    
    [HttpPost]
    public async Task<ActionResult<ProductDto>> CreateProduct(
        [FromBody] CreateProductDto dto)
    {
        var product = await _productService.CreateProductAsync(dto);
        return CreatedAtAction(nameof(GetProduct), new { id = product.Id }, product);
    }
    
    [HttpPut("{id}")]
    public async Task<ActionResult<ProductDto>> UpdateProduct(
        int id, 
        [FromBody] UpdateProductDto dto)
    {
        var product = await _productService.UpdateProductAsync(id, dto);
        return Ok(product);
    }
    
    [HttpDelete("{id}")]
    public async Task<IActionResult> DeleteProduct(int id)
    {
        var deleted = await _productService.DeleteProductAsync(id);
        return deleted ? NoContent() : NotFound();
    }
}
```