# C#과 .NET의 패턴 지원 기능

## 개요

C#과 .NET은 디자인 패턴을 쉽게 구현할 수 있도록 다양한 언어 기능과 프레임워크 지원을 제공합니다. 이 장에서는 패턴 구현을 더 깔끔하고 효율적으로 만드는 C#의 주요 기능들을 살펴봅니다.

## 제네릭과 패턴

### 제네릭을 활용한 Repository 패턴

```csharp
// 기본 제네릭 Repository
public interface IRepository<T> where T : class
{
    T GetById(int id);
    IEnumerable<T> GetAll();
    void Add(T entity);
    void Update(T entity);
    void Delete(T entity);
    void SaveChanges();
}

// 제약 조건을 활용한 고급 Repository
public interface IEntity
{
    int Id { get; set; }
    DateTime CreatedAt { get; set; }
    DateTime? UpdatedAt { get; set; }
    bool IsDeleted { get; set; }
}

public interface IAdvancedRepository<T> where T : class, IEntity, new()
{
    T GetById(int id);
    IQueryable<T> Query();
    void Add(T entity);
    void Update(T entity);
    void SoftDelete(T entity);
    Task<int> SaveChangesAsync();
}

public class EntityFrameworkRepository<T> : IAdvancedRepository<T> 
    where T : class, IEntity, new()
{
    private readonly DbContext _context;
    private readonly DbSet<T> _dbSet;
    
    public EntityFrameworkRepository(DbContext context)
    {
        _context = context;
        _dbSet = context.Set<T>();
    }
    
    public T GetById(int id)
    {
        return _dbSet.FirstOrDefault(e => e.Id == id && !e.IsDeleted);
    }
    
    public IQueryable<T> Query()
    {
        return _dbSet.Where(e => !e.IsDeleted);
    }
    
    public void Add(T entity)
    {
        entity.CreatedAt = DateTime.UtcNow;
        _dbSet.Add(entity);
    }
    
    public void Update(T entity)
    {
        entity.UpdatedAt = DateTime.UtcNow;
        _context.Entry(entity).State = EntityState.Modified;
    }
    
    public void SoftDelete(T entity)
    {
        entity.IsDeleted = true;
        entity.UpdatedAt = DateTime.UtcNow;
        Update(entity);
    }
    
    public async Task<int> SaveChangesAsync()
    {
        return await _context.SaveChangesAsync();
    }
}
```

### 제네릭 Factory 패턴

```csharp
// 제네릭 Factory 인터페이스
public interface IFactory<T>
{
    T Create();
}

public interface IFactory<TProduct, TConfig>
{
    TProduct Create(TConfig configuration);
}

// 구현 예시
public class ConnectionFactory<TConnection> : IFactory<TConnection, string> 
    where TConnection : IDbConnection, new()
{
    public TConnection Create(string connectionString)
    {
        var connection = new TConnection();
        connection.ConnectionString = connectionString;
        return connection;
    }
}

// 사용
var sqlFactory = new ConnectionFactory<SqlConnection>();
var sqlConnection = sqlFactory.Create("Server=localhost;Database=MyDb;");

var npgsqlFactory = new ConnectionFactory<NpgsqlConnection>();
var npgsqlConnection = npgsqlFactory.Create("Host=localhost;Database=MyDb;");
```

## LINQ와 함수형 패턴

### Chain of Responsibility with LINQ

