# JavaScript Interop 심화

## 개요

Blazor와 JavaScript 간의 상호 운용성(Interop)은 기존 JavaScript 라이브러리 활용과 브라우저 API 접근을 가능하게 합니다. 이 장에서는 IJSRuntime, JSObjectReference, 그리고 메모리 관리 등 고급 JavaScript Interop 기법을 학습합니다.

## 1. IJSRuntime 고급 사용법

### 1.1 JavaScript 함수 호출

```csharp
// wwwroot/js/interop.js
window.blazorInterop = {
    showAlert: function (message) {
        alert(message);
        return true;
    },
    
    getWindowSize: function () {
        return {
            width: window.innerWidth,
            height: window.innerHeight
        };
    },
    
    setLocalStorage: function (key, value) {
        localStorage.setItem(key, JSON.stringify(value));
    },
    
    getLocalStorage: function (key) {
        const value = localStorage.getItem(key);
        return value ? JSON.parse(value) : null;
    },
    
    createChart: function (elementId, data) {
        const canvas = document.getElementById(elementId);
        if (!canvas) return null;
        
        // Chart.js 예제
        return new Chart(canvas, {
            type: 'line',
            data: data,
            options: {
                responsive: true,
                maintainAspectRatio: false
            }
        });
    }
};

// InteropExample.razor
@page "/interop/basic"
@inject IJSRuntime JS

<h3>JavaScript Interop Examples</h3>

<div class="interop-demo">
    <button @onclick="ShowAlert">Show Alert</button>
    <button @onclick="GetWindowSize">Get Window Size</button>
    <button @onclick="SaveToLocalStorage">Save Data</button>
    <button @onclick="LoadFromLocalStorage">Load Data</button>
</div>

<div class="results">
    <p>Window Size: @windowSize</p>
    <p>Loaded Data: @loadedData</p>
</div>

<canvas id="myChart" width="400" height="200"></canvas>

@code {
    private string windowSize = "";
    private string loadedData = "";
    
    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            var chartData = new
            {
                labels = new[] { "Jan", "Feb", "Mar", "Apr", "May" },
                datasets = new[]
                {
                    new
                    {
                        label = "Sales",
                        data = new[] { 12, 19, 3, 5, 2 },
                        borderColor = "rgb(75, 192, 192)",
                        tension = 0.1
                    }
                }
            };
            
            await JS.InvokeVoidAsync("blazorInterop.createChart", "myChart", chartData);
        }
    }
    
    private async Task ShowAlert()
    {
        var result = await JS.InvokeAsync<bool>("blazorInterop.showAlert", "Hello from Blazor!");
        Console.WriteLine($"Alert shown: {result}");
    }
    
    private async Task GetWindowSize()
    {
        var size = await JS.InvokeAsync<WindowSize>("blazorInterop.getWindowSize");
        windowSize = $"{size.Width} x {size.Height}";
    }
    
    private async Task SaveToLocalStorage()
    {
        var data = new
        {
            Name = "Blazor User",
            SavedAt = DateTime.Now,
            Settings = new { Theme = "dark", Language = "en" }
        };
        
        await JS.InvokeVoidAsync("blazorInterop.setLocalStorage", "userData", data);
    }
    
    private async Task LoadFromLocalStorage()
    {
        var data = await JS.InvokeAsync<JsonElement?>("blazorInterop.getLocalStorage", "userData");
        loadedData = data?.ToString() ?? "No data found";
    }
    
    private class WindowSize
    {
        public int Width { get; set; }
        public int Height { get; set; }
    }
}
```

### 1.2 비동기 JavaScript 처리

