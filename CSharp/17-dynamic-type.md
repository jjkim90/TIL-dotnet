# dynamic 형식

## dynamic 형식 소개

dynamic은 컴파일 시점이 아닌 런타임에 타입이 결정되는 특별한 형식입니다.

### dynamic vs object
```csharp
// object - 컴파일 시점에 타입 체크
object obj = 10;
// obj.ToString();  // 가능
// obj.DoSomething();  // 컴파일 오류! object에 DoSomething 메서드 없음

// dynamic - 런타임에 타입 체크
dynamic dyn = 10;
dyn.ToString();     // 가능
dyn.DoSomething();  // 컴파일은 되지만 런타임 오류 발생
```

### dynamic 기본 사용
```csharp
// 타입 변환 없이 다양한 작업 수행
dynamic value = 42;
Console.WriteLine(value.GetType());  // System.Int32

value = "Hello";
Console.WriteLine(value.GetType());  // System.String
Console.WriteLine(value.Length);     // 5

value = new List<int> { 1, 2, 3 };
value.Add(4);  // 리스트 메서드 호출
Console.WriteLine(value.Count);  // 4

// 산술 연산
dynamic a = 10;
dynamic b = 20;
dynamic result = a + b;  // 30
```

### 동적 멤버 접근
```csharp
public class Person
{
    public string Name { get; set; }
    public int Age { get; set; }
    
    public void Introduce()
    {
        Console.WriteLine($"Hi, I'm {Name}, {Age} years old.");
    }
}

// dynamic으로 사용
dynamic person = new Person { Name = "John", Age = 30 };
person.Introduce();  // 메서드 호출
person.Name = "Jane";  // 프로퍼티 설정
Console.WriteLine(person.Age);  // 프로퍼티 읽기

// 존재하지 않는 멤버 접근 시
try
{
    person.NonExistentMethod();  // RuntimeBinderException
}
catch (Microsoft.CSharp.RuntimeBinder.RuntimeBinderException ex)
{
    Console.WriteLine($"런타임 오류: {ex.Message}");
}
```

## 덕 타이핑 (Duck Typing)

"오리처럼 걷고 오리처럼 소리내면 오리다"

```csharp
public class Duck
{
    public void Quack() => Console.WriteLine("Quack!");
    public void Walk() => Console.WriteLine("Duck walking");
}

public class Person
{
    public void Quack() => Console.WriteLine("Person imitating: Quack!");
    public void Walk() => Console.WriteLine("Person walking");
}

public class Robot
{
    public void Quack() => Console.WriteLine("Robot sound: Quack!");
    public void Walk() => Console.WriteLine("Robot moving");
}

// 덕 타이핑 활용
void MakeDuckBehavior(dynamic duck)
{
    duck.Quack();
    duck.Walk();
}

// 사용
MakeDuckBehavior(new Duck());    // Duck 클래스
MakeDuckBehavior(new Person());  // Person 클래스
MakeDuckBehavior(new Robot());   // Robot 클래스
```

### 동적 메서드 디스패치
```csharp
public class Calculator
{
    public int Add(int a, int b) => a + b;
    public double Add(double a, double b) => a + b;
    public string Add(string a, string b) => a + b;
}

dynamic calc = new Calculator();
dynamic result1 = calc.Add(5, 3);        // int 버전 호출
dynamic result2 = calc.Add(5.5, 3.2);    // double 버전 호출
dynamic result3 = calc.Add("Hello", " World");  // string 버전 호출

Console.WriteLine($"{result1}, {result2}, {result3}");
```

## COM과 .NET 사이의 상호 운용성

### Office Interop 예제
```csharp
// Excel 자동화 (Microsoft.Office.Interop.Excel 참조 필요)
dynamic excel = Activator.CreateInstance(Type.GetTypeFromProgID("Excel.Application"));
excel.Visible = true;

dynamic workbook = excel.Workbooks.Add();
dynamic worksheet = workbook.ActiveSheet;

// 셀에 값 설정
worksheet.Cells[1, 1] = "Name";
worksheet.Cells[1, 2] = "Age";
worksheet.Cells[2, 1] = "John";
worksheet.Cells[2, 2] = 30;

// 차트 생성
dynamic charts = worksheet.ChartObjects();
dynamic chartObject = charts.Add(60, 10, 300, 300);
dynamic chart = chartObject.Chart;
chart.ChartType = -4169;  // xlXYScatter

// 정리
workbook.SaveAs(@"C:\temp\test.xlsx");
excel.Quit();
```

