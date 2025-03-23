# 엔터프라이즈 패턴

## 개요

대규모 엔터프라이즈 애플리케이션 개발에는 확장 가능하고 유지보수 가능한 아키텍처가 필요합니다. 이 장에서는 Clean Architecture, Domain-Driven Design(DDD), CQRS, Event Sourcing 등 엔터프라이즈급 Blazor 애플리케이션 개발을 위한 고급 패턴과 실무 전략을 학습합니다.

## 1. Clean Architecture

### 1.1 프로젝트 구조

```csharp
// Solution Structure
BlazorEnterprise/
├── src/
│   ├── Core/
│   │   ├── BlazorEnterprise.Domain/
│   │   │   ├── Entities/
│   │   │   ├── ValueObjects/
│   │   │   ├── Aggregates/
│   │   │   ├── Events/
│   │   │   └── Specifications/
│   │   └── BlazorEnterprise.Application/
│   │       ├── Common/
│   │       │   ├── Interfaces/
│   │       │   ├── Mappings/
│   │       │   └── Behaviours/
│   │       ├── Features/
│   │       └── DTOs/
│   ├── Infrastructure/
│   │   ├── BlazorEnterprise.Infrastructure/
│   │   │   ├── Persistence/
│   │   │   ├── Services/
│   │   │   └── Identity/
│   │   └── BlazorEnterprise.Shared/
│   │       ├── Constants/
│   │       └── Extensions/
│   └── Presentation/
│       ├── BlazorEnterprise.Server/
│       └── BlazorEnterprise.Client/
└── tests/
    ├── UnitTests/
    ├── IntegrationTests/
    └── E2ETests/
```

### 1.2 도메인 엔티티

```csharp
// Domain/Entities/BaseEntity.cs
public abstract class BaseEntity
{
    public Guid Id { get; protected set; }
    public DateTime CreatedAt { get; protected set; }
    public string CreatedBy { get; protected set; } = "";
    public DateTime? ModifiedAt { get; protected set; }
    public string? ModifiedBy { get; protected set; }
    public bool IsDeleted { get; protected set; }
    
    private readonly List<IDomainEvent> _domainEvents = new();
    public IReadOnlyCollection<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();
    
    protected BaseEntity()
    {
        Id = Guid.NewGuid();
        CreatedAt = DateTime.UtcNow;
    }
    
    public void AddDomainEvent(IDomainEvent domainEvent)
    {
        _domainEvents.Add(domainEvent);
    }
    
    public void RemoveDomainEvent(IDomainEvent domainEvent)
    {
        _domainEvents.Remove(domainEvent);
    }
    
    public void ClearDomainEvents()
    {
        _domainEvents.Clear();
    }
}

// Domain/Entities/Customer.cs
public class Customer : BaseEntity, IAggregateRoot
{
    private readonly List<Order> _orders = new();
    
    public string Name { get; private set; }
    public Email Email { get; private set; }
    public PhoneNumber PhoneNumber { get; private set; }
    public Address Address { get; private set; }
    public CustomerStatus Status { get; private set; }
    public CustomerTier Tier { get; private set; }
    public decimal CreditLimit { get; private set; }
    
    public IReadOnlyCollection<Order> Orders => _orders.AsReadOnly();
    
    protected Customer() { } // For EF
    
    public Customer(string name, Email email, PhoneNumber phoneNumber, Address address)
    {
        Name = Guard.Against.NullOrWhiteSpace(name, nameof(name));
        Email = Guard.Against.Null(email, nameof(email));
        PhoneNumber = Guard.Against.Null(phoneNumber, nameof(phoneNumber));
        Address = Guard.Against.Null(address, nameof(address));
        
        Status = CustomerStatus.Active;
        Tier = CustomerTier.Basic;
        CreditLimit = 1000m;
        
        AddDomainEvent(new CustomerCreatedEvent(Id, name, email.Value));
    }
    
    public void UpdateContactInfo(Email email, PhoneNumber phoneNumber)
    {
        var oldEmail = Email;
        
        Email = Guard.Against.Null(email, nameof(email));
        PhoneNumber = Guard.Against.Null(phoneNumber, nameof(phoneNumber));
        
        if (oldEmail.Value != email.Value)
        {
            AddDomainEvent(new CustomerEmailChangedEvent(Id, oldEmail.Value, email.Value));
        }
    }
    
    public Order PlaceOrder(IEnumerable<OrderItem> items, ShippingMethod shippingMethod)
    {
        if (Status != CustomerStatus.Active)
        {
            throw new DomainException("Inactive customers cannot place orders");
        }
        
        var order = new Order(this, items, shippingMethod);
        
        var totalAmount = order.TotalAmount;
        var currentOrdersTotal = _orders
            .Where(o => o.Status == OrderStatus.Pending || o.Status == OrderStatus.Processing)
            .Sum(o => o.TotalAmount);
        
        if (currentOrdersTotal + totalAmount > CreditLimit)
        {
            throw new DomainException($"Order exceeds credit limit of {CreditLimit:C}");
        }
        
        _orders.Add(order);
        
        AddDomainEvent(new OrderPlacedEvent(Id, order.Id, totalAmount));
        
        // Check for tier upgrade
        CheckTierUpgrade();
        
        return order;
    }
    
    private void CheckTierUpgrade()
    {
        var totalOrderValue = _orders
            .Where(o => o.Status == OrderStatus.Completed)
            .Sum(o => o.TotalAmount);
        
        var newTier = totalOrderValue switch
        {
            >= 50000 => CustomerTier.Platinum,
            >= 20000 => CustomerTier.Gold,
            >= 5000 => CustomerTier.Silver,
            _ => CustomerTier.Basic
        };
        
        if (newTier > Tier)
        {
            var oldTier = Tier;
            Tier = newTier;
            CreditLimit = newTier switch
            {
                CustomerTier.Platinum => 50000m,
                CustomerTier.Gold => 20000m,
                CustomerTier.Silver => 5000m,
                _ => 1000m
            };
            
            AddDomainEvent(new CustomerTierUpgradedEvent(Id, oldTier, newTier));
        }
    }
}
```

