# 모델 바인딩과 검증

## 모델 바인딩 개요

모델 바인딩은 HTTP 요청 데이터를 액션 메서드의 매개변수나 모델 객체로 자동 매핑하는 프로세스입니다.

### 바인딩 소스
- **Route Values**: URL 경로의 값
- **Query String**: URL의 쿼리 문자열
- **Form Data**: HTML 폼에서 전송된 데이터
- **Request Body**: JSON, XML 등의 요청 본문
- **Headers**: HTTP 헤더
- **Services**: DI 컨테이너의 서비스

## 기본 모델 바인딩

### 단순 타입 바인딩
```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    // GET: api/products/search?query=laptop&minPrice=500&maxPrice=2000
    [HttpGet("search")]
    public IActionResult Search(
        string query,           // Query string
        decimal minPrice = 0,   // Query string with default
        decimal? maxPrice = null) // Nullable query string
    {
        // 자동으로 쿼리 문자열에서 바인딩됨
        return Ok(new { query, minPrice, maxPrice });
    }
    
    // GET: api/products/5/reviews/10
    [HttpGet("{productId}/reviews/{reviewId}")]
    public IActionResult GetReview(
        int productId,    // Route value
        int reviewId)     // Route value
    {
        return Ok(new { productId, reviewId });
    }
}
```

### 복잡한 타입 바인딩
```csharp
// 모델 클래스
public class ProductSearchModel
{
    public string Query { get; set; }
    public decimal? MinPrice { get; set; }
    public decimal? MaxPrice { get; set; }
    public string Category { get; set; }
    public int Page { get; set; } = 1;
    public int PageSize { get; set; } = 10;
}

[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    // GET: api/products/search?query=laptop&minPrice=500&category=electronics
    [HttpGet("search")]
    public IActionResult Search([FromQuery] ProductSearchModel model)
    {
        // 쿼리 문자열의 모든 값이 모델 객체로 바인딩됨
        return Ok(model);
    }
    
    // POST: api/products
    [HttpPost]
    public IActionResult Create([FromBody] CreateProductDto product)
    {
        // JSON 요청 본문이 자동으로 객체로 변환됨
        return CreatedAtAction(nameof(GetById), new { id = product.Id }, product);
    }
}
```

## 바인딩 소스 특성

### FromQuery, FromRoute, FromBody
```csharp
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    // 명시적 바인딩 소스 지정
    [HttpGet("{id}/items")]
    public IActionResult GetOrderItems(
        [FromRoute] int id,                    // URL 경로에서
        [FromQuery] int page = 1,              // 쿼리 문자열에서
        [FromQuery] int pageSize = 10,         // 쿼리 문자열에서
        [FromHeader] string authorization = "") // 헤더에서
    {
        return Ok(new { id, page, pageSize, authorization });
    }
    
    // 복합 바인딩
    [HttpPut("{id}")]
    public IActionResult Update(
        [FromRoute] int id,
        [FromBody] UpdateOrderDto order,
        [FromHeader(Name = "X-Request-ID")] string requestId)
    {
        return Ok(new { id, order, requestId });
    }
}
```

### FromForm과 파일 업로드
```csharp
[ApiController]
[Route("api/[controller]")]
public class FilesController : ControllerBase
{
    // 단일 파일 업로드
    [HttpPost("upload")]
    public async Task<IActionResult> Upload([FromForm] FileUploadModel model)
    {
        if (model.File != null && model.File.Length > 0)
        {
            var filePath = Path.Combine("uploads", model.File.FileName);
            
            using (var stream = new FileStream(filePath, FileMode.Create))
            {
                await model.File.CopyToAsync(stream);
            }
            
            return Ok(new 
            { 
                fileName = model.File.FileName, 
                size = model.File.Length,
                description = model.Description 
            });
        }
        
        return BadRequest("No file uploaded");
    }
    
    // 다중 파일 업로드
    [HttpPost("upload-multiple")]
    public async Task<IActionResult> UploadMultiple([FromForm] MultiFileUploadModel model)
    {
        var uploadedFiles = new List<object>();
        
        foreach (var file in model.Files)
        {
            if (file.Length > 0)
            {
                var filePath = Path.Combine("uploads", file.FileName);
                
                using (var stream = new FileStream(filePath, FileMode.Create))
                {
                    await file.CopyToAsync(stream);
                }
                
                uploadedFiles.Add(new
                {
                    fileName = file.FileName,
                    size = file.Length
                });
            }
        }
        
        return Ok(new { files = uploadedFiles, tags = model.Tags });
    }
}

public class FileUploadModel
{
    public IFormFile File { get; set; }
    public string Description { get; set; }
}

public class MultiFileUploadModel
{
    public List<IFormFile> Files { get; set; }
    public string[] Tags { get; set; }
}
```

