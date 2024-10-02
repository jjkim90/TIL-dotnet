# 커스텀 컨트롤

## UserControl vs CustomControl

WPF에서 커스텀 컨트롤을 만드는 두 가지 주요 방법이 있습니다.

### UserControl
- **용도**: 애플리케이션 특화 컨트롤
- **상속**: UserControl 클래스 상속
- **템플릿**: 고정된 XAML 레이아웃
- **재사용성**: 제한적 (같은 프로젝트 내)

### CustomControl
- **용도**: 재사용 가능한 범용 컨트롤
- **상속**: Control 또는 기존 컨트롤 상속
- **템플릿**: Generic.xaml의 ControlTemplate
- **재사용성**: 높음 (다른 프로젝트에서도 사용)

## UserControl 만들기

### 기본 UserControl
```xml
<!-- SearchBox.xaml -->
<UserControl x:Class="MyApp.Controls.SearchBox"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             x:Name="searchBoxControl">
    <Grid>
        <Grid.ColumnDefinitions>
            <ColumnDefinition Width="*"/>
            <ColumnDefinition Width="Auto"/>
        </Grid.ColumnDefinitions>
        
        <TextBox x:Name="searchTextBox" 
                 Grid.Column="0"
                 Text="{Binding SearchText, ElementName=searchBoxControl}"
                 KeyDown="SearchTextBox_KeyDown"/>
        
        <Button Grid.Column="1" 
                Content="Search"
                Command="{Binding SearchCommand, ElementName=searchBoxControl}"
                CommandParameter="{Binding Text, ElementName=searchTextBox}"/>
    </Grid>
</UserControl>
```

```csharp
// SearchBox.xaml.cs
public partial class SearchBox : UserControl
{
    // 의존성 속성
    public static readonly DependencyProperty SearchTextProperty =
        DependencyProperty.Register(
            nameof(SearchText), 
            typeof(string), 
            typeof(SearchBox),
            new PropertyMetadata(string.Empty, OnSearchTextChanged));
    
    public static readonly DependencyProperty SearchCommandProperty =
        DependencyProperty.Register(
            nameof(SearchCommand), 
            typeof(ICommand), 
            typeof(SearchBox));
    
    // CLR 속성
    public string SearchText
    {
        get => (string)GetValue(SearchTextProperty);
        set => SetValue(SearchTextProperty, value);
    }
    
    public ICommand SearchCommand
    {
        get => (ICommand)GetValue(SearchCommandProperty);
        set => SetValue(SearchCommandProperty, value);
    }
    
    public SearchBox()
    {
        InitializeComponent();
    }
    
    private static void OnSearchTextChanged(DependencyObject d, 
                                           DependencyPropertyChangedEventArgs e)
    {
        var searchBox = (SearchBox)d;
        // 속성 변경 처리
    }
    
    private void SearchTextBox_KeyDown(object sender, KeyEventArgs e)
    {
        if (e.Key == Key.Enter && SearchCommand?.CanExecute(SearchText) == true)
        {
            SearchCommand.Execute(SearchText);
        }
    }
}
```

### 사용 예제
```xml
<Window xmlns:controls="clr-namespace:MyApp.Controls">
    <controls:SearchBox SearchText="{Binding QueryText}"
                        SearchCommand="{Binding PerformSearchCommand}"/>
</Window>
```

## CustomControl 만들기

