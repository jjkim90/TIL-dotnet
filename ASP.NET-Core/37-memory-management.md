# ASP.NET Core 메모리 관리

## 메모리 관리 기초

ASP.NET Core 애플리케이션의 성능과 안정성을 위해서는 효율적인 메모리 관리가 필수적입니다. .NET 런타임은 자동 메모리 관리를 제공하지만, 개발자가 메모리 사용 패턴을 이해하고 최적화하는 것이 중요합니다.

### .NET 메모리 구조
```csharp
// 스택 메모리: 값 타입, 메서드 매개변수, 로컬 변수
public void StackExample()
{
    int value = 42;              // 스택에 할당
    DateTime now = DateTime.Now; // 구조체도 스택에 할당
}

// 힙 메모리: 참조 타입, 클래스 인스턴스
public void HeapExample()
{
    string text = "Hello";       // 힙에 할당
    var list = new List<int>();  // 힙에 할당
    var array = new int[100];    // 배열도 힙에 할당
}
```

### 메모리 세대 (Generation) 개념
```csharp
public class GenerationExample
{
    public void DemonstrateGenerations()
    {
        // 새로 생성된 객체는 Gen 0에 할당
        var shortLived = new byte[1024];
        
        // GC를 여러 번 살아남으면 Gen 1, Gen 2로 승격
        var longLived = new List<string>();
        for (int i = 0; i < 1000; i++)
        {
            longLived.Add($"Item {i}");
        }
        
        // 현재 세대 확인
        Console.WriteLine($"shortLived Generation: {GC.GetGeneration(shortLived)}");
        Console.WriteLine($"longLived Generation: {GC.GetGeneration(longLived)}");
    }
}
```

## 가비지 컬렉션 전략

### GC 모드 설정
```json
// appsettings.json 또는 runtimeconfig.json
{
  "configProperties": {
    "System.GC.Server": true,           // 서버 GC 사용
    "System.GC.Concurrent": true,       // 동시 GC 활성화
    "System.GC.RetainVM": false,        // 메모리 반환 설정
    "System.GC.HeapCount": 4,           // 힙 개수 지정
    "System.GC.HeapAffinitizeMask": 15 // CPU 친화성 마스크
  }
}
```

### 프로그래밍 방식 GC 구성
```csharp
public class GarbageCollectionConfig
{
    public void ConfigureGC()
    {
        // GC 설정 확인
        var isServerGC = GCSettings.IsServerGC;
        var latencyMode = GCSettings.LatencyMode;
        
        Console.WriteLine($"Server GC: {isServerGC}");
        Console.WriteLine($"Latency Mode: {latencyMode}");
        
        // 낮은 지연 시간 모드 설정
        GCSettings.LatencyMode = GCLatencyMode.SustainedLowLatency;
    }
    
    public void ManualGCControl()
    {
        // 강제 가비지 컬렉션 (권장하지 않음)
        GC.Collect();
        GC.WaitForPendingFinalizers();
        GC.Collect();
        
        // 특정 세대만 수집
        GC.Collect(0, GCCollectionMode.Forced);
        
        // 메모리 압력 추가/제거
        GC.AddMemoryPressure(1024 * 1024); // 1MB
        GC.RemoveMemoryPressure(1024 * 1024);
    }
}
```

## 메모리 프로파일링과 진단

### 메모리 사용량 모니터링
```csharp
public class MemoryMonitoring
{
    private readonly ILogger<MemoryMonitoring> _logger;
    
    public MemoryMonitoring(ILogger<MemoryMonitoring> logger)
    {
        _logger = logger;
    }
    
    public void MonitorMemory()
    {
        // GC 메모리 정보
        var gcInfo = GC.GetTotalMemory(false);
        var gen0Count = GC.CollectionCount(0);
        var gen1Count = GC.CollectionCount(1);
        var gen2Count = GC.CollectionCount(2);
        
        _logger.LogInformation($"Total Memory: {gcInfo / 1024 / 1024} MB");
        _logger.LogInformation($"Gen 0 Collections: {gen0Count}");
        _logger.LogInformation($"Gen 1 Collections: {gen1Count}");
        _logger.LogInformation($"Gen 2 Collections: {gen2Count}");
        
        // Process 메모리 정보
        using var process = Process.GetCurrentProcess();
        var workingSet = process.WorkingSet64;
        var privateMemory = process.PrivateMemorySize64;
        var virtualMemory = process.VirtualMemorySize64;
        
        _logger.LogInformation($"Working Set: {workingSet / 1024 / 1024} MB");
        _logger.LogInformation($"Private Memory: {privateMemory / 1024 / 1024} MB");
        _logger.LogInformation($"Virtual Memory: {virtualMemory / 1024 / 1024} MB");
    }
}
```

