# 상태 관리 패턴

## 개요

Blazor 애플리케이션에서 상태 관리는 컴포넌트 간 데이터 공유와 동기화를 위한 핵심 요소입니다. 이 장에서는 Flux/Redux 패턴, Fluxor 라이브러리 활용, 그리고 커스텀 상태 관리 솔루션 구현 방법을 심화 학습합니다.

## 1. 상태 관리의 필요성

### 1.1 상태 관리가 필요한 경우

```csharp
// 문제 상황: Props Drilling
// GrandParent.razor
<Parent UserData="@userData" OnUserUpdate="HandleUserUpdate" />

@code {
    private UserData userData = new();
    
    private void HandleUserUpdate(UserData updated)
    {
        userData = updated;
    }
}

// Parent.razor
<Child UserData="@UserData" OnUserUpdate="@OnUserUpdate" />

@code {
    [Parameter] public UserData UserData { get; set; } = default!;
    [Parameter] public EventCallback<UserData> OnUserUpdate { get; set; }
}

// Child.razor
<GrandChild UserData="@UserData" OnUserUpdate="@OnUserUpdate" />

@code {
    [Parameter] public UserData UserData { get; set; } = default!;
    [Parameter] public EventCallback<UserData> OnUserUpdate { get; set; }
}

// 해결책: 중앙 집중식 상태 관리
// AppState.cs
public class AppState
{
    private UserData _userData = new();
    public UserData UserData => _userData;
    public event Action? OnChange;
    
    public void UpdateUser(UserData userData)
    {
        _userData = userData;
        NotifyStateChanged();
    }
    
    private void NotifyStateChanged() => OnChange?.Invoke();
}
```

### 1.2 상태 관리 패턴 비교

```csharp
// 1. 단순 서비스 패턴
public class SimpleStateService
{
    private readonly Dictionary<string, object> _state = new();
    
    public T? Get<T>(string key)
    {
        return _state.TryGetValue(key, out var value) ? (T)value : default;
    }
    
    public void Set<T>(string key, T value)
    {
        _state[key] = value!;
    }
}

// 2. Observable 패턴
public class ObservableState<T>
{
    private T _value;
    private readonly List<Action<T>> _observers = new();
    
    public T Value
    {
        get => _value;
        set
        {
            _value = value;
            NotifyObservers();
        }
    }
    
    public void Subscribe(Action<T> observer)
    {
        _observers.Add(observer);
        observer(_value); // 초기값 전달
    }
    
    public void Unsubscribe(Action<T> observer)
    {
        _observers.Remove(observer);
    }
    
    private void NotifyObservers()
    {
        foreach (var observer in _observers.ToList())
        {
            observer(_value);
        }
    }
}

// 3. Redux 스타일 패턴
public interface IAction { }
public interface IReducer<TState>
{
    TState Reduce(TState state, IAction action);
}

public class Store<TState>
{
    private TState _state;
    private readonly IReducer<TState> _reducer;
    private readonly List<Action<TState>> _subscribers = new();
    
    public TState State => _state;
    
    public Store(TState initialState, IReducer<TState> reducer)
    {
        _state = initialState;
        _reducer = reducer;
    }
    
    public void Dispatch(IAction action)
    {
        _state = _reducer.Reduce(_state, action);
        NotifySubscribers();
    }
    
    private void NotifySubscribers()
    {
        foreach (var subscriber in _subscribers)
        {
            subscriber(_state);
        }
    }
}
```

## 2. Flux/Redux 패턴 구현

### 2.1 기본 Redux 구현

