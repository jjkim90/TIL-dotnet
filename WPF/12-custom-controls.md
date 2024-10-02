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

## Attached Behaviors

Attached Behaviors는 XAML에서 컨트롤에 동작을 추가하는 패턴입니다.

### 드래그 가능한 동작
```csharp
public static class DragBehavior
{
    public static readonly DependencyProperty IsDraggableProperty =
        DependencyProperty.RegisterAttached(
            "IsDraggable",
            typeof(bool),
            typeof(DragBehavior),
            new PropertyMetadata(false, OnIsDraggableChanged));
    
    private static readonly DependencyProperty IsDraggingProperty =
        DependencyProperty.RegisterAttached(
            "IsDragging",
            typeof(bool),
            typeof(DragBehavior),
            new PropertyMetadata(false));
    
    private static readonly DependencyProperty ClickPositionProperty =
        DependencyProperty.RegisterAttached(
            "ClickPosition",
            typeof(Point),
            typeof(DragBehavior),
            new PropertyMetadata(new Point()));
    
    public static bool GetIsDraggable(DependencyObject obj)
    {
        return (bool)obj.GetValue(IsDraggableProperty);
    }
    
    public static void SetIsDraggable(DependencyObject obj, bool value)
    {
        obj.SetValue(IsDraggableProperty, value);
    }
    
    private static void OnIsDraggableChanged(DependencyObject d, 
                                            DependencyPropertyChangedEventArgs e)
    {
        var element = d as UIElement;
        if (element == null) return;
        
        if ((bool)e.NewValue)
        {
            element.MouseLeftButtonDown += OnMouseLeftButtonDown;
            element.MouseLeftButtonUp += OnMouseLeftButtonUp;
            element.MouseMove += OnMouseMove;
        }
        else
        {
            element.MouseLeftButtonDown -= OnMouseLeftButtonDown;
            element.MouseLeftButtonUp -= OnMouseLeftButtonUp;
            element.MouseMove -= OnMouseMove;
        }
    }
    
    private static void OnMouseLeftButtonDown(object sender, MouseButtonEventArgs e)
    {
        var element = sender as UIElement;
        var canvas = element.Parent as Canvas;
        
        if (canvas != null)
        {
            element.SetValue(IsDraggingProperty, true);
            element.SetValue(ClickPositionProperty, e.GetPosition(element));
            element.CaptureMouse();
        }
    }
    
    private static void OnMouseMove(object sender, MouseEventArgs e)
    {
        var element = sender as UIElement;
        var canvas = element.Parent as Canvas;
        
        if (canvas != null && (bool)element.GetValue(IsDraggingProperty))
        {
            var currentPosition = e.GetPosition(canvas);
            var clickPosition = (Point)element.GetValue(ClickPositionProperty);
            
            Canvas.SetLeft(element, currentPosition.X - clickPosition.X);
            Canvas.SetTop(element, currentPosition.Y - clickPosition.Y);
        }
    }
    
    private static void OnMouseLeftButtonUp(object sender, MouseButtonEventArgs e)
    {
        var element = sender as UIElement;
        element.SetValue(IsDraggingProperty, false);
        element.ReleaseMouseCapture();
    }
}
```

### 사용 예제
```xml
<Canvas>
    <Rectangle Width="100" Height="100" Fill="Blue"
               local:DragBehavior.IsDraggable="True"
               Canvas.Left="50" Canvas.Top="50"/>
    
    <Ellipse Width="80" Height="80" Fill="Red"
             local:DragBehavior.IsDraggable="True"
             Canvas.Left="200" Canvas.Top="100"/>
</Canvas>
```

## Control Library 프로젝트

### 프로젝트 구조
```
CustomControlLibrary/
├── Properties/
│   └── AssemblyInfo.cs
├── Themes/
│   └── Generic.xaml
├── Controls/
│   ├── NumericUpDown.cs
│   ├── ColorPicker.cs
│   └── RatingControl.cs
├── Converters/
│   └── ColorToHexConverter.cs
└── CustomControlLibrary.csproj
```