### 기본 구조
```csharp
// NumericUpDown.cs
[TemplatePart(Name = "PART_UpButton", Type = typeof(Button))]
[TemplatePart(Name = "PART_DownButton", Type = typeof(Button))]
[TemplatePart(Name = "PART_TextBox", Type = typeof(TextBox))]
public class NumericUpDown : Control
{
    // Template parts
    private Button _upButton;
    private Button _downButton;
    private TextBox _textBox;
    
    static NumericUpDown()
    {
        DefaultStyleKeyProperty.OverrideMetadata(
            typeof(NumericUpDown), 
            new FrameworkPropertyMetadata(typeof(NumericUpDown)));
    }
    
    // 의존성 속성
    public static readonly DependencyProperty ValueProperty =
        DependencyProperty.Register(
            nameof(Value), 
            typeof(double), 
            typeof(NumericUpDown),
            new FrameworkPropertyMetadata(
                0.0,
                FrameworkPropertyMetadataOptions.BindsTwoWayByDefault,
                OnValueChanged,
                CoerceValue));
    
    public static readonly DependencyProperty MinimumProperty =
        DependencyProperty.Register(
            nameof(Minimum), 
            typeof(double), 
            typeof(NumericUpDown),
            new PropertyMetadata(double.MinValue));
    
    public static readonly DependencyProperty MaximumProperty =
        DependencyProperty.Register(
            nameof(Maximum), 
            typeof(double), 
            typeof(NumericUpDown),
            new PropertyMetadata(double.MaxValue));
    
    public static readonly DependencyProperty StepProperty =
        DependencyProperty.Register(
            nameof(Step), 
            typeof(double), 
            typeof(NumericUpDown),
            new PropertyMetadata(1.0));
    
    // CLR 속성
    public double Value
    {
        get => (double)GetValue(ValueProperty);
        set => SetValue(ValueProperty, value);
    }
    
    public double Minimum
    {
        get => (double)GetValue(MinimumProperty);
        set => SetValue(MinimumProperty, value);
    }
    
    public double Maximum
    {
        get => (double)GetValue(MaximumProperty);
        set => SetValue(MaximumProperty, value);
    }
    
    public double Step
    {
        get => (double)GetValue(StepProperty);
        set => SetValue(StepProperty, value);
    }
    
    // 이벤트
    public static readonly RoutedEvent ValueChangedEvent =
        EventManager.RegisterRoutedEvent(
            nameof(ValueChanged),
            RoutingStrategy.Bubble,
            typeof(RoutedPropertyChangedEventHandler<double>),
            typeof(NumericUpDown));
    
    public event RoutedPropertyChangedEventHandler<double> ValueChanged
    {
        add { AddHandler(ValueChangedEvent, value); }
        remove { RemoveHandler(ValueChangedEvent, value); }
    }
    
    public override void OnApplyTemplate()
    {
        base.OnApplyTemplate();
        
        // 이전 이벤트 핸들러 제거
        if (_upButton != null)
            _upButton.Click -= UpButton_Click;
        if (_downButton != null)
            _downButton.Click -= DownButton_Click;
        if (_textBox != null)
            _textBox.TextChanged -= TextBox_TextChanged;
        
        // Template parts 가져오기
        _upButton = GetTemplateChild("PART_UpButton") as Button;
        _downButton = GetTemplateChild("PART_DownButton") as Button;
        _textBox = GetTemplateChild("PART_TextBox") as TextBox;
        
        // 이벤트 핸들러 연결
        if (_upButton != null)
            _upButton.Click += UpButton_Click;
        if (_downButton != null)
            _downButton.Click += DownButton_Click;
        if (_textBox != null)
        {
            _textBox.TextChanged += TextBox_TextChanged;
            _textBox.Text = Value.ToString();
        }
    }
    
    private void UpButton_Click(object sender, RoutedEventArgs e)
    {
        Value = Math.Min(Value + Step, Maximum);
    }
    
    private void DownButton_Click(object sender, RoutedEventArgs e)
    {
        Value = Math.Max(Value - Step, Minimum);
    }
    
    private void TextBox_TextChanged(object sender, TextChangedEventArgs e)
    {
        if (double.TryParse(_textBox.Text, out double newValue))
        {
            Value = newValue;
        }
    }
    
    private static void OnValueChanged(DependencyObject d, 
                                      DependencyPropertyChangedEventArgs e)
    {
        var control = (NumericUpDown)d;
        
        // TextBox 업데이트
        if (control._textBox != null)
            control._textBox.Text = e.NewValue.ToString();
        
        // 이벤트 발생
        var args = new RoutedPropertyChangedEventArgs<double>(
            (double)e.OldValue, 
            (double)e.NewValue, 
            ValueChangedEvent);
        control.RaiseEvent(args);
    }
    
    private static object CoerceValue(DependencyObject d, object value)
    {
        var control = (NumericUpDown)d;
        double newValue = (double)value;
        
        // 범위 제한
        newValue = Math.Max(control.Minimum, Math.Min(control.Maximum, newValue));
        
        return newValue;
    }
}
```

