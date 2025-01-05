# CQRS와 MediatR

## CQRS 소개

CQRS(Command Query Responsibility Segregation)는 명령(Command)과 조회(Query)의 책임을 분리하는 아키텍처 패턴입니다. 데이터를 변경하는 작업과 데이터를 읽는 작업을 서로 다른 모델로 처리하여 각각의 요구사항에 최적화된 설계를 가능하게 합니다.

### CQRS의 핵심 개념

1. **Command**: 시스템의 상태를 변경하는 작업 (Create, Update, Delete)
2. **Query**: 시스템의 상태를 조회하는 작업 (Read)
3. **책임 분리**: 읽기와 쓰기 모델의 분리
4. **최적화**: 각 작업에 맞는 최적화 가능

### CQRS의 장점

- 읽기와 쓰기 작업의 독립적인 확장
- 복잡한 도메인 로직의 단순화
- 성능 최적화 용이
- 보안 강화 (읽기/쓰기 권한 분리)

## MediatR 소개

MediatR은 .NET에서 중재자 패턴을 구현한 라이브러리로, CQRS 패턴 구현을 단순화합니다. 요청과 핸들러 간의 느슨한 결합을 제공하여 유지보수성을 향상시킵니다.

### MediatR 설치

```bash
dotnet add package MediatR
dotnet add package MediatR.Extensions.Microsoft.DependencyInjection
```

### 기본 설정

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// MediatR 등록
builder.Services.AddMediatR(cfg => 
    cfg.RegisterServicesFromAssembly(typeof(Program).Assembly));

builder.Services.AddControllers();

var app = builder.Build();

app.MapControllers();
app.Run();
```

## Command 구현

### Command 정의

```csharp
// Commands/CreateProductCommand.cs
using MediatR;

namespace ECommerce.Application.Commands
{
    public class CreateProductCommand : IRequest<CreateProductResponse>
    {
        public string Name { get; set; }
        public string Description { get; set; }
        public decimal Price { get; set; }
        public int Stock { get; set; }
        public string Category { get; set; }
    }
    
    public class CreateProductResponse
    {
        public Guid ProductId { get; set; }
        public string Name { get; set; }
        public decimal Price { get; set; }
        public DateTime CreatedAt { get; set; }
    }
}
```

### Command Handler

```csharp
// Handlers/CreateProductCommandHandler.cs
namespace ECommerce.Application.Handlers
{
    public class CreateProductCommandHandler 
        : IRequestHandler<CreateProductCommand, CreateProductResponse>
    {
        private readonly IProductRepository _repository;
        private readonly IUnitOfWork _unitOfWork;
        private readonly ILogger<CreateProductCommandHandler> _logger;
        
        public CreateProductCommandHandler(
            IProductRepository repository,
            IUnitOfWork unitOfWork,
            ILogger<CreateProductCommandHandler> logger)
        {
            _repository = repository;
            _unitOfWork = unitOfWork;
            _logger = logger;
        }
        
        public async Task<CreateProductResponse> Handle(
            CreateProductCommand request, 
            CancellationToken cancellationToken)
        {
            _logger.LogInformation("Creating new product: {ProductName}", request.Name);
            
            // 비즈니스 규칙 검증
            if (request.Price <= 0)
                throw new ArgumentException("상품 가격은 0보다 커야 합니다.");
                
            if (string.IsNullOrWhiteSpace(request.Name))
                throw new ArgumentException("상품명은 필수입니다.");
            
            // 엔티티 생성
            var product = new Product(
                request.Name,
                request.Description,
                request.Price,
                request.Stock,
                request.Category
            );
            
            // 저장
            await _repository.AddAsync(product);
            await _unitOfWork.SaveChangesAsync();
            
            _logger.LogInformation("Product created successfully: {ProductId}", product.Id);
            
            return new CreateProductResponse
            {
                ProductId = product.Id,
                Name = product.Name,
                Price = product.Price,
                CreatedAt = product.CreatedAt
            };
        }
    }
}
```

### 복잡한 Command 예제

```csharp
// Commands/ProcessOrderCommand.cs
public class ProcessOrderCommand : IRequest<ProcessOrderResponse>
{
    public Guid OrderId { get; set; }
    public PaymentInfo PaymentInfo { get; set; }
    public ShippingAddress ShippingAddress { get; set; }
}

