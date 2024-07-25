# 대리자와 이벤트 (Delegates and Events)

## 대리자(Delegate)란?

대리자는 메서드에 대한 참조를 가지는 타입입니다. 메서드를 변수처럼 다룰 수 있게 해줍니다.

### 대리자 선언과 사용
```csharp
// 대리자 선언
public delegate int Calculate(int a, int b);

// 대리자와 시그니처가 일치하는 메서드들
public static int Add(int x, int y) => x + y;
public static int Multiply(int x, int y) => x * y;

// 사용
Calculate calc = Add;
int result1 = calc(5, 3);  // 8

calc = Multiply;
int result2 = calc(5, 3);  // 15
```

### 인스턴스 메서드와 정적 메서드
```csharp
public delegate void MyDelegate(string message);

public class Printer
{
    public void PrintToConsole(string msg)
    {
        Console.WriteLine($"Console: {msg}");
    }
    
    public static void PrintToFile(string msg)
    {
        File.WriteAllText("log.txt", msg);
    }
}

// 사용
Printer printer = new Printer();
MyDelegate del1 = printer.PrintToConsole;  // 인스턴스 메서드
MyDelegate del2 = Printer.PrintToFile;     // 정적 메서드

del1("Hello");  // Console: Hello
del2("World");  // 파일에 기록
```

## 대리자는 왜, 언제 사용하나요?

### 1. 콜백 메서드
```csharp
public class Timer
{
    public delegate void TimerCallback(DateTime time);
    
    public void Start(int intervalMs, TimerCallback callback)
    {
        while (true)
        {
            Thread.Sleep(intervalMs);
            callback(DateTime.Now);
        }
    }
}

// 사용
Timer timer = new Timer();
timer.Start(1000, time => Console.WriteLine($"현재 시간: {time}"));
```

### 2. 메서드를 매개변수로 전달
```csharp
public class Calculator
{
    public static double[] ProcessArray(double[] array, Func<double, double> operation)
    {
        double[] result = new double[array.Length];
        for (int i = 0; i < array.Length; i++)
        {
            result[i] = operation(array[i]);
        }
        return result;
    }
}

// 사용
double[] numbers = { 1, 2, 3, 4, 5 };
double[] squared = Calculator.ProcessArray(numbers, x => x * x);
double[] roots = Calculator.ProcessArray(numbers, Math.Sqrt);
```

### 3. 전략 패턴 구현
```csharp
public interface ISortStrategy
{
    void Sort(int[] array);
}

public class DataProcessor
{
    private Action<int[]> sortStrategy;
    
    public void SetSortStrategy(Action<int[]> strategy)
    {
        sortStrategy = strategy;
    }
    
    public void ProcessData(int[] data)
    {
        // 전처리
        sortStrategy?.Invoke(data);
        // 후처리
    }
}
```

## 일반화 대리자

### 제네릭 대리자
```csharp
public delegate T Operation<T>(T a, T b);

// 사용
Operation<int> intOp = (x, y) => x + y;
Operation<double> doubleOp = (x, y) => x * y;
Operation<string> stringOp = (x, y) => x + y;

int intResult = intOp(10, 20);        // 30
double doubleResult = doubleOp(3.5, 2.0);  // 7.0
string stringResult = stringOp("Hello", " World");  // "Hello World"
```

### 내장 제네릭 대리자
```csharp
// Action<T> - 반환값이 없는 대리자
Action<string> print = Console.WriteLine;
Action<int, int> printSum = (a, b) => Console.WriteLine(a + b);

// Func<T, TResult> - 반환값이 있는 대리자
Func<int, int, int> add = (a, b) => a + b;
Func<string, int> getLength = s => s.Length;

// Predicate<T> - bool을 반환하는 대리자
Predicate<int> isEven = x => x % 2 == 0;
```

## 대리자 체인

### 멀티캐스트 대리자
```csharp
public delegate void Notify(string message);

public class NotificationService
{
    public static void EmailNotification(string msg)
    {
        Console.WriteLine($"Email: {msg}");
    }
    
    public static void SmsNotification(string msg)
    {
        Console.WriteLine($"SMS: {msg}");
    }
    
    public static void PushNotification(string msg)
    {
        Console.WriteLine($"Push: {msg}");
    }
}

// 대리자 체인
Notify notify = NotificationService.EmailNotification;
notify += NotificationService.SmsNotification;
notify += NotificationService.PushNotification;

notify("새로운 메시지가 도착했습니다.");
// 출력:
// Email: 새로운 메시지가 도착했습니다.
// SMS: 새로운 메시지가 도착했습니다.
// Push: 새로운 메시지가 도착했습니다.
```

