# 리플렉션과 애트리뷰트 (Reflection and Attributes)

## 리플렉션 (Reflection)

리플렉션은 런타임에 타입의 메타데이터를 검사하고 조작할 수 있는 기능입니다.

### Type 클래스
```csharp
// Type 객체 얻기
Type type1 = typeof(string);
Type type2 = "Hello".GetType();
Type type3 = Type.GetType("System.String");

// 타입 정보 확인
Console.WriteLine($"Name: {type1.Name}");
Console.WriteLine($"FullName: {type1.FullName}");
Console.WriteLine($"Namespace: {type1.Namespace}");
Console.WriteLine($"IsClass: {type1.IsClass}");
Console.WriteLine($"IsValueType: {type1.IsValueType}");
Console.WriteLine($"BaseType: {type1.BaseType?.Name}");
```

### 멤버 정보 얻기
```csharp
public class Person
{
    public string Name { get; set; }
    private int age;
    
    public Person(string name) => Name = name;
    
    public void SayHello() => Console.WriteLine($"Hello, I'm {Name}");
    private void Secret() => Console.WriteLine("This is private");
}

// 타입 멤버 검사
Type personType = typeof(Person);

// 프로퍼티
PropertyInfo[] properties = personType.GetProperties();
foreach (var prop in properties)
{
    Console.WriteLine($"Property: {prop.Name}, Type: {prop.PropertyType}");
}

// 필드
FieldInfo[] fields = personType.GetFields(BindingFlags.NonPublic | BindingFlags.Instance);
foreach (var field in fields)
{
    Console.WriteLine($"Field: {field.Name}, Type: {field.FieldType}");
}

// 메서드
MethodInfo[] methods = personType.GetMethods();
foreach (var method in methods)
{
    Console.WriteLine($"Method: {method.Name}, Return: {method.ReturnType}");
}

// 생성자
ConstructorInfo[] constructors = personType.GetConstructors();
foreach (var ctor in constructors)
{
    var parameters = ctor.GetParameters();
    Console.WriteLine($"Constructor with {parameters.Length} parameters");
}
```

### 리플렉션을 이용한 객체 생성
```csharp
// 1. Activator 사용
Type personType = typeof(Person);
object person1 = Activator.CreateInstance(personType, "John");

// 2. ConstructorInfo 사용
ConstructorInfo ctor = personType.GetConstructor(new[] { typeof(string) });
object person2 = ctor.Invoke(new object[] { "Jane" });

// 3. 제네릭 타입 생성
Type listType = typeof(List<>);
Type constructedType = listType.MakeGenericType(typeof(int));
object list = Activator.CreateInstance(constructedType);
```

### 동적 메서드 호출
```csharp
// 메서드 호출
Person person = new Person("John");
Type type = person.GetType();

// public 메서드
MethodInfo sayHello = type.GetMethod("SayHello");
sayHello.Invoke(person, null);

// private 메서드
MethodInfo secret = type.GetMethod("Secret", BindingFlags.NonPublic | BindingFlags.Instance);
secret.Invoke(person, null);

// 프로퍼티 값 설정/가져오기
PropertyInfo nameProp = type.GetProperty("Name");
nameProp.SetValue(person, "Jane");
string name = (string)nameProp.GetValue(person);
```

### 어셈블리 탐색
```csharp
// 현재 어셈블리
Assembly currentAssembly = Assembly.GetExecutingAssembly();
Console.WriteLine($"Assembly: {currentAssembly.FullName}");

// 어셈블리의 모든 타입
Type[] types = currentAssembly.GetTypes();
foreach (Type type in types)
{
    Console.WriteLine($"Type: {type.FullName}");
}

// 특정 어셈블리 로드
Assembly systemAssembly = Assembly.Load("System.Core");

// 외부 어셈블리 로드
Assembly external = Assembly.LoadFrom(@"C:\path\to\assembly.dll");
```

## 애트리뷰트 (Attributes)

애트리뷰트는 코드에 메타데이터를 추가하는 방법입니다.

### 기본 애트리뷰트 사용
```csharp
[Serializable]
public class Product
{
    [Required]
    public string Name { get; set; }
    
    [Range(0, 1000)]
    public decimal Price { get; set; }
    
    [Obsolete("Use NewMethod instead")]
    public void OldMethod() { }
    
    [Conditional("DEBUG")]
    public void DebugInfo()
    {
        Console.WriteLine("Debug information");
    }
}

// 메서드 애트리뷰트
public class Service
{
    [DebuggerStepThrough]
    public void UtilityMethod() { }
    
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public int FastMethod() => 42;
}
```

