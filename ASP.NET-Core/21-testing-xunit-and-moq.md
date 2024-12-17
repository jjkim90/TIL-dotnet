# Testing - xUnit and Moq

## 단위 테스트 개요

단위 테스트는 소프트웨어의 가장 작은 테스트 가능한 부분을 검증하는 자동화된 테스트입니다. ASP.NET Core에서는 xUnit이 가장 널리 사용되는 테스트 프레임워크이며, Moq는 모킹(Mocking)을 위한 표준 라이브러리입니다.

### 단위 테스트의 중요성
- **빠른 피드백**: 코드 변경사항에 대한 즉각적인 검증
- **리팩토링 안정성**: 기능 변경 없이 코드 구조 개선 가능
- **문서화**: 테스트 코드가 실제 사용 예제 역할
- **디버깅 용이성**: 문제 발생 지점을 정확히 파악
- **설계 개선**: 테스트 가능한 코드는 일반적으로 더 나은 설계

## xUnit 기초

### 프로젝트 설정
```xml
<!-- TestProject.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <IsPackable>false</IsPackable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.8.0" />
    <PackageReference Include="xunit" Version="2.6.1" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.5.3">
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>
    <PackageReference Include="Moq" Version="4.20.69" />
    <PackageReference Include="FluentAssertions" Version="6.12.0" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\MyApi\MyApi.csproj" />
  </ItemGroup>
</Project>
```

### 기본 테스트 구조
```csharp
public class CalculatorTests
{
    [Fact]
    public void Add_TwoPositiveNumbers_ReturnsCorrectSum()
    {
        // Arrange
        var calculator = new Calculator();
        int a = 5;
        int b = 3;
        
        // Act
        var result = calculator.Add(a, b);
        
        // Assert
        Assert.Equal(8, result);
    }
    
    [Theory]
    [InlineData(1, 2, 3)]
    [InlineData(-1, -1, -2)]
    [InlineData(0, 0, 0)]
    [InlineData(int.MaxValue, 1, int.MinValue)] // 오버플로우 테스트
    public void Add_VariousInputs_ReturnsExpectedResult(int a, int b, int expected)
    {
        // Arrange
        var calculator = new Calculator();
        
        // Act
        var result = calculator.Add(a, b);
        
        // Assert
        Assert.Equal(expected, result);
    }
}
```

### xUnit 특성(Attributes)
```csharp
public class XUnitFeaturesTests
{
    // 기본 테스트
    [Fact]
    public void Fact_SimpleTest()
    {
        Assert.True(1 + 1 == 2);
    }
    
    // 매개변수화된 테스트
    [Theory]
    [InlineData("hello", 5)]
    [InlineData("", 0)]
    [InlineData(null, 0)]
    public void Theory_WithInlineData(string input, int expectedLength)
    {
        var length = input?.Length ?? 0;
        Assert.Equal(expectedLength, length);
    }
    
    // 클래스 데이터 사용
    [Theory]
    [ClassData(typeof(CalculatorTestData))]
    public void Theory_WithClassData(int a, int b, int expected)
    {
        var result = a + b;
        Assert.Equal(expected, result);
    }
    
    // 멤버 데이터 사용
    [Theory]
    [MemberData(nameof(GetTestData))]
    public void Theory_WithMemberData(int a, int b, int expected)
    {
        var result = a * b;
        Assert.Equal(expected, result);
    }
    
    public static IEnumerable<object[]> GetTestData()
    {
        yield return new object[] { 2, 3, 6 };
        yield return new object[] { 0, 5, 0 };
        yield return new object[] { -2, 4, -8 };
    }
    
    // 특정 조건에서만 실행
    [Fact(Skip = "Not implemented yet")]
    public void SkippedTest()
    {
        // 이 테스트는 건너뜀
    }
    
    [Fact]
    [Trait("Category", "Integration")]
    public void TestWithTrait()
    {
        // 카테고리로 그룹화된 테스트
    }
}

public class CalculatorTestData : IEnumerable<object[]>
{
    public IEnumerator<object[]> GetEnumerator()
    {
        yield return new object[] { 1, 2, 3 };
        yield return new object[] { -1, 1, 0 };
        yield return new object[] { 0, 0, 0 };
    }
    
    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}
```

## Controller 테스트

