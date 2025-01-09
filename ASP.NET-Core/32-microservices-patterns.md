# Microservices Patterns

## 마이크로서비스 아키텍처 소개

마이크로서비스 아키텍처는 애플리케이션을 작고 독립적인 서비스의 집합으로 구성하는 소프트웨어 개발 접근 방식입니다. 각 서비스는 특정 비즈니스 기능을 담당하며, 독립적으로 개발, 배포, 확장할 수 있습니다.

### 마이크로서비스의 특징

1. **비즈니스 중심**: 각 서비스는 특정 비즈니스 기능을 담당
2. **독립적 배포**: 서비스별로 독립적인 배포 가능
3. **기술 다양성**: 서비스별로 적합한 기술 스택 선택 가능
4. **분산 시스템**: 네트워크를 통한 서비스 간 통신
5. **장애 격리**: 한 서비스의 장애가 전체 시스템에 영향을 미치지 않음

## API Gateway 패턴

API Gateway는 모든 클라이언트 요청의 단일 진입점 역할을 합니다.

### Ocelot을 사용한 API Gateway

```csharp
// Gateway/Program.cs
var builder = WebApplication.CreateBuilder(args);

// Ocelot 설정
builder.Configuration.AddJsonFile("ocelot.json", optional: false, reloadOnChange: true);
builder.Services.AddOcelot();

// 인증 추가
builder.Services.AddAuthentication("Bearer")
    .AddJwtBearer("Bearer", options =>
    {
        options.Authority = "https://localhost:5001";
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateAudience = false
        };
    });

// CORS
builder.Services.AddCors(options =>
{
    options.AddPolicy("CorsPolicy",
        builder => builder
            .AllowAnyOrigin()
            .AllowAnyMethod()
            .AllowAnyHeader());
});

var app = builder.Build();

app.UseCors("CorsPolicy");
app.UseAuthentication();
await app.UseOcelot();

app.Run();
```

### Ocelot 설정 파일

```json
// ocelot.json
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/products/{everything}",
      "DownstreamScheme": "https",
      "DownstreamHostAndPorts": [
        {
          "Host": "localhost",
          "Port": 5002
        }
      ],
      "UpstreamPathTemplate": "/api/products/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE" ],
      "AuthenticationOptions": {
        "AuthenticationProviderKey": "Bearer",
        "AllowedScopes": []
      },
      "RateLimitOptions": {
        "ClientWhitelist": [],
        "EnableRateLimiting": true,
        "Period": "1s",
        "PeriodTimespan": 1,
        "Limit": 10
      }
    },
    {
      "DownstreamPathTemplate": "/api/orders/{everything}",
      "DownstreamScheme": "https",
      "DownstreamHostAndPorts": [
        {
          "Host": "localhost",
          "Port": 5003
        }
      ],
      "UpstreamPathTemplate": "/api/orders/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE" ],
      "LoadBalancerOptions": {
        "Type": "RoundRobin"
      }
    }
  ],
  "GlobalConfiguration": {
    "BaseUrl": "https://localhost:5000"
  }
}
```

## Service Discovery 패턴

서비스가 서로를 동적으로 찾을 수 있도록 하는 패턴입니다.

### Consul을 사용한 Service Discovery

```csharp
// Shared/ServiceDiscovery/ConsulConfig.cs
public class ConsulConfig
{
    public string ServiceId { get; set; }
    public string ServiceName { get; set; }
    public string ServiceHost { get; set; }
    public int ServicePort { get; set; }
    public string ConsulAddress { get; set; }
}

// Shared/ServiceDiscovery/ConsulHostedService.cs
public class ConsulHostedService : IHostedService
{
    private readonly IConsulClient _consulClient;
    private readonly ConsulConfig _config;
    private readonly ILogger<ConsulHostedService> _logger;
    private string _registrationId;
    
    public ConsulHostedService(
        IConsulClient consulClient,
        ConsulConfig config,
        ILogger<ConsulHostedService> logger)
    {
        _consulClient = consulClient;
        _config = config;
        _logger = logger;
    }
    
    public async Task StartAsync(CancellationToken cancellationToken)
    {
        _registrationId = $"{_config.ServiceName}-{_config.ServiceId}";
        
        var registration = new AgentServiceRegistration
        {
            ID = _registrationId,
            Name = _config.ServiceName,
            Address = _config.ServiceHost,
            Port = _config.ServicePort,
            Tags = new[] { "api", "microservice" },
            Check = new AgentServiceCheck
            {
                DeregisterCriticalServiceAfter = TimeSpan.FromSeconds(30),
                Interval = TimeSpan.FromSeconds(10),
                HTTP = $"http://{_config.ServiceHost}:{_config.ServicePort}/health",
                Timeout = TimeSpan.FromSeconds(5)
            }
        };
        
        await _consulClient.Agent.ServiceDeregister(_registrationId, cancellationToken);
        await _consulClient.Agent.ServiceRegister(registration, cancellationToken);
        
        _logger.LogInformation($"Service {_config.ServiceName} registered with Consul");
    }
    
    public async Task StopAsync(CancellationToken cancellationToken)
    {
        await _consulClient.Agent.ServiceDeregister(_registrationId, cancellationToken);
        _logger.LogInformation($"Service {_config.ServiceName} deregistered from Consul");
    }
}

// Program.cs에 등록
builder.Services.AddSingleton<IConsulClient>(p => new ConsulClient(config =>
{
    config.Address = new Uri(builder.Configuration["Consul:Address"]);
}));

builder.Services.AddSingleton<ConsulConfig>(p => 
    builder.Configuration.GetSection("Consul").Get<ConsulConfig>());
    
builder.Services.AddHostedService<ConsulHostedService>();
```

