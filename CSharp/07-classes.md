# 클래스 (Classes)

## 객체지향 프로그래밍과 클래스

### 객체지향의 핵심 개념
- **캡슐화**: 데이터와 메서드를 하나로 묶음
- **상속**: 기존 코드 재사용
- **다형성**: 하나의 인터페이스로 여러 구현
- **추상화**: 복잡한 것을 단순화

### 클래스와 객체
```csharp
// 클래스: 설계도
public class Car
{
    public string Model;
    public int Year;
    
    public void Start()
    {
        Console.WriteLine("Engine started!");
    }
}

// 객체: 인스턴스
Car myCar = new Car();
myCar.Model = "Tesla Model 3";
myCar.Year = 2023;
myCar.Start();
```

## 클래스 선언과 객체 생성

### 클래스 구성 요소
```csharp
public class Person
{
    // 필드
    private string name;
    private int age;
    
    // 프로퍼티
    public string Name
    {
        get { return name; }
        set { name = value; }
    }
    
    // 메서드
    public void Introduce()
    {
        Console.WriteLine($"Hi, I'm {name}");
    }
    
    // 이벤트
    public event EventHandler NameChanged;
}
```

### 객체 생성과 초기화
```csharp
// 기본 생성
Person person1 = new Person();
person1.Name = "John";

// 객체 초기화자
Person person2 = new Person
{
    Name = "Jane",
    Age = 25
};

// var 키워드 사용
var person3 = new Person();
```

## 생성자와 종료자

### 생성자 (Constructor)
```csharp
public class Product
{
    private string name;
    private decimal price;
    
    // 기본 생성자
    public Product()
    {
        name = "Unknown";
        price = 0;
    }
    
    // 매개변수가 있는 생성자
    public Product(string name)
    {
        this.name = name;
        this.price = 0;
    }
    
    // 생성자 오버로딩
    public Product(string name, decimal price)
    {
        this.name = name;
        this.price = price;
    }
}
```

### 생성자 체이닝
```csharp
public class Employee
{
    private string name;
    private int id;
    private string department;
    
    public Employee() : this("Unknown", 0)
    {
    }
    
    public Employee(string name, int id) : this(name, id, "General")
    {
    }
    
    public Employee(string name, int id, string department)
    {
        this.name = name;
        this.id = id;
        this.department = department;
    }
}
```

### 종료자 (Destructor/Finalizer)
```csharp
public class FileHandler
{
    private FileStream fileStream;
    
    public FileHandler(string filename)
    {
        fileStream = new FileStream(filename, FileMode.Open);
    }
    
    // 종료자
    ~FileHandler()
    {
        // 관리되지 않는 리소스 정리
        fileStream?.Close();
    }
}
```

## 정적 필드와 메서드

### static 멤버
```csharp
public class MathUtils
{
    // 정적 필드
    public static readonly double PI = 3.14159;
    private static int callCount = 0;
    
    // 정적 메서드
    public static double Square(double x)
    {
        callCount++;
        return x * x;
    }
    
    // 정적 프로퍼티
    public static int CallCount
    {
        get { return callCount; }
    }
}

// 사용
double area = MathUtils.PI * MathUtils.Square(5);
Console.WriteLine($"Called {MathUtils.CallCount} times");
```

### 정적 생성자
```csharp
public class Configuration
{
    public static string ConnectionString { get; private set; }
    
    // 정적 생성자
    static Configuration()
    {
        // 한 번만 실행됨
        ConnectionString = LoadFromFile();
    }
    
    private static string LoadFromFile()
    {
        // 설정 파일에서 읽기
        return "Server=localhost;Database=MyDB";
    }
}
```

## 객체 복사

### 얕은 복사 (Shallow Copy)
```csharp
public class Address
{
    public string City { get; set; }
}

public class Person
{
    public string Name { get; set; }
    public Address Address { get; set; }
    
    // 얕은 복사
    public Person ShallowCopy()
    {
        return (Person)this.MemberwiseClone();
    }
}

// 사용
Person original = new Person
{
    Name = "John",
    Address = new Address { City = "Seoul" }
};

Person copy = original.ShallowCopy();
copy.Address.City = "Busan";  // 원본도 변경됨!
```

