# 테스트 전략

## 개요

Blazor 애플리케이션의 품질을 보장하기 위해서는 체계적인 테스트 전략이 필수적입니다. 이 장에서는 단위 테스트, 통합 테스트, E2E 테스트, 컴포넌트 테스트 등 다양한 테스트 기법을 학습합니다.

## 1. 단위 테스트

### 1.1 컴포넌트 단위 테스트

```csharp
// Tests/Components/CounterComponentTests.cs
using Bunit;
using FluentAssertions;
using Microsoft.Extensions.DependencyInjection;
using Moq;
using Xunit;

public class CounterComponentTests : TestContext
{
    [Fact]
    public void Counter_ShouldStartAtZero()
    {
        // Arrange & Act
        var component = RenderComponent<Counter>();
        
        // Assert
        component.Find("p").TextContent.Should().Be("Current count: 0");
    }
    
    [Fact]
    public void Counter_ShouldIncrementWhenButtonClicked()
    {
        // Arrange
        var component = RenderComponent<Counter>();
        var button = component.Find("button");
        
        // Act
        button.Click();
        
        // Assert
        component.Find("p").TextContent.Should().Be("Current count: 1");
    }
    
    [Theory]
    [InlineData(1, 1)]
    [InlineData(5, 5)]
    [InlineData(10, 10)]
    public void Counter_ShouldIncrementBySpecifiedAmount(int clicks, int expectedCount)
    {
        // Arrange
        var component = RenderComponent<Counter>();
        var button = component.Find("button");
        
        // Act
        for (int i = 0; i < clicks; i++)
        {
            button.Click();
        }
        
        // Assert
        component.Find("p").TextContent.Should().Be($"Current count: {expectedCount}");
    }
}

// Tests/Components/TodoListComponentTests.cs
public class TodoListComponentTests : TestContext
{
    private readonly Mock<ITodoService> _mockTodoService;
    
    public TodoListComponentTests()
    {
        _mockTodoService = new Mock<ITodoService>();
        Services.AddSingleton(_mockTodoService.Object);
    }
    
    [Fact]
    public async Task TodoList_ShouldDisplayLoadingInitially()
    {
        // Arrange
        var tcs = new TaskCompletionSource<List<TodoItem>>();
        _mockTodoService.Setup(x => x.GetTodosAsync())
            .Returns(tcs.Task);
        
        // Act
        var component = RenderComponent<TodoList>();
        
        // Assert
        component.Find(".loading").Should().NotBeNull();
        
        // Complete loading
        tcs.SetResult(new List<TodoItem>());
        await Task.Delay(50); // Allow render to complete
        
        Assert.Throws<ElementNotFoundException>(() => component.Find(".loading"));
    }
    
    [Fact]
    public async Task TodoList_ShouldDisplayTodos()
    {
        // Arrange
        var todos = new List<TodoItem>
        {
            new() { Id = 1, Title = "Test 1", IsCompleted = false },
            new() { Id = 2, Title = "Test 2", IsCompleted = true }
        };
        
        _mockTodoService.Setup(x => x.GetTodosAsync())
            .ReturnsAsync(todos);
        
        // Act
        var component = RenderComponent<TodoList>();
        await Task.Delay(50); // Allow async operation to complete
        
        // Assert
        var todoItems = component.FindAll(".todo-item");
        todoItems.Should().HaveCount(2);
        
        todoItems[0].QuerySelector(".title")?.TextContent.Should().Be("Test 1");
        todoItems[1].QuerySelector(".title")?.TextContent.Should().Be("Test 2");
    }
    
    [Fact]
    public async Task TodoList_ShouldAddNewTodo()
    {
        // Arrange
        _mockTodoService.Setup(x => x.GetTodosAsync())
            .ReturnsAsync(new List<TodoItem>());
        
        _mockTodoService.Setup(x => x.CreateTodoAsync(It.IsAny<string>()))
            .ReturnsAsync(new TodoItem { Id = 1, Title = "New Todo" });
        
        var component = RenderComponent<TodoList>();
        
        // Act
        var input = component.Find("input[type='text']");
        await input.ChangeAsync(new ChangeEventArgs { Value = "New Todo" });
        
        var addButton = component.Find("button.add-todo");
        await addButton.ClickAsync();
        
        // Assert
        _mockTodoService.Verify(x => x.CreateTodoAsync("New Todo"), Times.Once);
    }
}
```

