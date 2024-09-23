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

## 서비스와 의존성 주입

### 서비스 인터페이스
```csharp
public interface IDataService
{
    Task<IEnumerable<Person>> GetPeopleAsync();
    Task<Person> GetPersonByIdAsync(int id);
    Task<Person> SavePersonAsync(Person person);
    Task DeletePersonAsync(int id);
}

public interface IDialogService
{
    bool ShowConfirmation(string message, string title = "Confirm");
    void ShowError(string message, string title = "Error");
    void ShowInfo(string message, string title = "Information");
    string ShowOpenFileDialog(string filter = "All files (*.*)|*.*");
    string ShowSaveFileDialog(string filter = "All files (*.*)|*.*");
}

public interface INavigationService
{
    void NavigateTo(string viewName, object parameter = null);
    void GoBack();
    bool CanGoBack { get; }
}
```

### 서비스 구현
```csharp
public class DialogService : IDialogService
{
    public bool ShowConfirmation(string message, string title = "Confirm")
    {
        var result = MessageBox.Show(message, title, 
                                   MessageBoxButton.YesNo, 
                                   MessageBoxImage.Question);
        return result == MessageBoxResult.Yes;
    }
    
    public void ShowError(string message, string title = "Error")
    {
        MessageBox.Show(message, title, 
                       MessageBoxButton.OK, 
                       MessageBoxImage.Error);
    }
    
    public void ShowInfo(string message, string title = "Information")
    {
        MessageBox.Show(message, title, 
                       MessageBoxButton.OK, 
                       MessageBoxImage.Information);
    }
    
    public string ShowOpenFileDialog(string filter = "All files (*.*)|*.*")
    {
        var dialog = new OpenFileDialog { Filter = filter };
        return dialog.ShowDialog() == true ? dialog.FileName : null;
    }
    
    public string ShowSaveFileDialog(string filter = "All files (*.*)|*.*")
    {
        var dialog = new SaveFileDialog { Filter = filter };
        return dialog.ShowDialog() == true ? dialog.FileName : null;
    }
}
```

### ViewModel에서 서비스 사용
```csharp
public class PersonDetailsViewModel : ViewModelBase
{
    private readonly IDataService _dataService;
    private readonly IDialogService _dialogService;
    private readonly INavigationService _navigationService;
    
    private Person _person;
    private bool _isLoading;
    
    public PersonDetailsViewModel(
        IDataService dataService,
        IDialogService dialogService,
        INavigationService navigationService)
    {
        _dataService = dataService;
        _dialogService = dialogService;
        _navigationService = navigationService;
        
        SaveCommand = new AsyncRelayCommand(SaveAsync);
        CancelCommand = new RelayCommand(Cancel);
    }
    
    public Person Person
    {
        get => _person;
        set => SetProperty(ref _person, value);
    }
    
    public bool IsLoading
    {
        get => _isLoading;
        set => SetProperty(ref _isLoading, value);
    }
    
    public ICommand SaveCommand { get; }
    public ICommand CancelCommand { get; }
    
    public async Task LoadPersonAsync(int personId)
    {
        try
        {
            IsLoading = true;
            Person = await _dataService.GetPersonByIdAsync(personId);
        }
        catch (Exception ex)
        {
            _dialogService.ShowError($"Failed to load person: {ex.Message}");
        }
        finally
        {
            IsLoading = false;
        }
    }
    
    private async Task SaveAsync(object parameter)
    {
        try
        {
            IsLoading = true;
            await _dataService.SavePersonAsync(Person);
            _dialogService.ShowInfo("Person saved successfully!");
            _navigationService.GoBack();
        }
        catch (Exception ex)
        {
            _dialogService.ShowError($"Failed to save person: {ex.Message}");
        }
        finally
        {
            IsLoading = false;
        }
    }
    
    private void Cancel(object parameter)
    {
        if (_dialogService.ShowConfirmation("Discard changes?"))
        {
            _navigationService.GoBack();
        }
    }
}
```

## 비동기 패턴과 MVVM

