# 내비게이션과 창

## Window 클래스

Window는 WPF 애플리케이션의 최상위 UI 요소입니다.

### Window 속성
```xml
<Window x:Class="MyApp.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="My Application"
        Height="450" Width="800"
        WindowStartupLocation="CenterScreen"
        WindowStyle="SingleBorderWindow"
        ResizeMode="CanResize"
        Icon="app.ico"
        ShowInTaskbar="True"
        Topmost="False">
</Window>
```

### Window 라이프사이클
```csharp
public partial class MainWindow : Window
{
    public MainWindow()
    {
        InitializeComponent();
        
        // 이벤트 등록
        SourceInitialized += OnSourceInitialized;
        Activated += OnActivated;
        Loaded += OnLoaded;
        ContentRendered += OnContentRendered;
        Deactivated += OnDeactivated;
        Closing += OnClosing;
        Closed += OnClosed;
    }
    
    private void OnSourceInitialized(object sender, EventArgs e)
    {
        // Win32 창 핸들이 생성됨
        Debug.WriteLine("SourceInitialized");
    }
    
    private void OnActivated(object sender, EventArgs e)
    {
        // 창이 활성화됨
        Debug.WriteLine("Activated");
    }
    
    private void OnLoaded(object sender, RoutedEventArgs e)
    {
        // 요소 트리가 생성되고 렌더링 준비됨
        Debug.WriteLine("Loaded");
    }
    
    private void OnContentRendered(object sender, EventArgs e)
    {
        // 콘텐츠가 렌더링됨
        Debug.WriteLine("ContentRendered");
    }
    
    private void OnClosing(object sender, CancelEventArgs e)
    {
        // 창이 닫히기 전
        var result = MessageBox.Show("정말 종료하시겠습니까?", "확인", 
                                   MessageBoxButton.YesNo);
        if (result == MessageBoxResult.No)
        {
            e.Cancel = true;
        }
    }
    
    private void OnClosed(object sender, EventArgs e)
    {
        // 창이 닫힘
        Debug.WriteLine("Closed");
    }
}
```

## Window 관리

### 창 표시 모드
```csharp
public class WindowManager
{
    // 모달 창
    public bool ShowModalDialog(Window owner)
    {
        var dialog = new DialogWindow();
        dialog.Owner = owner;
        return dialog.ShowDialog() ?? false;
    }
    
    // 모달리스 창
    public void ShowModelessWindow(Window owner)
    {
        var window = new ChildWindow();
        window.Owner = owner;
        window.Show();
    }
    
    // 창 위치 설정
    public void PositionWindow(Window window, Window owner)
    {
        // 소유자 창 중앙에 배치
        window.WindowStartupLocation = WindowStartupLocation.CenterOwner;
        
        // 수동 위치 설정
        window.WindowStartupLocation = WindowStartupLocation.Manual;
        window.Left = owner.Left + 50;
        window.Top = owner.Top + 50;
    }
}
```

### 창 상태 관리
```csharp
public class WindowStateManager
{
    public void SaveWindowState(Window window, string windowName)
    {
        var settings = new WindowSettings
        {
            Left = window.Left,
            Top = window.Top,
            Width = window.Width,
            Height = window.Height,
            WindowState = window.WindowState
        };
        
        // 설정 저장
        Properties.Settings.Default[$"{windowName}_Settings"] = 
            JsonSerializer.Serialize(settings);
        Properties.Settings.Default.Save();
    }
    
    public void RestoreWindowState(Window window, string windowName)
    {
        var settingsJson = Properties.Settings.Default[$"{windowName}_Settings"] as string;
        if (!string.IsNullOrEmpty(settingsJson))
        {
            var settings = JsonSerializer.Deserialize<WindowSettings>(settingsJson);
            
            window.Left = settings.Left;
            window.Top = settings.Top;
            window.Width = settings.Width;
            window.Height = settings.Height;
            window.WindowState = settings.WindowState;
        }
    }
}

public class WindowSettings
{
    public double Left { get; set; }
    public double Top { get; set; }
    public double Width { get; set; }
    public double Height { get; set; }
    public WindowState WindowState { get; set; }
}
```

