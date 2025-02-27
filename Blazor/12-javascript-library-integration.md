# JavaScript 라이브러리 통합

## 개요

Blazor에서 기존 JavaScript 라이브러리를 통합하면 풍부한 생태계를 활용할 수 있습니다. 이 장에서는 Chart.js, Monaco Editor 등 인기 있는 JavaScript 라이브러리를 Blazor에 통합하는 패턴을 학습합니다.

## 1. Chart.js 통합

### 1.1 Chart.js 래퍼 구현

```csharp
// wwwroot/js/chart-wrapper.js
export class ChartWrapper {
    constructor() {
        this.charts = new Map();
    }
    
    createChart(elementId, config) {
        const canvas = document.getElementById(elementId);
        if (!canvas) return null;
        
        // 기존 차트가 있으면 제거
        if (this.charts.has(elementId)) {
            this.charts.get(elementId).destroy();
        }
        
        const chart = new Chart(canvas, config);
        this.charts.set(elementId, chart);
        return elementId;
    }
    
    updateChart(elementId, data) {
        const chart = this.charts.get(elementId);
        if (!chart) return;
        
        chart.data = data;
        chart.update();
    }
    
    updateOptions(elementId, options) {
        const chart = this.charts.get(elementId);
        if (!chart) return;
        
        chart.options = { ...chart.options, ...options };
        chart.update();
    }
    
    destroyChart(elementId) {
        const chart = this.charts.get(elementId);
        if (chart) {
            chart.destroy();
            this.charts.delete(elementId);
        }
    }
    
    dispose() {
        this.charts.forEach(chart => chart.destroy());
        this.charts.clear();
    }
}

// ChartComponent.razor
@implements IAsyncDisposable
@inject IJSRuntime JS

<div class="chart-container">
    <canvas id="@chartId"></canvas>
</div>

@code {
    [Parameter] public ChartType Type { get; set; } = ChartType.Line;
    [Parameter] public ChartData Data { get; set; } = new();
    [Parameter] public ChartOptions Options { get; set; } = new();
    
    private string chartId = $"chart-{Guid.NewGuid():N}";
    private IJSObjectReference? module;
    private IJSObjectReference? chartWrapper;
    
    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            module = await JS.InvokeAsync<IJSObjectReference>(
                "import", "./js/chart-wrapper.js");
            
            chartWrapper = await module.InvokeAsync<IJSObjectReference>(
                "new ChartWrapper");
            
            await CreateChart();
        }
    }
    
    protected override async Task OnParametersSetAsync()
    {
        if (chartWrapper != null)
        {
            await UpdateChart();
        }
    }
    
    private async Task CreateChart()
    {
        if (chartWrapper == null) return;
        
        var config = new
        {
            type = Type.ToString().ToLower(),
            data = Data,
            options = Options
        };
        
        await chartWrapper.InvokeVoidAsync("createChart", chartId, config);
    }
    
    private async Task UpdateChart()
    {
        if (chartWrapper == null) return;
        
        await chartWrapper.InvokeVoidAsync("updateChart", chartId, Data);
        await chartWrapper.InvokeVoidAsync("updateOptions", chartId, Options);
    }
    
    public async ValueTask DisposeAsync()
    {
        if (chartWrapper != null)
        {
            await chartWrapper.InvokeVoidAsync("destroyChart", chartId);
            await chartWrapper.DisposeAsync();
        }
        
        if (module != null)
        {
            await module.DisposeAsync();
        }
    }
    
    public enum ChartType
    {
        Line, Bar, Pie, Doughnut, Radar, PolarArea, Bubble, Scatter
    }
    
    public class ChartData
    {
        public List<string> Labels { get; set; } = new();
        public List<ChartDataset> Datasets { get; set; } = new();
    }
    
    public class ChartDataset
    {
        public string Label { get; set; } = "";
        public List<double> Data { get; set; } = new();
        public string BackgroundColor { get; set; } = "";
        public string BorderColor { get; set; } = "";
        public int BorderWidth { get; set; } = 1;
    }
    
    public class ChartOptions
    {
        public bool Responsive { get; set; } = true;
        public bool MaintainAspectRatio { get; set; } = true;
        public ChartPlugins Plugins { get; set; } = new();
    }
    
    public class ChartPlugins
    {
        public ChartLegend Legend { get; set; } = new();
        public ChartTitle Title { get; set; } = new();
    }
    
    public class ChartLegend
    {
        public bool Display { get; set; } = true;
        public string Position { get; set; } = "top";
    }
    
    public class ChartTitle
    {
        public bool Display { get; set; } = false;
        public string Text { get; set; } = "";
    }
}
```

