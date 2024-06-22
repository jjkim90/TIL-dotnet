# 첫 C# 프로그램

## Hello, World!

### 가장 간단한 C# 프로그램
```csharp
Console.WriteLine("Hello, World!");
```

### 전통적인 방식
```csharp
using System;

namespace HelloWorld
{
    class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine("Hello, World!");
        }
    }
}
```

## 프로그램 구조 분석

### using System;
- 네임스페이스를 사용하기 위한 선언
- System 네임스페이스에는 Console 클래스 포함
- C# 9.0 이후 최상위 문(Top-level statements)으로 생략 가능

### namespace
```csharp
namespace MyApp
{
    // 관련된 클래스들을 그룹화
    // 이름 충돌 방지
}
```

### class
```csharp
class Program
{
    // 프로그램의 기본 단위
    // 데이터와 메서드를 포함
}
```

### Main 메서드
```csharp
static void Main(string[] args)
{
    // 프로그램의 진입점
    // static: 인스턴스 생성 없이 실행
    // void: 반환값 없음
    // string[] args: 명령줄 인수
}
```

## 최신 C# 문법 (C# 9.0+)

### Top-level statements
```csharp
// Program.cs
Console.WriteLine("Hello, World!");

// 컴파일러가 자동으로 Main 메서드 생성
```

### Global using
```csharp
// GlobalUsings.cs
global using System;
global using System.Collections.Generic;

// 모든 파일에서 자동으로 사용 가능
```

### 명령줄 인수 처리
```csharp
// args는 자동으로 사용 가능
if (args.Length > 0)
{
    Console.WriteLine($"Hello, {args[0]}!");
}
else
{
    Console.WriteLine("Hello, World!");
}
```

## Console 클래스 활용

### 기본 출력
```csharp
Console.WriteLine("줄바꿈 있는 출력");
Console.Write("줄바꿈 없는 출력");
```

### 입력 받기
```csharp
Console.Write("이름을 입력하세요: ");
string name = Console.ReadLine();
Console.WriteLine($"안녕하세요, {name}님!");
```

### 색상 변경
```csharp
Console.ForegroundColor = ConsoleColor.Green;
Console.WriteLine("녹색 텍스트");
Console.ResetColor();
```

## 주석 작성법

```csharp
// 한 줄 주석

/*
 * 여러 줄
 * 주석
 */

/// <summary>
/// XML 문서 주석
/// </summary>
```

## 컴파일과 실행

### dotnet CLI 사용
```bash
# 프로젝트 생성
dotnet new console -n MyApp

# 실행
dotnet run

# 빌드
dotnet build

# 릴리스 빌드
dotnet build -c Release
```

### 실행 파일 생성
```bash
# 자체 포함 실행 파일
dotnet publish -c Release -r win-x64 --self-contained
```

## CLR (Common Language Runtime)

### 컴파일 과정
1. C# 소스 코드 작성
2. C# 컴파일러(csc)가 IL(Intermediate Language)로 변환
3. JIT(Just-In-Time) 컴파일러가 기계어로 변환
4. CPU에서 실행

### .NET 아키텍처
```
C# 코드 → IL 코드 → JIT 컴파일 → 네이티브 코드
         ↓
      Assembly(.dll/.exe)
         ↓
        CLR
```

## 핵심 개념 정리
- **Main 메서드**: 프로그램의 시작점
- **Console**: 콘솔 입출력을 담당하는 클래스
- **namespace**: 코드를 논리적으로 그룹화
- **CLR**: .NET 프로그램을 실행하는 런타임 환경