## 사용자 정의 모델 바인더

### 기본 사용자 정의 바인더
```csharp
// 사용자 정의 타입
public class GeoLocation
{
    public double Latitude { get; set; }
    public double Longitude { get; set; }
}

// 모델 바인더
public class GeoLocationModelBinder : IModelBinder
{
    public Task BindModelAsync(ModelBindingContext bindingContext)
    {
        if (bindingContext == null)
            throw new ArgumentNullException(nameof(bindingContext));
        
        var valueProvider = bindingContext.ValueProvider;
        var value = valueProvider.GetValue(bindingContext.ModelName);
        
        if (value == ValueProviderResult.None)
        {
            return Task.CompletedTask;
        }
        
        bindingContext.ModelState.SetModelValue(bindingContext.ModelName, value);
        
        var locationString = value.FirstValue;
        
        // 형식: "latitude,longitude" (예: "37.5665,126.9780")
        var parts = locationString?.Split(',');
        
        if (parts?.Length == 2 && 
            double.TryParse(parts[0], out var lat) && 
            double.TryParse(parts[1], out var lng))
        {
            var result = new GeoLocation
            {
                Latitude = lat,
                Longitude = lng
            };
            
            bindingContext.Result = ModelBindingResult.Success(result);
        }
        else
        {
            bindingContext.ModelState.TryAddModelError(
                bindingContext.ModelName, 
                "Invalid location format. Use: latitude,longitude");
            bindingContext.Result = ModelBindingResult.Failed();
        }
        
        return Task.CompletedTask;
    }
}

// 모델 바인더 프로바이더
public class GeoLocationModelBinderProvider : IModelBinderProvider
{
    public IModelBinder GetBinder(ModelBinderProviderContext context)
    {
        if (context.Metadata.ModelType == typeof(GeoLocation))
        {
            return new GeoLocationModelBinder();
        }
        
        return null;
    }
}

// Program.cs에서 등록
builder.Services.AddControllers(options =>
{
    options.ModelBinderProviders.Insert(0, new GeoLocationModelBinderProvider());
});

// 컨트롤러에서 사용
[HttpGet("nearby")]
public IActionResult GetNearbyStores([FromQuery] GeoLocation location)
{
    // location이 자동으로 "37.5665,126.9780" 형식에서 파싱됨
    return Ok(new { latitude = location.Latitude, longitude = location.Longitude });
}
```

## 모델 검증

### Data Annotations 사용
```csharp
using System.ComponentModel.DataAnnotations;

public class CreateProductDto
{
    [Required(ErrorMessage = "Product name is required")]
    [StringLength(100, MinimumLength = 3, 
        ErrorMessage = "Product name must be between 3 and 100 characters")]
    public string Name { get; set; }
    
    [Required]
    [StringLength(500)]
    public string Description { get; set; }
    
    [Required]
    [Range(0.01, 10000.00, ErrorMessage = "Price must be between 0.01 and 10000")]
    public decimal Price { get; set; }
    
    [Required]
    [RegularExpression(@"^[A-Z]{3}-\d{6}$", 
        ErrorMessage = "SKU must be in format: ABC-123456")]
    public string SKU { get; set; }
    
    [Range(0, int.MaxValue, ErrorMessage = "Stock cannot be negative")]
    public int Stock { get; set; }
    
    [Required]
    [EmailAddress(ErrorMessage = "Invalid email format")]
    public string ContactEmail { get; set; }
    
    [Url(ErrorMessage = "Invalid URL format")]
    public string ProductUrl { get; set; }
    
    [DataType(DataType.Date)]
    [Display(Name = "Release Date")]
    public DateTime ReleaseDate { get; set; }
}

[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpPost]
    public IActionResult Create([FromBody] CreateProductDto product)
    {
        // ApiController 특성이 자동으로 ModelState 검증
        // ModelState.IsValid가 false면 자동으로 400 Bad Request 반환
        
        return Ok(product);
    }
}
```

