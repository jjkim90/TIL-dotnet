# API 보안 - CORS, HTTPS, Security Headers

## API 보안 개요

웹 API 보안은 여러 계층에서 구현되어야 합니다. 이 장에서는 CORS(Cross-Origin Resource Sharing), HTTPS 구성, 그리고 다양한 보안 헤더 설정에 대해 다룹니다.

### 보안의 핵심 요소
- **CORS**: 크로스 오리진 요청 제어
- **HTTPS**: 전송 계층 보안
- **Security Headers**: HTTP 보안 헤더
- **Rate Limiting**: API 요청 제한
- **Input Validation**: 입력 검증

## CORS (Cross-Origin Resource Sharing)

### CORS 이해하기
```csharp
// CORS는 웹 브라우저가 한 출처(origin)에서 실행 중인 웹 애플리케이션이
// 다른 출처의 자원에 접근할 수 있는 권한을 부여하는 메커니즘

// Origin = Protocol + Domain + Port
// https://example.com:443 (기본 포트는 생략 가능)
// http://localhost:3000

// Same-Origin Policy (동일 출처 정책)
// 브라우저는 기본적으로 동일 출처 정책을 적용하여
// 다른 출처의 리소스 접근을 제한
```

### 기본 CORS 설정
```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// CORS 서비스 추가
builder.Services.AddCors(options =>
{
    // 기본 정책
    options.AddDefaultPolicy(builder =>
    {
        builder.WithOrigins("https://localhost:3000", "https://myapp.com")
               .AllowAnyHeader()
               .AllowAnyMethod();
    });
    
    // 명명된 정책
    options.AddPolicy("AllowSpecificOrigin", builder =>
    {
        builder.WithOrigins("https://trusted-app.com")
               .WithHeaders("Content-Type", "Authorization")
               .WithMethods("GET", "POST");
    });
    
    // 개발 환경용 정책
    options.AddPolicy("DevelopmentPolicy", builder =>
    {
        builder.AllowAnyOrigin()
               .AllowAnyHeader()
               .AllowAnyMethod();
    });
});

var app = builder.Build();

// CORS 미들웨어 사용 (UseRouting 전에 위치)
app.UseCors();

// 특정 엔드포인트에 다른 정책 적용
app.MapControllers().RequireCors("AllowSpecificOrigin");
```

### 세부적인 CORS 구성
```csharp
public class CorsConfiguration
{
    public static void ConfigureCors(IServiceCollection services, IConfiguration configuration)
    {
        var allowedOrigins = configuration.GetSection("Cors:AllowedOrigins").Get<string[]>();
        
        services.AddCors(options =>
        {
            options.AddPolicy("ProductionPolicy", builder =>
            {
                builder.WithOrigins(allowedOrigins)
                       .AllowAnyHeader()
                       .AllowAnyMethod()
                       .AllowCredentials() // 쿠키/인증 정보 포함 허용
                       .SetPreflightMaxAge(TimeSpan.FromSeconds(3600)) // Preflight 캐시
                       .WithExposedHeaders("X-Total-Count", "X-Page-Number"); // 클라이언트에 노출할 헤더
            });
            
            // 동적 Origin 검증
            options.AddPolicy("DynamicOriginPolicy", builder =>
            {
                builder.SetIsOriginAllowed(origin =>
                {
                    // 동적으로 Origin 검증
                    var uri = new Uri(origin);
                    
                    // 서브도메인 허용
                    if (uri.Host.EndsWith(".mycompany.com"))
                        return true;
                    
                    // 특정 포트 범위 허용
                    if (uri.Host == "localhost" && uri.Port >= 3000 && uri.Port <= 3999)
                        return true;
                    
                    return false;
                })
                .AllowAnyHeader()
                .AllowAnyMethod()
                .AllowCredentials();
            });
        });
    }
}

// appsettings.json
{
  "Cors": {
    "AllowedOrigins": [
      "https://app.mycompany.com",
      "https://admin.mycompany.com"
    ]
  }
}
```

