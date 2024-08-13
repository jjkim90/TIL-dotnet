# 스레드와 태스크

## 프로세스와 스레드

### 기본 개념
- **프로세스**: 실행 중인 프로그램의 인스턴스
- **스레드**: 프로세스 내에서 실행되는 작업 단위

### 스레드 시작하기
```csharp
// Thread 클래스 사용
Thread thread1 = new Thread(DoWork);
thread1.Start();

Thread thread2 = new Thread(() => 
{
    Console.WriteLine("람다식으로 스레드 실행");
});
thread2.Start();

// 매개변수 전달
Thread thread3 = new Thread(DoWorkWithParam);
thread3.Start("Hello Thread");

void DoWork()
{
    for (int i = 0; i < 5; i++)
    {
        Console.WriteLine($"Working... {i}");
        Thread.Sleep(1000);
    }
}

void DoWorkWithParam(object data)
{
    string message = (string)data;
    Console.WriteLine($"Received: {message}");
}
```

### 스레드 종료와 Join
```csharp
Thread worker = new Thread(() =>
{
    for (int i = 0; i < 10; i++)
    {
        Console.WriteLine($"Working {i}");
        Thread.Sleep(500);
    }
});

worker.Start();

// 스레드가 끝날 때까지 대기
worker.Join();
Console.WriteLine("스레드 작업 완료");

// 타임아웃 설정
if (worker.Join(3000))  // 3초 대기
{
    Console.WriteLine("정상 종료");
}
else
{
    Console.WriteLine("타임아웃");
}
```

### 스레드 상태
```csharp
Thread thread = new Thread(() =>
{
    Thread.Sleep(2000);
});

Console.WriteLine($"상태: {thread.ThreadState}");  // Unstarted

thread.Start();
Console.WriteLine($"상태: {thread.ThreadState}");  // Running

Thread.Sleep(100);
Console.WriteLine($"상태: {thread.ThreadState}");  // WaitSleepJoin

thread.Join();
Console.WriteLine($"상태: {thread.ThreadState}");  // Stopped
```

### 스레드 인터럽트
```csharp
Thread sleepingThread = new Thread(() =>
{
    try
    {
        Console.WriteLine("긴 작업 시작...");
        Thread.Sleep(10000);  // 10초 대기
        Console.WriteLine("작업 완료");
    }
    catch (ThreadInterruptedException)
    {
        Console.WriteLine("스레드가 인터럽트됨!");
    }
});

sleepingThread.Start();
Thread.Sleep(2000);  // 2초 후
sleepingThread.Interrupt();  // 인터럽트 발생
```

## 스레드 동기화

### lock 키워드
```csharp
public class Counter
{
    private int count = 0;
    private readonly object lockObj = new object();
    
    public void Increment()
    {
        lock (lockObj)
        {
            count++;
        }
    }
    
    public int GetCount()
    {
        lock (lockObj)
        {
            return count;
        }
    }
}

// 사용 예제
Counter counter = new Counter();
Thread[] threads = new Thread[10];

for (int i = 0; i < threads.Length; i++)
{
    threads[i] = new Thread(() =>
    {
        for (int j = 0; j < 1000; j++)
        {
            counter.Increment();
        }
    });
    threads[i].Start();
}

foreach (var thread in threads)
{
    thread.Join();
}

Console.WriteLine($"최종 카운트: {counter.GetCount()}");  // 10000
```

### Monitor 클래스
```csharp
public class SharedResource
{
    private readonly object syncRoot = new object();
    private int resource = 0;
    
    public void UseResource()
    {
        Monitor.Enter(syncRoot);
        try
        {
            // 임계 영역
            resource++;
            Console.WriteLine($"Resource: {resource}");
        }
        finally
        {
            Monitor.Exit(syncRoot);
        }
    }
    
    public bool TryUseResource(int timeoutMs)
    {
        if (Monitor.TryEnter(syncRoot, timeoutMs))
        {
            try
            {
                resource++;
                return true;
            }
            finally
            {
                Monitor.Exit(syncRoot);
            }
        }
        return false;
    }
}
```

### Mutex
```csharp
// 시스템 전역 뮤텍스
public class SingleInstance
{
    private static Mutex mutex;
    
    public static bool IsSingleInstance()
    {
        bool createdNew;
        mutex = new Mutex(true, "MyAppMutex", out createdNew);
        
        if (!createdNew)
        {
            Console.WriteLine("애플리케이션이 이미 실행 중입니다.");
            return false;
        }
        
        return true;
    }
}
```