### 사용자 정의 검증 특성
```csharp
// 사용자 정의 검증 특성
public class FutureDateAttribute : ValidationAttribute
{
    protected override ValidationResult IsValid(object value, ValidationContext validationContext)
    {
        if (value is DateTime dateTime)
        {
            if (dateTime <= DateTime.Now)
            {
                return new ValidationResult(
                    ErrorMessage ?? "Date must be in the future");
            }
        }
        
        return ValidationResult.Success;
    }
}

// 비교 검증 특성
public class DateRangeAttribute : ValidationAttribute
{
    private readonly string _comparisonProperty;
    
    public DateRangeAttribute(string comparisonProperty)
    {
        _comparisonProperty = comparisonProperty;
    }
    
    protected override ValidationResult IsValid(object value, ValidationContext validationContext)
    {
        var currentValue = (DateTime?)value;
        var property = validationContext.ObjectType.GetProperty(_comparisonProperty);
        
        if (property == null)
            throw new ArgumentException("Property not found");
        
        var comparisonValue = (DateTime?)property.GetValue(validationContext.ObjectInstance);
        
        if (currentValue.HasValue && comparisonValue.HasValue && currentValue > comparisonValue)
        {
            return new ValidationResult(
                ErrorMessage ?? $"{validationContext.DisplayName} must be less than or equal to {_comparisonProperty}");
        }
        
        return ValidationResult.Success;
    }
}

// 모델에 적용
public class EventDto
{
    [Required]
    public string Name { get; set; }
    
    [Required]
    [DataType(DataType.DateTime)]
    public DateTime StartDate { get; set; }
    
    [Required]
    [DataType(DataType.DateTime)]
    [DateRange(nameof(StartDate), ErrorMessage = "End date must be after start date")]
    public DateTime EndDate { get; set; }
    
    [FutureDate(ErrorMessage = "Registration deadline must be in the future")]
    public DateTime RegistrationDeadline { get; set; }
}
```

### IValidatableObject 인터페이스
```csharp
public class CreateOrderDto : IValidatableObject
{
    public int CustomerId { get; set; }
    public List<OrderItemDto> Items { get; set; }
    public string PromoCode { get; set; }
    public decimal? DiscountAmount { get; set; }
    public DateTime? DeliveryDate { get; set; }
    
    public IEnumerable<ValidationResult> Validate(ValidationContext validationContext)
    {
        // 복잡한 비즈니스 규칙 검증
        if (Items == null || !Items.Any())
        {
            yield return new ValidationResult(
                "Order must contain at least one item",
                new[] { nameof(Items) });
        }
        
        if (Items?.Any(i => i.Quantity <= 0) == true)
        {
            yield return new ValidationResult(
                "All items must have positive quantity",
                new[] { nameof(Items) });
        }
        
        var totalAmount = Items?.Sum(i => i.Price * i.Quantity) ?? 0;
        
        if (!string.IsNullOrEmpty(PromoCode) && !DiscountAmount.HasValue)
        {
            yield return new ValidationResult(
                "Discount amount is required when promo code is provided",
                new[] { nameof(DiscountAmount) });
        }
        
        if (DiscountAmount > totalAmount)
        {
            yield return new ValidationResult(
                "Discount cannot exceed total order amount",
                new[] { nameof(DiscountAmount) });
        }
        
        if (DeliveryDate.HasValue && DeliveryDate < DateTime.Now.AddDays(1))
        {
            yield return new ValidationResult(
                "Delivery date must be at least 1 day in the future",
                new[] { nameof(DeliveryDate) });
        }
    }
}

public class OrderItemDto
{
    [Required]
    public int ProductId { get; set; }
    
    [Range(1, 100)]
    public int Quantity { get; set; }
    
    [Range(0.01, double.MaxValue)]
    public decimal Price { get; set; }
}
```

## FluentValidation 통합