```csharp
public abstract class ValidationRule<T>
{
    public abstract bool IsSatisfiedBy(T entity);
    public abstract string ErrorMessage { get; }
}

public class ValidationChain<T>
{
    private readonly List<ValidationRule<T>> _rules = new();
    
    public ValidationChain<T> AddRule(ValidationRule<T> rule)
    {
        _rules.Add(rule);
        return this;
    }
    
    public ValidationResult Validate(T entity)
    {
        var errors = _rules
            .Where(rule => !rule.IsSatisfiedBy(entity))
            .Select(rule => rule.ErrorMessage)
            .ToList();
            
        return new ValidationResult
        {
            IsValid = !errors.Any(),
            Errors = errors
        };
    }
}

// LINQ를 활용한 Specification 패턴
public interface ISpecification<T>
{
    Expression<Func<T, bool>> ToExpression();
}

public class Specification<T> : ISpecification<T>
{
    private readonly Expression<Func<T, bool>> _expression;
    
    public Specification(Expression<Func<T, bool>> expression)
    {
        _expression = expression;
    }
    
    public Expression<Func<T, bool>> ToExpression() => _expression;
    
    public ISpecification<T> And(ISpecification<T> specification)
    {
        var invokedExpr = Expression.Invoke(specification.ToExpression(), 
            _expression.Parameters.Cast<Expression>());
            
        var andExpr = Expression.AndAlso(_expression.Body, invokedExpr);
        
        return new Specification<T>(
            Expression.Lambda<Func<T, bool>>(andExpr, _expression.Parameters));
    }
    
    public ISpecification<T> Or(ISpecification<T> specification)
    {
        var invokedExpr = Expression.Invoke(specification.ToExpression(), 
            _expression.Parameters.Cast<Expression>());
            
        var orExpr = Expression.OrElse(_expression.Body, invokedExpr);
        
        return new Specification<T>(
            Expression.Lambda<Func<T, bool>>(orExpr, _expression.Parameters));
    }
}

// 사용 예시
public class ProductSpecifications
{
    public static ISpecification<Product> InStock()
    {
        return new Specification<Product>(p => p.StockQuantity > 0);
    }
    
    public static ISpecification<Product> InPriceRange(decimal min, decimal max)
    {
        return new Specification<Product>(p => p.Price >= min && p.Price <= max);
    }
    
    public static ISpecification<Product> ByCategory(string category)
    {
        return new Specification<Product>(p => p.Category == category);
    }
}

// 조합 사용
var affordableInStockElectronics = ProductSpecifications.InStock()
    .And(ProductSpecifications.InPriceRange(100, 1000))
    .And(ProductSpecifications.ByCategory("Electronics"));

var products = repository.Query()
    .Where(affordableInStockElectronics.ToExpression())
    .ToList();
```

### 함수형 Builder 패턴

```csharp
public class EmailBuilder
{
    private readonly Email _email = new();
    
    public EmailBuilder To(string address) => 
        With(e => e.ToAddress = address);
    
    public EmailBuilder Cc(string address) => 
        With(e => e.CcAddresses.Add(address));
    
    public EmailBuilder Subject(string subject) => 
        With(e => e.Subject = subject);
    
    public EmailBuilder Body(string body) => 
        With(e => e.Body = body);
    
    public EmailBuilder Attachment(string filePath) => 
        With(e => e.Attachments.Add(filePath));
    
    private EmailBuilder With(Action<Email> action)
    {
        action(_email);
        return this;
    }
    
    public Email Build() => _email;
    
    // Implicit operator for convenience
    public static implicit operator Email(EmailBuilder builder) => builder.Build();
}

// 사용
Email email = new EmailBuilder()
    .To("user@example.com")
    .Subject("Hello")
    .Body("This is a test email")
    .Cc("cc@example.com")
    .Attachment("document.pdf");
```

## 최신 C# 기능 활용

### Pattern Matching과 Switch Expressions

