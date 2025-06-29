# 이벤트와 동작

## WPF 이벤트 시스템

WPF의 이벤트 시스템은 기존 .NET 이벤트를 확장하여 더 강력한 기능을 제공합니다.

### 라우팅 이벤트 (Routed Events)
라우팅 이벤트는 요소 트리를 통해 전파되는 이벤트입니다.

#### 라우팅 전략
- **Bubbling**: 자식에서 부모로 전파
- **Tunneling**: 부모에서 자식으로 전파 (Preview 접두사)
- **Direct**: 일반 .NET 이벤트처럼 동작

## 이벤트 처리

### XAML에서 이벤트 처리
```xml
<Button Name="myButton" 
        Click="Button_Click"
        MouseEnter="Button_MouseEnter"
        MouseLeave="Button_MouseLeave"
        PreviewMouseDown="Button_PreviewMouseDown">
    Click Me
</Button>
```

### 코드비하인드에서 이벤트 처리
```csharp
public partial class MainWindow : Window
{
    public MainWindow()
    {
        InitializeComponent();
        
        // 이벤트 핸들러 등록
        myButton.Click += Button_Click;
        myButton.MouseEnter += Button_MouseEnter;
        
        // 람다 표현식으로 등록
        myButton.MouseLeave += (sender, e) => 
        {
            myButton.Background = Brushes.LightGray;
        };
    }
    
    private void Button_Click(object sender, RoutedEventArgs e)
    {
        MessageBox.Show("Button clicked!");
    }
    
    private void Button_MouseEnter(object sender, MouseEventArgs e)
    {
        var button = sender as Button;
        button.Background = Brushes.LightBlue;
    }
    
    private void Button_PreviewMouseDown(object sender, MouseButtonEventArgs e)
    {
        // Tunneling 이벤트 (Bubbling보다 먼저 발생)
        Debug.WriteLine("Preview MouseDown");
    }
}
```

## 라우팅 이벤트 전파

### Bubbling 예제
```xml
<Border Name="outerBorder" MouseLeftButtonDown="Border_MouseLeftButtonDown"
        Background="Red" Width="300" Height="300">
    <Border Name="middleBorder" MouseLeftButtonDown="Border_MouseLeftButtonDown"
            Background="Green" Width="200" Height="200" Margin="50">
        <Button Name="innerButton" MouseLeftButtonDown="Border_MouseLeftButtonDown"
                Content="Click Me" Width="100" Height="50"/>
    </Border>
</Border>
```

```csharp
private void Border_MouseLeftButtonDown(object sender, MouseButtonEventArgs e)
{
    var element = sender as FrameworkElement;
    Debug.WriteLine($"MouseLeftButtonDown on {element.Name}");
    
    // 이벤트 전파 중지
    if (element.Name == "middleBorder")
    {
        e.Handled = true;
    }
}
```

### Tunneling과 Bubbling 순서
```xml
<StackPanel PreviewMouseDown="StackPanel_PreviewMouseDown"
            MouseDown="StackPanel_MouseDown">
    <TextBlock Text="Parent"/>
    <Button PreviewMouseDown="Button_PreviewMouseDown"
            MouseDown="Button_MouseDown"
            Content="Child Button"/>
</StackPanel>
```

```csharp
// 실행 순서:
// 1. StackPanel_PreviewMouseDown (Tunneling)
// 2. Button_PreviewMouseDown (Tunneling)
// 3. Button_MouseDown (Bubbling)
// 4. StackPanel_MouseDown (Bubbling)
```

## 사용자 정의 라우팅 이벤트

### 라우팅 이벤트 정의
```csharp
public class CustomControl : Control
{
    // 라우팅 이벤트 등록
    public static readonly RoutedEvent CustomClickEvent = 
        EventManager.RegisterRoutedEvent(
            "CustomClick", 
            RoutingStrategy.Bubble,
            typeof(RoutedEventHandler), 
            typeof(CustomControl));
    
    // CLR 이벤트 래퍼
    public event RoutedEventHandler CustomClick
    {
        add { AddHandler(CustomClickEvent, value); }
        remove { RemoveHandler(CustomClickEvent, value); }
    }
    
    // 이벤트 발생
    protected virtual void OnCustomClick()
    {
        RoutedEventArgs args = new RoutedEventArgs(CustomClickEvent);
        RaiseEvent(args);
    }
    
    // 사용자 정의 이벤트 인수
    public static readonly RoutedEvent ValueChangedEvent = 
        EventManager.RegisterRoutedEvent(
            "ValueChanged",
            RoutingStrategy.Bubble,
            typeof(RoutedPropertyChangedEventHandler<double>),
            typeof(CustomControl));
    
    public event RoutedPropertyChangedEventHandler<double> ValueChanged
    {
        add { AddHandler(ValueChangedEvent, value); }
        remove { RemoveHandler(ValueChangedEvent, value); }
    }
    
    protected virtual void OnValueChanged(double oldValue, double newValue)
    {
        var args = new RoutedPropertyChangedEventArgs<double>(
            oldValue, newValue, ValueChangedEvent);
        RaiseEvent(args);
    }
}
```

### 사용자 정의 이벤트 인수
```csharp
public class CustomEventArgs : RoutedEventArgs
{
    public string Data { get; set; }
    public DateTime Timestamp { get; set; }
    
    public CustomEventArgs(RoutedEvent routedEvent, string data) 
        : base(routedEvent)
    {
        Data = data;
        Timestamp = DateTime.Now;
    }
}

public delegate void CustomEventHandler(object sender, CustomEventArgs e);
```

## 명령 (Commands)

### 기본 명령 사용
```xml
<Window.CommandBindings>
    <CommandBinding Command="ApplicationCommands.Open"
                    Executed="OpenCommand_Executed"
                    CanExecute="OpenCommand_CanExecute"/>
    <CommandBinding Command="ApplicationCommands.Save"
                    Executed="SaveCommand_Executed"
                    CanExecute="SaveCommand_CanExecute"/>
</Window.CommandBindings>

<ToolBar>
    <Button Command="ApplicationCommands.Open" Content="Open"/>
    <Button Command="ApplicationCommands.Save" Content="Save"/>
    <Separator/>
    <Button Command="ApplicationCommands.Cut"/>
    <Button Command="ApplicationCommands.Copy"/>
    <Button Command="ApplicationCommands.Paste"/>
</ToolBar>
```

