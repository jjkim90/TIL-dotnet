# Clean Architecture

## Clean Architecture 소개

Clean Architecture는 Robert C. Martin(Uncle Bob)이 제안한 소프트웨어 아키텍처 패턴입니다. 비즈니스 로직을 외부 종속성으로부터 분리하여 테스트 가능하고, 유지보수가 쉬우며, 변경에 유연한 시스템을 만드는 것을 목표로 합니다.

### 핵심 원칙

1. **프레임워크 독립성**: 비즈니스 규칙이 프레임워크에 의존하지 않음
2. **테스트 용이성**: 비즈니스 규칙을 UI, 데이터베이스, 외부 시스템 없이 테스트 가능
3. **UI 독립성**: UI를 비즈니스 규칙 변경 없이 변경 가능
4. **데이터베이스 독립성**: 데이터베이스를 비즈니스 규칙 변경 없이 교체 가능
5. **외부 시스템 독립성**: 비즈니스 규칙이 외부 인터페이스에 대해 알지 못함

### 의존성 규칙

Clean Architecture의 가장 중요한 규칙은 **의존성 방향**입니다:
- 소스 코드 의존성은 항상 안쪽을 향해야 함
- 내부 원은 외부 원에 대해 전혀 알지 못함
- 외부 원의 이름이 내부 원의 코드에 나타나면 안 됨

## 계층 구조

### 1. Domain Layer (중심부)

가장 안쪽 계층으로 비즈니스 규칙과 엔티티를 포함합니다.

```csharp
// Domain/Entities/Product.cs
namespace CleanArchitecture.Domain.Entities
{
    public class Product
    {
        public int Id { get; private set; }
        public string Name { get; private set; }
        public decimal Price { get; private set; }
        public int Stock { get; private set; }
        
        public Product(string name, decimal price, int stock)
        {
            ValidateName(name);
            ValidatePrice(price);
            ValidateStock(stock);
            
            Name = name;
            Price = price;
            Stock = stock;
        }
        
        public void UpdatePrice(decimal newPrice)
        {
            ValidatePrice(newPrice);
            Price = newPrice;
        }
        
        public void AdjustStock(int quantity)
        {
            if (Stock + quantity < 0)
                throw new InvalidOperationException("재고가 부족합니다.");
                
            Stock += quantity;
        }
        
        private void ValidateName(string name)
        {
            if (string.IsNullOrWhiteSpace(name))
                throw new ArgumentException("상품명은 필수입니다.");
        }
        
        private void ValidatePrice(decimal price)
        {
            if (price <= 0)
                throw new ArgumentException("가격은 0보다 커야 합니다.");
        }
        
        private void ValidateStock(int stock)
        {
            if (stock < 0)
                throw new ArgumentException("재고는 0 이상이어야 합니다.");
        }
    }
}
```

### 2. Application Layer

비즈니스 로직과 유스케이스를 포함합니다.

```csharp
// Application/Interfaces/IProductRepository.cs
namespace CleanArchitecture.Application.Interfaces
{
    public interface IProductRepository
    {
        Task<Product?> GetByIdAsync(int id);
        Task<IEnumerable<Product>> GetAllAsync();
        Task<int> AddAsync(Product product);
        Task UpdateAsync(Product product);
        Task DeleteAsync(int id);
    }
}

// Application/UseCases/CreateProduct/CreateProductCommand.cs
namespace CleanArchitecture.Application.UseCases.CreateProduct
{
    public class CreateProductCommand
    {
        public string Name { get; set; }
        public decimal Price { get; set; }
        public int InitialStock { get; set; }
    }
    
    public class CreateProductResponse
    {
        public int Id { get; set; }
        public string Name { get; set; }
        public decimal Price { get; set; }
        public int Stock { get; set; }
    }
}

// Application/UseCases/CreateProduct/CreateProductHandler.cs
namespace CleanArchitecture.Application.UseCases.CreateProduct
{
    public class CreateProductHandler
    {
        private readonly IProductRepository _productRepository;
        
        public CreateProductHandler(IProductRepository productRepository)
        {
            _productRepository = productRepository;
        }
        
        public async Task<CreateProductResponse> HandleAsync(CreateProductCommand command)
        {
            var product = new Product(
                command.Name,
                command.Price,
                command.InitialStock
            );
            
            var id = await _productRepository.AddAsync(product);
            
            return new CreateProductResponse
            {
                Id = id,
                Name = product.Name,
                Price = product.Price,
                Stock = product.Stock
            };
        }
    }
}
```

