# Factory Method와 Abstract Factory

## 개요

Factory 패턴은 객체 생성을 캡슐화하여 클라이언트 코드가 구체적인 클래스에 의존하지 않도록 하는 생성 패턴입니다. Simple Factory, Factory Method, Abstract Factory의 세 가지 변형이 있으며, 각각 다른 수준의 추상화와 유연성을 제공합니다.

## Simple Factory

### 기본 구현

```csharp
// 제품 인터페이스
public interface IVehicle
{
    string Type { get; }
    void Start();
    void Stop();
}

// 구체적인 제품들
public class Car : IVehicle
{
    public string Type => "Car";
    
    public void Start()
    {
        Console.WriteLine("Car engine started");
    }
    
    public void Stop()
    {
        Console.WriteLine("Car engine stopped");
    }
}

public class Motorcycle : IVehicle
{
    public string Type => "Motorcycle";
    
    public void Start()
    {
        Console.WriteLine("Motorcycle engine started");
    }
    
    public void Stop()
    {
        Console.WriteLine("Motorcycle engine stopped");
    }
}

public class Truck : IVehicle
{
    public string Type => "Truck";
    
    public void Start()
    {
        Console.WriteLine("Truck engine started");
    }
    
    public void Stop()
    {
        Console.WriteLine("Truck engine stopped");
    }
}

// Simple Factory
public class VehicleFactory
{
    public static IVehicle CreateVehicle(string vehicleType)
    {
        return vehicleType.ToLower() switch
        {
            "car" => new Car(),
            "motorcycle" => new Motorcycle(),
            "truck" => new Truck(),
            _ => throw new ArgumentException($"Unknown vehicle type: {vehicleType}")
        };
    }
}

// 사용
var car = VehicleFactory.CreateVehicle("car");
car.Start();
```

### 매개변수를 활용한 Simple Factory

```csharp
// 설정 클래스
public class VehicleConfiguration
{
    public string Type { get; set; }
    public string Model { get; set; }
    public int Year { get; set; }
    public Dictionary<string, object> Options { get; set; } = new();
}

// 향상된 Factory
public class ConfigurableVehicleFactory
{
    private readonly Dictionary<string, Func<VehicleConfiguration, IVehicle>> _creators;
    
    public ConfigurableVehicleFactory()
    {
        _creators = new Dictionary<string, Func<VehicleConfiguration, IVehicle>>
        {
            ["car"] = config => new ConfigurableCar(config),
            ["motorcycle"] = config => new ConfigurableMotorcycle(config),
            ["truck"] = config => new ConfigurableTruck(config)
        };
    }
    
    public IVehicle CreateVehicle(VehicleConfiguration configuration)
    {
        if (_creators.TryGetValue(configuration.Type.ToLower(), out var creator))
        {
            return creator(configuration);
        }
        
        throw new NotSupportedException($"Vehicle type '{configuration.Type}' is not supported");
    }
    
    public void RegisterVehicleType(string type, Func<VehicleConfiguration, IVehicle> creator)
    {
        _creators[type.ToLower()] = creator;
    }
}

public class ConfigurableCar : IVehicle
{
    private readonly VehicleConfiguration _config;
    
    public ConfigurableCar(VehicleConfiguration config)
    {
        _config = config;
    }
    
    public string Type => $"{_config.Model} Car ({_config.Year})";
    
    public void Start()
    {
        var hasRemoteStart = _config.Options.TryGetValue("RemoteStart", out var remote) && 
                            (bool)remote;
        
        if (hasRemoteStart)
        {
            Console.WriteLine($"{Type}: Remote start activated");
        }
        else
        {
            Console.WriteLine($"{Type}: Key start");
        }
    }
    
    public void Stop()
    {
        Console.WriteLine($"{Type} stopped");
    }
}
```

## Factory Method 패턴

### 기본 구현