```csharp
private void OpenCommand_Executed(object sender, ExecutedRoutedEventArgs e)
{
    var dialog = new OpenFileDialog();
    if (dialog.ShowDialog() == true)
    {
        // 파일 열기 로직
    }
}

private void OpenCommand_CanExecute(object sender, CanExecuteRoutedEventArgs e)
{
    e.CanExecute = true;
}

private void SaveCommand_Executed(object sender, ExecutedRoutedEventArgs e)
{
    // 저장 로직
}

private void SaveCommand_CanExecute(object sender, CanExecuteRoutedEventArgs e)
{
    e.CanExecute = HasUnsavedChanges;
}
```

### 사용자 정의 명령
```csharp
public static class CustomCommands
{
    public static readonly RoutedUICommand Exit = new RoutedUICommand(
        "Exit",
        "Exit",
        typeof(CustomCommands),
        new InputGestureCollection()
        {
            new KeyGesture(Key.F4, ModifierKeys.Alt)
        });
    
    public static readonly RoutedUICommand About = new RoutedUICommand(
        "About",
        "About",
        typeof(CustomCommands));
}
```

## Attached Events

### Attached Event 정의
```csharp
public static class DragDropHelper
{
    public static readonly RoutedEvent ItemDroppedEvent = 
        EventManager.RegisterRoutedEvent(
            "ItemDropped",
            RoutingStrategy.Bubble,
            typeof(RoutedEventHandler),
            typeof(DragDropHelper));
    
    public static void AddItemDroppedHandler(DependencyObject d, RoutedEventHandler handler)
    {
        UIElement uie = d as UIElement;
        if (uie != null)
        {
            uie.AddHandler(ItemDroppedEvent, handler);
        }
    }
    
    public static void RemoveItemDroppedHandler(DependencyObject d, RoutedEventHandler handler)
    {
        UIElement uie = d as UIElement;
        if (uie != null)
        {
            uie.RemoveHandler(ItemDroppedEvent, handler);
        }
    }
    
    // 이벤트 발생 헬퍼
    public static void RaiseItemDropped(UIElement target)
    {
        target.RaiseEvent(new RoutedEventArgs(ItemDroppedEvent));
    }
}
```

### XAML에서 Attached Event 사용
```xml
<Grid local:DragDropHelper.ItemDropped="Grid_ItemDropped">
    <!-- 내용 -->
</Grid>
```

## WeakEvent 패턴

### WeakEventManager 구현
```csharp
public class PropertyChangedEventManager : WeakEventManager
{
    private PropertyChangedEventManager() { }
    
    public static void AddHandler(INotifyPropertyChanged source, 
                                  EventHandler<PropertyChangedEventArgs> handler)
    {
        if (source == null)
            throw new ArgumentNullException(nameof(source));
        if (handler == null)
            throw new ArgumentNullException(nameof(handler));
        
        CurrentManager.ProtectedAddHandler(source, handler);
    }
    
    public static void RemoveHandler(INotifyPropertyChanged source,
                                     EventHandler<PropertyChangedEventArgs> handler)
    {
        if (source == null)
            throw new ArgumentNullException(nameof(source));
        if (handler == null)
            throw new ArgumentNullException(nameof(handler));
        
        CurrentManager.ProtectedRemoveHandler(source, handler);
    }
    
    private static PropertyChangedEventManager CurrentManager
    {
        get
        {
            Type managerType = typeof(PropertyChangedEventManager);
            PropertyChangedEventManager manager = 
                (PropertyChangedEventManager)GetCurrentManager(managerType);
            
            if (manager == null)
            {
                manager = new PropertyChangedEventManager();
                SetCurrentManager(managerType, manager);
            }
            
            return manager;
        }
    }
    
    protected override void StartListening(object source)
    {
        INotifyPropertyChanged typedSource = (INotifyPropertyChanged)source;
        typedSource.PropertyChanged += OnPropertyChanged;
    }
    
    protected override void StopListening(object source)
    {
        INotifyPropertyChanged typedSource = (INotifyPropertyChanged)source;
        typedSource.PropertyChanged -= OnPropertyChanged;
    }
    
    private void OnPropertyChanged(object sender, PropertyChangedEventArgs e)
    {
        DeliverEvent(sender, e);
    }
}
```

### WeakEvent 사용
```csharp
public class DataConsumer
{
    private DataProvider _provider;
    
    public DataConsumer(DataProvider provider)
    {
        _provider = provider;
        
        // 강한 참조 (메모리 누수 위험)
        // _provider.PropertyChanged += Provider_PropertyChanged;
        
        // 약한 참조 (안전)
        PropertyChangedEventManager.AddHandler(_provider, Provider_PropertyChanged);
    }
    
    private void Provider_PropertyChanged(object sender, PropertyChangedEventArgs e)
    {
        // 속성 변경 처리
    }
    
    public void Cleanup()
    {
        PropertyChangedEventManager.RemoveHandler(_provider, Provider_PropertyChanged);
    }
}
```

## Behavior (System.Windows.Interactivity)

### 기본 Behavior 구현
```csharp
using System.Windows.Interactivity;

public class DragBehavior : Behavior<UIElement>
{
    private Point _mousePosition;
    private bool _isDragging;
    
    protected override void OnAttached()
    {
        base.OnAttached();
        
        AssociatedObject.MouseLeftButtonDown += OnMouseLeftButtonDown;
        AssociatedObject.MouseLeftButtonUp += OnMouseLeftButtonUp;
        AssociatedObject.MouseMove += OnMouseMove;
    }
    
    protected override void OnDetaching()
    {
        base.OnDetaching();
        
        AssociatedObject.MouseLeftButtonDown -= OnMouseLeftButtonDown;
        AssociatedObject.MouseLeftButtonUp -= OnMouseLeftButtonUp;
        AssociatedObject.MouseMove -= OnMouseMove;
    }
    
    private void OnMouseLeftButtonDown(object sender, MouseButtonEventArgs e)
    {
        _isDragging = true;
        _mousePosition = e.GetPosition(null);
        AssociatedObject.CaptureMouse();
    }
    
    private void OnMouseMove(object sender, MouseEventArgs e)
    {
        if (_isDragging)
        {
            var currentPosition = e.GetPosition(null);
            var deltaX = currentPosition.X - _mousePosition.X;
            var deltaY = currentPosition.Y - _mousePosition.Y;
            
            if (AssociatedObject is FrameworkElement element)
            {
                Canvas.SetLeft(element, Canvas.GetLeft(element) + deltaX);
                Canvas.SetTop(element, Canvas.GetTop(element) + deltaY);
            }
            
            _mousePosition = currentPosition;
        }
    }
    
    private void OnMouseLeftButtonUp(object sender, MouseButtonEventArgs e)
    {
        _isDragging = false;
        AssociatedObject.ReleaseMouseCapture();
    }
}
```