```csharp
// Actions.cs
public record IncrementAction : IAction
{
    public int Amount { get; init; } = 1;
}

public record DecrementAction : IAction
{
    public int Amount { get; init; } = 1;
}

public record SetCounterAction : IAction
{
    public int Value { get; init; }
}

// State.cs
public record CounterState
{
    public int Count { get; init; }
    public DateTime LastUpdated { get; init; } = DateTime.Now;
    public List<string> History { get; init; } = new();
}

// Reducer.cs
public class CounterReducer : IReducer<CounterState>
{
    public CounterState Reduce(CounterState state, IAction action)
    {
        return action switch
        {
            IncrementAction increment => state with
            {
                Count = state.Count + increment.Amount,
                LastUpdated = DateTime.Now,
                History = state.History.Append($"Incremented by {increment.Amount}").ToList()
            },
            
            DecrementAction decrement => state with
            {
                Count = state.Count - decrement.Amount,
                LastUpdated = DateTime.Now,
                History = state.History.Append($"Decremented by {decrement.Amount}").ToList()
            },
            
            SetCounterAction set => state with
            {
                Count = set.Value,
                LastUpdated = DateTime.Now,
                History = state.History.Append($"Set to {set.Value}").ToList()
            },
            
            _ => state
        };
    }
}

// Store.cs
public class ReduxStore<TState> : IDisposable
{
    private TState _state;
    private readonly IReducer<TState> _rootReducer;
    private readonly List<Func<IAction, IAction>> _middleware = new();
    private readonly List<Action<TState>> _subscribers = new();
    private readonly SemaphoreSlim _semaphore = new(1, 1);
    
    public TState State => _state;
    
    public ReduxStore(TState initialState, IReducer<TState> rootReducer)
    {
        _state = initialState;
        _rootReducer = rootReducer;
    }
    
    public void AddMiddleware(Func<IAction, IAction> middleware)
    {
        _middleware.Add(middleware);
    }
    
    public async Task DispatchAsync(IAction action)
    {
        await _semaphore.WaitAsync();
        try
        {
            // 미들웨어 적용
            var processedAction = _middleware.Aggregate(action, (current, middleware) => middleware(current));
            
            // 리듀서 실행
            _state = _rootReducer.Reduce(_state, processedAction);
            
            // 구독자 알림
            NotifySubscribers();
        }
        finally
        {
            _semaphore.Release();
        }
    }
    
    public IDisposable Subscribe(Action<TState> subscriber)
    {
        _subscribers.Add(subscriber);
        subscriber(_state); // 초기 상태 전달
        
        return new Subscription(() => _subscribers.Remove(subscriber));
    }
    
    private void NotifySubscribers()
    {
        foreach (var subscriber in _subscribers.ToList())
        {
            subscriber(_state);
        }
    }
    
    public void Dispose()
    {
        _semaphore?.Dispose();
    }
    
    private class Subscription : IDisposable
    {
        private readonly Action _unsubscribe;
        
        public Subscription(Action unsubscribe)
        {
            _unsubscribe = unsubscribe;
        }
        
        public void Dispose()
        {
            _unsubscribe();
        }
    }
}

// CounterComponent.razor
@implements IDisposable
@inject ReduxStore<CounterState> Store

<div class="counter">
    <h3>Redux Counter</h3>
    <p>Count: @state?.Count</p>
    <p>Last Updated: @state?.LastUpdated.ToString("HH:mm:ss")</p>
    
    <button @onclick="Increment">+1</button>
    <button @onclick="() => IncrementBy(5)">+5</button>
    <button @onclick="Decrement">-1</button>
    <button @onclick="Reset">Reset</button>
    
    <h4>History:</h4>
    <ul>
        @if (state?.History != null)
        {
            @foreach (var item in state.History.TakeLast(5))
            {
                <li>@item</li>
            }
        }
    </ul>
</div>

@code {
    private CounterState? state;
    private IDisposable? subscription;
    
    protected override void OnInitialized()
    {
        subscription = Store.Subscribe(newState =>
        {
            state = newState;
            InvokeAsync(StateHasChanged);
        });
    }
    
    private async Task Increment()
    {
        await Store.DispatchAsync(new IncrementAction());
    }
    
    private async Task IncrementBy(int amount)
    {
        await Store.DispatchAsync(new IncrementAction { Amount = amount });
    }
    
    private async Task Decrement()
    {
        await Store.DispatchAsync(new DecrementAction());
    }
    
    private async Task Reset()
    {
        await Store.DispatchAsync(new SetCounterAction { Value = 0 });
    }
    
    public void Dispose()
    {
        subscription?.Dispose();
    }
}
```

