# 연산자 (Operators)

## 산술 연산자

### 기본 산술 연산
```csharp
int a = 10, b = 3;

int sum = a + b;        // 13 (덧셈)
int diff = a - b;       // 7  (뺄셈)
int product = a * b;    // 30 (곱셈)
int quotient = a / b;   // 3  (정수 나눗셈)
int remainder = a % b;  // 1  (나머지)

// 실수 나눗셈
double result = 10.0 / 3.0;  // 3.3333...
```

### 오버플로우와 언더플로우
```csharp
byte max = 255;
// byte overflow = max + 1;  // 0 (오버플로우)

// checked 블록으로 오버플로우 검사
checked
{
    byte safe = max + 1;  // OverflowException 발생
}

// unchecked 블록 (기본값)
unchecked
{
    byte overflow = max + 1;  // 0
}
```

## 증가/감소 연산자

### 전위와 후위 연산
```csharp
int x = 5;

// 전위 증가 (먼저 증가, 후 사용)
int a = ++x;  // x = 6, a = 6

// 후위 증가 (먼저 사용, 후 증가)
int b = x++;  // b = 6, x = 7

// 감소 연산자
int y = 10;
int c = --y;  // y = 9, c = 9
int d = y--;  // d = 9, y = 8
```

## 문자열 연결 연산자

```csharp
string first = "Hello";
string second = "World";

// + 연산자
string greeting = first + " " + second;  // "Hello World"

// 문자열과 다른 타입 연결
int age = 25;
string message = "Age: " + age;  // "Age: 25"

// StringBuilder (성능이 중요한 경우)
var sb = new StringBuilder();
sb.Append("Hello");
sb.Append(" ");
sb.Append("World");
string result = sb.ToString();
```

## 관계 연산자

```csharp
int x = 10, y = 20;

bool isEqual = x == y;      // false (같음)
bool isNotEqual = x != y;   // true  (다름)
bool isLess = x < y;        // true  (작음)
bool isGreater = x > y;     // false (큼)
bool isLessOrEqual = x <= y;    // true  (작거나 같음)
bool isGreaterOrEqual = x >= y; // false (크거나 같음)

// 문자열 비교
string str1 = "Hello";
string str2 = "Hello";
bool strEqual = str1 == str2;  // true
```

## 논리 연산자

### AND, OR, NOT
```csharp
bool a = true, b = false;

bool and = a && b;   // false (AND)
bool or = a || b;    // true  (OR)
bool not = !a;       // false (NOT)

// 단락 평가 (Short-circuit)
int x = 0;
bool result = (x != 0) && (10 / x > 1);  // false, 뒤는 평가 안 함
```

### 논리 연산자 활용
```csharp
int age = 25;
bool hasLicense = true;

bool canDrive = age >= 18 && hasLicense;  // true

// 복잡한 조건
int score = 85;
bool passed = score >= 60 && score <= 100;
bool failed = score < 60 || score > 100;
```

## 조건 연산자 (삼항 연산자)

```csharp
int age = 20;
string status = age >= 18 ? "성인" : "미성년자";  // "성인"

// 중첩 사용
int score = 85;
string grade = score >= 90 ? "A" :
               score >= 80 ? "B" :
               score >= 70 ? "C" :
               score >= 60 ? "D" : "F";  // "B"

// null 체크와 함께 사용
string name = null;
string displayName = name != null ? name : "Anonymous";
```

## null 조건부 연산자

### ?. 연산자
```csharp
string str = null;
int? length = str?.Length;  // null (NullReferenceException 방지)

// 메서드 호출
string upper = str?.ToUpper();  // null

// 체이닝
Person person = null;
string city = person?.Address?.City;  // null
```

### ?[] 연산자
```csharp
int[] numbers = null;
int? first = numbers?[0];  // null

// 다차원 배열
int[,] matrix = null;
int? element = matrix?[0, 0];  // null
```

## 비트 연산자

