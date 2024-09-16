# 리소스와 테마

## 리소스 시스템 개요

WPF의 리소스 시스템은 재사용 가능한 객체를 정의하고 관리하는 메커니즘입니다.

### 리소스의 장점
- **재사용성**: 한 번 정의하고 여러 곳에서 사용
- **일관성**: 중앙 집중식 관리로 일관된 UI
- **유지보수성**: 한 곳에서 수정하면 모든 곳에 반영
- **테마 지원**: 런타임에 리소스 교체 가능

## 리소스 정의와 사용

### 기본 리소스 정의
```xml
<Window.Resources>
    <!-- Brush 리소스 -->
    <SolidColorBrush x:Key="PrimaryBrush" Color="#007ACC"/>
    <LinearGradientBrush x:Key="BackgroundBrush">
        <GradientStop Color="#E0E0E0" Offset="0"/>
        <GradientStop Color="#FFFFFF" Offset="1"/>
    </LinearGradientBrush>
    
    <!-- 값 리소스 -->
    <sys:Double x:Key="StandardFontSize">14</sys:Double>
    <Thickness x:Key="StandardMargin">5</Thickness>
    
    <!-- 문자열 리소스 -->
    <sys:String x:Key="AppTitle">My WPF Application</sys:String>
</Window.Resources>

<!-- 리소스 사용 -->
<TextBlock Text="{StaticResource AppTitle}"
           FontSize="{StaticResource StandardFontSize}"
           Margin="{StaticResource StandardMargin}"
           Foreground="{StaticResource PrimaryBrush}"/>
```

### 리소스 스코프
```xml
<!-- Application 레벨 (App.xaml) -->
<Application.Resources>
    <Color x:Key="AppPrimaryColor">#007ACC</Color>
</Application.Resources>

<!-- Window 레벨 -->
<Window.Resources>
    <Style x:Key="WindowButtonStyle" TargetType="Button">
        <!-- Window 내에서만 사용 가능 -->
    </Style>
</Window.Resources>

<!-- Panel 레벨 -->
<StackPanel>
    <StackPanel.Resources>
        <sys:Double x:Key="LocalFontSize">16</sys:Double>
    </StackPanel.Resources>
    
    <!-- 가장 가까운 리소스 사용 -->
    <TextBlock FontSize="{StaticResource LocalFontSize}"/>
</StackPanel>
```

## StaticResource vs DynamicResource

### StaticResource
```xml
<!-- 컴파일 타임에 해결, 성능이 좋음 -->
<Button Background="{StaticResource PrimaryBrush}" 
        Content="Static Resource"/>

<!-- 리소스가 먼저 정의되어야 함 -->
<Window>
    <Window.Resources>
        <SolidColorBrush x:Key="MyBrush" Color="Blue"/>
    </Window.Resources>
    
    <Button Background="{StaticResource MyBrush}"/>
</Window>
```

### DynamicResource
```xml
<!-- 런타임에 해결, 동적 변경 가능 -->
<Button Background="{DynamicResource ThemeBrush}" 
        Content="Dynamic Resource"/>

<!-- 런타임에 리소스 변경 -->
<Button Click="ChangeTheme_Click" Content="Change Theme"/>
```

```csharp
private void ChangeTheme_Click(object sender, RoutedEventArgs e)
{
    // 리소스 동적 변경
    this.Resources["ThemeBrush"] = new SolidColorBrush(Colors.Red);
}
```

### 선택 기준
```xml
<!-- StaticResource 사용 사례 -->
<!-- - 변경되지 않는 리소스 -->
<!-- - 성능이 중요한 경우 -->
<Border BorderBrush="{StaticResource BorderBrush}"/>

<!-- DynamicResource 사용 사례 -->
<!-- - 테마 변경이 필요한 경우 -->
<!-- - 리소스가 나중에 정의되는 경우 -->
<Button Background="{DynamicResource ThemeBackground}"/>
```

## ResourceDictionary

### 기본 ResourceDictionary
```xml
<!-- Colors.xaml -->
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <Color x:Key="PrimaryColor">#007ACC</Color>
    <Color x:Key="SecondaryColor">#F0F0F0</Color>
    <Color x:Key="AccentColor">#FFA500</Color>
    
    <SolidColorBrush x:Key="PrimaryBrush" Color="{StaticResource PrimaryColor}"/>
    <SolidColorBrush x:Key="SecondaryBrush" Color="{StaticResource SecondaryColor}"/>
    <SolidColorBrush x:Key="AccentBrush" Color="{StaticResource AccentColor}"/>
</ResourceDictionary>
```