### 기본 Controller 테스트
```csharp
public class ProductControllerTests
{
    private readonly Mock<IProductService> _productServiceMock;
    private readonly Mock<ILogger<ProductController>> _loggerMock;
    private readonly ProductController _controller;
    
    public ProductControllerTests()
    {
        _productServiceMock = new Mock<IProductService>();
        _loggerMock = new Mock<ILogger<ProductController>>();
        _controller = new ProductController(_productServiceMock.Object, _loggerMock.Object);
    }
    
    [Fact]
    public async Task GetProducts_ReturnsOkResult_WithListOfProducts()
    {
        // Arrange
        var products = new List<Product>
        {
            new Product { Id = 1, Name = "Product 1", Price = 10.00m },
            new Product { Id = 2, Name = "Product 2", Price = 20.00m }
        };
        
        _productServiceMock.Setup(x => x.GetAllProductsAsync())
            .ReturnsAsync(products);
        
        // Act
        var result = await _controller.GetProducts();
        
        // Assert
        var okResult = Assert.IsType<OkObjectResult>(result.Result);
        var returnedProducts = Assert.IsAssignableFrom<IEnumerable<Product>>(okResult.Value);
        Assert.Equal(2, returnedProducts.Count());
    }
    
    [Fact]
    public async Task GetProduct_WithValidId_ReturnsOkResult()
    {
        // Arrange
        var productId = 1;
        var product = new Product { Id = productId, Name = "Test Product", Price = 15.00m };
        
        _productServiceMock.Setup(x => x.GetProductByIdAsync(productId))
            .ReturnsAsync(product);
        
        // Act
        var result = await _controller.GetProduct(productId);
        
        // Assert
        var okResult = Assert.IsType<OkObjectResult>(result.Result);
        var returnedProduct = Assert.IsType<Product>(okResult.Value);
        Assert.Equal(productId, returnedProduct.Id);
    }
    
    [Fact]
    public async Task GetProduct_WithInvalidId_ReturnsNotFound()
    {
        // Arrange
        var productId = 999;
        _productServiceMock.Setup(x => x.GetProductByIdAsync(productId))
            .ReturnsAsync((Product)null);
        
        // Act
        var result = await _controller.GetProduct(productId);
        
        // Assert
        Assert.IsType<NotFoundResult>(result.Result);
    }
    
    [Fact]
    public async Task CreateProduct_WithValidModel_ReturnsCreatedResult()
    {
        // Arrange
        var createDto = new CreateProductDto
        {
            Name = "New Product",
            Price = 25.00m,
            Description = "Test description"
        };
        
        var createdProduct = new Product
        {
            Id = 3,
            Name = createDto.Name,
            Price = createDto.Price,
            Description = createDto.Description
        };
        
        _productServiceMock.Setup(x => x.CreateProductAsync(It.IsAny<CreateProductDto>()))
            .ReturnsAsync(createdProduct);
        
        // Act
        var result = await _controller.CreateProduct(createDto);
        
        // Assert
        var createdResult = Assert.IsType<CreatedAtActionResult>(result.Result);
        var returnedProduct = Assert.IsType<Product>(createdResult.Value);
        Assert.Equal(createdProduct.Id, returnedProduct.Id);
        Assert.Equal("GetProduct", createdResult.ActionName);
    }
    
    [Fact]
    public async Task CreateProduct_WithInvalidModel_ReturnsBadRequest()
    {
        // Arrange
        _controller.ModelState.AddModelError("Name", "Name is required");
        var createDto = new CreateProductDto();
        
        // Act
        var result = await _controller.CreateProduct(createDto);
        
        // Assert
        Assert.IsType<BadRequestObjectResult>(result.Result);
    }
}
```

### Controller 액션 필터 테스트
```csharp
public class ValidationFilterTests
{
    [Fact]
    public void OnActionExecuting_WithInvalidModel_ReturnsBadRequest()
    {
        // Arrange
        var filter = new ValidationActionFilter();
        var modelState = new ModelStateDictionary();
        modelState.AddModelError("Name", "Name is required");
        
        var actionContext = new ActionContext(
            new DefaultHttpContext(),
            new RouteData(),
            new ActionDescriptor(),
            modelState);
        
        var actionExecutingContext = new ActionExecutingContext(
            actionContext,
            new List<IFilterMetadata>(),
            new Dictionary<string, object>(),
            new object());
        
        // Act
        filter.OnActionExecuting(actionExecutingContext);
        
        // Assert
        Assert.IsType<BadRequestObjectResult>(actionExecutingContext.Result);
    }
}
```

## Service Layer 테스트

