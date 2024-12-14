# Health Checks와 모니터링

## Health Checks 개요

Health Checks는 애플리케이션과 의존성의 상태를 모니터링하는 기능입니다. ASP.NET Core는 애플리케이션의 건강 상태를 확인하고 보고하는 내장 프레임워크를 제공합니다.

### Health Checks의 주요 용도
- **가동 시간 모니터링**: 애플리케이션이 정상적으로 실행 중인지 확인
- **의존성 확인**: 데이터베이스, 외부 API, 메시지 큐 등의 상태 확인
- **로드 밸런서 통합**: 비정상 인스턴스 자동 제거
- **자동 복구**: 문제 감지 시 자동 재시작 또는 복구
- **알림**: 문제 발생 시 관리자에게 알림

## 기본 Health Checks 설정

### 기본 구성
```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Health Checks 서비스 추가
builder.Services.AddHealthChecks();

var app = builder.Build();

// Health Check 엔드포인트 매핑
app.MapHealthChecks("/health");

// 상세 정보를 포함한 Health Check
app.MapHealthChecks("/health/detailed", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse,
    ResultStatusCodes =
    {
        [HealthStatus.Healthy] = StatusCodes.Status200OK,
        [HealthStatus.Degraded] = StatusCodes.Status200OK,
        [HealthStatus.Unhealthy] = StatusCodes.Status503ServiceUnavailable
    },
    AllowCachingResponses = false
});

app.Run();
```

### 사용자 정의 Health Check
```csharp
public class DatabaseHealthCheck : IHealthCheck
{
    private readonly ApplicationDbContext _context;
    
    public DatabaseHealthCheck(ApplicationDbContext context)
    {
        _context = context;
    }
    
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            // 데이터베이스 연결 테스트
            await _context.Database.CanConnectAsync(cancellationToken);
            
            // 추가 검증: 테이블 존재 여부
            var tableExists = await _context.Database
                .ExecuteSqlRawAsync(
                    "SELECT 1 FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_NAME = 'Users'",
                    cancellationToken) > 0;
            
            if (!tableExists)
            {
                return HealthCheckResult.Unhealthy(
                    "Database is accessible but required tables are missing");
            }
            
            return HealthCheckResult.Healthy("Database is healthy");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy(
                "Database connection failed",
                exception: ex,
                data: new Dictionary<string, object>
                {
                    ["ConnectionString"] = _context.Database.GetConnectionString()?.Split(';')[0]
                });
        }
    }
}

// 등록
builder.Services.AddHealthChecks()
    .AddCheck<DatabaseHealthCheck>("database", tags: new[] { "db", "sql" });
```

## 다양한 Health Check 구현

### 외부 서비스 Health Check
```csharp
public class ExternalApiHealthCheck : IHealthCheck
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<ExternalApiHealthCheck> _logger;
    private readonly string _apiUrl;
    
    public ExternalApiHealthCheck(
        HttpClient httpClient,
        IConfiguration configuration,
        ILogger<ExternalApiHealthCheck> logger)
    {
        _httpClient = httpClient;
        _logger = logger;
        _apiUrl = configuration["ExternalApi:HealthCheckUrl"];
    }
    
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var stopwatch = Stopwatch.StartNew();
            
            var response = await _httpClient.GetAsync(_apiUrl, cancellationToken);
            
            stopwatch.Stop();
            
            var data = new Dictionary<string, object>
            {
                ["StatusCode"] = (int)response.StatusCode,
                ["ResponseTime"] = stopwatch.ElapsedMilliseconds,
                ["Endpoint"] = _apiUrl
            };
            
            if (response.IsSuccessStatusCode)
            {
                return HealthCheckResult.Healthy(
                    $"External API is healthy (Response time: {stopwatch.ElapsedMilliseconds}ms)",
                    data);
            }
            
            if (response.StatusCode == HttpStatusCode.TooManyRequests)
            {
                return HealthCheckResult.Degraded(
                    "External API rate limit reached",
                    data: data);
            }
            
            return HealthCheckResult.Unhealthy(
                $"External API returned {response.StatusCode}",
                data: data);
        }
        catch (TaskCanceledException)
        {
            return HealthCheckResult.Unhealthy("External API request timeout");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "External API health check failed");
            
            return HealthCheckResult.Unhealthy(
                "External API health check failed",
                exception: ex);
        }
    }
}
```

