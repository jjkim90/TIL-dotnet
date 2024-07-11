# 인터페이스와 추상 클래스

## 인터페이스 (Interface)

### 인터페이스 선언
```csharp
public interface IDrawable
{
    void Draw();
}

public interface IResizable
{
    void Resize(double factor);
    double Width { get; set; }
    double Height { get; set; }
}

// C# 8.0+ 기본 구현
public interface ILogger
{
    void Log(string message);
    
    // 기본 구현 제공
    void LogError(string message)
    {
        Log($"ERROR: {message}");
    }
}
```

### 인터페이스 구현
```csharp
public class Circle : IDrawable, IResizable
{
    public double Radius { get; set; }
    
    // IDrawable 구현
    public void Draw()
    {
        Console.WriteLine($"Drawing a circle with radius {Radius}");
    }
    
    // IResizable 구현
    public double Width 
    { 
        get => Radius * 2;
        set => Radius = value / 2;
    }
    
    public double Height
    {
        get => Radius * 2;
        set => Radius = value / 2;
    }
    
    public void Resize(double factor)
    {
        Radius *= factor;
    }
}
```

## 인터페이스는 약속이다

### 다중 구현을 통한 다형성
```csharp
public interface IPlayable
{
    void Play();
    void Pause();
    void Stop();
}

public interface IRecordable
{
    void Record();
    void StopRecording();
}

public class MediaPlayer : IPlayable
{
    public void Play() => Console.WriteLine("Playing...");
    public void Pause() => Console.WriteLine("Paused");
    public void Stop() => Console.WriteLine("Stopped");
}

public class SmartPlayer : IPlayable, IRecordable
{
    public void Play() => Console.WriteLine("Playing...");
    public void Pause() => Console.WriteLine("Paused");
    public void Stop() => Console.WriteLine("Stopped");
    public void Record() => Console.WriteLine("Recording...");
    public void StopRecording() => Console.WriteLine("Recording stopped");
}
```

### 인터페이스를 통한 의존성 주입
```csharp
public interface IDatabase
{
    void Save(string data);
    string Load(int id);
}

public class SqlDatabase : IDatabase
{
    public void Save(string data)
    {
        Console.WriteLine($"Saving to SQL: {data}");
    }
    
    public string Load(int id)
    {
        return $"Data from SQL (ID: {id})";
    }
}

public class DataService
{
    private readonly IDatabase database;
    
    public DataService(IDatabase database)
    {
        this.database = database;
    }
    
    public void ProcessData(string data)
    {
        database.Save(data);
    }
}
```

## 인터페이스 상속

```csharp
public interface IAnimal
{
    string Name { get; set; }
    void Eat();
}

public interface IMammal : IAnimal
{
    int NumberOfLegs { get; set; }
    void Walk();
}

public interface IFlyable
{
    void Fly();
    double WingSpan { get; set; }
}

// 다중 인터페이스 상속
public interface IBat : IMammal, IFlyable
{
    void UseEcholocation();
}

public class Bat : IBat
{
    public string Name { get; set; }
    public int NumberOfLegs { get; set; } = 2;
    public double WingSpan { get; set; }
    
    public void Eat() => Console.WriteLine("Eating insects");
    public void Walk() => Console.WriteLine("Walking upside down");
    public void Fly() => Console.WriteLine("Flying at night");
    public void UseEcholocation() => Console.WriteLine("Using echolocation");
}
```

## 여러 인터페이스 구현

### 명시적 인터페이스 구현
```csharp
public interface IEnglishSpeaker
{
    void Speak();
}

public interface IKoreanSpeaker
{
    void Speak();
}

public class Translator : IEnglishSpeaker, IKoreanSpeaker
{
    // 암시적 구현 (기본)
    public void Speak()
    {
        Console.WriteLine("Speaking in default language");
    }
    
    // 명시적 구현
    void IEnglishSpeaker.Speak()
    {
        Console.WriteLine("Speaking in English");
    }
    
    void IKoreanSpeaker.Speak()
    {
        Console.WriteLine("한국어로 말하기");
    }
}

// 사용
Translator translator = new Translator();
translator.Speak();  // 기본 구현

IEnglishSpeaker english = translator;
english.Speak();  // Speaking in English

IKoreanSpeaker korean = translator;
korean.Speak();  // 한국어로 말하기
```

