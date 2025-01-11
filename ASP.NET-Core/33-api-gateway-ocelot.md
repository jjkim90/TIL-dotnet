# API Gateway - Ocelot

## API Gateway 소개

API Gateway는 마이크로서비스 아키텍처에서 클라이언트와 백엔드 서비스 간의 단일 진입점 역할을 합니다. 모든 클라이언트 요청을 받아 적절한 마이크로서비스로 라우팅하며, 인증, 로드 밸런싱, 캐싱, 요청 제한 등의 공통 기능을 제공합니다.

### API Gateway의 주요 기능

1. **요청 라우팅**: 클라이언트 요청을 적절한 서비스로 전달
2. **인증 및 권한 부여**: 중앙화된 보안 처리
3. **요청/응답 변환**: 프로토콜 변환 및 데이터 변환
4. **부하 분산**: 여러 서비스 인스턴스 간 트래픽 분배
5. **서비스 집계**: 여러 서비스의 응답을 결합
6. **캐싱**: 응답 캐싱으로 성능 향상
7. **요청 제한**: API 사용량 제어

## Ocelot 소개

Ocelot은 .NET Core 기반의 오픈소스 API Gateway입니다. 가볍고 빠르며, ASP.NET Core와 완벽하게 통합되어 마이크로서비스 아키텍처에 적합합니다.

### Ocelot의 특징

- **간단한 설정**: JSON 기반 구성
- **미들웨어 통합**: ASP.NET Core 미들웨어 파이프라인 활용
- **다양한 기능**: 라우팅, 인증, 캐싱, QoS 등
- **확장 가능**: 커스텀 미들웨어 추가 가능
- **Service Discovery 지원**: Consul, Eureka 등과 통합

## Ocelot 설치 및 기본 설정

### 프로젝트 생성 및 패키지 설치

```bash
# API Gateway 프로젝트 생성
dotnet new webapi -n ApiGateway
cd ApiGateway

# Ocelot 패키지 설치
dotnet add package Ocelot
dotnet add package Ocelot.Cache.CacheManager
```

### Program.cs 설정

```csharp
using Ocelot.DependencyInjection;
using Ocelot.Middleware;

var builder = WebApplication.CreateBuilder(args);

// Ocelot 설정 파일 추가
builder.Configuration.AddJsonFile("ocelot.json", optional: false, reloadOnChange: true);

// Ocelot 서비스 등록
builder.Services.AddOcelot(builder.Configuration);

// CORS 설정
builder.Services.AddCors(options =>
{
    options.AddPolicy("CorsPolicy", builder =>
    {
        builder.AllowAnyOrigin()
               .AllowAnyMethod()
               .AllowAnyHeader();
    });
});

var app = builder.Build();

// 미들웨어 파이프라인
app.UseCors("CorsPolicy");

// Ocelot 미들웨어
await app.UseOcelot();

app.Run();
```

## 라우팅 설정