### AsyncRelayCommand
```csharp
public class AsyncRelayCommand : ICommand
{
    private readonly Func<object, Task> _execute;
    private readonly Predicate<object> _canExecute;
    private bool _isExecuting;
    
    public AsyncRelayCommand(Func<object, Task> execute, Predicate<object> canExecute = null)
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
        RaiseCanExecuteChanged();
        
        try
        {
            await _execute(parameter);
        }
        finally
        {
            _isExecuting = false;
            RaiseCanExecuteChanged();
        }
    }
    
    public void RaiseCanExecuteChanged()
    {
        CommandManager.InvalidateRequerySuggested();
    }
}
```

### 비동기 데이터 로딩
```csharp
public class AsyncDataViewModel : ViewModelBase
{
    private ObservableCollection<PersonViewModel> _people;
    private bool _isLoading;
    private string _loadingMessage;
    
    public ObservableCollection<PersonViewModel> People
    {
        get => _people;
        set => SetProperty(ref _people, value);
    }
    
    public bool IsLoading
    {
        get => _isLoading;
        set => SetProperty(ref _isLoading, value);
    }
    
    public string LoadingMessage
    {
        get => _loadingMessage;
        set => SetProperty(ref _loadingMessage, value);
    }
    
    public ICommand LoadDataCommand { get; }
    public ICommand RefreshCommand { get; }
    
    public AsyncDataViewModel()
    {
        People = new ObservableCollection<PersonViewModel>();
        LoadDataCommand = new AsyncRelayCommand(LoadDataAsync);
        RefreshCommand = new AsyncRelayCommand(RefreshAsync);
    }
    
    private async Task LoadDataAsync(object parameter)
    {
        if (IsLoading) return;
        
        try
        {
            IsLoading = true;
            LoadingMessage = "Loading people...";
            
            var people = await Task.Run(async () =>
            {
                // Simulate long-running operation
                await Task.Delay(2000);
                return await DataService.GetPeopleAsync();
            });
            
            // Update UI on main thread
            await Application.Current.Dispatcher.InvokeAsync(() =>
            {
                People.Clear();
                foreach (var person in people)
                {
                    People.Add(new PersonViewModel(person));
                }
            });
        }
        catch (Exception ex)
        {
            LoadingMessage = $"Error: {ex.Message}";
        }
        finally
        {
            IsLoading = false;
        }
    }
    
    private async Task RefreshAsync(object parameter)
    {
        People.Clear();
        await LoadDataAsync(parameter);
    }
}
```

## MVVM 프레임워크 기능

### Navigation Service 구현
```csharp
public class NavigationService : INavigationService
{
    private readonly Dictionary<string, Type> _viewTypes = new Dictionary<string, Type>();
    private readonly Stack<Page> _navigationStack = new Stack<Page>();
    private Frame _navigationFrame;
    
    public void Configure(string key, Type viewType)
    {
        _viewTypes[key] = viewType;
    }
    
    public void SetNavigationFrame(Frame frame)
    {
        _navigationFrame = frame;
    }
    
    public void NavigateTo(string viewName, object parameter = null)
    {
        if (!_viewTypes.ContainsKey(viewName))
            throw new ArgumentException($"View '{viewName}' not registered");
        
        var viewType = _viewTypes[viewName];
        var view = Activator.CreateInstance(viewType) as Page;
        
        if (view?.DataContext is INavigationAware navigationAware)
        {
            navigationAware.OnNavigatedTo(parameter);
        }
        
        _navigationStack.Push(view);
        _navigationFrame.Navigate(view);
    }
    
    public void GoBack()
    {
        if (CanGoBack)
        {
            var currentView = _navigationStack.Pop();
            
            if (currentView?.DataContext is INavigationAware navigationAware)
            {
                navigationAware.OnNavigatedFrom();
            }
            
            _navigationFrame.GoBack();
        }
    }
    
    public bool CanGoBack => _navigationStack.Count > 1;
}

public interface INavigationAware
{
    void OnNavigatedTo(object parameter);
    void OnNavigatedFrom();
}
```

## 실전 예제: 완전한 MVVM 애플리케이션

