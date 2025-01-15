# Distributed Tracing

## 분산 추적 소개

분산 추적(Distributed Tracing)은 마이크로서비스 아키텍처에서 요청이 여러 서비스를 거치며 처리되는 과정을 추적하고 모니터링하는 기술입니다. 단일 요청이 여러 서비스, 데이터베이스, 메시지 큐를 거치는 복잡한 분산 시스템에서 성능 문제와 오류를 진단하는 데 필수적입니다.

### 분산 추적의 필요성

1. **요청 흐름 가시성**: 요청이 시스템을 통과하는 전체 경로 파악
2. **성능 병목 지점 식별**: 느린 서비스나 작업 발견
3. **오류 원인 분석**: 분산 시스템에서 오류 발생 위치 추적
4. **의존성 매핑**: 서비스 간 의존 관계 이해
5. **SLA 모니터링**: 서비스 수준 계약 준수 여부 확인

### 핵심 개념

- **Trace**: 하나의 요청이 시스템을 통과하는 전체 경로
- **Span**: 하나의 작업 단위 (예: HTTP 요청, DB 쿼리)
- **Context Propagation**: 서비스 간 추적 정보 전달
- **Sampling**: 성능 영향을 최소화하기 위한 추적 데이터 샘플링

## OpenTelemetry 소개

OpenTelemetry는 분산 추적, 메트릭, 로그를 수집하고 처리하는 표준화된 오픈소스 프레임워크입니다. 벤더 중립적이며 다양한 백엔드 시스템과 통합됩니다.

### OpenTelemetry 구성 요소

1. **API**: 계측을 위한 인터페이스
2. **SDK**: API의 구현체
3. **Instrumentation Libraries**: 자동 계측 라이브러리
4. **Collector**: 텔레메트리 데이터 수집 및 처리
5. **Exporters**: 백엔드 시스템으로 데이터 전송

## ASP.NET Core에서 OpenTelemetry 설정

### NuGet 패키지 설치

```bash
# OpenTelemetry 기본 패키지
dotnet add package OpenTelemetry
dotnet add package OpenTelemetry.Extensions.Hosting
dotnet add package OpenTelemetry.Instrumentation.AspNetCore
dotnet add package OpenTelemetry.Instrumentation.Http

# Exporter 패키지 (Jaeger 예시)
dotnet add package OpenTelemetry.Exporter.Jaeger

# 추가 계측 패키지
dotnet add package OpenTelemetry.Instrumentation.SqlClient
dotnet add package OpenTelemetry.Instrumentation.StackExchangeRedis
```

### 기본 설정

```csharp
// Program.cs
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;

var builder = WebApplication.CreateBuilder(args);

// OpenTelemetry 설정
builder.Services.AddOpenTelemetry()
    .WithTracing(tracerProviderBuilder =>
    {
        tracerProviderBuilder
            .AddSource("MyCompany.MyService")
            .SetResourceBuilder(
                ResourceBuilder.CreateDefault()
                    .AddService(serviceName: "MyService", serviceVersion: "1.0.0"))
            .AddAspNetCoreInstrumentation(options =>
            {
                options.RecordException = true;
                options.Filter = (httpContext) =>
                {
                    // 헬스체크 엔드포인트 제외
                    return !httpContext.Request.Path.StartsWithSegments("/health");
                };
            })
            .AddHttpClientInstrumentation(options =>
            {
                options.RecordException = true;
                options.SetHttpFlavor = true;
            })
            .AddSqlClientInstrumentation(options =>
            {
                options.SetDbStatementForText = true;
                options.RecordException = true;
            })
            .AddJaegerExporter(options =>
            {
                options.AgentHost = builder.Configuration["Jaeger:AgentHost"] ?? "localhost";
                options.AgentPort = builder.Configuration.GetValue<int>("Jaeger:AgentPort", 6831);
            });
    });

builder.Services.AddControllers();

var app = builder.Build();

app.MapControllers();
app.Run();
```