### 컨트롤러별 CORS 설정
```csharp
[ApiController]
[Route("api/[controller]")]
[EnableCors("AllowSpecificOrigin")] // 컨트롤러 레벨
public class ProductsController : ControllerBase
{
    [HttpGet]
    public IActionResult GetProducts()
    {
        return Ok(new { products = new[] { "Product1", "Product2" } });
    }
    
    [HttpPost]
    [DisableCors] // 특정 액션에서 CORS 비활성화
    public IActionResult CreateProduct([FromBody] ProductDto product)
    {
        // 이 엔드포인트는 CORS를 허용하지 않음
        return Ok();
    }
    
    [HttpGet("public")]
    [EnableCors("DevelopmentPolicy")] // 액션별 다른 정책
    public IActionResult GetPublicData()
    {
        return Ok(new { data = "Public data" });
    }
}
```

## HTTPS 구성

### HTTPS 리다이렉션
```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// HTTPS 리다이렉션 구성
builder.Services.AddHttpsRedirection(options =>
{
    options.RedirectStatusCode = StatusCodes.Status307TemporaryRedirect;
    options.HttpsPort = 443; // 기본 HTTPS 포트
});

// Kestrel HTTPS 구성
builder.WebHost.ConfigureKestrel(serverOptions =>
{
    serverOptions.ConfigureHttpsDefaults(httpsOptions =>
    {
        // TLS 버전 제한
        httpsOptions.SslProtocols = SslProtocols.Tls12 | SslProtocols.Tls13;
        
        // 클라이언트 인증서 요구
        httpsOptions.ClientCertificateMode = ClientCertificateMode.AllowCertificate;
        
        // 인증서 유효성 검사
        httpsOptions.ClientCertificateValidation = (certificate, chain, errors) =>
        {
            // 사용자 정의 인증서 검증 로직
            return errors == SslPolicyErrors.None;
        };
    });
});

var app = builder.Build();

// 환경별 HTTPS 설정
if (!app.Environment.IsDevelopment())
{
    app.UseHttpsRedirection();
}
```

### HSTS (HTTP Strict Transport Security)
```csharp
// Program.cs
builder.Services.AddHsts(options =>
{
    options.Preload = true; // 브라우저 preload 목록에 포함
    options.IncludeSubDomains = true; // 서브도메인 포함
    options.MaxAge = TimeSpan.FromDays(365); // 1년간 유지
    options.ExcludedHosts.Add("localhost"); // 제외할 호스트
    options.ExcludedHosts.Add("127.0.0.1");
});

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseHsts(); // HSTS 헤더 추가
}

// 수동으로 HSTS 헤더 추가
app.Use(async (context, next) =>
{
    if (context.Request.IsHttps)
    {
        context.Response.Headers.Add("Strict-Transport-Security", 
            "max-age=31536000; includeSubDomains; preload");
    }
    await next();
});
```

### 인증서 관리
```csharp
// 개발 환경 인증서 설정
public class CertificateConfiguration
{
    public static X509Certificate2 LoadCertificate(IConfiguration configuration)
    {
        var certConfig = configuration.GetSection("Certificate");
        var certPath = certConfig["Path"];
        var certPassword = certConfig["Password"];
        
        if (File.Exists(certPath))
        {
            return new X509Certificate2(certPath, certPassword);
        }
        
        // Store에서 인증서 로드
        using var store = new X509Store(StoreName.My, StoreLocation.CurrentUser);
        store.Open(OpenFlags.ReadOnly);
        
        var certificate = store.Certificates
            .Find(X509FindType.FindByThumbprint, certConfig["Thumbprint"], false)
            .FirstOrDefault();
            
        return certificate ?? throw new InvalidOperationException("Certificate not found");
    }
}

// Kestrel에 인증서 적용
builder.WebHost.ConfigureKestrel(options =>
{
    options.Listen(IPAddress.Any, 5001, listenOptions =>
    {
        listenOptions.UseHttps(CertificateConfiguration.LoadCertificate(
            builder.Configuration));
    });
});
```

## Security Headers

