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

## ComboBox와 ListBox

### ComboBox
```xml
<!-- 기본 콤보박스 -->
<ComboBox Width="150">
    <ComboBoxItem Content="Option 1"/>
    <ComboBoxItem Content="Option 2" IsSelected="True"/>
    <ComboBoxItem Content="Option 3"/>
</ComboBox>

<!-- 편집 가능한 콤보박스 -->
<ComboBox IsEditable="True" 
          Text="Type or select"
          Width="150">
    <ComboBoxItem>Apple</ComboBoxItem>
    <ComboBoxItem>Banana</ComboBoxItem>
    <ComboBoxItem>Orange</ComboBoxItem>
</ComboBox>

<!-- 복잡한 아이템 템플릿 -->
<ComboBox Width="200">
    <ComboBox.ItemTemplate>
        <DataTemplate>
            <StackPanel Orientation="Horizontal">
                <Rectangle Width="16" Height="16" Fill="{Binding Color}"/>
                <TextBlock Text="{Binding Name}" Margin="5,0,0,0"/>
            </StackPanel>
        </DataTemplate>
    </ComboBox.ItemTemplate>
</ComboBox>
```

### ListBox
```xml
<!-- 기본 리스트박스 -->
<ListBox Height="100">
    <ListBoxItem Content="Item 1"/>
    <ListBoxItem Content="Item 2"/>
    <ListBoxItem Content="Item 3"/>
</ListBox>

<!-- 다중 선택 -->
<ListBox SelectionMode="Multiple" Height="100">
    <ListBoxItem>Red</ListBoxItem>
    <ListBoxItem>Green</ListBoxItem>
    <ListBoxItem>Blue</ListBoxItem>
    <ListBoxItem>Yellow</ListBoxItem>
</ListBox>

<!-- 가로 방향 리스트 -->
<ListBox ScrollViewer.HorizontalScrollBarVisibility="Auto"
         ScrollViewer.VerticalScrollBarVisibility="Disabled">
    <ListBox.ItemsPanel>
        <ItemsPanelTemplate>
            <StackPanel Orientation="Horizontal"/>
        </ItemsPanelTemplate>
    </ListBox.ItemsPanel>
</ListBox>
```

## ProgressBar와 Slider

### ProgressBar
```xml
<!-- 기본 프로그레스바 -->
<ProgressBar Value="60" 
             Minimum="0" 
             Maximum="100" 
             Height="20"/>

<!-- 불확정 프로그레스바 -->
<ProgressBar IsIndeterminate="True" 
             Height="20"/>

<!-- 값 바인딩 -->
<Grid>
    <Grid.RowDefinitions>
        <RowDefinition Height="Auto"/>
        <RowDefinition Height="Auto"/>
    </Grid.RowDefinitions>
    
    <ProgressBar x:Name="downloadProgress" 
                 Grid.Row="0"
                 Height="20"
                 Maximum="100"/>
    <TextBlock Grid.Row="1" 
               HorizontalAlignment="Center"
               Text="{Binding Value, ElementName=downloadProgress, StringFormat={}{0:0}%}"/>
</Grid>
```

### Slider
```xml
<!-- 기본 슬라이더 -->
<Slider Minimum="0" 
        Maximum="100" 
        Value="50"
        Width="200"/>

<!-- 틱이 있는 슬라이더 -->
<Slider Minimum="0" 
        Maximum="10" 
        TickFrequency="1"
        TickPlacement="BottomRight"
        IsSnapToTickEnabled="True"
        Width="200"/>

<!-- 수직 슬라이더 -->
<Slider Orientation="Vertical"
        Minimum="0"
        Maximum="100"
        Height="200"/>

<!-- 값 표시가 있는 슬라이더 -->
<StackPanel>
    <TextBlock Text="{Binding Value, ElementName=volumeSlider, StringFormat=Volume: {0:0}}"/>
    <Slider x:Name="volumeSlider" 
            Minimum="0" 
            Maximum="100" 
            Value="50"
            Width="200"/>
</StackPanel>
```

## Image와 MediaElement

### Image
```xml
<!-- 기본 이미지 -->
<Image Source="/Images/photo.jpg" 
       Width="200" 
       Height="150"/>

<!-- Stretch 모드 -->
<Image Source="/Images/logo.png" 
       Stretch="Uniform"
       Width="100"
       Height="100"/>

<!-- 다양한 Stretch 옵션 -->
<UniformGrid Rows="2" Columns="2">
    <Image Source="/Images/sample.jpg" Stretch="None"/>
    <Image Source="/Images/sample.jpg" Stretch="Fill"/>
    <Image Source="/Images/sample.jpg" Stretch="Uniform"/>
    <Image Source="/Images/sample.jpg" Stretch="UniformToFill"/>
</UniformGrid>

<!-- 이미지 로딩 실패 처리 -->
<Image Source="{Binding ImagePath}" 
       ImageFailed="Image_ImageFailed"/>
```

### MediaElement
```xml
<!-- 비디오 재생 -->
<Grid>
    <MediaElement x:Name="mediaPlayer" 
                  Source="/Videos/sample.mp4"
                  LoadedBehavior="Manual"
                  Width="400"
                  Height="300"/>
    
    <StackPanel Orientation="Horizontal" 
                VerticalAlignment="Bottom"
                HorizontalAlignment="Center">
        <Button Content="Play" Click="Play_Click" Margin="5"/>
        <Button Content="Pause" Click="Pause_Click" Margin="5"/>
        <Button Content="Stop" Click="Stop_Click" Margin="5"/>
    </StackPanel>
</Grid>
```

