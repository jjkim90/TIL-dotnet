# 객체지향 설계 원칙과 패턴 적용 전략

## 개요

좋은 소프트웨어 설계는 단순히 동작하는 코드를 작성하는 것이 아니라, 유지보수가 쉽고 확장 가능하며 이해하기 쉬운 코드를 만드는 것입니다. 이 장에서는 SOLID 원칙 외에도 중요한 객체지향 설계 원칙들과 패턴을 적용하는 전략에 대해 알아봅니다.

## DRY (Don't Repeat Yourself)

### DRY 원칙의 의미

"모든 지식은 시스템 내에서 단일하고 명확하며 권위있는 표현을 가져야 한다"는 원칙입니다.

```csharp
// DRY 위반 예시
public class OrderService
{
    public decimal CalculateOrderTotal(List<OrderItem> items)
    {
        decimal total = 0;
        foreach (var item in items)
        {
            decimal itemPrice = item.Price * item.Quantity;
            decimal tax = itemPrice * 0.08m; // 세율 8%
            total += itemPrice + tax;
        }
        return total;
    }
    
    public decimal CalculateCartTotal(List<CartItem> items)
    {
        decimal total = 0;
        foreach (var item in items)
        {
            decimal itemPrice = item.Price * item.Quantity;
            decimal tax = itemPrice * 0.08m; // 세율 8% (중복!)
            total += itemPrice + tax;
        }
        return total;
    }
}

// DRY 원칙 적용
public interface IPriceable
{
    decimal Price { get; }
    int Quantity { get; }
}

public class PriceCalculator
{
    private const decimal TaxRate = 0.08m;
    
    public decimal CalculateTotal<T>(IEnumerable<T> items) where T : IPriceable
    {
        return items.Sum(item => CalculateItemTotal(item));
    }
    
    private decimal CalculateItemTotal(IPriceable item)
    {
        var subtotal = item.Price * item.Quantity;
        var tax = subtotal * TaxRate;
        return subtotal + tax;
    }
}

public class OrderService
{
    private readonly PriceCalculator _priceCalculator;
    
    public OrderService(PriceCalculator priceCalculator)
    {
        _priceCalculator = priceCalculator;
    }
    
    public decimal CalculateOrderTotal(List<OrderItem> items)
    {
        return _priceCalculator.CalculateTotal(items);
    }
    
    public decimal CalculateCartTotal(List<CartItem> items)
    {
        return _priceCalculator.CalculateTotal(items);
    }
}
```

### DRY의 함정

DRY를 잘못 이해하면 과도한 추상화를 만들 수 있습니다.

```csharp
// 과도한 DRY - 우연의 일치를 제거하려는 시도
public class User
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
}

public class Product
{
    public string Name { get; set; }
    public string Category { get; set; }
}

// 잘못된 추상화 - User와 Product는 본질적으로 다름
public abstract class TwoStringEntity
{
    public string Field1 { get; set; }
    public string Field2 { get; set; }
}

// 올바른 접근 - 진짜 중복만 제거
public interface IEntity
{
    int Id { get; set; }
    DateTime CreatedAt { get; set; }
    DateTime UpdatedAt { get; set; }
}

public class User : IEntity
{
    public int Id { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
}
```

## KISS (Keep It Simple, Stupid)

### 단순함의 중요성

복잡한 해결책보다 단순한 해결책이 대부분 더 낫습니다.

```csharp
// 과도하게 복잡한 예시
public interface IDataProcessor<T>
{
    T Process(T data);
}

public abstract class AbstractDataProcessor<T> : IDataProcessor<T>
{
    protected abstract T PreProcess(T data);
    protected abstract T MainProcess(T data);
    protected abstract T PostProcess(T data);
    
    public T Process(T data)
    {
        var preprocessed = PreProcess(data);
        var processed = MainProcess(preprocessed);
        return PostProcess(processed);
    }
}

public class StringUpperCaseProcessor : AbstractDataProcessor<string>
{
    protected override string PreProcess(string data) => data?.Trim();
    protected override string MainProcess(string data) => data?.ToUpper();
    protected override string PostProcess(string data) => data;
}

// KISS 원칙 적용
public class StringHelper
{
    public static string ToUpperCase(string input)
    {
        return input?.Trim().ToUpper();
    }
}
```

