# Performance Optimization

## 성능 최적화 소개

ASP.NET Core 애플리케이션의 성능 최적화는 응답 시간 단축, 처리량 증가, 리소스 사용량 감소를 통해 사용자 경험을 개선하고 운영 비용을 절감하는 중요한 과정입니다.

### 성능 최적화의 주요 영역

1. **응답 시간**: 요청에서 응답까지의 시간 단축
2. **처리량**: 단위 시간당 처리할 수 있는 요청 수 증가
3. **리소스 효율성**: CPU, 메모리, 네트워크 사용량 최적화
4. **확장성**: 부하 증가에 대한 대응 능력 향상
5. **안정성**: 고부하 상황에서도 안정적인 서비스 제공

## 성능 측정과 프로파일링

### BenchmarkDotNet을 사용한 벤치마킹

```csharp
// Benchmarks/StringConcatenationBenchmark.cs
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;

[MemoryDiagnoser]
[SimpleJob(warmupCount: 3, targetCount: 5)]
public class StringConcatenationBenchmark
{
    private string[] _items;
    
    [GlobalSetup]
    public void Setup()
    {
        _items = Enumerable.Range(0, 1000).Select(i => $"Item{i}").ToArray();
    }
    
    [Benchmark(Baseline = true)]
    public string StringConcatenation()
    {
        var result = "";
        foreach (var item in _items)
        {
            result += item + ", ";
        }
        return result;
    }
    
    [Benchmark]
    public string StringBuilder()
    {
        var sb = new StringBuilder();
        foreach (var item in _items)
        {
            sb.Append(item).Append(", ");
        }
        return sb.ToString();
    }
    
    [Benchmark]
    public string StringJoin()
    {
        return string.Join(", ", _items);
    }
}

// Program.cs
class Program
{
    static void Main(string[] args)
    {
        var summary = BenchmarkRunner.Run<StringConcatenationBenchmark>();
    }
}
```

### Application Insights 통합

```csharp
// Program.cs
builder.Services.AddApplicationInsightsTelemetry(options =>
{
    options.ConnectionString = builder.Configuration["ApplicationInsights:ConnectionString"];
    options.EnableAdaptiveSampling = true;
    options.EnablePerformanceCounterCollectionModule = true;
    options.EnableDependencyTrackingTelemetryModule = true;
});

// 커스텀 메트릭
builder.Services.AddSingleton<ITelemetryInitializer, CustomTelemetryInitializer>();

// Telemetry/CustomTelemetryInitializer.cs
public class CustomTelemetryInitializer : ITelemetryInitializer
{
    public void Initialize(ITelemetry telemetry)
    {
        if (telemetry is RequestTelemetry requestTelemetry)
        {
            // 커스텀 속성 추가
            requestTelemetry.Properties["Environment"] = Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT");
            requestTelemetry.Properties["ServerName"] = Environment.MachineName;
        }
    }
}
```

## 메모리 최적화

### 객체 풀링

```csharp
// ObjectPools/StringBuilderPooledObjectPolicy.cs
public class StringBuilderPooledObjectPolicy : PooledObjectPolicy<StringBuilder>
{
    public int InitialCapacity { get; set; } = 256;
    public int MaximumRetainedCapacity { get; set; } = 4096;
    
    public override StringBuilder Create()
    {
        return new StringBuilder(InitialCapacity);
    }
    
    public override bool Return(StringBuilder obj)
    {
        if (obj.Capacity > MaximumRetainedCapacity)
        {
            // 너무 큰 객체는 풀에 반환하지 않음
            return false;
        }
        
        obj.Clear();
        return true;
    }
}

// Services/StringService.cs
public class StringService
{
    private readonly ObjectPool<StringBuilder> _stringBuilderPool;
    
    public StringService(ObjectPool<StringBuilder> stringBuilderPool)
    {
        _stringBuilderPool = stringBuilderPool;
    }
    
    public string ProcessLargeText(string[] items)
    {
        var sb = _stringBuilderPool.Get();
        try
        {
            foreach (var item in items)
            {
                sb.AppendLine(ProcessItem(item));
            }
            
            return sb.ToString();
        }
        finally
        {
            _stringBuilderPool.Return(sb);
        }
    }
    
    private string ProcessItem(string item)
    {
        // 처리 로직
        return item.ToUpper();
    }
}

// Program.cs
builder.Services.AddSingleton<ObjectPoolProvider, DefaultObjectPoolProvider>();
builder.Services.AddSingleton<ObjectPool<StringBuilder>>(serviceProvider =>
{
    var provider = serviceProvider.GetRequiredService<ObjectPoolProvider>();
    var policy = new StringBuilderPooledObjectPolicy();
    return provider.Create(policy);
});
```

