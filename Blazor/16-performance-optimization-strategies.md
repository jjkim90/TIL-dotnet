# 성능 최적화 전략

## 개요

Blazor 애플리케이션의 성능 최적화는 사용자 경험을 개선하고 서버 리소스를 효율적으로 사용하기 위해 필수적입니다. 이 장에서는 렌더링 최적화, 네트워크 최적화, 메모리 관리, 코드 분할 등 다양한 성능 최적화 기법을 학습합니다.

## 1. 렌더링 최적화

### 1.1 컴포넌트 렌더링 최적화

```csharp
// OptimizedComponent.razor
@implements IDisposable
@inject ILogger<OptimizedComponent> Logger

<div class="optimized-component">
    @if (shouldRender)
    {
        <h3>@Title</h3>
        <p>Render Count: @renderCount</p>
        
        @foreach (var item in FilteredItems)
        {
            <OptimizedChild Item="item" @key="item.Id" />
        }
    }
</div>

@code {
    [Parameter] public string Title { get; set; } = "";
    [Parameter] public List<DataItem> Items { get; set; } = new();
    [Parameter] public string Filter { get; set; } = "";
    
    private bool shouldRender = true;
    private int renderCount = 0;
    private List<DataItem> FilteredItems = new();
    private string lastFilter = "";
    
    protected override void OnInitialized()
    {
        Logger.LogInformation("Component initialized");
    }
    
    protected override void OnParametersSet()
    {
        // Only update filtered items if filter changed
        if (Filter != lastFilter)
        {
            FilteredItems = Items.Where(i => 
                string.IsNullOrEmpty(Filter) || 
                i.Name.Contains(Filter, StringComparison.OrdinalIgnoreCase))
                .ToList();
            lastFilter = Filter;
        }
    }
    
    protected override bool ShouldRender()
    {
        renderCount++;
        
        // Implement custom logic to prevent unnecessary renders
        var shouldRenderNow = base.ShouldRender() && shouldRender;
        
        if (shouldRenderNow)
        {
            Logger.LogDebug("Component rendering: {Count}", renderCount);
        }
        
        return shouldRenderNow;
    }
    
    public void Dispose()
    {
        Logger.LogInformation("Component disposed after {Count} renders", renderCount);
    }
}

// OptimizedChild.razor
@implements IDisposable

<div class="child-item" @onclick="HandleClick">
    <span>@Item.Name</span>
    <span class="value">@Item.Value</span>
</div>

@code {
    [Parameter] public DataItem Item { get; set; } = default!;
    [Parameter] public EventCallback<DataItem> OnItemClick { get; set; }
    
    private DataItem? previousItem;
    
    protected override bool ShouldRender()
    {
        // Only render if item actually changed
        if (previousItem != null && 
            previousItem.Id == Item.Id && 
            previousItem.Name == Item.Name && 
            previousItem.Value == Item.Value)
        {
            return false;
        }
        
        previousItem = Item;
        return true;
    }
    
    private async Task HandleClick()
    {
        await OnItemClick.InvokeAsync(Item);
    }
    
    public void Dispose()
    {
        previousItem = null;
    }
}
```

### 1.2 StateHasChanged 최적화

