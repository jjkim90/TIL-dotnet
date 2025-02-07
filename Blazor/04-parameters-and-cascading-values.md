# 파라미터와 캐스케이딩 값

## 개요

Blazor 컴포넌트 간의 데이터 전달은 파라미터와 캐스케이딩 값을 통해 이루어집니다. 이 장에서는 다양한 파라미터 타입, 캐스케이딩 파라미터의 고급 사용법, 그리고 효율적인 값 전파 패턴에 대해 심화 학습합니다.

## 1. Parameter 특성 심화

### 1.1 기본 파라미터

```csharp
// BasicParameters.razor
@page "/basic-params"

<h3>Basic Parameters</h3>
<p>Name: @Name</p>
<p>Age: @Age</p>
<p>IsActive: @IsActive</p>

@code {
    [Parameter] public string Name { get; set; } = "Unknown";
    [Parameter] public int Age { get; set; }
    [Parameter] public bool IsActive { get; set; } = true;
    
    // 파라미터 유효성 검증
    protected override void OnParametersSet()
    {
        if (Age < 0)
        {
            throw new ArgumentException("Age cannot be negative");
        }
    }
}

// 사용 예제
<BasicParameters Name="John" Age="30" IsActive="true" />
```

### 1.2 복잡한 타입 파라미터

```csharp
// ComplexTypeParameters.razor
<div class="user-profile">
    <h4>@User.Name</h4>
    <p>Email: @User.Email</p>
    <p>Role: @User.Role</p>
    
    @if (Permissions?.Any() == true)
    {
        <h5>Permissions:</h5>
        <ul>
            @foreach (var permission in Permissions)
            {
                <li>@permission</li>
            }
        </ul>
    }
</div>

@code {
    [Parameter, EditorRequired] 
    public UserModel User { get; set; } = default!;
    
    [Parameter] 
    public IReadOnlyList<string>? Permissions { get; set; }
    
    public class UserModel
    {
        public string Name { get; set; } = "";
        public string Email { get; set; } = "";
        public string Role { get; set; } = "";
    }
}
```

### 1.3 파라미터 변경 감지와 반응

```csharp
// ReactiveParameters.razor
@implements IDisposable

<div class="reactive-component">
    <h4>Current Value: @Value</h4>
    <p>Previous Value: @previousValue</p>
    <p>Change Count: @changeCount</p>
    
    @if (history.Any())
    {
        <h5>History:</h5>
        <ul>
            @foreach (var item in history.TakeLast(5))
            {
                <li>@item.Timestamp: @item.OldValue → @item.NewValue</li>
            }
        </ul>
    }
</div>

@code {
    private int previousValue;
    private int changeCount;
    private List<ChangeRecord> history = new();
    private Timer? debounceTimer;
    
    [Parameter] public int Value { get; set; }
    [Parameter] public EventCallback<int> ValueChanged { get; set; }
    [Parameter] public int DebounceDelay { get; set; } = 300;
    
    protected override void OnParametersSet()
    {
        if (Value != previousValue)
        {
            var changeRecord = new ChangeRecord
            {
                OldValue = previousValue,
                NewValue = Value,
                Timestamp = DateTime.Now
            };
            
            history.Add(changeRecord);
            changeCount++;
            
            // 디바운싱 적용
            debounceTimer?.Dispose();
            debounceTimer = new Timer(async _ =>
            {
                await InvokeAsync(async () =>
                {
                    await ValueChanged.InvokeAsync(Value);
                    StateHasChanged();
                });
            }, null, DebounceDelay, Timeout.Infinite);
            
            previousValue = Value;
        }
    }
    
    public void Dispose()
    {
        debounceTimer?.Dispose();
    }
    
    private class ChangeRecord
    {
        public int OldValue { get; set; }
        public int NewValue { get; set; }
        public DateTime Timestamp { get; set; }
    }
}
```

## 2. CascadingParameter 심화

### 2.1 기본 캐스케이딩 파라미터

