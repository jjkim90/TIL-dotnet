# 캐싱 전략 - In-Memory, Distributed, Response

## 캐싱 개요

캐싱은 자주 사용되는 데이터를 메모리나 빠른 저장소에 임시로 저장하여 애플리케이션 성능을 향상시키는 기술입니다. ASP.NET Core는 다양한 캐싱 메커니즘을 제공합니다.

### 캐싱의 장점
- **성능 향상**: 데이터베이스 쿼리 감소, 응답 시간 단축
- **확장성**: 서버 부하 감소로 더 많은 요청 처리 가능
- **비용 절감**: 데이터베이스 호출 감소로 인한 리소스 절약
- **가용성**: 원본 데이터 소스가 일시적으로 사용 불가능해도 서비스 제공

## In-Memory 캐싱

### 기본 In-Memory 캐싱 설정
```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// In-Memory 캐싱 서비스 등록
builder.Services.AddMemoryCache();

// 옵션과 함께 설정
builder.Services.AddMemoryCache(options =>
{
    options.SizeLimit = 1024; // 캐시 항목 수 제한
    options.CompactionPercentage = 0.25; // 압축 비율 (25%)
    options.ExpirationScanFrequency = TimeSpan.FromMinutes(5); // 만료 검사 주기
});

var app = builder.Build();
```

### IMemoryCache 사용
```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IMemoryCache _cache;
    private readonly IProductService _productService;
    private readonly ILogger<ProductsController> _logger;
    
    public ProductsController(
        IMemoryCache cache,
        IProductService productService,
        ILogger<ProductsController> logger)
    {
        _cache = cache;
        _productService = productService;
        _logger = logger;
    }
    
    [HttpGet("{id}")]
    public async Task<IActionResult> GetProduct(int id)
    {
        var cacheKey = $"product_{id}";
        
        // 캐시에서 조회 시도
        if (_cache.TryGetValue(cacheKey, out Product cachedProduct))
        {
            _logger.LogInformation("Product {Id} retrieved from cache", id);
            return Ok(cachedProduct);
        }
        
        // 캐시에 없으면 데이터베이스에서 조회
        var product = await _productService.GetByIdAsync(id);
        
        if (product == null)
        {
            return NotFound();
        }
        
        // 캐시에 저장
        var cacheOptions = new MemoryCacheEntryOptions()
            .SetSlidingExpiration(TimeSpan.FromMinutes(5)) // 슬라이딩 만료
            .SetAbsoluteExpiration(TimeSpan.FromHours(1)) // 절대 만료
            .SetPriority(CacheItemPriority.Normal)
            .SetSize(1); // 캐시 크기 (SizeLimit 사용 시)
        
        _cache.Set(cacheKey, product, cacheOptions);
        _logger.LogInformation("Product {Id} added to cache", id);
        
        return Ok(product);
    }
    
    [HttpPut("{id}")]
    public async Task<IActionResult> UpdateProduct(int id, [FromBody] ProductDto dto)
    {
        var product = await _productService.UpdateAsync(id, dto);
        
        if (product == null)
        {
            return NotFound();
        }
        
        // 캐시 무효화
        _cache.Remove($"product_{id}");
        _cache.Remove("products_all"); // 관련된 다른 캐시도 제거
        
        return Ok(product);
    }
}
```

