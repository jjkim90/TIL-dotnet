# RESTful API 설계와 컨트롤러

## REST 원칙

REST(Representational State Transfer)는 웹 서비스 설계를 위한 아키텍처 스타일입니다.

### REST의 6가지 제약 조건
1. **Client-Server**: 클라이언트와 서버의 관심사 분리
2. **Stateless**: 각 요청은 독립적이며 서버는 클라이언트 상태를 저장하지 않음
3. **Cacheable**: 응답은 캐시 가능 여부를 명시
4. **Uniform Interface**: 일관된 인터페이스
5. **Layered System**: 계층화된 시스템 구조
6. **Code on Demand** (선택적): 클라이언트에서 실행 가능한 코드 전송

### RESTful API 설계 원칙
- 리소스 중심 설계
- HTTP 메서드 활용 (GET, POST, PUT, DELETE, PATCH)
- 명사형 URI 사용
- 계층적 구조 표현
- 일관된 명명 규칙

## API 컨트롤러 기초

### ApiController 특성
```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    // ApiController 특성이 제공하는 기능:
    // - 자동 모델 검증
    // - 바인딩 소스 추론
    // - 문제 세부 정보 응답
    // - Multipart/form-data 추론
}
```

### 기본 CRUD 작업
```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _productService;
    private readonly ILogger<ProductsController> _logger;
    
    public ProductsController(
        IProductService productService,
        ILogger<ProductsController> logger)
    {
        _productService = productService;
        _logger = logger;
    }
    
    // GET: api/products
    [HttpGet]
    public async Task<ActionResult<IEnumerable<ProductDto>>> GetAll()
    {
        var products = await _productService.GetAllAsync();
        return Ok(products);
    }
    
    // GET: api/products/5
    [HttpGet("{id}")]
    public async Task<ActionResult<ProductDto>> GetById(int id)
    {
        var product = await _productService.GetByIdAsync(id);
        
        if (product == null)
        {
            return NotFound();
        }
        
        return Ok(product);
    }
    
    // POST: api/products
    [HttpPost]
    public async Task<ActionResult<ProductDto>> Create(CreateProductDto dto)
    {
        var product = await _productService.CreateAsync(dto);
        
        return CreatedAtAction(
            nameof(GetById), 
            new { id = product.Id }, 
            product);
    }
    
    // PUT: api/products/5
    [HttpPut("{id}")]
    public async Task<IActionResult> Update(int id, UpdateProductDto dto)
    {
        if (id != dto.Id)
        {
            return BadRequest();
        }
        
        var updated = await _productService.UpdateAsync(dto);
        
        if (!updated)
        {
            return NotFound();
        }
        
        return NoContent();
    }
    
    // DELETE: api/products/5
    [HttpDelete("{id}")]
    public async Task<IActionResult> Delete(int id)
    {
        var deleted = await _productService.DeleteAsync(id);
        
        if (!deleted)
        {
            return NotFound();
        }
        
        return NoContent();
    }
}
```

## 고급 라우팅