public class ProcessOrderCommandHandler 
    : IRequestHandler<ProcessOrderCommand, ProcessOrderResponse>
{
    private readonly IOrderRepository _orderRepository;
    private readonly IPaymentService _paymentService;
    private readonly IInventoryService _inventoryService;
    private readonly IShippingService _shippingService;
    private readonly IMediator _mediator;
    private readonly IUnitOfWork _unitOfWork;
    
    public ProcessOrderCommandHandler(
        IOrderRepository orderRepository,
        IPaymentService paymentService,
        IInventoryService inventoryService,
        IShippingService shippingService,
        IMediator mediator,
        IUnitOfWork unitOfWork)
    {
        _orderRepository = orderRepository;
        _paymentService = paymentService;
        _inventoryService = inventoryService;
        _shippingService = shippingService;
        _mediator = mediator;
        _unitOfWork = unitOfWork;
    }
    
    public async Task<ProcessOrderResponse> Handle(
        ProcessOrderCommand request, 
        CancellationToken cancellationToken)
    {
        // 1. 주문 조회
        var order = await _orderRepository.GetByIdAsync(request.OrderId);
        if (order == null)
            throw new OrderNotFoundException(request.OrderId);
            
        // 2. 재고 확인 및 예약
        foreach (var item in order.Items)
        {
            var available = await _inventoryService.CheckAndReserveAsync(
                item.ProductId, item.Quantity);
                
            if (!available)
                throw new InsufficientStockException(item.ProductId);
        }
        
        try
        {
            // 3. 결제 처리
            var paymentResult = await _paymentService.ProcessPaymentAsync(
                request.PaymentInfo, order.TotalAmount);
                
            if (!paymentResult.IsSuccessful)
                throw new PaymentFailedException(paymentResult.ErrorMessage);
                
            // 4. 주문 상태 업데이트
            order.MarkAsPaid(paymentResult.TransactionId);
            order.SetShippingAddress(request.ShippingAddress);
            
            // 5. 배송 준비
            var shippingId = await _shippingService.CreateShipmentAsync(order);
            order.MarkAsShipped(shippingId);
            
            // 6. 저장
            await _unitOfWork.SaveChangesAsync();
            
            // 7. 이벤트 발행
            await _mediator.Publish(new OrderProcessedNotification
            {
                OrderId = order.Id,
                CustomerId = order.CustomerId,
                TotalAmount = order.TotalAmount
            });
            
            return new ProcessOrderResponse
            {
                OrderId = order.Id,
                Status = order.Status,
                ShippingId = shippingId,
                EstimatedDeliveryDate = order.EstimatedDeliveryDate
            };
        }
        catch
        {
            // 재고 예약 취소
            await _inventoryService.ReleaseReservationAsync(request.OrderId);
            throw;
        }
    }
}
```

## Query 구현

### Query 정의

```csharp
// Queries/GetProductByIdQuery.cs
public class GetProductByIdQuery : IRequest<ProductDto>
{
    public Guid ProductId { get; set; }
    
    public GetProductByIdQuery(Guid productId)
    {
        ProductId = productId;
    }
}

public class ProductDto
{
    public Guid Id { get; set; }
    public string Name { get; set; }
    public string Description { get; set; }
    public decimal Price { get; set; }
    public int Stock { get; set; }
    public string Category { get; set; }
    public List<string> Images { get; set; }
    public double Rating { get; set; }
    public int ReviewCount { get; set; }
}
```

### Query Handler

```csharp
// Handlers/GetProductByIdQueryHandler.cs
public class GetProductByIdQueryHandler : IRequestHandler<GetProductByIdQuery, ProductDto>
{
    private readonly IProductReadRepository _readRepository;
    private readonly IMemoryCache _cache;
    
    public GetProductByIdQueryHandler(
        IProductReadRepository readRepository,
        IMemoryCache cache)
    {
        _readRepository = readRepository;
        _cache = cache;
    }
    
    public async Task<ProductDto> Handle(
        GetProductByIdQuery request, 
        CancellationToken cancellationToken)
    {
        // 캐시 확인
        var cacheKey = $"product_{request.ProductId}";
        if (_cache.TryGetValue<ProductDto>(cacheKey, out var cachedProduct))
            return cachedProduct;
            
        // 데이터베이스 조회
        var product = await _readRepository.GetProductDetailAsync(request.ProductId);
        
        if (product == null)
            throw new ProductNotFoundException(request.ProductId);
            
        // 캐시 저장
        _cache.Set(cacheKey, product, TimeSpan.FromMinutes(5));
        
        return product;
    }
}
```

### 복잡한 Query 예제

```csharp
// Queries/SearchProductsQuery.cs
public class SearchProductsQuery : IRequest<PagedResult<ProductSearchDto>>
{
    public string SearchTerm { get; set; }
    public string Category { get; set; }
    public decimal? MinPrice { get; set; }
    public decimal? MaxPrice { get; set; }
    public string SortBy { get; set; } = "Relevance";
    public int PageNumber { get; set; } = 1;
    public int PageSize { get; set; } = 20;
}