### 디스크 공간 Health Check
```csharp
public class DiskSpaceHealthCheck : IHealthCheck
{
    private readonly long _minimumFreeMb;
    private readonly string _driveName;
    
    public DiskSpaceHealthCheck(IConfiguration configuration)
    {
        _minimumFreeMb = configuration.GetValue<long>("HealthChecks:MinimumFreeDiskSpaceMb", 1024);
        _driveName = configuration["HealthChecks:DriveName"] ?? "C:\\";
    }
    
    public Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var driveInfo = new DriveInfo(_driveName);
            
            if (!driveInfo.IsReady)
            {
                return Task.FromResult(HealthCheckResult.Unhealthy($"Drive {_driveName} is not ready"));
            }
            
            var freeSpaceMb = driveInfo.AvailableFreeSpace / (1024 * 1024);
            var totalSpaceMb = driveInfo.TotalSize / (1024 * 1024);
            var usedPercentage = 100 - (freeSpaceMb * 100.0 / totalSpaceMb);
            
            var data = new Dictionary<string, object>
            {
                ["FreeSpaceMB"] = freeSpaceMb,
                ["TotalSpaceMB"] = totalSpaceMb,
                ["UsedPercentage"] = Math.Round(usedPercentage, 2),
                ["Drive"] = _driveName
            };
            
            if (freeSpaceMb < _minimumFreeMb)
            {
                return Task.FromResult(HealthCheckResult.Unhealthy(
                    $"Insufficient disk space. Free: {freeSpaceMb}MB, Required: {_minimumFreeMb}MB",
                    data: data));
            }
            
            if (usedPercentage > 90)
            {
                return Task.FromResult(HealthCheckResult.Degraded(
                    $"Disk space usage is high: {usedPercentage:F2}%",
                    data: data));
            }
            
            return Task.FromResult(HealthCheckResult.Healthy(
                $"Disk space is healthy. Free: {freeSpaceMb}MB ({100 - usedPercentage:F2}%)",
                data: data));
        }
        catch (Exception ex)
        {
            return Task.FromResult(HealthCheckResult.Unhealthy(
                "Failed to check disk space",
                exception: ex));
        }
    }
}
```

### 메모리 사용량 Health Check
```csharp
public class MemoryHealthCheck : IHealthCheck
{
    private readonly long _maxMemoryMb;
    
    public MemoryHealthCheck(IConfiguration configuration)
    {
        _maxMemoryMb = configuration.GetValue<long>("HealthChecks:MaxMemoryMb", 1024);
    }
    
    public Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        var allocatedMb = GC.GetTotalMemory(false) / (1024 * 1024);
        var gen0Collections = GC.CollectionCount(0);
        var gen1Collections = GC.CollectionCount(1);
        var gen2Collections = GC.CollectionCount(2);
        
        var data = new Dictionary<string, object>
        {
            ["AllocatedMB"] = allocatedMb,
            ["Gen0Collections"] = gen0Collections,
            ["Gen1Collections"] = gen1Collections,
            ["Gen2Collections"] = gen2Collections,
            ["WorkingSetMB"] = Process.GetCurrentProcess().WorkingSet64 / (1024 * 1024)
        };
        
        if (allocatedMb > _maxMemoryMb)
        {
            return Task.FromResult(HealthCheckResult.Unhealthy(
                $"Memory usage too high: {allocatedMb}MB (Max: {_maxMemoryMb}MB)",
                data: data));
        }
        
        if (allocatedMb > _maxMemoryMb * 0.8)
        {
            return Task.FromResult(HealthCheckResult.Degraded(
                $"Memory usage is high: {allocatedMb}MB",
                data: data));
        }
        
        return Task.FromResult(HealthCheckResult.Healthy(
            $"Memory usage is healthy: {allocatedMb}MB",
            data: data));
    }
}
```