```csharp
// 제품 인터페이스
public interface IDocument
{
    void Open();
    void Save();
    void Close();
}

// 구체적인 제품들
public class WordDocument : IDocument
{
    public void Open() => Console.WriteLine("Opening Word document");
    public void Save() => Console.WriteLine("Saving Word document");
    public void Close() => Console.WriteLine("Closing Word document");
}

public class PdfDocument : IDocument
{
    public void Open() => Console.WriteLine("Opening PDF document");
    public void Save() => Console.WriteLine("Saving PDF document");
    public void Close() => Console.WriteLine("Closing PDF document");
}

public class ExcelDocument : IDocument
{
    public void Open() => Console.WriteLine("Opening Excel document");
    public void Save() => Console.WriteLine("Saving Excel document");
    public void Close() => Console.WriteLine("Closing Excel document");
}

// Creator 추상 클래스
public abstract class DocumentCreator
{
    // Factory Method
    protected abstract IDocument CreateDocument();
    
    // Template Method 패턴도 함께 사용
    public void NewDocument()
    {
        var document = CreateDocument();
        document.Open();
        Console.WriteLine("Document created and opened");
    }
    
    public void ProcessDocument(string fileName)
    {
        var document = CreateDocument();
        document.Open();
        // 문서 처리 로직
        document.Save();
        document.Close();
    }
}

// Concrete Creators
public class WordDocumentCreator : DocumentCreator
{
    protected override IDocument CreateDocument()
    {
        return new WordDocument();
    }
}

public class PdfDocumentCreator : DocumentCreator
{
    protected override IDocument CreateDocument()
    {
        return new PdfDocument();
    }
}

public class ExcelDocumentCreator : DocumentCreator
{
    protected override IDocument CreateDocument()
    {
        return new ExcelDocument();
    }
}

// 사용
DocumentCreator creator = new WordDocumentCreator();
creator.NewDocument();
creator.ProcessDocument("report.docx");
```

### 제네릭 Factory Method

```csharp
// 제네릭 Factory Method 인터페이스
public interface IFactory<T>
{
    T Create();
}

// 매개변수가 있는 Factory
public interface IFactory<TProduct, TConfig>
{
    TProduct Create(TConfig configuration);
}

// Repository Factory 예제
public interface IRepository<T> where T : class
{
    Task<T> GetByIdAsync(int id);
    Task<IEnumerable<T>> GetAllAsync();
    Task AddAsync(T entity);
    Task UpdateAsync(T entity);
    Task DeleteAsync(T entity);
}

public abstract class RepositoryFactory<T> : IFactory<IRepository<T>> where T : class
{
    protected readonly DbContext _context;
    
    protected RepositoryFactory(DbContext context)
    {
        _context = context;
    }
    
    public abstract IRepository<T> Create();
}

public class GenericRepositoryFactory<T> : RepositoryFactory<T> where T : class
{
    private readonly ILogger<Repository<T>> _logger;
    
    public GenericRepositoryFactory(DbContext context, ILogger<Repository<T>> logger) 
        : base(context)
    {
        _logger = logger;
    }
    
    public override IRepository<T> Create()
    {
        return new Repository<T>(_context, _logger);
    }
}

// 특화된 Repository Factory
public class CustomerRepositoryFactory : RepositoryFactory<Customer>
{
    private readonly ICacheService _cacheService;
    private readonly ILogger<CustomerRepository> _logger;
    
    public CustomerRepositoryFactory(
        DbContext context, 
        ICacheService cacheService,
        ILogger<CustomerRepository> logger) : base(context)
    {
        _cacheService = cacheService;
        _logger = logger;
    }
    
    public override IRepository<Customer> Create()
    {
        // 캐싱이 적용된 특별한 Repository 반환
        return new CustomerRepository(_context, _cacheService, _logger);
    }
}
```

## Abstract Factory 패턴

### 기본 구현

