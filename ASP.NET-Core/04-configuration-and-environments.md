# 구성(Configuration)과 환경 설정

## Configuration 시스템 개요

ASP.NET Core의 구성 시스템은 다양한 소스에서 설정을 읽고 통합 관리할 수 있는 유연한 시스템입니다.

### 구성 소스 우선순위 (나중에 로드된 것이 우선)
1. appsettings.json
2. appsettings.{Environment}.json
3. 사용자 비밀(User Secrets)
4. 환경 변수
5. 명령줄 인수

## 기본 구성 파일

### appsettings.json
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=MyDb;Trusted_Connection=True;"
  },
  "ApiSettings": {
    "BaseUrl": "https://api.example.com",
    "ApiKey": "your-api-key",
    "Timeout": 30
  }
}
```

### 환경별 설정 파일
```json
// appsettings.Development.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "Microsoft.AspNetCore": "Debug"
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MyDb_Dev;User Id=dev;Password=dev123;"
  }
}

// appsettings.Production.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Error",
      "Microsoft.AspNetCore": "Error"
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=prod-server;Database=MyDb;User Id=prod;Password=prod123;"
  }
}
```

## Configuration 읽기 (.NET 6.0+)

### Program.cs에서 직접 사용
```csharp
var builder = WebApplication.CreateBuilder(args);

// 구성 값 읽기
var connectionString = builder.Configuration.GetConnectionString("DefaultConnection");
var apiKey = builder.Configuration["ApiSettings:ApiKey"];
var timeout = builder.Configuration.GetValue<int>("ApiSettings:Timeout", 30); // 기본값 지정

// 섹션 읽기
var apiSettings = builder.Configuration.GetSection("ApiSettings");
Console.WriteLine($"API Base URL: {apiSettings["BaseUrl"]}");

// 서비스에 구성 주입
builder.Services.AddSingleton<IConfiguration>(builder.Configuration);

var app = builder.Build();
```

### Options 패턴 사용
```csharp
// 설정 클래스
public class ApiSettings
{
    public string BaseUrl { get; set; } = string.Empty;
    public string ApiKey { get; set; } = string.Empty;
    public int Timeout { get; set; } = 30;
}

public class EmailSettings
{
    public string SmtpServer { get; set; } = string.Empty;
    public int Port { get; set; } = 587;
    public string Username { get; set; } = string.Empty;
    public string Password { get; set; } = string.Empty;
    public bool EnableSsl { get; set; } = true;
}

// Program.cs에서 등록
var builder = WebApplication.CreateBuilder(args);

// Options 패턴으로 등록
builder.Services.Configure<ApiSettings>(
    builder.Configuration.GetSection("ApiSettings"));
builder.Services.Configure<EmailSettings>(
    builder.Configuration.GetSection("EmailSettings"));

// 서비스에서 사용
public class ApiService
{
    private readonly ApiSettings _apiSettings;
    private readonly ILogger<ApiService> _logger;
    
    public ApiService(IOptions<ApiSettings> apiSettings, ILogger<ApiService> logger)
    {
        _apiSettings = apiSettings.Value;
        _logger = logger;
    }
    
    public async Task<string> GetDataAsync()
    {
        using var client = new HttpClient();
        client.BaseAddress = new Uri(_apiSettings.BaseUrl);
        client.Timeout = TimeSpan.FromSeconds(_apiSettings.Timeout);
        client.DefaultRequestHeaders.Add("X-API-Key", _apiSettings.ApiKey);
        
        _logger.LogInformation("Calling API at {BaseUrl}", _apiSettings.BaseUrl);
        return await client.GetStringAsync("/data");
    }
}
```

## 환경 관리 (.NET 6.0+)

### WebApplication에서 환경 확인
```csharp
var builder = WebApplication.CreateBuilder(args);

// 환경 정보 출력
Console.WriteLine($"Environment: {builder.Environment.EnvironmentName}");
Console.WriteLine($"ApplicationName: {builder.Environment.ApplicationName}");
Console.WriteLine($"ContentRootPath: {builder.Environment.ContentRootPath}");
Console.WriteLine($"WebRootPath: {builder.Environment.WebRootPath}");

// 환경별 서비스 등록
if (builder.Environment.IsDevelopment())
{
    builder.Services.AddDbContext<AppDbContext>(options =>
        options.UseInMemoryDatabase("DevDb"));
}
else if (builder.Environment.IsProduction())
{
    builder.Services.AddDbContext<AppDbContext>(options =>
        options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));
}

// 사용자 정의 환경 확인
if (builder.Environment.IsEnvironment("Staging"))
{
    builder.Services.AddApplicationInsightsTelemetry();
}

