# 컴포넌트 생명주기와 렌더링 최적화

## 개요

Blazor 컴포넌트는 생성부터 소멸까지 명확한 생명주기를 가지며, 각 단계에서 특정 메서드가 호출됩니다. 이러한 생명주기를 이해하고 적절히 활용하면 성능이 뛰어난 Blazor 애플리케이션을 만들 수 있습니다.

## 1. 컴포넌트 생명주기 심화

### 1.1 생명주기 메서드 실행 순서

```csharp
@page "/lifecycle"
@implements IDisposable
@implements IAsyncDisposable

<h3>Component Lifecycle Demo</h3>
<p>Counter: @counter</p>
<button @onclick="IncrementCounter">Increment</button>

@code {
    private int counter = 0;
    private readonly List<string> lifecycleLog = new();

    // 1. 컴포넌트 인스턴스 생성
    public LifecycleComponent()
    {
        Log("Constructor");
    }

    // 2. 파라미터 설정 (첫 번째 호출)
    public override async Task SetParametersAsync(ParameterView parameters)
    {
        Log("SetParametersAsync - Before");
        await base.SetParametersAsync(parameters);
        Log("SetParametersAsync - After");
    }

    // 3. 컴포넌트 초기화
    protected override void OnInitialized()
    {
        Log("OnInitialized");
    }

    // 4. 컴포넌트 초기화 (비동기)
    protected override async Task OnInitializedAsync()
    {
        Log("OnInitializedAsync - Start");
        await Task.Delay(100);
        Log("OnInitializedAsync - End");
    }

    // 5. 파라미터 설정 후
    protected override void OnParametersSet()
    {
        Log("OnParametersSet");
    }

    // 6. 파라미터 설정 후 (비동기)
    protected override async Task OnParametersSetAsync()
    {
        Log("OnParametersSetAsync - Start");
        await Task.Delay(50);
        Log("OnParametersSetAsync - End");
    }

    // 7. 렌더링 여부 결정
    protected override bool ShouldRender()
    {
        var shouldRender = true; // 조건에 따라 변경
        Log($"ShouldRender: {shouldRender}");
        return shouldRender;
    }

    // 8. 렌더링 후
    protected override void OnAfterRender(bool firstRender)
    {
        Log($"OnAfterRender - FirstRender: {firstRender}");
    }

    // 9. 렌더링 후 (비동기)
    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        Log($"OnAfterRenderAsync - FirstRender: {firstRender}");
        if (firstRender)
        {
            // JavaScript interop 등 DOM 조작
            await Task.Delay(50);
        }
    }

    // 10. 컴포넌트 해제
    public void Dispose()
    {
        Log("Dispose");
    }

    // 11. 컴포넌트 해제 (비동기)
    public async ValueTask DisposeAsync()
    {
        Log("DisposeAsync");
        await Task.Delay(50);
    }

    private void IncrementCounter()
    {
        counter++;
        StateHasChanged();
    }

    private void Log(string message)
    {
        lifecycleLog.Add($"{DateTime.Now:HH:mm:ss.fff} - {message}");
        Console.WriteLine(message);
    }
}
```

### 1.2 파라미터 변경 시 생명주기

```csharp
// ParentComponent.razor
@page "/parent"

<h3>Parent Component</h3>
<button @onclick="UpdateChildParameter">Update Child Parameter</button>

<ChildComponent Value="@childValue" OnValueChanged="HandleValueChanged" />

@code {
    private int childValue = 0;

    private void UpdateChildParameter()
    {
        childValue++;
    }

    private void HandleValueChanged(int newValue)
    {
        Console.WriteLine($"Child value changed to: {newValue}");
    }
}

// ChildComponent.razor
<h4>Child Component</h4>
<p>Value: @Value</p>
<button @onclick="NotifyParent">Notify Parent</button>

@code {
    [Parameter] public int Value { get; set; }
    [Parameter] public EventCallback<int> OnValueChanged { get; set; }

    private int previousValue;

    protected override void OnParametersSet()
    {
        if (Value != previousValue)
        {
            Console.WriteLine($"Parameter changed from {previousValue} to {Value}");
            previousValue = Value;
        }
    }

    private async Task NotifyParent()
    {
        await OnValueChanged.InvokeAsync(Value + 1);
    }
}
```