```csharp
// 제품 인터페이스들
public interface IButton
{
    void Click();
    void Render();
}

public interface ITextBox
{
    string Text { get; set; }
    void Render();
}

public interface ICheckBox
{
    bool IsChecked { get; set; }
    void Render();
}

// Windows 제품군
public class WindowsButton : IButton
{
    public void Click() => Console.WriteLine("Windows button clicked");
    public void Render() => Console.WriteLine("Rendering Windows button");
}

public class WindowsTextBox : ITextBox
{
    public string Text { get; set; } = "";
    public void Render() => Console.WriteLine($"Rendering Windows textbox: {Text}");
}

public class WindowsCheckBox : ICheckBox
{
    public bool IsChecked { get; set; }
    public void Render() => Console.WriteLine($"Rendering Windows checkbox: {(IsChecked ? "☑" : "☐")}");
}

// macOS 제품군
public class MacButton : IButton
{
    public void Click() => Console.WriteLine("macOS button clicked");
    public void Render() => Console.WriteLine("Rendering macOS button");
}

public class MacTextBox : ITextBox
{
    public string Text { get; set; } = "";
    public void Render() => Console.WriteLine($"Rendering macOS textbox: {Text}");
}

public class MacCheckBox : ICheckBox
{
    public bool IsChecked { get; set; }
    public void Render() => Console.WriteLine($"Rendering macOS checkbox: {(IsChecked ? "✓" : "✗")}");
}

// Abstract Factory 인터페이스
public interface IUIFactory
{
    IButton CreateButton();
    ITextBox CreateTextBox();
    ICheckBox CreateCheckBox();
}

// Concrete Factories
public class WindowsUIFactory : IUIFactory
{
    public IButton CreateButton() => new WindowsButton();
    public ITextBox CreateTextBox() => new WindowsTextBox();
    public ICheckBox CreateCheckBox() => new WindowsCheckBox();
}

public class MacUIFactory : IUIFactory
{
    public IButton CreateButton() => new MacButton();
    public ITextBox CreateTextBox() => new MacTextBox();
    public ICheckBox CreateCheckBox() => new MacCheckBox();
}

// Client 코드
public class Application
{
    private readonly IUIFactory _uiFactory;
    
    public Application(IUIFactory uiFactory)
    {
        _uiFactory = uiFactory;
    }
    
    public void CreateLoginForm()
    {
        var usernameTextBox = _uiFactory.CreateTextBox();
        usernameTextBox.Text = "Username";
        
        var passwordTextBox = _uiFactory.CreateTextBox();
        passwordTextBox.Text = "Password";
        
        var rememberMeCheckBox = _uiFactory.CreateCheckBox();
        rememberMeCheckBox.IsChecked = false;
        
        var loginButton = _uiFactory.CreateButton();
        
        // 렌더링
        Console.WriteLine("=== Login Form ===");
        usernameTextBox.Render();
        passwordTextBox.Render();
        rememberMeCheckBox.Render();
        loginButton.Render();
    }
}

// 사용
IUIFactory factory = Environment.OSVersion.Platform == PlatformID.Win32NT
    ? new WindowsUIFactory()
    : new MacUIFactory();

var app = new Application(factory);
app.CreateLoginForm();
```

### 데이터베이스 연결 Abstract Factory