### 비즈니스 로직 테스트
```csharp
public class OrderServiceTests
{
    private readonly Mock<IOrderRepository> _orderRepositoryMock;
    private readonly Mock<IProductRepository> _productRepositoryMock;
    private readonly Mock<IEmailService> _emailServiceMock;
    private readonly Mock<ILogger<OrderService>> _loggerMock;
    private readonly OrderService _orderService;
    
    public OrderServiceTests()
    {
        _orderRepositoryMock = new Mock<IOrderRepository>();
        _productRepositoryMock = new Mock<IProductRepository>();
        _emailServiceMock = new Mock<IEmailService>();
        _loggerMock = new Mock<ILogger<OrderService>>();
        
        _orderService = new OrderService(
            _orderRepositoryMock.Object,
            _productRepositoryMock.Object,
            _emailServiceMock.Object,
            _loggerMock.Object);
    }
    
    [Fact]
    public async Task CreateOrder_WithValidData_CreatesOrderSuccessfully()
    {
        // Arrange
        var customerId = "user123";
        var orderItems = new List<OrderItemDto>
        {
            new OrderItemDto { ProductId = 1, Quantity = 2 },
            new OrderItemDto { ProductId = 2, Quantity = 1 }
        };
        
        var product1 = new Product { Id = 1, Name = "Product 1", Price = 10.00m, Stock = 10 };
        var product2 = new Product { Id = 2, Name = "Product 2", Price = 20.00m, Stock = 5 };
        
        _productRepositoryMock.Setup(x => x.GetByIdAsync(1))
            .ReturnsAsync(product1);
        _productRepositoryMock.Setup(x => x.GetByIdAsync(2))
            .ReturnsAsync(product2);
        
        _orderRepositoryMock.Setup(x => x.AddAsync(It.IsAny<Order>()))
            .ReturnsAsync((Order order) => 
            {
                order.Id = 123;
                return order;
            });
        
        // Act
        var result = await _orderService.CreateOrderAsync(customerId, orderItems);
        
        // Assert
        Assert.NotNull(result);
        Assert.Equal(123, result.Id);
        Assert.Equal(40.00m, result.TotalAmount); // (10*2) + (20*1)
        Assert.Equal(OrderStatus.Pending, result.Status);
        
        // Verify repository calls
        _orderRepositoryMock.Verify(x => x.AddAsync(It.IsAny<Order>()), Times.Once);
        _productRepositoryMock.Verify(x => x.UpdateAsync(It.IsAny<Product>()), Times.Exactly(2));
        
        // Verify email was sent
        _emailServiceMock.Verify(x => x.SendOrderConfirmationAsync(
            It.IsAny<string>(), 
            It.IsAny<Order>()), Times.Once);
    }
    
    [Fact]
    public async Task CreateOrder_WithInsufficientStock_ThrowsException()
    {
        // Arrange
        var customerId = "user123";
        var orderItems = new List<OrderItemDto>
        {
            new OrderItemDto { ProductId = 1, Quantity = 100 } // 재고보다 많은 수량
        };
        
        var product = new Product { Id = 1, Name = "Product 1", Price = 10.00m, Stock = 5 };
        
        _productRepositoryMock.Setup(x => x.GetByIdAsync(1))
            .ReturnsAsync(product);
        
        // Act & Assert
        var exception = await Assert.ThrowsAsync<InsufficientStockException>(
            () => _orderService.CreateOrderAsync(customerId, orderItems));
        
        Assert.Equal("Insufficient stock for product: Product 1", exception.Message);
        
        // Verify no order was created
        _orderRepositoryMock.Verify(x => x.AddAsync(It.IsAny<Order>()), Times.Never);
    }
    
    [Fact]
    public async Task ProcessPayment_WithValidPayment_UpdatesOrderStatus()
    {
        // Arrange
        var orderId = 123;
        var order = new Order
        {
            Id = orderId,
            Status = OrderStatus.Pending,
            TotalAmount = 50.00m
        };
        
        var paymentInfo = new PaymentInfo
        {
            Amount = 50.00m,
            PaymentMethod = PaymentMethod.CreditCard
        };
        
        _orderRepositoryMock.Setup(x => x.GetByIdAsync(orderId))
            .ReturnsAsync(order);
        
        // Act
        await _orderService.ProcessPaymentAsync(orderId, paymentInfo);
        
        // Assert
        Assert.Equal(OrderStatus.Paid, order.Status);
        Assert.NotNull(order.PaymentDate);
        
        _orderRepositoryMock.Verify(x => x.UpdateAsync(order), Times.Once);
    }
}
```