### Semaphore
```csharp
public class ResourcePool
{
    private readonly Semaphore semaphore;
    
    public ResourcePool(int maxCount)
    {
        semaphore = new Semaphore(maxCount, maxCount);
    }
    
    public void UseResource(int threadId)
    {
        Console.WriteLine($"스레드 {threadId}: 리소스 대기");
        semaphore.WaitOne();
        
        try
        {
            Console.WriteLine($"스레드 {threadId}: 리소스 사용 중");
            Thread.Sleep(2000);  // 작업 시뮬레이션
        }
        finally
        {
            Console.WriteLine($"스레드 {threadId}: 리소스 반환");
            semaphore.Release();
        }
    }
}

// 사용
ResourcePool pool = new ResourcePool(3);  // 최대 3개 동시 접근

for (int i = 0; i < 10; i++)
{
    int threadId = i;
    new Thread(() => pool.UseResource(threadId)).Start();
}
```

## Task와 Task<TResult>

### Task 기본 사용
```csharp
// Task 생성과 실행
Task task1 = new Task(() => Console.WriteLine("Task 1"));
task1.Start();

// Task.Run 사용 (권장)
Task task2 = Task.Run(() => 
{
    Console.WriteLine("Task 2 시작");
    Thread.Sleep(1000);
    Console.WriteLine("Task 2 완료");
});

// 여러 태스크 대기
Task[] tasks = new Task[5];
for (int i = 0; i < tasks.Length; i++)
{
    int taskNum = i;
    tasks[i] = Task.Run(() =>
    {
        Thread.Sleep(1000);
        Console.WriteLine($"Task {taskNum} 완료");
    });
}

Task.WaitAll(tasks);  // 모든 태스크 대기
Console.WriteLine("모든 태스크 완료");
```

### Task<TResult>
```csharp
// 결과를 반환하는 Task
Task<int> calculateTask = Task.Run(() =>
{
    int sum = 0;
    for (int i = 1; i <= 100; i++)
    {
        sum += i;
    }
    return sum;
});

// 결과 얻기
int result = calculateTask.Result;  // 블로킹
Console.WriteLine($"합계: {result}");

// 여러 Task의 결과
Task<int>[] tasks = new Task<int>[3];
tasks[0] = Task.Run(() => 10);
tasks[1] = Task.Run(() => 20);
tasks[2] = Task.Run(() => 30);

Task.WaitAll(tasks);
int sum = tasks.Sum(t => t.Result);
Console.WriteLine($"총합: {sum}");
```

### Task 연속 작업
```csharp
Task<string> downloadTask = Task.Run(() =>
{
    // 다운로드 시뮬레이션
    Thread.Sleep(2000);
    return "Downloaded data";
});

Task processTask = downloadTask.ContinueWith(antecedent =>
{
    string data = antecedent.Result;
    Console.WriteLine($"처리 중: {data}");
    // 데이터 처리
});

Task finalTask = processTask.ContinueWith(antecedent =>
{
    Console.WriteLine("모든 작업 완료");
});

finalTask.Wait();
```

## Parallel 클래스

### Parallel.For
```csharp
// 순차 실행
var sw1 = Stopwatch.StartNew();
for (int i = 0; i < 100; i++)
{
    Thread.Sleep(10);  // 작업 시뮬레이션
}
sw1.Stop();

// 병렬 실행
var sw2 = Stopwatch.StartNew();
Parallel.For(0, 100, i =>
{
    Thread.Sleep(10);  // 작업 시뮬레이션
});
sw2.Stop();

Console.WriteLine($"순차: {sw1.ElapsedMilliseconds}ms");
Console.WriteLine($"병렬: {sw2.ElapsedMilliseconds}ms");
```

### Parallel.ForEach
```csharp
List<string> urls = new List<string>
{
    "http://example1.com",
    "http://example2.com",
    "http://example3.com"
};

Parallel.ForEach(urls, url =>
{
    Console.WriteLine($"처리 중: {url} (스레드: {Thread.CurrentThread.ManagedThreadId})");
    // URL 처리 로직
    Thread.Sleep(1000);
});
```

### Parallel.Invoke
```csharp
Parallel.Invoke(
    () => DoWork("작업 1"),
    () => DoWork("작업 2"),
    () => DoWork("작업 3"),
    () => DoWork("작업 4")
);

void DoWork(string name)
{
    Console.WriteLine($"{name} 시작 (스레드: {Thread.CurrentThread.ManagedThreadId})");
    Thread.Sleep(1000);
    Console.WriteLine($"{name} 완료");
}
```

## async와 await

### 비동기 메서드 기본
```csharp
public async Task<string> DownloadDataAsync(string url)
{
    using (HttpClient client = new HttpClient())
    {
        Console.WriteLine("다운로드 시작");
        string result = await client.GetStringAsync(url);
        Console.WriteLine("다운로드 완료");
        return result;
    }
}

// 사용
public async Task UseAsync()
{
    string data = await DownloadDataAsync("http://example.com");
    Console.WriteLine($"데이터 길이: {data.Length}");
}
```