```csharp
// wwwroot/js/async-interop.js
window.asyncInterop = {
    fetchDataWithDelay: function (delay) {
        return new Promise((resolve) => {
            setTimeout(() => {
                resolve({
                    data: "Async data loaded",
                    timestamp: new Date().toISOString()
                });
            }, delay);
        });
    },
    
    streamData: async function* (count, delay) {
        for (let i = 0; i < count; i++) {
            await new Promise(resolve => setTimeout(resolve, delay));
            yield { index: i, value: Math.random() * 100 };
        }
    },
    
    longRunningTask: function (dotNetHelper) {
        let progress = 0;
        const interval = setInterval(() => {
            progress += 10;
            dotNetHelper.invokeMethodAsync('UpdateProgress', progress);
            
            if (progress >= 100) {
                clearInterval(interval);
                dotNetHelper.invokeMethodAsync('TaskCompleted');
            }
        }, 500);
        
        return interval;
    }
};

// AsyncInterop.razor
@page "/interop/async"
@implements IAsyncDisposable

<h3>Async JavaScript Interop</h3>

<div class="async-demo">
    <button @onclick="FetchAsyncData" disabled="@isLoading">
        @if (isLoading)
        {
            <span>Loading...</span>
        }
        else
        {
            <span>Fetch Async Data</span>
        }
    </button>
    
    <button @onclick="StartLongRunningTask" disabled="@isTaskRunning">
        Start Long Task
    </button>
    
    <button @onclick="CancelTask" disabled="@(!isTaskRunning)">
        Cancel Task
    </button>
</div>

<div class="progress" style="display: @(progress > 0 ? "block" : "none")">
    <div class="progress-bar" style="width: @progress%">@progress%</div>
</div>

<div class="results">
    @if (asyncData != null)
    {
        <p>Data: @asyncData.Data</p>
        <p>Timestamp: @asyncData.Timestamp</p>
    }
</div>

@code {
    private bool isLoading = false;
    private bool isTaskRunning = false;
    private int progress = 0;
    private AsyncData? asyncData;
    private DotNetObjectReference<AsyncInterop>? dotNetRef;
    private IJSObjectReference? taskHandle;
    
    protected override void OnInitialized()
    {
        dotNetRef = DotNetObjectReference.Create(this);
    }
    
    private async Task FetchAsyncData()
    {
        isLoading = true;
        
        try
        {
            asyncData = await JS.InvokeAsync<AsyncData>(
                "asyncInterop.fetchDataWithDelay", 2000);
        }
        finally
        {
            isLoading = false;
        }
    }
    
    private async Task StartLongRunningTask()
    {
        isTaskRunning = true;
        progress = 0;
        
        taskHandle = await JS.InvokeAsync<IJSObjectReference>(
            "asyncInterop.longRunningTask", dotNetRef);
    }
    
    [JSInvokable]
    public void UpdateProgress(int newProgress)
    {
        progress = newProgress;
        InvokeAsync(StateHasChanged);
    }
    
    [JSInvokable]
    public void TaskCompleted()
    {
        isTaskRunning = false;
        InvokeAsync(StateHasChanged);
    }
    
    private async Task CancelTask()
    {
        if (taskHandle != null)
        {
            await JS.InvokeVoidAsync("clearInterval", taskHandle);
            isTaskRunning = false;
            progress = 0;
        }
    }
    
    public async ValueTask DisposeAsync()
    {
        if (taskHandle != null)
        {
            await JS.InvokeVoidAsync("clearInterval", taskHandle);
        }
        
        dotNetRef?.Dispose();
    }
    
    private class AsyncData
    {
        public string Data { get; set; } = "";
        public string Timestamp { get; set; } = "";
    }
}
```

## 2. JSObjectReference 관리

### 2.1 JavaScript 객체 생명주기

