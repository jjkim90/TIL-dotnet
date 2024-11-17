# 응답 포맷팅과 콘텐츠 협상

## 콘텐츠 협상 개요

콘텐츠 협상(Content Negotiation)은 클라이언트와 서버가 리소스의 최적 표현 형식을 결정하는 프로세스입니다. ASP.NET Core는 Accept 헤더를 기반으로 자동으로 응답 형식을 결정합니다.

### 콘텐츠 협상 원리
- 클라이언트가 Accept 헤더로 선호하는 형식 지정
- 서버가 지원 가능한 형식 중 최적의 것 선택
- 기본 형식은 JSON

## 기본 응답 포맷터

### JSON 응답 (기본)
```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    // 기본적으로 JSON으로 응답
    [HttpGet]
    public IActionResult GetProducts()
    {
        var products = new[]
        {
            new { Id = 1, Name = "Laptop", Price = 999.99 },
            new { Id = 2, Name = "Mouse", Price = 29.99 }
        };
        
        return Ok(products); // JSON으로 자동 직렬화
    }
    
    // 명시적 JSON 반환
    [HttpGet("{id}")]
    [Produces("application/json")]
    public IActionResult GetProduct(int id)
    {
        var product = new { Id = id, Name = "Laptop", Price = 999.99 };
        return Ok(product);
    }
}
```

### XML 응답 지원 추가
```csharp
// Program.cs
builder.Services.AddControllers(options =>
{
    // 기본 포맷터 순서 변경 가능
    options.RespectBrowserAcceptHeader = true; // false가 기본값
})
.AddXmlSerializerFormatters() // XML 지원 추가
.AddXmlDataContractSerializerFormatters(); // DataContract XML 지원

// XML 직렬화를 위한 모델 설정
[XmlRoot("Product")]
public class ProductXmlDto
{
    [XmlAttribute("id")]
    public int Id { get; set; }
    
    [XmlElement("ProductName")]
    public string Name { get; set; }
    
    [XmlElement("Price")]
    public decimal Price { get; set; }
    
    [XmlArray("Categories")]
    [XmlArrayItem("Category")]
    public List<string> Categories { get; set; }
}

[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    // Accept: application/xml 헤더로 요청하면 XML 응답
    [HttpGet("{id}")]
    [Produces("application/json", "application/xml")]
    public ActionResult<ProductXmlDto> GetProduct(int id)
    {
        var product = new ProductXmlDto
        {
            Id = id,
            Name = "Laptop",
            Price = 999.99m,
            Categories = new List<string> { "Electronics", "Computers" }
        };
        
        return Ok(product);
    }
}
```

## 사용자 정의 포맷터

### 사용자 정의 출력 포맷터
```csharp
// CSV 출력 포맷터
public class CsvOutputFormatter : TextOutputFormatter
{
    public CsvOutputFormatter()
    {
        SupportedMediaTypes.Add(MediaTypeHeaderValue.Parse("text/csv"));
        SupportedEncodings.Add(Encoding.UTF8);
        SupportedEncodings.Add(Encoding.Unicode);
    }
    
    protected override bool CanWriteType(Type type)
    {
        // IEnumerable 타입만 지원
        return type.IsGenericType && 
               type.GetGenericTypeDefinition() == typeof(IEnumerable<>);
    }
    
    public override async Task WriteResponseBodyAsync(
        OutputFormatterWriteContext context, 
        Encoding selectedEncoding)
    {
        var response = context.HttpContext.Response;
        var buffer = new StringBuilder();
        
        if (context.Object is IEnumerable<object> items)
        {
            // 헤더 작성
            var firstItem = items.FirstOrDefault();
            if (firstItem != null)
            {
                var properties = firstItem.GetType().GetProperties();
                buffer.AppendLine(string.Join(",", properties.Select(p => p.Name)));
                
                // 데이터 작성
                foreach (var item in items)
                {
                    var values = properties.Select(p => 
                    {
                        var value = p.GetValue(item)?.ToString() ?? "";
                        // CSV 이스케이프 처리
                        return value.Contains(',') || value.Contains('"') 
                            ? $"\"{value.Replace("\"", "\"\"")}\"" 
                            : value;
                    });
                    buffer.AppendLine(string.Join(",", values));
                }
            }
        }
        
        await response.WriteAsync(buffer.ToString(), selectedEncoding);
    }
}

// Program.cs에서 등록
builder.Services.AddControllers(options =>
{
    options.OutputFormatters.Add(new CsvOutputFormatter());
});
```

