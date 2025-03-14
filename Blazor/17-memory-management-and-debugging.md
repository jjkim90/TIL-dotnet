# 메모리 관리와 디버깅

## 개요

Blazor 애플리케이션의 효율적인 메모리 관리와 체계적인 디버깅은 안정적이고 성능이 뛰어난 애플리케이션 개발의 핵심입니다. 이 장에서는 메모리 누수 감지, 가비지 커렉션 최적화, 디버깅 도구 활용, 성능 프로파일링 등을 학습합니다.

## 1. 메모리 누수 감지와 방지

### 1.1 컴포넌트 수명 주기 관리

```csharp
// DisposableComponent.cs
public abstract class DisposableComponentBase : ComponentBase, IDisposable, IAsyncDisposable
{
    private readonly HashSet<IDisposable> _disposables = new();
    private readonly HashSet<IAsyncDisposable> _asyncDisposables = new();
    private bool _disposed;
    
    protected ILogger Logger { get; }
    
    protected DisposableComponentBase(ILogger logger)
    {
        Logger = logger;
        Logger.LogDebug("Component created: {ComponentType}", GetType().Name);
    }
    
    protected T RegisterDisposable<T>(T disposable) where T : IDisposable
    {
        ThrowIfDisposed();
        _disposables.Add(disposable);
        return disposable;
    }
    
    protected T RegisterAsyncDisposable<T>(T disposable) where T : IAsyncDisposable
    {
        ThrowIfDisposed();
        _asyncDisposables.Add(disposable);
        return disposable;
    }
    
    protected void ThrowIfDisposed()
    {
        if (_disposed)
        {
            throw new ObjectDisposedException(GetType().Name);
        }
    }
    
    public void Dispose()
    {
        Dispose(disposing: true);
        GC.SuppressFinalize(this);
    }
    
    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return;
        
        if (disposing)
        {
            Logger.LogDebug("Disposing component: {ComponentType}", GetType().Name);
            
            // Dispose synchronous resources
            foreach (var disposable in _disposables)
            {
                try
                {
                    disposable.Dispose();
                }
                catch (Exception ex)
                {
                    Logger.LogError(ex, "Error disposing resource");
                }
            }
            
            // Block on async disposables (not ideal but necessary for IDisposable)
            foreach (var asyncDisposable in _asyncDisposables)
            {
                try
                {
                    asyncDisposable.DisposeAsync().AsTask().Wait();
                }
                catch (Exception ex)
                {
                    Logger.LogError(ex, "Error disposing async resource");
                }
            }
        }
        
        _disposed = true;
    }
    
    public async ValueTask DisposeAsync()
    {
        if (_disposed) return;
        
        Logger.LogDebug("Async disposing component: {ComponentType}", GetType().Name);
        
        // Dispose async resources first
        foreach (var asyncDisposable in _asyncDisposables)
        {
            try
            {
                await asyncDisposable.DisposeAsync();
            }
            catch (Exception ex)
            {
                Logger.LogError(ex, "Error disposing async resource");
            }
        }
        
        // Then dispose synchronous resources
        foreach (var disposable in _disposables)
        {
            try
            {
                disposable.Dispose();
            }
            catch (Exception ex)
            {
                Logger.LogError(ex, "Error disposing resource");
            }
        }
        
        _disposed = true;
        GC.SuppressFinalize(this);
    }
}

// Example usage
@inherits DisposableComponentBase
@inject IDataService DataService
@inject ILogger<MyComponent> Logger

<div class="my-component">
    <!-- Component content -->
</div>

@code {
    private Timer? refreshTimer;
    private CancellationTokenSource? cts;
    private IDisposable? subscription;
    
    protected override void OnInitialized()
    {
        base.OnInitialized();
        
        // Register disposables
        cts = RegisterDisposable(new CancellationTokenSource());
        
        refreshTimer = RegisterDisposable(new Timer(_ => 
        {
            InvokeAsync(StateHasChanged);
        }, null, TimeSpan.Zero, TimeSpan.FromSeconds(30)));
        
        subscription = RegisterDisposable(
            DataService.Subscribe(data => HandleDataUpdate(data)));
    }
}
```

