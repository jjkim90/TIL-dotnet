# Minimal APIs

## Minimal APIs 개요

Minimal APIs는 ASP.NET Core 6에서 도입된 새로운 API 작성 방식으로, 최소한의 코드로 HTTP API를 구축할 수 있게 해줍니다. 기존의 Controller 기반 API에 비해 더 간결하고 가벼운 구문을 제공합니다.

### Minimal APIs의 특징
- **간결한 구문**: 보일러플레이트 코드 최소화
- **빠른 시작**: 몇 줄의 코드로 API 구축 가능
- **성능 최적화**: 불필요한 오버헤드 제거
- **함수형 프로그래밍 스타일**: 람다 표현식과 로컬 함수 활용
- **완전한 기능**: 라우팅, 의존성 주입, 미들웨어 등 모든 ASP.NET Core 기능 지원

## 기본 구조

### 간단한 Minimal API
```csharp
var builder = WebApplication.CreateBuilder(args);

// 서비스 등록
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// 개발 환경에서 Swagger 활성화
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

// 간단한 엔드포인트
app.MapGet("/", () => "Hello World!");

// JSON 응답
app.MapGet("/api/greeting", () => new { Message = "Hello from Minimal API!", Time = DateTime.Now });

// 라우트 매개변수
app.MapGet("/api/users/{id}", (int id) => new { UserId = id, Name = $"User {id}" });

// 쿼리 문자열
app.MapGet("/api/search", (string? q) => new { Query = q ?? "empty", Results = new[] { "Result 1", "Result 2" } });

app.Run();
```

### HTTP 메서드 매핑
```csharp
// GET
app.MapGet("/api/products", () => 
{
    return Results.Ok(new[] { "Product 1", "Product 2" });
});

// POST
app.MapPost("/api/products", (Product product) => 
{
    // 제품 생성 로직
    return Results.Created($"/api/products/{product.Id}", product);
});

// PUT
app.MapPut("/api/products/{id}", (int id, Product product) => 
{
    product.Id = id;
    // 업데이트 로직
    return Results.NoContent();
});

// DELETE
app.MapDelete("/api/products/{id}", (int id) => 
{
    // 삭제 로직
    return Results.NoContent();
});

// PATCH
app.MapPatch("/api/products/{id}", (int id, JsonPatchDocument<Product> patchDoc) => 
{
    // 부분 업데이트 로직
    return Results.Ok();
});

// 여러 HTTP 메서드 지원
app.MapMethods("/api/products/{id}", new[] { "GET", "HEAD" }, (int id) => 
{
    var product = GetProduct(id);
    return product != null ? Results.Ok(product) : Results.NotFound();
});
```

## 의존성 주입

### 서비스 주입
```csharp
// 서비스 등록
builder.Services.AddScoped<IProductService, ProductService>();
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

var app = builder.Build();

// 서비스 주입을 통한 엔드포인트
app.MapGet("/api/products", async (IProductService productService) => 
{
    var products = await productService.GetAllAsync();
    return Results.Ok(products);
});

// 여러 서비스 주입
app.MapPost("/api/orders", async (
    CreateOrderDto dto, 
    IOrderService orderService, 
    ILogger<Program> logger) => 
{
    try
    {
        var order = await orderService.CreateOrderAsync(dto);
        logger.LogInformation("Order created with ID: {OrderId}", order.Id);
        return Results.Created($"/api/orders/{order.Id}", order);
    }
    catch (Exception ex)
    {
        logger.LogError(ex, "Error creating order");
        return Results.Problem("An error occurred while creating the order");
    }
});

// HttpContext 접근
app.MapGet("/api/user/info", (HttpContext context, IUserService userService) => 
{
    var userId = context.User.FindFirst("UserId")?.Value;
    if (string.IsNullOrEmpty(userId))
    {
        return Results.Unauthorized();
    }
    
    var userInfo = userService.GetUserInfo(userId);
    return Results.Ok(userInfo);
});
```