### 사용자 정의 입력 포맷터
```csharp
// YAML 입력 포맷터
public class YamlInputFormatter : TextInputFormatter
{
    public YamlInputFormatter()
    {
        SupportedMediaTypes.Add(MediaTypeHeaderValue.Parse("application/x-yaml"));
        SupportedMediaTypes.Add(MediaTypeHeaderValue.Parse("text/yaml"));
        SupportedEncodings.Add(Encoding.UTF8);
    }
    
    public override async Task<InputFormatterResult> ReadRequestBodyAsync(
        InputFormatterContext context, 
        Encoding encoding)
    {
        var request = context.HttpContext.Request;
        
        using (var reader = new StreamReader(request.Body, encoding))
        {
            try
            {
                var yaml = await reader.ReadToEndAsync();
                
                // YamlDotNet 라이브러리 사용
                var deserializer = new DeserializerBuilder()
                    .WithNamingConvention(CamelCaseNamingConvention.Instance)
                    .Build();
                
                var model = deserializer.Deserialize(yaml, context.ModelType);
                return await InputFormatterResult.SuccessAsync(model);
            }
            catch (Exception ex)
            {
                context.ModelState.AddModelError(context.ModelName, ex.Message);
                return await InputFormatterResult.FailureAsync();
            }
        }
    }
}
```

## 응답 형식 제어

### Produces 특성 사용
```csharp
[ApiController]
[Route("api/[controller]")]
[Produces("application/json")] // 컨트롤러 레벨 기본 형식
public class ReportsController : ControllerBase
{
    // JSON만 지원
    [HttpGet("summary")]
    [Produces("application/json")]
    public IActionResult GetSummary()
    {
        return Ok(new { Total = 100, Average = 50 });
    }
    
    // 여러 형식 지원
    [HttpGet("detailed")]
    [Produces("application/json", "application/xml", "text/csv")]
    public IActionResult GetDetailedReport()
    {
        var report = GenerateReport();
        return Ok(report);
    }
    
    // 특정 상태 코드별 응답 타입
    [HttpGet("{id}")]
    [ProducesResponseType(typeof(ReportDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    [ProducesResponseType(typeof(ErrorResponse), StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> GetReport(int id)
    {
        var report = await _reportService.GetByIdAsync(id);
        return report != null ? Ok(report) : NotFound();
    }
}
```

### FormatFilter 사용
```csharp
// Program.cs
builder.Services.AddControllers(options =>
{
    options.RespectBrowserAcceptHeader = true;
    options.ReturnHttpNotAcceptable = true; // 지원하지 않는 형식 요청 시 406 반환
})
.AddMvcOptions(options =>
{
    // URL에서 형식 지정 가능 (예: /api/products/1.json)
    options.FormatterMappings.SetMediaTypeMappingForFormat("json", "application/json");
    options.FormatterMappings.SetMediaTypeMappingForFormat("xml", "application/xml");
    options.FormatterMappings.SetMediaTypeMappingForFormat("csv", "text/csv");
});

[ApiController]
[Route("api/[controller]")]
[FormatFilter] // URL 형식 필터 활성화
public class ProductsController : ControllerBase
{
    // GET: api/products/1
    // GET: api/products/1.json
    // GET: api/products/1.xml
    // GET: api/products/1.csv
    [HttpGet("{id:int}.{format?}")]
    public IActionResult GetProduct(int id)
    {
        var product = _productService.GetById(id);
        return Ok(product);
    }
}
```

