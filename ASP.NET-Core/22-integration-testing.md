# Integration Testing

## 통합 테스트 개요

통합 테스트는 애플리케이션의 여러 구성 요소가 함께 올바르게 작동하는지 확인하는 테스트입니다. 단위 테스트와 달리 실제 데이터베이스, 파일 시스템, 네트워크 등의 외부 종속성을 포함하여 테스트합니다.

### 통합 테스트의 특징
- **실제 환경과 유사**: 프로덕션과 유사한 환경에서 테스트
- **전체 요청 파이프라인**: HTTP 요청부터 응답까지 전체 흐름 테스트
- **실제 종속성 사용**: 데이터베이스, 캐시, 외부 서비스 등 실제 구성 요소 사용
- **느린 실행 속도**: 단위 테스트보다 실행 시간이 오래 걸림
- **복잡한 설정**: 테스트 환경 구성이 복잡할 수 있음

## WebApplicationFactory 설정

### 기본 설정
```csharp
public class BasicIntegrationTest : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;
    
    public BasicIntegrationTest(WebApplicationFactory<Program> factory)
    {
        _factory = factory;
    }
    
    [Theory]
    [InlineData("/")]
    [InlineData("/api/products")]
    [InlineData("/api/health")]
    public async Task Get_EndpointsReturnSuccessAndCorrectContentType(string url)
    {
        // Arrange
        var client = _factory.CreateClient();
        
        // Act
        var response = await client.GetAsync(url);
        
        // Assert
        response.EnsureSuccessStatusCode();
        Assert.Equal("application/json; charset=utf-8", 
            response.Content.Headers.ContentType.ToString());
    }
}
```

### Custom WebApplicationFactory
```csharp
public class CustomWebApplicationFactory<TStartup> 
    : WebApplicationFactory<TStartup> where TStartup : class
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // 기존 DbContext 제거
            var descriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<ApplicationDbContext>));
            
            if (descriptor != null)
            {
                services.Remove(descriptor);
            }
            
            // 테스트용 In-Memory 데이터베이스 추가
            services.AddDbContext<ApplicationDbContext>(options =>
            {
                options.UseInMemoryDatabase("InMemoryDbForTesting");
            });
            
            // 테스트용 서비스 재정의
            services.AddScoped<IEmailService, FakeEmailService>();
            services.AddScoped<IExternalApiClient, FakeExternalApiClient>();
            
            // 테스트 데이터 시딩을 위한 서비스
            var sp = services.BuildServiceProvider();
            using (var scope = sp.CreateScope())
            {
                var scopedServices = scope.ServiceProvider;
                var db = scopedServices.GetRequiredService<ApplicationDbContext>();
                var logger = scopedServices.GetRequiredService<ILogger<CustomWebApplicationFactory<TStartup>>>();
                
                db.Database.EnsureCreated();
                
                try
                {
                    SeedDatabase(db);
                }
                catch (Exception ex)
                {
                    logger.LogError(ex, "An error occurred seeding the database with test data.");
                }
            }
        });
        
        builder.ConfigureAppConfiguration((context, config) =>
        {
            // 테스트용 설정 추가
            config.AddInMemoryCollection(new[]
            {
                new KeyValuePair<string, string>("ConnectionStrings:DefaultConnection", "InMemory"),
                new KeyValuePair<string, string>("ExternalApi:BaseUrl", "https://test-api.example.com"),
                new KeyValuePair<string, string>("Features:EnableEmailNotifications", "false")
            });
        });
    }
    
    private void SeedDatabase(ApplicationDbContext context)
    {
        // 테스트 데이터 추가
        var products = new List<Product>
        {
            new Product { Id = 1, Name = "Test Product 1", Price = 10.00m, Stock = 100 },
            new Product { Id = 2, Name = "Test Product 2", Price = 20.00m, Stock = 50 },
            new Product { Id = 3, Name = "Test Product 3", Price = 30.00m, Stock = 0 }
        };
        
        context.Products.AddRange(products);
        
        var users = new List<User>
        {
            new User { Id = "user1", UserName = "testuser1", Email = "test1@example.com" },
            new User { Id = "user2", UserName = "testuser2", Email = "test2@example.com" }
        };
        
        context.Users.AddRange(users);
        context.SaveChanges();
    }
}
```

## API 통합 테스트

