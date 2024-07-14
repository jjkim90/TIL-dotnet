# 프로퍼티 (Properties)

## public 필드의 문제점

### 캡슐화 위반
```csharp
// 나쁜 예 - public 필드
public class Person
{
    public string name;  // 직접 접근 가능
    public int age;      // 유효성 검사 불가
}

// 문제점
Person person = new Person();
person.age = -10;  // 잘못된 값 설정 가능
person.name = "";  // 빈 문자열 허용
```

## 메서드보다 프로퍼티

### 프로퍼티 기본 구조
```csharp
public class Person
{
    private string name;
    private int age;
    
    // 프로퍼티
    public string Name
    {
        get { return name; }
        set 
        { 
            if (string.IsNullOrWhiteSpace(value))
                throw new ArgumentException("Name cannot be empty");
            name = value; 
        }
    }
    
    public int Age
    {
        get { return age; }
        set 
        { 
            if (value < 0 || value > 150)
                throw new ArgumentOutOfRangeException("Age must be between 0 and 150");
            age = value; 
        }
    }
}
```

### 프로퍼티 vs 메서드
```csharp
public class BankAccount
{
    private decimal balance;
    
    // 프로퍼티 - 상태를 나타냄
    public decimal Balance
    {
        get { return balance; }
        private set { balance = value; }
    }
    
    // 메서드 - 동작을 나타냄
    public void Deposit(decimal amount)
    {
        if (amount <= 0)
            throw new ArgumentException("Amount must be positive");
        balance += amount;
    }
}
```

## 자동 구현 프로퍼티

### 기본 사용법
```csharp
public class Product
{
    // 자동 구현 프로퍼티
    public string Name { get; set; }
    public decimal Price { get; set; }
    public int Stock { get; private set; }  // setter는 private
    
    // 읽기 전용 (생성자에서만 설정 가능)
    public string Id { get; }
    
    public Product(string id)
    {
        Id = id;  // 생성자에서 설정
    }
    
    public void UpdateStock(int quantity)
    {
        Stock = quantity;  // 클래스 내부에서 설정
    }
}
```

### 프로퍼티 초기화
```csharp
public class Configuration
{
    // 프로퍼티 초기화
    public string ServerName { get; set; } = "localhost";
    public int Port { get; set; } = 8080;
    public bool IsEnabled { get; set; } = true;
    
    // 컬렉션 초기화
    public List<string> AllowedHosts { get; set; } = new List<string>();
}
```

## 프로퍼티와 생성자

### 생성자에서 프로퍼티 설정
```csharp
public class Employee
{
    public string Name { get; set; }
    public string Department { get; set; }
    public decimal Salary { get; private set; }
    
    // 기본 생성자
    public Employee()
    {
        Department = "General";
    }
    
    // 매개변수가 있는 생성자
    public Employee(string name, decimal salary)
    {
        Name = name;
        Salary = salary;
        Department = "General";
    }
    
    // 모든 프로퍼티를 설정하는 생성자
    public Employee(string name, string department, decimal salary)
    {
        Name = name;
        Department = department;
        Salary = salary;
    }
}
```

### 객체 초기화자
```csharp
// 객체 초기화자 사용
Employee emp1 = new Employee
{
    Name = "John Doe",
    Department = "IT"
    // Salary는 private set이므로 설정 불가
};

// 생성자와 초기화자 함께 사용
Employee emp2 = new Employee("Jane Doe", 50000)
{
    Department = "HR"  // 생성자 후 추가 설정
};
```

## 초기화 전용 자동 구현 프로퍼티 (C# 9.0+)

### init 접근자
```csharp
public class Person
{
    public string FirstName { get; init; }
    public string LastName { get; init; }
    public DateTime BirthDate { get; init; }
    
    // 계산된 프로퍼티
    public int Age => DateTime.Now.Year - BirthDate.Year;
}

// 사용
var person = new Person
{
    FirstName = "John",
    LastName = "Doe",
    BirthDate = new DateTime(1990, 1, 1)
};

// person.FirstName = "Jane";  // 오류! init 후에는 변경 불가
```

## required 한정자 (C# 11+)

```csharp
public class User
{
    public required string Username { get; set; }
    public required string Email { get; set; }
    public string? PhoneNumber { get; set; }  // 선택적
    
    // required 프로퍼티가 있어도 기본 생성자 필요
    public User() { }
}

// 사용
// var user1 = new User();  // 오류! required 프로퍼티 설정 필요

var user2 = new User
{
    Username = "johndoe",  // 필수
    Email = "john@example.com"  // 필수
};
```

## 레코드 타입 (C# 9.0+)

### 불변 레코드
```csharp
// 위치 레코드
public record Point(double X, double Y);

// 프로퍼티가 있는 레코드
public record Person
{
    public string FirstName { get; init; }
    public string LastName { get; init; }
    public int Age { get; init; }
}

// 사용
var point = new Point(3.0, 4.0);
var person = new Person 
{ 
    FirstName = "John", 
    LastName = "Doe", 
    Age = 30 
};
```

### with 표현식
```csharp
var original = new Person 
{ 
    FirstName = "John", 
    LastName = "Doe", 
    Age = 30 
};

// with를 사용한 복사와 수정
var modified = original with { Age = 31 };

Console.WriteLine(original.Age);   // 30
Console.WriteLine(modified.Age);   // 31
```