### 특수 매개변수 바인딩
```csharp
// HttpRequest와 HttpResponse
app.MapPost("/api/upload", async (HttpRequest request, HttpResponse response) => 
{
    if (!request.HasFormContentType)
    {
        response.StatusCode = 400;
        await response.WriteAsync("Invalid content type");
        return;
    }
    
    var form = await request.ReadFormAsync();
    var file = form.Files["file"];
    
    if (file != null)
    {
        // 파일 처리 로직
        await response.WriteAsJsonAsync(new { FileName = file.FileName, Size = file.Length });
    }
});

// CancellationToken
app.MapGet("/api/long-running", async (CancellationToken cancellationToken) => 
{
    await Task.Delay(5000, cancellationToken);
    return Results.Ok("Operation completed");
});

// ClaimsPrincipal
app.MapGet("/api/current-user", (ClaimsPrincipal user) => 
{
    return Results.Ok(new
    {
        IsAuthenticated = user.Identity?.IsAuthenticated ?? false,
        Name = user.Identity?.Name,
        Claims = user.Claims.Select(c => new { c.Type, c.Value })
    });
});
```

## 모델 바인딩과 검증

### 복잡한 모델 바인딩
```csharp
// FromBody (기본값)
app.MapPost("/api/products", async (Product product) => 
{
    // product는 요청 본문에서 자동으로 역직렬화됨
    return Results.Created($"/api/products/{product.Id}", product);
});

// 명시적 바인딩 소스
app.MapPost("/api/products/search", async (
    [FromQuery] string category,
    [FromHeader(Name = "X-API-Key")] string apiKey,
    [FromBody] SearchCriteria criteria) => 
{
    if (string.IsNullOrEmpty(apiKey))
    {
        return Results.Unauthorized();
    }
    
    var results = await SearchProducts(category, criteria);
    return Results.Ok(results);
});

// 사용자 정의 바인딩
app.MapPost("/api/custom", async (HttpContext context) => 
{
    // 사용자 정의 모델 바인딩
    var model = await BindCustomModelAsync(context);
    return Results.Ok(model);
});

async Task<CustomModel> BindCustomModelAsync(HttpContext context)
{
    using var reader = new StreamReader(context.Request.Body);
    var body = await reader.ReadToEndAsync();
    
    // 사용자 정의 파싱 로직
    return ParseCustomModel(body);
}
```

### 유효성 검사
```csharp
// FluentValidation 통합
builder.Services.AddValidatorsFromAssemblyContaining<ProductValidator>();

app.MapPost("/api/products", async (
    Product product, 
    IValidator<Product> validator) => 
{
    var validationResult = await validator.ValidateAsync(product);
    
    if (!validationResult.IsValid)
    {
        return Results.ValidationProblem(validationResult.ToDictionary());
    }
    
    // 제품 생성 로직
    return Results.Created($"/api/products/{product.Id}", product);
});

// 수동 검증
app.MapPost("/api/users", (CreateUserDto dto) => 
{
    var errors = new Dictionary<string, string[]>();
    
    if (string.IsNullOrWhiteSpace(dto.Email))
    {
        errors.Add("Email", new[] { "Email is required" });
    }
    else if (!IsValidEmail(dto.Email))
    {
        errors.Add("Email", new[] { "Invalid email format" });
    }
    
    if (dto.Password?.Length < 8)
    {
        errors.Add("Password", new[] { "Password must be at least 8 characters" });
    }
    
    if (errors.Any())
    {
        return Results.ValidationProblem(errors);
    }
    
    // 사용자 생성 로직
    return Results.Ok();
});

// ValidationProblem 확장
public static class ValidationExtensions
{
    public static Dictionary<string, string[]> ToDictionary(this ValidationResult validationResult)
    {
        return validationResult.Errors
            .GroupBy(x => x.PropertyName)
            .ToDictionary(
                g => g.Key,
                g => g.Select(x => x.ErrorMessage).ToArray()
            );
    }
}
```

## 라우팅과 라우트 그룹