### 적절한 복잡도 판단

```csharp
// 상황에 따른 적절한 복잡도 선택
public class ValidationService
{
    // 단순한 경우 - 직접 구현
    public bool IsValidEmail(string email)
    {
        if (string.IsNullOrWhiteSpace(email))
            return false;
            
        return email.Contains("@") && email.Contains(".");
    }
    
    // 복잡한 경우 - 패턴 사용
    private readonly List<IValidationRule<Customer>> _customerRules;
    
    public ValidationResult ValidateCustomer(Customer customer)
    {
        var result = new ValidationResult();
        
        foreach (var rule in _customerRules)
        {
            if (!rule.IsSatisfiedBy(customer))
            {
                result.AddError(rule.ErrorMessage);
            }
        }
        
        return result;
    }
}

public interface IValidationRule<T>
{
    bool IsSatisfiedBy(T entity);
    string ErrorMessage { get; }
}

public class CustomerAgeRule : IValidationRule<Customer>
{
    public string ErrorMessage => "고객은 18세 이상이어야 합니다.";
    
    public bool IsSatisfiedBy(Customer customer)
    {
        return customer.Age >= 18;
    }
}
```

## YAGNI (You Aren't Gonna Need It)

### 미래를 위한 과도한 설계 피하기

```csharp
// YAGNI 위반 - 현재 필요하지 않은 기능
public interface IRepository<T>
{
    T GetById(int id);
    IEnumerable<T> GetAll();
    void Add(T entity);
    void Update(T entity);
    void Delete(T entity);
    
    // 현재 사용하지 않는 메서드들
    IEnumerable<T> GetByPredicate(Expression<Func<T, bool>> predicate);
    void BulkInsert(IEnumerable<T> entities);
    void BulkUpdate(IEnumerable<T> entities);
    void BulkDelete(IEnumerable<T> entities);
    Task<T> GetByIdAsync(int id);
    Task<IEnumerable<T>> GetAllAsync();
    // ... 더 많은 미사용 메서드
}

// YAGNI 적용 - 필요한 것만 구현
public interface IProductRepository
{
    Product GetById(int id);
    IEnumerable<Product> GetAll();
    void Save(Product product);
}

public class ProductRepository : IProductRepository
{
    private readonly DbContext _context;
    
    public Product GetById(int id)
    {
        return _context.Products.Find(id);
    }
    
    public IEnumerable<Product> GetAll()
    {
        return _context.Products.ToList();
    }
    
    public void Save(Product product)
    {
        if (product.Id == 0)
            _context.Products.Add(product);
        else
            _context.Entry(product).State = EntityState.Modified;
            
        _context.SaveChanges();
    }
}
```

### 진화적 설계

```csharp
// 1단계: 단순한 시작
public class EmailService
{
    public void SendEmail(string to, string subject, string body)
    {
        // SMTP로 이메일 전송
    }
}

// 2단계: 요구사항이 생기면 확장
public interface IEmailService
{
    void SendEmail(string to, string subject, string body);
}

public class SmtpEmailService : IEmailService
{
    public void SendEmail(string to, string subject, string body)
    {
        // SMTP 구현
    }
}

// 3단계: 새로운 요구사항에 따라 추가 확장
public class SendGridEmailService : IEmailService
{
    public void SendEmail(string to, string subject, string body)
    {
        // SendGrid API 사용
    }
}
```

## 패턴 선택 가이드

### 문제 인식이 먼저

