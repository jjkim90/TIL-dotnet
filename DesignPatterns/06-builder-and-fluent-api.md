# Builder 패턴과 Fluent API

## 개요

Builder 패턴은 복잡한 객체를 단계별로 생성할 수 있게 해주는 생성 패턴입니다. Fluent API는 메서드 체이닝을 통해 더 읽기 쉽고 표현력 있는 코드를 작성할 수 있게 해주는 기법으로, Builder 패턴과 함께 자주 사용됩니다.

## 기본 Builder 패턴

### 클래식 Builder 구현

```csharp
// Product 클래스
public class Computer
{
    public string CPU { get; set; }
    public string GPU { get; set; }
    public int RAM { get; set; }
    public int Storage { get; set; }
    public string MotherBoard { get; set; }
    public string PowerSupply { get; set; }
    public List<string> Peripherals { get; set; } = new();
    
    public override string ToString()
    {
        return $"Computer Configuration:\n" +
               $"  CPU: {CPU}\n" +
               $"  GPU: {GPU}\n" +
               $"  RAM: {RAM}GB\n" +
               $"  Storage: {Storage}GB\n" +
               $"  MotherBoard: {MotherBoard}\n" +
               $"  Power Supply: {PowerSupply}\n" +
               $"  Peripherals: {string.Join(", ", Peripherals)}";
    }
}

// Builder 인터페이스
public interface IComputerBuilder
{
    IComputerBuilder SetCPU(string cpu);
    IComputerBuilder SetGPU(string gpu);
    IComputerBuilder SetRAM(int ram);
    IComputerBuilder SetStorage(int storage);
    IComputerBuilder SetMotherBoard(string motherBoard);
    IComputerBuilder SetPowerSupply(string powerSupply);
    IComputerBuilder AddPeripheral(string peripheral);
    Computer Build();
}

// Concrete Builder
public class ComputerBuilder : IComputerBuilder
{
    private readonly Computer _computer = new();
    
    public IComputerBuilder SetCPU(string cpu)
    {
        _computer.CPU = cpu;
        return this;
    }
    
    public IComputerBuilder SetGPU(string gpu)
    {
        _computer.GPU = gpu;
        return this;
    }
    
    public IComputerBuilder SetRAM(int ram)
    {
        _computer.RAM = ram;
        return this;
    }
    
    public IComputerBuilder SetStorage(int storage)
    {
        _computer.Storage = storage;
        return this;
    }
    
    public IComputerBuilder SetMotherBoard(string motherBoard)
    {
        _computer.MotherBoard = motherBoard;
        return this;
    }
    
    public IComputerBuilder SetPowerSupply(string powerSupply)
    {
        _computer.PowerSupply = powerSupply;
        return this;
    }
    
    public IComputerBuilder AddPeripheral(string peripheral)
    {
        _computer.Peripherals.Add(peripheral);
        return this;
    }
    
    public Computer Build()
    {
        ValidateComputer();
        return _computer;
    }
    
    private void ValidateComputer()
    {
        if (string.IsNullOrEmpty(_computer.CPU))
            throw new InvalidOperationException("CPU is required");
            
        if (string.IsNullOrEmpty(_computer.MotherBoard))
            throw new InvalidOperationException("MotherBoard is required");
            
        if (_computer.RAM <= 0)
            throw new InvalidOperationException("RAM must be greater than 0");
    }
}

// Director (선택적)
public class ComputerShop
{
    private readonly IComputerBuilder _builder;
    
    public ComputerShop(IComputerBuilder builder)
    {
        _builder = builder;
    }
    
    public Computer BuildGamingComputer()
    {
        return _builder
            .SetCPU("Intel Core i9-12900K")
            .SetGPU("NVIDIA RTX 4090")
            .SetRAM(32)
            .SetStorage(2000)
            .SetMotherBoard("ASUS ROG Maximus Z690")
            .SetPowerSupply("1000W Gold")
            .AddPeripheral("RGB Keyboard")
            .AddPeripheral("Gaming Mouse")
            .AddPeripheral("27\" 4K Monitor")
            .Build();
    }
    
    public Computer BuildOfficeComputer()
    {
        return _builder
            .SetCPU("Intel Core i5-12400")
            .SetGPU("Integrated Graphics")
            .SetRAM(16)
            .SetStorage(512)
            .SetMotherBoard("MSI B660M")
            .SetPowerSupply("550W Bronze")
            .AddPeripheral("Wireless Keyboard & Mouse")
            .AddPeripheral("24\" FHD Monitor")
            .Build();
    }
}
```

