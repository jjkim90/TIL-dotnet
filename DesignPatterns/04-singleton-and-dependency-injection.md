# Singleton 패턴과 의존성 주입

## 개요

Singleton 패턴은 클래스의 인스턴스가 오직 하나만 생성되도록 보장하는 패턴입니다. 하지만 현대적인 애플리케이션 개발에서는 Singleton 패턴의 단점을 해결하기 위해 의존성 주입(Dependency Injection)을 사용하는 것이 권장됩니다. 이 장에서는 두 접근 방식을 비교하고 올바른 사용법을 알아봅니다.

## 클래식 Singleton 패턴

### 기본 구현

```csharp
public sealed class BasicSingleton
{
    private static BasicSingleton? _instance;
    
    private BasicSingleton()
    {
        // private 생성자로 외부 인스턴스 생성 방지
    }
    
    public static BasicSingleton Instance
    {
        get
        {
            if (_instance == null)
            {
                _instance = new BasicSingleton();
            }
            return _instance;
        }
    }
    
    public void DoSomething()
    {
        Console.WriteLine("Singleton method called");
    }
}

// 사용
var singleton = BasicSingleton.Instance;
singleton.DoSomething();
```

### Thread-Safe 구현

```csharp
// 방법 1: Lock을 사용한 구현
public sealed class ThreadSafeSingleton
{
    private static ThreadSafeSingleton? _instance;
    private static readonly object _lock = new object();
    
    private ThreadSafeSingleton() { }
    
    public static ThreadSafeSingleton Instance
    {
        get
        {
            if (_instance == null)
            {
                lock (_lock)
                {
                    if (_instance == null)
                    {
                        _instance = new ThreadSafeSingleton();
                    }
                }
            }
            return _instance;
        }
    }
}

// 방법 2: Lazy<T>를 사용한 구현 (권장)
public sealed class LazySingleton
{
    private static readonly Lazy<LazySingleton> _lazy = 
        new Lazy<LazySingleton>(() => new LazySingleton());
    
    private LazySingleton() 
    {
        Console.WriteLine("Singleton instance created");
    }
    
    public static LazySingleton Instance => _lazy.Value;
    
    public string ConnectionString { get; set; } = "Default Connection";
}

// 방법 3: Static 생성자를 활용한 구현
public sealed class StaticSingleton
{
    private static readonly StaticSingleton _instance = new StaticSingleton();
    
    static StaticSingleton()
    {
        // 타입 초기화 시 한 번만 실행
    }
    
    private StaticSingleton() { }
    
    public static StaticSingleton Instance => _instance;
}
```

### 실용적인 Singleton 예제

```csharp
// 설정 관리자 Singleton
public sealed class ConfigurationManager
{
    private static readonly Lazy<ConfigurationManager> _lazy = 
        new Lazy<ConfigurationManager>(() => new ConfigurationManager());
    
    private readonly Dictionary<string, string> _settings;
    
    private ConfigurationManager()
    {
        _settings = LoadSettings();
    }
    
    public static ConfigurationManager Instance => _lazy.Value;
    
    public string GetSetting(string key)
    {
        return _settings.TryGetValue(key, out var value) ? value : string.Empty;
    }
    
    public void UpdateSetting(string key, string value)
    {
        _settings[key] = value;
        SaveSettings();
    }
    
    private Dictionary<string, string> LoadSettings()
    {
        // 파일이나 데이터베이스에서 설정 로드
        return new Dictionary<string, string>
        {
            ["DatabaseConnection"] = "Server=localhost;Database=MyDb;",
            ["ApiKey"] = "abc123",
            ["MaxRetries"] = "3"
        };
    }
    
    private void SaveSettings()
    {
        // 설정 저장 로직
    }
}

// 로거 Singleton
public sealed class Logger
{
    private static readonly Lazy<Logger> _lazy = 
        new Lazy<Logger>(() => new Logger());
    
    private readonly string _logPath;
    private readonly object _lockObject = new object();
    
    private Logger()
    {
        _logPath = Path.Combine(
            Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData),
            "MyApp",
            "app.log");
            
        Directory.CreateDirectory(Path.GetDirectoryName(_logPath)!);
    }
    
    public static Logger Instance => _lazy.Value;
    
    public void Log(string message, LogLevel level = LogLevel.Info)
    {
        var logEntry = $"[{DateTime.Now:yyyy-MM-dd HH:mm:ss}] [{level}] {message}";
        
        lock (_lockObject)
        {
            File.AppendAllText(_logPath, logEntry + Environment.NewLine);
        }
    }
    
    public enum LogLevel
    {
        Debug,
        Info,
        Warning,
        Error
    }
}
```