## Health Check 고급 기능

### Health Check 그룹화 및 필터링
```csharp
// Program.cs
builder.Services.AddHealthChecks()
    .AddCheck("database", () => HealthCheckResult.Healthy(), tags: new[] { "db", "critical" })
    .AddCheck("redis", () => HealthCheckResult.Healthy(), tags: new[] { "cache" })
    .AddCheck("external-api", () => HealthCheckResult.Healthy(), tags: new[] { "external" });

// 태그별 엔드포인트
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("critical"),
    ResponseWriter = WriteResponse
});

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false, // 라이브니스 체크는 Health Check 없이 200 반환
});

app.MapHealthChecks("/health/external", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("external"),
    ResponseWriter = WriteResponse
});

// 사용자 정의 응답 작성
static Task WriteResponse(HttpContext context, HealthReport report)
{
    context.Response.ContentType = "application/json";
    
    var response = new
    {
        status = report.Status.ToString(),
        checks = report.Entries.Select(x => new
        {
            name = x.Key,
            status = x.Value.Status.ToString(),
            description = x.Value.Description,
            duration = x.Value.Duration.TotalMilliseconds,
            exception = x.Value.Exception?.Message,
            data = x.Value.Data
        }),
        totalDuration = report.TotalDuration.TotalMilliseconds
    };
    
    return context.Response.WriteAsync(JsonSerializer.Serialize(response));
}
```

### Health Check Publisher
```csharp
public class ApplicationInsightsHealthCheckPublisher : IHealthCheckPublisher
{
    private readonly TelemetryClient _telemetryClient;
    private readonly ILogger<ApplicationInsightsHealthCheckPublisher> _logger;
    
    public ApplicationInsightsHealthCheckPublisher(
        TelemetryClient telemetryClient,
        ILogger<ApplicationInsightsHealthCheckPublisher> logger)
    {
        _telemetryClient = telemetryClient;
        _logger = logger;
    }
    
    public Task PublishAsync(HealthReport report, CancellationToken cancellationToken)
    {
        foreach (var entry in report.Entries)
        {
            var telemetry = new EventTelemetry("HealthCheck");
            telemetry.Properties["Name"] = entry.Key;
            telemetry.Properties["Status"] = entry.Value.Status.ToString();
            telemetry.Properties["Duration"] = entry.Value.Duration.TotalMilliseconds.ToString();
            
            if (entry.Value.Description != null)
            {
                telemetry.Properties["Description"] = entry.Value.Description;
            }
            
            if (entry.Value.Exception != null)
            {
                telemetry.Properties["Exception"] = entry.Value.Exception.ToString();
            }
            
            foreach (var data in entry.Value.Data)
            {
                telemetry.Properties[$"Data_{data.Key}"] = data.Value?.ToString();
            }
            
            _telemetryClient.TrackEvent(telemetry);
            
            // 비정상 상태 로깅
            if (entry.Value.Status != HealthStatus.Healthy)
            {
                _logger.LogWarning(
                    "Health check {HealthCheckName} is {Status}: {Description}",
                    entry.Key,
                    entry.Value.Status,
                    entry.Value.Description);
            }
        }
        
        return Task.CompletedTask;
    }
}

// 등록
builder.Services.Configure<HealthCheckPublisherOptions>(options =>
{
    options.Delay = TimeSpan.FromSeconds(30);
    options.Period = TimeSpan.FromSeconds(60);
    options.Timeout = TimeSpan.FromSeconds(30);
});

builder.Services.AddSingleton<IHealthCheckPublisher, ApplicationInsightsHealthCheckPublisher>();
```

