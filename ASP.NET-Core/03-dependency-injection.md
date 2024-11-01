# 의존성 주입 (DI Container)

## 의존성 주입이란?

의존성 주입(Dependency Injection, DI)은 객체가 필요로 하는 의존성을 외부에서 제공하는 디자인 패턴입니다. ASP.NET Core는 DI를 기본으로 내장하고 있어 느슨한 결합과 테스트 가능한 코드를 작성할 수 있습니다.

### DI의 장점
- **느슨한 결합**: 클래스 간 의존성 감소
- **테스트 용이성**: Mock 객체를 쉽게 주입
- **유지보수성**: 구현체 변경이 용이
- **코드 재사용성**: 인터페이스 기반 프로그래밍

## 서비스 수명 주기

### Transient (일시적)
```csharp
builder.Services.AddTransient<IEmailService, EmailService>();
```
- 요청할 때마다 새 인스턴스 생성
- 가장 짧은 수명
- 상태를 유지하지 않는 가벼운 서비스에 적합

### Scoped (범위)
```csharp
builder.Services.AddScoped<IUserRepository, UserRepository>();
```
- HTTP 요청당 하나의 인스턴스 생성
- 같은 요청 내에서는 동일한 인스턴스 공유
- 데이터베이스 컨텍스트에 주로 사용

### Singleton (단일)
```csharp
builder.Services.AddSingleton<ICacheService, CacheService>();
```
- 애플리케이션 생명주기 동안 하나의 인스턴스만 생성
- 모든 요청에서 동일한 인스턴스 공유
- 상태를 공유해야 하는 서비스에 적합

## 서비스 등록

### 기본 등록
```csharp
var builder = WebApplication.CreateBuilder(args);

// 인터페이스와 구현체 매핑
builder.Services.AddTransient<IProductService, ProductService>();
builder.Services.AddScoped<IOrderRepository, OrderRepository>();
builder.Services.AddSingleton<IConfiguration>(builder.Configuration);

// 구체 타입 등록
builder.Services.AddTransient<EmailService>();

var app = builder.Build();
```

### 팩토리를 사용한 등록
```csharp
builder.Services.AddTransient<ILogger>(provider =>
{
    var config = provider.GetRequiredService<IConfiguration>();
    return new FileLogger(config["LogPath"]);
});

builder.Services.AddScoped<IDbConnection>(provider =>
{
    var connectionString = provider.GetRequiredService<IConfiguration>()
        .GetConnectionString("DefaultConnection");
    return new SqlConnection(connectionString);
});
```

### 다중 구현체 등록
```csharp
// 여러 구현체 등록
builder.Services.AddTransient<INotificationService, EmailNotificationService>();
builder.Services.AddTransient<INotificationService, SmsNotificationService>();

// 모든 구현체 주입받기
public class NotificationManager
{
    private readonly IEnumerable<INotificationService> _services;
    
    public NotificationManager(IEnumerable<INotificationService> services)
    {
        _services = services;
    }
    
    public async Task NotifyAll(string message)
    {
        foreach (var service in _services)
        {
            await service.SendAsync(message);
        }
    }
}
```

## 서비스 주입 방법

### 생성자 주입 (권장)
```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _productService;
    private readonly ILogger<ProductsController> _logger;
    
    public ProductsController(
        IProductService productService,
        ILogger<ProductsController> logger)
    {
        _productService = productService;
        _logger = logger;
    }
    
    [HttpGet]
    public async Task<IActionResult> GetAll()
    {
        _logger.LogInformation("Getting all products");
        var products = await _productService.GetAllAsync();
        return Ok(products);
    }
}
```

### Action 메서드 주입
```csharp
[HttpPost]
public async Task<IActionResult> Create(
    [FromBody] Product product,
    [FromServices] IEmailService emailService)
{
    var created = await _productService.CreateAsync(product);
    await emailService.SendAsync("admin@example.com", 
        $"New product created: {product.Name}");
    
    return CreatedAtAction(nameof(GetById), 
        new { id = created.Id }, created);
}
```