### 2.2 고급 Redux 기능

```csharp
// Middleware.cs
public class LoggingMiddleware
{
    private readonly ILogger<LoggingMiddleware> _logger;
    
    public LoggingMiddleware(ILogger<LoggingMiddleware> logger)
    {
        _logger = logger;
    }
    
    public IAction Process(IAction action)
    {
        _logger.LogInformation($"Action dispatched: {action.GetType().Name}");
        return action;
    }
}

public class ThunkMiddleware<TState>
{
    private readonly ReduxStore<TState> _store;
    
    public ThunkMiddleware(ReduxStore<TState> store)
    {
        _store = store;
    }
    
    public IAction Process(IAction action)
    {
        if (action is ThunkAction<TState> thunk)
        {
            thunk.Execute(_store);
            return NoOpAction.Instance;
        }
        return action;
    }
}

public abstract class ThunkAction<TState> : IAction
{
    public abstract Task Execute(ReduxStore<TState> store);
}

public class NoOpAction : IAction
{
    public static readonly NoOpAction Instance = new();
}

// 비동기 액션 예제
public class LoadDataThunk : ThunkAction<AppState>
{
    private readonly IDataService _dataService;
    
    public LoadDataThunk(IDataService dataService)
    {
        _dataService = dataService;
    }
    
    public override async Task Execute(ReduxStore<AppState> store)
    {
        await store.DispatchAsync(new SetLoadingAction { IsLoading = true });
        
        try
        {
            var data = await _dataService.LoadDataAsync();
            await store.DispatchAsync(new SetDataAction { Data = data });
        }
        catch (Exception ex)
        {
            await store.DispatchAsync(new SetErrorAction { Error = ex.Message });
        }
        finally
        {
            await store.DispatchAsync(new SetLoadingAction { IsLoading = false });
        }
    }
}

// DevTools 지원
public class ReduxDevTools<TState>
{
    private readonly ReduxStore<TState> _store;
    private readonly List<(IAction action, TState state, DateTime timestamp)> _history = new();
    
    public ReduxDevTools(ReduxStore<TState> store)
    {
        _store = store;
        _store.AddMiddleware(RecordAction);
    }
    
    private IAction RecordAction(IAction action)
    {
        _history.Add((action, _store.State, DateTime.Now));
        return action;
    }
    
    public void TimeTravel(int index)
    {
        if (index >= 0 && index < _history.Count)
        {
            var (_, state, _) = _history[index];
            // 상태 복원 로직
        }
    }
    
    public IEnumerable<ActionInfo> GetHistory()
    {
        return _history.Select((h, i) => new ActionInfo
        {
            Index = i,
            ActionType = h.action.GetType().Name,
            Timestamp = h.timestamp,
            StateSummary = GenerateStateSummary(h.state)
        });
    }
    
    private string GenerateStateSummary(TState state)
    {
        // 상태 요약 생성 로직
        return JsonSerializer.Serialize(state, new JsonSerializerOptions 
        { 
            WriteIndented = true,
            MaxDepth = 3
        });
    }
    
    public class ActionInfo
    {
        public int Index { get; set; }
        public string ActionType { get; set; } = "";
        public DateTime Timestamp { get; set; }
        public string StateSummary { get; set; } = "";
    }
}
```

## 3. Fluxor 라이브러리 활용

### 3.1 Fluxor 기본 설정