## 대화상자

### MessageBox
```csharp
public class DialogService
{
    public MessageBoxResult ShowMessage(string message, string title = "알림")
    {
        return MessageBox.Show(message, title, MessageBoxButton.OK, 
                             MessageBoxImage.Information);
    }
    
    public bool ShowConfirmation(string message, string title = "확인")
    {
        var result = MessageBox.Show(message, title, MessageBoxButton.YesNo, 
                                   MessageBoxImage.Question);
        return result == MessageBoxResult.Yes;
    }
    
    public MessageBoxResult ShowYesNoCancel(string message, string title = "선택")
    {
        return MessageBox.Show(message, title, MessageBoxButton.YesNoCancel,
                             MessageBoxImage.Question, 
                             MessageBoxResult.Cancel);
    }
    
    public void ShowError(string message, string title = "오류")
    {
        MessageBox.Show(message, title, MessageBoxButton.OK, 
                       MessageBoxImage.Error);
    }
}
```

### 사용자 정의 대화상자
```xml
<!-- CustomDialog.xaml -->
<Window x:Class="MyApp.CustomDialog"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Custom Dialog"
        Height="200" Width="400"
        WindowStartupLocation="CenterOwner"
        WindowStyle="ToolWindow"
        ResizeMode="NoResize"
        ShowInTaskbar="False">
    <Grid>
        <Grid.RowDefinitions>
            <RowDefinition Height="*"/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>
        
        <StackPanel Grid.Row="0" Margin="20">
            <TextBlock Text="이름을 입력하세요:" Margin="0,0,0,10"/>
            <TextBox x:Name="nameTextBox" />
        </StackPanel>
        
        <StackPanel Grid.Row="1" Orientation="Horizontal" 
                    HorizontalAlignment="Right" Margin="20">
            <Button Content="확인" Width="80" Margin="5" 
                    Click="OkButton_Click" IsDefault="True"/>
            <Button Content="취소" Width="80" Margin="5" 
                    Click="CancelButton_Click" IsCancel="True"/>
        </StackPanel>
    </Grid>
</Window>
```

```csharp
// CustomDialog.xaml.cs
public partial class CustomDialog : Window
{
    public string UserName { get; private set; }
    
    public CustomDialog()
    {
        InitializeComponent();
        nameTextBox.Focus();
    }
    
    private void OkButton_Click(object sender, RoutedEventArgs e)
    {
        if (string.IsNullOrWhiteSpace(nameTextBox.Text))
        {
            MessageBox.Show("이름을 입력해주세요.", "경고", 
                          MessageBoxButton.OK, MessageBoxImage.Warning);
            return;
        }
        
        UserName = nameTextBox.Text;
        DialogResult = true;
    }
    
    private void CancelButton_Click(object sender, RoutedEventArgs e)
    {
        DialogResult = false;
    }
}

// 사용
var dialog = new CustomDialog();
if (dialog.ShowDialog() == true)
{
    string name = dialog.UserName;
    // 이름 사용
}
```

## 파일 대화상자

### OpenFileDialog
```csharp
public class FileDialogService
{
    public string OpenFile(string filter = "All files (*.*)|*.*")
    {
        var dialog = new OpenFileDialog
        {
            Filter = filter,
            InitialDirectory = Environment.GetFolderPath(Environment.SpecialFolder.MyDocuments),
            Title = "파일 선택"
        };
        
        if (dialog.ShowDialog() == true)
        {
            return dialog.FileName;
        }
        
        return null;
    }
    
    public string[] OpenMultipleFiles(string filter = "All files (*.*)|*.*")
    {
        var dialog = new OpenFileDialog
        {
            Filter = filter,
            Multiselect = true,
            Title = "파일 선택 (여러 개 가능)"
        };
        
        if (dialog.ShowDialog() == true)
        {
            return dialog.FileNames;
        }
        
        return Array.Empty<string>();
    }
}
```

