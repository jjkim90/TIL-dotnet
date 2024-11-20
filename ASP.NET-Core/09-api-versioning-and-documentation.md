# API 버저닝과 문서화 (Swagger)

## API 버저닝의 필요성

API는 시간이 지남에 따라 변경되지만, 기존 클라이언트를 손상시키지 않아야 합니다. API 버저닝은 이러한 문제를 해결합니다.

### 버저닝 전략
- **URI 버저닝**: `/api/v1/products`
- **쿼리 문자열 버저닝**: `/api/products?api-version=1.0`
- **헤더 버저닝**: `X-API-Version: 1.0`
- **미디어 타입 버저닝**: `Accept: application/vnd.company.v1+json`

## ASP.NET Core API 버저닝 설정

### 패키지 설치 및 기본 구성
```csharp
// NuGet: Asp.Versioning.Mvc
// NuGet: Asp.Versioning.Mvc.ApiExplorer

// Program.cs
var builder = WebApplication.CreateBuilder(args);

// API 버저닝 추가
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
    
    // 여러 버전 리더 결합
    options.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),
        new QueryStringApiVersionReader("api-version"),
        new HeaderApiVersionReader("X-API-Version"),
        new MediaTypeApiVersionReader("version")
    );
})
.AddApiExplorer(options =>
{
    // 버전 형식: 'v'major[.minor][-status]
    options.GroupNameFormat = "'v'VVV";
    options.SubstituteApiVersionInUrl = true;
});

var app = builder.Build();
```

## URL 경로 버저닝

### 기본 URL 버저닝
```csharp
[ApiController]
[Route("api/v{version:apiVersion}/[controller]")]
[ApiVersion("1.0")]
[ApiVersion("2.0")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _productService;
    
    public ProductsController(IProductService productService)
    {
        _productService = productService;
    }
    
    // GET: api/v1/products
    [HttpGet]
    [MapToApiVersion("1.0")]
    public async Task<ActionResult<IEnumerable<ProductV1Dto>>> GetV1()
    {
        var products = await _productService.GetAllAsync();
        return Ok(products.Select(p => new ProductV1Dto
        {
            Id = p.Id,
            Name = p.Name,
            Price = p.Price
        }));
    }
    
    // GET: api/v2/products
    [HttpGet]
    [MapToApiVersion("2.0")]
    public async Task<ActionResult<IEnumerable<ProductV2Dto>>> GetV2()
    {
        var products = await _productService.GetAllAsync();
        return Ok(products.Select(p => new ProductV2Dto
        {
            Id = p.Id,
            Name = p.Name,
            Price = p.Price,
            Description = p.Description,
            Category = p.Category,
            Tags = p.Tags
        }));
    }
}

// V1 DTO
public class ProductV1Dto
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
}

// V2 DTO - 확장된 정보
public class ProductV2Dto : ProductV1Dto
{
    public string Description { get; set; }
    public string Category { get; set; }
    public List<string> Tags { get; set; }
}
```

### 별도 컨트롤러로 버전 분리
```csharp
// V1 컨트롤러
[ApiController]
[Route("api/v{version:apiVersion}/products")]
[ApiVersion("1.0", Deprecated = true)] // 더 이상 사용되지 않음 표시
public class ProductsV1Controller : ControllerBase
{
    [HttpGet]
    public ActionResult<IEnumerable<ProductV1Dto>> Get()
    {
        // V1 구현
        return Ok(new[] { new ProductV1Dto { Id = 1, Name = "Product 1", Price = 100 } });
    }
}

// V2 컨트롤러
[ApiController]
[Route("api/v{version:apiVersion}/products")]
[ApiVersion("2.0")]
public class ProductsV2Controller : ControllerBase
{
    [HttpGet]
    public ActionResult<IEnumerable<ProductV2Dto>> Get()
    {
        // V2 구현
        return Ok(new[] 
        { 
            new ProductV2Dto 
            { 
                Id = 1, 
                Name = "Product 1", 
                Price = 100,
                Description = "Description",
                Category = "Electronics"
            } 
        });
    }
}

// V3 컨트롤러 - 미리보기 버전
[ApiController]
[Route("api/v{version:apiVersion}/products")]
[ApiVersion("3.0-preview")]
public class ProductsV3Controller : ControllerBase
{
    [HttpGet]
    public ActionResult<IEnumerable<ProductV3Dto>> Get()
    {
        // V3 구현 (미리보기)
        return Ok(Array.Empty<ProductV3Dto>());
    }
}
```

## 쿼리 문자열 및 헤더 버저닝

