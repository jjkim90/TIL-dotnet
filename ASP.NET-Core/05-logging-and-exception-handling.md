# 로깅과 예외 처리

## 로깅 시스템 개요

ASP.NET Core는 구조화된 로깅을 위한 통합 로깅 API를 제공합니다. 다양한 로깅 공급자를 통해 콘솔, 파일, 데이터베이스 등으로 로그를 출력할 수 있습니다.

### 로그 레벨
```csharp
public enum LogLevel
{
    Trace = 0,      // 가장 상세한 정보
    Debug = 1,      // 디버깅 정보
    Information = 2, // 일반 정보
    Warning = 3,    // 경고
    Error = 4,      // 오류
    Critical = 5,   // 심각한 오류
    None = 6        // 로깅 안 함
}
```

## 기본 로깅 구성

### Program.cs에서 로깅 구성
```csharp
var builder = WebApplication.CreateBuilder(args);

// 로깅 구성
builder.Logging.ClearProviders(); // 기본 공급자 제거
builder.Logging.AddConsole();     // 콘솔 로깅 추가
builder.Logging.AddDebug();       // 디버그 로깅 추가

// 로그 레벨 구성
builder.Logging.SetMinimumLevel(LogLevel.Information);
builder.Logging.AddFilter("Microsoft", LogLevel.Warning);
builder.Logging.AddFilter("System", LogLevel.Warning);

var app = builder.Build();
```

### appsettings.json에서 로깅 구성
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information",
      "Microsoft.EntityFrameworkCore": "Warning"
    },
    "Console": {
      "IncludeScopes": true,
      "TimestampFormat": "yyyy-MM-dd HH:mm:ss "
    }
  }
}
```

## ILogger 사용하기

### 컨트롤러에서 로깅
```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly ILogger<ProductsController> _logger;
    private readonly IProductService _productService;
    
    public ProductsController(
        ILogger<ProductsController> logger,
        IProductService productService)
    {
        _logger = logger;
        _productService = productService;
    }
    
    [HttpGet("{id}")]
    public async Task<IActionResult> GetProduct(int id)
    {
        _logger.LogInformation("Getting product with id: {ProductId}", id);
        
        try
        {
            var product = await _productService.GetByIdAsync(id);
            
            if (product == null)
            {
                _logger.LogWarning("Product not found: {ProductId}", id);
                return NotFound();
            }
            
            _logger.LogInformation("Successfully retrieved product: {ProductId}", id);
            return Ok(product);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error retrieving product: {ProductId}", id);
            return StatusCode(500, "Internal server error");
        }
    }
}
```

### 서비스에서 로깅
```csharp
public interface IEmailService
{
    Task SendEmailAsync(string to, string subject, string body);
}

public class EmailService : IEmailService
{
    private readonly ILogger<EmailService> _logger;
    private readonly SmtpSettings _smtpSettings;
    
    public EmailService(
        ILogger<EmailService> logger,
        IOptions<SmtpSettings> smtpSettings)
    {
        _logger = logger;
        _smtpSettings = smtpSettings.Value;
    }
    
    public async Task SendEmailAsync(string to, string subject, string body)
    {
        using (_logger.BeginScope("Sending email to {EmailAddress}", to))
        {
            _logger.LogInformation("Starting email send");
            
            try
            {
                // 이메일 발송 로직
                await SendEmailInternal(to, subject, body);
                
                _logger.LogInformation(
                    "Email sent successfully to {EmailAddress} with subject {Subject}", 
                    to, subject);
            }
            catch (SmtpException ex)
            {
                _logger.LogError(ex, 
                    "SMTP error sending email to {EmailAddress}", to);
                throw;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, 
                    "Unexpected error sending email to {EmailAddress}", to);
                throw;
            }
        }
    }
}
```

## 구조화된 로깅

### 로그 메시지 템플릿
```csharp
public class OrderService
{
    private readonly ILogger<OrderService> _logger;
    
    public async Task<Order> CreateOrderAsync(CreateOrderDto dto)
    {
        // 구조화된 로깅 - 파라미터는 템플릿의 플레이스홀더와 매칭
        _logger.LogInformation(
            "Creating order for customer {CustomerId} with {ItemCount} items",
            dto.CustomerId, 
            dto.Items.Count);
        
        var order = new Order
        {
            Id = Guid.NewGuid(),
            CustomerId = dto.CustomerId,
            CreatedAt = DateTime.UtcNow,
            TotalAmount = dto.Items.Sum(i => i.Price * i.Quantity)
        };
        
        // 복잡한 객체 로깅
        _logger.LogInformation(
            "Order created: {@Order}", // @ 사용 시 객체 직렬화
            new { order.Id, order.CustomerId, order.TotalAmount });
        
        return order;
    }
}
```

### 로그 스코프 사용
```csharp
public class PaymentProcessor
{
    private readonly ILogger<PaymentProcessor> _logger;
    
