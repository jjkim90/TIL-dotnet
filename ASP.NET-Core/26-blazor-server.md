# Blazor Server

## Blazor 소개

Blazor는 Microsoft에서 개발한 웹 UI 프레임워크로, C#을 사용하여 대화형 웹 애플리케이션을 구축할 수 있습니다. JavaScript 대신 C#을 사용하여 클라이언트 측 로직을 작성할 수 있어, .NET 개발자들이 풀스택 웹 개발을 할 수 있게 해줍니다.

### Blazor의 호스팅 모델

Blazor는 두 가지 호스팅 모델을 제공합니다:

1. **Blazor Server**: UI 업데이트와 이벤트 처리가 서버에서 실행
2. **Blazor WebAssembly**: 브라우저에서 .NET 런타임이 실행

이 문서에서는 Blazor Server에 집중합니다.

### Blazor Server의 특징

- **실시간 연결**: SignalR을 통한 서버-클라이언트 연결
- **빠른 초기 로드**: 작은 다운로드 크기
- **서버 리소스 활용**: 클라이언트 장치 성능에 덜 의존적
- **완전한 .NET API 액세스**: 모든 .NET 기능 사용 가능
- **향상된 보안**: 앱 코드가 클라이언트에 다운로드되지 않음

## Blazor Server 프로젝트 생성

### .NET CLI를 사용한 프로젝트 생성

```bash
# Blazor Server 앱 생성
dotnet new blazorserver -n MyBlazorServerApp

# 프로젝트 실행
cd MyBlazorServerApp
dotnet run
```

### 프로젝트 구조

```
MyBlazorServerApp/
├── Data/                    # 데이터 서비스
├── Pages/                   # Razor 컴포넌트 페이지
├── Shared/                  # 공유 컴포넌트
├── wwwroot/                 # 정적 파일
├── _Imports.razor           # 전역 using 문
├── App.razor               # 루트 컴포넌트
├── appsettings.json        # 앱 설정
└── Program.cs              # 진입점
```

### Program.cs 설정

```csharp
var builder = WebApplication.CreateBuilder(args);

// Razor Pages와 Blazor Server 서비스 추가
builder.Services.AddRazorPages();
builder.Services.AddServerSideBlazor();

// 사용자 정의 서비스 등록
builder.Services.AddSingleton<WeatherForecastService>();

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();

app.MapBlazorHub();
app.MapFallbackToPage("/_Host");

app.Run();
```

## Razor 컴포넌트 기초

### 컴포넌트 생성

```razor
@* Counter.razor *@
@page "/counter"

<h3>Counter</h3>

<p>Current count: @currentCount</p>

<button class="btn btn-primary" @onclick="IncrementCount">Click me</button>

@code {
    private int currentCount = 0;

    private void IncrementCount()
    {
        currentCount++;
    }
}
```

### 컴포넌트 매개변수

```razor
@* ChildComponent.razor *@
<div class="alert alert-info">
    <h4>@Title</h4>
    <p>@ChildContent</p>
</div>

@code {
    [Parameter]
    public string Title { get; set; } = "Default Title";
    
    [Parameter]
    public RenderFragment? ChildContent { get; set; }
}

@* ParentComponent.razor *@
@page "/parent"

<ChildComponent Title="Welcome!">
    This is the child content.
</ChildComponent>
```

### 이벤트 콜백

```razor
@* ChildComponent.razor *@
<button @onclick="OnButtonClick">Click me</button>

@code {
    [Parameter]
    public EventCallback<string> OnClick { get; set; }
    
    private async Task OnButtonClick()
    {
        await OnClick.InvokeAsync("Button clicked!");
    }
}

@* ParentComponent.razor *@
<ChildComponent OnClick="HandleClick" />

<p>Message: @message</p>

@code {
    private string message = "";
    
    private void HandleClick(string msg)
    {
        message = msg;
    }
}
```

## 라우팅과 네비게이션

### 라우트 매개변수

```razor
@page "/product/{id:int}"

<h3>Product Details</h3>
<p>Product ID: @Id</p>

@code {
    [Parameter]
    public int Id { get; set; }
}
```

### 쿼리 문자열

```razor
@page "/search"
@inject NavigationManager Navigation

<h3>Search Results</h3>
<p>Query: @SearchQuery</p>

@code {
    private string SearchQuery { get; set; } = "";
    
    protected override void OnInitialized()
    {
        var uri = Navigation.ToAbsoluteUri(Navigation.Uri);
        var query = System.Web.HttpUtility.ParseQueryString(uri.Query);
        SearchQuery = query["q"] ?? "";
    }
}
```