## Circuit Breaker 패턴

장애가 발생한 서비스에 대한 요청을 차단하여 시스템 안정성을 보장합니다.

### Polly를 사용한 Circuit Breaker

```csharp
// Services/ResilientHttpClient.cs
public interface IResilientHttpClient
{
    Task<HttpResponseMessage> GetAsync(string uri);
    Task<HttpResponseMessage> PostAsync<T>(string uri, T item);
}

public class ResilientHttpClient : IResilientHttpClient
{
    private readonly HttpClient _httpClient;
    private readonly IAsyncPolicy<HttpResponseMessage> _retryPolicy;
    private readonly IAsyncPolicy<HttpResponseMessage> _circuitBreakerPolicy;
    private readonly IAsyncPolicy<HttpResponseMessage> _combinedPolicy;
    
    public ResilientHttpClient(HttpClient httpClient)
    {
        _httpClient = httpClient;
        
        // 재시도 정책
        _retryPolicy = HttpPolicyExtensions
            .HandleTransientHttpError()
            .WaitAndRetryAsync(
                3,
                retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)),
                onRetry: (outcome, timespan, retryCount, context) =>
                {
                    var logger = context.Values.ContainsKey("logger") 
                        ? context.Values["logger"] as ILogger 
                        : null;
                    logger?.LogWarning($"Retry {retryCount} after {timespan} seconds");
                });
                
        // Circuit Breaker 정책
        _circuitBreakerPolicy = HttpPolicyExtensions
            .HandleTransientHttpError()
            .CircuitBreakerAsync(
                5,  // 5번 연속 실패 시 차단
                TimeSpan.FromSeconds(30),  // 30초간 차단
                onBreak: (result, timespan) =>
                {
                    Console.WriteLine($"Circuit breaker opened for {timespan}");
                },
                onReset: () =>
                {
                    Console.WriteLine("Circuit breaker reset");
                });
                
        // 정책 조합
        _combinedPolicy = Policy.WrapAsync(_retryPolicy, _circuitBreakerPolicy);
    }
    
    public async Task<HttpResponseMessage> GetAsync(string uri)
    {
        return await _combinedPolicy.ExecuteAsync(async () => 
            await _httpClient.GetAsync(uri));
    }
    
    public async Task<HttpResponseMessage> PostAsync<T>(string uri, T item)
    {
        var json = JsonSerializer.Serialize(item);
        var content = new StringContent(json, Encoding.UTF8, "application/json");
        
        return await _combinedPolicy.ExecuteAsync(async () => 
            await _httpClient.PostAsync(uri, content));
    }
}

// DI 등록
builder.Services.AddHttpClient<IResilientHttpClient, ResilientHttpClient>()
    .SetHandlerLifetime(TimeSpan.FromMinutes(5))
    .AddPolicyHandler(GetRetryPolicy())
    .AddPolicyHandler(GetCircuitBreakerPolicy());
```

## Saga 패턴

분산 트랜잭션을 관리하는 패턴으로, 각 서비스의 로컬 트랜잭션을 연결합니다.

### Choreography 기반 Saga