### FluentValidation 설정
```csharp
// NuGet: FluentValidation.AspNetCore

// 검증자 클래스
public class CreateProductValidator : AbstractValidator<CreateProductDto>
{
    public CreateProductValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Product name is required")
            .Length(3, 100).WithMessage("Product name must be between 3 and 100 characters")
            .Must(BeUniqueName).WithMessage("Product name already exists");
        
        RuleFor(x => x.Price)
            .GreaterThan(0).WithMessage("Price must be greater than 0")
            .LessThanOrEqualTo(10000).WithMessage("Price cannot exceed 10000");
        
        RuleFor(x => x.SKU)
            .NotEmpty()
            .Matches(@"^[A-Z]{3}-\d{6}$").WithMessage("SKU must be in format: ABC-123456");
        
        RuleFor(x => x.Description)
            .NotEmpty()
            .MaximumLength(500)
            .When(x => !string.IsNullOrEmpty(x.Name));
        
        RuleFor(x => x.ReleaseDate)
            .GreaterThan(DateTime.Now).WithMessage("Release date must be in the future")
            .When(x => x.ReleaseDate.HasValue);
        
        // 복잡한 규칙
        RuleFor(x => x)
            .Must(HaveValidPriceForCategory)
            .WithMessage("Price is not valid for the selected category")
            .WithName("Product");
    }
    
    private bool BeUniqueName(string name)
    {
        // 데이터베이스 확인 로직
        return true; // 예시
    }
    
    private bool HaveValidPriceForCategory(CreateProductDto product)
    {
        // 카테고리별 가격 검증 로직
        return true; // 예시
    }
}

// Program.cs에서 등록
builder.Services.AddControllers()
    .AddFluentValidation(fv =>
    {
        fv.RegisterValidatorsFromAssemblyContaining<CreateProductValidator>();
        fv.ImplicitlyValidateChildProperties = true;
    });
```

### 조건부 검증
```csharp
public class UpdateUserValidator : AbstractValidator<UpdateUserDto>
{
    public UpdateUserValidator()
    {
        // 조건부 검증
        When(x => x.ChangePassword, () =>
        {
            RuleFor(x => x.CurrentPassword)
                .NotEmpty().WithMessage("Current password is required");
            
            RuleFor(x => x.NewPassword)
                .NotEmpty().WithMessage("New password is required")
                .MinimumLength(8).WithMessage("Password must be at least 8 characters")
                .Matches(@"[A-Z]").WithMessage("Password must contain uppercase letter")
                .Matches(@"[a-z]").WithMessage("Password must contain lowercase letter")
                .Matches(@"[0-9]").WithMessage("Password must contain number")
                .Matches(@"[^a-zA-Z0-9]").WithMessage("Password must contain special character");
            
            RuleFor(x => x.ConfirmPassword)
                .Equal(x => x.NewPassword).WithMessage("Passwords do not match");
        });
        
        // 이메일 변경 시
        When(x => !string.IsNullOrEmpty(x.Email), () =>
        {
            RuleFor(x => x.Email)
                .EmailAddress().WithMessage("Invalid email format")
                .MustAsync(BeUniqueEmail).WithMessage("Email already exists");
        });
    }
    
    private async Task<bool> BeUniqueEmail(string email, CancellationToken cancellationToken)
    {
        // 비동기 데이터베이스 확인
        await Task.Delay(100); // 예시
        return true;
    }
}
```

## 검증 오류 처리

### 사용자 정의 오류 응답
```csharp
public class ValidationErrorResponse
{
    public string Type { get; set; } = "https://tools.ietf.org/html/rfc7231#section-6.5.1";
    public string Title { get; set; } = "One or more validation errors occurred.";
    public int Status { get; set; } = 400;
    public Dictionary<string, string[]> Errors { get; set; }
    public string TraceId { get; set; }
}

// 사용자 정의 Action Filter
public class ValidationFilterAttribute : IActionFilter
{
    public void OnActionExecuting(ActionExecutingContext context)
    {
        if (!context.ModelState.IsValid)
        {
            var errors = context.ModelState
                .Where(x => x.Value.Errors.Count > 0)
                .ToDictionary(
                    kvp => kvp.Key,
                    kvp => kvp.Value.Errors.Select(e => e.ErrorMessage).ToArray()
                );
            
            var response = new ValidationErrorResponse
            {
                Errors = errors,
                TraceId = Activity.Current?.Id ?? context.HttpContext.TraceIdentifier
            };
            
            context.Result = new BadRequestObjectResult(response);
        }
    }
    
    public void OnActionExecuted(ActionExecutedContext context) { }
}

// Program.cs에서 전역 등록
builder.Services.AddControllers(options =>
{
    options.Filters.Add<ValidationFilterAttribute>();
});
```