```csharp
// StateManagementComponent.razor
@implements IDisposable

<div class="state-management">
    <h3>Efficient State Management</h3>
    
    <div class="counters">
        <p>Fast Counter: @fastCounter</p>
        <p>Slow Counter: @slowCounter</p>
        <p>Batch Counter: @batchCounter</p>
    </div>
    
    <button @onclick="StartCounters">Start All</button>
    <button @onclick="StopCounters">Stop All</button>
</div>

@code {
    private int fastCounter = 0;
    private int slowCounter = 0;
    private int batchCounter = 0;
    
    private CancellationTokenSource? cts;
    private readonly SemaphoreSlim renderSemaphore = new(1, 1);
    
    private async Task StartCounters()
    {
        cts = new CancellationTokenSource();
        
        // Fast counter - renders immediately
        _ = Task.Run(async () =>
        {
            while (!cts.Token.IsCancellationRequested)
            {
                fastCounter++;
                await InvokeAsync(StateHasChanged);
                await Task.Delay(100, cts.Token);
            }
        });
        
        // Slow counter - batches updates
        _ = Task.Run(async () =>
        {
            var updates = 0;
            while (!cts.Token.IsCancellationRequested)
            {
                slowCounter++;
                updates++;
                
                // Only render every 10 updates
                if (updates % 10 == 0)
                {
                    await InvokeAsync(StateHasChanged);
                }
                
                await Task.Delay(50, cts.Token);
            }
        });
        
        // Batch counter - uses throttling
        _ = Task.Run(async () =>
        {
            var lastRender = DateTime.UtcNow;
            
            while (!cts.Token.IsCancellationRequested)
            {
                batchCounter++;
                
                // Throttle renders to max once per 250ms
                if (DateTime.UtcNow - lastRender > TimeSpan.FromMilliseconds(250))
                {
                    await renderSemaphore.WaitAsync();
                    try
                    {
                        await InvokeAsync(StateHasChanged);
                        lastRender = DateTime.UtcNow;
                    }
                    finally
                    {
                        renderSemaphore.Release();
                    }
                }
                
                await Task.Delay(25, cts.Token);
            }
        });
    }
    
    private void StopCounters()
    {
        cts?.Cancel();
    }
    
    public void Dispose()
    {
        cts?.Cancel();
        cts?.Dispose();
        renderSemaphore?.Dispose();
    }
}
```

## 2. 코드 분할 및 Lazy Loading

### 2.1 어셈블리 Lazy Loading

```csharp
// Services/LazyLoader.cs
public interface ILazyLoader
{
    Task<T> LoadComponentAsync<T>() where T : IComponent;
    Task LoadAssemblyAsync(string assemblyName);
    bool IsLoaded(string assemblyName);
}

public class LazyLoader : ILazyLoader
{
    private readonly HttpClient _httpClient;
    private readonly IJSRuntime _jsRuntime;
    private readonly ILogger<LazyLoader> _logger;
    private readonly Dictionary<string, Assembly> _loadedAssemblies = new();
    
    public LazyLoader(HttpClient httpClient, IJSRuntime jsRuntime, ILogger<LazyLoader> logger)
    {
        _httpClient = httpClient;
        _jsRuntime = jsRuntime;
        _logger = logger;
    }
    
    public async Task<T> LoadComponentAsync<T>() where T : IComponent
    {
        var type = typeof(T);
        var assemblyName = type.Assembly.GetName().Name;
        
        if (!_loadedAssemblies.ContainsKey(assemblyName!))
        {
            await LoadAssemblyAsync(assemblyName!);
        }
        
        return Activator.CreateInstance<T>();
    }
    
    public async Task LoadAssemblyAsync(string assemblyName)
    {
        if (_loadedAssemblies.ContainsKey(assemblyName))
        {
            return;
        }
        
        _logger.LogInformation("Loading assembly: {AssemblyName}", assemblyName);
        
        try
        {
            // Load assembly and dependencies
            var assemblies = await LoadAssemblyAndDependenciesAsync(assemblyName);
            
            foreach (var assembly in assemblies)
            {
                var name = assembly.GetName().Name;
                if (name != null && !_loadedAssemblies.ContainsKey(name))
                {
                    _loadedAssemblies[name] = assembly;
                }
            }
            
            _logger.LogInformation("Successfully loaded {Count} assemblies", assemblies.Count);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to load assembly: {AssemblyName}", assemblyName);
            throw;
        }
    }
    
    private async Task<List<Assembly>> LoadAssemblyAndDependenciesAsync(string assemblyName)
    {
        var loadedAssemblies = new List<Assembly>();
        
        // Download assembly bytes
        var assemblyBytes = await _httpClient.GetByteArrayAsync($"_framework/{assemblyName}.dll");
        
        // Load assembly
        var assembly = Assembly.Load(assemblyBytes);
        loadedAssemblies.Add(assembly);
        
        // Load dependencies
        var references = assembly.GetReferencedAssemblies();
        
        foreach (var reference in references)
        {
            if (!IsSystemAssembly(reference.Name) && !_loadedAssemblies.ContainsKey(reference.Name!))
            {
                var depAssemblies = await LoadAssemblyAndDependenciesAsync(reference.Name!);
                loadedAssemblies.AddRange(depAssemblies);
            }
        }
        
        return loadedAssemblies;
    }
    
    private bool IsSystemAssembly(string? assemblyName)
    {
        if (string.IsNullOrEmpty(assemblyName))
            return true;
        
        return assemblyName.StartsWith("System.") || 
               assemblyName.StartsWith("Microsoft.") ||
               assemblyName == "mscorlib" ||
               assemblyName == "netstandard";
    }
    
    public bool IsLoaded(string assemblyName)
    {
        return _loadedAssemblies.ContainsKey(assemblyName);
    }
}

// LazyLoadedView.razor
@inject ILazyLoader LazyLoader
@inject ILogger<LazyLoadedView> Logger

<div class="lazy-loaded-view">
    @if (isLoading)
    {
        <div class="loading">
            <span>Loading component...</span>
            <div class="spinner"></div>
        </div>
    }
    else if (loadError != null)
    {
        <div class="error">
            <p>Failed to load component: @loadError</p>
            <button @onclick="RetryLoad">Retry</button>
        </div>
    }
    else if (dynamicComponent != null)
    {
        <DynamicComponent Type="dynamicComponent" Parameters="Parameters" />
    }
</div>

@code {
    [Parameter] public string ComponentTypeName { get; set; } = "";
    [Parameter] public string AssemblyName { get; set; } = "";
    [Parameter] public IDictionary<string, object>? Parameters { get; set; }
    
    private Type? dynamicComponent;
    private bool isLoading = false;
    private string? loadError;
    
    protected override async Task OnParametersSetAsync()
    {
        if (!string.IsNullOrEmpty(ComponentTypeName) && !string.IsNullOrEmpty(AssemblyName))
        {
            await LoadComponentAsync();
        }
    }
    
    private async Task LoadComponentAsync()
    {
        isLoading = true;
        loadError = null;
        
        try
        {
            if (!LazyLoader.IsLoaded(AssemblyName))
            {
                await LazyLoader.LoadAssemblyAsync(AssemblyName);
            }
            
            var assembly = AppDomain.CurrentDomain.GetAssemblies()
                .FirstOrDefault(a => a.GetName().Name == AssemblyName);
            
            if (assembly != null)
            {
                dynamicComponent = assembly.GetType(ComponentTypeName);
                
                if (dynamicComponent == null)
                {
                    throw new InvalidOperationException(
                        $"Type {ComponentTypeName} not found in assembly {AssemblyName}");
                }
            }
        }
        catch (Exception ex)
        {
            Logger.LogError(ex, "Failed to load component");
            loadError = ex.Message;
        }
        finally
        {
            isLoading = false;
        }
    }
    
    private async Task RetryLoad()
    {
        await LoadComponentAsync();
    }
}
```

