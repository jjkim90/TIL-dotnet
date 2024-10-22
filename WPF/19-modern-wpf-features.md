# 최신 WPF 기능

## .NET Core/.NET 5+ WPF

.NET Core 3.0부터 WPF를 지원하며, 성능 향상과 새로운 기능들이 추가되었습니다.

### 프로젝트 설정
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>net6.0-windows</TargetFramework>
    <UseWPF>true</UseWPF>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
</Project>
```

### 최신 C# 기능 활용
```csharp
// Record 타입 사용
public record Person(string FirstName, string LastName, int Age);

// Pattern Matching
public string GetPersonInfo(object obj) => obj switch
{
    Person { Age: < 18 } p => $"{p.FirstName} is a minor",
    Person { Age: >= 65 } p => $"{p.FirstName} is a senior",
    Person p => $"{p.FirstName} is an adult",
    _ => "Unknown"
};

// Init-only 속성
public class ImmutableViewModel
{
    public string Title { get; init; }
    public DateTime CreatedAt { get; init; } = DateTime.Now;
}
```

## High DPI 지원

### DPI 인식 설정
```xml
<!-- app.manifest -->
<application xmlns="urn:schemas-microsoft-com:asm.v3">
  <windowsSettings>
    <dpiAware xmlns="http://schemas.microsoft.com/SMI/2005/WindowsSettings">true/PM</dpiAware>
    <dpiAwareness xmlns="http://schemas.microsoft.com/SMI/2016/WindowsSettings">PerMonitorV2</dpiAwareness>
  </windowsSettings>
</application>
```

### DPI 변경 처리
```csharp
public partial class DpiAwareWindow : Window
{
    private double _currentDpi = 96.0;
    
    protected override void OnSourceInitialized(EventArgs e)
    {
        base.OnSourceInitialized(e);
        
        var source = PresentationSource.FromVisual(this);
        if (source?.CompositionTarget != null)
        {
            _currentDpi = 96.0 * source.CompositionTarget.TransformToDevice.M11;
            
            // DPI 변경 이벤트 구독
            var hwndSource = source as HwndSource;
            hwndSource?.AddHook(WndProc);
        }
    }
    
    private IntPtr WndProc(IntPtr hwnd, int msg, IntPtr wParam, IntPtr lParam, ref bool handled)
    {
        const int WM_DPICHANGED = 0x02E0;
        
        if (msg == WM_DPICHANGED)
        {
            var newDpi = (wParam.ToInt32() & 0xFFFF);
            OnDpiChanged(_currentDpi, newDpi);
            _currentDpi = newDpi;
        }
        
        return IntPtr.Zero;
    }
    
    private void OnDpiChanged(double oldDpi, double newDpi)
    {
        var scaleFactor = newDpi / oldDpi;
        
        // UI 요소 크기 조정
        Width *= scaleFactor;
        Height *= scaleFactor;
        
        // 폰트 크기 조정
        UpdateFontSizes(scaleFactor);
    }
}
```

## Modern UI 라이브러리

### ModernWpf UI Library
```xml
<!-- NuGet 패키지 설치 -->
<PackageReference Include="ModernWpfUI" Version="0.9.6" />
```

```csharp
// App.xaml.cs
public partial class App : Application
{
    protected override void OnStartup(StartupEventArgs e)
    {
        base.OnStartup(e);
        
        // ModernWpf 테마 적용
        ModernWpf.ThemeManager.Current.ApplicationTheme = 
            ModernWpf.ApplicationTheme.Dark;
    }
}
```

```xml
<!-- ModernWpf 컨트롤 사용 -->
<ui:NavigationView x:Name="navigationView"
                   PaneDisplayMode="LeftCompact"
                   IsBackButtonVisible="Visible"
                   BackRequested="NavigationView_BackRequested">
    <ui:NavigationView.MenuItems>
        <ui:NavigationViewItem Content="Home" Tag="home">
            <ui:NavigationViewItem.Icon>
                <ui:SymbolIcon Symbol="Home"/>
            </ui:NavigationViewItem.Icon>
        </ui:NavigationViewItem>
        <ui:NavigationViewItem Content="Settings" Tag="settings">
            <ui:NavigationViewItem.Icon>
                <ui:SymbolIcon Symbol="Setting"/>
            </ui:NavigationViewItem.Icon>
        </ui:NavigationViewItem>
    </ui:NavigationView.MenuItems>
    
    <Frame x:Name="contentFrame"/>
