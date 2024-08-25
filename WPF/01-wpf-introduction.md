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