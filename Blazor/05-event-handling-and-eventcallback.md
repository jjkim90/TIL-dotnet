# 이벤트 처리와 EventCallback

## 개요

Blazor에서 이벤트 처리는 사용자 상호작용을 다루는 핵심 메커니즘입니다. 이 장에서는 이벤트 버블링, 커스텀 이벤트 생성, 비동기 이벤트 처리, 그리고 EventCallback의 고급 활용법을 심화 학습합니다.

## 1. 이벤트 처리 기본

### 1.1 DOM 이벤트 처리

```csharp
// BasicEventHandling.razor
@page "/events/basic"

<h3>Basic Event Handling</h3>

<div class="event-demo">
    <!-- 클릭 이벤트 -->
    <button @onclick="HandleClick">Click Me</button>
    <button @onclick="@(() => HandleClickWithArgs("Button 1"))">Button 1</button>
    <button @onclick="@(e => HandleClickWithEventArgs(e, "Button 2"))">Button 2</button>
    
    <!-- 입력 이벤트 -->
    <input @oninput="HandleInput" placeholder="Type something..." />
    <input @onchange="HandleChange" placeholder="Change and blur..." />
    
    <!-- 키보드 이벤트 -->
    <input @onkeydown="HandleKeyDown" 
           @onkeyup="HandleKeyUp"
           @onkeypress="HandleKeyPress" 
           placeholder="Press keys..." />
    
    <!-- 마우스 이벤트 -->
    <div class="mouse-area" 
         @onmouseover="HandleMouseOver"
         @onmouseout="HandleMouseOut"
         @onmousemove="HandleMouseMove">
        Mouse Area (X: @mouseX, Y: @mouseY)
    </div>
</div>

<div class="event-log">
    <h4>Event Log:</h4>
    <ul>
        @foreach (var log in eventLogs.TakeLast(10))
        {
            <li>@log</li>
        }
    </ul>
</div>

@code {
    private List<string> eventLogs = new();
    private double mouseX;
    private double mouseY;
    
    private void HandleClick()
    {
        LogEvent("Button clicked");
    }
    
    private void HandleClickWithArgs(string buttonName)
    {
        LogEvent($"{buttonName} clicked");
    }
    
    private void HandleClickWithEventArgs(MouseEventArgs e, string buttonName)
    {
        LogEvent($"{buttonName} clicked at ({e.ClientX}, {e.ClientY})");
    }
    
    private void HandleInput(ChangeEventArgs e)
    {
        LogEvent($"Input: {e.Value}");
    }
    
    private void HandleChange(ChangeEventArgs e)
    {
        LogEvent($"Changed to: {e.Value}");
    }
    
    private void HandleKeyDown(KeyboardEventArgs e)
    {
        LogEvent($"Key down: {e.Key} (Code: {e.Code})");
    }
    
    private void HandleKeyUp(KeyboardEventArgs e)
    {
        LogEvent($"Key up: {e.Key}");
    }
    
    private void HandleKeyPress(KeyboardEventArgs e)
    {
        LogEvent($"Key press: {e.Key}");
    }
    
    private void HandleMouseOver(MouseEventArgs e)
    {
        LogEvent("Mouse entered");
    }
    
    private void HandleMouseOut(MouseEventArgs e)
    {
        LogEvent("Mouse left");
    }
    
    private void HandleMouseMove(MouseEventArgs e)
    {
        mouseX = e.ClientX;
        mouseY = e.ClientY;
    }
    
    private void LogEvent(string message)
    {
        eventLogs.Add($"{DateTime.Now:HH:mm:ss.fff} - {message}");
        StateHasChanged();
    }
}
```

### 1.2 이벤트 수정자