### 기본 API 테스트
```csharp
public class ProductsApiIntegrationTests : IClassFixture<CustomWebApplicationFactory<Program>>
{
    private readonly CustomWebApplicationFactory<Program> _factory;
    private readonly HttpClient _client;
    
    public ProductsApiIntegrationTests(CustomWebApplicationFactory<Program> factory)
    {
        _factory = factory;
        _client = _factory.CreateClient(new WebApplicationFactoryClientOptions
        {
            AllowAutoRedirect = false
        });
    }
    
    [Fact]
    public async Task GetProducts_ReturnsSuccessStatusCode()
    {
        // Act
        var response = await _client.GetAsync("/api/products");
        
        // Assert
        response.EnsureSuccessStatusCode();
        
        var content = await response.Content.ReadAsStringAsync();
        var products = JsonSerializer.Deserialize<List<Product>>(content, 
            new JsonSerializerOptions { PropertyNameCaseInsensitive = true });
        
        Assert.NotNull(products);
        Assert.Equal(3, products.Count);
    }
    
    [Fact]
    public async Task GetProduct_ValidId_ReturnsProduct()
    {
        // Arrange
        var productId = 1;
        
        // Act
        var response = await _client.GetAsync($"/api/products/{productId}");
        
        // Assert
        response.EnsureSuccessStatusCode();
        
        var content = await response.Content.ReadAsStringAsync();
        var product = JsonSerializer.Deserialize<Product>(content,
            new JsonSerializerOptions { PropertyNameCaseInsensitive = true });
        
        Assert.NotNull(product);
        Assert.Equal(productId, product.Id);
        Assert.Equal("Test Product 1", product.Name);
    }
    
    [Fact]
    public async Task GetProduct_InvalidId_ReturnsNotFound()
    {
        // Arrange
        var productId = 999;
        
        // Act
        var response = await _client.GetAsync($"/api/products/{productId}");
        
        // Assert
        Assert.Equal(HttpStatusCode.NotFound, response.StatusCode);
    }
    
    [Fact]
    public async Task CreateProduct_ValidData_ReturnsCreatedProduct()
    {
        // Arrange
        var newProduct = new CreateProductDto
        {
            Name = "New Test Product",
            Price = 25.00m,
            Stock = 75,
            Description = "Integration test product"
        };
        
        var json = JsonSerializer.Serialize(newProduct);
        var content = new StringContent(json, Encoding.UTF8, "application/json");
        
        // Act
        var response = await _client.PostAsync("/api/products", content);
        
        // Assert
        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
        
        var responseContent = await response.Content.ReadAsStringAsync();
        var createdProduct = JsonSerializer.Deserialize<Product>(responseContent,
            new JsonSerializerOptions { PropertyNameCaseInsensitive = true });
        
        Assert.NotNull(createdProduct);
        Assert.NotEqual(0, createdProduct.Id);
        Assert.Equal(newProduct.Name, createdProduct.Name);
        Assert.Equal(newProduct.Price, createdProduct.Price);
        
        // Location 헤더 확인
        Assert.NotNull(response.Headers.Location);
        Assert.Contains($"/api/products/{createdProduct.Id}", response.Headers.Location.ToString());
    }
    
    [Fact]
    public async Task UpdateProduct_ValidData_ReturnsNoContent()
    {
        // Arrange
        var productId = 1;
        var updateDto = new UpdateProductDto
        {
            Name = "Updated Product",
            Price = 15.00m,
            Stock = 200
        };
        
        var json = JsonSerializer.Serialize(updateDto);
        var content = new StringContent(json, Encoding.UTF8, "application/json");
        
        // Act
        var response = await _client.PutAsync($"/api/products/{productId}", content);
        
        // Assert
        Assert.Equal(HttpStatusCode.NoContent, response.StatusCode);
        
        // 업데이트 확인
        var getResponse = await _client.GetAsync($"/api/products/{productId}");
        var getContent = await getResponse.Content.ReadAsStringAsync();
        var updatedProduct = JsonSerializer.Deserialize<Product>(getContent,
            new JsonSerializerOptions { PropertyNameCaseInsensitive = true });
        
        Assert.Equal(updateDto.Name, updatedProduct.Name);
        Assert.Equal(updateDto.Price, updatedProduct.Price);
    }
    
    [Fact]
    public async Task DeleteProduct_ValidId_ReturnsNoContent()
    {
        // Arrange
        var productId = 3;
        
        // Act
        var response = await _client.DeleteAsync($"/api/products/{productId}");
        
        // Assert
        Assert.Equal(HttpStatusCode.NoContent, response.StatusCode);
        
        // 삭제 확인
        var getResponse = await _client.GetAsync($"/api/products/{productId}");
        Assert.Equal(HttpStatusCode.NotFound, getResponse.StatusCode);
    }
}
```