## 커스텀 추적 구현

### Activity를 사용한 수동 계측

```csharp
// Services/OrderService.cs
public class OrderService
{
    private static readonly ActivitySource ActivitySource = new("MyCompany.MyService");
    private readonly IOrderRepository _orderRepository;
    private readonly IInventoryService _inventoryService;
    private readonly IPaymentService _paymentService;
    
    public OrderService(
        IOrderRepository orderRepository,
        IInventoryService inventoryService,
        IPaymentService paymentService)
    {
        _orderRepository = orderRepository;
        _inventoryService = inventoryService;
        _paymentService = paymentService;
    }
    
    public async Task<OrderResult> ProcessOrderAsync(CreateOrderRequest request)
    {
        using var activity = ActivitySource.StartActivity("ProcessOrder", ActivityKind.Internal);
        
        try
        {
            // 속성 추가
            activity?.SetTag("order.customer_id", request.CustomerId);
            activity?.SetTag("order.items_count", request.Items.Count);
            
            // 1. 주문 생성
            Order order;
            using (var createActivity = ActivitySource.StartActivity("CreateOrder", ActivityKind.Internal))
            {
                order = await _orderRepository.CreateAsync(request);
                createActivity?.SetTag("order.id", order.Id);
            }
            
            // 2. 재고 확인
            using (var inventoryActivity = ActivitySource.StartActivity("CheckInventory", ActivityKind.Client))
            {
                var inventoryResult = await _inventoryService.CheckAndReserveAsync(order.Items);
                
                if (!inventoryResult.Success)
                {
                    inventoryActivity?.SetStatus(ActivityStatusCode.Error, "Insufficient inventory");
                    throw new InsufficientInventoryException(inventoryResult.Message);
                }
            }
            
            // 3. 결제 처리
            using (var paymentActivity = ActivitySource.StartActivity("ProcessPayment", ActivityKind.Client))
            {
                var paymentResult = await _paymentService.ProcessAsync(order.TotalAmount, request.PaymentMethod);
                
                if (!paymentResult.Success)
                {
                    paymentActivity?.SetStatus(ActivityStatusCode.Error, "Payment failed");
                    // 재고 롤백
                    await _inventoryService.ReleaseReservationAsync(order.Id);
                    throw new PaymentFailedException(paymentResult.Message);
                }
                
                paymentActivity?.SetTag("payment.transaction_id", paymentResult.TransactionId);
            }
            
            // 4. 주문 확정
            using (var confirmActivity = ActivitySource.StartActivity("ConfirmOrder", ActivityKind.Internal))
            {
                await _orderRepository.ConfirmAsync(order.Id);
            }
            
            activity?.SetStatus(ActivityStatusCode.Ok);
            
            return new OrderResult { Success = true, OrderId = order.Id };
        }
        catch (Exception ex)
        {
            activity?.RecordException(ex);
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            throw;
        }
    }
}
```

### 분산 컨텍스트 전파

```csharp
// HttpClient를 통한 컨텍스트 전파
public class TracedHttpClient
{
    private readonly HttpClient _httpClient;
    private static readonly ActivitySource ActivitySource = new("MyCompany.HttpClient");
    
    public TracedHttpClient(HttpClient httpClient)
    {
        _httpClient = httpClient;
    }
    
    public async Task<T> GetAsync<T>(string url)
    {
        using var activity = ActivitySource.StartActivity("HTTP GET", ActivityKind.Client);
        activity?.SetTag("http.method", "GET");
        activity?.SetTag("http.url", url);
        
        try
        {
            var response = await _httpClient.GetAsync(url);
            
            activity?.SetTag("http.status_code", (int)response.StatusCode);
            
            if (!response.IsSuccessStatusCode)
            {
                activity?.SetStatus(ActivityStatusCode.Error, $"HTTP {response.StatusCode}");
                throw new HttpRequestException($"Request failed with status {response.StatusCode}");
            }
            
            var content = await response.Content.ReadAsStringAsync();
            return JsonSerializer.Deserialize<T>(content);
        }
        catch (Exception ex)
        {
            activity?.RecordException(ex);
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            throw;
        }
    }
}
```