### 미들웨어에서 주입
```csharp
public class TenantMiddleware
{
    private readonly RequestDelegate _next;
    
    public TenantMiddleware(RequestDelegate next)
    {
        _next = next;
    }
    
    public async Task InvokeAsync(
        HttpContext context, 
        ITenantService tenantService) // Method injection
    {
        var tenantId = context.Request.Headers["X-Tenant-Id"];
        if (!string.IsNullOrEmpty(tenantId))
        {
            tenantService.SetCurrentTenant(tenantId);
        }
        
        await _next(context);
    }
}
```

## 서비스 구성 패턴

### Options 패턴
```csharp
// 설정 클래스
public class EmailSettings
{
    public string SmtpServer { get; set; }
    public int Port { get; set; }
    public string Username { get; set; }
    public string Password { get; set; }
}

// 등록
builder.Services.Configure<EmailSettings>(
    builder.Configuration.GetSection("Email"));

// 사용
public class EmailService
{
    private readonly EmailSettings _settings;
    
    public EmailService(IOptions<EmailSettings> options)
    {
        _settings = options.Value;
    }
}
```

### 서비스 그룹화
```csharp
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddDataServices(
        this IServiceCollection services, 
        IConfiguration configuration)
    {
        services.AddDbContext<AppDbContext>(options =>
            options.UseSqlServer(
                configuration.GetConnectionString("DefaultConnection")));
        
        services.AddScoped<IUnitOfWork, UnitOfWork>();
        services.AddScoped<IUserRepository, UserRepository>();
        services.AddScoped<IProductRepository, ProductRepository>();
        
        return services;
    }
    
    public static IServiceCollection AddBusinessServices(
        this IServiceCollection services)
    {
        services.AddScoped<IUserService, UserService>();
        services.AddScoped<IProductService, ProductService>();
        services.AddScoped<IOrderService, OrderService>();
        
        return services;
    }
}

// Program.cs
builder.Services.AddDataServices(builder.Configuration);
builder.Services.AddBusinessServices();
```

## 서비스 검증

### 시작 시 검증
```csharp
var app = builder.Build();

// 서비스 해결 가능 여부 확인
if (app.Environment.IsDevelopment())
{
    using (var scope = app.Services.CreateScope())
    {
        var services = scope.ServiceProvider;
        
        try
        {
            var productService = services.GetRequiredService<IProductService>();
            var logger = services.GetRequiredService<ILogger<Program>>();
            logger.LogInformation("All services resolved successfully");
        }
        catch (Exception ex)
        {
            var logger = services.GetRequiredService<ILogger<Program>>();
            logger.LogError(ex, "Service resolution failed");
            throw;
        }
    }
}
```

### 순환 의존성 방지
```csharp
// 잘못된 예 - 순환 의존성
public class ServiceA
{
    public ServiceA(ServiceB serviceB) { }
}

public class ServiceB
{
    public ServiceB(ServiceA serviceA) { } // 순환 의존성!
}

// 해결 방법 1: 인터페이스 사용
public interface IServiceA { }
public interface IServiceB { }

public class ServiceA : IServiceA
{
    public ServiceA(IServiceB serviceB) { }
}

// 해결 방법 2: 팩토리 패턴
public class ServiceA
{
    private readonly Func<ServiceB> _serviceBFactory;
    
    public ServiceA(Func<ServiceB> serviceBFactory)
    {
        _serviceBFactory = serviceBFactory;
    }
}
```

## 고급 시나리오

### 조건부 등록
```csharp
if (builder.Environment.IsDevelopment())
{
    builder.Services.AddTransient<IEmailService, MockEmailService>();
}
else
{
    builder.Services.AddTransient<IEmailService, SmtpEmailService>();
}
```

### 키 기반 서비스
```csharp
public interface IPaymentService
{
    string Provider { get; }
    Task<bool> ProcessPayment(decimal amount);
}

public class PaymentServiceFactory
{
    private readonly IEnumerable<IPaymentService> _services;
    
    public PaymentServiceFactory(IEnumerable<IPaymentService> services)
    {
        _services = services;
    }
    
    public IPaymentService GetService(string provider)
    {
        return _services.FirstOrDefault(s => s.Provider == provider) 
            ?? throw new NotSupportedException($"Provider {provider} not supported");
    }
}

// 등록
builder.Services.AddTransient<IPaymentService, PayPalService>();
builder.Services.AddTransient<IPaymentService, StripeService>();
builder.Services.AddTransient<PaymentServiceFactory>();
```