```csharp
// ThemeProvider.razor
<CascadingValue Value="currentTheme" Name="Theme">
    <CascadingValue Value="themeConfig" Name="ThemeConfig">
        @ChildContent
    </CascadingValue>
</CascadingValue>

@code {
    [Parameter] public RenderFragment ChildContent { get; set; } = default!;
    
    private string currentTheme = "light";
    private ThemeConfiguration themeConfig = new();
    
    protected override void OnInitialized()
    {
        themeConfig = new ThemeConfiguration
        {
            PrimaryColor = "#007bff",
            SecondaryColor = "#6c757d",
            FontFamily = "Arial, sans-serif"
        };
    }
    
    public void ChangeTheme(string theme)
    {
        currentTheme = theme;
        StateHasChanged();
    }
    
    public class ThemeConfiguration
    {
        public string PrimaryColor { get; set; } = "";
        public string SecondaryColor { get; set; } = "";
        public string FontFamily { get; set; } = "";
    }
}

// ThemedComponent.razor
<div style="color: @ThemeConfig?.PrimaryColor">
    <h3>Themed Component</h3>
    <p>Current theme: @Theme</p>
</div>

@code {
    [CascadingParameter(Name = "Theme")] 
    private string? Theme { get; set; }
    
    [CascadingParameter(Name = "ThemeConfig")] 
    private ThemeProvider.ThemeConfiguration? ThemeConfig { get; set; }
}
```

### 2.2 복잡한 캐스케이딩 시나리오

```csharp
// AppStateProvider.razor
@implements IDisposable

<CascadingValue Value="this" IsFixed="false">
    @ChildContent
</CascadingValue>

@code {
    [Parameter] public RenderFragment ChildContent { get; set; } = default!;
    
    private readonly Dictionary<string, object> stateData = new();
    private readonly Dictionary<string, List<Action>> subscribers = new();
    
    public T? GetState<T>(string key)
    {
        return stateData.TryGetValue(key, out var value) && value is T typedValue 
            ? typedValue 
            : default;
    }
    
    public void SetState<T>(string key, T value)
    {
        stateData[key] = value!;
        NotifySubscribers(key);
    }
    
    public void Subscribe(string key, Action callback)
    {
        if (!subscribers.ContainsKey(key))
        {
            subscribers[key] = new List<Action>();
        }
        subscribers[key].Add(callback);
    }
    
    public void Unsubscribe(string key, Action callback)
    {
        if (subscribers.TryGetValue(key, out var callbacks))
        {
            callbacks.Remove(callback);
        }
    }
    
    private void NotifySubscribers(string key)
    {
        if (subscribers.TryGetValue(key, out var callbacks))
        {
            foreach (var callback in callbacks.ToList())
            {
                callback.Invoke();
            }
        }
    }
    
    public void Dispose()
    {
        stateData.Clear();
        subscribers.Clear();
    }
}

// ConsumerComponent.razor
@implements IDisposable

<div class="consumer">
    <h4>User: @userName</h4>
    <p>Cart Items: @cartItemCount</p>
    <button @onclick="UpdateCart">Add to Cart</button>
</div>

@code {
    [CascadingParameter] private AppStateProvider AppState { get; set; } = default!;
    
    private string userName = "";
    private int cartItemCount = 0;
    
    protected override void OnInitialized()
    {
        userName = AppState.GetState<string>("userName") ?? "Guest";
        cartItemCount = AppState.GetState<int>("cartItemCount");
        
        AppState.Subscribe("userName", OnUserNameChanged);
        AppState.Subscribe("cartItemCount", OnCartChanged);
    }
    
    private void OnUserNameChanged()
    {
        userName = AppState.GetState<string>("userName") ?? "Guest";
        InvokeAsync(StateHasChanged);
    }
    
    private void OnCartChanged()
    {
        cartItemCount = AppState.GetState<int>("cartItemCount");
        InvokeAsync(StateHasChanged);
    }
    
    private void UpdateCart()
    {
        AppState.SetState("cartItemCount", cartItemCount + 1);
    }
    
    public void Dispose()
    {
        AppState.Unsubscribe("userName", OnUserNameChanged);
        AppState.Unsubscribe("cartItemCount", OnCartChanged);
    }
}
```

### 2.3 계층적 캐스케이딩 값