### Generic.xaml 템플릿
```xml
<!-- Themes/Generic.xaml -->
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
                    xmlns:local="clr-namespace:MyApp.Controls">
    
    <Style TargetType="{x:Type local:NumericUpDown}">
        <Setter Property="Template">
            <Setter.Value>
                <ControlTemplate TargetType="{x:Type local:NumericUpDown}">
                    <Border Background="{TemplateBinding Background}"
                            BorderBrush="{TemplateBinding BorderBrush}"
                            BorderThickness="{TemplateBinding BorderThickness}">
                        <Grid>
                            <Grid.ColumnDefinitions>
                                <ColumnDefinition Width="*"/>
                                <ColumnDefinition Width="Auto"/>
                            </Grid.ColumnDefinitions>
                            
                            <TextBox x:Name="PART_TextBox"
                                     Grid.Column="0"
                                     VerticalAlignment="Center"
                                     Margin="5,0"/>
                            
                            <Grid Grid.Column="1">
                                <Grid.RowDefinitions>
                                    <RowDefinition Height="*"/>
                                    <RowDefinition Height="*"/>
                                </Grid.RowDefinitions>
                                
                                <RepeatButton x:Name="PART_UpButton"
                                              Grid.Row="0"
                                              Content="▲"
                                              FontSize="8"/>
                                <RepeatButton x:Name="PART_DownButton"
                                              Grid.Row="1"
                                              Content="▼"
                                              FontSize="8"/>
                            </Grid>
                        </Grid>
                    </Border>
                </ControlTemplate>
            </Setter.Value>
        </Setter>
    </Style>
</ResourceDictionary>
```

## 고급 커스텀 컨트롤

### ColorPicker 컨트롤
```csharp
public class ColorPicker : Control
{
    private Slider _redSlider;
    private Slider _greenSlider;
    private Slider _blueSlider;
    private Rectangle _preview;
    
    static ColorPicker()
    {
        DefaultStyleKeyProperty.OverrideMetadata(
            typeof(ColorPicker), 
            new FrameworkPropertyMetadata(typeof(ColorPicker)));
    }
    
    // SelectedColor 의존성 속성
    public static readonly DependencyProperty SelectedColorProperty =
        DependencyProperty.Register(
            nameof(SelectedColor), 
            typeof(Color), 
            typeof(ColorPicker),
            new FrameworkPropertyMetadata(
                Colors.Black,
                FrameworkPropertyMetadataOptions.BindsTwoWayByDefault,
                OnSelectedColorChanged));
    
    public Color SelectedColor
    {
        get => (Color)GetValue(SelectedColorProperty);
        set => SetValue(SelectedColorProperty, value);
    }
    
    public override void OnApplyTemplate()
    {
        base.OnApplyTemplate();
        
        _redSlider = GetTemplateChild("PART_RedSlider") as Slider;
        _greenSlider = GetTemplateChild("PART_GreenSlider") as Slider;
        _blueSlider = GetTemplateChild("PART_BlueSlider") as Slider;
        _preview = GetTemplateChild("PART_Preview") as Rectangle;
        
        if (_redSlider != null)
            _redSlider.ValueChanged += Slider_ValueChanged;
        if (_greenSlider != null)
            _greenSlider.ValueChanged += Slider_ValueChanged;
        if (_blueSlider != null)
            _blueSlider.ValueChanged += Slider_ValueChanged;
        
        UpdateSlidersFromColor();
    }
    
    private void Slider_ValueChanged(object sender, 
                                   RoutedPropertyChangedEventArgs<double> e)
    {
        if (_redSlider != null && _greenSlider != null && _blueSlider != null)
        {
            SelectedColor = Color.FromRgb(
                (byte)_redSlider.Value,
                (byte)_greenSlider.Value,
                (byte)_blueSlider.Value);
        }
    }
    
    private static void OnSelectedColorChanged(DependencyObject d, 
                                              DependencyPropertyChangedEventArgs e)
    {
        var picker = (ColorPicker)d;
        picker.UpdateSlidersFromColor();
        picker.UpdatePreview();
    }
    
    private void UpdateSlidersFromColor()
    {
        if (_redSlider != null && _greenSlider != null && _blueSlider != null)
        {
            _redSlider.ValueChanged -= Slider_ValueChanged;
            _greenSlider.ValueChanged -= Slider_ValueChanged;
            _blueSlider.ValueChanged -= Slider_ValueChanged;
            
            _redSlider.Value = SelectedColor.R;
            _greenSlider.Value = SelectedColor.G;
            _blueSlider.Value = SelectedColor.B;
            
            _redSlider.ValueChanged += Slider_ValueChanged;
            _greenSlider.ValueChanged += Slider_ValueChanged;
            _blueSlider.ValueChanged += Slider_ValueChanged;
        }
    }
    
    private void UpdatePreview()
    {
        if (_preview != null)
        {
            _preview.Fill = new SolidColorBrush(SelectedColor);
        }
    }
}
```

