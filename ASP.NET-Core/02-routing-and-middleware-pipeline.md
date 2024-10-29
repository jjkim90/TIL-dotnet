# 라우팅과 미들웨어 파이프라인

## 미들웨어 개념

미들웨어는 요청과 응답을 처리하는 파이프라인을 구성하는 소프트웨어 컴포넌트입니다. 각 미들웨어는 다음 작업을 수행할 수 있습니다:
- 요청을 처리하고 다음 미들웨어로 전달
- 요청 처리를 중단하고 응답 반환
- 요청과 응답을 변경

### 미들웨어 파이프라인
```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// 미들웨어 파이프라인 구성
app.Use(async (context, next) =>
{
    // 요청 처리 전
    Console.WriteLine($"Request: {context.Request.Path}");
    
    await next(); // 다음 미들웨어 호출
    
    // 응답 처리 후
    Console.WriteLine($"Response: {context.Response.StatusCode}");
});

app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

## 내장 미들웨어

### 주요 내장 미들웨어
```csharp
var app = builder.Build();

// 개발 환경 예외 페이지
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}
else
{
    app.UseExceptionHandler("/Error");
    app.UseHsts(); // HTTPS 강제
}

// 정적 파일 제공
app.UseStaticFiles();

// HTTPS 리다이렉션
app.UseHttpsRedirection();

// 라우팅
app.UseRouting();

// CORS
app.UseCors("MyPolicy");

// 인증 및 권한
app.UseAuthentication();
app.UseAuthorization();

// 엔드포인트 매핑
app.MapControllers();
```

### 미들웨어 순서의 중요성
미들웨어의 등록 순서가 실행 순서를 결정합니다:
1. ExceptionHandler/HSTS
2. HttpsRedirection
3. StaticFiles
4. Routing
5. CORS
6. Authentication
7. Authorization
8. 사용자 정의 미들웨어
9. EndpointRouting

## 사용자 정의 미들웨어

### 인라인 미들웨어
```csharp
app.Use(async (context, next) =>
{
    var stopwatch = Stopwatch.StartNew();
    
    await next();
    
    stopwatch.Stop();
    var elapsed = stopwatch.ElapsedMilliseconds;
    Console.WriteLine($"Request took {elapsed}ms");
});
```

### 클래스 기반 미들웨어
```csharp
public class RequestLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestLoggingMiddleware> _logger;

    public RequestLoggingMiddleware(RequestDelegate next, 
        ILogger<RequestLoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // 요청 로깅
        _logger.LogInformation($"Request: {context.Request.Method} {context.Request.Path}");
        
        try
        {
            await _next(context);
        }
        finally
        {
            // 응답 로깅
            _logger.LogInformation($"Response: {context.Response.StatusCode}");
        }
    }
}

// 확장 메서드로 등록
public static class RequestLoggingMiddlewareExtensions
{
    public static IApplicationBuilder UseRequestLogging(
        this IApplicationBuilder builder)
    {
        return builder.UseMiddleware<RequestLoggingMiddleware>();
    }
}

// Program.cs에서 사용
app.UseRequestLogging();
```

## 라우팅 기초

### 엔드포인트 라우팅
```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// 간단한 라우트
app.MapGet("/", () => "Hello World!");
app.MapGet("/api/users", () => new[] { "User1", "User2" });
app.MapPost("/api/users", (User user) => Results.Created($"/api/users/{user.Id}", user));

// 라우트 매개변수
app.MapGet("/api/users/{id:int}", (int id) => $"User {id}");
app.MapGet("/api/posts/{year:int}/{month:int}", 
    (int year, int month) => $"Posts from {year}/{month}");

app.Run();
```

### 컨트롤러 기반 라우팅
```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    // GET: api/products
    [HttpGet]
    public IActionResult GetAll()
    {
        return Ok(new[] { "Product1", "Product2" });
    }

    // GET: api/products/5
    [HttpGet("{id:int}")]
    public IActionResult GetById(int id)
    {
        return Ok($"Product {id}");
    }

    // GET: api/products/search?name=laptop
    [HttpGet("search")]
    public IActionResult Search([FromQuery] string name)
    {
        return Ok($"Searching for {name}");
    }

    // POST: api/products
    [HttpPost]
    public IActionResult Create([FromBody] Product product)
    {
        return CreatedAtAction(nameof(GetById), 
            new { id = product.Id }, product);
    }
}
```

## 라우트 제약 조건

### 내장 제약 조건
```csharp
// 타입 제약
app.MapGet("/users/{id:int}", (int id) => $"User {id}");
app.MapGet("/posts/{slug:alpha}", (string slug) => $"Post {slug}");

