# 폼과 검증 심화

## 개요

Blazor의 폼 처리는 강력한 데이터 바인딩과 검증 기능을 제공합니다. 이 장에서는 EditForm의 고급 사용법, DataAnnotations와 FluentValidation 통합, 커스텀 검증 로직, 그리고 복잡한 폼 시나리오 처리 방법을 심화 학습합니다.

## 1. EditForm 고급 기능

### 1.1 EditContext 심화

```csharp
// EditContextExample.razor
@page "/forms/edit-context"
@implements IDisposable

<EditForm EditContext="editContext" OnSubmit="HandleSubmit">
    <div class="form-info">
        <p>Modified: @editContext.IsModified()</p>
        <p>Valid: @isValid</p>
        <p>Fields Modified: @string.Join(", ", modifiedFields)</p>
    </div>
    
    <div class="form-group">
        <label>Name:</label>
        <InputText @bind-Value="model.Name" class="form-control" />
        <ValidationMessage For="() => model.Name" />
    </div>
    
    <div class="form-group">
        <label>Email:</label>
        <InputText @bind-Value="model.Email" class="form-control" />
        <ValidationMessage For="() => model.Email" />
    </div>
    
    <div class="form-actions">
        <button type="submit" disabled="@(!isValid)">Submit</button>
        <button type="button" @onclick="ResetForm">Reset</button>
        <button type="button" @onclick="MarkAsUnmodified">Mark Unmodified</button>
    </div>
</EditForm>

@code {
    private EditContext editContext = default!;
    private UserModel model = new();
    private ValidationMessageStore messageStore = default!;
    private bool isValid;
    private List<string> modifiedFields = new();
    
    protected override void OnInitialized()
    {
        editContext = new EditContext(model);
        messageStore = new ValidationMessageStore(editContext);
        
        // 이벤트 구독
        editContext.OnValidationRequested += HandleValidationRequested;
        editContext.OnFieldChanged += HandleFieldChanged;
        editContext.OnValidationStateChanged += HandleValidationStateChanged;
    }
    
    private void HandleValidationRequested(object? sender, ValidationRequestedEventArgs e)
    {
        messageStore.Clear();
        
        // 커스텀 검증 로직
        if (string.IsNullOrWhiteSpace(model.Name))
        {
            messageStore.Add(() => model.Name, "Name is required");
        }
        
        if (!IsValidEmail(model.Email))
        {
            messageStore.Add(() => model.Email, "Invalid email format");
        }
        
        editContext.NotifyValidationStateChanged();
    }
    
    private void HandleFieldChanged(object? sender, FieldChangedEventArgs e)
    {
        // 필드 변경 추적
        var fieldName = e.FieldIdentifier.FieldName;
        if (!modifiedFields.Contains(fieldName))
        {
            modifiedFields.Add(fieldName);
        }
        
        // 실시간 검증
        ValidateField(e.FieldIdentifier);
    }
    
    private void HandleValidationStateChanged(object? sender, ValidationStateChangedEventArgs e)
    {
        isValid = !editContext.GetValidationMessages().Any();
        InvokeAsync(StateHasChanged);
    }
    
    private void ValidateField(FieldIdentifier fieldIdentifier)
    {
        messageStore.Clear(fieldIdentifier);
        
        switch (fieldIdentifier.FieldName)
        {
            case nameof(UserModel.Name):
                if (string.IsNullOrWhiteSpace(model.Name))
                {
                    messageStore.Add(fieldIdentifier, "Name is required");
                }
                else if (model.Name.Length < 3)
                {
                    messageStore.Add(fieldIdentifier, "Name must be at least 3 characters");
                }
                break;
                
            case nameof(UserModel.Email):
                if (!IsValidEmail(model.Email))
                {
                    messageStore.Add(fieldIdentifier, "Invalid email format");
                }
                break;
        }
        
        editContext.NotifyValidationStateChanged();
    }
    
    private void HandleSubmit()
    {
        if (editContext.Validate())
        {
            Console.WriteLine("Form submitted successfully");
            // 처리 로직
        }
    }
    
    private void ResetForm()
    {
        model = new UserModel();
        editContext = new EditContext(model);
        messageStore = new ValidationMessageStore(editContext);
        modifiedFields.Clear();
        StateHasChanged();
    }
    
    private void MarkAsUnmodified()
    {
        editContext.MarkAsUnmodified();
        modifiedFields.Clear();
    }
    
    private bool IsValidEmail(string email)
    {
        if (string.IsNullOrWhiteSpace(email)) return false;
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
    
    public void Dispose()
    {
        editContext.OnValidationRequested -= HandleValidationRequested;
        editContext.OnFieldChanged -= HandleFieldChanged;
        editContext.OnValidationStateChanged -= HandleValidationStateChanged;
    }
    
    public class UserModel
    {
        public string Name { get; set; } = "";
        public string Email { get; set; } = "";
    }
}
```

### 1.2 동적 폼 생성