```csharp
// EventModifiers.razor
@page "/events/modifiers"

<h3>Event Modifiers</h3>

<div class="modifiers-demo">
    <!-- preventDefault -->
    <form @onsubmit="HandleSubmit" @onsubmit:preventDefault="true">
        <input @bind="formData" />
        <button type="submit">Submit (Prevented)</button>
    </form>
    
    <!-- stopPropagation -->
    <div class="outer" @onclick="HandleOuterClick">
        Outer Div
        <div class="inner" @onclick="HandleInnerClick" @onclick:stopPropagation="true">
            Inner Div (Stop Propagation)
        </div>
    </div>
    
    <!-- 조건부 이벤트 처리 -->
    <button @onclick="HandleConditionalClick" 
            @onclick:preventDefault="@shouldPrevent"
            @onclick:stopPropagation="@shouldStop">
        Conditional Event
    </button>
    
    <!-- 다중 이벤트 수정자 -->
    <div @onmousedown="HandleMouseDown"
         @onmousedown:preventDefault="true"
         @onmousedown:stopPropagation="true">
        Drag Area (Both modifiers)
    </div>
</div>

@code {
    private string formData = "";
    private bool shouldPrevent = true;
    private bool shouldStop = false;
    
    private void HandleSubmit()
    {
        Console.WriteLine($"Form submitted with: {formData}");
    }
    
    private void HandleOuterClick()
    {
        Console.WriteLine("Outer clicked");
    }
    
    private void HandleInnerClick()
    {
        Console.WriteLine("Inner clicked");
    }
    
    private void HandleConditionalClick()
    {
        Console.WriteLine("Conditional click handled");
        shouldPrevent = !shouldPrevent;
        shouldStop = !shouldStop;
    }
    
    private void HandleMouseDown(MouseEventArgs e)
    {
        Console.WriteLine($"Mouse down at ({e.ClientX}, {e.ClientY})");
    }
}
```

## 2. EventCallback 심화

### 2.1 EventCallback vs Action

```csharp
// EventCallbackComparison.razor
@page "/events/callback-comparison"

<h3>EventCallback vs Action Comparison</h3>

<ChildWithAction OnAction="HandleAction" />
<ChildWithEventCallback OnCallback="HandleCallback" />
<ChildWithTypedEventCallback OnTypedCallback="HandleTypedCallback" />

@code {
    private void HandleAction()
    {
        Console.WriteLine("Action handled");
        // StateHasChanged() 필요
        StateHasChanged();
    }
    
    private void HandleCallback()
    {
        Console.WriteLine("EventCallback handled");
        // StateHasChanged() 자동 호출됨
    }
    
    private void HandleTypedCallback(CustomEventArgs args)
    {
        Console.WriteLine($"Typed EventCallback: {args.Message}");
    }
}

// ChildWithAction.razor
<button @onclick="InvokeAction">Trigger Action</button>

@code {
    [Parameter] public Action? OnAction { get; set; }
    
    private void InvokeAction()
    {
        OnAction?.Invoke();
    }
}

// ChildWithEventCallback.razor
<button @onclick="InvokeCallback">Trigger EventCallback</button>

@code {
    [Parameter] public EventCallback OnCallback { get; set; }
    
    private async Task InvokeCallback()
    {
        await OnCallback.InvokeAsync();
    }
}

// ChildWithTypedEventCallback.razor
<button @onclick="InvokeTypedCallback">Trigger Typed EventCallback</button>

@code {
    [Parameter] public EventCallback<CustomEventArgs> OnTypedCallback { get; set; }
    
    private async Task InvokeTypedCallback()
    {
        var args = new CustomEventArgs { Message = "Hello from child" };
        await OnTypedCallback.InvokeAsync(args);
    }
}

public class CustomEventArgs
{
    public string Message { get; set; } = "";
    public DateTime Timestamp { get; set; } = DateTime.Now;
}
```

### 2.2 체이닝과 조합

```csharp
// EventChaining.razor
<div class="event-chain">
    <h4>Event Chaining Example</h4>
    
    <ValidationInput Value="@inputValue"
                     ValueChanged="HandleValueChanged"
                     OnValidated="HandleValidated"
                     OnError="HandleError" />
    
    <p>Status: @status</p>
</div>

@code {
    private string inputValue = "";
    private string status = "Ready";
    
    private async Task HandleValueChanged(string value)
    {
        inputValue = value;
        status = "Processing...";
        await Task.Delay(500); // 시뮬레이션
    }
    
    private void HandleValidated(ValidationResult result)
    {
        status = result.IsValid ? "Valid" : "Invalid";
    }
    
    private void HandleError(string error)
    {
        status = $"Error: {error}";
    }
}

// ValidationInput.razor
<input type="text" 
       value="@Value"
       @oninput="HandleInput"
       @onblur="ValidateInput"
       class="@(isValid ? "" : "error")" />

@code {
    [Parameter] public string Value { get; set; } = "";
    [Parameter] public EventCallback<string> ValueChanged { get; set; }
    [Parameter] public EventCallback<ValidationResult> OnValidated { get; set; }
    [Parameter] public EventCallback<string> OnError { get; set; }
    
    private bool isValid = true;
    private CancellationTokenSource? cts;
    
    private async Task HandleInput(ChangeEventArgs e)
    {
        var newValue = e.Value?.ToString() ?? "";
        
        // 이전 검증 취소
        cts?.Cancel();
        cts = new CancellationTokenSource();
        
        try
        {
            // 값 변경 알림
            await ValueChanged.InvokeAsync(newValue);
            
            // 디바운스
            await Task.Delay(300, cts.Token);
            
            // 검증
            await ValidateValue(newValue, cts.Token);
        }
        catch (OperationCanceledException)
        {
            // 취소됨
        }
    }
    
    private async Task ValidateInput()
    {
        await ValidateValue(Value, CancellationToken.None);
    }
    
    private async Task ValidateValue(string value, CancellationToken cancellationToken)
    {
        try
        {
            // 비동기 검증 시뮬레이션
            await Task.Delay(100, cancellationToken);
            
            isValid = !string.IsNullOrWhiteSpace(value) && value.Length >= 3;
            
            var result = new ValidationResult
            {
                IsValid = isValid,
                Value = value,
                Message = isValid ? "Valid" : "Minimum 3 characters required"
            };
            
            await OnValidated.InvokeAsync(result);
        }
        catch (Exception ex)
        {
            await OnError.InvokeAsync(ex.Message);
        }
    }
    
    public class ValidationResult
    {
        public bool IsValid { get; set; }
        public string Value { get; set; } = "";
        public string Message { get; set; } = "";
    }
}
```

