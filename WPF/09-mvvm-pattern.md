# MVVM 패턴

## MVVM 패턴 개요

MVVM(Model-View-ViewModel)은 WPF 애플리케이션의 표준 아키텍처 패턴입니다.

### MVVM의 구성 요소
- **Model**: 비즈니스 로직과 데이터
- **View**: 사용자 인터페이스 (XAML)
- **ViewModel**: View와 Model 사이의 중재자

### MVVM의 장점
- **테스트 용이성**: UI 없이 로직 테스트 가능
- **재사용성**: ViewModel과 Model의 재사용
- **유지보수성**: 관심사의 분리
- **디자이너-개발자 협업**: XAML과 코드 분리

## Model 구현

### 기본 Model 클래스
```csharp
public class Person
{
    public int Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public DateTime BirthDate { get; set; }
    public string Email { get; set; }
    
    public string FullName => $"{FirstName} {LastName}";
    public int Age => DateTime.Now.Year - BirthDate.Year;
}
```

### INotifyPropertyChanged를 구현한 Model
```csharp
public class ObservablePerson : INotifyPropertyChanged
{
    private string _firstName;
    private string _lastName;
    private string _email;
    
    public string FirstName
    {
        get => _firstName;
        set
        {
            if (_firstName != value)
            {
                _firstName = value;
                OnPropertyChanged();
                OnPropertyChanged(nameof(FullName));
            }
        }
    }
    
    public string LastName
    {
        get => _lastName;
        set
        {
            if (_lastName != value)
            {
                _lastName = value;
                OnPropertyChanged();
                OnPropertyChanged(nameof(FullName));
            }
        }
    }
    
    public string Email
    {
        get => _email;
        set
        {
            if (_email != value)
            {
                _email = value;
                OnPropertyChanged();
            }
        }
    }
    
    public string FullName => $"{FirstName} {LastName}";
    
    public event PropertyChangedEventHandler PropertyChanged;
    
    protected virtual void OnPropertyChanged([CallerMemberName] string propertyName = null)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }
}
```

## ViewModel 기본 구조

### ViewModelBase 클래스
```csharp
public abstract class ViewModelBase : INotifyPropertyChanged
{
    public event PropertyChangedEventHandler PropertyChanged;
    
    protected virtual void OnPropertyChanged([CallerMemberName] string propertyName = null)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }
    
    protected bool SetProperty<T>(ref T field, T value, [CallerMemberName] string propertyName = null)
    {
        if (EqualityComparer<T>.Default.Equals(field, value))
            return false;
        
        field = value;
        OnPropertyChanged(propertyName);
        return true;
    }
}
```

### 기본 ViewModel 구현
```csharp
public class PersonViewModel : ViewModelBase
{
    private Person _model;
    private string _displayName;
    
    public PersonViewModel(Person person)
    {
        _model = person ?? new Person();
        _displayName = _model.FullName;
    }
    
    public string FirstName
    {
        get => _model.FirstName;
        set
        {
            if (_model.FirstName != value)
            {
                _model.FirstName = value;
                OnPropertyChanged();
                OnPropertyChanged(nameof(FullName));
                UpdateDisplayName();
            }
        }
    }
    
    public string LastName
    {
        get => _model.LastName;
        set
        {
            if (_model.LastName != value)
            {
                _model.LastName = value;
                OnPropertyChanged();
                OnPropertyChanged(nameof(FullName));
                UpdateDisplayName();
            }
        }
    }
    
    public string FullName => _model.FullName;
    
    public string DisplayName
    {
        get => _displayName;
        private set => SetProperty(ref _displayName, value);
    }
    
    private void UpdateDisplayName()
    {
        DisplayName = string.IsNullOrWhiteSpace(FullName) ? "Unknown" : FullName;
    }
}
```

## View와 ViewModel 연결

### DataContext 설정
```xml
<!-- XAML에서 직접 설정 -->
<Window x:Class="MvvmDemo.Views.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:vm="clr-namespace:MvvmDemo.ViewModels"
        Title="MVVM Demo">
    <Window.DataContext>
        <vm:MainViewModel/>
    </Window.DataContext>
</Window>
```

