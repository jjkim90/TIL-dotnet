# 데이터 바인딩 고급

## 개요

Blazor의 데이터 바인딩은 컴포넌트와 DOM 간의 동기화를 자동화합니다. 이 장에서는 양방향 바인딩의 내부 동작, 커스텀 바인딩 구현, 변경 알림 패턴 등 고급 데이터 바인딩 기법을 학습합니다.

## 1. 양방향 바인딩 심화

### 1.1 @bind 디렉티브의 내부 동작

```csharp
// @bind는 다음과 같이 변환됩니다:
// <input @bind="searchText" />
// ↓
// <input value="@searchText" @onchange="@((e) => searchText = e.Value.ToString())" />

// CustomBindingExample.razor
@page "/binding/custom"

<h3>Custom Two-Way Binding</h3>

<!-- 기본 양방향 바인딩 -->
<input @bind="text1" @bind:event="oninput" />

<!-- 커스텀 양방향 바인딩 -->
<input value="@text2" 
       @oninput="@((e) => UpdateText2(e.Value?.ToString()))" />

<!-- 형식과 문화권 지정 -->
<input @bind="dateValue" @bind:format="yyyy-MM-dd" />
<input @bind="decimalValue" @bind:culture="cultureInfo" />

@code {
    private string text1 = "";
    private string text2 = "";
    private DateTime dateValue = DateTime.Today;
    private decimal decimalValue = 1234.56m;
    private CultureInfo cultureInfo = new CultureInfo("ko-KR");
    
    private void UpdateText2(string? value)
    {
        text2 = value ?? "";
        // 추가 처리 가능
        Console.WriteLine($"Text updated: {text2}");
    }
}
```

### 1.2 커스텀 컴포넌트 양방향 바인딩

```csharp
// ColorPicker.razor
<div class="color-picker">
    <input type="color" value="@Value" @onchange="HandleChange" />
    <span>Selected: @Value</span>
</div>

@code {
    [Parameter] public string Value { get; set; } = "#000000";
    [Parameter] public EventCallback<string> ValueChanged { get; set; }
    
    private async Task HandleChange(ChangeEventArgs e)
    {
        var newValue = e.Value?.ToString() ?? "#000000";
        await ValueChanged.InvokeAsync(newValue);
    }
}

// NumericInput.razor
<div class="numeric-input">
    <button @onclick="Decrement" disabled="@(Value <= Min)">-</button>
    <input type="number" value="@Value" 
           @onchange="HandleChange"
           min="@Min" max="@Max" step="@Step" />
    <button @onclick="Increment" disabled="@(Value >= Max)">+</button>
</div>

@code {
    [Parameter] public decimal Value { get; set; }
    [Parameter] public EventCallback<decimal> ValueChanged { get; set; }
    [Parameter] public decimal Min { get; set; } = 0;
    [Parameter] public decimal Max { get; set; } = 100;
    [Parameter] public decimal Step { get; set; } = 1;
    
    private async Task HandleChange(ChangeEventArgs e)
    {
        if (decimal.TryParse(e.Value?.ToString(), out var value))
        {
            value = Math.Max(Min, Math.Min(Max, value));
            await ValueChanged.InvokeAsync(value);
        }
    }
    
    private async Task Increment()
    {
        var newValue = Math.Min(Value + Step, Max);
        await ValueChanged.InvokeAsync(newValue);
    }
    
    private async Task Decrement()
    {
        var newValue = Math.Max(Value - Step, Min);
        await ValueChanged.InvokeAsync(newValue);
    }
}

// 사용 예제
<ColorPicker @bind-Value="selectedColor" />
<NumericInput @bind-Value="quantity" Min="1" Max="10" />
```

## 2. 커스텀 바인딩 구현

### 2.1 복잡한 객체 바인딩