### AssemblyInfo.cs 설정
```csharp
[assembly: ThemeInfo(
    ResourceDictionaryLocation.None,
    ResourceDictionaryLocation.SourceAssembly
)]

[assembly: XmlnsDefinition("http://schemas.mycompany.com/wpf/controls", 
                          "CustomControlLibrary.Controls")]
[assembly: XmlnsPrefix("http://schemas.mycompany.com/wpf/controls", "custom")]
```

### RatingControl 예제
```csharp
[TemplatePart(Name = "PART_RatingItemsControl", Type = typeof(ItemsControl))]
public class RatingControl : Control
{
    private ItemsControl _itemsControl;
    
    static RatingControl()
    {
        DefaultStyleKeyProperty.OverrideMetadata(
            typeof(RatingControl), 
            new FrameworkPropertyMetadata(typeof(RatingControl)));
    }
    
    public static readonly DependencyProperty MaximumProperty =
        DependencyProperty.Register(
            nameof(Maximum), 
            typeof(int), 
            typeof(RatingControl),
            new PropertyMetadata(5, OnMaximumChanged));
    
    public static readonly DependencyProperty ValueProperty =
        DependencyProperty.Register(
            nameof(Value), 
            typeof(double), 
            typeof(RatingControl),
            new FrameworkPropertyMetadata(
                0.0,
                FrameworkPropertyMetadataOptions.BindsTwoWayByDefault,
                OnValueChanged,
                CoerceValue));
    
    public static readonly DependencyProperty IsReadOnlyProperty =
        DependencyProperty.Register(
            nameof(IsReadOnly), 
            typeof(bool), 
            typeof(RatingControl),
            new PropertyMetadata(false));
    
    public int Maximum
    {
        get => (int)GetValue(MaximumProperty);
        set => SetValue(MaximumProperty, value);
    }
    
    public double Value
    {
        get => (double)GetValue(ValueProperty);
        set => SetValue(ValueProperty, value);
    }
    
    public bool IsReadOnly
    {
        get => (bool)GetValue(IsReadOnlyProperty);
        set => SetValue(IsReadOnlyProperty, value);
    }
    
    public override void OnApplyTemplate()
    {
        base.OnApplyTemplate();
        
        _itemsControl = GetTemplateChild("PART_RatingItemsControl") as ItemsControl;
        
        UpdateStars();
    }
    
    private static void OnMaximumChanged(DependencyObject d, 
                                        DependencyPropertyChangedEventArgs e)
    {
        var control = (RatingControl)d;
        control.UpdateStars();
    }
    
    private static void OnValueChanged(DependencyObject d, 
                                      DependencyPropertyChangedEventArgs e)
    {
        var control = (RatingControl)d;
        control.UpdateStarsFill();
    }
    
    private static object CoerceValue(DependencyObject d, object value)
    {
        var control = (RatingControl)d;
        double newValue = (double)value;
        return Math.Max(0, Math.Min(control.Maximum, newValue));
    }
    
    private void UpdateStars()
    {
        if (_itemsControl == null) return;
        
        _itemsControl.Items.Clear();
        
        for (int i = 0; i < Maximum; i++)
        {
            var star = new StarItem { Index = i };
            star.MouseLeftButtonDown += Star_MouseLeftButtonDown;
            star.MouseMove += Star_MouseMove;
            _itemsControl.Items.Add(star);
        }
        
        UpdateStarsFill();
    }
    
    private void UpdateStarsFill()
    {
        if (_itemsControl == null) return;
        
        for (int i = 0; i < _itemsControl.Items.Count; i++)
        {
            if (_itemsControl.Items[i] is StarItem star)
            {
                if (i < Math.Floor(Value))
                {
                    star.FillPercentage = 1.0;
                }
                else if (i < Value)
                {
                    star.FillPercentage = Value - i;
                }
                else
                {
                    star.FillPercentage = 0.0;
                }
            }
        }
    }
    
    private void Star_MouseLeftButtonDown(object sender, MouseButtonEventArgs e)
    {
        if (!IsReadOnly && sender is StarItem star)
        {
            Value = star.Index + 1;
        }
    }
    
    private void Star_MouseMove(object sender, MouseEventArgs e)
    {
        if (!IsReadOnly && e.LeftButton == MouseButtonState.Pressed)
        {
            if (sender is StarItem star)
            {
                var position = e.GetPosition(star);
                double percentage = position.X / star.ActualWidth;
                Value = star.Index + percentage;
            }
        }
    }
}

// StarItem 내부 클래스
public class StarItem : Control
{
    static StarItem()
    {
        DefaultStyleKeyProperty.OverrideMetadata(
            typeof(StarItem), 
            new FrameworkPropertyMetadata(typeof(StarItem)));
    }
    
    public static readonly DependencyProperty FillPercentageProperty =
        DependencyProperty.Register(
            nameof(FillPercentage), 
            typeof(double), 
            typeof(StarItem),
            new PropertyMetadata(0.0));
    
    public double FillPercentage
    {
        get => (double)GetValue(FillPercentageProperty);
        set => SetValue(FillPercentageProperty, value);
    }
    
    public int Index { get; set; }
}
```

