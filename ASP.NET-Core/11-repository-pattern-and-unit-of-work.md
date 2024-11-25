# Repository 패턴과 Unit of Work

## Repository 패턴 개요

Repository 패턴은 도메인 모델과 데이터 액세스 레이어 사이의 중간 계층 역할을 하는 디자인 패턴입니다. 데이터 액세스 로직을 캡슐화하여 비즈니스 로직과 분리합니다.

### 주요 이점
- **추상화**: 데이터 액세스 로직을 비즈니스 로직에서 분리
- **테스트 용이성**: Mock 객체를 사용한 단위 테스트 가능
- **유연성**: 데이터 소스 변경 시 Repository만 수정
- **재사용성**: 공통 데이터 액세스 로직 재사용

## Generic Repository 구현

### Repository Interface 정의
```csharp
public interface IRepository<T> where T : class
{
    Task<T> GetByIdAsync(int id);
    Task<IEnumerable<T>> GetAllAsync();
    Task<IEnumerable<T>> FindAsync(Expression<Func<T, bool>> predicate);
    Task AddAsync(T entity);
    void Update(T entity);
    void Remove(T entity);
}
```

### Generic Repository 구현
```csharp
public class Repository<T> : IRepository<T> where T : class
{
    protected readonly ApplicationDbContext _context;
    protected readonly DbSet<T> _dbSet;

    public Repository(ApplicationDbContext context)
    {
        _context = context;
        _dbSet = context.Set<T>();
    }

    public virtual async Task<T> GetByIdAsync(int id)
    {
        return await _dbSet.FindAsync(id);
    }

    public virtual async Task<IEnumerable<T>> GetAllAsync()
    {
        return await _dbSet.ToListAsync();
    }

    public virtual async Task<IEnumerable<T>> FindAsync(
        Expression<Func<T, bool>> predicate)
    {
        return await _dbSet.Where(predicate).ToListAsync();
    }

    public virtual async Task AddAsync(T entity)
    {
        await _dbSet.AddAsync(entity);
    }

    public virtual void Update(T entity)
    {
        _dbSet.Attach(entity);
        _context.Entry(entity).State = EntityState.Modified;
    }

    public virtual void Remove(T entity)
    {
        if (_context.Entry(entity).State == EntityState.Detached)
        {
            _dbSet.Attach(entity);
        }
        _dbSet.Remove(entity);
    }
}
```

### Include 지원 추가
```csharp
public interface IRepository<T> where T : class
{
    Task<T> GetByIdAsync(int id, params Expression<Func<T, object>>[] includes);
    Task<IEnumerable<T>> GetAllAsync(params Expression<Func<T, object>>[] includes);
}

public class Repository<T> : IRepository<T> where T : class
{
    public virtual async Task<T> GetByIdAsync(
        int id, 
        params Expression<Func<T, object>>[] includes)
    {
        IQueryable<T> query = _dbSet;
        
        foreach (var include in includes)
        {
            query = query.Include(include);
        }
        
        return await query.FirstOrDefaultAsync(e => 
            EF.Property<int>(e, "Id") == id);
    }

    public virtual async Task<IEnumerable<T>> GetAllAsync(
        params Expression<Func<T, object>>[] includes)
    {
        IQueryable<T> query = _dbSet;
        
        foreach (var include in includes)
        {
            query = query.Include(include);
        }
        
        return await query.ToListAsync();
    }
}
```

## 특화된 Repository

### Entity별 Repository Interface
```csharp
// Product Repository
public interface IProductRepository : IRepository<Product>
{
    Task<IEnumerable<Product>> GetProductsByCategoryAsync(int categoryId);
    Task<IEnumerable<Product>> GetTopSellingProductsAsync(int count);
    Task<Product> GetProductWithReviewsAsync(int productId);
}

// Order Repository
public interface IOrderRepository : IRepository<Order>
{
    Task<IEnumerable<Order>> GetOrdersByCustomerAsync(int customerId);
    Task<Order> GetOrderWithDetailsAsync(int orderId);
    Task<decimal> GetTotalSalesAsync(DateTime startDate, DateTime endDate);
}
```

