# Blazor WebAssembly

## Blazor WebAssembly 소개

Blazor WebAssembly는 .NET 런타임을 WebAssembly로 컴파일하여 브라우저에서 직접 실행되는 클라이언트 측 웹 프레임워크입니다. JavaScript 대신 C#을 사용하여 대화형 웹 애플리케이션을 구축할 수 있으며, 완전한 SPA(Single Page Application)를 만들 수 있습니다.

### Blazor WebAssembly의 특징

- **진정한 클라이언트 측 실행**: 브라우저에서 .NET 코드 직접 실행
- **오프라인 지원**: Progressive Web App(PWA) 구축 가능
- **서버 독립성**: 정적 호스팅 가능
- **빠른 UI 상호작용**: 서버 왕복 없이 즉각적인 반응
- **표준 기반**: W3C WebAssembly 표준 사용

### Blazor Server vs WebAssembly

| 특징 | Blazor Server | Blazor WebAssembly |
|------|--------------|-------------------|
| 실행 위치 | 서버 | 브라우저 |
| 초기 로드 시간 | 빠름 | 느림 (런타임 다운로드) |
| 네트워크 요구사항 | 지속적 연결 필요 | 초기 다운로드만 필요 |
| 오프라인 지원 | 불가능 | 가능 (PWA) |
| 리소스 사용 | 서버 리소스 | 클라이언트 리소스 |
| 보안 | 앱 코드 서버에 유지 | 앱 코드 클라이언트에 노출 |

## 프로젝트 생성 및 구조

### 프로젝트 생성

```bash
# Blazor WebAssembly 앱 생성
dotnet new blazorwasm -n MyBlazorWasmApp

# ASP.NET Core 호스팅 포함
dotnet new blazorwasm -n MyBlazorWasmApp --hosted

# PWA 지원 포함
dotnet new blazorwasm -n MyBlazorWasmApp --pwa
```

### 프로젝트 구조

```
MyBlazorWasmApp/
├── wwwroot/                 # 정적 파일
│   ├── index.html          # 진입점 HTML
│   ├── css/               # 스타일시트
│   ├── js/                # JavaScript 파일
│   └── sample-data/       # 샘플 데이터
├── Pages/                  # Razor 컴포넌트 페이지
├── Shared/                 # 공유 컴포넌트
├── _Imports.razor          # 전역 using 문
├── App.razor              # 루트 컴포넌트
├── Program.cs             # 진입점
└── MyBlazorWasmApp.csproj # 프로젝트 파일
```

### Program.cs 설정

```csharp
using Microsoft.AspNetCore.Components.Web;
using Microsoft.AspNetCore.Components.WebAssembly.Hosting;
using MyBlazorWasmApp;

var builder = WebAssemblyHostBuilder.CreateDefault(args);
builder.RootComponents.Add<App>("#app");
builder.RootComponents.Add<HeadOutlet>("head::after");

// HttpClient 등록
builder.Services.AddScoped(sp => new HttpClient 
{ 
    BaseAddress = new Uri(builder.HostEnvironment.BaseAddress) 
});

// 추가 서비스 등록
builder.Services.AddScoped<IWeatherService, WeatherService>();

await builder.Build().RunAsync();
```

## HTTP 통신

### HttpClient 사용

```csharp
// Services/WeatherService.cs
public interface IWeatherService
{
    Task<WeatherForecast[]> GetForecastAsync();
}

public class WeatherService : IWeatherService
{
    private readonly HttpClient _httpClient;
    
    public WeatherService(HttpClient httpClient)
    {
        _httpClient = httpClient;
    }
    
    public async Task<WeatherForecast[]> GetForecastAsync()
    {
        return await _httpClient.GetFromJsonAsync<WeatherForecast[]>("sample-data/weather.json")
            ?? Array.Empty<WeatherForecast>();
    }
}
```

### API 호출