### 복잡한 비즈니스 규칙 테스트
```csharp
public class PricingServiceTests
{
    private readonly Mock<IDiscountRepository> _discountRepositoryMock;
    private readonly Mock<ICustomerRepository> _customerRepositoryMock;
    private readonly PricingService _pricingService;
    
    public PricingServiceTests()
    {
        _discountRepositoryMock = new Mock<IDiscountRepository>();
        _customerRepositoryMock = new Mock<ICustomerRepository>();
        _pricingService = new PricingService(
            _discountRepositoryMock.Object,
            _customerRepositoryMock.Object);
    }
    
    [Theory]
    [InlineData(100, CustomerType.Regular, 0, 100)]          // 일반 고객, 할인 없음
    [InlineData(100, CustomerType.Silver, 5, 95)]           // 실버 고객, 5% 할인
    [InlineData(100, CustomerType.Gold, 10, 90)]            // 골드 고객, 10% 할인
    [InlineData(100, CustomerType.Platinum, 15, 85)]        // 플래티넘 고객, 15% 할인
    public async Task CalculatePrice_WithCustomerType_AppliesCorrectDiscount(
        decimal basePrice, CustomerType customerType, decimal discountPercentage, decimal expectedPrice)
    {
        // Arrange
        var customerId = "customer123";
        var customer = new Customer
        {
            Id = customerId,
            Type = customerType
        };
        
        _customerRepositoryMock.Setup(x => x.GetByIdAsync(customerId))
            .ReturnsAsync(customer);
        
        // Act
        var result = await _pricingService.CalculateFinalPriceAsync(customerId, basePrice);
        
        // Assert
        Assert.Equal(expectedPrice, result);
    }
    
    [Fact]
    public async Task CalculatePrice_WithMultipleDiscounts_AppliesHighestDiscount()
    {
        // Arrange
        var customerId = "customer123";
        var basePrice = 100m;
        
        var customer = new Customer
        {
            Id = customerId,
            Type = CustomerType.Gold // 10% 할인
        };
        
        var discounts = new List<Discount>
        {
            new Discount { Code = "SUMMER20", Percentage = 20, IsActive = true },
            new Discount { Code = "LOYAL15", Percentage = 15, IsActive = true }
        };
        
        _customerRepositoryMock.Setup(x => x.GetByIdAsync(customerId))
            .ReturnsAsync(customer);
        
        _discountRepositoryMock.Setup(x => x.GetActiveDiscountsForCustomerAsync(customerId))
            .ReturnsAsync(discounts);
        
        // Act
        var result = await _pricingService.CalculateFinalPriceAsync(
            customerId, basePrice, applyAllDiscounts: true);
        
        // Assert
        Assert.Equal(80m, result); // 20% 할인 적용 (가장 높은 할인율)
    }
}
```

## Moq 고급 기능

### Setup과 Verification
```csharp
public class AdvancedMoqTests
{
    [Fact]
    public async Task Moq_SetupSequence_ReturnsValuesInOrder()
    {
        // Arrange
        var mock = new Mock<IDataService>();
        
        mock.SetupSequence(x => x.GetNextValueAsync())
            .ReturnsAsync(1)
            .ReturnsAsync(2)
            .ReturnsAsync(3)
            .ThrowsAsync(new InvalidOperationException("No more values"));
        
        // Act & Assert
        Assert.Equal(1, await mock.Object.GetNextValueAsync());
        Assert.Equal(2, await mock.Object.GetNextValueAsync());
        Assert.Equal(3, await mock.Object.GetNextValueAsync());
        await Assert.ThrowsAsync<InvalidOperationException>(
            () => mock.Object.GetNextValueAsync());
    }
    
    [Fact]
    public void Moq_CallbackAndReturns_ExecutesCallback()
    {
        // Arrange
        var mock = new Mock<ICalculator>();
        var callbackExecuted = false;
        int capturedA = 0, capturedB = 0;
        
        mock.Setup(x => x.Add(It.IsAny<int>(), It.IsAny<int>()))
            .Callback<int, int>((a, b) =>
            {
                callbackExecuted = true;
                capturedA = a;
                capturedB = b;
            })
            .Returns<int, int>((a, b) => a + b);
        
        // Act
        var result = mock.Object.Add(5, 3);
        
        // Assert
        Assert.True(callbackExecuted);
        Assert.Equal(5, capturedA);
        Assert.Equal(3, capturedB);
        Assert.Equal(8, result);
    }
    
    [Fact]
    public void Moq_ConditionalSetup_ReturnsBasedOnInput()
    {
        // Arrange
        var mock = new Mock<IUserService>();
        
        mock.Setup(x => x.GetUserName(It.Is<int>(id => id > 0)))
            .Returns<int>(id => $"User{id}");
        
        mock.Setup(x => x.GetUserName(It.Is<int>(id => id <= 0)))
            .Returns("Invalid User");
        
        // Act & Assert
        Assert.Equal("User5", mock.Object.GetUserName(5));
        Assert.Equal("Invalid User", mock.Object.GetUserName(0));
        Assert.Equal("Invalid User", mock.Object.GetUserName(-1));
    }
    
    [Fact]
    public void Moq_VerifyWithTimes_ChecksMethodCallCount()
    {
        // Arrange
        var mock = new Mock<ILogger>();
        var service = new ServiceWithLogging(mock.Object);
        
        // Act
        service.PerformOperation();
        service.PerformOperation();
        service.PerformCriticalOperation();
        
        // Assert
        mock.Verify(x => x.Log(LogLevel.Information, It.IsAny<string>()), Times.Exactly(2));
        mock.Verify(x => x.Log(LogLevel.Warning, It.IsAny<string>()), Times.Once);
        mock.Verify(x => x.Log(LogLevel.Error, It.IsAny<string>()), Times.Never);
    }
    
    [Fact]
    public void Moq_PropertySetup_ConfiguresProperties()
    {
        // Arrange
        var mock = new Mock<IConfiguration>();
        
        mock.Setup(x => x.ConnectionString)
            .Returns("Server=localhost;Database=TestDb");
        
        mock.SetupProperty(x => x.Timeout, 30);
        
        // Act
        var connectionString = mock.Object.ConnectionString;
        mock.Object.Timeout = 60;
        
        // Assert
        Assert.Equal("Server=localhost;Database=TestDb", connectionString);
        Assert.Equal(60, mock.Object.Timeout);
    }
}
```