### 특화된 Repository 구현
```csharp
public class ProductRepository : Repository<Product>, IProductRepository
{
    public ProductRepository(ApplicationDbContext context) : base(context)
    {
    }

    public async Task<IEnumerable<Product>> GetProductsByCategoryAsync(
        int categoryId)
    {
        return await _dbSet
            .Where(p => p.CategoryId == categoryId)
            .Include(p => p.Category)
            .ToListAsync();
    }

    public async Task<IEnumerable<Product>> GetTopSellingProductsAsync(
        int count)
    {
        return await _dbSet
            .OrderByDescending(p => p.OrderItems.Sum(oi => oi.Quantity))
            .Take(count)
            .ToListAsync();
    }

    public async Task<Product> GetProductWithReviewsAsync(int productId)
    {
        return await _dbSet
            .Include(p => p.Reviews)
                .ThenInclude(r => r.Customer)
            .FirstOrDefaultAsync(p => p.Id == productId);
    }
}
```

## Unit of Work 패턴

### Unit of Work Interface
```csharp
public interface IUnitOfWork : IDisposable
{
    IProductRepository Products { get; }
    IOrderRepository Orders { get; }
    ICustomerRepository Customers { get; }
    
    Task<int> CompleteAsync();
    Task BeginTransactionAsync();
    Task CommitAsync();
    Task RollbackAsync();
}
```

### Unit of Work 구현
```csharp
public class UnitOfWork : IUnitOfWork
{
    private readonly ApplicationDbContext _context;
    private IDbContextTransaction _transaction;
    
    private IProductRepository _products;
    private IOrderRepository _orders;
    private ICustomerRepository _customers;

    public UnitOfWork(ApplicationDbContext context)
    {
        _context = context;
    }

    public IProductRepository Products => 
        _products ??= new ProductRepository(_context);
        
    public IOrderRepository Orders => 
        _orders ??= new OrderRepository(_context);
        
    public ICustomerRepository Customers => 
        _customers ??= new CustomerRepository(_context);

    public async Task<int> CompleteAsync()
    {
        return await _context.SaveChangesAsync();
    }

    public async Task BeginTransactionAsync()
    {
        _transaction = await _context.Database.BeginTransactionAsync();
    }

    public async Task CommitAsync()
    {
        await _transaction?.CommitAsync();
    }

    public async Task RollbackAsync()
    {
        await _transaction?.RollbackAsync();
    }

    public void Dispose()
    {
        _transaction?.Dispose();
        _context.Dispose();
    }
}
```

## Service Layer에서 사용

### Service Interface
```csharp
public interface IOrderService
{
    Task<OrderDto> CreateOrderAsync(CreateOrderDto orderDto);
    Task<bool> CancelOrderAsync(int orderId);
}
```