### 진단 도구 통합
```csharp
public class DiagnosticsSetup
{
    public void ConfigureDiagnostics(IServiceCollection services)
    {
        // EventCounters 활성화
        services.AddSingleton<IHostedService, EventCounterService>();
        
        // 메모리 덤프 엔드포인트 추가
        services.Configure<KestrelServerOptions>(options =>
        {
            options.Limits.MaxRequestBodySize = 100 * 1024 * 1024; // 100MB
        });
    }
}

public class EventCounterService : IHostedService
{
    private Timer _timer;
    private readonly ILogger<EventCounterService> _logger;
    
    public EventCounterService(ILogger<EventCounterService> logger)
    {
        _logger = logger;
    }
    
    public Task StartAsync(CancellationToken cancellationToken)
    {
        _timer = new Timer(CollectMetrics, null, TimeSpan.Zero, TimeSpan.FromMinutes(1));
        return Task.CompletedTask;
    }
    
    private void CollectMetrics(object state)
    {
        var eventSource = new EventSource("Microsoft-Windows-DotNETRuntime");
        
        // 메모리 관련 카운터 수집
        var gcHeapSize = EventCounter.GetCounter("gc-heap-size");
        var gen0GcCount = EventCounter.GetCounter("gen-0-gc-count");
        var exceptionCount = EventCounter.GetCounter("exception-count");
        
        _logger.LogInformation($"GC Heap Size: {gcHeapSize}");
        _logger.LogInformation($"Gen 0 GC Count: {gen0GcCount}");
        _logger.LogInformation($"Exception Count: {exceptionCount}");
    }
    
    public Task StopAsync(CancellationToken cancellationToken)
    {
        _timer?.Dispose();
        return Task.CompletedTask;
    }
}
```

## 객체 풀링과 ArrayPool

### ObjectPool 구현
```csharp
public class DatabaseConnection
{
    public string ConnectionString { get; set; }
    public bool IsOpen { get; set; }
    
    public void Open() => IsOpen = true;
    public void Close() => IsOpen = false;
    public void Reset() => IsOpen = false;
}

public class DatabaseConnectionPooledObjectPolicy : IPooledObjectPolicy<DatabaseConnection>
{
    public DatabaseConnection Create()
    {
        return new DatabaseConnection();
    }
    
    public bool Return(DatabaseConnection obj)
    {
        obj.Reset();
        return true;
    }
}

// Startup.cs에서 구성
public void ConfigureServices(IServiceCollection services)
{
    // ObjectPool 등록
    services.AddSingleton<ObjectPoolProvider, DefaultObjectPoolProvider>();
    services.AddSingleton(serviceProvider =>
    {
        var provider = serviceProvider.GetRequiredService<ObjectPoolProvider>();
        var policy = new DatabaseConnectionPooledObjectPolicy();
        return provider.Create(policy);
    });
}

// 사용 예제
public class DatabaseService
{
    private readonly ObjectPool<DatabaseConnection> _connectionPool;
    
    public DatabaseService(ObjectPool<DatabaseConnection> connectionPool)
    {
        _connectionPool = connectionPool;
    }
    
    public async Task<string> GetDataAsync()
    {
        var connection = _connectionPool.Get();
        try
        {
            connection.Open();
            // 데이터베이스 작업 수행
            return "data";
        }
        finally
        {
            connection.Close();
            _connectionPool.Return(connection);
        }
    }
}
```

