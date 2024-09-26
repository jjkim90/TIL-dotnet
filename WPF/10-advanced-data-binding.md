# 고급 데이터 바인딩

## MultiBinding

MultiBinding은 여러 소스의 데이터를 하나의 타겟 속성에 바인딩합니다.

### 기본 MultiBinding
```xml
<TextBlock>
    <TextBlock.Text>
        <MultiBinding StringFormat="{}{0} {1}">
            <Binding Path="FirstName"/>
            <Binding Path="LastName"/>
        </MultiBinding>
    </TextBlock.Text>
</TextBlock>

<!-- 더 복잡한 형식 -->
<TextBlock>
    <TextBlock.Text>
        <MultiBinding StringFormat="Name: {0} {1}, Age: {2}">
            <Binding Path="FirstName"/>
            <Binding Path="LastName"/>
            <Binding Path="Age"/>
        </MultiBinding>
    </TextBlock.Text>
</TextBlock>
```

### MultiBinding Converter
```csharp
public class FullNameConverter : IMultiValueConverter
{
    public object Convert(object[] values, Type targetType, 
                         object parameter, CultureInfo culture)
    {
        if (values.Length >= 2 && 
            values[0] is string firstName && 
            values[1] is string lastName)
        {
            var format = parameter as string ?? "{0} {1}";
            return string.Format(format, firstName, lastName);
        }
        
        return string.Empty;
    }
    
    public object[] ConvertBack(object value, Type[] targetTypes, 
                               object parameter, CultureInfo culture)
    {
        if (value is string fullName)
        {
            var parts = fullName.Split(' ');
            if (parts.Length >= 2)
            {
                return new object[] { parts[0], parts[1] };
            }
        }
        
        return new object[] { null, null };
    }
}
```

### MultiBinding 사용 예
```xml
<Window.Resources>
    <local:FullNameConverter x:Key="FullNameConverter"/>
    <local:RgbToColorConverter x:Key="RgbToColorConverter"/>
</Window.Resources>

<!-- 이름 결합 -->
<TextBox>
    <TextBox.Text>
        <MultiBinding Converter="{StaticResource FullNameConverter}"
                      ConverterParameter="{1}, {0}">
            <Binding Path="FirstName" UpdateSourceTrigger="PropertyChanged"/>
            <Binding Path="LastName" UpdateSourceTrigger="PropertyChanged"/>
        </MultiBinding>
    </TextBox.Text>
</TextBox>

<!-- RGB 색상 생성 -->
<Rectangle Width="100" Height="100">
    <Rectangle.Fill>
        <MultiBinding Converter="{StaticResource RgbToColorConverter}">
            <Binding ElementName="redSlider" Path="Value"/>
            <Binding ElementName="greenSlider" Path="Value"/>
            <Binding ElementName="blueSlider" Path="Value"/>
        </MultiBinding>
    </Rectangle.Fill>
</Rectangle>
```

## PriorityBinding

PriorityBinding은 여러 바인딩을 우선순위에 따라 시도합니다.

### 기본 PriorityBinding
```xml
<TextBlock>
    <TextBlock.Text>
        <PriorityBinding>
            <!-- 느린 데이터 소스 (높은 우선순위) -->
            <Binding Path="SlowLoadingData" IsAsync="True"/>
            <!-- 빠른 대체 데이터 -->
            <Binding Path="CachedData"/>
            <!-- 기본값 -->
            <Binding Source="Loading..."/>
        </PriorityBinding>
    </TextBlock.Text>
</TextBlock>
```

### 실전 PriorityBinding 예제
```csharp
public class DataViewModel : ViewModelBase
{
    private string _cachedData = "Cached Value";
    private string _slowLoadingData;
    
    public string CachedData
    {
        get => _cachedData;
        set => SetProperty(ref _cachedData, value);
    }
    
    public string SlowLoadingData
    {
        get
        {
            if (_slowLoadingData == null)
            {
                Task.Run(async () =>
                {
                    await Task.Delay(3000); // 3초 지연
                    SlowLoadingData = "Fresh Data from Server";
                });
            }
            return _slowLoadingData;
        }
        set => SetProperty(ref _slowLoadingData, value);
    }
}
```

