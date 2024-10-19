# 국제화

## 국제화(i18n) 기초

WPF는 다국어 애플리케이션 개발을 위한 강력한 국제화 기능을 제공합니다.

### 기본 설정
```csharp
// App.xaml.cs
public partial class App : Application
{
    protected override void OnStartup(StartupEventArgs e)
    {
        // 현재 스레드의 문화권 설정
        Thread.CurrentThread.CurrentCulture = new CultureInfo("ko-KR");
        Thread.CurrentThread.CurrentUICulture = new CultureInfo("ko-KR");
        
        // WPF 언어 설정
        FrameworkElement.LanguageProperty.OverrideMetadata(
            typeof(FrameworkElement),
            new FrameworkPropertyMetadata(
                XmlLanguage.GetLanguage(CultureInfo.CurrentCulture.IetfLanguageTag)));
        
        base.OnStartup(e);
    }
}
```

### CultureInfo 사용
```csharp
public class CultureManager
{
    private static CultureInfo _currentCulture;
    
    public static CultureInfo CurrentCulture
    {
        get => _currentCulture ?? CultureInfo.CurrentUICulture;
        set
        {
            _currentCulture = value;
            Thread.CurrentThread.CurrentCulture = value;
            Thread.CurrentThread.CurrentUICulture = value;
            
            // 이벤트 발생
            CultureChanged?.Invoke(null, EventArgs.Empty);
        }
    }
    
    public static event EventHandler CultureChanged;
    
    public static void ChangeCulture(string cultureName)
    {
        try
        {
            var culture = new CultureInfo(cultureName);
            CurrentCulture = culture;
        }
        catch (CultureNotFoundException)
        {
            // 기본 문화권 사용
            CurrentCulture = CultureInfo.InvariantCulture;
        }
    }
    
    public static IEnumerable<CultureInfo> GetAvailableCultures()
    {
        return new[]
        {
            new CultureInfo("en-US"),
            new CultureInfo("ko-KR"),
            new CultureInfo("ja-JP"),
            new CultureInfo("zh-CN"),
            new CultureInfo("fr-FR"),
            new CultureInfo("de-DE")
        };
    }
}
```

## 리소스 파일 (.resx)

### 리소스 파일 구조
```xml
<!-- Resources/Strings.resx (기본 언어) -->
<?xml version="1.0" encoding="utf-8"?>
<root>
  <data name="AppTitle" xml:space="preserve">
    <value>My Application</value>
  </data>
  <data name="WelcomeMessage" xml:space="preserve">
    <value>Welcome to our application!</value>
  </data>
  <data name="SaveButton" xml:space="preserve">
    <value>Save</value>
  </data>
</root>

<!-- Resources/Strings.ko-KR.resx (한국어) -->
<?xml version="1.0" encoding="utf-8"?>
<root>
  <data name="AppTitle" xml:space="preserve">
    <value>내 애플리케이션</value>
  </data>
  <data name="WelcomeMessage" xml:space="preserve">
    <value>우리 애플리케이션에 오신 것을 환영합니다!</value>
  </data>
  <data name="SaveButton" xml:space="preserve">
    <value>저장</value>
  </data>
</root>
```

### 리소스 관리자
```csharp
public class LocalizationManager : INotifyPropertyChanged
{
    private static LocalizationManager _instance;
    private ResourceManager _resourceManager;
    
    public static LocalizationManager Instance => 
        _instance ??= new LocalizationManager();
    
    private LocalizationManager()
    {
        _resourceManager = new ResourceManager(
            "MyApp.Resources.Strings", 
            Assembly.GetExecutingAssembly());
        
        CultureManager.CultureChanged += OnCultureChanged;
    }
    
    public string this[string key]
    {
        get
        {
            try
            {
                return _resourceManager.GetString(key, CultureManager.CurrentCulture)
                       ?? $"[{key}]";
            }
            catch
            {
                return $"[{key}]";
            }
        }
    }
    
    private void OnCultureChanged(object sender, EventArgs e)
    {
        OnPropertyChanged("Item[]");
    }
    
    public event PropertyChangedEventHandler PropertyChanged;
    
    protected virtual void OnPropertyChanged(string propertyName)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }
}
```