## Health Check UI

### Health Check UI 설정
```csharp
// Program.cs
builder.Services.AddHealthChecks()
    .AddSqlServer(
        connectionString: builder.Configuration.GetConnectionString("DefaultConnection"),
        name: "SQL Server",
        tags: new[] { "db", "sql" })
    .AddRedis(
        builder.Configuration.GetConnectionString("Redis"),
        name: "Redis Cache",
        tags: new[] { "cache" })
    .AddUrlGroup(
        new Uri("https://api.example.com/health"),
        name: "External API",
        tags: new[] { "external" });

// Health Check UI 추가
builder.Services.AddHealthChecksUI(options =>
{
    options.SetEvaluationTimeInSeconds(30); // 30초마다 체크
    options.MaximumHistoryEntriesPerEndpoint(100);
    options.AddHealthCheckEndpoint("Local API", "/health");
    options.AddHealthCheckEndpoint("External Service", "https://external-api.com/health");
})
.AddInMemoryStorage(); // 또는 .AddSqlServerStorage()

var app = builder.Build();

app.MapHealthChecks("/health", new HealthCheckOptions
{
    Predicate = _ => true,
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.MapHealthChecksUI(options =>
{
    options.UIPath = "/health-ui";
    options.ApiPath = "/health-api";
});
```

### Health Check UI 사용자 정의
```csharp
// appsettings.json
{
  "HealthChecksUI": {
    "HealthChecks": [
      {
        "Name": "Production API",
        "Uri": "https://production.api.com/health"
      },
      {
        "Name": "Staging API",
        "Uri": "https://staging.api.com/health"
      }
    ],
    "Webhooks": [
      {
        "Name": "Slack",
        "Uri": "https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK",
        "Payload": "{\"text\": \"Health Check Status: [[LIVENESS]] - [[FAILURE]]\"}",
        "RestoredPayload": "{\"text\": \"Health Check Restored: [[LIVENESS]]\"}",
        "ShouldNotifyFunc": "[[FAILURE]] != \"\""
      }
    ],
    "EvaluationTimeInSeconds": 60,
    "MinimumSecondsBetweenFailureNotifications": 300
  }
}
```

## 모니터링 통합

### Application Insights 통합
```csharp
// Program.cs
builder.Services.AddApplicationInsightsTelemetry();
builder.Services.AddApplicationInsightsKubernetesEnricher();

// 사용자 정의 Telemetry Initializer
public class CustomTelemetryInitializer : ITelemetryInitializer
{
    private readonly IHttpContextAccessor _httpContextAccessor;
    
    public CustomTelemetryInitializer(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }
    
    public void Initialize(ITelemetry telemetry)
    {
        var httpContext = _httpContextAccessor.HttpContext;
        
        if (httpContext != null)
        {
            telemetry.Context.User.Id = httpContext.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
            telemetry.Context.Session.Id = httpContext.Session?.Id;
            
            if (telemetry is RequestTelemetry requestTelemetry)
            {
                requestTelemetry.Properties["RequestId"] = httpContext.TraceIdentifier;
                requestTelemetry.Properties["UserAgent"] = httpContext.Request.Headers["User-Agent"];
            }
        }
        
        telemetry.Context.Cloud.RoleName = "MyAPI";
        telemetry.Context.Cloud.RoleInstance = Environment.MachineName;
    }
}

builder.Services.AddSingleton<ITelemetryInitializer, CustomTelemetryInitializer>();
```

