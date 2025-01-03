# Domain-Driven Design (DDD)

## DDD 소개

Domain-Driven Design(DDD)은 Eric Evans가 제안한 소프트웨어 설계 접근법으로, 복잡한 비즈니스 도메인을 모델링하고 구현하는 방법론입니다. 도메인 전문가와 개발자 간의 협업을 통해 비즈니스 요구사항을 정확히 반영하는 소프트웨어를 만드는 것을 목표로 합니다.

### DDD의 핵심 개념

1. **유비쿼터스 언어(Ubiquitous Language)**: 도메인 전문가와 개발자가 공통으로 사용하는 언어
2. **바운디드 컨텍스트(Bounded Context)**: 특정 도메인 모델이 적용되는 경계
3. **전략적 설계(Strategic Design)**: 큰 그림에서의 시스템 설계
4. **전술적 설계(Tactical Design)**: 구체적인 도메인 모델 구현

## 전략적 설계

### 바운디드 컨텍스트

```csharp
// 판매 컨텍스트의 Product
namespace Sales.Domain
{
    public class Product
    {
        public Guid Id { get; private set; }
        public string Name { get; private set; }
        public Money Price { get; private set; }
        public bool IsAvailable { get; private set; }
        
        public void UpdatePrice(Money newPrice)
        {
            if (newPrice.Amount <= 0)
                throw new InvalidOperationException("가격은 0보다 커야 합니다.");
                
            Price = newPrice;
        }
    }
}

// 재고 컨텍스트의 Product
namespace Inventory.Domain
{
    public class Product
    {
        public Guid Id { get; private set; }
        public string SKU { get; private set; }
        public int QuantityOnHand { get; private set; }
        public int ReorderLevel { get; private set; }
        
        public void AdjustInventory(int quantity)
        {
            if (QuantityOnHand + quantity < 0)
                throw new InsufficientInventoryException();
                
            QuantityOnHand += quantity;
        }
    }
}
```

### 컨텍스트 맵

```csharp
// 컨텍스트 간 통합을 위한 Anti-Corruption Layer
namespace Sales.Infrastructure
{
    public interface IInventoryService
    {
        Task<bool> IsProductAvailable(Guid productId, int quantity);
        Task ReserveInventory(Guid productId, int quantity);
    }
    
    public class InventoryServiceAdapter : IInventoryService
    {
        private readonly HttpClient _httpClient;
        
        public async Task<bool> IsProductAvailable(Guid productId, int quantity)
        {
            // 재고 서비스 API 호출
            var response = await _httpClient.GetAsync(
                $"/api/inventory/{productId}/availability?quantity={quantity}");
                
            return response.IsSuccessStatusCode;
        }
        
        public async Task ReserveInventory(Guid productId, int quantity)
        {
            var request = new { ProductId = productId, Quantity = quantity };
            await _httpClient.PostAsJsonAsync("/api/inventory/reserve", request);
        }
    }
}
```

## 전술적 설계 패턴

### 엔티티(Entity)

```csharp
namespace ECommerce.Domain.Entities
{
    public abstract class Entity
    {
        public Guid Id { get; protected set; }
        
        protected Entity()
        {
            Id = Guid.NewGuid();
        }
        
        public override bool Equals(object? obj)
        {
            if (obj is not Entity other)
                return false;
                
            if (ReferenceEquals(this, other))
                return true;
                
            if (GetType() != other.GetType())
                return false;
                
            return Id == other.Id;
        }
        
        public override int GetHashCode()
        {
            return Id.GetHashCode();
        }
    }
    
    public class Customer : Entity
    {
        public string Name { get; private set; }
        public Email Email { get; private set; }
        public Address ShippingAddress { get; private set; }
        
        public Customer(string name, Email email, Address shippingAddress)
        {
            Name = name ?? throw new ArgumentNullException(nameof(name));
            Email = email ?? throw new ArgumentNullException(nameof(email));
            ShippingAddress = shippingAddress ?? throw new ArgumentNullException(nameof(shippingAddress));
        }
        
        public void UpdateShippingAddress(Address newAddress)
        {
            ShippingAddress = newAddress ?? throw new ArgumentNullException(nameof(newAddress));
        }
    }
}
```