```csharp
// DynamicFormBuilder.razor
@page "/forms/dynamic"

<h3>Dynamic Form Builder</h3>

<EditForm Model="dynamicModel" OnValidSubmit="HandleValidSubmit">
    <DataAnnotationsValidator />
    
    @foreach (var field in formFields)
    {
        <div class="form-group">
            @RenderField(field)
        </div>
    }
    
    <button type="submit" class="btn btn-primary">Submit</button>
</EditForm>

<div class="form-config">
    <h4>Add Field:</h4>
    <button @onclick="() => AddField(FieldType.Text)">Add Text</button>
    <button @onclick="() => AddField(FieldType.Number)">Add Number</button>
    <button @onclick="() => AddField(FieldType.Date)">Add Date</button>
    <button @onclick="() => AddField(FieldType.Select)">Add Select</button>
</div>

@code {
    private DynamicModel dynamicModel = new();
    private List<FormField> formFields = new();
    
    protected override void OnInitialized()
    {
        // 초기 필드 설정
        formFields.Add(new FormField
        {
            Name = "firstName",
            Label = "First Name",
            Type = FieldType.Text,
            Required = true,
            ValidationRules = new() { new RequiredRule(), new MinLengthRule(2) }
        });
    }
    
    private RenderFragment RenderField(FormField field) => builder =>
    {
        builder.OpenElement(0, "label");
        builder.AddContent(1, field.Label);
        if (field.Required)
        {
            builder.OpenElement(2, "span");
            builder.AddAttribute(3, "class", "required");
            builder.AddContent(4, " *");
            builder.CloseElement();
        }
        builder.CloseElement();
        
        switch (field.Type)
        {
            case FieldType.Text:
                RenderTextInput(builder, field);
                break;
            case FieldType.Number:
                RenderNumberInput(builder, field);
                break;
            case FieldType.Date:
                RenderDateInput(builder, field);
                break;
            case FieldType.Select:
                RenderSelectInput(builder, field);
                break;
        }
        
        // 검증 메시지
        builder.OpenComponent<ValidationMessage<string>>(10);
        builder.AddAttribute(11, "For", 
            Expression.Lambda<Func<string>>(
                Expression.Property(
                    Expression.Constant(dynamicModel),
                    field.Name
                )
            )
        );
        builder.CloseComponent();
    };
    
    private void RenderTextInput(RenderTreeBuilder builder, FormField field)
    {
        builder.OpenComponent<InputText>(5);
        builder.AddAttribute(6, "class", "form-control");
        builder.AddAttribute(7, "Value", dynamicModel.GetValue(field.Name));
        builder.AddAttribute(8, "ValueChanged", 
            EventCallback.Factory.Create<string>(this, value =>
            {
                dynamicModel.SetValue(field.Name, value);
                ValidateField(field, value);
            })
        );
        builder.AddAttribute(9, "ValueExpression", 
            Expression.Lambda<Func<string>>(
                Expression.Property(
                    Expression.Constant(dynamicModel),
                    field.Name
                )
            )
        );
        builder.CloseComponent();
    }
    
    private void ValidateField(FormField field, object? value)
    {
        var errors = new List<string>();
        
        foreach (var rule in field.ValidationRules)
        {
            if (!rule.Validate(value, out var error))
            {
                errors.Add(error);
            }
        }
        
        dynamicModel.SetErrors(field.Name, errors);
    }
    
    private void AddField(FieldType type)
    {
        var fieldCount = formFields.Count(f => f.Type == type) + 1;
        var field = new FormField
        {
            Name = $"{type.ToString().ToLower()}{fieldCount}",
            Label = $"{type} Field {fieldCount}",
            Type = type,
            Required = false,
            ValidationRules = new()
        };
        
        if (type == FieldType.Select)
        {
            field.Options = new List<SelectOption>
            {
                new() { Value = "1", Text = "Option 1" },
                new() { Value = "2", Text = "Option 2" },
                new() { Value = "3", Text = "Option 3" }
            };
        }
        
        formFields.Add(field);
        dynamicModel.AddField(field.Name);
    }
    
    private void HandleValidSubmit()
    {
        var values = dynamicModel.GetAllValues();
        Console.WriteLine("Form submitted with values:");
        foreach (var kvp in values)
        {
            Console.WriteLine($"{kvp.Key}: {kvp.Value}");
        }
    }
    
    public enum FieldType
    {
        Text, Number, Date, Select, Checkbox, Radio
    }
    
    public class FormField
    {
        public string Name { get; set; } = "";
        public string Label { get; set; } = "";
        public FieldType Type { get; set; }
        public bool Required { get; set; }
        public List<ValidationRule> ValidationRules { get; set; } = new();
        public List<SelectOption>? Options { get; set; }
    }
    
    public class SelectOption
    {
        public string Value { get; set; } = "";
        public string Text { get; set; } = "";
    }
    
    public abstract class ValidationRule
    {
        public abstract bool Validate(object? value, out string error);
    }
    
    public class RequiredRule : ValidationRule
    {
        public override bool Validate(object? value, out string error)
        {
            error = "This field is required";
            return value != null && !string.IsNullOrWhiteSpace(value.ToString());
        }
    }
    
    public class MinLengthRule : ValidationRule
    {
        private readonly int _minLength;
        
        public MinLengthRule(int minLength)
        {
            _minLength = minLength;
        }
        
        public override bool Validate(object? value, out string error)
        {
            error = $"Minimum length is {_minLength}";
            var str = value?.ToString() ?? "";
            return str.Length >= _minLength;
        }
    }
    
    public class DynamicModel
    {
        private readonly Dictionary<string, object?> _values = new();
        private readonly Dictionary<string, List<string>> _errors = new();
        
        public void AddField(string name)
        {
            _values[name] = null;
            _errors[name] = new List<string>();
        }
        
        public object? GetValue(string name)
        {
            return _values.TryGetValue(name, out var value) ? value : null;
        }
        
        public void SetValue(string name, object? value)
        {
            _values[name] = value;
        }
        
        public void SetErrors(string name, List<string> errors)
        {
            _errors[name] = errors;
        }
        
        public Dictionary<string, object?> GetAllValues()
        {
            return new Dictionary<string, object?>(_values);
        }
    }
}
```

