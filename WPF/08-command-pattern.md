# 커맨드 패턴

## 커맨드 패턴 개요

WPF의 커맨드 패턴은 사용자 인터페이스와 비즈니스 로직을 분리하는 디자인 패턴입니다.

### 커맨드의 장점
- **재사용성**: 여러 UI 요소에서 동일한 커맨드 사용
- **자동 상태 관리**: CanExecute에 따른 UI 자동 활성화/비활성화
- **MVVM 지원**: View와 ViewModel의 느슨한 결합
- **단축키 지원**: InputBinding을 통한 키보드 단축키

## ICommand 인터페이스

### ICommand 구조
```csharp
public interface ICommand
{
    // 커맨드 실행 가능 여부 확인
    bool CanExecute(object parameter);
    
    // 커맨드 실행
    void Execute(object parameter);
    
    // CanExecute 상태 변경 이벤트
    event EventHandler CanExecuteChanged;
}
```

### 기본 ICommand 구현
```csharp
public class RelayCommand : ICommand
{
    private readonly Action<object> _execute;
    private readonly Predicate<object> _canExecute;
    
    public RelayCommand(Action<object> execute, Predicate<object> canExecute = null)
    {
        _execute = execute ?? throw new ArgumentNullException(nameof(execute));
        _canExecute = canExecute;
    }
    
    public event EventHandler CanExecuteChanged
    {
        add { CommandManager.RequerySuggested += value; }
        remove { CommandManager.RequerySuggested -= value; }
    }
    
    public bool CanExecute(object parameter)
    {
        return _canExecute?.Invoke(parameter) ?? true;
    }
    
    public void Execute(object parameter)
    {
        _execute(parameter);
    }
}
```

### 제네릭 RelayCommand
```csharp
public class RelayCommand<T> : ICommand
{
    private readonly Action<T> _execute;
    private readonly Predicate<T> _canExecute;
    
    public RelayCommand(Action<T> execute, Predicate<T> canExecute = null)
    {
        _execute = execute ?? throw new ArgumentNullException(nameof(execute));
        _canExecute = canExecute;
    }
    
    public event EventHandler CanExecuteChanged
    {
        add { CommandManager.RequerySuggested += value; }
        remove { CommandManager.RequerySuggested -= value; }
    }
    
    public bool CanExecute(object parameter)
    {
        return _canExecute?.Invoke((T)parameter) ?? true;
    }
    
    public void Execute(object parameter)
    {
        _execute((T)parameter);
    }
}
```

## RoutedCommand

### 기본 RoutedCommand 사용
```xml
<Window x:Class="CommandDemo.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <Window.CommandBindings>
        <!-- ApplicationCommands 바인딩 -->
        <CommandBinding Command="ApplicationCommands.New"
                        Executed="NewCommand_Executed"
                        CanExecute="NewCommand_CanExecute"/>
        <CommandBinding Command="ApplicationCommands.Open"
                        Executed="OpenCommand_Executed"/>
        <CommandBinding Command="ApplicationCommands.Save"
                        Executed="SaveCommand_Executed"
                        CanExecute="SaveCommand_CanExecute"/>
    </Window.CommandBindings>
    
    <DockPanel>
        <Menu DockPanel.Dock="Top">
            <MenuItem Header="_File">
                <MenuItem Command="ApplicationCommands.New"/>
                <MenuItem Command="ApplicationCommands.Open"/>
                <MenuItem Command="ApplicationCommands.Save"/>
                <Separator/>
                <MenuItem Command="ApplicationCommands.Close"/>
            </MenuItem>
        </Menu>
        
        <ToolBar DockPanel.Dock="Top">
            <Button Command="ApplicationCommands.New" 
                    Content="{Binding RelativeSource={RelativeSource Self}, 
                                     Path=Command.Text}"/>
            <Button Command="ApplicationCommands.Open"/>
            <Button Command="ApplicationCommands.Save"/>
        </ToolBar>
        
        <TextBox Name="textEditor" AcceptsReturn="True"/>
    </DockPanel>
</Window>
```

