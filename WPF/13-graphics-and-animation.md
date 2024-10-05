# 그래픽과 애니메이션

## WPF 그래픽 기초

WPF는 강력한 벡터 기반 그래픽 시스템을 제공합니다.

### Shape 클래스
모든 도형의 기본 클래스로, 다음과 같은 공통 속성을 제공합니다:
- **Fill**: 도형 내부 채우기
- **Stroke**: 테두리 색상
- **StrokeThickness**: 테두리 두께
- **StrokeDashArray**: 점선 패턴

## 기본 도형

### Line (선)
```xml
<Line X1="10" Y1="10" X2="200" Y2="100"
      Stroke="Black" StrokeThickness="2"/>

<!-- 점선 -->
<Line X1="10" Y1="50" X2="200" Y2="50"
      Stroke="Blue" StrokeThickness="2"
      StrokeDashArray="5,3"/>
```

### Rectangle (사각형)
```xml
<!-- 기본 사각형 -->
<Rectangle Width="100" Height="50"
           Fill="LightBlue" Stroke="DarkBlue" StrokeThickness="2"/>

<!-- 둥근 모서리 -->
<Rectangle Width="100" Height="50"
           RadiusX="10" RadiusY="10"
           Fill="LightGreen" Stroke="DarkGreen"/>
```

### Ellipse (타원)
```xml
<!-- 원 -->
<Ellipse Width="100" Height="100"
         Fill="Yellow" Stroke="Orange" StrokeThickness="3"/>

<!-- 타원 -->
<Ellipse Width="150" Height="100"
         Fill="Pink" Stroke="Red" StrokeThickness="2"/>
```

### Polygon (다각형)
```xml
<!-- 삼각형 -->
<Polygon Points="50,10 10,90 90,90"
         Fill="LightCoral" Stroke="DarkRed" StrokeThickness="2"/>

<!-- 별 모양 -->
<Polygon Points="50,0 61,35 98,35 68,57 79,91 50,70 21,91 32,57 2,35 39,35"
         Fill="Gold" Stroke="Black" StrokeThickness="1"/>
```

### Polyline (연결선)
```xml
<Polyline Points="10,10 30,30 50,10 70,30 90,10"
          Stroke="Purple" StrokeThickness="3"/>
```

### Path (경로)
```xml
<!-- 간단한 경로 -->
<Path Stroke="Black" StrokeThickness="2" Fill="LightYellow">
    <Path.Data>
        <PathGeometry>
            <PathFigure StartPoint="10,10">
                <LineSegment Point="100,10"/>
                <LineSegment Point="100,100"/>
                <LineSegment Point="10,100"/>
                <LineSegment Point="10,10"/>
            </PathFigure>
        </PathGeometry>
    </Path.Data>
</Path>

<!-- 미니 언어 사용 -->
<Path Data="M 10,10 L 100,10 L 100,100 L 10,100 Z"
      Fill="LightBlue" Stroke="Blue" StrokeThickness="2"/>

<!-- 곡선 경로 -->
<Path Data="M 10,50 Q 50,10 90,50 T 170,50"
      Stroke="Green" StrokeThickness="3" Fill="Transparent"/>
```

## 브러시 (Brushes)

### SolidColorBrush
```xml
<Rectangle Width="100" Height="100">
    <Rectangle.Fill>
        <SolidColorBrush Color="#FF6B6B"/>
    </Rectangle.Fill>
</Rectangle>
```

### LinearGradientBrush
```xml
<!-- 수평 그라데이션 -->
<Rectangle Width="200" Height="100">
    <Rectangle.Fill>
        <LinearGradientBrush StartPoint="0,0" EndPoint="1,0">
            <GradientStop Color="Yellow" Offset="0"/>
            <GradientStop Color="Orange" Offset="0.5"/>
            <GradientStop Color="Red" Offset="1"/>
        </LinearGradientBrush>
    </Rectangle.Fill>
</Rectangle>

<!-- 대각선 그라데이션 -->
<Ellipse Width="150" Height="150">
    <Ellipse.Fill>
        <LinearGradientBrush StartPoint="0,0" EndPoint="1,1">
            <GradientStop Color="Blue" Offset="0"/>
            <GradientStop Color="Cyan" Offset="0.5"/>
            <GradientStop Color="White" Offset="1"/>
        </LinearGradientBrush>
    </Ellipse.Fill>
</Ellipse>
```