## 추적 데이터 강화

### 커스텀 속성과 이벤트

```csharp
// Middleware/TracingEnrichmentMiddleware.cs
public class TracingEnrichmentMiddleware
{
    private readonly RequestDelegate _next;
    
    public TracingEnrichmentMiddleware(RequestDelegate next)
    {
        _next = next;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        var activity = Activity.Current;
        
        if (activity != null)
        {
            // 요청 정보 추가
            activity.SetTag("http.request_content_length", context.Request.ContentLength ?? 0);
            activity.SetTag("http.scheme", context.Request.Scheme);
            activity.SetTag("http.host", context.Request.Host.ToString());
            activity.SetTag("http.user_agent", context.Request.Headers["User-Agent"].ToString());
            
            // 사용자 정보 추가
            if (context.User.Identity?.IsAuthenticated == true)
            {
                activity.SetTag("user.id", context.User.FindFirst("sub")?.Value);
                activity.SetTag("user.name", context.User.Identity.Name);
            }
            
            // 커스텀 헤더 추가
            if (context.Request.Headers.TryGetValue("X-Request-ID", out var requestId))
            {
                activity.SetTag("request.id", requestId.ToString());
            }
        }
        
        await _next(context);
        
        // 응답 정보 추가
        if (activity != null)
        {
            activity.SetTag("http.response_content_length", context.Response.ContentLength ?? 0);
            
            // 오류 상태 처리
            if (context.Response.StatusCode >= 400)
            {
                activity.SetStatus(ActivityStatusCode.Error, $"HTTP {context.Response.StatusCode}");
            }
        }
    }
}
```

### Baggage를 통한 컨텍스트 전달

```csharp
// Services/BaggageService.cs
public class BaggageService
{
    public void SetUserContext(string userId, string tenantId)
    {
        Baggage.SetBaggage("user.id", userId);
        Baggage.SetBaggage("tenant.id", tenantId);
    }
    
    public string GetUserId()
    {
        return Baggage.GetBaggage("user.id") ?? string.Empty;
    }
    
    public string GetTenantId()
    {
        return Baggage.GetBaggage("tenant.id") ?? string.Empty;
    }
}

// 사용 예시
public class MultiTenantMiddleware
{
    private readonly RequestDelegate _next;
    private readonly BaggageService _baggageService;
    
    public MultiTenantMiddleware(RequestDelegate next, BaggageService baggageService)
    {
        _next = next;
        _baggageService = baggageService;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        var tenantId = context.Request.Headers["X-Tenant-ID"].FirstOrDefault();
        var userId = context.User.FindFirst("sub")?.Value;
        
        if (!string.IsNullOrEmpty(tenantId) && !string.IsNullOrEmpty(userId))
        {
            _baggageService.SetUserContext(userId, tenantId);
        }
        
        await _next(context);
    }
}
```

## 샘플링 전략

### 적응형 샘플링