### 복잡한 라우트 패턴
```csharp
[ApiController]
[Route("api/v{version:apiVersion}/[controller]")]
public class OrdersController : ControllerBase
{
    // GET: api/v1/orders/customer/123/orders
    [HttpGet("customer/{customerId}/orders")]
    public async Task<ActionResult<IEnumerable<OrderDto>>> GetCustomerOrders(
        int customerId,
        [FromQuery] DateTime? fromDate,
        [FromQuery] DateTime? toDate,
        [FromQuery] int page = 1,
        [FromQuery] int pageSize = 10)
    {
        var orders = await _orderService.GetCustomerOrdersAsync(
            customerId, fromDate, toDate, page, pageSize);
        
        return Ok(orders);
    }
    
    // GET: api/v1/orders/2024/03
    [HttpGet("{year:int:min(2020)}/{month:int:range(1,12)}")]
    public async Task<ActionResult<IEnumerable<OrderDto>>> GetOrdersByMonth(
        int year, 
        int month)
    {
        var orders = await _orderService.GetOrdersByMonthAsync(year, month);
        return Ok(orders);
    }
    
    // POST: api/v1/orders/123/items
    [HttpPost("{orderId}/items")]
    public async Task<ActionResult<OrderItemDto>> AddOrderItem(
        int orderId,
        [FromBody] CreateOrderItemDto dto)
    {
        var orderItem = await _orderService.AddOrderItemAsync(orderId, dto);
        
        return CreatedAtAction(
            nameof(GetOrderItem),
            new { orderId, itemId = orderItem.Id },
            orderItem);
    }
    
    // GET: api/v1/orders/123/items/456
    [HttpGet("{orderId}/items/{itemId}", Name = nameof(GetOrderItem))]
    public async Task<ActionResult<OrderItemDto>> GetOrderItem(
        int orderId, 
        int itemId)
    {
        var item = await _orderService.GetOrderItemAsync(orderId, itemId);
        
        if (item == null)
        {
            return NotFound();
        }
        
        return Ok(item);
    }
}
```

### 액션 이름과 라우트 이름
```csharp
[ApiController]
[Route("api/[controller]")]
public class DocumentsController : ControllerBase
{
    // 액션 이름 지정
    [HttpGet("{id}")]
    [ActionName("GetDocument")]
    public async Task<ActionResult<DocumentDto>> GetById(int id)
    {
        // ...
    }
    
    // 라우트 이름 지정 (링크 생성용)
    [HttpGet("{id}/download", Name = "DownloadDocument")]
    public async Task<IActionResult> Download(int id)
    {
        var document = await _documentService.GetByIdAsync(id);
        
        if (document == null)
            return NotFound();
        
        return File(document.Content, document.ContentType, document.FileName);
    }
    
    [HttpPost]
    public async Task<ActionResult<DocumentDto>> Upload(IFormFile file)
    {
        var document = await _documentService.UploadAsync(file);
        
        // 라우트 이름을 사용한 URL 생성
        var downloadUrl = Url.Link("DownloadDocument", new { id = document.Id });
        
        return Created(downloadUrl, document);
    }
}
```

## HTTP 상태 코드 활용

### 적절한 상태 코드 반환
```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    private readonly IUserService _userService;
    
    // 200 OK - 성공적인 조회
    [HttpGet("{id}")]
    public async Task<ActionResult<UserDto>> GetUser(int id)
    {
        var user = await _userService.GetByIdAsync(id);
        return user != null ? Ok(user) : NotFound();
    }
    
    // 201 Created - 리소스 생성
    [HttpPost]
    public async Task<ActionResult<UserDto>> CreateUser(CreateUserDto dto)
    {
        var user = await _userService.CreateAsync(dto);
        return CreatedAtAction(nameof(GetUser), new { id = user.Id }, user);
    }
    
    // 204 No Content - 성공했지만 반환할 내용 없음
    [HttpPut("{id}")]
    public async Task<IActionResult> UpdateUser(int id, UpdateUserDto dto)
    {
        var result = await _userService.UpdateAsync(id, dto);
        return result ? NoContent() : NotFound();
    }
    
    // 202 Accepted - 비동기 처리 시작
    [HttpPost("{id}/send-verification")]
    public async Task<IActionResult> SendVerification(int id)
    {
        await _userService.QueueVerificationEmailAsync(id);
        return Accepted(new { message = "Verification email queued" });
    }
    
    // 409 Conflict - 리소스 충돌
    [HttpPost("register")]
    public async Task<ActionResult<UserDto>> Register(RegisterDto dto)
    {
        if (await _userService.EmailExistsAsync(dto.Email))
        {
            return Conflict(new { message = "Email already exists" });
        }
        
        var user = await _userService.RegisterAsync(dto);
        return CreatedAtAction(nameof(GetUser), new { id = user.Id }, user);
    }
    
    // 400 Bad Request - 잘못된 요청
    [HttpPatch("{id}")]
    public async Task<IActionResult> PatchUser(
        int id, 
        [FromBody] JsonPatchDocument<UserDto> patchDoc)
    {
        if (patchDoc == null)
        {
            return BadRequest();
        }
        
        var user = await _userService.GetByIdAsync(id);
        if (user == null)
        {
            return NotFound();
        }
        
        patchDoc.ApplyTo(user, ModelState);
        
        if (!ModelState.IsValid)
        {
            return BadRequest(ModelState);
        }
        
        await _userService.UpdateAsync(user);
        return NoContent();
    }
}
```