### 3. Infrastructure Layer

외부 시스템과의 통합을 담당합니다.

```csharp
// Infrastructure/Data/ApplicationDbContext.cs
namespace CleanArchitecture.Infrastructure.Data
{
    public class ApplicationDbContext : DbContext
    {
        public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
            : base(options)
        {
        }
        
        public DbSet<Product> Products { get; set; }
        
        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            modelBuilder.Entity<Product>(entity =>
            {
                entity.HasKey(e => e.Id);
                entity.Property(e => e.Name).IsRequired().HasMaxLength(200);
                entity.Property(e => e.Price).HasPrecision(18, 2);
            });
        }
    }
}

// Infrastructure/Repositories/ProductRepository.cs
namespace CleanArchitecture.Infrastructure.Repositories
{
    public class ProductRepository : IProductRepository
    {
        private readonly ApplicationDbContext _context;
        
        public ProductRepository(ApplicationDbContext context)
        {
            _context = context;
        }
        
        public async Task<Product?> GetByIdAsync(int id)
        {
            return await _context.Products.FindAsync(id);
        }
        
        public async Task<IEnumerable<Product>> GetAllAsync()
        {
            return await _context.Products.ToListAsync();
        }
        
        public async Task<int> AddAsync(Product product)
        {
            _context.Products.Add(product);
            await _context.SaveChangesAsync();
            return product.Id;
        }
        
        public async Task UpdateAsync(Product product)
        {
            _context.Entry(product).State = EntityState.Modified;
            await _context.SaveChangesAsync();
        }
        
        public async Task DeleteAsync(int id)
        {
            var product = await _context.Products.FindAsync(id);
            if (product != null)
            {
                _context.Products.Remove(product);
                await _context.SaveChangesAsync();
            }
        }
    }
}
```

### 4. Presentation Layer

사용자 인터페이스를 담당합니다.

```csharp
// WebApi/Controllers/ProductsController.cs
namespace CleanArchitecture.WebApi.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class ProductsController : ControllerBase
    {
        private readonly CreateProductHandler _createProductHandler;
        private readonly IProductRepository _productRepository;
        
        public ProductsController(
            CreateProductHandler createProductHandler,
            IProductRepository productRepository)
        {
            _createProductHandler = createProductHandler;
            _productRepository = productRepository;
        }
        
        [HttpGet]
        public async Task<ActionResult<IEnumerable<Product>>> GetProducts()
        {
            var products = await _productRepository.GetAllAsync();
            return Ok(products);
        }
        
        [HttpGet("{id}")]
        public async Task<ActionResult<Product>> GetProduct(int id)
        {
            var product = await _productRepository.GetByIdAsync(id);
            
            if (product == null)
                return NotFound();
                
            return Ok(product);
        }
        
        [HttpPost]
        public async Task<ActionResult<CreateProductResponse>> CreateProduct(
            CreateProductCommand command)
        {
            try
            {
                var response = await _createProductHandler.HandleAsync(command);
                return CreatedAtAction(
                    nameof(GetProduct), 
                    new { id = response.Id }, 
                    response);
            }
            catch (ArgumentException ex)
            {
                return BadRequest(new { error = ex.Message });
            }
        }
    }
}
```

## 프로젝트 구조

### 솔루션 구조

```
CleanArchitecture/
├── src/
│   ├── CleanArchitecture.Domain/
│   │   ├── Entities/
│   │   ├── ValueObjects/
│   │   └── Exceptions/
│   ├── CleanArchitecture.Application/
│   │   ├── Interfaces/
│   │   ├── UseCases/
│   │   └── Common/
│   ├── CleanArchitecture.Infrastructure/
│   │   ├── Data/
│   │   ├── Repositories/
│   │   └── Services/
│   └── CleanArchitecture.WebApi/
│       ├── Controllers/
│       ├── Filters/
│       └── Middleware/
└── tests/
    ├── CleanArchitecture.Domain.Tests/
    ├── CleanArchitecture.Application.Tests/
    └── CleanArchitecture.IntegrationTests/
```