### 기본 보안 헤더 구성
```csharp
// SecurityHeadersMiddleware.cs
public class SecurityHeadersMiddleware
{
    private readonly RequestDelegate _next;
    private readonly SecurityHeadersOptions _options;
    
    public SecurityHeadersMiddleware(RequestDelegate next, SecurityHeadersOptions options)
    {
        _next = next;
        _options = options;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        // X-Content-Type-Options
        context.Response.Headers.Add("X-Content-Type-Options", "nosniff");
        
        // X-Frame-Options
        context.Response.Headers.Add("X-Frame-Options", _options.XFrameOptions);
        
        // X-XSS-Protection
        context.Response.Headers.Add("X-XSS-Protection", "1; mode=block");
        
        // Referrer-Policy
        context.Response.Headers.Add("Referrer-Policy", _options.ReferrerPolicy);
        
        // Content-Security-Policy
        if (!string.IsNullOrEmpty(_options.ContentSecurityPolicy))
        {
            context.Response.Headers.Add("Content-Security-Policy", 
                _options.ContentSecurityPolicy);
        }
        
        // Permissions-Policy
        context.Response.Headers.Add("Permissions-Policy", 
            "camera=(), microphone=(), geolocation=()");
        
        // 기존 헤더 제거
        context.Response.Headers.Remove("Server");
        context.Response.Headers.Remove("X-Powered-By");
        
        await _next(context);
    }
}

// Options 클래스
public class SecurityHeadersOptions
{
    public string XFrameOptions { get; set; } = "DENY";
    public string ReferrerPolicy { get; set; } = "strict-origin-when-cross-origin";
    public string ContentSecurityPolicy { get; set; }
}

// Extension Method
public static class SecurityHeadersExtensions
{
    public static IApplicationBuilder UseSecurityHeaders(
        this IApplicationBuilder app, 
        Action<SecurityHeadersOptions> configureOptions = null)
    {
        var options = new SecurityHeadersOptions();
        configureOptions?.Invoke(options);
        
        return app.UseMiddleware<SecurityHeadersMiddleware>(options);
    }
}
```

### Content Security Policy (CSP)
```csharp
// CSP Builder
public class ContentSecurityPolicyBuilder
{
    private readonly Dictionary<string, List<string>> _directives = new();
    
    public ContentSecurityPolicyBuilder DefaultSrc(params string[] sources)
    {
        AddDirective("default-src", sources);
        return this;
    }
    
    public ContentSecurityPolicyBuilder ScriptSrc(params string[] sources)
    {
        AddDirective("script-src", sources);
        return this;
    }
    
    public ContentSecurityPolicyBuilder StyleSrc(params string[] sources)
    {
        AddDirective("style-src", sources);
        return this;
    }
    
    public ContentSecurityPolicyBuilder ImgSrc(params string[] sources)
    {
        AddDirective("img-src", sources);
        return this;
    }
    
    public ContentSecurityPolicyBuilder ConnectSrc(params string[] sources)
    {
        AddDirective("connect-src", sources);
        return this;
    }
    
    public ContentSecurityPolicyBuilder FontSrc(params string[] sources)
    {
        AddDirective("font-src", sources);
        return this;
    }
    
    public ContentSecurityPolicyBuilder FrameAncestors(params string[] sources)
    {
        AddDirective("frame-ancestors", sources);
        return this;
    }
    
    public ContentSecurityPolicyBuilder UpgradeInsecureRequests()
    {
        _directives["upgrade-insecure-requests"] = new List<string>();
        return this;
    }
    
    private void AddDirective(string directive, string[] sources)
    {
        if (!_directives.ContainsKey(directive))
        {
            _directives[directive] = new List<string>();
        }
        _directives[directive].AddRange(sources);
    }
    
    public string Build()
    {
        var parts = new List<string>();
        
        foreach (var directive in _directives)
        {
            if (directive.Value.Any())
            {
                parts.Add($"{directive.Key} {string.Join(" ", directive.Value)}");
            }
            else
            {
                parts.Add(directive.Key);
            }
        }
        
        return string.Join("; ", parts);
    }
}

// 사용 예제
app.UseSecurityHeaders(options =>
{
    var csp = new ContentSecurityPolicyBuilder()
        .DefaultSrc("'self'")
        .ScriptSrc("'self'", "'unsafe-inline'", "https://cdn.jsdelivr.net")
        .StyleSrc("'self'", "'unsafe-inline'", "https://fonts.googleapis.com")
        .FontSrc("'self'", "https://fonts.gstatic.com")
        .ImgSrc("'self'", "data:", "https:")
        .ConnectSrc("'self'", "https://api.mycompany.com")
        .FrameAncestors("'none'")
        .UpgradeInsecureRequests()
        .Build();
        
    options.ContentSecurityPolicy = csp;
});
```