public class SearchProductsQueryHandler 
    : IRequestHandler<SearchProductsQuery, PagedResult<ProductSearchDto>>
{
    private readonly IElasticsearchClient _elasticClient;
    private readonly IProductReadRepository _readRepository;
    
    public SearchProductsQueryHandler(
        IElasticsearchClient elasticClient,
        IProductReadRepository readRepository)
    {
        _elasticClient = elasticClient;
        _readRepository = readRepository;
    }
    
    public async Task<PagedResult<ProductSearchDto>> Handle(
        SearchProductsQuery request, 
        CancellationToken cancellationToken)
    {
        // Elasticsearch 쿼리 구성
        var searchDescriptor = new SearchDescriptor<ProductSearchDto>()
            .Index("products")
            .From((request.PageNumber - 1) * request.PageSize)
            .Size(request.PageSize);
            
        // 검색어 적용
        if (!string.IsNullOrWhiteSpace(request.SearchTerm))
        {
            searchDescriptor.Query(q => q
                .MultiMatch(m => m
                    .Fields(f => f
                        .Field(p => p.Name, boost: 2)
                        .Field(p => p.Description)
                        .Field(p => p.Category))
                    .Query(request.SearchTerm)
                    .Type(TextQueryType.BestFields)));
        }
        
        // 필터 적용
        var filters = new List<Func<QueryContainerDescriptor<ProductSearchDto>, QueryContainer>>();
        
        if (!string.IsNullOrWhiteSpace(request.Category))
            filters.Add(f => f.Term(t => t.Category, request.Category));
            
        if (request.MinPrice.HasValue)
            filters.Add(f => f.Range(r => r.Field(p => p.Price).GreaterThanOrEquals(request.MinPrice.Value)));
            
        if (request.MaxPrice.HasValue)
            filters.Add(f => f.Range(r => r.Field(p => p.Price).LessThanOrEquals(request.MaxPrice.Value)));
            
        if (filters.Any())
        {
            searchDescriptor.PostFilter(pf => pf.Bool(b => b.Must(filters)));
        }
        
        // 정렬 적용
        switch (request.SortBy?.ToLower())
        {
            case "price_asc":
                searchDescriptor.Sort(s => s.Ascending(p => p.Price));
                break;
            case "price_desc":
                searchDescriptor.Sort(s => s.Descending(p => p.Price));
                break;
            case "newest":
                searchDescriptor.Sort(s => s.Descending(p => p.CreatedAt));
                break;
            default:
                searchDescriptor.Sort(s => s.Descending("_score"));
                break;
        }
        
        // 검색 실행
        var searchResponse = await _elasticClient.SearchAsync<ProductSearchDto>(searchDescriptor);
        
        return new PagedResult<ProductSearchDto>
        {
            Items = searchResponse.Documents.ToList(),
            PageNumber = request.PageNumber,
            PageSize = request.PageSize,
            TotalCount = (int)searchResponse.Total,
            TotalPages = (int)Math.Ceiling(searchResponse.Total / (double)request.PageSize)
        };
    }
}
```

## 파이프라인 동작

### 검증 동작

```csharp
// Behaviors/ValidationBehavior.cs
public class ValidationBehavior<TRequest, TResponse> 
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;
    
    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators)
    {
        _validators = validators;
    }
    
    public async Task<TResponse> Handle(
        TRequest request, 
        RequestHandlerDelegate<TResponse> next, 
        CancellationToken cancellationToken)
    {
        if (_validators.Any())
        {
            var context = new ValidationContext<TRequest>(request);
            
            var validationResults = await Task.WhenAll(
                _validators.Select(v => v.ValidateAsync(context, cancellationToken)));
                
            var failures = validationResults
                .SelectMany(r => r.Errors)
                .Where(f => f != null)
                .ToList();
                
            if (failures.Count != 0)
                throw new ValidationException(failures);
        }
        
        return await next();
    }
}