### Prometheus 메트릭
```csharp
// NuGet: prometheus-net.AspNetCore
builder.Services.AddSingleton<MetricReporter>();

public class MetricReporter
{
    private readonly Counter _requestCounter;
    private readonly Histogram _requestDuration;
    private readonly Gauge _activeConnections;
    
    public MetricReporter()
    {
        _requestCounter = Metrics.CreateCounter(
            "http_requests_total",
            "Total HTTP requests",
            new CounterConfiguration
            {
                LabelNames = new[] { "method", "endpoint", "status" }
            });
        
        _requestDuration = Metrics.CreateHistogram(
            "http_request_duration_seconds",
            "HTTP request duration",
            new HistogramConfiguration
            {
                LabelNames = new[] { "method", "endpoint" },
                Buckets = Histogram.LinearBuckets(0.001, 0.001, 100)
            });
        
        _activeConnections = Metrics.CreateGauge(
            "active_connections",
            "Number of active connections");
    }
    
    public void RecordRequest(string method, string endpoint, int statusCode, double duration)
    {
        _requestCounter.WithLabels(method, endpoint, statusCode.ToString()).Inc();
        _requestDuration.WithLabels(method, endpoint).Observe(duration);
    }
    
    public void SetActiveConnections(int count)
    {
        _activeConnections.Set(count);
    }
}

// Middleware
public class MetricsMiddleware
{
    private readonly RequestDelegate _next;
    private readonly MetricReporter _reporter;
    
    public MetricsMiddleware(RequestDelegate next, MetricReporter reporter)
    {
        _next = next;
        _reporter = reporter;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        var stopwatch = Stopwatch.StartNew();
        
        try
        {
            await _next(context);
        }
        finally
        {
            stopwatch.Stop();
            
            _reporter.RecordRequest(
                context.Request.Method,
                context.Request.Path,
                context.Response.StatusCode,
                stopwatch.Elapsed.TotalSeconds);
        }
    }
}

// 엔드포인트 설정
app.UseHttpMetrics();
app.MapMetrics(); // /metrics 엔드포인트
```

## 로깅과 추적

### 구조화된 로깅
```csharp
public class OrderService
{
    private readonly ILogger<OrderService> _logger;
    
    public async Task<Order> CreateOrderAsync(CreateOrderDto dto)
    {
        using (_logger.BeginScope(new Dictionary<string, object>
        {
            ["OrderId"] = Guid.NewGuid(),
            ["CustomerId"] = dto.CustomerId,
            ["Operation"] = "CreateOrder"
        }))
        {
            _logger.LogInformation("Creating order for customer {CustomerId}", dto.CustomerId);
            
            try
            {
                var order = new Order
                {
                    Id = Guid.NewGuid(),
                    CustomerId = dto.CustomerId,
                    Items = dto.Items,
                    CreatedAt = DateTime.UtcNow
                };
                
                // 주문 처리 로직
                
                _logger.LogInformation(
                    "Order created successfully. OrderId: {OrderId}, Total: {Total}",
                    order.Id,
                    order.TotalAmount);
                
                return order;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, 
                    "Failed to create order for customer {CustomerId}",
                    dto.CustomerId);
                throw;
            }
        }
    }
}
```

### 분산 추적 (OpenTelemetry)
```csharp
// Program.cs
builder.Services.AddOpenTelemetry()
    .WithTracing(builder =>
    {
        builder
            .SetResourceBuilder(ResourceBuilder.CreateDefault()
                .AddService("MyAPI", serviceVersion: "1.0.0"))
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddSqlClientInstrumentation()
            .AddRedisInstrumentation()
            .AddSource("MyAPI.Orders")
            .AddJaegerExporter(options =>
            {
                options.AgentHost = "localhost";
                options.AgentPort = 6831;
            });
    })
    .WithMetrics(builder =>
    {
        builder
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddRuntimeInstrumentation()
            .AddPrometheusExporter();
    });

// 사용자 정의 추적
public class OrderService
{
    private static readonly ActivitySource ActivitySource = new("MyAPI.Orders");
    
    public async Task<Order> ProcessOrderAsync(Guid orderId)
    {
        using var activity = ActivitySource.StartActivity("ProcessOrder");
        activity?.SetTag("order.id", orderId.ToString());
        
        try
        {
            // 하위 작업 추적
            using (var validateActivity = ActivitySource.StartActivity("ValidateOrder"))
            {
                await ValidateOrderAsync(orderId);
            }
            
            using (var paymentActivity = ActivitySource.StartActivity("ProcessPayment"))
            {
                await ProcessPaymentAsync(orderId);
            }
            
            activity?.SetTag("order.status", "completed");
            
            return await GetOrderAsync(orderId);
        }
        catch (Exception ex)
        {
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            throw;
        }
    }
}
```