## 2. DataAnnotations 고급 활용

### 2.1 커스텀 검증 어트리뷰트

```csharp
// CustomValidationAttributes.cs
public class EmailDomainAttribute : ValidationAttribute
{
    private readonly string[] _allowedDomains;
    
    public EmailDomainAttribute(params string[] allowedDomains)
    {
        _allowedDomains = allowedDomains;
        ErrorMessage = $"Email must be from one of these domains: {string.Join(", ", allowedDomains)}";
    }
    
    protected override ValidationResult? IsValid(object? value, ValidationContext validationContext)
    {
        if (value is string email && !string.IsNullOrWhiteSpace(email))
        {
            var domain = email.Split('@').LastOrDefault();
            if (domain != null && _allowedDomains.Contains(domain))
            {
                return ValidationResult.Success;
            }
        }
        
        return new ValidationResult(ErrorMessage, new[] { validationContext.MemberName ?? "" });
    }
}

public class ConditionalRequiredAttribute : ValidationAttribute
{
    private readonly string _dependentProperty;
    private readonly object _targetValue;
    
    public ConditionalRequiredAttribute(string dependentProperty, object targetValue)
    {
        _dependentProperty = dependentProperty;
        _targetValue = targetValue;
    }
    
    protected override ValidationResult? IsValid(object? value, ValidationContext validationContext)
    {
        var instance = validationContext.ObjectInstance;
        var type = instance.GetType();
        var dependentValue = type.GetProperty(_dependentProperty)?.GetValue(instance);
        
        if (Equals(dependentValue, _targetValue))
        {
            if (value == null || string.IsNullOrWhiteSpace(value.ToString()))
            {
                return new ValidationResult(
                    $"This field is required when {_dependentProperty} is {_targetValue}",
                    new[] { validationContext.MemberName ?? "" }
                );
            }
        }
        
        return ValidationResult.Success;
    }
}

public class DateRangeAttribute : ValidationAttribute
{
    private readonly string _startDateProperty;
    
    public DateRangeAttribute(string startDateProperty)
    {
        _startDateProperty = startDateProperty;
    }
    
    protected override ValidationResult? IsValid(object? value, ValidationContext validationContext)
    {
        if (value is DateTime endDate)
        {
            var instance = validationContext.ObjectInstance;
            var startDateValue = instance.GetType()
                .GetProperty(_startDateProperty)?
                .GetValue(instance);
            
            if (startDateValue is DateTime startDate && endDate < startDate)
            {
                return new ValidationResult(
                    "End date must be after start date",
                    new[] { validationContext.MemberName ?? "" }
                );
            }
        }
        
        return ValidationResult.Success;
    }
}

// 사용 예제
public class AdvancedFormModel
{
    [Required]
    [StringLength(50, MinimumLength = 3)]
    public string Name { get; set; } = "";
    
    [Required]
    [EmailAddress]
    [EmailDomain("company.com", "partner.com")]
    public string Email { get; set; } = "";
    
    public bool IsEmployee { get; set; }
    
    [ConditionalRequired(nameof(IsEmployee), true)]
    public string? EmployeeId { get; set; }
    
    [Required]
    public DateTime StartDate { get; set; } = DateTime.Today;
    
    [Required]
    [DateRange(nameof(StartDate))]
    public DateTime EndDate { get; set; } = DateTime.Today.AddDays(1);
}
```

### 2.2 IValidatableObject 구현

