# Event Sourcing

## Event Sourcing 소개

Event Sourcing은 애플리케이션의 상태 변경을 이벤트의 시퀀스로 저장하는 패턴입니다. 현재 상태를 저장하는 대신, 상태를 변경시킨 모든 이벤트를 저장하고, 이 이벤트들을 재생하여 현재 상태를 구성합니다.

### 핵심 개념

1. **이벤트(Event)**: 발생한 사실을 나타내는 불변 객체
2. **이벤트 스토어(Event Store)**: 이벤트를 저장하는 저장소
3. **이벤트 스트림(Event Stream)**: 특정 애그리게이트의 이벤트 시퀀스
4. **프로젝션(Projection)**: 이벤트로부터 구성된 읽기 모델

### 장점

- **완전한 감사 로그**: 모든 변경 사항의 히스토리 보존
- **시간 여행**: 특정 시점의 상태로 되돌리기 가능
- **이벤트 리플레이**: 버그 수정 후 이벤트 재생으로 상태 복구
- **복잡한 비즈니스 분석**: 이벤트 데이터를 활용한 분석

### 단점

- **복잡성 증가**: 구현 및 이해가 어려움
- **최종 일관성**: 즉각적인 일관성 보장 어려움
- **저장 공간**: 모든 이벤트 저장으로 인한 용량 증가

## 기본 구현

### 이벤트 정의

```csharp
// Events/DomainEvent.cs
public abstract class DomainEvent
{
    public Guid EventId { get; }
    public Guid AggregateId { get; }
    public DateTime OccurredOn { get; }
    public int Version { get; }
    
    protected DomainEvent(Guid aggregateId, int version)
    {
        EventId = Guid.NewGuid();
        AggregateId = aggregateId;
        OccurredOn = DateTime.UtcNow;
        Version = version;
    }
}

// Events/AccountEvents.cs
public class AccountCreatedEvent : DomainEvent
{
    public string AccountNumber { get; }
    public string OwnerName { get; }
    public decimal InitialBalance { get; }
    
    public AccountCreatedEvent(
        Guid aggregateId, 
        int version,
        string accountNumber,
        string ownerName,
        decimal initialBalance) 
        : base(aggregateId, version)
    {
        AccountNumber = accountNumber;
        OwnerName = ownerName;
        InitialBalance = initialBalance;
    }
}

public class MoneyDepositedEvent : DomainEvent
{
    public decimal Amount { get; }
    public string Description { get; }
    
    public MoneyDepositedEvent(
        Guid aggregateId,
        int version,
        decimal amount,
        string description)
        : base(aggregateId, version)
    {
        Amount = amount;
        Description = description;
    }
}

public class MoneyWithdrawnEvent : DomainEvent
{
    public decimal Amount { get; }
    public string Description { get; }
    
    public MoneyWithdrawnEvent(
        Guid aggregateId,
        int version,
        decimal amount,
        string description)
        : base(aggregateId, version)
    {
        Amount = amount;
        Description = description;
    }
}
```

### 애그리게이트 루트