var app = builder.Build();

// 환경별 미들웨어 구성
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
    app.UseSwagger();
    app.UseSwaggerUI();
}
else
{
    app.UseExceptionHandler("/Error");
    app.UseHsts();
}
```

### launchSettings.json (개발 환경 전용)
```json
{
  "$schema": "https://json.schemastore.org/launchsettings.json",
  "profiles": {
    "http": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "launchUrl": "swagger",
      "applicationUrl": "http://localhost:5000",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    },
    "https": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "launchUrl": "swagger",
      "applicationUrl": "https://localhost:7000;http://localhost:5000",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    },
    "Staging": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": false,
      "applicationUrl": "https://localhost:7001",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Staging",
        "CUSTOM_SETTING": "StagingValue"
      }
    }
  }
}
```

## User Secrets (개발 환경 전용)

### User Secrets 설정
```bash
# 프로젝트에서 User Secrets 초기화
dotnet user-secrets init

# 비밀 값 설정
dotnet user-secrets set "ApiSettings:ApiKey" "dev-secret-key"
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Server=localhost;Database=DevDb;..."

# 비밀 값 목록 확인
dotnet user-secrets list

# 비밀 값 제거
dotnet user-secrets remove "ApiSettings:ApiKey"

# 모든 비밀 값 제거
dotnet user-secrets clear
```

### Program.cs에서 User Secrets 사용
```csharp
var builder = WebApplication.CreateBuilder(args);

// Development 환경에서는 자동으로 User Secrets가 로드됨
// 수동으로 추가하려면:
if (builder.Environment.IsDevelopment())
{
    builder.Configuration.AddUserSecrets<Program>();
}
```

## 사용자 정의 구성 소스

### 추가 JSON 파일
```csharp
var builder = WebApplication.CreateBuilder(args);

// 추가 구성 파일 로드
builder.Configuration
    .AddJsonFile("customsettings.json", optional: true, reloadOnChange: true)
    .AddJsonFile($"customsettings.{builder.Environment.EnvironmentName}.json", 
        optional: true, reloadOnChange: true);
```

### 환경 변수와 명령줄
```csharp
var builder = WebApplication.CreateBuilder(args);

// 특정 접두사로 환경 변수 필터링
builder.Configuration.AddEnvironmentVariables(prefix: "MYAPP_");

// 명령줄 인수 추가
builder.Configuration.AddCommandLine(args);

// 사용 예:
// dotnet run --ApiSettings:Timeout=60
// $env:MYAPP_ApiSettings__ApiKey="env-secret-key" (PowerShell)
// export MYAPP_ApiSettings__ApiKey="env-secret-key" (Linux/Mac)
```

### 메모리 구성
```csharp
var builder = WebApplication.CreateBuilder(args);

// 메모리에서 구성 추가
var inMemoryConfig = new Dictionary<string, string?>
{
    ["Features:EnableNewUI"] = "true",
    ["Features:MaxItemsPerPage"] = "50"
};

builder.Configuration.AddInMemoryCollection(inMemoryConfig);
```

## 구성 변경 감지

### IOptionsMonitor 사용
```csharp
public interface IFeatureService
{
    bool IsFeatureEnabled(string feature);
}

public class FeatureService : IFeatureService
{
    private readonly IOptionsMonitor<FeatureSettings> _optionsMonitor;
    private readonly ILogger<FeatureService> _logger;
    
    public FeatureService(
        IOptionsMonitor<FeatureSettings> optionsMonitor,
        ILogger<FeatureService> logger)
    {
        _optionsMonitor = optionsMonitor;
        _logger = logger;
        
        // 구성 변경 감지
        _optionsMonitor.OnChange((settings, name) =>
        {
            _logger.LogInformation("Feature settings changed");
            foreach (var feature in settings.Features)
            {
                _logger.LogInformation("Feature {Name}: {Enabled}", 
                    feature.Key, feature.Value);
            }
        });
    }
    
    public bool IsFeatureEnabled(string feature)
    {
        var settings = _optionsMonitor.CurrentValue;
        return settings.Features.TryGetValue(feature, out var enabled) && enabled;
    }
}

public class FeatureSettings
{
    public Dictionary<string, bool> Features { get; set; } = new();
}
```

### IOptionsSnapshot 사용 (요청별 새로고침)
```csharp
public class PerRequestConfigService
{
    private readonly IOptionsSnapshot<ApiSettings> _apiSettings;
    
    public PerRequestConfigService(IOptionsSnapshot<ApiSettings> apiSettings)
    {
        _apiSettings = apiSettings;
    }
    