### 1.2 메모리 누수 감지 도구

```csharp
// Services/MemoryLeakDetector.cs
public interface IMemoryLeakDetector
{
    void TrackObject(object obj, string identifier);
    void UntrackObject(object obj);
    MemoryLeakReport GenerateReport();
    void StartMonitoring();
    void StopMonitoring();
}

public class MemoryLeakDetector : IMemoryLeakDetector, IDisposable
{
    private readonly ConditionalWeakTable<object, ObjectTrackingInfo> _trackedObjects = new();
    private readonly ILogger<MemoryLeakDetector> _logger;
    private Timer? _monitoringTimer;
    private bool _isMonitoring;
    
    public MemoryLeakDetector(ILogger<MemoryLeakDetector> logger)
    {
        _logger = logger;
    }
    
    public void TrackObject(object obj, string identifier)
    {
        var info = new ObjectTrackingInfo
        {
            Identifier = identifier,
            Type = obj.GetType().FullName ?? "Unknown",
            CreatedAt = DateTime.UtcNow,
            Size = EstimateObjectSize(obj),
            StackTrace = Environment.StackTrace
        };
        
        _trackedObjects.AddOrUpdate(obj, info);
        
        _logger.LogDebug("Tracking object: {Identifier} ({Type})", 
            identifier, info.Type);
    }
    
    public void UntrackObject(object obj)
    {
        if (_trackedObjects.TryGetValue(obj, out var info))
        {
            _trackedObjects.Remove(obj);
            _logger.LogDebug("Untracking object: {Identifier}", info.Identifier);
        }
    }
    
    public void StartMonitoring()
    {
        if (_isMonitoring) return;
        
        _isMonitoring = true;
        _monitoringTimer = new Timer(CheckForLeaks, null, 
            TimeSpan.FromSeconds(30), TimeSpan.FromSeconds(30));
        
        _logger.LogInformation("Memory leak monitoring started");
    }
    
    public void StopMonitoring()
    {
        _isMonitoring = false;
        _monitoringTimer?.Dispose();
        
        _logger.LogInformation("Memory leak monitoring stopped");
    }
    
    private void CheckForLeaks(object? state)
    {
        // Force garbage collection
        GC.Collect(2, GCCollectionMode.Forced, true);
        GC.WaitForPendingFinalizers();
        GC.Collect(2, GCCollectionMode.Forced, true);
        
        var report = GenerateReport();
        
        if (report.PotentialLeaks.Any())
        {
            _logger.LogWarning(
                "Potential memory leaks detected: {Count} objects still alive",
                report.PotentialLeaks.Count);
            
            foreach (var leak in report.PotentialLeaks.Take(10))
            {
                _logger.LogWarning(
                    "Potential leak: {Identifier} ({Type}) created at {CreatedAt}, Age: {Age}s",
                    leak.Identifier, leak.Type, leak.CreatedAt, 
                    (DateTime.UtcNow - leak.CreatedAt).TotalSeconds);
            }
        }
    }
    
    public MemoryLeakReport GenerateReport()
    {
        var aliveObjects = new List<ObjectTrackingInfo>();
        
        // Iterate through tracked objects
        foreach (var kvp in _trackedObjects)
        {
            if (kvp.Value != null)
            {
                aliveObjects.Add(kvp.Value);
            }
        }
        
        // Identify potential leaks (objects alive for more than 5 minutes)
        var potentialLeaks = aliveObjects
            .Where(o => DateTime.UtcNow - o.CreatedAt > TimeSpan.FromMinutes(5))
            .OrderByDescending(o => o.CreatedAt)
            .ToList();
        
        return new MemoryLeakReport
        {
            TotalTrackedObjects = aliveObjects.Count,
            PotentialLeaks = potentialLeaks,
            MemoryUsage = GC.GetTotalMemory(false),
            GeneratedAt = DateTime.UtcNow
        };
    }
    
    private long EstimateObjectSize(object obj)
    {
        // Simplified size estimation
        return obj switch
        {
            string str => str.Length * 2,
            Array array => array.Length * 8,
            ICollection collection => collection.Count * 8,
            _ => 24 // Default object overhead
        };
    }
    
    public void Dispose()
    {
        StopMonitoring();
    }
}

public class ObjectTrackingInfo
{
    public string Identifier { get; set; } = "";
    public string Type { get; set; } = "";
    public DateTime CreatedAt { get; set; }
    public long Size { get; set; }
    public string StackTrace { get; set; } = "";
}

public class MemoryLeakReport
{
    public int TotalTrackedObjects { get; set; }
    public List<ObjectTrackingInfo> PotentialLeaks { get; set; } = new();
    public long MemoryUsage { get; set; }
    public DateTime GeneratedAt { get; set; }
}
```