## Fluent Builder 구현

### 메서드 체이닝을 활용한 Builder

```csharp
public class EmailMessage
{
    public string From { get; set; }
    public List<string> To { get; set; } = new();
    public List<string> Cc { get; set; } = new();
    public List<string> Bcc { get; set; } = new();
    public string Subject { get; set; }
    public string Body { get; set; }
    public bool IsHtml { get; set; }
    public List<EmailAttachment> Attachments { get; set; } = new();
    public Dictionary<string, string> Headers { get; set; } = new();
}

public class EmailAttachment
{
    public string FileName { get; set; }
    public byte[] Content { get; set; }
    public string ContentType { get; set; }
}

public class EmailBuilder
{
    private readonly EmailMessage _email = new();
    
    public EmailBuilder From(string address)
    {
        _email.From = address;
        return this;
    }
    
    public EmailBuilder To(params string[] addresses)
    {
        _email.To.AddRange(addresses);
        return this;
    }
    
    public EmailBuilder Cc(params string[] addresses)
    {
        _email.Cc.AddRange(addresses);
        return this;
    }
    
    public EmailBuilder Bcc(params string[] addresses)
    {
        _email.Bcc.AddRange(addresses);
        return this;
    }
    
    public EmailBuilder WithSubject(string subject)
    {
        _email.Subject = subject;
        return this;
    }
    
    public EmailBuilder WithBody(string body, bool isHtml = false)
    {
        _email.Body = body;
        _email.IsHtml = isHtml;
        return this;
    }
    
    public EmailBuilder WithHtmlBody(string htmlBody)
    {
        return WithBody(htmlBody, true);
    }
    
    public EmailBuilder AttachFile(string fileName, byte[] content, string contentType = "application/octet-stream")
    {
        _email.Attachments.Add(new EmailAttachment
        {
            FileName = fileName,
            Content = content,
            ContentType = contentType
        });
        return this;
    }
    
    public EmailBuilder AddHeader(string key, string value)
    {
        _email.Headers[key] = value;
        return this;
    }
    
    public EmailMessage Build()
    {
        Validate();
        return _email;
    }
    
    private void Validate()
    {
        if (string.IsNullOrWhiteSpace(_email.From))
            throw new InvalidOperationException("From address is required");
            
        if (!_email.To.Any())
            throw new InvalidOperationException("At least one recipient is required");
            
        if (string.IsNullOrWhiteSpace(_email.Subject))
            throw new InvalidOperationException("Subject is required");
    }
}

// 사용 예시
var email = new EmailBuilder()
    .From("sender@example.com")
    .To("recipient1@example.com", "recipient2@example.com")
    .Cc("cc@example.com")
    .WithSubject("Monthly Report")
    .WithHtmlBody("<h1>Report</h1><p>Please find attached the monthly report.</p>")
    .AttachFile("report.pdf", File.ReadAllBytes("report.pdf"), "application/pdf")
    .AddHeader("X-Priority", "High")
    .Build();
```

### 제네릭 Fluent Builder

