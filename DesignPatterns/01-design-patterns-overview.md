# 디자인 패턴 개요와 SOLID 원칙

## 디자인 패턴이란?

디자인 패턴은 소프트웨어 설계에서 자주 발생하는 문제들에 대한 재사용 가능한 해결책입니다. 1994년 GoF(Gang of Four)가 출간한 "Design Patterns: Elements of Reusable Object-Oriented Software"에서 23개의 패턴을 소개하면서 널리 알려지게 되었습니다.

### 디자인 패턴의 중요성

```csharp
// 패턴을 사용하지 않은 코드
public class OrderService
{
    public void ProcessOrder(Order order)
    {
        // 데이터베이스 연결
        using (var connection = new SqlConnection("..."))
        {
            connection.Open();
            // 주문 처리 로직
            
            // 이메일 발송
            var smtpClient = new SmtpClient();
            smtpClient.Send("order@example.com", "주문 완료");
            
            // 로깅
            File.AppendAllText("log.txt", $"주문 처리: {order.Id}");
        }
    }
}

// 패턴을 적용한 코드
public interface IOrderRepository
{
    void Save(Order order);
}

public interface IEmailService
{
    void SendOrderConfirmation(Order order);
}

public interface ILogger
{
    void Log(string message);
}

public class OrderService
{
    private readonly IOrderRepository _repository;
    private readonly IEmailService _emailService;
    private readonly ILogger _logger;
    
    public OrderService(
        IOrderRepository repository,
        IEmailService emailService,
        ILogger logger)
    {
        _repository = repository;
        _emailService = emailService;
        _logger = logger;
    }
    
    public void ProcessOrder(Order order)
    {
        _repository.Save(order);
        _emailService.SendOrderConfirmation(order);
        _logger.Log($"주문 처리: {order.Id}");
    }
}
```

### 패턴의 구성 요소

1. **패턴 이름**: 문제와 해결책을 한 두 단어로 설명
2. **문제**: 언제 패턴을 적용할지 설명
3. **해결책**: 설계를 구성하는 요소들과 관계
4. **결과**: 패턴 적용의 결과와 장단점

## SOLID 원칙 심화

SOLID는 객체지향 설계의 5가지 기본 원칙으로, 유지보수가 쉽고 확장 가능한 소프트웨어를 만드는 지침입니다.

### 1. Single Responsibility Principle (단일 책임 원칙)

클래스는 하나의 책임만 가져야 하며, 변경의 이유도 하나여야 합니다.

```csharp
// SRP 위반
public class Employee
{
    public string Name { get; set; }
    public decimal Salary { get; set; }
    
    public void Save()
    {
        // 데이터베이스에 저장
    }
    
    public void CalculateTax()
    {
        // 세금 계산
    }
    
    public void SendEmail()
    {
        // 이메일 발송
    }
}

// SRP 준수
public class Employee
{
    public string Name { get; set; }
    public decimal Salary { get; set; }
}

public class EmployeeRepository
{
    public void Save(Employee employee)
    {
        // 데이터베이스에 저장
    }
}

public class TaxCalculator
{
    public decimal CalculateTax(Employee employee)
    {
        // 세금 계산
        return employee.Salary * 0.15m;
    }
}

public class EmailService
{
    public void SendEmail(Employee employee, string message)
    {
        // 이메일 발송
    }
}
```

### 2. Open/Closed Principle (개방-폐쇄 원칙)

소프트웨어 요소는 확장에는 열려 있되, 수정에는 닫혀 있어야 합니다.