## 3. 이벤트 버블링과 캡처링

### 3.1 이벤트 전파 제어

```csharp
// EventPropagation.razor
@page "/events/propagation"

<h3>Event Propagation</h3>

<div class="propagation-demo">
    <!-- 이벤트 버블링 예제 -->
    <div class="grandparent" @onclick="HandleClick" data-level="grandparent">
        Grandparent
        <div class="parent" @onclick="HandleClick" data-level="parent">
            Parent
            <div class="child" @onclick="HandleClick" data-level="child">
                Child (Click me)
            </div>
        </div>
    </div>
    
    <!-- 이벤트 캡처링 예제 -->
    <div class="capture-demo" @onclick="HandleCapture" @onclick:capture="true">
        Capture Phase
        <button @onclick="HandleBubble">Button (Bubble)</button>
    </div>
    
    <!-- 복잡한 전파 시나리오 -->
    <EventPropagationHandler>
        <Level1>
            <Level2>
                <Level3 />
            </Level2>
        </Level1>
    </EventPropagationHandler>
</div>

<div class="event-path">
    <h4>Event Path:</h4>
    <ol>
        @foreach (var item in eventPath)
        {
            <li>@item</li>
        }
    </ol>
</div>

@code {
    private List<string> eventPath = new();
    
    private void HandleClick(MouseEventArgs e)
    {
        // 이벤트가 발생한 요소 확인
        var level = GetEventTargetLevel(e);
        eventPath.Add($"Bubble: {level} at {DateTime.Now:HH:mm:ss.fff}");
        StateHasChanged();
    }
    
    private void HandleCapture(MouseEventArgs e)
    {
        eventPath.Add($"Capture phase at {DateTime.Now:HH:mm:ss.fff}");
    }
    
    private void HandleBubble(MouseEventArgs e)
    {
        eventPath.Add($"Bubble phase (Button) at {DateTime.Now:HH:mm:ss.fff}");
    }
    
    private string GetEventTargetLevel(MouseEventArgs e)
    {
        // 실제로는 JavaScript interop으로 구현
        return "Unknown";
    }
}

// EventPropagationHandler.razor
<CascadingValue Value="this">
    <div @onclick="HandleRootClick">
        @ChildContent
    </div>
</CascadingValue>

@code {
    [Parameter] public RenderFragment ChildContent { get; set; } = default!;
    
    private List<string> propagationPath = new();
    
    public void RegisterClick(string level)
    {
        propagationPath.Add(level);
        if (propagationPath.Count == 3) // 모든 레벨 수집 완료
        {
            Console.WriteLine($"Propagation path: {string.Join(" -> ", propagationPath)}");
            propagationPath.Clear();
        }
    }
    
    private void HandleRootClick()
    {
        RegisterClick("Root");
    }
}
```

### 3.2 커스텀 이벤트 전파