### 코드 비하인드
```csharp
public partial class MainWindow : Window
{
    private bool _isDirty = false;
    
    private void NewCommand_Executed(object sender, ExecutedRoutedEventArgs e)
    {
        textEditor.Clear();
        _isDirty = false;
    }
    
    private void NewCommand_CanExecute(object sender, CanExecuteRoutedEventArgs e)
    {
        e.CanExecute = true;
    }
    
    private void OpenCommand_Executed(object sender, ExecutedRoutedEventArgs e)
    {
        var dialog = new OpenFileDialog();
        if (dialog.ShowDialog() == true)
        {
            textEditor.Text = File.ReadAllText(dialog.FileName);
            _isDirty = false;
        }
    }
    
    private void SaveCommand_Executed(object sender, ExecutedRoutedEventArgs e)
    {
        var dialog = new SaveFileDialog();
        if (dialog.ShowDialog() == true)
        {
            File.WriteAllText(dialog.FileName, textEditor.Text);
            _isDirty = false;
        }
    }
    
    private void SaveCommand_CanExecute(object sender, CanExecuteRoutedEventArgs e)
    {
        e.CanExecute = _isDirty;
    }
}
```

## 커스텀 RoutedCommand

### 커스텀 커맨드 정의
```csharp
public static class CustomCommands
{
    public static readonly RoutedUICommand Exit = new RoutedUICommand(
        "Exit",                    // Text
        "Exit",                    // Name
        typeof(CustomCommands),    // Owner Type
        new InputGestureCollection()
        {
            new KeyGesture(Key.F4, ModifierKeys.Alt)
        });
    
    public static readonly RoutedUICommand Options = new RoutedUICommand(
        "Options",
        "Options",
        typeof(CustomCommands),
        new InputGestureCollection()
        {
            new KeyGesture(Key.O, ModifierKeys.Control | ModifierKeys.Alt)
        });
    
    public static readonly RoutedUICommand Refresh = new RoutedUICommand(
        "Refresh",
        "Refresh",
        typeof(CustomCommands),
        new InputGestureCollection()
        {
            new KeyGesture(Key.F5)
        });
}
```

### 커스텀 커맨드 사용
```xml
<Window.CommandBindings>
    <CommandBinding Command="{x:Static local:CustomCommands.Exit}"
                    Executed="ExitCommand_Executed"/>
    <CommandBinding Command="{x:Static local:CustomCommands.Options}"
                    Executed="OptionsCommand_Executed"/>
    <CommandBinding Command="{x:Static local:CustomCommands.Refresh}"
                    Executed="RefreshCommand_Executed"
                    CanExecute="RefreshCommand_CanExecute"/>
</Window.CommandBindings>

<Menu>
    <MenuItem Header="_File">
        <MenuItem Command="{x:Static local:CustomCommands.Exit}"/>
    </MenuItem>
    <MenuItem Header="_Tools">
        <MenuItem Command="{x:Static local:CustomCommands.Options}"/>
    </MenuItem>
    <MenuItem Header="_View">
        <MenuItem Command="{x:Static local:CustomCommands.Refresh}"/>
    </MenuItem>
</Menu>
```

## CommandParameter

### 파라미터를 사용하는 커맨드
```xml
<StackPanel>
    <!-- 문자열 파라미터 -->
    <Button Command="{Binding NavigateCommand}" 
            CommandParameter="Home"
            Content="Go to Home"/>
    
    <!-- 바인딩된 파라미터 -->
    <Button Command="{Binding DeleteCommand}"
            CommandParameter="{Binding SelectedItem}"
            Content="Delete Selected"/>
    
    <!-- 다중 파라미터 (MultiBinding) -->
    <Button Command="{Binding SaveCommand}">
        <Button.CommandParameter>
            <MultiBinding Converter="{StaticResource ArrayConverter}">
                <Binding Path="FirstName"/>
                <Binding Path="LastName"/>
                <Binding Path="Email"/>
            </MultiBinding>
        </Button.CommandParameter>
        <Button.Content>Save All</Button.Content>
    </Button>
</StackPanel>
```

### ViewModel에서 파라미터 처리
```csharp
public class MainViewModel : ViewModelBase
{
    public ICommand NavigateCommand { get; }
    public ICommand DeleteCommand { get; }
    public ICommand SaveCommand { get; }
    
    public MainViewModel()
    {
        NavigateCommand = new RelayCommand<string>(Navigate);
        DeleteCommand = new RelayCommand<object>(Delete, CanDelete);
        SaveCommand = new RelayCommand<object[]>(SaveAll);
    }
    
    private void Navigate(string destination)
    {
        switch (destination)
        {
            case "Home":
                // Navigate to home
                break;
            case "Settings":
                // Navigate to settings
                break;
        }
    }
    
    private void Delete(object item)
    {
        if (item is Person person)
        {
            People.Remove(person);
        }
    }
    
    private bool CanDelete(object item)
    {
        return item != null;
    }
    
    private void SaveAll(object[] parameters)
    {
        if (parameters.Length >= 3)
        {
            string firstName = parameters[0] as string;
            string lastName = parameters[1] as string;
            string email = parameters[2] as string;
            
            // Save logic
        }
    }
}
```