```csharp
// OCP 위반
public class DiscountCalculator
{
    public decimal CalculateDiscount(CustomerType customerType, decimal amount)
    {
        switch (customerType)
        {
            case CustomerType.Regular:
                return amount * 0.05m;
            case CustomerType.Silver:
                return amount * 0.10m;
            case CustomerType.Gold:
                return amount * 0.15m;
            // 새로운 고객 유형을 추가하려면 이 메서드를 수정해야 함
            default:
                return 0;
        }
    }
}

// OCP 준수
public interface IDiscountStrategy
{
    decimal CalculateDiscount(decimal amount);
}

public class RegularDiscount : IDiscountStrategy
{
    public decimal CalculateDiscount(decimal amount)
    {
        return amount * 0.05m;
    }
}

public class SilverDiscount : IDiscountStrategy
{
    public decimal CalculateDiscount(decimal amount)
    {
        return amount * 0.10m;
    }
}

public class GoldDiscount : IDiscountStrategy
{
    public decimal CalculateDiscount(decimal amount)
    {
        return amount * 0.15m;
    }
}

// 새로운 할인 전략 추가 시 기존 코드 수정 없음
public class PlatinumDiscount : IDiscountStrategy
{
    public decimal CalculateDiscount(decimal amount)
    {
        return amount * 0.20m;
    }
}

public class DiscountCalculator
{
    public decimal CalculateDiscount(IDiscountStrategy strategy, decimal amount)
    {
        return strategy.CalculateDiscount(amount);
    }
}
```

### 3. Liskov Substitution Principle (리스코프 치환 원칙)

파생 클래스는 기반 클래스를 대체할 수 있어야 합니다.

```csharp
// LSP 위반
public class Rectangle
{
    public virtual int Width { get; set; }
    public virtual int Height { get; set; }
    
    public int CalculateArea()
    {
        return Width * Height;
    }
}

public class Square : Rectangle
{
    private int _side;
    
    public override int Width
    {
        get { return _side; }
        set { _side = value; }
    }
    
    public override int Height
    {
        get { return _side; }
        set { _side = value; }
    }
}

// 문제가 되는 코드
public void ProcessRectangle(Rectangle rectangle)
{
    rectangle.Width = 4;
    rectangle.Height = 5;
    // Rectangle이면 20, Square이면 25가 됨
    Console.WriteLine($"Area: {rectangle.CalculateArea()}");
}

// LSP 준수
public interface IShape
{
    int CalculateArea();
}

public class Rectangle : IShape
{
    public int Width { get; set; }
    public int Height { get; set; }
    
    public int CalculateArea()
    {
        return Width * Height;
    }
}

public class Square : IShape
{
    public int Side { get; set; }
    
    public int CalculateArea()
    {
        return Side * Side;
    }
}
```

### 4. Interface Segregation Principle (인터페이스 분리 원칙)

클라이언트는 자신이 사용하지 않는 메서드에 의존하지 않아야 합니다.

```csharp
// ISP 위반
public interface IWorker
{
    void Work();
    void Eat();
    void Sleep();
}

public class Human : IWorker
{
    public void Work() { /* 일하기 */ }
    public void Eat() { /* 먹기 */ }
    public void Sleep() { /* 자기 */ }
}

public class Robot : IWorker
{
    public void Work() { /* 일하기 */ }
    public void Eat() 
    { 
        throw new NotImplementedException(); // 로봇은 먹지 않음
    }
    public void Sleep() 
    { 
        throw new NotImplementedException(); // 로봇은 자지 않음
    }
}

// ISP 준수
public interface IWorkable
{
    void Work();
}

public interface IEatable
{
    void Eat();
}

public interface ISleepable
{
    void Sleep();
}

public class Human : IWorkable, IEatable, ISleepable
{
    public void Work() { /* 일하기 */ }
    public void Eat() { /* 먹기 */ }
    public void Sleep() { /* 자기 */ }
}

public class Robot : IWorkable
{
    public void Work() { /* 일하기 */ }
}
```

### 5. Dependency Inversion Principle (의존성 역전 원칙)

고수준 모듈은 저수준 모듈에 의존하지 않아야 하며, 둘 다 추상화에 의존해야 합니다.

```csharp
// DIP 위반
public class EmailNotification
{
    public void Send(string message)
    {
        // 이메일 발송 로직
    }
}

public class OrderService
{
    private EmailNotification _emailNotification = new EmailNotification();
    
    public void ProcessOrder(Order order)
    {
        // 주문 처리
        _emailNotification.Send("주문이 처리되었습니다.");
    }
}

// DIP 준수
public interface INotificationService
{
    void Send(string message);
}

public class EmailNotification : INotificationService
{
    public void Send(string message)
    {
        // 이메일 발송 로직
    }
}

public class SmsNotification : INotificationService
{
    public void Send(string message)
    {
        // SMS 발송 로직
    }
}

public class OrderService
{
    private readonly INotificationService _notificationService;
    
    public OrderService(INotificationService notificationService)
    {
        _notificationService = notificationService;
    }
    
    public void ProcessOrder(Order order)
    {
        // 주문 처리
        _notificationService.Send("주문이 처리되었습니다.");
    }
}
```