</ui:NavigationView>
```

### Fluent Design System
```csharp
public class FluentWindow : Window
{
    public FluentWindow()
    {
        // Acrylic 효과
        WindowStyle = WindowStyle.None;
        AllowsTransparency = true;
        Background = new SolidColorBrush(Color.FromArgb(200, 32, 32, 32));
        
        // 그림자 효과
        var shadowEffect = new DropShadowEffect
        {
            BlurRadius = 15,
            Direction = 270,
            ShadowDepth = 5,
            Opacity = 0.5
        };
        Effect = shadowEffect;
    }
}

// Reveal 효과 구현
public class RevealBorder : Border
{
    private readonly GradientStop _gradientStop;
    private readonly RadialGradientBrush _brush;
    
    public RevealBorder()
    {
        _gradientStop = new GradientStop(Colors.Transparent, 0);
        _brush = new RadialGradientBrush
        {
            GradientStops = { _gradientStop, 
                new GradientStop(Colors.Transparent, 1) }
        };
        
        BorderBrush = _brush;
        BorderThickness = new Thickness(1);
        
        MouseMove += OnMouseMove;
        MouseLeave += OnMouseLeave;
    }
    
    private void OnMouseMove(object sender, MouseEventArgs e)
    {
        var position = e.GetPosition(this);
        var relativeX = position.X / ActualWidth;
        var relativeY = position.Y / ActualHeight;
        
        _brush.Center = new Point(relativeX, relativeY);
        _brush.GradientOrigin = new Point(relativeX, relativeY);
        
        _gradientStop.Color = Color.FromArgb(60, 255, 255, 255);
    }
    
    private void OnMouseLeave(object sender, MouseEventArgs e)
    {
        _gradientStop.Color = Colors.Transparent;
    }
}
```

## WebView2 통합

### WebView2 설정
```xml
<PackageReference Include="Microsoft.Web.WebView2" Version="1.0.1210.39" />
```

```csharp
public partial class WebViewWindow : Window
{
    public WebViewWindow()
    {
        InitializeComponent();
        InitializeWebView();
    }
    
    private async void InitializeWebView()
    {
        await webView.EnsureCoreWebView2Async();
        
        // JavaScript 통신 설정
        webView.CoreWebView2.WebMessageReceived += OnWebMessageReceived;
        
        // 네비게이션 이벤트
        webView.NavigationStarting += OnNavigationStarting;
        webView.NavigationCompleted += OnNavigationCompleted;
        
        // 초기 페이지 로드
        webView.Source = new Uri("https://www.example.com");
    }
    
    private void OnWebMessageReceived(object sender, CoreWebView2WebMessageReceivedEventArgs e)
    {
        var message = e.TryGetWebMessageAsString();
        ProcessWebMessage(message);
    }
    
    // WPF에서 JavaScript 호출
    private async void CallJavaScript()
    {
        var result = await webView.ExecuteScriptAsync(
            "document.getElementById('result').innerText");
        
        MessageBox.Show($"Result: {result}");
    }
    
    // JavaScript에서 WPF 호출
    private async void InjectScript()
    {
        await webView.CoreWebView2.AddScriptToExecuteOnDocumentCreatedAsync(@"
            window.chrome.webview.postMessage('Hello from JavaScript');
            
            function callWpf(data) {
                window.chrome.webview.postMessage(JSON.stringify(data));
            }
        ");
    }
}
```

### WebView2와 WPF 양방향 통신
```csharp
public class WebViewBridge
{
    private readonly WebView2 _webView;
    private readonly Dictionary<string, Action<object>> _handlers = new();
    
    public WebViewBridge(WebView2 webView)
    {
        _webView = webView;
        _webView.CoreWebView2InitializationCompleted += OnInitialized;
    }
    
    private void OnInitialized(object sender, EventArgs e)
    {
        _webView.CoreWebView2.WebMessageReceived += OnMessageReceived;
        
        // 브리지 객체 주입
        _webView.CoreWebView2.AddHostObjectToScript("wpfBridge", this);
    }
    
    public void RegisterHandler(string command, Action<object> handler)
    {
        _handlers[command] = handler;
    }
    