## 2. 가비지 커렉션 최적화

### 2.1 GC 압력 관리

```csharp
// Services/GCManager.cs
public interface IGCManager
{
    GCMetrics GetMetrics();
    void OptimizeForLowLatency();
    void OptimizeForThroughput();
    void ForceCollection(int generation = 2);
    void RegisterLargeObjectAllocation(long size);
}

public class GCManager : IGCManager
{
    private readonly ILogger<GCManager> _logger;
    private readonly Queue<GCEvent> _gcEvents = new();
    private readonly object _lock = new();
    private GCLatencyMode _previousLatencyMode;
    
    public GCManager(ILogger<GCManager> logger)
    {
        _logger = logger;
        RegisterGCCallbacks();
    }
    
    private void RegisterGCCallbacks()
    {
        AppDomain.CurrentDomain.ProcessExit += (s, e) => 
        {
            GCSettings.LatencyMode = _previousLatencyMode;
        };
    }
    
    public GCMetrics GetMetrics()
    {
        var memoryInfo = GC.GetMemoryInfo();
        
        return new GCMetrics
        {
            TotalMemory = GC.GetTotalMemory(false),
            Gen0Collections = GC.CollectionCount(0),
            Gen1Collections = GC.CollectionCount(1),
            Gen2Collections = GC.CollectionCount(2),
            TotalAvailableMemory = memoryInfo.TotalAvailableMemoryBytes,
            HighMemoryLoadThreshold = memoryInfo.HighMemoryLoadThresholdBytes,
            FragmentedBytes = memoryInfo.FragmentedBytes,
            HeapSizeBytes = memoryInfo.HeapSizeBytes,
            LOHSizeBytes = GetLOHSize(),
            LatencyMode = GCSettings.LatencyMode,
            IsServerGC = GCSettings.IsServerGC,
            RecentGCEvents = GetRecentGCEvents()
        };
    }
    
    public void OptimizeForLowLatency()
    {
        _previousLatencyMode = GCSettings.LatencyMode;
        
        if (GCSettings.LatencyMode != GCLatencyMode.SustainedLowLatency)
        {
            GCSettings.LatencyMode = GCLatencyMode.SustainedLowLatency;
            _logger.LogInformation("GC optimized for low latency");
        }
    }
    
    public void OptimizeForThroughput()
    {
        if (GCSettings.LatencyMode != GCLatencyMode.Batch)
        {
            GCSettings.LatencyMode = GCLatencyMode.Batch;
            _logger.LogInformation("GC optimized for throughput");
        }
    }
    
    public void ForceCollection(int generation = 2)
    {
        var before = GC.GetTotalMemory(false);
        
        GC.Collect(generation, GCCollectionMode.Forced, true);
        GC.WaitForPendingFinalizers();
        GC.Collect(generation, GCCollectionMode.Forced, true);
        
        var after = GC.GetTotalMemory(false);
        var freed = before - after;
        
        _logger.LogInformation(
            "Forced GC collection. Generation: {Generation}, Freed: {Freed:N0} bytes",
            generation, freed);
        
        RecordGCEvent(new GCEvent
        {
            Generation = generation,
            Type = GCEventType.Forced,
            FreedBytes = freed,
            Timestamp = DateTime.UtcNow
        });
    }
    
    public void RegisterLargeObjectAllocation(long size)
    {
        if (size >= 85000) // Large Object Heap threshold
        {
            _logger.LogWarning(
                "Large object allocation detected: {Size:N0} bytes",
                size);
            
            RecordGCEvent(new GCEvent
            {
                Type = GCEventType.LargeObjectAllocation,
                AllocationSize = size,
                Timestamp = DateTime.UtcNow
            });
        }
    }
    
    private long GetLOHSize()
    {
        // This is an approximation
        var memoryInfo = GC.GetMemoryInfo();
        return memoryInfo.GenerationInfo.Length > 3 
            ? memoryInfo.GenerationInfo[3].SizeBytes 
            : 0;
    }
    
    private void RecordGCEvent(GCEvent gcEvent)
    {
        lock (_lock)
        {
            _gcEvents.Enqueue(gcEvent);
            
            // Keep only last 100 events
            while (_gcEvents.Count > 100)
            {
                _gcEvents.Dequeue();
            }
        }
    }
    
    private List<GCEvent> GetRecentGCEvents()
    {
        lock (_lock)
        {
            return _gcEvents.ToList();
        }
    }
}

public class GCMetrics
{
    public long TotalMemory { get; set; }
    public int Gen0Collections { get; set; }
    public int Gen1Collections { get; set; }
    public int Gen2Collections { get; set; }
    public long TotalAvailableMemory { get; set; }
    public long HighMemoryLoadThreshold { get; set; }
    public long FragmentedBytes { get; set; }
    public long HeapSizeBytes { get; set; }
    public long LOHSizeBytes { get; set; }
    public GCLatencyMode LatencyMode { get; set; }
    public bool IsServerGC { get; set; }
    public List<GCEvent> RecentGCEvents { get; set; } = new();
}

public class GCEvent
{
    public DateTime Timestamp { get; set; }
    public GCEventType Type { get; set; }
    public int Generation { get; set; }
    public long FreedBytes { get; set; }
    public long AllocationSize { get; set; }
}

public enum GCEventType
{
    Automatic,
    Forced,
    LargeObjectAllocation
}
```