## 2. 렌더링 파이프라인

### 2.1 렌더링 프로세스 이해

```csharp
public class RenderingDemoComponent : ComponentBase
{
    private readonly RenderFragment cachedRenderFragment;
    private bool shouldSkipRender;

    protected override void BuildRenderTree(RenderTreeBuilder builder)
    {
        var sequence = 0;

        builder.OpenElement(sequence++, "div");
        builder.AddAttribute(sequence++, "class", "container");

        if (!shouldSkipRender)
        {
            builder.OpenElement(sequence++, "h3");
            builder.AddContent(sequence++, "Dynamic Content");
            builder.CloseElement();

            builder.OpenComponent<ChildComponent>(sequence++);
            builder.AddAttribute(sequence++, "Value", currentValue);
            builder.CloseComponent();
        }

        builder.CloseElement();
    }

    // 수동 렌더 트리 구축
    private RenderFragment CreateDynamicContent() => builder =>
    {
        var items = GetDynamicItems();
        var sequence = 0;

        builder.OpenElement(sequence++, "ul");
        
        foreach (var item in items)
        {
            builder.OpenElement(sequence++, "li");
            builder.AddAttribute(sequence++, "key", item.Id);
            builder.AddContent(sequence++, item.Name);
            builder.CloseElement();
        }
        
        builder.CloseElement();
    };
}
```

### 2.2 StateHasChanged 심화

```csharp
@implements IHandleEvent

<h3>StateHasChanged Demo</h3>
<p>Counter: @counter</p>
<p>Auto Counter: @autoCounter</p>
<button @onclick="IncrementManual">Manual Increment</button>
<button @onclick="IncrementMultiple">Multiple Updates</button>

@code {
    private int counter = 0;
    private int autoCounter = 0;
    private System.Threading.Timer? timer;

    protected override void OnInitialized()
    {
        // 타이머로 자동 업데이트
        timer = new System.Threading.Timer(_ =>
        {
            autoCounter++;
            InvokeAsync(StateHasChanged); // UI 스레드에서 실행
        }, null, TimeSpan.FromSeconds(1), TimeSpan.FromSeconds(1));
    }

    private void IncrementManual()
    {
        counter++;
        // StateHasChanged() 자동 호출됨 (이벤트 핸들러)
    }

    private async Task IncrementMultiple()
    {
        for (int i = 0; i < 5; i++)
        {
            counter++;
            StateHasChanged(); // 각 업데이트마다 렌더링
            await Task.Delay(200);
        }
    }

    // 커스텀 이벤트 처리
    public async Task HandleEventAsync(EventCallbackWorkItem callback, object? arg)
    {
        var task = callback.InvokeAsync(arg);
        var shouldAwaitTask = task.Status != TaskStatus.RanToCompletion &&
                             task.Status != TaskStatus.Canceled;

        if (shouldAwaitTask)
        {
            await task;
        }

        StateHasChanged(); // 이벤트 처리 후 렌더링
    }

    public override void Dispose()
    {
        timer?.Dispose();
    }
}
```

## 3. ShouldRender 최적화

### 3.1 조건부 렌더링

```csharp
public class OptimizedList : ComponentBase
{
    [Parameter] public List<Item> Items { get; set; } = new();
    
    private List<Item> previousItems = new();
    private bool hasChanges;

    protected override void OnParametersSet()
    {
        hasChanges = !Items.SequenceEqual(previousItems);
        if (hasChanges)
        {
            previousItems = new List<Item>(Items);
        }
    }

    protected override bool ShouldRender()
    {
        // 변경사항이 있을 때만 렌더링
        return hasChanges;
    }

    protected override void BuildRenderTree(RenderTreeBuilder builder)
    {
        builder.OpenElement(0, "div");
        
        foreach (var item in Items)
        {
            builder.OpenComponent<ItemComponent>(1);
            builder.AddAttribute(2, "Item", item);
            builder.SetKey(item.Id); // 효율적인 diff를 위한 키 설정
            builder.CloseComponent();
        }
        
        builder.CloseElement();
    }
}
```

### 3.2 복잡한 렌더링 최적화