### RadialGradientBrush
```xml
<Ellipse Width="200" Height="200">
    <Ellipse.Fill>
        <RadialGradientBrush GradientOrigin="0.5,0.5" Center="0.5,0.5">
            <GradientStop Color="White" Offset="0"/>
            <GradientStop Color="LightBlue" Offset="0.5"/>
            <GradientStop Color="DarkBlue" Offset="1"/>
        </RadialGradientBrush>
    </Ellipse.Fill>
</Ellipse>

<!-- 오프셋 중심 -->
<Rectangle Width="200" Height="200">
    <Rectangle.Fill>
        <RadialGradientBrush GradientOrigin="0.2,0.2">
            <GradientStop Color="Yellow" Offset="0"/>
            <GradientStop Color="Orange" Offset="0.5"/>
            <GradientStop Color="Red" Offset="1"/>
        </RadialGradientBrush>
    </Rectangle.Fill>
</Rectangle>
```

### ImageBrush
```xml
<Rectangle Width="300" Height="200">
    <Rectangle.Fill>
        <ImageBrush ImageSource="/Images/background.jpg"
                    Stretch="UniformToFill"/>
    </Rectangle.Fill>
</Rectangle>

<!-- 타일 패턴 -->
<Rectangle Width="400" Height="300">
    <Rectangle.Fill>
        <ImageBrush ImageSource="/Images/pattern.png"
                    TileMode="Tile"
                    Viewport="0,0,50,50"
                    ViewportUnits="Absolute"/>
    </Rectangle.Fill>
</Rectangle>
```

### DrawingBrush
```xml
<Rectangle Width="300" Height="200">
    <Rectangle.Fill>
        <DrawingBrush Viewport="0,0,100,100" ViewportUnits="Absolute" TileMode="Tile">
            <DrawingBrush.Drawing>
                <DrawingGroup>
                    <GeometryDrawing Brush="LightBlue">
                        <GeometryDrawing.Geometry>
                            <RectangleGeometry Rect="0,0,50,50"/>
                        </GeometryDrawing.Geometry>
                    </GeometryDrawing>
                    <GeometryDrawing Brush="Blue">
                        <GeometryDrawing.Geometry>
                            <RectangleGeometry Rect="50,50,50,50"/>
                        </GeometryDrawing.Geometry>
                    </GeometryDrawing>
                </DrawingGroup>
            </DrawingBrush.Drawing>
        </DrawingBrush>
    </Rectangle.Fill>
</Rectangle>
```

### VisualBrush
```xml
<!-- 요소를 브러시로 사용 -->
<Rectangle Width="200" Height="200">
    <Rectangle.Fill>
        <VisualBrush>
            <VisualBrush.Visual>
                <StackPanel Background="White">
                    <TextBlock Text="WPF" FontSize="40" FontWeight="Bold"/>
                    <Button Content="Click Me" Margin="10"/>
                </StackPanel>
            </VisualBrush.Visual>
        </VisualBrush>
    </Rectangle.Fill>
</Rectangle>

<!-- 반사 효과 -->
<StackPanel>
    <Border x:Name="originalBorder" Background="LightBlue" 
            BorderBrush="DarkBlue" BorderThickness="2"
            Width="200" Height="100">
        <TextBlock Text="Reflection" HorizontalAlignment="Center" 
                   VerticalAlignment="Center" FontSize="24"/>
    </Border>
    
    <Rectangle Height="100" Width="200" Margin="0,2,0,0">
        <Rectangle.Fill>
            <VisualBrush Visual="{Binding ElementName=originalBorder}" 
                         Opacity="0.3">
                <VisualBrush.RelativeTransform>
                    <ScaleTransform ScaleY="-1" CenterY="0.5"/>
                </VisualBrush.RelativeTransform>
            </VisualBrush>
        </Rectangle.Fill>
        <Rectangle.OpacityMask>
            <LinearGradientBrush StartPoint="0,0" EndPoint="0,1">
                <GradientStop Color="Transparent" Offset="0"/>
                <GradientStop Color="Black" Offset="1"/>
            </LinearGradientBrush>
        </Rectangle.OpacityMask>
    </Rectangle>
</StackPanel>
```

## Transform (변환)

### TranslateTransform (이동)
```xml
<Rectangle Width="100" Height="50" Fill="Red">
    <Rectangle.RenderTransform>
        <TranslateTransform X="50" Y="20"/>
    </Rectangle.RenderTransform>
</Rectangle>
```

### RotateTransform (회전)
```xml
<Rectangle Width="100" Height="50" Fill="Blue">
    <Rectangle.RenderTransform>
        <RotateTransform Angle="45" CenterX="50" CenterY="25"/>
    </Rectangle.RenderTransform>
</Rectangle>
```