    public string GetCurrentApiUrl()
    {
        // 각 HTTP 요청마다 최신 설정값 사용
        return _apiSettings.Value.BaseUrl;
    }
}
```

## 구성 검증

### DataAnnotations를 사용한 검증
```csharp
using System.ComponentModel.DataAnnotations;

public class ApiSettings
{
    [Required(ErrorMessage = "API Base URL is required")]
    [Url(ErrorMessage = "API Base URL must be a valid URL")]
    public string BaseUrl { get; set; } = string.Empty;
    
    [Required(ErrorMessage = "API Key is required")]
    [MinLength(10, ErrorMessage = "API Key must be at least 10 characters")]
    public string ApiKey { get; set; } = string.Empty;
    
    [Range(1, 300, ErrorMessage = "Timeout must be between 1 and 300 seconds")]
    public int Timeout { get; set; } = 30;
}

// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddOptions<ApiSettings>()
    .Bind(builder.Configuration.GetSection("ApiSettings"))
    .ValidateDataAnnotations()
    .ValidateOnStart(); // 앱 시작 시 검증

// 사용자 정의 검증
builder.Services.AddOptions<EmailSettings>()
    .Bind(builder.Configuration.GetSection("EmailSettings"))
    .Validate(settings =>
    {
        if (settings.Port < 1 || settings.Port > 65535)
            return false;
            
        if (settings.EnableSsl && settings.Port == 25)
            return false; // SMTP 25번 포트는 일반적으로 SSL 미지원
            
        return true;
    }, "Invalid email settings");
```

### IValidateOptions 인터페이스 구현
```csharp
public class ApiSettingsValidator : IValidateOptions<ApiSettings>
{
    public ValidateOptionsResult Validate(string? name, ApiSettings options)
    {
        var errors = new List<string>();
        
        if (string.IsNullOrWhiteSpace(options.BaseUrl))
            errors.Add("BaseUrl is required");
        else if (!Uri.TryCreate(options.BaseUrl, UriKind.Absolute, out _))
            errors.Add("BaseUrl must be a valid URL");
            
        if (string.IsNullOrWhiteSpace(options.ApiKey))
            errors.Add("ApiKey is required");
        else if (options.ApiKey.Length < 10)
            errors.Add("ApiKey must be at least 10 characters");
            
        if (options.Timeout <= 0)
            errors.Add("Timeout must be greater than 0");
            
        return errors.Count > 0 
            ? ValidateOptionsResult.Fail(errors) 
            : ValidateOptionsResult.Success;
    }
}

// Program.cs
builder.Services.Configure<ApiSettings>(
    builder.Configuration.GetSection("ApiSettings"));
builder.Services.AddSingleton<IValidateOptions<ApiSettings>, ApiSettingsValidator>();
```

## 환경별 기능 토글

### Feature Flags 구현
```csharp
// appsettings.json
{
  "Features": {
    "EnableNewUI": false,
    "EnableBetaFeatures": false,
    "MaintenanceMode": false
  }
}

// appsettings.Development.json
{
  "Features": {
    "EnableNewUI": true,
    "EnableBetaFeatures": true
  }
}

// Feature Toggle Service
public interface IFeatureToggle
{
    bool IsEnabled(string feature);
    Task<bool> IsEnabledAsync(string feature);
}

public class ConfigurationFeatureToggle : IFeatureToggle
{
    private readonly IConfiguration _configuration;
    private readonly IWebHostEnvironment _environment;
    
    public ConfigurationFeatureToggle(
        IConfiguration configuration, 
        IWebHostEnvironment environment)
    {
        _configuration = configuration;
        _environment = environment;
    }
    
    public bool IsEnabled(string feature)
    {
        // 개발 환경에서는 모든 기능 활성화
        if (_environment.IsDevelopment() && feature != "MaintenanceMode")
            return true;
            
        return _configuration.GetValue<bool>($"Features:{feature}");
    }
    
    public Task<bool> IsEnabledAsync(string feature)
    {
        return Task.FromResult(IsEnabled(feature));
    }
}

// 컨트롤러에서 사용
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IFeatureToggle _featureToggle;
    
    public ProductsController(IFeatureToggle featureToggle)
    {
        _featureToggle = featureToggle;
    }
    
    [HttpGet("beta")]
    public IActionResult GetBetaProducts()
    {
        if (!_featureToggle.IsEnabled("EnableBetaFeatures"))
            return NotFound("Beta features are not enabled");
            
        return Ok(new[] { "Beta Product 1", "Beta Product 2" });
    }
}
```

## 보안 모범 사례

### Azure Key Vault 통합
```csharp
// NuGet: Azure.Extensions.AspNetCore.Configuration.Secrets
var builder = WebApplication.CreateBuilder(args);