```csharp
@page "/complex-optimization"

<h3>Complex Rendering Optimization</h3>

<div class="filters">
    <input @bind="searchTerm" @bind:event="oninput" />
    <select @bind="selectedCategory">
        @foreach (var category in categories)
        {
            <option value="@category">@category</option>
        }
    </select>
</div>

<VirtualizedList Items="filteredItems" ItemHeight="50">
    <ItemTemplate Context="item">
        <ProductCard Product="item" />
    </ItemTemplate>
</VirtualizedList>

@code {
    private string searchTerm = "";
    private string selectedCategory = "All";
    private List<Product> allProducts = new();
    private List<Product> filteredItems = new();
    private readonly string[] categories = { "All", "Electronics", "Books", "Clothing" };

    private CancellationTokenSource? searchCts;
    private readonly SemaphoreSlim searchSemaphore = new(1, 1);

    protected override async Task OnInitializedAsync()
    {
        allProducts = await LoadProductsAsync();
        await FilterProducts();
    }

    protected override async Task OnParametersSetAsync()
    {
        await FilterProducts();
    }

    private async Task FilterProducts()
    {
        // 이전 검색 취소
        searchCts?.Cancel();
        searchCts = new CancellationTokenSource();
        var token = searchCts.Token;

        await searchSemaphore.WaitAsync();
        try
        {
            // 디바운싱
            await Task.Delay(300, token);

            filteredItems = await Task.Run(() =>
            {
                return allProducts
                    .Where(p => 
                        (selectedCategory == "All" || p.Category == selectedCategory) &&
                        (string.IsNullOrEmpty(searchTerm) || 
                         p.Name.Contains(searchTerm, StringComparison.OrdinalIgnoreCase)))
                    .ToList();
            }, token);

            StateHasChanged();
        }
        catch (OperationCanceledException)
        {
            // 검색 취소됨
        }
        finally
        {
            searchSemaphore.Release();
        }
    }

    protected override bool ShouldRender()
    {
        // 필터링 중일 때는 렌더링 스킵
        return searchSemaphore.CurrentCount > 0;
    }
}
```

## 4. 메모리 관리와 성능

### 4.1 이벤트 구독 관리

```csharp
public class EventSubscriptionComponent : ComponentBase, IDisposable
{
    [Inject] private IEventAggregator EventAggregator { get; set; }
    [Inject] private IDataService DataService { get; set; }

    private readonly List<IDisposable> subscriptions = new();
    private readonly CancellationTokenSource cts = new();

    protected override void OnInitialized()
    {
        // 이벤트 구독
        subscriptions.Add(
            EventAggregator.Subscribe<DataUpdatedEvent>(OnDataUpdated)
        );

        subscriptions.Add(
            DataService.DataChanged.Subscribe(data =>
            {
                InvokeAsync(() =>
                {
                    UpdateUI(data);
                    StateHasChanged();
                });
            })
        );

        // 주기적 업데이트
        _ = PeriodicUpdateAsync(cts.Token);
    }

    private async Task PeriodicUpdateAsync(CancellationToken cancellationToken)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromSeconds(30));
        
        while (!cancellationToken.IsCancellationRequested)
        {
            try
            {
                await timer.WaitForNextTickAsync(cancellationToken);
                await RefreshDataAsync();
            }
            catch (OperationCanceledException)
            {
                break;
            }
        }
    }

    public void Dispose()
    {
        cts.Cancel();
        cts.Dispose();
        
        foreach (var subscription in subscriptions)
        {
            subscription.Dispose();
        }
        
        subscriptions.Clear();
    }
}
```

### 4.2 대용량 데이터 처리