### 1.2 고급 차트 기능

```csharp
// AdvancedChart.razor
@page "/charts/advanced"

<h3>Advanced Chart Examples</h3>

<div class="chart-grid">
    <div>
        <h4>Real-time Chart</h4>
        <RealtimeChart @ref="realtimeChart" />
    </div>
    
    <div>
        <h4>Interactive Chart</h4>
        <InteractiveChart />
    </div>
</div>

@code {
    private RealtimeChart? realtimeChart;
}

// RealtimeChart.razor
@implements IAsyncDisposable

<ChartComponent Type="ChartComponent.ChartType.Line" 
                Data="chartData" 
                Options="chartOptions" />

@code {
    private ChartComponent.ChartData chartData = new();
    private ChartComponent.ChartOptions chartOptions = new();
    private Timer? updateTimer;
    private Random random = new();
    
    protected override void OnInitialized()
    {
        InitializeChart();
        StartRealtimeUpdates();
    }
    
    private void InitializeChart()
    {
        chartData = new ChartComponent.ChartData
        {
            Labels = Enumerable.Range(0, 20).Select(i => $"{i}s").ToList(),
            Datasets = new List<ChartComponent.ChartDataset>
            {
                new()
                {
                    Label = "CPU Usage",
                    Data = new List<double>(new double[20]),
                    BorderColor = "rgb(75, 192, 192)",
                    BackgroundColor = "rgba(75, 192, 192, 0.2)"
                },
                new()
                {
                    Label = "Memory Usage",
                    Data = new List<double>(new double[20]),
                    BorderColor = "rgb(255, 99, 132)",
                    BackgroundColor = "rgba(255, 99, 132, 0.2)"
                }
            }
        };
        
        chartOptions = new ChartComponent.ChartOptions
        {
            Responsive = true,
            MaintainAspectRatio = false,
            Plugins = new ChartComponent.ChartPlugins
            {
                Title = new ChartComponent.ChartTitle
                {
                    Display = true,
                    Text = "System Performance"
                }
            }
        };
    }
    
    private void StartRealtimeUpdates()
    {
        updateTimer = new Timer(_ =>
        {
            UpdateChartData();
            InvokeAsync(StateHasChanged);
        }, null, TimeSpan.Zero, TimeSpan.FromSeconds(1));
    }
    
    private void UpdateChartData()
    {
        // Shift data
        foreach (var dataset in chartData.Datasets)
        {
            dataset.Data.RemoveAt(0);
            dataset.Data.Add(random.Next(0, 100));
        }
        
        // Update labels
        chartData.Labels.RemoveAt(0);
        chartData.Labels.Add($"{chartData.Labels.Count}s");
    }
    
    public void Dispose()
    {
        updateTimer?.Dispose();
    }
    
    public async ValueTask DisposeAsync()
    {
        updateTimer?.Dispose();
    }
}
```

## 2. Monaco Editor 통합

### 2.1 Monaco Editor 래퍼