## 응답 포맷팅

### ActionResult<T> 사용
```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    // ActionResult<T>를 사용하면 명시적 타입과 상태 코드 모두 반환 가능
    [HttpGet("{id}")]
    [ProducesResponseType(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<ProductDto>> GetProduct(int id)
    {
        var product = await _productService.GetByIdAsync(id);
        
        if (product == null)
        {
            return NotFound(); // ActionResult
        }
        
        return product; // T (암시적 Ok)
    }
    
    // 다양한 응답 타입 문서화
    [HttpPost]
    [ProducesResponseType(typeof(ProductDto), StatusCodes.Status201Created)]
    [ProducesResponseType(typeof(ValidationProblemDetails), StatusCodes.Status400BadRequest)]
    [ProducesDefaultResponseType]
    public async Task<ActionResult<ProductDto>> CreateProduct(CreateProductDto dto)
    {
        if (!ModelState.IsValid)
        {
            return ValidationProblem();
        }
        
        var product = await _productService.CreateAsync(dto);
        return CreatedAtAction(nameof(GetProduct), new { id = product.Id }, product);
    }
}
```

### 사용자 정의 응답 래퍼
```csharp
public class ApiResponse<T>
{
    public bool Success { get; set; }
    public T Data { get; set; }
    public string Message { get; set; }
    public List<string> Errors { get; set; }
    public DateTime Timestamp { get; set; } = DateTime.UtcNow;
}

public class BaseApiController : ControllerBase
{
    protected ActionResult<ApiResponse<T>> ApiOk<T>(T data, string message = null)
    {
        return Ok(new ApiResponse<T>
        {
            Success = true,
            Data = data,
            Message = message
        });
    }
    
    protected ActionResult<ApiResponse<T>> ApiError<T>(string message, List<string> errors = null)
    {
        return BadRequest(new ApiResponse<T>
        {
            Success = false,
            Message = message,
            Errors = errors
        });
    }
}

[ApiController]
[Route("api/[controller]")]
public class ProductsController : BaseApiController
{
    [HttpGet]
    public async Task<ActionResult<ApiResponse<IEnumerable<ProductDto>>>> GetAll()
    {
        var products = await _productService.GetAllAsync();
        return ApiOk(products, "Products retrieved successfully");
    }
}
```

## 페이징과 필터링

### 페이징 구현
```csharp
public class PaginationParameters
{
    private const int MaxPageSize = 50;
    private int _pageSize = 10;
    
    public int PageNumber { get; set; } = 1;
    
    public int PageSize
    {
        get => _pageSize;
        set => _pageSize = value > MaxPageSize ? MaxPageSize : value;
    }
}

public class PagedResult<T>
{
    public IEnumerable<T> Items { get; set; }
    public int PageNumber { get; set; }
    public int PageSize { get; set; }
    public int TotalPages { get; set; }
    public int TotalCount { get; set; }
    public bool HasPrevious => PageNumber > 1;
    public bool HasNext => PageNumber < TotalPages;
}

[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    public async Task<ActionResult<PagedResult<ProductDto>>> GetProducts(
        [FromQuery] PaginationParameters pagination,
        [FromQuery] string searchTerm = null,
        [FromQuery] string sortBy = "name",
        [FromQuery] bool descending = false)
    {
        var result = await _productService.GetPagedAsync(
            pagination.PageNumber,
            pagination.PageSize,
            searchTerm,
            sortBy,
            descending);
        
        // 페이징 메타데이터를 헤더에 추가
        Response.Headers.Add("X-Pagination", JsonSerializer.Serialize(new
        {
            result.PageNumber,
            result.PageSize,
            result.TotalPages,
            result.TotalCount,
            result.HasPrevious,
            result.HasNext
        }));
        
        return Ok(result);
    }
}
```