## Singleton 패턴의 문제점

### 1. 단위 테스트의 어려움

```csharp
// Singleton을 사용하는 클래스 - 테스트하기 어려움
public class OrderService
{
    public void ProcessOrder(Order order)
    {
        // Singleton에 직접 의존
        var connectionString = ConfigurationManager.Instance.GetSetting("DatabaseConnection");
        Logger.Instance.Log($"Processing order {order.Id}");
        
        // 데이터베이스 작업...
        
        Logger.Instance.Log($"Order {order.Id} processed");
    }
}

// 테스트 코드
[TestClass]
public class OrderServiceTests
{
    [TestMethod]
    public void ProcessOrder_Should_LogMessages()
    {
        // 문제: Logger.Instance를 Mock으로 대체할 수 없음
        // ConfigurationManager.Instance도 Mock으로 대체할 수 없음
        
        var orderService = new OrderService();
        var order = new Order { Id = 123 };
        
        // 실제 Logger와 ConfigurationManager가 사용됨
        orderService.ProcessOrder(order);
        
        // 로그가 제대로 작성되었는지 검증하기 어려움
    }
}
```

### 2. 전역 상태의 문제

```csharp
public class GlobalStateProblem
{
    // Singleton이 전역 상태를 가질 때의 문제
    public sealed class UserSession
    {
        private static readonly Lazy<UserSession> _lazy = 
            new Lazy<UserSession>(() => new UserSession());
        
        public static UserSession Instance => _lazy.Value;
        
        public User? CurrentUser { get; set; }
        public DateTime LoginTime { get; set; }
        
        private UserSession() { }
    }
    
    // 여러 곳에서 전역 상태에 의존
    public class ShoppingCart
    {
        public void AddItem(Product product)
        {
            var user = UserSession.Instance.CurrentUser;
            if (user == null)
            {
                throw new InvalidOperationException("User not logged in");
            }
            
            // 장바구니에 상품 추가...
        }
    }
    
    public class OrderProcessor
    {
        public void CreateOrder()
        {
            var user = UserSession.Instance.CurrentUser;
            // 전역 상태에 의존하는 로직...
        }
    }
}
```

## 의존성 주입 (Dependency Injection)

### DI의 기본 개념

```csharp
// 인터페이스 정의
public interface ILogger
{
    void Log(string message, LogLevel level = LogLevel.Info);
}

public interface IConfiguration
{
    string GetSetting(string key);
    void UpdateSetting(string key, string value);
}

// 구현체
public class FileLogger : ILogger
{
    private readonly string _logPath;
    
    public FileLogger(string logPath)
    {
        _logPath = logPath;
    }
    
    public void Log(string message, LogLevel level = LogLevel.Info)
    {
        var logEntry = $"[{DateTime.Now:yyyy-MM-dd HH:mm:ss}] [{level}] {message}";
        File.AppendAllText(_logPath, logEntry + Environment.NewLine);
    }
}

public class AppConfiguration : IConfiguration
{
    private readonly Dictionary<string, string> _settings;
    
    public AppConfiguration(Dictionary<string, string> settings)
    {
        _settings = settings;
    }
    
    public string GetSetting(string key)
    {
        return _settings.TryGetValue(key, out var value) ? value : string.Empty;
    }
    
    public void UpdateSetting(string key, string value)
    {
        _settings[key] = value;
    }
}

// DI를 사용하는 서비스
public class OrderServiceWithDI
{
    private readonly ILogger _logger;
    private readonly IConfiguration _configuration;
    
    public OrderServiceWithDI(ILogger logger, IConfiguration configuration)
    {
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
        _configuration = configuration ?? throw new ArgumentNullException(nameof(configuration));
    }
    
    public void ProcessOrder(Order order)
    {
        var connectionString = _configuration.GetSetting("DatabaseConnection");
        _logger.Log($"Processing order {order.Id}");
        
        // 데이터베이스 작업...
        
        _logger.Log($"Order {order.Id} processed");
    }
}
```

### .NET DI Container 사용