## Rate Limiting

### 기본 Rate Limiting 구성
```csharp
// Program.cs
using Microsoft.AspNetCore.RateLimiting;
using System.Threading.RateLimiting;

builder.Services.AddRateLimiter(options =>
{
    // 전역 제한
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(context =>
        RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: context.Connection.RemoteIpAddress?.ToString() ?? "unknown",
            factory: partition => new FixedWindowRateLimiterOptions
            {
                AutoReplenishment = true,
                PermitLimit = 100,
                Window = TimeSpan.FromMinutes(1)
            }));
    
    // Fixed Window Limiter
    options.AddFixedWindowLimiter("api", options =>
    {
        options.PermitLimit = 10;
        options.Window = TimeSpan.FromMinutes(1);
        options.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        options.QueueLimit = 5;
    });
    
    // Sliding Window Limiter
    options.AddSlidingWindowLimiter("sliding", options =>
    {
        options.PermitLimit = 100;
        options.Window = TimeSpan.FromMinutes(1);
        options.SegmentsPerWindow = 4;
        options.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        options.QueueLimit = 10;
    });
    
    // Token Bucket Limiter
    options.AddTokenBucketLimiter("token", options =>
    {
        options.TokenLimit = 100;
        options.ReplenishmentPeriod = TimeSpan.FromSeconds(10);
        options.TokensPerPeriod = 20;
        options.AutoReplenishment = true;
        options.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        options.QueueLimit = 10;
    });
    
    // Concurrency Limiter
    options.AddConcurrencyLimiter("concurrent", options =>
    {
        options.PermitLimit = 10;
        options.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        options.QueueLimit = 5;
    });
    
    // 거부 시 응답 설정
    options.OnRejected = async (context, token) =>
    {
        context.HttpContext.Response.StatusCode = 429; // Too Many Requests
        
        if (context.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retryAfter))
        {
            await context.HttpContext.Response.WriteAsync(
                $"Too many requests. Please retry after {retryAfter} seconds.",
                cancellationToken: token);
        }
        else
        {
            await context.HttpContext.Response.WriteAsync(
                "Too many requests. Please retry later.",
                cancellationToken: token);
        }
    };
});

var app = builder.Build();

app.UseRateLimiter();
```

### 사용자별 Rate Limiting
```csharp
// 사용자별 Rate Limiting 정책
public class UserBasedRateLimitPolicy : IRateLimiterPolicy<string>
{
    public Func<OnRejectedContext, CancellationToken, ValueTask>? OnRejected { get; }
    
    public RateLimitPartition<string> GetPartition(HttpContext httpContext)
    {
        var userId = httpContext.User.FindFirst(ClaimTypes.NameIdentifier)?.Value ?? "anonymous";
        
        return RateLimitPartition.GetSlidingWindowLimiter(
            partitionKey: userId,
            factory: partition => new SlidingWindowRateLimiterOptions
            {
                PermitLimit = GetUserLimit(partition),
                Window = TimeSpan.FromMinutes(1),
                SegmentsPerWindow = 4,
                QueueProcessingOrder = QueueProcessingOrder.OldestFirst,
                QueueLimit = 2
            });
    }
    
    private int GetUserLimit(string userId)
    {
        // 사용자 등급에 따른 제한 설정
        if (userId == "anonymous") return 10;
        
        // Premium 사용자
        if (IsPremiumUser(userId)) return 1000;
        
        // 일반 사용자
        return 100;
    }
    
    private bool IsPremiumUser(string userId)
    {
        // 실제 구현에서는 데이터베이스 조회
        return false;
    }
}

// 등록
builder.Services.AddRateLimiter(options =>
{
    options.AddPolicy<string, UserBasedRateLimitPolicy>("user-based");
});
```