### 값 객체(Value Object)

```csharp
namespace ECommerce.Domain.ValueObjects
{
    public record Email
    {
        public string Value { get; }
        
        public Email(string value)
        {
            if (string.IsNullOrWhiteSpace(value))
                throw new ArgumentException("이메일은 필수입니다.");
                
            if (!IsValidEmail(value))
                throw new ArgumentException("유효하지 않은 이메일 형식입니다.");
                
            Value = value.ToLower();
        }
        
        private static bool IsValidEmail(string email)
        {
            return Regex.IsMatch(email, @"^[^@\s]+@[^@\s]+\.[^@\s]+$");
        }
        
        public override string ToString() => Value;
    }
    
    public record Address
    {
        public string Street { get; }
        public string City { get; }
        public string ZipCode { get; }
        public string Country { get; }
        
        public Address(string street, string city, string zipCode, string country)
        {
            Street = street ?? throw new ArgumentNullException(nameof(street));
            City = city ?? throw new ArgumentNullException(nameof(city));
            ZipCode = zipCode ?? throw new ArgumentNullException(nameof(zipCode));
            Country = country ?? throw new ArgumentNullException(nameof(country));
        }
    }
    
    public record Money
    {
        public decimal Amount { get; }
        public string Currency { get; }
        
        public Money(decimal amount, string currency)
        {
            if (amount < 0)
                throw new ArgumentException("금액은 음수일 수 없습니다.");
                
            Amount = amount;
            Currency = currency ?? throw new ArgumentNullException(nameof(currency));
        }
        
        public Money Add(Money other)
        {
            if (Currency != other.Currency)
                throw new InvalidOperationException("다른 통화는 더할 수 없습니다.");
                
            return new Money(Amount + other.Amount, Currency);
        }
        
        public Money Multiply(int factor)
        {
            return new Money(Amount * factor, Currency);
        }
    }
}
```

### 애그리게이트(Aggregate)

```csharp
namespace ECommerce.Domain.Aggregates
{
    // 애그리게이트 루트
    public class Order : Entity, IAggregateRoot
    {
        private readonly List<OrderItem> _items = new();
        private readonly List<IDomainEvent> _domainEvents = new();
        
        public string OrderNumber { get; private set; }
        public Guid CustomerId { get; private set; }
        public OrderStatus Status { get; private set; }
        public Address ShippingAddress { get; private set; }
        public DateTime OrderDate { get; private set; }
        public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();
        public IReadOnlyCollection<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();
        
        public Money TotalAmount => 
            _items.Aggregate(
                new Money(0, "KRW"), 
                (total, item) => total.Add(item.TotalPrice));
        
        private Order() { } // EF Core를 위한 private 생성자
        
        public Order(Guid customerId, Address shippingAddress)
        {
            CustomerId = customerId;
            ShippingAddress = shippingAddress ?? throw new ArgumentNullException(nameof(shippingAddress));
            Status = OrderStatus.Pending;
            OrderDate = DateTime.UtcNow;
            OrderNumber = GenerateOrderNumber();
            
            AddDomainEvent(new OrderCreatedEvent(Id, customerId, OrderDate));
        }
        
        public void AddItem(Guid productId, string productName, Money unitPrice, int quantity)
        {
            if (Status != OrderStatus.Pending)
                throw new InvalidOperationException("확정된 주문은 수정할 수 없습니다.");
                
            var existingItem = _items.FirstOrDefault(x => x.ProductId == productId);
            
            if (existingItem != null)
            {
                existingItem.IncreaseQuantity(quantity);
            }
            else
            {
                var orderItem = new OrderItem(productId, productName, unitPrice, quantity);
                _items.Add(orderItem);
            }
        }
        
        public void Submit()
        {
            if (Status != OrderStatus.Pending)
                throw new InvalidOperationException("이미 제출된 주문입니다.");
                
            if (!_items.Any())
                throw new InvalidOperationException("주문 항목이 없습니다.");
                
            Status = OrderStatus.Submitted;
            AddDomainEvent(new OrderSubmittedEvent(Id, CustomerId, TotalAmount));
        }
        
        public void Cancel(string reason)
        {
            if (Status == OrderStatus.Shipped || Status == OrderStatus.Delivered)
                throw new InvalidOperationException("배송된 주문은 취소할 수 없습니다.");
                
            Status = OrderStatus.Cancelled;
            AddDomainEvent(new OrderCancelledEvent(Id, reason));
        }
        
        private string GenerateOrderNumber()
        {
            return $"ORD-{DateTime.UtcNow:yyyyMMdd}-{Guid.NewGuid().ToString().Substring(0, 8).ToUpper()}";
        }
        
        protected void AddDomainEvent(IDomainEvent eventItem)
        {
            _domainEvents.Add(eventItem);
        }
        
        public void ClearDomainEvents()
        {
            _domainEvents.Clear();
        }
    }
    
    // 애그리게이트 내부 엔티티
    public class OrderItem : Entity
    {
        public Guid ProductId { get; private set; }
        public string ProductName { get; private set; }
        public Money UnitPrice { get; private set; }
        public int Quantity { get; private set; }
        
        public Money TotalPrice => UnitPrice.Multiply(Quantity);
        
        internal OrderItem(Guid productId, string productName, Money unitPrice, int quantity)
        {
            ProductId = productId;
            ProductName = productName ?? throw new ArgumentNullException(nameof(productName));
            UnitPrice = unitPrice ?? throw new ArgumentNullException(nameof(unitPrice));
            
            if (quantity <= 0)
                throw new ArgumentException("수량은 0보다 커야 합니다.");
                
            Quantity = quantity;
        }
        
        internal void IncreaseQuantity(int additionalQuantity)
        {
            if (additionalQuantity <= 0)
                throw new ArgumentException("추가 수량은 0보다 커야 합니다.");
                
            Quantity += additionalQuantity;
        }
    }
}
```