```csharp
// wwwroot/js/object-reference.js
export class DataManager {
    constructor(initialData) {
        this.data = initialData || [];
        this.listeners = [];
    }
    
    addItem(item) {
        this.data.push(item);
        this.notifyListeners('add', item);
    }
    
    removeItem(index) {
        const removed = this.data.splice(index, 1)[0];
        this.notifyListeners('remove', removed);
    }
    
    getData() {
        return this.data;
    }
    
    subscribe(dotNetHelper) {
        this.listeners.push(dotNetHelper);
        return {
            unsubscribe: () => {
                const index = this.listeners.indexOf(dotNetHelper);
                if (index > -1) {
                    this.listeners.splice(index, 1);
                }
            }
        };
    }
    
    notifyListeners(action, data) {
        this.listeners.forEach(listener => {
            listener.invokeMethodAsync('OnDataChanged', action, data);
        });
    }
    
    dispose() {
        this.listeners = [];
        this.data = [];
    }
}

// ObjectReferenceExample.razor
@page "/interop/object-reference"
@implements IAsyncDisposable

<h3>JSObjectReference Example</h3>

<div class="data-manager">
    <input @bind="newItem" placeholder="New item" />
    <button @onclick="AddItem">Add</button>
    
    <ul>
        @foreach (var (item, index) in items.Select((item, i) => (item, i)))
        {
            <li>
                @item
                <button @onclick="@(() => RemoveItem(index))">Remove</button>
            </li>
        }
    </ul>
</div>

@code {
    private IJSObjectReference? module;
    private IJSObjectReference? dataManager;
    private IJSObjectReference? subscription;
    private DotNetObjectReference<ObjectReferenceExample>? dotNetRef;
    private List<string> items = new();
    private string newItem = "";
    
    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            module = await JS.InvokeAsync<IJSObjectReference>(
                "import", "./js/object-reference.js");
            
            dataManager = await module.InvokeAsync<IJSObjectReference>(
                "new DataManager", new[] { "Initial Item 1", "Initial Item 2" });
            
            dotNetRef = DotNetObjectReference.Create(this);
            subscription = await dataManager.InvokeAsync<IJSObjectReference>(
                "subscribe", dotNetRef);
            
            await RefreshItems();
        }
    }
    
    private async Task AddItem()
    {
        if (!string.IsNullOrWhiteSpace(newItem) && dataManager != null)
        {
            await dataManager.InvokeVoidAsync("addItem", newItem);
            newItem = "";
            await RefreshItems();
        }
    }
    
    private async Task RemoveItem(int index)
    {
        if (dataManager != null)
        {
            await dataManager.InvokeVoidAsync("removeItem", index);
            await RefreshItems();
        }
    }
    
    private async Task RefreshItems()
    {
        if (dataManager != null)
        {
            items = await dataManager.InvokeAsync<List<string>>("getData");
        }
    }
    
    [JSInvokable]
    public void OnDataChanged(string action, string data)
    {
        Console.WriteLine($"Data changed: {action} - {data}");
        InvokeAsync(RefreshItems);
    }
    
    public async ValueTask DisposeAsync()
    {
        if (subscription != null)
        {
            await subscription.InvokeVoidAsync("unsubscribe");
            await subscription.DisposeAsync();
        }
        
        if (dataManager != null)
        {
            await dataManager.InvokeVoidAsync("dispose");
            await dataManager.DisposeAsync();
        }
        
        if (module != null)
        {
            await module.DisposeAsync();
        }
        
        dotNetRef?.Dispose();
    }
}
```

### 2.2 복잡한 JavaScript 객체 래핑

```csharp
// wwwroot/js/media-player.js
export class MediaPlayer {
    constructor(elementId) {
        this.element = document.getElementById(elementId);
        this.video = document.createElement('video');
        this.element.appendChild(this.video);
        
        this.eventHandlers = new Map();
        this.setupEventListeners();
    }
    
    setupEventListeners() {
        ['play', 'pause', 'ended', 'timeupdate', 'loadedmetadata'].forEach(event => {
            this.video.addEventListener(event, (e) => {
                const handlers = this.eventHandlers.get(event) || [];
                handlers.forEach(handler => {
                    handler.invokeMethodAsync('HandleEvent', {
                        type: event,
                        currentTime: this.video.currentTime,
                        duration: this.video.duration,
                        paused: this.video.paused
                    });
                });
            });
        });
    }
    
    loadVideo(url) {
        this.video.src = url;
        this.video.load();
    }
    
    play() {
        return this.video.play();
    }
    
    pause() {
        this.video.pause();
    }
    
    seek(time) {
        this.video.currentTime = time;
    }
    
    setVolume(volume) {
        this.video.volume = Math.max(0, Math.min(1, volume));
    }
    
    addEventListener(event, dotNetHelper) {
        if (!this.eventHandlers.has(event)) {
            this.eventHandlers.set(event, []);
        }
        this.eventHandlers.get(event).push(dotNetHelper);
    }
    
    removeEventListener(event, dotNetHelper) {
        const handlers = this.eventHandlers.get(event) || [];
        const index = handlers.indexOf(dotNetHelper);
        if (index > -1) {
            handlers.splice(index, 1);
        }
    }
    
    getState() {
        return {
            currentTime: this.video.currentTime,
            duration: this.video.duration,
            paused: this.video.paused,
            volume: this.video.volume,
            readyState: this.video.readyState
        };
    }
    
    dispose() {
        this.video.remove();
        this.eventHandlers.clear();
    }
}

// MediaPlayerComponent.razor
@page "/interop/media-player"
@implements IAsyncDisposable

<h3>Media Player</h3>

<div class="media-player">
    <div id="videoContainer" style="width: 640px; height: 360px; background: black;"></div>
    
    <div class="controls">
        <button @onclick="Play" disabled="@(!isReady)">Play</button>
        <button @onclick="Pause" disabled="@(!isReady)">Pause</button>
        <input type="range" min="0" max="@duration" value="@currentTime" 
               @oninput="@(e => Seek(double.Parse(e.Value?.ToString() ?? "0")))" />
        <span>@TimeSpan.FromSeconds(currentTime).ToString(@"mm\:ss") / 
              @TimeSpan.FromSeconds(duration).ToString(@"mm\:ss")</span>
    </div>
    
    <div class="volume">
        <label>Volume:</label>
        <input type="range" min="0" max="100" value="@(volume * 100)" 
               @oninput="@(e => SetVolume(double.Parse(e.Value?.ToString() ?? "0") / 100))" />
    </div>
</div>

@code {
    private IJSObjectReference? module;
    private IJSObjectReference? player;
    private DotNetObjectReference<MediaPlayerComponent>? dotNetRef;
    
    private bool isReady = false;
    private double currentTime = 0;
    private double duration = 0;
    private double volume = 1;
    
    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            module = await JS.InvokeAsync<IJSObjectReference>(
                "import", "./js/media-player.js");
            
            player = await module.InvokeAsync<IJSObjectReference>(
                "new MediaPlayer", "videoContainer");
            
            dotNetRef = DotNetObjectReference.Create(this);
            
            await player.InvokeVoidAsync("addEventListener", "loadedmetadata", dotNetRef);
            await player.InvokeVoidAsync("addEventListener", "timeupdate", dotNetRef);
            
            await player.InvokeVoidAsync("loadVideo", 
                "https://www.w3schools.com/html/mov_bbb.mp4");
        }
    }
    
    [JSInvokable]
    public void HandleEvent(MediaEventData eventData)
    {
        switch (eventData.Type)
        {
            case "loadedmetadata":
                isReady = true;
                duration = eventData.Duration;
                break;
            case "timeupdate":
                currentTime = eventData.CurrentTime;
                break;
        }
        
        InvokeAsync(StateHasChanged);
    }
    
    private async Task Play()
    {
        if (player != null)
        {
            await player.InvokeVoidAsync("play");
        }
    }
    
    private async Task Pause()
    {
        if (player != null)
        {
            await player.InvokeVoidAsync("pause");
        }
    }
    
    private async Task Seek(double time)
    {
        if (player != null)
        {
            await player.InvokeVoidAsync("seek", time);
        }
    }
    
    private async Task SetVolume(double vol)
    {
        if (player != null)
        {
            volume = vol;
            await player.InvokeVoidAsync("setVolume", volume);
        }
    }
    
    public async ValueTask DisposeAsync()
    {
        if (player != null)
        {
            await player.InvokeVoidAsync("dispose");
            await player.DisposeAsync();
        }
        
        if (module != null)
        {
            await module.DisposeAsync();
        }
        
        dotNetRef?.Dispose();
    }
    
    private class MediaEventData
    {
        public string Type { get; set; } = "";
        public double CurrentTime { get; set; }
        public double Duration { get; set; }
        public bool Paused { get; set; }
    }
}
```