### 프로젝트 참조

```xml
<!-- Domain.csproj - 아무것도 참조하지 않음 -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net7.0</TargetFramework>
  </PropertyGroup>
</Project>

<!-- Application.csproj - Domain만 참조 -->
<Project Sdk="Microsoft.NET.Sdk">
  <ItemGroup>
    <ProjectReference Include="..\CleanArchitecture.Domain\CleanArchitecture.Domain.csproj" />
  </ItemGroup>
</Project>

<!-- Infrastructure.csproj - Application 참조 -->
<Project Sdk="Microsoft.NET.Sdk">
  <ItemGroup>
    <ProjectReference Include="..\CleanArchitecture.Application\CleanArchitecture.Application.csproj" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="7.0.0" />
  </ItemGroup>
</Project>

<!-- WebApi.csproj - Application과 Infrastructure 참조 -->
<Project Sdk="Microsoft.NET.Sdk.Web">
  <ItemGroup>
    <ProjectReference Include="..\CleanArchitecture.Application\CleanArchitecture.Application.csproj" />
    <ProjectReference Include="..\CleanArchitecture.Infrastructure\CleanArchitecture.Infrastructure.csproj" />
  </ItemGroup>
</Project>
```

## 고급 개념

### Value Objects

```csharp
// Domain/ValueObjects/Money.cs
namespace CleanArchitecture.Domain.ValueObjects
{
    public class Money : ValueObject
    {
        public decimal Amount { get; }
        public string Currency { get; }
        
        public Money(decimal amount, string currency)
        {
            if (amount < 0)
                throw new ArgumentException("금액은 음수일 수 없습니다.");
                
            if (string.IsNullOrWhiteSpace(currency))
                throw new ArgumentException("통화는 필수입니다.");
                
            Amount = amount;
            Currency = currency.ToUpper();
        }
        
        public Money Add(Money other)
        {
            if (Currency != other.Currency)
                throw new InvalidOperationException("다른 통화는 더할 수 없습니다.");
                
            return new Money(Amount + other.Amount, Currency);
        }
        
        protected override IEnumerable<object> GetEqualityComponents()
        {
            yield return Amount;
            yield return Currency;
        }
    }
}

// Domain/Common/ValueObject.cs
namespace CleanArchitecture.Domain.Common
{
    public abstract class ValueObject
    {
        protected abstract IEnumerable<object> GetEqualityComponents();
        
        public override bool Equals(object? obj)
        {
            if (obj == null || obj.GetType() != GetType())
                return false;
                
            var other = (ValueObject)obj;
            
            return GetEqualityComponents()
                .SequenceEqual(other.GetEqualityComponents());
        }
        
        public override int GetHashCode()
        {
            return GetEqualityComponents()
                .Select(x => x?.GetHashCode() ?? 0)
                .Aggregate((x, y) => x ^ y);
        }
    }
}
```

### Domain Events

```csharp
// Domain/Common/IDomainEvent.cs
namespace CleanArchitecture.Domain.Common
{
    public interface IDomainEvent
    {
        DateTime OccurredOn { get; }
    }
}

// Domain/Events/ProductCreatedEvent.cs
namespace CleanArchitecture.Domain.Events
{
    public class ProductCreatedEvent : IDomainEvent
    {
        public int ProductId { get; }
        public string ProductName { get; }
        public DateTime OccurredOn { get; }
        
        public ProductCreatedEvent(int productId, string productName)
        {
            ProductId = productId;
            ProductName = productName;
            OccurredOn = DateTime.UtcNow;
        }
    }
}

// Domain/Common/AggregateRoot.cs
namespace CleanArchitecture.Domain.Common
{
    public abstract class AggregateRoot
    {
        private readonly List<IDomainEvent> _domainEvents = new();
        
        public IReadOnlyCollection<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();
        
        protected void AddDomainEvent(IDomainEvent domainEvent)
        {
            _domainEvents.Add(domainEvent);
        }
        
        public void ClearDomainEvents()
        {
            _domainEvents.Clear();
        }
    }
}
```