```csharp
// Sampling/AdaptiveSampler.cs
public class AdaptiveSampler : Sampler
{
    private readonly double _targetQps;
    private readonly TimeSpan _adaptationInterval;
    private double _probability = 0.1;
    private DateTime _lastAdaptation = DateTime.UtcNow;
    private long _requestCount = 0;
    
    public AdaptiveSampler(double targetQps = 10, TimeSpan? adaptationInterval = null)
    {
        _targetQps = targetQps;
        _adaptationInterval = adaptationInterval ?? TimeSpan.FromMinutes(1);
    }
    
    public override SamplingResult ShouldSample(in SamplingParameters samplingParameters)
    {
        Interlocked.Increment(ref _requestCount);
        
        // 적응 주기 확인
        var now = DateTime.UtcNow;
        if (now - _lastAdaptation > _adaptationInterval)
        {
            AdaptSamplingRate(now);
        }
        
        // Priority sampling - 특정 조건은 항상 샘플링
        if (HasHighPriority(samplingParameters))
        {
            return new SamplingResult(SamplingDecision.RecordAndSample);
        }
        
        // 확률적 샘플링
        var random = new Random().NextDouble();
        if (random < _probability)
        {
            return new SamplingResult(SamplingDecision.RecordAndSample);
        }
        
        return new SamplingResult(SamplingDecision.Drop);
    }
    
    private void AdaptSamplingRate(DateTime now)
    {
        var elapsed = now - _lastAdaptation;
        var currentQps = _requestCount / elapsed.TotalSeconds;
        
        // 목표 QPS에 맞춰 샘플링 비율 조정
        if (currentQps > 0)
        {
            _probability = Math.Min(1.0, _targetQps / currentQps);
        }
        
        _lastAdaptation = now;
        _requestCount = 0;
    }
    
    private bool HasHighPriority(in SamplingParameters parameters)
    {
        // 오류나 특정 경로는 항상 샘플링
        if (parameters.Tags != null)
        {
            if (parameters.Tags.Any(tag => tag.Key == "error" && tag.Value?.ToString() == "true"))
                return true;
                
            if (parameters.Tags.Any(tag => tag.Key == "http.route" && tag.Value?.ToString()?.Contains("/api/critical") == true))
                return true;
        }
        
        return false;
    }
}

// Program.cs에서 사용
.AddSource("MyService")
.SetSampler(new AdaptiveSampler(targetQps: 100))
```

## 분산 추적 백엔드 통합

### Jaeger 통합

```csharp
// Jaeger 설정
builder.Services.AddOpenTelemetry()
    .WithTracing(builder =>
    {
        builder
            .AddJaegerExporter(options =>
            {
                options.AgentHost = "localhost";
                options.AgentPort = 6831;
                options.MaxPayloadSizeInBytes = 4096;
                options.ExportProcessorType = ExportProcessorType.Batch;
                options.BatchExportProcessorOptions = new BatchExportProcessorOptions<Activity>
                {
                    MaxQueueSize = 2048,
                    ScheduledDelayMilliseconds = 5000,
                    ExporterTimeoutMilliseconds = 30000,
                    MaxExportBatchSize = 512
                };
            });
    });
```

### Zipkin 통합

```csharp
// Zipkin 설정
builder.Services.AddOpenTelemetry()
    .WithTracing(builder =>
    {
        builder
            .AddZipkinExporter(options =>
            {
                options.Endpoint = new Uri("http://localhost:9411/api/v2/spans");
                options.UseShortTraceIds = false;
            });
    });
```

### Azure Application Insights 통합

```csharp
// Application Insights 설정
builder.Services.AddOpenTelemetry()
    .WithTracing(builder =>
    {
        builder
            .AddAzureMonitorTraceExporter(options =>
            {
                options.ConnectionString = builder.Configuration["ApplicationInsights:ConnectionString"];
            });
    });
```

## 성능 모니터링

### 지연 시간 추적

```csharp
// Monitoring/LatencyTracker.cs
public class LatencyTracker
{
    private static readonly Histogram<double> RequestDuration = Metrics
        .CreateHistogram<double>("http_request_duration_ms", "HTTP request duration in milliseconds");
        
    public static IDisposable TrackLatency(string endpoint, string method)
    {
        var stopwatch = Stopwatch.StartNew();
        
        return new LatencyTrackerScope(() =>
        {
            stopwatch.Stop();
            RequestDuration.Record(
                stopwatch.ElapsedMilliseconds,
                new KeyValuePair<string, object?>("endpoint", endpoint),
                new KeyValuePair<string, object?>("method", method));
        });
    }
    
    private class LatencyTrackerScope : IDisposable
    {
        private readonly Action _onDispose;
        
        public LatencyTrackerScope(Action onDispose)
        {
            _onDispose = onDispose;
        }
        
        public void Dispose()
        {
            _onDispose?.Invoke();
        }
    }
}

// 사용 예시
public async Task<IActionResult> GetProducts()
{
    using (LatencyTracker.TrackLatency("/api/products", "GET"))
    {
        var products = await _productService.GetAllAsync();
        return Ok(products);
    }
}
```