### ScaleTransform (크기 조정)
```xml
<Rectangle Width="100" Height="50" Fill="Green">
    <Rectangle.RenderTransform>
        <ScaleTransform ScaleX="1.5" ScaleY="2" CenterX="50" CenterY="25"/>
    </Rectangle.RenderTransform>
</Rectangle>
```

### SkewTransform (기울이기)
```xml
<Rectangle Width="100" Height="50" Fill="Orange">
    <Rectangle.RenderTransform>
        <SkewTransform AngleX="30" AngleY="0"/>
    </Rectangle.RenderTransform>
</Rectangle>
```

### TransformGroup (복합 변환)
```xml
<Rectangle Width="100" Height="50" Fill="Purple">
    <Rectangle.RenderTransform>
        <TransformGroup>
            <RotateTransform Angle="45"/>
            <ScaleTransform ScaleX="1.5" ScaleY="1.5"/>
            <TranslateTransform X="100" Y="50"/>
        </TransformGroup>
    </Rectangle.RenderTransform>
</Rectangle>
```

### MatrixTransform
```xml
<Rectangle Width="100" Height="50" Fill="Brown">
    <Rectangle.RenderTransform>
        <!-- Matrix: M11,M12,M21,M22,OffsetX,OffsetY -->
        <MatrixTransform Matrix="1,0.5,0,1,50,0"/>
    </Rectangle.RenderTransform>
</Rectangle>
```

## 애니메이션 기초

### Storyboard
```xml
<Window.Resources>
    <!-- 이동 애니메이션 -->
    <Storyboard x:Key="MoveAnimation">
        <DoubleAnimation Storyboard.TargetName="movingRect"
                         Storyboard.TargetProperty="(Canvas.Left)"
                         From="0" To="300" Duration="0:0:2"/>
    </Storyboard>
    
    <!-- 회전 애니메이션 -->
    <Storyboard x:Key="RotateAnimation">
        <DoubleAnimation Storyboard.TargetName="rotatingRect"
                         Storyboard.TargetProperty="(Rectangle.RenderTransform).(RotateTransform.Angle)"
                         From="0" To="360" Duration="0:0:3"
                         RepeatBehavior="Forever"/>
    </Storyboard>
</Window.Resources>

<Canvas>
    <Rectangle x:Name="movingRect" Width="50" Height="50" 
               Fill="Red" Canvas.Left="0" Canvas.Top="50"/>
    
    <Rectangle x:Name="rotatingRect" Width="50" Height="50" 
               Fill="Blue" Canvas.Left="150" Canvas.Top="150">
        <Rectangle.RenderTransform>
            <RotateTransform CenterX="25" CenterY="25"/>
        </Rectangle.RenderTransform>
    </Rectangle>
</Canvas>
```

### 애니메이션 트리거
```xml
<Rectangle Width="100" Height="100" Fill="Green">
    <Rectangle.Triggers>
        <EventTrigger RoutedEvent="MouseEnter">
            <BeginStoryboard>
                <Storyboard>
                    <DoubleAnimation Storyboard.TargetProperty="Width"
                                     To="150" Duration="0:0:0.3"/>
                    <DoubleAnimation Storyboard.TargetProperty="Height"
                                     To="150" Duration="0:0:0.3"/>
                </Storyboard>
            </BeginStoryboard>
        </EventTrigger>
        <EventTrigger RoutedEvent="MouseLeave">
            <BeginStoryboard>
                <Storyboard>
                    <DoubleAnimation Storyboard.TargetProperty="Width"
                                     To="100" Duration="0:0:0.3"/>
                    <DoubleAnimation Storyboard.TargetProperty="Height"
                                     To="100" Duration="0:0:0.3"/>
                </Storyboard>
            </BeginStoryboard>
        </EventTrigger>
    </Rectangle.Triggers>
</Rectangle>
```

## 애니메이션 타입

### DoubleAnimation
```xml
<!-- 투명도 애니메이션 -->
<Rectangle Width="100" Height="100" Fill="Red" x:Name="fadingRect">
    <Rectangle.Triggers>
        <EventTrigger RoutedEvent="Loaded">
            <BeginStoryboard>
                <Storyboard>
                    <DoubleAnimation Storyboard.TargetProperty="Opacity"
                                     From="1.0" To="0.0" Duration="0:0:2"
                                     AutoReverse="True" RepeatBehavior="Forever"/>
                </Storyboard>
            </BeginStoryboard>
        </EventTrigger>
    </Rectangle.Triggers>
</Rectangle>

<!-- 크기 애니메이션 -->
<Button Content="Animate Me" Width="100" Height="30">
    <Button.Triggers>
        <EventTrigger RoutedEvent="Click">
            <BeginStoryboard>
                <Storyboard>
                    <DoubleAnimation Storyboard.TargetProperty="Width"
                                     To="200" Duration="0:0:0.5"
                                     AccelerationRatio="0.5"
                                     DecelerationRatio="0.5"/>
                </Storyboard>
            </BeginStoryboard>
        </EventTrigger>
    </Button.Triggers>
</Button>
```

