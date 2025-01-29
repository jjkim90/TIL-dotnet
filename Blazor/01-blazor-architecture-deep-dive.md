# Blazor 아키텍처 심화

## 개요

Blazor는 .NET을 사용하여 대화형 웹 UI를 빌드하기 위한 프레임워크입니다. JavaScript 대신 C#을 사용하여 풍부한 대화형 UI를 만들 수 있으며, 서버 측 및 클라이언트 측 앱 논리를 .NET으로 작성할 수 있습니다.

## 1. Blazor의 핵심 아키텍처

### 1.1 컴포넌트 기반 아키텍처

Blazor는 React나 Vue.js와 같은 현대적인 프론트엔드 프레임워크처럼 컴포넌트 기반 아키텍처를 채택합니다.

```csharp
// Counter.razor
@page "/counter"

<h1>Counter</h1>
<p>Current count: @currentCount</p>
<button @onclick="IncrementCount">Click me</button>

@code {
    private int currentCount = 0;

    private void IncrementCount()
    {
        currentCount++;
    }
}
```

### 1.2 렌더링 프로세스

Blazor의 렌더링 프로세스는 다음과 같은 단계로 진행됩니다:

1. **Component Tree 생성**: 컴포넌트의 계층 구조 생성
2. **Render Tree 구축**: 각 컴포넌트의 BuildRenderTree 메서드 실행
3. **Diffing**: 이전 렌더링과 현재 렌더링 비교
4. **DOM 업데이트**: 변경된 부분만 실제 DOM에 반영

```csharp
public abstract class ComponentBase : IComponent
{
    protected virtual void BuildRenderTree(RenderTreeBuilder builder)
    {
        // 렌더 트리 구축 로직
    }
    
    protected void StateHasChanged()
    {
        // 리렌더링 트리거
    }
}
```

## 2. 호스팅 모델 비교

### 2.1 Blazor Server

Blazor Server는 ASP.NET Core 앱 내에서 서버 측에서 실행됩니다.

**아키텍처:**
```
클라이언트 (브라우저)
    ↕ SignalR Connection
서버 (ASP.NET Core)
    - Component State
    - .NET Runtime
    - App Logic
```

**특징:**
- 다운로드 크기가 작음 (초기 로딩 빠름)
- .NET Core의 전체 기능 사용 가능
- 서버 리소스 필요
- 네트워크 지연 시간에 민감

**구현 예제:**
```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddRazorPages();
builder.Services.AddServerSideBlazor();
builder.Services.AddSingleton<WeatherForecastService>();

var app = builder.Build();

app.UseStaticFiles();
app.UseRouting();

app.MapBlazorHub();
app.MapFallbackToPage("/_Host");

app.Run();
```

### 2.2 Blazor WebAssembly

Blazor WebAssembly는 WebAssembly 기반 .NET 런타임을 사용하여 브라우저에서 직접 실행됩니다.

**아키텍처:**
```
클라이언트 (브라우저)
    - WebAssembly (.NET Runtime)
    - Component State
    - App Logic
    ↕ HTTP/REST API
서버 (Any Backend)
    - API Endpoints
    - Static File Hosting
```

**특징:**
- 완전한 클라이언트 측 앱
- 오프라인 작동 가능
- 서버 부하 없음
- 초기 다운로드 크기가 큼

**구현 예제:**
```csharp
// Program.cs (Blazor WebAssembly)
var builder = WebAssemblyHostBuilder.CreateDefault(args);
builder.RootComponents.Add<App>("#app");

builder.Services.AddScoped(sp => 
    new HttpClient { BaseAddress = new Uri(builder.HostEnvironment.BaseAddress) });

await builder.Build().RunAsync();
```

### 2.3 Blazor Hybrid

.NET MAUI와 함께 사용하여 네이티브 앱에서 Blazor 컴포넌트를 실행합니다.