## XAML에서 지역화

### MarkupExtension 구현
```csharp
public class LocalizeExtension : MarkupExtension
{
    private string _key;
    
    public LocalizeExtension(string key)
    {
        _key = key;
    }
    
    public string Key
    {
        get => _key;
        set => _key = value;
    }
    
    public override object ProvideValue(IServiceProvider serviceProvider)
    {
        var binding = new Binding($"[{Key}]")
        {
            Source = LocalizationManager.Instance,
            Mode = BindingMode.OneWay
        };
        
        return binding.ProvideValue(serviceProvider);
    }
}
```

### XAML에서 사용
```xml
<Window x:Class="MyApp.MainWindow"
        xmlns:loc="clr-namespace:MyApp.Localization"
        Title="{loc:Localize AppTitle}">
    <Grid>
        <StackPanel>
            <TextBlock Text="{loc:Localize WelcomeMessage}" 
                       FontSize="24" 
                       HorizontalAlignment="Center"/>
            
            <Button Content="{loc:Localize SaveButton}" 
                    Command="{Binding SaveCommand}"/>
            
            <!-- 바인딩과 함께 사용 -->
            <TextBlock>
                <TextBlock.Text>
                    <MultiBinding StringFormat="{loc:Localize UserGreetingFormat}">
                        <Binding Path="UserName"/>
                        <Binding Path="LoginTime" StringFormat="{}{0:t}"/>
                    </MultiBinding>
                </TextBlock.Text>
            </TextBlock>
        </StackPanel>
    </Grid>
</Window>
```

## 동적 언어 변경

### 언어 선택기
```csharp
public class LanguageSelectorViewModel : ViewModelBase
{
    private CultureInfo _selectedCulture;
    
    public ObservableCollection<CultureDisplay> AvailableCultures { get; }
    
    public LanguageSelectorViewModel()
    {
        AvailableCultures = new ObservableCollection<CultureDisplay>(
            CultureManager.GetAvailableCultures()
                .Select(c => new CultureDisplay(c)));
        
        _selectedCulture = CultureManager.CurrentCulture;
    }
    
    public CultureDisplay SelectedCulture
    {
        get => AvailableCultures.FirstOrDefault(c => 
            c.Culture.Name == _selectedCulture.Name);
        set
        {
            if (value != null && value.Culture.Name != _selectedCulture.Name)
            {
                _selectedCulture = value.Culture;
                CultureManager.ChangeCulture(value.Culture.Name);
                OnPropertyChanged();
                
                // 애플리케이션 재시작 여부 확인
                if (RequiresRestart())
                {
                    var result = MessageBox.Show(
                        LocalizationManager.Instance["RestartRequired"],
                        LocalizationManager.Instance["LanguageChange"],
                        MessageBoxButton.YesNo);
                    
                    if (result == MessageBoxResult.Yes)
                    {
                        RestartApplication();
                    }
                }
            }
        }
    }
    
    private bool RequiresRestart()
    {
        // 일부 리소스는 재시작이 필요할 수 있음
        return false;
    }
    
    private void RestartApplication()
    {
        System.Diagnostics.Process.Start(Application.ResourceAssembly.Location);
        Application.Current.Shutdown();
    }
}

public class CultureDisplay
{
    public CultureInfo Culture { get; }
    public string DisplayName { get; }
    public string NativeName { get; }
    
    public CultureDisplay(CultureInfo culture)
    {
        Culture = culture;
        DisplayName = culture.DisplayName;
        NativeName = culture.NativeName;
    }
}
```

### 언어 선택 UI
```xml
<ComboBox ItemsSource="{Binding AvailableCultures}"
          SelectedItem="{Binding SelectedCulture}"
          DisplayMemberPath="NativeName"
          Width="200">
    <ComboBox.ItemTemplate>
        <DataTemplate>
            <StackPanel Orientation="Horizontal">
                <Image Source="{Binding Culture.Name, 
                               Converter={StaticResource CultureToFlagConverter}}"
                       Width="20" Height="15" Margin="0,0,5,0"/>
                <TextBlock Text="{Binding NativeName}"/>
            </StackPanel>
        </DataTemplate>
    </ComboBox.ItemTemplate>
</ComboBox>
```