## 3. 디버깅 도구

### 3.1 커스텀 디버거

```csharp
// Debugging/BlazorDebugger.cs
public interface IBlazorDebugger
{
    void Break(string message = "");
    void Log(string message, params object[] args);
    void LogComponent(ComponentBase component, string message);
    void StartProfiling(string operation);
    void EndProfiling(string operation);
    DebugSnapshot CaptureSnapshot(string identifier);
    void EnableTracing(bool enable);
}

public class BlazorDebugger : IBlazorDebugger
{
    private readonly ILogger<BlazorDebugger> _logger;
    private readonly Dictionary<string, Stopwatch> _profilingOperations = new();
    private readonly List<DebugSnapshot> _snapshots = new();
    private bool _tracingEnabled;
    
    public BlazorDebugger(ILogger<BlazorDebugger> logger)
    {
        _logger = logger;
    }
    
    public void Break(string message = "")
    {
        if (Debugger.IsAttached)
        {
            _logger.LogWarning("Debugger break: {Message}", message);
            Debugger.Break();
        }
    }
    
    public void Log(string message, params object[] args)
    {
        _logger.LogDebug(message, args);
        
        if (_tracingEnabled)
        {
            Console.WriteLine($"[DEBUG] {string.Format(message, args)}");
        }
    }
    
    public void LogComponent(ComponentBase component, string message)
    {
        var componentName = component.GetType().Name;
        var componentId = component.GetHashCode();
        
        _logger.LogDebug(
            "[{ComponentName}:{ComponentId}] {Message}",
            componentName, componentId, message);
    }
    
    public void StartProfiling(string operation)
    {
        lock (_profilingOperations)
        {
            if (_profilingOperations.ContainsKey(operation))
            {
                _logger.LogWarning(
                    "Profiling operation already started: {Operation}",
                    operation);
                return;
            }
            
            _profilingOperations[operation] = Stopwatch.StartNew();
            _logger.LogDebug("Profiling started: {Operation}", operation);
        }
    }
    
    public void EndProfiling(string operation)
    {
        lock (_profilingOperations)
        {
            if (!_profilingOperations.TryGetValue(operation, out var stopwatch))
            {
                _logger.LogWarning(
                    "Profiling operation not found: {Operation}",
                    operation);
                return;
            }
            
            stopwatch.Stop();
            _profilingOperations.Remove(operation);
            
            _logger.LogInformation(
                "Profiling completed: {Operation} took {ElapsedMs}ms",
                operation, stopwatch.ElapsedMilliseconds);
        }
    }
    
    public DebugSnapshot CaptureSnapshot(string identifier)
    {
        var snapshot = new DebugSnapshot
        {
            Identifier = identifier,
            Timestamp = DateTime.UtcNow,
            MemoryUsage = GC.GetTotalMemory(false),
            ThreadCount = Process.GetCurrentProcess().Threads.Count,
            HandleCount = Process.GetCurrentProcess().HandleCount,
            Gen0Collections = GC.CollectionCount(0),
            Gen1Collections = GC.CollectionCount(1),
            Gen2Collections = GC.CollectionCount(2),
            ActiveProfilingOperations = _profilingOperations.Keys.ToList(),
            StackTrace = Environment.StackTrace
        };
        
        _snapshots.Add(snapshot);
        
        // Keep only last 50 snapshots
        if (_snapshots.Count > 50)
        {
            _snapshots.RemoveAt(0);
        }
        
        _logger.LogDebug("Debug snapshot captured: {Identifier}", identifier);
        
        return snapshot;
    }
    
    public void EnableTracing(bool enable)
    {
        _tracingEnabled = enable;
        _logger.LogInformation("Tracing {Status}", 
            enable ? "enabled" : "disabled");
    }
}

public class DebugSnapshot
{
    public string Identifier { get; set; } = "";
    public DateTime Timestamp { get; set; }
    public long MemoryUsage { get; set; }
    public int ThreadCount { get; set; }
    public int HandleCount { get; set; }
    public int Gen0Collections { get; set; }
    public int Gen1Collections { get; set; }
    public int Gen2Collections { get; set; }
    public List<string> ActiveProfilingOperations { get; set; } = new();
    public string StackTrace { get; set; } = "";
}
```