### 1.2 서비스 단위 테스트

```csharp
// Tests/Services/UserServiceTests.cs
public class UserServiceTests
{
    private readonly Mock<HttpMessageHandler> _mockHttpHandler;
    private readonly HttpClient _httpClient;
    private readonly Mock<ILogger<UserService>> _mockLogger;
    private readonly UserService _userService;
    
    public UserServiceTests()
    {
        _mockHttpHandler = new Mock<HttpMessageHandler>(MockBehavior.Strict);
        _httpClient = new HttpClient(_mockHttpHandler.Object)
        {
            BaseAddress = new Uri("https://api.example.com/")
        };
        _mockLogger = new Mock<ILogger<UserService>>();
        _userService = new UserService(_httpClient, _mockLogger.Object);
    }
    
    [Fact]
    public async Task GetUserAsync_ShouldReturnUser_WhenUserExists()
    {
        // Arrange
        var userId = 123;
        var expectedUser = new User { Id = userId, Name = "John Doe" };
        
        _mockHttpHandler.Protected()
            .Setup<Task<HttpResponseMessage>>(
                "SendAsync",
                ItExpr.IsAny<HttpRequestMessage>(),
                ItExpr.IsAny<CancellationToken>())
            .ReturnsAsync(new HttpResponseMessage
            {
                StatusCode = HttpStatusCode.OK,
                Content = new StringContent(JsonSerializer.Serialize(expectedUser))
            });
        
        // Act
        var result = await _userService.GetUserAsync(userId);
        
        // Assert
        result.Should().NotBeNull();
        result.Id.Should().Be(userId);
        result.Name.Should().Be("John Doe");
    }
    
    [Fact]
    public async Task GetUserAsync_ShouldReturnNull_WhenUserNotFound()
    {
        // Arrange
        var userId = 999;
        
        _mockHttpHandler.Protected()
            .Setup<Task<HttpResponseMessage>>(
                "SendAsync",
                ItExpr.IsAny<HttpRequestMessage>(),
                ItExpr.IsAny<CancellationToken>())
            .ReturnsAsync(new HttpResponseMessage
            {
                StatusCode = HttpStatusCode.NotFound
            });
        
        // Act
        var result = await _userService.GetUserAsync(userId);
        
        // Assert
        result.Should().BeNull();
    }
    
    [Fact]
    public async Task CreateUserAsync_ShouldReturnCreatedUser()
    {
        // Arrange
        var newUser = new CreateUserRequest { Name = "Jane Doe", Email = "jane@example.com" };
        var createdUser = new User { Id = 456, Name = "Jane Doe", Email = "jane@example.com" };
        
        _mockHttpHandler.Protected()
            .Setup<Task<HttpResponseMessage>>(
                "SendAsync",
                ItExpr.Is<HttpRequestMessage>(req =>
                    req.Method == HttpMethod.Post &&
                    req.RequestUri!.ToString().EndsWith("/users")),
                ItExpr.IsAny<CancellationToken>())
            .ReturnsAsync(new HttpResponseMessage
            {
                StatusCode = HttpStatusCode.Created,
                Content = new StringContent(JsonSerializer.Serialize(createdUser))
            });
        
        // Act
        var result = await _userService.CreateUserAsync(newUser);
        
        // Assert
        result.Should().NotBeNull();
        result.Id.Should().Be(456);
        result.Name.Should().Be("Jane Doe");
    }
}
```

## 2. 통합 테스트

### 2.1 WebApplicationFactory를 사용한 통합 테스트