### App.xaml.cs - 부트스트래핑
```csharp
public partial class App : Application
{
    private ServiceContainer _container;
    
    protected override void OnStartup(StartupEventArgs e)
    {
        base.OnStartup(e);
        
        ConfigureServices();
        
        var mainWindow = new MainWindow
        {
            DataContext = _container.GetService<MainViewModel>()
        };
        
        mainWindow.Show();
    }
    
    private void ConfigureServices()
    {
        _container = new ServiceContainer();
        
        // Register services
        _container.RegisterSingleton<IDataService, DataService>();
        _container.RegisterSingleton<IDialogService, DialogService>();
        _container.RegisterSingleton<INavigationService, NavigationService>();
        
        // Register ViewModels
        _container.RegisterTransient<MainViewModel>();
        _container.RegisterTransient<PersonListViewModel>();
        _container.RegisterTransient<PersonDetailsViewModel>();
    }
}
```

### MainViewModel - 전체 애플리케이션 조정
```csharp
public class MainViewModel : ViewModelBase
{
    private readonly INavigationService _navigationService;
    private ViewModelBase _currentViewModel;
    
    public MainViewModel(INavigationService navigationService)
    {
        _navigationService = navigationService;
        
        // Register views
        _navigationService.Configure("PersonList", typeof(PersonListView));
        _navigationService.Configure("PersonDetails", typeof(PersonDetailsView));
        
        // Commands
        ShowPersonListCommand = new RelayCommand(_ => ShowPersonList());
        ShowSettingsCommand = new RelayCommand(_ => ShowSettings());
        
        // Subscribe to navigation events
        Messenger.Default.Register<NavigationMessage>(OnNavigationRequested);
        
        // Show initial view
        ShowPersonList();
    }
    
    public ViewModelBase CurrentViewModel
    {
        get => _currentViewModel;
        set => SetProperty(ref _currentViewModel, value);
    }
    
    public ICommand ShowPersonListCommand { get; }
    public ICommand ShowSettingsCommand { get; }
    
    private void ShowPersonList()
    {
        CurrentViewModel = new PersonListViewModel(_dataService, _dialogService);
    }
    
    private void ShowSettings()
    {
        CurrentViewModel = new SettingsViewModel();
    }
    
    private void OnNavigationRequested(NavigationMessage message)
    {
        switch (message.ViewName)
        {
            case "PersonDetails":
                CurrentViewModel = new PersonDetailsViewModel(_dataService, _dialogService)
                {
                    PersonId = (int)message.Parameter
                };
                break;
        }
    }
}
```

## 테스트와 MVVM

### ViewModel 단위 테스트
```csharp
[TestClass]
public class PersonViewModelTests
{
    private Mock<IDataService> _mockDataService;
    private Mock<IDialogService> _mockDialogService;
    
    [TestInitialize]
    public void Setup()
    {
        _mockDataService = new Mock<IDataService>();
        _mockDialogService = new Mock<IDialogService>();
    }
    
    [TestMethod]
    public async Task SaveCommand_ValidPerson_SavesSuccessfully()
    {
        // Arrange
        var person = new Person { FirstName = "John", LastName = "Doe" };
        var viewModel = new PersonDetailsViewModel(
            _mockDataService.Object,
            _mockDialogService.Object,
            null);
        
        viewModel.Person = person;
        
        _mockDataService
            .Setup(x => x.SavePersonAsync(It.IsAny<Person>()))
            .ReturnsAsync(person);
        
        // Act
        await viewModel.SaveCommand.Execute(null);
        
        // Assert
        _mockDataService.Verify(x => x.SavePersonAsync(person), Times.Once);
        _mockDialogService.Verify(x => x.ShowInfo(It.IsAny<string>()), Times.Once);
    }
    
    [TestMethod]
    public void FirstName_Changed_RaisesPropertyChanged()
    {
        // Arrange
        var viewModel = new PersonViewModel(new Person());
        var propertyChangedRaised = false;
        string changedPropertyName = null;
        
        viewModel.PropertyChanged += (sender, e) =>
        {
            propertyChangedRaised = true;
            changedPropertyName = e.PropertyName;
        };
        
        // Act
        viewModel.FirstName = "NewName";
        
        // Assert
        Assert.IsTrue(propertyChangedRaised);
        Assert.AreEqual(nameof(PersonViewModel.FirstName), changedPropertyName);
    }
}
```