### 2.2 코드 분할 전략

```csharp
// Program.cs - WebAssembly
builder.RootComponents.Add<App>("#app");
builder.RootComponents.Add<HeadOutlet>("head::after");

// Configure lazy loading
builder.Services.AddScoped<ILazyLoader, LazyLoader>();

// Configure assembly lazy loading
builder.Services.Configure<LazyLoadingOptions>(options =>
{
    options.AssembliesForLazyLoading = new[]
    {
        "MyApp.AdminModule.dll",
        "MyApp.ReportsModule.dll",
        "MyApp.AnalyticsModule.dll"
    };
});

// Router.razor - Enhanced with lazy loading
<Router AppAssembly="@typeof(App).Assembly" 
        AdditionalAssemblies="@lazyLoadedAssemblies"
        OnNavigateAsync="@OnNavigateAsync">
    <Navigating>
        <div class="loading-container">
            <p>Loading page...</p>
        </div>
    </Navigating>
    <Found Context="routeData">
        <RouteView RouteData="@routeData" DefaultLayout="@typeof(MainLayout)" />
    </Found>
    <NotFound>
        <PageTitle>Not found</PageTitle>
        <LayoutView Layout="@typeof(MainLayout)">
            <p role="alert">Sorry, there's nothing at this address.</p>
        </LayoutView>
    </NotFound>
</Router>

@code {
    private List<Assembly> lazyLoadedAssemblies = new();
    
    private async Task OnNavigateAsync(NavigationContext context)
    {
        // Load assemblies based on route
        if (context.Path.StartsWith("/admin") && 
            !LazyLoader.IsLoaded("MyApp.AdminModule"))
        {
            await LoadAssemblyForRoute("MyApp.AdminModule");
        }
        else if (context.Path.StartsWith("/reports") && 
                 !LazyLoader.IsLoaded("MyApp.ReportsModule"))
        {
            await LoadAssemblyForRoute("MyApp.ReportsModule");
        }
    }
    
    private async Task LoadAssemblyForRoute(string assemblyName)
    {
        await LazyLoader.LoadAssemblyAsync(assemblyName);
        
        var assembly = AppDomain.CurrentDomain.GetAssemblies()
            .FirstOrDefault(a => a.GetName().Name == assemblyName);
        
        if (assembly != null && !lazyLoadedAssemblies.Contains(assembly))
        {
            lazyLoadedAssemblies.Add(assembly);
        }
    }
}
```

