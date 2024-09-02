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

## WrapPanel

요소들을 줄바꿈하며 배치하는 레이아웃입니다.

### 기본 사용법
```xml
<!-- 수평 줄바꿈 (기본값) -->
<WrapPanel Width="200">
    <Button Content="Button 1" Width="70"/>
    <Button Content="Button 2" Width="70"/>
    <Button Content="Button 3" Width="70"/>
    <Button Content="Button 4" Width="70"/>
</WrapPanel>

<!-- 수직 줄바꿈 -->
<WrapPanel Orientation="Vertical" Height="100">
    <Button Content="A" Height="30"/>
    <Button Content="B" Height="30"/>
    <Button Content="C" Height="30"/>
    <Button Content="D" Height="30"/>
</WrapPanel>
```

### ItemWidth와 ItemHeight
```xml
<WrapPanel ItemWidth="100" ItemHeight="30">
    <!-- 모든 아이템이 동일한 크기 -->
    <Button Content="Uniform 1"/>
    <Button Content="Uniform 2"/>
    <TextBox Text="Uniform 3"/>
    <CheckBox Content="Uniform 4"/>
</WrapPanel>
```

## DockPanel

요소들을 가장자리에 도킹시키는 레이아웃입니다.

### 기본 사용법
```xml
<DockPanel LastChildFill="True">
    <Menu DockPanel.Dock="Top" Height="25">
        <MenuItem Header="File"/>
        <MenuItem Header="Edit"/>
    </Menu>
    
    <StatusBar DockPanel.Dock="Bottom" Height="25">
        <TextBlock Text="Ready"/>
    </StatusBar>
    
    <ToolBar DockPanel.Dock="Left" Width="50">
        <Button Content="T1"/>
        <Button Content="T2"/>
    </ToolBar>
    
    <!-- 마지막 자식이 남은 공간 채움 -->
    <TextBox Text="Main Content Area"/>
</DockPanel>
```

### DockPanel 고급 사용
```xml
<DockPanel>
    <!-- 여러 요소를 같은 방향에 도킹 -->
    <Button DockPanel.Dock="Top" Content="Top 1"/>
    <Button DockPanel.Dock="Top" Content="Top 2"/>
    
    <!-- LastChildFill="False" -->
    <DockPanel LastChildFill="False">
        <Button DockPanel.Dock="Left" Content="Left"/>
        <Button DockPanel.Dock="Right" Content="Right"/>
        <Button Content="Not Filled"/>
    </DockPanel>
</DockPanel>
```

## Canvas

절대 좌표를 사용하는 레이아웃입니다.

### 기본 사용법
```xml
<Canvas>
    <!-- Left, Top 좌표 지정 -->
    <Rectangle Canvas.Left="50" Canvas.Top="50" 
               Width="100" Height="100" Fill="Red"/>
    
    <!-- Right, Bottom 좌표 사용 -->
    <Ellipse Canvas.Right="50" Canvas.Bottom="50"
             Width="80" Height="80" Fill="Blue"/>
    
    <!-- Z-Index로 겹침 순서 제어 -->
    <Button Canvas.Left="100" Canvas.Top="100" 
            Canvas.ZIndex="1" Content="Front"/>
    <Rectangle Canvas.Left="120" Canvas.Top="120" 
               Canvas.ZIndex="0" Width="60" Height="60" Fill="Green"/>
</Canvas>
```

### Canvas 클리핑
```xml
<Canvas ClipToBounds="True" Width="200" Height="200" Background="LightGray">
    <!-- 캔버스 경계를 벗어나는 부분은 잘림 -->
    <Rectangle Canvas.Left="150" Canvas.Top="150" 
               Width="100" Height="100" Fill="Red"/>
</Canvas>
```

## UniformGrid

모든 셀이 동일한 크기를 갖는 그리드입니다.

### 기본 사용법
```xml
<!-- 자동 행/열 계산 -->
<UniformGrid>
    <Button Content="1"/>
    <Button Content="2"/>
    <Button Content="3"/>
    <Button Content="4"/>
    <Button Content="5"/>
    <Button Content="6"/>
</UniformGrid>

<!-- 행과 열 지정 -->
<UniformGrid Rows="2" Columns="3">
    <Button Content="A" Background="LightBlue"/>
    <Button Content="B" Background="LightGreen"/>
    <Button Content="C" Background="LightYellow"/>
    <Button Content="D" Background="LightPink"/>
    <Button Content="E" Background="LightGray"/>
    <Button Content="F" Background="LightCoral"/>
</UniformGrid>
```