```csharp
private void Play_Click(object sender, RoutedEventArgs e)
{
    mediaPlayer.Play();
}

private void Pause_Click(object sender, RoutedEventArgs e)
{
    mediaPlayer.Pause();
}

private void Stop_Click(object sender, RoutedEventArgs e)
{
    mediaPlayer.Stop();
}
```

## ToolTip과 Popup

### ToolTip
```xml
<!-- 간단한 툴팁 -->
<Button Content="Hover me" 
        ToolTip="This is a simple tooltip"/>

<!-- 복잡한 툴팁 -->
<Button Content="Info">
    <Button.ToolTip>
        <ToolTip>
            <StackPanel>
                <TextBlock Text="Advanced Tooltip" FontWeight="Bold"/>
                <TextBlock Text="This tooltip contains multiple elements"/>
                <Image Source="/Images/info.png" Width="16" Height="16"/>
            </StackPanel>
        </ToolTip>
    </Button.ToolTip>
</Button>

<!-- 툴팁 표시 시간 설정 -->
<Button Content="Custom Timing"
        ToolTip="This tooltip has custom timing"
        ToolTipService.InitialShowDelay="1000"
        ToolTipService.ShowDuration="5000"/>
```

### Popup
```xml
<!-- 토글 버튼과 연동된 팝업 -->
<Grid>
    <ToggleButton x:Name="popupToggle" 
                  Content="Show Popup"
                  Width="100"
                  Height="30"/>
    
    <Popup IsOpen="{Binding IsChecked, ElementName=popupToggle}"
           PlacementTarget="{Binding ElementName=popupToggle}"
           Placement="Bottom"
           StaysOpen="False">
        <Border Background="LightYellow" 
                BorderBrush="Black" 
                BorderThickness="1"
                Padding="10">
            <StackPanel>
                <TextBlock Text="Popup Content"/>
                <Button Content="Close" 
                        Click="ClosePopup_Click"/>
            </StackPanel>
        </Border>
    </Popup>
</Grid>
```

## 실전 예제: 사용자 입력 폼

```xml
<Grid Margin="20">
    <Grid.RowDefinitions>
        <RowDefinition Height="Auto"/>
        <RowDefinition Height="Auto"/>
        <RowDefinition Height="Auto"/>
        <RowDefinition Height="Auto"/>
        <RowDefinition Height="Auto"/>
        <RowDefinition Height="Auto"/>
        <RowDefinition Height="Auto"/>
        <RowDefinition Height="*"/>
    </Grid.RowDefinitions>
    <Grid.ColumnDefinitions>
        <ColumnDefinition Width="100"/>
        <ColumnDefinition Width="*"/>
    </Grid.ColumnDefinitions>
    
    <!-- 이름 -->
    <Label Grid.Row="0" Grid.Column="0" Content="Name:"/>
    <TextBox Grid.Row="0" Grid.Column="1" Margin="0,2"/>
    
    <!-- 이메일 -->
    <Label Grid.Row="1" Grid.Column="0" Content="Email:"/>
    <TextBox Grid.Row="1" Grid.Column="1" Margin="0,2"/>
    
    <!-- 성별 -->
    <Label Grid.Row="2" Grid.Column="0" Content="Gender:"/>
    <StackPanel Grid.Row="2" Grid.Column="1" Orientation="Horizontal">
        <RadioButton Content="Male" Margin="0,2,10,2"/>
        <RadioButton Content="Female" Margin="0,2,10,2"/>
    </StackPanel>
    
    <!-- 국가 -->
    <Label Grid.Row="3" Grid.Column="0" Content="Country:"/>
    <ComboBox Grid.Row="3" Grid.Column="1" Margin="0,2">
        <ComboBoxItem Content="Korea"/>
        <ComboBoxItem Content="USA"/>
        <ComboBoxItem Content="Japan"/>
    </ComboBox>
    
    <!-- 관심사 -->
    <Label Grid.Row="4" Grid.Column="0" Content="Interests:"/>
    <StackPanel Grid.Row="4" Grid.Column="1">
        <CheckBox Content="Programming" Margin="0,2"/>
        <CheckBox Content="Design" Margin="0,2"/>
        <CheckBox Content="Music" Margin="0,2"/>
    </StackPanel>
    
    <!-- 경험 -->
    <Label Grid.Row="5" Grid.Column="0" Content="Experience:"/>
    <StackPanel Grid.Row="5" Grid.Column="1">
        <Slider x:Name="expSlider" 
                Minimum="0" Maximum="10" 
                TickFrequency="1" 
                TickPlacement="BottomRight"
                IsSnapToTickEnabled="True"/>
        <TextBlock Text="{Binding Value, ElementName=expSlider, StringFormat={}{0} years}"/>
    </StackPanel>
    
    <!-- 자기소개 -->
    <Label Grid.Row="6" Grid.Column="0" Content="About:"/>
    <TextBox Grid.Row="6" Grid.Column="1" 
             TextWrapping="Wrap"
             AcceptsReturn="True"
             VerticalScrollBarVisibility="Auto"
             MinHeight="100"
             Margin="0,2"/>
    
    <!-- 제출 버튼 -->
    <Button Grid.Row="7" Grid.Column="1" 
            Content="Submit"
            Width="100"
            HorizontalAlignment="Right"
            Margin="0,10,0,0"/>
</Grid>
```

## 핵심 개념 정리
- **ContentControl**: Button, Label 등 단일 콘텐츠를 가지는 컨트롤
- **ItemsControl**: ListBox, ComboBox 등 여러 아이템을 가지는 컨트롤
- **RangeBase**: Slider, ProgressBar 등 범위 값을 다루는 컨트롤
- **이벤트 처리**: Click, Checked, SelectionChanged 등
- **데이터 바인딩**: ElementName 바인딩으로 컨트롤 간 연결