```csharp
// 데이터베이스 연결 관련 인터페이스들
public interface IDbConnectionFactory
{
    IDbConnection CreateConnection();
    IDbCommand CreateCommand();
    IDbDataParameter CreateParameter(string name, object value);
    IDbTransaction BeginTransaction(IDbConnection connection);
}

// SQL Server Factory
public class SqlServerConnectionFactory : IDbConnectionFactory
{
    private readonly string _connectionString;
    
    public SqlServerConnectionFactory(string connectionString)
    {
        _connectionString = connectionString;
    }
    
    public IDbConnection CreateConnection()
    {
        return new SqlConnection(_connectionString);
    }
    
    public IDbCommand CreateCommand()
    {
        return new SqlCommand();
    }
    
    public IDbDataParameter CreateParameter(string name, object value)
    {
        return new SqlParameter(name, value ?? DBNull.Value);
    }
    
    public IDbTransaction BeginTransaction(IDbConnection connection)
    {
        return connection.BeginTransaction();
    }
}

// PostgreSQL Factory
public class PostgreSqlConnectionFactory : IDbConnectionFactory
{
    private readonly string _connectionString;
    
    public PostgreSqlConnectionFactory(string connectionString)
    {
        _connectionString = connectionString;
    }
    
    public IDbConnection CreateConnection()
    {
        return new NpgsqlConnection(_connectionString);
    }
    
    public IDbCommand CreateCommand()
    {
        return new NpgsqlCommand();
    }
    
    public IDbDataParameter CreateParameter(string name, object value)
    {
        return new NpgsqlParameter(name, value ?? DBNull.Value);
    }
    
    public IDbTransaction BeginTransaction(IDbConnection connection)
    {
        return connection.BeginTransaction();
    }
}

// 데이터베이스 독립적인 Repository
public class DatabaseRepository
{
    private readonly IDbConnectionFactory _dbFactory;
    private readonly ILogger<DatabaseRepository> _logger;
    
    public DatabaseRepository(
        IDbConnectionFactory dbFactory, 
        ILogger<DatabaseRepository> logger)
    {
        _dbFactory = dbFactory;
        _logger = logger;
    }
    
    public async Task<T> ExecuteScalarAsync<T>(string sql, params (string name, object value)[] parameters)
    {
        using var connection = _dbFactory.CreateConnection();
        await connection.OpenAsync();
        
        using var command = _dbFactory.CreateCommand();
        command.Connection = connection;
        command.CommandText = sql;
        
        foreach (var (name, value) in parameters)
        {
            command.Parameters.Add(_dbFactory.CreateParameter(name, value));
        }
        
        var result = await command.ExecuteScalarAsync();
        return (T)Convert.ChangeType(result, typeof(T));
    }
    
    public async Task<IEnumerable<T>> ExecuteQueryAsync<T>(
        string sql, 
        Func<IDataReader, T> mapper,
        params (string name, object value)[] parameters)
    {
        using var connection = _dbFactory.CreateConnection();
        await connection.OpenAsync();
        
        using var command = _dbFactory.CreateCommand();
        command.Connection = connection;
        command.CommandText = sql;
        
        foreach (var (name, value) in parameters)
        {
            command.Parameters.Add(_dbFactory.CreateParameter(name, value));
        }
        
        using var reader = await command.ExecuteReaderAsync();
        var results = new List<T>();
        
        while (await reader.ReadAsync())
        {
            results.Add(mapper(reader));
        }
        
        return results;
    }
}
```

## ASP.NET Core에서의 Factory 패턴

### Service Factory 등록

```csharp
// Startup.cs 또는 Program.cs
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        // Simple Factory 등록
        services.AddSingleton<VehicleFactory>();
        
        // Factory Method 패턴 - 조건부 등록
        services.AddScoped<DocumentCreator>(provider =>
        {
            var httpContext = provider.GetRequiredService<IHttpContextAccessor>().HttpContext;
            var documentType = httpContext?.Request.Query["type"].ToString() ?? "word";
            
            return documentType.ToLower() switch
            {
                "pdf" => new PdfDocumentCreator(),
                "excel" => new ExcelDocumentCreator(),
                _ => new WordDocumentCreator()
            };
        });
        
        // Abstract Factory - 설정 기반
        services.AddSingleton<IUIFactory>(provider =>
        {
            var configuration = provider.GetRequiredService<IConfiguration>();
            var uiTheme = configuration["UI:Theme"];
            
            return uiTheme?.ToLower() switch
            {
                "mac" => new MacUIFactory(),
                "linux" => new LinuxUIFactory(),
                _ => new WindowsUIFactory()
            };
        });
        
        // 데이터베이스 Factory - 연결 문자열 기반
        services.AddScoped<IDbConnectionFactory>(provider =>
        {
            var configuration = provider.GetRequiredService<IConfiguration>();
            var dbProvider = configuration["Database:Provider"];
            var connectionString = configuration.GetConnectionString("DefaultConnection");
            
            return dbProvider?.ToLower() switch
            {
                "postgresql" => new PostgreSqlConnectionFactory(connectionString),
                "mysql" => new MySqlConnectionFactory(connectionString),
                _ => new SqlServerConnectionFactory(connectionString)
            };
        });
    }
}
```