### Controller에서 Rate Limiting 사용
```csharp
[ApiController]
[Route("api/[controller]")]
[EnableRateLimiting("api")] // 컨트롤러 레벨
public class DataController : ControllerBase
{
    [HttpGet]
    public IActionResult GetData()
    {
        return Ok(new { data = "Limited data" });
    }
    
    [HttpGet("unlimited")]
    [DisableRateLimiting] // Rate Limiting 비활성화
    public IActionResult GetUnlimitedData()
    {
        return Ok(new { data = "Unlimited data" });
    }
    
    [HttpPost("process")]
    [EnableRateLimiting("token")] // 다른 정책 적용
    public async Task<IActionResult> ProcessData([FromBody] DataDto data)
    {
        // Token bucket 정책 적용
        await Task.Delay(1000); // 처리 시뮬레이션
        return Ok(new { result = "Processed" });
    }
    
    [HttpGet("concurrent")]
    [EnableRateLimiting("concurrent")]
    public async Task<IActionResult> ConcurrentOperation()
    {
        // 동시 실행 제한
        await Task.Delay(5000); // 긴 작업 시뮬레이션
        return Ok(new { result = "Completed" });
    }
}
```

## Input Validation과 보안

### SQL Injection 방지
```csharp
// 안전한 쿼리 작성
public class SecureDataAccess
{
    private readonly ApplicationDbContext _context;
    
    public SecureDataAccess(ApplicationDbContext context)
    {
        _context = context;
    }
    
    // ❌ 위험: SQL Injection 가능
    public async Task<List<Product>> GetProductsUnsafe(string category)
    {
        var sql = $"SELECT * FROM Products WHERE Category = '{category}'";
        return await _context.Products.FromSqlRaw(sql).ToListAsync();
    }
    
    // ✅ 안전: 매개변수화된 쿼리
    public async Task<List<Product>> GetProductsSafe(string category)
    {
        return await _context.Products
            .FromSqlInterpolated($"SELECT * FROM Products WHERE Category = {category}")
            .ToListAsync();
        
        // 또는
        return await _context.Products
            .FromSqlRaw("SELECT * FROM Products WHERE Category = {0}", category)
            .ToListAsync();
    }
    
    // ✅ 가장 안전: LINQ 사용
    public async Task<List<Product>> GetProductsLinq(string category)
    {
        return await _context.Products
            .Where(p => p.Category == category)
            .ToListAsync();
    }
}
```

### XSS (Cross-Site Scripting) 방지
```csharp
// HTML 인코딩
public class AntiXssService
{
    public string SanitizeHtml(string input)
    {
        if (string.IsNullOrEmpty(input))
            return input;
        
        // HTML 인코딩
        var encoded = System.Net.WebUtility.HtmlEncode(input);
        
        // 추가 위험 문자 처리
        encoded = encoded.Replace("'", "&#39;");
        encoded = encoded.Replace("\"", "&quot;");
        
        return encoded;
    }
    
    // HtmlSanitizer 라이브러리 사용
    public string SanitizeRichText(string html)
    {
        var sanitizer = new HtmlSanitizer();
        
        // 허용할 태그 설정
        sanitizer.AllowedTags.Clear();
        sanitizer.AllowedTags.Add("p");
        sanitizer.AllowedTags.Add("strong");
        sanitizer.AllowedTags.Add("em");
        sanitizer.AllowedTags.Add("u");
        sanitizer.AllowedTags.Add("a");
        
        // 허용할 속성 설정
        sanitizer.AllowedAttributes.Clear();
        sanitizer.AllowedAttributes.Add("href");
        
        // URL 스킴 제한
        sanitizer.AllowedSchemes.Clear();
        sanitizer.AllowedSchemes.Add("http");
        sanitizer.AllowedSchemes.Add("https");
        
        return sanitizer.Sanitize(html);
    }
}

// 사용자 입력 검증
[ApiController]
[Route("api/[controller]")]
public class CommentController : ControllerBase
{
    private readonly AntiXssService _antiXss;
    
    [HttpPost]
    public async Task<IActionResult> CreateComment([FromBody] CommentDto dto)
    {
        // 입력 검증
        if (ContainsDangerousContent(dto.Content))
        {
            return BadRequest("Invalid content");
        }
        
        // HTML 새니타이징
        var sanitizedContent = _antiXss.SanitizeHtml(dto.Content);
        
        // 저장
        var comment = new Comment
        {
            Content = sanitizedContent,
            CreatedAt = DateTime.UtcNow
        };
        
        await _context.SaveChangesAsync();
        
        return Ok(comment);
    }
    
    private bool ContainsDangerousContent(string content)
    {
        var dangerousPatterns = new[]
        {
            "<script", "javascript:", "onerror=", "onload=", "onclick="
        };
        
        return dangerousPatterns.Any(pattern => 
            content.Contains(pattern, StringComparison.OrdinalIgnoreCase));
    }
}
```