### ArrayPool 사용

```csharp
// Services/DataProcessingService.cs
public class DataProcessingService
{
    private const int BufferSize = 4096;
    
    public async Task<byte[]> ProcessDataAsync(Stream inputStream)
    {
        var pool = ArrayPool<byte>.Shared;
        var buffer = pool.Rent(BufferSize);
        
        try
        {
            using var memoryStream = new MemoryStream();
            int bytesRead;
            
            while ((bytesRead = await inputStream.ReadAsync(buffer, 0, buffer.Length)) > 0)
            {
                // 데이터 처리
                ProcessBuffer(buffer, bytesRead);
                
                await memoryStream.WriteAsync(buffer, 0, bytesRead);
            }
            
            return memoryStream.ToArray();
        }
        finally
        {
            pool.Return(buffer, clearArray: true);
        }
    }
    
    private void ProcessBuffer(byte[] buffer, int length)
    {
        // 버퍼 처리 로직
        for (int i = 0; i < length; i++)
        {
            buffer[i] = (byte)(buffer[i] ^ 0xFF); // 예시: XOR 연산
        }
    }
}
```

### Span<T>와 Memory<T> 활용

```csharp
// Services/StringParsingService.cs
public class StringParsingService
{
    public List<int> ParseNumbers(string input)
    {
        var numbers = new List<int>();
        var span = input.AsSpan();
        
        int start = 0;
        for (int i = 0; i <= span.Length; i++)
        {
            if (i == span.Length || span[i] == ',')
            {
                if (i > start)
                {
                    var numberSpan = span.Slice(start, i - start);
                    if (int.TryParse(numberSpan, out int number))
                    {
                        numbers.Add(number);
                    }
                }
                start = i + 1;
            }
        }
        
        return numbers;
    }
    
    public async Task ProcessLargeFileAsync(string filePath)
    {
        const int bufferSize = 4096;
        using var fileStream = new FileStream(filePath, FileMode.Open, FileAccess.Read);
        using var reader = new StreamReader(fileStream);
        
        var buffer = new Memory<char>(new char[bufferSize]);
        
        while (!reader.EndOfStream)
        {
            var charsRead = await reader.ReadAsync(buffer);
            ProcessChunk(buffer.Slice(0, charsRead).Span);
        }
    }
    
    private void ProcessChunk(ReadOnlySpan<char> chunk)
    {
        // 청크 처리 로직
        int lineStart = 0;
        for (int i = 0; i < chunk.Length; i++)
        {
            if (chunk[i] == '\n')
            {
                var line = chunk.Slice(lineStart, i - lineStart);
                ProcessLine(line);
                lineStart = i + 1;
            }
        }
    }
    
    private void ProcessLine(ReadOnlySpan<char> line)
    {
        // 라인 처리 로직
    }
}
```

## 비동기 프로그래밍 최적화

### ValueTask 사용