## 3. 메모리 관리

### 3.1 메모리 누수 방지

```csharp
// MemoryManagement.razor
@page "/interop/memory"
@implements IAsyncDisposable

<h3>Memory Management</h3>

<div class="memory-demo">
    <button @onclick="CreateObjects">Create JS Objects</button>
    <button @onclick="DisposeObjects">Dispose Objects</button>
    <button @onclick="RunGarbageCollection">Run GC</button>
    
    <p>Created Objects: @createdObjects.Count</p>
    <p>Active References: @activeReferences</p>
</div>

@code {
    private List<IJSObjectReference> createdObjects = new();
    private List<DotNetObjectReference<CallbackHandler>> dotNetRefs = new();
    private int activeReferences = 0;
    
    private async Task CreateObjects()
    {
        for (int i = 0; i < 10; i++)
        {
            // Create JS object
            var jsObj = await JS.InvokeAsync<IJSObjectReference>(
                "eval", "(function() { return { id: " + i + ", data: new Array(1000).fill(0) }; })()");
            
            createdObjects.Add(jsObj);
            
            // Create .NET reference
            var handler = new CallbackHandler(i);
            var dotNetRef = DotNetObjectReference.Create(handler);
            dotNetRefs.Add(dotNetRef);
            
            // Pass reference to JS
            await JS.InvokeVoidAsync("eval", 
                $"window.handler{i} = arguments[0]", dotNetRef);
        }
        
        activeReferences = createdObjects.Count;
    }
    
    private async Task DisposeObjects()
    {
        // Dispose JS object references
        foreach (var obj in createdObjects)
        {
            await obj.DisposeAsync();
        }
        createdObjects.Clear();
        
        // Dispose .NET references
        foreach (var dotNetRef in dotNetRefs)
        {
            dotNetRef.Dispose();
        }
        dotNetRefs.Clear();
        
        // Clear JS references
        for (int i = 0; i < 10; i++)
        {
            await JS.InvokeVoidAsync("eval", $"delete window.handler{i}");
        }
        
        activeReferences = 0;
    }
    
    private void RunGarbageCollection()
    {
        GC.Collect();
        GC.WaitForPendingFinalizers();
        GC.Collect();
    }
    
    public async ValueTask DisposeAsync()
    {
        await DisposeObjects();
    }
    
    private class CallbackHandler
    {
        private readonly int id;
        
        public CallbackHandler(int id)
        {
            this.id = id;
        }
        
        [JSInvokable]
        public void HandleCallback(string message)
        {
            Console.WriteLine($"Handler {id}: {message}");
        }
    }
}
```

