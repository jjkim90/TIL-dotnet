# 제네릭 프로그래밍 (Generic Programming)

## 제네릭이란?

제네릭은 타입을 매개변수화하여 코드의 재사용성을 높이는 프로그래밍 기법입니다.

### 제네릭이 없었을 때의 문제점
```csharp
// object를 사용한 범용 클래스
public class ObjectBox
{
    private object data;
    
    public void SetData(object data)
    {
        this.data = data;
    }
    
    public object GetData()
    {
        return data;
    }
}

// 사용 시 문제점
ObjectBox box1 = new ObjectBox();
box1.SetData(123);  // 박싱
int value = (int)box1.GetData();  // 언박싱, 타입 캐스팅 필요

ObjectBox box2 = new ObjectBox();
box2.SetData("Hello");
int wrong = (int)box2.GetData();  // 런타임 에러!
```

## 제네릭 메서드

### 기본 제네릭 메서드
```csharp
// 제네릭 메서드 정의
public T GetDefault<T>()
{
    return default(T);
}

// 두 값 교환
public void Swap<T>(ref T a, ref T b)
{
    T temp = a;
    a = b;
    b = temp;
}

// 사용
int x = 10, y = 20;
Swap<int>(ref x, ref y);  // 명시적 타입 지정
Swap(ref x, ref y);       // 타입 추론

string s1 = "Hello", s2 = "World";
Swap(ref s1, ref s2);
```

### 여러 타입 매개변수
```csharp
public TOutput Convert<TInput, TOutput>(TInput input, Func<TInput, TOutput> converter)
{
    return converter(input);
}

// 사용
string result = Convert<int, string>(123, x => x.ToString());
// 또는 타입 추론
var result2 = Convert(123, x => x.ToString());
```

## 제네릭 클래스

### 기본 제네릭 클래스
```csharp
public class Box<T>
{
    private T data;
    
    public void SetData(T data)
    {
        this.data = data;
    }
    
    public T GetData()
    {
        return data;
    }
}

// 사용
Box<int> intBox = new Box<int>();
intBox.SetData(123);  // 박싱 없음
int value = intBox.GetData();  // 언박싱 없음

Box<string> stringBox = new Box<string>();
stringBox.SetData("Hello");
string text = stringBox.GetData();
```

### 여러 타입 매개변수를 가진 클래스
```csharp
public class KeyValuePair<TKey, TValue>
{
    public TKey Key { get; set; }
    public TValue Value { get; set; }
    
    public KeyValuePair(TKey key, TValue value)
    {
        Key = key;
        Value = value;
    }
}

// 사용
var pair = new KeyValuePair<string, int>("Age", 25);
Console.WriteLine($"{pair.Key}: {pair.Value}");
```

## 형식 매개변수 제약

### where 키워드
```csharp
// 참조 타입 제약
public class ReferenceContainer<T> where T : class
{
    private T item;
    
    public void SetItem(T item)
    {
        this.item = item ?? throw new ArgumentNullException();
    }
}

// 값 타입 제약
public class ValueContainer<T> where T : struct
{
    private T value;
    private T? nullable;
}

// 기본 생성자 제약
public class Factory<T> where T : new()
{
    public T Create()
    {
        return new T();
    }
}
```

### 인터페이스 제약
```csharp
public interface IComparable<T>
{
    int CompareTo(T other);
}

public class SortedList<T> where T : IComparable<T>
{
    private List<T> items = new List<T>();
    
    public void Add(T item)
    {
        items.Add(item);
        items.Sort((x, y) => x.CompareTo(y));
    }
}

// 사용
public class Person : IComparable<Person>
{
    public string Name { get; set; }
    public int Age { get; set; }
    
    public int CompareTo(Person other)
    {
        return Age.CompareTo(other.Age);
    }
}
```