### 파일 업로드 보안
```csharp
public class SecureFileUploadService
{
    private readonly string[] _allowedExtensions = { ".jpg", ".jpeg", ".png", ".gif", ".pdf" };
    private readonly Dictionary<string, string> _allowedMimeTypes = new()
    {
        { ".jpg", "image/jpeg" },
        { ".jpeg", "image/jpeg" },
        { ".png", "image/png" },
        { ".gif", "image/gif" },
        { ".pdf", "application/pdf" }
    };
    private const long MaxFileSize = 10 * 1024 * 1024; // 10MB
    
    public async Task<(bool Success, string Error)> ValidateAndSaveFile(
        IFormFile file, string uploadPath)
    {
        // 파일 크기 검증
        if (file.Length > MaxFileSize)
        {
            return (false, "File size exceeds maximum allowed size");
        }
        
        // 확장자 검증
        var extension = Path.GetExtension(file.FileName).ToLowerInvariant();
        if (!_allowedExtensions.Contains(extension))
        {
            return (false, "File type not allowed");
        }
        
        // MIME 타입 검증
        if (!_allowedMimeTypes.TryGetValue(extension, out var expectedMimeType) ||
            file.ContentType != expectedMimeType)
        {
            return (false, "Invalid file content type");
        }
        
        // 파일 내용 검증 (Magic Number)
        if (!await ValidateFileContent(file, extension))
        {
            return (false, "File content does not match extension");
        }
        
        // 안전한 파일명 생성
        var safeFileName = GenerateSafeFileName(file.FileName);
        var filePath = Path.Combine(uploadPath, safeFileName);
        
        // 바이러스 검사 (선택적)
        if (!await ScanForVirus(file))
        {
            return (false, "File failed virus scan");
        }
        
        // 파일 저장
        using (var stream = new FileStream(filePath, FileMode.Create))
        {
            await file.CopyToAsync(stream);
        }
        
        return (true, null);
    }
    
    private async Task<bool> ValidateFileContent(IFormFile file, string extension)
    {
        var buffer = new byte[8];
        using var stream = file.OpenReadStream();
        await stream.ReadAsync(buffer, 0, buffer.Length);
        stream.Seek(0, SeekOrigin.Begin);
        
        return extension switch
        {
            ".jpg" or ".jpeg" => buffer[0] == 0xFF && buffer[1] == 0xD8,
            ".png" => buffer[0] == 0x89 && buffer[1] == 0x50,
            ".gif" => buffer[0] == 0x47 && buffer[1] == 0x49,
            ".pdf" => buffer[0] == 0x25 && buffer[1] == 0x50,
            _ => false
        };
    }
    
    private string GenerateSafeFileName(string fileName)
    {
        var extension = Path.GetExtension(fileName);
        var safeFileName = $"{Guid.NewGuid()}{extension}";
        return safeFileName;
    }
    
    private async Task<bool> ScanForVirus(IFormFile file)
    {
        // 실제 구현에서는 안티바이러스 API 호출
        await Task.Delay(100); // 시뮬레이션
        return true;
    }
}
```

## API Key 및 Secret 관리