```razor
@page "/weather"
@inject IWeatherService WeatherService

<h3>Weather Forecast</h3>

@if (forecasts == null)
{
    <p><em>Loading...</em></p>
}
else
{
    <table class="table">
        <thead>
            <tr>
                <th>Date</th>
                <th>Temp. (C)</th>
                <th>Summary</th>
            </tr>
        </thead>
        <tbody>
            @foreach (var forecast in forecasts)
            {
                <tr>
                    <td>@forecast.Date.ToShortDateString()</td>
                    <td>@forecast.TemperatureC</td>
                    <td>@forecast.Summary</td>
                </tr>
            }
        </tbody>
    </table>
}

@code {
    private WeatherForecast[]? forecasts;
    
    protected override async Task OnInitializedAsync()
    {
        try
        {
            forecasts = await WeatherService.GetForecastAsync();
        }
        catch (HttpRequestException ex)
        {
            // 에러 처리
            Console.WriteLine($"Error loading weather data: {ex.Message}");
        }
    }
}
```

### 인증된 API 호출

```csharp
// Services/AuthorizedApiService.cs
public class AuthorizedApiService
{
    private readonly HttpClient _httpClient;
    private readonly IAccessTokenProvider _tokenProvider;
    
    public AuthorizedApiService(HttpClient httpClient, IAccessTokenProvider tokenProvider)
    {
        _httpClient = httpClient;
        _tokenProvider = tokenProvider;
    }
    
    public async Task<T?> GetAsync<T>(string endpoint)
    {
        var tokenResult = await _tokenProvider.RequestAccessToken();
        
        if (tokenResult.TryGetToken(out var token))
        {
            _httpClient.DefaultRequestHeaders.Authorization = 
                new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", token.Value);
                
            return await _httpClient.GetFromJsonAsync<T>(endpoint);
        }
        
        throw new UnauthorizedAccessException("Unable to acquire access token");
    }
}
```

## 라우팅

### 기본 라우팅

```razor
@page "/products"
@page "/items"  @* 여러 경로 지원 *@

<h3>Products</h3>

@code {
    // 컴포넌트 로직
}
```

### 매개변수가 있는 라우팅

```razor
@page "/product/{id:int}"
@page "/product/{id:int}/{category}"

<h3>Product Details</h3>
<p>ID: @Id</p>
<p>Category: @(Category ?? "All")</p>

@code {
    [Parameter]
    public int Id { get; set; }
    
    [Parameter]
    public string? Category { get; set; }
}
```

### 쿼리 문자열 처리

```razor
@page "/search"
@inject NavigationManager Navigation

<h3>Search Results</h3>
<p>Query: @Query</p>
<p>Page: @Page</p>

@code {
    [SupplyParameterFromQuery]
    public string Query { get; set; } = "";
    
    [SupplyParameterFromQuery(Name = "p")]
    public int Page { get; set; } = 1;
}
```

### 네비게이션 가드

```razor
@implements IDisposable
@inject NavigationManager Navigation

@code {
    protected override void OnInitialized()
    {
        Navigation.LocationChanged += HandleLocationChanged;
    }
    
    private void HandleLocationChanged(object? sender, LocationChangedEventArgs e)
    {
        // 네비게이션 발생 시 처리
        Console.WriteLine($"Navigated to: {e.Location}");
    }
    
    public void Dispose()
    {
        Navigation.LocationChanged -= HandleLocationChanged;
    }
}
```

## 상태 관리

### 로컬 스토리지 활용

```csharp
// Services/LocalStorageService.cs
public interface ILocalStorageService
{
    Task<T?> GetItemAsync<T>(string key);
    Task SetItemAsync<T>(string key, T value);
    Task RemoveItemAsync(string key);
}

public class LocalStorageService : ILocalStorageService
{
    private readonly IJSRuntime _jsRuntime;
    
    public LocalStorageService(IJSRuntime jsRuntime)
    {
        _jsRuntime = jsRuntime;
    }
    
    public async Task<T?> GetItemAsync<T>(string key)
    {
        var json = await _jsRuntime.InvokeAsync<string>("localStorage.getItem", key);
        return json == null ? default : JsonSerializer.Deserialize<T>(json);
    }
    
    public async Task SetItemAsync<T>(string key, T value)
    {
        await _jsRuntime.InvokeVoidAsync("localStorage.setItem", key, JsonSerializer.Serialize(value));
    }
    
    public async Task RemoveItemAsync(string key)
    {
        await _jsRuntime.InvokeVoidAsync("localStorage.removeItem", key);
    }
}
```