### Factory를 사용한 동적 서비스 생성

```csharp
// 메시지 핸들러 Factory
public interface IMessageHandler
{
    Task HandleAsync(Message message);
}

public interface IMessageHandlerFactory
{
    IMessageHandler CreateHandler(string messageType);
}

public class MessageHandlerFactory : IMessageHandlerFactory
{
    private readonly IServiceProvider _serviceProvider;
    private readonly Dictionary<string, Type> _handlerTypes;
    
    public MessageHandlerFactory(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
        _handlerTypes = new Dictionary<string, Type>
        {
            ["order.created"] = typeof(OrderCreatedHandler),
            ["order.shipped"] = typeof(OrderShippedHandler),
            ["payment.received"] = typeof(PaymentReceivedHandler),
            ["user.registered"] = typeof(UserRegisteredHandler)
        };
    }
    
    public IMessageHandler CreateHandler(string messageType)
    {
        if (!_handlerTypes.TryGetValue(messageType, out var handlerType))
        {
            throw new NotSupportedException($"No handler registered for message type: {messageType}");
        }
        
        return (IMessageHandler)_serviceProvider.GetRequiredService(handlerType);
    }
    
    public void RegisterHandler<THandler>(string messageType) where THandler : IMessageHandler
    {
        _handlerTypes[messageType] = typeof(THandler);
    }
}

// 메시지 처리 서비스
public class MessageProcessingService
{
    private readonly IMessageHandlerFactory _handlerFactory;
    private readonly ILogger<MessageProcessingService> _logger;
    
    public MessageProcessingService(
        IMessageHandlerFactory handlerFactory,
        ILogger<MessageProcessingService> logger)
    {
        _handlerFactory = handlerFactory;
        _logger = logger;
    }
    
    public async Task ProcessMessageAsync(Message message)
    {
        try
        {
            var handler = _handlerFactory.CreateHandler(message.Type);
            await handler.HandleAsync(message);
            
            _logger.LogInformation(
                "Message {MessageId} of type {MessageType} processed successfully",
                message.Id, message.Type);
        }
        catch (NotSupportedException ex)
        {
            _logger.LogWarning(ex, 
                "No handler found for message type {MessageType}", 
                message.Type);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, 
                "Error processing message {MessageId} of type {MessageType}",
                message.Id, message.Type);
            throw;
        }
    }
}
```

## Factory 패턴과 Builder 패턴 결합

```csharp
// 복잡한 객체를 위한 Factory + Builder
public interface IReportBuilder
{
    IReportBuilder SetTitle(string title);
    IReportBuilder SetAuthor(string author);
    IReportBuilder AddSection(ReportSection section);
    IReportBuilder SetFormat(ReportFormat format);
    Report Build();
}

public class ReportBuilder : IReportBuilder
{
    private readonly Report _report = new();
    
    public IReportBuilder SetTitle(string title)
    {
        _report.Title = title;
        return this;
    }
    
    public IReportBuilder SetAuthor(string author)
    {
        _report.Author = author;
        return this;
    }
    
    public IReportBuilder AddSection(ReportSection section)
    {
        _report.Sections.Add(section);
        return this;
    }
    
    public IReportBuilder SetFormat(ReportFormat format)
    {
        _report.Format = format;
        return this;
    }
    
    public Report Build()
    {
        // 검증
        if (string.IsNullOrEmpty(_report.Title))
            throw new InvalidOperationException("Report must have a title");
            
        return _report;
    }
}

// Factory가 Builder를 생성
public interface IReportBuilderFactory
{
    IReportBuilder CreateBuilder(string reportType);
}

public class ReportBuilderFactory : IReportBuilderFactory
{
    private readonly IServiceProvider _serviceProvider;
    
    public ReportBuilderFactory(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }
    
    public IReportBuilder CreateBuilder(string reportType)
    {
        var builder = new ReportBuilder();
        
        // 리포트 타입에 따른 기본 설정
        switch (reportType.ToLower())
        {
            case "sales":
                builder.SetTitle("Sales Report")
                       .SetFormat(ReportFormat.Pdf)
                       .AddSection(new SummarySection())
                       .AddSection(new ChartSection());
                break;
                
            case "inventory":
                builder.SetTitle("Inventory Report")
                       .SetFormat(ReportFormat.Excel)
                       .AddSection(new InventoryListSection());
                break;
                
            case "financial":
                builder.SetTitle("Financial Report")
                       .SetFormat(ReportFormat.Pdf)
                       .AddSection(new BalanceSheetSection())
                       .AddSection(new IncomeStatementSection());
                break;
        }
        
        return builder;
    }
}

// 사용
public class ReportService
{
    private readonly IReportBuilderFactory _builderFactory;
    
    public ReportService(IReportBuilderFactory builderFactory)
    {
        _builderFactory = builderFactory;
    }
    
    public Report GenerateSalesReport(string author, DateTime startDate, DateTime endDate)
    {
        var builder = _builderFactory.CreateBuilder("sales");
        
        return builder
            .SetAuthor(author)
            .AddSection(new DateRangeSection(startDate, endDate))
            .Build();
    }
}
```