### API Key 인증
```csharp
// API Key Authentication Handler
public class ApiKeyAuthenticationHandler : AuthenticationHandler<ApiKeyAuthenticationOptions>
{
    private readonly IApiKeyService _apiKeyService;
    
    public ApiKeyAuthenticationHandler(
        IOptionsMonitor<ApiKeyAuthenticationOptions> options,
        ILoggerFactory logger,
        UrlEncoder encoder,
        ISystemClock clock,
        IApiKeyService apiKeyService) 
        : base(options, logger, encoder, clock)
    {
        _apiKeyService = apiKeyService;
    }
    
    protected override async Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        if (!Request.Headers.TryGetValue(Options.HeaderName, out var apiKeyHeaderValues))
        {
            return AuthenticateResult.Fail("API Key not found in headers");
        }
        
        var apiKey = apiKeyHeaderValues.FirstOrDefault();
        
        if (string.IsNullOrEmpty(apiKey))
        {
            return AuthenticateResult.Fail("API Key not provided");
        }
        
        var validationResult = await _apiKeyService.ValidateApiKeyAsync(apiKey);
        
        if (!validationResult.IsValid)
        {
            return AuthenticateResult.Fail("Invalid API Key");
        }
        
        var claims = new[]
        {
            new Claim(ClaimTypes.NameIdentifier, validationResult.ClientId),
            new Claim("ApiKey", apiKey),
            new Claim("Tier", validationResult.Tier)
        };
        
        var identity = new ClaimsIdentity(claims, Scheme.Name);
        var principal = new ClaimsPrincipal(identity);
        var ticket = new AuthenticationTicket(principal, Scheme.Name);
        
        return AuthenticateResult.Success(ticket);
    }
}

// API Key Service
public interface IApiKeyService
{
    Task<ApiKeyValidationResult> ValidateApiKeyAsync(string apiKey);
    Task<string> GenerateApiKeyAsync(string clientId);
    Task RevokeApiKeyAsync(string apiKey);
}

public class ApiKeyService : IApiKeyService
{
    private readonly ApplicationDbContext _context;
    private readonly IMemoryCache _cache;
    
    public async Task<ApiKeyValidationResult> ValidateApiKeyAsync(string apiKey)
    {
        // 캐시 확인
        if (_cache.TryGetValue($"apikey_{apiKey}", out ApiKeyValidationResult cachedResult))
        {
            return cachedResult;
        }
        
        // 해시된 API Key로 조회
        var hashedKey = HashApiKey(apiKey);
        var apiKeyEntity = await _context.ApiKeys
            .FirstOrDefaultAsync(k => k.HashedKey == hashedKey && k.IsActive);
        
        if (apiKeyEntity == null)
        {
            return new ApiKeyValidationResult { IsValid = false };
        }
        
        // 만료 확인
        if (apiKeyEntity.ExpiresAt < DateTime.UtcNow)
        {
            return new ApiKeyValidationResult { IsValid = false };
        }
        
        var result = new ApiKeyValidationResult
        {
            IsValid = true,
            ClientId = apiKeyEntity.ClientId,
            Tier = apiKeyEntity.Tier
        };
        
        // 캐시에 저장
        _cache.Set($"apikey_{apiKey}", result, TimeSpan.FromMinutes(5));
        
        return result;
    }
    
    public async Task<string> GenerateApiKeyAsync(string clientId)
    {
        var apiKey = GenerateSecureApiKey();
        var hashedKey = HashApiKey(apiKey);
        
        var apiKeyEntity = new ApiKey
        {
            ClientId = clientId,
            HashedKey = hashedKey,
            CreatedAt = DateTime.UtcNow,
            ExpiresAt = DateTime.UtcNow.AddYears(1),
            IsActive = true,
            Tier = "Standard"
        };
        
        _context.ApiKeys.Add(apiKeyEntity);
        await _context.SaveChangesAsync();
        
        return apiKey;
    }
    
    private string GenerateSecureApiKey()
    {
        var randomBytes = new byte[32];
        using var rng = RandomNumberGenerator.Create();
        rng.GetBytes(randomBytes);
        return Convert.ToBase64String(randomBytes);
    }
    
    private string HashApiKey(string apiKey)
    {
        using var sha256 = SHA256.Create();
        var bytes = Encoding.UTF8.GetBytes(apiKey);
        var hash = sha256.ComputeHash(bytes);
        return Convert.ToBase64String(hash);
    }
}
```