### 전역 상태 관리

```csharp
// State/AppState.cs
public class AppState
{
    private string _userName = "";
    
    public string UserName
    {
        get => _userName;
        set
        {
            _userName = value;
            NotifyStateChanged();
        }
    }
    
    public event Action? OnChange;
    
    private void NotifyStateChanged() => OnChange?.Invoke();
}

// Program.cs
builder.Services.AddScoped<AppState>();

// Component 사용
@inject AppState AppState
@implements IDisposable

<p>Welcome, @AppState.UserName!</p>

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

### 상태 지속성

```csharp
// Services/PersistentStateService.cs
public class PersistentStateService
{
    private readonly ILocalStorageService _localStorage;
    
    public PersistentStateService(ILocalStorageService localStorage)
    {
        _localStorage = localStorage;
    }
    
    public async Task SaveStateAsync<T>(string key, T state)
    {
        await _localStorage.SetItemAsync(key, state);
    }
    
    public async Task<T?> LoadStateAsync<T>(string key)
    {
        return await _localStorage.GetItemAsync<T>(key);
    }
    
    public async Task<T> LoadOrCreateStateAsync<T>(string key) where T : new()
    {
        var state = await LoadStateAsync<T>(key);
        return state ?? new T();
    }
}
```

## 인증과 권한

### OIDC 인증 설정

```csharp
// Program.cs
builder.Services.AddOidcAuthentication(options =>
{
    builder.Configuration.Bind("Local", options.ProviderOptions);
    options.ProviderOptions.DefaultScopes.Add("api");
});

// appsettings.json
{
  "Local": {
    "Authority": "https://localhost:5001",
    "ClientId": "blazor-client",
    "ResponseType": "code",
    "DefaultScopes": [
      "openid",
      "profile"
    ],
    "PostLogoutRedirectUri": "https://localhost:5002/authentication/logout-callback",
    "RedirectUri": "https://localhost:5002/authentication/login-callback"
  }
}
```

### 인증 컴포넌트

```razor
@page "/authentication/{action}"
@using Microsoft.AspNetCore.Components.WebAssembly.Authentication

<RemoteAuthenticatorView Action="@Action" />

@code {
    [Parameter]
    public string Action { get; set; } = "";
}
```

### 권한 기반 렌더링

```razor
<CascadingAuthenticationState>
    <Router AppAssembly="@typeof(App).Assembly">
        <Found Context="routeData">
            <AuthorizeRouteView RouteData="@routeData" DefaultLayout="@typeof(MainLayout)">
                <NotAuthorized>
                    <p>You are not authorized to view this page.</p>
                    <a href="authentication/login">Log in</a>
                </NotAuthorized>
                <Authorizing>
                    <p>Checking authorization...</p>
                </Authorizing>
            </AuthorizeRouteView>
        </Found>
        <NotFound>
            <PageTitle>Not found</PageTitle>
            <LayoutView Layout="@typeof(MainLayout)">
                <p>Sorry, there's nothing at this address.</p>
            </LayoutView>
        </NotFound>
    </Router>
</CascadingAuthenticationState>
```

### 권한 있는 페이지

```razor
@page "/admin"
@attribute [Authorize(Roles = "Admin")]

<h3>Admin Dashboard</h3>

<AuthorizeView>
    <Authorized>
        <p>Welcome, @context.User.Identity?.Name!</p>
        <!-- 관리자 콘텐츠 -->
    </Authorized>