### ModelState 수동 처리
```csharp
[ApiController]
[Route("api/[controller]")]
public class AccountController : ControllerBase
{
    [HttpPost("register")]
    public async Task<IActionResult> Register([FromBody] RegisterDto model)
    {
        // 추가 검증
        if (await _userService.EmailExistsAsync(model.Email))
        {
            ModelState.AddModelError(nameof(model.Email), "Email is already registered");
        }
        
        if (await _userService.UsernameExistsAsync(model.Username))
        {
            ModelState.AddModelError(nameof(model.Username), "Username is already taken");
        }
        
        if (!ModelState.IsValid)
        {
            return ValidationProblem(ModelState);
        }
        
        var user = await _userService.RegisterAsync(model);
        return Ok(user);
    }
}
```

## 바인딩과 검증 커스터마이징

### 전역 모델 바인딩 옵션
```csharp
// Program.cs
builder.Services.AddControllers(options =>
{
    // 빈 문자열을 null로 변환
    options.ModelMetadataDetailsProviders.Add(
        new Microsoft.AspNetCore.Mvc.ModelBinding.Metadata.SystemTextJsonValidationMetadataProvider());
    
    // 최대 모델 바인딩 오류 수
    options.MaxModelValidationErrors = 200;
    
    // 모델 바인딩 오류 메시지 사용자 정의
    options.ModelBindingMessageProvider.SetValueMustNotBeNullAccessor(
        _ => "The field is required.");
    options.ModelBindingMessageProvider.SetValueIsInvalidAccessor(
        value => $"The value '{value}' is invalid.");
    options.ModelBindingMessageProvider.SetAttemptedValueIsInvalidAccessor(
        (value, field) => $"The value '{value}' is not valid for {field}.");
});

// JSON 옵션
builder.Services.ConfigureHttpJsonOptions(options =>
{
    options.SerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
    options.SerializerOptions.WriteIndented = true;
    options.SerializerOptions.Converters.Add(new JsonStringEnumConverter());
});
```

### 커스텀 메타데이터 프로바이더
```csharp
public class CustomValidationMetadataProvider : IValidationMetadataProvider
{
    public void CreateValidationMetadata(ValidationMetadataProviderContext context)
    {
        if (context.Key.ModelType == typeof(decimal) || 
            context.Key.ModelType == typeof(decimal?))
        {
            context.ValidationMetadata.ValidatorMetadata.Add(
                new RangeAttribute(0, 1000000) 
                { 
                    ErrorMessage = "Value must be between 0 and 1,000,000" 
                });
        }
    }
}

// 등록
builder.Services.AddControllers(options =>
{
    options.ModelMetadataDetailsProviders.Add(new CustomValidationMetadataProvider());
});
```

## 실전 예제: 복잡한 모델 바인딩과 검증