## 패턴 vs 안티패턴

### 안티패턴의 예시

1. **God Object**: 너무 많은 책임을 가진 클래스
```csharp
// 안티패턴
public class ApplicationManager
{
    public void ConnectDatabase() { }
    public void LoadConfiguration() { }
    public void AuthenticateUser() { }
    public void ProcessPayment() { }
    public void SendEmail() { }
    public void GenerateReport() { }
    // ... 수십 개의 메서드
}
```

2. **Spaghetti Code**: 구조화되지 않은 복잡한 코드
```csharp
// 안티패턴
public void ProcessData(string input)
{
    if (input != null)
    {
        if (input.Length > 0)
        {
            if (input.Contains("special"))
            {
                // 깊은 중첩
                for (int i = 0; i < input.Length; i++)
                {
                    if (input[i] == 'a')
                    {
                        // 더 깊은 중첩...
                    }
                }
            }
        }
    }
}
```

3. **Copy-Paste Programming**: 코드 복사-붙여넣기
```csharp
// 안티패턴
public void ProcessCustomerOrder() 
{
    // 100줄의 코드
}

public void ProcessVendorOrder() 
{
    // 위와 거의 동일한 100줄의 코드
}
```

### 올바른 패턴 적용

```csharp
// 리팩토링된 코드
public abstract class OrderProcessor
{
    public void ProcessOrder()
    {
        ValidateOrder();
        CalculateTotal();
        ApplyDiscount();
        SaveOrder();
        NotifyUser();
    }
    
    protected abstract void ValidateOrder();
    protected abstract void ApplyDiscount();
    
    protected virtual void CalculateTotal() { /* 공통 로직 */ }
    protected virtual void SaveOrder() { /* 공통 로직 */ }
    protected virtual void NotifyUser() { /* 공통 로직 */ }
}

public class CustomerOrderProcessor : OrderProcessor
{
    protected override void ValidateOrder()
    {
        // 고객 주문 검증 로직
    }
    
    protected override void ApplyDiscount()
    {
        // 고객 할인 적용
    }
}
```

## UML 클래스 다이어그램 읽기

### 기본 요소

1. **클래스 표현**
```
┌─────────────────┐
│   ClassName     │
├─────────────────┤
│ - privateField  │
│ + publicField   │
├─────────────────┤
│ + method()      │
│ # protectedMeth │
└─────────────────┘
```

2. **관계 표현**
- 상속: ──▷ (빈 삼각형)
- 구현: ╌╌▷ (점선 + 빈 삼각형)
- 연관: ───
- 의존: ╌╌>
- 집합: ◇── (빈 다이아몬드)
- 합성: ◆── (채워진 다이아몬드)

### C# 코드로 표현

```csharp
// 상속 관계
public abstract class Animal
{
    public abstract void MakeSound();
}

public class Dog : Animal  // Dog ──▷ Animal
{
    public override void MakeSound()
    {
        Console.WriteLine("Woof!");
    }
}

// 인터페이스 구현
public interface IFlyable
{
    void Fly();
}

public class Bird : Animal, IFlyable  // Bird ╌╌▷ IFlyable
{
    public override void MakeSound()
    {
        Console.WriteLine("Tweet!");
    }
    
    public void Fly()
    {
        Console.WriteLine("Flying...");
    }
}

// 연관 관계
public class Person
{
    public Address Address { get; set; }  // Person ─── Address
}

// 의존 관계
public class OrderService
{
    public void ProcessOrder(Order order)  // OrderService ╌╌> Order
    {
        // Order를 매개변수로 사용
    }
}

// 집합 관계
public class Department
{
    public List<Employee> Employees { get; set; }  // Department ◇── Employee
    // Employee는 Department 없이도 존재 가능
}

// 합성 관계
public class House
{
    private List<Room> rooms;  // House ◆── Room
    
    public House()
    {
        rooms = new List<Room>();
        // Room은 House 없이 존재할 수 없음
    }
}
```

