# LINQ (Language Integrated Query)

## LINQ란?

LINQ는 C#에 통합된 쿼리 언어로, 다양한 데이터 소스를 일관된 방식으로 쿼리할 수 있게 해줍니다.

### LINQ의 장점
```csharp
// LINQ 이전 - 명령형 프로그래밍
List<int> evenNumbers = new List<int>();
foreach (int num in numbers)
{
    if (num % 2 == 0)
        evenNumbers.Add(num);
}

// LINQ 사용 - 선언형 프로그래밍
var evenNumbers = numbers.Where(n => n % 2 == 0).ToList();
```

## LINQ의 기본

### from 절
```csharp
// 쿼리 구문
var query = from num in numbers
            select num;

// 메서드 구문
var query2 = numbers.Select(num => num);

// 복잡한 예제
var students = new[]
{
    new { Name = "John", Age = 20, Grade = 85 },
    new { Name = "Jane", Age = 22, Grade = 92 },
    new { Name = "Bob", Age = 21, Grade = 78 }
};

var names = from student in students
            select student.Name;
```

### where 절
```csharp
// 조건 필터링
var adults = from person in people
             where person.Age >= 18
             select person;

// 메서드 구문
var adults2 = people.Where(p => p.Age >= 18);

// 복합 조건
var topStudents = from student in students
                  where student.Grade >= 80 && student.Age < 25
                  select student;

// 메서드 체이닝
var topStudents2 = students
    .Where(s => s.Grade >= 80)
    .Where(s => s.Age < 25);
```

### orderby 절
```csharp
// 오름차순 정렬
var sortedByAge = from person in people
                  orderby person.Age
                  select person;

// 내림차순 정렬
var sortedByGradeDesc = from student in students
                        orderby student.Grade descending
                        select student;

// 메서드 구문
var sorted = students.OrderBy(s => s.Name)
                    .ThenByDescending(s => s.Grade);

// 복합 정렬
var complexSort = from student in students
                  orderby student.Grade descending, student.Name
                  select student;
```

### select 절
```csharp
// 프로젝션 (투영)
var names = from student in students
            select student.Name;

// 새로운 형식으로 변환
var studentInfo = from student in students
                  select new
                  {
                      student.Name,
                      student.Grade,
                      Status = student.Grade >= 80 ? "Pass" : "Fail"
                  };

// 계산된 값
var squared = from num in numbers
              select num * num;

// 메서드 구문으로 복잡한 변환
var transformed = students.Select(s => new
{
    FullName = $"{s.FirstName} {s.LastName}",
    GradeLevel = s.Grade switch
    {
        >= 90 => "A",
        >= 80 => "B",
        >= 70 => "C",
        >= 60 => "D",
        _ => "F"
    }
});
```

## 여러 개의 데이터 원본에 질의하기

### 복합 from 절 (SelectMany)
```csharp
// 중첩된 컬렉션
var classrooms = new[]
{
    new { Teacher = "Mr. Smith", Students = new[] { "John", "Jane", "Bob" } },
    new { Teacher = "Ms. Johnson", Students = new[] { "Alice", "David" } }
};

// 모든 학생 추출
var allStudents = from classroom in classrooms
                  from student in classroom.Students
                  select new { classroom.Teacher, Student = student };

// 메서드 구문
var allStudents2 = classrooms.SelectMany(
    c => c.Students,
    (c, s) => new { c.Teacher, Student = s }
);
```

### let 절
```csharp
// 중간 계산 결과 저장
var query = from student in students
            let average = students.Average(s => s.Grade)
            where student.Grade > average
            select new
            {
                student.Name,
                student.Grade,
                AboveAverage = student.Grade - average
            };
```

## group by로 데이터 분류하기

