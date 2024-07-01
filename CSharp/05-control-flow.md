# 제어문 (Control Flow)

## 분기문 (Branching Statements)

### if, else, else if
```csharp
int score = 85;

if (score >= 90)
{
    Console.WriteLine("A등급");
}
else if (score >= 80)
{
    Console.WriteLine("B등급");
}
else if (score >= 70)
{
    Console.WriteLine("C등급");
}
else
{
    Console.WriteLine("D등급");
}
```

### if 문 중첩
```csharp
bool isWeekend = true;
int hour = 10;

if (isWeekend)
{
    if (hour < 12)
    {
        Console.WriteLine("주말 오전");
    }
    else
    {
        Console.WriteLine("주말 오후");
    }
}
else
{
    Console.WriteLine("평일");
}
```

### 한 줄 if 문
```csharp
int age = 20;

// 중괄호 생략 (권장하지 않음)
if (age >= 18)
    Console.WriteLine("성인");

// 삼항 연산자 사용 (권장)
string status = age >= 18 ? "성인" : "미성년자";
```

### switch 문
```csharp
int day = 3;
string dayName;

switch (day)
{
    case 1:
        dayName = "월요일";
        break;
    case 2:
        dayName = "화요일";
        break;
    case 3:
        dayName = "수요일";
        break;
    case 4:
        dayName = "목요일";
        break;
    case 5:
        dayName = "금요일";
        break;
    case 6:
    case 7:
        dayName = "주말";
        break;
    default:
        dayName = "잘못된 날짜";
        break;
}
```

### switch 식 (C# 8.0+)
```csharp
string GetDayType(int day) => day switch
{
    1 or 2 or 3 or 4 or 5 => "평일",
    6 or 7 => "주말",
    _ => "잘못된 날짜"
};

// 튜플 패턴
string GetSeasonName(int month) => month switch
{
    3 or 4 or 5 => "봄",
    6 or 7 or 8 => "여름",
    9 or 10 or 11 => "가을",
    12 or 1 or 2 => "겨울",
    _ => "잘못된 월"
};

// 속성 패턴
string GetPersonType(Person person) => person switch
{
    { Age: < 20 } => "청소년",
    { Age: >= 20 and < 65 } => "성인",
    { Age: >= 65 } => "노인",
    _ => "알 수 없음"
};
```

## 반복문 (Loop Statements)

### while 문
```csharp
int count = 0;

while (count < 5)
{
    Console.WriteLine($"Count: {count}");
    count++;
}

// 조건이 거짓이면 실행 안 됨
bool condition = false;
while (condition)
{
    // 실행되지 않음
}
```

### do-while 문
```csharp
int number;

do
{
    Console.Write("1-10 사이의 숫자 입력: ");
    number = int.Parse(Console.ReadLine());
} while (number < 1 || number > 10);

// 최소 한 번은 실행
int x = 10;
do
{
    Console.WriteLine(x);  // 10 출력
} while (x < 5);
```

### for 문
```csharp
// 기본 형태
for (int i = 0; i < 10; i++)
{
    Console.WriteLine(i);
}

// 여러 변수 초기화
for (int i = 0, j = 10; i < j; i++, j--)
{
    Console.WriteLine($"i: {i}, j: {j}");
}

// 조건부 생략
int k = 0;
for (; k < 5; k++)  // 초기화 생략
{
    Console.WriteLine(k);
}
```

### 중첩 for 문
```csharp
// 구구단
for (int i = 2; i <= 9; i++)
{
    for (int j = 1; j <= 9; j++)
    {
        Console.WriteLine($"{i} × {j} = {i * j}");
    }
    Console.WriteLine();
}

// 2차원 배열 탐색
int[,] matrix = new int[3, 3];
for (int row = 0; row < 3; row++)
{
    for (int col = 0; col < 3; col++)
    {
        matrix[row, col] = row * 3 + col;
    }
}
```

### foreach 문
```csharp
int[] numbers = { 1, 2, 3, 4, 5 };

foreach (int num in numbers)
{
    Console.WriteLine(num);
}

// 문자열 순회
string text = "Hello";
foreach (char c in text)
{
    Console.WriteLine(c);
}

// Dictionary 순회
var dict = new Dictionary<string, int>
{
    ["apple"] = 100,
    ["banana"] = 200
};

foreach (var item in dict)
{
    Console.WriteLine($"{item.Key}: {item.Value}");
}
```

### 무한 반복
```csharp
// while을 이용한 무한 루프
while (true)
{
    // break로 탈출 필요
    if (someCondition)
        break;
}

// for를 이용한 무한 루프
for (;;)
{
    // 탈출 조건 필요
}
```

## 점프문 (Jump Statements)

### break
```csharp
// 반복문 탈출
for (int i = 0; i < 10; i++)
{
    if (i == 5)
        break;  // i가 5일 때 루프 종료
    Console.WriteLine(i);  // 0, 1, 2, 3, 4만 출력
}

// switch 문에서 사용
switch (value)
{
    case 1:
        DoSomething();
        break;  // 필수
}
```