```csharp
// Program.cs (ASP.NET Core)
var builder = WebApplication.CreateBuilder(args);

// 서비스 등록
builder.Services.AddSingleton<ILogger, FileLogger>(provider =>
    new FileLogger(Path.Combine(builder.Environment.ContentRootPath, "logs", "app.log")));

builder.Services.AddSingleton<IConfiguration, AppConfiguration>(provider =>
    new AppConfiguration(new Dictionary<string, string>
    {
        ["DatabaseConnection"] = builder.Configuration.GetConnectionString("DefaultConnection"),
        ["ApiKey"] = builder.Configuration["ApiKey"]
    }));

builder.Services.AddScoped<IOrderService, OrderServiceWithDI>();

// 더 복잡한 등록 예제
builder.Services.AddSingleton<IEmailService>(provider =>
{
    var config = provider.GetRequiredService<IConfiguration>();
    var smtpServer = config.GetSetting("SmtpServer");
    var smtpPort = int.Parse(config.GetSetting("SmtpPort"));
    
    return new SmtpEmailService(smtpServer, smtpPort);
});

// Options 패턴 사용
builder.Services.Configure<DatabaseOptions>(
    builder.Configuration.GetSection("Database"));

builder.Services.AddSingleton<IDatabaseConnection>(provider =>
{
    var options = provider.GetRequiredService<IOptions<DatabaseOptions>>();
    return new SqlDatabaseConnection(options.Value.ConnectionString);
});

var app = builder.Build();
```

### DI 생명주기 관리

```csharp
public class ServiceLifetimeExample
{
    // Transient: 매번 새 인스턴스 생성
    public interface ITransientService
    {
        Guid Id { get; }
    }
    
    public class TransientService : ITransientService
    {
        public Guid Id { get; } = Guid.NewGuid();
    }
    
    // Scoped: 요청당 하나의 인스턴스
    public interface IScopedService
    {
        Guid Id { get; }
        void AddItem(string item);
        List<string> GetItems();
    }
    
    public class ScopedService : IScopedService
    {
        public Guid Id { get; } = Guid.NewGuid();
        private readonly List<string> _items = new();
        
        public void AddItem(string item) => _items.Add(item);
        public List<string> GetItems() => _items.ToList();
    }
    
    // Singleton: 애플리케이션 생명주기 동안 하나의 인스턴스
    public interface ISingletonService
    {
        Guid Id { get; }
        int CallCount { get; }
        void IncrementCallCount();
    }
    
    public class SingletonService : ISingletonService
    {
        public Guid Id { get; } = Guid.NewGuid();
        public int CallCount { get; private set; }
        
        public void IncrementCallCount()
        {
            Interlocked.Increment(ref _callCount);
        }
        
        private int _callCount;
    }
    
    // 서비스 등록
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddTransient<ITransientService, TransientService>();
        services.AddScoped<IScopedService, ScopedService>();
        services.AddSingleton<ISingletonService, SingletonService>();
    }
}
```

### Factory 패턴과 DI 결합

```csharp
// Factory 인터페이스
public interface IRepositoryFactory
{
    IRepository<T> CreateRepository<T>() where T : class, IEntity;
}

// Factory 구현
public class RepositoryFactory : IRepositoryFactory
{
    private readonly IServiceProvider _serviceProvider;
    
    public RepositoryFactory(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }
    
    public IRepository<T> CreateRepository<T>() where T : class, IEntity
    {
        var dbContext = _serviceProvider.GetRequiredService<ApplicationDbContext>();
        var logger = _serviceProvider.GetRequiredService<ILogger<Repository<T>>>();
        
        return new Repository<T>(dbContext, logger);
    }
}

// Named 서비스 구현
public interface IPaymentGateway
{
    Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request);
}

public class PaymentGatewayFactory
{
    private readonly IServiceProvider _serviceProvider;
    
    public PaymentGatewayFactory(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }
    
    public IPaymentGateway GetPaymentGateway(string gatewayName)
    {
        return gatewayName switch
        {
            "Stripe" => _serviceProvider.GetRequiredService<StripePaymentGateway>(),
            "PayPal" => _serviceProvider.GetRequiredService<PayPalPaymentGateway>(),
            "Square" => _serviceProvider.GetRequiredService<SquarePaymentGateway>(),
            _ => throw new NotSupportedException($"Payment gateway {gatewayName} is not supported")
        };
    }
}

// 서비스 등록
services.AddSingleton<IRepositoryFactory, RepositoryFactory>();
services.AddSingleton<PaymentGatewayFactory>();
services.AddTransient<StripePaymentGateway>();
services.AddTransient<PayPalPaymentGateway>();
services.AddTransient<SquarePaymentGateway>();
```