### ColorAnimation
```xml
<Rectangle Width="200" Height="100">
    <Rectangle.Fill>
        <SolidColorBrush x:Name="animatedBrush" Color="Blue"/>
    </Rectangle.Fill>
    <Rectangle.Triggers>
        <EventTrigger RoutedEvent="Loaded">
            <BeginStoryboard>
                <Storyboard>
                    <ColorAnimation Storyboard.TargetName="animatedBrush"
                                    Storyboard.TargetProperty="Color"
                                    From="Blue" To="Red" Duration="0:0:3"
                                    RepeatBehavior="Forever" AutoReverse="True"/>
                </Storyboard>
            </BeginStoryboard>
        </EventTrigger>
    </Rectangle.Triggers>
</Rectangle>

<!-- 그라데이션 색상 애니메이션 -->
<Rectangle Width="300" Height="100">
    <Rectangle.Fill>
        <LinearGradientBrush>
            <GradientStop Color="Yellow" Offset="0" x:Name="gradStop1"/>
            <GradientStop Color="Orange" Offset="1" x:Name="gradStop2"/>
        </LinearGradientBrush>
    </Rectangle.Fill>
    <Rectangle.Triggers>
        <EventTrigger RoutedEvent="Loaded">
            <BeginStoryboard>
                <Storyboard>
                    <ColorAnimation Storyboard.TargetName="gradStop1"
                                    Storyboard.TargetProperty="Color"
                                    To="Red" Duration="0:0:2"
                                    RepeatBehavior="Forever" AutoReverse="True"/>
                    <ColorAnimation Storyboard.TargetName="gradStop2"
                                    Storyboard.TargetProperty="Color"
                                    To="Purple" Duration="0:0:2"
                                    RepeatBehavior="Forever" AutoReverse="True"/>
                </Storyboard>
            </BeginStoryboard>
        </EventTrigger>
    </Rectangle.Triggers>
</Rectangle>
```

### PointAnimation
```xml
<Path Fill="Blue" Stroke="Black" StrokeThickness="2">
    <Path.Data>
        <PathGeometry>
            <PathFigure StartPoint="10,10">
                <LineSegment x:Name="animatedLineSegment" Point="100,50"/>
            </PathFigure>
        </PathGeometry>
    </Path.Data>
    <Path.Triggers>
        <EventTrigger RoutedEvent="Loaded">
            <BeginStoryboard>
                <Storyboard>
                    <PointAnimation Storyboard.TargetName="animatedLineSegment"
                                    Storyboard.TargetProperty="Point"
                                    From="100,50" To="200,150" Duration="0:0:2"
                                    RepeatBehavior="Forever" AutoReverse="True"/>
                </Storyboard>
            </BeginStoryboard>
        </EventTrigger>
    </Path.Triggers>
</Path>
```

### VectorAnimation
```xml
<Rectangle Width="100" Height="100" Fill="Green">
    <Rectangle.RenderTransform>
        <TranslateTransform x:Name="translateTransform"/>
    </Rectangle.RenderTransform>
    <Rectangle.Triggers>
        <EventTrigger RoutedEvent="Loaded">
            <BeginStoryboard>
                <Storyboard>
                    <VectorAnimation Storyboard.TargetName="translateTransform"
                                     Storyboard.TargetProperty="(TranslateTransform.X)"
                                     From="0,0" To="100,100" Duration="0:0:3"/>
                </Storyboard>
            </BeginStoryboard>
        </EventTrigger>
    </Rectangle.Triggers>
</Rectangle>
```

## 키프레임 애니메이션

### DoubleAnimationUsingKeyFrames
```xml
<Rectangle Width="50" Height="50" Fill="Red" Canvas.Left="0" Canvas.Top="50">
    <Rectangle.Triggers>
        <EventTrigger RoutedEvent="Loaded">
            <BeginStoryboard>
                <Storyboard>
                    <DoubleAnimationUsingKeyFrames 
                        Storyboard.TargetProperty="(Canvas.Left)"
                        RepeatBehavior="Forever">
                        <LinearDoubleKeyFrame KeyTime="0:0:0" Value="0"/>
                        <LinearDoubleKeyFrame KeyTime="0:0:1" Value="100"/>
                        <SplineDoubleKeyFrame KeyTime="0:0:2" Value="300">
                            <SplineDoubleKeyFrame.KeySpline>
                                <KeySpline ControlPoint1="0.5,0" ControlPoint2="0.5,1"/>
                            </SplineDoubleKeyFrame.KeySpline>
                        </SplineDoubleKeyFrame>
                        <DiscreteDoubleKeyFrame KeyTime="0:0:3" Value="0"/>
                    </DoubleAnimationUsingKeyFrames>
                </Storyboard>
            </BeginStoryboard>
        </EventTrigger>
    </Rectangle.Triggers>
</Rectangle>
```

