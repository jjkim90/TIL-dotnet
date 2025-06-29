# 스타일과 템플릿

## Style 기초

스타일은 컨트롤의 속성을 일괄적으로 설정하는 방법입니다.

### 기본 스타일 정의
```xml
<Window.Resources>
    <!-- Button 스타일 정의 -->
    <Style x:Key="MyButtonStyle" TargetType="Button">
        <Setter Property="Background" Value="LightBlue"/>
        <Setter Property="Foreground" Value="DarkBlue"/>
        <Setter Property="FontSize" Value="14"/>
        <Setter Property="Padding" Value="10,5"/>
        <Setter Property="Margin" Value="5"/>
    </Style>
</Window.Resources>

<!-- 스타일 적용 -->
<Button Style="{StaticResource MyButtonStyle}" Content="Styled Button"/>
```

### 암시적 스타일 (x:Key 없이)
```xml
<Window.Resources>
    <!-- 모든 Button에 자동 적용 -->
    <Style TargetType="Button">
        <Setter Property="Background" Value="LightGray"/>
        <Setter Property="Height" Value="30"/>
        <Setter Property="Width" Value="100"/>
    </Style>
</Window.Resources>

<!-- 자동으로 스타일 적용됨 -->
<Button Content="Auto Styled"/>

<!-- 명시적으로 스타일 무시 -->
<Button Content="No Style" Style="{x:Null}"/>
```

### 스타일 상속
```xml
<Window.Resources>
    <!-- 기본 스타일 -->
    <Style x:Key="BaseButtonStyle" TargetType="Button">
        <Setter Property="FontFamily" Value="Arial"/>
        <Setter Property="FontSize" Value="12"/>
        <Setter Property="Margin" Value="5"/>
    </Style>
    
    <!-- 상속받은 스타일 -->
    <Style x:Key="PrimaryButtonStyle" 
           TargetType="Button" 
           BasedOn="{StaticResource BaseButtonStyle}">
        <Setter Property="Background" Value="Blue"/>
        <Setter Property="Foreground" Value="White"/>
    </Style>
    
    <Style x:Key="SecondaryButtonStyle" 
           TargetType="Button" 
           BasedOn="{StaticResource BaseButtonStyle}">
        <Setter Property="Background" Value="Gray"/>
        <Setter Property="Foreground" Value="Black"/>
    </Style>
</Window.Resources>
```

## Setter와 Trigger

### Property Trigger
```xml
<Style x:Key="HoverButtonStyle" TargetType="Button">
    <Setter Property="Background" Value="LightGray"/>
    <Setter Property="Foreground" Value="Black"/>
    
    <Style.Triggers>
        <!-- 마우스 오버 시 -->
        <Trigger Property="IsMouseOver" Value="True">
            <Setter Property="Background" Value="DarkGray"/>
            <Setter Property="Foreground" Value="White"/>
            <Setter Property="Cursor" Value="Hand"/>
        </Trigger>
        
        <!-- 눌렸을 때 -->
        <Trigger Property="IsPressed" Value="True">
            <Setter Property="Background" Value="Black"/>
        </Trigger>
        
        <!-- 비활성화 상태 -->
        <Trigger Property="IsEnabled" Value="False">
            <Setter Property="Opacity" Value="0.5"/>
        </Trigger>
    </Style.Triggers>
</Style>
```

### MultiTrigger
```xml
<Style x:Key="ConditionalStyle" TargetType="TextBox">
    <Setter Property="Background" Value="White"/>
    
    <Style.Triggers>
        <!-- 여러 조건을 모두 만족할 때 -->
        <MultiTrigger>
            <MultiTrigger.Conditions>
                <Condition Property="IsMouseOver" Value="True"/>
                <Condition Property="IsFocused" Value="True"/>
            </MultiTrigger.Conditions>
            <Setter Property="Background" Value="LightYellow"/>
            <Setter Property="BorderBrush" Value="Orange"/>
        </MultiTrigger>
    </Style.Triggers>
</Style>
```

### DataTrigger
```xml
<Style x:Key="DataBoundStyle" TargetType="TextBlock">
    <Setter Property="Foreground" Value="Black"/>
    
    <Style.Triggers>
        <!-- 바인딩된 데이터에 따른 트리거 -->
        <DataTrigger Binding="{Binding Path=Status}" Value="Error">
            <Setter Property="Foreground" Value="Red"/>
            <Setter Property="FontWeight" Value="Bold"/>
        </DataTrigger>
        
        <DataTrigger Binding="{Binding Path=Count}" Value="0">
            <Setter Property="Visibility" Value="Collapsed"/>
        </DataTrigger>
        
        <!-- 범위 확인을 위한 Converter 사용 -->
        <DataTrigger Binding="{Binding Path=Temperature, 
                               Converter={StaticResource GreaterThanConverter}, 
                               ConverterParameter=30}" 
                     Value="True">
            <Setter Property="Foreground" Value="Red"/>
            <Setter Property="Text" Value="High Temperature!"/>
        </DataTrigger>
    </Style.Triggers>
</Style>
```