### XAML에서 Behavior 사용
```xml
<Window xmlns:i="http://schemas.microsoft.com/expression/2010/interactivity">
    <Canvas>
        <Rectangle Width="100" Height="100" Fill="Blue"
                   Canvas.Left="50" Canvas.Top="50">
            <i:Interaction.Behaviors>
                <local:DragBehavior/>
            </i:Interaction.Behaviors>
        </Rectangle>
    </Canvas>
</Window>
```

### 속성이 있는 Behavior
```csharp
public class AutoScrollBehavior : Behavior<ScrollViewer>
{
    public static readonly DependencyProperty ScrollSpeedProperty =
        DependencyProperty.Register(
            nameof(ScrollSpeed),
            typeof(double),
            typeof(AutoScrollBehavior),
            new PropertyMetadata(1.0));
    
    public double ScrollSpeed
    {
        get => (double)GetValue(ScrollSpeedProperty);
        set => SetValue(ScrollSpeedProperty, value);
    }
    
    private DispatcherTimer _timer;
    
    protected override void OnAttached()
    {
        base.OnAttached();
        
        _timer = new DispatcherTimer
        {
            Interval = TimeSpan.FromMilliseconds(50)
        };
        _timer.Tick += OnTimerTick;
        _timer.Start();
    }
    
    protected override void OnDetaching()
    {
        base.OnDetaching();
        
        _timer?.Stop();
        _timer = null;
    }
    
    private void OnTimerTick(object sender, EventArgs e)
    {
        if (AssociatedObject != null)
        {
            AssociatedObject.ScrollToVerticalOffset(
                AssociatedObject.VerticalOffset + ScrollSpeed);
        }
    }
}
```

## Trigger와 Action

### EventTrigger와 Action
```xml
<Button Content="Click Me">
    <i:Interaction.Triggers>
        <i:EventTrigger EventName="Click">
            <i:InvokeCommandAction Command="{Binding ClickCommand}"/>
        </i:EventTrigger>
        
        <i:EventTrigger EventName="MouseEnter">
            <local:ChangePropertyAction 
                PropertyName="Background" 
                Value="LightBlue"/>
        </i:EventTrigger>
    </i:Interaction.Triggers>
</Button>
```

### 사용자 정의 Action
```csharp
public class ChangePropertyAction : TriggerAction<FrameworkElement>
{
    public static readonly DependencyProperty PropertyNameProperty =
        DependencyProperty.Register(
            nameof(PropertyName),
            typeof(string),
            typeof(ChangePropertyAction));
    
    public static readonly DependencyProperty ValueProperty =
        DependencyProperty.Register(
            nameof(Value),
            typeof(object),
            typeof(ChangePropertyAction));
    
    public string PropertyName
    {
        get => (string)GetValue(PropertyNameProperty);
        set => SetValue(PropertyNameProperty, value);
    }
    
    public object Value
    {
        get => GetValue(ValueProperty);
        set => SetValue(ValueProperty, value);
    }
    
    protected override void Invoke(object parameter)
    {
        if (AssociatedObject == null || string.IsNullOrEmpty(PropertyName))
            return;
        
        var propertyInfo = AssociatedObject.GetType().GetProperty(PropertyName);
        if (propertyInfo != null && propertyInfo.CanWrite)
        {
            propertyInfo.SetValue(AssociatedObject, Value);
        }
    }
}
```

### DataTrigger Behavior
```xml
<TextBox Text="{Binding SearchText}">
    <i:Interaction.Triggers>
        <i:DataTrigger Binding="{Binding RelativeSource={RelativeSource Self}, 
                                 Path=Text.Length}" 
                       Comparison="GreaterThan" 
                       Value="3">
            <i:InvokeCommandAction Command="{Binding SearchCommand}"/>
        </i:DataTrigger>
    </i:Interaction.Triggers>
</TextBox>
```

## 이벤트와 MVVM

### EventToCommand
```csharp
public class EventToCommand : TriggerAction<DependencyObject>
{
    public static readonly DependencyProperty CommandProperty =
        DependencyProperty.Register(
            nameof(Command),
            typeof(ICommand),
            typeof(EventToCommand));
    
    public static readonly DependencyProperty CommandParameterProperty =
        DependencyProperty.Register(
            nameof(CommandParameter),
            typeof(object),
            typeof(EventToCommand));
    
    public static readonly DependencyProperty PassEventArgsToCommandProperty =
        DependencyProperty.Register(
            nameof(PassEventArgsToCommand),
            typeof(bool),
            typeof(EventToCommand),
            new PropertyMetadata(false));
    
    public ICommand Command
    {
        get => (ICommand)GetValue(CommandProperty);
        set => SetValue(CommandProperty, value);
    }
    
    public object CommandParameter
    {
        get => GetValue(CommandParameterProperty);
        set => SetValue(CommandParameterProperty, value);
    }
    
    public bool PassEventArgsToCommand
    {
        get => (bool)GetValue(PassEventArgsToCommandProperty);
        set => SetValue(PassEventArgsToCommandProperty, value);
    }
    
    protected override void Invoke(object parameter)
    {
        if (Command == null)
            return;
        
        object commandParameter = CommandParameter;
        
        if (PassEventArgsToCommand && parameter is EventArgs)
        {
            commandParameter = parameter;
        }
        
        if (Command.CanExecute(commandParameter))
        {
            Command.Execute(commandParameter);
        }
    }
}
```

### XAML에서 EventToCommand 사용
```xml
<ListBox ItemsSource="{Binding Items}">
    <i:Interaction.Triggers>
        <i:EventTrigger EventName="SelectionChanged">
            <local:EventToCommand 
                Command="{Binding SelectionChangedCommand}"
                PassEventArgsToCommand="True"/>
        </i:EventTrigger>
    </i:Interaction.Triggers>
</ListBox>
```

## 고급 이벤트 처리

### 이벤트 필터링
```csharp
public class FilteredEventBehavior : Behavior<UIElement>
{
    public static readonly DependencyProperty FilterProperty =
        DependencyProperty.Register(
            nameof(Filter),
            typeof(Func<EventArgs, bool>),
            typeof(FilteredEventBehavior));
    
    public Func<EventArgs, bool> Filter
    {
        get => (Func<EventArgs, bool>)GetValue(FilterProperty);
        set => SetValue(FilterProperty, value);
    }
    
    protected override void OnAttached()
    {
        base.OnAttached();
        AssociatedObject.PreviewMouseDown += OnPreviewMouseDown;
    }
    
    private void OnPreviewMouseDown(object sender, MouseButtonEventArgs e)
    {
        if (Filter != null && !Filter(e))
        {
            e.Handled = true;
        }
    }
}
```