### ArrayPool 사용
```csharp
public class ArrayPoolExample
{
    public byte[] ProcessLargeData(Stream stream)
    {
        // ArrayPool에서 배열 대여
        var pool = ArrayPool<byte>.Shared;
        var buffer = pool.Rent(4096);
        
        try
        {
            var totalBytes = 0;
            var bytesRead = 0;
            
            while ((bytesRead = stream.Read(buffer, 0, buffer.Length)) > 0)
            {
                // 데이터 처리
                ProcessBuffer(buffer, bytesRead);
                totalBytes += bytesRead;
            }
            
            return buffer[..totalBytes];
        }
        finally
        {
            // 배열 반환
            pool.Return(buffer, clearArray: true);
        }
    }
    
    private void ProcessBuffer(byte[] buffer, int count)
    {
        // 버퍼 처리 로직
    }
}

// 커스텀 ArrayPool
public class CustomArrayPoolExample
{
    private readonly ArrayPool<byte> _customPool;
    
    public CustomArrayPoolExample()
    {
        _customPool = ArrayPool<byte>.Create(
            maxArrayLength: 1024 * 1024,  // 최대 1MB
            maxArraysPerBucket: 50         // 버킷당 최대 50개
        );
    }
    
    public void UseCustomPool()
    {
        var array = _customPool.Rent(1024);
        try
        {
            // 배열 사용
        }
        finally
        {
            _customPool.Return(array);
        }
    }
}
```

## 메모리 효율적인 데이터 구조

### 구조체 vs 클래스 선택
```csharp
// 작은 데이터는 구조체 사용 (스택 할당)
public struct Point
{
    public double X { get; set; }
    public double Y { get; set; }
}

// 큰 데이터나 참조가 필요한 경우 클래스 사용
public class LargeDataContainer
{
    public byte[] Data { get; set; }
    public Dictionary<string, object> Metadata { get; set; }
}

// 메모리 효율적인 컬렉션 사용
public class EfficientCollections
{
    public void UseEfficientStructures()
    {
        // List<T> 대신 용량 지정
        var list = new List<int>(1000);
        
        // Dictionary 초기 용량 설정
        var dict = new Dictionary<string, string>(100);
        
        // 읽기 전용인 경우 ImmutableArray 사용
        var immutableArray = ImmutableArray.Create(1, 2, 3, 4, 5);
        
        // 큰 컬렉션은 LinkedList 고려
        var linkedList = new LinkedList<string>();
    }
}
```

### 메모리 맵 파일
```csharp
public class MemoryMappedFileExample
{
    public void UseMemoryMappedFile()
    {
        using var mmf = MemoryMappedFile.CreateFromFile(
            "largefile.dat",
            FileMode.Open,
            "MyMemoryMappedFile",
            0,
            MemoryMappedFileAccess.Read
        );
        
        using var accessor = mmf.CreateViewAccessor(0, 1024);
        byte value = accessor.ReadByte(0);
        
        // 스트림으로 접근
        using var stream = mmf.CreateViewStream();
        var buffer = new byte[1024];
        stream.Read(buffer, 0, buffer.Length);
    }
}
```

## Span<T>와 Memory<T> 활용

### Span<T> 기본 사용
```csharp
public class SpanExample
{
    public void BasicSpanUsage()
    {
        // 배열에서 Span 생성
        int[] array = { 1, 2, 3, 4, 5 };
        Span<int> span = array.AsSpan();
        
        // 슬라이싱
        Span<int> slice = span.Slice(1, 3); // [2, 3, 4]
        
        // 스택 할당
        Span<byte> stackSpan = stackalloc byte[128];
        
        // 문자열 처리
        ReadOnlySpan<char> text = "Hello, World!".AsSpan();
        var hello = text.Slice(0, 5);
    }
    
    public int SumWithSpan(ReadOnlySpan<int> numbers)
    {
        int sum = 0;
        foreach (var number in numbers)
        {
            sum += number;
        }
        return sum;
    }
}
```