```csharp
// AddressInput.razor
<div class="address-input">
    <input placeholder="Street" value="@Value.Street" 
           @onchange="@((e) => UpdateField(nameof(Address.Street), e.Value))" />
    <input placeholder="City" value="@Value.City" 
           @onchange="@((e) => UpdateField(nameof(Address.City), e.Value))" />
    <input placeholder="PostalCode" value="@Value.PostalCode" 
           @onchange="@((e) => UpdateField(nameof(Address.PostalCode), e.Value))" />
</div>

@code {
    [Parameter] public Address Value { get; set; } = new();
    [Parameter] public EventCallback<Address> ValueChanged { get; set; }
    
    private async Task UpdateField(string fieldName, object? value)
    {
        var newAddress = new Address
        {
            Street = Value.Street,
            City = Value.City,
            PostalCode = Value.PostalCode
        };
        
        var property = typeof(Address).GetProperty(fieldName);
        property?.SetValue(newAddress, value?.ToString());
        
        await ValueChanged.InvokeAsync(newAddress);
    }
    
    public class Address
    {
        public string Street { get; set; } = "";
        public string City { get; set; } = "";
        public string PostalCode { get; set; } = "";
    }
}

// JsonEditor.razor - JSON 객체 바인딩
<div class="json-editor">
    <textarea value="@jsonText" @onchange="HandleJsonChange"></textarea>
    @if (!string.IsNullOrEmpty(errorMessage))
    {
        <div class="error">@errorMessage</div>
    }
</div>

@code {
    [Parameter] public object? Value { get; set; }
    [Parameter] public EventCallback<object?> ValueChanged { get; set; }
    
    private string jsonText = "";
    private string errorMessage = "";
    
    protected override void OnParametersSet()
    {
        try
        {
            jsonText = JsonSerializer.Serialize(Value, new JsonSerializerOptions 
            { 
                WriteIndented = true 
            });
        }
        catch
        {
            jsonText = "{}";
        }
    }
    
    private async Task HandleJsonChange(ChangeEventArgs e)
    {
        var text = e.Value?.ToString() ?? "";
        jsonText = text;
        errorMessage = "";
        
        try
        {
            var parsed = JsonSerializer.Deserialize<object>(text);
            await ValueChanged.InvokeAsync(parsed);
        }
        catch (JsonException ex)
        {
            errorMessage = $"Invalid JSON: {ex.Message}";
        }
    }
}
```

### 2.2 고급 바인딩 변환기

```csharp
// BindingConverters.cs
public static class BindingConverters
{
    public static Func<T1, T2> Create<T1, T2>(Func<T1, T2> converter) => converter;
    public static Func<T1, T2, T3> Create<T1, T2, T3>(Func<T1, T2, T3> converter) => converter;
}

// ConvertibleInput.razor
@typeparam TValue
@typeparam TDisplay

<input type="text" value="@displayValue" 
       @onchange="HandleChange" 
       placeholder="@Placeholder" />

@code {
    [Parameter] public TValue Value { get; set; } = default!;
    [Parameter] public EventCallback<TValue> ValueChanged { get; set; }
    [Parameter] public Func<TValue, TDisplay>? DisplayConverter { get; set; }
    [Parameter] public Func<string, TValue>? ValueConverter { get; set; }
    [Parameter] public string Placeholder { get; set; } = "";
    
    private TDisplay? displayValue;
    
    protected override void OnParametersSet()
    {
        if (DisplayConverter != null)
        {
            displayValue = DisplayConverter(Value);
        }
        else
        {
            displayValue = (TDisplay?)(object?)Value;
        }
    }
    
    private async Task HandleChange(ChangeEventArgs e)
    {
        var input = e.Value?.ToString() ?? "";
        
        TValue newValue;
        if (ValueConverter != null)
        {
            try
            {
                newValue = ValueConverter(input);
                await ValueChanged.InvokeAsync(newValue);
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Conversion error: {ex.Message}");
            }
        }
        else
        {
            // 기본 변환 시도
            try
            {
                var converted = Convert.ChangeType(input, typeof(TValue));
                if (converted != null)
                {
                    await ValueChanged.InvokeAsync((TValue)converted);
                }
            }
            catch
            {
                // 변환 실패
            }
        }
    }
}

// 사용 예제
<ConvertibleInput @bind-Value="phoneNumber"
                  DisplayConverter="@(FormatPhoneNumber)"
                  ValueConverter="@(ParsePhoneNumber)"
                  Placeholder="(123) 456-7890" />

@code {
    private string phoneNumber = "";
    
    private string FormatPhoneNumber(string phone)
    {
        if (phone.Length == 10)
        {
            return $"({phone.Substring(0, 3)}) {phone.Substring(3, 3)}-{phone.Substring(6)}";
        }
        return phone;
    }
    
    private string ParsePhoneNumber(string formatted)
    {
        return Regex.Replace(formatted, @"[^\d]", "");
    }
}
```