    public async Task ProcessPaymentAsync(PaymentRequest request)
    {
        // 스코프를 사용하여 관련 로그 그룹화
        using (_logger.BeginScope("Processing payment {PaymentId}", request.PaymentId))
        {
            _logger.LogInformation("Starting payment processing");
            
            // 스코프 내의 모든 로그에 PaymentId가 포함됨
            await ValidatePaymentAsync(request);
            await ChargePaymentAsync(request);
            await SendConfirmationAsync(request);
            
            _logger.LogInformation("Payment processing completed");
        }
    }
}
```

## 예외 처리

### 전역 예외 처리 미들웨어
```csharp
public class GlobalExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<GlobalExceptionHandlingMiddleware> _logger;
    
    public GlobalExceptionHandlingMiddleware(
        RequestDelegate next,
        ILogger<GlobalExceptionHandlingMiddleware> logger)
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
            _logger.LogError(ex, "An unhandled exception occurred");
            await HandleExceptionAsync(context, ex);
        }
    }
    
    private static async Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        context.Response.ContentType = "application/json";
        
        var response = new ErrorResponse
        {
            TraceId = Activity.Current?.Id ?? context.TraceIdentifier,
            Message = "An error occurred processing your request"
        };
        
        switch (exception)
        {
            case NotFoundException:
                context.Response.StatusCode = StatusCodes.Status404NotFound;
                response.Message = exception.Message;
                break;
                
            case ValidationException validationEx:
                context.Response.StatusCode = StatusCodes.Status400BadRequest;
                response.Message = "Validation failed";
                response.Errors = validationEx.Errors;
                break;
                
            case UnauthorizedException:
                context.Response.StatusCode = StatusCodes.Status401Unauthorized;
                response.Message = "Unauthorized";
                break;
                
            default:
                context.Response.StatusCode = StatusCodes.Status500InternalServerError;
                break;
        }
        
        var jsonResponse = JsonSerializer.Serialize(response);
        await context.Response.WriteAsync(jsonResponse);
    }
}

// ErrorResponse 모델
public class ErrorResponse
{
    public string TraceId { get; set; }
    public string Message { get; set; }
    public IDictionary<string, string[]> Errors { get; set; }
    public DateTime Timestamp { get; set; } = DateTime.UtcNow;
}

// Program.cs에서 등록
app.UseMiddleware<GlobalExceptionHandlingMiddleware>();
```

### 사용자 정의 예외
```csharp
public abstract class DomainException : Exception
{
    protected DomainException(string message) : base(message) { }
}

public class NotFoundException : DomainException
{
    public NotFoundException(string entityName, object id) 
        : base($"{entityName} with id '{id}' was not found") { }
}

public class ValidationException : DomainException
{
    public IDictionary<string, string[]> Errors { get; }
    
    public ValidationException(IDictionary<string, string[]> errors) 
        : base("Validation failed")
    {
        Errors = errors;
    }
}

public class BusinessRuleException : DomainException
{
    public string Code { get; }
    
    public BusinessRuleException(string code, string message) 
        : base(message)
    {
        Code = code;
    }
}
```

### 예외 필터 사용
```csharp
public class ApiExceptionFilterAttribute : ExceptionFilterAttribute
{
    private readonly ILogger<ApiExceptionFilterAttribute> _logger;
    
    public ApiExceptionFilterAttribute(ILogger<ApiExceptionFilterAttribute> logger)
    {
        _logger = logger;
    }
    
    public override void OnException(ExceptionContext context)
    {
        _logger.LogError(context.Exception, "An exception occurred");
        
        if (context.Exception is ValidationException validationException)
        {
            context.Result = new BadRequestObjectResult(new
            {
                Message = "Validation failed",
                Errors = validationException.Errors
            });
            context.ExceptionHandled = true;
        }
        else if (context.Exception is NotFoundException)
        {
            context.Result = new NotFoundObjectResult(new
            {
                Message = context.Exception.Message
            });
            context.ExceptionHandled = true;
        }
        
        base.OnException(context);
    }
}