### ColorAnimationUsingKeyFrames
```xml
<Rectangle Width="200" Height="100">
    <Rectangle.Fill>
        <SolidColorBrush x:Name="keyframeBrush" Color="Blue"/>
    </Rectangle.Fill>
    <Rectangle.Triggers>
        <EventTrigger RoutedEvent="Loaded">
            <BeginStoryboard>
                <Storyboard>
                    <ColorAnimationUsingKeyFrames 
                        Storyboard.TargetName="keyframeBrush"
                        Storyboard.TargetProperty="Color"
                        RepeatBehavior="Forever">
                        <LinearColorKeyFrame KeyTime="0:0:0" Value="Blue"/>
                        <LinearColorKeyFrame KeyTime="0:0:1" Value="Green"/>
                        <LinearColorKeyFrame KeyTime="0:0:2" Value="Yellow"/>
                        <DiscreteColorKeyFrame KeyTime="0:0:3" Value="Red"/>
                        <SplineColorKeyFrame KeyTime="0:0:4" Value="Blue">
                            <SplineColorKeyFrame.KeySpline>
                                <KeySpline ControlPoint1="0,0" ControlPoint2="1,1"/>
                            </SplineColorKeyFrame.KeySpline>
                        </SplineColorKeyFrame>
                    </ColorAnimationUsingKeyFrames>
                </Storyboard>
            </BeginStoryboard>
        </EventTrigger>
    </Rectangle.Triggers>
</Rectangle>
```

## 경로 애니메이션

### DoubleAnimationUsingPath
```xml
<Canvas>
    <Path x:Name="motionPath" Data="M 10,100 Q 100,10 200,100 T 400,100"
          Stroke="Gray" StrokeThickness="2" StrokeDashArray="5,2"/>
    
    <Rectangle Width="30" Height="30" Fill="Red">
        <Rectangle.RenderTransform>
            <TransformGroup>
                <TranslateTransform x:Name="translateTransform"/>
                <RotateTransform x:Name="rotateTransform" CenterX="15" CenterY="15"/>
            </TransformGroup>
        </Rectangle.RenderTransform>
        <Rectangle.Triggers>
            <EventTrigger RoutedEvent="Loaded">
                <BeginStoryboard>
                    <Storyboard RepeatBehavior="Forever">
                        <!-- X 좌표 애니메이션 -->
                        <DoubleAnimationUsingPath
                            Storyboard.TargetName="translateTransform"
                            Storyboard.TargetProperty="X"
                            PathGeometry="M 10,100 Q 100,10 200,100 T 400,100"
                            Source="X" Duration="0:0:5"/>
                        
                        <!-- Y 좌표 애니메이션 -->
                        <DoubleAnimationUsingPath
                            Storyboard.TargetName="translateTransform"
                            Storyboard.TargetProperty="Y"
                            PathGeometry="M 10,100 Q 100,10 200,100 T 400,100"
                            Source="Y" Duration="0:0:5"/>
                        
                        <!-- 회전 애니메이션 -->
                        <DoubleAnimationUsingPath
                            Storyboard.TargetName="rotateTransform"
                            Storyboard.TargetProperty="Angle"
                            PathGeometry="M 10,100 Q 100,10 200,100 T 400,100"
                            Source="Angle" Duration="0:0:5"/>
                    </Storyboard>
                </BeginStoryboard>
            </EventTrigger>
        </Rectangle.Triggers>
    </Rectangle>
</Canvas>
```

## 3D 그래픽

