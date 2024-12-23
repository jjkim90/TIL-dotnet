# gRPC Services

## gRPC 개요

gRPC는 Google에서 개발한 고성능 오픈소스 RPC(Remote Procedure Call) 프레임워크입니다. HTTP/2를 기반으로 하며, Protocol Buffers를 사용하여 효율적인 직렬화를 제공합니다.

### gRPC의 특징
- **고성능**: HTTP/2 기반으로 빠른 통신
- **언어 중립적**: 다양한 프로그래밍 언어 지원
- **스트리밍**: 단방향 및 양방향 스트리밍 지원
- **강타입**: Protocol Buffers를 통한 타입 안전성
- **효율적인 직렬화**: 바이너리 형식으로 작은 페이로드

## gRPC 프로젝트 설정

### 프로젝트 생성
```xml
<!-- GrpcService.csproj -->
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Grpc.AspNetCore" Version="2.59.0" />
    <PackageReference Include="Grpc.Tools" Version="2.59.0">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
    </PackageReference>
  </ItemGroup>

  <ItemGroup>
    <Protobuf Include="Protos\*.proto" GrpcServices="Server" />
  </ItemGroup>
</Project>
```

### Program.cs 설정
```csharp
var builder = WebApplication.CreateBuilder(args);

// gRPC 서비스 추가
builder.Services.AddGrpc(options =>
{
    options.EnableDetailedErrors = true;
    options.MaxReceiveMessageSize = 2 * 1024 * 1024; // 2MB
    options.MaxSendMessageSize = 5 * 1024 * 1024; // 5MB
});

// 서비스 등록
builder.Services.AddScoped<IProductRepository, ProductRepository>();
builder.Services.AddScoped<IOrderService, OrderService>();

var app = builder.Build();

// gRPC 엔드포인트 매핑
app.MapGrpcService<ProductService>();
app.MapGrpcService<OrderService>();

// gRPC 리플렉션 (개발 환경)
if (app.Environment.IsDevelopment())
{
    app.MapGrpcReflectionService();
}

app.MapGet("/", () => "gRPC 서비스가 실행 중입니다.");

app.Run();
```

## Protocol Buffers 정의

### 기본 메시지 정의
```protobuf
// Protos/product.proto
syntax = "proto3";

option csharp_namespace = "GrpcService";

package product;

// 메시지 정의
message Product {
    int32 id = 1;
    string name = 2;
    string description = 3;
    double price = 4;
    int32 stock = 5;
    string category = 6;
    repeated string tags = 7;
}

message GetProductRequest {
    int32 id = 1;
}

message GetProductsRequest {
    int32 page = 1;
    int32 page_size = 2;
    string sort_by = 3;
    bool descending = 4;
}

message GetProductsResponse {
    repeated Product products = 1;
    int32 total_count = 2;
    int32 page = 3;
    int32 page_size = 4;
}

message CreateProductRequest {
    string name = 1;
    string description = 2;
    double price = 3;
    int32 stock = 4;
    string category = 5;
}

message UpdateProductRequest {
    int32 id = 1;
    string name = 2;
    string description = 3;
    double price = 4;
    int32 stock = 5;
}

message DeleteProductRequest {
    int32 id = 1;
}

message Empty {}

// 서비스 정의
service ProductService {
    rpc GetProduct(GetProductRequest) returns (Product);
    rpc GetProducts(GetProductsRequest) returns (GetProductsResponse);
    rpc CreateProduct(CreateProductRequest) returns (Product);
    rpc UpdateProduct(UpdateProductRequest) returns (Product);
    rpc DeleteProduct(DeleteProductRequest) returns (Empty);
    
    // 스트리밍
    rpc StreamProducts(Empty) returns (stream Product);
    rpc BatchCreateProducts(stream CreateProductRequest) returns (stream Product);
}
```

## gRPC 서비스 구현