    private void OnMessageReceived(object sender, CoreWebView2WebMessageReceivedEventArgs e)
    {
        var message = JsonSerializer.Deserialize<BridgeMessage>(e.TryGetWebMessageAsString());
        
        if (_handlers.TryGetValue(message.Command, out var handler))
        {
            handler(message.Data);
        }
    }
    
    public async Task SendToJavaScript(string command, object data)
    {
        var message = new { command, data };
        var json = JsonSerializer.Serialize(message);
        
        await _webView.ExecuteScriptAsync($"handleWpfMessage({json})");
    }
}

public class BridgeMessage
{
    public string Command { get; set; }
    public object Data { get; set; }
}
```

## Touch 및 Stylus 지원

### 멀티터치 처리
```csharp
public class TouchCanvas : Canvas
{
    private readonly Dictionary<int, TouchPoint> _touchPoints = new();
    
    public TouchCanvas()
    {
        IsManipulationEnabled = true;
    }
    
    protected override void OnTouchDown(TouchEventArgs e)
    {
        base.OnTouchDown(e);
        
        var touchPoint = e.GetTouchPoint(this);
        _touchPoints[e.TouchDevice.Id] = touchPoint;
        
        // 터치 시각화
        DrawTouchPoint(touchPoint);
        
        e.Handled = true;
    }
    
    protected override void OnTouchMove(TouchEventArgs e)
    {
        base.OnTouchMove(e);
        
        var touchPoint = e.GetTouchPoint(this);
        if (_touchPoints.ContainsKey(e.TouchDevice.Id))
        {
            // 이동 경로 그리기
            DrawLine(_touchPoints[e.TouchDevice.Id].Position, 
                    touchPoint.Position);
            _touchPoints[e.TouchDevice.Id] = touchPoint;
        }
        
        e.Handled = true;
    }
    
    protected override void OnTouchUp(TouchEventArgs e)
    {
        base.OnTouchUp(e);
        _touchPoints.Remove(e.TouchDevice.Id);
        e.Handled = true;
    }
    
    // 제스처 인식
    protected override void OnManipulationDelta(ManipulationDeltaEventArgs e)
    {
        base.OnManipulationDelta(e);
        
        var element = e.Source as UIElement;
        if (element != null)
        {
            var transform = element.RenderTransform as MatrixTransform 
                          ?? new MatrixTransform();
            var matrix = transform.Matrix;
            
            // 이동
            matrix.Translate(e.DeltaManipulation.Translation.X,
                           e.DeltaManipulation.Translation.Y);
            
            // 회전
            matrix.RotateAt(e.DeltaManipulation.Rotation,
                          e.ManipulationOrigin.X,
                          e.ManipulationOrigin.Y);
            
            // 확대/축소
            matrix.ScaleAt(e.DeltaManipulation.Scale.X,
                         e.DeltaManipulation.Scale.Y,
                         e.ManipulationOrigin.X,
                         e.ManipulationOrigin.Y);
            
            transform.Matrix = matrix;
            element.RenderTransform = transform;
        }
        
        e.Handled = true;
    }
}
```

### Stylus 지원
```csharp
public class InkCanvas : Canvas
{
    private readonly InkPresenter _inkPresenter;
    private Stroke _currentStroke;
    
    public InkCanvas()
    {
        _inkPresenter = new InkPresenter();
        Children.Add(_inkPresenter);
        
        StylusDown += OnStylusDown;
        StylusMove += OnStylusMove;
        StylusUp += OnStylusUp;
    }
    
    private void OnStylusDown(object sender, StylusDownEventArgs e)
    {
        var points = e.GetStylusPoints(this);
        
        // 압력 감지
        var drawingAttributes = new DrawingAttributes
        {
            Color = Colors.Black,
            Width = 2,
            Height = 2,
            FitToCurve = true,
            IsHighlighter = false
        };
        
        _currentStroke = new Stroke(points, drawingAttributes);
        _inkPresenter.Strokes.Add(_currentStroke);
        
        e.Handled = true;
    }
    
    private void OnStylusMove(object sender, StylusEventArgs e)
    {
        if (_currentStroke != null)
        {
            var points = e.GetStylusPoints(this);
            _currentStroke.StylusPoints.Add(points);
        }
        
        e.Handled = true;
    }
    
    private void OnStylusUp(object sender, StylusEventArgs e)
    {
        _currentStroke = null;
        e.Handled = true;
    }
    