```csharp
// Services/CachedDataService.cs
public class CachedDataService
{
    private readonly IMemoryCache _cache;
    private readonly IDataRepository _repository;
    
    public CachedDataService(IMemoryCache cache, IDataRepository repository)
    {
        _cache = cache;
        _repository = repository;
    }
    
    public ValueTask<Product?> GetProductAsync(int id)
    {
        if (_cache.TryGetValue($"product_{id}", out Product cachedProduct))
        {
            // 캐시 히트 - 동기적으로 반환
            return new ValueTask<Product?>(cachedProduct);
        }
        
        // 캐시 미스 - 비동기 작업
        return new ValueTask<Product?>(LoadProductAsync(id));
    }
    
    private async Task<Product?> LoadProductAsync(int id)
    {
        var product = await _repository.GetByIdAsync(id);
        
        if (product != null)
        {
            _cache.Set($"product_{id}", product, TimeSpan.FromMinutes(5));
        }
        
        return product;
    }
}
```

### 비동기 세마포어를 사용한 동시성 제어

```csharp
// Services/RateLimitedService.cs
public class RateLimitedService
{
    private readonly SemaphoreSlim _semaphore;
    private readonly HttpClient _httpClient;
    
    public RateLimitedService(HttpClient httpClient, int maxConcurrency = 10)
    {
        _httpClient = httpClient;
        _semaphore = new SemaphoreSlim(maxConcurrency, maxConcurrency);
    }
    
    public async Task<T> ExecuteAsync<T>(Func<Task<T>> operation)
    {
        await _semaphore.WaitAsync();
        try
        {
            return await operation();
        }
        finally
        {
            _semaphore.Release();
        }
    }
    
    public async Task<string> GetDataAsync(string url)
    {
        return await ExecuteAsync(async () =>
        {
            var response = await _httpClient.GetAsync(url);
            response.EnsureSuccessStatusCode();
            return await response.Content.ReadAsStringAsync();
        });
    }
}
```

### 채널을 사용한 생산자-소비자 패턴

```csharp
// Services/DataProcessingPipeline.cs
public class DataProcessingPipeline
{
    private readonly Channel<DataItem> _channel;
    private readonly ILogger<DataProcessingPipeline> _logger;
    
    public DataProcessingPipeline(ILogger<DataProcessingPipeline> logger)
    {
        _logger = logger;
        _channel = Channel.CreateUnbounded<DataItem>(new UnboundedChannelOptions
        {
            SingleReader = false,
            SingleWriter = false
        });
    }
    
    public async Task ProduceAsync(IEnumerable<DataItem> items)
    {
        var writer = _channel.Writer;
        
        foreach (var item in items)
        {
            await writer.WriteAsync(item);
        }
    }
    
    public async Task ConsumeAsync(int consumerCount, CancellationToken cancellationToken)
    {
        var tasks = Enumerable.Range(0, consumerCount)
            .Select(i => ConsumeInternalAsync(i, cancellationToken))
            .ToArray();
            
        await Task.WhenAll(tasks);
    }
    
    private async Task ConsumeInternalAsync(int consumerId, CancellationToken cancellationToken)
    {
        var reader = _channel.Reader;
        
        while (await reader.WaitToReadAsync(cancellationToken))
        {
            while (reader.TryRead(out var item))
            {
                try
                {
                    await ProcessItemAsync(item);
                    _logger.LogDebug($"Consumer {consumerId} processed item {item.Id}");
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex, $"Error processing item {item.Id}");
                }
            }
        }
    }
    
    private async Task ProcessItemAsync(DataItem item)
    {
        // 항목 처리 로직
        await Task.Delay(100); // 시뮬레이션
    }
}

public class DataItem
{
    public int Id { get; set; }
    public string Data { get; set; }
}
```

## 데이터베이스 최적화

### 연결 풀링 최적화

```csharp
// appsettings.json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MyApp;User Id=sa;Password=Pass@word;Min Pool Size=10;Max Pool Size=100;Connection Lifetime=300;"
  }
}

// DbContext 최적화
public class OptimizedDbContext : DbContext
{
    public OptimizedDbContext(DbContextOptions<OptimizedDbContext> options)
        : base(options)
    {
        // 변경 추적 비활성화 (읽기 전용 쿼리)
        ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.NoTracking;
        
        // 지연 로딩 프록시 비활성화
        ChangeTracker.LazyLoadingEnabled = false;
    }
    
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder
            .UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking)
            .EnableServiceProviderCaching()
            .EnableSensitiveDataLogging(false);
    }
}
```

