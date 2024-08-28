# XAML 기초

## XAML이란?

XAML(eXtensible Application Markup Language)은 WPF 애플리케이션의 UI를 선언적으로 정의하는 마크업 언어입니다.

### XAML의 특징
- **선언적 프로그래밍**: UI를 직관적으로 정의
- **객체 생성**: CLR 객체를 XML 형식으로 인스턴스화
- **속성 설정**: 객체의 속성을 특성(attribute)으로 설정
- **이벤트 연결**: 이벤트 핸들러를 코드 비하인드와 연결

## XAML 문법과 구조

### 기본 구조
```xml
<Window x:Class="MyApp.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="XAML Example" Height="300" Width="400">
    <Grid>
        <!-- UI 요소들 -->
    </Grid>
</Window>
```

### 요소와 속성
```xml
<!-- 속성을 특성으로 설정 -->
<Button Name="myButton" 
        Content="Click Me" 
        Width="100" 
        Height="30"
        Background="LightBlue"/>

<!-- 속성을 요소로 설정 (Property Element Syntax) -->
<Button Name="myButton2">
    <Button.Content>
        Click Me
    </Button.Content>
    <Button.Background>
        <LinearGradientBrush>
            <GradientStop Color="Yellow" Offset="0"/>
            <GradientStop Color="Red" Offset="1"/>
        </LinearGradientBrush>
    </Button.Background>
</Button>
```

### 컨텐츠 속성
```xml
<!-- 명시적 컨텐츠 -->
<Button>
    <Button.Content>
        Click Me
    </Button.Content>
</Button>

<!-- 암시적 컨텐츠 (Content Property) -->
<Button>Click Me</Button>

<!-- 복잡한 컨텐츠 -->
<Button>
    <StackPanel Orientation="Horizontal">
        <Image Source="icon.png" Width="16" Height="16"/>
        <TextBlock Text="Save" Margin="5,0,0,0"/>
    </StackPanel>
</Button>
```

## 네임스페이스

### 기본 네임스페이스
```xml
<!-- WPF 프레젠테이션 네임스페이스 -->
xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"

<!-- XAML 언어 네임스페이스 -->
xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"

<!-- 사용자 정의 네임스페이스 -->
xmlns:local="clr-namespace:MyApp"
xmlns:sys="clr-namespace:System;assembly=mscorlib"
```

### 네임스페이스 사용 예제
```xml
<Window x:Class="MyApp.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:sys="clr-namespace:System;assembly=mscorlib"
        xmlns:local="clr-namespace:MyApp">
    <Grid>
        <!-- System 네임스페이스의 String 사용 -->
        <TextBlock>
            <sys:String>Hello from System.String</sys:String>
        </TextBlock>
        
        <!-- 로컬 클래스 사용 -->
        <local:MyCustomControl/>
    </Grid>
</Window>
```

## 속성과 이벤트

### 속성 설정 방법
```xml
<!-- 단순 속성 -->
<TextBlock Text="Hello" FontSize="20" Foreground="Blue"/>

<!-- 연결된 속성 (Attached Property) -->
<Button Grid.Row="0" Grid.Column="1" Content="OK"/>

<!-- 복잡한 속성 -->
<Rectangle>
    <Rectangle.Fill>
        <RadialGradientBrush>
            <GradientStop Color="Yellow" Offset="0"/>
            <GradientStop Color="Red" Offset="1"/>
        </RadialGradientBrush>
    </Rectangle.Fill>
</Rectangle>
```

### 이벤트 연결
```xml
<!-- 이벤트 핸들러 연결 -->
<Button Click="Button_Click" 
        MouseEnter="Button_MouseEnter"
        MouseLeave="Button_MouseLeave">
    Hover Me
</Button>

<!-- 코드 비하인드 -->
```
```csharp
private void Button_Click(object sender, RoutedEventArgs e)
{
    MessageBox.Show("Button clicked!");
}

private void Button_MouseEnter(object sender, MouseEventArgs e)
{
    ((Button)sender).Background = Brushes.LightYellow;
}

private void Button_MouseLeave(object sender, MouseEventArgs e)
{
    ((Button)sender).Background = Brushes.LightGray;
}
```

## 마크업 확장

### 바인딩 마크업 확장
```xml
<!-- 데이터 바인딩 -->
<TextBox Text="{Binding Path=UserName}"/>

<!-- ElementName 바인딩 -->
<TextBlock Text="{Binding ElementName=slider, Path=Value}"/>
<Slider x:Name="slider" Minimum="0" Maximum="100"/>

<!-- RelativeSource 바인딩 -->
<TextBlock Text="{Binding RelativeSource={RelativeSource Self}, Path=ActualWidth}"/>
```

### 리소스 마크업 확장
```xml
<Window.Resources>
    <SolidColorBrush x:Key="MyBrush" Color="LightBlue"/>
    <Style x:Key="MyButtonStyle" TargetType="Button">
        <Setter Property="Background" Value="Orange"/>
        <Setter Property="Foreground" Value="White"/>
    </Style>
</Window.Resources>

<Grid>
    <!-- StaticResource 사용 -->
    <Rectangle Fill="{StaticResource MyBrush}"/>
    <Button Style="{StaticResource MyButtonStyle}" Content="Styled Button"/>
    
    <!-- DynamicResource 사용 -->
    <Rectangle Fill="{DynamicResource MyBrush}"/>
</Grid>
```