### RatingControl 템플릿
```xml
<!-- Themes/Generic.xaml -->
<Style TargetType="{x:Type local:RatingControl}">
    <Setter Property="Template">
        <Setter.Value>
            <ControlTemplate TargetType="{x:Type local:RatingControl}">
                <ItemsControl x:Name="PART_RatingItemsControl">
                    <ItemsControl.ItemsPanel>
                        <ItemsPanelTemplate>
                            <StackPanel Orientation="Horizontal"/>
                        </ItemsPanelTemplate>
                    </ItemsControl.ItemsPanel>
                </ItemsControl>
            </ControlTemplate>
        </Setter.Value>
    </Setter>
</Style>

<Style TargetType="{x:Type local:StarItem}">
    <Setter Property="Width" Value="20"/>
    <Setter Property="Height" Value="20"/>
    <Setter Property="Margin" Value="2"/>
    <Setter Property="Template">
        <Setter.Value>
            <ControlTemplate TargetType="{x:Type local:StarItem}">
                <Grid>
                    <!-- 빈 별 -->
                    <Path Data="M12,2 L15,8 L22,9 L17,14 L18,21 L12,18 L6,21 L7,14 L2,9 L9,8 Z"
                          Fill="LightGray"
                          Stretch="Uniform"/>
                    
                    <!-- 채워진 별 -->
                    <Path Data="M12,2 L15,8 L22,9 L17,14 L18,21 L12,18 L6,21 L7,14 L2,9 L9,8 Z"
                          Fill="Gold"
                          Stretch="Uniform">
                        <Path.Clip>
                            <RectangleGeometry>
                                <RectangleGeometry.Rect>
                                    <MultiBinding Converter="{StaticResource FillPercentageToRectConverter}">
                                        <Binding Path="FillPercentage" 
                                               RelativeSource="{RelativeSource TemplatedParent}"/>
                                        <Binding Path="ActualWidth" 
                                               RelativeSource="{RelativeSource TemplatedParent}"/>
                                    </MultiBinding>
                                </RectangleGeometry.Rect>
                            </RectangleGeometry>
                        </Path.Clip>
                    </Path>
                </Grid>
            </ControlTemplate>
        </Setter.Value>
    </Setter>
</Style>
```

## Visual States

Visual States를 사용하여 컨트롤의 다양한 상태를 정의합니다.

### VisualStateManager 사용
```csharp
[TemplateVisualState(GroupName = "CommonStates", Name = "Normal")]
[TemplateVisualState(GroupName = "CommonStates", Name = "MouseOver")]
[TemplateVisualState(GroupName = "CommonStates", Name = "Pressed")]
[TemplateVisualState(GroupName = "CommonStates", Name = "Disabled")]
public class ModernButton : Button
{
    static ModernButton()
    {
        DefaultStyleKeyProperty.OverrideMetadata(
            typeof(ModernButton), 
            new FrameworkPropertyMetadata(typeof(ModernButton)));
    }
    
    protected override void OnMouseEnter(MouseEventArgs e)
    {
        base.OnMouseEnter(e);
        VisualStateManager.GoToState(this, "MouseOver", true);
    }
    
    protected override void OnMouseLeave(MouseEventArgs e)
    {
        base.OnMouseLeave(e);
        VisualStateManager.GoToState(this, "Normal", true);
    }
    
    protected override void OnMouseLeftButtonDown(MouseButtonEventArgs e)
    {
        base.OnMouseLeftButtonDown(e);
        VisualStateManager.GoToState(this, "Pressed", true);
    }
    
    protected override void OnMouseLeftButtonUp(MouseButtonEventArgs e)
    {
        base.OnMouseLeftButtonUp(e);
        VisualStateManager.GoToState(this, "MouseOver", true);
    }
}
```