## 3. 네트워크 최적화

### 3.1 데이터 캐싱

```csharp
// Services/CacheService.cs
public interface ICacheService
{
    Task<T?> GetAsync<T>(string key) where T : class;
    Task SetAsync<T>(string key, T value, TimeSpan? expiration = null) where T : class;
    Task RemoveAsync(string key);
    Task ClearAsync();
    Task<T> GetOrCreateAsync<T>(string key, Func<Task<T>> factory, TimeSpan? expiration = null) where T : class;
}

public class HybridCacheService : ICacheService
{
    private readonly IMemoryCache _memoryCache;
    private readonly ILocalStorageService _localStorage;
    private readonly ILogger<HybridCacheService> _logger;
    private readonly SemaphoreSlim _cacheLock = new(1, 1);
    
    public HybridCacheService(
        IMemoryCache memoryCache,
        ILocalStorageService localStorage,
        ILogger<HybridCacheService> logger)
    {
        _memoryCache = memoryCache;
        _localStorage = localStorage;
        _logger = logger;
    }
    
    public async Task<T?> GetAsync<T>(string key) where T : class
    {
        // Try memory cache first
        if (_memoryCache.TryGetValue<T>(key, out var cachedValue))
        {
            _logger.LogDebug("Cache hit (memory): {Key}", key);
            return cachedValue;
        }
        
        // Try local storage
        try
        {
            var stored = await _localStorage.GetItemAsync<CacheEntry<T>>(GetStorageKey(key));
            
            if (stored != null && stored.ExpiresAt > DateTime.UtcNow)
            {
                _logger.LogDebug("Cache hit (storage): {Key}", key);
                
                // Populate memory cache
                _memoryCache.Set(key, stored.Value, stored.ExpiresAt - DateTime.UtcNow);
                
                return stored.Value;
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error reading from local storage cache");
        }
        
        _logger.LogDebug("Cache miss: {Key}", key);
        return null;
    }
    
    public async Task SetAsync<T>(string key, T value, TimeSpan? expiration = null) where T : class
    {
        var expiresAt = DateTime.UtcNow.Add(expiration ?? TimeSpan.FromMinutes(30));
        
        // Set in memory cache
        _memoryCache.Set(key, value, expiration ?? TimeSpan.FromMinutes(30));
        
        // Set in local storage
        try
        {
            var entry = new CacheEntry<T>
            {
                Value = value,
                ExpiresAt = expiresAt
            };
            
            await _localStorage.SetItemAsync(GetStorageKey(key), entry);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error writing to local storage cache");
        }
    }
    
    public async Task<T> GetOrCreateAsync<T>(string key, Func<Task<T>> factory, TimeSpan? expiration = null) where T : class
    {
        await _cacheLock.WaitAsync();
        try
        {
            var cached = await GetAsync<T>(key);
            if (cached != null)
            {
                return cached;
            }
            
            _logger.LogDebug("Creating cache entry: {Key}", key);
            var value = await factory();
            
            await SetAsync(key, value, expiration);
            return value;
        }
        finally
        {
            _cacheLock.Release();
        }
    }
    
    private string GetStorageKey(string key) => $"cache_{key}";
    
    private class CacheEntry<T>
    {
        public T Value { get; set; } = default!;
        public DateTime ExpiresAt { get; set; }
    }
}
```

### 3.2 요청 배칭 및 중복 제거