## 알림 및 대응

### 알림 서비스
```csharp
public interface IAlertService
{
    Task SendAlertAsync(Alert alert);
}

public class MultiChannelAlertService : IAlertService
{
    private readonly IEnumerable<IAlertChannel> _channels;
    private readonly ILogger<MultiChannelAlertService> _logger;
    
    public MultiChannelAlertService(
        IEnumerable<IAlertChannel> channels,
        ILogger<MultiChannelAlertService> logger)
    {
        _channels = channels;
        _logger = logger;
    }
    
    public async Task SendAlertAsync(Alert alert)
    {
        var tasks = _channels
            .Where(channel => channel.ShouldSend(alert))
            .Select(async channel =>
            {
                try
                {
                    await channel.SendAsync(alert);
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex, 
                        "Failed to send alert via {Channel}",
                        channel.GetType().Name);
                }
            });
        
        await Task.WhenAll(tasks);
    }
}

// Slack 알림 채널
public class SlackAlertChannel : IAlertChannel
{
    private readonly HttpClient _httpClient;
    private readonly string _webhookUrl;
    
    public bool ShouldSend(Alert alert) => alert.Severity >= AlertSeverity.Warning;
    
    public async Task SendAsync(Alert alert)
    {
        var payload = new
        {
            text = $"🚨 *{alert.Title}*",
            attachments = new[]
            {
                new
                {
                    color = GetColor(alert.Severity),
                    fields = new[]
                    {
                        new { title = "Severity", value = alert.Severity.ToString(), @short = true },
                        new { title = "Time", value = alert.Timestamp.ToString("yyyy-MM-dd HH:mm:ss"), @short = true },
                        new { title = "Description", value = alert.Description, @short = false },
                        new { title = "Source", value = alert.Source, @short = true }
                    }
                }
            }
        };
        
        await _httpClient.PostAsJsonAsync(_webhookUrl, payload);
    }
    
    private string GetColor(AlertSeverity severity) => severity switch
    {
        AlertSeverity.Critical => "#FF0000",
        AlertSeverity.Warning => "#FFA500",
        AlertSeverity.Info => "#0000FF",
        _ => "#808080"
    };
}
```

### 자동 복구 시스템
```csharp
public class SelfHealingService : BackgroundService
{
    private readonly IHealthCheckService _healthCheckService;
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<SelfHealingService> _logger;
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                var report = await _healthCheckService.CheckHealthAsync(stoppingToken);
                
                foreach (var entry in report.Entries.Where(e => e.Value.Status == HealthStatus.Unhealthy))
                {
                    await TryHealAsync(entry.Key, entry.Value);
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error in self-healing service");
            }
            
            await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
        }
    }
    
    private async Task TryHealAsync(string healthCheckName, HealthCheckResult result)
    {
        _logger.LogWarning(
            "Attempting to heal unhealthy service: {Service}",
            healthCheckName);
        
        using var scope = _serviceProvider.CreateScope();
        
        switch (healthCheckName)
        {
            case "database":
                await HealDatabaseAsync(scope.ServiceProvider);
                break;
                
            case "redis":
                await HealRedisAsync(scope.ServiceProvider);
                break;
                
            case "message-queue":
                await HealMessageQueueAsync(scope.ServiceProvider);
                break;
                
            default:
                _logger.LogWarning(
                    "No healing action defined for {Service}",
                    healthCheckName);
                break;
        }
    }
    
    private async Task HealDatabaseAsync(IServiceProvider serviceProvider)
    {
        var dbContext = serviceProvider.GetRequiredService<ApplicationDbContext>();
        
        try
        {
            // 연결 풀 재설정
            await dbContext.Database.CloseConnectionAsync();
            await Task.Delay(1000);
            
            // 연결 재시도
            await dbContext.Database.CanConnectAsync();
            
            _logger.LogInformation("Database connection restored");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to heal database connection");
        }
    }
}
```