### SaveFileDialog
```csharp
public string SaveFile(string defaultFileName = "", 
                      string filter = "Text files (*.txt)|*.txt|All files (*.*)|*.*")
{
    var dialog = new SaveFileDialog
    {
        FileName = defaultFileName,
        Filter = filter,
        DefaultExt = ".txt",
        AddExtension = true,
        OverwritePrompt = true,
        Title = "파일 저장"
    };
    
    if (dialog.ShowDialog() == true)
    {
        return dialog.FileName;
    }
    
    return null;
}
```

### FolderBrowserDialog (Windows Forms)
```csharp
public string SelectFolder()
{
    using (var dialog = new System.Windows.Forms.FolderBrowserDialog())
    {
        dialog.Description = "폴더를 선택하세요";
        dialog.ShowNewFolderButton = true;
        
        if (dialog.ShowDialog() == System.Windows.Forms.DialogResult.OK)
        {
            return dialog.SelectedPath;
        }
    }
    
    return null;
}
```

## Page Navigation

### NavigationWindow
```xml
<NavigationWindow x:Class="MyApp.MainNavigationWindow"
                  xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                  xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
                  Title="Navigation Demo"
                  Height="450" Width="800"
                  Source="Pages/HomePage.xaml">
</NavigationWindow>
```

### Page 정의
```xml
<!-- HomePage.xaml -->
<Page x:Class="MyApp.Pages.HomePage"
      xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
      xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
      Title="Home Page">
    <Grid>
        <StackPanel VerticalAlignment="Center" HorizontalAlignment="Center">
            <TextBlock Text="홈 페이지" FontSize="24" Margin="10"/>
            <Button Content="다음 페이지로" Click="NextButton_Click" Width="150"/>
        </StackPanel>
    </Grid>
</Page>
```

```csharp
// HomePage.xaml.cs
public partial class HomePage : Page
{
    public HomePage()
    {
        InitializeComponent();
    }
    
    private void NextButton_Click(object sender, RoutedEventArgs e)
    {
        NavigationService.Navigate(new DetailsPage());
        
        // 또는 URI 사용
        // NavigationService.Navigate(new Uri("Pages/DetailsPage.xaml", UriKind.Relative));
    }
}
```

### NavigationService
```csharp
public class NavigationManager
{
    private NavigationService _navigationService;
    
    public NavigationManager(NavigationService navigationService)
    {
        _navigationService = navigationService;
        
        // 이벤트 등록
        _navigationService.Navigating += OnNavigating;
        _navigationService.Navigated += OnNavigated;
        _navigationService.NavigationFailed += OnNavigationFailed;
    }
    
    public void NavigateTo(Page page)
    {
        _navigationService.Navigate(page);
    }
    
    public void NavigateTo(string pageUri)
    {
        _navigationService.Navigate(new Uri(pageUri, UriKind.Relative));
    }
    
    public void GoBack()
    {
        if (_navigationService.CanGoBack)
        {
            _navigationService.GoBack();
        }
    }
    
    public void GoForward()
    {
        if (_navigationService.CanGoForward)
        {
            _navigationService.GoForward();
        }
    }
    
    private void OnNavigating(object sender, NavigatingCancelEventArgs e)
    {
        // 내비게이션 전 처리
        Debug.WriteLine($"Navigating to: {e.Uri}");
    }
    
    private void OnNavigated(object sender, NavigationEventArgs e)
    {
        // 내비게이션 완료
        Debug.WriteLine($"Navigated to: {e.Uri}");
    }
    
    private void OnNavigationFailed(object sender, NavigationFailedEventArgs e)
    {
        // 내비게이션 실패
        MessageBox.Show($"Navigation failed: {e.Exception.Message}");
        e.Handled = true;
    }
}
```

## Frame 기반 내비게이션