### 프로그래밍 방식의 네비게이션

```razor
@inject NavigationManager Navigation

<button @onclick="NavigateToCounter">Go to Counter</button>

@code {
    private void NavigateToCounter()
    {
        Navigation.NavigateTo("/counter");
    }
}
```

## 데이터 바인딩

### 단방향 바인딩

```razor
<h3>@title</h3>
<p>Count: @count</p>

@code {
    private string title = "My Component";
    private int count = 0;
}
```

### 양방향 바인딩

```razor
<input @bind="searchText" @bind:event="oninput" />
<p>You typed: @searchText</p>

<select @bind="selectedOption">
    <option value="">Select an option</option>
    <option value="1">Option 1</option>
    <option value="2">Option 2</option>
</select>

@code {
    private string searchText = "";
    private string selectedOption = "";
}
```

### 커스텀 바인딩

```razor
<input value="@Value" @onchange="HandleChange" />

@code {
    [Parameter]
    public string Value { get; set; } = "";
    
    [Parameter]
    public EventCallback<string> ValueChanged { get; set; }
    
    private async Task HandleChange(ChangeEventArgs e)
    {
        Value = e.Value?.ToString() ?? "";
        await ValueChanged.InvokeAsync(Value);
    }
}
```

## 폼과 검증

### EditForm 컴포넌트

```razor
@page "/register"
@using System.ComponentModel.DataAnnotations

<h3>User Registration</h3>

<EditForm Model="@model" OnValidSubmit="HandleValidSubmit">
    <DataAnnotationsValidator />
    <ValidationSummary />
    
    <div class="form-group">
        <label>이름:</label>
        <InputText @bind-Value="model.Name" class="form-control" />
        <ValidationMessage For="@(() => model.Name)" />
    </div>
    
    <div class="form-group">
        <label>이메일:</label>
        <InputText @bind-Value="model.Email" class="form-control" />
        <ValidationMessage For="@(() => model.Email)" />
    </div>
    
    <div class="form-group">
        <label>나이:</label>
        <InputNumber @bind-Value="model.Age" class="form-control" />
        <ValidationMessage For="@(() => model.Age)" />
    </div>
    
    <button type="submit" class="btn btn-primary">등록</button>
</EditForm>

@code {
    private UserModel model = new();
    
    private void HandleValidSubmit()
    {
        // 유효한 폼 제출 처리
        Console.WriteLine($"User registered: {model.Name}");
    }
    
    public class UserModel
    {
        [Required(ErrorMessage = "이름은 필수입니다.")]
        [StringLength(50, MinimumLength = 2)]
        public string Name { get; set; } = "";
        
        [Required(ErrorMessage = "이메일은 필수입니다.")]
        [EmailAddress(ErrorMessage = "유효한 이메일 주소를 입력하세요.")]
        public string Email { get; set; } = "";
        
        [Range(18, 100, ErrorMessage = "나이는 18-100 사이여야 합니다.")]
        public int Age { get; set; }
    }
}
```

### 커스텀 검증

```csharp
public class CustomValidator : ValidationAttribute
{
    protected override ValidationResult? IsValid(
        object? value, 
        ValidationContext validationContext)
    {
        if (value is string str && str.Contains("admin"))
        {
            return new ValidationResult("'admin'은 사용할 수 없습니다.");
        }
        
        return ValidationResult.Success;
    }
}

public class UserModel
{
    [CustomValidator]
    public string Username { get; set; } = "";
}
```

## 상태 관리

### 컴포넌트 상태

```razor
@implements IDisposable
@inject StateContainer StateContainer

<h3>Component State</h3>
<p>Shared Value: @StateContainer.Value</p>
<button @onclick="UpdateValue">Update</button>

@code {
    protected override void OnInitialized()
    {
        StateContainer.OnChange += StateHasChanged;
    }
    
    private void UpdateValue()
    {
        StateContainer.Value = DateTime.Now.ToString();
    }
    
    public void Dispose()
    {
        StateContainer.OnChange -= StateHasChanged;
    }
}
```

### 상태 컨테이너 서비스

```csharp
// Services/StateContainer.cs
public class StateContainer
{
    private string _value = "";
    
    public string Value
    {
        get => _value;
        set
        {
            _value = value;
            NotifyStateChanged();
        }
    }
    
    public event Action? OnChange;
    
    private void NotifyStateChanged() => OnChange?.Invoke();
}

// Program.cs
builder.Services.AddScoped<StateContainer>();
```

