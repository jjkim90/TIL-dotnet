# 람다식 (Lambda Expressions)

## 람다식의 역사

### 대리자에서 람다식까지
```csharp
// 1. 명명된 메서드 (C# 1.0)
public static bool IsEven(int n) => n % 2 == 0;
Predicate<int> predicate1 = IsEven;

// 2. 익명 메서드 (C# 2.0)
Predicate<int> predicate2 = delegate(int n) { return n % 2 == 0; };

// 3. 람다식 (C# 3.0)
Predicate<int> predicate3 = n => n % 2 == 0;
```

## 람다식 기본

### 람다식 구조
```csharp
// (매개변수) => 식
// (매개변수) => { 문장; }

// 매개변수가 하나일 때 괄호 생략 가능
Func<int, int> square = x => x * x;

// 매개변수가 여러 개일 때
Func<int, int, int> add = (x, y) => x + y;

// 매개변수가 없을 때
Func<string> getMessage = () => "Hello, Lambda!";

// 타입 명시
Func<double, double> sqrt = (double x) => Math.Sqrt(x);
```

### 람다식과 대리자
```csharp
// Func 대리자
Func<int, bool> isPositive = x => x > 0;
Func<string, string, string> concat = (s1, s2) => s1 + s2;
Func<int, int, int, int> sum3 = (a, b, c) => a + b + c;

// Action 대리자
Action<string> print = message => Console.WriteLine(message);
Action<int, int> printSum = (a, b) => Console.WriteLine($"Sum: {a + b}");

// Predicate 대리자
Predicate<string> isEmpty = string.IsNullOrEmpty;
Predicate<int> isEven = n => n % 2 == 0;
```

## 문 형식의 람다식

### 여러 문장을 포함하는 람다식
```csharp
// 문 형식 람다
Action<string> processString = text =>
{
    if (string.IsNullOrWhiteSpace(text))
    {
        Console.WriteLine("텍스트가 비어있습니다.");
        return;
    }
    
    string processed = text.Trim().ToUpper();
    Console.WriteLine($"처리된 텍스트: {processed}");
    Console.WriteLine($"길이: {processed.Length}");
};

// 반환값이 있는 문 형식
Func<int, int, int> complexCalculation = (x, y) =>
{
    int temp = x * y;
    if (temp > 100)
        return temp / 2;
    else
        return temp * 2;
};
```

### 로컬 변수와 람다식
```csharp
Func<int, int> createMultiplier(int factor)
{
    // 람다식에서 외부 변수 캡처 (클로저)
    return x => x * factor;
}

var times3 = createMultiplier(3);
var times5 = createMultiplier(5);

Console.WriteLine(times3(10));  // 30
Console.WriteLine(times5(10));  // 50
```

## Func와 Action

### Func<> 대리자
```csharp
// Func<TResult> - 매개변수 없음
Func<DateTime> getCurrentTime = () => DateTime.Now;

// Func<T, TResult> - 매개변수 1개
Func<double, double> square = x => x * x;

// Func<T1, T2, TResult> - 매개변수 2개
Func<int, int, int> max = (a, b) => a > b ? a : b;

// 최대 16개 매개변수까지 지원
Func<int, int, int, int, int, int, int, int, int, int, int, int, int, int, int, int, int> func16;
```

### Action<> 대리자
```csharp
// Action - 매개변수 없음
Action greet = () => Console.WriteLine("Hello!");

// Action<T> - 매개변수 1개
Action<string> log = message => Console.WriteLine($"[LOG] {message}");

// Action<T1, T2> - 매개변수 2개
Action<string, int> printInfo = (name, age) => 
    Console.WriteLine($"Name: {name}, Age: {age}");

// 최대 16개 매개변수까지 지원
```

### 실전 활용
```csharp
public class Calculator
{
    public T Calculate<T>(T a, T b, Func<T, T, T> operation)
    {
        return operation(a, b);
    }
}

var calc = new Calculator();
int sum = calc.Calculate(5, 3, (x, y) => x + y);        // 8
int product = calc.Calculate(5, 3, (x, y) => x * y);    // 15
double power = calc.Calculate(2.0, 3.0, Math.Pow);      // 8.0
```

## 식 트리 (Expression Trees)

### Expression<TDelegate>
```csharp
// 람다식을 데이터로 표현
Expression<Func<int, bool>> isEvenExpr = x => x % 2 == 0;

// 식 트리 분석
var parameter = isEvenExpr.Parameters[0];
var body = isEvenExpr.Body as BinaryExpression;
var left = body.Left;
var right = body.Right;

Console.WriteLine($"매개변수: {parameter.Name}");
Console.WriteLine($"연산자: {body.NodeType}");
```

### 식 트리 생성
```csharp
// x => x * 2 + 1 식 트리 생성
ParameterExpression x = Expression.Parameter(typeof(int), "x");
BinaryExpression multiply = Expression.Multiply(x, Expression.Constant(2));
BinaryExpression add = Expression.Add(multiply, Expression.Constant(1));

Expression<Func<int, int>> lambda = 
    Expression.Lambda<Func<int, int>>(add, x);

// 컴파일하여 실행
Func<int, int> compiled = lambda.Compile();
int result = compiled(5);  // 11
```

### 동적 쿼리 생성
```csharp
public static Expression<Func<T, bool>> BuildPredicate<T>(
    string propertyName, object value)
{
    var parameter = Expression.Parameter(typeof(T), "x");
    var property = Expression.Property(parameter, propertyName);
    var constant = Expression.Constant(value);
    var equality = Expression.Equal(property, constant);
    
    return Expression.Lambda<Func<T, bool>>(equality, parameter);
}

// 사용
var predicate = BuildPredicate<Person>("Name", "John");
var filtered = people.Where(predicate.Compile());
```