### 타입 마크업 확장
```xml
<!-- x:Type 사용 -->
<Style TargetType="{x:Type Button}">
    <Setter Property="Margin" Value="5"/>
</Style>

<!-- x:Static 사용 -->
<TextBlock Text="{x:Static SystemParameters.PrimaryScreenWidth}"/>

<!-- x:Null 사용 -->
<Button Background="{x:Null}" Content="Transparent Button"/>
```

## 컬렉션과 리스트

```xml
<!-- 명시적 컬렉션 -->
<ListBox>
    <ListBox.Items>
        <ListBoxItem>Item 1</ListBoxItem>
        <ListBoxItem>Item 2</ListBoxItem>
        <ListBoxItem>Item 3</ListBoxItem>
    </ListBox.Items>
</ListBox>

<!-- 암시적 컬렉션 -->
<ListBox>
    <ListBoxItem>Item 1</ListBoxItem>
    <ListBoxItem>Item 2</ListBoxItem>
    <ListBoxItem>Item 3</ListBoxItem>
</ListBox>

<!-- 복잡한 아이템 -->
<ListBox>
    <ListBoxItem>
        <StackPanel Orientation="Horizontal">
            <Ellipse Width="10" Height="10" Fill="Red"/>
            <TextBlock Text="Red Item" Margin="5,0,0,0"/>
        </StackPanel>
    </ListBoxItem>
    <ListBoxItem>
        <StackPanel Orientation="Horizontal">
            <Ellipse Width="10" Height="10" Fill="Blue"/>
            <TextBlock Text="Blue Item" Margin="5,0,0,0"/>
        </StackPanel>
    </ListBoxItem>
</ListBox>
```

## XAML 컴파일 과정

### BAML (Binary Application Markup Language)
- XAML은 컴파일 시 BAML로 변환
- 더 빠른 로딩과 파싱
- 리소스로 어셈블리에 포함

### x:Name vs Name
```xml
<!-- x:Name - XAML 네임스페이스, 모든 요소에 사용 가능 -->
<Rectangle x:Name="myRect" Fill="Red"/>

<!-- Name - FrameworkElement의 속성 -->
<Button Name="myButton" Content="Click"/>

<!-- 코드 비하인드에서 접근 -->
```
```csharp
public MainWindow()
{
    InitializeComponent();
    myRect.Fill = Brushes.Blue;
    myButton.Click += (s, e) => { };
}
```

## 실전 예제: 계산기 UI

```xml
<Window x:Class="Calculator.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Calculator" Height="400" Width="300">
    <Grid Margin="10">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
        </Grid.RowDefinitions>
        
        <!-- 디스플레이 -->
        <TextBox Grid.Row="0" 
                 x:Name="display"
                 FontSize="24" 
                 TextAlignment="Right"
                 Margin="0,0,0,10"
                 IsReadOnly="True"/>
        
        <!-- 버튼 그리드 -->
        <Grid Grid.Row="1">
            <Grid.RowDefinitions>
                <RowDefinition Height="*"/>
                <RowDefinition Height="*"/>
                <RowDefinition Height="*"/>
                <RowDefinition Height="*"/>
            </Grid.RowDefinitions>
            <Grid.ColumnDefinitions>
                <ColumnDefinition Width="*"/>
                <ColumnDefinition Width="*"/>
                <ColumnDefinition Width="*"/>
                <ColumnDefinition Width="*"/>
            </Grid.ColumnDefinitions>
            
            <!-- 숫자 버튼 -->
            <Button Grid.Row="0" Grid.Column="0" Content="7" Click="Number_Click"/>
            <Button Grid.Row="0" Grid.Column="1" Content="8" Click="Number_Click"/>
            <Button Grid.Row="0" Grid.Column="2" Content="9" Click="Number_Click"/>
            <Button Grid.Row="0" Grid.Column="3" Content="/" Click="Operator_Click"/>
            
            <Button Grid.Row="1" Grid.Column="0" Content="4" Click="Number_Click"/>
            <Button Grid.Row="1" Grid.Column="1" Content="5" Click="Number_Click"/>
            <Button Grid.Row="1" Grid.Column="2" Content="6" Click="Number_Click"/>
            <Button Grid.Row="1" Grid.Column="3" Content="*" Click="Operator_Click"/>
            
            <Button Grid.Row="2" Grid.Column="0" Content="1" Click="Number_Click"/>
            <Button Grid.Row="2" Grid.Column="1" Content="2" Click="Number_Click"/>
            <Button Grid.Row="2" Grid.Column="2" Content="3" Click="Number_Click"/>
            <Button Grid.Row="2" Grid.Column="3" Content="-" Click="Operator_Click"/>
            
            <Button Grid.Row="3" Grid.Column="0" Content="0" Click="Number_Click"/>
            <Button Grid.Row="3" Grid.Column="1" Content="C" Click="Clear_Click"/>
            <Button Grid.Row="3" Grid.Column="2" Content="=" Click="Equals_Click"/>
            <Button Grid.Row="3" Grid.Column="3" Content="+" Click="Operator_Click"/>
        </Grid>
    </Grid>
</Window>
```

## 핵심 개념 정리
- **XAML**: UI를 선언적으로 정의하는 마크업 언어
- **네임스페이스**: CLR 타입을 XAML에서 사용하기 위한 매핑
- **속성 요소**: 복잡한 속성값을 설정하는 문법
- **마크업 확장**: {}로 둘러싸인 특별한 XAML 기능
- **BAML**: 컴파일된 XAML의 이진 형식