```csharp
// HierarchicalProvider.razor
<CascadingValue Value="this" Name="RootProvider">
    <div class="hierarchy-root">
        @ChildContent
    </div>
</CascadingValue>

@code {
    [Parameter] public RenderFragment ChildContent { get; set; } = default!;
    
    private readonly List<IHierarchyNode> nodes = new();
    
    public void RegisterNode(IHierarchyNode node)
    {
        nodes.Add(node);
        RebuildHierarchy();
    }
    
    public void UnregisterNode(IHierarchyNode node)
    {
        nodes.Remove(node);
        RebuildHierarchy();
    }
    
    private void RebuildHierarchy()
    {
        // 계층 구조 재구성 로직
        foreach (var node in nodes)
        {
            node.UpdateHierarchy();
        }
    }
    
    public interface IHierarchyNode
    {
        string Id { get; }
        string? ParentId { get; }
        void UpdateHierarchy();
    }
}

// HierarchyNode.razor
@implements IDisposable
@implements HierarchicalProvider.IHierarchyNode

<CascadingValue Value="this" Name="ParentNode">
    <div class="hierarchy-node" style="margin-left: @(Level * 20)px">
        <h5>@Title (Level: @Level)</h5>
        @ChildContent
    </div>
</CascadingValue>

@code {
    [CascadingParameter(Name = "RootProvider")] 
    private HierarchicalProvider RootProvider { get; set; } = default!;
    
    [CascadingParameter(Name = "ParentNode")] 
    private HierarchyNode? ParentNode { get; set; }
    
    [Parameter] public string Id { get; set; } = Guid.NewGuid().ToString();
    [Parameter] public string Title { get; set; } = "";
    [Parameter] public RenderFragment? ChildContent { get; set; }
    
    public string? ParentId => ParentNode?.Id;
    public int Level { get; private set; }
    
    protected override void OnInitialized()
    {
        RootProvider.RegisterNode(this);
        UpdateHierarchy();
    }
    
    public void UpdateHierarchy()
    {
        Level = CalculateLevel();
        StateHasChanged();
    }
    
    private int CalculateLevel()
    {
        var level = 0;
        var current = ParentNode;
        while (current != null)
        {
            level++;
            current = current.ParentNode;
        }
        return level;
    }
    
    public void Dispose()
    {
        RootProvider.UnregisterNode(this);
    }
}
```

## 3. 값 전파 패턴

### 3.1 양방향 바인딩 패턴

```csharp
// TwoWayBindingComponent.razor
<div class="two-way-binding">
    <input type="text" value="@Value" 
           @oninput="@(e => HandleInput(e.Value?.ToString()))" />
    
    <input type="range" min="@Min" max="@Max" value="@NumericValue"
           @oninput="@(e => HandleNumericInput(e.Value?.ToString()))" />
</div>

@code {
    private string? _value;
    private int _numericValue;
    
    [Parameter]
    public string? Value 
    { 
        get => _value;
        set
        {
            if (_value != value)
            {
                _value = value;
                ValueChanged.InvokeAsync(value);
            }
        }
    }
    
    [Parameter] public EventCallback<string?> ValueChanged { get; set; }
    
    [Parameter]
    public int NumericValue
    {
        get => _numericValue;
        set
        {
            if (_numericValue != value)
            {
                _numericValue = value;
                NumericValueChanged.InvokeAsync(value);
            }
        }
    }
    
    [Parameter] public EventCallback<int> NumericValueChanged { get; set; }
    [Parameter] public int Min { get; set; } = 0;
    [Parameter] public int Max { get; set; } = 100;
    
    private async Task HandleInput(string? value)
    {
        _value = value;
        await ValueChanged.InvokeAsync(value);
    }
    
    private async Task HandleNumericInput(string? value)
    {
        if (int.TryParse(value, out var numValue))
        {
            _numericValue = Math.Max(Min, Math.Min(Max, numValue));
            await NumericValueChanged.InvokeAsync(_numericValue);
        }
    }
}
```

### 3.2 이벤트 버블링 패턴