### 동적 필터링
```csharp
public class ProductFilter
{
    public string Name { get; set; }
    public decimal? MinPrice { get; set; }
    public decimal? MaxPrice { get; set; }
    public string Category { get; set; }
    public bool? InStock { get; set; }
    public DateTime? CreatedAfter { get; set; }
    public DateTime? CreatedBefore { get; set; }
}

public interface IProductService
{
    Task<PagedResult<ProductDto>> SearchAsync(
        ProductFilter filter,
        PaginationParameters pagination,
        string sortBy = null,
        bool descending = false);
}

[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    // GET: api/products/search?name=laptop&minPrice=500&maxPrice=2000&category=electronics
    [HttpGet("search")]
    public async Task<ActionResult<PagedResult<ProductDto>>> Search(
        [FromQuery] ProductFilter filter,
        [FromQuery] PaginationParameters pagination,
        [FromQuery] string sortBy = "name",
        [FromQuery] bool descending = false)
    {
        var result = await _productService.SearchAsync(
            filter, pagination, sortBy, descending);
        
        return Ok(result);
    }
}
```

## HATEOAS 구현

### 하이퍼미디어 링크 추가
```csharp
public class Link
{
    public string Href { get; set; }
    public string Rel { get; set; }
    public string Method { get; set; }
}

public class Resource<T>
{
    public T Data { get; set; }
    public List<Link> Links { get; set; } = new List<Link>();
}

public abstract class HateoasController : ControllerBase
{
    protected Resource<T> CreateResource<T>(T data)
    {
        return new Resource<T> { Data = data };
    }
    
    protected Link CreateLink(string routeName, object routeValues, string rel, string method)
    {
        return new Link
        {
            Href = Url.Link(routeName, routeValues),
            Rel = rel,
            Method = method
        };
    }
}

[ApiController]
[Route("api/[controller]")]
public class OrdersController : HateoasController
{
    [HttpGet("{id}", Name = "GetOrder")]
    public async Task<ActionResult<Resource<OrderDto>>> GetOrder(int id)
    {
        var order = await _orderService.GetByIdAsync(id);
        
        if (order == null)
            return NotFound();
        
        var resource = CreateResource(order);
        
        // 자기 자신에 대한 링크
        resource.Links.Add(CreateLink("GetOrder", new { id }, "self", "GET"));
        
        // 주문 상태에 따른 가능한 작업
        if (order.Status == OrderStatus.Pending)
        {
            resource.Links.Add(CreateLink("CancelOrder", new { id }, "cancel", "POST"));
            resource.Links.Add(CreateLink("UpdateOrder", new { id }, "update", "PUT"));
        }
        
        if (order.Status == OrderStatus.Shipped)
        {
            resource.Links.Add(CreateLink("TrackOrder", new { id }, "track", "GET"));
        }
        
        // 관련 리소스 링크
        resource.Links.Add(CreateLink("GetCustomer", new { id = order.CustomerId }, "customer", "GET"));
        resource.Links.Add(CreateLink("GetOrderItems", new { orderId = id }, "items", "GET"));
        
        return Ok(resource);
    }
    
    [HttpPost("{id}/cancel", Name = "CancelOrder")]
    public async Task<IActionResult> CancelOrder(int id)
    {
        var cancelled = await _orderService.CancelAsync(id);
        return cancelled ? NoContent() : NotFound();
    }
}
```

## 파일 업로드/다운로드