// Validators/CreateProductCommandValidator.cs
public class CreateProductCommandValidator : AbstractValidator<CreateProductCommand>
{
    public CreateProductCommandValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("상품명은 필수입니다.")
            .MaximumLength(200).WithMessage("상품명은 200자를 초과할 수 없습니다.");
            
        RuleFor(x => x.Price)
            .GreaterThan(0).WithMessage("가격은 0보다 커야 합니다.")
            .LessThanOrEqualTo(1000000).WithMessage("가격은 1,000,000원을 초과할 수 없습니다.");
            
        RuleFor(x => x.Stock)
            .GreaterThanOrEqualTo(0).WithMessage("재고는 0 이상이어야 합니다.");
            
        RuleFor(x => x.Category)
            .NotEmpty().WithMessage("카테고리는 필수입니다.")
            .Must(BeValidCategory).WithMessage("유효하지 않은 카테고리입니다.");
    }
    
    private bool BeValidCategory(string category)
    {
        var validCategories = new[] { "Electronics", "Clothing", "Books", "Home" };
        return validCategories.Contains(category);
    }
}
```

### 로깅 동작

```csharp
// Behaviors/LoggingBehavior.cs
public class LoggingBehavior<TRequest, TResponse> 
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly ILogger<LoggingBehavior<TRequest, TResponse>> _logger;
    
    public LoggingBehavior(ILogger<LoggingBehavior<TRequest, TResponse>> logger)
    {
        _logger = logger;
    }
    
    public async Task<TResponse> Handle(
        TRequest request, 
        RequestHandlerDelegate<TResponse> next, 
        CancellationToken cancellationToken)
    {
        var requestName = typeof(TRequest).Name;
        var requestGuid = Guid.NewGuid().ToString();
        
        _logger.LogInformation(
            "Handling {RequestName} [{RequestGuid}] {@Request}", 
            requestName, requestGuid, request);
            
        var stopwatch = Stopwatch.StartNew();
        
        try
        {
            var response = await next();
            
            stopwatch.Stop();
            
            _logger.LogInformation(
                "Handled {RequestName} [{RequestGuid}] in {ElapsedMilliseconds}ms", 
                requestName, requestGuid, stopwatch.ElapsedMilliseconds);
                
            return response;
        }
        catch (Exception ex)
        {
            stopwatch.Stop();
            
            _logger.LogError(ex,
                "Error handling {RequestName} [{RequestGuid}] after {ElapsedMilliseconds}ms", 
                requestName, requestGuid, stopwatch.ElapsedMilliseconds);
                
            throw;
        }
    }
}
```

## 통지(Notification) 구현

### 통지 정의

```csharp
// Notifications/OrderProcessedNotification.cs
public class OrderProcessedNotification : INotification
{
    public Guid OrderId { get; set; }
    public Guid CustomerId { get; set; }
    public decimal TotalAmount { get; set; }
    public DateTime ProcessedAt { get; set; } = DateTime.UtcNow;
}
```

### 통지 핸들러

```csharp
// Handlers/SendOrderConfirmationEmailHandler.cs
public class SendOrderConfirmationEmailHandler 
    : INotificationHandler<OrderProcessedNotification>
{
    private readonly IEmailService _emailService;
    private readonly ICustomerRepository _customerRepository;
    
    public SendOrderConfirmationEmailHandler(
        IEmailService emailService,
        ICustomerRepository customerRepository)
    {
        _emailService = emailService;
        _customerRepository = customerRepository;
    }
    
    public async Task Handle(
        OrderProcessedNotification notification, 
        CancellationToken cancellationToken)
    {
        var customer = await _customerRepository.GetByIdAsync(notification.CustomerId);
        
        await _emailService.SendAsync(new EmailMessage
        {
            To = customer.Email,
            Subject = "주문 확인",
            Body = $"주문번호 {notification.OrderId}가 성공적으로 처리되었습니다."
        });
    }
}