```csharp
// ComplexValidationModel.cs
public class OrderFormModel : IValidatableObject
{
    [Required]
    public string CustomerName { get; set; } = "";
    
    [Required]
    [EmailAddress]
    public string CustomerEmail { get; set; } = "";
    
    [Required]
    [Phone]
    public string CustomerPhone { get; set; } = "";
    
    public List<OrderItem> Items { get; set; } = new();
    
    [Required]
    [CreditCard]
    public string? CreditCardNumber { get; set; }
    
    public string? PromoCode { get; set; }
    
    public decimal Discount { get; set; }
    
    public IEnumerable<ValidationResult> Validate(ValidationContext validationContext)
    {
        var results = new List<ValidationResult>();
        
        // 비즈니스 규칙 검증
        if (!Items.Any())
        {
            results.Add(new ValidationResult(
                "At least one item is required",
                new[] { nameof(Items) }
            ));
        }
        
        if (Items.Any(item => item.Quantity <= 0))
        {
            results.Add(new ValidationResult(
                "All items must have a positive quantity",
                new[] { nameof(Items) }
            ));
        }
        
        var totalAmount = Items.Sum(i => i.Price * i.Quantity);
        if (Discount > totalAmount * 0.5m)
        {
            results.Add(new ValidationResult(
                "Discount cannot exceed 50% of total amount",
                new[] { nameof(Discount) }
            ));
        }
        
        // 프로모 코드 검증
        if (!string.IsNullOrWhiteSpace(PromoCode))
        {
            var validPromoCodes = new[] { "SAVE10", "SAVE20", "VIP30" };
            if (!validPromoCodes.Contains(PromoCode.ToUpper()))
            {
                results.Add(new ValidationResult(
                    "Invalid promo code",
                    new[] { nameof(PromoCode) }
                ));
            }
        }
        
        // 의존성 주입을 통한 외부 서비스 검증
        if (validationContext.GetService(typeof(IInventoryService)) is IInventoryService inventoryService)
        {
            foreach (var item in Items)
            {
                if (!inventoryService.IsInStock(item.ProductId, item.Quantity))
                {
                    results.Add(new ValidationResult(
                        $"Product {item.ProductName} is out of stock",
                        new[] { nameof(Items) }
                    ));
                }
            }
        }
        
        return results;
    }
}

public class OrderItem
{
    public string ProductId { get; set; } = "";
    public string ProductName { get; set; } = "";
    public decimal Price { get; set; }
    public int Quantity { get; set; }
}
```

## 3. FluentValidation 통합

### 3.1 FluentValidation 설정

```csharp
// Program.cs
builder.Services.AddScoped<IValidator<UserRegistrationModel>, UserRegistrationValidator>();
builder.Services.AddScoped<IValidator<ProductModel>, ProductValidator>();

// UserRegistrationValidator.cs
public class UserRegistrationValidator : AbstractValidator<UserRegistrationModel>
{
    private readonly IUserService _userService;
    
    public UserRegistrationValidator(IUserService userService)
    {
        _userService = userService;
        
        RuleFor(x => x.Username)
            .NotEmpty().WithMessage("Username is required")
            .Length(3, 20).WithMessage("Username must be between 3 and 20 characters")
            .MustAsync(BeUniqueUsername).WithMessage("Username already exists");
        
        RuleFor(x => x.Email)
            .NotEmpty().WithMessage("Email is required")
            .EmailAddress().WithMessage("Invalid email format")
            .MustAsync(BeUniqueEmail).WithMessage("Email already registered");
        
        RuleFor(x => x.Password)
            .NotEmpty().WithMessage("Password is required")
            .MinimumLength(8).WithMessage("Password must be at least 8 characters")
            .Matches(@"[A-Z]").WithMessage("Password must contain uppercase letter")
            .Matches(@"[a-z]").WithMessage("Password must contain lowercase letter")
            .Matches(@"[0-9]").WithMessage("Password must contain number")
            .Matches(@"[^a-zA-Z0-9]").WithMessage("Password must contain special character");
        
        RuleFor(x => x.ConfirmPassword)
            .Equal(x => x.Password).WithMessage("Passwords do not match");
        
        RuleFor(x => x.DateOfBirth)
            .LessThan(DateTime.Today.AddYears(-18))
            .WithMessage("You must be at least 18 years old");
        
        RuleFor(x => x.PhoneNumber)
            .Matches(@"^\+?[1-9]\d{1,14}$")
            .When(x => !string.IsNullOrEmpty(x.PhoneNumber))
            .WithMessage("Invalid phone number format");
        
        RuleSet("Premium", () =>
        {
            RuleFor(x => x.CreditCard)
                .NotEmpty().WithMessage("Credit card is required for premium accounts")
                .CreditCard().WithMessage("Invalid credit card number");
        });
    }
    
    private async Task<bool> BeUniqueUsername(string username, CancellationToken cancellationToken)
    {
        return !await _userService.UsernameExistsAsync(username, cancellationToken);
    }
    
    private async Task<bool> BeUniqueEmail(string email, CancellationToken cancellationToken)
    {
        return !await _userService.EmailExistsAsync(email, cancellationToken);
    }
}

// FluentValidationValidator.razor
@typeparam TModel
@implements IDisposable

@if (CurrentEditContext != null)
{
    <FluentValidationMessages @ref="messages" />
}

@code {
    [CascadingParameter] private EditContext? CurrentEditContext { get; set; }
    [Parameter] public IValidator<TModel>? Validator { get; set; }
    [Parameter] public string? RuleSet { get; set; }
    
    private FluentValidationMessages? messages;
    private ValidationMessageStore? messageStore;
    
    protected override void OnInitialized()
    {
        if (CurrentEditContext == null)
        {
            throw new InvalidOperationException($"{nameof(FluentValidationValidator)} requires a cascading parameter of type {nameof(EditContext)}");
        }
        
        messageStore = new ValidationMessageStore(CurrentEditContext);
        CurrentEditContext.OnValidationRequested += async (s, e) => await ValidateModel();
        CurrentEditContext.OnFieldChanged += async (s, e) => await ValidateField(e.FieldIdentifier);
    }
    
    private async Task ValidateModel()
    {
        if (Validator == null || CurrentEditContext == null) return;
        
        messageStore?.Clear();
        
        var model = CurrentEditContext.Model;
        var context = new ValidationContext<TModel>((TModel)model);
        
        if (!string.IsNullOrEmpty(RuleSet))
        {
            context.IncludeRuleSets(RuleSet);
        }
        
        var result = await Validator.ValidateAsync(context);
        
        foreach (var error in result.Errors)
        {
            var fieldIdentifier = ToFieldIdentifier(CurrentEditContext, error.PropertyName);
            messageStore?.Add(fieldIdentifier, error.ErrorMessage);
        }
        
        CurrentEditContext.NotifyValidationStateChanged();
    }
    
    private async Task ValidateField(FieldIdentifier fieldIdentifier)
    {
        if (Validator == null || CurrentEditContext == null) return;
        
        var properties = new[] { fieldIdentifier.FieldName };
        var context = new ValidationContext<TModel>((TModel)CurrentEditContext.Model, 
            new PropertyChain(), 
            new MemberNameValidatorSelector(properties));
        
        var result = await Validator.ValidateAsync(context);
        
        messageStore?.Clear(fieldIdentifier);
        
        foreach (var error in result.Errors.Where(e => e.PropertyName == fieldIdentifier.FieldName))
        {
            messageStore?.Add(fieldIdentifier, error.ErrorMessage);
        }
        
        CurrentEditContext.NotifyValidationStateChanged();
    }
    
    private static FieldIdentifier ToFieldIdentifier(EditContext editContext, string propertyPath)
    {
        var obj = editContext.Model;
        
        while (true)
        {
            var nextTokenEnd = propertyPath.IndexOfAny(new[] { '.', '[' });
            if (nextTokenEnd < 0)
            {
                return new FieldIdentifier(obj, propertyPath);
            }
            
            var nextToken = propertyPath.Substring(0, nextTokenEnd);
            propertyPath = propertyPath.Substring(nextTokenEnd + 1);
            
            var prop = obj.GetType().GetProperty(nextToken);
            if (prop == null)
            {
                throw new InvalidOperationException($"Could not find property '{nextToken}'.");
            }
            
            obj = prop.GetValue(obj);
            if (obj == null)
            {
                return new FieldIdentifier(new object(), propertyPath);
            }
        }
    }
    
    public void Dispose()
    {
        if (CurrentEditContext != null)
        {
            CurrentEditContext.OnValidationRequested -= async (s, e) => await ValidateModel();
            CurrentEditContext.OnFieldChanged -= async (s, e) => await ValidateField(e.FieldIdentifier);
        }
    }
}
```