### 인증이 필요한 API 테스트
```csharp
public class AuthenticatedApiTests : IClassFixture<CustomWebApplicationFactory<Program>>
{
    private readonly CustomWebApplicationFactory<Program> _factory;
    
    public AuthenticatedApiTests(CustomWebApplicationFactory<Program> factory)
    {
        _factory = factory;
    }
    
    [Fact]
    public async Task GetOrders_WithoutAuth_ReturnsUnauthorized()
    {
        // Arrange
        var client = _factory.CreateClient();
        
        // Act
        var response = await client.GetAsync("/api/orders");
        
        // Assert
        Assert.Equal(HttpStatusCode.Unauthorized, response.StatusCode);
    }
    
    [Fact]
    public async Task GetOrders_WithValidToken_ReturnsOrders()
    {
        // Arrange
        var client = _factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                services.AddAuthentication("Test")
                    .AddScheme<TestAuthenticationSchemeOptions, TestAuthenticationHandler>(
                        "Test", options => { });
            });
        }).CreateClient();
        
        client.DefaultRequestHeaders.Authorization = 
            new AuthenticationHeaderValue("Test");
        
        // Act
        var response = await client.GetAsync("/api/orders");
        
        // Assert
        response.EnsureSuccessStatusCode();
    }
    
    [Fact]
    public async Task CreateOrder_WithAdminRole_Succeeds()
    {
        // Arrange
        var client = _factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                services.AddAuthentication("Test")
                    .AddScheme<TestAuthenticationSchemeOptions, TestAuthenticationHandler>(
                        "Test", options => 
                        {
                            options.DefaultUserId = "admin-user";
                            options.DefaultUserRole = "Admin";
                        });
            });
        }).CreateClient();
        
        client.DefaultRequestHeaders.Authorization = 
            new AuthenticationHeaderValue("Test");
        
        var orderData = new CreateOrderDto
        {
            CustomerId = "customer123",
            Items = new List<OrderItemDto>
            {
                new OrderItemDto { ProductId = 1, Quantity = 2 }
            }
        };
        
        var content = new StringContent(
            JsonSerializer.Serialize(orderData),
            Encoding.UTF8,
            "application/json");
        
        // Act
        var response = await client.PostAsync("/api/orders", content);
        
        // Assert
        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
    }
}

// 테스트용 인증 핸들러
public class TestAuthenticationHandler : AuthenticationHandler<TestAuthenticationSchemeOptions>
{
    public TestAuthenticationHandler(IOptionsMonitor<TestAuthenticationSchemeOptions> options,
        ILoggerFactory logger, UrlEncoder encoder, ISystemClock clock)
        : base(options, logger, encoder, clock)
    {
    }
    
    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        var claims = new[]
        {
            new Claim(ClaimTypes.NameIdentifier, Options.DefaultUserId ?? "test-user"),
            new Claim(ClaimTypes.Name, Options.DefaultUserName ?? "Test User"),
            new Claim(ClaimTypes.Role, Options.DefaultUserRole ?? "User")
        };
        
        var identity = new ClaimsIdentity(claims, "Test");
        var principal = new ClaimsPrincipal(identity);
        var ticket = new AuthenticationTicket(principal, "Test");
        
        return Task.FromResult(AuthenticateResult.Success(ticket));
    }
}

public class TestAuthenticationSchemeOptions : AuthenticationSchemeOptions
{
    public string DefaultUserId { get; set; }
    public string DefaultUserName { get; set; }
    public string DefaultUserRole { get; set; }
}
```

## 데이터베이스 통합 테스트

### SQL Server 통합 테스트
```csharp
public class SqlServerIntegrationTests : IAsyncLifetime
{
    private readonly string _connectionString;
    private ApplicationDbContext _context;
    
    public SqlServerIntegrationTests()
    {
        // 테스트용 데이터베이스 연결 문자열
        _connectionString = "Server=(localdb)\\mssqllocaldb;Database=IntegrationTestDb_" + 
            Guid.NewGuid() + ";Trusted_Connection=True;";
    }
    
    public async Task InitializeAsync()
    {
        var options = new DbContextOptionsBuilder<ApplicationDbContext>()
            .UseSqlServer(_connectionString)
            .Options;
        
        _context = new ApplicationDbContext(options);
        await _context.Database.EnsureCreatedAsync();
        
        // 테스트 데이터 시딩
        await SeedTestDataAsync();
    }
    
    public async Task DisposeAsync()
    {
        await _context.Database.EnsureDeletedAsync();
        await _context.DisposeAsync();
    }
    
    [Fact]
    public async Task Repository_CanPerformCrudOperations()
    {
        // Arrange
        var repository = new ProductRepository(_context);
        var newProduct = new Product
        {
            Name = "Integration Test Product",
            Price = 99.99m,
            Stock = 10,
            CreatedAt = DateTime.UtcNow
        };
        
        // Act - Create
        await repository.AddAsync(newProduct);
        await _context.SaveChangesAsync();
        
        // Assert - Create
        Assert.NotEqual(0, newProduct.Id);
        
        // Act - Read
        var retrievedProduct = await repository.GetByIdAsync(newProduct.Id);
        
        // Assert - Read
        Assert.NotNull(retrievedProduct);
        Assert.Equal(newProduct.Name, retrievedProduct.Name);
        
        // Act - Update
        retrievedProduct.Price = 149.99m;
        await repository.UpdateAsync(retrievedProduct);
        await _context.SaveChangesAsync();
        
        var updatedProduct = await repository.GetByIdAsync(newProduct.Id);
        
        // Assert - Update
        Assert.Equal(149.99m, updatedProduct.Price);
        
        // Act - Delete
        await repository.DeleteAsync(newProduct.Id);
        await _context.SaveChangesAsync();
        
        var deletedProduct = await repository.GetByIdAsync(newProduct.Id);
        
        // Assert - Delete
        Assert.Null(deletedProduct);
    }
    
    [Fact]
    public async Task Repository_ComplexQuery_ReturnsCorrectResults()
    {
        // Arrange
        var repository = new ProductRepository(_context);
        
        // Act
        var expensiveProducts = await repository.GetProductsAsync(
            filter: p => p.Price > 50,
            orderBy: q => q.OrderByDescending(p => p.Price),
            includeProperties: "Category,Reviews");
        
        // Assert
        Assert.NotEmpty(expensiveProducts);
        Assert.All(expensiveProducts, p => Assert.True(p.Price > 50));
        Assert.True(expensiveProducts.First().Price >= expensiveProducts.Last().Price);
    }
    
    private async Task SeedTestDataAsync()
    {
        var categories = new[]
        {
            new Category { Name = "Electronics" },
            new Category { Name = "Clothing" }
        };
        
        _context.Categories.AddRange(categories);
        await _context.SaveChangesAsync();
        
        var products = new[]
        {
            new Product 
            { 
                Name = "Laptop", 
                Price = 999.99m, 
                Stock = 5, 
                CategoryId = categories[0].Id 
            },
            new Product 
            { 
                Name = "T-Shirt", 
                Price = 19.99m, 
                Stock = 100, 
                CategoryId = categories[1].Id 
            },
            new Product 
            { 
                Name = "Smartphone", 
                Price = 699.99m, 
                Stock = 20, 
                CategoryId = categories[0].Id 
            }
        };
        
        _context.Products.AddRange(products);
        await _context.SaveChangesAsync();
    }
}
```