```csharp
// Program.cs
builder.Services.AddFluxor(options =>
{
    options.ScanAssemblies(typeof(Program).Assembly);
    options.UseReduxDevTools();
    options.UseRouting();
});

// CounterState.cs
[FeatureState]
public class CounterState
{
    public int Count { get; }
    public bool IsLoading { get; }
    
    private CounterState() { } // Fluxor용 기본 생성자
    
    public CounterState(int count, bool isLoading)
    {
        Count = count;
        IsLoading = isLoading;
    }
}

// CounterActions.cs
public class IncrementCounterAction
{
    public int Amount { get; }
    
    public IncrementCounterAction(int amount = 1)
    {
        Amount = amount;
    }
}

public class FetchCounterAction { }

public class FetchCounterSuccessAction
{
    public int Value { get; }
    
    public FetchCounterSuccessAction(int value)
    {
        Value = value;
    }
}

// CounterReducers.cs
public static class CounterReducers
{
    [ReducerMethod]
    public static CounterState ReduceIncrementCounterAction(CounterState state, IncrementCounterAction action)
    {
        return new CounterState(state.Count + action.Amount, state.IsLoading);
    }
    
    [ReducerMethod]
    public static CounterState ReduceFetchCounterAction(CounterState state, FetchCounterAction action)
    {
        return new CounterState(state.Count, true);
    }
    
    [ReducerMethod]
    public static CounterState ReduceFetchCounterSuccessAction(CounterState state, FetchCounterSuccessAction action)
    {
        return new CounterState(action.Value, false);
    }
}

// CounterEffects.cs
public class CounterEffects
{
    private readonly ICounterService _counterService;
    
    public CounterEffects(ICounterService counterService)
    {
        _counterService = counterService;
    }
    
    [EffectMethod]
    public async Task HandleFetchCounterAction(FetchCounterAction action, IDispatcher dispatcher)
    {
        var value = await _counterService.GetCounterValueAsync();
        dispatcher.Dispatch(new FetchCounterSuccessAction(value));
    }
}

// CounterComponent.razor
@page "/fluxor-counter"
@using Fluxor
@inherits FluxorComponent
@inject IState<CounterState> CounterState
@inject IDispatcher Dispatcher

<h3>Fluxor Counter</h3>

<p>Count: @CounterState.Value.Count</p>
<p>Loading: @CounterState.Value.IsLoading</p>

<button @onclick="IncrementCounter" disabled="@CounterState.Value.IsLoading">
    Increment
</button>

<button @onclick="FetchCounter" disabled="@CounterState.Value.IsLoading">
    @if (CounterState.Value.IsLoading)
    {
        <span>Loading...</span>
    }
    else
    {
        <span>Fetch from Server</span>
    }
</button>

@code {
    private void IncrementCounter()
    {
        Dispatcher.Dispatch(new IncrementCounterAction(1));
    }
    
    private void FetchCounter()
    {
        Dispatcher.Dispatch(new FetchCounterAction());
    }
}
```

### 3.2 Fluxor 고급 패턴