## 3. 변경 알림 패턴

### 3.1 INotifyPropertyChanged 구현

```csharp
// ObservableModel.cs
public class ObservableModel : INotifyPropertyChanged
{
    private string _name = "";
    private int _age;
    
    public string Name
    {
        get => _name;
        set
        {
            if (_name != value)
            {
                _name = value;
                OnPropertyChanged();
            }
        }
    }
    
    public int Age
    {
        get => _age;
        set
        {
            if (_age != value)
            {
                _age = value;
                OnPropertyChanged();
            }
        }
    }
    
    public event PropertyChangedEventHandler? PropertyChanged;
    
    protected virtual void OnPropertyChanged([CallerMemberName] string? propertyName = null)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }
}

// ObservableBinding.razor
@implements IDisposable

<div class="observable-binding">
    <h4>Observable Model Binding</h4>
    <input @bind="model.Name" @bind:event="oninput" />
    <input type="number" @bind="model.Age" @bind:event="oninput" />
    
    <p>Name: @model.Name</p>
    <p>Age: @model.Age</p>
    <p>Updates: @updateCount</p>
</div>

@code {
    private ObservableModel model = new();
    private int updateCount = 0;
    
    protected override void OnInitialized()
    {
        model.PropertyChanged += HandlePropertyChanged;
    }
    
    private void HandlePropertyChanged(object? sender, PropertyChangedEventArgs e)
    {
        updateCount++;
        InvokeAsync(StateHasChanged);
    }
    
    public void Dispose()
    {
        model.PropertyChanged -= HandlePropertyChanged;
    }
}
```

### 3.2 반응형 바인딩

```csharp
// ReactiveProperty.cs
public class ReactiveProperty<T> : INotifyPropertyChanged, IObservable<T>
{
    private T _value;
    private readonly List<IObserver<T>> _observers = new();
    
    public T Value
    {
        get => _value;
        set
        {
            if (!EqualityComparer<T>.Default.Equals(_value, value))
            {
                _value = value;
                OnPropertyChanged();
                NotifyObservers(value);
            }
        }
    }
    
    public ReactiveProperty(T initialValue = default!)
    {
        _value = initialValue;
    }
    
    public IDisposable Subscribe(IObserver<T> observer)
    {
        _observers.Add(observer);
        observer.OnNext(_value);
        return new Unsubscriber(_observers, observer);
    }
    
    private void NotifyObservers(T value)
    {
        foreach (var observer in _observers.ToList())
        {
            observer.OnNext(value);
        }
    }
    
    public event PropertyChangedEventHandler? PropertyChanged;
    
    protected virtual void OnPropertyChanged([CallerMemberName] string? propertyName = null)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }
    
    private class Unsubscriber : IDisposable
    {
        private readonly List<IObserver<T>> _observers;
        private readonly IObserver<T> _observer;
        
        public Unsubscriber(List<IObserver<T>> observers, IObserver<T> observer)
        {
            _observers = observers;
            _observer = observer;
        }
        
        public void Dispose()
        {
            _observers.Remove(_observer);
        }
    }
}

// ReactiveComponent.razor
@implements IDisposable

<div class="reactive-component">
    <h4>Reactive Binding</h4>
    <input value="@searchTerm.Value" 
           @oninput="@((e) => searchTerm.Value = e.Value?.ToString() ?? "")" />
    
    <h5>Filtered Results:</h5>
    <ul>
        @foreach (var item in filteredItems)
        {
            <li>@item</li>
        }
    </ul>
</div>

@code {
    private ReactiveProperty<string> searchTerm = new("");
    private List<string> allItems = new() { "Apple", "Banana", "Cherry", "Date", "Elderberry" };
    private List<string> filteredItems = new();
    private IDisposable? subscription;
    
    protected override void OnInitialized()
    {
        subscription = searchTerm
            .Select(term => term.ToLower())
            .Throttle(TimeSpan.FromMilliseconds(300))
            .DistinctUntilChanged()
            .Subscribe(new Observer<string>(term =>
            {
                filteredItems = allItems
                    .Where(item => item.ToLower().Contains(term))
                    .ToList();
                InvokeAsync(StateHasChanged);
            }));
    }
    
    public void Dispose()
    {
        subscription?.Dispose();
    }
    
    private class Observer<T> : IObserver<T>
    {
        private readonly Action<T> _onNext;
        
        public Observer(Action<T> onNext)
        {
            _onNext = onNext;
        }
        
        public void OnNext(T value) => _onNext(value);
        public void OnError(Exception error) { }
        public void OnCompleted() { }
    }
}
```