### 1.3 값 객체

```csharp
// Domain/ValueObjects/Email.cs
public class Email : ValueObject
{
    public string Value { get; }
    
    private Email(string value)
    {
        Value = value;
    }
    
    public static Result<Email> Create(string email)
    {
        if (string.IsNullOrWhiteSpace(email))
        {
            return Result<Email>.Failure("Email cannot be empty");
        }
        
        if (!IsValidEmail(email))
        {
            return Result<Email>.Failure("Invalid email format");
        }
        
        return Result<Email>.Success(new Email(email.ToLowerInvariant()));
    }
    
    private static bool IsValidEmail(string email)
    {
        try
        {
            var addr = new System.Net.Mail.MailAddress(email);
            return addr.Address == email;
        }
        catch
        {
            return false;
        }
    }
    
    protected override IEnumerable<object> GetEqualityComponents()
    {
        yield return Value;
    }
}

// Domain/ValueObjects/Address.cs
public class Address : ValueObject
{
    public string Street { get; }
    public string City { get; }
    public string State { get; }
    public string PostalCode { get; }
    public string Country { get; }
    
    private Address(string street, string city, string state, string postalCode, string country)
    {
        Street = street;
        City = city;
        State = state;
        PostalCode = postalCode;
        Country = country;
    }
    
    public static Result<Address> Create(string street, string city, string state, 
        string postalCode, string country)
    {
        if (string.IsNullOrWhiteSpace(street))
            return Result<Address>.Failure("Street is required");
        
        if (string.IsNullOrWhiteSpace(city))
            return Result<Address>.Failure("City is required");
        
        if (string.IsNullOrWhiteSpace(state))
            return Result<Address>.Failure("State is required");
        
        if (string.IsNullOrWhiteSpace(postalCode))
            return Result<Address>.Failure("Postal code is required");
        
        if (string.IsNullOrWhiteSpace(country))
            return Result<Address>.Failure("Country is required");
        
        return Result<Address>.Success(new Address(street, city, state, postalCode, country));
    }
    
    protected override IEnumerable<object> GetEqualityComponents()
    {
        yield return Street;
        yield return City;
        yield return State;
        yield return PostalCode;
        yield return Country;
    }
    
    public override string ToString()
    {
        return $"{Street}, {City}, {State} {PostalCode}, {Country}";
    }
}
```

## 2. CQRS (Command Query Responsibility Segregation)

### 2.1 Command 처리

```csharp
// Application/Features/Customers/Commands/CreateCustomer/CreateCustomerCommand.cs
public class CreateCustomerCommand : IRequest<Result<CustomerDto>>
{
    public string Name { get; set; } = "";
    public string Email { get; set; } = "";
    public string PhoneNumber { get; set; } = "";
    public AddressDto Address { get; set; } = new();
}

// CreateCustomerCommandValidator.cs
public class CreateCustomerCommandValidator : AbstractValidator<CreateCustomerCommand>
{
    public CreateCustomerCommandValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Name is required")
            .MaximumLength(100).WithMessage("Name must not exceed 100 characters");
        
        RuleFor(x => x.Email)
            .NotEmpty().WithMessage("Email is required")
            .EmailAddress().WithMessage("Invalid email format");
        
        RuleFor(x => x.PhoneNumber)
            .NotEmpty().WithMessage("Phone number is required")
            .Matches(@"^\+?[1-9]\d{1,14}$").WithMessage("Invalid phone number format");
        
        RuleFor(x => x.Address).SetValidator(new AddressDtoValidator());
    }
}

// CreateCustomerCommandHandler.cs
public class CreateCustomerCommandHandler : IRequestHandler<CreateCustomerCommand, Result<CustomerDto>>
{
    private readonly IApplicationDbContext _context;
    private readonly IMapper _mapper;
    private readonly IEventBus _eventBus;
    private readonly ILogger<CreateCustomerCommandHandler> _logger;
    
    public CreateCustomerCommandHandler(
        IApplicationDbContext context,
        IMapper mapper,
        IEventBus eventBus,
        ILogger<CreateCustomerCommandHandler> logger)
    {
        _context = context;
        _mapper = mapper;
        _eventBus = eventBus;
        _logger = logger;
    }
    
    public async Task<Result<CustomerDto>> Handle(
        CreateCustomerCommand request, 
        CancellationToken cancellationToken)
    {
        try
        {
            // Create value objects
            var emailResult = Email.Create(request.Email);
            if (emailResult.IsFailure)
                return Result<CustomerDto>.Failure(emailResult.Error);
            
            var phoneResult = PhoneNumber.Create(request.PhoneNumber);
            if (phoneResult.IsFailure)
                return Result<CustomerDto>.Failure(phoneResult.Error);
            
            var addressResult = Address.Create(
                request.Address.Street,
                request.Address.City,
                request.Address.State,
                request.Address.PostalCode,
                request.Address.Country);
            
            if (addressResult.IsFailure)
                return Result<CustomerDto>.Failure(addressResult.Error);
            
            // Check for duplicate email
            var existingCustomer = await _context.Customers
                .AnyAsync(c => c.Email == emailResult.Value, cancellationToken);
            
            if (existingCustomer)
                return Result<CustomerDto>.Failure("Customer with this email already exists");
            
            // Create customer
            var customer = new Customer(
                request.Name,
                emailResult.Value,
                phoneResult.Value,
                addressResult.Value);
            
            _context.Customers.Add(customer);
            
            await _context.SaveChangesAsync(cancellationToken);
            
            // Publish domain events
            foreach (var domainEvent in customer.DomainEvents)
            {
                await _eventBus.PublishAsync(domainEvent, cancellationToken);
            }
            
            _logger.LogInformation("Customer created: {CustomerId}", customer.Id);
            
            return Result<CustomerDto>.Success(_mapper.Map<CustomerDto>(customer));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error creating customer");
            return Result<CustomerDto>.Failure("An error occurred while creating the customer");
        }
    }
}
```

