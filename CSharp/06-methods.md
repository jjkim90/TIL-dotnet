# 메서드 (Methods)

## 메서드란?

메서드는 특정 작업을 수행하는 코드 블록입니다. 코드의 재사용성을 높이고 프로그램을 구조화합니다.

### 메서드 선언
```csharp
// 기본 구조
[접근제한자] [static] 반환타입 메서드명(매개변수목록)
{
    // 메서드 본문
    return 반환값;
}

// 예제
public static int Add(int a, int b)
{
    return a + b;
}
```

### 메서드 호출
```csharp
int result = Add(5, 3);  // 8
Console.WriteLine(result);

// void 메서드
PrintMessage("Hello");

void PrintMessage(string message)
{
    Console.WriteLine(message);
}
```

## return 문

### 값 반환
```csharp
int GetSquare(int number)
{
    return number * number;
}

// 조건부 반환
int GetAbsolute(int value)
{
    if (value >= 0)
        return value;
    return -value;
}
```

### 조기 종료
```csharp
void ProcessData(string data)
{
    if (string.IsNullOrEmpty(data))
        return;  // 조기 종료
    
    // 데이터 처리
    Console.WriteLine(data.ToUpper());
}
```

### 여러 값 반환 (튜플)
```csharp
(int min, int max) GetMinMax(int[] numbers)
{
    int min = numbers[0];
    int max = numbers[0];
    
    foreach (int num in numbers)
    {
        if (num < min) min = num;
        if (num > max) max = num;
    }
    
    return (min, max);
}

// 사용
var result = GetMinMax(new[] { 3, 1, 4, 1, 5 });
Console.WriteLine($"Min: {result.min}, Max: {result.max}");
```

## 매개변수

### 값 타입 매개변수
```csharp
void ModifyValue(int x)
{
    x = 100;  // 원본에 영향 없음
}

int number = 10;
ModifyValue(number);
Console.WriteLine(number);  // 10 (변경되지 않음)
```

### 참조 타입 매개변수
```csharp
void ModifyArray(int[] arr)
{
    arr[0] = 100;  // 원본 배열 변경
}

int[] numbers = { 1, 2, 3 };
ModifyArray(numbers);
Console.WriteLine(numbers[0]);  // 100
```

## 참조에 의한 매개변수 전달

### ref 키워드
```csharp
void Swap(ref int a, ref int b)
{
    int temp = a;
    a = b;
    b = temp;
}

int x = 10, y = 20;
Swap(ref x, ref y);
Console.WriteLine($"x: {x}, y: {y}");  // x: 20, y: 10
```

### in 키워드 (읽기 전용 참조)
```csharp
// 큰 구조체를 효율적으로 전달
struct LargeStruct
{
    public double X, Y, Z;
    // ... 많은 필드
}

void ProcessStruct(in LargeStruct data)
{
    // data는 읽기 전용
    Console.WriteLine(data.X);
    // data.X = 10;  // 오류!
}
```

## 메서드의 결과를 참조로 반환

### ref return
```csharp
class DataContainer
{
    private int[] numbers = { 1, 2, 3, 4, 5 };
    
    public ref int GetElement(int index)
    {
        return ref numbers[index];
    }
}

var container = new DataContainer();
ref int element = ref container.GetElement(2);
element = 100;  // 원본 배열 변경
```

## 출력 전용 매개변수 (out)

### out 매개변수
```csharp
bool TryDivide(int dividend, int divisor, out int result)
{
    if (divisor == 0)
    {
        result = 0;
        return false;
    }
    
    result = dividend / divisor;
    return true;
}

// 사용
if (TryDivide(10, 2, out int quotient))
{
    Console.WriteLine($"결과: {quotient}");
}
```

### 여러 out 매개변수
```csharp
void GetMinMaxAvg(int[] numbers, out int min, out int max, out double avg)
{
    min = max = numbers[0];
    int sum = 0;
    
    foreach (int num in numbers)
    {
        if (num < min) min = num;
        if (num > max) max = num;
        sum += num;
    }
    
    avg = (double)sum / numbers.Length;
}
```

### out 변수 선언
```csharp
// C# 7.0+ inline 선언
if (int.TryParse("123", out int number))
{
    Console.WriteLine(number);
}

// 무시 패턴
if (int.TryParse("123", out _))
{
    Console.WriteLine("유효한 숫자");
}
```

## 메서드 오버로딩

### 매개변수 개수가 다른 경우
```csharp
int Add(int a, int b)
{
    return a + b;
}

int Add(int a, int b, int c)
{
    return a + b + c;
}

// 사용
int sum1 = Add(1, 2);       // 3
int sum2 = Add(1, 2, 3);    // 6
```

### 매개변수 타입이 다른 경우
```csharp
void Print(int value)
{
    Console.WriteLine($"정수: {value}");
}

void Print(string value)
{
    Console.WriteLine($"문자열: {value}");
}

void Print(double value)
{
    Console.WriteLine($"실수: {value}");
}
```

## 가변 개수의 인수 (params)