## 식으로 이루어지는 멤버

### 식 본문 멤버 (Expression-bodied members)
```csharp
public class Circle
{
    private double radius;
    
    // 식 본문 프로퍼티
    public double Radius
    {
        get => radius;
        set => radius = value > 0 ? value : throw new ArgumentException();
    }
    
    // 식 본문 읽기 전용 프로퍼티
    public double Area => Math.PI * radius * radius;
    public double Circumference => 2 * Math.PI * radius;
    
    // 식 본문 메서드
    public double GetDiameter() => radius * 2;
    
    // 식 본문 생성자
    public Circle(double radius) => this.radius = radius;
    
    // 식 본문 종료자
    ~Circle() => Console.WriteLine("Circle destroyed");
    
    // 식 본문 인덱서
    public double this[string property] => property switch
    {
        "radius" => radius,
        "area" => Area,
        "circumference" => Circumference,
        _ => throw new ArgumentException()
    };
}
```

## 람다식 고급 활용

### 고차 함수
```csharp
// 함수를 반환하는 함수
public static Func<int, int> CreateAdder(int value)
{
    return x => x + value;
}

// 함수를 매개변수로 받는 함수
public static T[] Map<T>(T[] array, Func<T, T> transform)
{
    T[] result = new T[array.Length];
    for (int i = 0; i < array.Length; i++)
    {
        result[i] = transform(array[i]);
    }
    return result;
}

// 사용
var add5 = CreateAdder(5);
int[] numbers = { 1, 2, 3, 4, 5 };
int[] increased = Map(numbers, add5);  // { 6, 7, 8, 9, 10 }
```

### 함수 합성
```csharp
public static class FunctionExtensions
{
    public static Func<T, TResult2> Compose<T, TResult1, TResult2>(
        this Func<T, TResult1> f1,
        Func<TResult1, TResult2> f2)
    {
        return x => f2(f1(x));
    }
}

// 사용
Func<int, int> addOne = x => x + 1;
Func<int, int> multiplyByTwo = x => x * 2;
Func<int, int> addOneThenMultiply = addOne.Compose(multiplyByTwo);

int result = addOneThenMultiply(5);  // (5 + 1) * 2 = 12
```

### 커링 (Currying)
```csharp
// 일반 함수
Func<int, int, int> add = (x, y) => x + y;

// 커링된 함수
Func<int, Func<int, int>> curriedAdd = x => y => x + y;

// 사용
var add5 = curriedAdd(5);
int result1 = add5(3);  // 8
int result2 = add5(7);  // 12

// 여러 매개변수 커링
Func<int, Func<int, Func<int, int>>> curriedSum3 = 
    x => y => z => x + y + z;
var sum = curriedSum3(1)(2)(3);  // 6
```

### 메모이제이션 (Memoization)
```csharp
public static Func<T, TResult> Memoize<T, TResult>(Func<T, TResult> func)
{
    var cache = new Dictionary<T, TResult>();
    
    return arg =>
    {
        if (cache.TryGetValue(arg, out TResult result))
            return result;
        
        result = func(arg);
        cache[arg] = result;
        return result;
    };
}

// 사용 - 피보나치
Func<int, long> fibonacci = null;
fibonacci = Memoize<int, long>(n =>
    n <= 1 ? n : fibonacci(n - 1) + fibonacci(n - 2));

long fib40 = fibonacci(40);  // 빠른 계산
```

## 람다식과 LINQ

```csharp
var numbers = Enumerable.Range(1, 100);

// 람다식을 활용한 LINQ 쿼리
var result = numbers
    .Where(x => x % 2 == 0)           // 짝수만
    .Select(x => x * x)               // 제곱
    .Where(x => x > 100)              // 100보다 큰 것
    .OrderByDescending(x => x)        // 내림차순 정렬
    .Take(5)                          // 상위 5개
    .ToList();

// 복잡한 객체 처리
var people = GetPeople();
var adults = people
    .Where(p => p.Age >= 18)
    .OrderBy(p => p.LastName)
    .ThenBy(p => p.FirstName)
    .Select(p => new
    {
        FullName = $"{p.FirstName} {p.LastName}",
        p.Age,
        IsEligible = p.Age >= 21
    });
```

## 비동기 람다식

```csharp
// 비동기 람다식
Func<string, Task<string>> downloadAsync = async url =>
{
    using var client = new HttpClient();
    return await client.GetStringAsync(url);
};

// 비동기 Action
Action<string> processAsync = async message =>
{
    await Task.Delay(1000);
    Console.WriteLine(message);
};

// 이벤트 핸들러에서 사용
button.Click += async (sender, e) =>
{
    button.Enabled = false;
    await DoSomethingAsync();
    button.Enabled = true;
};
```

## 튜플과 람다식

```csharp
// 튜플을 반환하는 람다식
Func<int, (int square, int cube)> calculate = x => (x * x, x * x * x);

var result = calculate(3);
Console.WriteLine($"Square: {result.square}, Cube: {result.cube}");

// 튜플을 매개변수로 받는 람다식
Func<(int x, int y), int> addTuple = tuple => tuple.x + tuple.y;
int sum = addTuple((5, 3));  // 8

// 분해와 함께 사용
var points = new[] { (1, 2), (3, 4), (5, 6) };
var distances = points.Select(((int x, int y) p) => 
    Math.Sqrt(p.x * p.x + p.y * p.y));
```

## 핵심 개념 정리
- **람다식**: 간결한 익명 함수 표현
- **=> 연산자**: "goes to" 또는 "람다"
- **클로저**: 외부 변수 캡처
- **식 트리**: 람다식을 데이터로 표현
- **식 본문 멤버**: 간결한 멤버 정의