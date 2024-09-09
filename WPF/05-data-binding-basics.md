# 데이터 바인딩 기초

## 데이터 바인딩이란?

데이터 바인딩은 UI 요소와 데이터 소스를 자동으로 연결하는 WPF의 핵심 기능입니다.

### 데이터 바인딩의 장점
- **코드 감소**: UI 업데이트 코드 최소화
- **유지보수성**: UI와 로직의 분리
- **자동 동기화**: 데이터 변경 시 UI 자동 업데이트
- **양방향 통신**: UI에서 데이터로, 데이터에서 UI로

## Binding 표현식

### 기본 바인딩 문법
```xml
<!-- 기본 바인딩 -->
<TextBlock Text="{Binding Path=PropertyName}"/>

<!-- 축약형 -->
<TextBlock Text="{Binding PropertyName}"/>

<!-- 정적 텍스트와 함께 -->
<TextBlock>
    <TextBlock.Text>
        <Binding Path="UserName"/>
    </TextBlock.Text>
</TextBlock>
```

### ElementName 바인딩
```xml
<!-- 다른 요소의 속성에 바인딩 -->
<Slider x:Name="slider" Minimum="0" Maximum="100" Value="50"/>
<TextBlock Text="{Binding ElementName=slider, Path=Value}"/>

<!-- 여러 속성 바인딩 -->
<Rectangle Width="{Binding ElementName=slider, Path=Value}"
           Height="{Binding ElementName=slider, Path=Value}"
           Fill="Blue"/>
```

### RelativeSource 바인딩
```xml
<!-- Self: 자기 자신에게 바인딩 -->
<TextBlock Width="100"
           Text="{Binding RelativeSource={RelativeSource Self}, Path=Width}"/>

<!-- FindAncestor: 부모 요소에 바인딩 -->
<Grid Tag="Parent Grid">
    <Button Content="{Binding RelativeSource={RelativeSource FindAncestor, 
                              AncestorType={x:Type Grid}}, Path=Tag}"/>
</Grid>

<!-- TemplatedParent: 템플릿 부모에 바인딩 -->
<ControlTemplate TargetType="Button">
    <Border Background="{Binding RelativeSource={RelativeSource TemplatedParent}, 
                                Path=Background}">
        <ContentPresenter/>
    </Border>
</ControlTemplate>
```

## DataContext

### DataContext 설정
```xml
<Window x:Class="MyApp.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <Grid>
        <Grid.DataContext>
            <local:Person Name="John Doe" Age="30"/>
        </Grid.DataContext>
        
        <!-- DataContext의 속성에 바인딩 -->
        <TextBlock Text="{Binding Name}"/>
        <TextBlock Text="{Binding Age}"/>
    </Grid>
</Window>
```

### 코드에서 DataContext 설정
```csharp
public partial class MainWindow : Window
{
    public MainWindow()
    {
        InitializeComponent();
        
        // DataContext 설정
        this.DataContext = new Person 
        { 
            Name = "Jane Doe", 
            Age = 25 
        };
    }
}

public class Person
{
    public string Name { get; set; }
    public int Age { get; set; }
}
```

### DataContext 상속
```xml
<StackPanel DataContext="{Binding Person}">
    <!-- 모든 자식 요소가 같은 DataContext 상속 -->
    <TextBlock Text="{Binding Name}"/>
    <TextBlock Text="{Binding Age}"/>
    
    <!-- 개별 DataContext 재정의 -->
    <TextBlock DataContext="{Binding Address}" 
               Text="{Binding Street}"/>
</StackPanel>
```

## INotifyPropertyChanged

### INotifyPropertyChanged 구현
```csharp
using System.ComponentModel;
using System.Runtime.CompilerServices;

public class Person : INotifyPropertyChanged
{
    private string name;
    private int age;
    
    public string Name
    {
        get => name;
        set
        {
            if (name != value)
            {
                name = value;
                OnPropertyChanged();
            }
        }
    }
    
    public int Age
    {
        get => age;
        set
        {
            if (age != value)
            {
                age = value;
                OnPropertyChanged();
            }
        }
    }
    
    public event PropertyChangedEventHandler PropertyChanged;
    
    protected virtual void OnPropertyChanged([CallerMemberName] string propertyName = null)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }
}
```

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

// 사용 예제
public class PersonViewModel : ViewModelBase
{
    private string name;
    private int age;
    
    public string Name
    {
        get => name;
        set => SetProperty(ref name, value);
    }
    