### Frame 사용
```xml
<Window x:Class="MyApp.FrameHostWindow">
    <DockPanel>
        <!-- 내비게이션 바 -->
        <ToolBar DockPanel.Dock="Top">
            <Button Content="←" Command="{Binding GoBackCommand}"/>
            <Button Content="→" Command="{Binding GoForwardCommand}"/>
            <Button Content="🏠" Command="{Binding GoHomeCommand}"/>
            <TextBlock Text="{Binding CurrentPageTitle}" 
                       VerticalAlignment="Center" Margin="10,0"/>
        </ToolBar>
        
        <!-- 콘텐츠 프레임 -->
        <Frame x:Name="mainFrame" 
               NavigationUIVisibility="Hidden"
               Navigated="Frame_Navigated"/>
    </DockPanel>
</Window>
```

```csharp
public partial class FrameHostWindow : Window
{
    public FrameHostWindow()
    {
        InitializeComponent();
        
        // 초기 페이지 로드
        mainFrame.Navigate(new HomePage());
    }
    
    private void Frame_Navigated(object sender, NavigationEventArgs e)
    {
        // 프레임 내비게이션 이벤트 처리
        UpdateNavigationButtons();
    }
    
    private void UpdateNavigationButtons()
    {
        // 뒤로/앞으로 버튼 상태 업데이트
        var viewModel = DataContext as MainViewModel;
        if (viewModel != null)
        {
            viewModel.CanGoBack = mainFrame.CanGoBack;
            viewModel.CanGoForward = mainFrame.CanGoForward;
        }
    }
}
```

## Window Chrome 커스터마이징

### 사용자 정의 타이틀바
```xml
<Window x:Class="MyApp.CustomChromeWindow"
        WindowStyle="None"
        AllowsTransparency="True"
        Background="Transparent">
    <Border Background="White" BorderBrush="Gray" BorderThickness="1">
        <Grid>
            <Grid.RowDefinitions>
                <RowDefinition Height="30"/>
                <RowDefinition Height="*"/>
            </Grid.RowDefinitions>
            
            <!-- 사용자 정의 타이틀바 -->
            <Grid Grid.Row="0" Background="DarkBlue" 
                  MouseLeftButtonDown="TitleBar_MouseLeftButtonDown">
                <Grid.ColumnDefinitions>
                    <ColumnDefinition Width="Auto"/>
                    <ColumnDefinition Width="*"/>
                    <ColumnDefinition Width="Auto"/>
                </Grid.ColumnDefinitions>
                
                <Image Grid.Column="0" Source="app.ico" 
                       Width="20" Height="20" Margin="5"/>
                
                <TextBlock Grid.Column="1" Text="{Binding Title}" 
                           Foreground="White" VerticalAlignment="Center" 
                           Margin="5,0"/>
                
                <StackPanel Grid.Column="2" Orientation="Horizontal">
                    <Button Content="_" Width="30" Height="30" 
                            Click="MinimizeButton_Click"
                            Style="{StaticResource ChromeButtonStyle}"/>
                    <Button Content="□" Width="30" Height="30" 
                            Click="MaximizeButton_Click"
                            Style="{StaticResource ChromeButtonStyle}"/>
                    <Button Content="X" Width="30" Height="30" 
                            Click="CloseButton_Click"
                            Style="{StaticResource ChromeButtonStyle}"/>
                </StackPanel>
            </Grid>
            
            <!-- 콘텐츠 영역 -->
            <Grid Grid.Row="1">
                <!-- 실제 콘텐츠 -->
            </Grid>
        </Grid>
    </Border>
</Window>
```