## MVVM 베스트 프랙티스

### ViewModel 설계 원칙
1. **UI 독립성**: ViewModel은 View에 대한 참조를 가지지 않음
2. **테스트 가능성**: 모든 로직은 테스트 가능하게 작성
3. **단일 책임**: 각 ViewModel은 하나의 View를 담당
4. **재사용성**: 공통 기능은 기본 클래스나 서비스로 추출

### 폴더 구조
```
ProjectName/
├── Models/
│   ├── Person.cs
│   └── Address.cs
├── ViewModels/
│   ├── Base/
│   │   ├── ViewModelBase.cs
│   │   └── RelayCommand.cs
│   ├── MainViewModel.cs
│   ├── PersonListViewModel.cs
│   └── PersonDetailsViewModel.cs
├── Views/
│   ├── MainWindow.xaml
│   ├── PersonListView.xaml
│   └── PersonDetailsView.xaml
├── Services/
│   ├── IDataService.cs
│   ├── DataService.cs
│   └── DialogService.cs
└── Converters/
    ├── BooleanToVisibilityConverter.cs
    └── InverseBooleanConverter.cs
```

### 일반적인 실수와 해결책
```csharp
// ❌ 잘못된 예: View에 대한 직접 참조
public class BadViewModel
{
    private MainWindow _window;
    
    public void ShowMessage()
    {
        _window.statusLabel.Text = "Hello"; // View 직접 조작
    }
}

// ✅ 올바른 예: 데이터 바인딩 사용
public class GoodViewModel : ViewModelBase
{
    private string _statusMessage;
    
    public string StatusMessage
    {
        get => _statusMessage;
        set => SetProperty(ref _statusMessage, value);
    }
    
    public void ShowMessage()
    {
        StatusMessage = "Hello"; // 속성 변경으로 View 업데이트
    }
}
```

### 메모리 누수 방지
```csharp
public class LeakSafeViewModel : ViewModelBase, IDisposable
{
    private readonly CompositeDisposable _disposables = new CompositeDisposable();
    
    public LeakSafeViewModel()
    {
        // 이벤트 구독 시 약한 참조 사용
        WeakEventManager<DataService, EventArgs>
            .AddHandler(DataService.Instance, "DataChanged", OnDataChanged);
        
        // 또는 Disposable로 관리
        Messenger.Default.Register<UpdateMessage>(this, OnUpdateMessage);
        _disposables.Add(Disposable.Create(() => 
            Messenger.Default.Unregister<UpdateMessage>(this)));
    }
    
    private void OnDataChanged(object sender, EventArgs e)
    {
        // Handle event
    }
    
    private void OnUpdateMessage(UpdateMessage message)
    {
        // Handle message
    }
    
    public void Dispose()
    {
        _disposables.Dispose();
    }
}
```

### 비동기 작업 패턴
```csharp
public class AsyncViewModel : ViewModelBase
{
    private readonly SemaphoreSlim _loadingSemaphore = new SemaphoreSlim(1, 1);
    private CancellationTokenSource _loadingCts;
    
    public async Task LoadDataAsync()
    {
        await _loadingSemaphore.WaitAsync();
        
        try
        {
            // 이전 작업 취소
            _loadingCts?.Cancel();
            _loadingCts = new CancellationTokenSource();
            
            IsLoading = true;
            ErrorMessage = null;
            
            try
            {
                var data = await DataService.LoadAsync(_loadingCts.Token);
                UpdateUI(data);
            }
            catch (OperationCanceledException)
            {
                // 작업이 취소됨
            }
            catch (Exception ex)
            {
                ErrorMessage = ex.Message;
                Logger.LogError(ex);
            }
        }
        finally
        {
            IsLoading = false;
            _loadingSemaphore.Release();
        }
    }
}
```