### 트랜잭션 테스트
```csharp
public class TransactionIntegrationTests : IClassFixture<DatabaseFixture>
{
    private readonly DatabaseFixture _fixture;
    
    public TransactionIntegrationTests(DatabaseFixture fixture)
    {
        _fixture = fixture;
    }
    
    [Fact]
    public async Task OrderService_CreateOrder_RollbackOnFailure()
    {
        // Arrange
        using var transaction = await _fixture.Context.Database.BeginTransactionAsync();
        
        var orderService = new OrderService(
            _fixture.Context,
            new Mock<IEmailService>().Object,
            new Mock<ILogger<OrderService>>().Object);
        
        var orderDto = new CreateOrderDto
        {
            CustomerId = "customer123",
            Items = new List<OrderItemDto>
            {
                new OrderItemDto { ProductId = 1, Quantity = 1000 } // 재고보다 많은 수량
            }
        };
        
        try
        {
            // Act
            await orderService.CreateOrderAsync(orderDto);
            await transaction.CommitAsync();
            
            Assert.True(false, "예외가 발생해야 합니다");
        }
        catch (InsufficientStockException)
        {
            // Assert
            await transaction.RollbackAsync();
            
            // 제품 재고가 변경되지 않았는지 확인
            var product = await _fixture.Context.Products.FindAsync(1);
            Assert.Equal(100, product.Stock); // 원래 재고 유지
            
            // 주문이 생성되지 않았는지 확인
            var ordersCount = await _fixture.Context.Orders.CountAsync();
            Assert.Equal(0, ordersCount);
        }
    }
    
    [Fact]
    public async Task UnitOfWork_MultipleRepositories_ShareTransaction()
    {
        // Arrange
        using var unitOfWork = new UnitOfWork(_fixture.Context);
        
        var newCategory = new Category { Name = "Test Category" };
        var newProduct = new Product
        {
            Name = "Test Product",
            Price = 50.00m,
            Stock = 10
        };
        
        try
        {
            // Act
            await unitOfWork.Categories.AddAsync(newCategory);
            await unitOfWork.SaveChangesAsync();
            
            newProduct.CategoryId = newCategory.Id;
            await unitOfWork.Products.AddAsync(newProduct);
            
            // 의도적으로 오류 발생
            newProduct.Price = -10; // 유효성 검사 실패
            
            await unitOfWork.SaveChangesAsync();
            await unitOfWork.CommitAsync();
        }
        catch (DbUpdateException)
        {
            // Assert
            await unitOfWork.RollbackAsync();
            
            // 카테고리도 롤백되었는지 확인
            var categoriesCount = await _fixture.Context.Categories
                .Where(c => c.Name == "Test Category")
                .CountAsync();
            Assert.Equal(0, categoriesCount);
        }
    }
}
```

## 외부 서비스 통합 테스트