```csharp
// 복잡한 상태 관리
[FeatureState]
public class ShoppingCartState
{
    public ImmutableList<CartItem> Items { get; }
    public decimal TotalPrice { get; }
    public bool IsCheckingOut { get; }
    public string? CheckoutError { get; }
    
    private ShoppingCartState()
    {
        Items = ImmutableList<CartItem>.Empty;
    }
    
    public ShoppingCartState(
        ImmutableList<CartItem> items,
        bool isCheckingOut = false,
        string? checkoutError = null)
    {
        Items = items;
        TotalPrice = items.Sum(i => i.Price * i.Quantity);
        IsCheckingOut = isCheckingOut;
        CheckoutError = checkoutError;
    }
}

public class CartItem
{
    public string Id { get; init; } = "";
    public string Name { get; init; } = "";
    public decimal Price { get; init; }
    public int Quantity { get; init; }
}

// 액션 팩토리
public static class ShoppingCartActions
{
    public class AddToCart
    {
        public CartItem Item { get; }
        public AddToCart(CartItem item) => Item = item;
    }
    
    public class RemoveFromCart
    {
        public string ItemId { get; }
        public RemoveFromCart(string itemId) => ItemId = itemId;
    }
    
    public class UpdateQuantity
    {
        public string ItemId { get; }
        public int NewQuantity { get; }
        public UpdateQuantity(string itemId, int newQuantity)
        {
            ItemId = itemId;
            NewQuantity = newQuantity;
        }
    }
    
    public class StartCheckout { }
    
    public class CheckoutSuccess
    {
        public string OrderId { get; }
        public CheckoutSuccess(string orderId) => OrderId = orderId;
    }
    
    public class CheckoutFailure
    {
        public string Error { get; }
        public CheckoutFailure(string error) => Error = error;
    }
}

// 리듀서 with 불변성
public static class ShoppingCartReducers
{
    [ReducerMethod]
    public static ShoppingCartState ReduceAddToCart(
        ShoppingCartState state,
        ShoppingCartActions.AddToCart action)
    {
        var existingItem = state.Items.FirstOrDefault(i => i.Id == action.Item.Id);
        
        var newItems = existingItem != null
            ? state.Items.Replace(existingItem, existingItem with 
              { 
                  Quantity = existingItem.Quantity + action.Item.Quantity 
              })
            : state.Items.Add(action.Item);
        
        return new ShoppingCartState(newItems);
    }
    
    [ReducerMethod]
    public static ShoppingCartState ReduceRemoveFromCart(
        ShoppingCartState state,
        ShoppingCartActions.RemoveFromCart action)
    {
        var item = state.Items.FirstOrDefault(i => i.Id == action.ItemId);
        if (item == null) return state;
        
        return new ShoppingCartState(state.Items.Remove(item));
    }
    
    [ReducerMethod]
    public static ShoppingCartState ReduceStartCheckout(
        ShoppingCartState state,
        ShoppingCartActions.StartCheckout action)
    {
        return new ShoppingCartState(state.Items, true);
    }
}

// Effect with 의존성 주입
public class ShoppingCartEffects
{
    private readonly ICheckoutService _checkoutService;
    private readonly ILogger<ShoppingCartEffects> _logger;
    
    public ShoppingCartEffects(
        ICheckoutService checkoutService,
        ILogger<ShoppingCartEffects> logger)
    {
        _checkoutService = checkoutService;
        _logger = logger;
    }
    
    [EffectMethod]
    public async Task HandleStartCheckout(
        ShoppingCartActions.StartCheckout action,
        IDispatcher dispatcher)
    {
        try
        {
            var state = // Get current state
            var orderId = await _checkoutService.ProcessCheckoutAsync(state.Items);
            dispatcher.Dispatch(new ShoppingCartActions.CheckoutSuccess(orderId));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Checkout failed");
            dispatcher.Dispatch(new ShoppingCartActions.CheckoutFailure(ex.Message));
        }
    }
}

// 선택자 (Selectors)
public static class ShoppingCartSelectors
{
    public static decimal GetTotalPrice(ShoppingCartState state)
        => state.Items.Sum(i => i.Price * i.Quantity);
    
    public static int GetItemCount(ShoppingCartState state)
        => state.Items.Sum(i => i.Quantity);
    
    public static bool HasItems(ShoppingCartState state)
        => state.Items.Any();
    
    public static CartItem? GetItemById(ShoppingCartState state, string itemId)
        => state.Items.FirstOrDefault(i => i.Id == itemId);
}
```

## 4. 커스텀 상태 관리 솔루션

### 4.1 반응형 상태 관리