### 3.2 복잡한 FluentValidation 시나리오

```csharp
// ComplexValidator.cs
public class OrderValidator : AbstractValidator<Order>
{
    public OrderValidator(IDiscountService discountService)
    {
        // 기본 검증
        RuleFor(x => x.OrderDate)
            .GreaterThanOrEqualTo(DateTime.Today)
            .WithMessage("Order date cannot be in the past");
        
        // 컬렉션 검증
        RuleFor(x => x.Items)
            .NotEmpty().WithMessage("Order must contain at least one item")
            .Must(items => items.Sum(i => i.Quantity) <= 100)
            .WithMessage("Total quantity cannot exceed 100 items");
        
        RuleForEach(x => x.Items).SetValidator(new OrderItemValidator());
        
        // 조건부 검증
        When(x => x.IsRushOrder, () =>
        {
            RuleFor(x => x.RushOrderFee)
                .GreaterThan(0).WithMessage("Rush order fee is required");
        });
        
        // 복잡한 비즈니스 규칙
        RuleFor(x => x)
            .MustAsync(async (order, cancellation) =>
            {
                if (!string.IsNullOrEmpty(order.DiscountCode))
                {
                    var discount = await discountService.GetDiscountAsync(order.DiscountCode);
                    return discount != null && discount.IsValid;
                }
                return true;
            })
            .WithMessage("Invalid discount code")
            .WithName("DiscountCode");
        
        // 중첩 객체 검증
        RuleFor(x => x.ShippingAddress)
            .NotNull().WithMessage("Shipping address is required")
            .SetValidator(new AddressValidator());
        
        RuleFor(x => x.BillingAddress)
            .NotNull().WithMessage("Billing address is required")
            .SetValidator(new AddressValidator())
            .Unless(x => x.UseShippingAsBilling);
        
        // 커스텀 속성 검증
        RuleFor(x => x.CustomerNotes)
            .Must(BeValidNotes).WithMessage("Notes contain prohibited content")
            .When(x => !string.IsNullOrEmpty(x.CustomerNotes));
    }
    
    private bool BeValidNotes(string notes)
    {
        var prohibitedWords = new[] { "spam", "hack", "exploit" };
        return !prohibitedWords.Any(word => 
            notes.Contains(word, StringComparison.OrdinalIgnoreCase));
    }
}

public class OrderItemValidator : AbstractValidator<OrderItem>
{
    public OrderItemValidator()
    {
        RuleFor(x => x.ProductId)
            .NotEmpty().WithMessage("Product ID is required");
        
        RuleFor(x => x.Quantity)
            .InclusiveBetween(1, 10)
            .WithMessage("Quantity must be between 1 and 10");
        
        RuleFor(x => x.Price)
            .GreaterThan(0).WithMessage("Price must be greater than 0");
    }
}

public class AddressValidator : AbstractValidator<Address>
{
    public AddressValidator()
    {
        RuleFor(x => x.Street)
            .NotEmpty().WithMessage("Street is required")
            .MaximumLength(100);
        
        RuleFor(x => x.City)
            .NotEmpty().WithMessage("City is required")
            .Matches(@"^[a-zA-Z\s]+$").WithMessage("City must contain only letters");
        
        RuleFor(x => x.PostalCode)
            .NotEmpty().WithMessage("Postal code is required")
            .Matches(@"^\d{5}(-\d{4})?$").WithMessage("Invalid US postal code");
        
        RuleFor(x => x.Country)
            .NotEmpty().WithMessage("Country is required")
            .Must(BeValidCountry).WithMessage("Invalid country");
    }
    
    private bool BeValidCountry(string country)
    {
        var validCountries = new[] { "USA", "Canada", "Mexico" };
        return validCountries.Contains(country);
    }
}
```