## Value Converters

### 기본 Converter 구현
```csharp
public class BooleanToVisibilityConverter : IValueConverter
{
    public object Convert(object value, Type targetType, 
                         object parameter, CultureInfo culture)
    {
        if (value is bool boolValue)
        {
            var invert = parameter as string == "Invert";
            return (boolValue ^ invert) ? Visibility.Visible : Visibility.Collapsed;
        }
        
        return Visibility.Collapsed;
    }
    
    public object ConvertBack(object value, Type targetType, 
                            object parameter, CultureInfo culture)
    {
        if (value is Visibility visibility)
        {
            return visibility == Visibility.Visible;
        }
        
        return false;
    }
}
```

### 고급 Converter 예제
```csharp
// 열거형을 설명 문자열로 변환
public class EnumToDescriptionConverter : IValueConverter
{
    public object Convert(object value, Type targetType, 
                         object parameter, CultureInfo culture)
    {
        if (value is Enum enumValue)
        {
            var field = enumValue.GetType().GetField(enumValue.ToString());
            var attribute = field?.GetCustomAttribute<DescriptionAttribute>();
            return attribute?.Description ?? enumValue.ToString();
        }
        
        return value?.ToString() ?? string.Empty;
    }
    
    public object ConvertBack(object value, Type targetType, 
                            object parameter, CultureInfo culture)
    {
        throw new NotSupportedException();
    }
}

// 숫자 범위 변환
public class RangeConverter : IValueConverter
{
    public double Minimum { get; set; }
    public double Maximum { get; set; }
    
    public object Convert(object value, Type targetType, 
                         object parameter, CultureInfo culture)
    {
        if (value is double doubleValue)
        {
            return Math.Max(Minimum, Math.Min(Maximum, doubleValue));
        }
        
        return value;
    }
    
    public object ConvertBack(object value, Type targetType, 
                            object parameter, CultureInfo culture)
    {
        return Convert(value, targetType, parameter, culture);
    }
}
```

### Converter 체이닝
```csharp
public class ConverterChain : List<IValueConverter>, IValueConverter
{
    public object Convert(object value, Type targetType, 
                         object parameter, CultureInfo culture)
    {
        return this.Aggregate(value, (current, converter) => 
            converter.Convert(current, targetType, parameter, culture));
    }
    
    public object ConvertBack(object value, Type[] targetTypes, 
                            object parameter, CultureInfo culture)
    {
        return this.Reverse<IValueConverter>().Aggregate(value, (current, converter) => 
            converter.ConvertBack(current, targetTypes, parameter, culture));
    }
}
```

## CollectionViewSource

CollectionViewSource는 컬렉션에 대한 정렬, 필터링, 그룹화를 제공합니다.

### 정렬 (Sorting)
```xml
<Window.Resources>
    <CollectionViewSource x:Key="SortedPeople" 
                          Source="{Binding People}">
        <CollectionViewSource.SortDescriptions>
            <scm:SortDescription PropertyName="LastName" Direction="Ascending"/>
            <scm:SortDescription PropertyName="FirstName" Direction="Ascending"/>
        </CollectionViewSource.SortDescriptions>
    </CollectionViewSource>
</Window.Resources>

<ListBox ItemsSource="{Binding Source={StaticResource SortedPeople}}"/>
```

### 필터링 (Filtering)
```csharp
public partial class MainWindow : Window
{
    private CollectionViewSource _peopleViewSource;
    private string _filterText;
    
    public MainWindow()
    {
        InitializeComponent();
        
        _peopleViewSource = FindResource("PeopleViewSource") as CollectionViewSource;
        _peopleViewSource.Filter += OnPeopleFilter;
    }
    
    public string FilterText
    {
        get => _filterText;
        set
        {
            _filterText = value;
            _peopleViewSource?.View?.Refresh();
        }
    }
    
    private void OnPeopleFilter(object sender, FilterEventArgs e)
    {
        if (e.Item is Person person)
        {
            if (string.IsNullOrWhiteSpace(FilterText))
            {
                e.Accepted = true;
            }
            else
            {
                e.Accepted = person.FullName.Contains(FilterText, 
                    StringComparison.OrdinalIgnoreCase);
            }
        }
    }
}
```

