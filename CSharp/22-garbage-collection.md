# 가비지 컬렉션

## CLR과 가비지 컬렉션

### 가비지 컬렉션의 필요성
- **수동 메모리 관리의 문제점**
  - 메모리 누수 (Memory Leak)
  - 댕글링 포인터 (Dangling Pointer)
  - 이중 해제 (Double Free)

- **자동 메모리 관리의 이점**
  - 개발자 생산성 향상
  - 메모리 관련 버그 감소
  - 메모리 최적화

### CLR의 메모리 관리
```csharp
// C++과 비교
// C++
void CppExample()
{
    int* array = new int[100];
    // ... 사용
    delete[] array;  // 수동 해제 필요
}

// C#
void CSharpExample()
{
    int[] array = new int[100];
    // ... 사용
    // 가비지 컬렉터가 자동으로 정리
}
```

## 가비지 컬렉션의 작동 원리

### 세대별 가비지 컬렉션
```csharp
public class GenerationExample
{
    public static void ShowGenerations()
    {
        // 새 객체 생성
        object obj = new object();
        Console.WriteLine($"Generation of new object: {GC.GetGeneration(obj)}");  // 0
        
        // GC 실행
        GC.Collect(0);  // Gen 0 수집
        Console.WriteLine($"After Gen0 GC: {GC.GetGeneration(obj)}");  // 1
        
        GC.Collect(1);  // Gen 1 수집
        Console.WriteLine($"After Gen1 GC: {GC.GetGeneration(obj)}");  // 2
        
        // 최대 세대 확인
        Console.WriteLine($"Max Generation: {GC.MaxGeneration}");  // 2
    }
}
```

### 객체 할당과 해제
```csharp
public class AllocationExample
{
    private byte[] data;
    
    public AllocationExample(int size)
    {
        data = new byte[size];
        Console.WriteLine($"Allocated {size:N0} bytes");
    }
    
    ~AllocationExample()  // 소멸자 (Finalizer)
    {
        Console.WriteLine("Finalizer called");
    }
    
    public static void TestAllocation()
    {
        // 힙 메모리 상태
        long before = GC.GetTotalMemory(false);
        
        // 대량 할당
        var objects = new List<AllocationExample>();
        for (int i = 0; i < 1000; i++)
        {
            objects.Add(new AllocationExample(1024));  // 1KB씩
        }
        
        long after = GC.GetTotalMemory(false);
        Console.WriteLine($"Memory used: {(after - before):N0} bytes");
        
        // 참조 제거
        objects.Clear();
        objects = null;
        
        // 강제 GC
        GC.Collect();
        GC.WaitForPendingFinalizers();
        GC.Collect();
        
        long cleaned = GC.GetTotalMemory(false);
        Console.WriteLine($"Memory after GC: {(cleaned - before):N0} bytes");
    }
}
```

### 가비지 컬렉션 알고리즘
```csharp
public class GCAlgorithmDemo
{
    class Node
    {
        public string Name { get; set; }
        public Node Next { get; set; }
        public byte[] Data { get; set; }
        
        public Node(string name, int dataSize)
        {
            Name = name;
            Data = new byte[dataSize];
        }
    }
    
    public static void MarkAndSweepDemo()
    {
        // 루트에서 참조되는 객체들
        Node root = new Node("Root", 100);
        root.Next = new Node("Child1", 200);
        root.Next.Next = new Node("Child2", 300);
        
        // 참조되지 않는 객체
        Node orphan = new Node("Orphan", 400);
        orphan = null;  // 참조 제거 - GC 대상
        
        // GC 실행
        Console.WriteLine($"Before GC: {GC.GetTotalMemory(false):N0} bytes");
        GC.Collect();
        Console.WriteLine($"After GC: {GC.GetTotalMemory(false):N0} bytes");
    }
}
```

## Dispose 패턴