## 4. 실시간 검증과 비동기 검증

### 4.1 실시간 검증 구현

```csharp
// RealtimeValidation.razor
@page "/forms/realtime"
@inject IValidationService ValidationService

<EditForm Model="model" OnValidSubmit="HandleSubmit">
    <div class="form-group">
        <label>Username:</label>
        <InputText @bind-Value="model.Username" 
                   @bind-Value:event="oninput"
                   @onblur="() => ValidateFieldAsync(nameof(model.Username))"
                   class="form-control @GetFieldClass(nameof(model.Username))" />
        <div class="validation-feedback">
            @if (IsValidating(nameof(model.Username)))
            {
                <span class="text-info">Checking availability...</span>
            }
            else if (HasError(nameof(model.Username)))
            {
                <span class="text-danger">@GetError(nameof(model.Username))</span>
            }
            else if (IsValid(nameof(model.Username)))
            {
                <span class="text-success">Username available!</span>
            }
        </div>
    </div>
    
    <div class="form-group">
        <label>Email:</label>
        <InputText @bind-Value="model.Email"
                   @bind-Value:event="oninput"
                   @onblur="() => ValidateFieldAsync(nameof(model.Email))"
                   class="form-control @GetFieldClass(nameof(model.Email))" />
        <ValidationMessage For="() => model.Email" />
    </div>
    
    <button type="submit" disabled="@(!IsFormValid || IsAnyFieldValidating)">
        Submit
    </button>
</EditForm>

@code {
    private RegistrationModel model = new();
    private Dictionary<string, ValidationState> fieldStates = new();
    private CancellationTokenSource? cts;
    
    private bool IsFormValid => fieldStates.Values.All(v => v.IsValid);
    private bool IsAnyFieldValidating => fieldStates.Values.Any(v => v.IsValidating);
    
    protected override void OnInitialized()
    {
        // 초기화
        fieldStates[nameof(model.Username)] = new ValidationState();
        fieldStates[nameof(model.Email)] = new ValidationState();
    }
    
    private async Task ValidateFieldAsync(string fieldName)
    {
        // 이전 검증 취소
        cts?.Cancel();
        cts = new CancellationTokenSource();
        
        var state = fieldStates[fieldName];
        state.IsValidating = true;
        state.Error = null;
        StateHasChanged();
        
        try
        {
            await Task.Delay(500, cts.Token); // 디바운싱
            
            switch (fieldName)
            {
                case nameof(model.Username):
                    await ValidateUsernameAsync(cts.Token);
                    break;
                case nameof(model.Email):
                    await ValidateEmailAsync(cts.Token);
                    break;
            }
        }
        catch (OperationCanceledException)
        {
            // 취소됨
        }
        finally
        {
            state.IsValidating = false;
            StateHasChanged();
        }
    }
    
    private async Task ValidateUsernameAsync(CancellationToken cancellationToken)
    {
        var state = fieldStates[nameof(model.Username)];
        
        if (string.IsNullOrWhiteSpace(model.Username))
        {
            state.Error = "Username is required";
            state.IsValid = false;
            return;
        }
        
        if (model.Username.Length < 3)
        {
            state.Error = "Username must be at least 3 characters";
            state.IsValid = false;
            return;
        }
        
        // 서버 검증
        var isAvailable = await ValidationService.IsUsernameAvailableAsync(
            model.Username, cancellationToken);
        
        if (!isAvailable)
        {
            state.Error = "Username is already taken";
            state.IsValid = false;
        }
        else
        {
            state.Error = null;
            state.IsValid = true;
        }
    }
    
    private async Task ValidateEmailAsync(CancellationToken cancellationToken)
    {
        var state = fieldStates[nameof(model.Email)];
        
        if (string.IsNullOrWhiteSpace(model.Email))
        {
            state.Error = "Email is required";
            state.IsValid = false;
            return;
        }
        
        if (!IsValidEmail(model.Email))
        {
            state.Error = "Invalid email format";
            state.IsValid = false;
            return;
        }
        
        // 서버 검증
        var exists = await ValidationService.EmailExistsAsync(
            model.Email, cancellationToken);
        
        if (exists)
        {
            state.Error = "Email is already registered";
            state.IsValid = false;
        }
        else
        {
            state.Error = null;
            state.IsValid = true;
        }
    }
    
    private bool IsValidEmail(string email)
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
    
    private bool IsValidating(string fieldName) =>
        fieldStates.TryGetValue(fieldName, out var state) && state.IsValidating;
    
    private bool HasError(string fieldName) =>
        fieldStates.TryGetValue(fieldName, out var state) && !string.IsNullOrEmpty(state.Error);
    
    private bool IsValid(string fieldName) =>
        fieldStates.TryGetValue(fieldName, out var state) && state.IsValid && !state.IsValidating;
    
    private string? GetError(string fieldName) =>
        fieldStates.TryGetValue(fieldName, out var state) ? state.Error : null;
    
    private string GetFieldClass(string fieldName)
    {
        if (!fieldStates.TryGetValue(fieldName, out var state))
            return "";
        
        if (state.IsValidating)
            return "validating";
        
        if (!string.IsNullOrEmpty(state.Error))
            return "is-invalid";
        
        if (state.IsValid)
            return "is-valid";
        
        return "";
    }
    
    private async Task HandleSubmit()
    {
        // 최종 검증
        foreach (var fieldName in fieldStates.Keys)
        {
            await ValidateFieldAsync(fieldName);
        }
        
        if (IsFormValid)
        {
            // 제출 처리
            Console.WriteLine("Form submitted successfully");
        }
    }
    
    public class ValidationState
    {
        public bool IsValidating { get; set; }
        public bool IsValid { get; set; }
        public string? Error { get; set; }
    }
    
    public class RegistrationModel
    {
        public string Username { get; set; } = "";
        public string Email { get; set; } = "";
    }
}
```