```csharp
// ReactiveStateManager.cs
public interface IStateManager
{
    T GetState<T>(string key) where T : class, new();
    void SetState<T>(string key, T state) where T : class;
    IObservable<T> ObserveState<T>(string key) where T : class, new();
}

public class ReactiveStateManager : IStateManager, IDisposable
{
    private readonly Dictionary<string, object> _states = new();
    private readonly Dictionary<string, Subject<object>> _subjects = new();
    private readonly ReaderWriterLockSlim _lock = new();
    
    public T GetState<T>(string key) where T : class, new()
    {
        _lock.EnterReadLock();
        try
        {
            if (_states.TryGetValue(key, out var state) && state is T typedState)
            {
                return typedState;
            }
            
            var newState = new T();
            _lock.EnterWriteLock();
            try
            {
                _states[key] = newState;
                return newState;
            }
            finally
            {
                _lock.ExitWriteLock();
            }
        }
        finally
        {
            _lock.ExitReadLock();
        }
    }
    
    public void SetState<T>(string key, T state) where T : class
    {
        _lock.EnterWriteLock();
        try
        {
            _states[key] = state;
            
            if (_subjects.TryGetValue(key, out var subject))
            {
                subject.OnNext(state);
            }
        }
        finally
        {
            _lock.ExitWriteLock();
        }
    }
    
    public IObservable<T> ObserveState<T>(string key) where T : class, new()
    {
        _lock.EnterUpgradeableReadLock();
        try
        {
            if (!_subjects.TryGetValue(key, out var subject))
            {
                _lock.EnterWriteLock();
                try
                {
                    subject = new Subject<object>();
                    _subjects[key] = subject;
                }
                finally
                {
                    _lock.ExitWriteLock();
                }
            }
            
            var currentState = GetState<T>(key);
            
            return subject
                .StartWith(currentState)
                .Where(s => s is T)
                .Cast<T>()
                .DistinctUntilChanged();
        }
        finally
        {
            _lock.ExitUpgradeableReadLock();
        }
    }
    
    public void Dispose()
    {
        foreach (var subject in _subjects.Values)
        {
            subject.Dispose();
        }
        _subjects.Clear();
        _states.Clear();
        _lock.Dispose();
    }
}

// 사용 예제 컴포넌트
@implements IDisposable
@inject IStateManager StateManager

<div class="reactive-component">
    <h3>User: @userState?.Name</h3>
    <p>Email: @userState?.Email</p>
    
    <button @onclick="UpdateUser">Update User</button>
</div>

@code {
    private UserState? userState;
    private IDisposable? subscription;
    
    protected override void OnInitialized()
    {
        subscription = StateManager.ObserveState<UserState>("user")
            .Subscribe(state =>
            {
                userState = state;
                InvokeAsync(StateHasChanged);
            });
    }
    
    private void UpdateUser()
    {
        var newState = new UserState
        {
            Name = $"User {Random.Shared.Next(100)}",
            Email = $"user{Random.Shared.Next(100)}@example.com"
        };
        
        StateManager.SetState("user", newState);
    }
    
    public void Dispose()
    {
        subscription?.Dispose();
    }
    
    public class UserState
    {
        public string Name { get; set; } = "";
        public string Email { get; set; } = "";
    }
}
```

### 4.2 영속성 있는 상태 관리