### 라우트 그룹
```csharp
var app = builder.Build();

// 기본 그룹
var api = app.MapGroup("/api");

api.MapGet("/products", () => "All products");
api.MapGet("/products/{id}", (int id) => $"Product {id}");

// 중첩 그룹
var v1 = api.MapGroup("/v1");
var productsV1 = v1.MapGroup("/products");

productsV1.MapGet("/", async (IProductService service) => 
    await service.GetAllAsync());

productsV1.MapGet("/{id}", async (int id, IProductService service) => 
{
    var product = await service.GetByIdAsync(id);
    return product != null ? Results.Ok(product) : Results.NotFound();
});

// 그룹에 미들웨어 적용
var adminApi = app.MapGroup("/api/admin")
    .RequireAuthorization("AdminPolicy")
    .WithOpenApi()
    .WithTags("Admin");

adminApi.MapGet("/users", async (IUserService service) => 
    await service.GetAllUsersAsync());

adminApi.MapDelete("/users/{id}", async (string id, IUserService service) => 
{
    await service.DeleteUserAsync(id);
    return Results.NoContent();
});
```

### 라우트 제약
```csharp
// 타입 제약
app.MapGet("/api/products/{id:int}", (int id) => $"Product ID: {id}");
app.MapGet("/api/products/{slug:alpha}", (string slug) => $"Product Slug: {slug}");

// 범위 제약
app.MapGet("/api/products/{id:int:min(1)}", (int id) => $"Valid Product ID: {id}");
app.MapGet("/api/pages/{page:int:range(1,100)}", (int page) => $"Page: {page}");

// 정규식 제약
app.MapGet("/api/orders/{orderId:regex(^ORD-\\d{{4}}$)}", (string orderId) => 
    $"Order: {orderId}");

// 복합 제약
app.MapGet("/api/archive/{year:int:min(2000)}/{month:int:range(1,12)}", 
    (int year, int month) => $"Archive for {year}/{month}");

// 사용자 정의 제약
public class CustomConstraint : IRouteConstraint
{
    public bool Match(HttpContext? httpContext, IRouter? route, string routeKey, 
        RouteValueDictionary values, RouteDirection routeDirection)
    {
        // 사용자 정의 검증 로직
        return true;
    }
}
```

## 응답 처리

### Results 타입 사용
```csharp
// 다양한 Results 메서드
app.MapGet("/api/results/ok", () => Results.Ok(new { Message = "Success" }));
app.MapGet("/api/results/created", () => Results.Created("/api/resource/123", new { Id = 123 }));
app.MapGet("/api/results/accepted", () => Results.Accepted("/api/status/123"));
app.MapGet("/api/results/nocontent", () => Results.NoContent());
app.MapGet("/api/results/notfound", () => Results.NotFound());
app.MapGet("/api/results/badrequest", () => Results.BadRequest("Invalid input"));
app.MapGet("/api/results/unauthorized", () => Results.Unauthorized());
app.MapGet("/api/results/forbid", () => Results.Forbid());
app.MapGet("/api/results/conflict", () => Results.Conflict("Resource already exists"));

// 파일 응답
app.MapGet("/api/download/{filename}", (string filename) => 
{
    var path = Path.Combine("files", filename);
    if (!File.Exists(path))
    {
        return Results.NotFound();
    }
    
    var fileBytes = File.ReadAllBytes(path);
    return Results.File(fileBytes, "application/octet-stream", filename);
});

// 스트림 응답
app.MapGet("/api/stream", () => 
{
    var stream = new MemoryStream(Encoding.UTF8.GetBytes("Streamed content"));
    return Results.Stream(stream, "text/plain");
});

// 리다이렉트
app.MapGet("/api/redirect", () => Results.Redirect("/api/new-location"));
app.MapGet("/api/redirect-permanent", () => Results.RedirectPermanent("/api/new-location"));

// Problem Details
app.MapGet("/api/problem", () => Results.Problem(
    detail: "An error occurred while processing your request",
    instance: "/api/problem",
    statusCode: 500,
    title: "Internal Server Error",
    type: "https://example.com/errors/internal"));

// 사용자 정의 상태 코드
app.MapGet("/api/custom-status", () => Results.StatusCode(418)); // I'm a teapot
```

### 조건부 응답
```csharp
app.MapGet("/api/products/{id}", async (int id, IProductService service) => 
{
    var product = await service.GetByIdAsync(id);
    
    return product switch
    {
        null => Results.NotFound(new { Error = $"Product {id} not found" }),
        { Stock: 0 } => Results.Ok(new { product, Warning = "Out of stock" }),
        _ => Results.Ok(product)
    };
});

// 다중 반환 타입
app.MapGet("/api/orders/{id}", async Task<Results<Ok<Order>, NotFound, BadRequest<string>>> 
    (int id, IOrderService service) =>
{
    if (id <= 0)
    {
        return TypedResults.BadRequest("Invalid order ID");
    }
    
    var order = await service.GetOrderAsync(id);
    
    return order != null 
        ? TypedResults.Ok(order) 
        : TypedResults.NotFound();
});
```