```csharp
// Tests/Integration/CustomWebApplicationFactory.cs
public class CustomWebApplicationFactory<TStartup> : WebApplicationFactory<TStartup> 
    where TStartup : class
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // Remove real database
            var descriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<ApplicationDbContext>));
            
            if (descriptor != null)
            {
                services.Remove(descriptor);
            }
            
            // Add in-memory database for testing
            services.AddDbContext<ApplicationDbContext>(options =>
            {
                options.UseInMemoryDatabase("TestDb");
            });
            
            // Add test authentication
            services.AddAuthentication("Test")
                .AddScheme<TestAuthenticationSchemeOptions, TestAuthenticationHandler>(
                    "Test", options => { });
            
            // Build service provider
            var sp = services.BuildServiceProvider();
            
            // Create scope
            using var scope = sp.CreateScope();
            var scopedServices = scope.ServiceProvider;
            var db = scopedServices.GetRequiredService<ApplicationDbContext>();
            
            // Ensure database is created
            db.Database.EnsureCreated();
            
            // Seed test data
            SeedTestData(db);
        });
    }
    
    private void SeedTestData(ApplicationDbContext context)
    {
        context.Users.AddRange(
            new User { Id = 1, Name = "Test User 1", Email = "test1@example.com" },
            new User { Id = 2, Name = "Test User 2", Email = "test2@example.com" }
        );
        
        context.SaveChanges();
    }
}

// Tests/Integration/ApiIntegrationTests.cs
public class ApiIntegrationTests : IClassFixture<CustomWebApplicationFactory<Program>>
{
    private readonly CustomWebApplicationFactory<Program> _factory;
    private readonly HttpClient _client;
    
    public ApiIntegrationTests(CustomWebApplicationFactory<Program> factory)
    {
        _factory = factory;
        _client = _factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                // Override services if needed
            });
        }).CreateClient(new WebApplicationFactoryClientOptions
        {
            AllowAutoRedirect = false
        });
        
        // Set authentication header
        _client.DefaultRequestHeaders.Authorization = 
            new AuthenticationHeaderValue("Test");
    }
    
    [Fact]
    public async Task GetUsers_ShouldReturnAllUsers()
    {
        // Act
        var response = await _client.GetAsync("/api/users");
        
        // Assert
        response.EnsureSuccessStatusCode();
        var users = await response.Content.ReadFromJsonAsync<List<User>>();
        
        users.Should().NotBeNull();
        users.Should().HaveCount(2);
    }
    
    [Fact]
    public async Task CreateUser_ShouldReturnCreatedUser()
    {
        // Arrange
        var newUser = new CreateUserRequest
        {
            Name = "New User",
            Email = "newuser@example.com"
        };
        
        // Act
        var response = await _client.PostAsJsonAsync("/api/users", newUser);
        
        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        var createdUser = await response.Content.ReadFromJsonAsync<User>();
        
        createdUser.Should().NotBeNull();
        createdUser.Name.Should().Be(newUser.Name);
        createdUser.Email.Should().Be(newUser.Email);
    }
}
```

### 2.2 Blazor 통합 테스트

```csharp
// Tests/Integration/BlazorIntegrationTests.cs
public class BlazorIntegrationTests : IClassFixture<CustomWebApplicationFactory<Program>>
{
    private readonly CustomWebApplicationFactory<Program> _factory;
    
    public BlazorIntegrationTests(CustomWebApplicationFactory<Program> factory)
    {
        _factory = factory;
    }
    
    [Fact]
    public async Task HomePage_ShouldRenderCorrectly()
    {
        // Arrange
        var client = _factory.CreateClient();
        
        // Act
        var response = await client.GetAsync("/");
        
        // Assert
        response.EnsureSuccessStatusCode();
        var content = await response.Content.ReadAsStringAsync();
        
        content.Should().Contain("<h1>Welcome to Blazor</h1>");
        content.Should().Contain("<app>");
    }
    
    [Fact]
    public async Task Navigation_ShouldWorkCorrectly()
    {
        // Arrange
        using var ctx = new TestContext();
        ctx.Services.AddSingleton(_factory.Services);
        
        var navMan = ctx.Services.GetRequiredService<NavigationManager>();
        
        // Act & Assert
        var app = ctx.RenderComponent<App>();
        
        // Navigate to counter
        navMan.NavigateTo("/counter");
        app.WaitForState(() => app.Find("h1").TextContent == "Counter");
        
        // Navigate to fetch data
        navMan.NavigateTo("/fetchdata");
        app.WaitForState(() => app.Find("h1").TextContent == "Weather forecast");
    }
}
```

## 3. E2E 테스트

### 3.1 Playwright를 사용한 E2E 테스트