if (builder.Environment.IsProduction())
{
    var keyVaultUrl = builder.Configuration["KeyVault:Url"];
    if (!string.IsNullOrEmpty(keyVaultUrl))
    {
        builder.Configuration.AddAzureKeyVault(
            new Uri(keyVaultUrl),
            new DefaultAzureCredential());
    }
}
```

### 환경 변수 보안
```csharp
public class SecureConfigurationService
{
    private readonly IConfiguration _configuration;
    private readonly IWebHostEnvironment _environment;
    
    public SecureConfigurationService(
        IConfiguration configuration, 
        IWebHostEnvironment environment)
    {
        _configuration = configuration;
        _environment = environment;
    }
    
    public string GetSecureValue(string key)
    {
        // Production에서는 환경 변수 우선
        if (_environment.IsProduction())
        {
            var envValue = Environment.GetEnvironmentVariable($"SECURE_{key}");
            if (!string.IsNullOrEmpty(envValue))
                return envValue;
        }
        
        // 개발 환경에서는 User Secrets 사용
        return _configuration[key] ?? throw new InvalidOperationException($"Configuration key '{key}' not found");
    }
}
```

## 실전 예제: 다중 데이터베이스 구성

```csharp
// 데이터베이스 설정
public class DatabaseOptions
{
    public const string SectionName = "Database";
    
    public string Provider { get; set; } = "SqlServer";
    public string ConnectionString { get; set; } = string.Empty;
    public bool EnableSensitiveDataLogging { get; set; }
    public int CommandTimeout { get; set; } = 30;
    public bool EnableRetryOnFailure { get; set; } = true;
    public int MaxRetryCount { get; set; } = 3;
}

// appsettings.json
{
  "Database": {
    "Provider": "SqlServer",
    "ConnectionString": "Server=(localdb)\\mssqllocaldb;Database=MyApp;Trusted_Connection=True;",
    "EnableSensitiveDataLogging": false,
    "CommandTimeout": 30,
    "EnableRetryOnFailure": true,
    "MaxRetryCount": 3
  }
}

// Program.cs
var builder = WebApplication.CreateBuilder(args);

// 데이터베이스 옵션 등록
builder.Services.Configure<DatabaseOptions>(
    builder.Configuration.GetSection(DatabaseOptions.SectionName));

// 데이터베이스 컨텍스트 등록
builder.Services.AddDbContext<AppDbContext>((serviceProvider, options) =>
{
    var dbOptions = serviceProvider.GetRequiredService<IOptions<DatabaseOptions>>().Value;
    var env = serviceProvider.GetRequiredService<IWebHostEnvironment>();
    
    switch (dbOptions.Provider.ToLower())
    {
        case "sqlserver":
            options.UseSqlServer(dbOptions.ConnectionString, sqlOptions =>
            {
                sqlOptions.CommandTimeout(dbOptions.CommandTimeout);
                if (dbOptions.EnableRetryOnFailure)
                {
                    sqlOptions.EnableRetryOnFailure(
                        maxRetryCount: dbOptions.MaxRetryCount,
                        maxRetryDelay: TimeSpan.FromSeconds(30),
                        errorNumbersToAdd: null);
                }
            });
            break;
            
        case "postgresql":
            options.UseNpgsql(dbOptions.ConnectionString, npgsqlOptions =>
            {
                npgsqlOptions.CommandTimeout(dbOptions.CommandTimeout);
            });
            break;
            
        case "inmemory":
            options.UseInMemoryDatabase("TestDb");
            break;
            
        default:
            throw new NotSupportedException($"Database provider '{dbOptions.Provider}' is not supported");
    }
    
    if (env.IsDevelopment() && dbOptions.EnableSensitiveDataLogging)
    {
        options.EnableSensitiveDataLogging();
        options.EnableDetailedErrors();
    }
});

var app = builder.Build();

// 구성 확인 엔드포인트 (개발 환경 전용)
if (app.Environment.IsDevelopment())
{
    app.MapGet("/config/database", (IOptions<DatabaseOptions> dbOptions) =>
    {
        var options = dbOptions.Value;
        return Results.Ok(new
        {
            Provider = options.Provider,
            CommandTimeout = options.CommandTimeout,
            EnableRetryOnFailure = options.EnableRetryOnFailure,
            MaxRetryCount = options.MaxRetryCount
        });
    });
}

app.Run();
```