```csharp
// EventBubblingContainer.razor
<CascadingValue Value="this">
    <div @onclick="HandleContainerClick" @onclick:stopPropagation="false">
        @ChildContent
    </div>
</CascadingValue>

@code {
    [Parameter] public RenderFragment ChildContent { get; set; } = default!;
    
    private readonly List<Func<ClickEventArgs, Task>> handlers = new();
    
    public void RegisterClickHandler(Func<ClickEventArgs, Task> handler)
    {
        handlers.Add(handler);
    }
    
    public void UnregisterClickHandler(Func<ClickEventArgs, Task> handler)
    {
        handlers.Remove(handler);
    }
    
    private async Task HandleContainerClick(MouseEventArgs e)
    {
        var args = new ClickEventArgs
        {
            ClientX = e.ClientX,
            ClientY = e.ClientY,
            Button = e.Button,
            Timestamp = DateTime.Now
        };
        
        // 등록된 모든 핸들러에 이벤트 전파
        foreach (var handler in handlers.ToList())
        {
            await handler(args);
            if (args.StopPropagation)
                break;
        }
    }
    
    public class ClickEventArgs
    {
        public double ClientX { get; set; }
        public double ClientY { get; set; }
        public long Button { get; set; }
        public DateTime Timestamp { get; set; }
        public bool StopPropagation { get; set; }
    }
}

// ClickableChild.razor
@implements IDisposable

<div class="clickable-child" @onclick="HandleClick" @onclick:stopPropagation="true">
    <h5>@Title</h5>
    @ChildContent
</div>

@code {
    [CascadingParameter] private EventBubblingContainer Container { get; set; } = default!;
    [Parameter] public string Title { get; set; } = "";
    [Parameter] public RenderFragment? ChildContent { get; set; }
    
    protected override void OnInitialized()
    {
        Container.RegisterClickHandler(OnContainerClick);
    }
    
    private async Task OnContainerClick(EventBubblingContainer.ClickEventArgs args)
    {
        Console.WriteLine($"{Title} received click at {args.ClientX}, {args.ClientY}");
        await Task.CompletedTask;
    }
    
    private void HandleClick(MouseEventArgs e)
    {
        Console.WriteLine($"{Title} was clicked directly");
    }
    
    public void Dispose()
    {
        Container.UnregisterClickHandler(OnContainerClick);
    }
}
```

### 3.3 상태 동기화 패턴

```csharp
// SyncedStateProvider.razor
@implements IDisposable

<CascadingValue Value="this">
    @ChildContent
</CascadingValue>

@code {
    [Parameter] public RenderFragment ChildContent { get; set; } = default!;
    
    private readonly Dictionary<string, SyncedProperty> properties = new();
    
    public void RegisterProperty<T>(string key, T initialValue, Action<T> onChanged)
    {
        properties[key] = new SyncedProperty<T>
        {
            Value = initialValue,
            OnChanged = value => onChanged((T)value)
        };
    }
    
    public void UpdateProperty<T>(string key, T value)
    {
        if (properties.TryGetValue(key, out var property))
        {
            property.Value = value!;
            property.OnChanged?.Invoke(value!);
            
            // 다른 컴포넌트에 변경 사항 전파
            foreach (var kvp in properties.Where(p => p.Key != key))
            {
                if (kvp.Value is SyncedProperty<T> typedProp)
                {
                    // 동일한 타입의 프로퍼티 동기화
                    typedProp.OnChanged?.Invoke(value!);
                }
            }
        }
    }
    
    public T? GetProperty<T>(string key)
    {
        return properties.TryGetValue(key, out var property) && property.Value is T value
            ? value
            : default;
    }
    
    public void Dispose()
    {
        properties.Clear();
    }
    
    private abstract class SyncedProperty
    {
        public object? Value { get; set; }
        public Action<object>? OnChanged { get; set; }
    }
    
    private class SyncedProperty<T> : SyncedProperty
    {
        public new T? Value 
        { 
            get => (T?)base.Value;
            set => base.Value = value;
        }
    }
}

// SyncedComponent.razor
@implements IDisposable

<div class="synced-component">
    <h4>@ComponentId</h4>
    <input type="text" value="@syncedValue" 
           @oninput="@(e => UpdateValue(e.Value?.ToString() ?? ""))" />
    <p>Last Update: @lastUpdate</p>
</div>

@code {
    [CascadingParameter] private SyncedStateProvider StateProvider { get; set; } = default!;
    [Parameter] public string ComponentId { get; set; } = "";
    
    private string syncedValue = "";
    private DateTime lastUpdate = DateTime.Now;
    
    protected override void OnInitialized()
    {
        syncedValue = StateProvider.GetProperty<string>("sharedValue") ?? "";
        StateProvider.RegisterProperty("sharedValue", syncedValue, OnValueChanged);
    }
    
    private void OnValueChanged(string newValue)
    {
        syncedValue = newValue;
        lastUpdate = DateTime.Now;
        InvokeAsync(StateHasChanged);
    }
    
    private void UpdateValue(string value)
    {
        StateProvider.UpdateProperty("sharedValue", value);
    }
    
    public void Dispose()
    {
        // 정리 작업
    }
}
```

## 4. 고급 파라미터 패턴

### 4.1 파라미터 유효성 검증