### 2.2 Query 처리

```csharp
// Application/Features/Customers/Queries/GetCustomersPaged/GetCustomersPagedQuery.cs
public class GetCustomersPagedQuery : IRequest<Result<PagedResult<CustomerListDto>>>
{
    public int PageNumber { get; set; } = 1;
    public int PageSize { get; set; } = 10;
    public string? SearchTerm { get; set; }
    public CustomerStatus? Status { get; set; }
    public CustomerTier? Tier { get; set; }
    public string? SortBy { get; set; }
    public bool SortDescending { get; set; }
}

// GetCustomersPagedQueryHandler.cs
public class GetCustomersPagedQueryHandler : 
    IRequestHandler<GetCustomersPagedQuery, Result<PagedResult<CustomerListDto>>>
{
    private readonly IReadOnlyRepository<Customer> _repository;
    private readonly IMapper _mapper;
    
    public GetCustomersPagedQueryHandler(
        IReadOnlyRepository<Customer> repository,
        IMapper mapper)
    {
        _repository = repository;
        _mapper = mapper;
    }
    
    public async Task<Result<PagedResult<CustomerListDto>>> Handle(
        GetCustomersPagedQuery request,
        CancellationToken cancellationToken)
    {
        var specification = new CustomerSearchSpecification(
            request.SearchTerm,
            request.Status,
            request.Tier);
        
        var customers = await _repository
            .ListAsync(specification, cancellationToken);
        
        // Apply sorting
        customers = ApplySorting(customers, request.SortBy, request.SortDescending);
        
        // Apply paging
        var totalCount = customers.Count();
        var items = customers
            .Skip((request.PageNumber - 1) * request.PageSize)
            .Take(request.PageSize)
            .ToList();
        
        var dtos = _mapper.Map<List<CustomerListDto>>(items);
        
        var pagedResult = new PagedResult<CustomerListDto>
        {
            Items = dtos,
            PageNumber = request.PageNumber,
            PageSize = request.PageSize,
            TotalCount = totalCount,
            TotalPages = (int)Math.Ceiling(totalCount / (double)request.PageSize)
        };
        
        return Result<PagedResult<CustomerListDto>>.Success(pagedResult);
    }
    
    private IEnumerable<Customer> ApplySorting(
        IEnumerable<Customer> customers,
        string? sortBy,
        bool descending)
    {
        return (sortBy?.ToLower()) switch
        {
            "name" => descending 
                ? customers.OrderByDescending(c => c.Name)
                : customers.OrderBy(c => c.Name),
            "email" => descending
                ? customers.OrderByDescending(c => c.Email.Value)
                : customers.OrderBy(c => c.Email.Value),
            "tier" => descending
                ? customers.OrderByDescending(c => c.Tier)
                : customers.OrderBy(c => c.Tier),
            "createdat" => descending
                ? customers.OrderByDescending(c => c.CreatedAt)
                : customers.OrderBy(c => c.CreatedAt),
            _ => customers.OrderBy(c => c.Name)
        };
    }
}
```

## 3. Event Sourcing

### 3.1 이벤트 스토어