## 인기 MVVM 프레임워크

### 1. **Prism**
```csharp
// Module 정의
public class PersonModule : IModule
{
    public void OnInitialized(IContainerProvider containerProvider)
    {
        var regionManager = containerProvider.Resolve<IRegionManager>();
        regionManager.RegisterViewWithRegion("MainRegion", typeof(PersonListView));
    }
    
    public void RegisterTypes(IContainerRegistry containerRegistry)
    {
        containerRegistry.RegisterForNavigation<PersonDetailsView>();
        containerRegistry.RegisterSingleton<IDataService, DataService>();
    }
}
```

### 2. **MVVM Light**
```csharp
public class MainViewModel : ViewModelBase
{
    public RelayCommand<Person> SelectPersonCommand { get; }
    
    public MainViewModel()
    {
        SelectPersonCommand = new RelayCommand<Person>(
            person => MessengerInstance.Send(new PersonSelectedMessage(person)));
    }
}
```

### 3. **Caliburn.Micro**
```csharp
public class ShellViewModel : Screen
{
    public void SayHello(string name)
    {
        // Convention-based: 자동으로 버튼 Click과 연결
        MessageBox.Show($"Hello {name}!");
    }
    
    public bool CanSayHello(string name)
    {
        return !string.IsNullOrWhiteSpace(name);
    }
}
```

### 4. **ReactiveUI**
```csharp
public class SearchViewModel : ReactiveObject
{
    private string _searchText;
    private readonly ObservableAsPropertyHelper<List<SearchResult>> _searchResults;
    
    public SearchViewModel()
    {
        // Reactive Extensions 사용
        _searchResults = this
            .WhenAnyValue(x => x.SearchText)
            .Throttle(TimeSpan.FromMilliseconds(500))
            .Where(text => !string.IsNullOrWhiteSpace(text))
            .SelectMany(SearchAsync)
            .ObserveOnDispatcher()
            .ToProperty(this, x => x.SearchResults);
    }
    
    public string SearchText
    {
        get => _searchText;
        set => this.RaiseAndSetIfChanged(ref _searchText, value);
    }
    
    public List<SearchResult> SearchResults => _searchResults.Value;
}
```

## 고급 MVVM 패턴

### ViewModel 첫 번째 Navigation
```csharp
public interface IViewModelResolver
{
    Type ResolveViewType(Type viewModelType);
    object ResolveView(Type viewModelType);
}

public class ViewModelFirstNavigationService : INavigationService
{
    private readonly IViewModelResolver _resolver;
    private readonly Frame _frame;
    
    public async Task NavigateToAsync<TViewModel>(object parameter = null) 
        where TViewModel : ViewModelBase
    {
        var viewType = _resolver.ResolveViewType(typeof(TViewModel));
        var view = Activator.CreateInstance(viewType) as Page;
        var viewModel = Activator.CreateInstance<TViewModel>();
        
        view.DataContext = viewModel;
        
        if (viewModel is INavigationAware navigationAware)
        {
            await navigationAware.OnNavigatedToAsync(parameter);
        }
        
        _frame.Navigate(view);
    }
}
```

### 복합 ViewModel
```csharp
public class CompositeViewModel : ViewModelBase
{
    public HeaderViewModel Header { get; }
    public NavigationViewModel Navigation { get; }
    public ContentViewModel Content { get; set; }
    public StatusBarViewModel StatusBar { get; }
    
    public CompositeViewModel(
        HeaderViewModel header,
        NavigationViewModel navigation,
        StatusBarViewModel statusBar)
    {
        Header = header;
        Navigation = navigation;
        StatusBar = statusBar;
        
        // Navigation 변경 시 Content 업데이트
        Navigation.NavigationRequested += OnNavigationRequested;
    }
    
    private void OnNavigationRequested(object sender, NavigationEventArgs e)
    {
        Content = CreateContentViewModel(e.TargetView);
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
- **Services**: 의존성 주입을 통한 기능 제공
- **Testing**: ViewModel의 독립적 테스트
- **Best Practices**: MVVM 패턴의 올바른 구현 방법