## 4. 성능 최적화

### 4.1 바인딩 성능 최적화

```csharp
// OptimizedBinding.razor
@page "/binding/optimized"

<h3>Optimized Binding</h3>

<!-- 불필요한 렌더링 방지 -->
<OptimizedInput @bind-Value="value1" DebounceDelay="500" />

<!-- 대량 데이터 바인딩 -->
<VirtualizedList Items="largeDataSet" @bind-SelectedItem="selectedItem">
    <ItemTemplate Context="item">
        <div>@item.Name - @item.Value</div>
    </ItemTemplate>
</VirtualizedList>

@code {
    private string value1 = "";
    private DataItem? selectedItem;
    private List<DataItem> largeDataSet = GenerateLargeDataSet();
    
    private static List<DataItem> GenerateLargeDataSet()
    {
        return Enumerable.Range(1, 10000)
            .Select(i => new DataItem { Id = i, Name = $"Item {i}", Value = i * 10 })
            .ToList();
    }
    
    public class DataItem
    {
        public int Id { get; set; }
        public string Name { get; set; } = "";
        public int Value { get; set; }
    }
}

// OptimizedInput.razor
<input value="@currentValue" @oninput="HandleInput" />

@code {
    [Parameter] public string Value { get; set; } = "";
    [Parameter] public EventCallback<string> ValueChanged { get; set; }
    [Parameter] public int DebounceDelay { get; set; } = 300;
    
    private string currentValue = "";
    private Timer? debounceTimer;
    
    protected override void OnParametersSet()
    {
        if (currentValue != Value)
        {
            currentValue = Value;
        }
    }
    
    private void HandleInput(ChangeEventArgs e)
    {
        currentValue = e.Value?.ToString() ?? "";
        
        debounceTimer?.Dispose();
        debounceTimer = new Timer(async _ =>
        {
            if (currentValue != Value)
            {
                await InvokeAsync(async () =>
                {
                    await ValueChanged.InvokeAsync(currentValue);
                });
            }
        }, null, DebounceDelay, Timeout.Infinite);
    }
    
    public void Dispose()
    {
        debounceTimer?.Dispose();
    }
}
```

### 4.2 메모리 효율적인 바인딩

```csharp
// MemoryEfficientBinding.razor
@implements IDisposable

<div class="memory-efficient">
    <h4>Memory Efficient Binding</h4>
    
    <!-- WeakReference 사용 -->
    <button @onclick="CreateLargeObject">Create Large Object</button>
    <button @onclick="ClearReferences">Clear References</button>
    <button @onclick="GarbageCollect">Force GC</button>
    
    <p>Active Objects: @activeObjects</p>
    <p>Weak References: @weakReferences.Count</p>
</div>

@code {
    private List<WeakReference> weakReferences = new();
    private int activeObjects = 0;
    
    private void CreateLargeObject()
    {
        var largeObject = new LargeDataObject
        {
            Data = new byte[1024 * 1024], // 1MB
            Id = Guid.NewGuid()
        };
        
        weakReferences.Add(new WeakReference(largeObject));
        activeObjects++;
    }
    
    private void ClearReferences()
    {
        weakReferences.RemoveAll(wr => !wr.IsAlive);
        activeObjects = weakReferences.Count(wr => wr.IsAlive);
    }
    
    private void GarbageCollect()
    {
        GC.Collect();
        GC.WaitForPendingFinalizers();
        GC.Collect();
        
        activeObjects = weakReferences.Count(wr => wr.IsAlive);
    }
    
    public void Dispose()
    {
        weakReferences.Clear();
    }
    
    private class LargeDataObject
    {
        public Guid Id { get; set; }
        public byte[] Data { get; set; } = Array.Empty<byte>();
    }
}
```