### EventTrigger
```xml
<Style x:Key="AnimatedButtonStyle" TargetType="Button">
    <Setter Property="Background" Value="LightBlue"/>
    
    <Style.Triggers>
        <!-- 마우스 진입 시 애니메이션 -->
        <EventTrigger RoutedEvent="MouseEnter">
            <BeginStoryboard>
                <Storyboard>
                    <DoubleAnimation Storyboard.TargetProperty="FontSize"
                                   To="18" Duration="0:0:0.3"/>
                    <ColorAnimation Storyboard.TargetProperty="(Background).(SolidColorBrush.Color)"
                                  To="DarkBlue" Duration="0:0:0.3"/>
                </Storyboard>
            </BeginStoryboard>
        </EventTrigger>
        
        <!-- 마우스 나갈 때 애니메이션 -->
        <EventTrigger RoutedEvent="MouseLeave">
            <BeginStoryboard>
                <Storyboard>
                    <DoubleAnimation Storyboard.TargetProperty="FontSize"
                                   To="14" Duration="0:0:0.3"/>
                    <ColorAnimation Storyboard.TargetProperty="(Background).(SolidColorBrush.Color)"
                                  To="LightBlue" Duration="0:0:0.3"/>
                </Storyboard>
            </BeginStoryboard>
        </EventTrigger>
    </Style.Triggers>
</Style>
```

## ControlTemplate

### 기본 ControlTemplate
```xml
<Style x:Key="CustomButtonStyle" TargetType="Button">
    <Setter Property="Template">
        <Setter.Value>
            <ControlTemplate TargetType="Button">
                <!-- 버튼의 시각적 구조 재정의 -->
                <Border x:Name="border"
                        Background="{TemplateBinding Background}"
                        BorderBrush="{TemplateBinding BorderBrush}"
                        BorderThickness="{TemplateBinding BorderThickness}"
                        CornerRadius="5">
                    <ContentPresenter HorizontalAlignment="Center"
                                    VerticalAlignment="Center"/>
                </Border>
                
                <!-- 템플릿 트리거 -->
                <ControlTemplate.Triggers>
                    <Trigger Property="IsMouseOver" Value="True">
                        <Setter TargetName="border" Property="Background" Value="LightGray"/>
                    </Trigger>
                    <Trigger Property="IsPressed" Value="True">
                        <Setter TargetName="border" Property="Background" Value="DarkGray"/>
                    </Trigger>
                </ControlTemplate.Triggers>
            </ControlTemplate>
        </Setter.Value>
    </Setter>
</Style>
```

### 원형 버튼 템플릿
```xml
<Style x:Key="CircleButtonStyle" TargetType="Button">
    <Setter Property="Width" Value="50"/>
    <Setter Property="Height" Value="50"/>
    <Setter Property="Template">
        <Setter.Value>
            <ControlTemplate TargetType="Button">
                <Grid>
                    <!-- 원형 배경 -->
                    <Ellipse x:Name="background"
                             Fill="{TemplateBinding Background}"
                             Stroke="{TemplateBinding BorderBrush}"
                             StrokeThickness="2"/>
                    
                    <!-- 내용 -->
                    <ContentPresenter HorizontalAlignment="Center"
                                    VerticalAlignment="Center"/>
                </Grid>
                
                <ControlTemplate.Triggers>
                    <Trigger Property="IsMouseOver" Value="True">
                        <Setter TargetName="background" Property="Fill" Value="Orange"/>
                    </Trigger>
                    <Trigger Property="IsPressed" Value="True">
                        <Setter TargetName="background" Property="Fill" Value="DarkOrange"/>
                        <Setter TargetName="background" Property="RenderTransform">
                            <Setter.Value>
                                <ScaleTransform ScaleX="0.95" ScaleY="0.95" 
                                              CenterX="25" CenterY="25"/>
                            </Setter.Value>
                        </Setter>
                    </Trigger>
                </ControlTemplate.Triggers>
            </ControlTemplate>
        </Setter.Value>
    </Setter>
</Style>
```