### 도메인 서비스

```csharp
namespace ECommerce.Domain.Services
{
    public interface IPricingService
    {
        Money CalculateDiscount(Order order, PromotionCode promotionCode);
    }
    
    public class PricingService : IPricingService
    {
        private readonly IPromotionRepository _promotionRepository;
        
        public PricingService(IPromotionRepository promotionRepository)
        {
            _promotionRepository = promotionRepository;
        }
        
        public Money CalculateDiscount(Order order, PromotionCode promotionCode)
        {
            var promotion = _promotionRepository.GetByCode(promotionCode);
            
            if (promotion == null || !promotion.IsValid())
                return new Money(0, order.TotalAmount.Currency);
                
            return promotion.CalculateDiscount(order.TotalAmount);
        }
    }
    
    public interface IInventoryService
    {
        Task<bool> CheckAvailability(Guid productId, int quantity);
        Task ReserveItems(Order order);
    }
}
```

### 리포지토리

```csharp
namespace ECommerce.Domain.Repositories
{
    public interface IRepository<T> where T : IAggregateRoot
    {
        Task<T?> GetByIdAsync(Guid id);
        Task AddAsync(T entity);
        Task UpdateAsync(T entity);
        Task DeleteAsync(T entity);
    }
    
    public interface IOrderRepository : IRepository<Order>
    {
        Task<Order?> GetByOrderNumberAsync(string orderNumber);
        Task<IEnumerable<Order>> GetByCustomerIdAsync(Guid customerId);
        Task<IEnumerable<Order>> GetPendingOrdersAsync();
    }
}

// Infrastructure Layer 구현
namespace ECommerce.Infrastructure.Repositories
{
    public class OrderRepository : IOrderRepository
    {
        private readonly AppDbContext _context;
        
        public OrderRepository(AppDbContext context)
        {
            _context = context;
        }
        
        public async Task<Order?> GetByIdAsync(Guid id)
        {
            return await _context.Orders
                .Include(o => o.Items)
                .FirstOrDefaultAsync(o => o.Id == id);
        }
        
        public async Task<Order?> GetByOrderNumberAsync(string orderNumber)
        {
            return await _context.Orders
                .Include(o => o.Items)
                .FirstOrDefaultAsync(o => o.OrderNumber == orderNumber);
        }
        
        public async Task<IEnumerable<Order>> GetByCustomerIdAsync(Guid customerId)
        {
            return await _context.Orders
                .Include(o => o.Items)
                .Where(o => o.CustomerId == customerId)
                .ToListAsync();
        }
        
        public async Task AddAsync(Order order)
        {
            await _context.Orders.AddAsync(order);
        }
        
        public Task UpdateAsync(Order order)
        {
            _context.Orders.Update(order);
            return Task.CompletedTask;
        }
        
        public Task DeleteAsync(Order order)
        {
            _context.Orders.Remove(order);
            return Task.CompletedTask;
        }
        
        public async Task<IEnumerable<Order>> GetPendingOrdersAsync()
        {
            return await _context.Orders
                .Include(o => o.Items)
                .Where(o => o.Status == OrderStatus.Pending)
                .ToListAsync();
        }
    }
}
```