```csharp
// CustomEventBubbling.razor
@implements ICustomEventTarget

<div class="custom-event-container">
    <h4>Custom Event Bubbling</h4>
    
    <CustomEventSource OnCustomEvent="HandleCustomEvent" />
    
    @if (ChildContent != null)
    {
        <CascadingValue Value="this">
            @ChildContent
        </CascadingValue>
    }
</div>

@code {
    [Parameter] public RenderFragment? ChildContent { get; set; }
    [CascadingParameter] public ICustomEventTarget? Parent { get; set; }
    
    private async Task HandleCustomEvent(CustomEvent e)
    {
        Console.WriteLine($"Handling event at level: {GetType().Name}");
        
        // 이벤트 처리
        await ProcessEvent(e);
        
        // 부모로 전파
        if (!e.StopPropagation && Parent != null)
        {
            await Parent.OnCustomEvent(e);
        }
    }
    
    public async Task OnCustomEvent(CustomEvent e)
    {
        await HandleCustomEvent(e);
    }
    
    private async Task ProcessEvent(CustomEvent e)
    {
        // 이벤트 처리 로직
        await Task.Delay(10);
    }
}

// ICustomEventTarget.cs
public interface ICustomEventTarget
{
    Task OnCustomEvent(CustomEvent e);
}

// CustomEvent.cs
public class CustomEvent
{
    public string Type { get; set; } = "";
    public object? Data { get; set; }
    public bool StopPropagation { get; set; }
    public bool PreventDefault { get; set; }
    public DateTime Timestamp { get; set; } = DateTime.Now;
}

// CustomEventSource.razor
<button @onclick="RaiseCustomEvent">Raise Custom Event</button>

@code {
    [Parameter] public EventCallback<CustomEvent> OnCustomEvent { get; set; }
    
    private async Task RaiseCustomEvent()
    {
        var customEvent = new CustomEvent
        {
            Type = "custom-click",
            Data = new { Message = "Hello from source" }
        };
        
        await OnCustomEvent.InvokeAsync(customEvent);
    }
}
```

## 4. 비동기 이벤트 처리

### 4.1 비동기 이벤트 패턴

```csharp
// AsyncEventHandling.razor
@page "/events/async"
@implements IDisposable

<h3>Async Event Handling</h3>

<div class="async-demo">
    <!-- 기본 비동기 이벤트 -->
    <button @onclick="HandleAsyncClick" disabled="@isProcessing">
        @if (isProcessing)
        {
            <span>처리 중...</span>
        }
        else
        {
            <span>비동기 작업 시작</span>
        }
    </button>
    
    <!-- 취소 가능한 비동기 이벤트 -->
    <button @onclick="HandleCancellableAsync" disabled="@isCancellable">
        시작
    </button>
    <button @onclick="CancelOperation" disabled="@(!isCancellable)">
        취소
    </button>
    
    <!-- 진행률 표시 비동기 이벤트 -->
    <button @onclick="HandleProgressAsync">
        진행률 작업
    </button>
    @if (progress > 0)
    {
        <div class="progress">
            <div class="progress-bar" style="width: @progress%">
                @progress%
            </div>
        </div>
    }
</div>

<div class="results">
    @foreach (var result in results)
    {
        <p>@result</p>
    }
</div>

@code {
    private bool isProcessing = false;
    private bool isCancellable = false;
    private int progress = 0;
    private List<string> results = new();
    private CancellationTokenSource? cts;
    
    private async Task HandleAsyncClick()
    {
        isProcessing = true;
        try
        {
            results.Add($"작업 시작: {DateTime.Now:HH:mm:ss}");
            
            // 비동기 작업 시뮬레이션
            await Task.Delay(2000);
            
            results.Add($"작업 완료: {DateTime.Now:HH:mm:ss}");
        }
        catch (Exception ex)
        {
            results.Add($"오류 발생: {ex.Message}");
        }
        finally
        {
            isProcessing = false;
        }
    }
    
    private async Task HandleCancellableAsync()
    {
        cts = new CancellationTokenSource();
        isCancellable = true;
        
        try
        {
            results.Add("취소 가능 작업 시작");
            
            for (int i = 0; i < 10; i++)
            {
                cts.Token.ThrowIfCancellationRequested();
                
                await Task.Delay(500, cts.Token);
                results.Add($"단계 {i + 1}/10 완료");
                StateHasChanged();
            }
            
            results.Add("작업 완료!");
        }
        catch (OperationCanceledException)
        {
            results.Add("작업이 취소되었습니다");
        }
        finally
        {
            isCancellable = false;
            cts?.Dispose();
        }
    }
    
    private void CancelOperation()
    {
        cts?.Cancel();
    }
    
    private async Task HandleProgressAsync()
    {
        progress = 0;
        
        var progressHandler = new Progress<int>(value =>
        {
            progress = value;
            InvokeAsync(StateHasChanged);
        });
        
        await PerformProgressOperation(progressHandler);
    }
    
    private async Task PerformProgressOperation(IProgress<int> progress)
    {
        for (int i = 0; i <= 100; i += 10)
        {
            await Task.Delay(200);
            progress.Report(i);
        }
    }
    
    public void Dispose()
    {
        cts?.Cancel();
        cts?.Dispose();
    }
}
```