### 기본 라우팅

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
          "Port": 5001
        }
      ],
      "UpstreamPathTemplate": "/api/products/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE" ]
    },
    {
      "DownstreamPathTemplate": "/api/customers/{id}",
      "DownstreamScheme": "https",
      "DownstreamHostAndPorts": [
        {
          "Host": "localhost",
          "Port": 5002
        }
      ],
      "UpstreamPathTemplate": "/api/customers/{id}",
      "UpstreamHttpMethod": [ "GET" ]
    }
  ],
  "GlobalConfiguration": {
    "BaseUrl": "https://localhost:5000"
  }
}
```

### 경로 변환

```json
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/v1/orders/{orderId}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "order-service",
          "Port": 80
        }
      ],
      "UpstreamPathTemplate": "/orders/{orderId}",
      "UpstreamHttpMethod": [ "GET" ],
      "RouteClaimsRequirement": {
        "Role": "Customer"
      }
    },
    {
      "DownstreamPathTemplate": "/internal/inventory/check",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "inventory-service",
          "Port": 80
        }
      ],
      "UpstreamPathTemplate": "/api/inventory/availability",
      "UpstreamHttpMethod": [ "POST" ],
      "AddHeadersToRequest": {
        "X-Internal-Request": "true"
      }
    }
  ]
}
```

## 부하 분산

### Round Robin 부하 분산

```json
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/products/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "product-service-1",
          "Port": 80
        },
        {
          "Host": "product-service-2",
          "Port": 80
        },
        {
          "Host": "product-service-3",
          "Port": 80
        }
      ],
      "UpstreamPathTemplate": "/api/products/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE" ],
      "LoadBalancerOptions": {
        "Type": "RoundRobin"
      }
    }
  ]
}
```

### 부하 분산 옵션

```json
{
  "LoadBalancerOptions": {
    "Type": "LeastConnection", // 또는 "RoundRobin", "NoLoadBalancer", "CookieStickySessions"
    "Key": "SessionId",
    "Expiry": 3600000
  }
}
```

### 커스텀 부하 분산

```csharp
// CustomLoadBalancer.cs
public class CustomLoadBalancer : ILoadBalancer
{
    private readonly List<Service> _services;
    private int _counter;
    
    public CustomLoadBalancer(List<Service> services)
    {
        _services = services;
        _counter = 0;
    }
    
    public async Task<Response<ServiceHostAndPort>> Lease(HttpContext httpContext)
    {
        if (_services.Count == 0)
        {
            return new ErrorResponse<ServiceHostAndPort>(
                new ErrorInvalidStateReason("No services available"));
        }
        
        // 커스텀 로직: 헤더 기반 라우팅
        var preferredService = httpContext.Request.Headers["X-Preferred-Service"].FirstOrDefault();
        
        if (!string.IsNullOrEmpty(preferredService))
        {
            var service = _services.FirstOrDefault(s => s.HostAndPort.Host == preferredService);
            if (service != null)
            {
                return new OkResponse<ServiceHostAndPort>(service.HostAndPort);
            }
        }
        
        // 기본 Round Robin
        var serviceIndex = _counter++ % _services.Count;
        var selectedService = _services[serviceIndex];
        
        return new OkResponse<ServiceHostAndPort>(selectedService.HostAndPort);
    }
    
    public void Release(ServiceHostAndPort hostAndPort)
    {
        // 필요시 리소스 해제
    }
}
```

## 인증 및 권한 부여

### JWT 인증 설정

```csharp
// Program.cs
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer("Bearer", options =>
    {
        options.Authority = "https://localhost:5001"; // Identity Server URL
        options.RequireHttpsMetadata = false;
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateAudience = false,
            ValidateIssuer = true,
            ValidateLifetime = true,
            ClockSkew = TimeSpan.Zero
        };
    });