```csharp
int Sum(params int[] numbers)
{
    int total = 0;
    foreach (int num in numbers)
    {
        total += num;
    }
    return total;
}

// 사용
int result1 = Sum(1, 2, 3);        // 6
int result2 = Sum(1, 2, 3, 4, 5);  // 15
int result3 = Sum();               // 0

// 배열로도 전달 가능
int[] array = { 1, 2, 3 };
int result4 = Sum(array);
```

### params와 일반 매개변수
```csharp
void PrintValues(string prefix, params object[] values)
{
    Console.Write(prefix + ": ");
    foreach (object value in values)
    {
        Console.Write(value + " ");
    }
    Console.WriteLine();
}

PrintValues("Numbers", 1, 2, 3);
PrintValues("Mixed", 1, "Hello", 3.14);
```

## 명명된 인수

```csharp
void CreateUser(string name, int age, string city = "Unknown")
{
    Console.WriteLine($"{name}, {age}, {city}");
}

// 위치 기반 호출
CreateUser("John", 25, "Seoul");

// 명명된 인수 사용
CreateUser(name: "John", age: 25, city: "Seoul");
CreateUser(age: 25, name: "John", city: "Seoul");  // 순서 변경 가능

// 일부만 명명된 인수
CreateUser("John", city: "Seoul", age: 25);
```

## 선택적 인수 (기본값)

```csharp
void SendEmail(string to, string subject, string body = "", bool isHtml = false)
{
    Console.WriteLine($"To: {to}");
    Console.WriteLine($"Subject: {subject}");
    Console.WriteLine($"Body: {body}");
    Console.WriteLine($"HTML: {isHtml}");
}

// 사용
SendEmail("user@example.com", "Hello");  // body="", isHtml=false
SendEmail("user@example.com", "Hello", "Content");  // isHtml=false
SendEmail("user@example.com", "Hello", isHtml: true);  // body=""
```

### 선택적 인수 규칙
```csharp
// 올바른 예: 선택적 매개변수는 뒤에
void Method1(int required, string optional = "default") { }

// 잘못된 예: 선택적 매개변수 뒤에 필수 매개변수
// void Method2(string optional = "default", int required) { }  // 오류!
```

## 로컬 함수

```csharp
int Calculate(int x, int y, string operation)
{
    // 로컬 함수 정의
    int Add(int a, int b) => a + b;
    int Subtract(int a, int b) => a - b;
    int Multiply(int a, int b) => a * b;
    
    // 사용
    return operation switch
    {
        "add" => Add(x, y),
        "subtract" => Subtract(x, y),
        "multiply" => Multiply(x, y),
        _ => throw new ArgumentException("Invalid operation")
    };
}
```

### 로컬 함수와 클로저
```csharp
void ProcessItems(List<string> items)
{
    int processedCount = 0;
    
    void ProcessItem(string item)
    {
        Console.WriteLine($"Processing: {item}");
        processedCount++;  // 외부 변수 접근
    }
    
    foreach (string item in items)
    {
        ProcessItem(item);
    }
    
    Console.WriteLine($"Total processed: {processedCount}");
}
```

### static 로컬 함수
```csharp
double CalculateArea(double radius)
{
    // static 로컬 함수는 외부 변수 캡처 불가
    static double GetPi() => 3.14159;
    
    return GetPi() * radius * radius;
}
```

## 메서드 체이닝

```csharp
public class StringBuilder
{
    private string value = "";
    
    public StringBuilder Append(string text)
    {
        value += text;
        return this;  // 자기 자신 반환
    }
    
    public StringBuilder AppendLine(string text)
    {
        value += text + "\n";
        return this;
    }
    
    public override string ToString() => value;
}

// 사용
var sb = new StringBuilder()
    .Append("Hello")
    .Append(" ")
    .AppendLine("World")
    .Append("!");
```

## 재귀 메서드

```csharp
// 팩토리얼
int Factorial(int n)
{
    if (n <= 1)
        return 1;
    return n * Factorial(n - 1);
}

// 피보나치
int Fibonacci(int n)
{
    if (n <= 1)
        return n;
    return Fibonacci(n - 1) + Fibonacci(n - 2);
}

// 꼬리 재귀 최적화
int FactorialTailRecursive(int n, int accumulator = 1)
{
    if (n <= 1)
        return accumulator;
    return FactorialTailRecursive(n - 1, n * accumulator);
}
```

## 확장 메서드 미리보기

```csharp
public static class StringExtensions
{
    public static bool IsNullOrEmpty(this string str)
    {
        return string.IsNullOrEmpty(str);
    }
    
    public static string Reverse(this string str)
    {
        char[] chars = str.ToCharArray();
        Array.Reverse(chars);
        return new string(chars);
    }
}

// 사용
string text = "Hello";
string reversed = text.Reverse();  // "olleH"
```

## 핵심 개념 정리
- **메서드**: 재사용 가능한 코드 블록
- **매개변수**: 메서드에 전달되는 값
- **ref/out**: 참조에 의한 전달
- **오버로딩**: 같은 이름, 다른 시그니처
- **로컬 함수**: 메서드 내부의 함수