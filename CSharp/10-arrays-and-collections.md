# 배열과 컬렉션

## 배열 (Array)

### 배열 선언과 생성
```csharp
// 배열 선언
int[] numbers;

// 배열 생성
numbers = new int[5];  // 크기 5인 배열

// 선언과 동시에 생성
int[] scores = new int[10];

// 타입 추론
var names = new string[3];
```

### 배열 초기화

```csharp
// 방법 1: 선언 후 개별 할당
int[] arr1 = new int[3];
arr1[0] = 10;
arr1[1] = 20;
arr1[2] = 30;

// 방법 2: 초기화자 사용
int[] arr2 = new int[] { 10, 20, 30 };

// 방법 3: 간단한 초기화
int[] arr3 = { 10, 20, 30 };

// 방법 4: var와 함께 사용
var arr4 = new[] { 10, 20, 30 };

// 크기 지정과 초기화 동시에
int[] arr5 = new int[5] { 1, 2, 3, 4, 5 };
```

## System.Array 클래스

### 유용한 메서드들
```csharp
int[] numbers = { 3, 1, 4, 1, 5, 9, 2, 6 };

// 정렬
Array.Sort(numbers);  // 1, 1, 2, 3, 4, 5, 6, 9

// 역순 정렬
Array.Reverse(numbers);  // 9, 6, 5, 4, 3, 2, 1, 1

// 검색
int index = Array.IndexOf(numbers, 5);  // 2
int lastIndex = Array.LastIndexOf(numbers, 1);  // 7

// 이진 검색 (정렬된 배열에서)
Array.Sort(numbers);
int binaryIndex = Array.BinarySearch(numbers, 5);

// 복사
int[] copy = new int[numbers.Length];
Array.Copy(numbers, copy, numbers.Length);

// 조건 검색
bool exists = Array.Exists(numbers, x => x > 5);
int first = Array.Find(numbers, x => x > 5);
int[] all = Array.FindAll(numbers, x => x > 5);
```

### Array 속성
```csharp
int[] arr = { 1, 2, 3, 4, 5 };

Console.WriteLine(arr.Length);      // 5
Console.WriteLine(arr.Rank);        // 1 (차원)
Console.WriteLine(arr.GetLength(0)); // 5 (첫 번째 차원의 길이)

// 읽기 전용 확인
Console.WriteLine(arr.IsReadOnly);  // False
Console.WriteLine(arr.IsFixedSize); // True
```

## 배열 분할 (슬라이싱)

### Range와 Index (C# 8.0+)
```csharp
int[] numbers = { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9 };

// Index
int last = numbers[^1];     // 9 (뒤에서 첫 번째)
int secondLast = numbers[^2]; // 8

// Range
int[] slice1 = numbers[2..5];   // { 2, 3, 4 }
int[] slice2 = numbers[..3];    // { 0, 1, 2 }
int[] slice3 = numbers[7..];    // { 7, 8, 9 }
int[] slice4 = numbers[^3..];   // { 7, 8, 9 }

// 전체 복사
int[] copy = numbers[..];
```

### ArraySegment
```csharp
int[] array = { 1, 2, 3, 4, 5, 6, 7, 8 };

// 배열의 일부를 참조
ArraySegment<int> segment = new ArraySegment<int>(array, 2, 4);
// 인덱스 2부터 4개 요소: { 3, 4, 5, 6 }

foreach (int item in segment)
{
    Console.WriteLine(item);
}
```

## 2차원 배열

### 직사각형 배열
```csharp
// 선언과 생성
int[,] matrix = new int[3, 4];  // 3행 4열

// 초기화
int[,] scores = new int[,]
{
    { 90, 85, 78, 92 },
    { 88, 92, 85, 90 },
    { 76, 85, 90, 88 }
};

// 간단한 초기화
int[,] grid = 
{
    { 1, 2, 3 },
    { 4, 5, 6 }
};

// 접근
matrix[0, 0] = 10;
int value = scores[1, 2];  // 85

// 순회
for (int i = 0; i < scores.GetLength(0); i++)
{
    for (int j = 0; j < scores.GetLength(1); j++)
    {
        Console.Write($"{scores[i, j]} ");
    }
    Console.WriteLine();
}
```