builder.Services.AddOcelot(builder.Configuration);
```

### 라우트별 인증 설정

```json
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/orders/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "order-service",
          "Port": 80
        }
      ],
      "UpstreamPathTemplate": "/api/orders/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE" ],
      "AuthenticationOptions": {
        "AuthenticationProviderKey": "Bearer",
        "AllowedScopes": [ "orders.read", "orders.write" ]
      }
    },
    {
      "DownstreamPathTemplate": "/api/public/products",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "product-service",
          "Port": 80
        }
      ],
      "UpstreamPathTemplate": "/api/products/public",
      "UpstreamHttpMethod": [ "GET" ]
      // 인증 옵션 없음 - 공개 엔드포인트
    }
  ]
}
```

### Claims 기반 권한 부여

```json
{
  "RouteClaimsRequirement": {
    "UserType": "Premium",
    "Department": "Sales"
  }
}
```

## 요청 제한 (Rate Limiting)

### 기본 Rate Limiting 설정

```json
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/products/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "product-service",
          "Port": 80
        }
      ],
      "UpstreamPathTemplate": "/api/products/{everything}",
      "UpstreamHttpMethod": [ "GET" ],
      "RateLimitOptions": {
        "ClientWhitelist": [ "admin-client" ],
        "EnableRateLimiting": true,
        "Period": "1m",
        "PeriodTimespan": 60,
        "Limit": 100
      }
    }
  ],
  "GlobalConfiguration": {
    "RateLimitOptions": {
      "DisableRateLimitHeaders": false,
      "QuotaExceededMessage": "요청 한도를 초과했습니다.",
      "HttpStatusCode": 429,
      "ClientIdHeader": "X-ClientId"
    }
  }
}
```

### 클라이언트별 Rate Limiting

```csharp
// ClientRateLimitMiddleware.cs
public class ClientRateLimitMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IMemoryCache _cache;
    
    public ClientRateLimitMiddleware(RequestDelegate next, IMemoryCache cache)
    {
        _next = next;
        _cache = cache;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        var clientId = context.Request.Headers["X-ClientId"].FirstOrDefault() ?? "anonymous";
        var key = $"rate_limit_{clientId}_{DateTime.UtcNow:yyyyMMddHHmm}";
        
        var requestCount = await _cache.GetOrCreateAsync(key, async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(1);
            return 0;
        });
        
        if (requestCount >= 100) // 분당 100개 요청 제한
        {
            context.Response.StatusCode = 429;
            await context.Response.WriteAsync("Rate limit exceeded");
            return;
        }
        
        await _cache.SetAsync(key, requestCount + 1);
        await _next(context);
    }
}
```

## 캐싱

### 응답 캐싱 설정

```csharp
// Program.cs
builder.Services.AddCacheManager(x =>
{
    x.WithDictionaryHandle("OcelotCache");
});

builder.Services.AddOcelot(builder.Configuration)
    .AddCacheManager();
```

### 라우트별 캐싱 설정

```json
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/products/catalog",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "product-service",
          "Port": 80
        }
      ],
      "UpstreamPathTemplate": "/api/catalog",
      "UpstreamHttpMethod": [ "GET" ],
      "FileCacheOptions": {
        "TtlSeconds": 300,
        "Region": "products"
      }
    }
  ]
}
```

### Redis 캐싱

```csharp
// Program.cs
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
    options.InstanceName = "OcelotCache";
});

builder.Services.AddSingleton<IOcelotCache<CachedResponse>, RedisOcelotCache>();
```

## 요청/응답 변환

### 헤더 변환

```json
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/orders",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "order-service",
          "Port": 80
        }
      ],
      "UpstreamPathTemplate": "/orders",
      "UpstreamHttpMethod": [ "POST" ],
      "AddHeadersToRequest": {
        "X-Forwarded-Host": "{RemoteIpAddress}",
        "X-Service-Name": "API-Gateway",
        "X-Request-ID": "{TraceIdentifier}"
      },
      "UpstreamHeaderTransform": {
        "X-Custom-Header": "Delete"
      },
      "DownstreamHeaderTransform": {
        "X-Internal-Header": "Delete",
        "X-Response-Time": "{DownstreamElapsedTime}"
      }
    }
  ]
}
```

### 요청/응답 본문 변환

```csharp
// Middleware/RequestTransformMiddleware.cs
public class RequestTransformMiddleware
{
    private readonly RequestDelegate _next;
    
    public RequestTransformMiddleware(RequestDelegate next)
    {
        _next = next;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        // 요청 본문 읽기
        context.Request.EnableBuffering();
        using var reader = new StreamReader(context.Request.Body, Encoding.UTF8, leaveOpen: true);
        var requestBody = await reader.ReadToEndAsync();
        context.Request.Body.Position = 0;
        
        // 요청 변환
        if (context.Request.Path.StartsWithSegments("/api/legacy"))
        {
            var legacyRequest = JsonSerializer.Deserialize<LegacyRequest>(requestBody);
            var modernRequest = TransformToModernFormat(legacyRequest);
            
            var transformedBody = JsonSerializer.Serialize(modernRequest);
            var bytes = Encoding.UTF8.GetBytes(transformedBody);
            context.Request.Body = new MemoryStream(bytes);
            context.Request.ContentLength = bytes.Length;
        }
        
        await _next(context);
    }
    