// Handlers/UpdateCustomerStatisticsHandler.cs
public class UpdateCustomerStatisticsHandler 
    : INotificationHandler<OrderProcessedNotification>
{
    private readonly IStatisticsService _statisticsService;
    
    public UpdateCustomerStatisticsHandler(IStatisticsService statisticsService)
    {
        _statisticsService = statisticsService;
    }
    
    public async Task Handle(
        OrderProcessedNotification notification, 
        CancellationToken cancellationToken)
    {
        await _statisticsService.UpdateCustomerPurchaseStatsAsync(
            notification.CustomerId, 
            notification.TotalAmount);
    }
}
```

## Controller 통합

```csharp
// Controllers/ProductsController.cs
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IMediator _mediator;
    
    public ProductsController(IMediator mediator)
    {
        _mediator = mediator;
    }
    
    [HttpPost]
    public async Task<ActionResult<CreateProductResponse>> CreateProduct(
        [FromBody] CreateProductCommand command)
    {
        var response = await _mediator.Send(command);
        return CreatedAtAction(
            nameof(GetProduct), 
            new { id = response.ProductId }, 
            response);
    }
    
    [HttpGet("{id}")]
    public async Task<ActionResult<ProductDto>> GetProduct(Guid id)
    {
        var query = new GetProductByIdQuery(id);
        var product = await _mediator.Send(query);
        return Ok(product);
    }
    
    [HttpGet]
    public async Task<ActionResult<PagedResult<ProductSearchDto>>> SearchProducts(
        [FromQuery] SearchProductsQuery query)
    {
        var results = await _mediator.Send(query);
        return Ok(results);
    }
    
    [HttpPut("{id}")]
    public async Task<ActionResult> UpdateProduct(
        Guid id, 
        [FromBody] UpdateProductCommand command)
    {
        if (id != command.ProductId)
            return BadRequest("Product ID mismatch");
            
        await _mediator.Send(command);
        return NoContent();
    }
    
    [HttpDelete("{id}")]
    public async Task<ActionResult> DeleteProduct(Guid id)
    {
        await _mediator.Send(new DeleteProductCommand { ProductId = id });
        return NoContent();
    }
}
```

## 테스트

### 핸들러 단위 테스트

```csharp
public class CreateProductCommandHandlerTests
{
    private readonly Mock<IProductRepository> _mockRepository;
    private readonly Mock<IUnitOfWork> _mockUnitOfWork;
    private readonly CreateProductCommandHandler _handler;
    
    public CreateProductCommandHandlerTests()
    {
        _mockRepository = new Mock<IProductRepository>();
        _mockUnitOfWork = new Mock<IUnitOfWork>();
        var mockLogger = new Mock<ILogger<CreateProductCommandHandler>>();
        
        _handler = new CreateProductCommandHandler(
            _mockRepository.Object,
            _mockUnitOfWork.Object,
            mockLogger.Object);
    }
    
    [Fact]
    public async Task Handle_ValidCommand_CreatesProduct()
    {
        // Arrange
        var command = new CreateProductCommand
        {
            Name = "Test Product",
            Description = "Test Description",
            Price = 99.99m,
            Stock = 10,
            Category = "Electronics"
        };
        
        _mockRepository
            .Setup(x => x.AddAsync(It.IsAny<Product>()))
            .Returns(Task.CompletedTask);
            
        _mockUnitOfWork
            .Setup(x => x.SaveChangesAsync())
            .ReturnsAsync(1);
            
        // Act
        var result = await _handler.Handle(command, CancellationToken.None);
        
        // Assert
        Assert.NotNull(result);
        Assert.Equal(command.Name, result.Name);
        Assert.Equal(command.Price, result.Price);
        
        _mockRepository.Verify(x => x.AddAsync(It.IsAny<Product>()), Times.Once);
        _mockUnitOfWork.Verify(x => x.SaveChangesAsync(), Times.Once);
    }
}
```

## 모범 사례

### 1. 명명 규칙
- Command: 동사 + 명사 + Command (예: CreateProductCommand)
- Query: Get + 명사 + Query (예: GetProductByIdQuery)
- Handler: Command/Query 이름 + Handler

### 2. 단일 책임 원칙
- 각 핸들러는 하나의 작업만 처리
- 복잡한 로직은 도메인 서비스로 분리

### 3. 쿼리 최적화
- 읽기 전용 리포지토리 사용
- 프로젝션을 통한 필요한 데이터만 조회
- 캐싱 적극 활용

### 4. 트랜잭션 관리
- Command 핸들러에서만 트랜잭션 사용
- Query는 읽기 전용 트랜잭션 사용

## 마무리

CQRS와 MediatR을 함께 사용하면 명령과 조회를 명확히 분리하고, 각각의 요구사항에 최적화된 설계가 가능합니다. MediatR의 파이프라인 동작을 통해 횡단 관심사를 깔끔하게 처리할 수 있으며, 느슨한 결합으로 유지보수성이 향상됩니다.