```csharp
public partial class CustomChromeWindow : Window
{
    public CustomChromeWindow()
    {
        InitializeComponent();
    }
    
    private void TitleBar_MouseLeftButtonDown(object sender, MouseButtonEventArgs e)
    {
        if (e.ClickCount == 2)
        {
            // 더블클릭 시 최대화/복원
            WindowState = WindowState == WindowState.Maximized 
                ? WindowState.Normal : WindowState.Maximized;
        }
        else
        {
            // 드래그로 창 이동
            DragMove();
        }
    }
    
    private void MinimizeButton_Click(object sender, RoutedEventArgs e)
    {
        WindowState = WindowState.Minimized;
    }
    
    private void MaximizeButton_Click(object sender, RoutedEventArgs e)
    {
        WindowState = WindowState == WindowState.Maximized 
            ? WindowState.Normal : WindowState.Maximized;
    }
    
    private void CloseButton_Click(object sender, RoutedEventArgs e)
    {
        Close();
    }
}
```

## 다중 창 관리

### Window Manager
```csharp
public class WindowManager : IWindowManager
{
    private readonly Dictionary<string, Window> _windows = new Dictionary<string, Window>();
    
    public void RegisterWindow(string key, Window window)
    {
        if (_windows.ContainsKey(key))
        {
            _windows[key].Close();
        }
        
        _windows[key] = window;
        window.Closed += (s, e) => _windows.Remove(key);
    }
    
    public void ShowWindow(string key, object viewModel = null)
    {
        if (_windows.TryGetValue(key, out var window))
        {
            window.Activate();
            return;
        }
        
        window = CreateWindow(key);
        if (window != null)
        {
            if (viewModel != null)
            {
                window.DataContext = viewModel;
            }
            
            RegisterWindow(key, window);
            window.Show();
        }
    }
    
    public bool? ShowDialog(string key, object viewModel = null)
    {
        var window = CreateWindow(key);
        if (window != null)
        {
            if (viewModel != null)
            {
                window.DataContext = viewModel;
            }
            
            return window.ShowDialog();
        }
        
        return null;
    }
    
    private Window CreateWindow(string key)
    {
        return key switch
        {
            "Settings" => new SettingsWindow(),
            "About" => new AboutWindow(),
            "Editor" => new EditorWindow(),
            _ => null
        };
    }
    
    public void CloseWindow(string key)
    {
        if (_windows.TryGetValue(key, out var window))
        {
            window.Close();
        }
    }
    
    public void CloseAllWindows()
    {
        foreach (var window in _windows.Values.ToList())
        {
            window.Close();
        }
    }
}
```

## 스플래시 스크린

### 기본 스플래시 스크린
```csharp
// App.xaml.cs
public partial class App : Application
{
    protected override void OnStartup(StartupEventArgs e)
    {
        // 스플래시 스크린 표시
        var splash = new SplashScreen("Images/splash.png");
        splash.Show(true);
        
        // 초기화 작업
        InitializeApplication();
        
        base.OnStartup(e);
    }
}
```

### 사용자 정의 스플래시 윈도우
```xml
<!-- SplashWindow.xaml -->
<Window x:Class="MyApp.SplashWindow"
        WindowStyle="None"
        WindowStartupLocation="CenterScreen"
        Width="500" Height="300"
        ResizeMode="NoResize"
        ShowInTaskbar="False"
        Topmost="True">
    <Grid>
        <Grid.Background>
            <LinearGradientBrush StartPoint="0,0" EndPoint="1,1">
                <GradientStop Color="#FF1E3A8A" Offset="0"/>
                <GradientStop Color="#FF3B82F6" Offset="1"/>
            </LinearGradientBrush>
        </Grid.Background>
        
        <StackPanel VerticalAlignment="Center" HorizontalAlignment="Center">
            <TextBlock Text="My Application" FontSize="36" 
                       Foreground="White" FontWeight="Bold"/>
            <TextBlock Text="Loading..." FontSize="16" 
                       Foreground="White" Margin="0,20,0,0"/>
            <ProgressBar Width="200" Height="10" Margin="0,10,0,0" 
                        IsIndeterminate="True"/>
        </StackPanel>
    </Grid>
</Window>
```