```csharp
// Domain/AggregateRoot.cs
public abstract class AggregateRoot
{
    private readonly List<DomainEvent> _events = new();
    
    public Guid Id { get; protected set; }
    public int Version { get; private set; } = -1;
    
    public IReadOnlyCollection<DomainEvent> GetUncommittedEvents()
    {
        return _events.AsReadOnly();
    }
    
    public void MarkEventsAsCommitted()
    {
        _events.Clear();
    }
    
    public void LoadFromHistory(IEnumerable<DomainEvent> history)
    {
        foreach (var @event in history)
        {
            ApplyEvent(@event, false);
        }
    }
    
    protected void ApplyEvent(DomainEvent @event)
    {
        ApplyEvent(@event, true);
    }
    
    private void ApplyEvent(DomainEvent @event, bool isNew)
    {
        var method = GetType().GetMethod("Apply", 
            BindingFlags.NonPublic | BindingFlags.Instance,
            null,
            new[] { @event.GetType() },
            null);
            
        if (method == null)
        {
            throw new InvalidOperationException(
                $"Apply method not found for event type {@event.GetType().Name}");
        }
        
        method.Invoke(this, new object[] { @event });
        
        if (isNew)
        {
            _events.Add(@event);
        }
        
        Version = @event.Version;
    }
}

// Domain/BankAccount.cs
public class BankAccount : AggregateRoot
{
    public string AccountNumber { get; private set; }
    public string OwnerName { get; private set; }
    public decimal Balance { get; private set; }
    public bool IsActive { get; private set; }
    
    private BankAccount() { } // For rehydration
    
    public BankAccount(string accountNumber, string ownerName, decimal initialBalance)
    {
        if (string.IsNullOrWhiteSpace(accountNumber))
            throw new ArgumentException("계좌번호는 필수입니다.");
            
        if (string.IsNullOrWhiteSpace(ownerName))
            throw new ArgumentException("예금주명은 필수입니다.");
            
        if (initialBalance < 0)
            throw new ArgumentException("초기 잔액은 0 이상이어야 합니다.");
            
        Id = Guid.NewGuid();
        
        ApplyEvent(new AccountCreatedEvent(
            Id,
            Version + 1,
            accountNumber,
            ownerName,
            initialBalance));
    }
    
    public void Deposit(decimal amount, string description)
    {
        if (!IsActive)
            throw new InvalidOperationException("비활성화된 계좌입니다.");
            
        if (amount <= 0)
            throw new ArgumentException("입금액은 0보다 커야 합니다.");
            
        ApplyEvent(new MoneyDepositedEvent(
            Id,
            Version + 1,
            amount,
            description));
    }
    
    public void Withdraw(decimal amount, string description)
    {
        if (!IsActive)
            throw new InvalidOperationException("비활성화된 계좌입니다.");
            
        if (amount <= 0)
            throw new ArgumentException("출금액은 0보다 커야 합니다.");
            
        if (Balance < amount)
            throw new InvalidOperationException("잔액이 부족합니다.");
            
        ApplyEvent(new MoneyWithdrawnEvent(
            Id,
            Version + 1,
            amount,
            description));
    }
    
    // Event Handlers
    private void Apply(AccountCreatedEvent @event)
    {
        AccountNumber = @event.AccountNumber;
        OwnerName = @event.OwnerName;
        Balance = @event.InitialBalance;
        IsActive = true;
    }
    
    private void Apply(MoneyDepositedEvent @event)
    {
        Balance += @event.Amount;
    }
    
    private void Apply(MoneyWithdrawnEvent @event)
    {
        Balance -= @event.Amount;
    }
}
```

## Event Store 구현

### 인터페이스 정의

```csharp
// Infrastructure/IEventStore.cs
public interface IEventStore
{
    Task SaveEventsAsync(Guid aggregateId, IEnumerable<DomainEvent> events, int expectedVersion);
    Task<List<DomainEvent>> GetEventsAsync(Guid aggregateId);
    Task<List<DomainEvent>> GetEventsAsync(Guid aggregateId, int fromVersion);
}

// Infrastructure/EventStoreItem.cs
public class EventStoreItem
{
    public Guid Id { get; set; }
    public Guid AggregateId { get; set; }
    public string EventType { get; set; }
    public string EventData { get; set; }
    public int Version { get; set; }
    public DateTime CreatedAt { get; set; }
}
```

### SQL 기반 Event Store