### 이벤트 집계
```csharp
public class EventAggregator
{
    private readonly Dictionary<Type, List<WeakReference>> _eventSubscribers = 
        new Dictionary<Type, List<WeakReference>>();
    
    public void Subscribe<TEvent>(Action<TEvent> handler)
    {
        var eventType = typeof(TEvent);
        
        if (!_eventSubscribers.ContainsKey(eventType))
        {
            _eventSubscribers[eventType] = new List<WeakReference>();
        }
        
        _eventSubscribers[eventType].Add(new WeakReference(handler));
    }
    
    public void Publish<TEvent>(TEvent eventData)
    {
        var eventType = typeof(TEvent);
        
        if (_eventSubscribers.ContainsKey(eventType))
        {
            var subscribers = _eventSubscribers[eventType];
            var deadSubscribers = new List<WeakReference>();
            
            foreach (var weakRef in subscribers)
            {
                if (weakRef.IsAlive)
                {
                    var handler = weakRef.Target as Action<TEvent>;
                    handler?.Invoke(eventData);
                }
                else
                {
                    deadSubscribers.Add(weakRef);
                }
            }
            
            // 죽은 참조 제거
            foreach (var deadRef in deadSubscribers)
            {
                subscribers.Remove(deadRef);
            }
        }
    }
}
```

## 제스처 인식

### 기본 제스처
```xml
<Window>
    <Window.InputBindings>
        <KeyBinding Key="S" Modifiers="Control" 
                    Command="{Binding SaveCommand}"/>
        <KeyBinding Key="O" Modifiers="Control" 
                    Command="{Binding OpenCommand}"/>
        <MouseBinding Gesture="LeftDoubleClick" 
                      Command="{Binding DoubleClickCommand}"/>
    </Window.InputBindings>
    
    <TextBox>
        <TextBox.InputBindings>
            <KeyBinding Key="Enter" 
                        Command="{Binding SubmitCommand}"/>
        </TextBox.InputBindings>
    </TextBox>
</Window>
```

### 사용자 정의 제스처
```csharp
public class SwipeGesture : MouseGesture
{
    public SwipeDirection Direction { get; set; }
    public double MinimumLength { get; set; } = 50;
    
    private Point? _startPoint;
    
    public override bool Matches(object targetElement, InputEventArgs inputEventArgs)
    {
        if (inputEventArgs is MouseEventArgs mouseArgs)
        {
            var currentPoint = mouseArgs.GetPosition(targetElement as IInputElement);
            
            if (mouseArgs.LeftButton == MouseButtonState.Pressed)
            {
                if (_startPoint == null)
                {
                    _startPoint = currentPoint;
                }
                else
                {
                    var deltaX = currentPoint.X - _startPoint.Value.X;
                    var deltaY = currentPoint.Y - _startPoint.Value.Y;
                    var length = Math.Sqrt(deltaX * deltaX + deltaY * deltaY);
                    
                    if (length >= MinimumLength)
                    {
                        switch (Direction)
                        {
                            case SwipeDirection.Left:
                                return deltaX < -MinimumLength;
                            case SwipeDirection.Right:
                                return deltaX > MinimumLength;
                            case SwipeDirection.Up:
                                return deltaY < -MinimumLength;
                            case SwipeDirection.Down:
                                return deltaY > MinimumLength;
                        }
                    }
                }
            }
            else
            {
                _startPoint = null;
            }
        }
        
        return false;
    }
}

public enum SwipeDirection
{
    Left, Right, Up, Down
}
```

## 실전 예제: 드래그 앤 드롭

```csharp
public class DragDropBehavior : Behavior<FrameworkElement>
{
    public static readonly DependencyProperty IsDragSourceProperty =
        DependencyProperty.Register(
            nameof(IsDragSource),
            typeof(bool),
            typeof(DragDropBehavior),
            new PropertyMetadata(false));
    
    public static readonly DependencyProperty IsDropTargetProperty =
        DependencyProperty.Register(
            nameof(IsDropTarget),
            typeof(bool),
            typeof(DragDropBehavior),
            new PropertyMetadata(false));
    
    public static readonly DependencyProperty DropCommandProperty =
        DependencyProperty.Register(
            nameof(DropCommand),
            typeof(ICommand),
            typeof(DragDropBehavior));
    
    public bool IsDragSource
    {
        get => (bool)GetValue(IsDragSourceProperty);
        set => SetValue(IsDragSourceProperty, value);
    }
    
    public bool IsDropTarget
    {
        get => (bool)GetValue(IsDropTargetProperty);
        set => SetValue(IsDropTargetProperty, value);
    }
    
    public ICommand DropCommand
    {
        get => (ICommand)GetValue(DropCommandProperty);
        set => SetValue(DropCommandProperty, value);
    }
    
    protected override void OnAttached()
    {
        base.OnAttached();
        
        if (IsDragSource)
        {
            AssociatedObject.MouseMove += OnMouseMove;
        }
        
        if (IsDropTarget)
        {
            AssociatedObject.AllowDrop = true;
            AssociatedObject.DragOver += OnDragOver;
            AssociatedObject.Drop += OnDrop;
        }
    }
    
    private void OnMouseMove(object sender, MouseEventArgs e)
    {
        if (e.LeftButton == MouseButtonState.Pressed)
        {
            var data = new DataObject();
            data.SetData("DraggedItem", AssociatedObject.DataContext);
            
            DragDrop.DoDragDrop(AssociatedObject, data, DragDropEffects.Move);
        }
    }
    
    private void OnDragOver(object sender, DragEventArgs e)
    {
        if (e.Data.GetDataPresent("DraggedItem"))
        {
            e.Effects = DragDropEffects.Move;
        }
        else
        {
            e.Effects = DragDropEffects.None;
        }
        
        e.Handled = true;
    }
    
    private void OnDrop(object sender, DragEventArgs e)
    {
        if (e.Data.GetDataPresent("DraggedItem"))
        {
            var droppedItem = e.Data.GetData("DraggedItem");
            
            if (DropCommand?.CanExecute(droppedItem) == true)
            {
                DropCommand.Execute(droppedItem);
            }
        }
        
        e.Handled = true;
    }
}
```