### 복합 제약
```csharp
public class ComplexContainer<T> 
    where T : class, IDisposable, new()
{
    public T CreateAndUse()
    {
        T item = new T();
        using (item)
        {
            // 사용
        }
        return new T();
    }
}

// 상속 제약
public class AnimalShelter<T> where T : Animal
{
    private List<T> animals = new List<T>();
    
    public void AddAnimal(T animal)
    {
        animals.Add(animal);
    }
}
```

## 제네릭 컬렉션

### List<T>
```csharp
// 타입 안전한 리스트
List<int> numbers = new List<int>();
numbers.Add(10);
numbers.Add(20);
numbers.AddRange(new[] { 30, 40, 50 });

// LINQ 메서드 사용
int sum = numbers.Sum();
double average = numbers.Average();
List<int> filtered = numbers.Where(x => x > 20).ToList();

// 정렬
numbers.Sort();
numbers.Sort((a, b) => b.CompareTo(a));  // 역순
```

### Queue<T>
```csharp
Queue<string> queue = new Queue<string>();

// 인큐
queue.Enqueue("First");
queue.Enqueue("Second");
queue.Enqueue("Third");

// 디큐
string first = queue.Dequeue();  // "First"

// Peek
string next = queue.Peek();  // "Second"

// 변환
string[] array = queue.ToArray();
```

### Stack<T>
```csharp
Stack<int> stack = new Stack<int>();

// 푸시
stack.Push(10);
stack.Push(20);
stack.Push(30);

// 팝
int top = stack.Pop();  // 30

// Peek
int next = stack.Peek();  // 20

// 조건부 검색
bool contains = stack.Contains(10);
```

### Dictionary<TKey, TValue>
```csharp
Dictionary<string, int> ages = new Dictionary<string, int>();

// 추가
ages.Add("John", 25);
ages["Jane"] = 30;  // 인덱서 사용

// 접근
int johnAge = ages["John"];

// 안전한 접근
if (ages.TryGetValue("Bob", out int bobAge))
{
    Console.WriteLine($"Bob's age: {bobAge}");
}

// 순회
foreach (KeyValuePair<string, int> pair in ages)
{
    Console.WriteLine($"{pair.Key}: {pair.Value}");
}

// 키/값 컬렉션
ICollection<string> keys = ages.Keys;
ICollection<int> values = ages.Values;
```

### HashSet<T>
```csharp
HashSet<int> set1 = new HashSet<int> { 1, 2, 3, 4, 5 };
HashSet<int> set2 = new HashSet<int> { 4, 5, 6, 7, 8 };

// 중복 제거
set1.Add(3);  // false 반환, 이미 존재

// 집합 연산
set1.UnionWith(set2);      // 합집합
set1.IntersectWith(set2);  // 교집합
set1.ExceptWith(set2);     // 차집합

// 부분집합/상위집합 확인
bool isSubset = set1.IsSubsetOf(set2);
bool isSuperset = set1.IsSupersetOf(set2);
```

## foreach를 사용할 수 있는 제네릭 클래스

### IEnumerable<T> 구현
```csharp
public class MyList<T> : IEnumerable<T>
{
    private T[] items;
    private int count;
    
    public MyList(int capacity = 10)
    {
        items = new T[capacity];
    }
    
    public void Add(T item)
    {
        if (count >= items.Length)
        {
            Array.Resize(ref items, items.Length * 2);
        }
        items[count++] = item;
    }
    
    public IEnumerator<T> GetEnumerator()
    {
        for (int i = 0; i < count; i++)
        {
            yield return items[i];
        }
    }
    
    IEnumerator IEnumerable.GetEnumerator()
    {
        return GetEnumerator();
    }
}

// 사용
MyList<string> myList = new MyList<string>();
myList.Add("Hello");
myList.Add("World");

foreach (string item in myList)
{
    Console.WriteLine(item);
}
```

## 제네릭과 성능