```csharp
// OrderService/Events/OrderEvents.cs
public class OrderCreatedEvent
{
    public Guid OrderId { get; set; }
    public Guid CustomerId { get; set; }
    public List<OrderItem> Items { get; set; }
    public decimal TotalAmount { get; set; }
}

public class PaymentProcessedEvent
{
    public Guid OrderId { get; set; }
    public Guid PaymentId { get; set; }
    public bool IsSuccessful { get; set; }
    public string FailureReason { get; set; }
}

// OrderService/Handlers/OrderSaga.cs
public class OrderSaga : 
    INotificationHandler<OrderCreatedEvent>,
    INotificationHandler<PaymentProcessedEvent>,
    INotificationHandler<InventoryReservedEvent>
{
    private readonly IOrderRepository _orderRepository;
    private readonly IEventBus _eventBus;
    
    public OrderSaga(IOrderRepository orderRepository, IEventBus eventBus)
    {
        _orderRepository = orderRepository;
        _eventBus = eventBus;
    }
    
    public async Task Handle(OrderCreatedEvent notification, CancellationToken cancellationToken)
    {
        // 1. 재고 예약 요청
        await _eventBus.PublishAsync(new ReserveInventoryCommand
        {
            OrderId = notification.OrderId,
            Items = notification.Items
        });
    }
    
    public async Task Handle(InventoryReservedEvent notification, CancellationToken cancellationToken)
    {
        if (notification.IsSuccessful)
        {
            // 2. 결제 처리 요청
            var order = await _orderRepository.GetByIdAsync(notification.OrderId);
            await _eventBus.PublishAsync(new ProcessPaymentCommand
            {
                OrderId = notification.OrderId,
                Amount = order.TotalAmount,
                CustomerId = order.CustomerId
            });
        }
        else
        {
            // 재고 부족 - 주문 취소
            await CancelOrder(notification.OrderId, "재고 부족");
        }
    }
    
    public async Task Handle(PaymentProcessedEvent notification, CancellationToken cancellationToken)
    {
        if (notification.IsSuccessful)
        {
            // 3. 주문 확정
            await ConfirmOrder(notification.OrderId);
            
            // 배송 요청
            await _eventBus.PublishAsync(new CreateShipmentCommand
            {
                OrderId = notification.OrderId
            });
        }
        else
        {
            // 결제 실패 - 보상 트랜잭션
            await _eventBus.PublishAsync(new ReleaseInventoryCommand
            {
                OrderId = notification.OrderId
            });
            
            await CancelOrder(notification.OrderId, notification.FailureReason);
        }
    }
    
    private async Task ConfirmOrder(Guid orderId)
    {
        var order = await _orderRepository.GetByIdAsync(orderId);
        order.Confirm();
        await _orderRepository.UpdateAsync(order);
    }
    
    private async Task CancelOrder(Guid orderId, string reason)
    {
        var order = await _orderRepository.GetByIdAsync(orderId);
        order.Cancel(reason);
        await _orderRepository.UpdateAsync(order);
    }
}
```

## Event Sourcing과 CQRS

마이크로서비스 간 데이터 일관성을 위한 패턴입니다.

```csharp
// Shared/EventStore/IEventStore.cs
public interface IEventStore
{
    Task AppendEventAsync<T>(string stream, T @event) where T : class;
    Task<IEnumerable<T>> ReadEventsAsync<T>(string stream) where T : class;
}

// ProductService/Commands/CreateProductCommandHandler.cs
public class CreateProductCommandHandler : IRequestHandler<CreateProductCommand, Guid>
{
    private readonly IEventStore _eventStore;
    private readonly IEventBus _eventBus;
    
    public CreateProductCommandHandler(IEventStore eventStore, IEventBus eventBus)
    {
        _eventStore = eventStore;
        _eventBus = eventBus;
    }
    
    public async Task<Guid> Handle(CreateProductCommand request, CancellationToken cancellationToken)
    {
        var productId = Guid.NewGuid();
        
        var @event = new ProductCreatedEvent
        {
            ProductId = productId,
            Name = request.Name,
            Price = request.Price,
            Category = request.Category,
            CreatedAt = DateTime.UtcNow
        };
        
        // 이벤트 저장
        await _eventStore.AppendEventAsync($"product-{productId}", @event);
        
        // 다른 서비스에 이벤트 발행
        await _eventBus.PublishAsync(@event);
        
        return productId;
    }
}

// InventoryService/Handlers/ProductCreatedEventHandler.cs
public class ProductCreatedEventHandler : INotificationHandler<ProductCreatedEvent>
{
    private readonly IInventoryRepository _inventoryRepository;
    
    public async Task Handle(ProductCreatedEvent notification, CancellationToken cancellationToken)
    {
        // 새 상품에 대한 재고 레코드 생성
        var inventory = new Inventory
        {
            ProductId = notification.ProductId,
            Quantity = 0,
            ReorderLevel = 10,
            LastUpdated = DateTime.UtcNow
        };
        
        await _inventoryRepository.CreateAsync(inventory);
    }
}
```