    // 필기 인식
    public async Task<string> RecognizeHandwriting()
    {
        var analyzer = new InkAnalyzer();
        analyzer.AddStrokes(_inkPresenter.Strokes);
        
        var result = await Task.Run(() => analyzer.Analyze());
        
        if (result.Status == InkAnalysisStatus.Updated)
        {
            var text = string.Join(" ", 
                analyzer.AnalysisRoot.GetRecognizedString());
            return text;
        }
        
        return string.Empty;
    }
}
```

## XAML Islands

### Win32 앱에 WPF 컨트롤 호스팅
```csharp
public class XamlIslandHost
{
    private WindowsXamlManager _xamlManager;
    private DesktopWindowXamlSource _xamlSource;
    
    public void InitializeXamlIsland(IntPtr parentWindow)
    {
        // XAML Islands 초기화
        _xamlManager = WindowsXamlManager.InitializeForCurrentThread();
        _xamlSource = new DesktopWindowXamlSource();
        
        // 부모 창 설정
        var interop = _xamlSource.As<IDesktopWindowXamlSourceNative>();
        interop.AttachToWindow(parentWindow);
        
        // WPF 컨트롤 생성
        var wpfControl = new Button
        {
            Content = "WPF in Win32",
            Width = 200,
            Height = 50
        };
        
        wpfControl.Click += (s, e) => MessageBox.Show("Clicked!");
        
        _xamlSource.Content = wpfControl;
    }
    
    public void SetBounds(int x, int y, int width, int height)
    {
        var interop = _xamlSource.As<IDesktopWindowXamlSourceNative>();
        var hwnd = interop.WindowHandle;
        
        SetWindowPos(hwnd, IntPtr.Zero, x, y, width, height, 
                    SWP_SHOWWINDOW);
    }
}
```

## 비동기 데이터 바인딩

### 비동기 속성
```csharp
public class AsyncViewModel : ViewModelBase
{
    private readonly Lazy<Task<string>> _asyncData;
    
    public AsyncViewModel()
    {
        _asyncData = new Lazy<Task<string>>(LoadDataAsync);
    }
    
    public Task<string> AsyncData => _asyncData.Value;
    
    private async Task<string> LoadDataAsync()
    {
        await Task.Delay(2000); // 시뮬레이션
        return "Async loaded data";
    }
    
    // NotifyTask 패턴
    public NotifyTask<List<Item>> Items { get; }
    
    public AsyncViewModel()
    {
        Items = new NotifyTask<List<Item>>(LoadItemsAsync());
    }
    
    private async Task<List<Item>> LoadItemsAsync()
    {
        var items = await DataService.GetItemsAsync();
        return items;
    }
}

// NotifyTask 구현
public class NotifyTask<T> : INotifyPropertyChanged
{
    public Task<T> Task { get; private set; }
    public T Result => Task.Status == TaskStatus.RanToCompletion ? 
                       Task.Result : default(T);
    public TaskStatus Status => Task.Status;
    public bool IsCompleted => Task.IsCompleted;
    public bool IsNotCompleted => !Task.IsCompleted;
    public bool IsSuccessfullyCompleted => Task.Status == TaskStatus.RanToCompletion;
    public bool IsCanceled => Task.IsCanceled;
    public bool IsFaulted => Task.IsFaulted;
    public Exception Exception => Task.Exception;
    
    public NotifyTask(Task<T> task)
    {
        Task = task;
        TaskCompletion = WatchTaskAsync(task);
    }
    
    private async Task WatchTaskAsync(Task task)
    {
        try
        {
            await task;
        }
        catch { }
        
        OnPropertyChanged(nameof(Status));
        OnPropertyChanged(nameof(IsCompleted));
        OnPropertyChanged(nameof(IsNotCompleted));
        
        if (task.IsCanceled)
        {
            OnPropertyChanged(nameof(IsCanceled));
        }
        else if (task.IsFaulted)
        {
            OnPropertyChanged(nameof(IsFaulted));
            OnPropertyChanged(nameof(Exception));
        }
        else
        {
            OnPropertyChanged(nameof(IsSuccessfullyCompleted));
            OnPropertyChanged(nameof(Result));
        }
    }
    
    public Task TaskCompletion { get; private set; }
    
    public event PropertyChangedEventHandler PropertyChanged;
    
