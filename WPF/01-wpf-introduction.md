# WPF 소개와 첫 애플리케이션

## WPF란 무엇인가?

WPF(Windows Presentation Foundation)는 Microsoft가 개발한 데스크톱 애플리케이션 개발 프레임워크입니다.

### WPF의 특징
- **벡터 기반 렌더링**: 해상도 독립적인 UI
- **XAML**: 선언적 UI 정의
- **데이터 바인딩**: 강력한 데이터 연결 메커니즘
- **스타일과 템플릿**: 일관된 디자인 적용
- **애니메이션**: 부드러운 UI 전환 효과
- **3D 그래픽**: Direct3D 통합

### WPF 아키텍처
```
프레젠테이션 프레임워크 (PresentationFramework.dll)
    ↓
프레젠테이션 코어 (PresentationCore.dll)
    ↓
Milcore (비관리 코드)
    ↓
DirectX
```

## WinForms vs WPF

### WinForms
- 이벤트 기반 프로그래밍
- 픽셀 기반 렌더링
- 디자이너 중심 개발
- 가벼운 애플리케이션에 적합

### WPF
- 데이터 바인딩 중심
- 벡터 기반 렌더링
- XAML과 코드 분리
- 복잡한 UI와 애니메이션에 적합

## 첫 WPF 애플리케이션 만들기

### 프로젝트 생성
1. Visual Studio 실행
2. "새 프로젝트 만들기" 선택
3. "WPF 애플리케이션" 템플릿 선택
4. 프로젝트 이름과 위치 지정

### 기본 프로젝트 구조
```
MyWpfApp/
├── App.xaml           # 애플리케이션 정의
├── App.xaml.cs        # 애플리케이션 코드 비하인드
├── MainWindow.xaml    # 메인 윈도우 UI
└── MainWindow.xaml.cs # 메인 윈도우 코드 비하인드
```

### App.xaml
```xml
<Application x:Class="MyWpfApp.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             StartupUri="MainWindow.xaml">
    <Application.Resources>
        <!-- 전역 리소스 정의 -->
    </Application.Resources>
</Application>
```

### MainWindow.xaml
```xml
<Window x:Class="MyWpfApp.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="My First WPF App" Height="350" Width="500">
    <Grid>
        <TextBlock Text="Hello, WPF!" 
                   HorizontalAlignment="Center" 
                   VerticalAlignment="Center"
                   FontSize="24"/>
    </Grid>
</Window>
```

### MainWindow.xaml.cs
```csharp
using System.Windows;

namespace MyWpfApp
{
    public partial class MainWindow : Window
    {
        public MainWindow()
        {
            InitializeComponent();
        }
    }
}
```

## 간단한 대화형 예제

버튼 클릭 이벤트를 추가한 예제를 만들어보겠습니다.

### MainWindow.xaml 수정
```xml
<Window x:Class="MyWpfApp.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="My First WPF App" Height="350" Width="500">
    <Grid>
        <Grid.RowDefinitions>
            <RowDefinition Height="*"/>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
        </Grid.RowDefinitions>
        
        <TextBlock x:Name="messageText" 
                   Grid.Row="0"
                   Text="Hello, WPF!" 
                   HorizontalAlignment="Center" 
                   VerticalAlignment="Center"
                   FontSize="24"/>
        
        <Button x:Name="clickButton" 
                Grid.Row="1"
                Content="Click Me!" 
                Width="120" 
                Height="40"
                FontSize="16"
                Click="ClickButton_Click"/>
        
        <TextBox x:Name="inputText" 
                 Grid.Row="2"
                 Width="200" 
                 Height="30"
                 VerticalAlignment="Top"
                 Margin="0,20,0,0"
                 FontSize="14"/>
    </Grid>
</Window>
```

### MainWindow.xaml.cs 수정
```csharp
using System.Windows;

namespace MyWpfApp
{
    public partial class MainWindow : Window
    {
        private int clickCount = 0;
        
        public MainWindow()
        {
            InitializeComponent();
        }
        
        private void ClickButton_Click(object sender, RoutedEventArgs e)
        {
            clickCount++;
            string userName = string.IsNullOrEmpty(inputText.Text) ? "User" : inputText.Text;
            messageText.Text = $"Hello, {userName}! Button clicked {clickCount} times.";
        }
    }
}
```

## Visual Studio에서 WPF 개발

### 유용한 도구 창
1. **XAML 디자이너**: 시각적 UI 디자인
2. **속성 창**: 컨트롤 속성 편집
3. **도구 상자**: 컨트롤 드래그 앤 드롭
4. **문서 개요**: UI 계층 구조 확인

### 디버깅 팁
- **Live Visual Tree**: 실행 중 UI 구조 확인
- **Live Property Explorer**: 실시간 속성 변경
- **XAML Hot Reload**: 실행 중 XAML 수정 반영

## WPF의 장단점

### 장점
- 풍부한 UI 표현력
- 데이터 바인딩의 강력함
- 스타일과 템플릿으로 일관된 디자인
- 애니메이션과 멀티미디어 지원
- 해상도 독립적인 UI

### 단점
- 학습 곡선이 가파름
- 초기 로딩 시간이 김
- 메모리 사용량이 많음
- Windows 플랫폼 한정

## 핵심 개념 정리
- **WPF**: Windows Presentation Foundation
- **XAML**: UI를 정의하는 마크업 언어
- **코드 비하인드**: XAML과 연결된 C# 코드
- **벡터 그래픽**: 해상도 독립적인 렌더링
- **데이터 바인딩**: UI와 데이터의 자동 동기화