```csharp
@page "/large-data"
@using System.Runtime.CompilerServices

<h3>Large Data Processing</h3>

@if (isLoading)
{
    <p>Loading...</p>
}
else
{
    <div class="data-grid">
        @foreach (var chunk in dataChunks)
        {
            <DataChunk Items="chunk" @key="chunk.Id" />
        }
    </div>
}

@code {
    private bool isLoading = true;
    private List<DataChunk> dataChunks = new();
    private readonly int chunkSize = 100;

    protected override async Task OnInitializedAsync()
    {
        await LoadDataInChunksAsync();
    }

    private async Task LoadDataInChunksAsync()
    {
        var allData = await FetchLargeDataSetAsync();
        
        await foreach (var chunk in ProcessDataAsync(allData))
        {
            dataChunks.Add(chunk);
            
            // UI를 주기적으로 업데이트
            if (dataChunks.Count % 5 == 0)
            {
                StateHasChanged();
                await Task.Yield(); // UI 스레드에 제어권 양보
            }
        }
        
        isLoading = false;
        StateHasChanged();
    }

    private async IAsyncEnumerable<DataChunk> ProcessDataAsync(
        IEnumerable<DataItem> data,
        [EnumeratorCancellation] CancellationToken cancellationToken = default)
    {
        var chunk = new List<DataItem>(chunkSize);
        var chunkId = 0;

        foreach (var item in data)
        {
            cancellationToken.ThrowIfCancellationRequested();
            
            chunk.Add(item);
            
            if (chunk.Count >= chunkSize)
            {
                yield return new DataChunk 
                { 
                    Id = chunkId++, 
                    Items = chunk.ToList() 
                };
                
                chunk.Clear();
                await Task.Delay(10, cancellationToken); // 백프레셔
            }
        }

        if (chunk.Any())
        {
            yield return new DataChunk 
            { 
                Id = chunkId, 
                Items = chunk 
            };
        }
    }
}
```

## 5. 고급 렌더링 패턴

### 5.1 조건부 렌더링 최적화

```csharp
@page "/conditional-rendering"

<OptimizedConditionalRenderer Condition="showContent">
    <TrueContent>
        <ExpensiveComponent Data="complexData" />
    </TrueContent>
    <FalseContent>
        <p>Content is hidden</p>
    </FalseContent>
</OptimizedConditionalRenderer>

@code {
    private bool showContent = false;
    private ComplexData? complexData;

    protected override async Task OnInitializedAsync()
    {
        // 조건이 true일 때만 데이터 로드
        if (showContent)
        {
            complexData = await LoadComplexDataAsync();
        }
    }
}

// OptimizedConditionalRenderer.razor
@typeparam TItem

@if (Condition)
{
    @if (cachedTrueContent == null)
    {
        cachedTrueContent = TrueContent;
    }
    @cachedTrueContent
}
else
{
    @if (cachedFalseContent == null)
    {
        cachedFalseContent = FalseContent;
    }
    @cachedFalseContent
}

@code {
    [Parameter] public bool Condition { get; set; }
    [Parameter] public RenderFragment? TrueContent { get; set; }
    [Parameter] public RenderFragment? FalseContent { get; set; }

    private RenderFragment? cachedTrueContent;
    private RenderFragment? cachedFalseContent;

    protected override void OnParametersSet()
    {
        // 조건이 변경되면 캐시 무효화
        if (Condition != previousCondition)
        {
            if (Condition)
                cachedFalseContent = null;
            else
                cachedTrueContent = null;
                
            previousCondition = Condition;
        }
    }

    private bool previousCondition;
}
```

### 5.2 동적 컴포넌트 렌더링

```csharp
// DynamicComponentRenderer.razor
@using System.Reflection

<div class="dynamic-container">
    @if (componentType != null)
    {
        <DynamicComponent Type="componentType" Parameters="componentParameters" />
    }
</div>

@code {
    [Parameter] public string ComponentName { get; set; } = "";
    [Parameter] public Dictionary<string, object> Parameters { get; set; } = new();

    private Type? componentType;
    private Dictionary<string, object>? componentParameters;
    private readonly Dictionary<string, Type> componentCache = new();

    protected override void OnParametersSet()
    {
        if (!string.IsNullOrEmpty(ComponentName))
        {
            // 컴포넌트 타입 캐싱
            if (!componentCache.TryGetValue(ComponentName, out componentType))
            {
                componentType = FindComponentType(ComponentName);
                if (componentType != null)
                {
                    componentCache[ComponentName] = componentType;
                }
            }

            // 파라미터 검증 및 변환
            if (componentType != null)
            {
                componentParameters = ValidateAndConvertParameters(componentType, Parameters);
            }
        }
    }

    private Type? FindComponentType(string name)
    {
        // 현재 어셈블리에서 컴포넌트 찾기
        return Assembly.GetExecutingAssembly()
            .GetTypes()
            .FirstOrDefault(t => t.Name == name && t.IsSubclassOf(typeof(ComponentBase)));
    }

    private Dictionary<string, object> ValidateAndConvertParameters(
        Type componentType, 
        Dictionary<string, object> parameters)
    {
        var validatedParams = new Dictionary<string, object>();
        var componentProperties = componentType.GetProperties()
            .Where(p => p.GetCustomAttribute<ParameterAttribute>() != null)
            .ToDictionary(p => p.Name, p => p.PropertyType);

        foreach (var param in parameters)
        {
            if (componentProperties.TryGetValue(param.Key, out var propertyType))
            {
                try
                {
                    // 타입 변환
                    var convertedValue = Convert.ChangeType(param.Value, propertyType);
                    validatedParams[param.Key] = convertedValue;
                }
                catch (Exception ex)
                {
                    Console.WriteLine($"Parameter conversion failed: {param.Key} - {ex.Message}");
                }
            }
        }

        return validatedParams;
    }

    protected override bool ShouldRender()
    {
        // 컴포넌트 타입이 변경되었을 때만 렌더링
        return componentType != null;
    }
}
```