```csharp
// 문제: 객체 생성 로직이 복잡함
public class ReportGenerator
{
    public Report CreateReport(ReportType type, ReportData data)
    {
        Report report = null;
        
        switch (type)
        {
            case ReportType.Sales:
                report = new SalesReport();
                report.Title = "Sales Report";
                report.AddSection(new SummarySection(data));
                report.AddSection(new DetailSection(data));
                break;
                
            case ReportType.Inventory:
                report = new InventoryReport();
                report.Title = "Inventory Report";
                report.AddSection(new StockLevelSection(data));
                report.AddSection(new ReorderSection(data));
                break;
                
            // 더 많은 케이스...
        }
        
        return report;
    }
}

// 해결: Factory Pattern 적용
public interface IReportFactory
{
    Report CreateReport(ReportData data);
}

public class SalesReportFactory : IReportFactory
{
    public Report CreateReport(ReportData data)
    {
        var report = new SalesReport
        {
            Title = "Sales Report"
        };
        
        report.AddSection(new SummarySection(data));
        report.AddSection(new DetailSection(data));
        
        return report;
    }
}

public class ReportGenerator
{
    private readonly Dictionary<ReportType, IReportFactory> _factories;
    
    public ReportGenerator()
    {
        _factories = new Dictionary<ReportType, IReportFactory>
        {
            { ReportType.Sales, new SalesReportFactory() },
            { ReportType.Inventory, new InventoryReportFactory() }
        };
    }
    
    public Report CreateReport(ReportType type, ReportData data)
    {
        if (_factories.TryGetValue(type, out var factory))
        {
            return factory.CreateReport(data);
        }
        
        throw new NotSupportedException($"Report type {type} is not supported");
    }
}
```

### 패턴 적용 체크리스트

```csharp
public class PatternDecisionHelper
{
    // Singleton 패턴을 사용해야 하는가?
    public bool ShouldUseSingleton()
    {
        // ✓ 시스템 전체에서 하나의 인스턴스만 필요한가?
        // ✓ 전역적 접근이 필요한가?
        // ✓ 지연 초기화가 필요한가?
        // ✗ 단위 테스트가 어려워져도 괜찮은가?
        // ✗ 의존성 주입으로 해결할 수 있는가?
        
        return false; // 대부분의 경우 DI가 더 나은 선택
    }
    
    // Strategy 패턴을 사용해야 하는가?
    public bool ShouldUseStrategy()
    {
        // ✓ 런타임에 알고리즘을 선택해야 하는가?
        // ✓ 여러 알고리즘의 변형이 존재하는가?
        // ✓ 알고리즘이 독립적으로 변경될 수 있는가?
        
        return true;
    }
}

// 실제 적용 예시
public interface IPricingStrategy
{
    decimal CalculatePrice(Product product, Customer customer);
}

public class RegularPricingStrategy : IPricingStrategy
{
    public decimal CalculatePrice(Product product, Customer customer)
    {
        return product.BasePrice;
    }
}

public class VipPricingStrategy : IPricingStrategy
{
    public decimal CalculatePrice(Product product, Customer customer)
    {
        return product.BasePrice * 0.8m; // 20% 할인
    }
}

public class SeasonalPricingStrategy : IPricingStrategy
{
    private readonly Season _season;
    
    public SeasonalPricingStrategy(Season season)
    {
        _season = season;
    }
    
    public decimal CalculatePrice(Product product, Customer customer)
    {
        return _season switch
        {
            Season.Summer => product.BasePrice * 0.9m,
            Season.Winter => product.BasePrice * 1.1m,
            _ => product.BasePrice
        };
    }
}
```

## 과도한 패턴 사용의 위험

### 패턴 중독 (Pattern Addiction)

```csharp
// 과도한 패턴 사용
public interface IStringProcessor
{
    string Process(string input);
}

public abstract class StringProcessorDecorator : IStringProcessor
{
    protected IStringProcessor _processor;
    
    public StringProcessorDecorator(IStringProcessor processor)
    {
        _processor = processor;
    }
    
    public abstract string Process(string input);
}

public class UpperCaseDecorator : StringProcessorDecorator
{
    public UpperCaseDecorator(IStringProcessor processor) : base(processor) { }
    
    public override string Process(string input)
    {
        return _processor.Process(input).ToUpper();
    }
}

public class TrimDecorator : StringProcessorDecorator
{
    public TrimDecorator(IStringProcessor processor) : base(processor) { }
    
    public override string Process(string input)
    {
        return _processor.Process(input).Trim();
    }
}

// 사용
var processor = new TrimDecorator(new UpperCaseDecorator(new BasicStringProcessor()));
var result = processor.Process("  hello  ");

// 단순한 해결책
public static class StringExtensions
{
    public static string ProcessString(this string input)
    {
        return input?.Trim().ToUpper();
    }
}

var result = "  hello  ".ProcessString();
```