### 박싱/언박싱 제거
```csharp
// ArrayList (박싱 발생)
ArrayList list = new ArrayList();
for (int i = 0; i < 1000000; i++)
{
    list.Add(i);  // 박싱
}
int sum = 0;
foreach (object obj in list)
{
    sum += (int)obj;  // 언박싱
}

// List<T> (박싱 없음)
List<int> genericList = new List<int>();
for (int i = 0; i < 1000000; i++)
{
    genericList.Add(i);  // 박싱 없음
}
int genericSum = genericList.Sum();  // 언박싱 없음
```

## 고급 제네릭 패턴

### 제네릭 싱글톤
```csharp
public class Singleton<T> where T : class, new()
{
    private static readonly Lazy<T> instance = new Lazy<T>(() => new T());
    
    public static T Instance => instance.Value;
    
    protected Singleton() { }
}

// 사용
public class DatabaseConnection : Singleton<DatabaseConnection>
{
    public void Connect() { }
}

var db = DatabaseConnection.Instance;
```

### 제네릭 팩토리
```csharp
public interface IProduct
{
    string Name { get; }
}

public class ProductFactory<T> where T : IProduct, new()
{
    public T Create()
    {
        T product = new T();
        Console.WriteLine($"Created: {product.Name}");
        return product;
    }
}
```

### 공변성과 반공변성
```csharp
// 공변성 (out)
public interface IProducer<out T>
{
    T Produce();
}

// 반공변성 (in)
public interface IConsumer<in T>
{
    void Consume(T item);
}

// 사용
IProducer<string> stringProducer = null;
IProducer<object> objectProducer = stringProducer;  // 공변성

IConsumer<object> objectConsumer = null;
IConsumer<string> stringConsumer = objectConsumer;  // 반공변성
```

## 제네릭 메서드와 확장 메서드

```csharp
public static class Extensions
{
    // 제네릭 확장 메서드
    public static T SafeGet<T>(this IList<T> list, int index, T defaultValue = default)
    {
        if (index < 0 || index >= list.Count)
            return defaultValue;
        return list[index];
    }
    
    public static void ForEach<T>(this IEnumerable<T> source, Action<T> action)
    {
        foreach (T item in source)
        {
            action(item);
        }
    }
}

// 사용
List<int> numbers = new List<int> { 1, 2, 3 };
int value = numbers.SafeGet(10, -1);  // -1 반환

numbers.ForEach(x => Console.WriteLine(x * 2));
```

## 실전 예제: 제네릭 캐시

```csharp
public class Cache<TKey, TValue>
{
    private readonly Dictionary<TKey, CacheItem> cache = new Dictionary<TKey, CacheItem>();
    private readonly int maxSize;
    
    private class CacheItem
    {
        public TValue Value { get; set; }
        public DateTime LastAccessed { get; set; }
    }
    
    public Cache(int maxSize = 100)
    {
        this.maxSize = maxSize;
    }
    
    public void Add(TKey key, TValue value)
    {
        if (cache.Count >= maxSize)
        {
            RemoveOldest();
        }
        
        cache[key] = new CacheItem 
        { 
            Value = value, 
            LastAccessed = DateTime.Now 
        };
    }
    
    public bool TryGetValue(TKey key, out TValue value)
    {
        if (cache.TryGetValue(key, out CacheItem item))
        {
            item.LastAccessed = DateTime.Now;
            value = item.Value;
            return true;
        }
        
        value = default;
        return false;
    }
    
    private void RemoveOldest()
    {
        var oldest = cache.OrderBy(x => x.Value.LastAccessed).First();
        cache.Remove(oldest.Key);
    }
}
```

## 핵심 개념 정리
- **제네릭**: 타입을 매개변수화하여 재사용성 향상
- **타입 안전성**: 컴파일 시점에 타입 검사
- **성능**: 박싱/언박싱 제거
- **제약 조건**: where 키워드로 타입 제한
- **제네릭 컬렉션**: 타입 안전한 컬렉션 사용