### 4.2 이벤트 큐잉과 스로틀링

```csharp
// EventThrottling.razor
@implements IDisposable

<div class="throttling-demo">
    <h4>Event Throttling & Debouncing</h4>
    
    <!-- 스로틀링 -->
    <div class="throttle-area" @onmousemove="HandleMouseMoveThrottled">
        Throttled Mouse Move (Max 1 per 100ms)
        <p>Updates: @throttleCount</p>
    </div>
    
    <!-- 디바운싱 -->
    <input type="text" @oninput="HandleInputDebounced" 
           placeholder="Debounced input (300ms)" />
    <p>Debounced Value: @debouncedValue</p>
    
    <!-- 이벤트 큐 -->
    <button @onclick="QueueEvent">Queue Event</button>
    <p>Queue Size: @eventQueue.Count</p>
    <p>Processed: @processedCount</p>
</div>

@code {
    private int throttleCount = 0;
    private string debouncedValue = "";
    private int processedCount = 0;
    
    private DateTime lastThrottleTime = DateTime.MinValue;
    private Timer? debounceTimer;
    private readonly Queue<EventData> eventQueue = new();
    private readonly SemaphoreSlim processingSemaphore = new(1, 1);
    private CancellationTokenSource cts = new();
    
    protected override void OnInitialized()
    {
        _ = ProcessEventQueue(cts.Token);
    }
    
    // 스로틀링
    private void HandleMouseMoveThrottled(MouseEventArgs e)
    {
        var now = DateTime.Now;
        if (now - lastThrottleTime >= TimeSpan.FromMilliseconds(100))
        {
            lastThrottleTime = now;
            throttleCount++;
            StateHasChanged();
        }
    }
    
    // 디바운싱
    private void HandleInputDebounced(ChangeEventArgs e)
    {
        var value = e.Value?.ToString() ?? "";
        
        debounceTimer?.Dispose();
        debounceTimer = new Timer(_ =>
        {
            InvokeAsync(() =>
            {
                debouncedValue = value;
                StateHasChanged();
            });
        }, null, 300, Timeout.Infinite);
    }
    
    // 이벤트 큐잉
    private void QueueEvent()
    {
        var eventData = new EventData
        {
            Id = Guid.NewGuid(),
            Timestamp = DateTime.Now,
            Data = $"Event {eventQueue.Count + 1}"
        };
        
        eventQueue.Enqueue(eventData);
        StateHasChanged();
    }
    
    private async Task ProcessEventQueue(CancellationToken cancellationToken)
    {
        while (!cancellationToken.IsCancellationRequested)
        {
            await processingSemaphore.WaitAsync(cancellationToken);
            try
            {
                if (eventQueue.TryDequeue(out var eventData))
                {
                    await ProcessEvent(eventData);
                    processedCount++;
                    await InvokeAsync(StateHasChanged);
                }
                else
                {
                    await Task.Delay(100, cancellationToken);
                }
            }
            finally
            {
                processingSemaphore.Release();
            }
        }
    }
    
    private async Task ProcessEvent(EventData eventData)
    {
        // 이벤트 처리 시뮬레이션
        await Task.Delay(500);
        Console.WriteLine($"Processed: {eventData.Data}");
    }
    
    public void Dispose()
    {
        cts.Cancel();
        cts.Dispose();
        debounceTimer?.Dispose();
        processingSemaphore.Dispose();
    }
    
    private class EventData
    {
        public Guid Id { get; set; }
        public DateTime Timestamp { get; set; }
        public string Data { get; set; } = "";
    }
}
```

## 5. 커스텀 이벤트 생성

### 5.1 커스텀 이벤트 시스템