### 애트리뷰트 매개변수
```csharp
// 위치 매개변수와 명명된 매개변수
[AttributeUsage(AttributeTargets.Class | AttributeTargets.Method, 
                Inherited = false, 
                AllowMultiple = true)]
public class CustomAttribute : Attribute
{
    public string Description { get; set; }
    public int Version { get; set; }
    
    public CustomAttribute(string description)
    {
        Description = description;
    }
}

// 사용
[Custom("Main class", Version = 1)]
[Custom("Updated", Version = 2)]
public class MyClass { }
```

### 호출자 정보 애트리뷰트
```csharp
public class Logger
{
    public static void Log(
        string message,
        [CallerMemberName] string memberName = "",
        [CallerFilePath] string filePath = "",
        [CallerLineNumber] int lineNumber = 0)
    {
        Console.WriteLine($"{DateTime.Now}: {message}");
        Console.WriteLine($"  Member: {memberName}");
        Console.WriteLine($"  File: {Path.GetFileName(filePath)}");
        Console.WriteLine($"  Line: {lineNumber}");
    }
}

// 사용
public void DoSomething()
{
    Logger.Log("Something happened");  // 자동으로 호출자 정보 전달
}
```

## 사용자 정의 애트리뷰트

### 애트리뷰트 클래스 생성
```csharp
[AttributeUsage(AttributeTargets.Property)]
public class ColumnAttribute : Attribute
{
    public string Name { get; set; }
    public bool IsPrimaryKey { get; set; }
    public bool IsRequired { get; set; }
    
    public ColumnAttribute(string name)
    {
        Name = name;
    }
}

// 사용
public class User
{
    [Column("user_id", IsPrimaryKey = true)]
    public int Id { get; set; }
    
    [Column("username", IsRequired = true)]
    public string Username { get; set; }
    
    [Column("email", IsRequired = true)]
    public string Email { get; set; }
}
```

### 애트리뷰트 읽기
```csharp
public static class AttributeHelper
{
    public static T GetAttribute<T>(MemberInfo member) where T : Attribute
    {
        return member.GetCustomAttribute<T>();
    }
    
    public static bool HasAttribute<T>(MemberInfo member) where T : Attribute
    {
        return member.GetCustomAttribute<T>() != null;
    }
}

// 사용
Type userType = typeof(User);
foreach (PropertyInfo prop in userType.GetProperties())
{
    ColumnAttribute column = prop.GetCustomAttribute<ColumnAttribute>();
    if (column != null)
    {
        Console.WriteLine($"Property: {prop.Name}");
        Console.WriteLine($"  Column: {column.Name}");
        Console.WriteLine($"  Primary Key: {column.IsPrimaryKey}");
        Console.WriteLine($"  Required: {column.IsRequired}");
    }
}
```

## 실전 예제

### 1. 플러그인 시스템
```csharp
public interface IPlugin
{
    string Name { get; }
    void Execute();
}

public class PluginLoader
{
    public List<IPlugin> LoadPlugins(string directory)
    {
        var plugins = new List<IPlugin>();
        
        foreach (string file in Directory.GetFiles(directory, "*.dll"))
        {
            Assembly assembly = Assembly.LoadFrom(file);
            
            foreach (Type type in assembly.GetTypes())
            {
                if (typeof(IPlugin).IsAssignableFrom(type) && !type.IsAbstract)
                {
                    IPlugin plugin = (IPlugin)Activator.CreateInstance(type);
                    plugins.Add(plugin);
                }
            }
        }
        
        return plugins;
    }
}
```

### 2. 의존성 주입 컨테이너
```csharp
[AttributeUsage(AttributeTargets.Constructor)]
public class InjectAttribute : Attribute { }

public class SimpleContainer
{
    private Dictionary<Type, Type> registrations = new Dictionary<Type, Type>();
    
    public void Register<TInterface, TImplementation>()
        where TImplementation : TInterface
    {
        registrations[typeof(TInterface)] = typeof(TImplementation);
    }
    
    public T Resolve<T>()
    {
        return (T)Resolve(typeof(T));
    }
    
    private object Resolve(Type type)
    {
        if (registrations.TryGetValue(type, out Type implementationType))
        {
            type = implementationType;
        }
        
        // 생성자 찾기
        ConstructorInfo ctor = type.GetConstructors()
            .FirstOrDefault(c => c.GetCustomAttribute<InjectAttribute>() != null)
            ?? type.GetConstructors().FirstOrDefault();
        
        if (ctor == null)
            throw new InvalidOperationException($"No constructor found for {type}");
        
        // 매개변수 해결
        ParameterInfo[] parameters = ctor.GetParameters();
        object[] parameterInstances = new object[parameters.Length];
        
        for (int i = 0; i < parameters.Length; i++)
        {
            parameterInstances[i] = Resolve(parameters[i].ParameterType);
        }
        
        return ctor.Invoke(parameterInstances);
    }
}
```