### 3.2 대용량 데이터 전송 최적화

```csharp
// LargeDataTransfer.razor
@page "/interop/large-data"

<h3>Large Data Transfer</h3>

<div class="transfer-demo">
    <button @onclick="TransferLargeArray">Transfer Large Array</button>
    <button @onclick="TransferChunked">Transfer Chunked</button>
    <button @onclick="TransferStreaming">Transfer Streaming</button>
</div>

<div class="stats">
    <p>Transfer Time: @transferTime ms</p>
    <p>Data Size: @dataSize bytes</p>
</div>

@code {
    private long transferTime = 0;
    private long dataSize = 0;
    
    private async Task TransferLargeArray()
    {
        var stopwatch = Stopwatch.StartNew();
        
        // Create large array
        var largeArray = Enumerable.Range(0, 100000)
            .Select(i => new { Id = i, Value = $"Item {i}" })
            .ToArray();
        
        dataSize = EstimateSize(largeArray);
        
        // Transfer to JS
        await JS.InvokeVoidAsync("eval", 
            "window.largeData = arguments[0]", largeArray);
        
        stopwatch.Stop();
        transferTime = stopwatch.ElapsedMilliseconds;
    }
    
    private async Task TransferChunked()
    {
        var stopwatch = Stopwatch.StartNew();
        
        // Initialize JS array
        await JS.InvokeVoidAsync("eval", "window.chunkedData = []");
        
        const int chunkSize = 1000;
        var totalItems = 100000;
        
        for (int i = 0; i < totalItems; i += chunkSize)
        {
            var chunk = Enumerable.Range(i, Math.Min(chunkSize, totalItems - i))
                .Select(j => new { Id = j, Value = $"Item {j}" })
                .ToArray();
            
            await JS.InvokeVoidAsync("eval", 
                "window.chunkedData.push(...arguments[0])", chunk);
            
            // Allow UI to update
            if (i % 10000 == 0)
            {
                StateHasChanged();
                await Task.Yield();
            }
        }
        
        stopwatch.Stop();
        transferTime = stopwatch.ElapsedMilliseconds;
        dataSize = totalItems * 50; // Approximate
    }
    
    private async Task TransferStreaming()
    {
        var stopwatch = Stopwatch.StartNew();
        
        // Create stream reference
        using var stream = new MemoryStream();
        using var writer = new StreamWriter(stream);
        
        // Write data to stream
        for (int i = 0; i < 100000; i++)
        {
            await writer.WriteLineAsync($"{i},Item {i}");
        }
        await writer.FlushAsync();
        
        stream.Position = 0;
        dataSize = stream.Length;
        
        // Transfer stream to JS
        using var streamRef = new DotNetStreamReference(stream);
        await JS.InvokeVoidAsync("processStream", streamRef);
        
        stopwatch.Stop();
        transferTime = stopwatch.ElapsedMilliseconds;
    }
    
    private long EstimateSize(object obj)
    {
        var json = JsonSerializer.Serialize(obj);
        return Encoding.UTF8.GetByteCount(json);
    }
}

// wwwroot/js/stream-processing.js
window.processStream = async function(streamReference) {
    const response = await fetch(streamReference);
    const text = await response.text();
    
    // Process streamed data
    const lines = text.split('\n');
    const data = lines.map(line => {
        const [id, value] = line.split(',');
        return { id: parseInt(id), value };
    });
    
    window.streamedData = data;
    console.log(`Processed ${data.length} items from stream`);
};
```

## 마무리

JavaScript Interop은 Blazor 애플리케이션에서 브라우저의 전체 기능을 활용할 수 있게 해줍니다. IJSRuntime과 JSObjectReference를 통해 JavaScript 코드와 객체를 효과적으로 관리하고, 적절한 메모리 관리를 통해 성능을 최적화할 수 있습니다. 대용량 데이터 전송 시에는 청크 분할이나 스트리밍을 활용하여 효율성을 높일 수 있습니다.