### 3.2 컴포넌트 디버깅 확장

```csharp
// DebugExtensions.cs
public static class ComponentDebugExtensions
{
    public static void DebugRender(this ComponentBase component, ILogger logger)
    {
        var renderCount = GetRenderCount(component);
        logger.LogDebug(
            "Component {Type} rendering (count: {Count})",
            component.GetType().Name, renderCount);
    }
    
    public static void DebugParameters(this ComponentBase component, ILogger logger)
    {
        var parameters = GetComponentParameters(component);
        
        foreach (var param in parameters)
        {
            logger.LogDebug(
                "Parameter {Name}: {Value}",
                param.Key, param.Value ?? "null");
        }
    }
    
    public static void DebugState(this ComponentBase component, ILogger logger)
    {
        var state = GetComponentState(component);
        logger.LogDebug(
            "Component state: {State}",
            JsonSerializer.Serialize(state, new JsonSerializerOptions 
            { 
                WriteIndented = true 
            }));
    }
    
    private static int GetRenderCount(ComponentBase component)
    {
        // Use reflection to access internal render count
        var field = component.GetType()
            .GetField("_renderCount", BindingFlags.NonPublic | BindingFlags.Instance);
        
        return field?.GetValue(component) as int? ?? 0;
    }
    
    private static Dictionary<string, object?> GetComponentParameters(ComponentBase component)
    {
        var parameters = new Dictionary<string, object?>();
        
        var properties = component.GetType()
            .GetProperties(BindingFlags.Public | BindingFlags.Instance)
            .Where(p => p.GetCustomAttribute<ParameterAttribute>() != null);
        
        foreach (var property in properties)
        {
            parameters[property.Name] = property.GetValue(component);
        }
        
        return parameters;
    }
    
    private static Dictionary<string, object?> GetComponentState(ComponentBase component)
    {
        var state = new Dictionary<string, object?>();
        
        var fields = component.GetType()
            .GetFields(BindingFlags.NonPublic | BindingFlags.Instance)
            .Where(f => !f.Name.StartsWith("<"));
        
        foreach (var field in fields)
        {
            try
            {
                state[field.Name] = field.GetValue(component);
            }
            catch
            {
                state[field.Name] = "<error>";
            }
        }
        
        return state;
    }
}

// DebugComponent.razor
@if (ShowDebugInfo)
{
    <div class="debug-panel">
        <h4>Debug Information</h4>
        <button @onclick="CaptureSnapshot">Capture Snapshot</button>
        <button @onclick="ForceGC">Force GC</button>
        <button @onclick="ToggleProfiling">@(isProfiling ? "Stop" : "Start") Profiling</button>
        
        <div class="metrics">
            <p>Memory: @(GC.GetTotalMemory(false) / 1024 / 1024) MB</p>
            <p>Gen0: @GC.CollectionCount(0)</p>
            <p>Gen1: @GC.CollectionCount(1)</p>
            <p>Gen2: @GC.CollectionCount(2)</p>
        </div>
        
        @if (latestSnapshot != null)
        {
            <div class="snapshot">
                <h5>Latest Snapshot</h5>
                <pre>@JsonSerializer.Serialize(latestSnapshot, new JsonSerializerOptions { WriteIndented = true })</pre>
            </div>
        }
    </div>
}

@code {
    [Parameter] public bool ShowDebugInfo { get; set; }
    
    private DebugSnapshot? latestSnapshot;
    private bool isProfiling;
    
    private void CaptureSnapshot()
    {
        latestSnapshot = Debugger.CaptureSnapshot($"Manual-{DateTime.Now:HH:mm:ss}");
    }
    
    private void ForceGC()
    {
        GCManager.ForceCollection();
    }
    
    private void ToggleProfiling()
    {
        if (isProfiling)
        {
            Debugger.EndProfiling("ComponentProfiling");
        }
        else
        {
            Debugger.StartProfiling("ComponentProfiling");
        }
        
        isProfiling = !isProfiling;
    }
}
```