### COM 객체의 선택적 매개변수
```csharp
// dynamic 없이 (이전 방식)
object missing = Type.Missing;
worksheet.get_Range("A1", missing).set_Value(missing, "Hello");

// dynamic 사용 (간단)
worksheet.Range["A1"].Value = "Hello";
worksheet.Cells[1, 1].Value = "Hello";
```

## 동적 언어와의 상호 운용성

### IronPython 통합
```csharp
// IronPython 스크립트 실행
using Microsoft.Scripting.Hosting;
using IronPython.Hosting;

ScriptEngine engine = Python.CreateEngine();
ScriptScope scope = engine.CreateScope();

// Python 코드 실행
string pythonCode = @"
def greet(name):
    return f'Hello, {name} from Python!'

def calculate(x, y):
    return x ** y
";

engine.Execute(pythonCode, scope);

// Python 함수를 dynamic으로 호출
dynamic greet = scope.GetVariable("greet");
dynamic result = greet("C#");
Console.WriteLine(result);  // Hello, C# from Python!

dynamic calculate = scope.GetVariable("calculate");
Console.WriteLine(calculate(2, 10));  // 1024
```

### 동적 객체 생성
```csharp
// ExpandoObject 사용
dynamic expando = new System.Dynamic.ExpandoObject();
expando.Name = "Dynamic Object";
expando.Value = 42;
expando.Calculate = new Func<int, int, int>((x, y) => x + y);

Console.WriteLine(expando.Name);
Console.WriteLine(expando.Calculate(10, 20));  // 30

// 런타임에 멤버 추가
expando.NewProperty = "Added at runtime";
Console.WriteLine(expando.NewProperty);

// Dictionary로 변환
var dict = (IDictionary<string, object>)expando;
foreach (var kvp in dict)
{
    Console.WriteLine($"{kvp.Key}: {kvp.Value}");
}
```

## DynamicObject 상속

### 사용자 정의 동적 객체
```csharp
public class DynamicDictionary : DynamicObject
{
    private Dictionary<string, object> dictionary = new Dictionary<string, object>();
    
    public override bool TryGetMember(GetMemberBinder binder, out object result)
    {
        return dictionary.TryGetValue(binder.Name, out result);
    }
    
    public override bool TrySetMember(SetMemberBinder binder, object value)
    {
        dictionary[binder.Name] = value;
        return true;
    }
    
    public override bool TryInvokeMember(InvokeMemberBinder binder, object[] args, out object result)
    {
        if (dictionary.TryGetValue(binder.Name, out object value) && value is Delegate)
        {
            result = ((Delegate)value).DynamicInvoke(args);
            return true;
        }
        
        result = null;
        return false;
    }
}

// 사용
dynamic bag = new DynamicDictionary();
bag.Name = "John";
bag.Age = 30;
bag.Greet = new Action(() => Console.WriteLine($"Hello, I'm {bag.Name}"));

Console.WriteLine(bag.Name);  // John
bag.Greet();  // Hello, I'm John
```

### 동적 프록시 패턴
```csharp
public class DynamicProxy : DynamicObject
{
    private readonly object target;
    
    public DynamicProxy(object target)
    {
        this.target = target;
    }
    
    public override bool TryInvokeMember(InvokeMemberBinder binder, object[] args, out object result)
    {
        var method = target.GetType().GetMethod(binder.Name);
        if (method != null)
        {
            Console.WriteLine($"Before calling {binder.Name}");
            result = method.Invoke(target, args);
            Console.WriteLine($"After calling {binder.Name}");
            return true;
        }
        
        result = null;
        return false;
    }
}

// 사용
dynamic proxy = new DynamicProxy(new List<int>());
proxy.Add(10);  // 로깅과 함께 Add 호출
proxy.Add(20);
```

## 성능 고려사항