</AuthorizeView>
```

## Progressive Web App (PWA)

### PWA 설정

```json
// wwwroot/manifest.json
{
  "name": "My Blazor PWA",
  "short_name": "BlazorPWA",
  "start_url": "./",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#03173d",
  "prefer_related_applications": false,
  "icons": [
    {
      "src": "icon-512.png",
      "type": "image/png",
      "sizes": "512x512"
    },
    {
      "src": "icon-192.png",
      "type": "image/png",
      "sizes": "192x192"
    }
  ]
}
```

### Service Worker

```javascript
// wwwroot/service-worker.js
self.addEventListener('install', event => event.waitUntil(onInstall(event)));
self.addEventListener('activate', event => event.waitUntil(onActivate(event)));
self.addEventListener('fetch', event => event.respondWith(onFetch(event)));

const cacheNamePrefix = 'offline-cache-';
const cacheName = `${cacheNamePrefix}${self.assetsManifest.version}`;
const offlineAssetsInclude = [ /\.dll$/, /\.pdb$/, /\.wasm/, /\.html/, /\.js$/, /\.json$/, /\.css$/, /\.woff$/, /\.png$/, /\.jpe?g$/, /\.gif$/, /\.ico$/, /\.blat$/, /\.dat$/ ];
const offlineAssetsExclude = [ /^service-worker\.js$/ ];

async function onInstall(event) {
    console.info('Service worker: Install');
    
    const assetsRequests = self.assetsManifest.assets
        .filter(asset => offlineAssetsInclude.some(pattern => pattern.test(asset.url)))
        .filter(asset => !offlineAssetsExclude.some(pattern => pattern.test(asset.url)))
        .map(asset => new Request(asset.url, { integrity: asset.hash, cache: 'no-cache' }));
        
    await caches.open(cacheName).then(cache => cache.addAll(assetsRequests));
}

async function onActivate(event) {
    console.info('Service worker: Activate');
    
    const cacheWhitelist = [cacheName];
    await caches.keys().then(cacheNames => {
        return Promise.all(
            cacheNames.map(cacheName => {
                if (cacheWhitelist.indexOf(cacheName) === -1) {
                    return caches.delete(cacheName);
                }
            })
        );
    });
}

async function onFetch(event) {
    let cachedResponse = null;
    
    if (event.request.method === 'GET') {
        const shouldServeIndexHtml = event.request.mode === 'navigate';
        const request = shouldServeIndexHtml ? 'index.html' : event.request;
        const cache = await caches.open(cacheName);
        cachedResponse = await cache.match(request);
    }
    
    return cachedResponse || fetch(event.request);
}
```

### 오프라인 지원

```razor
@inject IJSRuntime JS

<h3>Network Status</h3>
<p>Status: @(isOnline ? "Online" : "Offline")</p>

@code {
    private bool isOnline = true;
    
    protected override async Task OnInitializedAsync()
    {
        isOnline = await JS.InvokeAsync<bool>("navigator.onLine");
        
        await JS.InvokeVoidAsync("blazorRegisterOnlineStatusCallback", 
            DotNetObjectReference.Create(this));
    }
    
    [JSInvokable]
    public void UpdateOnlineStatus(bool online)
    {
        isOnline = online;
        InvokeAsync(StateHasChanged);
    }
}
```

## JavaScript 상호 운용성

### JavaScript 함수 호출

```csharp
// Services/JsInteropService.cs
public class JsInteropService
{
    private readonly IJSRuntime _jsRuntime;
    
    public JsInteropService(IJSRuntime jsRuntime)
    {
        _jsRuntime = jsRuntime;
    }
    
    public async Task<string> GetBrowserInfo()
    {
        return await _jsRuntime.InvokeAsync<string>("getBrowserInfo");
    }
    
    public async Task ShowNotification(string title, string message)
    {
        await _jsRuntime.InvokeVoidAsync("showNotification", title, message);
    }
    