### 고급 Matching
```csharp
public class AdvancedMatchingTests
{
    [Fact]
    public async Task Moq_CustomMatcher_ValidatesComplexConditions()
    {
        // Arrange
        var mock = new Mock<IOrderService>();
        
        mock.Setup(x => x.ProcessOrderAsync(
            It.Is<Order>(o => o.TotalAmount > 100 && o.Items.Count > 2)))
            .ReturnsAsync(true);
        
        mock.Setup(x => x.ProcessOrderAsync(
            It.Is<Order>(o => o.TotalAmount <= 100 || o.Items.Count <= 2)))
            .ReturnsAsync(false);
        
        var largeOrder = new Order
        {
            TotalAmount = 150,
            Items = new List<OrderItem> { new(), new(), new() }
        };
        
        var smallOrder = new Order
        {
            TotalAmount = 50,
            Items = new List<OrderItem> { new() }
        };
        
        // Act & Assert
        Assert.True(await mock.Object.ProcessOrderAsync(largeOrder));
        Assert.False(await mock.Object.ProcessOrderAsync(smallOrder));
    }
    
    [Fact]
    public void Moq_CaptureArguments_StoresPassedValues()
    {
        // Arrange
        var mock = new Mock<IEmailService>();
        var capturedEmails = new List<string>();
        var capturedSubjects = new List<string>();
        
        mock.Setup(x => x.SendEmailAsync(
            Capture.In(capturedEmails),
            Capture.In(capturedSubjects),
            It.IsAny<string>()))
            .ReturnsAsync(true);
        
        // Act
        mock.Object.SendEmailAsync("user1@example.com", "Subject 1", "Body 1");
        mock.Object.SendEmailAsync("user2@example.com", "Subject 2", "Body 2");
        
        // Assert
        Assert.Equal(2, capturedEmails.Count);
        Assert.Contains("user1@example.com", capturedEmails);
        Assert.Contains("Subject 2", capturedSubjects);
    }
}
```

## 테스트 더블 패턴

### Stub, Mock, Fake 구현
```csharp
// Stub - 미리 정의된 답변만 반환
public class StubUserRepository : IUserRepository
{
    private readonly Dictionary<int, User> _users = new()
    {
        { 1, new User { Id = 1, Name = "Test User 1" } },
        { 2, new User { Id = 2, Name = "Test User 2" } }
    };
    
    public Task<User> GetByIdAsync(int id)
    {
        return Task.FromResult(_users.GetValueOrDefault(id));
    }
    
    public Task<IEnumerable<User>> GetAllAsync()
    {
        return Task.FromResult(_users.Values.AsEnumerable());
    }
    
    // 다른 메서드들은 NotImplementedException
    public Task AddAsync(User user) => throw new NotImplementedException();
    public Task UpdateAsync(User user) => throw new NotImplementedException();
    public Task DeleteAsync(int id) => throw new NotImplementedException();
}

// Fake - 실제 구현의 단순화된 버전
public class FakeEmailService : IEmailService
{
    public List<SentEmail> SentEmails { get; } = new();
    
    public Task<bool> SendEmailAsync(string to, string subject, string body)
    {
        SentEmails.Add(new SentEmail
        {
            To = to,
            Subject = subject,
            Body = body,
            SentAt = DateTime.UtcNow
        });
        
        return Task.FromResult(true);
    }
    
    public bool WasEmailSent(string to, string subject)
    {
        return SentEmails.Any(e => e.To == to && e.Subject == subject);
    }
}

// 테스트에서 사용
public class OrderServiceWithFakeTests
{
    [Fact]
    public async Task CreateOrder_SendsConfirmationEmail()
    {
        // Arrange
        var fakeEmailService = new FakeEmailService();
        var orderService = new OrderService(
            new StubOrderRepository(),
            new StubProductRepository(),
            fakeEmailService,
            new NullLogger<OrderService>());
        
        // Act
        var order = await orderService.CreateOrderAsync("customer@example.com", new List<OrderItemDto>());
        
        // Assert
        Assert.True(fakeEmailService.WasEmailSent(
            "customer@example.com", 
            "Order Confirmation"));
    }
}
```