### 파일 업로드
```csharp
[ApiController]
[Route("api/[controller]")]
public class FilesController : ControllerBase
{
    private readonly IFileService _fileService;
    private readonly ILogger<FilesController> _logger;
    
    // 단일 파일 업로드
    [HttpPost("upload")]
    [RequestSizeLimit(10_000_000)] // 10MB 제한
    public async Task<ActionResult<FileUploadResult>> UploadFile(IFormFile file)
    {
        if (file == null || file.Length == 0)
            return BadRequest("No file uploaded");
        
        var allowedExtensions = new[] { ".jpg", ".jpeg", ".png", ".pdf", ".docx" };
        var extension = Path.GetExtension(file.FileName).ToLowerInvariant();
        
        if (!allowedExtensions.Contains(extension))
            return BadRequest("Invalid file type");
        
        try
        {
            var result = await _fileService.UploadAsync(file);
            return Ok(result);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error uploading file");
            return StatusCode(500, "Error uploading file");
        }
    }
    
    // 다중 파일 업로드
    [HttpPost("upload-multiple")]
    [RequestSizeLimit(50_000_000)] // 50MB 제한
    public async Task<ActionResult<IEnumerable<FileUploadResult>>> UploadMultiple(
        List<IFormFile> files)
    {
        if (files == null || !files.Any())
            return BadRequest("No files uploaded");
        
        var results = new List<FileUploadResult>();
        
        foreach (var file in files)
        {
            try
            {
                var result = await _fileService.UploadAsync(file);
                results.Add(result);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, $"Error uploading file: {file.FileName}");
                results.Add(new FileUploadResult 
                { 
                    Success = false, 
                    FileName = file.FileName,
                    Error = "Upload failed"
                });
            }
        }
        
        return Ok(results);
    }
    
    // 파일 다운로드
    [HttpGet("download/{id}")]
    public async Task<IActionResult> DownloadFile(Guid id)
    {
        var file = await _fileService.GetFileAsync(id);
        
        if (file == null)
            return NotFound();
        
        return File(file.Content, file.ContentType, file.FileName);
    }
    
    // 스트리밍 다운로드 (큰 파일용)
    [HttpGet("stream/{id}")]
    public async Task<IActionResult> StreamFile(Guid id)
    {
        var fileInfo = await _fileService.GetFileInfoAsync(id);
        
        if (fileInfo == null)
            return NotFound();
        
        var stream = await _fileService.GetFileStreamAsync(id);
        
        return File(stream, fileInfo.ContentType, fileInfo.FileName);
    }
}

public class FileUploadResult
{
    public bool Success { get; set; } = true;
    public Guid FileId { get; set; }
    public string FileName { get; set; }
    public long Size { get; set; }
    public string ContentType { get; set; }
    public string Error { get; set; }
}
```

## API 보안 고려사항

### 입력 검증과 살균
```csharp
[ApiController]
[Route("api/[controller]")]
public class SecurityAwareController : ControllerBase
{
    // SQL Injection 방지
    [HttpGet("search")]
    public async Task<ActionResult<IEnumerable<ProductDto>>> Search(
        [FromQuery] string query)
    {
        // 파라미터화된 쿼리 사용 (EF Core는 자동으로 처리)
        var products = await _context.Products
            .Where(p => p.Name.Contains(query))
            .ToListAsync();
        
        return Ok(products);
    }
    
    // XSS 방지
    [HttpPost("comments")]
    public async Task<ActionResult<CommentDto>> CreateComment(
        [FromBody] CreateCommentDto dto)
    {
        // HTML 태그 제거 또는 인코딩
        dto.Content = HtmlEncoder.Default.Encode(dto.Content);
        
        var comment = await _commentService.CreateAsync(dto);
        return CreatedAtAction(nameof(GetComment), new { id = comment.Id }, comment);
    }
    
    // Rate Limiting
    [HttpPost("reset-password")]
    [ServiceFilter(typeof(RateLimitAttribute))]
    public async Task<IActionResult> ResetPassword(
        [FromBody] ResetPasswordDto dto)
    {
        await _userService.ResetPasswordAsync(dto);
        return Ok();
    }
}

// Rate Limiting 특성
public class RateLimitAttribute : ActionFilterAttribute
{
    private static readonly MemoryCache _cache = new MemoryCache(new MemoryCacheOptions());
    private readonly int _limit;
    private readonly TimeSpan _period;
    
    public RateLimitAttribute(int limit = 5, int periodInMinutes = 1)
    {
        _limit = limit;
        _period = TimeSpan.FromMinutes(periodInMinutes);
    }
    
    public override void OnActionExecuting(ActionExecutingContext context)
    {
        var ipAddress = context.HttpContext.Connection.RemoteIpAddress?.ToString();
        var key = $"rate_limit_{ipAddress}_{context.ActionDescriptor.DisplayName}";
        
        if (_cache.TryGetValue<int>(key, out var count))
        {
            if (count >= _limit)
            {
                context.Result = new ContentResult
                {
                    Content = "Rate limit exceeded",
                    StatusCode = 429
                };
                return;
            }
            
            _cache.Set(key, count + 1, _period);
        }
        else
        {
            _cache.Set(key, 1, _period);
        }
        
        base.OnActionExecuting(context);
    }
}
```