```xml
<!-- 사용 예제 -->
<ListBox ItemsSource="{Binding SourceItems}">
    <ListBox.ItemTemplate>
        <DataTemplate>
            <Border Background="LightBlue" Margin="2" Padding="5">
                <i:Interaction.Behaviors>
                    <local:DragDropBehavior IsDragSource="True"/>
                </i:Interaction.Behaviors>
                <TextBlock Text="{Binding Name}"/>
            </Border>
        </DataTemplate>
    </ListBox.ItemTemplate>
</ListBox>

<ListBox ItemsSource="{Binding TargetItems}">
    <i:Interaction.Behaviors>
        <local:DragDropBehavior 
            IsDropTarget="True"
            DropCommand="{Binding AddItemCommand}"/>
    </i:Interaction.Behaviors>
</ListBox>
```

## 실전 예제: 터치 제스처

```csharp
public class TouchGestureBehavior : Behavior<UIElement>
{
    private readonly Dictionary<int, Point> _touchPoints = new Dictionary<int, Point>();
    private double _lastPinchDistance;
    
    public event EventHandler<double> Pinch;
    public event EventHandler<Vector> Pan;
    public event EventHandler Tap;
    
    protected override void OnAttached()
    {
        base.OnAttached();
        
        AssociatedObject.TouchDown += OnTouchDown;
        AssociatedObject.TouchMove += OnTouchMove;
        AssociatedObject.TouchUp += OnTouchUp;
    }
    
    private void OnTouchDown(object sender, TouchEventArgs e)
    {
        var touchPoint = e.GetTouchPoint(AssociatedObject);
        _touchPoints[e.TouchDevice.Id] = touchPoint.Position;
        
        e.Handled = true;
    }
    
    private void OnTouchMove(object sender, TouchEventArgs e)
    {
        var touchPoint = e.GetTouchPoint(AssociatedObject);
        
        if (_touchPoints.Count == 1)
        {
            // Pan 제스처
            var oldPoint = _touchPoints[e.TouchDevice.Id];
            var delta = new Vector(
                touchPoint.Position.X - oldPoint.X,
                touchPoint.Position.Y - oldPoint.Y);
            
            Pan?.Invoke(this, delta);
        }
        else if (_touchPoints.Count == 2)
        {
            // Pinch 제스처
            var points = _touchPoints.Values.ToArray();
            var currentDistance = (points[0] - points[1]).Length;
            
            if (_lastPinchDistance > 0)
            {
                var scale = currentDistance / _lastPinchDistance;
                Pinch?.Invoke(this, scale);
            }
            
            _lastPinchDistance = currentDistance;
        }
        
        _touchPoints[e.TouchDevice.Id] = touchPoint.Position;
        e.Handled = true;
    }
    
    private void OnTouchUp(object sender, TouchEventArgs e)
    {
        _touchPoints.Remove(e.TouchDevice.Id);
        
        if (_touchPoints.Count == 0)
        {
            _lastPinchDistance = 0;
            
            // Tap 감지
            var touchPoint = e.GetTouchPoint(AssociatedObject);
            if (e.TouchDevice.GetIntermediateTouchPoints(AssociatedObject).Count <= 3)
            {
                Tap?.Invoke(this, EventArgs.Empty);
            }
        }
        
        e.Handled = true;
    }
}
```

## 마우스 캡처와 포커스

### 마우스 캡처
```csharp
public class MouseCaptureBehavior : Behavior<FrameworkElement>
{
    private bool _isCapturing;
    private Point _capturePoint;
    
    protected override void OnAttached()
    {
        base.OnAttached();
        
        AssociatedObject.MouseLeftButtonDown += OnMouseLeftButtonDown;
        AssociatedObject.MouseLeftButtonUp += OnMouseLeftButtonUp;
        AssociatedObject.MouseMove += OnMouseMove;
        AssociatedObject.LostMouseCapture += OnLostMouseCapture;
    }
    
    private void OnMouseLeftButtonDown(object sender, MouseButtonEventArgs e)
    {
        _isCapturing = true;
        _capturePoint = e.GetPosition(AssociatedObject);
        AssociatedObject.CaptureMouse();
        
        // 시각적 피드백
        AssociatedObject.Opacity = 0.7;
    }
    
    private void OnMouseMove(object sender, MouseEventArgs e)
    {
        if (_isCapturing)
        {
            var currentPoint = e.GetPosition(AssociatedObject);
            var distance = Math.Sqrt(
                Math.Pow(currentPoint.X - _capturePoint.X, 2) +
                Math.Pow(currentPoint.Y - _capturePoint.Y, 2));
            
            // 드래그 거리에 따른 처리
            if (distance > 5)
            {
                // 드래그 시작
                OnDragStarted();
            }
        }
    }
    
    private void OnMouseLeftButtonUp(object sender, MouseButtonEventArgs e)
    {
        if (_isCapturing)
        {
            _isCapturing = false;
            AssociatedObject.ReleaseMouseCapture();
            AssociatedObject.Opacity = 1.0;
        }
    }
    
    private void OnLostMouseCapture(object sender, MouseEventArgs e)
    {
        _isCapturing = false;
        AssociatedObject.Opacity = 1.0;
    }
    
    private void OnDragStarted()
    {
        // 드래그 시작 로직
    }
}
```

### 포커스 관리
```csharp
public class FocusManagerBehavior : Behavior<Control>
{
    public static readonly DependencyProperty FocusOnLoadProperty =
        DependencyProperty.Register(
            nameof(FocusOnLoad),
            typeof(bool),
            typeof(FocusManagerBehavior),
            new PropertyMetadata(false));
    
    public static readonly DependencyProperty SelectAllOnFocusProperty =
        DependencyProperty.Register(
            nameof(SelectAllOnFocus),
            typeof(bool),
            typeof(FocusManagerBehavior),
            new PropertyMetadata(false));
    
    public bool FocusOnLoad
    {
        get => (bool)GetValue(FocusOnLoadProperty);
        set => SetValue(FocusOnLoadProperty, value);
    }
    
    public bool SelectAllOnFocus
    {
        get => (bool)GetValue(SelectAllOnFocusProperty);
        set => SetValue(SelectAllOnFocusProperty, value);
    }
    
    protected override void OnAttached()
    {
        base.OnAttached();
        
        if (FocusOnLoad)
        {
            AssociatedObject.Loaded += OnLoaded;
        }
        
        if (SelectAllOnFocus && AssociatedObject is TextBox)
        {
            AssociatedObject.GotFocus += OnGotFocus;
        }
        
        // 포커스 추적
        AssociatedObject.GotFocus += OnFocusChanged;
        AssociatedObject.LostFocus += OnFocusChanged;
    }
    
    private void OnLoaded(object sender, RoutedEventArgs e)
    {
        AssociatedObject.Focus();
        Keyboard.Focus(AssociatedObject);
    }
    
    private void OnGotFocus(object sender, RoutedEventArgs e)
    {
        if (sender is TextBox textBox)
        {
            textBox.SelectAll();
        }
    }
    
    private void OnFocusChanged(object sender, RoutedEventArgs e)
    {
        Debug.WriteLine($"Focus changed: {e.RoutedEvent.Name} on {AssociatedObject.Name}");
    }
}
```

