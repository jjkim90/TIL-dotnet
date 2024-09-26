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

## INotifyDataErrorInfo

INotifyDataErrorInfo는 비동기 검증과 여러 오류를 지원하는 고급 검증 인터페이스입니다.

### INotifyDataErrorInfo 구현
```csharp
public class ValidatingViewModel : ViewModelBase, INotifyDataErrorInfo
{
    private readonly Dictionary<string, List<string>> _errors = new();
    private string _email;
    private int _age;
    
    public string Email
    {
        get => _email;
        set
        {
            if (SetProperty(ref _email, value))
            {
                ValidateEmailAsync();
            }
        }
    }
    
    public int Age
    {
        get => _age;
        set
        {
            if (SetProperty(ref _age, value))
            {
                ValidateAge();
            }
        }
    }
    
    // INotifyDataErrorInfo 구현
    public bool HasErrors => _errors.Count > 0;
    
    public event EventHandler<DataErrorsChangedEventArgs> ErrorsChanged;
    
    public IEnumerable GetErrors(string propertyName)
    {
        if (string.IsNullOrEmpty(propertyName))
            return _errors.SelectMany(kvp => kvp.Value);
        
        return _errors.ContainsKey(propertyName) ? _errors[propertyName] : null;
    }
    
    private async void ValidateEmailAsync()
    {
        ClearErrors(nameof(Email));
        
        if (string.IsNullOrWhiteSpace(Email))
        {
            AddError(nameof(Email), "Email is required");
        }
        else if (!IsValidEmail(Email))
        {
            AddError(nameof(Email), "Invalid email format");
        }
        else
        {
            // 비동기 중복 확인
            var isDuplicate = await CheckEmailDuplicateAsync(Email);
            if (isDuplicate)
            {
                AddError(nameof(Email), "Email already exists");
            }
        }
    }
    
    private void ValidateAge()
    {
        ClearErrors(nameof(Age));
        
        if (Age < 0)
        {
            AddError(nameof(Age), "Age cannot be negative");
        }
        else if (Age > 150)
        {
            AddError(nameof(Age), "Age cannot exceed 150");
        }
    }
    
    private void AddError(string propertyName, string error)
    {
        if (!_errors.ContainsKey(propertyName))
            _errors[propertyName] = new List<string>();
        
        if (!_errors[propertyName].Contains(error))
        {
            _errors[propertyName].Add(error);
            OnErrorsChanged(propertyName);
        }
    }
    
    private void ClearErrors(string propertyName)
    {
        if (_errors.ContainsKey(propertyName))
        {
            _errors.Remove(propertyName);
            OnErrorsChanged(propertyName);
        }
    }
    
    private void OnErrorsChanged(string propertyName)
    {
        ErrorsChanged?.Invoke(this, new DataErrorsChangedEventArgs(propertyName));
    }
}
```

## 동적 바인딩

### BindingExpression 조작
```csharp
public partial class DynamicBindingExample : UserControl
{
    public DynamicBindingExample()
    {
        InitializeComponent();
    }
    
    private void UpdateBinding_Click(object sender, RoutedEventArgs e)
    {
        // 기존 바인딩 가져오기
        var bindingExpression = targetTextBox.GetBindingExpression(TextBox.TextProperty);
        var oldBinding = bindingExpression?.ParentBinding;
        
        // 새 바인딩 생성
        var newBinding = new Binding
        {
            Path = new PropertyPath(pathTextBox.Text),
            Mode = BindingMode.TwoWay,
            UpdateSourceTrigger = UpdateSourceTrigger.PropertyChanged
        };
        
        // 바인딩 교체
        targetTextBox.SetBinding(TextBox.TextProperty, newBinding);
    }
    
    private void RefreshBinding_Click(object sender, RoutedEventArgs e)
    {
        // 바인딩 강제 업데이트
        var bindingExpression = targetTextBox.GetBindingExpression(TextBox.TextProperty);
        bindingExpression?.UpdateTarget();
        bindingExpression?.UpdateSource();
    }
}
```

### 프로그래밍 방식으로 바인딩 생성
```csharp
public class BindingFactory
{
    public static void CreateBinding(FrameworkElement target, 
                                   DependencyProperty targetProperty,
                                   object source,
                                   string sourcePath,
                                   BindingMode mode = BindingMode.TwoWay)
    {
        var binding = new Binding(sourcePath)
        {
            Source = source,
            Mode = mode,
            UpdateSourceTrigger = UpdateSourceTrigger.PropertyChanged
        };
        
        // Converter 추가
        if (targetProperty == UIElement.VisibilityProperty && 
            source.GetType().GetProperty(sourcePath)?.PropertyType == typeof(bool))
        {
            binding.Converter = new BooleanToVisibilityConverter();
        }
        
        target.SetBinding(targetProperty, binding);
    }
    
    public static void CreateMultiBinding(FrameworkElement target,
                                        DependencyProperty targetProperty,
                                        IMultiValueConverter converter,
                                        params BindingBase[] bindings)
    {
        var multiBinding = new MultiBinding
        {
            Converter = converter
        };
        
        foreach (var binding in bindings)
        {
            multiBinding.Bindings.Add(binding);
        }
        
        target.SetBinding(targetProperty, multiBinding);
    }
}
```