    protected virtual void OnPropertyChanged(string propertyName)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }
}
```

### 비동기 바인딩 XAML
```xml
<!-- 비동기 데이터 바인딩 -->
<TextBlock>
    <TextBlock.Style>
        <Style TargetType="TextBlock">
            <Setter Property="Text" Value="{Binding AsyncData.Result}"/>
            <Style.Triggers>
                <DataTrigger Binding="{Binding AsyncData.IsNotCompleted}" Value="True">
                    <Setter Property="Text" Value="Loading..."/>
                </DataTrigger>
                <DataTrigger Binding="{Binding AsyncData.IsFaulted}" Value="True">
                    <Setter Property="Text" Value="Error loading data"/>
                    <Setter Property="Foreground" Value="Red"/>
                </DataTrigger>
            </Style.Triggers>
        </Style>
    </TextBlock.Style>
</TextBlock>

<!-- NotifyTask 사용 -->
<ItemsControl ItemsSource="{Binding Items.Result}">
    <ItemsControl.Style>
        <Style TargetType="ItemsControl">
            <Style.Triggers>
                <DataTrigger Binding="{Binding Items.IsNotCompleted}" Value="True">
                    <Setter Property="Visibility" Value="Collapsed"/>
                </DataTrigger>
            </Style.Triggers>
        </Style>
    </ItemsControl.Style>
</ItemsControl>

<!-- 로딩 인디케이터 -->
<ProgressBar IsIndeterminate="True" 
             Visibility="{Binding Items.IsNotCompleted, 
                         Converter={StaticResource BooleanToVisibilityConverter}}"/>
```

## Source Generators

### Source Generator를 활용한 MVVM
```csharp
// Source Generator 정의
[Generator]
public class ViewModelGenerator : ISourceGenerator
{
    public void Initialize(GeneratorInitializationContext context)
    {
        context.RegisterForSyntaxNotifications(() => new SyntaxReceiver());
    }
    
    public void Execute(GeneratorExecutionContext context)
    {
        var receiver = context.SyntaxReceiver as SyntaxReceiver;
        
        foreach (var classDeclaration in receiver.CandidateClasses)
        {
            var model = context.Compilation.GetSemanticModel(
                classDeclaration.SyntaxTree);
            var symbol = model.GetDeclaredSymbol(classDeclaration);
            
            if (symbol.GetAttributes().Any(a => 
                a.AttributeClass.Name == "AutoNotifyAttribute"))
            {
                var source = GeneratePropertyCode(symbol);
                context.AddSource($"{symbol.Name}.Generated.cs", source);
            }
        }
    }
}

// 사용 예
[AutoNotify]
public partial class PersonViewModel
{
    private string _firstName;
    private string _lastName;
    private int _age;
}

// 생성된 코드
public partial class PersonViewModel : INotifyPropertyChanged
{
    public string FirstName
    {
        get => _firstName;
        set
        {
            if (_firstName != value)
            {
                _firstName = value;
                OnPropertyChanged();
            }
        }
    }
    
    // ... 나머지 속성들
    
    public event PropertyChangedEventHandler PropertyChanged;
    
    protected virtual void OnPropertyChanged([CallerMemberName] string propertyName = null)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }
}
```

## 성능 프로파일링 도구

### Visual Studio Diagnostic Tools
```csharp
public class PerformanceProfiler
{
    private readonly DiagnosticSource _diagnosticSource;
    
    public PerformanceProfiler()
    {
        _diagnosticSource = new DiagnosticListener("WPF.Performance");
    }
    
    public IDisposable MeasureActivity(string operationName)
    {
        if (_diagnosticSource.IsEnabled(operationName))
        {
            var activity = new Activity(operationName);
            _diagnosticSource.StartActivity(activity, new { Timestamp = DateTime.UtcNow });
            
            return new ActivityScope(_diagnosticSource, activity);
        }
        
        return null;
    }
    
    private class ActivityScope : IDisposable
    {
        private readonly DiagnosticSource _source;
        private readonly Activity _activity;
        
        public ActivityScope(DiagnosticSource source, Activity activity)
        {
            _source = source;
            _activity = activity;
        }
        
        public void Dispose()
        {
            _source.StopActivity(_activity, new 
            { 
                Duration = _activity.Duration,
                Timestamp = DateTime.UtcNow 
            });
        }
    }
}