### 캐스케이딩 값

```razor
@* App.razor *@
<CascadingValue Value="userTheme">
    <Router AppAssembly="@typeof(App).Assembly">
        ...
    </Router>
</CascadingValue>

@code {
    private ThemeInfo userTheme = new() { Name = "Dark" };
}

@* ChildComponent.razor *@
<h3>Current Theme: @Theme?.Name</h3>

@code {
    [CascadingParameter]
    protected ThemeInfo? Theme { get; set; }
}
```

## 의존성 주입

### 서비스 등록

```csharp
// Services/IWeatherService.cs
public interface IWeatherService
{
    Task<WeatherForecast[]> GetForecastAsync();
}

// Services/WeatherService.cs
public class WeatherService : IWeatherService
{
    private static readonly string[] Summaries = new[]
    {
        "Freezing", "Bracing", "Chilly", "Cool", "Mild",
        "Warm", "Balmy", "Hot", "Sweltering", "Scorching"
    };
    
    public Task<WeatherForecast[]> GetForecastAsync()
    {
        var forecast = Enumerable.Range(1, 5).Select(index => new WeatherForecast
        {
            Date = DateTime.Now.AddDays(index),
            TemperatureC = Random.Shared.Next(-20, 55),
            Summary = Summaries[Random.Shared.Next(Summaries.Length)]
        }).ToArray();
        
        return Task.FromResult(forecast);
    }
}

// Program.cs
builder.Services.AddScoped<IWeatherService, WeatherService>();
```

### 서비스 주입

```razor
@page "/weather"
@inject IWeatherService WeatherService

<h3>Weather Forecast</h3>

@if (forecasts == null)
{
    <p>Loading...</p>
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
        forecasts = await WeatherService.GetForecastAsync();
    }
}
```

## 라이프사이클 메서드

### 컴포넌트 라이프사이클

```razor
@implements IAsyncDisposable

<h3>Component Lifecycle</h3>

@code {
    // 1. 컴포넌트가 초기화될 때
    protected override void OnInitialized()
    {
        Console.WriteLine("OnInitialized");
    }
    
    protected override async Task OnInitializedAsync()
    {
        Console.WriteLine("OnInitializedAsync");
        await LoadDataAsync();
    }
    
    // 2. 매개변수가 설정된 후
    protected override void OnParametersSet()
    {
        Console.WriteLine("OnParametersSet");
    }
    
    protected override async Task OnParametersSetAsync()
    {
        Console.WriteLine("OnParametersSetAsync");
    }
    
    // 3. 렌더링 후
    protected override void OnAfterRender(bool firstRender)
    {
        if (firstRender)
        {
            Console.WriteLine("First render completed");
        }
    }
    
    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            await InitializeJavaScriptAsync();
        }
    }
    
    // 4. 컴포넌트 폐기
    public async ValueTask DisposeAsync()
    {
        Console.WriteLine("Component disposed");
        await CleanupResourcesAsync();
    }
    
    private Task LoadDataAsync() => Task.CompletedTask;
    private Task InitializeJavaScriptAsync() => Task.CompletedTask;
    private Task CleanupResourcesAsync() => Task.CompletedTask;
}
```

### 렌더링 제어

```razor
<h3>Render Control</h3>
<p>Count: @count</p>
<button @onclick="IncrementCount">Increment</button>

@code {
    private int count = 0;
    
    private void IncrementCount()
    {
        count++;
        
        // 조건부 렌더링
        if (count % 5 == 0)
        {
            StateHasChanged();
        }
    }
    
    protected override bool ShouldRender()
    {
        // 특정 조건에서만 렌더링
        return count % 2 == 0;
    }
}
```

## JavaScript 상호 운용

### JavaScript 호출

```razor
@inject IJSRuntime JS

<button @onclick="ShowAlert">Show Alert</button>
<button @onclick="GetWindowSize">Get Window Size</button>

<p>Window size: @windowSize</p>

@code {
    private string windowSize = "";
    
    private async Task ShowAlert()
    {
        await JS.InvokeVoidAsync("alert", "Hello from Blazor!");
    }
    
    private async Task GetWindowSize()
    {
        var size = await JS.InvokeAsync<WindowSize>("getWindowSize");
        windowSize = $"{size.Width} x {size.Height}";
    }
    
    public class WindowSize
    {
        public int Width { get; set; }
        public int Height { get; set; }
    }
}
```

### JavaScript 함수 정의