```csharp
// wwwroot/js/monaco-wrapper.js
export class MonacoWrapper {
    constructor() {
        this.editors = new Map();
    }
    
    async createEditor(containerId, options) {
        // Monaco Editor 로드 대기
        await this.waitForMonaco();
        
        const container = document.getElementById(containerId);
        if (!container) return null;
        
        const editor = monaco.editor.create(container, {
            value: options.value || '',
            language: options.language || 'javascript',
            theme: options.theme || 'vs-dark',
            automaticLayout: true,
            minimap: { enabled: options.minimap !== false },
            ...options
        });
        
        this.editors.set(containerId, editor);
        return containerId;
    }
    
    getValue(editorId) {
        const editor = this.editors.get(editorId);
        return editor ? editor.getValue() : null;
    }
    
    setValue(editorId, value) {
        const editor = this.editors.get(editorId);
        if (editor) {
            editor.setValue(value);
        }
    }
    
    setLanguage(editorId, language) {
        const editor = this.editors.get(editorId);
        if (editor) {
            monaco.editor.setModelLanguage(editor.getModel(), language);
        }
    }
    
    setTheme(theme) {
        monaco.editor.setTheme(theme);
    }
    
    addAction(editorId, action) {
        const editor = this.editors.get(editorId);
        if (editor) {
            editor.addAction({
                id: action.id,
                label: action.label,
                keybindings: action.keybindings,
                run: (ed) => {
                    action.callback.invokeMethodAsync('Execute', ed.getValue());
                }
            });
        }
    }
    
    dispose(editorId) {
        const editor = this.editors.get(editorId);
        if (editor) {
            editor.dispose();
            this.editors.delete(editorId);
        }
    }
    
    disposeAll() {
        this.editors.forEach(editor => editor.dispose());
        this.editors.clear();
    }
    
    async waitForMonaco() {
        while (typeof monaco === 'undefined') {
            await new Promise(resolve => setTimeout(resolve, 100));
        }
    }
}

// MonacoEditor.razor
@implements IAsyncDisposable
@inject IJSRuntime JS

<div class="monaco-editor-container">
    <div id="@editorId" style="height: @Height; width: 100%;"></div>
</div>

@code {
    [Parameter] public string Value { get; set; } = "";
    [Parameter] public EventCallback<string> ValueChanged { get; set; }
    [Parameter] public string Language { get; set; } = "javascript";
    [Parameter] public string Theme { get; set; } = "vs-dark";
    [Parameter] public string Height { get; set; } = "400px";
    [Parameter] public Dictionary<string, object> Options { get; set; } = new();
    
    private string editorId = $"monaco-{Guid.NewGuid():N}";
    private IJSObjectReference? module;
    private IJSObjectReference? monacoWrapper;
    private Timer? debounceTimer;
    
    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            module = await JS.InvokeAsync<IJSObjectReference>(
                "import", "./js/monaco-wrapper.js");
            
            monacoWrapper = await module.InvokeAsync<IJSObjectReference>(
                "new MonacoWrapper");
            
            var options = new Dictionary<string, object>(Options)
            {
                ["value"] = Value,
                ["language"] = Language,
                ["theme"] = Theme
            };
            
            await monacoWrapper.InvokeVoidAsync("createEditor", editorId, options);
            
            // 변경 감지 설정
            StartChangeDetection();
        }
    }
    
    private void StartChangeDetection()
    {
        debounceTimer = new Timer(async _ =>
        {
            if (monacoWrapper != null)
            {
                var currentValue = await monacoWrapper.InvokeAsync<string>("getValue", editorId);
                if (currentValue != Value)
                {
                    Value = currentValue;
                    await ValueChanged.InvokeAsync(Value);
                }
            }
        }, null, TimeSpan.Zero, TimeSpan.FromMilliseconds(500));
    }
    
    public async Task SetValueAsync(string value)
    {
        if (monacoWrapper != null)
        {
            await monacoWrapper.InvokeVoidAsync("setValue", editorId, value);
        }
    }
    
    public async Task SetLanguageAsync(string language)
    {
        if (monacoWrapper != null)
        {
            await monacoWrapper.InvokeVoidAsync("setLanguage", editorId, language);
        }
    }
    
    public async ValueTask DisposeAsync()
    {
        debounceTimer?.Dispose();
        
        if (monacoWrapper != null)
        {
            await monacoWrapper.InvokeVoidAsync("dispose", editorId);
            await monacoWrapper.DisposeAsync();
        }
        
        if (module != null)
        {
            await module.DisposeAsync();
        }
    }
}
```

### 2.2 코드 에디터 고급 기능