```csharp
// Infrastructure/EventSourcing/EventStore.cs
public interface IEventStore
{
    Task SaveEventsAsync(Guid aggregateId, IEnumerable<IEvent> events, int expectedVersion);
    Task<List<IEvent>> GetEventsAsync(Guid aggregateId);
    Task<List<IEvent>> GetEventsAsync(Guid aggregateId, int fromVersion);
}

public class EventStore : IEventStore
{
    private readonly IEventStoreDbContext _context;
    private readonly IEventSerializer _serializer;
    private readonly IEventBus _eventBus;
    private readonly ILogger<EventStore> _logger;
    
    public EventStore(
        IEventStoreDbContext context,
        IEventSerializer serializer,
        IEventBus eventBus,
        ILogger<EventStore> logger)
    {
        _context = context;
        _serializer = serializer;
        _eventBus = eventBus;
        _logger = logger;
    }
    
    public async Task SaveEventsAsync(
        Guid aggregateId, 
        IEnumerable<IEvent> events, 
        int expectedVersion)
    {
        using var transaction = await _context.Database.BeginTransactionAsync();
        
        try
        {
            // Check for concurrency
            var currentVersion = await _context.Events
                .Where(e => e.AggregateId == aggregateId)
                .MaxAsync(e => (int?)e.Version) ?? -1;
            
            if (currentVersion != expectedVersion)
            {
                throw new ConcurrencyException(
                    $"Expected version {expectedVersion} but current version is {currentVersion}");
            }
            
            var version = expectedVersion;
            
            foreach (var @event in events)
            {
                version++;
                
                var eventData = new EventData
                {
                    Id = Guid.NewGuid(),
                    AggregateId = aggregateId,
                    EventType = @event.GetType().FullName!,
                    Data = _serializer.Serialize(@event),
                    Metadata = _serializer.Serialize(new EventMetadata
                    {
                        UserId = @event.UserId,
                        CorrelationId = @event.CorrelationId,
                        CausationId = @event.CausationId
                    }),
                    Version = version,
                    Timestamp = @event.Timestamp
                };
                
                _context.Events.Add(eventData);
                
                // Create snapshot every N events
                if (version % 10 == 0)
                {
                    await CreateSnapshotAsync(aggregateId, version);
                }
            }
            
            await _context.SaveChangesAsync();
            await transaction.CommitAsync();
            
            // Publish events
            foreach (var @event in events)
            {
                await _eventBus.PublishAsync(@event);
            }
        }
        catch (Exception ex)
        {
            await transaction.RollbackAsync();
            _logger.LogError(ex, "Error saving events for aggregate {AggregateId}", aggregateId);
            throw;
        }
    }
    
    public async Task<List<IEvent>> GetEventsAsync(Guid aggregateId)
    {
        return await GetEventsAsync(aggregateId, -1);
    }
    
    public async Task<List<IEvent>> GetEventsAsync(Guid aggregateId, int fromVersion)
    {
        var eventData = await _context.Events
            .Where(e => e.AggregateId == aggregateId && e.Version > fromVersion)
            .OrderBy(e => e.Version)
            .ToListAsync();
        
        var events = new List<IEvent>();
        
        foreach (var data in eventData)
        {
            var @event = _serializer.Deserialize(data.Data, data.EventType);
            if (@event != null)
            {
                events.Add(@event);
            }
        }
        
        return events;
    }
    
    private async Task CreateSnapshotAsync(Guid aggregateId, int version)
    {
        // Implementation for creating snapshots
        _logger.LogInformation(
            "Creating snapshot for aggregate {AggregateId} at version {Version}",
            aggregateId, version);
    }
}
```

### 3.2 이벤트 소싱된 애그리게이트