```javascript
// wwwroot/js/interop.js
window.getWindowSize = () => {
    return {
        width: window.innerWidth,
        height: window.innerHeight
    };
};

window.blazorHelpers = {
    setFocus: (element) => {
        element.focus();
    },
    
    saveAsFile: (filename, bytesBase64) => {
        const link = document.createElement('a');
        link.download = filename;
        link.href = "data:application/octet-stream;base64," + bytesBase64;
        document.body.appendChild(link);
        link.click();
        document.body.removeChild(link);
    }
};
```

### .NET에서 JavaScript 호출 받기

```csharp
// JavaScript에서 호출 가능한 .NET 메서드
[JSInvokable]
public static Task<string> GetCurrentTime()
{
    return Task.FromResult(DateTime.Now.ToString("yyyy-MM-dd HH:mm:ss"));
}

// 인스턴스 메서드
[JSInvokable]
public async Task UpdateData(string data)
{
    // 데이터 처리
    await ProcessDataAsync(data);
}
```

## 실시간 업데이트

### SignalR 연결 관리

```razor
@page "/chat"
@using Microsoft.AspNetCore.SignalR.Client
@implements IAsyncDisposable

<h3>Real-time Chat</h3>

<div class="form-group">
    <input @bind="messageInput" @onkeypress="@(async (e) => { if (e.Key == "Enter") await SendMessage(); })" />
    <button @onclick="SendMessage">Send</button>
</div>

<ul>
    @foreach (var message in messages)
    {
        <li>@message</li>
    }
</ul>

@code {
    private HubConnection? hubConnection;
    private List<string> messages = new();
    private string messageInput = "";
    
    protected override async Task OnInitializedAsync()
    {
        hubConnection = new HubConnectionBuilder()
            .WithUrl(Navigation.ToAbsoluteUri("/chathub"))
            .Build();
            
        hubConnection.On<string, string>("ReceiveMessage", (user, message) =>
        {
            messages.Add($"{user}: {message}");
            InvokeAsync(StateHasChanged);
        });
        
        await hubConnection.StartAsync();
    }
    
    private async Task SendMessage()
    {
        if (hubConnection is not null && !string.IsNullOrWhiteSpace(messageInput))
        {
            await hubConnection.SendAsync("SendMessage", "User", messageInput);
            messageInput = "";
        }
    }
    
    public async ValueTask DisposeAsync()
    {
        if (hubConnection is not null)
        {
            await hubConnection.DisposeAsync();
        }
    }
}
```

### SignalR Hub

```csharp
// Hubs/ChatHub.cs
public class ChatHub : Hub
{
    public async Task SendMessage(string user, string message)
    {
        await Clients.All.SendAsync("ReceiveMessage", user, message);
    }
    
    public override async Task OnConnectedAsync()
    {
        await Clients.Caller.SendAsync("ReceiveMessage", "System", "Connected to chat");
        await base.OnConnectedAsync();
    }
}

// Program.cs
builder.Services.AddSignalR();
app.MapHub<ChatHub>("/chathub");
```

## 성능 최적화

### 가상화

```razor
@page "/virtualization"

<h3>Virtualized List</h3>

<div style="height:500px; overflow-y:scroll">
    <Virtualize Items="@items" Context="item">
        <div class="item">
            <h4>@item.Name</h4>
            <p>@item.Description</p>
        </div>
    </Virtualize>
</div>

@code {
    private List<Item> items = Enumerable.Range(1, 10000)
        .Select(i => new Item
        {
            Name = $"Item {i}",
            Description = $"Description for item {i}"
        }).ToList();
        
    public class Item
    {
        public string Name { get; set; } = "";
        public string Description { get; set; } = "";
    }
}
```

### 지연 로딩

```razor
<Virtualize ItemsProvider="LoadItems" Context="item">
    <div>@item.Name</div>
</Virtualize>

@code {
    private async ValueTask<ItemsProviderResult<Item>> LoadItems(
        ItemsProviderRequest request)
    {
        // 데이터베이스에서 항목 로드
        var items = await LoadItemsFromDatabase(
            request.StartIndex, 
            request.Count);
            
        var totalCount = await GetTotalItemCount();
        
        return new ItemsProviderResult<Item>(items, totalCount);
    }
}
```

### 프리렌더링 제어

```razor
@* _Host.cshtml *@
<component type="typeof(App)" render-mode="ServerPrerendered" />

@* 또는 프리렌더링 비활성화 *@
<component type="typeof(App)" render-mode="Server" />
```