### 기본 서비스 구현
```csharp
public class ProductService : Product.ProductBase
{
    private readonly IProductRepository _repository;
    private readonly ILogger<ProductService> _logger;
    
    public ProductService(IProductRepository repository, ILogger<ProductService> logger)
    {
        _repository = repository;
        _logger = logger;
    }
    
    public override async Task<Product> GetProduct(
        GetProductRequest request, 
        ServerCallContext context)
    {
        _logger.LogInformation("Getting product with ID: {ProductId}", request.Id);
        
        var product = await _repository.GetByIdAsync(request.Id);
        
        if (product == null)
        {
            throw new RpcException(new Status(
                StatusCode.NotFound, 
                $"Product with ID {request.Id} not found"));
        }
        
        return MapToProto(product);
    }
    
    public override async Task<GetProductsResponse> GetProducts(
        GetProductsRequest request, 
        ServerCallContext context)
    {
        var (products, totalCount) = await _repository.GetPagedAsync(
            request.Page, 
            request.PageSize, 
            request.SortBy, 
            request.Descending);
        
        return new GetProductsResponse
        {
            Products = { products.Select(MapToProto) },
            TotalCount = totalCount,
            Page = request.Page,
            PageSize = request.PageSize
        };
    }
    
    public override async Task<Product> CreateProduct(
        CreateProductRequest request, 
        ServerCallContext context)
    {
        // 유효성 검사
        if (string.IsNullOrWhiteSpace(request.Name))
        {
            throw new RpcException(new Status(
                StatusCode.InvalidArgument, 
                "Product name is required"));
        }
        
        if (request.Price <= 0)
        {
            throw new RpcException(new Status(
                StatusCode.InvalidArgument, 
                "Price must be greater than zero"));
        }
        
        var product = new Domain.Product
        {
            Name = request.Name,
            Description = request.Description,
            Price = (decimal)request.Price,
            Stock = request.Stock,
            Category = request.Category
        };
        
        var created = await _repository.CreateAsync(product);
        
        _logger.LogInformation("Product created with ID: {ProductId}", created.Id);
        
        return MapToProto(created);
    }
    
    private Product MapToProto(Domain.Product product)
    {
        return new Product
        {
            Id = product.Id,
            Name = product.Name,
            Description = product.Description ?? "",
            Price = (double)product.Price,
            Stock = product.Stock,
            Category = product.Category ?? "",
            Tags = { product.Tags ?? Enumerable.Empty<string>() }
        };
    }
}
```

### 스트리밍 구현
```csharp
public override async Task StreamProducts(
    Empty request, 
    IServerStreamWriter<Product> responseStream, 
    ServerCallContext context)
{
    var products = await _repository.GetAllAsync();
    
    foreach (var product in products)
    {
        // 클라이언트가 취소했는지 확인
        if (context.CancellationToken.IsCancellationRequested)
        {
            _logger.LogInformation("Client cancelled the streaming");
            break;
        }
        
        await responseStream.WriteAsync(MapToProto(product));
        
        // 시뮬레이션을 위한 지연
        await Task.Delay(100);
    }
}

public override async Task<Product> BatchCreateProducts(
    IAsyncStreamReader<CreateProductRequest> requestStream,
    IServerStreamWriter<Product> responseStream,
    ServerCallContext context)
{
    var createdProducts = new List<Product>();
    
    // 클라이언트 스트림 읽기
    await foreach (var request in requestStream.ReadAllAsync())
    {
        try
        {
            var product = await CreateProductFromRequest(request);
            var created = await _repository.CreateAsync(product);
            var protoProduct = MapToProto(created);
            
            // 생성된 제품을 즉시 스트리밍
            await responseStream.WriteAsync(protoProduct);
            createdProducts.Add(protoProduct);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error creating product: {ProductName}", request.Name);
            // 에러가 발생해도 계속 처리
        }
    }
    
    _logger.LogInformation("Batch created {Count} products", createdProducts.Count);
}
```

## gRPC 클라이언트

### 클라이언트 설정
```csharp
// 클라이언트 프로젝트
builder.Services.AddGrpcClient<ProductService.ProductServiceClient>(options =>
{
    options.Address = new Uri("https://localhost:5001");
})
.ConfigurePrimaryHttpMessageHandler(() =>
{
    var handler = new HttpClientHandler();
    // 개발 환경에서 SSL 인증서 검증 무시
    handler.ServerCertificateCustomValidationCallback = 
        HttpClientHandler.DangerousAcceptAnyServerCertificateValidator;
    return handler;
});

// 인터셉터 추가
builder.Services.AddSingleton<ClientLoggingInterceptor>();
builder.Services.AddGrpcClient<ProductService.ProductServiceClient>(options =>
{
    options.Address = new Uri("https://localhost:5001");
})
.AddInterceptor<ClientLoggingInterceptor>();
```

### 클라이언트 사용
```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly ProductService.ProductServiceClient _grpcClient;
    private readonly ILogger<ProductsController> _logger;
    
    public ProductsController(
        ProductService.ProductServiceClient grpcClient,
        ILogger<ProductsController> logger)
    {
        _grpcClient = grpcClient;
        _logger = logger;
    }
    
    [HttpGet("{id}")]
    public async Task<IActionResult> GetProduct(int id)
    {
        try
        {
            var request = new GetProductRequest { Id = id };
            var product = await _grpcClient.GetProductAsync(request);
            
            return Ok(new
            {
                product.Id,
                product.Name,
                product.Description,
                product.Price,
                product.Stock
            });
        }
        catch (RpcException ex) when (ex.StatusCode == StatusCode.NotFound)
        {
            return NotFound();
        }
        catch (RpcException ex)
        {
            _logger.LogError(ex, "gRPC call failed");
            return StatusCode(500, "Internal server error");
        }
    }
    
    [HttpGet("stream")]
    public async Task<IActionResult> StreamProducts()
    {
        var products = new List<object>();
        
        using var call = _grpcClient.StreamProducts(new Empty());
        
        await foreach (var product in call.ResponseStream.ReadAllAsync())
        {
            products.Add(new
            {
                product.Id,
                product.Name,
                product.Price
            });
        }
        
        return Ok(products);
    }
}
```