```csharp
// Tests/E2E/PlaywrightTests.cs
public class PlaywrightTests : IClassFixture<CustomWebApplicationFactory<Program>>, IAsyncLifetime
{
    private readonly CustomWebApplicationFactory<Program> _factory;
    private IPlaywright _playwright = null!;
    private IBrowser _browser = null!;
    private IBrowserContext _context = null!;
    private IPage _page = null!;
    
    public PlaywrightTests(CustomWebApplicationFactory<Program> factory)
    {
        _factory = factory;
    }
    
    public async Task InitializeAsync()
    {
        _playwright = await Playwright.CreateAsync();
        _browser = await _playwright.Chromium.LaunchAsync(new BrowserTypeLaunchOptions
        {
            Headless = true
        });
        _context = await _browser.NewContextAsync();
        _page = await _context.NewPageAsync();
    }
    
    public async Task DisposeAsync()
    {
        await _page.CloseAsync();
        await _context.CloseAsync();
        await _browser.CloseAsync();
        _playwright.Dispose();
    }
    
    [Fact]
    public async Task Counter_ShouldIncrementCorrectly()
    {
        // Arrange
        var client = _factory.CreateClient();
        var baseUrl = client.BaseAddress!.ToString();
        
        // Act
        await _page.GotoAsync($"{baseUrl}counter");
        
        // Wait for page to load
        await _page.WaitForSelectorAsync("h1:has-text('Counter')");
        
        // Get initial count
        var initialCount = await _page.TextContentAsync("p");
        initialCount.Should().Be("Current count: 0");
        
        // Click increment button
        await _page.ClickAsync("button:has-text('Click me')");
        
        // Verify count increased
        var updatedCount = await _page.TextContentAsync("p");
        updatedCount.Should().Be("Current count: 1");
        
        // Click multiple times
        for (int i = 0; i < 5; i++)
        {
            await _page.ClickAsync("button:has-text('Click me')");
        }
        
        // Verify final count
        var finalCount = await _page.TextContentAsync("p");
        finalCount.Should().Be("Current count: 6");
    }
    
    [Fact]
    public async Task TodoList_ShouldAddAndCompleteTodos()
    {
        // Arrange
        var client = _factory.CreateClient();
        var baseUrl = client.BaseAddress!.ToString();
        
        // Act
        await _page.GotoAsync($"{baseUrl}todos");
        
        // Wait for todos to load
        await _page.WaitForSelectorAsync(".todo-list");
        
        // Add new todo
        await _page.FillAsync("input[placeholder='What needs to be done?']", "Test Todo");
        await _page.PressAsync("input[placeholder='What needs to be done?']", "Enter");
        
        // Verify todo was added
        var todoText = await _page.TextContentAsync(".todo-item:last-child .todo-text");
        todoText.Should().Be("Test Todo");
        
        // Complete todo
        await _page.ClickAsync(".todo-item:last-child input[type='checkbox']");
        
        // Verify todo is marked as completed
        var isCompleted = await _page.IsCheckedAsync(".todo-item:last-child input[type='checkbox']");
        isCompleted.Should().BeTrue();
    }
}
```

### 3.2 Selenium을 사용한 E2E 테스트

```csharp
// Tests/E2E/SeleniumTests.cs
public class SeleniumTests : IClassFixture<CustomWebApplicationFactory<Program>>, IDisposable
{
    private readonly CustomWebApplicationFactory<Program> _factory;
    private readonly IWebDriver _driver;
    private readonly string _baseUrl;
    
    public SeleniumTests(CustomWebApplicationFactory<Program> factory)
    {
        _factory = factory;
        
        var options = new ChromeOptions();
        options.AddArgument("--headless");
        options.AddArgument("--no-sandbox");
        options.AddArgument("--disable-dev-shm-usage");
        
        _driver = new ChromeDriver(options);
        _baseUrl = _factory.CreateClient().BaseAddress!.ToString();
    }
    
    [Fact]
    public void FormValidation_ShouldShowErrors()
    {
        // Navigate to form
        _driver.Navigate().GoToUrl($"{_baseUrl}register");
        
        // Wait for page load
        var wait = new WebDriverWait(_driver, TimeSpan.FromSeconds(10));
        wait.Until(d => d.FindElement(By.TagName("h1")).Text == "Register");
        
        // Submit empty form
        _driver.FindElement(By.CssSelector("button[type='submit']")).Click();
        
        // Verify validation errors
        var errors = _driver.FindElements(By.CssSelector(".validation-message"));
        errors.Should().NotBeEmpty();
        
        // Fill partial form
        _driver.FindElement(By.Id("username")).SendKeys("testuser");
        _driver.FindElement(By.Id("email")).SendKeys("invalid-email");
        
        // Submit again
        _driver.FindElement(By.CssSelector("button[type='submit']")).Click();
        
        // Verify email validation error
        var emailError = _driver.FindElement(By.CssSelector("#email + .validation-message"));
        emailError.Text.Should().Contain("valid email");
    }
    
    public void Dispose()
    {
        _driver.Quit();
        _driver.Dispose();
    }
}
```