```csharp
// Domain/EventSourcing/EventSourcedAggregate.cs
public abstract class EventSourcedAggregate : IAggregateRoot
{
    private readonly List<IEvent> _uncommittedEvents = new();
    
    public Guid Id { get; protected set; }
    public int Version { get; private set; } = -1;
    
    public IReadOnlyList<IEvent> UncommittedEvents => _uncommittedEvents.AsReadOnly();
    
    protected EventSourcedAggregate()
    {
        Id = Guid.NewGuid();
    }
    
    protected EventSourcedAggregate(Guid id)
    {
        Id = id;
    }
    
    public void MarkEventsAsCommitted()
    {
        _uncommittedEvents.Clear();
    }
    
    public void LoadFromHistory(IEnumerable<IEvent> history)
    {
        foreach (var @event in history)
        {
            ApplyEvent(@event, false);
        }
    }
    
    protected void RaiseEvent(IEvent @event)
    {
        ApplyEvent(@event, true);
    }
    
    private void ApplyEvent(IEvent @event, bool isNew)
    {
        this.AsDynamic().Apply(@event);
        
        if (isNew)
        {
            _uncommittedEvents.Add(@event);
        }
        
        Version++;
    }
}

// Domain/Aggregates/ShoppingCart.cs
public class ShoppingCart : EventSourcedAggregate
{
    private readonly Dictionary<Guid, CartItem> _items = new();
    
    public Guid CustomerId { get; private set; }
    public CartStatus Status { get; private set; }
    public decimal TotalAmount => _items.Values.Sum(i => i.TotalPrice);
    public IReadOnlyDictionary<Guid, CartItem> Items => _items;
    
    private ShoppingCart() { }
    
    public ShoppingCart(Guid customerId) : base()
    {
        RaiseEvent(new ShoppingCartCreated(Id, customerId));
    }
    
    public void AddItem(Guid productId, string productName, decimal price, int quantity)
    {
        if (Status != CartStatus.Active)
            throw new InvalidOperationException("Cannot add items to inactive cart");
        
        RaiseEvent(new ItemAddedToCart(Id, productId, productName, price, quantity));
    }
    
    public void UpdateItemQuantity(Guid productId, int newQuantity)
    {
        if (!_items.ContainsKey(productId))
            throw new InvalidOperationException("Product not in cart");
        
        if (newQuantity <= 0)
        {
            RemoveItem(productId);
            return;
        }
        
        RaiseEvent(new CartItemQuantityUpdated(Id, productId, newQuantity));
    }
    
    public void RemoveItem(Guid productId)
    {
        if (!_items.ContainsKey(productId))
            throw new InvalidOperationException("Product not in cart");
        
        RaiseEvent(new ItemRemovedFromCart(Id, productId));
    }
    
    public void Checkout()
    {
        if (Status != CartStatus.Active)
            throw new InvalidOperationException("Cart is not active");
        
        if (!_items.Any())
            throw new InvalidOperationException("Cart is empty");
        
        RaiseEvent(new ShoppingCartCheckedOut(Id, TotalAmount));
    }
    
    // Event handlers
    private void Apply(ShoppingCartCreated @event)
    {
        Id = @event.CartId;
        CustomerId = @event.CustomerId;
        Status = CartStatus.Active;
    }
    
    private void Apply(ItemAddedToCart @event)
    {
        if (_items.ContainsKey(@event.ProductId))
        {
            var existingItem = _items[@event.ProductId];
            _items[@event.ProductId] = existingItem with
            {
                Quantity = existingItem.Quantity + @event.Quantity
            };
        }
        else
        {
            _items[@event.ProductId] = new CartItem(
                @event.ProductId,
                @event.ProductName,
                @event.Price,
                @event.Quantity);
        }
    }
    
    private void Apply(CartItemQuantityUpdated @event)
    {
        var item = _items[@event.ProductId];
        _items[@event.ProductId] = item with { Quantity = @event.NewQuantity };
    }
    
    private void Apply(ItemRemovedFromCart @event)
    {
        _items.Remove(@event.ProductId);
    }
    
    private void Apply(ShoppingCartCheckedOut @event)
    {
        Status = CartStatus.CheckedOut;
    }
}
```

## 4. 마이크로서비스 통신

### 4.1 Service Mesh Integration

```csharp
// Infrastructure/ServiceMesh/ServiceDiscovery.cs
public interface IServiceDiscovery
{
    Task<ServiceEndpoint?> GetServiceEndpointAsync(string serviceName);
    Task RegisterServiceAsync(ServiceRegistration registration);
    Task DeregisterServiceAsync(string serviceId);
    Task<HealthCheckResult> CheckHealthAsync(string serviceName);
}

public class ConsulServiceDiscovery : IServiceDiscovery
{
    private readonly IConsulClient _consulClient;
    private readonly ILogger<ConsulServiceDiscovery> _logger;
    private readonly ServiceDiscoveryOptions _options;
    
    public ConsulServiceDiscovery(
        IConsulClient consulClient,
        IOptions<ServiceDiscoveryOptions> options,
        ILogger<ConsulServiceDiscovery> logger)
    {
        _consulClient = consulClient;
        _options = options.Value;
        _logger = logger;
    }
    
    public async Task<ServiceEndpoint?> GetServiceEndpointAsync(string serviceName)
    {
        try
        {
            var services = await _consulClient.Health.Service(serviceName, string.Empty, true);
            
            if (!services.Response.Any())
            {
                _logger.LogWarning("No healthy instances found for service: {ServiceName}", serviceName);
                return null;
            }
            
            // Load balancing - round robin
            var service = services.Response[Random.Shared.Next(services.Response.Length)];
            
            return new ServiceEndpoint
            {
                Address = service.Service.Address,
                Port = service.Service.Port,
                ServiceName = serviceName,
                Tags = service.Service.Tags,
                Metadata = service.Service.Meta
            };
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error discovering service: {ServiceName}", serviceName);
            return null;
        }
    }
    
    public async Task RegisterServiceAsync(ServiceRegistration registration)
    {
        var agentServiceRegistration = new AgentServiceRegistration
        {
            ID = registration.ServiceId,
            Name = registration.ServiceName,
            Address = registration.Address,
            Port = registration.Port,
            Tags = registration.Tags.ToArray(),
            Meta = registration.Metadata,
            Check = new AgentServiceCheck
            {
                HTTP = $"http://{registration.Address}:{registration.Port}/health",
                Interval = TimeSpan.FromSeconds(10),
                Timeout = TimeSpan.FromSeconds(5),
                DeregisterCriticalServiceAfter = TimeSpan.FromMinutes(1)
            }
        };
        
        await _consulClient.Agent.ServiceRegister(agentServiceRegistration);
        
        _logger.LogInformation(
            "Service registered: {ServiceName} ({ServiceId}) at {Address}:{Port}",
            registration.ServiceName, registration.ServiceId, registration.Address, registration.Port);
    }
    
    public async Task DeregisterServiceAsync(string serviceId)
    {
        await _consulClient.Agent.ServiceDeregister(serviceId);
        _logger.LogInformation("Service deregistered: {ServiceId}", serviceId);
    }
    
    public async Task<HealthCheckResult> CheckHealthAsync(string serviceName)
    {
        var services = await _consulClient.Health.Service(serviceName);
        
        var healthyCount = services.Response.Count(s => s.Status.Status == "passing");
        var totalCount = services.Response.Length;
        
        return new HealthCheckResult
        {
            ServiceName = serviceName,
            HealthyInstances = healthyCount,
            TotalInstances = totalCount,
            IsHealthy = healthyCount > 0,
            LastChecked = DateTime.UtcNow
        };
    }
}
```