## CQRS 패턴 적용

```csharp
// Application/Common/Interfaces/ICommand.cs
namespace CleanArchitecture.Application.Common.Interfaces
{
    public interface ICommand<TResponse>
    {
    }
    
    public interface ICommandHandler<TCommand, TResponse>
        where TCommand : ICommand<TResponse>
    {
        Task<TResponse> HandleAsync(TCommand command);
    }
}

// Application/Common/Interfaces/IQuery.cs
namespace CleanArchitecture.Application.Common.Interfaces
{
    public interface IQuery<TResponse>
    {
    }
    
    public interface IQueryHandler<TQuery, TResponse>
        where TQuery : IQuery<TResponse>
    {
        Task<TResponse> HandleAsync(TQuery query);
    }
}

// Application/Commands/UpdateProduct/UpdateProductCommand.cs
namespace CleanArchitecture.Application.Commands.UpdateProduct
{
    public class UpdateProductCommand : ICommand<UpdateProductResponse>
    {
        public int Id { get; set; }
        public string Name { get; set; }
        public decimal Price { get; set; }
    }
    
    public class UpdateProductCommandHandler 
        : ICommandHandler<UpdateProductCommand, UpdateProductResponse>
    {
        private readonly IProductRepository _repository;
        
        public UpdateProductCommandHandler(IProductRepository repository)
        {
            _repository = repository;
        }
        
        public async Task<UpdateProductResponse> HandleAsync(UpdateProductCommand command)
        {
            var product = await _repository.GetByIdAsync(command.Id);
            
            if (product == null)
                throw new NotFoundException($"Product with ID {command.Id} not found");
                
            product.UpdateName(command.Name);
            product.UpdatePrice(command.Price);
            
            await _repository.UpdateAsync(product);
            
            return new UpdateProductResponse
            {
                Id = product.Id,
                Name = product.Name,
                Price = product.Price
            };
        }
    }
}
```

## 의존성 주입 설정

```csharp
// Infrastructure/DependencyInjection.cs
namespace CleanArchitecture.Infrastructure
{
    public static class DependencyInjection
    {
        public static IServiceCollection AddInfrastructure(
            this IServiceCollection services,
            IConfiguration configuration)
        {
            // 데이터베이스 컨텍스트
            services.AddDbContext<ApplicationDbContext>(options =>
                options.UseSqlServer(
                    configuration.GetConnectionString("DefaultConnection"),
                    b => b.MigrationsAssembly(typeof(ApplicationDbContext).Assembly.FullName)));
                    
            // 리포지토리
            services.AddScoped<IProductRepository, ProductRepository>();
            
            // 외부 서비스
            services.AddScoped<IEmailService, EmailService>();
            services.AddScoped<IFileStorageService, FileStorageService>();
            
            return services;
        }
    }
}

// Application/DependencyInjection.cs
namespace CleanArchitecture.Application
{
    public static class DependencyInjection
    {
        public static IServiceCollection AddApplication(this IServiceCollection services)
        {
            var assembly = typeof(DependencyInjection).Assembly;
            
            // 핸들러 자동 등록
            services.Scan(scan => scan
                .FromAssemblies(assembly)
                .AddClasses(classes => classes
                    .AssignableTo(typeof(ICommandHandler<,>)))
                .AsImplementedInterfaces()
                .WithScopedLifetime());
                
            services.Scan(scan => scan
                .FromAssemblies(assembly)
                .AddClasses(classes => classes
                    .AssignableTo(typeof(IQueryHandler<,>)))
                .AsImplementedInterfaces()
                .WithScopedLifetime());
                
            return services;
        }
    }
}

// WebApi/Program.cs
var builder = WebApplication.CreateBuilder(args);

// Clean Architecture 계층 등록
builder.Services.AddApplication();
builder.Services.AddInfrastructure(builder.Configuration);

// Web API 서비스
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

## 테스트 전략

### 단위 테스트 - Domain

```csharp
// Domain.Tests/Entities/ProductTests.cs
public class ProductTests
{
    [Fact]
    public void Constructor_ValidParameters_CreatesProduct()
    {
        // Arrange
        var name = "Test Product";
        var price = 100m;
        var stock = 10;
        
        // Act
        var product = new Product(name, price, stock);
        
        // Assert
        Assert.Equal(name, product.Name);
        Assert.Equal(price, product.Price);
        Assert.Equal(stock, product.Stock);
    }
    