## 4. 컴포넌트 테스트 고급 기법

### 4.1 파라미터와 캐스케이딩 테스트

```csharp
// Tests/Components/AdvancedComponentTests.cs
public class AdvancedComponentTests : TestContext
{
    [Fact]
    public void Component_WithParameters_ShouldRenderCorrectly()
    {
        // Arrange
        var title = "Test Title";
        var items = new List<string> { "Item 1", "Item 2", "Item 3" };
        
        // Act
        var component = RenderComponent<ItemList>(parameters => parameters
            .Add(p => p.Title, title)
            .Add(p => p.Items, items)
            .Add(p => p.ShowCount, true));
        
        // Assert
        component.Find("h2").TextContent.Should().Be(title);
        component.FindAll("li").Should().HaveCount(3);
        component.Find(".item-count").TextContent.Should().Be("Total: 3");
    }
    
    [Fact]
    public void Component_WithCascadingValues_ShouldReceiveValues()
    {
        // Arrange
        var theme = new ThemeInfo { PrimaryColor = "blue", SecondaryColor = "green" };
        var user = new UserInfo { Name = "Test User", Role = "Admin" };
        
        // Act
        var component = RenderComponent<ThemedComponent>(
            new CascadingValue<ThemeInfo>(theme),
            new CascadingValue<UserInfo>(user));
        
        // Assert
        var styles = component.Find(".themed-content").GetAttributes();
        styles["style"].Value.Should().Contain("color: blue");
        
        component.Find(".user-role").TextContent.Should().Be("Admin");
    }
    
    [Fact]
    public void Component_WithEventCallback_ShouldTriggerCorrectly()
    {
        // Arrange
        var eventTriggered = false;
        var eventValue = "";
        
        // Act
        var component = RenderComponent<CustomButton>(parameters => parameters
            .Add(p => p.OnClick, EventCallback.Factory.Create<string>(this, (value) =>
            {
                eventTriggered = true;
                eventValue = value;
            })));
        
        component.Find("button").Click();
        
        // Assert
        eventTriggered.Should().BeTrue();
        eventValue.Should().NotBeEmpty();
    }
}
```

### 4.2 비동기 컴포넌트 테스트

```csharp
// Tests/Components/AsyncComponentTests.cs
public class AsyncComponentTests : TestContext
{
    private readonly Mock<IDataService> _mockDataService;
    
    public AsyncComponentTests()
    {
        _mockDataService = new Mock<IDataService>();
        Services.AddSingleton(_mockDataService.Object);
    }
    
    [Fact]
    public async Task AsyncComponent_ShouldHandleLoadingStates()
    {
        // Arrange
        var tcs = new TaskCompletionSource<DataResult>();
        _mockDataService.Setup(x => x.LoadDataAsync())
            .Returns(tcs.Task);
        
        // Act - Initial render
        var component = RenderComponent<AsyncDataComponent>();
        
        // Assert - Loading state
        component.Find(".loading-spinner").Should().NotBeNull();
        component.FindAll(".data-content").Should().BeEmpty();
        
        // Act - Complete loading
        tcs.SetResult(new DataResult { Data = "Test Data" });
        await component.InvokeAsync(() => Task.Delay(50)); // Allow render cycle
        
        // Assert - Loaded state
        Assert.Throws<ElementNotFoundException>(() => component.Find(".loading-spinner"));
        component.Find(".data-content").TextContent.Should().Be("Test Data");
    }
    
    [Fact]
    public async Task AsyncComponent_ShouldHandleErrors()
    {
        // Arrange
        _mockDataService.Setup(x => x.LoadDataAsync())
            .ThrowsAsync(new Exception("Test error"));
        
        // Act
        var component = RenderComponent<AsyncDataComponent>();
        await component.InvokeAsync(() => Task.Delay(50));
        
        // Assert
        component.Find(".error-message").TextContent.Should().Contain("Test error");
        component.Find("button.retry").Should().NotBeNull();
    }
    
    [Fact]
    public async Task AsyncComponent_ShouldRetryOnError()
    {
        // Arrange
        var callCount = 0;
        _mockDataService.Setup(x => x.LoadDataAsync())
            .ReturnsAsync(() =>
            {
                callCount++;
                if (callCount == 1)
                    throw new Exception("First attempt failed");
                return new DataResult { Data = "Success" };
            });
        
        // Act
        var component = RenderComponent<AsyncDataComponent>();
        await component.InvokeAsync(() => Task.Delay(50));
        
        // Assert - Error state
        component.Find(".error-message").Should().NotBeNull();
        
        // Act - Retry
        await component.Find("button.retry").ClickAsync();
        await component.InvokeAsync(() => Task.Delay(50));
        
        // Assert - Success state
        component.Find(".data-content").TextContent.Should().Be("Success");
        _mockDataService.Verify(x => x.LoadDataAsync(), Times.Exactly(2));
    }
}
```