### 대리자 체인 관리
```csharp
// 대리자 제거
notify -= NotificationService.SmsNotification;

// null 체크
notify?.Invoke("메시지");

// 개별 호출
if (notify != null)
{
    foreach (Notify d in notify.GetInvocationList())
    {
        try
        {
            d("개별 호출");
        }
        catch (Exception ex)
        {
            Console.WriteLine($"오류: {ex.Message}");
        }
    }
}
```

## 익명 메서드

### 무명 대리자
```csharp
// C# 2.0 방식
Func<int, int, int> add = delegate(int a, int b)
{
    return a + b;
};

// 대리자 매개변수로 직접 전달
button.Click += delegate(object sender, EventArgs e)
{
    MessageBox.Show("버튼이 클릭되었습니다.");
};

// 클로저 (외부 변수 캡처)
int factor = 10;
Func<int, int> multiply = delegate(int x)
{
    return x * factor;  // 외부 변수 사용
};
```

## 이벤트 (Event)

### 이벤트 선언과 발생
```csharp
public class Publisher
{
    // 이벤트 선언
    public event EventHandler<string> SomethingHappened;
    
    public void DoSomething()
    {
        Console.WriteLine("작업 수행 중...");
        
        // 이벤트 발생
        OnSomethingHappened("작업이 완료되었습니다.");
    }
    
    protected virtual void OnSomethingHappened(string message)
    {
        SomethingHappened?.Invoke(this, message);
    }
}

// 구독자
public class Subscriber
{
    public void Subscribe(Publisher publisher)
    {
        publisher.SomethingHappened += OnSomethingHappened;
    }
    
    private void OnSomethingHappened(object sender, string message)
    {
        Console.WriteLine($"이벤트 수신: {message}");
    }
}
```

### EventHandler 대리자
```csharp
public class Button
{
    // 표준 EventHandler 사용
    public event EventHandler Clicked;
    public event EventHandler<MouseEventArgs> MouseMoved;
    
    public void Click()
    {
        Clicked?.Invoke(this, EventArgs.Empty);
    }
    
    public void OnMouseMove(int x, int y)
    {
        MouseMoved?.Invoke(this, new MouseEventArgs(x, y));
    }
}

public class MouseEventArgs : EventArgs
{
    public int X { get; }
    public int Y { get; }
    
    public MouseEventArgs(int x, int y)
    {
        X = x;
        Y = y;
    }
}
```

## 대리자와 이벤트의 차이

### 이벤트의 제약사항
```csharp
public class EventExample
{
    public delegate void MyDelegate();
    public event MyDelegate MyEvent;
    
    public void TestAccess()
    {
        // 클래스 내부에서는 가능
        MyEvent = Method1;
        MyEvent += Method2;
        MyEvent?.Invoke();
    }
    
    private void Method1() { }
    private void Method2() { }
}

public class ExternalClass
{
    public void TestExternalAccess(EventExample example)
    {
        // 외부에서는 += 또는 -= 만 가능
        example.MyEvent += Handler;
        // example.MyEvent = Handler;  // 컴파일 오류!
        // example.MyEvent.Invoke();   // 컴파일 오류!
    }
    
    private void Handler() { }
}
```

## 실전 예제

### 1. 프로퍼티 변경 알림
```csharp
public class ObservableProperty<T> : INotifyPropertyChanged
{
    private T value;
    
    public T Value
    {
        get => value;
        set
        {
            if (!EqualityComparer<T>.Default.Equals(this.value, value))
            {
                this.value = value;
                OnPropertyChanged();
            }
        }
    }
    
    public event PropertyChangedEventHandler PropertyChanged;
    
    protected virtual void OnPropertyChanged([CallerMemberName] string propertyName = null)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }
}
```