```csharp
// ValidatedParameters.razor
@using System.ComponentModel.DataAnnotations

<div class="validated-component">
    @if (validationErrors.Any())
    {
        <div class="alert alert-danger">
            <ul>
                @foreach (var error in validationErrors)
                {
                    <li>@error</li>
                }
            </ul>
        </div>
    }
    else
    {
        <div class="content">
            <h4>@Title</h4>
            <p>Count: @Count</p>
            <p>Email: @Email</p>
        </div>
    }
</div>

@code {
    private List<string> validationErrors = new();
    
    [Parameter, Required]
    public string Title { get; set; } = "";
    
    [Parameter, Range(1, 100)]
    public int Count { get; set; }
    
    [Parameter, EmailAddress]
    public string Email { get; set; } = "";
    
    protected override void OnParametersSet()
    {
        validationErrors.Clear();
        
        // 수동 유효성 검증
        if (string.IsNullOrWhiteSpace(Title))
        {
            validationErrors.Add("Title is required");
        }
        
        if (Count < 1 || Count > 100)
        {
            validationErrors.Add("Count must be between 1 and 100");
        }
        
        if (!string.IsNullOrEmpty(Email) && !IsValidEmail(Email))
        {
            validationErrors.Add("Invalid email format");
        }
    }
    
    private bool IsValidEmail(string email)
    {
        try
        {
            var addr = new System.Net.Mail.MailAddress(email);
            return addr.Address == email;
        }
        catch
        {
            return false;
        }
    }
}
```

### 4.2 동적 파라미터

```csharp
// DynamicParameters.razor
@foreach (var param in DynamicParams)
{
    <div class="dynamic-param">
        <label>@param.Key:</label>
        <span>@param.Value</span>
    </div>
}

@code {
    [Parameter(CaptureUnmatchedValues = true)]
    public Dictionary<string, object> DynamicParams { get; set; } = new();
    
    protected override void OnParametersSet()
    {
        // 동적 파라미터 처리
        foreach (var param in DynamicParams)
        {
            Console.WriteLine($"Dynamic parameter: {param.Key} = {param.Value}");
            
            // 타입별 처리
            switch (param.Value)
            {
                case string stringValue:
                    ProcessStringParameter(param.Key, stringValue);
                    break;
                case int intValue:
                    ProcessIntParameter(param.Key, intValue);
                    break;
                case EventCallback callback:
                    ProcessEventCallbackParameter(param.Key, callback);
                    break;
            }
        }
    }
    
    private void ProcessStringParameter(string key, string value)
    {
        // 문자열 파라미터 처리
    }
    
    private void ProcessIntParameter(string key, int value)
    {
        // 정수 파라미터 처리
    }
    
    private void ProcessEventCallbackParameter(string key, EventCallback callback)
    {
        // 이벤트 콜백 처리
    }
}
```

## 5. 성능 최적화 패턴

### 5.1 파라미터 메모이제이션

```csharp
// MemoizedComponent.razor
@implements IDisposable

<div class="memoized">
    <h4>Computed Result: @computedResult</h4>
    <p>Computation Count: @computationCount</p>
</div>

@code {
    private int previousInput1;
    private int previousInput2;
    private string computedResult = "";
    private int computationCount = 0;
    
    [Parameter] public int Input1 { get; set; }
    [Parameter] public int Input2 { get; set; }
    [Parameter] public Func<int, int, string> ExpensiveComputation { get; set; } = (a, b) => $"Result: {a + b}";
    
    private readonly Dictionary<string, (DateTime timestamp, string result)> cache = new();
    private Timer? cleanupTimer;
    
    protected override void OnInitialized()
    {
        // 캐시 정리 타이머
        cleanupTimer = new Timer(_ => CleanupCache(), null, 
            TimeSpan.FromMinutes(5), TimeSpan.FromMinutes(5));
    }
    
    protected override void OnParametersSet()
    {
        if (Input1 != previousInput1 || Input2 != previousInput2)
        {
            var cacheKey = $"{Input1}:{Input2}";
            
            if (cache.TryGetValue(cacheKey, out var cached) && 
                DateTime.Now - cached.timestamp < TimeSpan.FromMinutes(1))
            {
                computedResult = cached.result;
            }
            else
            {
                computedResult = ExpensiveComputation(Input1, Input2);
                cache[cacheKey] = (DateTime.Now, computedResult);
                computationCount++;
            }
            
            previousInput1 = Input1;
            previousInput2 = Input2;
        }
    }
    
    private void CleanupCache()
    {
        var expiredKeys = cache
            .Where(kvp => DateTime.Now - kvp.Value.timestamp > TimeSpan.FromMinutes(5))
            .Select(kvp => kvp.Key)
            .ToList();
        
        foreach (var key in expiredKeys)
        {
            cache.Remove(key);
        }
    }
    
    public void Dispose()
    {
        cleanupTimer?.Dispose();
        cache.Clear();
    }
}
```