## 보안 고려사항

### 인증과 권한

```razor
@page "/admin"
@attribute [Authorize(Roles = "Admin")]

<h3>Admin Dashboard</h3>

<AuthorizeView>
    <Authorized>
        <p>Welcome, @context.User.Identity?.Name!</p>
        <!-- 관리자 콘텐츠 -->
    </Authorized>
    <NotAuthorized>
        <p>You are not authorized to view this page.</p>
    </NotAuthorized>
</AuthorizeView>
```

### CSRF 보호

```razor
@* Blazor Server는 기본적으로 CSRF 보호 제공 *@
<EditForm Model="@model" OnValidSubmit="HandleSubmit">
    <AntiforgeryToken />
    <!-- 폼 필드 -->
</EditForm>
```

### 안전한 데이터 처리

```csharp
// 사용자 입력 검증
public class SecureDataService
{
    public string SanitizeInput(string input)
    {
        // HTML 인코딩
        return System.Net.WebUtility.HtmlEncode(input);
    }
    
    public bool ValidateInput(string input)
    {
        // 입력 검증 로직
        return !string.IsNullOrWhiteSpace(input) && 
               input.Length <= 100 &&
               !ContainsMaliciousContent(input);
    }
}
```

## 배포 고려사항

### 연결 관리

```csharp
// Program.cs
builder.Services.Configure<CircuitOptions>(options =>
{
    options.DetailedErrors = false;
    options.DisconnectedCircuitRetentionPeriod = TimeSpan.FromMinutes(3);
    options.DisconnectedCircuitMaxRetained = 100;
    options.JSInteropDefaultCallTimeout = TimeSpan.FromMinutes(1);
    options.MaxBufferedUnacknowledgedRenderBatches = 10;
});
```

### 확장성 설정

```csharp
// SignalR 설정
builder.Services.AddSignalR(options =>
{
    options.MaximumReceiveMessageSize = 32 * 1024; // 32KB
    options.ClientTimeoutInterval = TimeSpan.FromSeconds(60);
    options.KeepAliveInterval = TimeSpan.FromSeconds(30);
});

// Azure SignalR Service 사용
builder.Services.AddSignalR()
    .AddAzureSignalR(builder.Configuration.GetConnectionString("AzureSignalR"));
```

### 상태 서버 구성

```csharp
// Redis를 사용한 분산 상태 저장
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
});

// SQL Server를 사용한 세션 상태
builder.Services.AddDistributedSqlServerCache(options =>
{
    options.ConnectionString = builder.Configuration.GetConnectionString("SqlServer");
    options.SchemaName = "dbo";
    options.TableName = "SessionState";
});
```

## 모범 사례

### 컴포넌트 설계

1. **단일 책임 원칙**: 각 컴포넌트는 하나의 목적만 가져야 함
2. **재사용성**: 범용적으로 사용할 수 있는 컴포넌트 설계
3. **매개변수 최소화**: 필요한 매개변수만 정의
4. **이벤트 콜백 활용**: 부모-자식 통신에 EventCallback 사용

### 성능 최적화

1. **가상화 사용**: 대량 데이터 표시 시 Virtualize 컴포넌트 활용
2. **불필요한 렌더링 방지**: ShouldRender 메서드 활용
3. **비동기 작업**: 긴 작업은 비동기로 처리
4. **상태 관리**: 적절한 범위의 상태 관리 서비스 사용

### 보안

1. **입력 검증**: 모든 사용자 입력 검증
2. **권한 확인**: 민감한 작업 전 권한 확인
3. **안전한 통신**: HTTPS 사용
4. **세션 관리**: 적절한 세션 타임아웃 설정

## 마무리

Blazor Server는 C# 개발자가 웹 애플리케이션을 구축하는 강력한 방법을 제공합니다. SignalR을 통한 실시간 연결로 반응성 높은 UI를 제공하면서도, 서버 측 처리의 이점을 활용할 수 있습니다.

주요 장점:
- 빠른 초기 로드 시간
- 완전한 .NET API 액세스
- 향상된 보안 (코드가 클라이언트에 노출되지 않음)
- 구형 브라우저 지원

고려사항:
- 네트워크 지연 시간에 민감
- 서버 리소스 사용량
- 오프라인 시나리오 미지원

Blazor Server는 인트라넷 애플리케이션, 관리 도구, 실시간 대시보드 등에 특히 적합하며, 기존 .NET 개발 경험을 웹 개발에 그대로 활용할 수 있는 훌륭한 선택입니다.