### 쿼리 최적화

```csharp
// Repositories/OptimizedProductRepository.cs
public class OptimizedProductRepository
{
    private readonly OptimizedDbContext _context;
    
    public OptimizedProductRepository(OptimizedDbContext context)
    {
        _context = context;
    }
    
    // 프로젝션을 사용한 최적화
    public async Task<List<ProductDto>> GetProductListAsync()
    {
        return await _context.Products
            .Where(p => p.IsActive)
            .Select(p => new ProductDto
            {
                Id = p.Id,
                Name = p.Name,
                Price = p.Price
                // 필요한 필드만 선택
            })
            .ToListAsync();
    }
    
    // 분할 쿼리를 사용한 최적화
    public async Task<List<Order>> GetOrdersWithDetailsAsync()
    {
        return await _context.Orders
            .Include(o => o.OrderItems)
            .ThenInclude(oi => oi.Product)
            .AsSplitQuery() // 카테시안 곱 방지
            .Where(o => o.Status == OrderStatus.Active)
            .ToListAsync();
    }
    
    // 컴파일된 쿼리 사용
    private static readonly Func<OptimizedDbContext, int, Task<Product>> GetProductByIdQuery =
        EF.CompileAsyncQuery((OptimizedDbContext context, int id) =>
            context.Products.FirstOrDefault(p => p.Id == id));
            
    public Task<Product> GetProductByIdAsync(int id)
    {
        return GetProductByIdQuery(_context, id);
    }
    
    // 배치 작업 최적화
    public async Task BulkInsertProductsAsync(List<Product> products)
    {
        const int batchSize = 1000;
        
        for (int i = 0; i < products.Count; i += batchSize)
        {
            var batch = products.Skip(i).Take(batchSize);
            _context.Products.AddRange(batch);
            
            await _context.SaveChangesAsync();
            
            // 변경 추적기 정리
            _context.ChangeTracker.Clear();
        }
    }
}
```

### 읽기 전용 복제본 활용

```csharp
// Services/ReadWriteSplitService.cs
public class ReadWriteSplitService
{
    private readonly IDbContextFactory<WriteDbContext> _writeContextFactory;
    private readonly IDbContextFactory<ReadDbContext> _readContextFactory;
    
    public ReadWriteSplitService(
        IDbContextFactory<WriteDbContext> writeContextFactory,
        IDbContextFactory<ReadDbContext> readContextFactory)
    {
        _writeContextFactory = writeContextFactory;
        _readContextFactory = readContextFactory;
    }
    
    public async Task<Product> CreateProductAsync(Product product)
    {
        using var context = await _writeContextFactory.CreateDbContextAsync();
        context.Products.Add(product);
        await context.SaveChangesAsync();
        return product;
    }
    
    public async Task<List<Product>> GetProductsAsync()
    {
        using var context = await _readContextFactory.CreateDbContextAsync();
        return await context.Products
            .AsNoTracking()
            .ToListAsync();
    }
}
```

## 캐싱 전략

### 분산 캐싱 구현