## 테스트 조직화

### Test Fixtures와 Collection
```csharp
// Fixture - 테스트 간 공유되는 컨텍스트
public class DatabaseFixture : IDisposable
{
    public DatabaseFixture()
    {
        var options = new DbContextOptionsBuilder<ApplicationDbContext>()
            .UseInMemoryDatabase(databaseName: Guid.NewGuid().ToString())
            .Options;
        
        DbContext = new ApplicationDbContext(options);
        
        // 테스트 데이터 시드
        SeedTestData();
    }
    
    public ApplicationDbContext DbContext { get; }
    
    private void SeedTestData()
    {
        DbContext.Products.AddRange(
            new Product { Id = 1, Name = "Product 1", Price = 10.00m },
            new Product { Id = 2, Name = "Product 2", Price = 20.00m }
        );
        
        DbContext.SaveChanges();
    }
    
    public void Dispose()
    {
        DbContext?.Dispose();
    }
}

// Collection Definition
[CollectionDefinition("Database collection")]
public class DatabaseCollection : ICollectionFixture<DatabaseFixture>
{
    // 이 클래스는 비어있음 - xUnit이 사용
}

// Collection 사용
[Collection("Database collection")]
public class ProductRepositoryTests
{
    private readonly DatabaseFixture _fixture;
    
    public ProductRepositoryTests(DatabaseFixture fixture)
    {
        _fixture = fixture;
    }
    
    [Fact]
    public async Task GetAllProducts_ReturnsSeededProducts()
    {
        // Arrange
        var repository = new ProductRepository(_fixture.DbContext);
        
        // Act
        var products = await repository.GetAllAsync();
        
        // Assert
        Assert.Equal(2, products.Count());
    }
}
```

### 테스트 빌더 패턴
```csharp
public class OrderBuilder
{
    private Order _order;
    
    public OrderBuilder()
    {
        _order = new Order
        {
            Id = 1,
            CustomerId = "default-customer",
            OrderDate = DateTime.UtcNow,
            Status = OrderStatus.Pending,
            Items = new List<OrderItem>()
        };
    }
    
    public OrderBuilder WithId(int id)
    {
        _order.Id = id;
        return this;
    }
    
    public OrderBuilder WithCustomer(string customerId)
    {
        _order.CustomerId = customerId;
        return this;
    }
    
    public OrderBuilder WithStatus(OrderStatus status)
    {
        _order.Status = status;
        return this;
    }
    
    public OrderBuilder WithItem(int productId, int quantity, decimal price)
    {
        _order.Items.Add(new OrderItem
        {
            ProductId = productId,
            Quantity = quantity,
            UnitPrice = price
        });
        return this;
    }
    
    public OrderBuilder WithItems(params OrderItem[] items)
    {
        _order.Items.AddRange(items);
        return this;
    }
    
    public Order Build()
    {
        _order.TotalAmount = _order.Items.Sum(i => i.Quantity * i.UnitPrice);
        return _order;
    }
    
    public static implicit operator Order(OrderBuilder builder)
    {
        return builder.Build();
    }
}

// 사용 예제
public class OrderBuilderTests
{
    [Fact]
    public void OrderBuilder_CreatesValidOrder()
    {
        // Arrange & Act
        var order = new OrderBuilder()
            .WithCustomer("customer123")
            .WithStatus(OrderStatus.Processing)
            .WithItem(1, 2, 10.00m)
            .WithItem(2, 1, 20.00m)
            .Build();
        
        // Assert
        Assert.Equal("customer123", order.CustomerId);
        Assert.Equal(OrderStatus.Processing, order.Status);
        Assert.Equal(40.00m, order.TotalAmount);
        Assert.Equal(2, order.Items.Count);
    }
}
```