## 테스트 가능한 코드 작성

### Mock을 사용한 단위 테스트

```csharp
[TestClass]
public class OrderServiceTests
{
    private Mock<ILogger> _loggerMock;
    private Mock<IConfiguration> _configMock;
    private OrderServiceWithDI _orderService;
    
    [TestInitialize]
    public void Setup()
    {
        _loggerMock = new Mock<ILogger>();
        _configMock = new Mock<IConfiguration>();
        
        _configMock.Setup(c => c.GetSetting("DatabaseConnection"))
            .Returns("TestConnectionString");
            
        _orderService = new OrderServiceWithDI(
            _loggerMock.Object, 
            _configMock.Object);
    }
    
    [TestMethod]
    public void ProcessOrder_Should_LogStartAndEnd()
    {
        // Arrange
        var order = new Order { Id = 123 };
        
        // Act
        _orderService.ProcessOrder(order);
        
        // Assert
        _loggerMock.Verify(
            l => l.Log(
                It.Is<string>(msg => msg.Contains("Processing order 123")), 
                It.IsAny<LogLevel>()), 
            Times.Once);
            
        _loggerMock.Verify(
            l => l.Log(
                It.Is<string>(msg => msg.Contains("Order 123 processed")), 
                It.IsAny<LogLevel>()), 
            Times.Once);
    }
    
    [TestMethod]
    public void ProcessOrder_Should_UseCorrectConnectionString()
    {
        // Arrange
        var order = new Order { Id = 456 };
        
        // Act
        _orderService.ProcessOrder(order);
        
        // Assert
        _configMock.Verify(
            c => c.GetSetting("DatabaseConnection"), 
            Times.Once);
    }
}
```

### Integration Test with Test Container

```csharp
public class IntegrationTestBase
{
    protected IServiceProvider ServiceProvider { get; private set; }
    
    [TestInitialize]
    public void BaseSetup()
    {
        var services = new ServiceCollection();
        
        // 테스트용 서비스 등록
        services.AddSingleton<ILogger, TestLogger>();
        services.AddSingleton<IConfiguration>(new TestConfiguration
        {
            Settings = new Dictionary<string, string>
            {
                ["DatabaseConnection"] = "TestDb",
                ["ApiKey"] = "TestKey"
            }
        });
        
        ConfigureServices(services);
        
        ServiceProvider = services.BuildServiceProvider();
    }
    
    protected virtual void ConfigureServices(IServiceCollection services)
    {
        // 파생 클래스에서 추가 서비스 등록
    }
    
    protected T GetService<T>() where T : notnull
    {
        return ServiceProvider.GetRequiredService<T>();
    }
}

[TestClass]
public class OrderServiceIntegrationTests : IntegrationTestBase
{
    protected override void ConfigureServices(IServiceCollection services)
    {
        services.AddScoped<IOrderService, OrderServiceWithDI>();
    }
    
    [TestMethod]
    public void ProcessOrder_IntegrationTest()
    {
        // Arrange
        var orderService = GetService<IOrderService>();
        var order = new Order { Id = 789 };
        
        // Act
        orderService.ProcessOrder(order);
        
        // Assert
        var logger = GetService<ILogger>() as TestLogger;
        Assert.IsTrue(logger!.Logs.Any(log => log.Contains("Processing order 789")));
    }
}
```

## 실제 프로젝트에서의 적용

### 설정 관리

```csharp
// appsettings.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MyApp;"
  },
  "AppSettings": {
    "MaxRetryCount": 3,
    "CacheExpirationMinutes": 60
  }
}

// Options 클래스
public class AppSettings
{
    public int MaxRetryCount { get; set; }
    public int CacheExpirationMinutes { get; set; }
}

// 서비스에서 Options 사용
public class CacheService
{
    private readonly IMemoryCache _cache;
    private readonly AppSettings _settings;
    
    public CacheService(
        IMemoryCache cache, 
        IOptions<AppSettings> options)
    {
        _cache = cache;
        _settings = options.Value;
    }
    
    public async Task<T?> GetOrSetAsync<T>(
        string key, 
        Func<Task<T>> factory)
    {
        if (_cache.TryGetValue(key, out T? cached))
        {
            return cached;
        }
        
        var value = await factory();
        
        var cacheOptions = new MemoryCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = 
                TimeSpan.FromMinutes(_settings.CacheExpirationMinutes)
        };
        
        _cache.Set(key, value, cacheOptions);
        
        return value;
    }
}
```