## 고급 컬렉션 바인딩

### ICollectionView 직접 조작
```csharp
public class AdvancedCollectionViewModel : ViewModelBase
{
    private readonly ObservableCollection<Person> _people;
    private ICollectionView _peopleView;
    private string _filterText;
    
    public AdvancedCollectionViewModel()
    {
        _people = new ObservableCollection<Person>(DataService.GetPeople());
        _peopleView = CollectionViewSource.GetDefaultView(_people);
        
        // 라이브 정렬 활성화
        if (_peopleView is ListCollectionView listView)
        {
            listView.IsLiveSorting = true;
            listView.LiveSortingProperties.Add(nameof(Person.LastName));
        }
        
        // 라이브 필터링 활성화
        _peopleView.Filter = PersonFilter;
        
        // 라이브 그룹화
        _peopleView.GroupDescriptions.Add(
            new PropertyGroupDescription(nameof(Person.Department)));
    }
    
    public ICollectionView PeopleView => _peopleView;
    
    public string FilterText
    {
        get => _filterText;
        set
        {
            if (SetProperty(ref _filterText, value))
            {
                _peopleView.Refresh();
            }
        }
    }
    
    private bool PersonFilter(object obj)
    {
        if (string.IsNullOrWhiteSpace(FilterText))
            return true;
        
        if (obj is Person person)
        {
            return person.FullName.Contains(FilterText, StringComparison.OrdinalIgnoreCase) ||
                   person.Department.Contains(FilterText, StringComparison.OrdinalIgnoreCase);
        }
        
        return false;
    }
    
    public void AddSorting(string propertyName, ListSortDirection direction)
    {
        _peopleView.SortDescriptions.Add(
            new SortDescription(propertyName, direction));
    }
    
    public void RemoveSorting(string propertyName)
    {
        var sortToRemove = _peopleView.SortDescriptions
            .FirstOrDefault(sd => sd.PropertyName == propertyName);
        
        if (sortToRemove != null)
        {
            _peopleView.SortDescriptions.Remove(sortToRemove);
        }
    }
}
```

### 가상화된 컬렉션
```csharp
public class VirtualizingCollection<T> : IList<T>, INotifyCollectionChanged
{
    private readonly Func<int, int, IList<T>> _fetchCallback;
    private readonly Dictionary<int, T> _cache = new();
    private readonly int _pageSize;
    private int _count;
    
    public VirtualizingCollection(Func<int, int, IList<T>> fetchCallback, 
                                 int count, int pageSize = 50)
    {
        _fetchCallback = fetchCallback;
        _count = count;
        _pageSize = pageSize;
    }
    
    public T this[int index]
    {
        get
        {
            if (!_cache.ContainsKey(index))
            {
                LoadPage(index);
            }
            return _cache[index];
        }
        set => throw new NotSupportedException();
    }
    
    private void LoadPage(int index)
    {
        var pageIndex = index / _pageSize;
        var startIndex = pageIndex * _pageSize;
        
        var items = _fetchCallback(startIndex, _pageSize);
        
        for (int i = 0; i < items.Count; i++)
        {
            _cache[startIndex + i] = items[i];
        }
    }
    
    public int Count => _count;
    public bool IsReadOnly => true;
    
    public event NotifyCollectionChangedEventHandler CollectionChanged;
    
    // IList<T> 구현 생략...
}
```

## 바인딩 성능 최적화

### 바인딩 최적화 기법
```csharp
public class OptimizedViewModel : ViewModelBase
{
    private string _heavyProperty;
    private string _cachedHeavyProperty;
    private bool _isHeavyPropertyDirty = true;
    
    // 무거운 계산을 캐싱
    public string HeavyComputedProperty
    {
        get
        {
            if (_isHeavyPropertyDirty)
            {
                _cachedHeavyProperty = PerformHeavyComputation();
                _isHeavyPropertyDirty = false;
            }
            return _cachedHeavyProperty;
        }
    }
    
    public string BaseProperty
    {
        get => _heavyProperty;
        set
        {
            if (SetProperty(ref _heavyProperty, value))
            {
                _isHeavyPropertyDirty = true;
                OnPropertyChanged(nameof(HeavyComputedProperty));
            }
        }
    }
    
    // OneTime 바인딩을 위한 정적 속성
    public static string StaticConfigValue { get; } = 
        ConfigurationManager.AppSettings["SomeValue"];
    
    // 바인딩 지연
    private readonly DispatcherTimer _updateTimer;
    private string _pendingSearchText;
    
    public string SearchText
    {
        get => _pendingSearchText;
        set
        {
            _pendingSearchText = value;
            _updateTimer.Stop();
            _updateTimer.Start(); // 300ms 후 실제 업데이트
        }
    }
    
    private void OnUpdateTimerTick(object sender, EventArgs e)
    {
        _updateTimer.Stop();
        PerformSearch(_pendingSearchText);
    }
}
```