## 입력 검증 Behavior

### 실시간 입력 검증
```csharp
public class ValidationBehavior : Behavior<TextBox>
{
    public static readonly DependencyProperty ValidationRuleProperty =
        DependencyProperty.Register(
            nameof(ValidationRule),
            typeof(IValidationRule),
            typeof(ValidationBehavior));
    
    public static readonly DependencyProperty ErrorTemplateProperty =
        DependencyProperty.Register(
            nameof(ErrorTemplate),
            typeof(DataTemplate),
            typeof(ValidationBehavior));
    
    public IValidationRule ValidationRule
    {
        get => (IValidationRule)GetValue(ValidationRuleProperty);
        set => SetValue(ValidationRuleProperty, value);
    }
    
    public DataTemplate ErrorTemplate
    {
        get => (DataTemplate)GetValue(ErrorTemplateProperty);
        set => SetValue(ErrorTemplateProperty, value);
    }
    
    private Popup _errorPopup;
    private ContentControl _errorContent;
    
    protected override void OnAttached()
    {
        base.OnAttached();
        
        AssociatedObject.TextChanged += OnTextChanged;
        AssociatedObject.LostFocus += OnLostFocus;
        
        CreateErrorPopup();
    }
    
    private void CreateErrorPopup()
    {
        _errorPopup = new Popup
        {
            PlacementTarget = AssociatedObject,
            Placement = PlacementMode.Bottom,
            StaysOpen = false
        };
        
        _errorContent = new ContentControl
        {
            ContentTemplate = ErrorTemplate
        };
        
        _errorPopup.Child = _errorContent;
    }
    
    private void OnTextChanged(object sender, TextChangedEventArgs e)
    {
        Validate();
    }
    
    private void OnLostFocus(object sender, RoutedEventArgs e)
    {
        if (!Validate())
        {
            // 포커스 유지
            AssociatedObject.Focus();
        }
    }
    
    private bool Validate()
    {
        if (ValidationRule == null)
            return true;
        
        var result = ValidationRule.Validate(AssociatedObject.Text);
        
        if (result.IsValid)
        {
            AssociatedObject.BorderBrush = Brushes.Gray;
            _errorPopup.IsOpen = false;
        }
        else
        {
            AssociatedObject.BorderBrush = Brushes.Red;
            _errorContent.Content = result.ErrorMessage;
            _errorPopup.IsOpen = true;
        }
        
        return result.IsValid;
    }
}

public interface IValidationRule
{
    ValidationResult Validate(string value);
}

public class ValidationResult
{
    public bool IsValid { get; set; }
    public string ErrorMessage { get; set; }
}
```

## 애니메이션 Behavior

### 호버 애니메이션
```csharp
public class HoverAnimationBehavior : Behavior<FrameworkElement>
{
    public static readonly DependencyProperty ScaleProperty =
        DependencyProperty.Register(
            nameof(Scale),
            typeof(double),
            typeof(HoverAnimationBehavior),
            new PropertyMetadata(1.1));
    
    public static readonly DependencyProperty DurationProperty =
        DependencyProperty.Register(
            nameof(Duration),
            typeof(Duration),
            typeof(HoverAnimationBehavior),
            new PropertyMetadata(new Duration(TimeSpan.FromMilliseconds(200))));
    
    public double Scale
    {
        get => (double)GetValue(ScaleProperty);
        set => SetValue(ScaleProperty, value);
    }
    
    public Duration Duration
    {
        get => (Duration)GetValue(DurationProperty);
        set => SetValue(DurationProperty, value);
    }
    
    private ScaleTransform _scaleTransform;
    private Storyboard _mouseEnterStoryboard;
    private Storyboard _mouseLeaveStoryboard;
    
    protected override void OnAttached()
    {
        base.OnAttached();
        
        // Transform 설정
        _scaleTransform = new ScaleTransform(1, 1);
        AssociatedObject.RenderTransform = _scaleTransform;
        AssociatedObject.RenderTransformOrigin = new Point(0.5, 0.5);
        
        // 애니메이션 생성
        CreateAnimations();
        
        // 이벤트 연결
        AssociatedObject.MouseEnter += OnMouseEnter;
        AssociatedObject.MouseLeave += OnMouseLeave;
    }
    
    private void CreateAnimations()
    {
        // Mouse Enter 애니메이션
        _mouseEnterStoryboard = new Storyboard();
        
        var scaleXAnimation = new DoubleAnimation(Scale, Duration);
        Storyboard.SetTarget(scaleXAnimation, _scaleTransform);
        Storyboard.SetTargetProperty(scaleXAnimation, 
            new PropertyPath(ScaleTransform.ScaleXProperty));
        
        var scaleYAnimation = new DoubleAnimation(Scale, Duration);
        Storyboard.SetTarget(scaleYAnimation, _scaleTransform);
        Storyboard.SetTargetProperty(scaleYAnimation, 
            new PropertyPath(ScaleTransform.ScaleYProperty));
        
        _mouseEnterStoryboard.Children.Add(scaleXAnimation);
        _mouseEnterStoryboard.Children.Add(scaleYAnimation);
        
        // Mouse Leave 애니메이션
        _mouseLeaveStoryboard = new Storyboard();
        
        var scaleBackXAnimation = new DoubleAnimation(1.0, Duration);
        Storyboard.SetTarget(scaleBackXAnimation, _scaleTransform);
        Storyboard.SetTargetProperty(scaleBackXAnimation, 
            new PropertyPath(ScaleTransform.ScaleXProperty));
        
        var scaleBackYAnimation = new DoubleAnimation(1.0, Duration);
        Storyboard.SetTarget(scaleBackYAnimation, _scaleTransform);
        Storyboard.SetTargetProperty(scaleBackYAnimation, 
            new PropertyPath(ScaleTransform.ScaleYProperty));
        
        _mouseLeaveStoryboard.Children.Add(scaleBackXAnimation);
        _mouseLeaveStoryboard.Children.Add(scaleBackYAnimation);
    }
    
    private void OnMouseEnter(object sender, MouseEventArgs e)
    {
        _mouseLeaveStoryboard.Stop();
        _mouseEnterStoryboard.Begin();
    }
    
    private void OnMouseLeave(object sender, MouseEventArgs e)
    {
        _mouseEnterStoryboard.Stop();
        _mouseLeaveStoryboard.Begin();
    }
}
```