```csharp
public abstract class FluentBuilder<TBuilder, TProduct> 
    where TBuilder : FluentBuilder<TBuilder, TProduct>
    where TProduct : new()
{
    protected TProduct Product { get; }
    
    protected FluentBuilder()
    {
        Product = new TProduct();
    }
    
    public TProduct Build()
    {
        Validate();
        return Product;
    }
    
    protected abstract void Validate();
    
    protected TBuilder This => (TBuilder)this;
}

// 사용 예시
public class Person
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public DateTime DateOfBirth { get; set; }
    public string Email { get; set; }
    public string PhoneNumber { get; set; }
    public Address Address { get; set; }
}

public class PersonBuilder : FluentBuilder<PersonBuilder, Person>
{
    public PersonBuilder WithName(string firstName, string lastName)
    {
        Product.FirstName = firstName;
        Product.LastName = lastName;
        return This;
    }
    
    public PersonBuilder BornOn(DateTime dateOfBirth)
    {
        Product.DateOfBirth = dateOfBirth;
        return This;
    }
    
    public PersonBuilder WithEmail(string email)
    {
        Product.Email = email;
        return This;
    }
    
    public PersonBuilder WithPhone(string phoneNumber)
    {
        Product.PhoneNumber = phoneNumber;
        return This;
    }
    
    public PersonBuilder LivesAt(Action<AddressBuilder> addressBuilder)
    {
        var builder = new AddressBuilder();
        addressBuilder(builder);
        Product.Address = builder.Build();
        return This;
    }
    
    protected override void Validate()
    {
        if (string.IsNullOrWhiteSpace(Product.FirstName))
            throw new InvalidOperationException("First name is required");
            
        if (string.IsNullOrWhiteSpace(Product.LastName))
            throw new InvalidOperationException("Last name is required");
            
        if (Product.DateOfBirth > DateTime.Now)
            throw new InvalidOperationException("Date of birth cannot be in the future");
    }
}

public class AddressBuilder : FluentBuilder<AddressBuilder, Address>
{
    public AddressBuilder Street(string street)
    {
        Product.Street = street;
        return This;
    }
    
    public AddressBuilder City(string city)
    {
        Product.City = city;
        return This;
    }
    
    public AddressBuilder State(string state)
    {
        Product.State = state;
        return This;
    }
    
    public AddressBuilder ZipCode(string zipCode)
    {
        Product.ZipCode = zipCode;
        return This;
    }
    
    public AddressBuilder Country(string country)
    {
        Product.Country = country;
        return This;
    }
    
    protected override void Validate()
    {
        // Address validation logic
    }
}

// 사용
var person = new PersonBuilder()
    .WithName("John", "Doe")
    .BornOn(new DateTime(1990, 5, 15))
    .WithEmail("john.doe@example.com")
    .WithPhone("+1-555-123-4567")
    .LivesAt(address => address
        .Street("123 Main St")
        .City("Anytown")
        .State("CA")
        .ZipCode("12345")
        .Country("USA"))
    .Build();
```

## Entity Framework의 ModelBuilder

### EF Core에서의 Fluent API 사용