## Attached Properties

Attached Properties는 다른 요소에 속성을 추가할 수 있게 합니다.

### Attached Property 구현
```csharp
public static class WatermarkService
{
    // Watermark Attached Property
    public static readonly DependencyProperty WatermarkProperty =
        DependencyProperty.RegisterAttached(
            "Watermark",
            typeof(string),
            typeof(WatermarkService),
            new PropertyMetadata(string.Empty, OnWatermarkChanged));
    
    // Getter
    public static string GetWatermark(DependencyObject obj)
    {
        return (string)obj.GetValue(WatermarkProperty);
    }
    
    // Setter
    public static void SetWatermark(DependencyObject obj, string value)
    {
        obj.SetValue(WatermarkProperty, value);
    }
    
    private static void OnWatermarkChanged(DependencyObject d, 
                                         DependencyPropertyChangedEventArgs e)
    {
        if (d is TextBox textBox)
        {
            if (e.OldValue == null)
            {
                textBox.GotFocus += TextBox_GotFocus;
                textBox.LostFocus += TextBox_LostFocus;
                textBox.TextChanged += TextBox_TextChanged;
            }
            
            UpdateWatermark(textBox);
        }
    }
    
    private static void TextBox_GotFocus(object sender, RoutedEventArgs e)
    {
        var textBox = (TextBox)sender;
        if (textBox.Text == GetWatermark(textBox))
        {
            textBox.Text = string.Empty;
            textBox.Foreground = Brushes.Black;
        }
    }
    
    private static void TextBox_LostFocus(object sender, RoutedEventArgs e)
    {
        UpdateWatermark((TextBox)sender);
    }
    
    private static void TextBox_TextChanged(object sender, TextChangedEventArgs e)
    {
        var textBox = (TextBox)sender;
        if (!string.IsNullOrEmpty(textBox.Text) && 
            textBox.Text != GetWatermark(textBox))
        {
            textBox.Foreground = Brushes.Black;
        }
    }
    
    private static void UpdateWatermark(TextBox textBox)
    {
        if (string.IsNullOrEmpty(textBox.Text))
        {
            textBox.Text = GetWatermark(textBox);
            textBox.Foreground = Brushes.Gray;
        }
    }
}
```

### 사용 예제
```xml
<TextBox local:WatermarkService.Watermark="Enter your name..."
         Width="200"
         Height="30"/>
```

## 핵심 개념 정리
- **UserControl**: XAML 기반의 복합 컨트롤
- **CustomControl**: 템플릿 기반의 재사용 가능한 컨트롤
- **의존성 속성**: 데이터 바인딩과 스타일링 지원
- **TemplatePart**: 컨트롤 템플릿의 명명된 요소
- **Generic.xaml**: 기본 컨트롤 템플릿 정의
- **Attached Property**: 다른 요소에 속성 추가