## 실전 예제: 완전한 RESTful API

```csharp
// DTO 모델
public class BookDto
{
    public int Id { get; set; }
    public string Title { get; set; }
    public string Author { get; set; }
    public string ISBN { get; set; }
    public DateTime PublishedDate { get; set; }
    public decimal Price { get; set; }
    public int Stock { get; set; }
}

public class CreateBookDto
{
    [Required]
    [MaxLength(200)]
    public string Title { get; set; }
    
    [Required]
    [MaxLength(100)]
    public string Author { get; set; }
    
    [Required]
    [RegularExpression(@"^\d{3}-\d{10}$", ErrorMessage = "Invalid ISBN format")]
    public string ISBN { get; set; }
    
    public DateTime PublishedDate { get; set; }
    
    [Range(0.01, 9999.99)]
    public decimal Price { get; set; }
    
    [Range(0, int.MaxValue)]
    public int Stock { get; set; }
}

// 컨트롤러
[ApiController]
[Route("api/v1/[controller]")]
[Produces("application/json")]
public class BooksController : ControllerBase
{
    private readonly IBookService _bookService;
    private readonly ILogger<BooksController> _logger;
    
    public BooksController(IBookService bookService, ILogger<BooksController> logger)
    {
        _bookService = bookService;
        _logger = logger;
    }
    
    /// <summary>
    /// Get all books with pagination and filtering
    /// </summary>
    [HttpGet]
    [ProducesResponseType(typeof(PagedResult<BookDto>), StatusCodes.Status200OK)]
    public async Task<ActionResult<PagedResult<BookDto>>> GetBooks(
        [FromQuery] int pageNumber = 1,
        [FromQuery] int pageSize = 10,
        [FromQuery] string search = null,
        [FromQuery] string author = null,
        [FromQuery] decimal? minPrice = null,
        [FromQuery] decimal? maxPrice = null,
        [FromQuery] string sortBy = "title",
        [FromQuery] bool descending = false)
    {
        var filter = new BookFilter
        {
            Search = search,
            Author = author,
            MinPrice = minPrice,
            MaxPrice = maxPrice
        };
        
        var result = await _bookService.GetBooksAsync(
            filter, pageNumber, pageSize, sortBy, descending);
        
        // Add pagination headers
        var paginationMetadata = new
        {
            result.TotalCount,
            result.PageSize,
            result.PageNumber,
            result.TotalPages,
            result.HasNext,
            result.HasPrevious
        };
        
        Response.Headers.Add("X-Pagination", 
            JsonSerializer.Serialize(paginationMetadata));
        
        return Ok(result);
    }
    
    /// <summary>
    /// Get a specific book by ID
    /// </summary>
    [HttpGet("{id}")]
    [ProducesResponseType(typeof(BookDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<BookDto>> GetBook(int id)
    {
        var book = await _bookService.GetByIdAsync(id);
        
        if (book == null)
        {
            _logger.LogWarning("Book with id {BookId} not found", id);
            return NotFound(new { message = $"Book with id {id} not found" });
        }
        
        return Ok(book);
    }
    
    /// <summary>
    /// Create a new book
    /// </summary>
    [HttpPost]
    [ProducesResponseType(typeof(BookDto), StatusCodes.Status201Created)]
    [ProducesResponseType(typeof(ValidationProblemDetails), StatusCodes.Status400BadRequest)]
    [ProducesResponseType(StatusCodes.Status409Conflict)]
    public async Task<ActionResult<BookDto>> CreateBook([FromBody] CreateBookDto dto)
    {
        // Check if ISBN already exists
        if (await _bookService.IsbnExistsAsync(dto.ISBN))
        {
            return Conflict(new { message = $"Book with ISBN {dto.ISBN} already exists" });
        }
        
        var book = await _bookService.CreateAsync(dto);
        
        _logger.LogInformation("Book created with id {BookId}", book.Id);
        
        return CreatedAtAction(
            nameof(GetBook), 
            new { id = book.Id }, 
            book);
    }
    
    /// <summary>
    /// Update an existing book
    /// </summary>
    [HttpPut("{id}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    [ProducesResponseType(typeof(ValidationProblemDetails), StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> UpdateBook(int id, [FromBody] UpdateBookDto dto)
    {
        if (id != dto.Id)
        {
            return BadRequest(new { message = "ID mismatch" });
        }
        
        var updated = await _bookService.UpdateAsync(dto);
        
        if (!updated)
        {
            return NotFound(new { message = $"Book with id {id} not found" });
        }
        
        _logger.LogInformation("Book with id {BookId} updated", id);
        
        return NoContent();
    }
    
    /// <summary>
    /// Partially update a book
    /// </summary>
    [HttpPatch("{id}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    [ProducesResponseType(typeof(ValidationProblemDetails), StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> PatchBook(
        int id, 
        [FromBody] JsonPatchDocument<UpdateBookDto> patchDoc)
    {
        if (patchDoc == null)
        {
            return BadRequest();
        }
        
        var book = await _bookService.GetByIdAsync(id);
        if (book == null)
        {
            return NotFound();
        }
        
        var bookToPatch = new UpdateBookDto
        {
            Id = book.Id,
            Title = book.Title,
            Author = book.Author,
            Price = book.Price,
            Stock = book.Stock
        };
        
        patchDoc.ApplyTo(bookToPatch, ModelState);
        
        if (!ModelState.IsValid)
        {
            return ValidationProblem(ModelState);
        }
        
        var updated = await _bookService.UpdateAsync(bookToPatch);
        
        if (!updated)
        {
            return NotFound();
        }
        
        return NoContent();
    }
    
    /// <summary>
    /// Delete a book
    /// </summary>
    [HttpDelete("{id}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> DeleteBook(int id)
    {
        var deleted = await _bookService.DeleteAsync(id);
        
        if (!deleted)
        {
            return NotFound(new { message = $"Book with id {id} not found" });
        }
        
        _logger.LogInformation("Book with id {BookId} deleted", id);
        
        return NoContent();
    }
    
    /// <summary>
    /// Get books by author
    /// </summary>
    [HttpGet("author/{author}")]
    [ProducesResponseType(typeof(IEnumerable<BookDto>), StatusCodes.Status200OK)]
    public async Task<ActionResult<IEnumerable<BookDto>>> GetBooksByAuthor(string author)
    {
        var books = await _bookService.GetByAuthorAsync(author);
        return Ok(books);
    }
    
    /// <summary>
    /// Check book availability
    /// </summary>
    [HttpHead("{id}")]
    [ProducesResponseType(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> CheckBookExists(int id)
    {
        var exists = await _bookService.ExistsAsync(id);
        return exists ? Ok() : NotFound();
    }
}
```