## 숫자 및 날짜 형식

### 문화권별 형식 지정
```csharp
public class FormattingService
{
    public static string FormatCurrency(decimal amount, CultureInfo culture = null)
    {
        culture ??= CultureManager.CurrentCulture;
        return amount.ToString("C", culture);
    }
    
    public static string FormatNumber(double number, int decimals = 2, 
                                     CultureInfo culture = null)
    {
        culture ??= CultureManager.CurrentCulture;
        return number.ToString($"N{decimals}", culture);
    }
    
    public static string FormatDate(DateTime date, string format = "d", 
                                   CultureInfo culture = null)
    {
        culture ??= CultureManager.CurrentCulture;
        return date.ToString(format, culture);
    }
    
    public static string FormatPercent(double value, int decimals = 2, 
                                      CultureInfo culture = null)
    {
        culture ??= CultureManager.CurrentCulture;
        return value.ToString($"P{decimals}", culture);
    }
}
```

### XAML Converter
```csharp
public class CultureAwareNumberConverter : IValueConverter
{
    public object Convert(object value, Type targetType, object parameter, 
                         CultureInfo culture)
    {
        if (value == null) return string.Empty;
        
        var format = parameter as string ?? "N2";
        culture = CultureManager.CurrentCulture;
        
        return value switch
        {
            decimal d => d.ToString(format, culture),
            double db => db.ToString(format, culture),
            float f => f.ToString(format, culture),
            int i => i.ToString(format, culture),
            _ => value.ToString()
        };
    }
    
    public object ConvertBack(object value, Type targetType, object parameter, 
                             CultureInfo culture)
    {
        if (string.IsNullOrEmpty(value?.ToString()))
            return null;
        
        culture = CultureManager.CurrentCulture;
        var text = value.ToString();
        
        try
        {
            if (targetType == typeof(decimal))
                return decimal.Parse(text, NumberStyles.Number, culture);
            if (targetType == typeof(double))
                return double.Parse(text, NumberStyles.Number, culture);
            if (targetType == typeof(int))
                return int.Parse(text, NumberStyles.Integer, culture);
        }
        catch
        {
            return DependencyProperty.UnsetValue;
        }
        
        return value;
    }
}
```

## 텍스트 방향 (RTL/LTR)

### FlowDirection 처리
```csharp
public class FlowDirectionManager
{
    private static readonly HashSet<string> RtlLanguages = new()
    {
        "ar", // Arabic
        "he", // Hebrew
        "fa", // Persian
        "ur"  // Urdu
    };
    
    public static FlowDirection GetFlowDirection(CultureInfo culture)
    {
        var languageCode = culture.TwoLetterISOLanguageName;
        return RtlLanguages.Contains(languageCode) 
            ? FlowDirection.RightToLeft 
            : FlowDirection.LeftToRight;
    }
    
    public static void ApplyFlowDirection(FrameworkElement element)
    {
        element.FlowDirection = GetFlowDirection(CultureManager.CurrentCulture);
    }
}
```

### 동적 FlowDirection 바인딩
```csharp
public class LocalizationViewModel : ViewModelBase
{
    private FlowDirection _flowDirection;
    
    public LocalizationViewModel()
    {
        UpdateFlowDirection();
        CultureManager.CultureChanged += OnCultureChanged;
    }
    
    public FlowDirection FlowDirection
    {
        get => _flowDirection;
        set => SetProperty(ref _flowDirection, value);
    }
    
    private void OnCultureChanged(object sender, EventArgs e)
    {
        UpdateFlowDirection();
    }
    
    private void UpdateFlowDirection()
    {
        FlowDirection = FlowDirectionManager.GetFlowDirection(
            CultureManager.CurrentCulture);
    }
}
```

## 이미지 및 리소스 지역화