### HTTP 클라이언트 테스트
```csharp
public class ExternalApiIntegrationTests
{
    [Fact]
    public async Task ExternalApiClient_GetWeatherData_ReturnsValidData()
    {
        // Arrange
        var handlerMock = new Mock<HttpMessageHandler>();
        var responseData = new WeatherData
        {
            Temperature = 25.5,
            Humidity = 60,
            Description = "Sunny"
        };
        
        handlerMock
            .Protected()
            .Setup<Task<HttpResponseMessage>>(
                "SendAsync",
                ItExpr.IsAny<HttpRequestMessage>(),
                ItExpr.IsAny<CancellationToken>())
            .ReturnsAsync(new HttpResponseMessage
            {
                StatusCode = HttpStatusCode.OK,
                Content = new StringContent(
                    JsonSerializer.Serialize(responseData),
                    Encoding.UTF8,
                    "application/json")
            });
        
        var httpClient = new HttpClient(handlerMock.Object)
        {
            BaseAddress = new Uri("https://api.weather.com/")
        };
        
        var apiClient = new WeatherApiClient(httpClient, 
            new Mock<ILogger<WeatherApiClient>>().Object);
        
        // Act
        var result = await apiClient.GetWeatherAsync("London");
        
        // Assert
        Assert.NotNull(result);
        Assert.Equal(25.5, result.Temperature);
        Assert.Equal("Sunny", result.Description);
        
        handlerMock.Protected().Verify(
            "SendAsync",
            Times.Once(),
            ItExpr.Is<HttpRequestMessage>(req =>
                req.Method == HttpMethod.Get &&
                req.RequestUri.ToString().Contains("London")),
            ItExpr.IsAny<CancellationToken>());
    }
    
    [Fact]
    public async Task ExternalApiClient_WithRetry_HandlesTransientErrors()
    {
        // Arrange
        var handlerMock = new Mock<HttpMessageHandler>();
        var callCount = 0;
        
        handlerMock
            .Protected()
            .Setup<Task<HttpResponseMessage>>(
                "SendAsync",
                ItExpr.IsAny<HttpRequestMessage>(),
                ItExpr.IsAny<CancellationToken>())
            .ReturnsAsync(() =>
            {
                callCount++;
                if (callCount < 3)
                {
                    return new HttpResponseMessage(HttpStatusCode.ServiceUnavailable);
                }
                return new HttpResponseMessage
                {
                    StatusCode = HttpStatusCode.OK,
                    Content = new StringContent("{\"result\":\"success\"}")
                };
            });
        
        var httpClient = new HttpClient(handlerMock.Object);
        
        var retryPolicy = Policy
            .HandleResult<HttpResponseMessage>(r => !r.IsSuccessStatusCode)
            .WaitAndRetryAsync(
                3,
                retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)));
        
        var apiClient = new ResilientApiClient(httpClient, retryPolicy);
        
        // Act
        var result = await apiClient.GetDataAsync();
        
        // Assert
        Assert.NotNull(result);
        Assert.Equal(3, callCount); // 2 실패 + 1 성공
    }
}
```

### 메시지 큐 통합 테스트
```csharp
public class MessageQueueIntegrationTests : IAsyncLifetime
{
    private IConnection _connection;
    private IModel _channel;
    private readonly string _queueName = "test-queue-" + Guid.NewGuid();
    
    public async Task InitializeAsync()
    {
        var factory = new ConnectionFactory
        {
            HostName = "localhost",
            UserName = "guest",
            Password = "guest"
        };
        
        _connection = factory.CreateConnection();
        _channel = _connection.CreateModel();
        
        _channel.QueueDeclare(
            queue: _queueName,
            durable: false,
            exclusive: false,
            autoDelete: true,
            arguments: null);
    }
    
    public Task DisposeAsync()
    {
        _channel?.QueueDelete(_queueName);
        _channel?.Dispose();
        _connection?.Dispose();
        return Task.CompletedTask;
    }
    
    [Fact]
    public async Task MessagePublisher_PublishMessage_CanBeConsumed()
    {
        // Arrange
        var publisher = new RabbitMQPublisher(_connection);
        var consumer = new RabbitMQConsumer(_connection);
        
        var message = new OrderMessage
        {
            OrderId = Guid.NewGuid(),
            CustomerId = "customer123",
            TotalAmount = 99.99m,
            CreatedAt = DateTime.UtcNow
        };
        
        var receivedMessage = (OrderMessage)null;
        var tcs = new TaskCompletionSource<bool>();
        
        // Act
        await consumer.ConsumeAsync<OrderMessage>(_queueName, async msg =>
        {
            receivedMessage = msg;
            tcs.SetResult(true);
            return true;
        });
        
        await publisher.PublishAsync(_queueName, message);
        
        // Assert
        var received = await tcs.Task.WaitAsync(TimeSpan.FromSeconds(5));
        Assert.True(received);
        Assert.NotNull(receivedMessage);
        Assert.Equal(message.OrderId, receivedMessage.OrderId);
        Assert.Equal(message.CustomerId, receivedMessage.CustomerId);
    }
}
```

## 성능 통합 테스트

