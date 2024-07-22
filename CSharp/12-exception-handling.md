# 예외 처리 (Exception Handling)

## 예외란?

예외는 프로그램 실행 중 발생하는 예기치 않은 상황이나 오류를 나타냅니다.

### 예외가 발생하는 경우
```csharp
// 0으로 나누기
int result = 10 / 0;  // DivideByZeroException

// null 참조
string str = null;
int length = str.Length;  // NullReferenceException

// 배열 범위 초과
int[] array = new int[5];
int value = array[10];  // IndexOutOfRangeException

// 형변환 실패
object obj = "Hello";
int number = (int)obj;  // InvalidCastException
```

## try-catch로 예외 받기

### 기본 구조
```csharp
try
{
    // 예외가 발생할 수 있는 코드
    int[] numbers = { 1, 2, 3 };
    Console.WriteLine(numbers[5]);  // 예외 발생
}
catch (IndexOutOfRangeException ex)
{
    // 예외 처리
    Console.WriteLine($"배열 인덱스 오류: {ex.Message}");
}
```

### 여러 예외 처리
```csharp
try
{
    Console.Write("숫자를 입력하세요: ");
    string input = Console.ReadLine();
    int number = int.Parse(input);
    int result = 100 / number;
    Console.WriteLine($"결과: {result}");
}
catch (FormatException)
{
    Console.WriteLine("올바른 숫자 형식이 아닙니다.");
}
catch (DivideByZeroException)
{
    Console.WriteLine("0으로 나눌 수 없습니다.");
}
catch (Exception ex)  // 모든 예외 포착
{
    Console.WriteLine($"예기치 않은 오류: {ex.Message}");
}
```

### 예외 재발생
```csharp
public void ProcessData(string data)
{
    try
    {
        // 데이터 처리
        if (string.IsNullOrEmpty(data))
            throw new ArgumentException("데이터가 비어있습니다.");
    }
    catch (ArgumentException ex)
    {
        // 로깅
        Console.WriteLine($"로그: {ex.Message}");
        throw;  // 예외 재발생
    }
}
```

## System.Exception 클래스

### Exception 속성
```csharp
try
{
    throw new InvalidOperationException("잘못된 작업입니다.");
}
catch (Exception ex)
{
    Console.WriteLine($"Message: {ex.Message}");
    Console.WriteLine($"Type: {ex.GetType().Name}");
    Console.WriteLine($"Source: {ex.Source}");
    Console.WriteLine($"StackTrace:\n{ex.StackTrace}");
    
    if (ex.InnerException != null)
    {
        Console.WriteLine($"Inner Exception: {ex.InnerException.Message}");
    }
}
```

### 주요 예외 클래스 계층
```csharp
// System.Exception
//   ├─ SystemException
//   │   ├─ ArgumentException
//   │   │   ├─ ArgumentNullException
//   │   │   └─ ArgumentOutOfRangeException
//   │   ├─ ArithmeticException
//   │   │   ├─ DivideByZeroException
//   │   │   └─ OverflowException
//   │   ├─ IndexOutOfRangeException
//   │   ├─ InvalidCastException
//   │   ├─ InvalidOperationException
//   │   ├─ NullReferenceException
//   │   └─ OutOfMemoryException
//   └─ ApplicationException (사용 권장하지 않음)
```

## 예외 던지기

### throw 문
```csharp
public void SetAge(int age)
{
    if (age < 0)
    {
        throw new ArgumentException("나이는 음수일 수 없습니다.", nameof(age));
    }
    
    if (age > 150)
    {
        throw new ArgumentOutOfRangeException(nameof(age), age, "나이는 150을 초과할 수 없습니다.");
    }
    
    this.age = age;
}
```

### 조건부 예외 던지기
```csharp
public void Withdraw(decimal amount)
{
    if (amount <= 0)
        throw new ArgumentException("금액은 0보다 커야 합니다.");
    
    if (amount > balance)
        throw new InvalidOperationException($"잔액이 부족합니다. 현재 잔액: {balance}");
    
    balance -= amount;
}
```