## FluentAssertions 사용

### 가독성 높은 Assertions
```csharp
public class FluentAssertionsTests
{
    [Fact]
    public void FluentAssertions_BasicAssertions()
    {
        // 문자열 검증
        string name = "John Doe";
        name.Should().NotBeNullOrEmpty()
            .And.StartWith("John")
            .And.EndWith("Doe")
            .And.Contain("hn D")
            .And.HaveLength(8);
        
        // 숫자 검증
        int age = 25;
        age.Should().BePositive()
            .And.BeGreaterThan(18)
            .And.BeLessThanOrEqualTo(100)
            .And.BeInRange(20, 30);
        
        // 날짜 검증
        var date = new DateTime(2024, 1, 15);
        date.Should().BeAfter(new DateTime(2024, 1, 1))
            .And.BeBefore(new DateTime(2024, 12, 31))
            .And.HaveYear(2024)
            .And.HaveMonth(1)
            .And.HaveDay(15);
    }
    
    [Fact]
    public void FluentAssertions_CollectionAssertions()
    {
        // Arrange
        var numbers = new[] { 1, 2, 3, 4, 5 };
        var products = new List<Product>
        {
            new Product { Id = 1, Name = "Product A", Price = 10.00m },
            new Product { Id = 2, Name = "Product B", Price = 20.00m },
            new Product { Id = 3, Name = "Product C", Price = 15.00m }
        };
        
        // Assert
        numbers.Should().HaveCount(5)
            .And.Contain(3)
            .And.NotContain(0)
            .And.BeInAscendingOrder()
            .And.OnlyHaveUniqueItems();
        
        products.Should().NotBeEmpty()
            .And.HaveCount(3)
            .And.Contain(p => p.Price > 10)
            .And.NotContain(p => p.Name == "Product D")
            .And.AllSatisfy(p => p.Price.Should().BePositive());
        
        products.Should().BeEquivalentTo(new[]
        {
            new { Id = 1, Name = "Product A" },
            new { Id = 2, Name = "Product B" },
            new { Id = 3, Name = "Product C" }
        }, options => options.ExcludingMissingMembers());
    }
    
    [Fact]
    public void FluentAssertions_ExceptionAssertions()
    {
        // Arrange
        var calculator = new Calculator();
        
        // Act & Assert
        Action divideByZero = () => calculator.Divide(10, 0);
        
        divideByZero.Should().Throw<DivideByZeroException>()
            .WithMessage("*divide by zero*")
            .And.HaveInnerException<ArgumentException>();
        
        // Async exception
        Func<Task> asyncAction = async () => 
            await calculator.DivideAsync(10, 0);
        
        asyncAction.Should().ThrowAsync<DivideByZeroException>()
            .WithMessage("Cannot divide by zero");
    }
    
    [Fact]
    public void FluentAssertions_ObjectComparison()
    {
        // Arrange
        var expected = new Order
        {
            Id = 1,
            CustomerId = "CUST001",
            OrderDate = new DateTime(2024, 1, 15),
            Items = new List<OrderItem>
            {
                new OrderItem { ProductId = 1, Quantity = 2 },
                new OrderItem { ProductId = 2, Quantity = 1 }
            }
        };
        
        var actual = new Order
        {
            Id = 1,
            CustomerId = "CUST001",
            OrderDate = new DateTime(2024, 1, 15),
            Items = new List<OrderItem>
            {
                new OrderItem { ProductId = 1, Quantity = 2 },
                new OrderItem { ProductId = 2, Quantity = 1 }
            }
        };
        
        // Assert
        actual.Should().BeEquivalentTo(expected, options => options
            .Including(o => o.Id)
            .Including(o => o.CustomerId)
            .Including(o => o.OrderDate)
            .Including(o => o.Items)
            .Using<DateTime>(ctx => ctx.Subject.Should()
                .BeCloseTo(ctx.Expectation, TimeSpan.FromSeconds(1)))
            .WhenTypeIs<DateTime>());
    }
}
```

## 테스트 성능 최적화

### 병렬 테스트 실행
```csharp
// Assembly level - 병렬 실행 비활성화
[assembly: CollectionBehavior(DisableTestParallelization = true)]

// Collection level - 특정 컬렉션만 순차 실행
[Collection("Sequential Tests")]
public class SequentialTestClass1
{
    [Fact]
    public void Test1() { /* ... */ }
}

[Collection("Sequential Tests")]
public class SequentialTestClass2
{
    [Fact]
    public void Test2() { /* ... */ }
}

// 병렬 실행 제어
public class ParallelTestsConfiguration : ICollectionFixture<TestServerFixture>
{
    // 이 컬렉션의 테스트들은 병렬로 실행됨
}
```

