# 기본 컨트롤

## WPF 컨트롤 개요

WPF 컨트롤들은 사용자와 상호작용하는 UI 요소들입니다.

### 컨트롤 계층 구조
```
Object
└── DispatcherObject
    └── DependencyObject
        └── Visual
            └── UIElement
                └── FrameworkElement
                    └── Control
                        ├── ContentControl
                        └── ItemsControl
```

## Button과 RepeatButton

### Button
```xml
<!-- 기본 버튼 -->
<Button Content="Click Me" 
        Width="100" 
        Height="30"
        Click="Button_Click"/>

<!-- 이미지가 있는 버튼 -->
<Button Width="120" Height="40">
    <StackPanel Orientation="Horizontal">
        <Image Source="/Images/save.png" Width="16" Height="16"/>
        <TextBlock Text="Save" Margin="5,0,0,0" VerticalAlignment="Center"/>
    </StackPanel>
</Button>

<!-- 스타일이 적용된 버튼 -->
<Button Content="Styled Button"
        Background="DarkBlue"
        Foreground="White"
        FontWeight="Bold"
        BorderBrush="Navy"
        BorderThickness="2"/>
```

### RepeatButton
```xml
<!-- 누르고 있는 동안 계속 이벤트 발생 -->
<RepeatButton Content="+" 
              Width="30" 
              Height="30"
              Delay="500"
              Interval="100"
              Click="RepeatButton_Click"/>
```

### 코드 비하인드
```csharp
private int counter = 0;

private void Button_Click(object sender, RoutedEventArgs e)
{
    MessageBox.Show("Button clicked!");
}

private void RepeatButton_Click(object sender, RoutedEventArgs e)
{
    counter++;
    counterLabel.Content = counter.ToString();
}
```

## TextBox와 PasswordBox

### TextBox
```xml
<!-- 기본 텍스트박스 -->
<TextBox Text="Enter text here" 
         Width="200" 
         Height="25"/>

<!-- 여러 줄 텍스트박스 -->
<TextBox TextWrapping="Wrap" 
         AcceptsReturn="True"
         VerticalScrollBarVisibility="Auto"
         Height="100"
         Width="300"/>

<!-- 워터마크가 있는 텍스트박스 -->
<Grid>
    <TextBox x:Name="searchBox" Width="200"/>
    <TextBlock Text="Search..." 
               Foreground="Gray"
               IsHitTestVisible="False">
        <TextBlock.Style>
            <Style TargetType="TextBlock">
                <Setter Property="Visibility" Value="Collapsed"/>
                <Style.Triggers>
                    <DataTrigger Binding="{Binding Text, ElementName=searchBox}" Value="">
                        <Setter Property="Visibility" Value="Visible"/>
                    </DataTrigger>
                </Style.Triggers>
            </Style>
        </TextBlock.Style>
    </TextBlock>
</Grid>
```

### PasswordBox
```xml
<PasswordBox x:Name="passwordBox"
             Width="200"
             PasswordChar="●"
             MaxLength="20"/>

<!-- 코드에서 비밀번호 가져오기 -->
<Button Content="Login" Click="Login_Click"/>
```

```csharp
private void Login_Click(object sender, RoutedEventArgs e)
{
    string password = passwordBox.Password;
    // 보안 주의: 실제로는 SecureString 사용 권장
}
```

## Label

```xml
<!-- 기본 레이블 -->
<Label Content="Name:" />

<!-- 액세스 키가 있는 레이블 -->
<Label Content="_Username:" Target="{Binding ElementName=usernameBox}"/>
<TextBox x:Name="usernameBox"/>

<!-- 복잡한 콘텐츠 -->
<Label>
    <StackPanel Orientation="Horizontal">
        <Image Source="/Images/info.png" Width="16" Height="16"/>
        <TextBlock Text="Information" Margin="5,0,0,0"/>
    </StackPanel>
</Label>
```

## CheckBox와 RadioButton

### CheckBox
```xml
<!-- 기본 체크박스 -->
<CheckBox Content="I agree to the terms" 
          IsChecked="True"/>

<!-- 3상태 체크박스 -->
<CheckBox Content="Select All" 
          IsThreeState="True"
          IsChecked="{x:Null}"/>

<!-- 이벤트 처리 -->
<CheckBox Content="Enable feature" 
          Checked="CheckBox_Checked"
          Unchecked="CheckBox_Unchecked"
          Indeterminate="CheckBox_Indeterminate"/>
```

### RadioButton
```xml
<!-- 라디오 버튼 그룹 -->
<StackPanel>
    <RadioButton GroupName="Size" Content="Small" IsChecked="True"/>
    <RadioButton GroupName="Size" Content="Medium"/>
    <RadioButton GroupName="Size" Content="Large"/>
</StackPanel>

<!-- 다른 그룹 -->
<StackPanel>
    <RadioButton GroupName="Color" Content="Red"/>
    <RadioButton GroupName="Color" Content="Green"/>
    <RadioButton GroupName="Color" Content="Blue"/>
</StackPanel>
```