### 코드에서 DataContext 설정
```csharp
public partial class MainWindow : Window
{
    public MainWindow()
    {
        InitializeComponent();
        DataContext = new MainViewModel();
    }
}
```

### ViewModelLocator 패턴
```csharp
public class ViewModelLocator
{
    private static MainViewModel _mainViewModel;
    
    public MainViewModel MainViewModel
    {
        get => _mainViewModel ?? (_mainViewModel = new MainViewModel());
    }
    
    public PersonListViewModel PersonListViewModel => new PersonListViewModel();
    
    public static void Cleanup()
    {
        _mainViewModel = null;
    }
}
```

```xml
<!-- App.xaml -->
<Application.Resources>
    <local:ViewModelLocator x:Key="Locator"/>
</Application.Resources>

<!-- View에서 사용 -->
<Window DataContext="{Binding MainViewModel, Source={StaticResource Locator}}">
</Window>
```

## Commands in ViewModel

### RelayCommand 구현
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
    
    public bool CanExecute(object parameter) => _canExecute?.Invoke(parameter) ?? true;
    
    public void Execute(object parameter) => _execute(parameter);
    
    public void RaiseCanExecuteChanged() => CommandManager.InvalidateRequerySuggested();
}
```

### ViewModel에서 Commands 사용
```csharp
public class PersonListViewModel : ViewModelBase
{
    private ObservableCollection<PersonViewModel> _people;
    private PersonViewModel _selectedPerson;
    
    public ObservableCollection<PersonViewModel> People
    {
        get => _people;
        set => SetProperty(ref _people, value);
    }
    
    public PersonViewModel SelectedPerson
    {
        get => _selectedPerson;
        set => SetProperty(ref _selectedPerson, value);
    }
    
    // Commands
    public ICommand AddPersonCommand { get; }
    public ICommand DeletePersonCommand { get; }
    public ICommand EditPersonCommand { get; }
    public ICommand RefreshCommand { get; }
    
    public PersonListViewModel()
    {
        People = new ObservableCollection<PersonViewModel>();
        
        AddPersonCommand = new RelayCommand(AddPerson);
        DeletePersonCommand = new RelayCommand(DeletePerson, CanDeletePerson);
        EditPersonCommand = new RelayCommand(EditPerson, CanEditPerson);
        RefreshCommand = new RelayCommand(Refresh);
        
        LoadPeople();
    }
    
    private void AddPerson(object parameter)
    {
        var newPerson = new Person
        {
            FirstName = "New",
            LastName = "Person",
            Email = "new@example.com"
        };
        
        var personVm = new PersonViewModel(newPerson);
        People.Add(personVm);
        SelectedPerson = personVm;
    }
    
    private void DeletePerson(object parameter)
    {
        if (SelectedPerson != null)
        {
            People.Remove(SelectedPerson);
            SelectedPerson = null;
        }
    }
    
    private bool CanDeletePerson(object parameter) => SelectedPerson != null;
    
    private void EditPerson(object parameter)
    {
        if (SelectedPerson != null)
        {
            // Open edit dialog
            var editDialog = new EditPersonDialog
            {
                DataContext = new EditPersonViewModel(SelectedPerson)
            };
            editDialog.ShowDialog();
        }
    }
    
    private bool CanEditPerson(object parameter) => SelectedPerson != null;
    
    private void Refresh(object parameter) => LoadPeople();
    