## 인증과 권한 부여

### 기본 인증
```csharp
// JWT 인증 설정
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]))
        };
    });

builder.Services.AddAuthorization();

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

// 인증이 필요한 엔드포인트
app.MapGet("/api/secure", () => "This is secured!")
    .RequireAuthorization();

// 특정 정책 요구
app.MapGet("/api/admin", () => "Admin only!")
    .RequireAuthorization("AdminPolicy");

// 익명 접근 허용
app.MapGet("/api/public", () => "Public endpoint")
    .AllowAnonymous();
```

### 권한 부여 정책
```csharp
// 정책 정의
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AdminPolicy", policy =>
        policy.RequireRole("Admin"));
    
    options.AddPolicy("MinimumAge", policy =>
        policy.RequireClaim("Age", age => int.Parse(age) >= 18));
    
    options.AddPolicy("PremiumUser", policy =>
        policy.RequireAssertion(context =>
            context.User.HasClaim(c => c.Type == "Subscription" && c.Value == "Premium")));
});

// 정책 적용
var userApi = app.MapGroup("/api/users")
    .RequireAuthorization();

userApi.MapGet("/profile", (ClaimsPrincipal user) => 
    new { Name = user.Identity?.Name, Claims = user.Claims.Select(c => c.Type) });

userApi.MapPost("/premium-feature", () => "Premium feature accessed!")
    .RequireAuthorization("PremiumUser");

// 사용자 정의 권한 부여
app.MapDelete("/api/posts/{id}", async (int id, ClaimsPrincipal user, IPostService service) =>
{
    var post = await service.GetPostAsync(id);
    
    if (post == null)
        return Results.NotFound();
    
    var userId = user.FindFirst("UserId")?.Value;
    if (post.AuthorId != userId && !user.IsInRole("Admin"))
        return Results.Forbid();
    
    await service.DeletePostAsync(id);
    return Results.NoContent();
});
```

## 필터와 미들웨어

### 엔드포인트 필터
```csharp
// 필터 정의
public class ValidationFilter<T> : IEndpointFilter where T : class
{
    private readonly IValidator<T> _validator;
    
    public ValidationFilter(IValidator<T> validator)
    {
        _validator = validator;
    }
    
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context, 
        EndpointFilterDelegate next)
    {
        var argument = context.GetArgument<T>(0);
        var validationResult = await _validator.ValidateAsync(argument);
        
        if (!validationResult.IsValid)
        {
            return Results.ValidationProblem(validationResult.ToDictionary());
        }
        
        return await next(context);
    }
}

// 필터 적용
app.MapPost("/api/products", async (Product product, IProductService service) =>
{
    var created = await service.CreateAsync(product);
    return Results.Created($"/api/products/{created.Id}", created);
})
.AddEndpointFilter<ValidationFilter<Product>>();

// 인라인 필터
app.MapGet("/api/cached", async (context, next) =>
{
    var cacheKey = "api_cached_data";
    var cache = context.RequestServices.GetRequiredService<IMemoryCache>();
    
    if (cache.TryGetValue(cacheKey, out var cachedData))
    {
        return Results.Ok(cachedData);
    }
    
    var result = await next(context);
    
    if (result is IStatusCodeHttpResult { StatusCode: 200 } okResult)
    {
        cache.Set(cacheKey, okResult, TimeSpan.FromMinutes(5));
    }
    
    return result;
})
.AddEndpointFilter(async (context, next) =>
{
    var stopwatch = Stopwatch.StartNew();
    var result = await next(context);
    stopwatch.Stop();
    
    var logger = context.HttpContext.RequestServices.GetRequiredService<ILogger<Program>>();
    logger.LogInformation("Endpoint executed in {ElapsedMilliseconds}ms", 
        stopwatch.ElapsedMilliseconds);
    
    return result;
});
```

