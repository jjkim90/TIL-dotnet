# 레이아웃 시스템

## WPF 레이아웃 개요

WPF의 레이아웃 시스템은 UI 요소들의 크기와 위치를 자동으로 관리합니다.

### 레이아웃의 특징
- **자동 크기 조절**: 컨텐츠에 따른 동적 크기
- **해상도 독립성**: DPI에 관계없이 일관된 UI
- **유연한 배치**: 다양한 화면 크기 대응
- **중첩 가능**: 레이아웃 패널 조합

### 레이아웃 프로세스
1. **Measure**: 필요한 크기 계산
2. **Arrange**: 실제 위치와 크기 결정

## Grid

Grid는 가장 강력하고 유연한 레이아웃 패널입니다.

### 기본 사용법
```xml
<Grid>
    <Grid.RowDefinitions>
        <RowDefinition Height="Auto"/>
        <RowDefinition Height="*"/>
        <RowDefinition Height="100"/>
    </Grid.RowDefinitions>
    <Grid.ColumnDefinitions>
        <ColumnDefinition Width="200"/>
        <ColumnDefinition Width="*"/>
        <ColumnDefinition Width="2*"/>
    </Grid.ColumnDefinitions>
    
    <TextBlock Grid.Row="0" Grid.Column="0" Text="(0,0)"/>
    <Button Grid.Row="1" Grid.Column="1" Content="Center"/>
    <Rectangle Grid.Row="2" Grid.Column="2" Fill="Blue"/>
</Grid>
```

### 크기 지정 방법
```xml
<!-- 고정 크기 -->
<RowDefinition Height="100"/>

<!-- 자동 크기 (content에 맞춤) -->
<RowDefinition Height="Auto"/>

<!-- 비례 크기 (Star sizing) -->
<RowDefinition Height="*"/>      <!-- 1* -->
<RowDefinition Height="2*"/>     <!-- 2배 -->

<!-- 최소/최대 크기 -->
<RowDefinition Height="*" MinHeight="50" MaxHeight="200"/>
```

### Grid Span
```xml
<Grid>
    <Grid.RowDefinitions>
        <RowDefinition/>
        <RowDefinition/>
        <RowDefinition/>
    </Grid.RowDefinitions>
    <Grid.ColumnDefinitions>
        <ColumnDefinition/>
        <ColumnDefinition/>
        <ColumnDefinition/>
    </Grid.ColumnDefinitions>
    
    <!-- 2개 열 차지 -->
    <Button Grid.Column="0" Grid.ColumnSpan="2" Content="Wide Button"/>
    
    <!-- 2개 행 차지 -->
    <TextBox Grid.Row="1" Grid.RowSpan="2" Grid.Column="2"/>
</Grid>
```

## StackPanel

요소들을 수직 또는 수평으로 쌓는 레이아웃입니다.

### 기본 사용법
```xml
<!-- 수직 정렬 (기본값) -->
<StackPanel>
    <Button Content="Button 1"/>
    <Button Content="Button 2"/>
    <Button Content="Button 3"/>
</StackPanel>

<!-- 수평 정렬 -->
<StackPanel Orientation="Horizontal">
    <Button Content="Left"/>
    <Button Content="Center"/>
    <Button Content="Right"/>
</StackPanel>
```

### StackPanel의 특성
```xml
<StackPanel Orientation="Vertical">
    <!-- 수직 방향: Width는 늘어나지만 Height는 컨텐츠에 맞춤 -->
    <Button Content="Full Width" HorizontalAlignment="Stretch"/>
    <Button Content="Center" HorizontalAlignment="Center"/>
    <Button Content="Left" HorizontalAlignment="Left"/>
</StackPanel>
```