### IDisposable 인터페이스
```csharp
public class ResourceHolder : IDisposable
{
    private bool disposed = false;
    private IntPtr unmanagedResource;  // 비관리 리소스
    private FileStream managedResource; // 관리 리소스
    
    public ResourceHolder(string fileName)
    {
        // 리소스 할당
        unmanagedResource = Marshal.AllocHGlobal(1024);
        managedResource = new FileStream(fileName, FileMode.Create);
    }
    
    // IDisposable 구현
    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);  // 소멸자 호출 방지
    }
    
    protected virtual void Dispose(bool disposing)
    {
        if (!disposed)
        {
            if (disposing)
            {
                // 관리 리소스 해제
                managedResource?.Dispose();
            }
            
            // 비관리 리소스 해제
            if (unmanagedResource != IntPtr.Zero)
            {
                Marshal.FreeHGlobal(unmanagedResource);
                unmanagedResource = IntPtr.Zero;
            }
            
            disposed = true;
        }
    }
    
    // 소멸자
    ~ResourceHolder()
    {
        Dispose(false);
    }
}
```

### using 문과 자동 해제
```csharp
public class UsingPatternExample
{
    public static void FileOperations()
    {
        // using 문 - 자동으로 Dispose 호출
        using (FileStream fs = new FileStream("test.txt", FileMode.Create))
        using (StreamWriter writer = new StreamWriter(fs))
        {
            writer.WriteLine("Hello, World!");
        }  // 여기서 writer.Dispose()와 fs.Dispose() 자동 호출
        
        // using 선언 (C# 8.0+)
        using var file = new FileStream("test2.txt", FileMode.Create);
        using var writer2 = new StreamWriter(file);
        writer2.WriteLine("Modern using syntax");
        // 메서드 끝에서 자동 Dispose
    }
    
    // 커스텀 Disposable 클래스
    public class DatabaseConnection : IDisposable
    {
        private SqlConnection connection;
        
        public DatabaseConnection(string connectionString)
        {
            connection = new SqlConnection(connectionString);
            connection.Open();
        }
        
        public void Dispose()
        {
            connection?.Close();
            connection?.Dispose();
        }
    }
}
```

### Dispose 패턴 베스트 프랙티스
```csharp
public class DisposableBase : IDisposable
{
    private bool disposed = false;
    
    // 자식 클래스가 오버라이드할 수 있도록 virtual
    protected virtual void Dispose(bool disposing)
    {
        if (!disposed)
        {
            if (disposing)
            {
                // 관리 리소스 해제
                DisposeManagedResources();
            }
            
            // 비관리 리소스 해제
            DisposeUnmanagedResources();
            
            disposed = true;
        }
    }
    
    protected virtual void DisposeManagedResources()
    {
        // 자식 클래스에서 구현
    }
    
    protected virtual void DisposeUnmanagedResources()
    {
        // 자식 클래스에서 구현
    }
    
    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }
    
    ~DisposableBase()
    {
        Dispose(false);
    }
}

// 사용 예
public class MyResource : DisposableBase
{
    private FileStream fileStream;
    private IntPtr handle;
    
    protected override void DisposeManagedResources()
    {
        fileStream?.Dispose();
    }
    
    protected override void DisposeUnmanagedResources()
    {
        if (handle != IntPtr.Zero)
        {
            // 핸들 해제
            CloseHandle(handle);
        }
    }
    
    [DllImport("kernel32.dll")]
    private static extern bool CloseHandle(IntPtr handle);
}
```

## 가비지 컬렉션 튜닝

### GC 모드 설정
```csharp
public class GCConfiguration
{
    public static void ConfigureGC()
    {
        // GC 설정 확인
        Console.WriteLine($"Server GC: {GCSettings.IsServerGC}");
        Console.WriteLine($"Latency Mode: {GCSettings.LatencyMode}");
        
        // Latency 모드 변경
        GCLatencyMode oldMode = GCSettings.LatencyMode;
        try
        {
            // 낮은 지연시간 모드
            GCSettings.LatencyMode = GCLatencyMode.LowLatency;
            
            // 시간이 중요한 작업 수행
            PerformTimeCriticalOperation();
        }
        finally
        {
            GCSettings.LatencyMode = oldMode;
        }
    }
    
    private static void PerformTimeCriticalOperation()
    {
        // 실시간 처리 등
    }
}
```