### 문화권별 이미지 관리
```csharp
public class LocalizedImageSource : IValueConverter
{
    public object Convert(object value, Type targetType, object parameter, 
                         CultureInfo culture)
    {
        if (value == null || parameter == null)
            return null;
        
        var imageName = parameter.ToString();
        var cultureName = CultureManager.CurrentCulture.Name;
        
        // 문화권별 이미지 경로 생성
        var culturePath = $"/Images/{cultureName}/{imageName}";
        var defaultPath = $"/Images/{imageName}";
        
        // 문화권별 이미지 확인
        var uri = new Uri(culturePath, UriKind.Relative);
        if (Application.GetResourceStream(uri) != null)
        {
            return new BitmapImage(uri);
        }
        
        // 기본 이미지 반환
        return new BitmapImage(new Uri(defaultPath, UriKind.Relative));
    }
    
    public object ConvertBack(object value, Type targetType, object parameter, 
                             CultureInfo culture)
    {
        throw new NotSupportedException();
    }
}
```

### 리소스 사전 동적 로드
```csharp
public class ThemeManager
{
    private static ResourceDictionary _currentTheme;
    
    public static void LoadCultureResources(CultureInfo culture)
    {
        var cultureName = culture.Name;
        var resourcePath = $"/Themes/{cultureName}/Theme.xaml";
        
        try
        {
            var resourceDict = new ResourceDictionary
            {
                Source = new Uri(resourcePath, UriKind.Relative)
            };
            
            // 기존 테마 제거
            if (_currentTheme != null)
            {
                Application.Current.Resources.MergedDictionaries.Remove(_currentTheme);
            }
            
            // 새 테마 추가
            Application.Current.Resources.MergedDictionaries.Add(resourceDict);
            _currentTheme = resourceDict;
        }
        catch
        {
            // 기본 테마 사용
            LoadDefaultTheme();
        }
    }
    
    private static void LoadDefaultTheme()
    {
        var resourceDict = new ResourceDictionary
        {
            Source = new Uri("/Themes/Default/Theme.xaml", UriKind.Relative)
        };
        
        Application.Current.Resources.MergedDictionaries.Add(resourceDict);
        _currentTheme = resourceDict;
    }
}
```

## 입력 방법 편집기 (IME)

### IME 지원
```csharp
public class ImeManager
{
    public static void ConfigureImeForTextBox(TextBox textBox, CultureInfo culture)
    {
        var languageCode = culture.TwoLetterISOLanguageName;
        
        switch (languageCode)
        {
            case "ko": // 한국어
                InputMethod.SetIsInputMethodEnabled(textBox, true);
                InputMethod.SetPreferredImeState(textBox, InputMethodState.On);
                InputMethod.SetPreferredImeConversionMode(textBox, 
                    ImeConversionModeValues.Native | ImeConversionModeValues.FullShape);
                break;
                
            case "ja": // 일본어
                InputMethod.SetIsInputMethodEnabled(textBox, true);
                InputMethod.SetPreferredImeState(textBox, InputMethodState.On);
                InputMethod.SetPreferredImeConversionMode(textBox, 
                    ImeConversionModeValues.Native | ImeConversionModeValues.Roman);
                break;
                
            case "zh": // 중국어
                InputMethod.SetIsInputMethodEnabled(textBox, true);
                InputMethod.SetPreferredImeState(textBox, InputMethodState.On);
                break;
                
            default:
                InputMethod.SetIsInputMethodEnabled(textBox, false);
                break;
        }
    }
}
```

### 동적 IME 구성
```csharp
public class LocalizedTextBox : TextBox
{
    static LocalizedTextBox()
    {
        DefaultStyleKeyProperty.OverrideMetadata(
            typeof(LocalizedTextBox), 
            new FrameworkPropertyMetadata(typeof(LocalizedTextBox)));
    }
    
    public LocalizedTextBox()
    {
        Loaded += OnLoaded;
        CultureManager.CultureChanged += OnCultureChanged;
    }
    
    private void OnLoaded(object sender, RoutedEventArgs e)
    {
        ConfigureIme();
    }
    
    private void OnCultureChanged(object sender, EventArgs e)
    {
        ConfigureIme();
    }
    
    private void ConfigureIme()
    {
        ImeManager.ConfigureImeForTextBox(this, CultureManager.CurrentCulture);
    }
}
```