### 부하 테스트
```csharp
public class PerformanceIntegrationTests : IClassFixture<CustomWebApplicationFactory<Program>>
{
    private readonly CustomWebApplicationFactory<Program> _factory;
    
    public PerformanceIntegrationTests(CustomWebApplicationFactory<Program> factory)
    {
        _factory = factory;
    }
    
    [Fact]
    public async Task Api_CanHandleConcurrentRequests()
    {
        // Arrange
        var client = _factory.CreateClient();
        var concurrentRequests = 100;
        var tasks = new List<Task<HttpResponseMessage>>();
        
        // Act
        var stopwatch = Stopwatch.StartNew();
        
        for (int i = 0; i < concurrentRequests; i++)
        {
            tasks.Add(client.GetAsync("/api/products"));
        }
        
        var responses = await Task.WhenAll(tasks);
        
        stopwatch.Stop();
        
        // Assert
        Assert.All(responses, response => 
        {
            Assert.Equal(HttpStatusCode.OK, response.StatusCode);
        });
        
        var averageResponseTime = stopwatch.ElapsedMilliseconds / (double)concurrentRequests;
        Assert.True(averageResponseTime < 100, 
            $"평균 응답 시간이 너무 깁니다: {averageResponseTime}ms");
    }
    
    [Fact]
    public async Task Database_OptimizedQueries_PerformWell()
    {
        // Arrange
        using var scope = _factory.Services.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
        var repository = new ProductRepository(context);
        
        // 많은 테스트 데이터 생성
        var products = Enumerable.Range(1, 1000).Select(i => new Product
        {
            Name = $"Product {i}",
            Price = Random.Shared.Next(10, 1000),
            Stock = Random.Shared.Next(0, 100),
            CategoryId = Random.Shared.Next(1, 10)
        }).ToList();
        
        await context.Products.AddRangeAsync(products);
        await context.SaveChangesAsync();
        
        // Act
        var stopwatch = Stopwatch.StartNew();
        
        var results = await repository.GetProductsAsync(
            filter: p => p.Price > 500 && p.Stock > 0,
            orderBy: q => q.OrderBy(p => p.Price),
            includeProperties: "Category",
            pageIndex: 1,
            pageSize: 50);
        
        stopwatch.Stop();
        
        // Assert
        Assert.True(stopwatch.ElapsedMilliseconds < 1000,
            $"쿼리 실행 시간이 너무 깁니다: {stopwatch.ElapsedMilliseconds}ms");
        Assert.NotEmpty(results);
    }
}
```

### 메모리 사용량 테스트
```csharp
public class MemoryIntegrationTests
{
    [Fact]
    public async Task LargeDataProcessing_DoesNotLeakMemory()
    {
        // Arrange
        var initialMemory = GC.GetTotalMemory(true);
        var service = new DataProcessingService();
        
        // Act
        for (int i = 0; i < 10; i++)
        {
            var data = GenerateLargeDataSet(10000);
            await service.ProcessDataAsync(data);
            
            // 명시적 가비지 컬렉션
            GC.Collect();
            GC.WaitForPendingFinalizers();
            GC.Collect();
        }
        
        var finalMemory = GC.GetTotalMemory(true);
        
        // Assert
        var memoryIncrease = finalMemory - initialMemory;
        var memoryIncreaseMB = memoryIncrease / (1024 * 1024);
        
        Assert.True(memoryIncreaseMB < 50, 
            $"메모리 사용량이 너무 많이 증가했습니다: {memoryIncreaseMB}MB");
    }
    
    private List<DataItem> GenerateLargeDataSet(int count)
    {
        return Enumerable.Range(1, count).Select(i => new DataItem
        {
            Id = i,
            Data = new byte[1024], // 1KB per item
            Timestamp = DateTime.UtcNow
        }).ToList();
    }
}
```

## End-to-End 테스트