### XAML 최적화
```xml
<!-- VirtualizingStackPanel 사용 -->
<ListBox VirtualizingPanel.IsVirtualizing="True"
         VirtualizingPanel.VirtualizationMode="Recycling"
         ScrollViewer.CanContentScroll="True">
    <ListBox.ItemsPanel>
        <ItemsPanelTemplate>
            <VirtualizingStackPanel/>
        </ItemsPanelTemplate>
    </ListBox.ItemsPanel>
</ListBox>

<!-- 바인딩 모드 최적화 -->
<TextBlock Text="{Binding ReadOnlyProperty, Mode=OneWay}"/>
<TextBlock Text="{Binding StaticProperty, Mode=OneTime}"/>

<!-- UpdateSourceTrigger 최적화 -->
<TextBox Text="{Binding SearchText, 
                UpdateSourceTrigger=LostFocus, 
                Delay=500}"/>
```

## 실전 예제: 고급 검색 필터

### ViewModel
```csharp
public class AdvancedSearchViewModel : ViewModelBase
{
    private readonly ObservableCollection<Product> _allProducts;
    private readonly CollectionViewSource _productsViewSource;
    
    public AdvancedSearchViewModel()
    {
        _allProducts = new ObservableCollection<Product>(LoadProducts());
        _productsViewSource = new CollectionViewSource { Source = _allProducts };
        _productsViewSource.Filter += OnProductsFilter;
        
        // 초기 정렬
        _productsViewSource.SortDescriptions.Add(
            new SortDescription(nameof(Product.Name), ListSortDirection.Ascending));
    }
    
    public ICollectionView ProductsView => _productsViewSource.View;
    
    // 필터 속성들
    public string NameFilter { get; set; }
    public decimal? MinPrice { get; set; }
    public decimal? MaxPrice { get; set; }
    public string CategoryFilter { get; set; }
    public bool InStockOnly { get; set; }
    
    // 필터 적용
    public void ApplyFilters()
    {
        ProductsView.Refresh();
    }
    
    private void OnProductsFilter(object sender, FilterEventArgs e)
    {
        if (e.Item is Product product)
        {
            bool accepted = true;
            
            // 이름 필터
            if (!string.IsNullOrWhiteSpace(NameFilter))
            {
                accepted &= product.Name.Contains(NameFilter, 
                    StringComparison.OrdinalIgnoreCase);
            }
            
            // 가격 필터
            if (MinPrice.HasValue)
                accepted &= product.Price >= MinPrice.Value;
            
            if (MaxPrice.HasValue)
                accepted &= product.Price <= MaxPrice.Value;
            
            // 카테고리 필터
            if (!string.IsNullOrWhiteSpace(CategoryFilter))
                accepted &= product.Category == CategoryFilter;
            
            // 재고 필터
            if (InStockOnly)
                accepted &= product.InStock;
            
            e.Accepted = accepted;
        }
    }
    
    // 정렬 변경
    public void ChangeSorting(string propertyName)
    {
        var currentSort = _productsViewSource.SortDescriptions
            .FirstOrDefault(sd => sd.PropertyName == propertyName);
        
        _productsViewSource.SortDescriptions.Clear();
        
        if (currentSort != null && currentSort.Direction == ListSortDirection.Ascending)
        {
            _productsViewSource.SortDescriptions.Add(
                new SortDescription(propertyName, ListSortDirection.Descending));
        }
        else
        {
            _productsViewSource.SortDescriptions.Add(
                new SortDescription(propertyName, ListSortDirection.Ascending));
        }
    }
}
```

## 핵심 개념 정리
- **MultiBinding**: 여러 소스를 하나의 타겟에 바인딩
- **PriorityBinding**: 우선순위에 따른 바인딩 시도
- **Value Converters**: 데이터 타입 변환
- **CollectionViewSource**: 컬렉션 정렬, 필터링, 그룹화
- **DataTemplateSelector**: 조건에 따른 템플릿 선택
- **Method Binding**: ObjectDataProvider를 통한 메서드 바인딩
- **Validation**: ValidationRule과 INotifyDataErrorInfo
- **동적 바인딩**: 런타임 바인딩 조작
- **성능 최적화**: 가상화와 바인딩 최적화 기법