## InputBinding

### 키보드 단축키
```xml
<Window.InputBindings>
    <!-- 단일 키 -->
    <KeyBinding Key="F5" Command="{Binding RefreshCommand}"/>
    
    <!-- 조합 키 -->
    <KeyBinding Modifiers="Control" Key="S" 
                Command="{Binding SaveCommand}"/>
    <KeyBinding Modifiers="Control+Shift" Key="S" 
                Command="{Binding SaveAsCommand}"/>
    
    <!-- 파라미터와 함께 -->
    <KeyBinding Gesture="Control+1" 
                Command="{Binding SwitchViewCommand}"
                CommandParameter="View1"/>
    <KeyBinding Gesture="Control+2" 
                Command="{Binding SwitchViewCommand}"
                CommandParameter="View2"/>
</Window.InputBindings>
```

### 마우스 바인딩
```xml
<Grid>
    <Grid.InputBindings>
        <!-- 더블 클릭 -->
        <MouseBinding Gesture="LeftDoubleClick" 
                      Command="{Binding EditCommand}"
                      CommandParameter="{Binding SelectedItem}"/>
        
        <!-- 휠 클릭 -->
        <MouseBinding Gesture="MiddleClick" 
                      Command="{Binding CloseTabCommand}"/>
        
        <!-- Ctrl + 휠 -->
        <MouseBinding Gesture="Control+WheelClick" 
                      Command="{Binding OpenInNewTabCommand}"/>
    </Grid.InputBindings>
</Grid>
```

## 비동기 커맨드

### AsyncRelayCommand 구현
```csharp
public class AsyncRelayCommand : ICommand
{
    private readonly Func<object, Task> _execute;
    private readonly Predicate<object> _canExecute;
    private bool _isExecuting;
    
    public AsyncRelayCommand(Func<object, Task> execute, 
                           Predicate<object> canExecute = null)
    {
        _execute = execute ?? throw new ArgumentNullException(nameof(execute));
        _canExecute = canExecute;
    }
    
    public event EventHandler CanExecuteChanged
    {
        add { CommandManager.RequerySuggested += value; }
        remove { CommandManager.RequerySuggested -= value; }
    }
    
    public bool CanExecute(object parameter)
    {
        return !_isExecuting && (_canExecute?.Invoke(parameter) ?? true);
    }
    
    public async void Execute(object parameter)
    {
        if (!CanExecute(parameter))
            return;
        
        _isExecuting = true;
        CommandManager.InvalidateRequerySuggested();
        
        try
        {
            await _execute(parameter);
        }
        finally
        {
            _isExecuting = false;
            CommandManager.InvalidateRequerySuggested();
        }
    }
}
```

### 비동기 커맨드 사용
```csharp
public class DataViewModel : ViewModelBase
{
    private bool _isLoading;
    
    public bool IsLoading
    {
        get => _isLoading;
        set => SetProperty(ref _isLoading, value);
    }
    
    public ICommand LoadDataCommand { get; }
    
    public DataViewModel()
    {
        LoadDataCommand = new AsyncRelayCommand(LoadDataAsync);
    }
    
    private async Task LoadDataAsync(object parameter)
    {
        IsLoading = true;
        
        try
        {
            // 비동기 데이터 로딩
            var data = await DataService.GetDataAsync();
            
            // UI 스레드에서 컬렉션 업데이트
            await Application.Current.Dispatcher.InvokeAsync(() =>
            {
                Items.Clear();
                foreach (var item in data)
                {
                    Items.Add(item);
                }
            });
        }
        catch (Exception ex)
        {
            MessageBox.Show($"Error loading data: {ex.Message}");
        }
        finally
        {
            IsLoading = false;
        }
    }
}
```

## 커맨드 체이닝