    private ModernRequest TransformToModernFormat(LegacyRequest legacy)
    {
        return new ModernRequest
        {
            Id = legacy.ID,
            Name = legacy.CustomerName,
            Items = legacy.OrderItems.Select(i => new ModernItem
            {
                ProductId = i.ItemCode,
                Quantity = i.Qty,
                Price = i.UnitPrice
            }).ToList()
        };
    }
}
```

## Quality of Service

### Circuit Breaker와 Retry 설정

```json
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/payments/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "payment-service",
          "Port": 80
        }
      ],
      "UpstreamPathTemplate": "/api/payments/{everything}",
      "UpstreamHttpMethod": [ "POST" ],
      "QoSOptions": {
        "ExceptionsAllowedBeforeBreaking": 3,
        "DurationOfBreak": 10000,
        "TimeoutValue": 5000
      },
      "HttpHandlerOptions": {
        "AllowAutoRedirect": false,
        "UseCookieContainer": false,
        "UseTracing": true
      }
    }
  ]
}
```

### Polly 통합

```csharp
// Program.cs
builder.Services.AddOcelot(builder.Configuration)
    .AddPolly();

// 커스텀 Polly 정책
builder.Services.AddSingleton<IAsyncPolicy<HttpResponseMessage>>(
    Policy.HandleResult<HttpResponseMessage>(r => !r.IsSuccessStatusCode)
        .WaitAndRetryAsync(
            3,
            retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)),
            onRetry: (outcome, timespan, retryCount, context) =>
            {
                var logger = context.Values["Logger"] as ILogger;
                logger?.LogWarning($"Retry {retryCount} after {timespan} seconds");
            }));
```

## Service Discovery 통합

### Consul 통합

```csharp
// Program.cs
builder.Services.AddOcelot(builder.Configuration)
    .AddConsul()
    .AddConfigStoredInConsul();

builder.Services.AddSingleton<IConsulClient>(p => new ConsulClient(config =>
{
    config.Address = new Uri("http://localhost:8500");
}));
```

### Consul을 사용한 동적 라우팅

```json
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/{everything}",
      "DownstreamScheme": "http",
      "UpstreamPathTemplate": "/api/products/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE" ],
      "ServiceName": "product-service",
      "LoadBalancerOptions": {
        "Type": "LeastConnection"
      },
      "UseServiceDiscovery": true
    }
  ],
  "GlobalConfiguration": {
    "ServiceDiscoveryProvider": {
      "Scheme": "http",
      "Host": "localhost",
      "Port": 8500,
      "Type": "Consul"
    }
  }
}
```

## 모니터링과 로깅

### 구조화된 로깅

```csharp
// Program.cs
builder.Host.UseSerilog((context, services, configuration) => configuration
    .ReadFrom.Configuration(context.Configuration)
    .ReadFrom.Services(services)
    .Enrich.FromLogContext()
    .Enrich.WithProperty("Service", "API-Gateway")
    .WriteTo.Console(new RenderedCompactJsonFormatter())
    .WriteTo.Elasticsearch(new ElasticsearchSinkOptions(new Uri("http://localhost:9200"))
    {
        IndexFormat = "ocelot-logs-{0:yyyy.MM.dd}",
        AutoRegisterTemplate = true
    }));