### 대용량 객체 힙 (LOH)
```csharp
public class LargeObjectHeap
{
    public static void LOHDemo()
    {
        // 85,000 바이트 이상은 LOH에 할당
        byte[] smallArray = new byte[84_999];  // SOH (Small Object Heap)
        byte[] largeArray = new byte[85_000];  // LOH (Large Object Heap)
        
        Console.WriteLine($"Small array generation: {GC.GetGeneration(smallArray)}");
        Console.WriteLine($"Large array generation: {GC.GetGeneration(largeArray)}");
        
        // LOH 압축
        GCSettings.LargeObjectHeapCompactionMode = GCLargeObjectHeapCompactionMode.CompactOnce;
        GC.Collect();
    }
    
    // ArrayPool을 사용한 최적화
    public static void UseArrayPool()
    {
        var pool = ArrayPool<byte>.Shared;
        
        // 배열 대여
        byte[] buffer = pool.Rent(1024 * 1024);  // 1MB
        try
        {
            // 버퍼 사용
            ProcessData(buffer);
        }
        finally
        {
            // 배열 반환
            pool.Return(buffer, clearArray: true);
        }
    }
    
    private static void ProcessData(byte[] buffer)
    {
        // 데이터 처리
    }
}
```

### 메모리 압력과 GC
```csharp
public class MemoryPressure
{
    class UnmanagedMemoryConsumer
    {
        private IntPtr nativeMemory;
        private long memorySize;
        
        public UnmanagedMemoryConsumer(long size)
        {
            memorySize = size;
            nativeMemory = Marshal.AllocHGlobal(new IntPtr(size));
            
            // GC에 메모리 압력 알림
            GC.AddMemoryPressure(size);
        }
        
        ~UnmanagedMemoryConsumer()
        {
            if (nativeMemory != IntPtr.Zero)
            {
                Marshal.FreeHGlobal(nativeMemory);
                GC.RemoveMemoryPressure(memorySize);
            }
        }
    }
    
    public static void TestMemoryPressure()
    {
        Console.WriteLine($"Initial memory: {GC.GetTotalMemory(false):N0}");
        
        // 큰 비관리 메모리 할당
        var consumer = new UnmanagedMemoryConsumer(100 * 1024 * 1024);  // 100MB
        
        Console.WriteLine($"After allocation: {GC.GetTotalMemory(false):N0}");
        
        // GC가 더 자주 실행됨
        for (int i = 0; i < 10; i++)
        {
            var temp = new byte[1024 * 1024];  // 1MB
            Console.WriteLine($"Gen0 collections: {GC.CollectionCount(0)}");
        }
    }
}
```

## 약한 참조 (Weak Reference)

```csharp
public class WeakReferenceExample
{
    class ExpensiveData
    {
        public byte[] Data { get; }
        public string Name { get; }
        
        public ExpensiveData(string name, int size)
        {
            Name = name;
            Data = new byte[size];
            Console.WriteLine($"Created: {name}");
        }
        
        ~ExpensiveData()
        {
            Console.WriteLine($"Finalized: {Name}");
        }
    }
    
    public static void WeakReferenceDemo()
    {
        // 강한 참조
        var strongRef = new ExpensiveData("Strong", 1024 * 1024);
        
        // 약한 참조
        var weakRef = new WeakReference(new ExpensiveData("Weak", 1024 * 1024));
        
        // 약한 참조 확인
        if (weakRef.IsAlive)
        {
            var data = (ExpensiveData)weakRef.Target;
            Console.WriteLine($"Weak reference alive: {data.Name}");
        }
        
        // GC 실행
        GC.Collect();
        GC.WaitForPendingFinalizers();
        
        // 강한 참조는 살아있음
        Console.WriteLine($"Strong reference: {strongRef.Name}");
        
        // 약한 참조는 수집됨
        if (!weakRef.IsAlive)
        {
            Console.WriteLine("Weak reference collected");
        }
    }
    
    // 캐시에서 약한 참조 활용
    public class WeakCache<TKey, TValue> where TValue : class
    {
        private readonly Dictionary<TKey, WeakReference> cache = new();
        
        public void Add(TKey key, TValue value)
        {
            cache[key] = new WeakReference(value);
        }
        
        public bool TryGetValue(TKey key, out TValue value)
        {
            if (cache.TryGetValue(key, out WeakReference weakRef) && weakRef.IsAlive)
            {
                value = (TValue)weakRef.Target;
                return true;
            }
            
            value = null;
            return false;
        }
        
        public void CleanUp()
        {
            var deadKeys = cache.Where(kvp => !kvp.Value.IsAlive)
                                .Select(kvp => kvp.Key)
                                .ToList();
            
            foreach (var key in deadKeys)
            {
                cache.Remove(key);
            }
        }
    }
}
```