```csharp
// Infrastructure/SqlEventStore.cs
public class SqlEventStore : IEventStore
{
    private readonly IDbConnection _connection;
    private readonly IEventSerializer _serializer;
    
    public SqlEventStore(IDbConnection connection, IEventSerializer serializer)
    {
        _connection = connection;
        _serializer = serializer;
    }
    
    public async Task SaveEventsAsync(
        Guid aggregateId, 
        IEnumerable<DomainEvent> events, 
        int expectedVersion)
    {
        using var transaction = _connection.BeginTransaction();
        
        try
        {
            // 동시성 체크
            var currentVersion = await GetLastVersionAsync(aggregateId);
            if (currentVersion != expectedVersion)
            {
                throw new ConcurrencyException(
                    $"Expected version {expectedVersion} but was {currentVersion}");
            }
            
            // 이벤트 저장
            foreach (var @event in events)
            {
                var eventData = _serializer.Serialize(@event);
                
                await _connection.ExecuteAsync(@"
                    INSERT INTO EventStore (Id, AggregateId, EventType, EventData, Version, CreatedAt)
                    VALUES (@Id, @AggregateId, @EventType, @EventData, @Version, @CreatedAt)",
                    new EventStoreItem
                    {
                        Id = @event.EventId,
                        AggregateId = @event.AggregateId,
                        EventType = @event.GetType().AssemblyQualifiedName,
                        EventData = eventData,
                        Version = @event.Version,
                        CreatedAt = @event.OccurredOn
                    },
                    transaction);
            }
            
            transaction.Commit();
        }
        catch
        {
            transaction.Rollback();
            throw;
        }
    }
    
    public async Task<List<DomainEvent>> GetEventsAsync(Guid aggregateId)
    {
        return await GetEventsAsync(aggregateId, -1);
    }
    
    public async Task<List<DomainEvent>> GetEventsAsync(Guid aggregateId, int fromVersion)
    {
        var items = await _connection.QueryAsync<EventStoreItem>(@"
            SELECT * FROM EventStore 
            WHERE AggregateId = @AggregateId AND Version > @FromVersion
            ORDER BY Version",
            new { AggregateId = aggregateId, FromVersion = fromVersion });
            
        var events = new List<DomainEvent>();
        
        foreach (var item in items)
        {
            var @event = _serializer.Deserialize(item.EventData, item.EventType);
            events.Add(@event);
        }
        
        return events;
    }
    
    private async Task<int> GetLastVersionAsync(Guid aggregateId)
    {
        var version = await _connection.QuerySingleOrDefaultAsync<int?>(@"
            SELECT MAX(Version) FROM EventStore WHERE AggregateId = @AggregateId",
            new { AggregateId = aggregateId });
            
        return version ?? -1;
    }
}
```

### Event Serializer

```csharp
// Infrastructure/EventSerializer.cs
public interface IEventSerializer
{
    string Serialize(DomainEvent @event);
    DomainEvent Deserialize(string data, string typeName);
}

public class JsonEventSerializer : IEventSerializer
{
    private readonly JsonSerializerOptions _options;
    
    public JsonEventSerializer()
    {
        _options = new JsonSerializerOptions
        {
            PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
            WriteIndented = false
        };
    }
    
    public string Serialize(DomainEvent @event)
    {
        return JsonSerializer.Serialize(@event, @event.GetType(), _options);
    }
    
    public DomainEvent Deserialize(string data, string typeName)
    {
        var type = Type.GetType(typeName);
        if (type == null)
            throw new InvalidOperationException($"Type {typeName} not found");
            
        return (DomainEvent)JsonSerializer.Deserialize(data, type, _options)!;
    }
}
```

## Repository 패턴

```csharp
// Domain/IEventSourcedRepository.cs
public interface IEventSourcedRepository<T> where T : AggregateRoot
{
    Task<T?> GetByIdAsync(Guid id);
    Task SaveAsync(T aggregate, int expectedVersion);
}

// Infrastructure/EventSourcedRepository.cs
public class EventSourcedRepository<T> : IEventSourcedRepository<T> 
    where T : AggregateRoot, new()
{
    private readonly IEventStore _eventStore;
    
    public EventSourcedRepository(IEventStore eventStore)
    {
        _eventStore = eventStore;
    }
    
    public async Task<T?> GetByIdAsync(Guid id)
    {
        var events = await _eventStore.GetEventsAsync(id);
        
        if (!events.Any())
            return null;
            
        var aggregate = new T();
        aggregate.LoadFromHistory(events);
        
        return aggregate;
    }
    
    public async Task SaveAsync(T aggregate, int expectedVersion)
    {
        var events = aggregate.GetUncommittedEvents();
        
        if (!events.Any())
            return;
            
        await _eventStore.SaveEventsAsync(aggregate.Id, events, expectedVersion);
        aggregate.MarkEventsAsCommitted();
    }
}
```

## 프로젝션 구현

### 프로젝션 인터페이스