    [Fact]
    public void Constructor_InvalidPrice_ThrowsException()
    {
        // Arrange & Act & Assert
        Assert.Throws<ArgumentException>(() => 
            new Product("Test", -10m, 10));
    }
}
```

### 단위 테스트 - Application

```csharp
// Application.Tests/UseCases/CreateProductHandlerTests.cs
public class CreateProductHandlerTests
{
    private readonly Mock<IProductRepository> _mockRepository;
    private readonly CreateProductHandler _handler;
    
    public CreateProductHandlerTests()
    {
        _mockRepository = new Mock<IProductRepository>();
        _handler = new CreateProductHandler(_mockRepository.Object);
    }
    
    [Fact]
    public async Task HandleAsync_ValidCommand_CreatesProduct()
    {
        // Arrange
        var command = new CreateProductCommand
        {
            Name = "Test Product",
            Price = 100m,
            InitialStock = 10
        };
        
        _mockRepository
            .Setup(x => x.AddAsync(It.IsAny<Product>()))
            .ReturnsAsync(1);
            
        // Act
        var response = await _handler.HandleAsync(command);
        
        // Assert
        Assert.Equal(command.Name, response.Name);
        _mockRepository.Verify(x => x.AddAsync(It.IsAny<Product>()), Times.Once);
    }
}
```

## 예외 처리

```csharp
// Domain/Exceptions/DomainException.cs
namespace CleanArchitecture.Domain.Exceptions
{
    public abstract class DomainException : Exception
    {
        protected DomainException(string message) : base(message)
        {
        }
    }
    
    public class InsufficientStockException : DomainException
    {
        public InsufficientStockException(int requested, int available)
            : base($"요청한 수량({requested})이 재고({available})보다 많습니다.")
        {
            Requested = requested;
            Available = available;
        }
        
        public int Requested { get; }
        public int Available { get; }
    }
}

// WebApi/Middleware/ExceptionHandlingMiddleware.cs
public class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;
    
    public ExceptionHandlingMiddleware(
        RequestDelegate next,
        ILogger<ExceptionHandlingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            await HandleExceptionAsync(context, ex);
        }
    }
    
    private async Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        _logger.LogError(exception, "An error occurred: {Message}", exception.Message);
        
        var response = exception switch
        {
            DomainException => new
            {
                StatusCode = StatusCodes.Status400BadRequest,
                Message = exception.Message
            },
            NotFoundException => new
            {
                StatusCode = StatusCodes.Status404NotFound,
                Message = exception.Message
            },
            _ => new
            {
                StatusCode = StatusCodes.Status500InternalServerError,
                Message = "An error occurred while processing your request"
            }
        };
        
        context.Response.StatusCode = response.StatusCode;
        await context.Response.WriteAsJsonAsync(response);
    }
}
```

## 모범 사례

### 1. 의존성 방향 준수
- Domain은 아무것도 참조하지 않음
- Application은 Domain만 참조
- Infrastructure는 Application을 참조
- Presentation은 Application과 Infrastructure를 참조

### 2. 비즈니스 로직 위치
- 엔티티 관련 로직은 Domain Layer에
- 유스케이스 로직은 Application Layer에
- 인프라 관련 로직은 Infrastructure Layer에

### 3. 인터페이스 정의
- 리포지토리 인터페이스는 Application Layer에
- 외부 서비스 인터페이스는 Application Layer에
- 구현체는 Infrastructure Layer에

### 4. 테스트 전략
- Domain Layer: 단위 테스트
- Application Layer: 단위 테스트 (Mock 사용)
- Infrastructure Layer: 통합 테스트
- API Layer: 통합 테스트

## 마무리

Clean Architecture는 복잡한 비즈니스 로직을 가진 애플리케이션에 특히 유용합니다. 초기 설정이 복잡하지만 장기적으로 유지보수성과 테스트 용이성에서 큰 이점을 제공합니다. 핵심은 의존성 방향을 엄격히 관리하여 변경에 유연한 시스템을 만드는 것입니다.