### 그룹화 (Grouping)
```xml
<Window.Resources>
    <CollectionViewSource x:Key="GroupedPeople" Source="{Binding People}">
        <CollectionViewSource.GroupDescriptions>
            <PropertyGroupDescription PropertyName="Department"/>
        </CollectionViewSource.GroupDescriptions>
        <CollectionViewSource.SortDescriptions>
            <scm:SortDescription PropertyName="Department"/>
            <scm:SortDescription PropertyName="LastName"/>
        </CollectionViewSource.SortDescriptions>
    </CollectionViewSource>
</Window.Resources>

<ListBox ItemsSource="{Binding Source={StaticResource GroupedPeople}}">
    <ListBox.GroupStyle>
        <GroupStyle>
            <GroupStyle.HeaderTemplate>
                <DataTemplate>
                    <Border Background="LightGray" Padding="5">
                        <TextBlock Text="{Binding Name}" 
                                   FontWeight="Bold" 
                                   FontSize="14"/>
                    </Border>
                </DataTemplate>
            </GroupStyle.HeaderTemplate>
        </GroupStyle>
    </ListBox.GroupStyle>
</ListBox>
```

## DataTemplateSelector

DataTemplateSelector는 데이터 타입이나 조건에 따라 템플릿을 동적으로 선택합니다.

### DataTemplateSelector 구현
```csharp
public class PersonDataTemplateSelector : DataTemplateSelector
{
    public DataTemplate NormalTemplate { get; set; }
    public DataTemplate ManagerTemplate { get; set; }
    public DataTemplate VipTemplate { get; set; }
    
    public override DataTemplate SelectTemplate(object item, DependencyObject container)
    {
        if (item is Person person)
        {
            if (person.IsVip)
                return VipTemplate;
            else if (person.IsManager)
                return ManagerTemplate;
            else
                return NormalTemplate;
        }
        
        return base.SelectTemplate(item, container);
    }
}
```

### XAML에서 사용
```xml
<Window.Resources>
    <!-- 일반 직원 템플릿 -->
    <DataTemplate x:Key="NormalPersonTemplate">
        <Border Background="White" BorderBrush="Gray" BorderThickness="1" 
                Padding="5" Margin="2">
            <TextBlock Text="{Binding FullName}"/>
        </Border>
    </DataTemplate>
    
    <!-- 관리자 템플릿 -->
    <DataTemplate x:Key="ManagerTemplate">
        <Border Background="LightBlue" BorderBrush="Blue" BorderThickness="2" 
                Padding="5" Margin="2">
            <StackPanel>
                <TextBlock Text="{Binding FullName}" FontWeight="Bold"/>
                <TextBlock Text="Manager" FontSize="10" Foreground="Blue"/>
            </StackPanel>
        </Border>
    </DataTemplate>
    
    <!-- VIP 템플릿 -->
    <DataTemplate x:Key="VipTemplate">
        <Border Background="Gold" BorderBrush="DarkGoldenrod" BorderThickness="2" 
                Padding="5" Margin="2">
            <StackPanel>
                <TextBlock Text="{Binding FullName}" FontWeight="Bold"/>
                <TextBlock Text="⭐ VIP" FontSize="10"/>
            </StackPanel>
        </Border>
    </DataTemplate>
    
    <!-- DataTemplateSelector -->
    <local:PersonDataTemplateSelector x:Key="PersonTemplateSelector"
                                      NormalTemplate="{StaticResource NormalPersonTemplate}"
                                      ManagerTemplate="{StaticResource ManagerTemplate}"
                                      VipTemplate="{StaticResource VipTemplate}"/>
</Window.Resources>

<ListBox ItemsSource="{Binding People}"
         ItemTemplateSelector="{StaticResource PersonTemplateSelector}"/>
```

## Binding to Methods

### ObjectDataProvider를 사용한 메서드 바인딩
```xml
<Window.Resources>
    <ObjectDataProvider x:Key="FormattedDate" 
                        ObjectType="{x:Type sys:DateTime}"
                        MethodName="Now">
        <ObjectDataProvider.MethodParameters>
            <!-- 파라미터가 필요한 경우 -->
        </ObjectDataProvider.MethodParameters>
    </ObjectDataProvider>
</Window.Resources>

<TextBlock Text="{Binding Source={StaticResource FormattedDate}, 
                          StringFormat='Current time: {0:HH:mm:ss}'}"/>
```