### Memory<T>와 비동기 작업
```csharp
public class MemoryExample
{
    public async Task ProcessDataAsync(Memory<byte> buffer)
    {
        // Memory<T>는 비동기 메서드에서 사용 가능
        await SomeAsyncOperation(buffer);
        
        // Span으로 변환하여 동기 작업
        ProcessSpan(buffer.Span);
    }
    
    private void ProcessSpan(Span<byte> span)
    {
        for (int i = 0; i < span.Length; i++)
        {
            span[i] = (byte)(span[i] ^ 0xFF); // XOR 연산
        }
    }
    
    private Task SomeAsyncOperation(Memory<byte> memory)
    {
        return Task.CompletedTask;
    }
}

// 파이프라인에서 Memory<T> 사용
public class PipelineExample
{
    public async Task ProcessPipelineAsync(PipeReader reader)
    {
        while (true)
        {
            ReadResult result = await reader.ReadAsync();
            ReadOnlySequence<byte> buffer = result.Buffer;
            
            foreach (var segment in buffer)
            {
                ProcessMemory(segment);
            }
            
            reader.AdvanceTo(buffer.End);
            
            if (result.IsCompleted)
                break;
        }
    }
    
    private void ProcessMemory(ReadOnlyMemory<byte> memory)
    {
        // 메모리 세그먼트 처리
    }
}
```

## 메모리 캐싱 전략

### IMemoryCache 구현
```csharp
public class MemoryCacheService
{
    private readonly IMemoryCache _cache;
    private readonly ILogger<MemoryCacheService> _logger;
    
    public MemoryCacheService(IMemoryCache cache, ILogger<MemoryCacheService> logger)
    {
        _cache = cache;
        _logger = logger;
    }
    
    public async Task<T> GetOrCreateAsync<T>(string key, Func<Task<T>> factory, MemoryCacheEntryOptions options = null)
    {
        if (_cache.TryGetValue(key, out T cachedValue))
        {
            _logger.LogDebug($"Cache hit for key: {key}");
            return cachedValue;
        }
        
        _logger.LogDebug($"Cache miss for key: {key}");
        
        var value = await factory();
        
        var cacheOptions = options ?? new MemoryCacheEntryOptions()
            .SetSlidingExpiration(TimeSpan.FromMinutes(5))
            .SetAbsoluteExpiration(TimeSpan.FromHours(1))
            .SetPriority(CacheItemPriority.Normal)
            .SetSize(1);
        
        _cache.Set(key, value, cacheOptions);
        
        return value;
    }
}

// 캐시 크기 제한 설정
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddMemoryCache(options =>
        {
            options.SizeLimit = 1000; // 최대 1000개 항목
            options.CompactionPercentage = 0.25; // 25% 압축
            options.ExpirationScanFrequency = TimeSpan.FromMinutes(5);
        });
    }
}
```

### 캐시 무효화 전략
```csharp
public class CacheInvalidationService
{
    private readonly IMemoryCache _cache;
    private readonly Dictionary<string, HashSet<string>> _taggedKeys;
    
    public CacheInvalidationService(IMemoryCache cache)
    {
        _cache = cache;
        _taggedKeys = new Dictionary<string, HashSet<string>>();
    }
    
    public void SetWithTags(string key, object value, string[] tags, MemoryCacheEntryOptions options)
    {
        _cache.Set(key, value, options);
        
        foreach (var tag in tags)
        {
            if (!_taggedKeys.ContainsKey(tag))
            {
                _taggedKeys[tag] = new HashSet<string>();
            }
            _taggedKeys[tag].Add(key);
        }
    }
    
    public void InvalidateByTag(string tag)
    {
        if (_taggedKeys.TryGetValue(tag, out var keys))
        {
            foreach (var key in keys)
            {
                _cache.Remove(key);
            }
            _taggedKeys.Remove(tag);
        }
    }
}

// 변경 토큰을 사용한 캐시 무효화
public class ChangeTokenCacheExample
{
    private readonly IMemoryCache _cache;
    private CancellationTokenSource _resetToken = new();
    
    public async Task<string> GetDataAsync(string key)
    {
        var cts = new CancellationTokenSource();
        var options = new MemoryCacheEntryOptions()
            .AddExpirationToken(new CancellationChangeToken(_resetToken.Token))
            .RegisterPostEvictionCallback(PostEvictionCallback);
        
        return await _cache.GetOrCreateAsync(key, async entry =>
        {
            entry.SetOptions(options);
            return await LoadDataAsync(key);
        });
    }
    
    public void ResetCache()
    {
        _resetToken.Cancel();
        _resetToken = new CancellationTokenSource();
    }
    
    private void PostEvictionCallback(object key, object value, EvictionReason reason, object state)
    {
        Console.WriteLine($"Cache entry {key} was evicted. Reason: {reason}");
    }
    
    private Task<string> LoadDataAsync(string key) => Task.FromResult($"Data for {key}");
}
```