    public async Task<T> InvokeAsync<T>(string identifier, params object[] args)
    {
        return await _jsRuntime.InvokeAsync<T>(identifier, args);
    }
}
```

### JavaScript 모듈 사용

```csharp
// Services/ChartService.cs
public class ChartService : IAsyncDisposable
{
    private readonly IJSRuntime _jsRuntime;
    private IJSObjectReference? _module;
    
    public ChartService(IJSRuntime jsRuntime)
    {
        _jsRuntime = jsRuntime;
    }
    
    public async Task InitializeAsync()
    {
        _module = await _jsRuntime.InvokeAsync<IJSObjectReference>(
            "import", "./js/chart.js");
    }
    
    public async Task CreateChart(string elementId, object data)
    {
        if (_module != null)
        {
            await _module.InvokeVoidAsync("createChart", elementId, data);
        }
    }
    
    public async ValueTask DisposeAsync()
    {
        if (_module != null)
        {
            await _module.DisposeAsync();
        }
    }
}
```

### .NET 메서드 호출

```razor
@page "/js-callback"
@inject IJSRuntime JS

<button @onclick="RegisterCallback">Register Callback</button>

@code {
    private DotNetObjectReference<JsCallback>? objRef;
    
    protected override void OnInitialized()
    {
        objRef = DotNetObjectReference.Create(this);
    }
    
    private async Task RegisterCallback()
    {
        await JS.InvokeVoidAsync("registerDotNetCallback", objRef);
    }
    
    [JSInvokable]
    public void ReceiveMessage(string message)
    {
        Console.WriteLine($"Received from JS: {message}");
    }
    
    public void Dispose()
    {
        objRef?.Dispose();
    }
}
```

## 성능 최적화

### 지연 로딩

```razor
@* App.razor *@
<Router AppAssembly="@typeof(App).Assembly"
        AdditionalAssemblies="@lazyLoadedAssemblies">
    ...
</Router>

@code {
    private List<Assembly> lazyLoadedAssemblies = new();
    
    private async Task OnNavigateAsync(NavigationContext context)
    {
        if (context.Path.StartsWith("/admin"))
        {
            var assemblies = await AssemblyLoader.LoadAssembliesAsync(
                new[] { "Admin.dll" });
            lazyLoadedAssemblies.AddRange(assemblies);
        }
    }
}
```

### 가상화

```razor
<div style="height:500px; overflow-y:scroll">
    <Virtualize Items="@items" Context="item" ItemSize="50">
        <ItemContent>
            <div class="item">
                <h4>@item.Name</h4>
                <p>@item.Description</p>
            </div>
        </ItemContent>
        <Placeholder>
            <div class="item-placeholder">
                <div class="placeholder-animation"></div>
            </div>
        </Placeholder>
    </Virtualize>
</div>

@code {
    private List<Item> items = GenerateItems(10000);
    
    private static List<Item> GenerateItems(int count)
    {
        return Enumerable.Range(1, count)
            .Select(i => new Item 
            { 
                Name = $"Item {i}", 
                Description = $"Description {i}" 
            })
            .ToList();
    }
}
```

### 사전 렌더링

```html
<!-- index.html -->
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8" />
    <title>My Blazor App</title>
    <base href="/" />
    <link href="css/app.css" rel="stylesheet" />
</head>
<body>
    <div id="app">
        <!-- 로딩 중 표시 -->
        <div class="loading">
            <div class="spinner"></div>
            <p>Loading...</p>
        </div>
    </div>
    
    <script src="_framework/blazor.webassembly.js"></script>
</body>
</html>
```

### 컴파일 최적화

```xml
<!-- .csproj 파일 -->
<PropertyGroup>
    <!-- IL 링킹 활성화 -->
    <PublishTrimmed>true</PublishTrimmed>
    
    <!-- AOT 컴파일 (실험적) -->
    <RunAOTCompilation>true</RunAOTCompilation>
    
    <!-- 압축 활성화 -->
    <BlazorEnableCompression>true</BlazorEnableCompression>
</PropertyGroup>
```

## 파일 처리

### 파일 업로드

```razor
@page "/upload"
@inject HttpClient Http

<h3>File Upload</h3>