### 쿼리 문자열 버저닝
```csharp
// Program.cs
builder.Services.AddApiVersioning(options =>
{
    options.ApiVersionReader = new QueryStringApiVersionReader("api-version");
});

[ApiController]
[Route("api/[controller]")]
[ApiVersion("1.0")]
[ApiVersion("2.0")]
public class OrdersController : ControllerBase
{
    // GET: api/orders?api-version=1.0
    [HttpGet]
    [MapToApiVersion("1.0")]
    public ActionResult<IEnumerable<OrderV1Dto>> GetV1()
    {
        return Ok(new[] { new OrderV1Dto() });
    }
    
    // GET: api/orders?api-version=2.0
    [HttpGet]
    [MapToApiVersion("2.0")]
    public ActionResult<IEnumerable<OrderV2Dto>> GetV2()
    {
        return Ok(new[] { new OrderV2Dto() });
    }
}
```

### 헤더 버저닝
```csharp
// Program.cs
builder.Services.AddApiVersioning(options =>
{
    options.ApiVersionReader = new HeaderApiVersionReader("X-API-Version");
});

[ApiController]
[Route("api/[controller]")]
public class CustomersController : ControllerBase
{
    // GET: api/customers
    // Header: X-API-Version: 1.0
    [HttpGet]
    [ApiVersion("1.0")]
    public ActionResult Get()
    {
        return Ok(new { version = "1.0" });
    }
    
    // GET: api/customers
    // Header: X-API-Version: 2.0
    [HttpGet]
    [ApiVersion("2.0")]
    public ActionResult GetV2()
    {
        return Ok(new { version = "2.0", enhanced = true });
    }
}
```

## Swagger/OpenAPI 통합

### Swagger 설정
```csharp
// NuGet: Swashbuckle.AspNetCore

// Program.cs
builder.Services.AddEndpointsApiExplorer();

// 버전별 Swagger 문서 생성
builder.Services.AddSwaggerGen(options =>
{
    // API 정보
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "My API",
        Version = "v1",
        Description = "My API version 1",
        Contact = new OpenApiContact
        {
            Name = "Support Team",
            Email = "support@example.com",
            Url = new Uri("https://example.com")
        },
        License = new OpenApiLicense
        {
            Name = "MIT",
            Url = new Uri("https://opensource.org/licenses/MIT")
        }
    });
    
    options.SwaggerDoc("v2", new OpenApiInfo
    {
        Title = "My API",
        Version = "v2",
        Description = "My API version 2 with enhanced features"
    });
    
    // XML 주석 포함
    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
    options.IncludeXmlComments(xmlPath);
    
    // 버전 필터
    options.OperationFilter<ApiVersionOperationFilter>();
    
    // 보안 정의
    options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Description = "JWT Authorization header using the Bearer scheme",
        Name = "Authorization",
        In = ParameterLocation.Header,
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
            Array.Empty<string>()
        }
    });
});

var app = builder.Build();

// Swagger UI 활성화
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI(options =>
    {
        options.SwaggerEndpoint("/swagger/v1/swagger.json", "My API V1");
        options.SwaggerEndpoint("/swagger/v2/swagger.json", "My API V2");
        
        // 추가 UI 옵션
        options.DocExpansion(DocExpansion.None);
        options.DefaultModelsExpandDepth(-1);
        options.DisplayRequestDuration();
    });
}
```

### API 버전 필터
```csharp
public class ApiVersionOperationFilter : IOperationFilter
{
    public void Apply(OpenApiOperation operation, OperationFilterContext context)
    {
        var apiVersion = context.ApiDescription
            .GetApiVersion()
            .ToString();
        
        operation.Parameters ??= new List<OpenApiParameter>();
        
        // URL 경로에 버전이 있는 경우 제거
        var versionParameter = operation.Parameters
            .FirstOrDefault(p => p.Name == "version");
        
        if (versionParameter != null)
        {
            operation.Parameters.Remove(versionParameter);
        }
    }
}
```

## XML 주석을 통한 API 문서화

### 프로젝트 설정
```xml
<!-- .csproj 파일 -->
<PropertyGroup>
  <GenerateDocumentationFile>true</GenerateDocumentationFile>
  <NoWarn>$(NoWarn);1591</NoWarn>
</PropertyGroup>
```