### 라우트별 미들웨어
```csharp
// 특정 라우트에만 미들웨어 적용
app.Map("/api/v2", apiApp =>
{
    apiApp.UseRateLimiter();
    apiApp.UseAuthentication();
    apiApp.UseAuthorization();
    
    apiApp.MapGet("/products", () => "V2 Products");
});

// 조건부 미들웨어
app.MapWhen(context => context.Request.Path.StartsWithSegments("/api/admin"), adminApp =>
{
    adminApp.Use(async (context, next) =>
    {
        if (!context.User.IsInRole("Admin"))
        {
            context.Response.StatusCode = 403;
            await context.Response.WriteAsync("Forbidden");
            return;
        }
        
        await next();
    });
    
    adminApp.MapGet("/users", () => "Admin users endpoint");
});
```

## OpenAPI (Swagger) 통합

### 기본 OpenAPI 설정
```csharp
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "My Minimal API",
        Version = "v1",
        Description = "A simple example of Minimal APIs",
        Contact = new OpenApiContact
        {
            Name = "API Support",
            Email = "support@example.com"
        }
    });
    
    // JWT 인증 설정
    options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Description = "JWT Authorization header using the Bearer scheme",
        Type = SecuritySchemeType.Http,
        Scheme = "bearer"
    });
    
    options.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference
                {
                    Type = ReferenceType.SecurityScheme,
                    Id = "Bearer"
                }
            },
            new string[] { }
        }
    });
});

var app = builder.Build();

app.UseSwagger();
app.UseSwaggerUI();

// OpenAPI 메타데이터 추가
app.MapGet("/api/products", () => new[] { "Product 1", "Product 2" })
    .WithName("GetProducts")
    .WithOpenApi(operation => new(operation)
    {
        Tags = new List<OpenApiTag> { new() { Name = "Products" } },
        Summary = "Gets all products",
        Description = "Returns a list of all available products"
    });

app.MapPost("/api/products", (Product product) => Results.Created($"/api/products/{product.Id}", product))
    .WithName("CreateProduct")
    .WithOpenApi()
    .Produces<Product>(StatusCodes.Status201Created)
    .ProducesProblem(StatusCodes.Status400BadRequest);

// 응답 예제 추가
app.MapGet("/api/products/{id}", (int id) => new Product { Id = id, Name = $"Product {id}" })
    .WithName("GetProductById")
    .WithOpenApi(operation => 
    {
        operation.Parameters[0].Description = "The ID of the product to retrieve";
        return operation;
    })
    .Produces<Product>(StatusCodes.Status200OK)
    .Produces(StatusCodes.Status404NotFound);
```

### 고급 OpenAPI 구성
```csharp
// 사용자 정의 스키마 필터
public class EnumSchemaFilter : ISchemaFilter
{
    public void Apply(OpenApiSchema schema, SchemaFilterContext context)
    {
        if (context.Type.IsEnum)
        {
            schema.Enum.Clear();
            Enum.GetNames(context.Type)
                .ToList()
                .ForEach(name => schema.Enum.Add(new OpenApiString(name)));
        }
    }
}

builder.Services.AddSwaggerGen(options =>
{
    options.SchemaFilter<EnumSchemaFilter>();
    
    // XML 주석 포함
    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
    options.IncludeXmlComments(xmlPath);
});

// 엔드포인트별 설명
app.MapPost("/api/orders", 
    [SwaggerOperation(
        Summary = "Creates a new order",
        Description = "Creates a new order with the provided items")]
    [SwaggerResponse(201, "The order was created", typeof(Order))]
    [SwaggerResponse(400, "The order data is invalid")]
    async (CreateOrderDto dto, IOrderService service) =>
    {
        var order = await service.CreateOrderAsync(dto);
        return Results.Created($"/api/orders/{order.Id}", order);
    });
```

## 테스트