### Visual State 템플릿
```xml
<Style TargetType="{x:Type local:ModernButton}">
    <Setter Property="Template">
        <Setter.Value>
            <ControlTemplate TargetType="{x:Type local:ModernButton}">
                <Grid x:Name="RootGrid">
                    <VisualStateManager.VisualStateGroups>
                        <VisualStateGroup x:Name="CommonStates">
                            <VisualState x:Name="Normal">
                                <Storyboard>
                                    <ColorAnimation Storyboard.TargetName="BackgroundBorder"
                                                  Storyboard.TargetProperty="(Background).(SolidColorBrush.Color)"
                                                  To="#007ACC" Duration="0:0:0.2"/>
                                </Storyboard>
                            </VisualState>
                            
                            <VisualState x:Name="MouseOver">
                                <Storyboard>
                                    <ColorAnimation Storyboard.TargetName="BackgroundBorder"
                                                  Storyboard.TargetProperty="(Background).(SolidColorBrush.Color)"
                                                  To="#005A9E" Duration="0:0:0.2"/>
                                    <DoubleAnimation Storyboard.TargetName="ScaleTransform"
                                                   Storyboard.TargetProperty="ScaleX"
                                                   To="1.05" Duration="0:0:0.2"/>
                                    <DoubleAnimation Storyboard.TargetName="ScaleTransform"
                                                   Storyboard.TargetProperty="ScaleY"
                                                   To="1.05" Duration="0:0:0.2"/>
                                </Storyboard>
                            </VisualState>
                            
                            <VisualState x:Name="Pressed">
                                <Storyboard>
                                    <ColorAnimation Storyboard.TargetName="BackgroundBorder"
                                                  Storyboard.TargetProperty="(Background).(SolidColorBrush.Color)"
                                                  To="#003D6B" Duration="0:0:0.1"/>
                                    <DoubleAnimation Storyboard.TargetName="ScaleTransform"
                                                   Storyboard.TargetProperty="ScaleX"
                                                   To="0.95" Duration="0:0:0.1"/>
                                    <DoubleAnimation Storyboard.TargetName="ScaleTransform"
                                                   Storyboard.TargetProperty="ScaleY"
                                                   To="0.95" Duration="0:0:0.1"/>
                                </Storyboard>
                            </VisualState>
                            
                            <VisualState x:Name="Disabled">
                                <Storyboard>
                                    <DoubleAnimation Storyboard.TargetName="RootGrid"
                                                   Storyboard.TargetProperty="Opacity"
                                                   To="0.5" Duration="0:0:0.1"/>
                                </Storyboard>
                            </VisualState>
                        </VisualStateGroup>
                    </VisualStateManager.VisualStateGroups>
                    
                    <Border x:Name="BackgroundBorder"
                            Background="#007ACC"
                            CornerRadius="4"
                            RenderTransformOrigin="0.5,0.5">
                        <Border.RenderTransform>
                            <ScaleTransform x:Name="ScaleTransform" ScaleX="1" ScaleY="1"/>
                        </Border.RenderTransform>
                        
                        <ContentPresenter HorizontalAlignment="Center"
                                        VerticalAlignment="Center"
                                        Margin="10,5"/>
                    </Border>
                </Grid>
            </ControlTemplate>
        </Setter.Value>
    </Setter>
</Style>
```

## 실전 예제: 자동 완성 텍스트박스