### 분산 추적 대시보드

```csharp
// Controllers/TracingDashboardController.cs
[ApiController]
[Route("api/tracing")]
public class TracingDashboardController : ControllerBase
{
    private readonly ILogger<TracingDashboardController> _logger;
    
    [HttpGet("stats")]
    public IActionResult GetTracingStats()
    {
        var stats = new
        {
            ActiveSpans = Activity.Current != null ? 1 : 0,
            TraceId = Activity.Current?.TraceId.ToString(),
            SpanId = Activity.Current?.SpanId.ToString(),
            ServiceName = Activity.Current?.GetTagItem("service.name"),
            Baggage = Baggage.GetBaggage().ToDictionary(kvp => kvp.Key, kvp => kvp.Value)
        };
        
        return Ok(stats);
    }
    
    [HttpGet("dependencies")]
    public async Task<IActionResult> GetServiceDependencies()
    {
        // 실제로는 추적 백엔드에서 데이터를 가져와야 함
        var dependencies = new[]
        {
            new { Source = "api-gateway", Target = "order-service", CallCount = 1000 },
            new { Source = "order-service", Target = "inventory-service", CallCount = 800 },
            new { Source = "order-service", Target = "payment-service", CallCount = 750 }
        };
        
        return Ok(dependencies);
    }
}
```

## 로그와 추적 상관관계

### 구조화된 로깅과 추적 연결

```csharp
// Logging/TracingLogger.cs
public static class TracingLogger
{
    public static ILogger<T> CreateLogger<T>(ILoggerFactory loggerFactory)
    {
        return new TracingLoggerWrapper<T>(loggerFactory.CreateLogger<T>());
    }
}

public class TracingLoggerWrapper<T> : ILogger<T>
{
    private readonly ILogger<T> _innerLogger;
    
    public TracingLoggerWrapper(ILogger<T> innerLogger)
    {
        _innerLogger = innerLogger;
    }
    
    public void Log<TState>(
        LogLevel logLevel,
        EventId eventId,
        TState state,
        Exception? exception,
        Func<TState, Exception?, string> formatter)
    {
        var activity = Activity.Current;
        
        if (activity != null)
        {
            using (_innerLogger.BeginScope(new Dictionary<string, object>
            {
                ["TraceId"] = activity.TraceId.ToString(),
                ["SpanId"] = activity.SpanId.ToString(),
                ["ParentSpanId"] = activity.ParentSpanId.ToString()
            }))
            {
                _innerLogger.Log(logLevel, eventId, state, exception, formatter);
            }
        }
        else
        {
            _innerLogger.Log(logLevel, eventId, state, exception, formatter);
        }
    }
    
    public bool IsEnabled(LogLevel logLevel) => _innerLogger.IsEnabled(logLevel);
    
    public IDisposable BeginScope<TState>(TState state) => _innerLogger.BeginScope(state);
}
```

### Serilog 통합