```csharp
// Services/BatchRequestService.cs
public interface IBatchRequestService
{
    Task<Dictionary<string, T>> ExecuteBatchAsync<T>(IEnumerable<string> ids, 
        Func<IEnumerable<string>, Task<Dictionary<string, T>>> batchFunc);
    void QueueRequest<T>(string id, TaskCompletionSource<T> tcs);
}

public class BatchRequestService : IBatchRequestService, IHostedService
{
    private readonly ILogger<BatchRequestService> _logger;
    private readonly Dictionary<Type, BatchQueue> _queues = new();
    private Timer? _processTimer;
    
    public BatchRequestService(ILogger<BatchRequestService> logger)
    {
        _logger = logger;
    }
    
    public Task StartAsync(CancellationToken cancellationToken)
    {
        _processTimer = new Timer(ProcessQueues, null, 
            TimeSpan.FromMilliseconds(50), 
            TimeSpan.FromMilliseconds(50));
        return Task.CompletedTask;
    }
    
    public Task StopAsync(CancellationToken cancellationToken)
    {
        _processTimer?.Dispose();
        return Task.CompletedTask;
    }
    
    public async Task<Dictionary<string, T>> ExecuteBatchAsync<T>(
        IEnumerable<string> ids, 
        Func<IEnumerable<string>, Task<Dictionary<string, T>>> batchFunc)
    {
        var uniqueIds = ids.Distinct().ToList();
        
        if (!uniqueIds.Any())
        {
            return new Dictionary<string, T>();
        }
        
        _logger.LogDebug("Executing batch request for {Count} items", uniqueIds.Count);
        
        try
        {
            return await batchFunc(uniqueIds);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Batch request failed");
            throw;
        }
    }
    
    public void QueueRequest<T>(string id, TaskCompletionSource<T> tcs)
    {
        var type = typeof(T);
        
        lock (_queues)
        {
            if (!_queues.TryGetValue(type, out var queue))
            {
                queue = new BatchQueue(type);
                _queues[type] = queue;
            }
            
            queue.AddRequest(id, tcs);
        }
    }
    
    private void ProcessQueues(object? state)
    {
        List<BatchQueue> queuesToProcess;
        
        lock (_queues)
        {
            queuesToProcess = _queues.Values
                .Where(q => q.HasPendingRequests && q.IsReady)
                .ToList();
        }
        
        foreach (var queue in queuesToProcess)
        {
            _ = ProcessQueueAsync(queue);
        }
    }
    
    private async Task ProcessQueueAsync(BatchQueue queue)
    {
        var requests = queue.TakePendingRequests();
        
        if (!requests.Any())
            return;
        
        try
        {
            // Process batch
            var ids = requests.Keys.ToList();
            _logger.LogDebug("Processing batch of {Count} requests for type {Type}", 
                ids.Count, queue.Type.Name);
            
            // Here you would call the actual batch function
            // For demonstration, we'll simulate a delay
            await Task.Delay(100);
            
            // Complete all requests
            foreach (var (id, tcs) in requests)
            {
                // In real implementation, set actual result
                tcs.TrySetResult(default!);
            }
        }
        catch (Exception ex)
        {
            // Fail all requests
            foreach (var (_, tcs) in requests)
            {
                tcs.TrySetException(ex);
            }
        }
    }
    
    private class BatchQueue
    {
        private readonly Dictionary<string, TaskCompletionSource<object>> _pending = new();
        private DateTime _lastProcess = DateTime.UtcNow;
        
        public Type Type { get; }
        public bool HasPendingRequests => _pending.Any();
        public bool IsReady => DateTime.UtcNow - _lastProcess > TimeSpan.FromMilliseconds(50);
        
        public BatchQueue(Type type)
        {
            Type = type;
        }
        
        public void AddRequest(string id, object tcs)
        {
            lock (_pending)
            {
                if (!_pending.ContainsKey(id))
                {
                    _pending[id] = (TaskCompletionSource<object>)tcs;
                }
            }
        }
        
        public Dictionary<string, TaskCompletionSource<object>> TakePendingRequests()
        {
            lock (_pending)
            {
                var requests = new Dictionary<string, TaskCompletionSource<object>>(_pending);
                _pending.Clear();
                _lastProcess = DateTime.UtcNow;
                return requests;
            }
        }
    }
}
```