### 레코드 비교
```csharp
var person1 = new Person { FirstName = "John", LastName = "Doe", Age = 30 };
var person2 = new Person { FirstName = "John", LastName = "Doe", Age = 30 };
var person3 = new Person { FirstName = "Jane", LastName = "Doe", Age = 30 };

Console.WriteLine(person1 == person2);  // True (값 비교)
Console.WriteLine(person1 == person3);  // False

// 일반 클래스는 참조 비교
public class PersonClass
{
    public string Name { get; set; }
}

var p1 = new PersonClass { Name = "John" };
var p2 = new PersonClass { Name = "John" };
Console.WriteLine(p1 == p2);  // False (다른 인스턴스)
```

## 무명 타입

### 익명 타입 생성
```csharp
// 무명 타입
var person = new
{
    Name = "John Doe",
    Age = 30,
    Address = new
    {
        Street = "123 Main St",
        City = "Seoul"
    }
};

Console.WriteLine($"{person.Name}, {person.Age}");
Console.WriteLine($"{person.Address.City}");

// person.Age = 31;  // 오류! 무명 타입은 읽기 전용
```

### LINQ에서 활용
```csharp
var employees = new[]
{
    new { Name = "John", Department = "IT", Salary = 50000 },
    new { Name = "Jane", Department = "HR", Salary = 55000 },
    new { Name = "Bob", Department = "IT", Salary = 60000 }
};

var summary = employees
    .GroupBy(e => e.Department)
    .Select(g => new
    {
        Department = g.Key,
        Count = g.Count(),
        AverageSalary = g.Average(e => e.Salary)
    });
```

## 인터페이스의 프로퍼티

```csharp
public interface IShape
{
    // 인터페이스 프로퍼티
    double Area { get; }
    double Perimeter { get; }
    string Name { get; set; }
}

public class Rectangle : IShape
{
    public double Width { get; set; }
    public double Height { get; set; }
    
    // 인터페이스 구현
    public double Area => Width * Height;
    public double Perimeter => 2 * (Width + Height);
    public string Name { get; set; } = "Rectangle";
}

// 명시적 구현
public class Square : IShape
{
    public double Side { get; set; }
    
    double IShape.Area => Side * Side;
    double IShape.Perimeter => 4 * Side;
    string IShape.Name { get; set; } = "Square";
}
```

## 추상 클래스의 프로퍼티

```csharp
public abstract class Vehicle
{
    // 일반 프로퍼티
    public string Brand { get; set; }
    public int Year { get; set; }
    
    // 추상 프로퍼티
    public abstract double MaxSpeed { get; }
    public abstract int NumberOfWheels { get; }
    
    // virtual 프로퍼티
    public virtual string Description => $"{Year} {Brand}";
}

public class Car : Vehicle
{
    public override double MaxSpeed => 200;
    public override int NumberOfWheels => 4;
    
    public override string Description => $"{base.Description} Car";
}

public class Motorcycle : Vehicle
{
    public override double MaxSpeed => 180;
    public override int NumberOfWheels => 2;
}
```

## 고급 프로퍼티 패턴

### 계산된 프로퍼티
```csharp
public class Circle
{
    private double radius;
    
    public double Radius
    {
        get => radius;
        set
        {
            if (value <= 0)
                throw new ArgumentException("Radius must be positive");
            radius = value;
        }
    }
    
    // 계산된 프로퍼티
    public double Area => Math.PI * radius * radius;
    public double Circumference => 2 * Math.PI * radius;
}
```

### 지연 로딩 프로퍼티
```csharp
public class DataService
{
    private List<string> _data;
    
    public List<string> Data
    {
        get
        {
            if (_data == null)
            {
                _data = LoadData();  // 첫 접근 시 로드
            }
            return _data;
        }
    }
    
    private List<string> LoadData()
    {
        // 데이터베이스나 파일에서 데이터 로드
        return new List<string> { "Item1", "Item2", "Item3" };
    }
}
```

### 프로퍼티 변경 알림
```csharp
public class ObservableProperty : INotifyPropertyChanged
{
    private string _name;
    
    public string Name
    {
        get => _name;
        set
        {
            if (_name != value)
            {
                _name = value;
                OnPropertyChanged(nameof(Name));
            }
        }
    }
    
    public event PropertyChangedEventHandler PropertyChanged;
    
    protected virtual void OnPropertyChanged(string propertyName)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }
}
```

## 인덱서 (미리보기)

```csharp
public class StringCollection
{
    private string[] items = new string[10];
    
    // 인덱서 프로퍼티
    public string this[int index]
    {
        get
        {
            if (index < 0 || index >= items.Length)
                throw new IndexOutOfRangeException();
            return items[index];
        }
        set
        {
            if (index < 0 || index >= items.Length)
                throw new IndexOutOfRangeException();
            items[index] = value;
        }
    }
}

// 사용
var collection = new StringCollection();
collection[0] = "Hello";
collection[1] = "World";
Console.WriteLine(collection[0]);  // "Hello"
```

## 표현식 본문 프로퍼티

```csharp
public class Temperature
{
    private double celsius;
    
    // 표현식 본문 프로퍼티
    public double Celsius
    {
        get => celsius;
        set => celsius = value;
    }
    
    // 읽기 전용 표현식 본문
    public double Fahrenheit => celsius * 9 / 5 + 32;
    public double Kelvin => celsius + 273.15;
}
```

## 핵심 개념 정리
- **프로퍼티**: 필드처럼 보이지만 메서드처럼 동작
- **자동 구현**: 간단한 프로퍼티를 쉽게 작성
- **init**: 초기화 후 불변성 보장
- **required**: 필수 프로퍼티 지정
- **레코드**: 불변 데이터를 위한 특별한 타입