### Viewport3D 기초
```xml
<Viewport3D>
    <!-- 카메라 -->
    <Viewport3D.Camera>
        <PerspectiveCamera Position="0,0,5" LookDirection="0,0,-1" 
                          FieldOfView="45"/>
    </Viewport3D.Camera>
    
    <!-- 조명 -->
    <ModelVisual3D>
        <ModelVisual3D.Content>
            <DirectionalLight Color="White" Direction="-1,-1,-1"/>
        </ModelVisual3D.Content>
    </ModelVisual3D>
    
    <!-- 3D 모델 -->
    <ModelVisual3D>
        <ModelVisual3D.Content>
            <GeometryModel3D>
                <!-- 메시 -->
                <GeometryModel3D.Geometry>
                    <MeshGeometry3D
                        Positions="-1,-1,0 1,-1,0 1,1,0 -1,1,0"
                        TriangleIndices="0,1,2 0,2,3"
                        TextureCoordinates="0,1 1,1 1,0 0,0"/>
                </GeometryModel3D.Geometry>
                
                <!-- 재질 -->
                <GeometryModel3D.Material>
                    <DiffuseMaterial>
                        <DiffuseMaterial.Brush>
                            <LinearGradientBrush>
                                <GradientStop Color="Blue" Offset="0"/>
                                <GradientStop Color="Red" Offset="1"/>
                            </LinearGradientBrush>
                        </DiffuseMaterial.Brush>
                    </DiffuseMaterial>
                </GeometryModel3D.Material>
            </GeometryModel3D>
        </ModelVisual3D.Content>
    </ModelVisual3D>
</Viewport3D>
```

### 3D 회전 애니메이션
```xml
<Viewport3D>
    <Viewport3D.Camera>
        <PerspectiveCamera Position="0,0,5" LookDirection="0,0,-1"/>
    </Viewport3D.Camera>
    
    <ModelVisual3D>
        <ModelVisual3D.Content>
            <DirectionalLight Color="White" Direction="-1,-1,-1"/>
        </ModelVisual3D.Content>
    </ModelVisual3D>
    
    <ModelVisual3D>
        <ModelVisual3D.Content>
            <GeometryModel3D>
                <GeometryModel3D.Geometry>
                    <!-- 큐브 정의 -->
                    <MeshGeometry3D
                        Positions="-1,-1,-1 1,-1,-1 1,1,-1 -1,1,-1
                                   -1,-1,1 1,-1,1 1,1,1 -1,1,1"
                        TriangleIndices="0,1,2 0,2,3
                                        4,7,6 4,6,5
                                        0,4,5 0,5,1
                                        2,6,7 2,7,3
                                        0,3,7 0,7,4
                                        1,5,6 1,6,2"/>
                </GeometryModel3D.Geometry>
                
                <GeometryModel3D.Material>
                    <DiffuseMaterial Brush="Red"/>
                </GeometryModel3D.Material>
                
                <GeometryModel3D.Transform>
                    <RotateTransform3D>
                        <RotateTransform3D.Rotation>
                            <AxisAngleRotation3D x:Name="rotation" 
                                               Axis="0,1,0" Angle="0"/>
                        </RotateTransform3D.Rotation>
                    </RotateTransform3D>
                </GeometryModel3D.Transform>
            </GeometryModel3D>
        </ModelVisual3D.Content>
        
        <ModelVisual3D.Triggers>
            <EventTrigger RoutedEvent="Loaded">
                <BeginStoryboard>
                    <Storyboard>
                        <DoubleAnimation Storyboard.TargetName="rotation"
                                       Storyboard.TargetProperty="Angle"
                                       From="0" To="360" Duration="0:0:5"
                                       RepeatBehavior="Forever"/>
                    </Storyboard>
                </BeginStoryboard>
            </EventTrigger>
        </ModelVisual3D.Triggers>
    </ModelVisual3D>
</Viewport3D>
```

## 고급 그래픽 효과

### BitmapEffects (레거시)
```xml
<!-- BlurBitmapEffect -->
<Image Source="/Images/photo.jpg" Width="200" Height="150">
    <Image.BitmapEffect>
        <BlurBitmapEffect Radius="5"/>
    </Image.BitmapEffect>
</Image>
```

### Effects (권장)
```xml
<!-- DropShadowEffect -->
<TextBlock Text="Shadow Text" FontSize="36" FontWeight="Bold">
    <TextBlock.Effect>
        <DropShadowEffect Color="Gray" Direction="320" 
                         ShadowDepth="5" Opacity="0.5"/>
    </TextBlock.Effect>
</TextBlock>

<!-- BlurEffect -->
<Button Content="Blurred Button" Width="150" Height="50">
    <Button.Effect>
        <BlurEffect Radius="3"/>
    </Button.Effect>
</Button>
```

### 사용자 정의 셰이더 효과
```csharp
public class GrayscaleEffect : ShaderEffect
{
    private static PixelShader _pixelShader = 
        new PixelShader() { UriSource = new Uri("pack://application:,,,/GrayscaleEffect.ps") };
    
    public GrayscaleEffect()
    {
        PixelShader = _pixelShader;
        UpdateShaderValue(InputProperty);
    }
    
    public static readonly DependencyProperty InputProperty = 
        ShaderEffect.RegisterPixelShaderSamplerProperty("Input", typeof(GrayscaleEffect), 0);
    
    public Brush Input
    {
        get => (Brush)GetValue(InputProperty);
        set => SetValue(InputProperty, value);
    }
}
```