### MergedDictionaries
```xml
<!-- App.xaml -->
<Application.Resources>
    <ResourceDictionary>
        <ResourceDictionary.MergedDictionaries>
            <!-- 여러 리소스 딕셔너리 병합 -->
            <ResourceDictionary Source="Resources/Colors.xaml"/>
            <ResourceDictionary Source="Resources/Brushes.xaml"/>
            <ResourceDictionary Source="Resources/Styles.xaml"/>
            <ResourceDictionary Source="Resources/Templates.xaml"/>
        </ResourceDictionary.MergedDictionaries>
        
        <!-- 로컬 리소스 -->
        <sys:String x:Key="AppName">My Application</sys:String>
    </ResourceDictionary>
</Application.Resources>
```

### 조건부 리소스 로딩
```csharp
public partial class App : Application
{
    protected override void OnStartup(StartupEventArgs e)
    {
        base.OnStartup(e);
        
        // 조건에 따른 리소스 로딩
        string theme = ConfigurationManager.AppSettings["Theme"];
        LoadTheme(theme);
    }
    
    private void LoadTheme(string themeName)
    {
        var dict = new ResourceDictionary();
        dict.Source = new Uri($"/Themes/{themeName}Theme.xaml", UriKind.Relative);
        
        // 기존 테마 제거
        this.Resources.MergedDictionaries.Clear();
        
        // 새 테마 추가
        this.Resources.MergedDictionaries.Add(dict);
    }
}
```

## 테마 시스템 구현

### 테마 구조
```xml
<!-- LightTheme.xaml -->
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <!-- 색상 정의 -->
    <Color x:Key="BackgroundColor">White</Color>
    <Color x:Key="ForegroundColor">Black</Color>
    <Color x:Key="BorderColor">#CCCCCC</Color>
    <Color x:Key="HighlightColor">#007ACC</Color>
    
    <!-- 브러시 정의 -->
    <SolidColorBrush x:Key="BackgroundBrush" Color="{StaticResource BackgroundColor}"/>
    <SolidColorBrush x:Key="ForegroundBrush" Color="{StaticResource ForegroundColor}"/>
    <SolidColorBrush x:Key="BorderBrush" Color="{StaticResource BorderColor}"/>
    <SolidColorBrush x:Key="HighlightBrush" Color="{StaticResource HighlightColor}"/>
    
    <!-- 스타일 정의 -->
    <Style x:Key="ThemedButton" TargetType="Button">
        <Setter Property="Background" Value="{DynamicResource BackgroundBrush}"/>
        <Setter Property="Foreground" Value="{DynamicResource ForegroundBrush}"/>
        <Setter Property="BorderBrush" Value="{DynamicResource BorderBrush}"/>
        <Setter Property="Template">
            <Setter.Value>
                <ControlTemplate TargetType="Button">
                    <Border Background="{TemplateBinding Background}"
                            BorderBrush="{TemplateBinding BorderBrush}"
                            BorderThickness="1"
                            CornerRadius="4"
                            Padding="10,5">
                        <ContentPresenter HorizontalAlignment="Center"
                                        VerticalAlignment="Center"/>
                    </Border>
                    <ControlTemplate.Triggers>
                        <Trigger Property="IsMouseOver" Value="True">
                            <Setter Property="Background" 
                                    Value="{DynamicResource HighlightBrush}"/>
                            <Setter Property="Foreground" Value="White"/>
                        </Trigger>
                    </ControlTemplate.Triggers>
                </ControlTemplate>
            </Setter.Value>
        </Setter>
    </Style>
</ResourceDictionary>
```

### 테마 전환 구현
```csharp
public class ThemeManager
{
    private static ThemeManager _instance;
    public static ThemeManager Instance => _instance ??= new ThemeManager();
    
    private string _currentTheme = "Light";
    
    public string CurrentTheme
    {
        get => _currentTheme;
        set
        {
            if (_currentTheme != value)
            {
                _currentTheme = value;
                ApplyTheme(value);
            }
        }
    }
    
    public void ApplyTheme(string themeName)
    {
        var app = Application.Current;
        var themeUri = new Uri($"/Themes/{themeName}Theme.xaml", UriKind.Relative);
        
        // 기존 테마 리소스 찾기
        var oldTheme = app.Resources.MergedDictionaries
            .FirstOrDefault(d => d.Source?.ToString().Contains("Theme.xaml") == true);
        
        // 새 테마 로드
        var newTheme = new ResourceDictionary { Source = themeUri };
        
        // 테마 교체
        if (oldTheme != null)
        {
            int index = app.Resources.MergedDictionaries.IndexOf(oldTheme);
            app.Resources.MergedDictionaries[index] = newTheme;
        }
        else
        {
            app.Resources.MergedDictionaries.Add(newTheme);
        }
    }
    
    public IEnumerable<string> AvailableThemes => new[] { "Light", "Dark", "Blue" };
}
```

## 시스템 리소스