### 깊은 복사 (Deep Copy)
```csharp
public class Person : ICloneable
{
    public string Name { get; set; }
    public Address Address { get; set; }
    
    // 깊은 복사
    public object Clone()
    {
        Person clone = (Person)this.MemberwiseClone();
        clone.Address = new Address { City = this.Address.City };
        return clone;
    }
}
```

## this 키워드

### 인스턴스 참조
```csharp
public class Rectangle
{
    private double width;
    private double height;
    
    public Rectangle(double width, double height)
    {
        this.width = width;    // this로 필드 구분
        this.height = height;
    }
    
    public Rectangle GetScaled(double factor)
    {
        return new Rectangle(this.width * factor, this.height * factor);
    }
}
```

### this() 생성자
```csharp
public class Book
{
    public string Title { get; set; }
    public string Author { get; set; }
    public int Pages { get; set; }
    
    public Book() : this("Unknown", "Unknown", 0)
    {
    }
    
    public Book(string title) : this(title, "Unknown", 0)
    {
    }
    
    public Book(string title, string author, int pages)
    {
        Title = title;
        Author = author;
        Pages = pages;
    }
}
```

## 접근 한정자

### 접근 수준
```csharp
public class AccessExample
{
    public int PublicField;           // 모든 곳에서 접근
    private int privateField;         // 클래스 내부만
    protected int protectedField;     // 클래스와 파생 클래스
    internal int internalField;       // 같은 어셈블리
    protected internal int piField;   // 같은 어셈블리 + 파생 클래스
    private protected int ppField;    // 같은 어셈블리의 파생 클래스
}
```

### 캡슐화 예제
```csharp
public class BankAccount
{
    private decimal balance;
    
    public decimal Balance
    {
        get { return balance; }
        private set { balance = value; }
    }
    
    public void Deposit(decimal amount)
    {
        if (amount > 0)
            balance += amount;
    }
    
    public bool Withdraw(decimal amount)
    {
        if (amount > 0 && amount <= balance)
        {
            balance -= amount;
            return true;
        }
        return false;
    }
}
```

## 상속

### 기본 상속
```csharp
// 기반 클래스
public class Animal
{
    public string Name { get; set; }
    
    public virtual void MakeSound()
    {
        Console.WriteLine("Some generic animal sound");
    }
}

// 파생 클래스
public class Dog : Animal
{
    public string Breed { get; set; }
    
    public override void MakeSound()
    {
        Console.WriteLine("Woof!");
    }
}
```

### base 키워드
```csharp
public class Vehicle
{
    protected string brand;
    
    public Vehicle(string brand)
    {
        this.brand = brand;
    }
    
    public virtual void Start()
    {
        Console.WriteLine($"{brand} is starting");
    }
}

public class Car : Vehicle
{
    private int doors;
    
    public Car(string brand, int doors) : base(brand)
    {
        this.doors = doors;
    }
    
    public override void Start()
    {
        base.Start();  // 부모 메서드 호출
        Console.WriteLine($"Car with {doors} doors is ready");
    }
}
```

## 형식 변환

### 업캐스팅과 다운캐스팅
```csharp
Dog dog = new Dog { Name = "Buddy" };

// 업캐스팅 (암시적)
Animal animal = dog;

// 다운캐스팅 (명시적)
Dog sameDog = (Dog)animal;

// 안전한 캐스팅
if (animal is Dog d)
{
    Console.WriteLine($"{d.Name} is a dog");
}

// as 연산자
Dog anotherDog = animal as Dog;
if (anotherDog != null)
{
    anotherDog.MakeSound();
}
```

## 다형성

### 메서드 오버라이딩
```csharp
public abstract class Shape
{
    public abstract double CalculateArea();
    
    public virtual void Draw()
    {
        Console.WriteLine("Drawing shape");
    }
}

public class Circle : Shape
{
    public double Radius { get; set; }
    
    public override double CalculateArea()
    {
        return Math.PI * Radius * Radius;
    }
    
    public override void Draw()
    {
        base.Draw();
        Console.WriteLine($"Circle with radius {Radius}");
    }
}

public class Rectangle : Shape
{
    public double Width { get; set; }
    public double Height { get; set; }
    
    public override double CalculateArea()
    {
        return Width * Height;
    }
}
```