## 인터페이스의 기본 구현 메서드 (C# 8.0+)

```csharp
public interface IVersionable
{
    int Version { get; set; }
    
    // 기본 구현
    void IncrementVersion()
    {
        Version++;
    }
    
    // virtual 기본 구현
    virtual void ResetVersion()
    {
        Version = 1;
    }
}

public class Document : IVersionable
{
    public int Version { get; set; } = 1;
    
    // 기본 구현 사용 또는 재정의 가능
    public void ResetVersion()
    {
        Version = 0;  // 재정의
    }
}
```

### 정적 멤버 (C# 8.0+)
```csharp
public interface ICalculator
{
    // 정적 멤버
    static double PI = 3.14159;
    
    static double CircleArea(double radius)
    {
        return PI * radius * radius;
    }
    
    // 추상 정적 멤버 (C# 11+)
    static abstract int Calculate(int a, int b);
}

public class AddCalculator : ICalculator
{
    public static int Calculate(int a, int b) => a + b;
}
```

## 추상 클래스 (Abstract Class)

### 추상 클래스 선언
```csharp
public abstract class Shape
{
    // 일반 필드와 프로퍼티
    public string Name { get; set; }
    protected ConsoleColor color;
    
    // 생성자
    protected Shape(string name)
    {
        Name = name;
    }
    
    // 추상 메서드 (구현 없음)
    public abstract double CalculateArea();
    public abstract double CalculatePerimeter();
    
    // 일반 메서드 (구현 있음)
    public void Display()
    {
        Console.ForegroundColor = color;
        Console.WriteLine($"{Name}: Area = {CalculateArea():F2}");
        Console.ResetColor();
    }
    
    // virtual 메서드
    public virtual void Draw()
    {
        Console.WriteLine($"Drawing {Name}");
    }
}
```

### 추상 클래스 구현
```csharp
public class Rectangle : Shape
{
    public double Width { get; set; }
    public double Height { get; set; }
    
    public Rectangle(double width, double height) : base("Rectangle")
    {
        Width = width;
        Height = height;
        color = ConsoleColor.Blue;
    }
    
    public override double CalculateArea()
    {
        return Width * Height;
    }
    
    public override double CalculatePerimeter()
    {
        return 2 * (Width + Height);
    }
    
    public override void Draw()
    {
        base.Draw();
        Console.WriteLine($"Width: {Width}, Height: {Height}");
    }
}

public class Circle : Shape
{
    public double Radius { get; set; }
    
    public Circle(double radius) : base("Circle")
    {
        Radius = radius;
        color = ConsoleColor.Red;
    }
    
    public override double CalculateArea()
    {
        return Math.PI * Radius * Radius;
    }
    
    public override double CalculatePerimeter()
    {
        return 2 * Math.PI * Radius;
    }
}
```

## 인터페이스 vs 추상 클래스

### 언제 인터페이스를 사용할까?
```csharp
// 여러 타입이 공통 기능을 구현해야 할 때
public interface IComparable<T>
{
    int CompareTo(T other);
}

// 다중 상속이 필요할 때
public interface IDisposable
{
    void Dispose();
}

public interface ICloneable
{
    object Clone();
}

public class Resource : IDisposable, ICloneable
{
    public void Dispose() { /* 리소스 정리 */ }
    public object Clone() { /* 복제 로직 */ }
}
```