### 기본 그룹화
```csharp
// 나이별 그룹화
var ageGroups = from person in people
                group person by person.Age into ageGroup
                select new
                {
                    Age = ageGroup.Key,
                    Count = ageGroup.Count(),
                    People = ageGroup.ToList()
                };

// 메서드 구문
var ageGroups2 = people.GroupBy(p => p.Age)
                       .Select(g => new
                       {
                           Age = g.Key,
                           Count = g.Count(),
                           People = g.ToList()
                       });
```

### 복합 키로 그룹화
```csharp
var groups = from student in students
             group student by new { student.Grade / 10, student.Year } into g
             select new
             {
                 GradeRange = $"{g.Key.Item1 * 10}-{g.Key.Item1 * 10 + 9}",
                 Year = g.Key.Year,
                 Students = g.ToList()
             };
```

## join으로 데이터 연결하기

### 내부 조인 (Inner Join)
```csharp
var students = new[]
{
    new { Id = 1, Name = "John", DeptId = 1 },
    new { Id = 2, Name = "Jane", DeptId = 2 },
    new { Id = 3, Name = "Bob", DeptId = 1 }
};

var departments = new[]
{
    new { Id = 1, Name = "Computer Science" },
    new { Id = 2, Name = "Mathematics" }
};

// 내부 조인
var studentDept = from student in students
                  join dept in departments on student.DeptId equals dept.Id
                  select new
                  {
                      StudentName = student.Name,
                      DepartmentName = dept.Name
                  };

// 메서드 구문
var studentDept2 = students.Join(
    departments,
    student => student.DeptId,
    dept => dept.Id,
    (student, dept) => new
    {
        StudentName = student.Name,
        DepartmentName = dept.Name
    }
);
```

### 외부 조인 (Left Outer Join)
```csharp
// 왼쪽 외부 조인
var leftJoin = from student in students
               join dept in departments on student.DeptId equals dept.Id into deptGroup
               from dept in deptGroup.DefaultIfEmpty()
               select new
               {
                   StudentName = student.Name,
                   DepartmentName = dept?.Name ?? "No Department"
               };

// 그룹 조인
var groupJoin = from dept in departments
                join student in students on dept.Id equals student.DeptId into studentGroup
                select new
                {
                    Department = dept.Name,
                    Students = studentGroup.ToList()
                };
```

## LINQ 표준 연산자

### 필터링 연산자
```csharp
// Where - 조건 필터링
var adults = people.Where(p => p.Age >= 18);

// OfType - 특정 타입만 필터링
var strings = mixedList.OfType<string>();

// Distinct - 중복 제거
var uniqueNumbers = numbers.Distinct();

// Take/Skip - 요소 선택
var firstFive = numbers.Take(5);
var afterFive = numbers.Skip(5);
var page2 = numbers.Skip(10).Take(10);  // 페이징

// TakeWhile/SkipWhile - 조건부 선택
var takeWhileLess = numbers.TakeWhile(n => n < 50);
var skipWhileLess = numbers.SkipWhile(n => n < 50);
```

### 집계 연산자
```csharp
var numbers = new[] { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };

// 기본 집계
int count = numbers.Count();
int sum = numbers.Sum();
double average = numbers.Average();
int min = numbers.Min();
int max = numbers.Max();

// 조건부 집계
int evenCount = numbers.Count(n => n % 2 == 0);
int sumOfSquares = numbers.Sum(n => n * n);

// Aggregate - 사용자 정의 집계
string csv = numbers.Aggregate("", (acc, n) => acc + n + ",").TrimEnd(',');
int product = numbers.Aggregate(1, (acc, n) => acc * n);
```

### 집합 연산자
```csharp
var set1 = new[] { 1, 2, 3, 4, 5 };
var set2 = new[] { 4, 5, 6, 7, 8 };

// 합집합
var union = set1.Union(set2);  // 1, 2, 3, 4, 5, 6, 7, 8

// 교집합
var intersect = set1.Intersect(set2);  // 4, 5

// 차집합
var except = set1.Except(set2);  // 1, 2, 3

// 연결
var concat = set1.Concat(set2);  // 1, 2, 3, 4, 5, 4, 5, 6, 7, 8
```