### 적절한 균형 찾기

```csharp
public class OrderProcessingExample
{
    // 너무 단순한 접근
    public class SimpleOrderService
    {
        public void ProcessOrder(Order order)
        {
            // 1000줄의 코드...
            // 모든 로직이 한 곳에
        }
    }
    
    // 과도하게 복잡한 접근
    public interface IOrderValidator { }
    public interface IOrderCalculator { }
    public interface IOrderPersister { }
    public interface IOrderNotifier { }
    public interface IOrderLogger { }
    public interface IOrderEventPublisher { }
    // ... 수십 개의 인터페이스
    
    // 적절한 균형
    public class BalancedOrderService
    {
        private readonly IOrderRepository _repository;
        private readonly IPaymentService _paymentService;
        private readonly INotificationService _notificationService;
        
        public BalancedOrderService(
            IOrderRepository repository,
            IPaymentService paymentService,
            INotificationService notificationService)
        {
            _repository = repository;
            _paymentService = paymentService;
            _notificationService = notificationService;
        }
        
        public async Task<OrderResult> ProcessOrderAsync(Order order)
        {
            // 검증
            var validationResult = ValidateOrder(order);
            if (!validationResult.IsValid)
                return OrderResult.Failed(validationResult.Errors);
            
            // 결제 처리
            var paymentResult = await _paymentService.ProcessPaymentAsync(order);
            if (!paymentResult.IsSuccessful)
                return OrderResult.Failed("결제 실패");
            
            // 저장
            order.Status = OrderStatus.Confirmed;
            order.PaymentId = paymentResult.PaymentId;
            await _repository.SaveAsync(order);
            
            // 알림
            await _notificationService.SendOrderConfirmationAsync(order);
            
            return OrderResult.Success(order.Id);
        }
        
        private ValidationResult ValidateOrder(Order order)
        {
            var result = new ValidationResult();
            
            if (order.Items == null || !order.Items.Any())
                result.AddError("주문 항목이 없습니다.");
                
            if (order.TotalAmount <= 0)
                result.AddError("주문 금액이 올바르지 않습니다.");
                
            return result;
        }
    }
}
```

## 리팩토링과 패턴

### 패턴으로의 리팩토링

```csharp
// 시작: 조건문이 많은 코드
public class DocumentProcessor
{
    public void ProcessDocument(Document doc)
    {
        if (doc.Type == "PDF")
        {
            // PDF 처리 로직
            ConvertPdfToText(doc);
            ExtractPdfMetadata(doc);
        }
        else if (doc.Type == "Word")
        {
            // Word 처리 로직
            ConvertWordToText(doc);
            ExtractWordMetadata(doc);
        }
        else if (doc.Type == "Excel")
        {
            // Excel 처리 로직
            ConvertExcelToText(doc);
            ExtractExcelMetadata(doc);
        }
    }
}

// 1단계: 메서드 추출
public class DocumentProcessorStep1
{
    public void ProcessDocument(Document doc)
    {
        switch (doc.Type)
        {
            case "PDF":
                ProcessPdf(doc);
                break;
            case "Word":
                ProcessWord(doc);
                break;
            case "Excel":
                ProcessExcel(doc);
                break;
        }
    }
    
    private void ProcessPdf(Document doc) { /* PDF 로직 */ }
    private void ProcessWord(Document doc) { /* Word 로직 */ }
    private void ProcessExcel(Document doc) { /* Excel 로직 */ }
}

// 2단계: Strategy 패턴 적용
public interface IDocumentProcessor
{
    void Process(Document document);
}

public class PdfProcessor : IDocumentProcessor
{
    public void Process(Document document)
    {
        ConvertPdfToText(document);
        ExtractPdfMetadata(document);
    }
}

public class WordProcessor : IDocumentProcessor
{
    public void Process(Document document)
    {
        ConvertWordToText(document);
        ExtractWordMetadata(document);
    }
}

public class DocumentProcessorService
{
    private readonly Dictionary<string, IDocumentProcessor> _processors;
    
    public DocumentProcessorService()
    {
        _processors = new Dictionary<string, IDocumentProcessor>
        {
            ["PDF"] = new PdfProcessor(),
            ["Word"] = new WordProcessor(),
            ["Excel"] = new ExcelProcessor()
        };
    }
    
    public void ProcessDocument(Document doc)
    {
        if (_processors.TryGetValue(doc.Type, out var processor))
        {
            processor.Process(doc);
        }
        else
        {
            throw new NotSupportedException($"Document type {doc.Type} is not supported");
        }
    }
}
```