```csharp
// MauiProgram.cs
public static class MauiProgram
{
    public static MauiApp CreateMauiApp()
    {
        var builder = MauiApp.CreateBuilder();
        builder
            .UseMauiApp<App>()
            .ConfigureFonts(fonts =>
            {
                fonts.AddFont("OpenSans-Regular.ttf", "OpenSansRegular");
            });

        builder.Services.AddMauiBlazorWebView();
        
        return builder.Build();
    }
}
```

## 3. .NET 8 통합 렌더링 모드

.NET 8에서는 Blazor의 렌더링 모드가 크게 개선되었습니다.

### 3.1 렌더링 모드 종류

```csharp
@rendermode InteractiveServer  // 서버 측 렌더링
@rendermode InteractiveWebAssembly  // 클라이언트 측 렌더링
@rendermode InteractiveAuto  // 자동 선택
@rendermode @(new InteractiveServerRenderMode(prerender: false))  // 프리렌더링 비활성화
```

### 3.2 페이지별 렌더링 모드 설정

```razor
@page "/weather"
@rendermode InteractiveServer

<h3>Weather</h3>

@if (forecasts == null)
{
    <p>Loading...</p>
}
else
{
    <table class="table">
        <!-- 날씨 데이터 표시 -->
    </table>
}

@code {
    private WeatherForecast[]? forecasts;

    protected override async Task OnInitializedAsync()
    {
        forecasts = await ForecastService.GetForecastAsync();
    }
}
```

### 3.3 컴포넌트별 렌더링 모드

```razor
<!-- App.razor -->
<Router AppAssembly="typeof(App).Assembly">
    <Found Context="routeData">
        <RouteView RouteData="routeData" DefaultLayout="typeof(MainLayout)" />
    </Found>
</Router>

<!-- 사용 예 -->
<Counter @rendermode="InteractiveWebAssembly" />
<ServerClock @rendermode="InteractiveServer" />
```

## 4. 렌더링 최적화 전략

### 4.1 Streaming Rendering

.NET 8에서 도입된 스트리밍 렌더링으로 점진적 UI 업데이트가 가능합니다.

```csharp
@page "/products"
@attribute [StreamRendering]

<h1>Products</h1>

@if (products == null)
{
    <p>Loading products...</p>
}
else
{
    @foreach (var product in products)
    {
        <ProductCard Product="product" />
    }
}

@code {
    private List<Product>? products;

    protected override async Task OnInitializedAsync()
    {
        // 스트리밍으로 데이터 로드
        products = await ProductService.GetProductsAsync();
    }
}
```

### 4.2 Enhanced Navigation

향상된 내비게이션으로 SPA와 같은 사용자 경험을 제공합니다.

```html
<!-- _Host.cshtml 또는 index.html -->
<script src="_framework/blazor.web.js"></script>
<script>
    Blazor.start({
        ssr: { disableDomPreservation: true }
    });
</script>
```

## 5. 실전 아키텍처 패턴

### 5.1 Clean Architecture 적용

```csharp
// Domain Layer
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
}

// Application Layer
public interface IProductService
{
    Task<IEnumerable<Product>> GetAllProductsAsync();
    Task<Product> GetProductByIdAsync(int id);
}

// Infrastructure Layer
public class ProductService : IProductService
{
    private readonly HttpClient _httpClient;
    
    public ProductService(HttpClient httpClient)
    {
        _httpClient = httpClient;
    }
    
    public async Task<IEnumerable<Product>> GetAllProductsAsync()
    {
        return await _httpClient.GetFromJsonAsync<IEnumerable<Product>>("api/products");
    }
}

// Presentation Layer (Blazor Component)
@page "/products"
@inject IProductService ProductService

<h1>Products</h1>

@if (products == null)
{
    <p>Loading...</p>
}
else
{
    @foreach (var product in products)
    {
        <div>@product.Name - $@product.Price</div>
    }
}

@code {
    private IEnumerable<Product>? products;
    
    protected override async Task OnInitializedAsync()
    {
        products = await ProductService.GetAllProductsAsync();
    }
}
```