### 테스트 데이터 재사용
```csharp
public class TestDataCache
{
    private static readonly Lazy<List<Product>> _products = new(() =>
    {
        // 비용이 큰 데이터 생성
        return GenerateProducts(1000);
    });
    
    public static List<Product> Products => _products.Value;
    
    private static List<Product> GenerateProducts(int count)
    {
        var products = new List<Product>();
        for (int i = 1; i <= count; i++)
        {
            products.Add(new Product
            {
                Id = i,
                Name = $"Product {i}",
                Price = Random.Shared.Next(10, 1000),
                Category = $"Category {i % 10}"
            });
        }
        return products;
    }
}

public class PerformanceOptimizedTests
{
    [Fact]
    public void Test_UsesSharedTestData()
    {
        // Arrange
        var products = TestDataCache.Products;
        
        // Act
        var expensiveProducts = products.Where(p => p.Price > 500).ToList();
        
        // Assert
        expensiveProducts.Should().NotBeEmpty();
    }
}
```

## 테스트 모범 사례

### AAA (Arrange-Act-Assert) 패턴
```csharp
public class BestPracticesTests
{
    [Fact]
    public void GoodTest_FollowsAAAPattern()
    {
        // Arrange - 테스트 설정
        var repository = new Mock<IProductRepository>();
        var product = new Product { Id = 1, Name = "Test Product", Price = 10.00m };
        repository.Setup(x => x.GetByIdAsync(1)).ReturnsAsync(product);
        
        var service = new ProductService(repository.Object);
        
        // Act - 테스트 대상 실행
        var result = service.GetProductByIdAsync(1).Result;
        
        // Assert - 결과 검증
        result.Should().NotBeNull();
        result.Id.Should().Be(1);
        result.Name.Should().Be("Test Product");
    }
    
    [Theory]
    [InlineData("", false)]           // 빈 문자열
    [InlineData(" ", false)]          // 공백
    [InlineData(null, false)]         // null
    [InlineData("a", false)]          // 너무 짧음
    [InlineData("ab", false)]         // 여전히 짧음
    [InlineData("abc", true)]         // 최소 길이
    [InlineData("valid", true)]       // 유효
    [InlineData("toolongname", false)] // 너무 김
    public void ValidateUsername_VariousInputs_ReturnsExpectedResult(
        string username, bool expectedValid)
    {
        // Arrange
        var validator = new UsernameValidator();
        
        // Act
        var result = validator.IsValid(username);
        
        // Assert
        result.Should().Be(expectedValid);
    }
}

// 테스트 이름 규칙
public class NamingConventionTests
{
    [Fact]
    public void MethodName_StateUnderTest_ExpectedBehavior()
    {
        // 예: CalculateDiscount_PremiumCustomer_Returns20PercentOff
    }
    
    [Fact]
    public void Should_ExpectedBehavior_When_StateUnderTest()
    {
        // 예: Should_ThrowException_When_DividingByZero
    }
    
    [Fact]
    public void Given_Preconditions_When_StateUnderTest_Then_ExpectedBehavior()
    {
        // 예: Given_EmptyCart_When_AddingProduct_Then_CartHasOneItem
    }
}
```

### 테스트 격리
```csharp
public class TestIsolationExample
{
    [Fact]
    public void Test_ShouldBeIndependent()
    {
        // 각 테스트는 다른 테스트에 의존하지 않아야 함
        // 자체 설정과 정리 수행
        
        // Arrange
        var dbContext = CreateInMemoryDbContext();
        var repository = new ProductRepository(dbContext);
        
        try
        {
            // Act
            var product = new Product { Name = "Test Product" };
            await repository.AddAsync(product);
            
            // Assert
            var saved = await repository.GetByIdAsync(product.Id);
            saved.Should().NotBeNull();
        }
        finally
        {
            // Cleanup
            dbContext.Dispose();
        }
    }
    
    private ApplicationDbContext CreateInMemoryDbContext()
    {
        var options = new DbContextOptionsBuilder<ApplicationDbContext>()
            .UseInMemoryDatabase(databaseName: Guid.NewGuid().ToString())
            .Options;
            
        return new ApplicationDbContext(options);
    }
}
```

단위 테스트는 코드 품질을 보장하고 리팩토링을 안전하게 수행할 수 있게 해주는 중요한 도구입니다. xUnit과 Moq를 활용하여 효과적인 테스트를 작성할 수 있습니다.