### 고급 In-Memory 캐싱 패턴
```csharp
public class CachedProductService : IProductService
{
    private readonly IMemoryCache _cache;
    private readonly IProductRepository _repository;
    private readonly ILogger<CachedProductService> _logger;
    
    public CachedProductService(
        IMemoryCache cache,
        IProductRepository repository,
        ILogger<CachedProductService> logger)
    {
        _cache = cache;
        _repository = repository;
        _logger = logger;
    }
    
    public async Task<Product> GetByIdAsync(int id)
    {
        return await _cache.GetOrCreateAsync($"product_{id}", async entry =>
        {
            entry.SetOptions(GetCacheOptions());
            
            _logger.LogInformation("Loading product {Id} from database", id);
            return await _repository.GetByIdAsync(id);
        });
    }
    
    public async Task<IList<Product>> GetByCategoryAsync(string category)
    {
        var cacheKey = $"products_category_{category}";
        
        return await _cache.GetOrCreateAsync(cacheKey, async entry =>
        {
            entry.SetOptions(GetCacheOptions());
            
            // 캐시 종속성 설정
            using var cts = new CancellationTokenSource();
            entry.AddExpirationToken(new CancellationChangeToken(cts.Token));
            
            var products = await _repository.GetByCategoryAsync(category);
            
            // 각 제품도 개별적으로 캐시
            foreach (var product in products)
            {
                _cache.Set($"product_{product.Id}", product, GetCacheOptions());
            }
            
            return products;
        });
    }
    
    public async Task<Product> UpdateAsync(int id, ProductDto dto)
    {
        var product = await _repository.UpdateAsync(id, dto);
        
        if (product != null)
        {
            // 캐시 무효화 전략
            InvalidateProductCache(product);
        }
        
        return product;
    }
    
    private void InvalidateProductCache(Product product)
    {
        // 직접적인 캐시 제거
        _cache.Remove($"product_{product.Id}");
        
        // 카테고리 캐시 제거
        _cache.Remove($"products_category_{product.Category}");
        
        // 태그 기반 캐시 제거 (사용자 정의 구현)
        var cacheKeys = GetRelatedCacheKeys(product);
        foreach (var key in cacheKeys)
        {
            _cache.Remove(key);
        }
    }
    
    private MemoryCacheEntryOptions GetCacheOptions()
    {
        return new MemoryCacheEntryOptions()
            .SetSlidingExpiration(TimeSpan.FromMinutes(15))
            .SetAbsoluteExpiration(TimeSpan.FromHours(2))
            .SetPriority(CacheItemPriority.Normal)
            .RegisterPostEvictionCallback(OnCacheEviction);
    }
    
    private void OnCacheEviction(object key, object value, EvictionReason reason, object state)
    {
        _logger.LogInformation("Cache item {Key} was evicted. Reason: {Reason}", key, reason);
    }
}
```

## Distributed 캐싱

### Redis 캐싱 설정
```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Redis 분산 캐싱
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
    options.InstanceName = "MyApp"; // 키 접두사
});

// 또는 SQL Server 분산 캐싱
builder.Services.AddDistributedSqlServerCache(options =>
{
    options.ConnectionString = builder.Configuration.GetConnectionString("SqlServer");
    options.SchemaName = "dbo";
    options.TableName = "AppCache";
    options.DefaultSlidingExpiration = TimeSpan.FromMinutes(20);
});

// appsettings.json
{
  "ConnectionStrings": {
    "Redis": "localhost:6379,password=mypassword,ssl=false,connectTimeout=5000",
    "SqlServer": "Server=localhost;Database=MyApp;Trusted_Connection=true;"
  }
}
```

### IDistributedCache 사용
```csharp
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
    
    public async Task<T> GetAsync<T>(string key) where T : class
    {
        try
        {
            var cachedData = await _cache.GetStringAsync(key);
            
            if (string.IsNullOrEmpty(cachedData))
            {
                return null;
            }
            
            return JsonSerializer.Deserialize<T>(cachedData);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error retrieving cache key {Key}", key);
            return null;
        }
    }
    
    public async Task SetAsync<T>(string key, T value, DistributedCacheEntryOptions options = null)
    {
        try
        {
            var serializedData = JsonSerializer.Serialize(value);
            
            options ??= new DistributedCacheEntryOptions
            {
                SlidingExpiration = TimeSpan.FromMinutes(15),
                AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1)
            };
            
            await _cache.SetStringAsync(key, serializedData, options);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error setting cache key {Key}", key);
        }
    }
    
    public async Task<T> GetOrCreateAsync<T>(
        string key, 
        Func<Task<T>> factory, 
        DistributedCacheEntryOptions options = null) where T : class
    {
        var cached = await GetAsync<T>(key);
        
        if (cached != null)
        {
            return cached;
        }
        
        var value = await factory();
        
        if (value != null)
        {
            await SetAsync(key, value, options);
        }
        
        return value;
    }
    
    public async Task RemoveAsync(string key)
    {
        try
        {
            await _cache.RemoveAsync(key);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error removing cache key {Key}", key);
        }
    }
    
    public async Task RemoveByPrefixAsync(string prefix)
    {
        // Redis 전용 구현
        if (_cache is RedisCache redisCache)
        {
            var connection = await GetRedisConnection();
            var server = connection.GetServer(connection.GetEndPoints().First());
            var keys = server.Keys(pattern: $"{prefix}*");
            
            foreach (var key in keys)
            {
                await _cache.RemoveAsync(key);
            }
        }
    }
}
```