## 4. 메모리 최적화

### 4.1 객체 풀링

```csharp
// ObjectPooling/ObjectPool.cs
public interface IObjectPool<T> where T : class
{
    T Rent();
    void Return(T obj);
    int Available { get; }
    int InUse { get; }
}

public class ObjectPool<T> : IObjectPool<T> where T : class, new()
{
    private readonly ConcurrentBag<T> _objects = new();
    private readonly Func<T> _objectGenerator;
    private readonly Action<T>? _resetAction;
    private readonly int _maxSize;
    private int _currentCount;
    private int _inUseCount;
    
    public int Available => _objects.Count;
    public int InUse => _inUseCount;
    
    public ObjectPool(
        Func<T>? objectGenerator = null,
        Action<T>? resetAction = null,
        int maxSize = 100)
    {
        _objectGenerator = objectGenerator ?? (() => new T());
        _resetAction = resetAction;
        _maxSize = maxSize;
    }
    
    public T Rent()
    {
        T obj;
        
        if (_objects.TryTake(out obj))
        {
            Interlocked.Increment(ref _inUseCount);
            return obj;
        }
        
        if (_currentCount < _maxSize)
        {
            Interlocked.Increment(ref _currentCount);
            Interlocked.Increment(ref _inUseCount);
            return _objectGenerator();
        }
        
        // Pool exhausted, create new object (not pooled)
        return _objectGenerator();
    }
    
    public void Return(T obj)
    {
        if (obj == null) return;
        
        _resetAction?.Invoke(obj);
        
        if (_objects.Count < _maxSize)
        {
            _objects.Add(obj);
            Interlocked.Decrement(ref _inUseCount);
        }
        else
        {
            // Pool is full, let GC collect the object
            if (obj is IDisposable disposable)
            {
                disposable.Dispose();
            }
        }
    }
}

// Usage example
public class PooledStringBuilder
{
    private static readonly ObjectPool<StringBuilder> _pool = new(
        objectGenerator: () => new StringBuilder(256),
        resetAction: sb => sb.Clear(),
        maxSize: 50
    );
    
    public static PooledObject<StringBuilder> Rent()
    {
        return new PooledObject<StringBuilder>(_pool);
    }
}

public struct PooledObject<T> : IDisposable where T : class
{
    private readonly IObjectPool<T> _pool;
    
    public T Value { get; }
    
    public PooledObject(IObjectPool<T> pool)
    {
        _pool = pool;
        Value = pool.Rent();
    }
    
    public void Dispose()
    {
        _pool.Return(Value);
    }
}

// Component using object pooling
@code {
    private string BuildLargeString(List<string> items)
    {
        using var pooled = PooledStringBuilder.Rent();
        var sb = pooled.Value;
        
        foreach (var item in items)
        {
            sb.AppendLine(item);
        }
        
        return sb.ToString();
    }
}
```

### 4.2 메모리 누수 방지