### 점진적 개선

```csharp
// 레거시 코드
public class LegacyPaymentService
{
    public bool ProcessPayment(decimal amount, string cardNumber, string cvv)
    {
        // 복잡한 레거시 로직
        return true;
    }
}

// Adapter 패턴으로 점진적 개선
public interface IPaymentGateway
{
    Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request);
}

public class LegacyPaymentAdapter : IPaymentGateway
{
    private readonly LegacyPaymentService _legacyService;
    
    public LegacyPaymentAdapter(LegacyPaymentService legacyService)
    {
        _legacyService = legacyService;
    }
    
    public Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        var success = _legacyService.ProcessPayment(
            request.Amount, 
            request.CardNumber, 
            request.CVV);
            
        var result = new PaymentResult
        {
            IsSuccessful = success,
            TransactionId = Guid.NewGuid().ToString()
        };
        
        return Task.FromResult(result);
    }
}

// 새로운 구현체 추가 가능
public class StripePaymentGateway : IPaymentGateway
{
    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request)
    {
        // Stripe API 호출
        return new PaymentResult { IsSuccessful = true };
    }
}
```

## 실전 패턴 적용 사례

### 복잡한 비즈니스 로직 처리

```csharp
// 주문 할인 계산 - 여러 규칙이 복합적으로 적용
public class DiscountCalculationService
{
    private readonly List<IDiscountRule> _rules;
    
    public DiscountCalculationService()
    {
        _rules = new List<IDiscountRule>
        {
            new QuantityDiscountRule(),
            new CustomerLoyaltyDiscountRule(),
            new SeasonalDiscountRule(),
            new CouponDiscountRule()
        };
    }
    
    public decimal CalculateTotalDiscount(Order order)
    {
        var context = new DiscountContext(order);
        
        foreach (var rule in _rules)
        {
            rule.Apply(context);
        }
        
        return context.TotalDiscount;
    }
}

public interface IDiscountRule
{
    void Apply(DiscountContext context);
}

public class DiscountContext
{
    public Order Order { get; }
    public decimal TotalDiscount { get; set; }
    public List<string> AppliedDiscounts { get; } = new List<string>();
    
    public DiscountContext(Order order)
    {
        Order = order;
    }
    
    public void AddDiscount(decimal amount, string reason)
    {
        TotalDiscount += amount;
        AppliedDiscounts.Add($"{reason}: {amount:C}");
    }
}

public class QuantityDiscountRule : IDiscountRule
{
    public void Apply(DiscountContext context)
    {
        var totalQuantity = context.Order.Items.Sum(i => i.Quantity);
        
        if (totalQuantity >= 10)
        {
            var discount = context.Order.SubTotal * 0.1m;
            context.AddDiscount(discount, "수량 할인 10%");
        }
    }
}

public class CustomerLoyaltyDiscountRule : IDiscountRule
{
    public void Apply(DiscountContext context)
    {
        if (context.Order.Customer.LoyaltyYears >= 5)
        {
            var discount = context.Order.SubTotal * 0.05m;
            context.AddDiscount(discount, "단골 고객 할인 5%");
        }
    }
}
```

## 마무리

객체지향 설계 원칙들은 서로 보완적인 관계에 있습니다:

1. **DRY**: 중복을 제거하되, 진짜 중복인지 확인
2. **KISS**: 단순함을 추구하되, 필요한 복잡성은 수용
3. **YAGNI**: 현재 필요한 것만 구현하되, 확장 가능한 구조 유지
4. **패턴 적용**: 문제를 먼저 이해하고, 적절한 패턴 선택
5. **균형**: 과도한 설계와 부족한 설계 사이의 균형

다음 장에서는 C#과 .NET의 언어 기능을 활용한 패턴 구현에 대해 알아보겠습니다.