### 세션 상태 관리
```csharp
// Program.cs - 분산 세션 설정
builder.Services.AddSession(options =>
{
    options.IdleTimeout = TimeSpan.FromMinutes(30);
    options.Cookie.HttpOnly = true;
    options.Cookie.IsEssential = true;
    options.Cookie.SameSite = SameSiteMode.Strict;
});

// Redis를 세션 저장소로 사용
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = "localhost:6379";
});

var app = builder.Build();

app.UseSession();

// 세션 사용 예제
[ApiController]
[Route("api/[controller]")]
public class CartController : ControllerBase
{
    [HttpPost("add")]
    public IActionResult AddToCart([FromBody] CartItemDto item)
    {
        var cart = HttpContext.Session.GetObjectFromJson<List<CartItemDto>>("Cart") ?? new List<CartItemDto>();
        
        cart.Add(item);
        
        HttpContext.Session.SetObjectAsJson("Cart", cart);
        
        return Ok(new { itemCount = cart.Count });
    }
    
    [HttpGet]
    public IActionResult GetCart()
    {
        var cart = HttpContext.Session.GetObjectFromJson<List<CartItemDto>>("Cart") ?? new List<CartItemDto>();
        
        return Ok(cart);
    }
}

// Session Extension Methods
public static class SessionExtensions
{
    public static void SetObjectAsJson(this ISession session, string key, object value)
    {
        session.SetString(key, JsonSerializer.Serialize(value));
    }
    
    public static T GetObjectFromJson<T>(this ISession session, string key)
    {
        var value = session.GetString(key);
        return value == null ? default : JsonSerializer.Deserialize<T>(value);
    }
}
```

## Response 캐싱

### Response 캐싱 미들웨어 설정
```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Response 캐싱 서비스 추가
builder.Services.AddResponseCaching(options =>
{
    options.UseCaseSensitivePaths = false; // 대소문자 구분 안 함
    options.MaximumBodySize = 1024 * 1024 * 10; // 10MB
});

var app = builder.Build();

// Response 캐싱 미들웨어 사용 (UseRouting 전에)
app.UseResponseCaching();

// 캐시 제어 헤더 설정
app.Use(async (context, next) =>
{
    context.Response.GetTypedHeaders().CacheControl = 
        new Microsoft.Net.Http.Headers.CacheControlHeaderValue()
        {
            Public = true,
            MaxAge = TimeSpan.FromSeconds(60)
        };
    
    await next();
});
```

### ResponseCache 특성 사용
```csharp
[ApiController]
[Route("api/[controller]")]
public class NewsController : ControllerBase
{
    private readonly INewsService _newsService;
    
    public NewsController(INewsService newsService)
    {
        _newsService = newsService;
    }
    
    // 60초 동안 캐시
    [HttpGet("latest")]
    [ResponseCache(Duration = 60, Location = ResponseCacheLocation.Any)]
    public async Task<IActionResult> GetLatestNews()
    {
        var news = await _newsService.GetLatestAsync();
        return Ok(news);
    }
    
    // VaryByQueryKeys로 쿼리 파라미터별 캐싱
    [HttpGet]
    [ResponseCache(Duration = 300, VaryByQueryKeys = new[] { "category", "page" })]
    public async Task<IActionResult> GetNews(string category, int page = 1)
    {
        var news = await _newsService.GetByCategoryAsync(category, page);
        return Ok(news);
    }
    
    // VaryByHeader로 헤더별 캐싱
    [HttpGet("personalized")]
    [ResponseCache(Duration = 180, VaryByHeader = "User-Agent,Accept-Language")]
    public async Task<IActionResult> GetPersonalizedNews()
    {
        var userAgent = Request.Headers["User-Agent"].ToString();
        var language = Request.Headers["Accept-Language"].ToString();
        
        var news = await _newsService.GetPersonalizedAsync(userAgent, language);
        return Ok(news);
    }
    
    // 캐싱하지 않음
    [HttpGet("breaking")]
    [ResponseCache(NoStore = true, Location = ResponseCacheLocation.None)]
    public async Task<IActionResult> GetBreakingNews()
    {
        var news = await _newsService.GetBreakingNewsAsync();
        return Ok(news);
    }
}
```