### 전체 시나리오 테스트
```csharp
public class EndToEndTests : IClassFixture<CustomWebApplicationFactory<Program>>
{
    private readonly CustomWebApplicationFactory<Program> _factory;
    private readonly HttpClient _client;
    
    public EndToEndTests(CustomWebApplicationFactory<Program> factory)
    {
        _factory = factory;
        _client = _factory.CreateClient();
    }
    
    [Fact]
    public async Task CompleteOrderScenario_FromBrowsingToPayment()
    {
        // 1. 상품 목록 조회
        var productsResponse = await _client.GetAsync("/api/products");
        productsResponse.EnsureSuccessStatusCode();
        
        var productsJson = await productsResponse.Content.ReadAsStringAsync();
        var products = JsonSerializer.Deserialize<List<Product>>(productsJson,
            new JsonSerializerOptions { PropertyNameCaseInsensitive = true });
        
        Assert.NotEmpty(products);
        
        // 2. 사용자 등록
        var registerData = new RegisterDto
        {
            Email = "testuser@example.com",
            Password = "Test123!",
            ConfirmPassword = "Test123!"
        };
        
        var registerResponse = await _client.PostAsJsonAsync("/api/auth/register", registerData);
        Assert.Equal(HttpStatusCode.OK, registerResponse.StatusCode);
        
        // 3. 로그인
        var loginData = new LoginDto
        {
            Email = registerData.Email,
            Password = registerData.Password
        };
        
        var loginResponse = await _client.PostAsJsonAsync("/api/auth/login", loginData);
        loginResponse.EnsureSuccessStatusCode();
        
        var loginResult = await loginResponse.Content.ReadFromJsonAsync<LoginResult>();
        _client.DefaultRequestHeaders.Authorization = 
            new AuthenticationHeaderValue("Bearer", loginResult.Token);
        
        // 4. 장바구니에 상품 추가
        var cartItem = new AddToCartDto
        {
            ProductId = products.First().Id,
            Quantity = 2
        };
        
        var addToCartResponse = await _client.PostAsJsonAsync("/api/cart", cartItem);
        Assert.Equal(HttpStatusCode.OK, addToCartResponse.StatusCode);
        
        // 5. 장바구니 조회
        var cartResponse = await _client.GetAsync("/api/cart");
        cartResponse.EnsureSuccessStatusCode();
        
        var cart = await cartResponse.Content.ReadFromJsonAsync<CartDto>();
        Assert.Single(cart.Items);
        Assert.Equal(2, cart.Items.First().Quantity);
        
        // 6. 주문 생성
        var createOrderData = new CreateOrderDto
        {
            ShippingAddress = new AddressDto
            {
                Street = "123 Test St",
                City = "Test City",
                PostalCode = "12345",
                Country = "Test Country"
            },
            PaymentMethod = "CreditCard"
        };
        
        var orderResponse = await _client.PostAsJsonAsync("/api/orders", createOrderData);
        Assert.Equal(HttpStatusCode.Created, orderResponse.StatusCode);
        
        var order = await orderResponse.Content.ReadFromJsonAsync<OrderDto>();
        Assert.NotNull(order);
        Assert.Equal(OrderStatus.Pending, order.Status);
        
        // 7. 결제 처리
        var paymentData = new ProcessPaymentDto
        {
            OrderId = order.Id,
            CardNumber = "4111111111111111",
            ExpiryMonth = 12,
            ExpiryYear = 2025,
            CVV = "123"
        };
        
        var paymentResponse = await _client.PostAsJsonAsync("/api/payments", paymentData);
        Assert.Equal(HttpStatusCode.OK, paymentResponse.StatusCode);
        
        // 8. 주문 상태 확인
        var orderStatusResponse = await _client.GetAsync($"/api/orders/{order.Id}");
        orderStatusResponse.EnsureSuccessStatusCode();
        
        var updatedOrder = await orderStatusResponse.Content.ReadFromJsonAsync<OrderDto>();
        Assert.Equal(OrderStatus.Paid, updatedOrder.Status);
    }
}
```

## 테스트 데이터 관리

### Test Fixture 공유
```csharp
public class SharedDatabaseFixture : IDisposable
{
    private readonly string _connectionString;
    public ApplicationDbContext Context { get; private set; }
    
    public SharedDatabaseFixture()
    {
        _connectionString = "Server=(localdb)\\mssqllocaldb;Database=SharedTestDb_" + 
            Guid.NewGuid() + ";Trusted_Connection=True;";
        
        var options = new DbContextOptionsBuilder<ApplicationDbContext>()
            .UseSqlServer(_connectionString)
            .Options;
        
        Context = new ApplicationDbContext(options);
        Context.Database.EnsureCreated();
        
        SeedData();
    }
    
    private void SeedData()
    {
        // 기본 테스트 데이터 시딩
        var testDataSeeder = new TestDataSeeder(Context);
        testDataSeeder.SeedAll();
        Context.SaveChanges();
    }
    
    public void ResetDatabase()
    {
        // 트랜잭션으로 데이터 리셋
        using var transaction = Context.Database.BeginTransaction();
        
        Context.RemoveRange(Context.Orders);
        Context.RemoveRange(Context.OrderItems);
        Context.RemoveRange(Context.Products);
        Context.RemoveRange(Context.Categories);
        Context.RemoveRange(Context.Users);
        
        Context.SaveChanges();
        
        SeedData();
        
        transaction.Commit();
    }
    
    public void Dispose()
    {
        Context?.Dispose();
        
        // 데이터베이스 삭제
        using var cleanupContext = new ApplicationDbContext(
            new DbContextOptionsBuilder<ApplicationDbContext>()
                .UseSqlServer(_connectionString)
                .Options);
        
        cleanupContext.Database.EnsureDeleted();
    }
}

[CollectionDefinition("Database collection")]
public class DatabaseCollection : ICollectionFixture<SharedDatabaseFixture>
{
}

[Collection("Database collection")]
public class ProductTests
{
    private readonly SharedDatabaseFixture _fixture;
    
    public ProductTests(SharedDatabaseFixture fixture)
    {
        _fixture = fixture;
        _fixture.ResetDatabase(); // 각 테스트 클래스마다 리셋
    }
    
    [Fact]
    public async Task CanCreateProduct()
    {
        // 테스트 구현
    }
}
```