### Secret 관리
```csharp
// Azure Key Vault 통합
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .ConfigureAppConfiguration((context, config) =>
        {
            if (context.HostingEnvironment.IsProduction())
            {
                var builtConfig = config.Build();
                var keyVaultEndpoint = builtConfig["KeyVault:Endpoint"];
                
                if (!string.IsNullOrEmpty(keyVaultEndpoint))
                {
                    var credential = new DefaultAzureCredential();
                    config.AddAzureKeyVault(new Uri(keyVaultEndpoint), credential);
                }
            }
        })
        .ConfigureWebHostDefaults(webBuilder =>
        {
            webBuilder.UseStartup<Startup>();
        });

// User Secrets (개발 환경)
// dotnet user-secrets init
// dotnet user-secrets set "ApiKey" "development-key-12345"

// Secret Manager Service
public interface ISecretManager
{
    Task<string> GetSecretAsync(string secretName);
    Task SetSecretAsync(string secretName, string secretValue);
    Task DeleteSecretAsync(string secretName);
}

public class SecretManager : ISecretManager
{
    private readonly IConfiguration _configuration;
    private readonly IHostEnvironment _environment;
    
    public SecretManager(IConfiguration configuration, IHostEnvironment environment)
    {
        _configuration = configuration;
        _environment = environment;
    }
    
    public async Task<string> GetSecretAsync(string secretName)
    {
        // 환경에 따라 다른 소스에서 시크릿 조회
        if (_environment.IsDevelopment())
        {
            // User Secrets 또는 appsettings.Development.json
            return _configuration[secretName];
        }
        else
        {
            // Azure Key Vault 또는 다른 시크릿 관리 서비스
            return _configuration[secretName];
        }
    }
}
```

## 모범 사례

### 종합적인 보안 구성
```csharp
// SecurityConfiguration.cs
public static class SecurityConfiguration
{
    public static void ConfigureSecurity(this IServiceCollection services, IConfiguration configuration)
    {
        // CORS
        services.AddCors(options =>
        {
            options.AddPolicy("SecurePolicy", builder =>
            {
                builder.WithOrigins(configuration.GetSection("Security:AllowedOrigins").Get<string[]>())
                       .AllowAnyHeader()
                       .WithMethods("GET", "POST", "PUT", "DELETE")
                       .AllowCredentials()
                       .SetPreflightMaxAge(TimeSpan.FromHours(1));
            });
        });
        
        // HTTPS
        services.AddHttpsRedirection(options =>
        {
            options.RedirectStatusCode = StatusCodes.Status308PermanentRedirect;
            options.HttpsPort = 443;
        });
        
        // HSTS
        services.AddHsts(options =>
        {
            options.Preload = true;
            options.IncludeSubDomains = true;
            options.MaxAge = TimeSpan.FromDays(365);
        });
        
        // Rate Limiting
        services.AddRateLimiter(options =>
        {
            options.AddPolicy("ApiPolicy", context =>
                RateLimitPartition.GetFixedWindowLimiter(
                    partitionKey: context.Connection.RemoteIpAddress?.ToString(),
                    factory: partition => new FixedWindowRateLimiterOptions
                    {
                        AutoReplenishment = true,
                        PermitLimit = 100,
                        Window = TimeSpan.FromMinutes(1)
                    }));
        });
        
        // 보안 헤더 서비스
        services.AddSingleton<ISecurityHeaderService, SecurityHeaderService>();
    }
    
    public static void UseSecurity(this IApplicationBuilder app, IWebHostEnvironment env)
    {
        if (!env.IsDevelopment())
        {
            app.UseHsts();
        }
        
        app.UseHttpsRedirection();
        app.UseCors("SecurePolicy");
        app.UseRateLimiter();
        app.UseSecurityHeaders();
        
        // CSP Report 엔드포인트
        app.Map("/csp-report", cspApp =>
        {
            cspApp.Run(async context =>
            {
                context.Response.StatusCode = 204; // No Content
                await Task.CompletedTask;
            });
        });
    }
}
```

### 보안 감사 로깅
```csharp
public class SecurityAuditService
{
    private readonly ILogger<SecurityAuditService> _logger;
    
    public void LogSecurityEvent(SecurityEventType eventType, string userId, string details, string ipAddress)
    {
        _logger.LogInformation("Security Event: {EventType} | User: {UserId} | IP: {IpAddress} | Details: {Details}",
            eventType, userId ?? "Anonymous", ipAddress, details);
    }
    
    public void LogFailedAuthentication(string username, string ipAddress)
    {
        _logger.LogWarning("Failed authentication attempt | Username: {Username} | IP: {IpAddress}",
            username, ipAddress);
    }
    
    public void LogRateLimitExceeded(string ipAddress, string endpoint)
    {
        _logger.LogWarning("Rate limit exceeded | IP: {IpAddress} | Endpoint: {Endpoint}",
            ipAddress, endpoint);
    }
}
```

API 보안은 다층 방어 전략이 필요합니다. CORS, HTTPS, 보안 헤더, Rate Limiting 등을 적절히 조합하여 안전한 API를 구축할 수 있습니다.