### CompositeCommand
```csharp
public class CompositeCommand : ICommand
{
    private readonly List<ICommand> _commands = new List<ICommand>();
    
    public void RegisterCommand(ICommand command)
    {
        _commands.Add(command);
        command.CanExecuteChanged += OnCanExecuteChanged;
    }
    
    public void UnregisterCommand(ICommand command)
    {
        _commands.Remove(command);
        command.CanExecuteChanged -= OnCanExecuteChanged;
    }
    
    public event EventHandler CanExecuteChanged;
    
    private void OnCanExecuteChanged(object sender, EventArgs e)
    {
        CanExecuteChanged?.Invoke(this, e);
    }
    
    public bool CanExecute(object parameter)
    {
        return _commands.All(cmd => cmd.CanExecute(parameter));
    }
    
    public void Execute(object parameter)
    {
        foreach (var command in _commands)
        {
            if (command.CanExecute(parameter))
            {
                command.Execute(parameter);
            }
        }
    }
}
```

## 실전 예제: 텍스트 에디터 커맨드

### ViewModel
```csharp
public class TextEditorViewModel : ViewModelBase
{
    private string _text;
    private string _fileName;
    private bool _isDirty;
    
    public string Text
    {
        get => _text;
        set
        {
            if (SetProperty(ref _text, value))
            {
                IsDirty = true;
            }
        }
    }
    
    public string FileName
    {
        get => _fileName;
        set => SetProperty(ref _fileName, value);
    }
    
    public bool IsDirty
    {
        get => _isDirty;
        private set => SetProperty(ref _isDirty, value);
    }
    
    // Commands
    public ICommand NewCommand { get; }
    public ICommand OpenCommand { get; }
    public ICommand SaveCommand { get; }
    public ICommand SaveAsCommand { get; }
    public ICommand CutCommand { get; }
    public ICommand CopyCommand { get; }
    public ICommand PasteCommand { get; }
    
    public TextEditorViewModel()
    {
        NewCommand = new RelayCommand(New);
        OpenCommand = new AsyncRelayCommand(OpenAsync);
        SaveCommand = new AsyncRelayCommand(SaveAsync, _ => IsDirty);
        SaveAsCommand = new AsyncRelayCommand(SaveAsAsync);
        
        CutCommand = new RelayCommand(Cut, CanCut);
        CopyCommand = new RelayCommand(Copy, CanCopy);
        PasteCommand = new RelayCommand(Paste, CanPaste);
    }
    
    private void New(object parameter)
    {
        if (IsDirty)
        {
            var result = MessageBox.Show("Save changes?", "Confirm", 
                                       MessageBoxButton.YesNoCancel);
            if (result == MessageBoxResult.Cancel)
                return;
            if (result == MessageBoxResult.Yes)
                SaveAsync(null).Wait();
        }
        
        Text = string.Empty;
        FileName = null;
        IsDirty = false;
    }
    
    private async Task OpenAsync(object parameter)
    {
        var dialog = new OpenFileDialog
        {
            Filter = "Text files (*.txt)|*.txt|All files (*.*)|*.*"
        };
        
        if (dialog.ShowDialog() == true)
        {
            Text = await File.ReadAllTextAsync(dialog.FileName);
            FileName = dialog.FileName;
            IsDirty = false;
        }
    }
    
    private async Task SaveAsync(object parameter)
    {
        if (string.IsNullOrEmpty(FileName))
        {
            await SaveAsAsync(parameter);
        }
        else
        {
            await File.WriteAllTextAsync(FileName, Text);
            IsDirty = false;
        }
    }
    
    private async Task SaveAsAsync(object parameter)
    {
        var dialog = new SaveFileDialog
        {
            Filter = "Text files (*.txt)|*.txt|All files (*.*)|*.*"
        };
        
        if (dialog.ShowDialog() == true)
        {
            await File.WriteAllTextAsync(dialog.FileName, Text);
            FileName = dialog.FileName;
            IsDirty = false;
        }
    }
    
    // Clipboard operations would need TextBox reference
    private void Cut(object parameter) { /* Implementation */ }
    private bool CanCut(object parameter) => !string.IsNullOrEmpty(Text);
    
    private void Copy(object parameter) { /* Implementation */ }
    private bool CanCopy(object parameter) => !string.IsNullOrEmpty(Text);
    
    private void Paste(object parameter) { /* Implementation */ }
    private bool CanPaste(object parameter) => Clipboard.ContainsText();
}
```

## 핵심 개념 정리
- **ICommand**: 커맨드 패턴의 기본 인터페이스
- **RelayCommand**: ICommand의 범용 구현체
- **RoutedCommand**: WPF 내장 커맨드 시스템
- **CommandParameter**: 커맨드에 데이터 전달
- **InputBinding**: 키보드/마우스 단축키 바인딩
- **비동기 커맨드**: 장시간 작업을 위한 async 지원