## 5. 성능 테스트

### 5.1 렌더링 성능 테스트

```csharp
// Tests/Performance/RenderPerformanceTests.cs
public class RenderPerformanceTests : TestContext
{
    [Fact]
    public void LargeList_ShouldRenderWithinTimeLimit()
    {
        // Arrange
        var items = Enumerable.Range(1, 1000)
            .Select(i => new Item { Id = i, Name = $"Item {i}" })
            .ToList();
        
        var stopwatch = Stopwatch.StartNew();
        
        // Act
        var component = RenderComponent<LargeListComponent>(parameters => parameters
            .Add(p => p.Items, items));
        
        stopwatch.Stop();
        
        // Assert
        stopwatch.ElapsedMilliseconds.Should().BeLessThan(1000); // 1 second limit
        component.FindAll(".list-item").Should().HaveCount(1000);
    }
    
    [Fact]
    public void Component_ShouldNotRerenderUnnecessarily()
    {
        // Arrange
        var renderCount = 0;
        var component = RenderComponent<OptimizedComponent>(parameters => parameters
            .Add(p => p.OnRender, () => renderCount++));
        
        // Initial render
        renderCount.Should().Be(1);
        
        // Act - Update parameter that shouldn't cause rerender
        component.SetParametersAndRender(parameters => parameters
            .Add(p => p.UnusedParameter, "new value"));
        
        // Assert - No additional render
        renderCount.Should().Be(1);
        
        // Act - Update parameter that should cause rerender
        component.SetParametersAndRender(parameters => parameters
            .Add(p => p.DisplayValue, "new display value"));
        
        // Assert - One additional render
        renderCount.Should().Be(2);
    }
}
```

### 5.2 메모리 사용량 테스트

```csharp
// Tests/Performance/MemoryUsageTests.cs
public class MemoryUsageTests : TestContext, IDisposable
{
    private long _initialMemory;
    
    public MemoryUsageTests()
    {
        GC.Collect();
        GC.WaitForPendingFinalizers();
        GC.Collect();
        _initialMemory = GC.GetTotalMemory(false);
    }
    
    [Fact]
    public void Component_ShouldNotLeakMemory()
    {
        // Arrange
        var components = new List<IRenderedComponent<DataIntensiveComponent>>();
        
        // Act - Create and destroy components
        for (int i = 0; i < 100; i++)
        {
            var component = RenderComponent<DataIntensiveComponent>();
            components.Add(component);
        }
        
        // Dispose all components
        foreach (var component in components)
        {
            component.Instance.Dispose();
        }
        components.Clear();
        
        // Force garbage collection
        GC.Collect();
        GC.WaitForPendingFinalizers();
        GC.Collect();
        
        // Assert
        var finalMemory = GC.GetTotalMemory(false);
        var memoryIncrease = finalMemory - _initialMemory;
        
        // Allow for some memory increase, but not excessive
        memoryIncrease.Should().BeLessThan(10 * 1024 * 1024); // 10 MB
    }
    
    public new void Dispose()
    {
        base.Dispose();
        GC.Collect();
        GC.WaitForPendingFinalizers();
        GC.Collect();
    }
}
```