```csharp
// C# 8.0 Switch Expression을 활용한 Factory
public class ShapeFactory
{
    public IShape CreateShape(string shapeType, params object[] parameters)
    {
        return shapeType switch
        {
            "Circle" when parameters.Length >= 1 => 
                new Circle((double)parameters[0]),
                
            "Rectangle" when parameters.Length >= 2 => 
                new Rectangle((double)parameters[0], (double)parameters[1]),
                
            "Triangle" when parameters.Length >= 3 => 
                new Triangle((double)parameters[0], (double)parameters[1], (double)parameters[2]),
                
            _ => throw new ArgumentException($"Unknown shape type: {shapeType}")
        };
    }
}

// Pattern Matching을 활용한 Visitor 패턴
public abstract class PaymentMethod
{
    public abstract T Accept<T>(IPaymentVisitor<T> visitor);
}

public class CreditCard : PaymentMethod
{
    public string CardNumber { get; set; }
    public override T Accept<T>(IPaymentVisitor<T> visitor) => visitor.Visit(this);
}

public class PayPal : PaymentMethod
{
    public string Email { get; set; }
    public override T Accept<T>(IPaymentVisitor<T> visitor) => visitor.Visit(this);
}

// C# 9.0 Pattern Matching 활용
public class PaymentProcessor
{
    public PaymentResult ProcessPayment(PaymentMethod payment, decimal amount)
    {
        return payment switch
        {
            CreditCard { CardNumber: var number } when IsValidCard(number) => 
                ProcessCreditCard(number, amount),
                
            PayPal { Email: var email } when IsValidEmail(email) => 
                ProcessPayPal(email, amount),
                
            CreditCard => new PaymentResult { Success = false, Message = "Invalid card" },
            PayPal => new PaymentResult { Success = false, Message = "Invalid email" },
            
            _ => new PaymentResult { Success = false, Message = "Unsupported payment method" }
        };
    }
}
```

### Records와 불변 패턴

```csharp
// C# 9.0 Records를 활용한 Value Object
public record Money(decimal Amount, string Currency)
{
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("Cannot add different currencies");
            
        return this with { Amount = Amount + other.Amount };
    }
    
    public Money Multiply(decimal factor)
    {
        return this with { Amount = Amount * factor };
    }
}

// Record를 활용한 Command 패턴
public abstract record Command
{
    public DateTime Timestamp { get; init; } = DateTime.UtcNow;
    public Guid Id { get; init; } = Guid.NewGuid();
}

public record CreateOrderCommand(
    string CustomerId,
    List<OrderItem> Items,
    Address ShippingAddress
) : Command;

public record UpdateOrderStatusCommand(
    Guid OrderId,
    OrderStatus NewStatus,
    string Reason
) : Command;

// Immutable Event Sourcing
public abstract record DomainEvent
{
    public DateTime OccurredAt { get; init; } = DateTime.UtcNow;
    public Guid AggregateId { get; init; }
    public int Version { get; init; }
}

public record OrderCreatedEvent(
    Guid OrderId,
    string CustomerId,
    decimal TotalAmount
) : DomainEvent;

public record OrderShippedEvent(
    Guid OrderId,
    string TrackingNumber,
    DateTime ShippedAt
) : DomainEvent;
```

### Nullable Reference Types

```csharp
#nullable enable

public interface IRepository<T> where T : class
{
    T? GetById(int id);
    IEnumerable<T> GetAll();
    Task<T?> GetByIdAsync(int id);
}

public class CustomerService
{
    private readonly IRepository<Customer> _repository;
    
    public CustomerService(IRepository<Customer> repository)
    {
        _repository = repository ?? throw new ArgumentNullException(nameof(repository));
    }
    
    public CustomerDto? GetCustomerInfo(int id)
    {
        var customer = _repository.GetById(id);
        
        // Null 체크가 강제됨
        if (customer is null)
            return null;
            
        return new CustomerDto
        {
            Id = customer.Id,
            Name = customer.Name,
            Email = customer.Email ?? "No email provided"
        };
    }
    
    // Null-forgiving operator 사용
    public void UpdateCustomer(Customer customer)
    {
        ArgumentNullException.ThrowIfNull(customer);
        
        // 여기서는 customer가 null이 아님을 보장
        _repository.Update(customer!);
    }
}

#nullable restore
```

## 비동기 패턴

### 비동기 Factory 패턴

```csharp
public interface IAsyncFactory<T>
{
    Task<T> CreateAsync();
}

public class DatabaseConnectionFactory : IAsyncFactory<IDbConnection>
{
    private readonly string _connectionString;
    
    public DatabaseConnectionFactory(string connectionString)
    {
        _connectionString = connectionString;
    }
    
    public async Task<IDbConnection> CreateAsync()
    {
        var connection = new SqlConnection(_connectionString);
        await connection.OpenAsync();
        
        // 추가 초기화 작업
        await InitializeConnectionAsync(connection);
        
        return connection;
    }
    
    private async Task InitializeConnectionAsync(IDbConnection connection)
    {
        // 세션 설정, 타임아웃 설정 등
        await Task.Delay(100); // 시뮬레이션
    }
}
```