## Drawing과 Visual

### DrawingVisual 사용
```csharp
public class CustomDrawingElement : FrameworkElement
{
    private VisualCollection _visuals;
    
    public CustomDrawingElement()
    {
        _visuals = new VisualCollection(this);
        CreateDrawing();
    }
    
    private void CreateDrawing()
    {
        DrawingVisual drawingVisual = new DrawingVisual();
        
        using (DrawingContext dc = drawingVisual.RenderOpen())
        {
            // 사각형 그리기
            dc.DrawRectangle(Brushes.LightBlue, new Pen(Brushes.Blue, 2), 
                           new Rect(10, 10, 100, 100));
            
            // 원 그리기
            dc.DrawEllipse(Brushes.Yellow, new Pen(Brushes.Orange, 2), 
                         new Point(60, 60), 40, 40);
            
            // 텍스트 그리기
            FormattedText formattedText = new FormattedText(
                "Custom Drawing",
                CultureInfo.CurrentCulture,
                FlowDirection.LeftToRight,
                new Typeface("Arial"),
                16,
                Brushes.Black,
                VisualTreeHelper.GetDpi(this).PixelsPerDip);
            
            dc.DrawText(formattedText, new Point(10, 120));
        }
        
        _visuals.Add(drawingVisual);
    }
    
    protected override int VisualChildrenCount => _visuals.Count;
    
    protected override Visual GetVisualChild(int index) => _visuals[index];
}
```

### GeometryDrawing
```xml
<Image Width="200" Height="200">
    <Image.Source>
        <DrawingImage>
            <DrawingImage.Drawing>
                <DrawingGroup>
                    <!-- 배경 -->
                    <GeometryDrawing Brush="LightGray">
                        <GeometryDrawing.Geometry>
                            <RectangleGeometry Rect="0,0,200,200"/>
                        </GeometryDrawing.Geometry>
                    </GeometryDrawing>
                    
                    <!-- 원 -->
                    <GeometryDrawing Brush="Yellow" 
                                   Pen="{x:Null}">
                        <GeometryDrawing.Geometry>
                            <EllipseGeometry Center="100,100" RadiusX="80" RadiusY="80"/>
                        </GeometryDrawing.Geometry>
                    </GeometryDrawing>
                    
                    <!-- 얼굴 -->
                    <GeometryDrawing Brush="Black">
                        <GeometryDrawing.Geometry>
                            <GeometryGroup>
                                <EllipseGeometry Center="70,80" RadiusX="10" RadiusY="10"/>
                                <EllipseGeometry Center="130,80" RadiusX="10" RadiusY="10"/>
                            </GeometryGroup>
                        </GeometryDrawing.Geometry>
                    </GeometryDrawing>
                    
                    <!-- 입 -->
                    <GeometryDrawing Pen="{StaticResource {x:Static SystemColors.ControlTextBrushKey}}" 
                                   Brush="{x:Null}">
                        <GeometryDrawing.Pen>
                            <Pen Brush="Black" Thickness="3"/>
                        </GeometryDrawing.Pen>
                        <GeometryDrawing.Geometry>
                            <PathGeometry>
                                <PathFigure StartPoint="60,120">
                                    <ArcSegment Point="140,120" Size="40,30" 
                                              SweepDirection="Clockwise"/>
                                </PathFigure>
                            </PathGeometry>
                        </GeometryDrawing.Geometry>
                    </GeometryDrawing>
                </DrawingGroup>
            </DrawingImage.Drawing>
        </DrawingImage>
    </Image.Source>
</Image>
```

## 실전 예제: 차트 그리기