```csharp
// 복잡한 DTO
public class CreateInvoiceDto : IValidatableObject
{
    [Required]
    public string InvoiceNumber { get; set; }
    
    [Required]
    public int CustomerId { get; set; }
    
    [Required]
    public DateTime InvoiceDate { get; set; }
    
    public DateTime? DueDate { get; set; }
    
    [Required]
    [MinLength(1, ErrorMessage = "Invoice must have at least one line item")]
    public List<InvoiceLineDto> LineItems { get; set; }
    
    public decimal? DiscountPercentage { get; set; }
    
    public string Notes { get; set; }
    
    public PaymentTerms PaymentTerms { get; set; }
    
    public IEnumerable<ValidationResult> Validate(ValidationContext validationContext)
    {
        // 마감일 검증
        if (DueDate.HasValue && DueDate < InvoiceDate)
        {
            yield return new ValidationResult(
                "Due date cannot be before invoice date",
                new[] { nameof(DueDate) });
        }
        
        // 할인율 검증
        if (DiscountPercentage.HasValue && (DiscountPercentage < 0 || DiscountPercentage > 100))
        {
            yield return new ValidationResult(
                "Discount percentage must be between 0 and 100",
                new[] { nameof(DiscountPercentage) });
        }
        
        // 라인 아이템 검증
        if (LineItems?.Any(item => item.Quantity <= 0) == true)
        {
            yield return new ValidationResult(
                "All line items must have positive quantity",
                new[] { nameof(LineItems) });
        }
        
        // 중복 제품 검증
        var duplicateProducts = LineItems?
            .GroupBy(x => x.ProductId)
            .Where(g => g.Count() > 1)
            .Select(g => g.Key);
        
        if (duplicateProducts?.Any() == true)
        {
            yield return new ValidationResult(
                $"Duplicate products found: {string.Join(", ", duplicateProducts)}",
                new[] { nameof(LineItems) });
        }
    }
}

public class InvoiceLineDto
{
    [Required]
    public int ProductId { get; set; }
    
    [Required]
    [Range(0.01, double.MaxValue)]
    public decimal Quantity { get; set; }
    
    [Required]
    [Range(0.01, double.MaxValue)]
    public decimal UnitPrice { get; set; }
    
    [Range(0, 100)]
    public decimal? DiscountPercentage { get; set; }
    
    public string Description { get; set; }
}

// FluentValidation 검증자
public class CreateInvoiceValidator : AbstractValidator<CreateInvoiceDto>
{
    private readonly IInvoiceService _invoiceService;
    
    public CreateInvoiceValidator(IInvoiceService invoiceService)
    {
        _invoiceService = invoiceService;
        
        RuleFor(x => x.InvoiceNumber)
            .NotEmpty()
            .Matches(@"^INV-\d{6}$").WithMessage("Invoice number must be in format: INV-000000")
            .MustAsync(BeUniqueInvoiceNumber).WithMessage("Invoice number already exists");
        
        RuleFor(x => x.CustomerId)
            .MustAsync(CustomerExists).WithMessage("Customer not found");
        
        RuleFor(x => x.InvoiceDate)
            .LessThanOrEqualTo(DateTime.Now).WithMessage("Invoice date cannot be in the future");
        
        RuleForEach(x => x.LineItems).SetValidator(new InvoiceLineValidator());
        
        RuleFor(x => x)
            .MustAsync(HaveSufficientStock)
            .WithMessage("Insufficient stock for one or more products");
    }
    
    private async Task<bool> BeUniqueInvoiceNumber(string invoiceNumber, CancellationToken cancellation)
    {
        return !await _invoiceService.InvoiceNumberExistsAsync(invoiceNumber);
    }
    
    private async Task<bool> CustomerExists(int customerId, CancellationToken cancellation)
    {
        return await _invoiceService.CustomerExistsAsync(customerId);
    }
    
    private async Task<bool> HaveSufficientStock(CreateInvoiceDto invoice, CancellationToken cancellation)
    {
        foreach (var item in invoice.LineItems)
        {
            var availableStock = await _invoiceService.GetAvailableStockAsync(item.ProductId);
            if (availableStock < item.Quantity)
                return false;
        }
        return true;
    }
}

// 컨트롤러
[ApiController]
[Route("api/[controller]")]
public class InvoicesController : ControllerBase
{
    private readonly IInvoiceService _invoiceService;
    private readonly ILogger<InvoicesController> _logger;
    
    [HttpPost]
    [ProducesResponseType(typeof(InvoiceDto), StatusCodes.Status201Created)]
    [ProducesResponseType(typeof(ValidationProblemDetails), StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> CreateInvoice([FromBody] CreateInvoiceDto model)
    {
        // ApiController가 자동으로 Data Annotations 검증
        // FluentValidation이 추가 검증 수행
        // IValidatableObject.Validate도 자동 호출
        
        try
        {
            var invoice = await _invoiceService.CreateInvoiceAsync(model);
            return CreatedAtAction(nameof(GetInvoice), new { id = invoice.Id }, invoice);
        }
        catch (BusinessException ex)
        {
            ModelState.AddModelError("", ex.Message);
            return ValidationProblem(ModelState);
        }
    }
    
    [HttpGet("{id}")]
    public async Task<ActionResult<InvoiceDto>> GetInvoice(int id)
    {
        var invoice = await _invoiceService.GetInvoiceAsync(id);
        return invoice == null ? NotFound() : Ok(invoice);
    }
}
```