### 여러 비동기 작업
```csharp
public async Task MultipleAsyncOperations()
{
    Task<string> task1 = DownloadDataAsync("http://example1.com");
    Task<string> task2 = DownloadDataAsync("http://example2.com");
    Task<string> task3 = DownloadDataAsync("http://example3.com");
    
    // 모두 완료 대기
    string[] results = await Task.WhenAll(task1, task2, task3);
    
    foreach (string result in results)
    {
        Console.WriteLine($"결과 길이: {result.Length}");
    }
}

// 첫 번째 완료 대기
public async Task<string> GetFirstResponseAsync()
{
    Task<string> task1 = GetDataFromServer1Async();
    Task<string> task2 = GetDataFromServer2Async();
    
    Task<string> completedTask = await Task.WhenAny(task1, task2);
    return await completedTask;
}
```

### ConfigureAwait
```csharp
public async Task<string> GetDataAsync()
{
    using (HttpClient client = new HttpClient())
    {
        // UI 컨텍스트로 돌아갈 필요 없음
        string data = await client.GetStringAsync("http://example.com")
                                 .ConfigureAwait(false);
        
        // CPU 집약적 작업
        return ProcessData(data);
    }
}
```

## 비동기 패턴

### Task 기반 비동기 패턴 (TAP)
```csharp
public class FileService
{
    public async Task<string> ReadFileAsync(string path)
    {
        using (StreamReader reader = new StreamReader(path))
        {
            return await reader.ReadToEndAsync();
        }
    }
    
    public async Task WriteFileAsync(string path, string content)
    {
        using (StreamWriter writer = new StreamWriter(path))
        {
            await writer.WriteAsync(content);
        }
    }
    
    public async Task CopyFileAsync(string source, string destination)
    {
        string content = await ReadFileAsync(source);
        await WriteFileAsync(destination, content);
    }
}
```

### 취소 토큰
```csharp
public async Task LongRunningOperationAsync(CancellationToken cancellationToken)
{
    for (int i = 0; i < 100; i++)
    {
        cancellationToken.ThrowIfCancellationRequested();
        
        await Task.Delay(100, cancellationToken);
        Console.WriteLine($"진행률: {i}%");
    }
}

// 사용
CancellationTokenSource cts = new CancellationTokenSource();

Task operation = LongRunningOperationAsync(cts.Token);

// 5초 후 취소
await Task.Delay(5000);
cts.Cancel();

try
{
    await operation;
}
catch (OperationCanceledException)
{
    Console.WriteLine("작업이 취소되었습니다.");
}
```

### Progress 보고
```csharp
public async Task DownloadWithProgressAsync(
    string url, 
    IProgress<int> progress,
    CancellationToken cancellationToken)
{
    using (HttpClient client = new HttpClient())
    {
        var response = await client.GetAsync(url, HttpCompletionOption.ResponseHeadersRead);
        var totalBytes = response.Content.Headers.ContentLength ?? 0;
        
        using (var stream = await response.Content.ReadAsStreamAsync())
        {
            var buffer = new byte[8192];
            var totalRead = 0L;
            
            while (true)
            {
                var read = await stream.ReadAsync(buffer, 0, buffer.Length, cancellationToken);
                if (read == 0) break;
                
                totalRead += read;
                var percentage = (int)((totalRead * 100) / totalBytes);
                progress?.Report(percentage);
            }
        }
    }
}

// 사용
var progress = new Progress<int>(percent =>
{
    Console.WriteLine($"진행률: {percent}%");
});

await DownloadWithProgressAsync("http://example.com/largefile", progress, CancellationToken.None);
```

## 동기화 프리미티브

### SemaphoreSlim (비동기 세마포어)
```csharp
public class AsyncResourcePool
{
    private readonly SemaphoreSlim semaphore;
    
    public AsyncResourcePool(int maxCount)
    {
        semaphore = new SemaphoreSlim(maxCount, maxCount);
    }
    
    public async Task<T> UseResourceAsync<T>(Func<Task<T>> action)
    {
        await semaphore.WaitAsync();
        try
        {
            return await action();
        }
        finally
        {
            semaphore.Release();
        }
    }
}
```

### AsyncLock 패턴
```csharp
public class AsyncLock
{
    private readonly SemaphoreSlim semaphore = new SemaphoreSlim(1, 1);
    
    public async Task<IDisposable> LockAsync()
    {
        await semaphore.WaitAsync();
        return new Releaser(semaphore);
    }
    
    private class Releaser : IDisposable
    {
        private readonly SemaphoreSlim semaphore;
        
        public Releaser(SemaphoreSlim semaphore)
        {
            this.semaphore = semaphore;
        }
        
        public void Dispose()
        {
            semaphore.Release();
        }
    }
}

// 사용
private readonly AsyncLock asyncLock = new AsyncLock();

public async Task CriticalSectionAsync()
{
    using (await asyncLock.LockAsync())
    {
        // 임계 영역
        await Task.Delay(100);
    }
}
```

## 핵심 개념 정리
- **Thread**: 저수준 스레드 제어
- **Task**: 고수준 비동기 작업 추상화
- **async/await**: 비동기 코드를 동기 코드처럼 작성
- **Parallel**: 데이터 병렬 처리
- **동기화**: lock, Monitor, Semaphore 등으로 스레드 안전성 확보