### 시프트 연산자
```csharp
int value = 8;  // 0000 1000

int leftShift = value << 2;   // 32 (0010 0000)
int rightShift = value >> 2;  // 2  (0000 0010)

// 곱셈/나눗셈 대체
int doubled = value << 1;     // 16 (×2)
int halved = value >> 1;      // 4  (÷2)
```

### 비트 논리 연산자
```csharp
int a = 12;  // 1100
int b = 10;  // 1010

int and = a & b;   // 8   (1000) - AND
int or = a | b;    // 14  (1110) - OR
int xor = a ^ b;   // 6   (0110) - XOR
int not = ~a;      // -13 (비트 반전)

// 플래그 연산
[Flags]
enum Permissions
{
    None = 0,
    Read = 1,    // 001
    Write = 2,   // 010
    Execute = 4  // 100
}

Permissions perm = Permissions.Read | Permissions.Write;  // 011
bool canRead = (perm & Permissions.Read) != 0;  // true
```

## 할당 연산자

### 기본 할당
```csharp
int x = 10;  // 기본 할당
```

### 복합 할당 연산자
```csharp
int a = 10;
a += 5;   // a = a + 5  → 15
a -= 3;   // a = a - 3  → 12
a *= 2;   // a = a * 2  → 24
a /= 4;   // a = a / 4  → 6
a %= 4;   // a = a % 4  → 2

// 비트 복합 할당
int b = 12;
b &= 10;  // b = b & 10
b |= 5;   // b = b | 5
b ^= 3;   // b = b ^ 3
b <<= 2;  // b = b << 2
b >>= 1;  // b = b >> 1
```

## null 병합 연산자

### ?? 연산자
```csharp
string name = null;
string displayName = name ?? "Unknown";  // "Unknown"

// 체이닝
string first = null;
string second = null;
string third = "Default";
string result = first ?? second ?? third;  // "Default"
```

### ??= 연산자 (C# 8.0+)
```csharp
string name = null;
name ??= "Default Name";  // name이 null이면 할당

List<int> numbers = null;
numbers ??= new List<int>();  // null이면 새 리스트 생성
```

## 연산자 우선순위

### 우선순위 순서 (높음 → 낮음)
1. 기본: `x.y`, `x?.y`, `x?[i]`, `x++`, `x--`, `new`, `typeof`
2. 단항: `+x`, `-x`, `!x`, `~x`, `++x`, `--x`, `(T)x`
3. 산술: `*`, `/`, `%`
4. 산술: `+`, `-`
5. 시프트: `<<`, `>>`
6. 관계: `<`, `>`, `<=`, `>=`, `is`, `as`
7. 동등: `==`, `!=`
8. 논리 AND: `&`
9. 논리 XOR: `^`
10. 논리 OR: `|`
11. 조건 AND: `&&`
12. 조건 OR: `||`
13. null 병합: `??`
14. 조건: `?:`
15. 할당: `=`, `+=`, `-=`, etc.

### 우선순위 예제
```csharp
int result = 2 + 3 * 4;     // 14 (곱셈 먼저)
bool check = 5 > 3 && 2 < 4;  // true (관계 연산 먼저)

// 괄호로 우선순위 변경
int result2 = (2 + 3) * 4;  // 20
```

## 패턴 매칭 연산자

### is 연산자
```csharp
object obj = "Hello";

if (obj is string str)
{
    Console.WriteLine(str.Length);  // 5
}

// 타입 패턴
if (obj is string)
{
    // obj가 string 타입
}

// null 체크
if (obj is not null)
{
    // obj가 null이 아님
}
```

### switch 식 (C# 8.0+)
```csharp
string GetDayType(DayOfWeek day) => day switch
{
    DayOfWeek.Saturday or DayOfWeek.Sunday => "Weekend",
    _ => "Weekday"
};
```

## 핵심 개념 정리
- **산술 연산자**: 기본 수학 연산 수행
- **논리 연산자**: 조건 평가 및 제어 흐름
- **비트 연산자**: 저수준 비트 조작
- **null 관련 연산자**: null 안전성 향상
- **연산자 우선순위**: 괄호로 명시적 제어 권장