### 비동기 Observer 패턴

```csharp
public interface IAsyncObserver<T>
{
    Task OnNextAsync(T value);
    Task OnErrorAsync(Exception error);
    Task OnCompletedAsync();
}

public interface IAsyncObservable<T>
{
    Task<IDisposable> SubscribeAsync(IAsyncObserver<T> observer);
}

public class StockPriceService : IAsyncObservable<StockPrice>
{
    private readonly List<IAsyncObserver<StockPrice>> _observers = new();
    
    public async Task<IDisposable> SubscribeAsync(IAsyncObserver<StockPrice> observer)
    {
        _observers.Add(observer);
        
        // 초기 데이터 전송
        var currentPrice = await GetCurrentPriceAsync();
        await observer.OnNextAsync(currentPrice);
        
        return new Unsubscriber(_observers, observer);
    }
    
    public async Task NotifyPriceChangeAsync(StockPrice newPrice)
    {
        var tasks = _observers.Select(observer => observer.OnNextAsync(newPrice));
        await Task.WhenAll(tasks);
    }
    
    private class Unsubscriber : IDisposable
    {
        private readonly List<IAsyncObserver<StockPrice>> _observers;
        private readonly IAsyncObserver<StockPrice> _observer;
        
        public Unsubscriber(
            List<IAsyncObserver<StockPrice>> observers,
            IAsyncObserver<StockPrice> observer)
        {
            _observers = observers;
            _observer = observer;
        }
        
        public void Dispose()
        {
            _observers.Remove(_observer);
        }
    }
}
```

## Attributes와 Reflection

### Attribute 기반 Command 패턴

```csharp
[AttributeUsage(AttributeTargets.Method)]
public class CommandHandlerAttribute : Attribute
{
    public Type CommandType { get; }
    
    public CommandHandlerAttribute(Type commandType)
    {
        CommandType = commandType;
    }
}

public interface ICommand { }

public interface ICommandHandler<TCommand> where TCommand : ICommand
{
    Task HandleAsync(TCommand command);
}

public class CommandDispatcher
{
    private readonly Dictionary<Type, object> _handlers = new();
    
    public CommandDispatcher(Assembly assembly)
    {
        RegisterHandlers(assembly);
    }
    
    private void RegisterHandlers(Assembly assembly)
    {
        var handlerTypes = assembly.GetTypes()
            .Where(t => t.GetInterfaces()
                .Any(i => i.IsGenericType && 
                          i.GetGenericTypeDefinition() == typeof(ICommandHandler<>)));
                          
        foreach (var handlerType in handlerTypes)
        {
            var commandType = handlerType.GetInterfaces()
                .First(i => i.IsGenericType && 
                           i.GetGenericTypeDefinition() == typeof(ICommandHandler<>))
                .GetGenericArguments()[0];
                
            _handlers[commandType] = Activator.CreateInstance(handlerType)!;
        }
    }
    
    public async Task DispatchAsync<TCommand>(TCommand command) where TCommand : ICommand
    {
        if (_handlers.TryGetValue(typeof(TCommand), out var handler))
        {
            await ((ICommandHandler<TCommand>)handler).HandleAsync(command);
        }
        else
        {
            throw new InvalidOperationException(
                $"No handler registered for command type {typeof(TCommand).Name}");
        }
    }
}
```

### Attribute 기반 Validation