## 성능 모니터링

```csharp
public class GCMonitoring
{
    public static void MonitorGC()
    {
        // GC 이벤트 등록
        GC.RegisterForFullGCNotification(10, 10);
        
        Thread monitorThread = new Thread(() =>
        {
            while (true)
            {
                GCNotificationStatus status = GC.WaitForFullGCApproach();
                if (status == GCNotificationStatus.Succeeded)
                {
                    Console.WriteLine("Full GC is approaching");
                    OnFullGCApproaching();
                }
                
                status = GC.WaitForFullGCComplete();
                if (status == GCNotificationStatus.Succeeded)
                {
                    Console.WriteLine("Full GC completed");
                    OnFullGCCompleted();
                }
            }
        });
        
        monitorThread.IsBackground = true;
        monitorThread.Start();
    }
    
    private static void OnFullGCApproaching()
    {
        // GC 전 처리 (예: 캐시 정리)
    }
    
    private static void OnFullGCCompleted()
    {
        // GC 후 처리
        Console.WriteLine($"Gen0: {GC.CollectionCount(0)}");
        Console.WriteLine($"Gen1: {GC.CollectionCount(1)}");
        Console.WriteLine($"Gen2: {GC.CollectionCount(2)}");
        Console.WriteLine($"Total Memory: {GC.GetTotalMemory(false):N0}");
    }
    
    // 성능 카운터
    public static void MeasureGCImpact()
    {
        var sw = Stopwatch.StartNew();
        long initialGen0 = GC.CollectionCount(0);
        long initialGen1 = GC.CollectionCount(1);
        long initialGen2 = GC.CollectionCount(2);
        
        // 작업 수행
        DoWork();
        
        sw.Stop();
        
        Console.WriteLine($"Elapsed: {sw.ElapsedMilliseconds}ms");
        Console.WriteLine($"Gen0 Collections: {GC.CollectionCount(0) - initialGen0}");
        Console.WriteLine($"Gen1 Collections: {GC.CollectionCount(1) - initialGen1}");
        Console.WriteLine($"Gen2 Collections: {GC.CollectionCount(2) - initialGen2}");
    }
    
    private static void DoWork()
    {
        // 메모리 집약적 작업
        for (int i = 0; i < 10000; i++)
        {
            var data = new byte[1024];
        }
    }
}
```

## 메모리 최적화 기법

```csharp
public class MemoryOptimization
{
    // 구조체 사용으로 힙 할당 줄이기
    public struct Point3D
    {
        public float X, Y, Z;
        
        public Point3D(float x, float y, float z)
        {
            X = x; Y = y; Z = z;
        }
    }
    
    // 객체 풀링
    public class ObjectPool<T> where T : class, new()
    {
        private readonly ConcurrentBag<T> objects = new();
        private readonly Func<T> objectGenerator;
        private readonly Action<T> resetAction;
        
        public ObjectPool(Func<T> generator = null, Action<T> reset = null)
        {
            objectGenerator = generator ?? (() => new T());
            resetAction = reset;
        }
        
        public T Rent()
        {
            if (objects.TryTake(out T item))
            {
                return item;
            }
            
            return objectGenerator();
        }
        
        public void Return(T item)
        {
            resetAction?.Invoke(item);
            objects.Add(item);
        }
    }
    
    // Span<T> 사용
    public static void UseSpan()
    {
        // 스택 할당
        Span<int> numbers = stackalloc int[100];
        
        for (int i = 0; i < numbers.Length; i++)
        {
            numbers[i] = i;
        }
        
        // 배열의 일부분을 Span으로
        int[] array = new int[1000];
        Span<int> slice = array.AsSpan(100, 50);
        
        ProcessNumbers(slice);
    }
    
    private static void ProcessNumbers(Span<int> numbers)
    {
        for (int i = 0; i < numbers.Length; i++)
        {
            numbers[i] *= 2;
        }
    }
}
```

## 핵심 개념 정리
- **가비지 컬렉션**: 자동 메모리 관리 시스템
- **세대별 GC**: Gen0, Gen1, Gen2로 객체 수명 관리
- **Dispose 패턴**: 명시적 리소스 해제
- **약한 참조**: GC 대상이 될 수 있는 참조
- **메모리 최적화**: 구조체, 객체 풀, Span<T> 활용