// 컨트롤러에 적용
[ApiController]
[ServiceFilter(typeof(ApiExceptionFilterAttribute))]
public class ProductsController : ControllerBase
{
    // ...
}
```

## 고급 로깅 기법

### 성능 로깅
```csharp
public class PerformanceLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<PerformanceLoggingMiddleware> _logger;
    
    public PerformanceLoggingMiddleware(
        RequestDelegate next,
        ILogger<PerformanceLoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
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
            var elapsedMilliseconds = stopwatch.ElapsedMilliseconds;
            
            if (elapsedMilliseconds > 500) // 500ms 이상 걸린 요청
            {
                _logger.LogWarning(
                    "Long running request: {Method} {Path} took {ElapsedMilliseconds}ms",
                    context.Request.Method,
                    context.Request.Path,
                    elapsedMilliseconds);
            }
            else
            {
                _logger.LogInformation(
                    "Request completed: {Method} {Path} in {ElapsedMilliseconds}ms",
                    context.Request.Method,
                    context.Request.Path,
                    elapsedMilliseconds);
            }
        }
    }
}
```

### 조건부 로깅
```csharp
public class OptimizedLoggingService
{
    private readonly ILogger<OptimizedLoggingService> _logger;
    
    public void ProcessData(IEnumerable<DataItem> items)
    {
        // 로그 레벨 확인으로 불필요한 문자열 생성 방지
        if (_logger.IsEnabled(LogLevel.Debug))
        {
            _logger.LogDebug("Processing {Count} items: {Items}", 
                items.Count(), 
                string.Join(", ", items.Select(i => i.Id)));
        }
        
        // LoggerMessage 사용으로 성능 최적화
        LogMessages.ProcessingStarted(_logger, items.Count());
        
        foreach (var item in items)
        {
            ProcessItem(item);
        }
        
        LogMessages.ProcessingCompleted(_logger, items.Count());
    }
}

// 고성능 로깅을 위한 LoggerMessage 정의
public static class LogMessages
{
    private static readonly Action<ILogger, int, Exception?> _processingStarted =
        LoggerMessage.Define<int>(
            LogLevel.Information,
            new EventId(1, nameof(ProcessingStarted)),
            "Started processing {Count} items");
    
    private static readonly Action<ILogger, int, Exception?> _processingCompleted =
        LoggerMessage.Define<int>(
            LogLevel.Information,
            new EventId(2, nameof(ProcessingCompleted)),
            "Completed processing {Count} items");
    
    public static void ProcessingStarted(ILogger logger, int count)
        => _processingStarted(logger, count, null);
    
    public static void ProcessingCompleted(ILogger logger, int count)
        => _processingCompleted(logger, count, null);
}
```

## 타사 로깅 프레임워크 통합

### Serilog 통합
```csharp
// Program.cs
using Serilog;

var builder = WebApplication.CreateBuilder(args);

// Serilog 구성
Log.Logger = new LoggerConfiguration()
    .ReadFrom.Configuration(builder.Configuration)
    .Enrich.FromLogContext()
    .Enrich.WithMachineName()
    .Enrich.WithThreadId()
    .WriteTo.Console()
    .WriteTo.File("logs/log-.txt", rollingInterval: RollingInterval.Day)
    .WriteTo.Seq("http://localhost:5341") // 구조화된 로그 뷰어
    .CreateLogger();

builder.Host.UseSerilog();

var app = builder.Build();

// 요청 로깅
app.UseSerilogRequestLogging(options =>
{
    options.MessageTemplate = "HTTP {RequestMethod} {RequestPath} responded {StatusCode} in {Elapsed:0.0000} ms";
    options.EnrichDiagnosticContext = (diagnosticContext, httpContext) =>
    {
        diagnosticContext.Set("RequestHost", httpContext.Request.Host.Value);
        diagnosticContext.Set("RequestScheme", httpContext.Request.Scheme);
        diagnosticContext.Set("UserAgent", httpContext.Request.Headers["User-Agent"].ToString());
    };
});
```

### NLog 통합
```csharp
// Program.cs
using NLog.Web;

var builder = WebApplication.CreateBuilder(args);

// NLog 설정
builder.Logging.ClearProviders();
builder.Host.UseNLog();

// nlog.config
<?xml version="1.0" encoding="utf-8" ?>
<nlog xmlns="http://www.nlog-project.org/schemas/NLog.xsd"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  
  <targets>
    <target xsi:type="File" name="file" fileName="logs/${shortdate}.log"
            layout="${longdate} ${uppercase:${level}} ${message} ${exception:format=tostring}" />
    <target xsi:type="Console" name="console"
            layout="${time} ${level:uppercase=true} ${message}" />
  </targets>
  
  <rules>
    <logger name="*" minlevel="Info" writeTo="console" />
    <logger name="*" minlevel="Warning" writeTo="file" />
  </rules>