    public int Age
    {
        get => age;
        set => SetProperty(ref age, value);
    }
}
```

## 바인딩 모드

### Mode 속성
```xml
<!-- OneWay: 소스 → 타겟 (기본값: 대부분의 속성) -->
<TextBlock Text="{Binding Name, Mode=OneWay}"/>

<!-- TwoWay: 소스 ↔ 타겟 (기본값: TextBox.Text) -->
<TextBox Text="{Binding Name, Mode=TwoWay}"/>

<!-- OneTime: 최초 1회만 -->
<TextBlock Text="{Binding InitialValue, Mode=OneTime}"/>

<!-- OneWayToSource: 타겟 → 소스 -->
<TextBox Text="{Binding LastInput, Mode=OneWayToSource}"/>

<!-- Default: 속성의 기본 모드 사용 -->
<TextBox Text="{Binding Name, Mode=Default}"/>
```

### UpdateSourceTrigger
```xml
<!-- PropertyChanged: 즉시 업데이트 -->
<TextBox Text="{Binding Name, UpdateSourceTrigger=PropertyChanged}"/>

<!-- LostFocus: 포커스를 잃을 때 (TextBox의 기본값) -->
<TextBox Text="{Binding Name, UpdateSourceTrigger=LostFocus}"/>

<!-- Explicit: 코드에서 명시적으로 업데이트 -->
<TextBox x:Name="explicitTextBox" 
         Text="{Binding Name, UpdateSourceTrigger=Explicit}"/>
<Button Click="UpdateButton_Click" Content="Update"/>
```

```csharp
private void UpdateButton_Click(object sender, RoutedEventArgs e)
{
    // 명시적 업데이트
    BindingExpression be = explicitTextBox.GetBindingExpression(TextBox.TextProperty);
    be?.UpdateSource();
}
```

## StringFormat

```xml
<!-- 기본 문자열 포맷 -->
<TextBlock Text="{Binding Price, StringFormat=Price: {0:C}}"/>
<TextBlock Text="{Binding Date, StringFormat={}{0:yyyy-MM-dd}}"/>
<TextBlock Text="{Binding Percentage, StringFormat={}{0:P}}"/>

<!-- 다중 바인딩과 StringFormat -->
<TextBlock>
    <TextBlock.Text>
        <MultiBinding StringFormat="{0} {1}, Age: {2}">
            <Binding Path="FirstName"/>
            <Binding Path="LastName"/>
            <Binding Path="Age"/>
        </MultiBinding>
    </TextBlock.Text>
</TextBlock>
```

## FallbackValue와 TargetNullValue

```xml
<!-- FallbackValue: 바인딩 실패 시 표시할 값 -->
<TextBlock Text="{Binding MissingProperty, FallbackValue='No data available'}"/>

<!-- TargetNullValue: null일 때 표시할 값 -->
<TextBlock Text="{Binding Description, TargetNullValue='No description'}"/>

<!-- 이미지에서 활용 -->
<Image Source="{Binding ProfileImage, 
                TargetNullValue=/Images/default-profile.png,
                FallbackValue=/Images/error.png}"/>
```

## 컬렉션 바인딩

### ObservableCollection
```csharp
public class MainViewModel : ViewModelBase
{
    public ObservableCollection<Person> People { get; }
    
    public MainViewModel()
    {
        People = new ObservableCollection<Person>
        {
            new Person { Name = "John", Age = 30 },
            new Person { Name = "Jane", Age = 25 },
            new Person { Name = "Bob", Age = 35 }
        };
    }
    
    public void AddPerson()
    {
        People.Add(new Person { Name = "New Person", Age = 20 });
    }
    
    public void RemovePerson(Person person)
    {
        People.Remove(person);
    }
}
```

### ListBox 바인딩
```xml
<ListBox ItemsSource="{Binding People}">
    <ListBox.ItemTemplate>
        <DataTemplate>
            <Border BorderBrush="Gray" BorderThickness="1" 
                    Margin="2" Padding="5">
                <StackPanel>
                    <TextBlock Text="{Binding Name}" FontWeight="Bold"/>
                    <TextBlock Text="{Binding Age, StringFormat=Age: {0}}"/>
                </StackPanel>
            </Border>
        </DataTemplate>
    </ListBox.ItemTemplate>
</ListBox>
```

## 실전 예제: 간단한 연락처 관리

### ViewModel
```csharp
public class ContactViewModel : ViewModelBase
{
    private string name;
    private string email;
    private string phone;
    private Contact selectedContact;
    