## 6. 실전 최적화 예제

### 6.1 대시보드 컴포넌트 최적화

```csharp
@page "/dashboard"
@implements IAsyncDisposable

<div class="dashboard">
    <div class="widgets">
        @foreach (var widget in widgets)
        {
            <DashboardWidget @key="widget.Id" 
                           WidgetData="widget" 
                           OnUpdate="HandleWidgetUpdate" />
        }
    </div>
</div>

@code {
    private List<WidgetData> widgets = new();
    private readonly Dictionary<string, DateTime> lastUpdateTimes = new();
    private readonly TimeSpan minimumUpdateInterval = TimeSpan.FromSeconds(5);
    private Timer? refreshTimer;

    protected override async Task OnInitializedAsync()
    {
        widgets = await LoadWidgetsAsync();
        
        // 주기적 새로고침 설정
        refreshTimer = new Timer(async _ => await RefreshWidgetsAsync(), 
                                null, 
                                TimeSpan.FromMinutes(1), 
                                TimeSpan.FromMinutes(1));
    }

    private async Task HandleWidgetUpdate(WidgetUpdateEvent updateEvent)
    {
        // 업데이트 속도 제한
        if (lastUpdateTimes.TryGetValue(updateEvent.WidgetId, out var lastUpdate))
        {
            if (DateTime.UtcNow - lastUpdate < minimumUpdateInterval)
            {
                return; // 너무 빈번한 업데이트 무시
            }
        }

        lastUpdateTimes[updateEvent.WidgetId] = DateTime.UtcNow;

        // 특정 위젯만 업데이트
        var widget = widgets.FirstOrDefault(w => w.Id == updateEvent.WidgetId);
        if (widget != null)
        {
            await UpdateWidgetDataAsync(widget);
            StateHasChanged();
        }
    }

    private async Task RefreshWidgetsAsync()
    {
        try
        {
            var updatedWidgets = await LoadWidgetsAsync();
            
            // 변경된 위젯만 업데이트
            var hasChanges = false;
            foreach (var updated in updatedWidgets)
            {
                var existing = widgets.FirstOrDefault(w => w.Id == updated.Id);
                if (existing == null || !existing.Equals(updated))
                {
                    hasChanges = true;
                    if (existing != null)
                    {
                        var index = widgets.IndexOf(existing);
                        widgets[index] = updated;
                    }
                }
            }

            if (hasChanges)
            {
                await InvokeAsync(StateHasChanged);
            }
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Widget refresh error: {ex.Message}");
        }
    }

    public async ValueTask DisposeAsync()
    {
        if (refreshTimer != null)
        {
            await refreshTimer.DisposeAsync();
        }
    }
}
```

## 마무리

Blazor 컴포넌트의 생명주기를 이해하고 렌더링을 최적화하는 것은 고성능 애플리케이션을 만드는 데 필수적입니다. StateHasChanged를 적절히 사용하고, ShouldRender로 불필요한 렌더링을 방지하며, 메모리 누수를 방지하는 적절한 리소스 관리를 통해 효율적인 Blazor 애플리케이션을 구축할 수 있습니다.

핵심은 각 생명주기 메서드의 목적을 이해하고, 적절한 시점에 적절한 작업을 수행하는 것입니다. 또한 대용량 데이터나 빈번한 업데이트가 필요한 경우, 가상화, 청크 처리, 디바운싱 등의 기법을 활용하여 성능을 최적화할 수 있습니다.