### 4.2 분산 트랜잭션 (Saga Pattern)

```csharp
// Infrastructure/Sagas/SagaOrchestrator.cs
public interface ISagaOrchestrator<TData> where TData : SagaData
{
    Task<SagaResult> ExecuteAsync(TData data, CancellationToken cancellationToken = default);
}

public abstract class SagaOrchestrator<TData> : ISagaOrchestrator<TData> where TData : SagaData
{
    private readonly ILogger _logger;
    private readonly ISagaRepository _repository;
    
    protected SagaOrchestrator(ILogger logger, ISagaRepository repository)
    {
        _logger = logger;
        _repository = repository;
    }
    
    protected abstract Task<SagaStepResult> ExecuteStepAsync(
        string stepName, TData data, CancellationToken cancellationToken);
    
    protected abstract Task<SagaStepResult> CompensateStepAsync(
        string stepName, TData data, CancellationToken cancellationToken);
    
    protected abstract IEnumerable<SagaStep> GetSteps();
    
    public async Task<SagaResult> ExecuteAsync(TData data, CancellationToken cancellationToken = default)
    {
        var saga = new Saga
        {
            Id = Guid.NewGuid(),
            Type = GetType().Name,
            Data = JsonSerializer.Serialize(data),
            Status = SagaStatus.Started,
            StartedAt = DateTime.UtcNow
        };
        
        await _repository.CreateAsync(saga);
        
        var executedSteps = new Stack<string>();
        
        try
        {
            foreach (var step in GetSteps())
            {
                _logger.LogInformation(
                    "Executing saga step: {StepName} for saga {SagaId}",
                    step.Name, saga.Id);
                
                var result = await ExecuteStepAsync(step.Name, data, cancellationToken);
                
                if (!result.Success)
                {
                    _logger.LogError(
                        "Saga step failed: {StepName} for saga {SagaId}. Error: {Error}",
                        step.Name, saga.Id, result.Error);
                    
                    // Start compensation
                    await CompensateAsync(executedSteps, data, cancellationToken);
                    
                    saga.Status = SagaStatus.Failed;
                    saga.Error = result.Error;
                    await _repository.UpdateAsync(saga);
                    
                    return new SagaResult
                    {
                        Success = false,
                        SagaId = saga.Id,
                        Error = result.Error
                    };
                }
                
                executedSteps.Push(step.Name);
                
                // Update saga progress
                saga.CompletedSteps.Add(step.Name);
                await _repository.UpdateAsync(saga);
            }
            
            saga.Status = SagaStatus.Completed;
            saga.CompletedAt = DateTime.UtcNow;
            await _repository.UpdateAsync(saga);
            
            _logger.LogInformation("Saga completed successfully: {SagaId}", saga.Id);
            
            return new SagaResult
            {
                Success = true,
                SagaId = saga.Id
            };
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Saga execution failed: {SagaId}", saga.Id);
            
            await CompensateAsync(executedSteps, data, cancellationToken);
            
            saga.Status = SagaStatus.Failed;
            saga.Error = ex.Message;
            await _repository.UpdateAsync(saga);
            
            return new SagaResult
            {
                Success = false,
                SagaId = saga.Id,
                Error = ex.Message
            };
        }
    }
    
    private async Task CompensateAsync(
        Stack<string> executedSteps,
        TData data,
        CancellationToken cancellationToken)
    {
        _logger.LogWarning("Starting saga compensation...");
        
        while (executedSteps.Count > 0)
        {
            var stepName = executedSteps.Pop();
            
            try
            {
                _logger.LogInformation("Compensating step: {StepName}", stepName);
                
                var result = await CompensateStepAsync(stepName, data, cancellationToken);
                
                if (!result.Success)
                {
                    _logger.LogError(
                        "Failed to compensate step: {StepName}. Error: {Error}",
                        stepName, result.Error);
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error compensating step: {StepName}", stepName);
            }
        }
    }
}

// Example Order Processing Saga
public class OrderProcessingSaga : SagaOrchestrator<OrderSagaData>
{
    private readonly IInventoryService _inventoryService;
    private readonly IPaymentService _paymentService;
    private readonly IShippingService _shippingService;
    private readonly INotificationService _notificationService;
    
    public OrderProcessingSaga(
        IInventoryService inventoryService,
        IPaymentService paymentService,
        IShippingService shippingService,
        INotificationService notificationService,
        ILogger<OrderProcessingSaga> logger,
        ISagaRepository repository)
        : base(logger, repository)
    {
        _inventoryService = inventoryService;
        _paymentService = paymentService;
        _shippingService = shippingService;
        _notificationService = notificationService;
    }
    
    protected override IEnumerable<SagaStep> GetSteps()
    {
        yield return new SagaStep("ReserveInventory", 1);
        yield return new SagaStep("ProcessPayment", 2);
        yield return new SagaStep("CreateShipment", 3);
        yield return new SagaStep("SendNotification", 4);
    }
    
    protected override async Task<SagaStepResult> ExecuteStepAsync(
        string stepName, OrderSagaData data, CancellationToken cancellationToken)
    {
        return stepName switch
        {
            "ReserveInventory" => await ReserveInventory(data),
            "ProcessPayment" => await ProcessPayment(data),
            "CreateShipment" => await CreateShipment(data),
            "SendNotification" => await SendNotification(data),
            _ => SagaStepResult.Failure($"Unknown step: {stepName}")
        };
    }
    
    protected override async Task<SagaStepResult> CompensateStepAsync(
        string stepName, OrderSagaData data, CancellationToken cancellationToken)
    {
        return stepName switch
        {
            "ReserveInventory" => await ReleaseInventory(data),
            "ProcessPayment" => await RefundPayment(data),
            "CreateShipment" => await CancelShipment(data),
            "SendNotification" => SagaStepResult.Success(), // No compensation needed
            _ => SagaStepResult.Failure($"Unknown step: {stepName}")
        };
    }
    
    private async Task<SagaStepResult> ReserveInventory(OrderSagaData data)
    {
        foreach (var item in data.Items)
        {
            var result = await _inventoryService.ReserveAsync(
                item.ProductId, item.Quantity, data.OrderId);
            
            if (!result.Success)
            {
                return SagaStepResult.Failure(
                    $"Failed to reserve inventory for product {item.ProductId}");
            }
        }
        
        return SagaStepResult.Success();
    }
    
    private async Task<SagaStepResult> ProcessPayment(OrderSagaData data)
    {
        var paymentResult = await _paymentService.ChargeAsync(new PaymentRequest
        {
            OrderId = data.OrderId,
            Amount = data.TotalAmount,
            CustomerId = data.CustomerId,
            PaymentMethod = data.PaymentMethod
        });
        
        if (!paymentResult.Success)
        {
            return SagaStepResult.Failure("Payment processing failed");
        }
        
        data.PaymentId = paymentResult.PaymentId;
        return SagaStepResult.Success();
    }
    
    // Additional step implementations...
}
```