## 다차원 배열

```csharp
// 3차원 배열
int[,,] cube = new int[2, 3, 4];

// 초기화
int[,,] data = new int[,,]
{
    {
        { 1, 2, 3, 4 },
        { 5, 6, 7, 8 },
        { 9, 10, 11, 12 }
    },
    {
        { 13, 14, 15, 16 },
        { 17, 18, 19, 20 },
        { 21, 22, 23, 24 }
    }
};

// 차원 정보
Console.WriteLine($"Rank: {data.Rank}");  // 3
Console.WriteLine($"Dimension 0: {data.GetLength(0)}");  // 2
Console.WriteLine($"Dimension 1: {data.GetLength(1)}");  // 3
Console.WriteLine($"Dimension 2: {data.GetLength(2)}");  // 4
```

## 가변 배열 (Jagged Array)

```csharp
// 가변 배열 선언
int[][] jagged = new int[3][];

// 각 행에 다른 크기의 배열 할당
jagged[0] = new int[5];
jagged[1] = new int[3];
jagged[2] = new int[4];

// 초기화와 함께 생성
int[][] scores = new int[][]
{
    new int[] { 90, 85 },
    new int[] { 88, 92, 85, 90 },
    new int[] { 76 }
};

// 접근
jagged[0][0] = 10;
int value = scores[1][2];  // 85

// 순회
for (int i = 0; i < scores.Length; i++)
{
    for (int j = 0; j < scores[i].Length; j++)
    {
        Console.Write($"{scores[i][j]} ");
    }
    Console.WriteLine();
}
```

## 컬렉션 (Collections)

### ArrayList
```csharp
using System.Collections;

ArrayList list = new ArrayList();

// 추가
list.Add(10);
list.Add("Hello");
list.Add(3.14);
list.Add(true);

// 삽입
list.Insert(1, "World");

// 제거
list.Remove("Hello");
list.RemoveAt(0);

// 접근 (박싱/언박싱 주의)
object item = list[0];
int number = (int)list[0];  // 언박싱

// 순회
foreach (object obj in list)
{
    Console.WriteLine(obj);
}

// 정렬 (같은 타입일 때만)
ArrayList numbers = new ArrayList { 3, 1, 4, 1, 5 };
numbers.Sort();
```

### Queue (FIFO)
```csharp
Queue queue = new Queue();

// 인큐
queue.Enqueue("First");
queue.Enqueue("Second");
queue.Enqueue("Third");

// 디큐
object first = queue.Dequeue();  // "First"

// Peek (제거하지 않고 확인)
object next = queue.Peek();  // "Second"

// 순회
foreach (object item in queue)
{
    Console.WriteLine(item);
}
```

### Stack (LIFO)
```csharp
Stack stack = new Stack();

// 푸시
stack.Push("First");
stack.Push("Second");
stack.Push("Third");

// 팝
object top = stack.Pop();  // "Third"

// Peek
object next = stack.Peek();  // "Second"

// 순회 (역순)
foreach (object item in stack)
{
    Console.WriteLine(item);
}
```

### Hashtable
```csharp
Hashtable table = new Hashtable();

// 추가
table.Add("name", "John");
table.Add("age", 30);
table["city"] = "Seoul";  // 인덱서 사용

// 접근
string name = (string)table["name"];

// 키 존재 확인
if (table.ContainsKey("age"))
{
    int age = (int)table["age"];
}

// 순회
foreach (DictionaryEntry entry in table)
{
    Console.WriteLine($"{entry.Key}: {entry.Value}");
}

// 키와 값 컬렉션
ICollection keys = table.Keys;
ICollection values = table.Values;
```

## 컬렉션 초기화