```csharp
// Projections/IProjection.cs
public interface IProjection
{
    Task HandleAsync(DomainEvent @event);
}

// Projections/IProjectionEngine.cs
public interface IProjectionEngine
{
    Task ProjectAsync(DomainEvent @event);
    Task RebuildProjectionsAsync();
}
```

### 계좌 잔액 프로젝션

```csharp
// Projections/AccountBalanceProjection.cs
public class AccountBalance
{
    public Guid AccountId { get; set; }
    public string AccountNumber { get; set; }
    public string OwnerName { get; set; }
    public decimal Balance { get; set; }
    public DateTime LastUpdated { get; set; }
}

public class AccountBalanceProjection : IProjection
{
    private readonly IDbConnection _connection;
    
    public AccountBalanceProjection(IDbConnection connection)
    {
        _connection = connection;
    }
    
    public async Task HandleAsync(DomainEvent @event)
    {
        switch (@event)
        {
            case AccountCreatedEvent created:
                await HandleAccountCreated(created);
                break;
            case MoneyDepositedEvent deposited:
                await HandleMoneyDeposited(deposited);
                break;
            case MoneyWithdrawnEvent withdrawn:
                await HandleMoneyWithdrawn(withdrawn);
                break;
        }
    }
    
    private async Task HandleAccountCreated(AccountCreatedEvent @event)
    {
        await _connection.ExecuteAsync(@"
            INSERT INTO AccountBalances (AccountId, AccountNumber, OwnerName, Balance, LastUpdated)
            VALUES (@AccountId, @AccountNumber, @OwnerName, @Balance, @LastUpdated)",
            new
            {
                AccountId = @event.AggregateId,
                @event.AccountNumber,
                @event.OwnerName,
                Balance = @event.InitialBalance,
                LastUpdated = @event.OccurredOn
            });
    }
    
    private async Task HandleMoneyDeposited(MoneyDepositedEvent @event)
    {
        await _connection.ExecuteAsync(@"
            UPDATE AccountBalances 
            SET Balance = Balance + @Amount, LastUpdated = @LastUpdated
            WHERE AccountId = @AccountId",
            new
            {
                @event.Amount,
                AccountId = @event.AggregateId,
                LastUpdated = @event.OccurredOn
            });
    }
    
    private async Task HandleMoneyWithdrawn(MoneyWithdrawnEvent @event)
    {
        await _connection.ExecuteAsync(@"
            UPDATE AccountBalances 
            SET Balance = Balance - @Amount, LastUpdated = @LastUpdated
            WHERE AccountId = @AccountId",
            new
            {
                @event.Amount,
                AccountId = @event.AggregateId,
                LastUpdated = @event.OccurredOn
            });
    }
}
```

### 거래 내역 프로젝션

```csharp
// Projections/TransactionHistoryProjection.cs
public class TransactionHistory
{
    public Guid TransactionId { get; set; }
    public Guid AccountId { get; set; }
    public string Type { get; set; }
    public decimal Amount { get; set; }
    public string Description { get; set; }
    public decimal BalanceAfter { get; set; }
    public DateTime TransactionDate { get; set; }
}

public class TransactionHistoryProjection : IProjection
{
    private readonly IDbConnection _connection;
    
    public TransactionHistoryProjection(IDbConnection connection)
    {
        _connection = connection;
    }
    
    public async Task HandleAsync(DomainEvent @event)
    {
        switch (@event)
        {
            case MoneyDepositedEvent deposited:
                await RecordTransaction(deposited, "DEPOSIT", deposited.Amount);
                break;
            case MoneyWithdrawnEvent withdrawn:
                await RecordTransaction(withdrawn, "WITHDRAWAL", -withdrawn.Amount);
                break;
        }
    }
    
    private async Task RecordTransaction(
        DomainEvent @event, 
        string type, 
        decimal amount)
    {
        // 현재 잔액 조회
        var currentBalance = await _connection.QuerySingleAsync<decimal>(@"
            SELECT Balance FROM AccountBalances WHERE AccountId = @AccountId",
            new { AccountId = @event.AggregateId });
            
        var description = @event switch
        {
            MoneyDepositedEvent d => d.Description,
            MoneyWithdrawnEvent w => w.Description,
            _ => ""
        };
        
        await _connection.ExecuteAsync(@"
            INSERT INTO TransactionHistory 
            (TransactionId, AccountId, Type, Amount, Description, BalanceAfter, TransactionDate)
            VALUES (@TransactionId, @AccountId, @Type, @Amount, @Description, @BalanceAfter, @TransactionDate)",
            new TransactionHistory
            {
                TransactionId = @event.EventId,
                AccountId = @event.AggregateId,
                Type = type,
                Amount = Math.Abs(amount),
                Description = description,
                BalanceAfter = currentBalance,
                TransactionDate = @event.OccurredOn
            });
    }
}
```