## 고급 JSON 구성

### System.Text.Json 옵션
```csharp
// Program.cs
builder.Services.ConfigureHttpJsonOptions(options =>
{
    // 대소문자 정책
    options.SerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
    
    // 들여쓰기
    options.SerializerOptions.WriteIndented = true;
    
    // null 값 포함
    options.SerializerOptions.DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull;
    
    // 순환 참조 처리
    options.SerializerOptions.ReferenceHandler = ReferenceHandler.IgnoreCycles;
    
    // 숫자를 문자열로 읽기 허용
    options.SerializerOptions.NumberHandling = JsonNumberHandling.AllowReadingFromString;
    
    // 사용자 정의 변환기
    options.SerializerOptions.Converters.Add(new JsonStringEnumConverter());
    options.SerializerOptions.Converters.Add(new DateOnlyJsonConverter());
});

// 사용자 정의 JSON 변환기
public class DateOnlyJsonConverter : JsonConverter<DateOnly>
{
    private readonly string _format = "yyyy-MM-dd";
    
    public override DateOnly Read(ref Utf8JsonReader reader, Type typeToConvert, JsonSerializerOptions options)
    {
        return DateOnly.ParseExact(reader.GetString()!, _format);
    }
    
    public override void Write(Utf8JsonWriter writer, DateOnly value, JsonSerializerOptions options)
    {
        writer.WriteStringValue(value.ToString(_format));
    }
}
```

### JsonSerializerOptions 런타임 구성
```csharp
[ApiController]
[Route("api/[controller]")]
public class DataController : ControllerBase
{
    private readonly JsonSerializerOptions _compactOptions;
    private readonly JsonSerializerOptions _verboseOptions;
    
    public DataController()
    {
        _compactOptions = new JsonSerializerOptions
        {
            DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull,
            WriteIndented = false
        };
        
        _verboseOptions = new JsonSerializerOptions
        {
            WriteIndented = true,
            DefaultIgnoreCondition = JsonIgnoreCondition.Never,
            Converters = { new JsonStringEnumConverter() }
        };
    }
    
    [HttpGet("compact")]
    public IActionResult GetCompactData()
    {
        var data = GetData();
        return new JsonResult(data, _compactOptions);
    }
    
    [HttpGet("verbose")]
    public IActionResult GetVerboseData()
    {
        var data = GetData();
        return new JsonResult(data, _verboseOptions);
    }
}
```

## 응답 압축

### 응답 압축 미들웨어
```csharp
// Program.cs
builder.Services.AddResponseCompression(options =>
{
    options.Providers.Add<BrotliCompressionProvider>();
    options.Providers.Add<GzipCompressionProvider>();
    
    // 압축할 MIME 타입
    options.MimeTypes = ResponseCompressionDefaults.MimeTypes.Concat(new[]
    {
        "application/json",
        "text/csv",
        "application/xml"
    });
    
    options.EnableForHttps = true; // HTTPS에서도 압축 활성화
});

// 압축 수준 구성
builder.Services.Configure<BrotliCompressionProviderOptions>(options =>
{
    options.Level = CompressionLevel.Optimal;
});

builder.Services.Configure<GzipCompressionProviderOptions>(options =>
{
    options.Level = CompressionLevel.Fastest;
});

var app = builder.Build();

// 미들웨어 파이프라인에 추가 (정적 파일 미들웨어 전에)
app.UseResponseCompression();
```

## Problem Details 응답