## 6. 테스트 유틀리티

### 6.1 테스트 헬퍼

```csharp
// Tests/Helpers/TestAuthenticationHandler.cs
public class TestAuthenticationHandler : AuthenticationHandler<TestAuthenticationSchemeOptions>
{
    public TestAuthenticationHandler(IOptionsMonitor<TestAuthenticationSchemeOptions> options,
        ILoggerFactory logger, UrlEncoder encoder, ISystemClock clock)
        : base(options, logger, encoder, clock)
    {
    }
    
    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        var claims = new[]
        {
            new Claim(ClaimTypes.Name, "Test User"),
            new Claim(ClaimTypes.NameIdentifier, "123"),
            new Claim(ClaimTypes.Role, "Admin")
        };
        
        var identity = new ClaimsIdentity(claims, "Test");
        var principal = new ClaimsPrincipal(identity);
        var ticket = new AuthenticationTicket(principal, "Test");
        
        return Task.FromResult(AuthenticateResult.Success(ticket));
    }
}

public class TestAuthenticationSchemeOptions : AuthenticationSchemeOptions { }

// Tests/Helpers/BlazorTestExtensions.cs
public static class BlazorTestExtensions
{
    public static void WaitForState<TComponent>(this IRenderedComponent<TComponent> component,
        Func<bool> predicate, TimeSpan? timeout = null) where TComponent : IComponent
    {
        var timeoutMs = (int)(timeout ?? TimeSpan.FromSeconds(5)).TotalMilliseconds;
        var stopwatch = Stopwatch.StartNew();
        
        while (!predicate() && stopwatch.ElapsedMilliseconds < timeoutMs)
        {
            Thread.Sleep(50);
        }
        
        if (!predicate())
        {
            throw new TimeoutException($"State condition not met within {timeoutMs}ms");
        }
    }
    
    public static async Task<T> WaitForAsync<T>(Func<Task<T>> func, 
        Func<T, bool> predicate, TimeSpan? timeout = null)
    {
        var timeoutMs = (int)(timeout ?? TimeSpan.FromSeconds(5)).TotalMilliseconds;
        var stopwatch = Stopwatch.StartNew();
        
        while (stopwatch.ElapsedMilliseconds < timeoutMs)
        {
            var result = await func();
            if (predicate(result))
            {
                return result;
            }
            
            await Task.Delay(50);
        }
        
        throw new TimeoutException($"Async condition not met within {timeoutMs}ms");
    }
}
```

### 6.2 테스트 데이터 빌더

```csharp
// Tests/Builders/TestDataBuilder.cs
public class UserBuilder
{
    private User _user = new();
    
    public UserBuilder WithId(int id)
    {
        _user.Id = id;
        return this;
    }
    
    public UserBuilder WithName(string name)
    {
        _user.Name = name;
        return this;
    }
    
    public UserBuilder WithEmail(string email)
    {
        _user.Email = email;
        return this;
    }
    
    public UserBuilder WithRole(string role)
    {
        _user.Role = role;
        return this;
    }
    
    public UserBuilder AsAdmin()
    {
        _user.Role = "Admin";
        _user.Permissions = new[] { "Read", "Write", "Delete" };
        return this;
    }
    
    public User Build() => _user;
    
    public static implicit operator User(UserBuilder builder) => builder.Build();
}

// Usage
var adminUser = new UserBuilder()
    .WithName("Admin User")
    .WithEmail("admin@example.com")
    .AsAdmin()
    .Build();
```

## 마무리

Blazor 애플리케이션의 효과적인 테스트 전략은 단위 테스트, 통합 테스트, E2E 테스트를 적절히 조합하여 구현합니다. bUnit을 활용한 컴포넌트 테스트, WebApplicationFactory를 통한 통합 테스트, Playwright/Selenium을 사용한 E2E 테스트를 통해 애플리케이션의 모든 계층을 검증할 수 있습니다. 성능 테스트를 포함하여 애플리케이션의 품질과 안정성을 보장하는 것이 중요합니다.