```csharp
[AttributeUsage(AttributeTargets.Property)]
public abstract class ValidationAttribute : Attribute
{
    public abstract bool IsValid(object? value);
    public abstract string ErrorMessage { get; }
}

public class RequiredAttribute : ValidationAttribute
{
    public override bool IsValid(object? value)
    {
        return value != null && !string.IsNullOrWhiteSpace(value.ToString());
    }
    
    public override string ErrorMessage => "This field is required";
}

public class RangeAttribute : ValidationAttribute
{
    public double Minimum { get; }
    public double Maximum { get; }
    
    public RangeAttribute(double minimum, double maximum)
    {
        Minimum = minimum;
        Maximum = maximum;
    }
    
    public override bool IsValid(object? value)
    {
        if (value == null) return false;
        
        if (double.TryParse(value.ToString(), out var doubleValue))
        {
            return doubleValue >= Minimum && doubleValue <= Maximum;
        }
        
        return false;
    }
    
    public override string ErrorMessage => 
        $"Value must be between {Minimum} and {Maximum}";
}

public class Validator
{
    public static ValidationResult Validate<T>(T obj) where T : class
    {
        var errors = new List<string>();
        var properties = typeof(T).GetProperties();
        
        foreach (var property in properties)
        {
            var validationAttributes = property
                .GetCustomAttributes<ValidationAttribute>()
                .ToList();
                
            if (!validationAttributes.Any()) continue;
            
            var value = property.GetValue(obj);
            
            foreach (var attribute in validationAttributes)
            {
                if (!attribute.IsValid(value))
                {
                    errors.Add($"{property.Name}: {attribute.ErrorMessage}");
                }
            }
        }
        
        return new ValidationResult
        {
            IsValid = !errors.Any(),
            Errors = errors
        };
    }
}

// 사용 예시
public class Product
{
    [Required]
    public string Name { get; set; }
    
    [Range(0, 10000)]
    public decimal Price { get; set; }
    
    [Range(0, int.MaxValue)]
    public int StockQuantity { get; set; }
}
```

## Expression Trees와 동적 패턴

### Dynamic Specification Builder

```csharp
public class DynamicSpecificationBuilder<T>
{
    private Expression<Func<T, bool>>? _expression;
    
    public DynamicSpecificationBuilder<T> Where(string propertyName, 
        ComparisonOperator op, object value)
    {
        var parameter = Expression.Parameter(typeof(T), "x");
        var property = Expression.Property(parameter, propertyName);
        var constant = Expression.Constant(value);
        
        Expression comparison = op switch
        {
            ComparisonOperator.Equal => Expression.Equal(property, constant),
            ComparisonOperator.NotEqual => Expression.NotEqual(property, constant),
            ComparisonOperator.GreaterThan => Expression.GreaterThan(property, constant),
            ComparisonOperator.LessThan => Expression.LessThan(property, constant),
            ComparisonOperator.Contains => Expression.Call(property, 
                typeof(string).GetMethod("Contains", new[] { typeof(string) })!, constant),
            _ => throw new NotSupportedException($"Operator {op} is not supported")
        };
        
        var lambda = Expression.Lambda<Func<T, bool>>(comparison, parameter);
        
        _expression = _expression == null 
            ? lambda 
            : Expression.Lambda<Func<T, bool>>(
                Expression.AndAlso(_expression.Body, lambda.Body), parameter);
                
        return this;
    }
    
    public Expression<Func<T, bool>> Build()
    {
        return _expression ?? (x => true);
    }
}

public enum ComparisonOperator
{
    Equal,
    NotEqual,
    GreaterThan,
    LessThan,
    Contains
}

// 사용 예시
var spec = new DynamicSpecificationBuilder<Product>()
    .Where("Category", ComparisonOperator.Equal, "Electronics")
    .Where("Price", ComparisonOperator.LessThan, 1000m)
    .Where("Name", ComparisonOperator.Contains, "Phone")
    .Build();

var products = dbContext.Products.Where(spec).ToList();
```

## 마무리

C#과 .NET의 언어 기능들은 디자인 패턴을 더 효과적으로 구현할 수 있게 해줍니다:

1. **제네릭**: 타입 안전성과 재사용성 향상
2. **LINQ**: 함수형 프로그래밍 패턴 지원
3. **Pattern Matching**: 복잡한 조건 로직 단순화
4. **Records**: 불변 객체 패턴 쉽게 구현
5. **Async/Await**: 비동기 패턴 구현 간소화
6. **Attributes**: 메타데이터 기반 패턴 구현
7. **Expression Trees**: 동적 쿼리 및 사양 패턴

다음 장에서는 Singleton 패턴과 의존성 주입에 대해 자세히 알아보겠습니다.