### 변환 연산자
```csharp
// ToList, ToArray, ToDictionary
List<int> list = numbers.Where(n => n > 5).ToList();
int[] array = numbers.Where(n => n > 5).ToArray();
Dictionary<int, string> dict = students.ToDictionary(s => s.Id, s => s.Name);

// ToLookup - 하나의 키에 여러 값
var lookup = students.ToLookup(s => s.Grade / 10);

// Cast - 형변환
var objects = new object[] { 1, 2, 3 };
var integers = objects.Cast<int>();
```

### 요소 연산자
```csharp
// First/FirstOrDefault
var first = numbers.First();  // 예외 가능
var firstOrDefault = numbers.FirstOrDefault();  // 기본값 반환

// Last/LastOrDefault
var last = numbers.Last(n => n > 5);
var lastOrDefault = numbers.LastOrDefault(n => n > 100);  // 0

// Single/SingleOrDefault - 정확히 하나
var single = numbers.Single(n => n == 5);  // 5
// var error = numbers.Single(n => n > 5);  // 예외!

// ElementAt/ElementAtOrDefault
var fifth = numbers.ElementAt(4);  // 5
var outOfRange = numbers.ElementAtOrDefault(100);  // 0
```

## LINQ to Objects 실전 예제

### 복잡한 데이터 처리
```csharp
public class Order
{
    public int Id { get; set; }
    public DateTime Date { get; set; }
    public string Customer { get; set; }
    public List<OrderItem> Items { get; set; }
}

public class OrderItem
{
    public string Product { get; set; }
    public decimal Price { get; set; }
    public int Quantity { get; set; }
}

// 월별 매출 분석
var monthlySales = orders
    .GroupBy(o => new { o.Date.Year, o.Date.Month })
    .Select(g => new
    {
        Year = g.Key.Year,
        Month = g.Key.Month,
        TotalSales = g.Sum(o => o.Items.Sum(i => i.Price * i.Quantity)),
        OrderCount = g.Count(),
        TopProduct = g.SelectMany(o => o.Items)
                      .GroupBy(i => i.Product)
                      .OrderByDescending(p => p.Sum(i => i.Quantity))
                      .First().Key
    })
    .OrderBy(m => m.Year)
    .ThenBy(m => m.Month);
```

### 성능 최적화
```csharp
// 지연 실행 활용
var query = numbers.Where(n => n > 0)
                   .Select(n => ExpensiveOperation(n));

// 필요할 때만 실행
foreach (var item in query.Take(5))  // 5개만 처리
{
    Console.WriteLine(item);
}

// 즉시 실행이 필요한 경우
var results = query.ToList();  // 전체 실행

// Any vs Count
bool hasItems = list.Any();  // 효율적
bool hasItems2 = list.Count() > 0;  // 비효율적

// 병렬 LINQ (PLINQ)
var parallelResults = numbers.AsParallel()
                            .Where(n => IsPrime(n))
                            .Select(n => n * n)
                            .ToList();
```

## 쿼리 구문 vs 메서드 구문

```csharp
// 쿼리 구문
var query1 = from student in students
             where student.Grade >= 80
             orderby student.Name
             select student;

// 메서드 구문
var query2 = students.Where(s => s.Grade >= 80)
                    .OrderBy(s => s.Name);

// 혼합 사용
var query3 = (from student in students
              where student.Grade >= 80
              select student).Take(10);

// 일부 연산자는 메서드 구문만 지원
var distinctGrades = students.Select(s => s.Grade).Distinct();
var averageGrade = students.Average(s => s.Grade);
```

## 핵심 개념 정리
- **LINQ**: 통합 쿼리 언어
- **지연 실행**: 필요할 때까지 실행 연기
- **쿼리 구문**: SQL과 유사한 구문
- **메서드 구문**: 확장 메서드 체이닝
- **표준 쿼리 연산자**: Where, Select, GroupBy 등