```csharp
// CustomEventSystem.razor
<div class="custom-event-system">
    <h4>Custom Event System</h4>
    
    <EventEmitter @ref="emitter" />
    <EventListener EventSource="emitter" />
</div>

@code {
    private EventEmitter? emitter;
}

// EventEmitter.razor
<div class="event-emitter">
    <button @onclick="EmitEvent">Emit Custom Event</button>
    <input @bind="eventData" placeholder="Event data" />
</div>

@code {
    private string eventData = "";
    private readonly Dictionary<string, List<Func<CustomEventArgs, Task>>> handlers = new();
    
    public void Subscribe(string eventType, Func<CustomEventArgs, Task> handler)
    {
        if (!handlers.ContainsKey(eventType))
        {
            handlers[eventType] = new List<Func<CustomEventArgs, Task>>();
        }
        handlers[eventType].Add(handler);
    }
    
    public void Unsubscribe(string eventType, Func<CustomEventArgs, Task> handler)
    {
        if (handlers.TryGetValue(eventType, out var list))
        {
            list.Remove(handler);
        }
    }
    
    private async Task EmitEvent()
    {
        await Emit("custom-event", new CustomEventArgs
        {
            Type = "custom-event",
            Data = eventData,
            Source = this
        });
    }
    
    public async Task Emit(string eventType, CustomEventArgs args)
    {
        if (handlers.TryGetValue(eventType, out var list))
        {
            foreach (var handler in list.ToList())
            {
                await handler(args);
            }
        }
    }
    
    public class CustomEventArgs : EventArgs
    {
        public string Type { get; set; } = "";
        public object? Data { get; set; }
        public object? Source { get; set; }
        public DateTime Timestamp { get; set; } = DateTime.Now;
    }
}

// EventListener.razor
@implements IDisposable

<div class="event-listener">
    <h5>Event Listener</h5>
    <ul>
        @foreach (var evt in receivedEvents)
        {
            <li>@evt</li>
        }
    </ul>
</div>

@code {
    [Parameter] public EventEmitter? EventSource { get; set; }
    
    private List<string> receivedEvents = new();
    
    protected override void OnInitialized()
    {
        if (EventSource != null)
        {
            EventSource.Subscribe("custom-event", HandleCustomEvent);
        }
    }
    
    private async Task HandleCustomEvent(EventEmitter.CustomEventArgs args)
    {
        receivedEvents.Add($"{args.Timestamp:HH:mm:ss} - {args.Type}: {args.Data}");
        await InvokeAsync(StateHasChanged);
    }
    
    public void Dispose()
    {
        if (EventSource != null)
        {
            EventSource.Unsubscribe("custom-event", HandleCustomEvent);
        }
    }
}
```

### 5.2 글로벌 이벤트 버스

```csharp
// IEventBus.cs
public interface IEventBus
{
    void Subscribe<TEvent>(Action<TEvent> handler) where TEvent : IEvent;
    void Unsubscribe<TEvent>(Action<TEvent> handler) where TEvent : IEvent;
    Task PublishAsync<TEvent>(TEvent eventData) where TEvent : IEvent;
}

// EventBus.cs
public class EventBus : IEventBus
{
    private readonly Dictionary<Type, List<Delegate>> _handlers = new();
    private readonly SemaphoreSlim _semaphore = new(1, 1);
    
    public void Subscribe<TEvent>(Action<TEvent> handler) where TEvent : IEvent
    {
        var eventType = typeof(TEvent);
        
        _semaphore.Wait();
        try
        {
            if (!_handlers.ContainsKey(eventType))
            {
                _handlers[eventType] = new List<Delegate>();
            }
            _handlers[eventType].Add(handler);
        }
        finally
        {
            _semaphore.Release();
        }
    }
    
    public void Unsubscribe<TEvent>(Action<TEvent> handler) where TEvent : IEvent
    {
        var eventType = typeof(TEvent);
        
        _semaphore.Wait();
        try
        {
            if (_handlers.TryGetValue(eventType, out var list))
            {
                list.Remove(handler);
            }
        }
        finally
        {
            _semaphore.Release();
        }
    }
    
    public async Task PublishAsync<TEvent>(TEvent eventData) where TEvent : IEvent
    {
        var eventType = typeof(TEvent);
        
        await _semaphore.WaitAsync();
        List<Delegate> handlers;
        try
        {
            if (!_handlers.TryGetValue(eventType, out var list))
            {
                return;
            }
            handlers = list.ToList();
        }
        finally
        {
            _semaphore.Release();
        }
        
        foreach (var handler in handlers)
        {
            if (handler is Action<TEvent> action)
            {
                await Task.Run(() => action(eventData));
            }
        }
    }
}

// IEvent.cs
public interface IEvent
{
    Guid Id { get; }
    DateTime Timestamp { get; }
}

// UserLoggedInEvent.cs
public class UserLoggedInEvent : IEvent
{
    public Guid Id { get; } = Guid.NewGuid();
    public DateTime Timestamp { get; } = DateTime.Now;
    public string UserId { get; set; } = "";
    public string UserName { get; set; } = "";
}

// EventBusComponent.razor
@inject IEventBus EventBus
@implements IDisposable

<div class="event-bus-component">
    <button @onclick="PublishUserLogin">Publish User Login</button>
    
    <h5>Event Log:</h5>
    <ul>
        @foreach (var log in eventLog)
        {
            <li>@log</li>
        }
    </ul>
</div>

@code {
    private List<string> eventLog = new();
    
    protected override void OnInitialized()
    {
        EventBus.Subscribe<UserLoggedInEvent>(HandleUserLogin);
    }
    
    private async Task PublishUserLogin()
    {
        var loginEvent = new UserLoggedInEvent
        {
            UserId = Guid.NewGuid().ToString(),
            UserName = "TestUser"
        };
        
        await EventBus.PublishAsync(loginEvent);
    }
    
    private void HandleUserLogin(UserLoggedInEvent e)
    {
        InvokeAsync(() =>
        {
            eventLog.Add($"User logged in: {e.UserName} at {e.Timestamp:HH:mm:ss}");
            StateHasChanged();
        });
    }
    
    public void Dispose()
    {
        EventBus.Unsubscribe<UserLoggedInEvent>(HandleUserLogin);
    }
}
```