```csharp
// Program.cs - Serilog 설정
Log.Logger = new LoggerConfiguration()
    .Enrich.FromLogContext()
    .Enrich.WithProperty("ServiceName", "MyService")
    .Enrich.WithSpan() // OpenTelemetry span 정보 추가
    .WriteTo.Console(new RenderedCompactJsonFormatter())
    .WriteTo.Seq("http://localhost:5341")
    .CreateLogger();

builder.Host.UseSerilog();

// Enricher 구현
public class SpanEnricher : ILogEventEnricher
{
    public void Enrich(LogEvent logEvent, ILogEventPropertyFactory propertyFactory)
    {
        var activity = Activity.Current;
        
        if (activity != null)
        {
            logEvent.AddPropertyIfAbsent(propertyFactory.CreateProperty("TraceId", activity.TraceId.ToString()));
            logEvent.AddPropertyIfAbsent(propertyFactory.CreateProperty("SpanId", activity.SpanId.ToString()));
            
            if (activity.ParentSpanId != default)
            {
                logEvent.AddPropertyIfAbsent(propertyFactory.CreateProperty("ParentSpanId", activity.ParentSpanId.ToString()));
            }
        }
    }
}
```

## 오류 추적과 진단

### 예외 추적

```csharp
// Middleware/ExceptionTracingMiddleware.cs
public class ExceptionTracingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionTracingMiddleware> _logger;
    
    public ExceptionTracingMiddleware(
        RequestDelegate next,
        ILogger<ExceptionTracingMiddleware> logger)
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
            var activity = Activity.Current;
            
            if (activity != null)
            {
                activity.RecordException(ex);
                activity.SetStatus(ActivityStatusCode.Error, ex.Message);
                
                // 추가 컨텍스트 정보
                activity.SetTag("error.type", ex.GetType().FullName);
                activity.SetTag("error.stacktrace", ex.StackTrace);
                
                if (ex.InnerException != null)
                {
                    activity.SetTag("error.inner_exception", ex.InnerException.Message);
                }
                
                // 요청 정보 추가
                activity.SetTag("error.request.path", context.Request.Path);
                activity.SetTag("error.request.method", context.Request.Method);
                
                if (context.Request.ContentLength > 0)
                {
                    context.Request.EnableBuffering();
                    using var reader = new StreamReader(context.Request.Body, leaveOpen: true);
                    var body = await reader.ReadToEndAsync();
                    context.Request.Body.Position = 0;
                    
                    activity.SetTag("error.request.body", body);
                }
            }
            
            _logger.LogError(ex, "Unhandled exception occurred");
            
            throw;
        }
    }
}
```

### 분산 추적 문제 해결

```csharp
// Diagnostics/TracingDiagnostics.cs
public class TracingDiagnostics
{
    private readonly ILogger<TracingDiagnostics> _logger;
    
    public TracingDiagnostics(ILogger<TracingDiagnostics> logger)
    {
        _logger = logger;
    }
    
    public void ValidateTracingContext()
    {
        var activity = Activity.Current;
        
        if (activity == null)
        {
            _logger.LogWarning("No active trace context found");
            return;
        }
        
        _logger.LogInformation("Trace Context Validation:");
        _logger.LogInformation($"  TraceId: {activity.TraceId}");
        _logger.LogInformation($"  SpanId: {activity.SpanId}");
        _logger.LogInformation($"  ParentSpanId: {activity.ParentSpanId}");
        _logger.LogInformation($"  TraceState: {activity.TraceStateString}");
        _logger.LogInformation($"  IsRecorded: {activity.Recorded}");
        
        // Baggage 검증
        var baggage = Baggage.GetBaggage();
        if (baggage.Any())
        {
            _logger.LogInformation("Baggage items:");
            foreach (var item in baggage)
            {
                _logger.LogInformation($"  {item.Key}: {item.Value}");
            }
        }
        
        // Tags 검증
        if (activity.TagObjects.Any())
        {
            _logger.LogInformation("Activity tags:");
            foreach (var tag in activity.TagObjects)
            {
                _logger.LogInformation($"  {tag.Key}: {tag.Value}");
            }
        }
    }
}
```

## 마무리

분산 추적은 마이크로서비스 아키텍처에서 시스템의 가시성을 확보하고 문제를 진단하는 데 필수적인 도구입니다. OpenTelemetry를 통해 표준화된 방식으로 추적을 구현하고, 다양한 백엔드 시스템과 통합하여 효과적인 모니터링 환경을 구축할 수 있습니다.