## 메모리 누수 감지와 예방

### 일반적인 메모리 누수 패턴
```csharp
public class MemoryLeakExamples
{
    // 잘못된 예: 이벤트 핸들러 미해제
    public class EventLeakExample
    {
        public event EventHandler SomeEvent;
        
        public void SubscribeToEvent()
        {
            var handler = new EventHandlerClass();
            SomeEvent += handler.HandleEvent; // 누수 발생 가능
        }
    }
    
    // 올바른 예: IDisposable 구현
    public class EventHandlerClass : IDisposable
    {
        private EventLeakExample _eventSource;
        
        public void Subscribe(EventLeakExample eventSource)
        {
            _eventSource = eventSource;
            _eventSource.SomeEvent += HandleEvent;
        }
        
        public void HandleEvent(object sender, EventArgs e) { }
        
        public void Dispose()
        {
            if (_eventSource != null)
            {
                _eventSource.SomeEvent -= HandleEvent;
                _eventSource = null;
            }
        }
    }
}

// HttpClient 누수 방지
public class HttpClientService
{
    private readonly IHttpClientFactory _httpClientFactory;
    
    public HttpClientService(IHttpClientFactory httpClientFactory)
    {
        _httpClientFactory = httpClientFactory;
    }
    
    public async Task<string> GetDataAsync(string url)
    {
        // HttpClient 인스턴스를 직접 생성하지 않음
        var client = _httpClientFactory.CreateClient();
        return await client.GetStringAsync(url);
    }
}
```

### 메모리 누수 진단
```csharp
public class MemoryLeakDetector
{
    private readonly ILogger<MemoryLeakDetector> _logger;
    private long _lastGen2Count;
    private long _lastTotalMemory;
    
    public MemoryLeakDetector(ILogger<MemoryLeakDetector> logger)
    {
        _logger = logger;
    }
    
    public void CheckForLeaks()
    {
        var currentGen2Count = GC.CollectionCount(2);
        var currentTotalMemory = GC.GetTotalMemory(false);
        
        if (currentGen2Count > _lastGen2Count)
        {
            var memoryGrowth = currentTotalMemory - _lastTotalMemory;
            if (memoryGrowth > 10 * 1024 * 1024) // 10MB 이상 증가
            {
                _logger.LogWarning($"Potential memory leak detected. Memory growth: {memoryGrowth / 1024 / 1024} MB");
            }
        }
        
        _lastGen2Count = currentGen2Count;
        _lastTotalMemory = currentTotalMemory;
    }
}

// WeakReference를 사용한 객체 추적
public class ObjectTracker<T> where T : class
{
    private readonly List<WeakReference> _trackedObjects = new();
    
    public void Track(T obj)
    {
        _trackedObjects.Add(new WeakReference(obj));
    }
    
    public int GetAliveCount()
    {
        return _trackedObjects.Count(wr => wr.IsAlive);
    }
    
    public void CleanupDeadReferences()
    {
        _trackedObjects.RemoveAll(wr => !wr.IsAlive);
    }
}
```

## 성능 카운터와 모니터링