```csharp
// PersistentStateManager.cs
public interface IPersistentStateManager
{
    Task<T?> GetStateAsync<T>(string key) where T : class;
    Task SetStateAsync<T>(string key, T state) where T : class;
    Task RemoveStateAsync(string key);
    Task ClearAllAsync();
}

public class LocalStorageStateManager : IPersistentStateManager
{
    private readonly IJSRuntime _jsRuntime;
    private readonly JsonSerializerOptions _jsonOptions;
    
    public LocalStorageStateManager(IJSRuntime jsRuntime)
    {
        _jsRuntime = jsRuntime;
        _jsonOptions = new JsonSerializerOptions
        {
            PropertyNameCaseInsensitive = true,
            WriteIndented = false
        };
    }
    
    public async Task<T?> GetStateAsync<T>(string key) where T : class
    {
        try
        {
            var json = await _jsRuntime.InvokeAsync<string?>("localStorage.getItem", key);
            if (string.IsNullOrEmpty(json))
                return null;
            
            return JsonSerializer.Deserialize<T>(json, _jsonOptions);
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Error loading state for key {key}: {ex.Message}");
            return null;
        }
    }
    
    public async Task SetStateAsync<T>(string key, T state) where T : class
    {
        try
        {
            var json = JsonSerializer.Serialize(state, _jsonOptions);
            await _jsRuntime.InvokeVoidAsync("localStorage.setItem", key, json);
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Error saving state for key {key}: {ex.Message}");
        }
    }
    
    public async Task RemoveStateAsync(string key)
    {
        await _jsRuntime.InvokeVoidAsync("localStorage.removeItem", key);
    }
    
    public async Task ClearAllAsync()
    {
        await _jsRuntime.InvokeVoidAsync("localStorage.clear");
    }
}

// HybridStateManager.cs - 메모리 + 영속성
public class HybridStateManager : IDisposable
{
    private readonly ReactiveStateManager _memoryState;
    private readonly IPersistentStateManager _persistentState;
    private readonly HashSet<string> _persistentKeys = new();
    
    public HybridStateManager(
        ReactiveStateManager memoryState,
        IPersistentStateManager persistentState)
    {
        _memoryState = memoryState;
        _persistentState = persistentState;
    }
    
    public async Task<T> GetStateAsync<T>(string key, bool persist = false) where T : class, new()
    {
        // 먼저 메모리에서 확인
        var memoryValue = _memoryState.GetState<T>(key);
        if (!IsDefaultValue(memoryValue))
        {
            return memoryValue;
        }
        
        // 영속성 스토리지에서 로드
        if (persist || _persistentKeys.Contains(key))
        {
            var persistedValue = await _persistentState.GetStateAsync<T>(key);
            if (persistedValue != null)
            {
                _memoryState.SetState(key, persistedValue);
                _persistentKeys.Add(key);
                return persistedValue;
            }
        }
        
        return memoryValue;
    }
    
    public async Task SetStateAsync<T>(string key, T state, bool persist = false) where T : class
    {
        _memoryState.SetState(key, state);
        
        if (persist || _persistentKeys.Contains(key))
        {
            await _persistentState.SetStateAsync(key, state);
            _persistentKeys.Add(key);
        }
    }
    
    public IObservable<T> ObserveState<T>(string key, bool persist = false) where T : class, new()
    {
        if (persist)
        {
            _persistentKeys.Add(key);
            
            // 초기 로드
            _ = GetStateAsync<T>(key, true);
        }
        
        return _memoryState.ObserveState<T>(key);
    }
    
    private bool IsDefaultValue<T>(T value) where T : class
    {
        if (value == null) return true;
        
        var type = typeof(T);
        var defaultInstance = Activator.CreateInstance(type);
        return value.Equals(defaultInstance);
    }
    
    public void Dispose()
    {
        _memoryState.Dispose();
    }
}
```

## 5. 상태 관리 최적화

### 5.1 선택적 구독과 메모이제이션