### RFC 7807 Problem Details
```csharp
// Program.cs
builder.Services.AddProblemDetails(options =>
{
    options.CustomizeProblemDetails = context =>
    {
        context.ProblemDetails.Extensions["traceId"] = 
            Activity.Current?.Id ?? context.HttpContext.TraceIdentifier;
        
        context.ProblemDetails.Extensions["timestamp"] = DateTime.UtcNow;
        
        // 개발 환경에서만 상세 정보 포함
        if (context.HttpContext.RequestServices
            .GetRequiredService<IWebHostEnvironment>().IsDevelopment())
        {
            context.ProblemDetails.Extensions["machineName"] = 
                Environment.MachineName;
        }
    };
});

[ApiController]
[Route("api/[controller]")]
public class AccountController : ControllerBase
{
    [HttpPost("login")]
    public IActionResult Login(LoginDto model)
    {
        if (!IsValidCredentials(model))
        {
            return Problem(
                title: "Invalid credentials",
                detail: "The username or password is incorrect",
                statusCode: StatusCodes.Status401Unauthorized,
                instance: HttpContext.Request.Path
            );
        }
        
        return Ok(new { token = GenerateToken(model.Username) });
    }
    
    [HttpPost("register")]
    public IActionResult Register(RegisterDto model)
    {
        var validationErrors = ValidateRegistration(model);
        if (validationErrors.Any())
        {
            return ValidationProblem(new ValidationProblemDetails
            {
                Title = "Validation failed",
                Detail = "One or more validation errors occurred",
                Errors = validationErrors,
                Type = "https://example.com/problems/validation"
            });
        }
        
        return Ok();
    }
}
```

## 버전별 응답 포맷

### API 버전별 다른 응답 형식
```csharp
// V1 DTO
public class ProductV1Dto
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
}

// V2 DTO - 확장된 정보
public class ProductV2Dto
{
    public int Id { get; set; }
    public string Name { get; set; }
    public PriceInfoDto Pricing { get; set; }
    public List<string> Tags { get; set; }
    public DateTime LastUpdated { get; set; }
}

public class PriceInfoDto
{
    public decimal BasePrice { get; set; }
    public decimal? DiscountedPrice { get; set; }
    public string Currency { get; set; }
}

// 버전별 컨트롤러
[ApiController]
[Route("api/v{version:apiVersion}/products")]
[ApiVersion("1.0")]
public class ProductsV1Controller : ControllerBase
{
    [HttpGet("{id}")]
    [ProducesResponseType(typeof(ProductV1Dto), StatusCodes.Status200OK)]
    public IActionResult GetProduct(int id)
    {
        var product = new ProductV1Dto
        {
            Id = id,
            Name = "Laptop",
            Price = 999.99m
        };
        
        return Ok(product);
    }
}

[ApiController]
[Route("api/v{version:apiVersion}/products")]
[ApiVersion("2.0")]
public class ProductsV2Controller : ControllerBase
{
    [HttpGet("{id}")]
    [ProducesResponseType(typeof(ProductV2Dto), StatusCodes.Status200OK)]
    public IActionResult GetProduct(int id)
    {
        var product = new ProductV2Dto
        {
            Id = id,
            Name = "Laptop",
            Pricing = new PriceInfoDto
            {
                BasePrice = 999.99m,
                DiscountedPrice = 899.99m,
                Currency = "USD"
            },
            Tags = new List<string> { "Electronics", "Computers" },
            LastUpdated = DateTime.UtcNow
        };
        
        return Ok(product);
    }
}
```

## 응답 캐싱

### 응답 캐싱 헤더
```csharp
[ApiController]
[Route("api/[controller]")]
public class CatalogController : ControllerBase
{
    // 기본 캐싱
    [HttpGet("categories")]
    [ResponseCache(Duration = 3600)] // 1시간 캐싱
    public IActionResult GetCategories()
    {
        var categories = _categoryService.GetAll();
        return Ok(categories);
    }
    
    // 조건부 캐싱
    [HttpGet("products")]
    [ResponseCache(Duration = 300, VaryByQueryKeys = new[] { "category", "page" })]
    public IActionResult GetProducts(string category, int page = 1)
    {
        var products = _productService.GetByCategory(category, page);
        return Ok(products);
    }
    
    // 캐싱 방지
    [HttpGet("user/profile")]
    [ResponseCache(NoStore = true, Location = ResponseCacheLocation.None)]
    public IActionResult GetUserProfile()
    {
        var profile = _userService.GetCurrentUserProfile();
        return Ok(profile);
    }
    
    // ETag 사용
    [HttpGet("products/{id}")]
    public IActionResult GetProduct(int id)
    {
        var product = _productService.GetById(id);
        if (product == null)
            return NotFound();
        
        var etag = GenerateETag(product);
        
        // If-None-Match 헤더 확인
        if (Request.Headers.TryGetValue("If-None-Match", out var requestETag) 
            && requestETag == etag)
        {
            return StatusCode(StatusCodes.Status304NotModified);
        }
        
        Response.Headers.Add("ETag", etag);
        return Ok(product);
    }
    
    private string GenerateETag(object obj)
    {
        var json = JsonSerializer.Serialize(obj);
        var bytes = Encoding.UTF8.GetBytes(json);
        var hash = SHA256.HashData(bytes);
        return Convert.ToBase64String(hash);
    }
}
```