### CheckBox 커스텀 템플릿
```xml
<Style x:Key="SwitchCheckBoxStyle" TargetType="CheckBox">
    <Setter Property="Template">
        <Setter.Value>
            <ControlTemplate TargetType="CheckBox">
                <Grid>
                    <Grid.ColumnDefinitions>
                        <ColumnDefinition Width="Auto"/>
                        <ColumnDefinition Width="*"/>
                    </Grid.ColumnDefinitions>
                    
                    <!-- 스위치 -->
                    <Border x:Name="switchBorder"
                            Grid.Column="0"
                            Width="40" Height="20"
                            Background="LightGray"
                            CornerRadius="10"
                            Margin="0,0,5,0">
                        <Canvas>
                            <Ellipse x:Name="switchButton"
                                   Width="16" Height="16"
                                   Fill="White"
                                   Canvas.Left="2" Canvas.Top="2"/>
                        </Canvas>
                    </Border>
                    
                    <!-- 내용 -->
                    <ContentPresenter Grid.Column="1"
                                    VerticalAlignment="Center"/>
                </Grid>
                
                <ControlTemplate.Triggers>
                    <Trigger Property="IsChecked" Value="True">
                        <Setter TargetName="switchBorder" Property="Background" Value="LightGreen"/>
                        <Setter TargetName="switchButton" Property="Canvas.Left" Value="22"/>
                    </Trigger>
                </ControlTemplate.Triggers>
            </ControlTemplate>
        </Setter.Value>
    </Setter>
</Style>
```

## DataTemplate

### 기본 DataTemplate
```xml
<!-- ListBox용 DataTemplate -->
<ListBox ItemsSource="{Binding People}">
    <ListBox.ItemTemplate>
        <DataTemplate>
            <Border BorderBrush="Gray" BorderThickness="1" 
                    CornerRadius="5" Padding="10" Margin="5">
                <StackPanel>
                    <TextBlock Text="{Binding Name}" 
                             FontSize="16" FontWeight="Bold"/>
                    <TextBlock Text="{Binding Age, StringFormat='Age: {0}'}" 
                             FontSize="14" Foreground="Gray"/>
                    <TextBlock Text="{Binding Email}" 
                             FontSize="12" Foreground="Blue"/>
                </StackPanel>
            </Border>
        </DataTemplate>
    </ListBox.ItemTemplate>
</ListBox>
```

### DataType을 사용한 DataTemplate
```xml
<Window.Resources>
    <!-- Person 타입에 대한 DataTemplate -->
    <DataTemplate DataType="{x:Type local:Person}">
        <Border Background="LightBlue" Padding="5" Margin="2">
            <StackPanel Orientation="Horizontal">
                <TextBlock Text="{Binding Name}" Margin="0,0,10,0"/>
                <TextBlock Text="{Binding Age}" Foreground="Gray"/>
            </StackPanel>
        </Border>
    </DataTemplate>
    
    <!-- Product 타입에 대한 DataTemplate -->
    <DataTemplate DataType="{x:Type local:Product}">
        <Border Background="LightGreen" Padding="5" Margin="2">
            <StackPanel>
                <TextBlock Text="{Binding ProductName}" FontWeight="Bold"/>
                <TextBlock Text="{Binding Price, StringFormat='{}{0:C}'}"/>
            </StackPanel>
        </Border>
    </DataTemplate>
</Window.Resources>

<!-- 자동으로 적절한 템플릿 선택 -->
<ContentControl Content="{Binding SelectedItem}"/>
```

### DataTemplateSelector
```csharp
public class PersonDataTemplateSelector : DataTemplateSelector
{
    public DataTemplate AdultTemplate { get; set; }
    public DataTemplate ChildTemplate { get; set; }
    
    public override DataTemplate SelectTemplate(object item, DependencyObject container)
    {
        if (item is Person person)
        {
            return person.Age >= 18 ? AdultTemplate : ChildTemplate;
        }
        
        return base.SelectTemplate(item, container);
    }
}
```

```xml
<Window.Resources>
    <!-- 성인용 템플릿 -->
    <DataTemplate x:Key="AdultTemplate">
        <Border Background="LightBlue" Padding="5">
            <TextBlock>
                <Run Text="{Binding Name}"/>
                <Run Text="(Adult)" Foreground="Blue"/>
            </TextBlock>
        </Border>
    </DataTemplate>
    
    <!-- 미성년자용 템플릿 -->
    <DataTemplate x:Key="ChildTemplate">
        <Border Background="LightPink" Padding="5">
            <TextBlock>
                <Run Text="{Binding Name}"/>
                <Run Text="(Child)" Foreground="Red"/>
            </TextBlock>
        </Border>
    </DataTemplate>
    
    <!-- DataTemplateSelector -->
    <local:PersonDataTemplateSelector x:Key="personTemplateSelector"
                                     AdultTemplate="{StaticResource AdultTemplate}"
                                     ChildTemplate="{StaticResource ChildTemplate}"/>
</Window.Resources>

<ListBox ItemsSource="{Binding People}"
         ItemTemplateSelector="{StaticResource personTemplateSelector}"/>
```