    public ObservableCollection<Contact> Contacts { get; }
    
    public string Name
    {
        get => name;
        set => SetProperty(ref name, value);
    }
    
    public string Email
    {
        get => email;
        set => SetProperty(ref email, value);
    }
    
    public string Phone
    {
        get => phone;
        set => SetProperty(ref phone, value);
    }
    
    public Contact SelectedContact
    {
        get => selectedContact;
        set
        {
            if (SetProperty(ref selectedContact, value))
            {
                // 선택된 연락처의 정보를 입력 필드에 표시
                if (value != null)
                {
                    Name = value.Name;
                    Email = value.Email;
                    Phone = value.Phone;
                }
            }
        }
    }
    
    public ContactViewModel()
    {
        Contacts = new ObservableCollection<Contact>();
    }
    
    public void AddContact()
    {
        if (!string.IsNullOrWhiteSpace(Name))
        {
            Contacts.Add(new Contact 
            { 
                Name = Name, 
                Email = Email, 
                Phone = Phone 
            });
            
            // 입력 필드 초기화
            Name = string.Empty;
            Email = string.Empty;
            Phone = string.Empty;
        }
    }
    
    public void DeleteContact()
    {
        if (SelectedContact != null)
        {
            Contacts.Remove(SelectedContact);
            SelectedContact = null;
        }
    }
}

public class Contact
{
    public string Name { get; set; }
    public string Email { get; set; }
    public string Phone { get; set; }
}
```

### View (XAML)
```xml
<Window x:Class="ContactManager.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Contact Manager" Height="400" Width="600">
    <Grid Margin="10">
        <Grid.ColumnDefinitions>
            <ColumnDefinition Width="*"/>
            <ColumnDefinition Width="250"/>
        </Grid.ColumnDefinitions>
        
        <!-- 연락처 목록 -->
        <GroupBox Grid.Column="0" Header="Contacts" Margin="0,0,5,0">
            <ListBox ItemsSource="{Binding Contacts}"
                     SelectedItem="{Binding SelectedContact}">
                <ListBox.ItemTemplate>
                    <DataTemplate>
                        <Border BorderBrush="LightGray" 
                                BorderThickness="0,0,0,1" 
                                Padding="5">
                            <StackPanel>
                                <TextBlock Text="{Binding Name}" 
                                         FontWeight="Bold"/>
                                <TextBlock Text="{Binding Email}" 
                                         FontSize="11" 
                                         Foreground="Gray"/>
                                <TextBlock Text="{Binding Phone}" 
                                         FontSize="11" 
                                         Foreground="Gray"/>
                            </StackPanel>
                        </Border>
                    </DataTemplate>
                </ListBox.ItemTemplate>
            </ListBox>
        </GroupBox>
        
        <!-- 입력 폼 -->
        <GroupBox Grid.Column="1" Header="Contact Details">
            <StackPanel Margin="5">
                <Label Content="Name:"/>
                <TextBox Text="{Binding Name, UpdateSourceTrigger=PropertyChanged}"/>
                
                <Label Content="Email:" Margin="0,10,0,0"/>
                <TextBox Text="{Binding Email, UpdateSourceTrigger=PropertyChanged}"/>
                
                <Label Content="Phone:" Margin="0,10,0,0"/>
                <TextBox Text="{Binding Phone, UpdateSourceTrigger=PropertyChanged}"/>
                
                <StackPanel Orientation="Horizontal" 
                           Margin="0,20,0,0" 
                           HorizontalAlignment="Center">
                    <Button Content="Add" 
                            Width="70" 
                            Margin="0,0,10,0"
                            Click="AddButton_Click"/>
                    <Button Content="Delete" 
                            Width="70"
                            Click="DeleteButton_Click"
                            IsEnabled="{Binding SelectedContact, 
                                       Converter={StaticResource NullToBoolConverter}}"/>
                </StackPanel>
            </StackPanel>
        </GroupBox>
    </Grid>
</Window>
```

## 핵심 개념 정리
- **Binding**: UI와 데이터를 연결하는 표현식
- **DataContext**: 바인딩의 기본 데이터 소스
- **INotifyPropertyChanged**: 속성 변경 알림 인터페이스
- **바인딩 모드**: OneWay, TwoWay, OneTime, OneWayToSource
- **ObservableCollection**: 컬렉션 변경 알림 지원