    private void LoadPeople()
    {
        // Simulate loading from database
        var people = DataService.GetPeople();
        
        People.Clear();
        foreach (var person in people)
        {
            People.Add(new PersonViewModel(person));
        }
    }
}
```

## 데이터 바인딩과 MVVM

### View (XAML)
```xml
<Window x:Class="MvvmDemo.Views.PersonListView"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Person List" Height="400" Width="600">
    <Grid>
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>
        
        <!-- Toolbar -->
        <ToolBar Grid.Row="0">
            <Button Command="{Binding AddPersonCommand}" Content="Add"/>
            <Button Command="{Binding EditPersonCommand}" Content="Edit"/>
            <Button Command="{Binding DeletePersonCommand}" Content="Delete"/>
            <Separator/>
            <Button Command="{Binding RefreshCommand}" Content="Refresh"/>
        </ToolBar>
        
        <!-- Person List -->
        <DataGrid Grid.Row="1" 
                  ItemsSource="{Binding People}"
                  SelectedItem="{Binding SelectedPerson}"
                  AutoGenerateColumns="False">
            <DataGrid.Columns>
                <DataGridTextColumn Header="First Name" 
                                    Binding="{Binding FirstName}" 
                                    Width="150"/>
                <DataGridTextColumn Header="Last Name" 
                                    Binding="{Binding LastName}" 
                                    Width="150"/>
                <DataGridTextColumn Header="Email" 
                                    Binding="{Binding Email}" 
                                    Width="*"/>
            </DataGrid.Columns>
        </DataGrid>
        
        <!-- Status Bar -->
        <StatusBar Grid.Row="2">
            <StatusBarItem>
                <TextBlock Text="{Binding People.Count, StringFormat='Total: {0} people'}"/>
            </StatusBarItem>
        </StatusBar>
    </Grid>
</Window>
```

## Validation in MVVM

### IDataErrorInfo 구현
```csharp
public class ValidatingPersonViewModel : ViewModelBase, IDataErrorInfo
{
    private string _firstName;
    private string _lastName;
    private string _email;
    
    public string FirstName
    {
        get => _firstName;
        set => SetProperty(ref _firstName, value);
    }
    
    public string LastName
    {
        get => _lastName;
        set => SetProperty(ref _lastName, value);
    }
    
    public string Email
    {
        get => _email;
        set => SetProperty(ref _email, value);
    }
    
    // IDataErrorInfo implementation
    public string Error => null;
    
    public string this[string propertyName]
    {
        get
        {
            string error = null;
            
            switch (propertyName)
            {
                case nameof(FirstName):
                    if (string.IsNullOrWhiteSpace(FirstName))
                        error = "First name is required";
                    else if (FirstName.Length < 2)
                        error = "First name must be at least 2 characters";
                    break;
                    
                case nameof(LastName):
                    if (string.IsNullOrWhiteSpace(LastName))
                        error = "Last name is required";
                    break;
                    
                case nameof(Email):
                    if (string.IsNullOrWhiteSpace(Email))
                        error = "Email is required";
                    else if (!IsValidEmail(Email))
                        error = "Invalid email format";
                    break;
            }
            
            return error;
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
    
    public bool IsValid
    {
        get
        {
            return string.IsNullOrEmpty(this[nameof(FirstName)]) &&
                   string.IsNullOrEmpty(this[nameof(LastName)]) &&
                   string.IsNullOrEmpty(this[nameof(Email)]);
        }
    }
}
```

### Validation을 사용한 View
```xml
<Grid>
    <Grid.Resources>
        <Style TargetType="TextBox">
            <Setter Property="Validation.ErrorTemplate">
                <Setter.Value>
                    <ControlTemplate>
                        <DockPanel>
                            <TextBlock DockPanel.Dock="Right" 
                                     Foreground="Red" 
                                     FontSize="20" 
                                     Text="!" 
                                     Margin="5,0,0,0"/>
                            <Border BorderBrush="Red" BorderThickness="1">
                                <AdornedElementPlaceholder/>
                            </Border>
                        </DockPanel>
                    </ControlTemplate>
                </Setter.Value>
            </Setter>
            <Style.Triggers>
                <Trigger Property="Validation.HasError" Value="True">
                    <Setter Property="ToolTip" 
                            Value="{Binding RelativeSource={RelativeSource Self},
                                           Path=(Validation.Errors)[0].ErrorContent}"/>
                </Trigger>
            </Style.Triggers>
        </Style>
    </Grid.Resources>
    