```csharp
// Services/MemoryAwareService.cs
public interface IMemoryAwareService
{
    Task<MemoryStatus> GetMemoryStatusAsync();
    void RegisterPressureCallback(Action<MemoryPressure> callback);
    Task<T> ExecuteWithMemoryCheckAsync<T>(Func<Task<T>> operation, int requiredMb = 50);
}

public class MemoryAwareService : IMemoryAwareService, IDisposable
{
    private readonly ILogger<MemoryAwareService> _logger;
    private readonly List<WeakReference<Action<MemoryPressure>>> _callbacks = new();
    private readonly Timer _monitorTimer;
    private MemoryStatus _lastStatus = new();
    
    public MemoryAwareService(ILogger<MemoryAwareService> logger)
    {
        _logger = logger;
        _monitorTimer = new Timer(MonitorMemory, null, 
            TimeSpan.Zero, TimeSpan.FromSeconds(5));
    }
    
    public Task<MemoryStatus> GetMemoryStatusAsync()
    {
        return Task.FromResult(GetCurrentMemoryStatus());
    }
    
    public void RegisterPressureCallback(Action<MemoryPressure> callback)
    {
        _callbacks.Add(new WeakReference<Action<MemoryPressure>>(callback));
    }
    
    public async Task<T> ExecuteWithMemoryCheckAsync<T>(
        Func<Task<T>> operation, int requiredMb = 50)
    {
        var status = GetCurrentMemoryStatus();
        
        if (status.AvailableMb < requiredMb)
        {
            _logger.LogWarning(
                "Low memory detected. Available: {Available}MB, Required: {Required}MB",
                status.AvailableMb, requiredMb);
            
            // Force garbage collection
            GC.Collect(2, GCCollectionMode.Forced, true);
            GC.WaitForPendingFinalizers();
            GC.Collect(2, GCCollectionMode.Forced, true);
            
            status = GetCurrentMemoryStatus();
            
            if (status.AvailableMb < requiredMb)
            {
                throw new InsufficientMemoryException(
                    $"Insufficient memory. Available: {status.AvailableMb}MB, Required: {requiredMb}MB");
            }
        }
        
        return await operation();
    }
    
    private MemoryStatus GetCurrentMemoryStatus()
    {
        var info = GC.GetMemoryInfo();
        var totalMemory = GC.GetTotalMemory(false);
        
        return new MemoryStatus
        {
            TotalMemoryMb = (int)(info.TotalAvailableMemoryBytes / (1024 * 1024)),
            UsedMemoryMb = (int)(totalMemory / (1024 * 1024)),
            AvailableMb = (int)((info.TotalAvailableMemoryBytes - totalMemory) / (1024 * 1024)),
            GCGen0Count = GC.CollectionCount(0),
            GCGen1Count = GC.CollectionCount(1),
            GCGen2Count = GC.CollectionCount(2),
            Pressure = CalculatePressure(info, totalMemory)
        };
    }
    
    private MemoryPressure CalculatePressure(GCMemoryInfo info, long totalMemory)
    {
        var usagePercent = (double)totalMemory / info.TotalAvailableMemoryBytes * 100;
        
        return usagePercent switch
        {
            < 50 => MemoryPressure.Low,
            < 75 => MemoryPressure.Medium,
            < 90 => MemoryPressure.High,
            _ => MemoryPressure.Critical
        };
    }
    
    private void MonitorMemory(object? state)
    {
        var status = GetCurrentMemoryStatus();
        
        if (status.Pressure != _lastStatus.Pressure)
        {
            NotifyCallbacks(status.Pressure);
        }
        
        if (status.Pressure >= MemoryPressure.High)
        {
            _logger.LogWarning(
                "High memory pressure detected: {Pressure}, Used: {Used}MB",
                status.Pressure, status.UsedMemoryMb);
        }
        
        _lastStatus = status;
    }
    
    private void NotifyCallbacks(MemoryPressure pressure)
    {
        var deadCallbacks = new List<WeakReference<Action<MemoryPressure>>>();
        
        foreach (var weakRef in _callbacks)
        {
            if (weakRef.TryGetTarget(out var callback))
            {
                try
                {
                    callback(pressure);
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex, "Error in memory pressure callback");
                }
            }
            else
            {
                deadCallbacks.Add(weakRef);
            }
        }
        
        // Clean up dead references
        foreach (var dead in deadCallbacks)
        {
            _callbacks.Remove(dead);
        }
    }
    
    public void Dispose()
    {
        _monitorTimer?.Dispose();
    }
}

public class MemoryStatus
{
    public int TotalMemoryMb { get; set; }
    public int UsedMemoryMb { get; set; }
    public int AvailableMb { get; set; }
    public int GCGen0Count { get; set; }
    public int GCGen1Count { get; set; }
    public int GCGen2Count { get; set; }
    public MemoryPressure Pressure { get; set; }
}

public enum MemoryPressure
{
    Low,
    Medium,
    High,
    Critical
}
```

## 5. 성능 모니터링

### 5.1 성능 측정