### 3. ORM 매핑
```csharp
[AttributeUsage(AttributeTargets.Class)]
public class TableAttribute : Attribute
{
    public string Name { get; }
    public TableAttribute(string name) => Name = name;
}

public class MiniORM
{
    public string GenerateSelectQuery<T>()
    {
        Type type = typeof(T);
        TableAttribute table = type.GetCustomAttribute<TableAttribute>();
        string tableName = table?.Name ?? type.Name;
        
        var columns = type.GetProperties()
            .Select(p => 
            {
                var col = p.GetCustomAttribute<ColumnAttribute>();
                return col?.Name ?? p.Name;
            });
        
        return $"SELECT {string.Join(", ", columns)} FROM {tableName}";
    }
    
    public T MapToObject<T>(Dictionary<string, object> row) where T : new()
    {
        T obj = new T();
        Type type = typeof(T);
        
        foreach (PropertyInfo prop in type.GetProperties())
        {
            ColumnAttribute col = prop.GetCustomAttribute<ColumnAttribute>();
            string columnName = col?.Name ?? prop.Name;
            
            if (row.TryGetValue(columnName, out object value))
            {
                prop.SetValue(obj, Convert.ChangeType(value, prop.PropertyType));
            }
        }
        
        return obj;
    }
}
```

### 4. 검증 프레임워크
```csharp
[AttributeUsage(AttributeTargets.Property)]
public abstract class ValidationAttribute : Attribute
{
    public string ErrorMessage { get; set; }
    public abstract bool IsValid(object value);
}

public class RequiredAttribute : ValidationAttribute
{
    public override bool IsValid(object value)
    {
        return value != null && !string.IsNullOrWhiteSpace(value.ToString());
    }
}

public class RangeAttribute : ValidationAttribute
{
    public double Min { get; }
    public double Max { get; }
    
    public RangeAttribute(double min, double max)
    {
        Min = min;
        Max = max;
    }
    
    public override bool IsValid(object value)
    {
        if (value is IComparable comparable)
        {
            double val = Convert.ToDouble(value);
            return val >= Min && val <= Max;
        }
        return false;
    }
}

public class Validator
{
    public static List<string> Validate(object obj)
    {
        var errors = new List<string>();
        Type type = obj.GetType();
        
        foreach (PropertyInfo prop in type.GetProperties())
        {
            object value = prop.GetValue(obj);
            
            foreach (ValidationAttribute attr in prop.GetCustomAttributes<ValidationAttribute>())
            {
                if (!attr.IsValid(value))
                {
                    string error = attr.ErrorMessage ?? $"{prop.Name} is invalid";
                    errors.Add(error);
                }
            }
        }
        
        return errors;
    }
}
```

## 성능 고려사항

```csharp
// 리플렉션 성능 최적화
public class OptimizedReflection
{
    // 캐싱
    private static readonly Dictionary<Type, PropertyInfo[]> PropertyCache = new();
    
    public static PropertyInfo[] GetProperties(Type type)
    {
        if (!PropertyCache.TryGetValue(type, out PropertyInfo[] properties))
        {
            properties = type.GetProperties();
            PropertyCache[type] = properties;
        }
        return properties;
    }
    
    // 델리게이트로 변환
    public static Func<object, object> CreateGetter(PropertyInfo property)
    {
        var instance = Expression.Parameter(typeof(object), "instance");
        var instanceCast = Expression.Convert(instance, property.DeclaringType);
        var propertyAccess = Expression.Property(instanceCast, property);
        var castPropertyValue = Expression.Convert(propertyAccess, typeof(object));
        
        return Expression.Lambda<Func<object, object>>(castPropertyValue, instance).Compile();
    }
}
```

## 핵심 개념 정리
- **리플렉션**: 런타임에 타입 정보 검사 및 조작
- **Type 클래스**: 타입의 메타데이터 표현
- **애트리뷰트**: 선언적 메타데이터 추가
- **GetCustomAttribute**: 애트리뷰트 읽기
- **성능**: 리플렉션은 느리므로 캐싱 활용