## 5. 고급 아키텍처 패턴

### 5.1 플러그인 아키텍처

```csharp
// Infrastructure/Plugins/PluginSystem.cs
public interface IPlugin
{
    string Name { get; }
    string Version { get; }
    string Description { get; }
    Task InitializeAsync(IServiceProvider serviceProvider);
    Task ShutdownAsync();
}

public interface IPluginManager
{
    Task LoadPluginsAsync(string pluginDirectory);
    Task<T?> GetPluginAsync<T>(string pluginName) where T : IPlugin;
    IEnumerable<IPlugin> GetAllPlugins();
    Task UnloadPluginAsync(string pluginName);
}

public class PluginManager : IPluginManager, IDisposable
{
    private readonly Dictionary<string, PluginContext> _plugins = new();
    private readonly ILogger<PluginManager> _logger;
    private readonly IServiceProvider _serviceProvider;
    
    public PluginManager(ILogger<PluginManager> logger, IServiceProvider serviceProvider)
    {
        _logger = logger;
        _serviceProvider = serviceProvider;
    }
    
    public async Task LoadPluginsAsync(string pluginDirectory)
    {
        if (!Directory.Exists(pluginDirectory))
        {
            _logger.LogWarning("Plugin directory not found: {Directory}", pluginDirectory);
            return;
        }
        
        var pluginFiles = Directory.GetFiles(pluginDirectory, "*.dll");
        
        foreach (var pluginFile in pluginFiles)
        {
            try
            {
                await LoadPluginAsync(pluginFile);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to load plugin: {PluginFile}", pluginFile);
            }
        }
    }
    
    private async Task LoadPluginAsync(string pluginPath)
    {
        var loadContext = new PluginLoadContext(pluginPath);
        var assembly = loadContext.LoadFromAssemblyPath(pluginPath);
        
        var pluginTypes = assembly.GetTypes()
            .Where(t => typeof(IPlugin).IsAssignableFrom(t) && !t.IsAbstract);
        
        foreach (var pluginType in pluginTypes)
        {
            var plugin = Activator.CreateInstance(pluginType) as IPlugin;
            if (plugin == null) continue;
            
            await plugin.InitializeAsync(_serviceProvider);
            
            _plugins[plugin.Name] = new PluginContext
            {
                Plugin = plugin,
                LoadContext = loadContext,
                Assembly = assembly
            };
            
            _logger.LogInformation(
                "Plugin loaded: {PluginName} v{Version}",
                plugin.Name, plugin.Version);
        }
    }
    
    public async Task<T?> GetPluginAsync<T>(string pluginName) where T : IPlugin
    {
        if (_plugins.TryGetValue(pluginName, out var context))
        {
            return context.Plugin as T;
        }
        
        return default;
    }
    
    public IEnumerable<IPlugin> GetAllPlugins()
    {
        return _plugins.Values.Select(c => c.Plugin);
    }
    
    public async Task UnloadPluginAsync(string pluginName)
    {
        if (_plugins.TryGetValue(pluginName, out var context))
        {
            await context.Plugin.ShutdownAsync();
            context.LoadContext.Unload();
            _plugins.Remove(pluginName);
            
            _logger.LogInformation("Plugin unloaded: {PluginName}", pluginName);
        }
    }
    
    public void Dispose()
    {
        foreach (var context in _plugins.Values)
        {
            context.Plugin.ShutdownAsync().GetAwaiter().GetResult();
            context.LoadContext.Unload();
        }
        
        _plugins.Clear();
    }
    
    private class PluginContext
    {
        public IPlugin Plugin { get; set; } = null!;
        public PluginLoadContext LoadContext { get; set; } = null!;
        public Assembly Assembly { get; set; } = null!;
    }
}
```