```csharp
public partial class SplashWindow : Window
{
    public SplashWindow()
    {
        InitializeComponent();
    }
    
    public async Task ShowWithInitialization(Func<IProgress<string>, Task> initAction)
    {
        Show();
        
        var progress = new Progress<string>(message =>
        {
            // UI 업데이트
            Dispatcher.Invoke(() => statusText.Text = message);
        });
        
        await initAction(progress);
        
        Close();
    }
}

// 사용
var splash = new SplashWindow();
await splash.ShowWithInitialization(async progress =>
{
    progress.Report("데이터베이스 초기화 중...");
    await InitializeDatabase();
    
    progress.Report("설정 로드 중...");
    await LoadSettings();
    
    progress.Report("UI 준비 중...");
    await Task.Delay(500);
});

var mainWindow = new MainWindow();
mainWindow.Show();
```

## 내비게이션 패턴

### MVVM 내비게이션
```csharp
public interface INavigationService
{
    void NavigateTo(string pageKey, object parameter = null);
    void GoBack();
    bool CanGoBack { get; }
}

public class NavigationService : INavigationService
{
    private readonly Frame _frame;
    private readonly Dictionary<string, Type> _pageTypes = new Dictionary<string, Type>();
    
    public NavigationService(Frame frame)
    {
        _frame = frame;
    }
    
    public void RegisterPage(string key, Type pageType)
    {
        _pageTypes[key] = pageType;
    }
    
    public void NavigateTo(string pageKey, object parameter = null)
    {
        if (_pageTypes.TryGetValue(pageKey, out var pageType))
        {
            var page = Activator.CreateInstance(pageType) as Page;
            
            if (page?.DataContext is INavigationAware navigationAware)
            {
                navigationAware.OnNavigatedTo(parameter);
            }
            
            _frame.Navigate(page);
        }
    }
    
    public void GoBack()
    {
        if (_frame.CanGoBack)
        {
            _frame.GoBack();
        }
    }
    
    public bool CanGoBack => _frame.CanGoBack;
}

public interface INavigationAware
{
    void OnNavigatedTo(object parameter);
    void OnNavigatedFrom();
}
```

### 마법사 스타일 내비게이션
```csharp
public class WizardViewModel : ViewModelBase
{
    private readonly List<WizardPageViewModel> _pages;
    private int _currentPageIndex;
    
    public WizardPageViewModel CurrentPage => _pages[_currentPageIndex];
    
    public bool CanGoNext => _currentPageIndex < _pages.Count - 1 && CurrentPage.IsValid;
    public bool CanGoPrevious => _currentPageIndex > 0;
    public bool CanFinish => _currentPageIndex == _pages.Count - 1 && CurrentPage.IsValid;
    
    public ICommand NextCommand { get; }
    public ICommand PreviousCommand { get; }
    public ICommand FinishCommand { get; }
    
    public WizardViewModel()
    {
        _pages = new List<WizardPageViewModel>
        {
            new BasicInfoPageViewModel(),
            new OptionsPageViewModel(),
            new SummaryPageViewModel()
        };
        
        NextCommand = new RelayCommand(Next, () => CanGoNext);
        PreviousCommand = new RelayCommand(Previous, () => CanGoPrevious);
        FinishCommand = new RelayCommand(Finish, () => CanFinish);
    }
    
    private void Next()
    {
        if (CanGoNext)
        {
            _currentPageIndex++;
            OnPropertyChanged(nameof(CurrentPage));
            UpdateCommands();
        }
    }
    
    private void Previous()
    {
        if (CanGoPrevious)
        {
            _currentPageIndex--;
            OnPropertyChanged(nameof(CurrentPage));
            UpdateCommands();
        }
    }
    
    private void Finish()
    {
        // 완료 처리
    }
    
    private void UpdateCommands()
    {
        (NextCommand as RelayCommand)?.RaiseCanExecuteChanged();
        (PreviousCommand as RelayCommand)?.RaiseCanExecuteChanged();
        (FinishCommand as RelayCommand)?.RaiseCanExecuteChanged();
    }
}
```