```csharp
// CodeEditor.razor
@page "/editor/code"

<h3>Code Editor</h3>

<div class="editor-toolbar">
    <select @bind="selectedLanguage" @bind:event="onchange">
        <option value="javascript">JavaScript</option>
        <option value="csharp">C#</option>
        <option value="html">HTML</option>
        <option value="css">CSS</option>
        <option value="json">JSON</option>
    </select>
    
    <select @bind="selectedTheme" @bind:event="onchange">
        <option value="vs">Light</option>
        <option value="vs-dark">Dark</option>
        <option value="hc-black">High Contrast</option>
    </select>
    
    <button @onclick="FormatCode">Format</button>
    <button @onclick="RunCode">Run</button>
</div>

<MonacoEditor @ref="editor"
              @bind-Value="code"
              Language="@selectedLanguage"
              Theme="@selectedTheme"
              Height="500px"
              Options="editorOptions" />

<div class="output">
    <h4>Output:</h4>
    <pre>@output</pre>
</div>

@code {
    private MonacoEditor? editor;
    private string code = @"// Write your code here
function hello() {
    console.log('Hello, Blazor!');
}

hello();";
    private string selectedLanguage = "javascript";
    private string selectedTheme = "vs-dark";
    private string output = "";
    
    private Dictionary<string, object> editorOptions = new()
    {
        ["fontSize"] = 14,
        ["automaticLayout"] = true,
        ["formatOnType"] = true,
        ["formatOnPaste"] = true,
        ["suggestOnTriggerCharacters"] = true
    };
    
    private async Task FormatCode()
    {
        // JavaScript interop to format
        await JS.InvokeVoidAsync("eval", @"
            const editors = monaco.editor.getEditors();
            if (editors.length > 0) {
                editors[0].getAction('editor.action.formatDocument').run();
            }
        ");
    }
    
    private async Task RunCode()
    {
        output = "";
        
        if (selectedLanguage == "javascript")
        {
            try
            {
                // Create console capture
                await JS.InvokeVoidAsync("eval", @"
                    window.consoleOutput = [];
                    const originalLog = console.log;
                    console.log = function(...args) {
                        window.consoleOutput.push(args.join(' '));
                        originalLog.apply(console, args);
                    };
                ");
                
                // Run code
                await JS.InvokeVoidAsync("eval", code);
                
                // Get output
                var consoleOutput = await JS.InvokeAsync<string[]>("eval", 
                    "window.consoleOutput");
                output = string.Join("\n", consoleOutput);
                
                // Restore console
                await JS.InvokeVoidAsync("eval", 
                    "console.log = originalLog; delete window.consoleOutput;");
            }
            catch (Exception ex)
            {
                output = $"Error: {ex.Message}";
            }
        }
        else
        {
            output = $"Running {selectedLanguage} code is not supported in browser";
        }
    }
}
```

## 3. 기타 라이브러리 통합 패턴

### 3.1 일반적인 통합 패턴

```csharp
// LibraryWrapper.cs
public abstract class JavaScriptLibraryWrapper : IAsyncDisposable
{
    protected IJSRuntime JS { get; }
    protected IJSObjectReference? Module { get; set; }
    protected IJSObjectReference? Instance { get; set; }
    
    protected JavaScriptLibraryWrapper(IJSRuntime js)
    {
        JS = js;
    }
    
    public virtual async Task InitializeAsync(string modulePath, params object[] args)
    {
        Module = await JS.InvokeAsync<IJSObjectReference>("import", modulePath);
        Instance = await Module.InvokeAsync<IJSObjectReference>("initialize", args);
    }
    
    public virtual async ValueTask DisposeAsync()
    {
        if (Instance != null)
        {
            await Instance.InvokeVoidAsync("dispose");
            await Instance.DisposeAsync();
        }
        
        if (Module != null)
        {
            await Module.DisposeAsync();
        }
    }
}

// DatePickerWrapper.cs
public class DatePickerWrapper : JavaScriptLibraryWrapper
{
    public DatePickerWrapper(IJSRuntime js) : base(js) { }
    
    public async Task<IJSObjectReference> CreateDatePicker(string elementId, object options)
    {
        return await Instance!.InvokeAsync<IJSObjectReference>(
            "createDatePicker", elementId, options);
    }
    
    public async Task<DateTime?> GetDate(IJSObjectReference picker)
    {
        var dateString = await picker.InvokeAsync<string?>("getDate");
        return string.IsNullOrEmpty(dateString) 
            ? null 
            : DateTime.Parse(dateString);
    }
    
    public async Task SetDate(IJSObjectReference picker, DateTime date)
    {
        await picker.InvokeVoidAsync("setDate", date.ToString("yyyy-MM-dd"));
    }
}

// ThirdPartyComponent.razor
@implements IAsyncDisposable
@inject IJSRuntime JS

<div class="third-party-component">
    <div id="@componentId"></div>
</div>

@code {
    [Parameter] public string LibraryPath { get; set; } = "";
    [Parameter] public Dictionary<string, object> Options { get; set; } = new();
    
    private string componentId = $"component-{Guid.NewGuid():N}";
    private JavaScriptLibraryWrapper? wrapper;
    
    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender && !string.IsNullOrEmpty(LibraryPath))
        {
            // Dynamic wrapper creation based on library type
            wrapper = CreateWrapper(LibraryPath);
            await wrapper.InitializeAsync(LibraryPath, componentId, Options);
        }
    }
    
    private JavaScriptLibraryWrapper CreateWrapper(string libraryPath)
    {
        // Factory pattern for different library types
        return libraryPath switch
        {
            var path when path.Contains("datepicker") => new DatePickerWrapper(JS),
            _ => new GenericLibraryWrapper(JS)
        };
    }
    
    public async ValueTask DisposeAsync()
    {
        if (wrapper != null)
        {
            await wrapper.DisposeAsync();
        }
    }
    
    private class GenericLibraryWrapper : JavaScriptLibraryWrapper
    {
        public GenericLibraryWrapper(IJSRuntime js) : base(js) { }
    }
}
```