## Distributed Caching

여러 서비스 간 캐시 공유를 위한 패턴입니다.

```csharp
// Shared/Caching/IDistributedCacheService.cs
public interface IDistributedCacheService
{
    Task<T?> GetAsync<T>(string key);
    Task SetAsync<T>(string key, T value, TimeSpan? expiry = null);
    Task RemoveAsync(string key);
}

// Shared/Caching/RedisCacheService.cs
public class RedisCacheService : IDistributedCacheService
{
    private readonly IConnectionMultiplexer _redis;
    private readonly IDatabase _database;
    private readonly JsonSerializerOptions _jsonOptions;
    
    public RedisCacheService(IConnectionMultiplexer redis)
    {
        _redis = redis;
        _database = _redis.GetDatabase();
        _jsonOptions = new JsonSerializerOptions
        {
            PropertyNamingPolicy = JsonNamingPolicy.CamelCase
        };
    }
    
    public async Task<T?> GetAsync<T>(string key)
    {
        var value = await _database.StringGetAsync(key);
        
        if (value.IsNullOrEmpty)
            return default;
            
        return JsonSerializer.Deserialize<T>(value, _jsonOptions);
    }
    
    public async Task SetAsync<T>(string key, T value, TimeSpan? expiry = null)
    {
        var json = JsonSerializer.Serialize(value, _jsonOptions);
        await _database.StringSetAsync(key, json, expiry);
    }
    
    public async Task RemoveAsync(string key)
    {
        await _database.KeyDeleteAsync(key);
    }
}

// 사용 예시
public class ProductService
{
    private readonly IDistributedCacheService _cache;
    private readonly IProductRepository _repository;
    
    public async Task<Product?> GetProductAsync(Guid productId)
    {
        var cacheKey = $"product:{productId}";
        
        // 캐시 확인
        var cached = await _cache.GetAsync<Product>(cacheKey);
        if (cached != null)
            return cached;
            
        // DB 조회
        var product = await _repository.GetByIdAsync(productId);
        if (product != null)
        {
            // 캐시 저장
            await _cache.SetAsync(cacheKey, product, TimeSpan.FromMinutes(5));
        }
        
        return product;
    }
}
```

## Distributed Logging

모든 서비스의 로그를 중앙에서 수집하고 분석합니다.

```csharp
// Shared/Logging/CorrelationIdMiddleware.cs
public class CorrelationIdMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<CorrelationIdMiddleware> _logger;
    
    public CorrelationIdMiddleware(RequestDelegate next, ILogger<CorrelationIdMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        var correlationId = GetOrCreateCorrelationId(context);
        
        using (_logger.BeginScope(new Dictionary<string, object>
        {
            ["CorrelationId"] = correlationId
        }))
        {
            context.Response.OnStarting(() =>
            {
                context.Response.Headers.Add("X-Correlation-Id", correlationId);
                return Task.CompletedTask;
            });
            
            await _next(context);
        }
    }
    
    private string GetOrCreateCorrelationId(HttpContext context)
    {
        if (context.Request.Headers.TryGetValue("X-Correlation-Id", out var correlationId))
        {
            return correlationId;
        }
        
        return Guid.NewGuid().ToString();
    }
}

// Program.cs - Serilog 설정
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .Enrich.FromLogContext()
    .Enrich.WithProperty("ServiceName", "ProductService")
    .Enrich.WithMachineName()
    .WriteTo.Console(new JsonFormatter())
    .WriteTo.Elasticsearch(new ElasticsearchSinkOptions(new Uri("http://localhost:9200"))
    {
        AutoRegisterTemplate = true,
        IndexFormat = "microservices-logs-{0:yyyy.MM.dd}"
    })
    .CreateLogger();

builder.Host.UseSerilog();
```

## Health Checks

서비스의 상태를 모니터링하는 패턴입니다.

```csharp
// Program.cs
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy())
    .AddSqlServer(
        builder.Configuration.GetConnectionString("DefaultConnection"),
        name: "database",
        tags: new[] { "db", "sql" })
    .AddRedis(
        builder.Configuration.GetConnectionString("Redis"),
        name: "redis",
        tags: new[] { "cache" })
    .AddUrlGroup(
        new Uri("https://localhost:5002/health"),
        name: "product-service",
        tags: new[] { "service" })
    .AddCheck<CustomHealthCheck>("custom");

// CustomHealthCheck.cs
public class CustomHealthCheck : IHealthCheck
{
    private readonly IServiceProvider _serviceProvider;
    
    public CustomHealthCheck(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }
    
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            // 커스텀 상태 확인 로직
            using var scope = _serviceProvider.CreateScope();
            var service = scope.ServiceProvider.GetRequiredService<IMyService>();
            
            var isHealthy = await service.CheckHealthAsync();
            
            if (isHealthy)
            {
                return HealthCheckResult.Healthy("Service is running properly");
            }
            
            return HealthCheckResult.Unhealthy("Service is not responding");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy($"Service check failed: {ex.Message}");
        }
    }
}
```