```csharp
// SelectorStateManager.cs
public class SelectorStateManager<TState> where TState : class, new()
{
    private TState _state = new();
    private readonly Dictionary<string, object> _selectorCache = new();
    private readonly Dictionary<string, List<Action>> _selectorSubscribers = new();
    
    public TResult Select<TResult>(
        Expression<Func<TState, TResult>> selector,
        Action<TResult>? onChange = null)
    {
        var selectorKey = selector.ToString();
        var compiledSelector = selector.Compile();
        var currentValue = compiledSelector(_state);
        
        // 캐시 확인
        if (_selectorCache.TryGetValue(selectorKey, out var cached) && 
            EqualityComparer<TResult>.Default.Equals((TResult)cached, currentValue))
        {
            return (TResult)cached;
        }
        
        // 캐시 업데이트
        _selectorCache[selectorKey] = currentValue!;
        
        // 구독 등록
        if (onChange != null)
        {
            if (!_selectorSubscribers.ContainsKey(selectorKey))
            {
                _selectorSubscribers[selectorKey] = new List<Action>();
            }
            _selectorSubscribers[selectorKey].Add(() => onChange(compiledSelector(_state)));
        }
        
        return currentValue;
    }
    
    public void UpdateState(Action<TState> updater)
    {
        var previousCache = new Dictionary<string, object>(_selectorCache);
        
        updater(_state);
        
        // 변경된 선택자만 알림
        foreach (var (key, subscribers) in _selectorSubscribers)
        {
            if (previousCache.TryGetValue(key, out var previousValue))
            {
                var selector = ParseSelector(key);
                var newValue = selector.DynamicInvoke(_state);
                
                if (!Equals(previousValue, newValue))
                {
                    foreach (var subscriber in subscribers)
                    {
                        subscriber();
                    }
                }
            }
        }
    }
    
    private Delegate ParseSelector(string selectorKey)
    {
        // 실제 구현에서는 Expression을 파싱하거나 캐싱
        throw new NotImplementedException();
    }
}

// 사용 예제
public class AppStateManager : SelectorStateManager<AppState>
{
    public string GetUserName() => Select(s => s.User.Name);
    public int GetCartItemCount() => Select(s => s.Cart.Items.Count);
    public decimal GetCartTotal() => Select(s => s.Cart.Items.Sum(i => i.Price * i.Quantity));
    
    public void UpdateUser(string name, string email)
    {
        UpdateState(state =>
        {
            state.User.Name = name;
            state.User.Email = email;
        });
    }
}
```

### 5.2 상태 분할과 지연 로딩

```csharp
// ModularStateManager.cs
public interface IStateModule
{
    string Name { get; }
    Task InitializeAsync();
    void Dispose();
}

public class ModularStateManager
{
    private readonly Dictionary<string, IStateModule> _modules = new();
    private readonly Dictionary<string, Lazy<Task>> _lazyModules = new();
    
    public void RegisterModule<TModule>(string name, Func<Task<TModule>> factory) 
        where TModule : IStateModule
    {
        _lazyModules[name] = new Lazy<Task>(() => LoadModuleAsync(name, factory));
    }
    
    private async Task<TModule> LoadModuleAsync<TModule>(string name, Func<Task<TModule>> factory) 
        where TModule : IStateModule
    {
        var module = await factory();
        await module.InitializeAsync();
        _modules[name] = module;
        return module;
    }
    
    public async Task<TModule?> GetModuleAsync<TModule>(string name) 
        where TModule : IStateModule
    {
        if (_modules.TryGetValue(name, out var module))
        {
            return module as TModule;
        }
        
        if (_lazyModules.TryGetValue(name, out var lazyModule))
        {
            await lazyModule.Value;
            return _modules[name] as TModule;
        }
        
        return default;
    }
    
    public void UnloadModule(string name)
    {
        if (_modules.TryGetValue(name, out var module))
        {
            module.Dispose();
            _modules.Remove(name);
        }
        
        _lazyModules.Remove(name);
    }
}

// 모듈 예제
public class UserStateModule : IStateModule
{
    public string Name => "User";
    private UserState _state = new();
    
    public async Task InitializeAsync()
    {
        // 사용자 데이터 로드
        _state = await LoadUserDataAsync();
    }
    
    public void Dispose()
    {
        // 정리 작업
    }
    
    private async Task<UserState> LoadUserDataAsync()
    {
        // 구현
        await Task.Delay(100);
        return new UserState();
    }
}
```

## 마무리

Blazor에서의 상태 관리는 애플리케이션의 복잡도에 따라 다양한 패턴을 선택할 수 있습니다. 단순한 서비스 패턴부터 Redux/Flux 패턴, Fluxor 라이브러리, 그리고 커스텀 솔루션까지 각각의 장단점이 있습니다.

중요한 것은 애플리케이션의 요구사항에 맞는 적절한 패턴을 선택하고, 일관성 있게 적용하는 것입니다. 또한 성능 최적화를 위해 선택적 구독, 메모이제이션, 상태 분할 등의 기법을 활용하면 확장 가능하고 유지보수가 쉬운 Blazor 애플리케이션을 구축할 수 있습니다.