## 스냅샷 구현

```csharp
// Domain/ISnapshot.cs
public interface ISnapshot
{
    Guid AggregateId { get; }
    int Version { get; }
    DateTime CreatedAt { get; }
}

// Domain/BankAccountSnapshot.cs
public class BankAccountSnapshot : ISnapshot
{
    public Guid AggregateId { get; set; }
    public int Version { get; set; }
    public DateTime CreatedAt { get; set; }
    public string AccountNumber { get; set; }
    public string OwnerName { get; set; }
    public decimal Balance { get; set; }
    public bool IsActive { get; set; }
}

// Infrastructure/ISnapshotStore.cs
public interface ISnapshotStore
{
    Task SaveSnapshotAsync(ISnapshot snapshot);
    Task<T?> GetSnapshotAsync<T>(Guid aggregateId) where T : ISnapshot;
}

// Updated BankAccount with Snapshot
public class BankAccount : AggregateRoot
{
    // ... existing code ...
    
    public BankAccountSnapshot CreateSnapshot()
    {
        return new BankAccountSnapshot
        {
            AggregateId = Id,
            Version = Version,
            CreatedAt = DateTime.UtcNow,
            AccountNumber = AccountNumber,
            OwnerName = OwnerName,
            Balance = Balance,
            IsActive = IsActive
        };
    }
    
    public void RestoreFromSnapshot(BankAccountSnapshot snapshot)
    {
        Id = snapshot.AggregateId;
        Version = snapshot.Version;
        AccountNumber = snapshot.AccountNumber;
        OwnerName = snapshot.OwnerName;
        Balance = snapshot.Balance;
        IsActive = snapshot.IsActive;
    }
}
```

## Application Service

```csharp
// Application/BankAccountService.cs
public class BankAccountService
{
    private readonly IEventSourcedRepository<BankAccount> _repository;
    private readonly IProjectionEngine _projectionEngine;
    
    public BankAccountService(
        IEventSourcedRepository<BankAccount> repository,
        IProjectionEngine projectionEngine)
    {
        _repository = repository;
        _projectionEngine = projectionEngine;
    }
    
    public async Task<Guid> CreateAccountAsync(
        string accountNumber, 
        string ownerName, 
        decimal initialBalance)
    {
        var account = new BankAccount(accountNumber, ownerName, initialBalance);
        await _repository.SaveAsync(account, -1);
        
        // 프로젝션 업데이트
        foreach (var @event in account.GetUncommittedEvents())
        {
            await _projectionEngine.ProjectAsync(@event);
        }
        
        return account.Id;
    }
    
    public async Task DepositAsync(Guid accountId, decimal amount, string description)
    {
        var account = await _repository.GetByIdAsync(accountId);
        if (account == null)
            throw new AccountNotFoundException(accountId);
            
        var expectedVersion = account.Version;
        account.Deposit(amount, description);
        
        await _repository.SaveAsync(account, expectedVersion);
        
        foreach (var @event in account.GetUncommittedEvents())
        {
            await _projectionEngine.ProjectAsync(@event);
        }
    }
    
    public async Task WithdrawAsync(Guid accountId, decimal amount, string description)
    {
        var account = await _repository.GetByIdAsync(accountId);
        if (account == null)
            throw new AccountNotFoundException(accountId);
            
        var expectedVersion = account.Version;
        account.Withdraw(amount, description);
        
        await _repository.SaveAsync(account, expectedVersion);
        
        foreach (var @event in account.GetUncommittedEvents())
        {
            await _projectionEngine.ProjectAsync(@event);
        }
    }
}
```