```csharp
public class BloggingContext : DbContext
{
    public DbSet<Blog> Blogs { get; set; }
    public DbSet<Post> Posts { get; set; }
    public DbSet<Author> Authors { get; set; }
    public DbSet<Tag> Tags { get; set; }
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Blog 엔티티 구성
        modelBuilder.Entity<Blog>(entity =>
        {
            entity.ToTable("Blogs", "blogging");
            
            entity.HasKey(b => b.BlogId);
            
            entity.Property(b => b.Url)
                .IsRequired()
                .HasMaxLength(500);
                
            entity.Property(b => b.Rating)
                .HasDefaultValue(0)
                .HasColumnType("decimal(3,2)");
                
            entity.HasIndex(b => b.Url)
                .IsUnique()
                .HasDatabaseName("IX_Blogs_Url");
                
            // 관계 설정
            entity.HasMany(b => b.Posts)
                .WithOne(p => p.Blog)
                .HasForeignKey(p => p.BlogId)
                .OnDelete(DeleteBehavior.Cascade);
                
            entity.HasOne(b => b.Author)
                .WithMany(a => a.Blogs)
                .HasForeignKey(b => b.AuthorId);
        });
        
        // Post 엔티티 구성
        modelBuilder.Entity<Post>(entity =>
        {
            entity.HasKey(p => p.PostId);
            
            entity.Property(p => p.Title)
                .IsRequired()
                .HasMaxLength(200);
                
            entity.Property(p => p.Content)
                .HasColumnType("nvarchar(max)");
                
            entity.Property(p => p.PublishedOn)
                .HasDefaultValueSql("GETUTCDATE()");
                
            // 다대다 관계
            entity.HasMany(p => p.Tags)
                .WithMany(t => t.Posts)
                .UsingEntity<PostTag>(
                    j => j.HasOne(pt => pt.Tag)
                          .WithMany(t => t.PostTags)
                          .HasForeignKey(pt => pt.TagId),
                    j => j.HasOne(pt => pt.Post)
                          .WithMany(p => p.PostTags)
                          .HasForeignKey(pt => pt.PostId),
                    j =>
                    {
                        j.HasKey(pt => new { pt.PostId, pt.TagId });
                        j.ToTable("PostTags");
                    });
        });
        
        // Value Object 구성
        modelBuilder.Entity<Author>(entity =>
        {
            entity.OwnsOne(a => a.ContactInfo, contact =>
            {
                contact.Property(c => c.Email)
                    .HasColumnName("Email")
                    .HasMaxLength(256);
                    
                contact.Property(c => c.Phone)
                    .HasColumnName("PhoneNumber")
                    .HasMaxLength(20);
                    
                contact.OwnsOne(c => c.Address, address =>
                {
                    address.Property(a => a.Street).HasColumnName("Street");
                    address.Property(a => a.City).HasColumnName("City");
                    address.Property(a => a.Country).HasColumnName("Country");
                });
            });
        });
        
        // Global Query Filter
        modelBuilder.Entity<Post>()
            .HasQueryFilter(p => !p.IsDeleted);
    }
}

// 커스텀 Extension Methods
public static class ModelBuilderExtensions
{
    public static void ConfigureAuditableEntities(this ModelBuilder modelBuilder)
    {
        foreach (var entityType in modelBuilder.Model.GetEntityTypes())
        {
            if (typeof(IAuditable).IsAssignableFrom(entityType.ClrType))
            {
                modelBuilder.Entity(entityType.ClrType)
                    .Property<DateTime>("CreatedAt")
                    .HasDefaultValueSql("GETUTCDATE()");
                    
                modelBuilder.Entity(entityType.ClrType)
                    .Property<DateTime?>("UpdatedAt");
                    
                modelBuilder.Entity(entityType.ClrType)
                    .Property<string>("CreatedBy")
                    .HasMaxLength(256);
                    
                modelBuilder.Entity(entityType.ClrType)
                    .Property<string>("UpdatedBy")
                    .HasMaxLength(256);
            }
        }
    }
}
```

## 복잡한 Builder 시나리오

### 조건부 Builder