### 서비스 데코레이터
```csharp
public class CachedProductService : IProductService
{
    private readonly IProductService _innerService;
    private readonly IMemoryCache _cache;
    
    public CachedProductService(
        IProductService innerService, 
        IMemoryCache cache)
    {
        _innerService = innerService;
        _cache = cache;
    }
    
    public async Task<Product> GetByIdAsync(int id)
    {
        var cacheKey = $"product_{id}";
        
        if (!_cache.TryGetValue(cacheKey, out Product product))
        {
            product = await _innerService.GetByIdAsync(id);
            _cache.Set(cacheKey, product, TimeSpan.FromMinutes(5));
        }
        
        return product;
    }
}

// 등록
builder.Services.AddScoped<ProductService>();
builder.Services.AddScoped<IProductService>(provider =>
{
    var innerService = provider.GetRequiredService<ProductService>();
    var cache = provider.GetRequiredService<IMemoryCache>();
    return new CachedProductService(innerService, cache);
});
```

## 실습: 완전한 DI 예제

```csharp
// Domain Models
public class User
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Email { get; set; }
}

// Repository Interface
public interface IUserRepository
{
    Task<User> GetByIdAsync(int id);
    Task<IEnumerable<User>> GetAllAsync();
    Task<User> CreateAsync(User user);
}

// Repository Implementation
public class UserRepository : IUserRepository
{
    private readonly AppDbContext _context;
    
    public UserRepository(AppDbContext context)
    {
        _context = context;
    }
    
    public async Task<User> GetByIdAsync(int id)
    {
        return await _context.Users.FindAsync(id);
    }
    
    public async Task<IEnumerable<User>> GetAllAsync()
    {
        return await _context.Users.ToListAsync();
    }
    
    public async Task<User> CreateAsync(User user)
    {
        _context.Users.Add(user);
        await _context.SaveChangesAsync();
        return user;
    }
}

// Service Interface
public interface IUserService
{
    Task<UserDto> GetUserAsync(int id);
    Task<IEnumerable<UserDto>> GetAllUsersAsync();
    Task<UserDto> CreateUserAsync(CreateUserDto dto);
}

// Service Implementation
public class UserService : IUserService
{
    private readonly IUserRepository _repository;
    private readonly ILogger<UserService> _logger;
    private readonly IMapper _mapper;
    
    public UserService(
        IUserRepository repository,
        ILogger<UserService> logger,
        IMapper mapper)
    {
        _repository = repository;
        _logger = logger;
        _mapper = mapper;
    }
    
    public async Task<UserDto> GetUserAsync(int id)
    {
        _logger.LogInformation($"Getting user {id}");
        var user = await _repository.GetByIdAsync(id);
        return _mapper.Map<UserDto>(user);
    }
    
    public async Task<IEnumerable<UserDto>> GetAllUsersAsync()
    {
        var users = await _repository.GetAllAsync();
        return _mapper.Map<IEnumerable<UserDto>>(users);
    }
    
    public async Task<UserDto> CreateUserAsync(CreateUserDto dto)
    {
        var user = _mapper.Map<User>(dto);
        var created = await _repository.CreateAsync(user);
        _logger.LogInformation($"Created user {created.Id}");
        return _mapper.Map<UserDto>(created);
    }
}

// Program.cs - Service Registration
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

builder.Services.AddAutoMapper(typeof(Program));

builder.Services.AddScoped<IUserRepository, UserRepository>();
builder.Services.AddScoped<IUserService, UserService>();

// Controller
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    private readonly IUserService _userService;
    
    public UsersController(IUserService userService)
    {
        _userService = userService;
    }
    
    [HttpGet("{id}")]
    public async Task<IActionResult> GetUser(int id)
    {
        var user = await _userService.GetUserAsync(id);
        if (user == null) return NotFound();
        return Ok(user);
    }
    
    [HttpPost]
    public async Task<IActionResult> CreateUser(CreateUserDto dto)
    {
        var user = await _userService.CreateUserAsync(dto);
        return CreatedAtAction(nameof(GetUser), new { id = user.Id }, user);
    }
}
```