## 6. 실전 예제

### 6.1 드래그 앤 드롭

```csharp
// DragAndDrop.razor
@page "/events/drag-drop"

<h3>Drag and Drop Example</h3>

<div class="drag-drop-container">
    <div class="drag-source">
        <h4>Draggable Items</h4>
        @foreach (var item in sourceItems)
        {
            <div class="draggable-item"
                 draggable="true"
                 @ondragstart="@(() => HandleDragStart(item))"
                 @ondragend="HandleDragEnd">
                @item.Name
            </div>
        }
    </div>
    
    <div class="drop-target"
         @ondragover="HandleDragOver"
         @ondragover:preventDefault="true"
         @ondrop="HandleDrop"
         @ondrop:preventDefault="true"
         @ondragenter="HandleDragEnter"
         @ondragleave="HandleDragLeave">
        <h4>Drop Zone</h4>
        @if (isDropping)
        {
            <div class="drop-indicator">Drop here!</div>
        }
        @foreach (var item in targetItems)
        {
            <div class="dropped-item">
                @item.Name
                <button @onclick="@(() => RemoveItem(item))">×</button>
            </div>
        }
    </div>
</div>

@code {
    private List<DraggableItem> sourceItems = new();
    private List<DraggableItem> targetItems = new();
    private DraggableItem? draggedItem;
    private bool isDropping = false;
    
    protected override void OnInitialized()
    {
        sourceItems = Enumerable.Range(1, 5)
            .Select(i => new DraggableItem { Id = i, Name = $"Item {i}" })
            .ToList();
    }
    
    private void HandleDragStart(DraggableItem item)
    {
        draggedItem = item;
        Console.WriteLine($"Started dragging: {item.Name}");
    }
    
    private void HandleDragEnd(DragEventArgs e)
    {
        draggedItem = null;
        Console.WriteLine("Drag ended");
    }
    
    private void HandleDragOver(DragEventArgs e)
    {
        // preventDefault is handled by the attribute
    }
    
    private void HandleDragEnter(DragEventArgs e)
    {
        isDropping = true;
    }
    
    private void HandleDragLeave(DragEventArgs e)
    {
        isDropping = false;
    }
    
    private void HandleDrop(DragEventArgs e)
    {
        isDropping = false;
        
        if (draggedItem != null && !targetItems.Contains(draggedItem))
        {
            targetItems.Add(draggedItem);
            sourceItems.Remove(draggedItem);
            Console.WriteLine($"Dropped: {draggedItem.Name}");
        }
        
        draggedItem = null;
    }
    
    private void RemoveItem(DraggableItem item)
    {
        targetItems.Remove(item);
        sourceItems.Add(item);
        sourceItems = sourceItems.OrderBy(i => i.Id).ToList();
    }
    
    private class DraggableItem
    {
        public int Id { get; set; }
        public string Name { get; set; } = "";
    }
}
```

### 6.2 고급 입력 처리