### Service 구현
```csharp
public class OrderService : IOrderService
{
    private readonly IUnitOfWork _unitOfWork;
    private readonly ILogger<OrderService> _logger;

    public OrderService(IUnitOfWork unitOfWork, ILogger<OrderService> logger)
    {
        _unitOfWork = unitOfWork;
        _logger = logger;
    }

    public async Task<OrderDto> CreateOrderAsync(CreateOrderDto orderDto)
    {
        await _unitOfWork.BeginTransactionAsync();
        
        try
        {
            // 재고 확인
            foreach (var item in orderDto.Items)
            {
                var product = await _unitOfWork.Products
                    .GetByIdAsync(item.ProductId);
                    
                if (product.Stock < item.Quantity)
                {
                    throw new InsufficientStockException(
                        $"Insufficient stock for product {product.Name}");
                }
            }

            // 주문 생성
            var order = new Order
            {
                CustomerId = orderDto.CustomerId,
                OrderDate = DateTime.UtcNow,
                Status = OrderStatus.Pending,
                OrderItems = orderDto.Items.Select(item => new OrderItem
                {
                    ProductId = item.ProductId,
                    Quantity = item.Quantity,
                    UnitPrice = item.UnitPrice
                }).ToList()
            };

            await _unitOfWork.Orders.AddAsync(order);

            // 재고 감소
            foreach (var item in order.OrderItems)
            {
                var product = await _unitOfWork.Products
                    .GetByIdAsync(item.ProductId);
                product.Stock -= item.Quantity;
                _unitOfWork.Products.Update(product);
            }

            await _unitOfWork.CompleteAsync();
            await _unitOfWork.CommitAsync();

            _logger.LogInformation(
                "Order {OrderId} created successfully", order.Id);

            return MapToDto(order);
        }
        catch (Exception ex)
        {
            await _unitOfWork.RollbackAsync();
            _logger.LogError(ex, 
                "Error creating order for customer {CustomerId}", 
                orderDto.CustomerId);
            throw;
        }
    }

    public async Task<bool> CancelOrderAsync(int orderId)
    {
        var order = await _unitOfWork.Orders
            .GetOrderWithDetailsAsync(orderId);
            
        if (order == null)
            return false;

        if (order.Status != OrderStatus.Pending)
        {
            throw new InvalidOperationException(
                "Only pending orders can be cancelled");
        }

        order.Status = OrderStatus.Cancelled;
        _unitOfWork.Orders.Update(order);

        // 재고 복원
        foreach (var item in order.OrderItems)
        {
            var product = await _unitOfWork.Products
                .GetByIdAsync(item.ProductId);
            product.Stock += item.Quantity;
            _unitOfWork.Products.Update(product);
        }

        await _unitOfWork.CompleteAsync();
        
        return true;
    }

    private OrderDto MapToDto(Order order)
    {
        return new OrderDto
        {
            Id = order.Id,
            CustomerId = order.CustomerId,
            OrderDate = order.OrderDate,
            Status = order.Status.ToString(),
            TotalAmount = order.OrderItems.Sum(i => i.Quantity * i.UnitPrice)
        };
    }
}
```

## 의존성 주입 설정

### Program.cs 설정
```csharp
var builder = WebApplication.CreateBuilder(args);

// DbContext 등록
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("DefaultConnection")));

// Repository와 Unit of Work 등록
builder.Services.AddScoped(typeof(IRepository<>), typeof(Repository<>));
builder.Services.AddScoped<IProductRepository, ProductRepository>();
builder.Services.AddScoped<IOrderRepository, OrderRepository>();
builder.Services.AddScoped<ICustomerRepository, CustomerRepository>();
builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();

// Service 등록
builder.Services.AddScoped<IOrderService, OrderService>();
builder.Services.AddScoped<IProductService, ProductService>();

builder.Services.AddControllers();

var app = builder.Build();

// 미들웨어 설정
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

## 테스트 작성

### Repository 단위 테스트
```csharp
public class ProductRepositoryTests
{
    private ApplicationDbContext GetInMemoryContext()
    {
        var options = new DbContextOptionsBuilder<ApplicationDbContext>()
            .UseInMemoryDatabase(databaseName: Guid.NewGuid().ToString())
            .Options;
            
        return new ApplicationDbContext(options);
    }

    [Fact]
    public async Task GetProductsByCategory_ReturnsCorrectProducts()
    {
        // Arrange
        using var context = GetInMemoryContext();
        var repository = new ProductRepository(context);
        
        var category = new Category { Id = 1, Name = "Electronics" };
        var products = new List<Product>
        {
            new Product { Id = 1, Name = "Laptop", CategoryId = 1 },
            new Product { Id = 2, Name = "Phone", CategoryId = 1 },
            new Product { Id = 3, Name = "Book", CategoryId = 2 }
        };
        
        context.Categories.Add(category);
        context.Products.AddRange(products);
        await context.SaveChangesAsync();

        // Act
        var result = await repository.GetProductsByCategoryAsync(1);

        // Assert
        Assert.Equal(2, result.Count());
        Assert.All(result, p => Assert.Equal(1, p.CategoryId));
    }
}
```

### Service 단위 테스트 (Mock)
```csharp
public class OrderServiceTests
{
    private readonly Mock<IUnitOfWork> _unitOfWorkMock;
    private readonly Mock<ILogger<OrderService>> _loggerMock;
    private readonly OrderService _orderService;