```csharp
public class QueryBuilder
{
    private readonly StringBuilder _query = new();
    private readonly List<string> _conditions = new();
    private readonly List<string> _orderBy = new();
    private int? _limit;
    private int? _offset;
    
    public QueryBuilder Select(params string[] columns)
    {
        _query.Clear();
        _query.Append("SELECT ");
        _query.Append(columns.Any() ? string.Join(", ", columns) : "*");
        return this;
    }
    
    public QueryBuilder From(string table)
    {
        _query.Append($" FROM {table}");
        return this;
    }
    
    public QueryBuilder Where(string condition)
    {
        _conditions.Add(condition);
        return this;
    }
    
    public QueryBuilder WhereIf(bool condition, string whereClause)
    {
        if (condition)
        {
            _conditions.Add(whereClause);
        }
        return this;
    }
    
    public QueryBuilder OrderBy(string column, bool ascending = true)
    {
        _orderBy.Add($"{column} {(ascending ? "ASC" : "DESC")}");
        return this;
    }
    
    public QueryBuilder Limit(int limit)
    {
        _limit = limit;
        return this;
    }
    
    public QueryBuilder Offset(int offset)
    {
        _offset = offset;
        return this;
    }
    
    public string Build()
    {
        var query = new StringBuilder(_query.ToString());
        
        if (_conditions.Any())
        {
            query.Append(" WHERE ");
            query.Append(string.Join(" AND ", _conditions));
        }
        
        if (_orderBy.Any())
        {
            query.Append(" ORDER BY ");
            query.Append(string.Join(", ", _orderBy));
        }
        
        if (_limit.HasValue)
        {
            query.Append($" LIMIT {_limit.Value}");
        }
        
        if (_offset.HasValue)
        {
            query.Append($" OFFSET {_offset.Value}");
        }
        
        return query.ToString();
    }
}

// 사용 예시
public class ProductRepository
{
    public string BuildProductQuery(ProductFilter filter)
    {
        var query = new QueryBuilder()
            .Select("Id", "Name", "Price", "Category")
            .From("Products")
            .WhereIf(filter.MinPrice.HasValue, $"Price >= {filter.MinPrice}")
            .WhereIf(filter.MaxPrice.HasValue, $"Price <= {filter.MaxPrice}")
            .WhereIf(!string.IsNullOrEmpty(filter.Category), $"Category = '{filter.Category}'")
            .WhereIf(filter.InStock, "StockQuantity > 0")
            .OrderBy(filter.SortBy ?? "Name", filter.SortAscending)
            .Limit(filter.PageSize)
            .Offset((filter.Page - 1) * filter.PageSize)
            .Build();
            
        return query;
    }
}
```

### Nested Builder Pattern

```csharp
public class HttpRequestBuilder
{
    private readonly HttpRequestMessage _request = new();
    private readonly Dictionary<string, string> _headers = new();
    private readonly Dictionary<string, string> _queryParams = new();
    
    public HttpRequestBuilder WithMethod(HttpMethod method)
    {
        _request.Method = method;
        return this;
    }
    
    public HttpRequestBuilder WithUrl(string url)
    {
        _request.RequestUri = new Uri(url);
        return this;
    }
    
    public HttpRequestBuilder WithHeader(string key, string value)
    {
        _headers[key] = value;
        return this;
    }
    
    public HttpRequestBuilder WithQueryParam(string key, string value)
    {
        _queryParams[key] = value;
        return this;
    }
    
    public HttpRequestBuilder WithJsonBody<T>(T body)
    {
        var json = JsonSerializer.Serialize(body);
        _request.Content = new StringContent(json, Encoding.UTF8, "application/json");
        return this;
    }
    
    public HttpRequestBuilder WithFormData(Action<FormDataBuilder> formBuilder)
    {
        var builder = new FormDataBuilder();
        formBuilder(builder);
        _request.Content = builder.Build();
        return this;
    }
    
    public HttpRequestMessage Build()
    {
        // Add headers
        foreach (var header in _headers)
        {
            _request.Headers.TryAddWithoutValidation(header.Key, header.Value);
        }
        
        // Add query parameters
        if (_queryParams.Any() && _request.RequestUri != null)
        {
            var uriBuilder = new UriBuilder(_request.RequestUri);
            var query = HttpUtility.ParseQueryString(uriBuilder.Query);
            
            foreach (var param in _queryParams)
            {
                query[param.Key] = param.Value;
            }
            
            uriBuilder.Query = query.ToString();
            _request.RequestUri = uriBuilder.Uri;
        }
        
        return _request;
    }
}

public class FormDataBuilder
{
    private readonly MultipartFormDataContent _content = new();
    
    public FormDataBuilder AddField(string name, string value)
    {
        _content.Add(new StringContent(value), name);
        return this;
    }
    
    public FormDataBuilder AddFile(string name, byte[] fileContent, string fileName)
    {
        var content = new ByteArrayContent(fileContent);
        content.Headers.ContentDisposition = new ContentDispositionHeaderValue("form-data")
        {
            Name = name,
            FileName = fileName
        };
        _content.Add(content);
        return this;
    }
    
    public MultipartFormDataContent Build() => _content;
}

// 사용
var request = new HttpRequestBuilder()
    .WithMethod(HttpMethod.Post)
    .WithUrl("https://api.example.com/upload")
    .WithHeader("Authorization", "Bearer token123")
    .WithQueryParam("version", "v2")
    .WithFormData(form => form
        .AddField("title", "My Document")
        .AddField("description", "Important file")
        .AddFile("file", fileBytes, "document.pdf"))
    .Build();
```