```csharp
public class SimpleChart : FrameworkElement
{
    public static readonly DependencyProperty DataProperty =
        DependencyProperty.Register("Data", typeof(List<double>), typeof(SimpleChart),
            new FrameworkPropertyMetadata(null, FrameworkPropertyMetadataOptions.AffectsRender));
    
    public List<double> Data
    {
        get => (List<double>)GetValue(DataProperty);
        set => SetValue(DataProperty, value);
    }
    
    protected override void OnRender(DrawingContext drawingContext)
    {
        base.OnRender(drawingContext);
        
        if (Data == null || Data.Count == 0) return;
        
        double width = ActualWidth;
        double height = ActualHeight;
        double max = Data.Max();
        double barWidth = width / Data.Count;
        
        for (int i = 0; i < Data.Count; i++)
        {
            double barHeight = (Data[i] / max) * height * 0.8;
            double x = i * barWidth + barWidth * 0.1;
            double y = height - barHeight;
            double w = barWidth * 0.8;
            
            // 막대 그리기
            var rect = new Rect(x, y, w, barHeight);
            var gradient = new LinearGradientBrush(Colors.Blue, Colors.LightBlue, 90);
            drawingContext.DrawRectangle(gradient, new Pen(Brushes.DarkBlue, 1), rect);
            
            // 값 표시
            var text = new FormattedText(
                Data[i].ToString("F1"),
                CultureInfo.CurrentCulture,
                FlowDirection.LeftToRight,
                new Typeface("Arial"),
                12,
                Brushes.Black,
                VisualTreeHelper.GetDpi(this).PixelsPerDip);
            
            drawingContext.DrawText(text, new Point(x + w/2 - text.Width/2, y - 20));
        }
    }
}
```

```xml
<!-- 차트 사용 -->
<local:SimpleChart Width="400" Height="300">
    <local:SimpleChart.Data>
        <x:Array Type="sys:Double">
            <sys:Double>45</sys:Double>
            <sys:Double>78</sys:Double>
            <sys:Double>62</sys:Double>
            <sys:Double>93</sys:Double>
            <sys:Double>55</sys:Double>
        </x:Array>
    </local:SimpleChart.Data>
</local:SimpleChart>
```

## 실전 예제: 로딩 애니메이션

```xml
<UserControl x:Class="LoadingSpinner">
    <Grid Width="100" Height="100">
        <Canvas RenderTransformOrigin="0.5,0.5">
            <Canvas.RenderTransform>
                <RotateTransform x:Name="spinnerRotate" Angle="0"/>
            </Canvas.RenderTransform>
            <Canvas.Triggers>
                <EventTrigger RoutedEvent="Loaded">
                    <BeginStoryboard>
                        <Storyboard>
                            <DoubleAnimation Storyboard.TargetName="spinnerRotate"
                                           Storyboard.TargetProperty="Angle"
                                           From="0" To="360" Duration="0:0:2"
                                           RepeatBehavior="Forever"/>
                        </Storyboard>
                    </BeginStoryboard>
                </EventTrigger>
            </Canvas.Triggers>
            
            <!-- 12개의 점으로 구성된 스피너 -->
            <Ellipse Width="10" Height="10" Fill="Black" Opacity="1.0" 
                     Canvas.Left="45" Canvas.Top="0"/>
            <Ellipse Width="10" Height="10" Fill="Black" Opacity="0.9" 
                     Canvas.Left="70" Canvas.Top="7"/>
            <Ellipse Width="10" Height="10" Fill="Black" Opacity="0.8" 
                     Canvas.Left="85" Canvas.Top="30"/>
            <Ellipse Width="10" Height="10" Fill="Black" Opacity="0.7" 
                     Canvas.Left="85" Canvas.Top="55"/>
            <Ellipse Width="10" Height="10" Fill="Black" Opacity="0.6" 
                     Canvas.Left="70" Canvas.Top="78"/>
            <Ellipse Width="10" Height="10" Fill="Black" Opacity="0.5" 
                     Canvas.Left="45" Canvas.Top="85"/>
            <Ellipse Width="10" Height="10" Fill="Black" Opacity="0.4" 
                     Canvas.Left="20" Canvas.Top="78"/>
            <Ellipse Width="10" Height="10" Fill="Black" Opacity="0.3" 
                     Canvas.Left="5" Canvas.Top="55"/>
            <Ellipse Width="10" Height="10" Fill="Black" Opacity="0.25" 
                     Canvas.Left="5" Canvas.Top="30"/>
            <Ellipse Width="10" Height="10" Fill="Black" Opacity="0.2" 
                     Canvas.Left="20" Canvas.Top="7"/>
        </Canvas>
    </Grid>
</UserControl>
```

## 핵심 개념 정리
- **Shape**: 기본 도형 클래스 (Line, Rectangle, Ellipse, Path 등)
- **Brush**: 채우기 방식 (SolidColorBrush, LinearGradientBrush, ImageBrush 등)
- **Transform**: 변환 효과 (Translate, Rotate, Scale, Skew)
- **Animation**: 시간에 따른 속성 변경
- **Storyboard**: 애니메이션 컨테이너
- **KeyFrame**: 특정 시점의 값 지정
- **Path Animation**: 경로를 따라 이동
- **3D Graphics**: Viewport3D를 통한 3D 렌더링
- **Effects**: 그래픽 효과 (DropShadow, Blur 등)
- **DrawingContext**: 직접 그리기를 위한 API