```

### 요청 추적

```csharp
// Middleware/RequestTracingMiddleware.cs
public class RequestTracingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestTracingMiddleware> _logger;
    
    public RequestTracingMiddleware(
        RequestDelegate next, 
        ILogger<RequestTracingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        var correlationId = context.Request.Headers["X-Correlation-Id"].FirstOrDefault() 
            ?? Guid.NewGuid().ToString();
            
        context.Items["CorrelationId"] = correlationId;
        context.Response.Headers.Add("X-Correlation-Id", correlationId);
        
        var stopwatch = Stopwatch.StartNew();
        
        _logger.LogInformation("Request started: {Method} {Path} [{CorrelationId}]",
            context.Request.Method,
            context.Request.Path,
            correlationId);
            
        try
        {
            await _next(context);
        }
        finally
        {
            stopwatch.Stop();
            
            _logger.LogInformation(
                "Request completed: {Method} {Path} [{CorrelationId}] - Status: {StatusCode} - Duration: {Duration}ms",
                context.Request.Method,
                context.Request.Path,
                correlationId,
                context.Response.StatusCode,
                stopwatch.ElapsedMilliseconds);
        }
    }
}
```

## 보안 강화

### API Key 인증

```csharp
// Middleware/ApiKeyAuthenticationMiddleware.cs
public class ApiKeyAuthenticationMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IConfiguration _configuration;
    
    public ApiKeyAuthenticationMiddleware(
        RequestDelegate next,
        IConfiguration configuration)
    {
        _next = next;
        _configuration = configuration;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        if (!context.Request.Headers.TryGetValue("X-Api-Key", out var apiKey))
        {
            context.Response.StatusCode = 401;
            await context.Response.WriteAsync("API Key is missing");
            return;
        }
        
        var validApiKeys = _configuration.GetSection("ApiKeys").Get<List<string>>();
        
        if (!validApiKeys.Contains(apiKey))
        {
            context.Response.StatusCode = 401;
            await context.Response.WriteAsync("Invalid API Key");
            return;
        }
        
        await _next(context);
    }
}
```

### IP 화이트리스트

```json
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/admin/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        {
          "Host": "admin-service",
          "Port": 80
        }
      ],
      "UpstreamPathTemplate": "/api/admin/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE" ],
      "SecurityOptions": {
        "IPAllowedList": [ "192.168.1.0/24", "10.0.0.0/8" ],
        "IPBlockedList": [ "192.168.1.100" ]
      }
    }
  ]
}
```

## 고급 설정

### 동적 라우팅

```csharp
// DynamicRouting/IDynamicRouteProvider.cs
public interface IDynamicRouteProvider
{
    Task<List<FileRoute>> GetRoutesAsync();
}

// DynamicRouting/DatabaseRouteProvider.cs
public class DatabaseRouteProvider : IDynamicRouteProvider
{
    private readonly IDbConnection _connection;
    
    public DatabaseRouteProvider(IDbConnection connection)
    {
        _connection = connection;
    }
    
    public async Task<List<FileRoute>> GetRoutesAsync()
    {
        var routes = await _connection.QueryAsync<RouteConfig>(@"
            SELECT * FROM ApiRoutes WHERE IsActive = 1");
            
        return routes.Select(r => new FileRoute
        {
            DownstreamPathTemplate = r.DownstreamPath,
            DownstreamScheme = r.Scheme,
            DownstreamHostAndPorts = new List<FileHostAndPort>
            {
                new FileHostAndPort { Host = r.Host, Port = r.Port }
            },
            UpstreamPathTemplate = r.UpstreamPath,
            UpstreamHttpMethod = r.Methods.Split(',').ToList()
        }).ToList();
    }
}
```

### WebSocket 지원

```json
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/ws/notifications",
      "DownstreamScheme": "ws",
      "DownstreamHostAndPorts": [
        {
          "Host": "notification-service",
          "Port": 80
        }
      ],
      "UpstreamPathTemplate": "/ws/notifications",
      "UpstreamHttpMethod": [ "GET" ]
    }
  ]
}
```

## 마무리

Ocelot은 .NET 기반 마이크로서비스 아키텍처에서 강력하고 유연한 API Gateway 솔루션을 제공합니다. 라우팅, 인증, 부하 분산, 캐싱 등의 핵심 기능을 간단한 설정으로 구현할 수 있으며, 확장성과 커스터마이징이 용이합니다.