### Http Client Factory

```csharp
// HttpClient를 Singleton으로 사용하는 것보다 IHttpClientFactory 사용 권장
public interface IApiClient
{
    Task<T?> GetAsync<T>(string endpoint);
    Task<TResponse?> PostAsync<TRequest, TResponse>(string endpoint, TRequest data);
}

public class ApiClient : IApiClient
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<ApiClient> _logger;
    
    public ApiClient(HttpClient httpClient, ILogger<ApiClient> logger)
    {
        _httpClient = httpClient;
        _logger = logger;
    }
    
    public async Task<T?> GetAsync<T>(string endpoint)
    {
        try
        {
            var response = await _httpClient.GetAsync(endpoint);
            response.EnsureSuccessStatusCode();
            
            var json = await response.Content.ReadAsStringAsync();
            return JsonSerializer.Deserialize<T>(json);
        }
        catch (HttpRequestException ex)
        {
            _logger.LogError(ex, "Error calling API endpoint: {Endpoint}", endpoint);
            return default;
        }
    }
    
    public async Task<TResponse?> PostAsync<TRequest, TResponse>(
        string endpoint, 
        TRequest data)
    {
        try
        {
            var json = JsonSerializer.Serialize(data);
            var content = new StringContent(json, Encoding.UTF8, "application/json");
            
            var response = await _httpClient.PostAsync(endpoint, content);
            response.EnsureSuccessStatusCode();
            
            var responseJson = await response.Content.ReadAsStringAsync();
            return JsonSerializer.Deserialize<TResponse>(responseJson);
        }
        catch (HttpRequestException ex)
        {
            _logger.LogError(ex, "Error posting to API endpoint: {Endpoint}", endpoint);
            return default;
        }
    }
}

// 서비스 등록
services.AddHttpClient<IApiClient, ApiClient>(client =>
{
    client.BaseAddress = new Uri("https://api.example.com/");
    client.DefaultRequestHeaders.Add("User-Agent", "MyApp/1.0");
    client.Timeout = TimeSpan.FromSeconds(30);
})
.AddPolicyHandler(GetRetryPolicy())
.AddPolicyHandler(GetCircuitBreakerPolicy());

// Polly를 사용한 재시도 정책
static IAsyncPolicy<HttpResponseMessage> GetRetryPolicy()
{
    return HttpPolicyExtensions
        .HandleTransientHttpError()
        .WaitAndRetryAsync(
            3,
            retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)),
            onRetry: (outcome, timespan, retryCount, context) =>
            {
                var logger = context.Values["logger"] as ILogger;
                logger?.LogWarning(
                    "Delaying for {delay}ms, then making retry {retry}.",
                    timespan.TotalMilliseconds,
                    retryCount);
            });
}

// Circuit Breaker 정책
static IAsyncPolicy<HttpResponseMessage> GetCircuitBreakerPolicy()
{
    return HttpPolicyExtensions
        .HandleTransientHttpError()
        .CircuitBreakerAsync(
            5,
            TimeSpan.FromSeconds(30),
            onBreak: (result, duration) =>
            {
                // Circuit opened
            },
            onReset: () =>
            {
                // Circuit closed
            });
}
```

## 마무리

Singleton 패턴과 의존성 주입의 주요 차이점:

1. **Singleton 패턴**
   - 장점: 구현이 간단, 전역 접근 가능
   - 단점: 테스트 어려움, 강한 결합, 전역 상태 문제

2. **의존성 주입**
   - 장점: 테스트 용이, 느슨한 결합, 유연한 구성
   - 단점: 초기 설정 복잡, DI 컨테이너 학습 필요

현대적인 .NET 애플리케이션에서는 의존성 주입을 사용하여 Singleton의 이점을 얻으면서도 단점을 피하는 것이 권장됩니다. 다음 장에서는 Factory 패턴들에 대해 알아보겠습니다.