### Response 캐싱 프로파일
```csharp
// Program.cs - 캐시 프로파일 정의
builder.Services.AddControllers(options =>
{
    options.CacheProfiles.Add("Default30",
        new CacheProfile()
        {
            Duration = 30,
            Location = ResponseCacheLocation.Any
        });
    
    options.CacheProfiles.Add("Never",
        new CacheProfile()
        {
            NoStore = true,
            Location = ResponseCacheLocation.None
        });
    
    options.CacheProfiles.Add("Mobile",
        new CacheProfile()
        {
            Duration = 60,
            Location = ResponseCacheLocation.Client,
            VaryByHeader = "User-Agent"
        });
});

// 컨트롤러에서 프로파일 사용
[ApiController]
[Route("api/[controller]")]
public class ContentController : ControllerBase
{
    [HttpGet("articles")]
    [ResponseCache(CacheProfileName = "Default30")]
    public IActionResult GetArticles()
    {
        return Ok(new { articles = new[] { "Article1", "Article2" } });
    }
    
    [HttpGet("user-data")]
    [ResponseCache(CacheProfileName = "Never")]
    public IActionResult GetUserData()
    {
        return Ok(new { userData = "Sensitive data" });
    }
}
```

## Output 캐싱 (.NET 7+)

### Output 캐싱 설정
```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Output 캐싱 서비스 추가
builder.Services.AddOutputCache(options =>
{
    // 기본 정책
    options.AddBasePolicy(builder => 
        builder.Expire(TimeSpan.FromSeconds(60)));
    
    // 명명된 정책
    options.AddPolicy("Expire30", builder => 
        builder.Expire(TimeSpan.FromSeconds(30)));
    
    // VaryBy 정책
    options.AddPolicy("VaryByCategory", builder => 
        builder.SetVaryByQuery("category")
               .Expire(TimeSpan.FromMinutes(5)));
    
    // 사용자 정의 정책
    options.AddPolicy("AuthenticatedUsers", builder =>
        builder.AddPolicy<AuthenticatedUsersCachePolicy>(), true);
});

var app = builder.Build();

app.UseOutputCache();

// 엔드포인트별 적용
app.MapGet("/weather", () => new { temp = 22, condition = "Sunny" })
   .CacheOutput("Expire30");

app.MapGet("/products", (string? category) => new { products = new[] { "Product1", "Product2" } })
   .CacheOutput("VaryByCategory");
```

### 사용자 정의 Output 캐싱 정책
```csharp
public class AuthenticatedUsersCachePolicy : IOutputCachePolicy
{
    public static readonly AuthenticatedUsersCachePolicy Instance = new();
    
    private AuthenticatedUsersCachePolicy() { }
    
    public ValueTask CacheRequestAsync(
        OutputCacheContext context, 
        CancellationToken cancellation)
    {
        var attemptOutputCaching = context.HttpContext.User.Identity.IsAuthenticated;
        context.EnableOutputCaching = attemptOutputCaching;
        
        if (attemptOutputCaching)
        {
            // 사용자별로 캐시 구분
            context.CacheVaryByValue.Add("user", context.HttpContext.User.Identity.Name);
        }
        
        return ValueTask.CompletedTask;
    }
    
    public ValueTask ServeFromCacheAsync(
        OutputCacheContext context, 
        CancellationToken cancellation)
    {
        return ValueTask.CompletedTask;
    }
    
    public ValueTask ServeResponseAsync(
        OutputCacheContext context, 
        CancellationToken cancellation)
    {
        context.AllowCacheLookup = true;
        context.AllowCacheStorage = true;
        context.AllowLocking = true;
        
        context.ResponseExpirationTimeSpan = TimeSpan.FromMinutes(5);
        
        return ValueTask.CompletedTask;
    }
}
```

## 캐싱 전략 패턴

### Cache-Aside 패턴
```csharp
public class CacheAsideService<T> where T : class
{
    private readonly IDistributedCache _cache;
    private readonly ILogger<CacheAsideService<T>> _logger;
    
    public CacheAsideService(IDistributedCache cache, ILogger<CacheAsideService<T>> logger)
    {
        _cache = cache;
        _logger = logger;
    }
    
    public async Task<T> GetAsync(string key, Func<Task<T>> dataRetriever)
    {
        // 1. 캐시에서 조회
        var cached = await GetFromCacheAsync(key);
        if (cached != null)
        {
            _logger.LogInformation("Cache hit for key: {Key}", key);
            return cached;
        }
        
        // 2. 캐시 미스 - 데이터 소스에서 조회
        _logger.LogInformation("Cache miss for key: {Key}", key);
        var data = await dataRetriever();
        
        // 3. 캐시에 저장
        if (data != null)
        {
            await SetCacheAsync(key, data);
        }
        
        return data;
    }
    
    private async Task<T> GetFromCacheAsync(string key)
    {
        try
        {
            var cached = await _cache.GetStringAsync(key);
            return cached == null ? null : JsonSerializer.Deserialize<T>(cached);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error reading from cache");
            return null;
        }
    }
    
    private async Task SetCacheAsync(string key, T data)
    {
        try
        {
            var options = new DistributedCacheEntryOptions
            {
                SlidingExpiration = TimeSpan.FromMinutes(15),
                AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1)
            };
            
            await _cache.SetStringAsync(key, JsonSerializer.Serialize(data), options);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error writing to cache");
            // 캐시 실패가 애플리케이션을 중단시키지 않도록 함
        }
    }
}
```