### 도메인 이벤트

```csharp
namespace ECommerce.Domain.Events
{
    public interface IDomainEvent
    {
        Guid EventId { get; }
        DateTime OccurredOn { get; }
    }
    
    public abstract class DomainEvent : IDomainEvent
    {
        public Guid EventId { get; }
        public DateTime OccurredOn { get; }
        
        protected DomainEvent()
        {
            EventId = Guid.NewGuid();
            OccurredOn = DateTime.UtcNow;
        }
    }
    
    public class OrderCreatedEvent : DomainEvent
    {
        public Guid OrderId { get; }
        public Guid CustomerId { get; }
        public DateTime OrderDate { get; }
        
        public OrderCreatedEvent(Guid orderId, Guid customerId, DateTime orderDate)
        {
            OrderId = orderId;
            CustomerId = customerId;
            OrderDate = orderDate;
        }
    }
    
    public class OrderSubmittedEvent : DomainEvent
    {
        public Guid OrderId { get; }
        public Guid CustomerId { get; }
        public Money TotalAmount { get; }
        
        public OrderSubmittedEvent(Guid orderId, Guid customerId, Money totalAmount)
        {
            OrderId = orderId;
            CustomerId = customerId;
            TotalAmount = totalAmount;
        }
    }
}

// 이벤트 핸들러
namespace ECommerce.Application.EventHandlers
{
    public class OrderSubmittedEventHandler : INotificationHandler<OrderSubmittedEvent>
    {
        private readonly IInventoryService _inventoryService;
        private readonly IEmailService _emailService;
        
        public OrderSubmittedEventHandler(
            IInventoryService inventoryService,
            IEmailService emailService)
        {
            _inventoryService = inventoryService;
            _emailService = emailService;
        }
        
        public async Task Handle(OrderSubmittedEvent notification, CancellationToken cancellationToken)
        {
            // 재고 예약
            var order = await _orderRepository.GetByIdAsync(notification.OrderId);
            await _inventoryService.ReserveItems(order);
            
            // 확인 이메일 발송
            await _emailService.SendOrderConfirmationAsync(notification.CustomerId, notification.OrderId);
        }
    }
}
```

## 명세(Specification) 패턴

```csharp
namespace ECommerce.Domain.Specifications
{
    public abstract class Specification<T>
    {
        public abstract Expression<Func<T, bool>> ToExpression();
        
        public bool IsSatisfiedBy(T entity)
        {
            var predicate = ToExpression().Compile();
            return predicate(entity);
        }
        
        public Specification<T> And(Specification<T> specification)
        {
            return new AndSpecification<T>(this, specification);
        }
        
        public Specification<T> Or(Specification<T> specification)
        {
            return new OrSpecification<T>(this, specification);
        }
    }
    
    public class AndSpecification<T> : Specification<T>
    {
        private readonly Specification<T> _left;
        private readonly Specification<T> _right;
        
        public AndSpecification(Specification<T> left, Specification<T> right)
        {
            _left = left;
            _right = right;
        }
        
        public override Expression<Func<T, bool>> ToExpression()
        {
            var leftExpression = _left.ToExpression();
            var rightExpression = _right.ToExpression();
            
            var parameter = Expression.Parameter(typeof(T));
            var body = Expression.AndAlso(
                Expression.Invoke(leftExpression, parameter),
                Expression.Invoke(rightExpression, parameter)
            );
            
            return Expression.Lambda<Func<T, bool>>(body, parameter);
        }
    }
    
    // 구체적인 명세
    public class CustomerPremiumSpecification : Specification<Customer>
    {
        public override Expression<Func<Customer, bool>> ToExpression()
        {
            return customer => customer.TotalPurchases > 1000000;
        }
    }
    
    public class RecentOrderSpecification : Specification<Order>
    {
        private readonly int _days;
        
        public RecentOrderSpecification(int days)
        {
            _days = days;
        }
        
        public override Expression<Func<Order, bool>> ToExpression()
        {
            var dateThreshold = DateTime.UtcNow.AddDays(-_days);
            return order => order.OrderDate >= dateThreshold;
        }
    }
}
```