### 3.2 성능 최적화

```csharp
// OptimizedLibraryLoader.cs
public class OptimizedLibraryLoader
{
    private readonly IJSRuntime _js;
    private readonly Dictionary<string, Task<IJSObjectReference>> _loadingModules = new();
    private readonly Dictionary<string, IJSObjectReference> _loadedModules = new();
    
    public OptimizedLibraryLoader(IJSRuntime js)
    {
        _js = js;
    }
    
    public async Task<IJSObjectReference> LoadModuleAsync(string path)
    {
        // 이미 로드된 모듈 반환
        if (_loadedModules.TryGetValue(path, out var module))
        {
            return module;
        }
        
        // 로딩 중인 모듈 대기
        if (_loadingModules.TryGetValue(path, out var loadingTask))
        {
            return await loadingTask;
        }
        
        // 새 모듈 로드
        var task = LoadModuleInternalAsync(path);
        _loadingModules[path] = task;
        
        try
        {
            module = await task;
            _loadedModules[path] = module;
            return module;
        }
        finally
        {
            _loadingModules.Remove(path);
        }
    }
    
    private async Task<IJSObjectReference> LoadModuleInternalAsync(string path)
    {
        // 동적 import with error handling
        try
        {
            return await _js.InvokeAsync<IJSObjectReference>("import", path);
        }
        catch (JSException ex)
        {
            Console.WriteLine($"Failed to load module {path}: {ex.Message}");
            throw;
        }
    }
    
    public async Task PreloadModulesAsync(params string[] paths)
    {
        var tasks = paths.Select(LoadModuleAsync);
        await Task.WhenAll(tasks);
    }
}

// LazyLoadedChart.razor
@implements IAsyncDisposable
@inject OptimizedLibraryLoader LibraryLoader

@if (isLoaded)
{
    <div id="@chartId"></div>
}
else
{
    <div class="chart-placeholder">
        <span>Loading chart...</span>
    </div>
}

@code {
    [Parameter] public ChartConfiguration Config { get; set; } = new();
    
    private string chartId = $"chart-{Guid.NewGuid():N}";
    private bool isLoaded = false;
    private IJSObjectReference? chartModule;
    private IJSObjectReference? chart;
    
    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            // Lazy load chart library
            chartModule = await LibraryLoader.LoadModuleAsync("./js/chart-module.js");
            isLoaded = true;
            StateHasChanged();
            
            // Wait for re-render
            await Task.Yield();
            
            // Create chart
            chart = await chartModule.InvokeAsync<IJSObjectReference>(
                "createChart", chartId, Config);
        }
    }
    
    public async ValueTask DisposeAsync()
    {
        if (chart != null)
        {
            await chart.InvokeVoidAsync("destroy");
            await chart.DisposeAsync();
        }
    }
}
```

## 마무리

JavaScript 라이브러리 통합은 Blazor 애플리케이션의 기능을 크게 확장할 수 있습니다. 적절한 래퍼 패턴을 사용하고, 생명주기를 관리하며, 성능을 최적화하면 복잡한 JavaScript 라이브러리도 효과적으로 통합할 수 있습니다. Chart.js, Monaco Editor 같은 인기 라이브러리부터 시작하여 필요에 따라 다양한 라이브러리를 통합할 수 있습니다.