<InputFile OnChange="@LoadFiles" multiple />

@if (loadedFiles.Count > 0)
{
    <h4>Loaded Files:</h4>
    <ul>
        @foreach (var file in loadedFiles)
        {
            <li>
                @file.Name - @file.Size bytes
                <button @onclick="() => UploadFile(file)">Upload</button>
            </li>
        }
    </ul>
}

@code {
    private List<IBrowserFile> loadedFiles = new();
    private long maxFileSize = 1024 * 1024 * 10; // 10MB
    
    private void LoadFiles(InputFileChangeEventArgs e)
    {
        loadedFiles.Clear();
        
        foreach (var file in e.GetMultipleFiles())
        {
            loadedFiles.Add(file);
        }
    }
    
    private async Task UploadFile(IBrowserFile file)
    {
        try
        {
            using var content = new MultipartFormDataContent();
            using var fileStream = file.OpenReadStream(maxFileSize);
            using var streamContent = new StreamContent(fileStream);
            
            content.Add(streamContent, "file", file.Name);
            
            var response = await Http.PostAsync("/api/upload", content);
            
            if (response.IsSuccessStatusCode)
            {
                Console.WriteLine("File uploaded successfully");
            }
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Upload failed: {ex.Message}");
        }
    }
}
```

### 파일 다운로드

```csharp
// Services/FileDownloadService.cs
public class FileDownloadService
{
    private readonly HttpClient _httpClient;
    private readonly IJSRuntime _jsRuntime;
    
    public FileDownloadService(HttpClient httpClient, IJSRuntime jsRuntime)
    {
        _httpClient = httpClient;
        _jsRuntime = jsRuntime;
    }
    
    public async Task DownloadFileAsync(string url, string fileName)
    {
        var response = await _httpClient.GetAsync(url);
        var fileBytes = await response.Content.ReadAsByteArrayAsync();
        
        await _jsRuntime.InvokeVoidAsync(
            "saveAsFile", 
            fileName, 
            Convert.ToBase64String(fileBytes));
    }
}

// JavaScript 함수
// wwwroot/js/download.js
window.saveAsFile = (filename, bytesBase64) => {
    const link = document.createElement('a');
    link.download = filename;
    link.href = 'data:application/octet-stream;base64,' + bytesBase64;
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
};
```

## 테스트

### 단위 테스트

```csharp
// Tests/ComponentTests.cs
public class CounterTests : TestContext
{
    [Fact]
    public void CounterShouldIncrementWhenClicked()
    {
        // Arrange
        var cut = RenderComponent<Counter>();
        
        // Act
        cut.Find("button").Click();
        
        // Assert
        Assert.Equal("Current count: 1", cut.Find("p").TextContent);
    }
    
    [Fact]
    public void WeatherComponentShouldShowLoading()
    {
        // Arrange
        var mockHttp = Services.AddMockHttpClient();
        mockHttp.When("/sample-data/weather.json")
                .RespondJson(new WeatherForecast[] { });
                
        // Act
        var cut = RenderComponent<FetchData>();
        
        // Assert
        Assert.Contains("Loading...", cut.Markup);
    }
}
```

### 서비스 테스트

```csharp
public class WeatherServiceTests
{
    [Fact]
    public async Task GetForecastAsync_ReturnsWeatherData()
    {
        // Arrange
        var mockHttp = new MockHttpMessageHandler();
        var forecasts = new[] 
        { 
            new WeatherForecast { Date = DateTime.Now, TemperatureC = 25 } 
        };
        
        mockHttp.When("*/weather.json")
                .RespondJson(forecasts);
                
        var httpClient = mockHttp.ToHttpClient();
        httpClient.BaseAddress = new Uri("https://localhost/");
        
        var service = new WeatherService(httpClient);
        
        // Act
        var result = await service.GetForecastAsync();
        
        // Assert
        Assert.Single(result);
        Assert.Equal(25, result[0].TemperatureC);
    }
}
```

## 배포

### 정적 웹 호스팅

```bash
# 빌드
dotnet publish -c Release