## 애플리케이션 서비스

```csharp
namespace ECommerce.Application.Services
{
    public class OrderService
    {
        private readonly IOrderRepository _orderRepository;
        private readonly ICustomerRepository _customerRepository;
        private readonly IInventoryService _inventoryService;
        private readonly IUnitOfWork _unitOfWork;
        private readonly IEventDispatcher _eventDispatcher;
        
        public OrderService(
            IOrderRepository orderRepository,
            ICustomerRepository customerRepository,
            IInventoryService inventoryService,
            IUnitOfWork unitOfWork,
            IEventDispatcher eventDispatcher)
        {
            _orderRepository = orderRepository;
            _customerRepository = customerRepository;
            _inventoryService = inventoryService;
            _unitOfWork = unitOfWork;
            _eventDispatcher = eventDispatcher;
        }
        
        public async Task<OrderDto> CreateOrderAsync(CreateOrderCommand command)
        {
            // 고객 확인
            var customer = await _customerRepository.GetByIdAsync(command.CustomerId);
            if (customer == null)
                throw new CustomerNotFoundException(command.CustomerId);
                
            // 주문 생성
            var order = new Order(customer.Id, customer.ShippingAddress);
            
            // 상품 추가 및 재고 확인
            foreach (var item in command.Items)
            {
                var isAvailable = await _inventoryService.CheckAvailability(
                    item.ProductId, item.Quantity);
                    
                if (!isAvailable)
                    throw new InsufficientInventoryException(item.ProductId);
                    
                order.AddItem(item.ProductId, item.ProductName, 
                    new Money(item.UnitPrice, "KRW"), item.Quantity);
            }
            
            // 저장
            await _orderRepository.AddAsync(order);
            await _unitOfWork.SaveChangesAsync();
            
            // 도메인 이벤트 발행
            await _eventDispatcher.DispatchAsync(order.DomainEvents);
            
            return OrderDto.FromDomain(order);
        }
        
        public async Task SubmitOrderAsync(Guid orderId)
        {
            var order = await _orderRepository.GetByIdAsync(orderId);
            if (order == null)
                throw new OrderNotFoundException(orderId);
                
            order.Submit();
            
            await _unitOfWork.SaveChangesAsync();
            await _eventDispatcher.DispatchAsync(order.DomainEvents);
        }
    }
}
```

## DDD 적용 모범 사례

### 1. 유비쿼터스 언어
- 도메인 전문가와 동일한 용어 사용
- 코드에 비즈니스 용어 직접 반영
- 모호한 기술 용어 대신 도메인 용어 사용

### 2. 애그리게이트 설계
- 트랜잭션 경계를 고려한 크기 설정
- 불변식 보호를 위한 캡슐화
- ID를 통한 애그리게이트 간 참조

### 3. 도메인 이벤트 활용
- 비즈니스 이벤트를 명시적으로 모델링
- 느슨한 결합을 위한 이벤트 기반 통신
- 이벤트 소싱과 CQRS 고려

### 4. 리포지토리 패턴
- 애그리게이트 단위로 저장소 접근
- 도메인 로직과 영속성 분리
- 명세 패턴을 통한 유연한 쿼리

## 마무리

DDD는 복잡한 비즈니스 도메인을 효과적으로 모델링하고 구현하는 강력한 방법론입니다. 단순히 기술적인 패턴을 적용하는 것이 아니라, 비즈니스 도메인에 대한 깊은 이해를 바탕으로 소프트웨어를 설계하는 것이 핵심입니다.