## Event Store 마이그레이션

```sql
-- Event Store 테이블
CREATE TABLE EventStore (
    Id UNIQUEIDENTIFIER PRIMARY KEY,
    AggregateId UNIQUEIDENTIFIER NOT NULL,
    EventType NVARCHAR(500) NOT NULL,
    EventData NVARCHAR(MAX) NOT NULL,
    Version INT NOT NULL,
    CreatedAt DATETIME2 NOT NULL,
    INDEX IX_EventStore_AggregateId_Version (AggregateId, Version)
);

-- 스냅샷 테이블
CREATE TABLE Snapshots (
    AggregateId UNIQUEIDENTIFIER PRIMARY KEY,
    SnapshotType NVARCHAR(500) NOT NULL,
    SnapshotData NVARCHAR(MAX) NOT NULL,
    Version INT NOT NULL,
    CreatedAt DATETIME2 NOT NULL
);

-- 프로젝션 테이블
CREATE TABLE AccountBalances (
    AccountId UNIQUEIDENTIFIER PRIMARY KEY,
    AccountNumber NVARCHAR(50) NOT NULL,
    OwnerName NVARCHAR(100) NOT NULL,
    Balance DECIMAL(18,2) NOT NULL,
    LastUpdated DATETIME2 NOT NULL
);

CREATE TABLE TransactionHistory (
    TransactionId UNIQUEIDENTIFIER PRIMARY KEY,
    AccountId UNIQUEIDENTIFIER NOT NULL,
    Type NVARCHAR(50) NOT NULL,
    Amount DECIMAL(18,2) NOT NULL,
    Description NVARCHAR(500),
    BalanceAfter DECIMAL(18,2) NOT NULL,
    TransactionDate DATETIME2 NOT NULL,
    INDEX IX_TransactionHistory_AccountId (AccountId)
);
```

## 테스트

```csharp
public class BankAccountTests
{
    [Fact]
    public void CreateAccount_Should_RaiseAccountCreatedEvent()
    {
        // Arrange & Act
        var account = new BankAccount("123456", "John Doe", 1000m);
        
        // Assert
        var events = account.GetUncommittedEvents();
        Assert.Single(events);
        
        var createdEvent = Assert.IsType<AccountCreatedEvent>(events.First());
        Assert.Equal("123456", createdEvent.AccountNumber);
        Assert.Equal("John Doe", createdEvent.OwnerName);
        Assert.Equal(1000m, createdEvent.InitialBalance);
    }
    
    [Fact]
    public void Deposit_Should_IncreaseBalance()
    {
        // Arrange
        var account = new BankAccount("123456", "John Doe", 1000m);
        account.MarkEventsAsCommitted();
        
        // Act
        account.Deposit(500m, "Salary");
        
        // Assert
        Assert.Equal(1500m, account.Balance);
        
        var events = account.GetUncommittedEvents();
        var depositEvent = Assert.IsType<MoneyDepositedEvent>(events.First());
        Assert.Equal(500m, depositEvent.Amount);
    }
    
    [Fact]
    public async Task EventStore_Should_SaveAndLoadEvents()
    {
        // Arrange
        var connection = new SqliteConnection("DataSource=:memory:");
        await connection.OpenAsync();
        await CreateTables(connection);
        
        var eventStore = new SqlEventStore(connection, new JsonEventSerializer());
        var repository = new EventSourcedRepository<BankAccount>(eventStore);
        
        // Act
        var account = new BankAccount("123456", "John Doe", 1000m);
        account.Deposit(500m, "Test");
        await repository.SaveAsync(account, -1);
        
        var loaded = await repository.GetByIdAsync(account.Id);
        
        // Assert
        Assert.NotNull(loaded);
        Assert.Equal(1500m, loaded.Balance);
        Assert.Equal(1, loaded.Version);
    }
}
```

## 마무리

Event Sourcing은 시스템의 모든 상태 변경을 이벤트로 저장하여 완전한 감사 추적과 시간 여행을 가능하게 합니다. 복잡성이 증가하지만, 복잡한 도메인과 규정 준수가 중요한 시스템에서는 강력한 이점을 제공합니다.