```csharp
// Services/DistributedCacheService.cs
public class DistributedCacheService
{
    private readonly IDistributedCache _cache;
    private readonly ILogger<DistributedCacheService> _logger;
    
    public DistributedCacheService(
        IDistributedCache cache,
        ILogger<DistributedCacheService> logger)
    {
        _cache = cache;
        _logger = logger;
    }
    
    public async Task<T?> GetAsync<T>(string key) where T : class
    {
        try
        {
            var cachedData = await _cache.GetStringAsync(key);
            
            if (cachedData != null)
            {
                return JsonSerializer.Deserialize<T>(cachedData);
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error retrieving from cache: {Key}", key);
        }
        
        return null;
    }
    
    public async Task<T> GetOrSetAsync<T>(
        string key,
        Func<Task<T>> factory,
        DistributedCacheEntryOptions? options = null) where T : class
    {
        var cached = await GetAsync<T>(key);
        
        if (cached != null)
        {
            return cached;
        }
        
        var value = await factory();
        
        await SetAsync(key, value, options ?? new DistributedCacheEntryOptions
        {
            SlidingExpiration = TimeSpan.FromMinutes(5)
        });
        
        return value;
    }
    
    public async Task SetAsync<T>(
        string key,
        T value,
        DistributedCacheEntryOptions options) where T : class
    {
        try
        {
            var serialized = JsonSerializer.Serialize(value);
            await _cache.SetStringAsync(key, serialized, options);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error setting cache: {Key}", key);
        }
    }
    
    public async Task InvalidateAsync(string key)
    {
        await _cache.RemoveAsync(key);
    }
    
    public async Task InvalidateByPrefixAsync(string prefix)
    {
        // Redis 구현 예시
        if (_cache is RedisCache redisCache)
        {
            var connection = await GetRedisConnection(redisCache);
            var server = connection.GetServer(connection.GetEndPoints().First());
            var keys = server.Keys(pattern: $"{prefix}*");
            
            foreach (var key in keys)
            {
                await _cache.RemoveAsync(key);
            }
        }
    }
    
    private async Task<IConnectionMultiplexer> GetRedisConnection(RedisCache redisCache)
    {
        // Reflection을 사용해 private 연결 가져오기 (실제 구현에서는 더 나은 방법 사용)
        var connectionField = typeof(RedisCache).GetField("_connection", BindingFlags.NonPublic | BindingFlags.Instance);
        return (IConnectionMultiplexer)connectionField.GetValue(redisCache);
    }
}
```

### 캐시 워밍

```csharp
// Services/CacheWarmingService.cs
public class CacheWarmingService : BackgroundService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<CacheWarmingService> _logger;
    
    public CacheWarmingService(
        IServiceProvider serviceProvider,
        ILogger<CacheWarmingService> logger)
    {
        _serviceProvider = serviceProvider;
        _logger = logger;
    }
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // 앱 시작 시 초기 캐시 워밍
        await WarmCacheAsync();
        
        // 주기적으로 캐시 갱신
        using var timer = new PeriodicTimer(TimeSpan.FromMinutes(30));
        
        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            await WarmCacheAsync();
        }
    }
    
    private async Task WarmCacheAsync()
    {
        using var scope = _serviceProvider.CreateScope();
        var cacheService = scope.ServiceProvider.GetRequiredService<DistributedCacheService>();
        var productService = scope.ServiceProvider.GetRequiredService<IProductService>();
        
        try
        {
            _logger.LogInformation("Starting cache warming");
            
            // 인기 상품 캐싱
            var popularProducts = await productService.GetPopularProductsAsync(100);
            
            foreach (var product in popularProducts)
            {
                await cacheService.SetAsync(
                    $"product:{product.Id}",
                    product,
                    new DistributedCacheEntryOptions
                    {
                        SlidingExpiration = TimeSpan.FromHours(1)
                    });
            }
            
            _logger.LogInformation("Cache warming completed");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error during cache warming");
        }
    }
}
```

## HTTP 응답 압축

```csharp
// Program.cs
builder.Services.AddResponseCompression(options =>
{
    options.EnableForHttps = true;
    options.Providers.Add<BrotliCompressionProvider>();
    options.Providers.Add<GzipCompressionProvider>();
    options.MimeTypes = ResponseCompressionDefaults.MimeTypes.Concat(
        new[] { "application/json", "application/xml", "text/csv" });
});

builder.Services.Configure<BrotliCompressionProviderOptions>(options =>
{
    options.Level = CompressionLevel.Fastest;
});

builder.Services.Configure<GzipCompressionProviderOptions>(options =>
{
    options.Level = CompressionLevel.Optimal;
});

var app = builder.Build();

app.UseResponseCompression();
```