## 4. 성능 프로파일링

### 4.1 컴포넌트 프로파일링

```csharp
// Profiling/ComponentProfiler.cs
public interface IComponentProfiler
{
    void StartComponentProfiling(ComponentBase component);
    void EndComponentProfiling(ComponentBase component);
    ComponentProfilingReport GetReport();
    void Reset();
}

public class ComponentProfiler : IComponentProfiler
{
    private readonly ConcurrentDictionary<string, ComponentMetrics> _metrics = new();
    private readonly ILogger<ComponentProfiler> _logger;
    
    public ComponentProfiler(ILogger<ComponentProfiler> logger)
    {
        _logger = logger;
    }
    
    public void StartComponentProfiling(ComponentBase component)
    {
        var key = GetComponentKey(component);
        var metrics = _metrics.GetOrAdd(key, _ => new ComponentMetrics
        {
            ComponentType = component.GetType().FullName ?? "Unknown"
        });
        
        metrics.StartOperation();
    }
    
    public void EndComponentProfiling(ComponentBase component)
    {
        var key = GetComponentKey(component);
        
        if (_metrics.TryGetValue(key, out var metrics))
        {
            metrics.EndOperation();
        }
    }
    
    public ComponentProfilingReport GetReport()
    {
        var componentStats = _metrics
            .Select(kvp => new ComponentStats
            {
                ComponentType = kvp.Value.ComponentType,
                TotalRenders = kvp.Value.RenderCount,
                AverageRenderTime = kvp.Value.GetAverageRenderTime(),
                MinRenderTime = kvp.Value.MinRenderTime,
                MaxRenderTime = kvp.Value.MaxRenderTime,
                TotalRenderTime = kvp.Value.TotalRenderTime
            })
            .OrderByDescending(s => s.TotalRenderTime)
            .ToList();
        
        return new ComponentProfilingReport
        {
            TotalComponents = componentStats.Count,
            TotalRenders = componentStats.Sum(s => s.TotalRenders),
            ComponentStats = componentStats,
            GeneratedAt = DateTime.UtcNow
        };
    }
    
    public void Reset()
    {
        _metrics.Clear();
        _logger.LogInformation("Component profiler reset");
    }
    
    private string GetComponentKey(ComponentBase component)
    {
        return $"{component.GetType().FullName}#{component.GetHashCode()}";
    }
    
    private class ComponentMetrics
    {
        private readonly object _lock = new();
        private readonly List<long> _renderTimes = new();
        private Stopwatch? _currentOperation;
        
        public string ComponentType { get; set; } = "";
        public int RenderCount => _renderTimes.Count;
        public long TotalRenderTime => _renderTimes.Sum();
        public long MinRenderTime => _renderTimes.Any() ? _renderTimes.Min() : 0;
        public long MaxRenderTime => _renderTimes.Any() ? _renderTimes.Max() : 0;
        
        public void StartOperation()
        {
            lock (_lock)
            {
                _currentOperation = Stopwatch.StartNew();
            }
        }
        
        public void EndOperation()
        {
            lock (_lock)
            {
                if (_currentOperation != null)
                {
                    _currentOperation.Stop();
                    _renderTimes.Add(_currentOperation.ElapsedMilliseconds);
                    _currentOperation = null;
                }
            }
        }
        
        public double GetAverageRenderTime()
        {
            lock (_lock)
            {
                return _renderTimes.Any() ? _renderTimes.Average() : 0;
            }
        }
    }
}

public class ComponentProfilingReport
{
    public int TotalComponents { get; set; }
    public int TotalRenders { get; set; }
    public List<ComponentStats> ComponentStats { get; set; } = new();
    public DateTime GeneratedAt { get; set; }
}

public class ComponentStats
{
    public string ComponentType { get; set; } = "";
    public int TotalRenders { get; set; }
    public double AverageRenderTime { get; set; }
    public long MinRenderTime { get; set; }
    public long MaxRenderTime { get; set; }
    public long TotalRenderTime { get; set; }
}

// ProfiledComponentBase.cs
public abstract class ProfiledComponentBase : ComponentBase
{
    [Inject] protected IComponentProfiler Profiler { get; set; } = default!;
    
    protected override void OnInitialized()
    {
        Profiler.StartComponentProfiling(this);
        base.OnInitialized();
    }
    
    protected override bool ShouldRender()
    {
        Profiler.StartComponentProfiling(this);
        return base.ShouldRender();
    }
    
    protected override void OnAfterRender(bool firstRender)
    {
        Profiler.EndComponentProfiling(this);
        base.OnAfterRender(firstRender);
    }
}
```