## 실전 예제: 탭 MDI 구현

```csharp
public class TabbedMdiViewModel : ViewModelBase
{
    private readonly ObservableCollection<TabItemViewModel> _tabs;
    private TabItemViewModel _selectedTab;
    
    public ObservableCollection<TabItemViewModel> Tabs => _tabs;
    
    public TabItemViewModel SelectedTab
    {
        get => _selectedTab;
        set => SetProperty(ref _selectedTab, value);
    }
    
    public ICommand NewTabCommand { get; }
    public ICommand CloseTabCommand { get; }
    
    public TabbedMdiViewModel()
    {
        _tabs = new ObservableCollection<TabItemViewModel>();
        
        NewTabCommand = new RelayCommand<string>(CreateNewTab);
        CloseTabCommand = new RelayCommand<TabItemViewModel>(CloseTab);
    }
    
    private void CreateNewTab(string tabType)
    {
        TabItemViewModel newTab = tabType switch
        {
            "Editor" => new EditorTabViewModel(),
            "Browser" => new BrowserTabViewModel(),
            _ => new DefaultTabViewModel()
        };
        
        _tabs.Add(newTab);
        SelectedTab = newTab;
    }
    
    private void CloseTab(TabItemViewModel tab)
    {
        if (tab?.CanClose() == true)
        {
            _tabs.Remove(tab);
            
            if (SelectedTab == tab)
            {
                SelectedTab = _tabs.LastOrDefault();
            }
        }
    }
}

public abstract class TabItemViewModel : ViewModelBase
{
    public abstract string Header { get; }
    public abstract object Content { get; }
    
    public virtual bool CanClose()
    {
        return true;
    }
}
```

```xml
<!-- TabbedMdiView.xaml -->
<UserControl x:Class="MyApp.TabbedMdiView">
    <DockPanel>
        <Menu DockPanel.Dock="Top">
            <MenuItem Header="파일">
                <MenuItem Header="새 편집기" 
                         Command="{Binding NewTabCommand}" 
                         CommandParameter="Editor"/>
                <MenuItem Header="새 브라우저" 
                         Command="{Binding NewTabCommand}" 
                         CommandParameter="Browser"/>
            </MenuItem>
        </Menu>
        
        <TabControl ItemsSource="{Binding Tabs}" 
                    SelectedItem="{Binding SelectedTab}">
            <TabControl.ItemTemplate>
                <DataTemplate>
                    <DockPanel>
                        <Button DockPanel.Dock="Right" 
                               Content="×" 
                               Command="{Binding DataContext.CloseTabCommand, 
                                         RelativeSource={RelativeSource AncestorType=TabControl}}"
                               CommandParameter="{Binding}"
                               Background="Transparent"
                               BorderThickness="0"/>
                        <TextBlock Text="{Binding Header}"/>
                    </DockPanel>
                </DataTemplate>
            </TabControl.ItemTemplate>
            
            <TabControl.ContentTemplate>
                <DataTemplate>
                    <ContentControl Content="{Binding Content}"/>
                </DataTemplate>
            </TabControl.ContentTemplate>
        </TabControl>
    </DockPanel>
</UserControl>
```

## 핵심 개념 정리
- **Window**: WPF 애플리케이션의 최상위 UI 컨테이너
- **Window 라이프사이클**: 창의 생성부터 소멸까지 이벤트
- **대화상자**: MessageBox, 사용자 정의 대화상자, 파일 대화상자
- **NavigationWindow/Page**: 페이지 기반 내비게이션
- **Frame**: 창 내에서 페이지 호스팅
- **Window Chrome**: 창 테두리와 타이틀바 커스터마이징
- **다중 창 관리**: 여러 창의 생성과 관리
- **스플래시 스크린**: 애플리케이션 시작 화면
- **내비게이션 서비스**: MVVM 패턴에서의 내비게이션
- **MDI**: 다중 문서 인터페이스 구현