## 번역 도구 통합

### 번역 서비스 인터페이스
```csharp
public interface ITranslationService
{
    Task<string> TranslateAsync(string text, string targetLanguage, 
                               string sourceLanguage = null);
    Task<Dictionary<string, string>> TranslateBatchAsync(
        IEnumerable<string> texts, string targetLanguage, 
        string sourceLanguage = null);
}

public class GoogleTranslateService : ITranslationService
{
    private readonly string _apiKey;
    private readonly HttpClient _httpClient;
    
    public GoogleTranslateService(string apiKey)
    {
        _apiKey = apiKey;
        _httpClient = new HttpClient();
    }
    
    public async Task<string> TranslateAsync(string text, string targetLanguage, 
                                           string sourceLanguage = null)
    {
        var url = $"https://translation.googleapis.com/language/translate/v2" +
                 $"?key={_apiKey}" +
                 $"&q={Uri.EscapeDataString(text)}" +
                 $"&target={targetLanguage}";
        
        if (!string.IsNullOrEmpty(sourceLanguage))
        {
            url += $"&source={sourceLanguage}";
        }
        
        var response = await _httpClient.GetStringAsync(url);
        // JSON 파싱 및 번역 결과 반환
        return ParseTranslationResponse(response);
    }
    
    public async Task<Dictionary<string, string>> TranslateBatchAsync(
        IEnumerable<string> texts, string targetLanguage, 
        string sourceLanguage = null)
    {
        var tasks = texts.Select(text => 
            TranslateAsync(text, targetLanguage, sourceLanguage));
        
        var results = await Task.WhenAll(tasks);
        
        return texts.Zip(results, (text, translation) => 
            new { text, translation })
            .ToDictionary(x => x.text, x => x.translation);
    }
    
    private string ParseTranslationResponse(string json)
    {
        // JSON 파싱 로직
        dynamic result = JsonSerializer.Deserialize<dynamic>(json);
        return result.data.translations[0].translatedText;
    }
}
```

## 지역화 테스트

### 의사 지역화 (Pseudo-localization)
```csharp
public class PseudoLocalizationService
{
    private static readonly Dictionary<char, string> CharacterMap = new()
    {
        {'a', 'ȧ'}, {'e', 'ḗ'}, {'i', 'ī'}, {'o', 'ō'}, {'u', 'ū'},
        {'A', 'Ȧ'}, {'E', 'Ḗ'}, {'I', 'Ī'}, {'O', 'Ō'}, {'U', 'Ū'}
    };
    
    public static string PseudoLocalize(string text)
    {
        if (string.IsNullOrEmpty(text))
            return text;
        
        var result = new StringBuilder();
        result.Append("[");
        
        foreach (char c in text)
        {
            if (CharacterMap.TryGetValue(c, out var replacement))
            {
                result.Append(replacement);
            }
            else
            {
                result.Append(c);
            }
        }
        
        // 텍스트 확장 (30% 정도)
        int expansionLength = (int)(text.Length * 0.3);
        result.Append('~', expansionLength);
        
        result.Append("]");
        
        return result.ToString();
    }
}

public class PseudoLocalizationResourceManager : ResourceManager
{
    private readonly bool _enablePseudoLocalization;
    
    public PseudoLocalizationResourceManager(string baseName, Assembly assembly, 
                                           bool enablePseudoLocalization) 
        : base(baseName, assembly)
    {
        _enablePseudoLocalization = enablePseudoLocalization;
    }
    
    public override string GetString(string name, CultureInfo culture)
    {
        var value = base.GetString(name, culture);
        
        if (_enablePseudoLocalization && culture.Name == "qps-ploc")
        {
            return PseudoLocalizationService.PseudoLocalize(value);
        }
        
        return value;
    }
}
```