### dynamic의 오버헤드
```csharp
// 성능 비교
public void PerformanceTest()
{
    const int iterations = 1000000;
    
    // 정적 타입
    var sw1 = Stopwatch.StartNew();
    int sum1 = 0;
    for (int i = 0; i < iterations; i++)
    {
        sum1 = Add(i, i);
    }
    sw1.Stop();
    
    // dynamic
    var sw2 = Stopwatch.StartNew();
    dynamic sum2 = 0;
    for (int i = 0; i < iterations; i++)
    {
        sum2 = AddDynamic(i, i);
    }
    sw2.Stop();
    
    Console.WriteLine($"Static: {sw1.ElapsedMilliseconds}ms");
    Console.WriteLine($"Dynamic: {sw2.ElapsedMilliseconds}ms");
}

int Add(int a, int b) => a + b;
dynamic AddDynamic(dynamic a, dynamic b) => a + b;
```

### 캐싱을 통한 최적화
```csharp
public class DynamicCache
{
    private static readonly Dictionary<string, CallSite<Func<CallSite, object, object>>> 
        getterCache = new Dictionary<string, CallSite<Func<CallSite, object, object>>>();
    
    public static object GetProperty(object obj, string propertyName)
    {
        var key = $"{obj.GetType().FullName}.{propertyName}";
        
        if (!getterCache.TryGetValue(key, out var callSite))
        {
            var binder = Microsoft.CSharp.RuntimeBinder.Binder.GetMember(
                CSharpBinderFlags.None,
                propertyName,
                obj.GetType(),
                new[] { CSharpArgumentInfo.Create(CSharpArgumentInfoFlags.None, null) }
            );
            
            callSite = CallSite<Func<CallSite, object, object>>.Create(binder);
            getterCache[key] = callSite;
        }
        
        return callSite.Target(callSite, obj);
    }
}
```

## 실전 활용 예제

### 동적 JSON 처리
```csharp
using Newtonsoft.Json;

// JSON을 dynamic으로 파싱
string json = @"{
    'name': 'John Doe',
    'age': 30,
    'address': {
        'street': '123 Main St',
        'city': 'New York'
    },
    'hobbies': ['reading', 'gaming']
}";

dynamic data = JsonConvert.DeserializeObject(json);

Console.WriteLine(data.name);  // John Doe
Console.WriteLine(data.address.city);  // New York
Console.WriteLine(data.hobbies[0]);  // reading

// 동적으로 값 변경
data.age = 31;
data.email = "john@example.com";

string updatedJson = JsonConvert.SerializeObject(data, Formatting.Indented);
```

### 동적 설정 시스템
```csharp
public class DynamicConfig : DynamicObject
{
    private readonly string filePath;
    private Dictionary<string, object> settings;
    
    public DynamicConfig(string filePath)
    {
        this.filePath = filePath;
        LoadSettings();
    }
    
    private void LoadSettings()
    {
        if (File.Exists(filePath))
        {
            string json = File.ReadAllText(filePath);
            settings = JsonConvert.DeserializeObject<Dictionary<string, object>>(json) 
                      ?? new Dictionary<string, object>();
        }
        else
        {
            settings = new Dictionary<string, object>();
        }
    }
    
    public override bool TryGetMember(GetMemberBinder binder, out object result)
    {
        return settings.TryGetValue(binder.Name, out result);
    }
    
    public override bool TrySetMember(SetMemberBinder binder, object value)
    {
        settings[binder.Name] = value;
        SaveSettings();
        return true;
    }
    
    private void SaveSettings()
    {
        string json = JsonConvert.SerializeObject(settings, Formatting.Indented);
        File.WriteAllText(filePath, json);
    }
}

// 사용
dynamic config = new DynamicConfig("config.json");
config.DatabaseConnection = "Server=localhost;Database=MyDB";
config.MaxRetries = 3;
config.EnableLogging = true;
```

## 핵심 개념 정리
- **dynamic**: 런타임 타입 바인딩
- **덕 타이핑**: 메서드/프로퍼티 존재 여부로 타입 판단
- **COM Interop**: Office 등 COM 객체와 쉬운 상호작용
- **ExpandoObject**: 런타임에 멤버 추가 가능
- **성능**: 정적 타입보다 느림, 필요한 곳에만 사용