### SystemColors 사용
```xml
<!-- Windows 시스템 색상 사용 -->
<Border Background="{x:Static SystemColors.ControlBrush}"
        BorderBrush="{x:Static SystemColors.ActiveBorderBrush}">
    <TextBlock Foreground="{x:Static SystemColors.ControlTextBrush}"
               Text="System Colors"/>
</Border>

<!-- 동적 시스템 색상 -->
<Button Background="{DynamicResource {x:Static SystemColors.HighlightBrushKey}}"
        Foreground="{DynamicResource {x:Static SystemColors.HighlightTextBrushKey}}"/>
```

### SystemFonts 사용
```xml
<TextBlock FontFamily="{x:Static SystemFonts.MessageFontFamily}"
           FontSize="{x:Static SystemFonts.MessageFontSize}"
           FontWeight="{x:Static SystemFonts.MessageFontWeight}"
           Text="System Font"/>
```

### SystemParameters 사용
```xml
<Window Height="{x:Static SystemParameters.PrimaryScreenHeight}"
        Width="{x:Static SystemParameters.PrimaryScreenWidth}">
    <Border Margin="{x:Static SystemParameters.WindowResizeBorderThickness}"/>
</Window>
```

## 리소스 최적화

### 리소스 공유
```xml
<!-- Shared 리소스 -->
<DrawingBrush x:Key="CheckeredBrush" x:Shared="True">
    <DrawingBrush.Drawing>
        <DrawingGroup>
            <GeometryDrawing Brush="LightGray">
                <GeometryDrawing.Geometry>
                    <RectangleGeometry Rect="0,0,10,10"/>
                </GeometryDrawing.Geometry>
            </GeometryDrawing>
        </DrawingGroup>
    </DrawingBrush.Drawing>
</DrawingBrush>

<!-- Non-Shared 리소스 (각 사용처마다 새 인스턴스) -->
<Style x:Key="AnimatedStyle" TargetType="Button" x:Shared="False">
    <!-- 애니메이션이 포함된 스타일 -->
</Style>
```

### 리소스 지연 로딩
```csharp
public partial class MainWindow : Window
{
    private bool _resourcesLoaded = false;
    
    private void LoadResourcesOnDemand()
    {
        if (!_resourcesLoaded)
        {
            var dict = new ResourceDictionary
            {
                Source = new Uri("/HeavyResources.xaml", UriKind.Relative)
            };
            
            this.Resources.MergedDictionaries.Add(dict);
            _resourcesLoaded = true;
        }
    }
}
```

## 실전 예제: 테마 전환 애플리케이션

### MainWindow.xaml
```xml
<Window x:Class="ThemeDemo.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Theme Demo" Height="400" Width="600"
        Background="{DynamicResource BackgroundBrush}">
    <Grid Margin="20">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
        </Grid.RowDefinitions>
        
        <!-- 테마 선택 -->
        <StackPanel Grid.Row="0" Orientation="Horizontal" Margin="0,0,0,20">
            <TextBlock Text="Theme:" 
                       Foreground="{DynamicResource ForegroundBrush}"
                       VerticalAlignment="Center"
                       Margin="0,0,10,0"/>
            <ComboBox x:Name="ThemeSelector" 
                      Width="150"
                      SelectionChanged="ThemeSelector_SelectionChanged"/>
        </StackPanel>
        
        <!-- 콘텐츠 영역 -->
        <Border Grid.Row="1" 
                BorderBrush="{DynamicResource BorderBrush}"
                BorderThickness="1"
                CornerRadius="8"
                Padding="20">
            <StackPanel>
                <TextBlock Text="Sample Application"
                           FontSize="24"
                           Foreground="{DynamicResource ForegroundBrush}"
                           Margin="0,0,0,20"/>
                
                <Button Content="Primary Button"
                        Style="{DynamicResource ThemedButton}"
                        Margin="0,0,0,10"/>
                
                <TextBox Text="Sample text input"
                         Background="{DynamicResource BackgroundBrush}"
                         Foreground="{DynamicResource ForegroundBrush}"
                         BorderBrush="{DynamicResource BorderBrush}"
                         Padding="5"
                         Margin="0,0,0,10"/>
                
                <CheckBox Content="Enable feature"
                          Foreground="{DynamicResource ForegroundBrush}"
                          Margin="0,0,0,10"/>
                
                <ProgressBar Value="60"
                             Height="20"
                             Foreground="{DynamicResource HighlightBrush}"
                             Background="{DynamicResource SecondaryBrush}"/>
            </StackPanel>
        </Border>
    </Grid>
</Window>
```

## 핵심 개념 정리
- **리소스**: 재사용 가능한 객체 정의
- **StaticResource**: 컴파일 타임 바인딩, 성능 우선
- **DynamicResource**: 런타임 바인딩, 유연성 우선
- **ResourceDictionary**: 리소스 컬렉션 관리
- **MergedDictionaries**: 여러 리소스 딕셔너리 통합
- **테마 시스템**: 동적 리소스 교체로 구현