# 출력 폴더
# bin/Release/net7.0/publish/wwwroot/

# nginx 설정
server {
    listen 80;
    server_name example.com;
    root /var/www/blazor;
    index index.html;
    
    location / {
        try_files $uri $uri/ /index.html;
    }
    
    # 압축 설정
    gzip on;
    gzip_types text/css application/javascript application/wasm;
}
```

### ASP.NET Core 호스팅

```csharp
// Server/Program.cs (Hosted Blazor WebAssembly)
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllersWithViews();
builder.Services.AddRazorPages();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseWebAssemblyDebugging();
}
else
{
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseBlazorFrameworkFiles();
app.UseStaticFiles();
app.UseRouting();

app.MapRazorPages();
app.MapControllers();
app.MapFallbackToFile("index.html");

app.Run();
```

### Azure Static Web Apps

```yaml
# .github/workflows/azure-static-web-apps.yml
name: Azure Static Web Apps CI/CD

on:
  push:
    branches: [ main ]

jobs:
  build_and_deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Setup .NET
      uses: actions/setup-dotnet@v1
      with:
        dotnet-version: 7.0.x
        
    - name: Build
      run: dotnet publish -c Release -o ${{env.DOTNET_ROOT}}/myapp
      
    - name: Deploy
      uses: Azure/static-web-apps-deploy@v1
      with:
        azure_static_web_apps_api_token: ${{ secrets.AZURE_STATIC_WEB_APPS_API_TOKEN }}
        repo_token: ${{ secrets.GITHUB_TOKEN }}
        action: "upload"
        app_location: "${{env.DOTNET_ROOT}}/myapp/wwwroot"
        output_location: ""
```

## 모범 사례

### 컴포넌트 설계

1. **작은 컴포넌트**: 단일 책임 원칙 준수
2. **재사용 가능한 컴포넌트**: 범용적 설계
3. **명확한 인터페이스**: Parameter와 EventCallback 활용
4. **상태 최소화**: 필요한 상태만 유지

### 성능 최적화

1. **지연 로딩**: 큰 어셈블리는 필요할 때 로드
2. **가상화**: 큰 목록은 Virtualize 컴포넌트 사용
3. **캐싱**: 자주 사용하는 데이터는 로컬 스토리지에 캐시
4. **번들 크기 최소화**: 불필요한 종속성 제거

### 보안

1. **API 키 보호**: 클라이언트에 민감한 정보 노출 금지
2. **입력 검증**: 모든 사용자 입력 검증
3. **HTTPS 사용**: 항상 안전한 연결 사용
4. **CORS 설정**: 적절한 CORS 정책 설정

### 접근성

1. **시맨틱 HTML**: 적절한 HTML 요소 사용
2. **ARIA 속성**: 필요한 경우 ARIA 속성 추가
3. **키보드 탐색**: 모든 기능이 키보드로 접근 가능
4. **색상 대비**: WCAG 기준 준수

## 마무리

Blazor WebAssembly는 .NET 개발자가 풀스택 웹 애플리케이션을 구축할 수 있는 강력한 프레임워크입니다. C#과 .NET의 강력한 기능을 브라우저에서 직접 활용할 수 있으며, 기존 .NET 생태계와 완벽하게 통합됩니다.

주요 장점:
- 진정한 클라이언트 측 실행
- 오프라인 지원 (PWA)
- 정적 호스팅 가능
- 풍부한 .NET 생태계 활용
- 강력한 개발 도구 지원

고려사항:
- 초기 다운로드 크기가 큼
- 구형 브라우저 미지원
- 디버깅이 상대적으로 어려움
- SEO 최적화 필요

Blazor WebAssembly는 특히 기업용 웹 애플리케이션, 관리 도구, 오프라인 지원이 필요한 애플리케이션에 적합하며, .NET 개발자가 JavaScript 없이도 현대적인 웹 애플리케이션을 구축할 수 있게 해줍니다.