### 2. 비동기 작업 진행률
```csharp
public class FileDownloader
{
    public event EventHandler<ProgressEventArgs> ProgressChanged;
    public event EventHandler<DownloadCompletedEventArgs> DownloadCompleted;
    
    public async Task DownloadAsync(string url, string filePath)
    {
        using var client = new HttpClient();
        using var response = await client.GetAsync(url, HttpCompletionOption.ResponseHeadersRead);
        
        var totalBytes = response.Content.Headers.ContentLength ?? 0;
        var buffer = new byte[8192];
        var totalRead = 0L;
        
        using var stream = await response.Content.ReadAsStreamAsync();
        using var fileStream = File.Create(filePath);
        
        int read;
        while ((read = await stream.ReadAsync(buffer, 0, buffer.Length)) > 0)
        {
            await fileStream.WriteAsync(buffer, 0, read);
            totalRead += read;
            
            var progress = (int)((totalRead * 100) / totalBytes);
            OnProgressChanged(new ProgressEventArgs(progress, totalRead, totalBytes));
        }
        
        OnDownloadCompleted(new DownloadCompletedEventArgs(filePath, totalRead));
    }
    
    protected virtual void OnProgressChanged(ProgressEventArgs e)
    {
        ProgressChanged?.Invoke(this, e);
    }
    
    protected virtual void OnDownloadCompleted(DownloadCompletedEventArgs e)
    {
        DownloadCompleted?.Invoke(this, e);
    }
}

public class ProgressEventArgs : EventArgs
{
    public int Percentage { get; }
    public long BytesReceived { get; }
    public long TotalBytes { get; }
    
    public ProgressEventArgs(int percentage, long bytesReceived, long totalBytes)
    {
        Percentage = percentage;
        BytesReceived = bytesReceived;
        TotalBytes = totalBytes;
    }
}
```

### 3. 이벤트 집계기 패턴
```csharp
public interface IEventAggregator
{
    void Subscribe<TEvent>(Action<TEvent> handler);
    void Unsubscribe<TEvent>(Action<TEvent> handler);
    void Publish<TEvent>(TEvent eventToPublish);
}

public class EventAggregator : IEventAggregator
{
    private readonly Dictionary<Type, List<Delegate>> handlers = new();
    
    public void Subscribe<TEvent>(Action<TEvent> handler)
    {
        var eventType = typeof(TEvent);
        if (!handlers.ContainsKey(eventType))
        {
            handlers[eventType] = new List<Delegate>();
        }
        handlers[eventType].Add(handler);
    }
    
    public void Unsubscribe<TEvent>(Action<TEvent> handler)
    {
        var eventType = typeof(TEvent);
        if (handlers.ContainsKey(eventType))
        {
            handlers[eventType].Remove(handler);
        }
    }
    
    public void Publish<TEvent>(TEvent eventToPublish)
    {
        var eventType = typeof(TEvent);
        if (handlers.ContainsKey(eventType))
        {
            foreach (Action<TEvent> handler in handlers[eventType])
            {
                handler(eventToPublish);
            }
        }
    }
}

// 이벤트 정의
public class UserLoggedInEvent
{
    public string Username { get; set; }
    public DateTime LoginTime { get; set; }
}

// 사용
var eventAggregator = new EventAggregator();

// 구독
eventAggregator.Subscribe<UserLoggedInEvent>(e =>
{
    Console.WriteLine($"{e.Username}이(가) {e.LoginTime}에 로그인했습니다.");
});

// 발행
eventAggregator.Publish(new UserLoggedInEvent 
{ 
    Username = "user123", 
    LoginTime = DateTime.Now 
});
```

## 약한 이벤트 패턴

```csharp
public class WeakEventManager
{
    private readonly List<WeakReference> handlers = new();
    
    public void AddHandler(EventHandler handler)
    {
        handlers.Add(new WeakReference(handler));
    }
    
    public void RemoveHandler(EventHandler handler)
    {
        handlers.RemoveAll(wr => !wr.IsAlive || wr.Target.Equals(handler));
    }
    
    public void RaiseEvent(object sender, EventArgs e)
    {
        handlers.RemoveAll(wr => !wr.IsAlive);
        
        foreach (var weakRef in handlers)
        {
            if (weakRef.Target is EventHandler handler)
            {
                handler(sender, e);
            }
        }
    }
}
```

## 핵심 개념 정리
- **대리자**: 메서드에 대한 타입 안전한 참조
- **이벤트**: 캡슐화된 대리자, 발행-구독 패턴
- **멀티캐스트**: 여러 메서드를 연결하여 호출
- **익명 메서드**: 이름 없는 인라인 메서드
- **이벤트 vs 대리자**: 이벤트는 외부에서 직접 호출 불가