## 불변 객체와 Builder

### Immutable Object Builder

```csharp
public class ImmutableConfiguration
{
    public string ConnectionString { get; }
    public int MaxRetries { get; }
    public TimeSpan Timeout { get; }
    public bool EnableLogging { get; }
    public LogLevel MinimumLogLevel { get; }
    public IReadOnlyDictionary<string, string> CustomSettings { get; }
    
    private ImmutableConfiguration(Builder builder)
    {
        ConnectionString = builder.ConnectionString;
        MaxRetries = builder.MaxRetries;
        Timeout = builder.Timeout;
        EnableLogging = builder.EnableLogging;
        MinimumLogLevel = builder.MinimumLogLevel;
        CustomSettings = new Dictionary<string, string>(builder.CustomSettings);
    }
    
    public class Builder
    {
        public string ConnectionString { get; private set; }
        public int MaxRetries { get; private set; } = 3;
        public TimeSpan Timeout { get; private set; } = TimeSpan.FromSeconds(30);
        public bool EnableLogging { get; private set; } = true;
        public LogLevel MinimumLogLevel { get; private set; } = LogLevel.Information;
        public Dictionary<string, string> CustomSettings { get; } = new();
        
        public Builder WithConnectionString(string connectionString)
        {
            ConnectionString = connectionString ?? throw new ArgumentNullException(nameof(connectionString));
            return this;
        }
        
        public Builder WithMaxRetries(int maxRetries)
        {
            if (maxRetries < 0)
                throw new ArgumentException("Max retries must be non-negative", nameof(maxRetries));
                
            MaxRetries = maxRetries;
            return this;
        }
        
        public Builder WithTimeout(TimeSpan timeout)
        {
            if (timeout <= TimeSpan.Zero)
                throw new ArgumentException("Timeout must be positive", nameof(timeout));
                
            Timeout = timeout;
            return this;
        }
        
        public Builder WithLogging(bool enable, LogLevel minimumLevel = LogLevel.Information)
        {
            EnableLogging = enable;
            MinimumLogLevel = minimumLevel;
            return this;
        }
        
        public Builder AddCustomSetting(string key, string value)
        {
            CustomSettings[key] = value;
            return this;
        }
        
        public ImmutableConfiguration Build()
        {
            if (string.IsNullOrWhiteSpace(ConnectionString))
                throw new InvalidOperationException("Connection string is required");
                
            return new ImmutableConfiguration(this);
        }
    }
    
    public static Builder CreateBuilder() => new Builder();
}

// 사용
var config = ImmutableConfiguration.CreateBuilder()
    .WithConnectionString("Server=localhost;Database=MyDb;")
    .WithMaxRetries(5)
    .WithTimeout(TimeSpan.FromMinutes(1))
    .WithLogging(true, LogLevel.Debug)
    .AddCustomSetting("Feature.X", "enabled")
    .AddCustomSetting("BatchSize", "100")
    .Build();
```

## 실제 활용 예제

### ASP.NET Core에서의 Builder 패턴