## 메서드 숨기기

### new 키워드
```csharp
public class BaseClass
{
    public void Method()
    {
        Console.WriteLine("Base method");
    }
}

public class DerivedClass : BaseClass
{
    public new void Method()  // 메서드 숨기기
    {
        Console.WriteLine("Derived method");
    }
}

// 사용
DerivedClass derived = new DerivedClass();
derived.Method();  // "Derived method"

BaseClass baseRef = derived;
baseRef.Method();  // "Base method"
```

## sealed 클래스와 메서드

```csharp
// sealed 클래스
public sealed class FinalClass
{
    // 이 클래스는 상속할 수 없음
}

// sealed 메서드
public class Parent
{
    public virtual void Method() { }
}

public class Child : Parent
{
    public sealed override void Method()
    {
        // 더 이상 오버라이드 불가
    }
}
```

## 읽기 전용 필드

```csharp
public class Constants
{
    // 컴파일 시간 상수
    public const double PI = 3.14159;
    
    // 런타임 상수
    public readonly DateTime CreatedAt;
    public readonly string Version;
    
    public Constants()
    {
        CreatedAt = DateTime.Now;
        Version = GetVersionFromFile();
    }
}
```

## 중첩 클래스

```csharp
public class OuterClass
{
    private int outerField = 10;
    
    public class NestedClass
    {
        public void AccessOuter(OuterClass outer)
        {
            Console.WriteLine(outer.outerField);  // private 접근 가능
        }
    }
    
    private class PrivateNested
    {
        // 외부에서 접근 불가
    }
}

// 사용
OuterClass.NestedClass nested = new OuterClass.NestedClass();
```

## 분할 클래스 (Partial Class)

```csharp
// File1.cs
public partial class Employee
{
    public string Name { get; set; }
    public int Id { get; set; }
}

// File2.cs
public partial class Employee
{
    public void Work()
    {
        Console.WriteLine($"{Name} is working");
    }
}
```

## 확장 메서드

```csharp
public static class StringExtensions
{
    public static int WordCount(this string str)
    {
        return str.Split(' ', StringSplitOptions.RemoveEmptyEntries).Length;
    }
    
    public static string Truncate(this string str, int maxLength)
    {
        if (str.Length <= maxLength)
            return str;
        return str.Substring(0, maxLength) + "...";
    }
}

// 사용
string text = "Hello world from C#";
int count = text.WordCount();  // 4
string truncated = text.Truncate(10);  // "Hello worl..."
```

## 구조체 (Struct)

```csharp
public struct Point
{
    public double X { get; set; }
    public double Y { get; set; }
    
    public Point(double x, double y)
    {
        X = x;
        Y = y;
    }
    
    public double DistanceFromOrigin()
    {
        return Math.Sqrt(X * X + Y * Y);
    }
}

// 구조체는 값 타입
Point p1 = new Point(3, 4);
Point p2 = p1;  // 값 복사
p2.X = 5;      // p1은 변경되지 않음
```

## 튜플 (Tuple)

```csharp
// 명명된 튜플
(string Name, int Age) person = ("John", 25);
Console.WriteLine($"{person.Name} is {person.Age} years old");

// 튜플 반환
(int Min, int Max) GetMinMax(int[] numbers)
{
    return (numbers.Min(), numbers.Max());
}

// 튜플 분해
var (min, max) = GetMinMax(new[] { 3, 1, 4, 1, 5 });

// 무시 패턴
var (name, _) = person;  // Age 무시
```

## 핵심 개념 정리
- **클래스**: 객체를 만들기 위한 설계도
- **객체**: 클래스의 인스턴스
- **상속**: 코드 재사용과 계층 구조
- **다형성**: 동일한 인터페이스로 다양한 구현
- **캡슐화**: 데이터 보호와 은닉