## ItemsPanel Template

### 기본 ItemsPanelTemplate
```xml
<!-- 수평 ListBox -->
<ListBox ItemsSource="{Binding Items}">
    <ListBox.ItemsPanel>
        <ItemsPanelTemplate>
            <StackPanel Orientation="Horizontal"/>
        </ItemsPanelTemplate>
    </ListBox.ItemsPanel>
</ListBox>

<!-- WrapPanel을 사용한 ListBox -->
<ListBox ItemsSource="{Binding Items}" ScrollViewer.HorizontalScrollBarVisibility="Disabled">
    <ListBox.ItemsPanel>
        <ItemsPanelTemplate>
            <WrapPanel Orientation="Horizontal" ItemWidth="100" ItemHeight="100"/>
        </ItemsPanelTemplate>
    </ListBox.ItemsPanel>
</ListBox>

<!-- UniformGrid를 사용한 ItemsControl -->
<ItemsControl ItemsSource="{Binding Items}">
    <ItemsControl.ItemsPanel>
        <ItemsPanelTemplate>
            <UniformGrid Columns="4"/>
        </ItemsPanelTemplate>
    </ItemsControl.ItemsPanel>
</ItemsControl>
```

## 실전 예제: 테마 시스템

### 라이트/다크 테마 스타일
```xml
<!-- 라이트 테마 -->
<ResourceDictionary x:Key="LightTheme">
    <SolidColorBrush x:Key="BackgroundBrush" Color="White"/>
    <SolidColorBrush x:Key="ForegroundBrush" Color="Black"/>
    <SolidColorBrush x:Key="AccentBrush" Color="#007ACC"/>
    <SolidColorBrush x:Key="BorderBrush" Color="LightGray"/>
    
    <Style x:Key="ThemedButtonStyle" TargetType="Button">
        <Setter Property="Background" Value="{DynamicResource BackgroundBrush}"/>
        <Setter Property="Foreground" Value="{DynamicResource ForegroundBrush}"/>
        <Setter Property="BorderBrush" Value="{DynamicResource BorderBrush}"/>
        <Setter Property="Padding" Value="10,5"/>
        <Setter Property="Template">
            <Setter.Value>
                <ControlTemplate TargetType="Button">
                    <Border Background="{TemplateBinding Background}"
                            BorderBrush="{TemplateBinding BorderBrush}"
                            BorderThickness="1"
                            CornerRadius="3">
                        <ContentPresenter HorizontalAlignment="Center"
                                        VerticalAlignment="Center"/>
                    </Border>
                    <ControlTemplate.Triggers>
                        <Trigger Property="IsMouseOver" Value="True">
                            <Setter Property="Background" Value="{DynamicResource AccentBrush}"/>
                            <Setter Property="Foreground" Value="White"/>
                        </Trigger>
                    </ControlTemplate.Triggers>
                </ControlTemplate>
            </Setter.Value>
        </Setter>
    </Style>
</ResourceDictionary>

<!-- 다크 테마 -->
<ResourceDictionary x:Key="DarkTheme">
    <SolidColorBrush x:Key="BackgroundBrush" Color="#1E1E1E"/>
    <SolidColorBrush x:Key="ForegroundBrush" Color="White"/>
    <SolidColorBrush x:Key="AccentBrush" Color="#00A0E9"/>
    <SolidColorBrush x:Key="BorderBrush" Color="#3C3C3C"/>
    
    <!-- 동일한 키의 스타일이지만 다른 색상 -->
    <Style x:Key="ThemedButtonStyle" TargetType="Button">
        <!-- 라이트 테마와 동일한 구조, 다른 색상 -->
    </Style>
</ResourceDictionary>
```