## HATEOAS 응답 포맷

### 하이퍼미디어 링크 포함
```csharp
public class HateoasResponse<T>
{
    public T Data { get; set; }
    public List<Link> Links { get; set; } = new();
    public Dictionary<string, object> Metadata { get; set; } = new();
}

public class Link
{
    public string Href { get; set; }
    public string Rel { get; set; }
    public string Method { get; set; }
    public string Title { get; set; }
}

public interface IHateoasService
{
    HateoasResponse<T> CreateResponse<T>(T data, HttpContext context);
}

public class HateoasService : IHateoasService
{
    private readonly LinkGenerator _linkGenerator;
    
    public HateoasService(LinkGenerator linkGenerator)
    {
        _linkGenerator = linkGenerator;
    }
    
    public HateoasResponse<T> CreateResponse<T>(T data, HttpContext context)
    {
        var response = new HateoasResponse<T> { Data = data };
        
        // 동적으로 링크 생성
        if (data is IHateoasResource resource)
        {
            response.Links = resource.GetLinks(_linkGenerator, context);
        }
        
        return response;
    }
}

// 리소스 인터페이스
public interface IHateoasResource
{
    List<Link> GetLinks(LinkGenerator linkGenerator, HttpContext context);
}

// 구현 예제
public class OrderDto : IHateoasResource
{
    public int Id { get; set; }
    public string Status { get; set; }
    public decimal Total { get; set; }
    
    public List<Link> GetLinks(LinkGenerator linkGenerator, HttpContext context)
    {
        var links = new List<Link>
        {
            new Link
            {
                Href = linkGenerator.GetUriByAction(context, "GetOrder", "Orders", new { id = Id }),
                Rel = "self",
                Method = "GET",
                Title = "Get order details"
            }
        };
        
        if (Status == "Pending")
        {
            links.Add(new Link
            {
                Href = linkGenerator.GetUriByAction(context, "CancelOrder", "Orders", new { id = Id }),
                Rel = "cancel",
                Method = "POST",
                Title = "Cancel this order"
            });
        }
        
        return links;
    }
}
```

## 실전 예제: 다양한 응답 포맷 지원 API