### continue
```csharp
// 다음 반복으로 건너뛰기
for (int i = 0; i < 10; i++)
{
    if (i % 2 == 0)
        continue;  // 짝수 건너뛰기
    Console.WriteLine(i);  // 1, 3, 5, 7, 9만 출력
}

// while 문에서 사용
int j = 0;
while (j < 10)
{
    j++;
    if (j % 3 == 0)
        continue;
    Console.WriteLine(j);
}
```

### goto
```csharp
// 레이블로 점프 (권장하지 않음)
int x = 0;

start:
    Console.WriteLine(x);
    x++;
    if (x < 5)
        goto start;

// switch 문에서 fall-through
switch (value)
{
    case 1:
        Console.WriteLine("One");
        goto case 2;
    case 2:
        Console.WriteLine("Two");
        break;
}
```

### return
```csharp
// 메서드 종료
void ProcessNumber(int number)
{
    if (number < 0)
        return;  // 조기 종료
    
    Console.WriteLine(number);
}

// 값 반환
int GetAbsolute(int value)
{
    if (value >= 0)
        return value;
    else
        return -value;
}
```

## 패턴 매칭

### 타입 패턴
```csharp
object obj = "Hello";

if (obj is string str)
{
    Console.WriteLine($"문자열: {str}");
}
else if (obj is int number)
{
    Console.WriteLine($"정수: {number}");
}
```

### 상수 패턴
```csharp
int value = 42;

if (value is 42)
{
    Console.WriteLine("The answer!");
}

// switch 문에서
string result = value switch
{
    0 => "Zero",
    1 => "One",
    42 => "The answer",
    _ => "Other"
};
```

### 속성 패턴
```csharp
public class Person
{
    public string Name { get; set; }
    public int Age { get; set; }
}

Person person = new Person { Name = "John", Age = 25 };

if (person is { Age: > 18 })
{
    Console.WriteLine("성인입니다");
}

// 복잡한 패턴
string category = person switch
{
    { Age: < 13 } => "어린이",
    { Age: >= 13 and < 20 } => "청소년",
    { Age: >= 20 and < 65 } => "성인",
    _ => "노인"
};
```

### 관계 패턴 (C# 9.0+)
```csharp
int temperature = 25;

string feeling = temperature switch
{
    < 0 => "매우 춥다",
    >= 0 and < 10 => "춥다",
    >= 10 and < 20 => "서늘하다",
    >= 20 and < 30 => "따뜻하다",
    >= 30 => "덥다"
};
```

### 논리 패턴
```csharp
char c = 'A';

if (c is >= 'A' and <= 'Z')
{
    Console.WriteLine("대문자");
}
else if (c is >= 'a' and <= 'z')
{
    Console.WriteLine("소문자");
}

// or 패턴
if (c is 'a' or 'e' or 'i' or 'o' or 'u')
{
    Console.WriteLine("모음");
}
```

### 목록 패턴 (C# 11+)
```csharp
int[] numbers = { 1, 2, 3 };

if (numbers is [1, 2, 3])
{
    Console.WriteLine("1, 2, 3 배열");
}

// 와일드카드 사용
if (numbers is [1, _, 3])
{
    Console.WriteLine("첫 번째는 1, 세 번째는 3");
}

// 슬라이스 패턴
if (numbers is [1, .. var middle, 3])
{
    // middle는 중간 요소들
}
```

## 실전 예제

### 메뉴 시스템
```csharp
bool running = true;

while (running)
{
    Console.WriteLine("1. 시작");
    Console.WriteLine("2. 설정");
    Console.WriteLine("3. 종료");
    Console.Write("선택: ");
    
    string choice = Console.ReadLine();
    
    switch (choice)
    {
        case "1":
            Console.WriteLine("게임 시작!");
            break;
        case "2":
            Console.WriteLine("설정 열기");
            break;
        case "3":
            running = false;
            Console.WriteLine("종료합니다.");
            break;
        default:
            Console.WriteLine("잘못된 선택");
            break;
    }
}
```

### 검색 알고리즘
```csharp
int[] array = { 1, 5, 3, 9, 2, 8, 4 };
int target = 9;
bool found = false;

for (int i = 0; i < array.Length; i++)
{
    if (array[i] == target)
    {
        Console.WriteLine($"찾았습니다! 인덱스: {i}");
        found = true;
        break;
    }
}

if (!found)
{
    Console.WriteLine("찾을 수 없습니다.");
}
```

## 핵심 개념 정리
- **분기문**: 조건에 따라 실행 경로 결정
- **반복문**: 코드를 반복적으로 실행
- **점프문**: 실행 흐름을 강제로 변경
- **패턴 매칭**: 값과 타입을 효율적으로 검사
- **제어 흐름**: 프로그램의 논리적 흐름 구성