// 길이 제약
app.MapGet("/codes/{code:length(6)}", (string code) => $"Code: {code}");
app.MapGet("/tags/{tag:minlength(3):maxlength(10)}", (string tag) => $"Tag: {tag}");

// 범위 제약
app.MapGet("/pages/{page:range(1,100)}", (int page) => $"Page {page}");

// 정규식 제약
app.MapGet("/products/{sku:regex(^\\d{{3}}-\\d{{3}}$)}", 
    (string sku) => $"SKU: {sku}");
```

### 사용자 정의 제약 조건
```csharp
public class GuidConstraint : IRouteConstraint
{
    public bool Match(HttpContext? httpContext, IRouter? route, 
        string routeKey, RouteValueDictionary values, 
        RouteDirection routeDirection)
    {
        if (values.TryGetValue(routeKey, out var value))
        {
            var stringValue = value?.ToString();
            return Guid.TryParse(stringValue, out _);
        }
        return false;
    }
}

// 등록
builder.Services.Configure<RouteOptions>(options =>
{
    options.ConstraintMap.Add("guid", typeof(GuidConstraint));
});

// 사용
app.MapGet("/items/{id:guid}", (Guid id) => $"Item {id}");
```

## Map, Use, Run 메서드

### Use - 미들웨어 체인
```csharp
app.Use(async (context, next) =>
{
    // 다음 미들웨어 호출 전
    await next();
    // 다음 미들웨어 호출 후
});
```

### Map - 경로 분기
```csharp
app.Map("/api", apiApp =>
{
    apiApp.Use(async (context, next) =>
    {
        context.Response.Headers.Add("X-API-Version", "1.0");
        await next();
    });
    
    apiApp.UseRouting();
    apiApp.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
});

app.Map("/admin", adminApp =>
{
    adminApp.UseAuthentication();
    adminApp.UseAuthorization();
    // 관리자 전용 미들웨어
});
```

### Run - 터미널 미들웨어
```csharp
app.Run(async context =>
{
    await context.Response.WriteAsync("Hello from Run!");
    // 파이프라인 종료 - 다음 미들웨어 없음
});
```

## 조건부 미들웨어

### UseWhen
```csharp
app.UseWhen(context => context.Request.Path.StartsWith("/api"), 
    apiApp =>
    {
        apiApp.Use(async (context, next) =>
        {
            context.Response.Headers.Add("X-API-Call", "true");
            await next();
        });
    });
```

### MapWhen
```csharp
app.MapWhen(context => context.Request.Query.ContainsKey("debug"), 
    debugApp =>
    {
        debugApp.Run(async context =>
        {
            var debugInfo = new
            {
                Path = context.Request.Path,
                Headers = context.Request.Headers,
                Time = DateTime.Now
            };
            await context.Response.WriteAsJsonAsync(debugInfo);
        });
    });
```

## 미들웨어 단락 (Short-circuiting)

```csharp
public class AuthorizationMiddleware
{
    private readonly RequestDelegate _next;

    public AuthorizationMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        if (!context.User.Identity.IsAuthenticated)
        {
            context.Response.StatusCode = 401;
            await context.Response.WriteAsync("Unauthorized");
            return; // 단락 - 다음 미들웨어 호출하지 않음
        }

        await _next(context);
    }
}
```

## 실습 예제: API 키 인증 미들웨어

```csharp
public class ApiKeyMiddleware
{
    private readonly RequestDelegate _next;
    private readonly string _apiKey;

    public ApiKeyMiddleware(RequestDelegate next, IConfiguration config)
    {
        _next = next;
        _apiKey = config["ApiKey"] ?? throw new InvalidOperationException("API Key not configured");
    }

    public async Task InvokeAsync(HttpContext context)
    {
        if (!context.Request.Headers.TryGetValue("X-API-Key", out var extractedApiKey))
        {
            context.Response.StatusCode = 401;
            await context.Response.WriteAsync("API Key missing");
            return;
        }

        if (!_apiKey.Equals(extractedApiKey))
        {
            context.Response.StatusCode = 401;
            await context.Response.WriteAsync("Invalid API Key");
            return;
        }

        await _next(context);
    }
}

// Program.cs
app.UseWhen(context => context.Request.Path.StartsWith("/api/admin"), 
    adminApi =>
    {
        adminApi.UseMiddleware<ApiKeyMiddleware>();
    });
```