```csharp
// Startup 구성 Builder
public class ServiceConfigurationBuilder
{
    private readonly IServiceCollection _services;
    private readonly IConfiguration _configuration;
    
    public ServiceConfigurationBuilder(IServiceCollection services, IConfiguration configuration)
    {
        _services = services;
        _configuration = configuration;
    }
    
    public ServiceConfigurationBuilder AddDatabase(Action<DatabaseOptionsBuilder> configure = null)
    {
        var builder = new DatabaseOptionsBuilder(_services, _configuration);
        configure?.Invoke(builder);
        builder.Build();
        return this;
    }
    
    public ServiceConfigurationBuilder AddAuthentication(Action<AuthenticationOptionsBuilder> configure = null)
    {
        var builder = new AuthenticationOptionsBuilder(_services, _configuration);
        configure?.Invoke(builder);
        builder.Build();
        return this;
    }
    
    public ServiceConfigurationBuilder AddCaching(Action<CachingOptionsBuilder> configure = null)
    {
        var builder = new CachingOptionsBuilder(_services, _configuration);
        configure?.Invoke(builder);
        builder.Build();
        return this;
    }
    
    public ServiceConfigurationBuilder AddMessaging(Action<MessagingOptionsBuilder> configure = null)
    {
        var builder = new MessagingOptionsBuilder(_services, _configuration);
        configure?.Invoke(builder);
        builder.Build();
        return this;
    }
}

public class DatabaseOptionsBuilder
{
    private readonly IServiceCollection _services;
    private readonly IConfiguration _configuration;
    private string _provider = "SqlServer";
    private string _connectionString;
    private bool _enableSensitiveDataLogging = false;
    private int _commandTimeout = 30;
    
    public DatabaseOptionsBuilder(IServiceCollection services, IConfiguration configuration)
    {
        _services = services;
        _configuration = configuration;
        _connectionString = configuration.GetConnectionString("DefaultConnection");
    }
    
    public DatabaseOptionsBuilder UseProvider(string provider)
    {
        _provider = provider;
        return this;
    }
    
    public DatabaseOptionsBuilder WithConnectionString(string connectionString)
    {
        _connectionString = connectionString;
        return this;
    }
    
    public DatabaseOptionsBuilder EnableSensitiveDataLogging()
    {
        _enableSensitiveDataLogging = true;
        return this;
    }
    
    public DatabaseOptionsBuilder WithCommandTimeout(int seconds)
    {
        _commandTimeout = seconds;
        return this;
    }
    
    public void Build()
    {
        _services.AddDbContext<ApplicationDbContext>(options =>
        {
            switch (_provider.ToLower())
            {
                case "sqlserver":
                    options.UseSqlServer(_connectionString, sqlOptions =>
                    {
                        sqlOptions.CommandTimeout(_commandTimeout);
                        sqlOptions.EnableRetryOnFailure();
                    });
                    break;
                    
                case "postgresql":
                    options.UseNpgsql(_connectionString, npgsqlOptions =>
                    {
                        npgsqlOptions.CommandTimeout(_commandTimeout);
                    });
                    break;
                    
                case "sqlite":
                    options.UseSqlite(_connectionString);
                    break;
                    
                default:
                    throw new NotSupportedException($"Database provider '{_provider}' is not supported");
            }
            
            if (_enableSensitiveDataLogging)
            {
                options.EnableSensitiveDataLogging();
            }
            
            options.EnableServiceProviderCaching();
        });
    }
}

// Program.cs에서 사용
builder.Services.AddServiceConfiguration(configuration =>
{
    configuration
        .AddDatabase(db => db
            .UseProvider("SqlServer")
            .EnableSensitiveDataLogging()
            .WithCommandTimeout(60))
        .AddAuthentication(auth => auth
            .UseJwtBearer()
            .WithIssuer("https://myapp.com")
            .WithAudience("api")
            .RequireHttpsMetadata())
        .AddCaching(cache => cache
            .UseRedis("localhost:6379")
            .WithExpiration(TimeSpan.FromMinutes(20)))
        .AddMessaging(messaging => messaging
            .UseRabbitMQ("amqp://localhost")
            .WithExchange("myapp-events")
            .EnableRetry(3));
});
```

## 마무리

Builder 패턴과 Fluent API의 주요 장점:

1. **가독성**: 메서드 체이닝으로 읽기 쉬운 코드
2. **유연성**: 선택적 매개변수 처리 용이
3. **불변성**: 불변 객체 생성 지원
4. **검증**: 객체 생성 시 유효성 검사
5. **단계별 생성**: 복잡한 객체를 단계별로 구성

다음 장에서는 Prototype 패턴과 객체 복제에 대해 알아보겠습니다.