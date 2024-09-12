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