</nlog>
```

## 로그 분석과 모니터링

### Application Insights 통합
```csharp
// Program.cs
builder.Services.AddApplicationInsightsTelemetry();

// 사용자 정의 텔레메트리
public class TelemetryService
{
    private readonly TelemetryClient _telemetryClient;
    private readonly ILogger<TelemetryService> _logger;
    
    public TelemetryService(
        TelemetryClient telemetryClient,
        ILogger<TelemetryService> logger)
    {
        _telemetryClient = telemetryClient;
        _logger = logger;
    }
    
    public void TrackCustomEvent(string eventName, IDictionary<string, string> properties)
    {
        _telemetryClient.TrackEvent(eventName, properties);
        _logger.LogInformation("Custom event tracked: {EventName}", eventName);
    }
    
    public void TrackException(Exception exception, IDictionary<string, string> properties)
    {
        _telemetryClient.TrackException(exception, properties);
        _logger.LogError(exception, "Exception tracked with properties: {@Properties}", properties);
    }
}
```

## 실전 예제: 통합 로깅 시스템

```csharp
// 로깅 확장 메서드
public static class LoggingExtensions
{
    public static ILoggingBuilder AddCustomLogging(
        this ILoggingBuilder builder, 
        IConfiguration configuration)
    {
        builder.ClearProviders();
        
        // 콘솔 로깅
        builder.AddConsole(options =>
        {
            options.IncludeScopes = true;
            options.TimestampFormat = "yyyy-MM-dd HH:mm:ss ";
        });
        
        // 디버그 로깅 (개발 환경)
        if (builder.Services.BuildServiceProvider()
            .GetRequiredService<IWebHostEnvironment>().IsDevelopment())
        {
            builder.AddDebug();
        }
        
        // 파일 로깅 (간단한 구현)
        builder.Services.AddSingleton<ILoggerProvider, FileLoggerProvider>();
        
        return builder;
    }
}

// 파일 로거 공급자
public class FileLoggerProvider : ILoggerProvider
{
    private readonly string _path;
    private readonly object _lock = new object();
    
    public FileLoggerProvider()
    {
        _path = Path.Combine(Directory.GetCurrentDirectory(), "logs");
        Directory.CreateDirectory(_path);
    }
    
    public ILogger CreateLogger(string categoryName)
    {
        return new FileLogger(categoryName, _path, _lock);
    }
    
    public void Dispose() { }
}

// 파일 로거
public class FileLogger : ILogger
{
    private readonly string _categoryName;
    private readonly string _path;
    private readonly object _lock;
    
    public FileLogger(string categoryName, string path, object @lock)
    {
        _categoryName = categoryName;
        _path = path;
        _lock = @lock;
    }
    
    public IDisposable BeginScope<TState>(TState state) => null;
    
    public bool IsEnabled(LogLevel logLevel) => logLevel >= LogLevel.Information;
    
    public void Log<TState>(
        LogLevel logLevel, 
        EventId eventId, 
        TState state, 
        Exception exception, 
        Func<TState, Exception, string> formatter)
    {
        if (!IsEnabled(logLevel)) return;
        
        var fileName = $"{DateTime.Now:yyyy-MM-dd}.log";
        var fullPath = Path.Combine(_path, fileName);
        var message = $"{DateTime.Now:yyyy-MM-dd HH:mm:ss} [{logLevel}] {_categoryName} - {formatter(state, exception)}";
        
        if (exception != null)
        {
            message += Environment.NewLine + exception.ToString();
        }
        
        lock (_lock)
        {
            File.AppendAllText(fullPath, message + Environment.NewLine);
        }
    }
}

// Program.cs
var builder = WebApplication.CreateBuilder(args);

// 사용자 정의 로깅 구성
builder.Logging.AddCustomLogging(builder.Configuration);

var app = builder.Build();

// 로깅 테스트 엔드포인트
app.MapGet("/test-logging", (ILogger<Program> logger) =>
{
    logger.LogTrace("This is trace");
    logger.LogDebug("This is debug");
    logger.LogInformation("This is information");
    logger.LogWarning("This is warning");
    logger.LogError("This is error");
    logger.LogCritical("This is critical");
    
    return "Logging test completed";
});

app.Run();
```