### 지역화 검증
```csharp
public class LocalizationValidator
{
    public class ValidationResult
    {
        public string Key { get; set; }
        public string Culture { get; set; }
        public string Issue { get; set; }
        public ValidationSeverity Severity { get; set; }
    }
    
    public enum ValidationSeverity
    {
        Info,
        Warning,
        Error
    }
    
    public List<ValidationResult> ValidateResources(
        ResourceManager resourceManager, 
        IEnumerable<CultureInfo> cultures)
    {
        var results = new List<ValidationResult>();
        var defaultCulture = CultureInfo.InvariantCulture;
        
        // 기본 리소스 키 수집
        var resourceSet = resourceManager.GetResourceSet(
            defaultCulture, true, true);
        var defaultKeys = new HashSet<string>();
        
        foreach (DictionaryEntry entry in resourceSet)
        {
            defaultKeys.Add(entry.Key.ToString());
        }
        
        // 각 문화권 검증
        foreach (var culture in cultures)
        {
            var cultureResourceSet = resourceManager.GetResourceSet(
                culture, true, false);
            
            if (cultureResourceSet == null)
            {
                results.Add(new ValidationResult
                {
                    Culture = culture.Name,
                    Issue = "리소스 파일이 없습니다",
                    Severity = ValidationSeverity.Error
                });
                continue;
            }
            
            var cultureKeys = new HashSet<string>();
            
            foreach (DictionaryEntry entry in cultureResourceSet)
            {
                var key = entry.Key.ToString();
                var value = entry.Value?.ToString();
                
                cultureKeys.Add(key);
                
                // 빈 값 검사
                if (string.IsNullOrWhiteSpace(value))
                {
                    results.Add(new ValidationResult
                    {
                        Key = key,
                        Culture = culture.Name,
                        Issue = "빈 값",
                        Severity = ValidationSeverity.Warning
                    });
                }
                
                // 텍스트 길이 검사
                var defaultValue = resourceManager.GetString(key, defaultCulture);
                if (defaultValue != null && value != null)
                {
                    var lengthRatio = (double)value.Length / defaultValue.Length;
                    if (lengthRatio > 1.5)
                    {
                        results.Add(new ValidationResult
                        {
                            Key = key,
                            Culture = culture.Name,
                            Issue = $"텍스트가 너무 깁니다 ({lengthRatio:P0} 증가)",
                            Severity = ValidationSeverity.Info
                        });
                    }
                }
            }
            
            // 누락된 키 검사
            var missingKeys = defaultKeys.Except(cultureKeys);
            foreach (var missingKey in missingKeys)
            {
                results.Add(new ValidationResult
                {
                    Key = missingKey,
                    Culture = culture.Name,
                    Issue = "키 누락",
                    Severity = ValidationSeverity.Error
                });
            }
        }
        
        return results;
    }
}
```

## 실전 예제: 다국어 설정 화면