## 5. 복잡한 폼 시나리오

### 5.1 마법사 스타일 폼

```csharp
// WizardForm.razor
@page "/forms/wizard"

<div class="wizard-form">
    <div class="wizard-steps">
        @for (int i = 0; i < steps.Count; i++)
        {
            var index = i;
            <div class="step @GetStepClass(index)" @onclick="() => GoToStep(index)">
                <span class="step-number">@(index + 1)</span>
                <span class="step-title">@steps[index].Title</span>
            </div>
        }
    </div>
    
    <div class="wizard-content">
        <EditForm EditContext="editContext" OnValidSubmit="HandleValidSubmit">
            @steps[currentStep].Content
            
            <div class="wizard-actions">
                <button type="button" 
                        class="btn btn-secondary" 
                        disabled="@(currentStep == 0)"
                        @onclick="PreviousStep">
                    Previous
                </button>
                
                @if (currentStep < steps.Count - 1)
                {
                    <button type="button" 
                            class="btn btn-primary"
                            disabled="@(!IsCurrentStepValid())"
                            @onclick="NextStep">
                        Next
                    </button>
                }
                else
                {
                    <button type="submit" 
                            class="btn btn-success"
                            disabled="@(!IsAllStepsValid())">
                        Submit
                    </button>
                }
            </div>
        </EditForm>
    </div>
    
    <div class="wizard-summary">
        <h4>Summary:</h4>
        <pre>@JsonSerializer.Serialize(model, new JsonSerializerOptions { WriteIndented = true })</pre>
    </div>
</div>

@code {
    private WizardModel model = new();
    private EditContext editContext = default!;
    private int currentStep = 0;
    private List<WizardStep> steps = new();
    private Dictionary<int, bool> stepValidationStates = new();
    
    protected override void OnInitialized()
    {
        editContext = new EditContext(model);
        
        steps = new List<WizardStep>
        {
            new WizardStep
            {
                Title = "Personal Info",
                Content = BuildPersonalInfoStep(),
                ValidationProperties = new[] { nameof(model.FirstName), nameof(model.LastName), nameof(model.Email) }
            },
            new WizardStep
            {
                Title = "Address",
                Content = BuildAddressStep(),
                ValidationProperties = new[] { nameof(model.Street), nameof(model.City), nameof(model.PostalCode) }
            },
            new WizardStep
            {
                Title = "Preferences",
                Content = BuildPreferencesStep(),
                ValidationProperties = new[] { nameof(model.Newsletter), nameof(model.PreferredContact) }
            },
            new WizardStep
            {
                Title = "Review",
                Content = BuildReviewStep(),
                ValidationProperties = Array.Empty<string>()
            }
        };
        
        // 초기 검증 상태
        ValidateAllSteps();
    }
    
    private RenderFragment BuildPersonalInfoStep() => builder =>
    {
        builder.OpenElement(0, "div");
        builder.AddAttribute(1, "class", "step-content");
        
        builder.OpenElement(2, "h3");
        builder.AddContent(3, "Personal Information");
        builder.CloseElement();
        
        // First Name
        builder.OpenElement(4, "div");
        builder.AddAttribute(5, "class", "form-group");
        builder.OpenElement(6, "label");
        builder.AddContent(7, "First Name:");
        builder.CloseElement();
        builder.OpenComponent<InputText>(8);
        builder.AddAttribute(9, "class", "form-control");
        builder.AddAttribute(10, "Value", model.FirstName);
        builder.AddAttribute(11, "ValueChanged", EventCallback.Factory.Create<string>(this, value =>
        {
            model.FirstName = value;
            ValidateStep(0);
        }));
        builder.AddAttribute(12, "ValueExpression", () => model.FirstName);
        builder.CloseComponent();
        builder.OpenComponent<ValidationMessage<string>>(13);
        builder.AddAttribute(14, "For", () => model.FirstName);
        builder.CloseComponent();
        builder.CloseElement();
        
        // Similar for Last Name and Email...
        
        builder.CloseElement();
    };
    
    private void ValidateStep(int stepIndex)
    {
        var step = steps[stepIndex];
        var isValid = true;
        
        foreach (var property in step.ValidationProperties)
        {
            var fieldIdentifier = new FieldIdentifier(model, property);
            var validationMessages = editContext.GetValidationMessages(fieldIdentifier);
            if (validationMessages.Any())
            {
                isValid = false;
                break;
            }
        }
        
        stepValidationStates[stepIndex] = isValid;
        StateHasChanged();
    }
    
    private void ValidateAllSteps()
    {
        for (int i = 0; i < steps.Count; i++)
        {
            ValidateStep(i);
        }
    }
    
    private bool IsCurrentStepValid()
    {
        return stepValidationStates.TryGetValue(currentStep, out var isValid) && isValid;
    }
    
    private bool IsAllStepsValid()
    {
        return stepValidationStates.Values.All(v => v);
    }
    
    private string GetStepClass(int stepIndex)
    {
        if (stepIndex < currentStep) return "completed";
        if (stepIndex == currentStep) return "active";
        return "pending";
    }
    
    private void GoToStep(int stepIndex)
    {
        if (stepIndex <= currentStep || IsStepAccessible(stepIndex))
        {
            currentStep = stepIndex;
        }
    }
    
    private bool IsStepAccessible(int stepIndex)
    {
        // 이전 단계가 모두 유효한 경우에만 접근 가능
        for (int i = 0; i < stepIndex; i++)
        {
            if (!stepValidationStates.TryGetValue(i, out var isValid) || !isValid)
            {
                return false;
            }
        }
        return true;
    }
    
    private void PreviousStep()
    {
        if (currentStep > 0)
        {
            currentStep--;
        }
    }
    
    private void NextStep()
    {
        if (currentStep < steps.Count - 1 && IsCurrentStepValid())
        {
            currentStep++;
        }
    }
    
    private void HandleValidSubmit()
    {
        Console.WriteLine("Wizard completed!");
        // 제출 처리
    }
    
    public class WizardStep
    {
        public string Title { get; set; } = "";
        public RenderFragment Content { get; set; } = default!;
        public string[] ValidationProperties { get; set; } = Array.Empty<string>();
    }
    
    public class WizardModel
    {
        // Personal Info
        [Required]
        public string FirstName { get; set; } = "";
        
        [Required]
        public string LastName { get; set; } = "";
        
        [Required]
        [EmailAddress]
        public string Email { get; set; } = "";
        
        // Address
        [Required]
        public string Street { get; set; } = "";
        
        [Required]
        public string City { get; set; } = "";
        
        [Required]
        [RegularExpression(@"^\d{5}$", ErrorMessage = "Invalid postal code")]
        public string PostalCode { get; set; } = "";
        
        // Preferences
        public bool Newsletter { get; set; }
        public string PreferredContact { get; set; } = "email";
    }
}
```

## 마무리

Blazor의 폼과 검증 시스템은 매우 강력하고 유연합니다. EditForm과 EditContext를 통한 세밀한 제어, DataAnnotations와 FluentValidation을 통한 선언적 검증, 그리고 커스텀 검증 로직을 통해 복잡한 비즈니스 요구사항도 충족할 수 있습니다.

실시간 검증, 비동기 검증, 조건부 검증 등 다양한 시나리오를 지원하며, 마법사 폼과 같은 복잡한 UI 패턴도 구현할 수 있습니다. 중요한 것은 사용자 경험을 고려하여 적절한 검증 전략을 선택하고, 성능과 사용성의 균형을 맞추는 것입니다.