## 패턴 학습을 위한 실습 프로젝트

### 온라인 쇼핑몰 시스템 설계

```csharp
// 도메인 모델
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
    public int StockQuantity { get; set; }
}

public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Email { get; set; }
    public CustomerType Type { get; set; }
}

public class Order
{
    public int Id { get; set; }
    public Customer Customer { get; set; }
    public List<OrderItem> Items { get; set; }
    public DateTime OrderDate { get; set; }
    public OrderStatus Status { get; set; }
}

// 서비스 레이어 (여러 패턴 적용)
public interface IProductRepository
{
    Product GetById(int id);
    void Update(Product product);
}

public interface IOrderRepository
{
    void Save(Order order);
    Order GetById(int id);
}

public interface IPaymentGateway
{
    PaymentResult ProcessPayment(PaymentInfo info);
}

public interface IInventoryService
{
    bool CheckAvailability(int productId, int quantity);
    void UpdateStock(int productId, int quantity);
}

public class OrderService
{
    private readonly IOrderRepository _orderRepository;
    private readonly IProductRepository _productRepository;
    private readonly IPaymentGateway _paymentGateway;
    private readonly IInventoryService _inventoryService;
    private readonly INotificationService _notificationService;
    private readonly IDiscountStrategy _discountStrategy;
    
    public OrderService(
        IOrderRepository orderRepository,
        IProductRepository productRepository,
        IPaymentGateway paymentGateway,
        IInventoryService inventoryService,
        INotificationService notificationService,
        IDiscountStrategy discountStrategy)
    {
        _orderRepository = orderRepository;
        _productRepository = productRepository;
        _paymentGateway = paymentGateway;
        _inventoryService = inventoryService;
        _notificationService = notificationService;
        _discountStrategy = discountStrategy;
    }
    
    public async Task<OrderResult> PlaceOrderAsync(Order order)
    {
        // 1. 재고 확인
        foreach (var item in order.Items)
        {
            if (!_inventoryService.CheckAvailability(item.ProductId, item.Quantity))
            {
                return new OrderResult { Success = false, Message = "재고 부족" };
            }
        }
        
        // 2. 가격 계산 및 할인 적용
        var totalAmount = CalculateTotal(order);
        var discountAmount = _discountStrategy.CalculateDiscount(totalAmount);
        var finalAmount = totalAmount - discountAmount;
        
        // 3. 결제 처리
        var paymentResult = await _paymentGateway.ProcessPaymentAsync(
            new PaymentInfo { Amount = finalAmount, CustomerId = order.Customer.Id });
            
        if (!paymentResult.Success)
        {
            return new OrderResult { Success = false, Message = "결제 실패" };
        }
        
        // 4. 재고 업데이트
        foreach (var item in order.Items)
        {
            _inventoryService.UpdateStock(item.ProductId, -item.Quantity);
        }
        
        // 5. 주문 저장
        order.Status = OrderStatus.Confirmed;
        _orderRepository.Save(order);
        
        // 6. 알림 발송
        await _notificationService.SendAsync($"주문 {order.Id}가 확인되었습니다.");
        
        return new OrderResult { Success = true, OrderId = order.Id };
    }
    
    private decimal CalculateTotal(Order order)
    {
        decimal total = 0;
        foreach (var item in order.Items)
        {
            var product = _productRepository.GetById(item.ProductId);
            total += product.Price * item.Quantity;
        }
        return total;
    }
}
```

## 마무리

디자인 패턴은 검증된 해결책을 제공하지만, 모든 상황에 적합한 것은 아닙니다. 패턴을 적용하기 전에 다음을 고려해야 합니다:

1. **문제를 정확히 이해**: 패턴이 해결하려는 문제가 실제로 존재하는가?
2. **복잡도 증가**: 패턴 적용이 오히려 복잡도를 증가시키지는 않는가?
3. **팀의 이해도**: 팀원들이 패턴을 이해하고 유지보수할 수 있는가?
4. **성능 영향**: 패턴 적용이 성능에 부정적 영향을 미치지는 않는가?

다음 장에서는 객체지향 설계 원칙과 패턴 적용 전략에 대해 더 자세히 알아보겠습니다.