### 설정 ViewModel
```csharp
public class LocalizationSettingsViewModel : ViewModelBase
{
    private readonly ITranslationService _translationService;
    private CultureDisplay _selectedCulture;
    private bool _enableAutoTranslation;
    private bool _usePseudoLocalization;
    
    public LocalizationSettingsViewModel(ITranslationService translationService)
    {
        _translationService = translationService;
        LoadSettings();
        
        ValidateResourcesCommand = new RelayCommand(ValidateResources);
        ExportTranslationsCommand = new RelayCommand(ExportTranslations);
        ImportTranslationsCommand = new RelayCommand(ImportTranslations);
    }
    
    public ObservableCollection<CultureDisplay> AvailableCultures { get; } = 
        new(CultureManager.GetAvailableCultures()
            .Select(c => new CultureDisplay(c)));
    
    public CultureDisplay SelectedCulture
    {
        get => _selectedCulture;
        set
        {
            if (SetProperty(ref _selectedCulture, value))
            {
                ApplyCulture();
            }
        }
    }
    
    public bool EnableAutoTranslation
    {
        get => _enableAutoTranslation;
        set => SetProperty(ref _enableAutoTranslation, value);
    }
    
    public bool UsePseudoLocalization
    {
        get => _usePseudoLocalization;
        set
        {
            if (SetProperty(ref _usePseudoLocalization, value))
            {
                ApplyPseudoLocalization();
            }
        }
    }
    
    public ICommand ValidateResourcesCommand { get; }
    public ICommand ExportTranslationsCommand { get; }
    public ICommand ImportTranslationsCommand { get; }
    
    private void LoadSettings()
    {
        var settings = Properties.Settings.Default;
        var cultureName = settings.Culture ?? "en-US";
        
        _selectedCulture = AvailableCultures.FirstOrDefault(
            c => c.Culture.Name == cultureName) ?? AvailableCultures.First();
        
        _enableAutoTranslation = settings.EnableAutoTranslation;
        _usePseudoLocalization = settings.UsePseudoLocalization;
    }
    
    private void ApplyCulture()
    {
        if (_selectedCulture != null)
        {
            CultureManager.ChangeCulture(_selectedCulture.Culture.Name);
            
            var settings = Properties.Settings.Default;
            settings.Culture = _selectedCulture.Culture.Name;
            settings.Save();
        }
    }
    
    private void ApplyPseudoLocalization()
    {
        if (_usePseudoLocalization)
        {
            CultureManager.ChangeCulture("qps-ploc");
        }
        else
        {
            ApplyCulture();
        }
    }
    
    private async void ValidateResources()
    {
        var validator = new LocalizationValidator();
        var resourceManager = new ResourceManager(
            "MyApp.Resources.Strings", 
            Assembly.GetExecutingAssembly());
        
        var results = validator.ValidateResources(
            resourceManager, 
            AvailableCultures.Select(c => c.Culture));
        
        // 결과 표시
        var window = new ValidationResultsWindow(results);
        window.ShowDialog();
    }
    
    private async void ExportTranslations()
    {
        var dialog = new SaveFileDialog
        {
            Filter = "Excel Files (*.xlsx)|*.xlsx|CSV Files (*.csv)|*.csv",
            DefaultExt = ".xlsx"
        };
        
        if (dialog.ShowDialog() == true)
        {
            await ExportToFileAsync(dialog.FileName);
        }
    }
    
    private async Task ExportToFileAsync(string filePath)
    {
        // 리소스 내보내기 로직
        var resourceManager = new ResourceManager(
            "MyApp.Resources.Strings", 
            Assembly.GetExecutingAssembly());
        
        var data = new List<TranslationEntry>();
        
        foreach (var culture in AvailableCultures)
        {
            var resourceSet = resourceManager.GetResourceSet(
                culture.Culture, true, false);
            
            if (resourceSet != null)
            {
                foreach (DictionaryEntry entry in resourceSet)
                {
                    data.Add(new TranslationEntry
                    {
                        Key = entry.Key.ToString(),
                        Culture = culture.Culture.Name,
                        Value = entry.Value?.ToString() ?? string.Empty
                    });
                }
            }
        }
        
        // Excel 또는 CSV로 저장
        if (Path.GetExtension(filePath).ToLower() == ".xlsx")
        {
            await ExportToExcelAsync(filePath, data);
        }
        else
        {
            await ExportToCsvAsync(filePath, data);
        }
    }
    
    private async void ImportTranslations()
    {
        var dialog = new OpenFileDialog
        {
            Filter = "Excel Files (*.xlsx)|*.xlsx|CSV Files (*.csv)|*.csv"
        };
        
        if (dialog.ShowDialog() == true)
        {
            await ImportFromFileAsync(dialog.FileName);
        }
    }
}

public class TranslationEntry
{
    public string Key { get; set; }
    public string Culture { get; set; }
    public string Value { get; set; }
}
```