### 복잡한 메서드 바인딩
```csharp
public class MathHelper
{
    public double Calculate(double a, double b, string operation)
    {
        return operation switch
        {
            "Add" => a + b,
            "Subtract" => a - b,
            "Multiply" => a * b,
            "Divide" => b != 0 ? a / b : 0,
            _ => 0
        };
    }
}
```

```xml
<Window.Resources>
    <local:MathHelper x:Key="MathHelper"/>
    <ObjectDataProvider x:Key="CalculationResult" 
                        ObjectInstance="{StaticResource MathHelper}"
                        MethodName="Calculate">
        <ObjectDataProvider.MethodParameters>
            <sys:Double>10</sys:Double>
            <sys:Double>5</sys:Double>
            <sys:String>Add</sys:String>
        </ObjectDataProvider.MethodParameters>
    </ObjectDataProvider>
</Window.Resources>

<TextBlock Text="{Binding Source={StaticResource CalculationResult}}"/>
```

## Binding Validation

### ValidationRule 구현
```csharp
public class EmailValidationRule : ValidationRule
{
    public override ValidationResult Validate(object value, CultureInfo cultureInfo)
    {
        if (value is string email)
        {
            if (string.IsNullOrWhiteSpace(email))
                return new ValidationResult(false, "Email is required");
            
            var regex = new Regex(@"^[^@\s]+@[^@\s]+\.[^@\s]+$");
            if (!regex.IsMatch(email))
                return new ValidationResult(false, "Invalid email format");
            
            return ValidationResult.ValidResult;
        }
        
        return new ValidationResult(false, "Invalid input");
    }
}

public class RangeValidationRule : ValidationRule
{
    public double Minimum { get; set; }
    public double Maximum { get; set; }
    
    public override ValidationResult Validate(object value, CultureInfo cultureInfo)
    {
        if (double.TryParse(value?.ToString(), out double number))
        {
            if (number < Minimum || number > Maximum)
            {
                return new ValidationResult(false, 
                    $"Value must be between {Minimum} and {Maximum}");
            }
            
            return ValidationResult.ValidResult;
        }
        
        return new ValidationResult(false, "Invalid number");
    }
}
```

### XAML에서 ValidationRule 사용
```xml
<TextBox>
    <TextBox.Text>
        <Binding Path="Email" UpdateSourceTrigger="PropertyChanged">
            <Binding.ValidationRules>
                <local:EmailValidationRule/>
            </Binding.ValidationRules>
        </Binding>
    </TextBox.Text>
</TextBox>

<!-- 사용자 정의 오류 템플릿 -->
<Style TargetType="TextBox">
    <Setter Property="Validation.ErrorTemplate">
        <Setter.Value>
            <ControlTemplate>
                <Grid>
                    <Grid.RowDefinitions>
                        <RowDefinition/>
                        <RowDefinition Height="Auto"/>
                    </Grid.RowDefinitions>
                    
                    <Border Grid.Row="0" BorderBrush="Red" BorderThickness="1">
                        <AdornedElementPlaceholder/>
                    </Border>
                    
                    <TextBlock Grid.Row="1" 
                               Text="{Binding [0].ErrorContent}" 
                               Foreground="Red" 
                               FontSize="11"
                               Margin="2,2,0,0"/>
                </Grid>
            </ControlTemplate>
        </Setter.Value>
    </Setter>
</Style>
```

## 핵심 개념 정리
- **MultiBinding**: 여러 소스를 하나의 타겟에 바인딩
- **PriorityBinding**: 우선순위에 따른 바인딩 시도
- **Value Converters**: 데이터 타입 변환
- **CollectionViewSource**: 컬렉션 정렬, 필터링, 그룹화
- **DataTemplateSelector**: 조건에 따른 템플릿 선택
- **Method Binding**: ObjectDataProvider를 통한 메서드 바인딩
- **Validation**: ValidationRule을 통한 입력 검증