### XML 주석 사용
```csharp
/// <summary>
/// 제품 관리 API
/// </summary>
[ApiController]
[Route("api/v{version:apiVersion}/[controller]")]
[ApiVersion("1.0")]
[ApiVersion("2.0")]
public class ProductsController : ControllerBase
{
    /// <summary>
    /// 모든 제품을 조회합니다
    /// </summary>
    /// <returns>제품 목록</returns>
    /// <response code="200">제품 목록 반환</response>
    /// <response code="500">서버 오류</response>
    [HttpGet]
    [ProducesResponseType(typeof(IEnumerable<ProductDto>), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status500InternalServerError)]
    public async Task<ActionResult<IEnumerable<ProductDto>>> GetProducts()
    {
        var products = await _productService.GetAllAsync();
        return Ok(products);
    }
    
    /// <summary>
    /// 특정 제품을 조회합니다
    /// </summary>
    /// <param name="id">제품 ID</param>
    /// <returns>제품 정보</returns>
    /// <remarks>
    /// 샘플 요청:
    /// 
    ///     GET /api/v1/products/123
    /// 
    /// </remarks>
    /// <response code="200">제품 반환</response>
    /// <response code="404">제품을 찾을 수 없음</response>
    [HttpGet("{id}")]
    [ProducesResponseType(typeof(ProductDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<ProductDto>> GetProduct(int id)
    {
        var product = await _productService.GetByIdAsync(id);
        return product != null ? Ok(product) : NotFound();
    }
    
    /// <summary>
    /// 새 제품을 생성합니다
    /// </summary>
    /// <param name="product">제품 정보</param>
    /// <returns>생성된 제품</returns>
    /// <response code="201">제품이 성공적으로 생성됨</response>
    /// <response code="400">잘못된 요청</response>
    [HttpPost]
    [ProducesResponseType(typeof(ProductDto), StatusCodes.Status201Created)]
    [ProducesResponseType(typeof(ValidationProblemDetails), StatusCodes.Status400BadRequest)]
    public async Task<ActionResult<ProductDto>> CreateProduct(
        [FromBody] CreateProductDto product)
    {
        var created = await _productService.CreateAsync(product);
        return CreatedAtAction(nameof(GetProduct), new { id = created.Id }, created);
    }
}

/// <summary>
/// 제품 DTO
/// </summary>
public class ProductDto
{
    /// <summary>
    /// 제품 ID
    /// </summary>
    /// <example>123</example>
    public int Id { get; set; }
    
    /// <summary>
    /// 제품명
    /// </summary>
    /// <example>노트북</example>
    [Required]
    public string Name { get; set; }
    
    /// <summary>
    /// 가격 (원)
    /// </summary>
    /// <example>1500000</example>
    [Range(0, double.MaxValue)]
    public decimal Price { get; set; }
    
    /// <summary>
    /// 재고 수량
    /// </summary>
    /// <example>50</example>
    [Range(0, int.MaxValue)]
    public int Stock { get; set; }
}
```

## Swagger 고급 구성

### 사용자 정의 스키마 필터
```csharp
public class EnumSchemaFilter : ISchemaFilter
{
    public void Apply(OpenApiSchema schema, SchemaFilterContext context)
    {
        if (context.Type.IsEnum)
        {
            schema.Enum.Clear();
            var enumNames = Enum.GetNames(context.Type);
            
            foreach (var enumName in enumNames)
            {
                var enumMemberValue = Enum.Parse(context.Type, enumName);
                schema.Enum.Add(new OpenApiString($"{enumName} = {(int)enumMemberValue}"));
            }
        }
    }
}

// Program.cs에서 등록
builder.Services.AddSwaggerGen(options =>
{
    options.SchemaFilter<EnumSchemaFilter>();
});
```

### 파일 업로드 지원
```csharp
public class FileUploadOperationFilter : IOperationFilter
{
    public void Apply(OpenApiOperation operation, OperationFilterContext context)
    {
        var fileParameters = context.MethodInfo
            .GetParameters()
            .Where(p => p.ParameterType == typeof(IFormFile) ||
                       p.ParameterType == typeof(IEnumerable<IFormFile>))
            .ToList();
        
        if (!fileParameters.Any())
            return;
        
        operation.RequestBody = new OpenApiRequestBody
        {
            Content = new Dictionary<string, OpenApiMediaType>
            {
                ["multipart/form-data"] = new OpenApiMediaType
                {
                    Schema = new OpenApiSchema
                    {
                        Type = "object",
                        Properties = fileParameters.ToDictionary(
                            p => p.Name,
                            p => new OpenApiSchema
                            {
                                Type = "string",
                                Format = "binary"
                            })
                    }
                }
            }
        };
    }
}

[ApiController]
[Route("api/[controller]")]
public class FilesController : ControllerBase
{
    /// <summary>
    /// 파일을 업로드합니다
    /// </summary>
    /// <param name="file">업로드할 파일</param>
    /// <returns>업로드 결과</returns>
    [HttpPost("upload")]
    [ProducesResponseType(typeof(FileUploadResult), StatusCodes.Status200OK)]
    public async Task<ActionResult<FileUploadResult>> Upload(IFormFile file)
    {
        // 파일 처리 로직
        return Ok(new FileUploadResult { FileName = file.FileName });
    }
}
```