```csharp
// Program.cs 구성
var builder = WebApplication.CreateBuilder(args);

// 서비스 등록
builder.Services.AddControllers(options =>
{
    options.RespectBrowserAcceptHeader = true;
    options.ReturnHttpNotAcceptable = true;
    
    // 사용자 정의 포맷터 추가
    options.OutputFormatters.Add(new CsvOutputFormatter());
    options.OutputFormatters.Add(new ExcelOutputFormatter());
    
    // 형식 매핑
    options.FormatterMappings.SetMediaTypeMappingForFormat("csv", "text/csv");
    options.FormatterMappings.SetMediaTypeMappingForFormat("xlsx", "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet");
})
.AddXmlSerializerFormatters()
.ConfigureApiBehaviorOptions(options =>
{
    // 사용자 정의 문제 세부 정보 팩토리
    options.InvalidModelStateResponseFactory = context =>
    {
        var problemDetails = new ValidationProblemDetails(context.ModelState)
        {
            Type = "https://example.com/problems/validation",
            Title = "Validation error",
            Status = StatusCodes.Status422UnprocessableEntity,
            Detail = "See errors property for details",
            Instance = context.HttpContext.Request.Path
        };
        
        return new UnprocessableEntityObjectResult(problemDetails)
        {
            ContentTypes = { "application/problem+json" }
        };
    };
});

// Excel 출력 포맷터
public class ExcelOutputFormatter : OutputFormatter
{
    public ExcelOutputFormatter()
    {
        SupportedMediaTypes.Add(MediaTypeHeaderValue.Parse("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"));
    }
    
    public override bool CanWriteResult(OutputFormatterCanWriteContext context)
    {
        return context.Object is IEnumerable<object>;
    }
    
    public override async Task WriteResponseBodyAsync(OutputFormatterWriteContext context)
    {
        var response = context.HttpContext.Response;
        response.ContentType = "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet";
        
        if (context.Object is IEnumerable<object> items)
        {
            using var package = new ExcelPackage();
            var worksheet = package.Workbook.Worksheets.Add("Data");
            
            // 헤더 작성
            var firstItem = items.FirstOrDefault();
            if (firstItem != null)
            {
                var properties = firstItem.GetType().GetProperties();
                for (int i = 0; i < properties.Length; i++)
                {
                    worksheet.Cells[1, i + 1].Value = properties[i].Name;
                }
                
                // 데이터 작성
                int row = 2;
                foreach (var item in items)
                {
                    for (int i = 0; i < properties.Length; i++)
                    {
                        worksheet.Cells[row, i + 1].Value = properties[i].GetValue(item);
                    }
                    row++;
                }
                
                // 자동 맞춤
                worksheet.Cells.AutoFitColumns();
            }
            
            var bytes = package.GetAsByteArray();
            await response.Body.WriteAsync(bytes, 0, bytes.Length);
        }
    }
}

// 통합 리포트 컨트롤러
[ApiController]
[Route("api/[controller]")]
[FormatFilter]
public class ReportsController : ControllerBase
{
    private readonly IReportService _reportService;
    private readonly IHateoasService _hateoasService;
    
    public ReportsController(IReportService reportService, IHateoasService hateoasService)
    {
        _reportService = reportService;
        _hateoasService = hateoasService;
    }
    
    // GET: api/reports/sales.json
    // GET: api/reports/sales.xml
    // GET: api/reports/sales.csv
    // GET: api/reports/sales.xlsx
    [HttpGet("sales.{format?}")]
    [Produces("application/json", "application/xml", "text/csv", "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet")]
    public async Task<IActionResult> GetSalesReport(
        [FromQuery] DateTime? from,
        [FromQuery] DateTime? to,
        [FromQuery] bool includeDetails = false)
    {
        var report = await _reportService.GetSalesReportAsync(from, to, includeDetails);
        
        // Accept 헤더에 따라 다른 응답
        if (Request.Headers.Accept.Contains("application/hal+json"))
        {
            var hateoasResponse = _hateoasService.CreateResponse(report, HttpContext);
            return Ok(hateoasResponse);
        }
        
        return Ok(report);
    }
    
    // 동적 필드 선택
    [HttpGet("custom")]
    public IActionResult GetCustomReport([FromQuery] string fields)
    {
        var allData = _reportService.GetAllData();
        
        if (!string.IsNullOrEmpty(fields))
        {
            var selectedFields = fields.Split(',');
            var filteredData = allData.Select(item =>
            {
                var expando = new ExpandoObject() as IDictionary<string, object>;
                var properties = item.GetType().GetProperties()
                    .Where(p => selectedFields.Contains(p.Name, StringComparer.OrdinalIgnoreCase));
                
                foreach (var prop in properties)
                {
                    expando[prop.Name] = prop.GetValue(item);
                }
                
                return expando;
            });
            
            return Ok(filteredData);
        }
        
        return Ok(allData);
    }
}
```