## 비동기 이벤트 처리

### 비동기 작업과 이벤트
```csharp
public class AsyncEventBehavior : Behavior<Button>
{
    private CancellationTokenSource _cancellationTokenSource;
    
    public static readonly DependencyProperty AsyncCommandProperty =
        DependencyProperty.Register(
            nameof(AsyncCommand),
            typeof(IAsyncCommand),
            typeof(AsyncEventBehavior));
    
    public IAsyncCommand AsyncCommand
    {
        get => (IAsyncCommand)GetValue(AsyncCommandProperty);
        set => SetValue(AsyncCommandProperty, value);
    }
    
    protected override void OnAttached()
    {
        base.OnAttached();
        AssociatedObject.Click += OnClick;
    }
    
    private async void OnClick(object sender, RoutedEventArgs e)
    {
        if (AsyncCommand == null || !AsyncCommand.CanExecute(null))
            return;
        
        // 이전 작업 취소
        _cancellationTokenSource?.Cancel();
        _cancellationTokenSource = new CancellationTokenSource();
        
        // UI 업데이트
        AssociatedObject.IsEnabled = false;
        
        try
        {
            await AsyncCommand.ExecuteAsync(null, _cancellationTokenSource.Token);
        }
        catch (OperationCanceledException)
        {
            // 취소됨
        }
        catch (Exception ex)
        {
            // 오류 처리
            MessageBox.Show($"Error: {ex.Message}");
        }
        finally
        {
            AssociatedObject.IsEnabled = true;
        }
    }
    
    protected override void OnDetaching()
    {
        base.OnDetaching();
        _cancellationTokenSource?.Cancel();
        AssociatedObject.Click -= OnClick;
    }
}

public interface IAsyncCommand
{
    bool CanExecute(object parameter);
    Task ExecuteAsync(object parameter, CancellationToken cancellationToken);
}
```

## 컴포지트 Behavior

### 여러 Behavior 결합
```csharp
public class CompositeBehavior : Behavior<FrameworkElement>
{
    private readonly List<Behavior> _behaviors = new List<Behavior>();
    
    public List<Behavior> Behaviors => _behaviors;
    
    protected override void OnAttached()
    {
        base.OnAttached();
        
        foreach (var behavior in _behaviors)
        {
            behavior.Attach(AssociatedObject);
        }
    }
    
    protected override void OnDetaching()
    {
        base.OnDetaching();
        
        foreach (var behavior in _behaviors)
        {
            behavior.Detach();
        }
    }
}

// 사용 예제
public class InteractiveButtonBehavior : CompositeBehavior
{
    public InteractiveButtonBehavior()
    {
        Behaviors.Add(new HoverAnimationBehavior { Scale = 1.1 });
        Behaviors.Add(new SoundEffectBehavior { SoundFile = "click.wav" });
        Behaviors.Add(new RippleEffectBehavior { Color = Colors.Blue });
    }
}
```

## 상태 기반 Behavior

### Visual State를 사용한 Behavior
```csharp
public class StateBasedBehavior : Behavior<Control>
{
    public static readonly DependencyProperty StateProperty =
        DependencyProperty.Register(
            nameof(State),
            typeof(string),
            typeof(StateBasedBehavior),
            new PropertyMetadata(string.Empty, OnStateChanged));
    
    public string State
    {
        get => (string)GetValue(StateProperty);
        set => SetValue(StateProperty, value);
    }
    
    protected override void OnAttached()
    {
        base.OnAttached();
        UpdateVisualState();
    }
    
    private static void OnStateChanged(DependencyObject d, 
                                      DependencyPropertyChangedEventArgs e)
    {
        var behavior = (StateBasedBehavior)d;
        behavior.UpdateVisualState();
    }
    
    private void UpdateVisualState()
    {
        if (AssociatedObject != null && !string.IsNullOrEmpty(State))
        {
            VisualStateManager.GoToState(AssociatedObject, State, true);
        }
    }
}
```

## 성능 최적화를 위한 이벤트 처리

### 이벤트 쓰로틀링
```csharp
public class ThrottledEventBehavior : Behavior<UIElement>
{
    private readonly DispatcherTimer _timer;
    private EventArgs _lastEventArgs;
    
    public static readonly DependencyProperty ThrottleIntervalProperty =
        DependencyProperty.Register(
            nameof(ThrottleInterval),
            typeof(TimeSpan),
            typeof(ThrottledEventBehavior),
            new PropertyMetadata(TimeSpan.FromMilliseconds(100)));
    
    public static readonly DependencyProperty CommandProperty =
        DependencyProperty.Register(
            nameof(Command),
            typeof(ICommand),
            typeof(ThrottledEventBehavior));
    
    public TimeSpan ThrottleInterval
    {
        get => (TimeSpan)GetValue(ThrottleIntervalProperty);
        set => SetValue(ThrottleIntervalProperty, value);
    }
    
    public ICommand Command
    {
        get => (ICommand)GetValue(CommandProperty);
        set => SetValue(CommandProperty, value);
    }
    
    public ThrottledEventBehavior()
    {
        _timer = new DispatcherTimer();
        _timer.Tick += OnTimerTick;
    }
    
    protected override void OnAttached()
    {
        base.OnAttached();
        
        _timer.Interval = ThrottleInterval;
        AssociatedObject.MouseMove += OnMouseMove;
    }
    
    private void OnMouseMove(object sender, MouseEventArgs e)
    {
        _lastEventArgs = e;
        
        if (!_timer.IsEnabled)
        {
            _timer.Start();
            ExecuteCommand();
        }
    }
    
    private void OnTimerTick(object sender, EventArgs e)
    {
        _timer.Stop();
    }
    
    private void ExecuteCommand()
    {
        if (Command?.CanExecute(_lastEventArgs) == true)
        {
            Command.Execute(_lastEventArgs);
        }
    }
}
```