    <StackPanel Margin="20">
        <Label Content="First Name:"/>
        <TextBox Text="{Binding FirstName, UpdateSourceTrigger=PropertyChanged, 
                                ValidatesOnDataErrors=True}"/>
        
        <Label Content="Last Name:" Margin="0,10,0,0"/>
        <TextBox Text="{Binding LastName, UpdateSourceTrigger=PropertyChanged,
                                ValidatesOnDataErrors=True}"/>
        
        <Label Content="Email:" Margin="0,10,0,0"/>
        <TextBox Text="{Binding Email, UpdateSourceTrigger=PropertyChanged,
                                ValidatesOnDataErrors=True}"/>
        
        <Button Content="Save" 
                Margin="0,20,0,0"
                Command="{Binding SaveCommand}"
                IsEnabled="{Binding IsValid}"/>
    </StackPanel>
</Grid>
```

## 메시징 패턴

### 간단한 Messenger 구현
```csharp
public class Messenger
{
    private static readonly Messenger _instance = new Messenger();
    public static Messenger Default => _instance;
    
    private readonly Dictionary<Type, List<Delegate>> _subscribers = 
        new Dictionary<Type, List<Delegate>>();
    
    public void Register<TMessage>(Action<TMessage> action)
    {
        var messageType = typeof(TMessage);
        
        if (!_subscribers.ContainsKey(messageType))
        {
            _subscribers[messageType] = new List<Delegate>();
        }
        
        _subscribers[messageType].Add(action);
    }
    
    public void Unregister<TMessage>(Action<TMessage> action)
    {
        var messageType = typeof(TMessage);
        
        if (_subscribers.ContainsKey(messageType))
        {
            _subscribers[messageType].Remove(action);
        }
    }
    
    public void Send<TMessage>(TMessage message)
    {
        var messageType = typeof(TMessage);
        
        if (_subscribers.ContainsKey(messageType))
        {
            foreach (var subscriber in _subscribers[messageType].ToList())
            {
                (subscriber as Action<TMessage>)?.Invoke(message);
            }
        }
    }
}
```

### Message 클래스들
```csharp
public class PersonSelectedMessage
{
    public PersonViewModel SelectedPerson { get; set; }
}

public class PersonUpdatedMessage
{
    public PersonViewModel UpdatedPerson { get; set; }
}

public class StatusMessage
{
    public string Text { get; set; }
    public bool IsError { get; set; }
}
```

### Messenger 사용 예제
```csharp
public class MainViewModel : ViewModelBase
{
    public MainViewModel()
    {
        // 메시지 구독
        Messenger.Default.Register<PersonSelectedMessage>(OnPersonSelected);
        Messenger.Default.Register<StatusMessage>(OnStatusMessage);
    }
    
    private void OnPersonSelected(PersonSelectedMessage message)
    {
        CurrentPerson = message.SelectedPerson;
        // Update UI or perform actions
    }
    
    private void OnStatusMessage(StatusMessage message)
    {
        StatusText = message.Text;
        IsError = message.IsError;
    }
}

public class PersonListViewModel : ViewModelBase
{
    private PersonViewModel _selectedPerson;
    
    public PersonViewModel SelectedPerson
    {
        get => _selectedPerson;
        set
        {
            if (SetProperty(ref _selectedPerson, value))
            {
                // 다른 ViewModel에 선택 알림
                Messenger.Default.Send(new PersonSelectedMessage 
                { 
                    SelectedPerson = value 
                });
            }
        }
    }
}
```

## 핵심 개념 정리
- **Model**: 비즈니스 로직과 데이터 표현
- **View**: XAML로 작성된 사용자 인터페이스
- **ViewModel**: View와 Model 사이의 중재자
- **Data Binding**: View와 ViewModel 연결
- **Commands**: 사용자 상호작용 처리
- **Validation**: 데이터 유효성 검사
- **Messaging**: ViewModel 간 통신