### 5.2 캐스케이딩 값 최적화

```csharp
// OptimizedCascadingProvider.razor
<CascadingValue Value="stableValue" IsFixed="true">
    <CascadingValue Value="changingValue" IsFixed="false">
        @ChildContent
    </CascadingValue>
</CascadingValue>

@code {
    [Parameter] public RenderFragment ChildContent { get; set; } = default!;
    
    // 변경되지 않는 값 (IsFixed=true)
    private readonly StableConfiguration stableValue = new()
    {
        AppName = "My App",
        Version = "1.0.0"
    };
    
    // 변경될 수 있는 값 (IsFixed=false)
    private ChangingState changingValue = new();
    
    protected override void OnInitialized()
    {
        changingValue = new ChangingState
        {
            CurrentUser = "Guest",
            Theme = "light"
        };
    }
    
    public class StableConfiguration
    {
        public string AppName { get; set; } = "";
        public string Version { get; set; } = "";
    }
    
    public class ChangingState
    {
        public string CurrentUser { get; set; } = "";
        public string Theme { get; set; } = "";
    }
}
```

## 6. 실전 예제

### 6.1 폼 상태 관리

```csharp
// FormStateProvider.razor
<CascadingValue Value="this">
    <EditForm Model="formModel" OnValidSubmit="HandleValidSubmit">
        <DataAnnotationsValidator />
        @ChildContent
        <button type="submit" disabled="@(!IsValid || IsSubmitting)">
            @if (IsSubmitting)
            {
                <span>제출 중...</span>
            }
            else
            {
                <span>제출</span>
            }
        </button>
    </EditForm>
</CascadingValue>

@code {
    [Parameter] public RenderFragment ChildContent { get; set; } = default!;
    [Parameter] public object FormModel { get; set; } = new();
    [Parameter] public EventCallback<object> OnSubmit { get; set; }
    
    private object formModel = new();
    private EditContext? editContext;
    private ValidationMessageStore? messageStore;
    
    public bool IsValid => editContext?.Validate() ?? false;
    public bool IsSubmitting { get; private set; }
    
    protected override void OnInitialized()
    {
        formModel = FormModel;
        editContext = new EditContext(formModel);
        messageStore = new ValidationMessageStore(editContext);
        
        editContext.OnFieldChanged += HandleFieldChanged;
    }
    
    private void HandleFieldChanged(object? sender, FieldChangedEventArgs e)
    {
        StateHasChanged();
    }
    
    private async Task HandleValidSubmit()
    {
        IsSubmitting = true;
        StateHasChanged();
        
        try
        {
            await OnSubmit.InvokeAsync(formModel);
        }
        finally
        {
            IsSubmitting = false;
            StateHasChanged();
        }
    }
    
    public void AddError(string fieldName, string errorMessage)
    {
        if (editContext != null && messageStore != null)
        {
            var fieldIdentifier = editContext.Field(fieldName);
            messageStore.Add(fieldIdentifier, errorMessage);
            editContext.NotifyValidationStateChanged();
        }
    }
    
    public void ClearErrors()
    {
        messageStore?.Clear();
        editContext?.NotifyValidationStateChanged();
    }
}

// FormField.razor
<div class="form-field">
    <label>@Label</label>
    @ChildContent
    <ValidationMessage For="@For" />
</div>

@code {
    [CascadingParameter] private FormStateProvider? FormState { get; set; }
    [Parameter] public string Label { get; set; } = "";
    [Parameter] public RenderFragment ChildContent { get; set; } = default!;
    [Parameter] public Expression<Func<object>> For { get; set; } = default!;
}
```

## 마무리

파라미터와 캐스케이딩 값은 Blazor 컴포넌트 간의 데이터 통신을 위한 핵심 메커니즘입니다. 적절한 파라미터 설계와 캐스케이딩 값 활용을 통해 유지보수가 쉽고 성능이 뛰어난 컴포넌트를 만들 수 있습니다.

특히 복잡한 애플리케이션에서는 값 전파 패턴을 잘 이해하고 활용하는 것이 중요하며, 성능을 고려한 최적화 기법도 함께 적용해야 합니다. 파라미터 유효성 검증, 메모이제이션, 그리고 적절한 캐스케이딩 값 설정을 통해 안정적이고 효율적인 Blazor 애플리케이션을 구축할 수 있습니다.