## 대시보드 구성

### 실시간 모니터링 대시보드
```csharp
public class MonitoringHub : Hub
{
    private readonly IMetricsCollector _metricsCollector;
    
    public override async Task OnConnectedAsync()
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, "monitoring");
        await Clients.Caller.SendAsync("InitialData", await _metricsCollector.GetCurrentMetricsAsync());
        await base.OnConnectedAsync();
    }
    
    public async Task SubscribeToMetric(string metricName)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, $"metric:{metricName}");
    }
}

// 백그라운드 서비스로 메트릭 브로드캐스트
public class MetricsBroadcastService : BackgroundService
{
    private readonly IHubContext<MonitoringHub> _hubContext;
    private readonly IMetricsCollector _metricsCollector;
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var metrics = await _metricsCollector.GetCurrentMetricsAsync();
            
            await _hubContext.Clients.Group("monitoring").SendAsync(
                "MetricsUpdate",
                metrics,
                stoppingToken);
            
            // 특정 메트릭 업데이트
            foreach (var metric in metrics)
            {
                await _hubContext.Clients.Group($"metric:{metric.Name}").SendAsync(
                    "MetricUpdate",
                    metric,
                    stoppingToken);
            }
            
            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }
    }
}
```

## 모범 사례

### Health Check 구성 관리
```csharp
public class HealthCheckConfiguration
{
    public Dictionary<string, HealthCheckConfig> Checks { get; set; } = new();
}

public class HealthCheckConfig
{
    public bool Enabled { get; set; } = true;
    public string[] Tags { get; set; } = Array.Empty<string>();
    public int TimeoutSeconds { get; set; } = 30;
    public Dictionary<string, string> Settings { get; set; } = new();
}

// 동적 Health Check 등록
public static class HealthCheckExtensions
{
    public static IHealthChecksBuilder AddConfiguredHealthChecks(
        this IHealthChecksBuilder builder,
        IConfiguration configuration)
    {
        var healthCheckConfig = configuration
            .GetSection("HealthChecks")
            .Get<HealthCheckConfiguration>();
        
        foreach (var (name, config) in healthCheckConfig.Checks)
        {
            if (!config.Enabled) continue;
            
            switch (name)
            {
                case "SqlServer":
                    builder.AddSqlServer(
                        config.Settings["ConnectionString"],
                        name: name,
                        tags: config.Tags,
                        timeout: TimeSpan.FromSeconds(config.TimeoutSeconds));
                    break;
                    
                case "Redis":
                    builder.AddRedis(
                        config.Settings["ConnectionString"],
                        name: name,
                        tags: config.Tags,
                        timeout: TimeSpan.FromSeconds(config.TimeoutSeconds));
                    break;
                    
                case "HttpEndpoint":
                    builder.AddUrlGroup(
                        new Uri(config.Settings["Url"]),
                        name: name,
                        tags: config.Tags,
                        timeout: TimeSpan.FromSeconds(config.TimeoutSeconds));
                    break;
            }
        }
        
        return builder;
    }
}
```

Health Checks와 모니터링은 프로덕션 환경에서 애플리케이션의 안정성과 가용성을 보장하는 핵심 요소입니다. 적절한 모니터링 전략을 통해 문제를 조기에 발견하고 대응할 수 있습니다.