## API 버전 협상

### 버전 선택기 구현
```csharp
public class CustomApiVersionSelector : IApiVersionSelector
{
    public ApiVersion SelectVersion(
        HttpRequest request,
        ApiVersionModel model)
    {
        // 클라이언트가 지원하는 버전 확인
        var requestedVersion = request.GetRequestedApiVersion();
        
        if (requestedVersion != null && model.SupportedApiVersions.Contains(requestedVersion))
        {
            return requestedVersion;
        }
        
        // 지원되지 않는 버전 요청 시 가장 높은 안정 버전 반환
        return model.SupportedApiVersions
            .Where(v => !v.Status.HasValue) // 프리뷰가 아닌 버전
            .OrderByDescending(v => v)
            .FirstOrDefault() ?? model.DefaultApiVersion;
    }
}

// Program.cs
builder.Services.AddApiVersioning(options =>
{
    options.ApiVersionSelector = new CustomApiVersionSelector();
});
```

### 버전별 정책 적용
```csharp
[ApiController]
[Route("api/v{version:apiVersion}/[controller]")]
public class PaymentController : ControllerBase
{
    // V1: 기본 결제
    [HttpPost]
    [ApiVersion("1.0")]
    [Obsolete("Use version 2.0 for enhanced security")]
    public ActionResult ProcessPaymentV1(PaymentRequestV1 request)
    {
        // 간단한 결제 처리
        return Ok(new { success = true });
    }
    
    // V2: 강화된 보안
    [HttpPost]
    [ApiVersion("2.0")]
    [Authorize]
    public ActionResult ProcessPaymentV2(PaymentRequestV2 request)
    {
        // 2FA 검증 포함
        if (!request.TwoFactorCode.IsValid())
        {
            return BadRequest("Invalid 2FA code");
        }
        
        // 강화된 결제 처리
        return Ok(new { success = true, transactionId = Guid.NewGuid() });
    }
}
```

## 실시간 API 문서화

### ReDoc 통합
```csharp
// Program.cs
app.UseReDoc(options =>
{
    options.DocumentTitle = "My API Documentation";
    options.SpecUrl = "/swagger/v1/swagger.json";
    options.RoutePrefix = "api-docs";
});
```

### API 버전 정보 엔드포인트
```csharp
[ApiController]
[Route("api")]
[ApiVersionNeutral] // 버전 중립적 엔드포인트
public class ApiInfoController : ControllerBase
{
    private readonly IApiVersionDescriptionProvider _provider;
    
    public ApiInfoController(IApiVersionDescriptionProvider provider)
    {
        _provider = provider;
    }
    
    /// <summary>
    /// API 버전 정보를 반환합니다
    /// </summary>
    [HttpGet("versions")]
    public ActionResult GetVersions()
    {
        var versions = _provider.ApiVersionDescriptions
            .Select(desc => new
            {
                Version = desc.ApiVersion.ToString(),
                IsDeprecated = desc.IsDeprecated,
                SunsetPolicy = desc.SunsetPolicy,
                GroupName = desc.GroupName
            });
        
        return Ok(new
        {
            CurrentVersion = "2.0",
            SupportedVersions = versions,
            DeprecationPolicy = "Versions are supported for 12 months after deprecation"
        });
    }
}
```

## 실전 예제: 완전한 API 버저닝과 문서화

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// API 버저닝 구성
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
    options.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),
        new QueryStringApiVersionReader("api-version"),
        new HeaderApiVersionReader("X-API-Version")
    );
})
.AddMvc()
.AddApiExplorer(options =>
{
    options.GroupNameFormat = "'v'VVV";
    options.SubstituteApiVersionInUrl = true;
});

// Swagger 구성
builder.Services.AddTransient<IConfigureOptions<SwaggerGenOptions>, ConfigureSwaggerOptions>();
builder.Services.AddSwaggerGen(options =>
{
    options.OperationFilter<SwaggerDefaultValues>();
    
    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
    options.IncludeXmlComments(xmlPath);
});

var app = builder.Build();