## 인터셉터

### 서버 인터셉터
```csharp
public class ServerLoggingInterceptor : Interceptor
{
    private readonly ILogger<ServerLoggingInterceptor> _logger;
    
    public ServerLoggingInterceptor(ILogger<ServerLoggingInterceptor> logger)
    {
        _logger = logger;
    }
    
    public override async Task<TResponse> UnaryServerHandler<TRequest, TResponse>(
        TRequest request,
        ServerCallContext context,
        UnaryServerMethod<TRequest, TResponse> continuation)
    {
        _logger.LogInformation(
            "Starting call. Type: {MethodType}. Method: {Method}",
            MethodType.Unary,
            context.Method);
        
        try
        {
            return await continuation(request, context);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error in method {Method}", context.Method);
            throw;
        }
    }
    
    public override Task ServerStreamingServerHandler<TRequest, TResponse>(
        TRequest request,
        IServerStreamWriter<TResponse> responseStream,
        ServerCallContext context,
        ServerStreamingServerMethod<TRequest, TResponse> continuation)
    {
        _logger.LogInformation("Starting server streaming call for {Method}", context.Method);
        return continuation(request, responseStream, context);
    }
}

// 등록
builder.Services.AddGrpc(options =>
{
    options.Interceptors.Add<ServerLoggingInterceptor>();
});
```

### 클라이언트 인터셉터
```csharp
public class ClientLoggingInterceptor : Interceptor
{
    private readonly ILogger<ClientLoggingInterceptor> _logger;
    
    public ClientLoggingInterceptor(ILogger<ClientLoggingInterceptor> logger)
    {
        _logger = logger;
    }
    
    public override AsyncUnaryCall<TResponse> AsyncUnaryCall<TRequest, TResponse>(
        TRequest request,
        ClientInterceptorContext<TRequest, TResponse> context,
        AsyncUnaryCallContinuation<TRequest, TResponse> continuation)
    {
        _logger.LogInformation("Starting client call to {Method}", context.Method.FullName);
        
        var call = continuation(request, context);
        
        return new AsyncUnaryCall<TResponse>(
            HandleResponse(call.ResponseAsync),
            call.ResponseHeadersAsync,
            call.GetStatus,
            call.GetTrailers,
            call.Dispose);
    }
    
    private async Task<TResponse> HandleResponse<TResponse>(Task<TResponse> response)
    {
        try
        {
            var result = await response;
            _logger.LogInformation("Call completed successfully");
            return result;
        }
        catch (RpcException ex)
        {
            _logger.LogError(ex, "Call failed with status {StatusCode}", ex.StatusCode);
            throw;
        }
    }
}
```

## 인증과 권한 부여

### JWT 인증
```csharp
// 서버 설정
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

// 서비스에 인증 적용
public class SecureProductService : Product.ProductBase
{
    public override async Task<Product> CreateProduct(
        CreateProductRequest request,
        ServerCallContext context)
    {
        var user = context.GetHttpContext().User;
        
        if (!user.Identity.IsAuthenticated)
        {
            throw new RpcException(new Status(StatusCode.Unauthenticated, "Not authenticated"));
        }
        
        if (!user.IsInRole("Admin"))
        {
            throw new RpcException(new Status(StatusCode.PermissionDenied, "Requires Admin role"));
        }
        
        // 제품 생성 로직
        return await base.CreateProduct(request, context);
    }
}
```

### 클라이언트 인증
```csharp
public class AuthenticationInterceptor : Interceptor
{
    private readonly ITokenService _tokenService;
    
    public AuthenticationInterceptor(ITokenService tokenService)
    {
        _tokenService = tokenService;
    }
    
    public override AsyncUnaryCall<TResponse> AsyncUnaryCall<TRequest, TResponse>(
        TRequest request,
        ClientInterceptorContext<TRequest, TResponse> context,
        AsyncUnaryCallContinuation<TRequest, TResponse> continuation)
    {
        var token = _tokenService.GetToken();
        
        if (!string.IsNullOrEmpty(token))
        {
            var headers = context.Options.Headers ?? new Metadata();
            headers.Add("Authorization", $"Bearer {token}");
            
            var newContext = new ClientInterceptorContext<TRequest, TResponse>(
                context.Method,
                context.Host,
                new CallOptions(headers));
            
            return continuation(request, newContext);
        }
        
        return continuation(request, context);
    }
}
```

## 에러 처리