### Minimal API 테스트
```csharp
public class MinimalApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;
    
    public MinimalApiTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory;
    }
    
    [Fact]
    public async Task GetProducts_ReturnsSuccessAndCorrectContentType()
    {
        // Arrange
        var client = _factory.CreateClient();
        
        // Act
        var response = await client.GetAsync("/api/products");
        
        // Assert
        response.EnsureSuccessStatusCode();
        Assert.Equal("application/json; charset=utf-8", 
            response.Content.Headers.ContentType?.ToString());
    }
    
    [Fact]
    public async Task CreateProduct_WithValidData_ReturnsCreated()
    {
        // Arrange
        var client = _factory.CreateClient();
        var product = new Product { Name = "Test Product", Price = 99.99m };
        
        // Act
        var response = await client.PostAsJsonAsync("/api/products", product);
        
        // Assert
        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
        Assert.NotNull(response.Headers.Location);
    }
}

// 통합 테스트
public class IntegrationTests
{
    [Fact]
    public async Task MinimalApi_WithDependencyInjection_Works()
    {
        var builder = WebApplication.CreateBuilder();
        
        // 테스트용 서비스 등록
        builder.Services.AddSingleton<IProductService, MockProductService>();
        
        await using var app = builder.Build();
        
        app.MapGet("/api/test", (IProductService service) => service.GetAllAsync());
        
        await app.StartAsync();
        
        using var client = new HttpClient { BaseAddress = new Uri("http://localhost:5000") };
        var response = await client.GetAsync("/api/test");
        
        response.EnsureSuccessStatusCode();
    }
}
```

## 고급 시나리오

### 백그라운드 작업
```csharp
app.MapPost("/api/import", async (
    IFormFile file, 
    IBackgroundTaskQueue taskQueue,
    ILogger<Program> logger) =>
{
    var fileId = Guid.NewGuid();
    
    // 파일을 임시 위치에 저장
    var tempPath = Path.GetTempFileName();
    using (var stream = File.Create(tempPath))
    {
        await file.CopyToAsync(stream);
    }
    
    // 백그라운드 작업 큐에 추가
    await taskQueue.QueueBackgroundWorkItemAsync(async token =>
    {
        try
        {
            await ProcessFileAsync(tempPath, fileId, token);
            logger.LogInformation("File {FileId} processed successfully", fileId);
        }
        finally
        {
            File.Delete(tempPath);
        }
    });
    
    return Results.Accepted(new { FileId = fileId, Status = "Processing" });
});

// 처리 상태 확인
app.MapGet("/api/import/{fileId}/status", async (
    Guid fileId,
    IFileProcessingService service) =>
{
    var status = await service.GetProcessingStatusAsync(fileId);
    
    return status != null 
        ? Results.Ok(status) 
        : Results.NotFound();
});
```

### 실시간 통신
```csharp
// SignalR Hub
public class NotificationHub : Hub
{
    public async Task SendMessage(string message)
    {
        await Clients.All.SendAsync("ReceiveMessage", message);
    }
}

builder.Services.AddSignalR();

var app = builder.Build();

app.MapHub<NotificationHub>("/notifications");

// REST API와 SignalR 통합
app.MapPost("/api/notifications", async (
    string message,
    IHubContext<NotificationHub> hubContext) =>
{
    await hubContext.Clients.All.SendAsync("ReceiveMessage", message);
    return Results.Ok();
});
```

### 사용자 정의 결과 타입
```csharp
public class CsvResult : IResult
{
    private readonly string _csv;
    private readonly string _fileName;
    
    public CsvResult(string csv, string fileName)
    {
        _csv = csv;
        _fileName = fileName;
    }
    
    public async Task ExecuteAsync(HttpContext httpContext)
    {
        httpContext.Response.ContentType = "text/csv";
        httpContext.Response.Headers.Add("Content-Disposition", 
            $"attachment; filename=\"{_fileName}\"");
        
        await httpContext.Response.WriteAsync(_csv);
    }
}

// 사용
app.MapGet("/api/export/csv", async (IDataService service) =>
{
    var data = await service.GetExportDataAsync();
    var csv = ConvertToCsv(data);
    
    return new CsvResult(csv, "export.csv");
});

// Results 확장 메서드
public static class ResultsExtensions
{
    public static IResult Csv(this IResultExtensions _, string csv, string fileName)
    {
        return new CsvResult(csv, fileName);
    }
}

// 사용
app.MapGet("/api/export/csv2", () => Results.Extensions.Csv("id,name\n1,test", "data.csv"));
```

## 성능 최적화