    public OrderServiceTests()
    {
        _unitOfWorkMock = new Mock<IUnitOfWork>();
        _loggerMock = new Mock<ILogger<OrderService>>();
        _orderService = new OrderService(_unitOfWorkMock.Object, _loggerMock.Object);
    }

    [Fact]
    public async Task CreateOrder_WithSufficientStock_CreatesOrderSuccessfully()
    {
        // Arrange
        var createOrderDto = new CreateOrderDto
        {
            CustomerId = 1,
            Items = new List<OrderItemDto>
            {
                new OrderItemDto { ProductId = 1, Quantity = 2, UnitPrice = 100 }
            }
        };

        var product = new Product { Id = 1, Name = "Test Product", Stock = 10 };
        
        _unitOfWorkMock.Setup(u => u.Products.GetByIdAsync(1))
            .ReturnsAsync(product);
        _unitOfWorkMock.Setup(u => u.Orders.AddAsync(It.IsAny<Order>()))
            .Returns(Task.CompletedTask);
        _unitOfWorkMock.Setup(u => u.CompleteAsync())
            .ReturnsAsync(1);

        // Act
        var result = await _orderService.CreateOrderAsync(createOrderDto);

        // Assert
        Assert.NotNull(result);
        _unitOfWorkMock.Verify(u => u.BeginTransactionAsync(), Times.Once);
        _unitOfWorkMock.Verify(u => u.CommitAsync(), Times.Once);
        _unitOfWorkMock.Verify(u => u.Products.Update(It.IsAny<Product>()), Times.Once);
    }

    [Fact]
    public async Task CreateOrder_WithInsufficientStock_ThrowsException()
    {
        // Arrange
        var createOrderDto = new CreateOrderDto
        {
            CustomerId = 1,
            Items = new List<OrderItemDto>
            {
                new OrderItemDto { ProductId = 1, Quantity = 20, UnitPrice = 100 }
            }
        };

        var product = new Product { Id = 1, Name = "Test Product", Stock = 10 };
        
        _unitOfWorkMock.Setup(u => u.Products.GetByIdAsync(1))
            .ReturnsAsync(product);

        // Act & Assert
        await Assert.ThrowsAsync<InsufficientStockException>(
            () => _orderService.CreateOrderAsync(createOrderDto));
            
        _unitOfWorkMock.Verify(u => u.RollbackAsync(), Times.Once);
    }
}
```

## Specification 패턴

### Specification Interface
```csharp
public interface ISpecification<T>
{
    Expression<Func<T, bool>> Criteria { get; }
    List<Expression<Func<T, object>>> Includes { get; }
    Expression<Func<T, object>> OrderBy { get; }
    Expression<Func<T, object>> OrderByDescending { get; }
    int Take { get; }
    int Skip { get; }
    bool IsPagingEnabled { get; }
}
```

### Base Specification
```csharp
public abstract class BaseSpecification<T> : ISpecification<T>
{
    public Expression<Func<T, bool>> Criteria { get; private set; }
    public List<Expression<Func<T, object>>> Includes { get; } = 
        new List<Expression<Func<T, object>>>();
    public Expression<Func<T, object>> OrderBy { get; private set; }
    public Expression<Func<T, object>> OrderByDescending { get; private set; }
    public int Take { get; private set; }
    public int Skip { get; private set; }
    public bool IsPagingEnabled { get; private set; }

    protected void AddCriteria(Expression<Func<T, bool>> criteria)
    {
        Criteria = criteria;
    }

    protected void AddInclude(Expression<Func<T, object>> includeExpression)
    {
        Includes.Add(includeExpression);
    }

