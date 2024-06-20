# 프로그래밍 기초

## 프로그래밍 언어의 역사

### 컴퓨터의 시작, 프로그래밍의 시작
- 초기 컴퓨터는 기계어(0과 1)로만 프로그래밍
- 어셈블리어의 등장으로 기계어보다 읽기 쉬운 명령어 사용 가능

### 고급 프로그래밍 언어의 발전
- **FORTRAN (1957)**: 과학 계산용 최초의 고급 언어
- **BASIC (1964)**: 초보자도 쉽게 배울 수 있는 언어
- **C (1972)**: UNIX 운영체제 개발을 위해 만들어진 시스템 프로그래밍 언어
- **C++ (1985)**: C에 객체지향 개념을 추가
- **C# (2000)**: Microsoft가 .NET Framework를 위해 개발한 현대적 언어

### C#의 탄생 배경
- Java의 장점 + C++의 강력함
- .NET Framework와 함께 성장
- 지속적인 언어 개선과 현대적 기능 추가

## C#의 특징

### 1. 객체지향 프로그래밍
```csharp
public class Person
{
    public string Name { get; set; }
    public int Age { get; set; }
}
```

### 2. 타입 안정성
- 컴파일 시점에 타입 검사
- 런타임 오류 최소화

### 3. 가비지 컬렉션
- 자동 메모리 관리
- 메모리 누수 방지

### 4. 크로스 플랫폼
- .NET Core/.NET 5+ 를 통한 다양한 플랫폼 지원
- Windows, Linux, macOS에서 실행 가능

## 개발 환경 설정

### Visual Studio 설치
1. Visual Studio Community 다운로드
2. .NET 데스크톱 개발 워크로드 선택
3. 설치 완료 후 새 프로젝트 생성

### 프로젝트 구조
```
MyProject/
├── MyProject.csproj    # 프로젝트 파일
├── Program.cs          # 진입점
├── bin/               # 빌드 출력
└── obj/               # 임시 파일
```

### 기본 프로젝트 파일 (.csproj)
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
  </PropertyGroup>
</Project>
```

## 핵심 개념 정리
- **프로그래밍 언어**: 사람이 컴퓨터에게 명령을 내리기 위한 도구
- **컴파일**: 소스 코드를 기계어로 변환하는 과정
- **IDE**: 통합 개발 환경 (Integrated Development Environment)
- **.NET**: Microsoft의 개발 플랫폼