### 테스트 데이터 빌더
```csharp
public class TestDataBuilder
{
    private readonly ApplicationDbContext _context;
    
    public TestDataBuilder(ApplicationDbContext context)
    {
        _context = context;
    }
    
    public async Task<User> CreateUserAsync(Action<User> customize = null)
    {
        var user = new User
        {
            Id = Guid.NewGuid().ToString(),
            UserName = $"testuser_{Guid.NewGuid()}",
            Email = $"test_{Guid.NewGuid()}@example.com",
            EmailConfirmed = true
        };
        
        customize?.Invoke(user);
        
        _context.Users.Add(user);
        await _context.SaveChangesAsync();
        
        return user;
    }
    
    public async Task<Product> CreateProductAsync(Action<Product> customize = null)
    {
        var product = new Product
        {
            Name = $"Test Product {Guid.NewGuid()}",
            Price = Random.Shared.Next(10, 1000),
            Stock = Random.Shared.Next(0, 100),
            CreatedAt = DateTime.UtcNow
        };
        
        customize?.Invoke(product);
        
        _context.Products.Add(product);
        await _context.SaveChangesAsync();
        
        return product;
    }
    
    public async Task<Order> CreateOrderWithItemsAsync(
        string userId = null, 
        int itemCount = 3)
    {
        var user = userId != null 
            ? await _context.Users.FindAsync(userId) 
            : await CreateUserAsync();
        
        var order = new Order
        {
            UserId = user.Id,
            OrderDate = DateTime.UtcNow,
            Status = OrderStatus.Pending,
            Items = new List<OrderItem>()
        };
        
        for (int i = 0; i < itemCount; i++)
        {
            var product = await CreateProductAsync();
            order.Items.Add(new OrderItem
            {
                Product = product,
                Quantity = Random.Shared.Next(1, 5),
                UnitPrice = product.Price
            });
        }
        
        order.TotalAmount = order.Items.Sum(i => i.Quantity * i.UnitPrice);
        
        _context.Orders.Add(order);
        await _context.SaveChangesAsync();
        
        return order;
    }
}

// 사용 예제
public class OrderIntegrationTests
{
    private readonly TestDataBuilder _dataBuilder;
    
    [Fact]
    public async Task GetOrders_ReturnsUserOrders()
    {
        // Arrange
        var user = await _dataBuilder.CreateUserAsync(u => u.Email = "specific@example.com");
        var order1 = await _dataBuilder.CreateOrderWithItemsAsync(user.Id, 2);
        var order2 = await _dataBuilder.CreateOrderWithItemsAsync(user.Id, 3);
        var otherUserOrder = await _dataBuilder.CreateOrderWithItemsAsync(); // 다른 사용자
        
        // Act & Assert
        // ...
    }
}
```

## 모범 사례

### 테스트 격리
```csharp
public class IsolatedIntegrationTests : IDisposable
{
    private readonly WebApplicationFactory<Program> _factory;
    private readonly HttpClient _client;
    private readonly IServiceScope _scope;
    
    public IsolatedIntegrationTests()
    {
        _factory = new WebApplicationFactory<Program>()
            .WithWebHostBuilder(builder =>
            {
                builder.ConfigureServices(services =>
                {
                    // 각 테스트마다 새로운 데이터베이스
                    var dbName = "TestDb_" + Guid.NewGuid();
                    services.AddDbContext<ApplicationDbContext>(options =>
                    {
                        options.UseInMemoryDatabase(dbName);
                    });
                });
            });
        
        _client = _factory.CreateClient();
        _scope = _factory.Services.CreateScope();
    }
    
    [Fact]
    public async Task Test_IsCompletelyIsolated()
    {
        // 이 테스트는 다른 테스트와 완전히 격리됨
        var context = _scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
        
        // 테스트 실행
    }
    
    public void Dispose()
    {
        _scope?.Dispose();
        _client?.Dispose();
        _factory?.Dispose();
    }
}
```

### 테스트 가독성
```csharp
public class ReadableIntegrationTests
{
    [Fact]
    public async Task Should_ReturnUnauthorized_When_AccessingProtectedEndpoint_Without_Authentication()
    {
        // Given - 인증되지 않은 클라이언트
        var client = CreateUnauthenticatedClient();
        
        // When - 보호된 엔드포인트 접근 시도
        var response = await client.GetAsync("/api/admin/users");
        
        // Then - 401 Unauthorized 반환
        response.StatusCode.Should().Be(HttpStatusCode.Unauthorized);
    }
    
    [Fact]
    public async Task Should_CreateOrder_When_ValidDataProvided()
    {
        // Given
        var client = await CreateAuthenticatedClientAsync("customer@example.com");
        var orderData = new CreateOrderDtoBuilder()
            .WithProduct(productId: 1, quantity: 2)
            .WithProduct(productId: 2, quantity: 1)
            .WithShippingAddress("123 Main St", "City", "12345")
            .Build();
        
        // When
        var response = await client.PostAsJsonAsync("/api/orders", orderData);
        
        // Then
        response.Should().HaveStatusCode(HttpStatusCode.Created);
        response.Headers.Location.Should().NotBeNull();
        
        var createdOrder = await response.Content.ReadFromJsonAsync<OrderDto>();
        createdOrder.Should().NotBeNull();
        createdOrder.Items.Should().HaveCount(2);
        createdOrder.Status.Should().Be(OrderStatus.Pending);
    }
}
```

통합 테스트는 시스템의 여러 구성 요소가 함께 올바르게 작동하는지 확인하는 중요한 테스트입니다. WebApplicationFactory와 적절한 테스트 전략을 사용하여 안정적이고 유지보수가 쉬운 통합 테스트를 작성할 수 있습니다.