## 5. 고급 시나리오

### 5.1 다중 값 바인딩

```csharp
// MultiValueBinding.razor
<div class="multi-value-binding">
    <h4>RGB Color Selector</h4>
    <input type="range" min="0" max="255" @bind="red" @bind:event="oninput" />
    <input type="range" min="0" max="255" @bind="green" @bind:event="oninput" />
    <input type="range" min="0" max="255" @bind="blue" @bind:event="oninput" />
    
    <div style="background-color: @rgbColor; width: 100px; height: 100px;"></div>
    <p>RGB: @rgbColor</p>
    <p>Hex: @hexColor</p>
</div>

@code {
    private int red = 128;
    private int green = 128;
    private int blue = 128;
    
    private string rgbColor => $"rgb({red},{green},{blue})";
    private string hexColor => $"#{red:X2}{green:X2}{blue:X2}";
    
    [Parameter] public string Value { get; set; } = "#808080";
    [Parameter] public EventCallback<string> ValueChanged { get; set; }
    
    protected override async Task OnParametersSetAsync()
    {
        if (Value.StartsWith("#") && Value.Length == 7)
        {
            red = Convert.ToInt32(Value.Substring(1, 2), 16);
            green = Convert.ToInt32(Value.Substring(3, 2), 16);
            blue = Convert.ToInt32(Value.Substring(5, 2), 16);
        }
        
        await ValueChanged.InvokeAsync(hexColor);
    }
}
```

### 5.2 계산된 바인딩

```csharp
// ComputedBinding.razor
<div class="computed-binding">
    <h4>Shopping Cart</h4>
    
    @foreach (var item in cartItems)
    {
        <div class="cart-item">
            <span>@item.Name</span>
            <input type="number" min="1" value="@item.Quantity" 
                   @onchange="@((e) => UpdateQuantity(item, e.Value))" />
            <span>$@item.Price</span>
            <span>$@(item.Price * item.Quantity)</span>
        </div>
    }
    
    <div class="totals">
        <p>Subtotal: $@Subtotal</p>
        <p>Tax (@(TaxRate:P)): $@Tax</p>
        <p>Total: $@Total</p>
    </div>
</div>

@code {
    private List<CartItem> cartItems = new()
    {
        new() { Name = "Item 1", Price = 10.00m, Quantity = 1 },
        new() { Name = "Item 2", Price = 20.00m, Quantity = 2 }
    };
    
    private decimal TaxRate = 0.08m;
    
    private decimal Subtotal => cartItems.Sum(i => i.Price * i.Quantity);
    private decimal Tax => Subtotal * TaxRate;
    private decimal Total => Subtotal + Tax;
    
    private void UpdateQuantity(CartItem item, object? value)
    {
        if (int.TryParse(value?.ToString(), out var quantity))
        {
            item.Quantity = Math.Max(1, quantity);
        }
    }
    
    private class CartItem
    {
        public string Name { get; set; } = "";
        public decimal Price { get; set; }
        public int Quantity { get; set; }
    }
}
```

## 마무리

Blazor의 고급 데이터 바인딩 기능을 활용하면 복잡한 UI 시나리오도 효율적으로 구현할 수 있습니다. 양방향 바인딩, 커스텀 바인딩 구현, 변경 알림 패턴을 통해 반응형 UI를 구축하고, 성능 최적화 기법을 적용하여 대량의 데이터도 효과적으로 처리할 수 있습니다.