// 사용 예
public async Task LoadDataAsync()
{
    using (profiler.MeasureActivity("DataLoading"))
    {
        var data = await dataService.GetDataAsync();
        
        using (profiler.MeasureActivity("DataProcessing"))
        {
            ProcessData(data);
        }
    }
}
```

## 실전 예제: 모던 대시보드 애플리케이션

### MainWindow.xaml
```xml
<Window x:Class="ModernDashboard.MainWindow"
        xmlns:ui="http://schemas.modernwpf.com/2019"
        ui:WindowHelper.UseModernWindowStyle="True">
    
    <Grid>
        <Grid.ColumnDefinitions>
            <ColumnDefinition Width="Auto"/>
            <ColumnDefinition Width="*"/>
        </Grid.ColumnDefinitions>
        
        <!-- 네비게이션 -->
        <ui:NavigationView Grid.Column="0" 
                           x:Name="navigationView"
                           PaneDisplayMode="Left"
                           IsBackButtonVisible="Collapsed"
                           SelectionChanged="NavigationView_SelectionChanged">
            <ui:NavigationView.MenuItems>
                <ui:NavigationViewItem Content="Dashboard" Tag="Dashboard">
                    <ui:NavigationViewItem.Icon>
                        <ui:SymbolIcon Symbol="Home"/>
                    </ui:NavigationViewItem.Icon>
                </ui:NavigationViewItem>
                <ui:NavigationViewItem Content="Analytics" Tag="Analytics">
                    <ui:NavigationViewItem.Icon>
                        <ui:SymbolIcon Symbol="ReportHacked"/>
                    </ui:NavigationViewItem.Icon>
                </ui:NavigationViewItem>
            </ui:NavigationView.MenuItems>
        </ui:NavigationView>
        
        <!-- 콘텐츠 영역 -->
        <Frame Grid.Column="1" x:Name="contentFrame"/>
    </Grid>
</Window>
```

### DashboardViewModel.cs
```csharp
public class DashboardViewModel : ViewModelBase
{
    private readonly IDataService _dataService;
    
    public DashboardViewModel(IDataService dataService)
    {
        _dataService = dataService;
        
        // 비동기 데이터 로드
        RevenueData = new NotifyTask<ObservableCollection<RevenueItem>>(
            LoadRevenueDataAsync());
        UserMetrics = new NotifyTask<UserMetrics>(
            LoadUserMetricsAsync());
        
        // 실시간 업데이트
        StartRealtimeUpdates();
    }
    
    public NotifyTask<ObservableCollection<RevenueItem>> RevenueData { get; }
    public NotifyTask<UserMetrics> UserMetrics { get; }
    
    public ObservableCollection<LiveDataPoint> LiveData { get; } = new();
    
    private async Task<ObservableCollection<RevenueItem>> LoadRevenueDataAsync()
    {
        var data = await _dataService.GetRevenueDataAsync();
        return new ObservableCollection<RevenueItem>(data);
    }
    
    private async Task<UserMetrics> LoadUserMetricsAsync()
    {
        return await _dataService.GetUserMetricsAsync();
    }
    
    private void StartRealtimeUpdates()
    {
        var timer = new DispatcherTimer
        {
            Interval = TimeSpan.FromSeconds(1)
        };
        
        timer.Tick += async (s, e) =>
        {
            var newPoint = await _dataService.GetLatestDataPointAsync();
            
            Application.Current.Dispatcher.Invoke(() =>
            {
                LiveData.Add(newPoint);
                
                // 최대 100개 포인트 유지
                if (LiveData.Count > 100)
                {
                    LiveData.RemoveAt(0);
                }
            });
        };
        
        timer.Start();
    }
}
```

## 핵심 개념 정리
- **.NET Core/5+**: 최신 .NET 플랫폼에서의 WPF
- **High DPI**: Per-Monitor DPI 인식 지원
- **Modern UI**: Fluent Design, ModernWpf 라이브러리
- **WebView2**: 최신 웹 기술 통합
- **Touch/Stylus**: 멀티터치 및 펜 입력 지원
- **XAML Islands**: Win32 앱에 WPF 통합
- **비동기 바인딩**: Task 기반 데이터 바인딩
- **Source Generators**: 컴파일 타임 코드 생성
- **성능 도구**: 진단 및 프로파일링
- **C# 최신 기능**: Record, Pattern Matching 등
- **Fluent Design**: Acrylic, Reveal 효과
- **실시간 데이터**: 라이브 업데이트 처리