### 응답 압축
```csharp
builder.Services.AddResponseCompression(options =>
{
    options.EnableForHttps = true;
    options.Providers.Add<BrotliCompressionProvider>();
    options.Providers.Add<GzipCompressionProvider>();
});

var app = builder.Build();

app.UseResponseCompression();

// 특정 엔드포인트에만 압축 적용
app.MapGet("/api/large-data", () => GenerateLargeData())
    .CacheOutput(options => options.Expire(TimeSpan.FromMinutes(10)));
```

### 출력 캐싱
```csharp
builder.Services.AddOutputCache(options =>
{
    options.AddBasePolicy(builder => builder.Expire(TimeSpan.FromMinutes(5)));
    
    options.AddPolicy("Vary", builder => 
        builder.Expire(TimeSpan.FromMinutes(10))
               .SetVaryByQuery("page", "size"));
});

var app = builder.Build();

app.UseOutputCache();

// 캐싱 적용
app.MapGet("/api/products", () => GetProducts())
    .CacheOutput();

app.MapGet("/api/products/search", (string? q) => SearchProducts(q))
    .CacheOutput("Vary");
```

## 모범 사례

### 구조화된 Minimal API
```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// 서비스 구성
builder.Services.AddServices(builder.Configuration);

var app = builder.Build();

// 미들웨어 구성
app.ConfigureMiddleware();

// 엔드포인트 매핑
app.MapEndpoints();

app.Run();

// 확장 메서드들
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddServices(
        this IServiceCollection services, 
        IConfiguration configuration)
    {
        services.AddEndpointsApiExplorer();
        services.AddSwaggerGen();
        
        services.AddDbContext<ApplicationDbContext>(options =>
            options.UseSqlServer(configuration.GetConnectionString("DefaultConnection")));
        
        services.AddScoped<IProductService, ProductService>();
        services.AddScoped<IOrderService, OrderService>();
        
        return services;
    }
}

public static class WebApplicationExtensions
{
    public static WebApplication ConfigureMiddleware(this WebApplication app)
    {
        if (app.Environment.IsDevelopment())
        {
            app.UseSwagger();
            app.UseSwaggerUI();
        }
        
        app.UseAuthentication();
        app.UseAuthorization();
        
        return app;
    }
    
    public static WebApplication MapEndpoints(this WebApplication app)
    {
        app.MapProductEndpoints();
        app.MapOrderEndpoints();
        app.MapAuthEndpoints();
        
        return app;
    }
}

// 엔드포인트 모듈
public static class ProductEndpoints
{
    public static WebApplication MapProductEndpoints(this WebApplication app)
    {
        var group = app.MapGroup("/api/products")
            .WithTags("Products")
            .WithOpenApi();
        
        group.MapGet("/", GetAllProducts);
        group.MapGet("/{id}", GetProductById);
        group.MapPost("/", CreateProduct).RequireAuthorization();
        group.MapPut("/{id}", UpdateProduct).RequireAuthorization();
        group.MapDelete("/{id}", DeleteProduct).RequireAuthorization("Admin");
        
        return app;
    }
    
    private static async Task<IResult> GetAllProducts(IProductService service)
    {
        var products = await service.GetAllAsync();
        return Results.Ok(products);
    }
    
    private static async Task<IResult> GetProductById(int id, IProductService service)
    {
        var product = await service.GetByIdAsync(id);
        return product != null ? Results.Ok(product) : Results.NotFound();
    }
    
    private static async Task<IResult> CreateProduct(
        Product product, 
        IProductService service,
        IValidator<Product> validator)
    {
        var validationResult = await validator.ValidateAsync(product);
        if (!validationResult.IsValid)
        {
            return Results.ValidationProblem(validationResult.ToDictionary());
        }
        
        var created = await service.CreateAsync(product);
        return Results.Created($"/api/products/{created.Id}", created);
    }
    
    private static async Task<IResult> UpdateProduct(
        int id, 
        Product product, 
        IProductService service)
    {
        product.Id = id;
        await service.UpdateAsync(product);
        return Results.NoContent();
    }
    
    private static async Task<IResult> DeleteProduct(int id, IProductService service)
    {
        await service.DeleteAsync(id);
        return Results.NoContent();
    }
}
```

Minimal APIs는 간결하고 성능이 뛰어난 API를 구축할 수 있는 현대적인 접근 방식입니다. 적절한 구조화와 모범 사례를 따르면 대규모 애플리케이션에서도 효과적으로 사용할 수 있습니다.