## Factory 패턴 선택 가이드

### Simple Factory를 사용할 때
```csharp
// 단순한 객체 생성 로직
// 생성 로직이 자주 변경되지 않을 때
public class NotificationFactory
{
    public static INotification CreateNotification(NotificationType type, string message)
    {
        return type switch
        {
            NotificationType.Email => new EmailNotification(message),
            NotificationType.Sms => new SmsNotification(message),
            NotificationType.Push => new PushNotification(message),
            _ => throw new NotSupportedException()
        };
    }
}
```

### Factory Method를 사용할 때
```csharp
// 서브클래스가 생성할 객체를 결정해야 할 때
// Template Method와 함께 사용할 때
public abstract class DataProcessor
{
    protected abstract IDataReader CreateReader();
    protected abstract IDataWriter CreateWriter();
    
    public async Task ProcessDataAsync(string source, string destination)
    {
        var reader = CreateReader();
        var writer = CreateWriter();
        
        var data = await reader.ReadAsync(source);
        var processedData = await ProcessData(data);
        await writer.WriteAsync(destination, processedData);
    }
    
    protected abstract Task<Data> ProcessData(Data data);
}
```

### Abstract Factory를 사용할 때
```csharp
// 관련된 객체들의 제품군을 생성해야 할 때
// 제품군 간의 일관성이 중요할 때
public interface ICloudServiceFactory
{
    IStorageService CreateStorage();
    IComputeService CreateCompute();
    IDatabaseService CreateDatabase();
    IMessagingService CreateMessaging();
}

public class AzureServiceFactory : ICloudServiceFactory
{
    public IStorageService CreateStorage() => new AzureBlobStorage();
    public IComputeService CreateCompute() => new AzureVirtualMachine();
    public IDatabaseService CreateDatabase() => new AzureSqlDatabase();
    public IMessagingService CreateMessaging() => new AzureServiceBus();
}

public class AwsServiceFactory : ICloudServiceFactory
{
    public IStorageService CreateStorage() => new S3Storage();
    public IComputeService CreateCompute() => new EC2Instance();
    public IDatabaseService CreateDatabase() => new RdsDatabase();
    public IMessagingService CreateMessaging() => new SqsMessaging();
}
```

## 마무리

Factory 패턴들의 주요 특징:

1. **Simple Factory**
   - 장점: 구현이 간단, 클라이언트 코드가 깔끔
   - 단점: OCP 위반, 새 타입 추가 시 수정 필요

2. **Factory Method**
   - 장점: OCP 준수, 확장 가능
   - 단점: 클래스 수 증가, 복잡도 상승

3. **Abstract Factory**
   - 장점: 제품군 일관성 보장, 제품군 교체 용이
   - 단점: 새 제품 추가 시 모든 Factory 수정 필요

다음 장에서는 Builder 패턴과 Fluent API에 대해 알아보겠습니다.