## 정적 파일 최적화

```csharp
// Program.cs
app.UseStaticFiles(new StaticFileOptions
{
    OnPrepareResponse = ctx =>
    {
        const int durationInSeconds = 60 * 60 * 24 * 365; // 1년
        ctx.Context.Response.Headers[HeaderNames.CacheControl] =
            $"public,max-age={durationInSeconds}";
            
        // ETag 지원
        var etag = $"\"{ctx.File.LastModified.ToFileTime()}\"";
        ctx.Context.Response.Headers[HeaderNames.ETag] = etag;
        
        // 조건부 요청 처리
        var ifNoneMatch = ctx.Context.Request.Headers[HeaderNames.IfNoneMatch];
        if (ifNoneMatch == etag)
        {
            ctx.Context.Response.StatusCode = StatusCodes.Status304NotModified;
            ctx.Context.Response.ContentLength = 0;
            ctx.Context.Response.Body = Stream.Null;
        }
    }
});

// 정적 파일 미들웨어 최적화
app.UseStaticFiles(new StaticFileOptions
{
    ServeUnknownFileTypes = false,
    DefaultContentType = "application/octet-stream",
    HttpsCompression = HttpsCompressionMode.Compress,
    FileProvider = new PhysicalFileProvider(
        Path.Combine(builder.Environment.WebRootPath, "static")),
    RequestPath = "/static"
});
```

## 로드 밸런싱과 확장

### Health Check 구현

```csharp
// HealthChecks/CustomHealthCheck.cs
public class CustomHealthCheck : IHealthCheck
{
    private readonly IDbContextFactory<ApplicationDbContext> _contextFactory;
    private readonly IMemoryCache _cache;
    
    public CustomHealthCheck(
        IDbContextFactory<ApplicationDbContext> contextFactory,
        IMemoryCache cache)
    {
        _contextFactory = contextFactory;
        _cache = cache;
    }
    
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        var data = new Dictionary<string, object>();
        
        try
        {
            // 데이터베이스 연결 확인
            using (var dbContext = await _contextFactory.CreateDbContextAsync(cancellationToken))
            {
                var canConnect = await dbContext.Database.CanConnectAsync(cancellationToken);
                data["database"] = canConnect ? "healthy" : "unhealthy";
                
                if (!canConnect)
                {
                    return HealthCheckResult.Unhealthy("Database connection failed", data: data);
                }
            }
            
            // 캐시 확인
            _cache.Set("health_check", DateTime.UtcNow);
            var cached = _cache.Get<DateTime>("health_check");
            data["cache"] = "healthy";
            
            // 디스크 공간 확인
            var drives = DriveInfo.GetDrives().Where(d => d.IsReady);
            foreach (var drive in drives)
            {
                var freeSpacePercent = (double)drive.AvailableFreeSpace / drive.TotalSize * 100;
                data[$"disk_{drive.Name}"] = $"{freeSpacePercent:F2}% free";
                
                if (freeSpacePercent < 10)
                {
                    return HealthCheckResult.Degraded("Low disk space", data: data);
                }
            }
            
            return HealthCheckResult.Healthy("All checks passed", data);
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Health check failed", ex, data);
        }
    }
}

// Program.cs
builder.Services.AddHealthChecks()
    .AddCheck<CustomHealthCheck>("custom_health_check")
    .AddDbContextCheck<ApplicationDbContext>()
    .AddRedis(builder.Configuration.GetConnectionString("Redis"))
    .AddUrlGroup(new Uri("https://api.external.com/health"), "external_api");

app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});
```

## 마무리

성능 최적화는 지속적인 과정이며, 측정과 프로파일링을 통해 병목 지점을 식별하고 개선해야 합니다. 메모리 효율성, 비동기 프로그래밍, 캐싱, 데이터베이스 최적화 등 다양한 기법을 적절히 조합하여 애플리케이션의 성능을 향상시킬 수 있습니다.