## API Composition

여러 서비스의 데이터를 조합하여 클라이언트에 제공합니다.

```csharp
// ApiComposition/Services/OrderCompositionService.cs
public class OrderCompositionService
{
    private readonly IOrderService _orderService;
    private readonly ICustomerService _customerService;
    private readonly IProductService _productService;
    private readonly IShippingService _shippingService;
    
    public OrderCompositionService(
        IOrderService orderService,
        ICustomerService customerService,
        IProductService productService,
        IShippingService shippingService)
    {
        _orderService = orderService;
        _customerService = customerService;
        _productService = productService;
        _shippingService = shippingService;
    }
    
    public async Task<OrderDetailsDto> GetOrderDetailsAsync(Guid orderId)
    {
        // 병렬로 데이터 조회
        var orderTask = _orderService.GetOrderAsync(orderId);
        var customerTask = Task.CompletedTask;
        var productTasks = new List<Task<Product>>();
        var shippingTask = Task.CompletedTask;
        
        var order = await orderTask;
        
        if (order != null)
        {
            customerTask = _customerService.GetCustomerAsync(order.CustomerId);
            productTasks = order.Items.Select(item => 
                _productService.GetProductAsync(item.ProductId)).ToList();
            shippingTask = _shippingService.GetShippingInfoAsync(orderId);
        }
        
        await Task.WhenAll(
            customerTask,
            Task.WhenAll(productTasks),
            shippingTask);
            
        // 데이터 조합
        var orderDetails = new OrderDetailsDto
        {
            OrderId = order.Id,
            OrderDate = order.OrderDate,
            Status = order.Status,
            Customer = await customerTask as Customer,
            Items = order.Items.Select((item, index) => new OrderItemDetailsDto
            {
                ProductId = item.ProductId,
                ProductName = productTasks[index].Result?.Name,
                Quantity = item.Quantity,
                UnitPrice = item.UnitPrice,
                TotalPrice = item.TotalPrice
            }).ToList(),
            ShippingInfo = await shippingTask as ShippingInfo,
            TotalAmount = order.TotalAmount
        };
        
        return orderDetails;
    }
}
```

## 보안 패턴

### Service-to-Service 인증

```csharp
// Shared/Security/ServiceAuthenticationHandler.cs
public class ServiceAuthenticationHandler : DelegatingHandler
{
    private readonly ITokenService _tokenService;
    
    public ServiceAuthenticationHandler(ITokenService tokenService)
    {
        _tokenService = tokenService;
    }
    
    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request,
        CancellationToken cancellationToken)
    {
        // 서비스 간 통신을 위한 토큰 획득
        var token = await _tokenService.GetServiceTokenAsync();
        request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", token);
        
        return await base.SendAsync(request, cancellationToken);
    }
}

// Shared/Security/TokenService.cs
public class TokenService : ITokenService
{
    private readonly HttpClient _httpClient;
    private readonly IMemoryCache _cache;
    private readonly IConfiguration _configuration;
    
    public async Task<string> GetServiceTokenAsync()
    {
        const string cacheKey = "service_token";
        
        if (_cache.TryGetValue<string>(cacheKey, out var cachedToken))
            return cachedToken;
            
        var tokenRequest = new
        {
            client_id = _configuration["Auth:ClientId"],
            client_secret = _configuration["Auth:ClientSecret"],
            grant_type = "client_credentials",
            scope = "microservices.api"
        };
        
        var response = await _httpClient.PostAsJsonAsync("/connect/token", tokenRequest);
        var tokenResponse = await response.Content.ReadFromJsonAsync<TokenResponse>();
        
        _cache.Set(cacheKey, tokenResponse.AccessToken, 
            TimeSpan.FromSeconds(tokenResponse.ExpiresIn - 60));
            
        return tokenResponse.AccessToken;
    }
}
```

## 마무리

마이크로서비스 패턴들은 분산 시스템의 복잡성을 관리하고 안정성을 보장하는 데 필수적입니다. 각 패턴은 특정 문제를 해결하기 위해 설계되었으며, 적절히 조합하여 사용하면 확장 가능하고 유지보수가 쉬운 시스템을 구축할 수 있습니다.