## ScrollViewer

스크롤 가능한 영역을 제공합니다.

### 기본 사용법
```xml
<ScrollViewer Height="200">
    <StackPanel>
        <TextBlock Text="Line 1" Height="50"/>
        <TextBlock Text="Line 2" Height="50"/>
        <TextBlock Text="Line 3" Height="50"/>
        <TextBlock Text="Line 4" Height="50"/>
        <TextBlock Text="Line 5" Height="50"/>
    </StackPanel>
</ScrollViewer>
```

### ScrollViewer 속성
```xml
<ScrollViewer HorizontalScrollBarVisibility="Auto"
              VerticalScrollBarVisibility="Visible"
              CanContentScroll="True">
    <Grid Width="500" Height="500">
        <!-- 큰 콘텐츠 -->
    </Grid>
</ScrollViewer>
```

## 레이아웃 중첩과 최적화

### 복잡한 레이아웃 구성
```xml
<Grid>
    <Grid.RowDefinitions>
        <RowDefinition Height="Auto"/>
        <RowDefinition Height="*"/>
        <RowDefinition Height="Auto"/>
    </Grid.RowDefinitions>
    
    <!-- 헤더 -->
    <Border Grid.Row="0" Background="DarkBlue" Height="50">
        <TextBlock Text="Application Header" 
                   Foreground="White" 
                   FontSize="20"
                   VerticalAlignment="Center"
                   HorizontalAlignment="Center"/>
    </Border>
    
    <!-- 메인 콘텐츠 -->
    <Grid Grid.Row="1">
        <Grid.ColumnDefinitions>
            <ColumnDefinition Width="200"/>
            <ColumnDefinition Width="*"/>
        </Grid.ColumnDefinitions>
        
        <!-- 사이드바 -->
        <ScrollViewer Grid.Column="0">
            <StackPanel Background="LightGray">
                <Button Content="Menu 1" Margin="5"/>
                <Button Content="Menu 2" Margin="5"/>
                <Button Content="Menu 3" Margin="5"/>
            </StackPanel>
        </ScrollViewer>
        
        <!-- 콘텐츠 영역 -->
        <DockPanel Grid.Column="1" Margin="10">
            <TextBlock DockPanel.Dock="Top" 
                       Text="Content Title" 
                       FontSize="18" 
                       Margin="0,0,0,10"/>
            <ScrollViewer>
                <WrapPanel>
                    <!-- 동적 콘텐츠 -->
                </WrapPanel>
            </ScrollViewer>
        </DockPanel>
    </Grid>
    
    <!-- 푸터 -->
    <StatusBar Grid.Row="2" Height="25">
        <StatusBarItem>
            <TextBlock Text="Ready"/>
        </StatusBarItem>
    </StatusBar>
</Grid>
```

### 레이아웃 성능 팁
1. **가상화 사용**: 많은 아이템을 표시할 때 VirtualizingStackPanel 활용
2. **고정 크기 지정**: 가능한 경우 Auto보다 고정 크기 사용
3. **레이아웃 깊이 최소화**: 중첩된 패널 수 줄이기
4. **적절한 패널 선택**: 용도에 맞는 레이아웃 패널 사용

## ViewBox

콘텐츠를 확대/축소하여 사용 가능한 공간에 맞춥니다.

```xml
<ViewBox Stretch="Uniform">
    <Canvas Width="100" Height="100">
        <Ellipse Width="50" Height="50" Fill="Blue" 
                 Canvas.Left="25" Canvas.Top="25"/>
        <Rectangle Width="80" Height="20" Fill="Red" 
                   Canvas.Left="10" Canvas.Top="70"/>
    </Canvas>
</ViewBox>
```

## 핵심 개념 정리
- **Grid**: 행과 열로 구성된 유연한 레이아웃
- **StackPanel**: 단순한 수직/수평 배치
- **WrapPanel**: 자동 줄바꿈 레이아웃
- **DockPanel**: 가장자리 도킹 레이아웃
- **Canvas**: 절대 좌표 기반 레이아웃
- **레이아웃 프로세스**: Measure → Arrange 단계