### 4.2 예시 - 프로파일링 대시보드

```csharp
// ProfilingDashboard.razor
@page "/debug/profiling"
@inject IComponentProfiler ComponentProfiler
@inject IGCManager GCManager
@inject IMemoryLeakDetector LeakDetector
@implements IDisposable

<h3>Performance Profiling Dashboard</h3>

<div class="profiling-controls">
    <button @onclick="RefreshData">Refresh</button>
    <button @onclick="ResetProfiler">Reset Profiler</button>
    <button @onclick="ForceGC">Force GC</button>
    <button @onclick="GenerateLeakReport">Check for Leaks</button>
</div>

<div class="profiling-grid">
    <div class="memory-stats">
        <h4>Memory Statistics</h4>
        <table>
            <tr>
                <td>Total Memory:</td>
                <td>@(gcMetrics.TotalMemory / 1024 / 1024) MB</td>
            </tr>
            <tr>
                <td>Available Memory:</td>
                <td>@(gcMetrics.TotalAvailableMemory / 1024 / 1024 / 1024) GB</td>
            </tr>
            <tr>
                <td>Gen0 Collections:</td>
                <td>@gcMetrics.Gen0Collections</td>
            </tr>
            <tr>
                <td>Gen1 Collections:</td>
                <td>@gcMetrics.Gen1Collections</td>
            </tr>
            <tr>
                <td>Gen2 Collections:</td>
                <td>@gcMetrics.Gen2Collections</td>
            </tr>
            <tr>
                <td>LOH Size:</td>
                <td>@(gcMetrics.LOHSizeBytes / 1024 / 1024) MB</td>
            </tr>
        </table>
    </div>
    
    <div class="component-stats">
        <h4>Component Performance</h4>
        <table>
            <thead>
                <tr>
                    <th>Component</th>
                    <th>Renders</th>
                    <th>Avg Time</th>
                    <th>Total Time</th>
                </tr>
            </thead>
            <tbody>
                @foreach (var stat in componentReport.ComponentStats.Take(20))
                {
                    <tr>
                        <td>@GetShortTypeName(stat.ComponentType)</td>
                        <td>@stat.TotalRenders</td>
                        <td>@stat.AverageRenderTime.ToString("F2") ms</td>
                        <td>@stat.TotalRenderTime ms</td>
                    </tr>
                }
            </tbody>
        </table>
    </div>
</div>

@if (leakReport != null && leakReport.PotentialLeaks.Any())
{
    <div class="leak-report">
        <h4>Potential Memory Leaks</h4>
        <table>
            <thead>
                <tr>
                    <th>Identifier</th>
                    <th>Type</th>
                    <th>Age</th>
                    <th>Size</th>
                </tr>
            </thead>
            <tbody>
                @foreach (var leak in leakReport.PotentialLeaks.Take(10))
                {
                    <tr>
                        <td>@leak.Identifier</td>
                        <td>@GetShortTypeName(leak.Type)</td>
                        <td>@((DateTime.UtcNow - leak.CreatedAt).ToString(@"mm\:ss"))</td>
                        <td>@leak.Size bytes</td>
                    </tr>
                }
            </tbody>
        </table>
    </div>
}

@code {
    private Timer? refreshTimer;
    private GCMetrics gcMetrics = new();
    private ComponentProfilingReport componentReport = new();
    private MemoryLeakReport? leakReport;
    
    protected override async Task OnInitializedAsync()
    {
        await RefreshData();
        
        // Auto-refresh every 5 seconds
        refreshTimer = new Timer(async _ => 
        {
            await RefreshData();
            await InvokeAsync(StateHasChanged);
        }, null, TimeSpan.FromSeconds(5), TimeSpan.FromSeconds(5));
    }
    
    private Task RefreshData()
    {
        gcMetrics = GCManager.GetMetrics();
        componentReport = ComponentProfiler.GetReport();
        return Task.CompletedTask;
    }
    
    private void ResetProfiler()
    {
        ComponentProfiler.Reset();
        RefreshData();
    }
    
    private void ForceGC()
    {
        GCManager.ForceCollection();
        RefreshData();
    }
    
    private void GenerateLeakReport()
    {
        leakReport = LeakDetector.GenerateReport();
    }
    
    private string GetShortTypeName(string fullTypeName)
    {
        var lastDot = fullTypeName.LastIndexOf('.');
        return lastDot >= 0 ? fullTypeName.Substring(lastDot + 1) : fullTypeName;
    }
    
    public void Dispose()
    {
        refreshTimer?.Dispose();
    }
}
```

## 마무리

효과적인 메모리 관리와 체계적인 디버깅은 고품질 Blazor 애플리케이션 개발의 핵심입니다. 메모리 누수 감지, GC 최적화, 성능 프로파일링, 커스텀 디버깅 도구를 활용하여 성능 문제를 조기에 발견하고 해결할 수 있습니다. 정기적인 모니터링과 프로파일링을 통해 애플리케이션의 안정성과 성능을 지속적으로 개선할 수 있습니다.