### throw 식 (C# 7.0+)
```csharp
// null 병합 연산자와 함께
public Person(string name)
{
    Name = name ?? throw new ArgumentNullException(nameof(name));
}

// 삼항 연산자와 함께
public void SetValue(int value)
{
    this.value = value >= 0 ? value : throw new ArgumentException("값은 0 이상이어야 합니다.");
}
```

## try-catch-finally

### finally 블록
```csharp
FileStream file = null;
try
{
    file = new FileStream("data.txt", FileMode.Open);
    // 파일 작업
}
catch (FileNotFoundException)
{
    Console.WriteLine("파일을 찾을 수 없습니다.");
}
catch (IOException ex)
{
    Console.WriteLine($"I/O 오류: {ex.Message}");
}
finally
{
    // 항상 실행
    file?.Close();
    Console.WriteLine("파일 스트림을 닫았습니다.");
}
```

### using 문과 IDisposable
```csharp
// using 문 사용 (자동으로 Dispose 호출)
using (FileStream file = new FileStream("data.txt", FileMode.Open))
{
    // 파일 작업
}  // 여기서 자동으로 file.Dispose() 호출

// using 선언 (C# 8.0+)
using FileStream file = new FileStream("data.txt", FileMode.Open);
// 파일 작업
// 스코프 끝에서 자동으로 Dispose
```

## 사용자 정의 예외 클래스

### 기본 사용자 정의 예외
```csharp
public class BusinessException : Exception
{
    public string ErrorCode { get; }
    
    public BusinessException(string message, string errorCode) 
        : base(message)
    {
        ErrorCode = errorCode;
    }
    
    public BusinessException(string message, string errorCode, Exception innerException)
        : base(message, innerException)
    {
        ErrorCode = errorCode;
    }
}

// 사용
throw new BusinessException("주문 처리 실패", "ORD001");
```

### 계층적 예외 구조
```csharp
// 기본 도메인 예외
public abstract class DomainException : Exception
{
    public DomainException(string message) : base(message) { }
    public DomainException(string message, Exception innerException) 
        : base(message, innerException) { }
}

// 검증 예외
public class ValidationException : DomainException
{
    public Dictionary<string, string> Errors { get; }
    
    public ValidationException(string message, Dictionary<string, string> errors) 
        : base(message)
    {
        Errors = errors;
    }
}

// 비즈니스 규칙 예외
public class BusinessRuleException : DomainException
{
    public string RuleCode { get; }
    
    public BusinessRuleException(string message, string ruleCode) 
        : base(message)
    {
        RuleCode = ruleCode;
    }
}
```

## 예외 필터 (C# 6.0+)

### when 키워드
```csharp
try
{
    // 작업 수행
}
catch (HttpRequestException ex) when (ex.Message.Contains("404"))
{
    Console.WriteLine("페이지를 찾을 수 없습니다.");
}
catch (HttpRequestException ex) when (ex.Message.Contains("500"))
{
    Console.WriteLine("서버 오류가 발생했습니다.");
}
catch (HttpRequestException ex)
{
    Console.WriteLine($"HTTP 오류: {ex.Message}");
}
```

### 예외 필터 활용
```csharp
public async Task<string> DownloadDataAsync(string url, int maxRetries = 3)
{
    int retryCount = 0;
    
    while (true)
    {
        try
        {
            using var client = new HttpClient();
            return await client.GetStringAsync(url);
        }
        catch (HttpRequestException ex) when (retryCount < maxRetries)
        {
            retryCount++;
            Console.WriteLine($"재시도 {retryCount}/{maxRetries}");
            await Task.Delay(1000 * retryCount);
        }
    }
}
```

## 예외 처리 모범 사례