### Write-Through 캐싱
```csharp
public class WriteThroughCacheService
{
    private readonly IDistributedCache _cache;
    private readonly IRepository _repository;
    
    public async Task<T> UpdateAsync<T>(string key, T data) where T : class
    {
        // 1. 데이터베이스에 먼저 저장
        await _repository.UpdateAsync(key, data);
        
        // 2. 캐시 업데이트
        await _cache.SetStringAsync(key, JsonSerializer.Serialize(data));
        
        return data;
    }
}
```

### 캐시 워밍
```csharp
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
        _logger.LogInformation("Cache warming service started");
        
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                using var scope = _serviceProvider.CreateScope();
                var cache = scope.ServiceProvider.GetRequiredService<IDistributedCache>();
                var productService = scope.ServiceProvider.GetRequiredService<IProductService>();
                
                // 인기 상품 미리 캐싱
                var popularProducts = await productService.GetPopularProductsAsync();
                
                foreach (var product in popularProducts)
                {
                    var cacheKey = $"product_{product.Id}";
                    await cache.SetStringAsync(
                        cacheKey, 
                        JsonSerializer.Serialize(product),
                        new DistributedCacheEntryOptions
                        {
                            AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1)
                        });
                }
                
                _logger.LogInformation("Cache warmed with {Count} products", popularProducts.Count);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error during cache warming");
            }
            
            // 30분마다 실행
            await Task.Delay(TimeSpan.FromMinutes(30), stoppingToken);
        }
    }
}

// Program.cs에 등록
builder.Services.AddHostedService<CacheWarmingService>();
```

## 캐싱 모니터링과 관리

### 캐시 통계 수집
```csharp
public class CacheMetrics
{
    private long _hits;
    private long _misses;
    private long _evictions;
    
    public void RecordHit() => Interlocked.Increment(ref _hits);
    public void RecordMiss() => Interlocked.Increment(ref _misses);
    public void RecordEviction() => Interlocked.Increment(ref _evictions);
    
    public double GetHitRate()
    {
        var total = _hits + _misses;
        return total > 0 ? (double)_hits / total : 0;
    }
    
    public CacheStatistics GetStatistics()
    {
        return new CacheStatistics
        {
            Hits = _hits,
            Misses = _misses,
            Evictions = _evictions,
            HitRate = GetHitRate()
        };
    }
}

public class MonitoredCacheService
{
    private readonly IDistributedCache _cache;
    private readonly CacheMetrics _metrics;
    private readonly ILogger<MonitoredCacheService> _logger;
    
    public async Task<T> GetAsync<T>(string key) where T : class
    {
        var cached = await _cache.GetStringAsync(key);
        
        if (cached != null)
        {
            _metrics.RecordHit();
            return JsonSerializer.Deserialize<T>(cached);
        }
        
        _metrics.RecordMiss();
        return null;
    }
    
    // 관리 엔드포인트
    [ApiController]
    [Route("api/admin/cache")]
    public class CacheManagementController : ControllerBase
    {
        private readonly CacheMetrics _metrics;
        private readonly IDistributedCache _cache;
        
        [HttpGet("stats")]
        [Authorize(Roles = "Admin")]
        public IActionResult GetCacheStatistics()
        {
            return Ok(_metrics.GetStatistics());
        }
        
        [HttpDelete("{key}")]
        [Authorize(Roles = "Admin")]
        public async Task<IActionResult> InvalidateCache(string key)
        {
            await _cache.RemoveAsync(key);
            return Ok(new { message = $"Cache key '{key}' invalidated" });
        }
        
        [HttpPost("clear")]
        [Authorize(Roles = "Admin")]
        public async Task<IActionResult> ClearAllCache()
        {
            // Redis 전용 구현
            // 실제 구현에서는 주의해서 사용
            return Ok(new { message = "All cache cleared" });
        }
    }
}
```