### 설정 UI
```xml
<UserControl x:Class="MyApp.LocalizationSettingsView">
    <Grid>
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>
        
        <!-- 언어 선택 -->
        <GroupBox Grid.Row="0" Header="{loc:Localize LanguageSettings}">
            <StackPanel>
                <Label Content="{loc:Localize SelectLanguage}"/>
                <ComboBox ItemsSource="{Binding AvailableCultures}"
                          SelectedItem="{Binding SelectedCulture}"
                          DisplayMemberPath="NativeName">
                    <ComboBox.ItemTemplate>
                        <DataTemplate>
                            <StackPanel Orientation="Horizontal">
                                <Image Source="{Binding Culture, 
                                       Converter={StaticResource CultureToFlagConverter}}"
                                       Width="24" Height="16" Margin="0,0,8,0"/>
                                <TextBlock>
                                    <TextBlock.Text>
                                        <MultiBinding StringFormat="{}{0} ({1})">
                                            <Binding Path="NativeName"/>
                                            <Binding Path="Culture.Name"/>
                                        </MultiBinding>
                                    </TextBlock.Text>
                                </TextBlock>
                            </StackPanel>
                        </DataTemplate>
                    </ComboBox.ItemTemplate>
                </ComboBox>
                
                <CheckBox Content="{loc:Localize EnableAutoTranslation}"
                          IsChecked="{Binding EnableAutoTranslation}"
                          Margin="0,10,0,0"/>
                
                <CheckBox Content="{loc:Localize UsePseudoLocalization}"
                          IsChecked="{Binding UsePseudoLocalization}"
                          ToolTip="{loc:Localize PseudoLocalizationTooltip}"/>
            </StackPanel>
        </GroupBox>
        
        <!-- 번역 관리 -->
        <GroupBox Grid.Row="1" Header="{loc:Localize TranslationManagement}">
            <Grid>
                <Grid.RowDefinitions>
                    <RowDefinition Height="Auto"/>
                    <RowDefinition Height="*"/>
                </Grid.RowDefinitions>
                
                <ToolBar Grid.Row="0">
                    <Button Command="{Binding ValidateResourcesCommand}"
                            Content="{loc:Localize ValidateResources}"/>
                    <Separator/>
                    <Button Command="{Binding ExportTranslationsCommand}"
                            Content="{loc:Localize ExportTranslations}"/>
                    <Button Command="{Binding ImportTranslationsCommand}"
                            Content="{loc:Localize ImportTranslations}"/>
                </ToolBar>
                
                <!-- 번역 상태 표시 -->
                <DataGrid Grid.Row="1" 
                          ItemsSource="{Binding TranslationStatus}"
                          AutoGenerateColumns="False"
                          CanUserAddRows="False">
                    <DataGrid.Columns>
                        <DataGridTextColumn Header="{loc:Localize Language}"
                                          Binding="{Binding CultureName}"
                                          Width="150"/>
                        <DataGridTextColumn Header="{loc:Localize TotalKeys}"
                                          Binding="{Binding TotalKeys}"
                                          Width="100"/>
                        <DataGridTextColumn Header="{loc:Localize TranslatedKeys}"
                                          Binding="{Binding TranslatedKeys}"
                                          Width="120"/>
                        <DataGridTextColumn Header="{loc:Localize Progress}"
                                          Binding="{Binding Progress, StringFormat=P}"
                                          Width="100"/>
                    </DataGrid.Columns>
                </DataGrid>
            </Grid>
        </GroupBox>
        
        <!-- 미리보기 -->
        <GroupBox Grid.Row="2" Header="{loc:Localize Preview}">
            <StackPanel>
                <TextBlock Text="{loc:Localize SampleText}" FontSize="14"/>
                <TextBlock Text="{Binding SampleDate, 
                           Converter={StaticResource DateTimeConverter}}"
                           Margin="0,5"/>
                <TextBlock Text="{Binding SampleCurrency, 
                           Converter={StaticResource CurrencyConverter}}"
                           Margin="0,5"/>
            </StackPanel>
        </GroupBox>
    </Grid>
</UserControl>
```

## 핵심 개념 정리
- **CultureInfo**: 문화권 정보 관리
- **ResourceManager**: 리소스 파일 관리
- **리소스 파일 (.resx)**: 언어별 문자열 저장
- **MarkupExtension**: XAML에서 지역화 지원
- **동적 언어 변경**: 런타임 언어 전환
- **숫자/날짜 형식**: 문화권별 형식 지정
- **RTL/LTR**: 텍스트 방향 처리
- **IME**: 입력 방법 편집기 지원
- **번역 서비스**: 자동 번역 통합
- **의사 지역화**: 지역화 테스트
- **검증 도구**: 번역 누락 및 오류 검사
- **문화권별 리소스**: 이미지, 사운드 등 미디어 지역화