### 5.2 Feature Flags와 A/B Testing

```csharp
// Infrastructure/FeatureManagement/FeatureFlagService.cs
public interface IFeatureFlagService
{
    Task<bool> IsEnabledAsync(string feature, string? userId = null);
    Task<T> GetVariantAsync<T>(string feature, string? userId = null) where T : class;
    Task<FeatureContext> GetContextAsync(string feature);
}

public class FeatureFlagService : IFeatureFlagService
{
    private readonly IFeatureFlagProvider _provider;
    private readonly IUserContextProvider _userContext;
    private readonly ILogger<FeatureFlagService> _logger;
    
    public FeatureFlagService(
        IFeatureFlagProvider provider,
        IUserContextProvider userContext,
        ILogger<FeatureFlagService> logger)
    {
        _provider = provider;
        _userContext = userContext;
        _logger = logger;
    }
    
    public async Task<bool> IsEnabledAsync(string feature, string? userId = null)
    {
        try
        {
            var flag = await _provider.GetFeatureFlagAsync(feature);
            if (flag == null) return false;
            
            var context = await BuildEvaluationContext(userId);
            
            return await EvaluateFlag(flag, context);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error evaluating feature flag: {Feature}", feature);
            return false; // Fail closed
        }
    }
    
    public async Task<T> GetVariantAsync<T>(string feature, string? userId = null) where T : class
    {
        var flag = await _provider.GetFeatureFlagAsync(feature);
        if (flag == null)
            throw new FeatureFlagNotFoundException(feature);
        
        var context = await BuildEvaluationContext(userId);
        var variant = await EvaluateVariant(flag, context);
        
        return JsonSerializer.Deserialize<T>(variant.Value) 
            ?? throw new InvalidOperationException("Failed to deserialize variant");
    }
    
    private async Task<EvaluationContext> BuildEvaluationContext(string? userId)
    {
        var user = await _userContext.GetCurrentUserAsync();
        
        return new EvaluationContext
        {
            UserId = userId ?? user?.Id,
            UserEmail = user?.Email,
            UserRoles = user?.Roles ?? Array.Empty<string>(),
            UserAttributes = user?.Attributes ?? new Dictionary<string, string>(),
            Environment = Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT") ?? "Production",
            Timestamp = DateTime.UtcNow
        };
    }
    
    private async Task<bool> EvaluateFlag(FeatureFlag flag, EvaluationContext context)
    {
        // Check if globally enabled/disabled
        if (!flag.Enabled) return false;
        
        // Check targeting rules
        foreach (var rule in flag.Rules.OrderBy(r => r.Priority))
        {
            if (await EvaluateRule(rule, context))
            {
                return rule.Enabled;
            }
        }
        
        // Check percentage rollout
        if (flag.PercentageRollout.HasValue && !string.IsNullOrEmpty(context.UserId))
        {
            var hash = GetConsistentHash(flag.Key + context.UserId);
            return hash <= flag.PercentageRollout.Value;
        }
        
        return flag.DefaultEnabled;
    }
    
    private async Task<bool> EvaluateRule(TargetingRule rule, EvaluationContext context)
    {
        foreach (var condition in rule.Conditions)
        {
            if (!await EvaluateCondition(condition, context))
                return false;
        }
        
        return true;
    }
    
    private Task<bool> EvaluateCondition(Condition condition, EvaluationContext context)
    {
        var value = GetContextValue(condition.Attribute, context);
        
        return Task.FromResult(condition.Operator switch
        {
            "equals" => value?.Equals(condition.Value) ?? false,
            "not_equals" => !value?.Equals(condition.Value) ?? true,
            "contains" => value?.Contains(condition.Value) ?? false,
            "in" => condition.Values?.Contains(value) ?? false,
            "not_in" => !condition.Values?.Contains(value) ?? true,
            _ => false
        });
    }
    
    private int GetConsistentHash(string value)
    {
        using var md5 = MD5.Create();
        var hash = md5.ComputeHash(Encoding.UTF8.GetBytes(value));
        return Math.Abs(BitConverter.ToInt32(hash, 0)) % 100;
    }
}
```

## 마무리

엔터프라이즈급 Blazor 애플리케이션 개발에는 확장 가능하고 유지보수 가능한 아키텍처가 필수적입니다. Clean Architecture, CQRS, Event Sourcing, 마이크로서비스 패턴 등을 적절히 조합하여 비즈니스 요구사항에 맞는 견고한 시스템을 구축할 수 있습니다. 지속적인 모니터링, 테스트, 리팩토링을 통해 시스템의 품질을 유지하고 발전시켜 나가는 것이 중요합니다.