## 모범 사례

### 캐싱 구성 관리
```csharp
// CacheOptions.cs
public class CacheOptions
{
    public const string Section = "Caching";
    
    public bool Enabled { get; set; } = true;
    public int DefaultExpirationMinutes { get; set; } = 15;
    public int MaxExpirationHours { get; set; } = 24;
    public Dictionary<string, CacheProfile> Profiles { get; set; } = new();
}

public class CacheProfile
{
    public int? ExpirationMinutes { get; set; }
    public bool? SlidingExpiration { get; set; }
    public CacheLocation Location { get; set; } = CacheLocation.Distributed;
}

// appsettings.json
{
  "Caching": {
    "Enabled": true,
    "DefaultExpirationMinutes": 15,
    "MaxExpirationHours": 24,
    "Profiles": {
      "ShortLived": {
        "ExpirationMinutes": 5,
        "SlidingExpiration": true
      },
      "LongLived": {
        "ExpirationMinutes": 60,
        "SlidingExpiration": false
      },
      "UserSpecific": {
        "ExpirationMinutes": 30,
        "Location": "InMemory"
      }
    }
  }
}

// 사용
public class ConfigurableCacheService
{
    private readonly IOptionsMonitor<CacheOptions> _options;
    private readonly IMemoryCache _memoryCache;
    private readonly IDistributedCache _distributedCache;
    
    public ConfigurableCacheService(
        IOptionsMonitor<CacheOptions> options,
        IMemoryCache memoryCache,
        IDistributedCache distributedCache)
    {
        _options = options;
        _memoryCache = memoryCache;
        _distributedCache = distributedCache;
    }
    
    public async Task<T> GetOrCreateAsync<T>(
        string key, 
        Func<Task<T>> factory, 
        string profileName = null) where T : class
    {
        var options = _options.CurrentValue;
        
        if (!options.Enabled)
        {
            return await factory();
        }
        
        var profile = GetCacheProfile(profileName);
        
        if (profile.Location == CacheLocation.InMemory)
        {
            return await GetOrCreateInMemoryAsync(key, factory, profile);
        }
        else
        {
            return await GetOrCreateDistributedAsync(key, factory, profile);
        }
    }
    
    private CacheProfile GetCacheProfile(string profileName)
    {
        var options = _options.CurrentValue;
        
        if (!string.IsNullOrEmpty(profileName) && 
            options.Profiles.TryGetValue(profileName, out var profile))
        {
            return profile;
        }
        
        return new CacheProfile
        {
            ExpirationMinutes = options.DefaultExpirationMinutes,
            SlidingExpiration = true,
            Location = CacheLocation.Distributed
        };
    }
}
```

### 캐시 키 생성 전략
```csharp
public static class CacheKeyGenerator
{
    private const string Separator = ":";
    
    public static string Generate(params object[] parts)
    {
        return string.Join(Separator, parts.Select(p => p?.ToString() ?? "null"));
    }
    
    public static string GenerateForUser(string userId, string resource, params object[] additionalParts)
    {
        var parts = new List<object> { "user", userId, resource };
        parts.AddRange(additionalParts);
        return Generate(parts.ToArray());
    }
    
    public static string GenerateForType<T>(params object[] parts)
    {
        var typeName = typeof(T).Name.ToLower();
        var allParts = new List<object> { typeName };
        allParts.AddRange(parts);
        return Generate(allParts.ToArray());
    }
    
    public static string GenerateWithHash(object complexObject)
    {
        var json = JsonSerializer.Serialize(complexObject);
        using var sha256 = SHA256.Create();
        var hash = sha256.ComputeHash(Encoding.UTF8.GetBytes(json));
        return Convert.ToBase64String(hash);
    }
}

// 사용 예제
var productKey = CacheKeyGenerator.GenerateForType<Product>(productId);
var userCartKey = CacheKeyGenerator.GenerateForUser(userId, "cart");
var complexKey = CacheKeyGenerator.GenerateWithHash(new { Category = "Electronics", MinPrice = 100 });
```

효과적인 캐싱 전략은 애플리케이션 성능을 크게 향상시킬 수 있습니다. 적절한 캐싱 레벨과 만료 정책을 선택하는 것이 중요합니다.