```csharp
public class AutoCompleteTextBox : TextBox
{
    private ListBox _suggestionListBox;
    private Popup _popup;
    
    static AutoCompleteTextBox()
    {
        DefaultStyleKeyProperty.OverrideMetadata(
            typeof(AutoCompleteTextBox), 
            new FrameworkPropertyMetadata(typeof(AutoCompleteTextBox)));
    }
    
    public static readonly DependencyProperty ItemsSourceProperty =
        DependencyProperty.Register(
            nameof(ItemsSource), 
            typeof(IEnumerable), 
            typeof(AutoCompleteTextBox));
    
    public static readonly DependencyProperty SelectedItemProperty =
        DependencyProperty.Register(
            nameof(SelectedItem), 
            typeof(object), 
            typeof(AutoCompleteTextBox));
    
    public IEnumerable ItemsSource
    {
        get => (IEnumerable)GetValue(ItemsSourceProperty);
        set => SetValue(ItemsSourceProperty, value);
    }
    
    public object SelectedItem
    {
        get => GetValue(SelectedItemProperty);
        set => SetValue(SelectedItemProperty, value);
    }
    
    public override void OnApplyTemplate()
    {
        base.OnApplyTemplate();
        
        _popup = GetTemplateChild("PART_Popup") as Popup;
        _suggestionListBox = GetTemplateChild("PART_ListBox") as ListBox;
        
        if (_suggestionListBox != null)
        {
            _suggestionListBox.SelectionChanged += SuggestionListBox_SelectionChanged;
        }
        
        TextChanged += AutoCompleteTextBox_TextChanged;
        PreviewKeyDown += AutoCompleteTextBox_PreviewKeyDown;
    }
    
    private void AutoCompleteTextBox_TextChanged(object sender, TextChangedEventArgs e)
    {
        if (string.IsNullOrEmpty(Text))
        {
            _popup.IsOpen = false;
            return;
        }
        
        var filteredItems = ItemsSource?.Cast<object>()
            .Where(item => item.ToString().ToLower().Contains(Text.ToLower()))
            .ToList();
        
        if (filteredItems?.Any() == true)
        {
            _suggestionListBox.ItemsSource = filteredItems;
            _popup.IsOpen = true;
        }
        else
        {
            _popup.IsOpen = false;
        }
    }
    
    private void SuggestionListBox_SelectionChanged(object sender, 
                                                   SelectionChangedEventArgs e)
    {
        if (_suggestionListBox.SelectedItem != null)
        {
            SelectedItem = _suggestionListBox.SelectedItem;
            Text = SelectedItem.ToString();
            _popup.IsOpen = false;
            
            // 커서를 텍스트 끝으로 이동
            CaretIndex = Text.Length;
        }
    }
    
    private void AutoCompleteTextBox_PreviewKeyDown(object sender, KeyEventArgs e)
    {
        if (_popup.IsOpen)
        {
            switch (e.Key)
            {
                case Key.Down:
                    _suggestionListBox.SelectedIndex = 
                        Math.Min(_suggestionListBox.SelectedIndex + 1, 
                               _suggestionListBox.Items.Count - 1);
                    e.Handled = true;
                    break;
                    
                case Key.Up:
                    _suggestionListBox.SelectedIndex = 
                        Math.Max(_suggestionListBox.SelectedIndex - 1, 0);
                    e.Handled = true;
                    break;
                    
                case Key.Enter:
                    if (_suggestionListBox.SelectedItem != null)
                    {
                        SuggestionListBox_SelectionChanged(_suggestionListBox, null);
                        e.Handled = true;
                    }
                    break;
                    
                case Key.Escape:
                    _popup.IsOpen = false;
                    e.Handled = true;
                    break;
            }
        }
    }
}
```

## 핵심 개념 정리
- **UserControl**: XAML 기반의 복합 컨트롤
- **CustomControl**: 템플릿 기반의 재사용 가능한 컨트롤
- **의존성 속성**: 데이터 바인딩과 스타일링 지원
- **TemplatePart**: 컨트롤 템플릿의 명명된 요소
- **Generic.xaml**: 기본 컨트롤 템플릿 정의
- **Attached Property**: 다른 요소에 속성 추가
- **Attached Behavior**: XAML에서 동작 추가
- **Visual States**: 컨트롤의 시각적 상태 관리