### 상세한 에러 처리
```csharp
public class ErrorHandlingInterceptor : Interceptor
{
    private readonly ILogger<ErrorHandlingInterceptor> _logger;
    
    public override async Task<TResponse> UnaryServerHandler<TRequest, TResponse>(
        TRequest request,
        ServerCallContext context,
        UnaryServerMethod<TRequest, TResponse> continuation)
    {
        try
        {
            return await continuation(request, context);
        }
        catch (ValidationException ex)
        {
            _logger.LogWarning(ex, "Validation error");
            
            var status = new Status(StatusCode.InvalidArgument, ex.Message);
            var metadata = new Metadata();
            
            foreach (var error in ex.Errors)
            {
                metadata.Add($"field-{error.PropertyName}", error.ErrorMessage);
            }
            
            throw new RpcException(status, metadata);
        }
        catch (NotFoundException ex)
        {
            _logger.LogWarning(ex, "Resource not found");
            throw new RpcException(new Status(StatusCode.NotFound, ex.Message));
        }
        catch (UnauthorizedException ex)
        {
            _logger.LogWarning(ex, "Unauthorized access");
            throw new RpcException(new Status(StatusCode.Unauthenticated, ex.Message));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled exception");
            throw new RpcException(new Status(StatusCode.Internal, "An error occurred"));
        }
    }
}
```

## 성능 최적화

### 연결 관리
```csharp
// 연결 풀링
builder.Services.AddGrpcClient<ProductService.ProductServiceClient>(options =>
{
    options.Address = new Uri("https://localhost:5001");
})
.ConfigurePrimaryHttpMessageHandler(() =>
{
    return new SocketsHttpHandler
    {
        PooledConnectionIdleTimeout = TimeSpan.FromMinutes(5),
        KeepAlivePingDelay = TimeSpan.FromSeconds(60),
        KeepAlivePingTimeout = TimeSpan.FromSeconds(30),
        EnableMultipleHttp2Connections = true
    };
});

// 압축 설정
builder.Services.AddGrpc(options =>
{
    options.ResponseCompressionLevel = CompressionLevel.Optimal;
    options.ResponseCompressionAlgorithm = "gzip";
});
```

## 테스트

### gRPC 서비스 테스트
```csharp
public class ProductServiceTests
{
    [Fact]
    public async Task GetProduct_ReturnsProduct()
    {
        // Arrange
        var mockRepository = new Mock<IProductRepository>();
        var expectedProduct = new Domain.Product { Id = 1, Name = "Test Product" };
        mockRepository.Setup(x => x.GetByIdAsync(1)).ReturnsAsync(expectedProduct);
        
        var service = new ProductService(
            mockRepository.Object, 
            new NullLogger<ProductService>());
        
        var request = new GetProductRequest { Id = 1 };
        var context = TestServerCallContext.Create();
        
        // Act
        var response = await service.GetProduct(request, context);
        
        // Assert
        Assert.Equal(expectedProduct.Id, response.Id);
        Assert.Equal(expectedProduct.Name, response.Name);
    }
    
    [Fact]
    public async Task GetProduct_NotFound_ThrowsRpcException()
    {
        // Arrange
        var mockRepository = new Mock<IProductRepository>();
        mockRepository.Setup(x => x.GetByIdAsync(It.IsAny<int>())).ReturnsAsync((Domain.Product)null);
        
        var service = new ProductService(
            mockRepository.Object, 
            new NullLogger<ProductService>());
        
        var request = new GetProductRequest { Id = 999 };
        var context = TestServerCallContext.Create();
        
        // Act & Assert
        var exception = await Assert.ThrowsAsync<RpcException>(
            () => service.GetProduct(request, context));
        
        Assert.Equal(StatusCode.NotFound, exception.StatusCode);
    }
}
```

## 모범 사례

### 서비스 설계
```csharp
// 버전 관리를 위한 패키지 구조
// Protos/v1/product.proto
package product.v1;

// 메시지 재사용
message BaseResponse {
    bool success = 1;
    string message = 2;
    google.protobuf.Timestamp timestamp = 3;
}

message ProductResponse {
    BaseResponse base = 1;
    Product product = 2;
}

// 명확한 명명 규칙
service ProductService {
    // 동사_명사 패턴
    rpc GetProduct(GetProductRequest) returns (ProductResponse);
    rpc ListProducts(ListProductsRequest) returns (ListProductsResponse);
    rpc CreateProduct(CreateProductRequest) returns (ProductResponse);
    rpc UpdateProduct(UpdateProductRequest) returns (ProductResponse);
    rpc DeleteProduct(DeleteProductRequest) returns (BaseResponse);
}
```

gRPC는 마이크로서비스 간 통신에 매우 효율적인 프로토콜입니다. 강타입과 고성능을 제공하며, 다양한 언어 간 상호 운용성을 보장합니다.