### 5.2 상태 관리 아키텍처

```csharp
// State Container
public class AppState
{
    private readonly List<Product> _cartItems = new();
    
    public IReadOnlyList<Product> CartItems => _cartItems.AsReadOnly();
    
    public event Action? OnChange;
    
    public void AddToCart(Product product)
    {
        _cartItems.Add(product);
        NotifyStateChanged();
    }
    
    public void RemoveFromCart(Product product)
    {
        _cartItems.Remove(product);
        NotifyStateChanged();
    }
    
    private void NotifyStateChanged() => OnChange?.Invoke();
}

// Program.cs
builder.Services.AddScoped<AppState>();

// Component
@implements IDisposable
@inject AppState AppState

<h3>Shopping Cart (@AppState.CartItems.Count items)</h3>

@code {
    protected override void OnInitialized()
    {
        AppState.OnChange += StateHasChanged;
    }
    
    public void Dispose()
    {
        AppState.OnChange -= StateHasChanged;
    }
}
```

## 6. 성능 고려사항

### 6.1 컴포넌트 수명 주기 최적화

```csharp
public class OptimizedComponent : ComponentBase, IDisposable
{
    private CancellationTokenSource? _cancellationTokenSource;
    
    protected override async Task OnInitializedAsync()
    {
        _cancellationTokenSource = new CancellationTokenSource();
        
        try
        {
            // 취소 가능한 비동기 작업
            var data = await LoadDataAsync(_cancellationTokenSource.Token);
            ProcessData(data);
        }
        catch (OperationCanceledException)
        {
            // 컴포넌트가 dispose되면 작업 취소
        }
    }
    
    public void Dispose()
    {
        _cancellationTokenSource?.Cancel();
        _cancellationTokenSource?.Dispose();
    }
}
```

### 6.2 렌더링 최적화

```csharp
@implements IComponent

@code {
    private RenderHandle _renderHandle;
    private bool _shouldRender = true;
    
    public void Attach(RenderHandle renderHandle)
    {
        _renderHandle = renderHandle;
    }
    
    public Task SetParametersAsync(ParameterView parameters)
    {
        // 파라미터 변경 시에만 렌더링
        if (HasParametersChanged(parameters))
        {
            _shouldRender = true;
        }
        
        if (_shouldRender)
        {
            _renderHandle.Render(BuildRenderTree);
            _shouldRender = false;
        }
        
        return Task.CompletedTask;
    }
}
```

## 7. 마이그레이션 전략

### 7.1 Blazor Server에서 WebAssembly로

```csharp
// Shared Component Library
public class WeatherComponent : ComponentBase
{
    [Inject] public IWeatherService WeatherService { get; set; }
    
    protected override async Task OnInitializedAsync()
    {
        // 동일한 로직, 다른 구현
        var data = await WeatherService.GetWeatherAsync();
    }
}

// Blazor Server Implementation
public class ServerWeatherService : IWeatherService
{
    private readonly WeatherDbContext _context;
    
    public async Task<Weather> GetWeatherAsync()
    {
        return await _context.Weather.FirstOrDefaultAsync();
    }
}

// Blazor WebAssembly Implementation  
public class ClientWeatherService : IWeatherService
{
    private readonly HttpClient _httpClient;
    
    public async Task<Weather> GetWeatherAsync()
    {
        return await _httpClient.GetFromJsonAsync<Weather>("api/weather");
    }
}
```

## 마무리

Blazor의 아키텍처는 유연하고 확장 가능하도록 설계되었습니다. 서버 측, 클라이언트 측, 하이브리드 등 다양한 호스팅 모델을 지원하며, .NET 8의 통합 렌더링 모드로 더욱 강력해졌습니다. 

각 호스팅 모델의 장단점을 이해하고, 프로젝트 요구사항에 맞는 적절한 아키텍처를 선택하는 것이 중요합니다. 또한 렌더링 최적화와 상태 관리 전략을 통해 고성능 Blazor 애플리케이션을 구축할 수 있습니다.