### 커스텀 성능 카운터
```csharp
public class PerformanceMetrics
{
    private readonly Meter _meter;
    private readonly Counter<long> _requestCounter;
    private readonly Histogram<double> _requestDuration;
    private readonly ObservableGauge<long> _memoryGauge;
    
    public PerformanceMetrics()
    {
        _meter = new Meter("MyApp.Performance", "1.0");
        
        _requestCounter = _meter.CreateCounter<long>(
            "myapp_requests_total",
            description: "Total number of requests"
        );
        
        _requestDuration = _meter.CreateHistogram<double>(
            "myapp_request_duration_ms",
            description: "Request duration in milliseconds"
        );
        
        _memoryGauge = _meter.CreateObservableGauge<long>(
            "myapp_memory_usage_bytes",
            () => GC.GetTotalMemory(false),
            description: "Current memory usage in bytes"
        );
    }
    
    public void RecordRequest(string endpoint, double durationMs)
    {
        _requestCounter.Add(1, new KeyValuePair<string, object>("endpoint", endpoint));
        _requestDuration.Record(durationMs, new KeyValuePair<string, object>("endpoint", endpoint));
    }
}

// ASP.NET Core 미들웨어로 통합
public class PerformanceMiddleware
{
    private readonly RequestDelegate _next;
    private readonly PerformanceMetrics _metrics;
    
    public PerformanceMiddleware(RequestDelegate next, PerformanceMetrics metrics)
    {
        _next = next;
        _metrics = metrics;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        var stopwatch = Stopwatch.StartNew();
        
        try
        {
            await _next(context);
        }
        finally
        {
            stopwatch.Stop();
            _metrics.RecordRequest(context.Request.Path, stopwatch.ElapsedMilliseconds);
        }
    }
}
```

### 실시간 모니터링 엔드포인트
```csharp
[ApiController]
[Route("api/[controller]")]
public class DiagnosticsController : ControllerBase
{
    private readonly ILogger<DiagnosticsController> _logger;
    
    public DiagnosticsController(ILogger<DiagnosticsController> logger)
    {
        _logger = logger;
    }
    
    [HttpGet("memory")]
    public IActionResult GetMemoryInfo()
    {
        var memoryInfo = new
        {
            TotalMemory = GC.GetTotalMemory(false) / 1024 / 1024,
            Gen0Collections = GC.CollectionCount(0),
            Gen1Collections = GC.CollectionCount(1),
            Gen2Collections = GC.CollectionCount(2),
            WorkingSet = Process.GetCurrentProcess().WorkingSet64 / 1024 / 1024,
            GCMode = new
            {
                IsServerGC = GCSettings.IsServerGC,
                LatencyMode = GCSettings.LatencyMode.ToString(),
                LOHCompactionMode = GCSettings.LargeObjectHeapCompactionMode.ToString()
            }
        };
        
        return Ok(memoryInfo);
    }
    
    [HttpPost("gc")]
    public IActionResult ForceGC([FromQuery] int generation = -1)
    {
        if (generation < 0)
        {
            GC.Collect();
        }
        else
        {
            GC.Collect(generation, GCCollectionMode.Forced);
        }
        
        return Ok("GC triggered");
    }
}
```

## 메모리 최적화 모범 사례

### 문자열 처리 최적화
```csharp
public class StringOptimization
{
    // StringBuilder 사용
    public string ConcatenateMany(string[] values)
    {
        var sb = new StringBuilder();
        foreach (var value in values)
        {
            sb.Append(value);
        }
        return sb.ToString();
    }
    
    // String.Create 사용 (C# 10+)
    public string FormatEfficiently(int id, string name)
    {
        return String.Create(name.Length + 10, (id, name), (span, state) =>
        {
            var (id, name) = state;
            id.TryFormat(span, out int written);
            span = span[written..];
            name.AsSpan().CopyTo(span);
        });
    }
    
    // StringPool 사용
    public class StringPoolExample
    {
        private readonly HashSet<string> _stringPool = new();
        
        public string Intern(string value)
        {
            if (_stringPool.TryGetValue(value, out var pooled))
            {
                return pooled;
            }
            
            _stringPool.Add(value);
            return value;
        }
    }
}
```

### 비동기 스트림 처리
```csharp
public class AsyncStreamProcessing
{
    public async IAsyncEnumerable<ProcessedData> ProcessLargeDataSetAsync(
        Stream dataStream,
        [EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        using var reader = new StreamReader(dataStream);
        var buffer = new char[4096];
        
        while (!reader.EndOfStream)
        {
            cancellationToken.ThrowIfCancellationRequested();
            
            var charsRead = await reader.ReadAsync(buffer, cancellationToken);
            var data = ProcessBuffer(buffer.AsMemory(0, charsRead));
            
            yield return data;
        }
    }
    
    private ProcessedData ProcessBuffer(ReadOnlyMemory<char> buffer)
    {
        // 버퍼 처리 로직
        return new ProcessedData();
    }
}

public class ProcessedData { }
```