### 이벤트 디바운싱
```csharp
public class DebouncedEventBehavior : Behavior<TextBox>
{
    private DispatcherTimer _debounceTimer;
    
    public static readonly DependencyProperty DebounceDelayProperty =
        DependencyProperty.Register(
            nameof(DebounceDelay),
            typeof(TimeSpan),
            typeof(DebouncedEventBehavior),
            new PropertyMetadata(TimeSpan.FromMilliseconds(500)));
    
    public static readonly DependencyProperty CommandProperty =
        DependencyProperty.Register(
            nameof(Command),
            typeof(ICommand),
            typeof(DebouncedEventBehavior));
    
    public TimeSpan DebounceDelay
    {
        get => (TimeSpan)GetValue(DebounceDelayProperty);
        set => SetValue(DebounceDelayProperty, value);
    }
    
    public ICommand Command
    {
        get => (ICommand)GetValue(CommandProperty);
        set => SetValue(CommandProperty, value);
    }
    
    protected override void OnAttached()
    {
        base.OnAttached();
        
        _debounceTimer = new DispatcherTimer
        {
            Interval = DebounceDelay
        };
        _debounceTimer.Tick += OnDebounceTimerTick;
        
        AssociatedObject.TextChanged += OnTextChanged;
    }
    
    private void OnTextChanged(object sender, TextChangedEventArgs e)
    {
        _debounceTimer.Stop();
        _debounceTimer.Start();
    }
    
    private void OnDebounceTimerTick(object sender, EventArgs e)
    {
        _debounceTimer.Stop();
        
        if (Command?.CanExecute(AssociatedObject.Text) == true)
        {
            Command.Execute(AssociatedObject.Text);
        }
    }
    
    protected override void OnDetaching()
    {
        base.OnDetaching();
        
        _debounceTimer?.Stop();
        AssociatedObject.TextChanged -= OnTextChanged;
    }
}
```

## 실전 예제: 상호작용 리스트

```csharp
public class InteractiveListBehavior : Behavior<ListBox>
{
    private ListBoxItem _draggedItem;
    private Point _dragStartPoint;
    
    public static readonly DependencyProperty AllowReorderProperty =
        DependencyProperty.Register(
            nameof(AllowReorder),
            typeof(bool),
            typeof(InteractiveListBehavior),
            new PropertyMetadata(true));
    
    public static readonly DependencyProperty ReorderCommandProperty =
        DependencyProperty.Register(
            nameof(ReorderCommand),
            typeof(ICommand),
            typeof(InteractiveListBehavior));
    
    public bool AllowReorder
    {
        get => (bool)GetValue(AllowReorderProperty);
        set => SetValue(AllowReorderProperty, value);
    }
    
    public ICommand ReorderCommand
    {
        get => (ICommand)GetValue(ReorderCommandProperty);
        set => SetValue(ReorderCommandProperty, value);
    }
    
    protected override void OnAttached()
    {
        base.OnAttached();
        
        AssociatedObject.PreviewMouseLeftButtonDown += OnPreviewMouseLeftButtonDown;
        AssociatedObject.PreviewMouseMove += OnPreviewMouseMove;
        AssociatedObject.PreviewMouseLeftButtonUp += OnPreviewMouseLeftButtonUp;
        AssociatedObject.Drop += OnDrop;
        AssociatedObject.DragOver += OnDragOver;
    }
    
    private void OnPreviewMouseLeftButtonDown(object sender, MouseButtonEventArgs e)
    {
        if (!AllowReorder) return;
        
        _dragStartPoint = e.GetPosition(null);
        _draggedItem = FindAncestor<ListBoxItem>((DependencyObject)e.OriginalSource);
    }
    
    private void OnPreviewMouseMove(object sender, MouseEventArgs e)
    {
        if (_draggedItem == null || e.LeftButton != MouseButtonState.Pressed)
            return;
        
        var currentPosition = e.GetPosition(null);
        var diff = _dragStartPoint - currentPosition;
        
        if (Math.Abs(diff.X) > SystemParameters.MinimumHorizontalDragDistance ||
            Math.Abs(diff.Y) > SystemParameters.MinimumVerticalDragDistance)
        {
            // 드래그 시작
            var data = new DataObject("ListBoxItem", _draggedItem.DataContext);
            DragDrop.DoDragDrop(_draggedItem, data, DragDropEffects.Move);
        }
    }
    
    private void OnDragOver(object sender, DragEventArgs e)
    {
        if (!e.Data.GetDataPresent("ListBoxItem"))
        {
            e.Effects = DragDropEffects.None;
            return;
        }
        
        // 드롭 위치 시각화
        var targetItem = FindAncestor<ListBoxItem>((DependencyObject)e.OriginalSource);
        if (targetItem != null)
        {
            var adornerLayer = AdornerLayer.GetAdornerLayer(targetItem);
            // 삽입 위치 표시 로직
        }
    }
    
    private void OnDrop(object sender, DragEventArgs e)
    {
        if (!e.Data.GetDataPresent("ListBoxItem")) return;
        
        var droppedData = e.Data.GetData("ListBoxItem");
        var targetItem = FindAncestor<ListBoxItem>((DependencyObject)e.OriginalSource);
        
        if (targetItem != null && ReorderCommand?.CanExecute(null) == true)
        {
            var reorderInfo = new ReorderInfo
            {
                Item = droppedData,
                NewIndex = AssociatedObject.ItemContainerGenerator.IndexFromContainer(targetItem)
            };
            
            ReorderCommand.Execute(reorderInfo);
        }
    }
    
    private static T FindAncestor<T>(DependencyObject current) where T : DependencyObject
    {
        while (current != null)
        {
            if (current is T ancestor)
                return ancestor;
            
            current = VisualTreeHelper.GetParent(current);
        }
        return null;
    }
}

public class ReorderInfo
{
    public object Item { get; set; }
    public int NewIndex { get; set; }
}
```

## 핵심 개념 정리
- **라우팅 이벤트**: Bubbling, Tunneling, Direct 전략
- **이벤트 핸들러**: XAML과 코드비하인드에서 처리
- **사용자 정의 이벤트**: RoutedEvent 등록과 발생
- **Commands**: 재사용 가능한 동작 캡슐화
- **WeakEvent**: 메모리 누수 방지
- **Behavior**: 재사용 가능한 동작 컴포넌트
- **Trigger/Action**: 선언적 이벤트 처리
- **EventToCommand**: MVVM에서 이벤트 처리
- **제스처**: 키보드, 마우스, 터치 입력 처리
- **드래그 앤 드롭**: 데이터 전송 구현
- **마우스 캡처**: 마우스 입력 독점
- **포커스 관리**: 키보드 입력 처리
- **입력 검증**: 실시간 유효성 검사
- **비동기 이벤트**: async/await와 함께 사용
- **성능 최적화**: 쓰로틀링과 디바운싱