### 머티리얼 디자인 스타일 카드
```xml
<Style x:Key="MaterialCardStyle" TargetType="Border">
    <Setter Property="Background" Value="White"/>
    <Setter Property="CornerRadius" Value="4"/>
    <Setter Property="Padding" Value="16"/>
    <Setter Property="Margin" Value="8"/>
    <Setter Property="Effect">
        <Setter.Value>
            <DropShadowEffect Color="Black" 
                            Direction="270" 
                            BlurRadius="10" 
                            ShadowDepth="2" 
                            Opacity="0.3"/>
        </Setter.Value>
    </Setter>
    <Style.Triggers>
        <EventTrigger RoutedEvent="MouseEnter">
            <BeginStoryboard>
                <Storyboard>
                    <DoubleAnimation Storyboard.TargetProperty="Effect.ShadowDepth"
                                   To="8" Duration="0:0:0.2"/>
                    <DoubleAnimation Storyboard.TargetProperty="Effect.BlurRadius"
                                   To="20" Duration="0:0:0.2"/>
                </Storyboard>
            </BeginStoryboard>
        </EventTrigger>
        <EventTrigger RoutedEvent="MouseLeave">
            <BeginStoryboard>
                <Storyboard>
                    <DoubleAnimation Storyboard.TargetProperty="Effect.ShadowDepth"
                                   To="2" Duration="0:0:0.2"/>
                    <DoubleAnimation Storyboard.TargetProperty="Effect.BlurRadius"
                                   To="10" Duration="0:0:0.2"/>
                </Storyboard>
            </BeginStoryboard>
        </EventTrigger>
    </Style.Triggers>
</Style>
```

## 고급 템플릿 기법

### TemplateBinding vs Binding
```xml
<ControlTemplate TargetType="Button">
    <!-- TemplateBinding: 가벼움, OneWay만 지원 -->
    <Border Background="{TemplateBinding Background}">
        
        <!-- Binding with RelativeSource: 더 유연함 -->
        <TextBlock Text="{Binding RelativeSource={RelativeSource TemplatedParent}, 
                                 Path=Content, 
                                 Converter={StaticResource UpperCaseConverter}}"/>
    </Border>
</ControlTemplate>
```

### ContentPresenter vs ContentControl
```xml
<ControlTemplate TargetType="Button">
    <Grid>
        <!-- ContentPresenter: 가벼움, 템플릿 내에서 사용 -->
        <ContentPresenter Content="{TemplateBinding Content}"
                        ContentTemplate="{TemplateBinding ContentTemplate}"
                        HorizontalAlignment="Center"
                        VerticalAlignment="Center"/>
        
        <!-- ContentControl: 더 많은 기능, 독립적 사용 가능 -->
        <!-- 일반적으로 ControlTemplate 내에서는 ContentPresenter 권장 -->
    </Grid>
</ControlTemplate>
```

### Visual State Manager (VSM)
```xml
<Style x:Key="ModernButtonStyle" TargetType="Button">
    <Setter Property="Template">
        <Setter.Value>
            <ControlTemplate TargetType="Button">
                <Grid x:Name="RootGrid">
                    <VisualStateManager.VisualStateGroups>
                        <VisualStateGroup x:Name="CommonStates">
                            <VisualState x:Name="Normal">
                                <Storyboard>
                                    <ColorAnimation Storyboard.TargetName="BackgroundBorder"
                                                  Storyboard.TargetProperty="(Background).(SolidColorBrush.Color)"
                                                  To="LightGray" Duration="0:0:0.1"/>
                                </Storyboard>
                            </VisualState>
                            <VisualState x:Name="MouseOver">
                                <Storyboard>
                                    <ColorAnimation Storyboard.TargetName="BackgroundBorder"
                                                  Storyboard.TargetProperty="(Background).(SolidColorBrush.Color)"
                                                  To="DarkGray" Duration="0:0:0.1"/>
                                </Storyboard>
                            </VisualState>
                            <VisualState x:Name="Pressed">
                                <Storyboard>
                                    <ColorAnimation Storyboard.TargetName="BackgroundBorder"
                                                  Storyboard.TargetProperty="(Background).(SolidColorBrush.Color)"
                                                  To="Black" Duration="0:0:0.1"/>
                                </Storyboard>
                            </VisualState>
                        </VisualStateGroup>
                    </VisualStateManager.VisualStateGroups>
                    
                    <Border x:Name="BackgroundBorder" 
                            Background="LightGray"
                            CornerRadius="4">
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

## 핵심 개념 정리
- **Style**: 컨트롤의 속성을 일괄 설정
- **Trigger**: 조건에 따른 동적 스타일 변경
- **ControlTemplate**: 컨트롤의 시각적 구조 재정의
- **DataTemplate**: 데이터 객체의 시각적 표현 정의
- **TemplateBinding**: 템플릿 내에서 부모 속성 참조
- **Visual State Manager**: 상태별 시각적 변화 관리