    protected void ApplyPaging(int skip, int take)
    {
        Skip = skip;
        Take = take;
        IsPagingEnabled = true;
    }

    protected void ApplyOrderBy(Expression<Func<T, object>> orderByExpression)
    {
        OrderBy = orderByExpression;
    }

    protected void ApplyOrderByDescending(
        Expression<Func<T, object>> orderByDescExpression)
    {
        OrderByDescending = orderByDescExpression;
    }
}
```

### Specification 사용 예
```csharp
public class ProductsWithReviewsSpecification : BaseSpecification<Product>
{
    public ProductsWithReviewsSpecification(int minRating)
    {
        AddCriteria(p => p.Reviews.Any(r => r.Rating >= minRating));
        AddInclude(p => p.Reviews);
        AddInclude(p => p.Category);
        ApplyOrderByDescending(p => p.Reviews.Average(r => r.Rating));
    }
}

// Repository에서 Specification 사용
public async Task<IEnumerable<T>> FindAsync(ISpecification<T> spec)
{
    return await ApplySpecification(spec).ToListAsync();
}

private IQueryable<T> ApplySpecification(ISpecification<T> spec)
{
    var query = _dbSet.AsQueryable();

    if (spec.Criteria != null)
    {
        query = query.Where(spec.Criteria);
    }

    query = spec.Includes.Aggregate(query, 
        (current, include) => current.Include(include));

    if (spec.OrderBy != null)
    {
        query = query.OrderBy(spec.OrderBy);
    }
    else if (spec.OrderByDescending != null)
    {
        query = query.OrderByDescending(spec.OrderByDescending);
    }

    if (spec.IsPagingEnabled)
    {
        query = query.Skip(spec.Skip).Take(spec.Take);
    }

    return query;
}
```

## 성능 최적화

### AsNoTracking 사용
```csharp
public async Task<IEnumerable<ProductDto>> GetProductsForDisplayAsync()
{
    return await _context.Products
        .AsNoTracking()  // 변경 추적 비활성화
        .Include(p => p.Category)
        .Select(p => new ProductDto
        {
            Id = p.Id,
            Name = p.Name,
            CategoryName = p.Category.Name,
            Price = p.Price
        })
        .ToListAsync();
}
```

### 컴파일된 쿼리
```csharp
private static readonly Func<ApplicationDbContext, int, Task<Product>> 
    GetProductByIdQuery = EF.CompileAsyncQuery(
        (ApplicationDbContext context, int id) =>
            context.Products.FirstOrDefault(p => p.Id == id));

public async Task<Product> GetByIdOptimizedAsync(int id)
{
    return await GetProductByIdQuery(_context, id);
}
```

## 주의사항

### Repository에 비즈니스 로직 배제
```csharp
// 나쁜 예
public class BadProductRepository : IProductRepository
{
    public async Task<Product> CreateProductWithDiscountAsync(
        Product product, decimal discountPercentage)
    {
        product.Price = product.Price * (1 - discountPercentage / 100);
        await _dbSet.AddAsync(product);
        return product;
    }
}

// 좋은 예
public class ProductService
{
    public async Task<Product> CreateProductWithDiscountAsync(
        Product product, decimal discountPercentage)
    {
        product.Price = product.Price * (1 - discountPercentage / 100);
        await _unitOfWork.Products.AddAsync(product);
        await _unitOfWork.CompleteAsync();
        return product;
    }
}
```

### 과도한 추상화 피하기
```csharp
// 불필요한 추상화
public interface IReadOnlyRepository<T> { }
public interface IWriteOnlyRepository<T> { }
public interface IRepository<T> : IReadOnlyRepository<T>, IWriteOnlyRepository<T> { }

// 필요한 만큼만 추상화
public interface IRepository<T> where T : class
{
    Task<T> GetByIdAsync(int id);
    Task<IEnumerable<T>> GetAllAsync();
    Task AddAsync(T entity);
    void Update(T entity);
    void Remove(T entity);
}
```