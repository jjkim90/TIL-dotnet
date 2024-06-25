# 데이터 타입과 변수

## 변수(Variable)

### 변수 선언과 초기화
```csharp
// 선언
int number;

// 초기화
number = 10;

// 선언과 동시에 초기화
int age = 25;
string name = "John";
```

### 변수 명명 규칙
- 영문자, 숫자, 언더스코어(_) 사용 가능
- 숫자로 시작할 수 없음
- C# 키워드 사용 불가
- 대소문자 구분

```csharp
// 올바른 변수명
int userAge;
string firstName;
double _salary;

// 잘못된 변수명
// int 2ndPlace;  // 숫자로 시작
// string class;  // 키워드 사용
```

## 값 형식(Value Type)과 참조 형식(Reference Type)

### 값 형식
- 스택 메모리에 직접 저장
- 변수가 실제 데이터를 포함

```csharp
int a = 10;
int b = a;  // 값이 복사됨
b = 20;     // a는 여전히 10
```

### 참조 형식
- 힙 메모리에 저장
- 변수는 데이터의 주소를 가짐

```csharp
int[] arr1 = {1, 2, 3};
int[] arr2 = arr1;  // 주소가 복사됨
arr2[0] = 10;       // arr1[0]도 10으로 변경
```

## 기본 데이터 형식

### 정수형
```csharp
byte b = 255;        // 8비트, 0 ~ 255
short s = 32767;     // 16비트, -32,768 ~ 32,767
int i = 2147483647;  // 32비트, 약 -21억 ~ 21억
long l = 9223372036854775807L;  // 64비트

// 부호 없는 정수
uint ui = 4294967295U;
ulong ul = 18446744073709551615UL;
```

### 부동소수점형
```csharp
float f = 3.14159f;    // 32비트, 7자리 정밀도
double d = 3.141592653589793;  // 64비트, 15-16자리 정밀도
decimal m = 3.14159265358979323846m;  // 128비트, 28-29자리 정밀도

// 특수값
double infinity = double.PositiveInfinity;
double nan = double.NaN;
```

### 문자형과 문자열
```csharp
char ch = 'A';          // 16비트 유니코드 문자
char unicode = '\u0041'; // 'A'
string str = "Hello";    // 문자열

// 문자열 이스케이프 시퀀스
string path = "C:\\Users\\Name";
string quote = "He said \"Hello\"";
string newLine = "First line\nSecond line";
```

### 논리형
```csharp
bool isTrue = true;
bool isFalse = false;
bool result = 10 > 5;  // true
```

### object 형식
```csharp
object obj = 123;       // 정수 저장
obj = "Hello";         // 문자열 저장
obj = 3.14;           // 실수 저장

// 모든 형식의 최상위 부모
```

## 박싱과 언박싱

### 박싱(Boxing)
- 값 형식을 object로 변환
```csharp
int value = 123;
object obj = value;  // 박싱
```

### 언박싱(Unboxing)
- object를 값 형식으로 변환
```csharp
object obj = 123;
int value = (int)obj;  // 언박싱
```

## 형식 변환

### 암시적 변환
```csharp
int i = 100;
long l = i;     // int → long
float f = i;    // int → float
```

### 명시적 변환(캐스팅)
```csharp
double d = 3.14;
int i = (int)d;  // 3 (소수점 버림)

long l = 1234567890123;
int truncated = (int)l;  // 오버플로우 가능
```

### Convert 클래스
```csharp
string str = "123";
int i = Convert.ToInt32(str);
double d = Convert.ToDouble(str);
bool b = Convert.ToBoolean(1);  // true
```

### Parse 메서드
```csharp
string str = "123";
int i = int.Parse(str);
double d = double.Parse("3.14");

// TryParse - 안전한 변환
if (int.TryParse("123", out int result))
{
    Console.WriteLine(result);
}
```

## 상수와 열거형

### 상수(const)
```csharp
const double PI = 3.14159;
const int MAX_SIZE = 100;
// PI = 3.14;  // 오류! 상수는 변경 불가
```

### 열거형(enum)
```csharp
enum Days
{
    Sunday,    // 0
    Monday,    // 1
    Tuesday,   // 2
    Wednesday, // 3
    Thursday,  // 4
    Friday,    // 5
    Saturday   // 6
}

Days today = Days.Monday;
int dayNumber = (int)today;  // 1
```

### 플래그 열거형
```csharp
[Flags]
enum FilePermission
{
    None = 0,
    Read = 1,
    Write = 2,
    Execute = 4
}

FilePermission permission = FilePermission.Read | FilePermission.Write;
```

## Nullable 형식

### nullable 값 형식
```csharp
int? nullableInt = null;
double? nullableDouble = 3.14;

if (nullableInt.HasValue)
{
    int value = nullableInt.Value;
}

// null 병합 연산자
int result = nullableInt ?? 0;  // null이면 0
```

## var 키워드

### 암시적 형식 지역 변수
```csharp
var i = 10;        // int로 추론
var s = "Hello";   // string으로 추론
var list = new List<int>();  // List<int>로 추론

// var는 선언과 동시에 초기화 필요
// var x;  // 오류!
```

## 문자열 다루기

### 문자열 검색
```csharp
string text = "Hello, World!";
bool contains = text.Contains("World");  // true
int index = text.IndexOf("World");       // 7
bool startsWith = text.StartsWith("Hello");  // true
```

### 문자열 변형
```csharp
string text = "Hello, World!";
string upper = text.ToUpper();     // "HELLO, WORLD!"
string lower = text.ToLower();     // "hello, world!"
string trimmed = "  Hello  ".Trim();  // "Hello"
string replaced = text.Replace("World", "C#");  // "Hello, C#!"
```

### 문자열 분할과 결합
```csharp
string csv = "apple,banana,orange";
string[] fruits = csv.Split(',');  // ["apple", "banana", "orange"]

string joined = string.Join(", ", fruits);  // "apple, banana, orange"
```

### 문자열 보간
```csharp
string name = "John";
int age = 25;
string message = $"My name is {name} and I am {age} years old.";

// 형식 지정
double price = 19.99;
string formatted = $"Price: {price:C}";  // "Price: $19.99"
```

## 핵심 개념 정리
- **값 형식**: 스택에 저장, 값 복사
- **참조 형식**: 힙에 저장, 주소 복사
- **박싱/언박싱**: 값 형식과 object 간 변환
- **nullable**: null을 가질 수 있는 값 형식
- **var**: 컴파일러가 형식을 추론