```csharp
// AdvancedInput.razor
@page "/events/advanced-input"

<h3>Advanced Input Handling</h3>

<div class="advanced-input">
    <!-- 단축키 처리 -->
    <div class="shortcut-area" 
         tabindex="0"
         @onkeydown="HandleKeyDown"
         @onkeydown:preventDefault="@preventDefaultKey">
        <p>Press shortcuts: Ctrl+S, Ctrl+Z, Ctrl+Y</p>
        <p>Last action: @lastAction</p>
    </div>
    
    <!-- 텍스트 입력 제한 -->
    <input type="text" 
           @bind="limitedText"
           @oninput="HandleLimitedInput"
           maxlength="10"
           placeholder="Max 10 chars, numbers only" />
    
    <!-- 자동 완성 -->
    <div class="autocomplete">
        <input type="text"
               @bind="searchText"
               @oninput="HandleSearchInput"
               @onfocus="ShowSuggestions"
               @onblur="HideSuggestions"
               placeholder="Type to search..." />
        @if (showSuggestions && filteredSuggestions.Any())
        {
            <div class="suggestions">
                @foreach (var suggestion in filteredSuggestions)
                {
                    <div class="suggestion-item"
                         @onmousedown="@(() => SelectSuggestion(suggestion))"
                         @onmousedown:preventDefault="true">
                        @suggestion
                    </div>
                }
            </div>
        }
    </div>
</div>

@code {
    private string lastAction = "None";
    private string limitedText = "";
    private string searchText = "";
    private bool showSuggestions = false;
    private bool preventDefaultKey = false;
    
    private readonly List<string> suggestions = new()
    {
        "Apple", "Banana", "Cherry", "Date", "Elderberry",
        "Fig", "Grape", "Honeydew", "Kiwi", "Lemon"
    };
    
    private List<string> filteredSuggestions = new();
    
    private void HandleKeyDown(KeyboardEventArgs e)
    {
        preventDefaultKey = false;
        
        // Ctrl/Cmd + 키 조합
        if (e.CtrlKey || e.MetaKey)
        {
            switch (e.Key.ToUpper())
            {
                case "S":
                    lastAction = "Save (Ctrl+S)";
                    preventDefaultKey = true;
                    break;
                case "Z":
                    lastAction = "Undo (Ctrl+Z)";
                    preventDefaultKey = true;
                    break;
                case "Y":
                    lastAction = "Redo (Ctrl+Y)";
                    preventDefaultKey = true;
                    break;
            }
        }
        
        // Function 키
        if (e.Key.StartsWith("F") && e.Key.Length <= 3)
        {
            lastAction = $"Function key: {e.Key}";
            preventDefaultKey = true;
        }
    }
    
    private void HandleLimitedInput(ChangeEventArgs e)
    {
        var value = e.Value?.ToString() ?? "";
        // 숫자만 허용
        limitedText = new string(value.Where(char.IsDigit).ToArray());
    }
    
    private void HandleSearchInput(ChangeEventArgs e)
    {
        searchText = e.Value?.ToString() ?? "";
        FilterSuggestions();
    }
    
    private void FilterSuggestions()
    {
        if (string.IsNullOrWhiteSpace(searchText))
        {
            filteredSuggestions.Clear();
        }
        else
        {
            filteredSuggestions = suggestions
                .Where(s => s.StartsWith(searchText, StringComparison.OrdinalIgnoreCase))
                .Take(5)
                .ToList();
        }
    }
    
    private void ShowSuggestions()
    {
        showSuggestions = true;
        FilterSuggestions();
    }
    
    private async Task HideSuggestions()
    {
        // 딜레이를 주어 클릭 이벤트가 먼저 처리되도록 함
        await Task.Delay(200);
        showSuggestions = false;
    }
    
    private void SelectSuggestion(string suggestion)
    {
        searchText = suggestion;
        showSuggestions = false;
    }
}
```

## 마무리

Blazor의 이벤트 처리 시스템은 강력하고 유연합니다. EventCallback을 통한 컴포넌트 간 통신, 이벤트 버블링과 캡처링 제어, 비동기 이벤트 처리, 그리고 커스텀 이벤트 시스템 구축 등 다양한 패턴을 활용할 수 있습니다.

특히 EventCallback은 자동 StateHasChanged 호출, 비동기 지원, 타입 안전성 등의 장점을 제공하여 Blazor 애플리케이션에서 이벤트를 다루는 표준 방법입니다. 복잡한 상호작용이 필요한 애플리케이션에서는 이벤트 버스나 커스텀 이벤트 시스템을 구현하여 확장성을 높일 수 있습니다.