### 1. 구체적인 예외 처리
```csharp
// 나쁜 예
try
{
    // 모든 코드
}
catch (Exception ex)
{
    // 모든 예외를 같은 방식으로 처리
}

// 좋은 예
try
{
    var result = int.Parse(userInput);
    var calculation = 100 / result;
}
catch (FormatException)
{
    Console.WriteLine("올바른 숫자를 입력하세요.");
}
catch (DivideByZeroException)
{
    Console.WriteLine("0으로 나눌 수 없습니다.");
}
```

### 2. 예외 래핑
```csharp
public class DataService
{
    public Data LoadData(string id)
    {
        try
        {
            // 데이터베이스 접근
            return database.GetData(id);
        }
        catch (SqlException ex)
        {
            // 구현 세부사항을 숨기고 도메인 예외로 래핑
            throw new DataAccessException($"데이터 로드 실패: {id}", ex);
        }
    }
}
```

### 3. 리소스 정리
```csharp
public void ProcessFile(string fileName)
{
    FileStream stream = null;
    try
    {
        stream = File.OpenRead(fileName);
        // 처리
    }
    finally
    {
        stream?.Dispose();
    }
}

// 또는 using 사용
public void ProcessFileWithUsing(string fileName)
{
    using (var stream = File.OpenRead(fileName))
    {
        // 처리
    }
}
```

### 4. 예외 로깅
```csharp
public class Logger
{
    public static void LogException(Exception ex, string context = null)
    {
        var timestamp = DateTime.Now.ToString("yyyy-MM-dd HH:mm:ss");
        var logMessage = $"[{timestamp}] {context ?? "Error"}: {ex.GetType().Name}\n" +
                        $"Message: {ex.Message}\n" +
                        $"StackTrace: {ex.StackTrace}";
        
        if (ex.InnerException != null)
        {
            logMessage += $"\nInner Exception: {ex.InnerException.Message}";
        }
        
        File.AppendAllText("error.log", logMessage + "\n\n");
    }
}
```

## 비동기 예외 처리

```csharp
// async 메서드의 예외 처리
public async Task<string> GetDataAsync(string url)
{
    try
    {
        using var client = new HttpClient();
        return await client.GetStringAsync(url);
    }
    catch (HttpRequestException ex)
    {
        Console.WriteLine($"네트워크 오류: {ex.Message}");
        throw;
    }
    catch (TaskCanceledException)
    {
        Console.WriteLine("요청이 취소되었습니다.");
        throw;
    }
}

// 여러 태스크의 예외 처리
public async Task ProcessMultipleAsync()
{
    var tasks = new[]
    {
        Task.Run(() => DoWork1()),
        Task.Run(() => DoWork2()),
        Task.Run(() => DoWork3())
    };
    
    try
    {
        await Task.WhenAll(tasks);
    }
    catch (Exception ex)
    {
        // 첫 번째 예외만 캐치됨
        Console.WriteLine($"오류: {ex.Message}");
        
        // 모든 예외 확인
        foreach (var task in tasks.Where(t => t.IsFaulted))
        {
            foreach (var exception in task.Exception.InnerExceptions)
            {
                Console.WriteLine($"Task 예외: {exception.Message}");
            }
        }
    }
}
```

## 성능 고려사항

```csharp
// 예외는 비용이 높음 - 일반적인 흐름 제어에 사용하지 말 것
// 나쁜 예
public bool IsNumeric(string input)
{
    try
    {
        int.Parse(input);
        return true;
    }
    catch (FormatException)
    {
        return false;
    }
}

// 좋은 예
public bool IsNumericBetter(string input)
{
    return int.TryParse(input, out _);
}
```

## 핵심 개념 정리
- **예외**: 프로그램 실행 중 발생하는 오류
- **try-catch**: 예외를 포착하고 처리
- **finally**: 항상 실행되는 정리 코드
- **throw**: 예외를 명시적으로 발생
- **사용자 정의 예외**: 도메인 특화 예외 생성