### 대용량 데이터 처리
```csharp
public class LargeDataProcessor
{
    private readonly IMemoryCache _cache;
    private readonly ArrayPool<byte> _arrayPool;
    
    public LargeDataProcessor(IMemoryCache cache)
    {
        _cache = cache;
        _arrayPool = ArrayPool<byte>.Create();
    }
    
    public async Task ProcessFileInChunksAsync(string filePath)
    {
        const int chunkSize = 1024 * 1024; // 1MB chunks
        
        await using var fileStream = new FileStream(
            filePath, 
            FileMode.Open, 
            FileAccess.Read, 
            FileShare.Read, 
            bufferSize: 4096, 
            useAsync: true
        );
        
        var buffer = _arrayPool.Rent(chunkSize);
        
        try
        {
            int bytesRead;
            while ((bytesRead = await fileStream.ReadAsync(buffer, 0, chunkSize)) > 0)
            {
                await ProcessChunkAsync(buffer.AsMemory(0, bytesRead));
            }
        }
        finally
        {
            _arrayPool.Return(buffer, clearArray: true);
        }
    }
    
    private async Task ProcessChunkAsync(Memory<byte> chunk)
    {
        // 청크 처리 로직
        await Task.Yield();
    }
}
```

### 메모리 친화적 API 설계
```csharp
[ApiController]
[Route("api/[controller]")]
public class StreamingController : ControllerBase
{
    [HttpPost("upload")]
    [DisableRequestSizeLimit]
    public async Task<IActionResult> UploadLargeFile()
    {
        if (!Request.HasFormContentType)
        {
            return BadRequest();
        }
        
        var boundary = Request.GetMultipartBoundary();
        var reader = new MultipartReader(boundary, Request.Body);
        
        var section = await reader.ReadNextSectionAsync();
        while (section != null)
        {
            if (ContentDispositionHeaderValue.TryParse(
                section.ContentDisposition, out var contentDisposition))
            {
                if (contentDisposition.HasFile())
                {
                    var fileName = contentDisposition.FileName.Value;
                    await ProcessFileStreamAsync(section.Body, fileName);
                }
            }
            
            section = await reader.ReadNextSectionAsync();
        }
        
        return Ok();
    }
    
    private async Task ProcessFileStreamAsync(Stream stream, string fileName)
    {
        // 스트림 직접 처리, 메모리에 전체 로드하지 않음
        using var fileStream = File.Create($"uploads/{fileName}");
        await stream.CopyToAsync(fileStream);
    }
}
```

### 구성 권장사항
```csharp
public class MemoryOptimizedStartup
{
    public void ConfigureServices(IServiceCollection services)
    {
        // Response 압축 활성화
        services.AddResponseCompression(options =>
        {
            options.EnableForHttps = true;
            options.Providers.Add<BrotliCompressionProvider>();
            options.Providers.Add<GzipCompressionProvider>();
        });
        
        // 메모리 캐시 구성
        services.AddMemoryCache(options =>
        {
            options.SizeLimit = 100_000;
            options.CompactionPercentage = 0.25;
        });
        
        // HttpClient 재사용
        services.AddHttpClient();
        
        // ObjectPool 등록
        services.AddSingleton<ObjectPoolProvider, DefaultObjectPoolProvider>();
        
        // 백그라운드 서비스로 메모리 모니터링
        services.AddHostedService<MemoryMonitoringService>();
    }
    
    public void Configure(IApplicationBuilder app)
    {
        // Response 압축 미들웨어
        app.UseResponseCompression();
        
        // 정적 파일 캐싱
        app.UseStaticFiles(new StaticFileOptions
        {
            OnPrepareResponse = ctx =>
            {
                ctx.Context.Response.Headers.Append(
                    "Cache-Control", "public,max-age=3600");
            }
        });
    }
}

public class MemoryMonitoringService : BackgroundService
{
    private readonly ILogger<MemoryMonitoringService> _logger;
    
    public MemoryMonitoringService(ILogger<MemoryMonitoringService> logger)
    {
        _logger = logger;
    }
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var memoryInfo = GC.GetTotalMemory(false) / 1024 / 1024;
            _logger.LogInformation($"Current memory usage: {memoryInfo} MB");
            
            await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
        }
    }
}
```