### 컬렉션 초기화자
```csharp
// ArrayList
ArrayList list = new ArrayList { 1, 2, 3, "Hello" };

// Hashtable
Hashtable table = new Hashtable
{
    { "name", "John" },
    { "age", 30 },
    { "city", "Seoul" }
};

// 중첩 컬렉션
ArrayList nestedList = new ArrayList
{
    new ArrayList { 1, 2, 3 },
    new ArrayList { 4, 5, 6 }
};
```

## 인덱서

### 인덱서 구현
```csharp
public class MyList
{
    private int[] data = new int[10];
    
    // 인덱서
    public int this[int index]
    {
        get
        {
            if (index < 0 || index >= data.Length)
                throw new IndexOutOfRangeException();
            return data[index];
        }
        set
        {
            if (index < 0 || index >= data.Length)
                throw new IndexOutOfRangeException();
            data[index] = value;
        }
    }
    
    // 다중 매개변수 인덱서
    public int this[int row, int col]
    {
        get => data[row * 5 + col];
        set => data[row * 5 + col] = value;
    }
}

// 사용
MyList list = new MyList();
list[0] = 10;
list[1, 2] = 20;
```

## foreach와 IEnumerable

### IEnumerable 구현
```csharp
public class MyCollection : IEnumerable
{
    private int[] items = { 1, 2, 3, 4, 5 };
    
    public IEnumerator GetEnumerator()
    {
        return items.GetEnumerator();
    }
}

// 사용
MyCollection collection = new MyCollection();
foreach (int item in collection)
{
    Console.WriteLine(item);
}
```

### yield return 사용
```csharp
public class FibonacciSequence : IEnumerable<int>
{
    private int count;
    
    public FibonacciSequence(int count)
    {
        this.count = count;
    }
    
    public IEnumerator<int> GetEnumerator()
    {
        int prev = 0, curr = 1;
        
        for (int i = 0; i < count; i++)
        {
            yield return prev;
            int next = prev + curr;
            prev = curr;
            curr = next;
        }
    }
    
    IEnumerator IEnumerable.GetEnumerator()
    {
        return GetEnumerator();
    }
}

// 사용
foreach (int fib in new FibonacciSequence(10))
{
    Console.WriteLine(fib);
}
```

## 배열과 컬렉션 성능 비교

```csharp
// 배열: 고정 크기, 타입 안전, 빠른 접근
int[] array = new int[1000];
array[500] = 42;  // O(1)

// ArrayList: 가변 크기, 타입 안전하지 않음, 박싱/언박싱
ArrayList list = new ArrayList();
list.Add(42);  // 박싱 발생
int value = (int)list[0];  // 언박싱

// List<T>: 가변 크기, 타입 안전, 제네릭 (권장)
List<int> genericList = new List<int>();
genericList.Add(42);  // 박싱 없음
int val = genericList[0];  // 언박싱 없음
```

## 실전 예제

### 성적 관리 시스템
```csharp
public class GradeManager
{
    private Dictionary<string, List<int>> studentGrades = new Dictionary<string, List<int>>();
    
    public void AddGrade(string student, int grade)
    {
        if (!studentGrades.ContainsKey(student))
        {
            studentGrades[student] = new List<int>();
        }
        studentGrades[student].Add(grade);
    }
    
    public double GetAverage(string student)
    {
        if (!studentGrades.ContainsKey(student))
            return 0;
        
        return studentGrades[student].Average();
    }
    
    public void DisplayAll()
    {
        foreach (var pair in studentGrades)
        {
            Console.WriteLine($"{pair.Key}: {string.Join(", ", pair.Value)} (Avg: {GetAverage(pair.Key):F2})");
        }
    }
}
```

## 핵심 개념 정리
- **배열**: 고정 크기, 같은 타입, 빠른 접근
- **컬렉션**: 가변 크기, 유연한 데이터 구조
- **인덱서**: 객체를 배열처럼 사용
- **IEnumerable**: foreach 지원
- **제네릭 컬렉션**: 타입 안전성과 성능 (권장)