```csharp
// Services/PerformanceMonitor.cs
public interface IPerformanceMonitor
{
    IDisposable MeasureTime(string operation);
    Task<T> MeasureAsync<T>(string operation, Func<Task<T>> func);
    PerformanceMetrics GetMetrics();
    void Reset();
}

public class PerformanceMonitor : IPerformanceMonitor
{
    private readonly ConcurrentDictionary<string, OperationMetrics> _metrics = new();
    private readonly ILogger<PerformanceMonitor> _logger;
    
    public PerformanceMonitor(ILogger<PerformanceMonitor> logger)
    {
        _logger = logger;
    }
    
    public IDisposable MeasureTime(string operation)
    {
        return new TimeMeasurement(this, operation);
    }
    
    public async Task<T> MeasureAsync<T>(string operation, Func<Task<T>> func)
    {
        using (MeasureTime(operation))
        {
            return await func();
        }
    }
    
    public PerformanceMetrics GetMetrics()
    {
        return new PerformanceMetrics
        {
            Operations = _metrics.ToDictionary(
                kvp => kvp.Key,
                kvp => kvp.Value.GetSummary())
        };
    }
    
    public void Reset()
    {
        _metrics.Clear();
    }
    
    internal void RecordOperation(string operation, long elapsedMs)
    {
        var metrics = _metrics.GetOrAdd(operation, _ => new OperationMetrics());
        metrics.Record(elapsedMs);
        
        if (elapsedMs > 1000)
        {
            _logger.LogWarning(
                "Slow operation detected: {Operation} took {ElapsedMs}ms",
                operation, elapsedMs);
        }
    }
    
    private class TimeMeasurement : IDisposable
    {
        private readonly PerformanceMonitor _monitor;
        private readonly string _operation;
        private readonly Stopwatch _stopwatch;
        
        public TimeMeasurement(PerformanceMonitor monitor, string operation)
        {
            _monitor = monitor;
            _operation = operation;
            _stopwatch = Stopwatch.StartNew();
        }
        
        public void Dispose()
        {
            _stopwatch.Stop();
            _monitor.RecordOperation(_operation, _stopwatch.ElapsedMilliseconds);
        }
    }
    
    private class OperationMetrics
    {
        private readonly object _lock = new();
        private readonly List<long> _measurements = new();
        
        public void Record(long elapsedMs)
        {
            lock (_lock)
            {
                _measurements.Add(elapsedMs);
                
                // Keep only last 100 measurements
                if (_measurements.Count > 100)
                {
                    _measurements.RemoveAt(0);
                }
            }
        }
        
        public OperationSummary GetSummary()
        {
            lock (_lock)
            {
                if (!_measurements.Any())
                {
                    return new OperationSummary();
                }
                
                var sorted = _measurements.OrderBy(x => x).ToList();
                
                return new OperationSummary
                {
                    Count = sorted.Count,
                    MinMs = sorted.First(),
                    MaxMs = sorted.Last(),
                    AvgMs = (long)sorted.Average(),
                    P50Ms = GetPercentile(sorted, 50),
                    P95Ms = GetPercentile(sorted, 95),
                    P99Ms = GetPercentile(sorted, 99)
                };
            }
        }
        
        private long GetPercentile(List<long> sorted, int percentile)
        {
            var index = (int)Math.Ceiling(sorted.Count * percentile / 100.0) - 1;
            return sorted[Math.Max(0, Math.Min(index, sorted.Count - 1))];
        }
    }
}

// PerformanceMetrics.cs
public class PerformanceMetrics
{
    public Dictionary<string, OperationSummary> Operations { get; set; } = new();
}

public class OperationSummary
{
    public int Count { get; set; }
    public long MinMs { get; set; }
    public long MaxMs { get; set; }
    public long AvgMs { get; set; }
    public long P50Ms { get; set; }
    public long P95Ms { get; set; }
    public long P99Ms { get; set; }
}

// Usage in component
@inject IPerformanceMonitor PerformanceMonitor

@code {
    private async Task LoadDataAsync()
    {
        using (PerformanceMonitor.MeasureTime("LoadData"))
        {
            // Simulate data loading
            await Task.Delay(100);
            
            using (PerformanceMonitor.MeasureTime("ProcessData"))
            {
                // Process data
                await ProcessDataAsync();
            }
        }
    }
}
```

## 마무리

Blazor 애플리케이션의 성능 최적화는 사용자 경험과 시스템 효율성에 직접적인 영향을 미칩니다. 렌더링 최적화, 코드 분할, 캐싱, 메모리 관리, 성능 모니터링 등 다양한 기법을 적절히 조합하여 사용하면 빠르고 반응성이 뛰어난 애플리케이션을 구축할 수 있습니다. 항상 성능을 측정하고 병목 현상을 찾아 개선하는 지속적인 노력이 필요합니다.