### 언제 추상 클래스를 사용할까?
```csharp
// 공통 구현을 공유해야 할 때
public abstract class Animal
{
    private string name;
    protected int age;
    
    protected Animal(string name)
    {
        this.name = name;
    }
    
    // 공통 구현
    public void Sleep()
    {
        Console.WriteLine($"{name} is sleeping");
    }
    
    // 추상 메서드
    public abstract void MakeSound();
}

public class Dog : Animal
{
    public Dog(string name) : base(name) { }
    
    public override void MakeSound()
    {
        Console.WriteLine("Woof!");
    }
}
```

## 실전 예제: 게임 캐릭터 시스템

```csharp
// 인터페이스들
public interface IAttackable
{
    void Attack(ICharacter target);
    int AttackPower { get; }
}

public interface IDefendable
{
    void Defend(int damage);
    int DefensePower { get; }
}

public interface IHealable
{
    void Heal(int amount);
    int MaxHealth { get; }
}

// 추상 클래스
public abstract class Character : ICharacter
{
    public string Name { get; protected set; }
    public int Health { get; protected set; }
    public bool IsAlive => Health > 0;
    
    protected Character(string name, int health)
    {
        Name = name;
        Health = health;
    }
    
    public abstract void PerformAction();
    
    public virtual void TakeDamage(int damage)
    {
        Health -= damage;
        if (Health < 0) Health = 0;
        Console.WriteLine($"{Name} took {damage} damage. Health: {Health}");
    }
}

// 구체 클래스들
public class Warrior : Character, IAttackable, IDefendable
{
    public int AttackPower { get; private set; }
    public int DefensePower { get; private set; }
    
    public Warrior(string name) : base(name, 100)
    {
        AttackPower = 15;
        DefensePower = 10;
    }
    
    public void Attack(ICharacter target)
    {
        Console.WriteLine($"{Name} attacks with sword!");
        if (target is Character character)
        {
            character.TakeDamage(AttackPower);
        }
    }
    
    public void Defend(int damage)
    {
        int reducedDamage = Math.Max(0, damage - DefensePower);
        TakeDamage(reducedDamage);
    }
    
    public override void PerformAction()
    {
        Console.WriteLine($"{Name} is ready for battle!");
    }
}

public class Healer : Character, IHealable, IAttackable
{
    public int MaxHealth { get; private set; }
    public int AttackPower { get; private set; }
    
    public Healer(string name) : base(name, 80)
    {
        MaxHealth = 80;
        AttackPower = 5;
    }
    
    public void Heal(int amount)
    {
        Health = Math.Min(Health + amount, MaxHealth);
        Console.WriteLine($"{Name} healed for {amount}. Health: {Health}");
    }
    
    public void Attack(ICharacter target)
    {
        Console.WriteLine($"{Name} casts holy bolt!");
        if (target is Character character)
        {
            character.TakeDamage(AttackPower);
        }
    }
    
    public override void PerformAction()
    {
        Heal(20);
    }
}
```

## 고급 패턴

### 인터페이스 분리 원칙 (ISP)
```csharp
// 나쁜 예 - 너무 큰 인터페이스
public interface IWorker
{
    void Work();
    void Eat();
    void Sleep();
}

// 좋은 예 - 분리된 인터페이스
public interface IWorkable
{
    void Work();
}

public interface IFeedable
{
    void Eat();
}

public interface ISleepable
{
    void Sleep();
}

public class Human : IWorkable, IFeedable, ISleepable
{
    public void Work() => Console.WriteLine("Working");
    public void Eat() => Console.WriteLine("Eating");
    public void Sleep() => Console.WriteLine("Sleeping");
}

public class Robot : IWorkable
{
    public void Work() => Console.WriteLine("Working 24/7");
    // 로봇은 먹거나 자지 않음
}
```

## 핵심 개념 정리
- **인터페이스**: 구현해야 할 멤버들의 계약
- **추상 클래스**: 공통 구현을 포함할 수 있는 불완전한 클래스
- **다중 상속**: 인터페이스만 가능
- **기본 구현**: C# 8.0부터 인터페이스도 기본 구현 제공
- **사용 시점**: 인터페이스는 "무엇을", 추상 클래스는 "어떻게"