// API 탐색기에서 버전 정보 가져오기
var apiVersionDescriptionProvider = app.Services.GetRequiredService<IApiVersionDescriptionProvider>();

// Swagger 미들웨어
app.UseSwagger();
app.UseSwaggerUI(options =>
{
    foreach (var description in apiVersionDescriptionProvider.ApiVersionDescriptions)
    {
        options.SwaggerEndpoint(
            $"/swagger/{description.GroupName}/swagger.json",
            description.GroupName.ToUpperInvariant());
    }
});

// Swagger 옵션 구성 클래스
public class ConfigureSwaggerOptions : IConfigureOptions<SwaggerGenOptions>
{
    private readonly IApiVersionDescriptionProvider _provider;
    
    public ConfigureSwaggerOptions(IApiVersionDescriptionProvider provider)
    {
        _provider = provider;
    }
    
    public void Configure(SwaggerGenOptions options)
    {
        foreach (var description in _provider.ApiVersionDescriptions)
        {
            options.SwaggerDoc(description.GroupName, CreateInfoForApiVersion(description));
        }
    }
    
    private static OpenApiInfo CreateInfoForApiVersion(ApiVersionDescription description)
    {
        var info = new OpenApiInfo
        {
            Title = "Sample API",
            Version = description.ApiVersion.ToString(),
            Description = "A sample API with versioning and Swagger documentation",
            Contact = new OpenApiContact
            {
                Name = "Development Team",
                Email = "dev@example.com"
            }
        };
        
        if (description.IsDeprecated)
        {
            info.Description += " This API version has been deprecated.";
        }
        
        return info;
    }
}

// Swagger 기본값 필터
public class SwaggerDefaultValues : IOperationFilter
{
    public void Apply(OpenApiOperation operation, OperationFilterContext context)
    {
        var apiDescription = context.ApiDescription;
        
        operation.Deprecated |= apiDescription.IsDeprecated();
        
        foreach (var responseType in context.ApiDescription.SupportedResponseTypes)
        {
            var responseKey = responseType.IsDefaultResponse 
                ? "default" 
                : responseType.StatusCode.ToString();
            var response = operation.Responses[responseKey];
            
            foreach (var contentType in response.Content.Keys)
            {
                if (!responseType.ApiResponseFormats.Any(x => x.MediaType == contentType))
                {
                    response.Content.Remove(contentType);
                }
            }
        }
        
        if (operation.Parameters == null)
        {
            return;
        }
        
        foreach (var parameter in operation.Parameters)
        {
            var description = apiDescription.ParameterDescriptions
                .First(p => p.Name == parameter.Name);
            
            parameter.Description ??= description.ModelMetadata?.Description;
            
            if (parameter.Schema.Default == null && 
                description.DefaultValue != null)
            {
                parameter.Schema.Default = new OpenApiString(
                    description.DefaultValue.ToString());
            }
            
            parameter.Required |= description.IsRequired;
        }
    }
}

// 버전별 컨트롤러 예제
/// <summary>
/// 사용자 관리 API
/// </summary>
[ApiController]
[Route("api/v{version:apiVersion}/users")]
public class UsersController : ControllerBase
{
    /// <summary>
    /// 사용자 목록 조회 (V1)
    /// </summary>
    [HttpGet]
    [ApiVersion("1.0")]
    [ProducesResponseType(typeof(IEnumerable<UserV1Dto>), StatusCodes.Status200OK)]
    public ActionResult<IEnumerable<UserV1Dto>> GetUsersV1()
    {
        var users = new[]
        {
            new UserV1Dto { Id = 1, Name = "John Doe", Email = "john@example.com" }
        };
        return Ok(users);
    }
    
    /// <summary>
    /// 사용자 목록 조회 (V2) - 페이징 지원
    /// </summary>
    [HttpGet]
    [ApiVersion("2.0")]
    [ProducesResponseType(typeof(PagedResult<UserV2Dto>), StatusCodes.Status200OK)]
    public ActionResult<PagedResult<UserV2Dto>> GetUsersV2(
        [FromQuery] int page = 1,
        [FromQuery] int pageSize = 10)
    {
        var users = new[]
        {
            new UserV2Dto 
            { 
                Id = 1, 
                Name = "John Doe", 
                Email = "john@example.com",
                CreatedAt = DateTime.UtcNow,
                Roles = new[] { "User" }
            }
        };
        
        return Ok(new PagedResult<UserV2Dto>
        {
            Items = users,
            TotalCount = 1,
            Page = page,
            PageSize = pageSize
        });
    }
}
```