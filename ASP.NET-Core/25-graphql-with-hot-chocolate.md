# GraphQL과 Hot Chocolate

## GraphQL 소개

GraphQL은 Facebook에서 개발한 API를 위한 쿼리 언어이자 런타임입니다. 클라이언트가 필요한 데이터를 정확히 요청할 수 있게 해주며, API의 진화를 쉽게 만들어줍니다.

### GraphQL의 핵심 개념

- **스키마 정의 언어(SDL)**: API의 타입 시스템을 정의
- **쿼리(Query)**: 데이터를 읽기 위한 작업
- **뮤테이션(Mutation)**: 데이터를 변경하기 위한 작업  
- **구독(Subscription)**: 실시간 데이터 업데이트를 위한 작업
- **리졸버(Resolver)**: 각 필드의 데이터를 가져오는 함수

### REST API와의 차이점

| REST API | GraphQL |
|----------|---------|
| 여러 엔드포인트 | 단일 엔드포인트 |
| 고정된 데이터 구조 | 유연한 데이터 구조 |
| Over/Under-fetching 문제 | 필요한 데이터만 정확히 요청 |
| 버저닝 필요 | 스키마 진화로 해결 |

## Hot Chocolate 소개

Hot Chocolate는 .NET을 위한 오픈소스 GraphQL 서버 구현체입니다. ChilliCream에서 개발했으며, GraphQL 사양을 완벽하게 준수합니다.

### 주요 특징

- **고성능**: 최적화된 실행 엔진
- **Code-First & Schema-First**: 두 가지 접근 방식 모두 지원
- **필터링 & 정렬**: 내장된 필터링과 정렬 기능
- **실시간 구독**: WebSocket을 통한 실시간 데이터
- **DataLoader 패턴**: N+1 문제 해결
- **인증 & 권한**: ASP.NET Core 보안 통합

## Hot Chocolate 설치 및 설정

### NuGet 패키지 설치

```bash
# 기본 패키지
dotnet add package HotChocolate.AspNetCore

# 추가 기능 패키지
dotnet add package HotChocolate.AspNetCore.Authorization
dotnet add package HotChocolate.Data.EntityFramework
```

### 기본 설정

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// GraphQL 서비스 등록
builder.Services
    .AddGraphQLServer()
    .AddQueryType<Query>()
    .AddMutationType<Mutation>()
    .AddSubscriptionType<Subscription>();

var app = builder.Build();

// GraphQL 엔드포인트 매핑
app.MapGraphQL();

// GraphQL IDE (Banana Cake Pop) 활성화
app.MapBananaCakePop("/graphql-ui");

app.Run();
```

## 스키마 정의

### Code-First 접근 방식

```csharp
// Models/Book.cs
public class Book
{
    public int Id { get; set; }
    public string Title { get; set; }
    public string Author { get; set; }
    public decimal Price { get; set; }
    public DateTime PublishedDate { get; set; }
}

// Models/Author.cs
public class Author
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Biography { get; set; }
    public ICollection<Book> Books { get; set; }
}
```

### 타입 설정

```csharp
// Types/BookType.cs
public class BookType : ObjectType<Book>
{
    protected override void Configure(IObjectTypeDescriptor<Book> descriptor)
    {
        descriptor.Description("도서 정보를 나타내는 타입");
        
        descriptor
            .Field(f => f.Title)
            .Description("도서 제목");
            
        descriptor
            .Field(f => f.Price)
            .Description("도서 가격 (원화)");
            
        // 계산된 필드 추가
        descriptor
            .Field("discountedPrice")
            .Description("할인가격")
            .Type<NonNullType<DecimalType>>()
            .Resolve(context =>
            {
                var book = context.Parent<Book>();
                return book.Price * 0.9m; // 10% 할인
            });
    }
}
```

## Query 구현

### 기본 쿼리

```csharp
// Queries/Query.cs
public class Query
{
    // 단일 도서 조회
    public Book GetBook(int id, [Service] IBookRepository repository)
        => repository.GetById(id);
    
    // 모든 도서 조회
    public IQueryable<Book> GetBooks([Service] IBookRepository repository)
        => repository.GetAll();
    
    // 검색 기능
    public async Task<IEnumerable<Book>> SearchBooks(
        string keyword,
        [Service] IBookService bookService)
    {
        return await bookService.SearchAsync(keyword);
    }
}
```

### 필터링과 정렬

```csharp
public class Query
{
    // 필터링 활성화
    [UseFiltering]
    [UseSorting]
    public IQueryable<Book> GetBooks([Service] ApplicationDbContext context)
    {
        return context.Books;
    }
    
    // 페이징 활성화
    [UsePaging]
    public IQueryable<Author> GetAuthors([Service] ApplicationDbContext context)
    {
        return context.Authors;
    }
}

// 사용 예시 (GraphQL 쿼리)
// query {
//   books(
//     where: { price: { gte: 10000 } }
//     order: { title: ASC }
//   ) {
//     title
//     price
//   }
// }
```

### 커스텀 필터

```csharp
// Filters/BookFilterType.cs
public class BookFilterType : FilterInputType<Book>
{
    protected override void Configure(
        IFilterInputTypeDescriptor<Book> descriptor)
    {
        descriptor.BindFieldsImplicitly();
        
        // 가격 범위 필터
        descriptor.Field(f => f.Price)
            .Name("priceRange");
            
        // 출판일 필터
        descriptor.Field(f => f.PublishedDate)
            .Name("publishedAfter");
    }
}
```

## Mutation 구현

### 기본 뮤테이션

```csharp
// Mutations/Mutation.cs
public class Mutation
{
    // 도서 생성
    public async Task<Book> CreateBook(
        CreateBookInput input,
        [Service] IBookRepository repository)
    {
        var book = new Book
        {
            Title = input.Title,
            Author = input.Author,
            Price = input.Price,
            PublishedDate = DateTime.Now
        };
        
        return await repository.AddAsync(book);
    }
    
    // 도서 수정
    public async Task<Book> UpdateBook(
        int id,
        UpdateBookInput input,
        [Service] IBookRepository repository)
    {
        var book = await repository.GetByIdAsync(id);
        if (book == null)
            throw new GraphQLException("도서를 찾을 수 없습니다.");
            
        book.Title = input.Title ?? book.Title;
        book.Price = input.Price ?? book.Price;
        
        return await repository.UpdateAsync(book);
    }
    
    // 도서 삭제
    public async Task<bool> DeleteBook(
        int id,
        [Service] IBookRepository repository)
    {
        return await repository.DeleteAsync(id);
    }
}
```

### 입력 타입 정의

```csharp
// Inputs/CreateBookInput.cs
public class CreateBookInput
{
    [Required]
    [StringLength(200)]
    public string Title { get; set; }
    
    [Required]
    public string Author { get; set; }
    
    [Range(0, 1000000)]
    public decimal Price { get; set; }
}

// Inputs/UpdateBookInput.cs  
public class UpdateBookInput
{
    [StringLength(200)]
    public string? Title { get; set; }
    
    [Range(0, 1000000)]
    public decimal? Price { get; set; }
}
```

### 에러 처리

```csharp
public class Mutation
{
    public async Task<CreateBookPayload> CreateBook(
        CreateBookInput input,
        [Service] IBookRepository repository)
    {
        try
        {
            // 중복 검사
            if (await repository.ExistsByTitleAsync(input.Title))
            {
                return new CreateBookPayload(
                    null,
                    new[] { new UserError("이미 존재하는 도서입니다.", "DUPLICATE_BOOK") }
                );
            }
            
            var book = await repository.AddAsync(new Book { /* ... */ });
            return new CreateBookPayload(book, null);
        }
        catch (Exception ex)
        {
            return new CreateBookPayload(
                null,
                new[] { new UserError("도서 생성 중 오류가 발생했습니다.", "CREATE_ERROR") }
            );
        }
    }
}

// Payloads/CreateBookPayload.cs
public class CreateBookPayload
{
    public Book? Book { get; }
    public IReadOnlyList<UserError>? Errors { get; }
    
    public CreateBookPayload(Book? book, IReadOnlyList<UserError>? errors)
    {
        Book = book;
        Errors = errors;
    }
}
```

## Subscription 구현

### 실시간 업데이트

```csharp
// Subscriptions/Subscription.cs
public class Subscription
{
    // 도서 생성 구독
    [Subscribe]
    public Book OnBookCreated([EventMessage] Book book) => book;
    
    // 특정 저자의 도서 업데이트 구독
    [Subscribe]
    [Topic("{authorName}")]
    public Book OnBookUpdatedByAuthor(
        string authorName,
        [EventMessage] Book book) => book;
}

// Mutation에서 이벤트 발행
public class Mutation
{
    public async Task<Book> CreateBook(
        CreateBookInput input,
        [Service] IBookRepository repository,
        [Service] ITopicEventSender eventSender)
    {
        var book = await repository.AddAsync(new Book { /* ... */ });
        
        // 구독자에게 이벤트 전송
        await eventSender.SendAsync(
            nameof(Subscription.OnBookCreated),
            book);
            
        await eventSender.SendAsync(
            book.Author,
            book);
            
        return book;
    }
}
```

### WebSocket 설정

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddGraphQLServer()
    .AddInMemorySubscriptions(); // In-Memory 구독 프로바이더

var app = builder.Build();

app.UseWebSockets(); // WebSocket 미들웨어 추가
app.MapGraphQL();

app.Run();
```

## DataLoader 패턴

### N+1 문제 해결

```csharp
// DataLoaders/AuthorDataLoader.cs
public class AuthorDataLoader : BatchDataLoader<int, Author>
{
    private readonly IDbContextFactory<BookDbContext> _dbContextFactory;
    
    public AuthorDataLoader(
        IDbContextFactory<BookDbContext> dbContextFactory,
        IBatchScheduler batchScheduler,
        DataLoaderOptions? options = null)
        : base(batchScheduler, options)
    {
        _dbContextFactory = dbContextFactory;
    }
    
    protected override async Task<IReadOnlyDictionary<int, Author>> LoadBatchAsync(
        IReadOnlyList<int> keys,
        CancellationToken cancellationToken)
    {
        await using var dbContext = await _dbContextFactory.CreateDbContextAsync();
        
        return await dbContext.Authors
            .Where(a => keys.Contains(a.Id))
            .ToDictionaryAsync(a => a.Id, cancellationToken);
    }
}

// 사용 예시
public class BookType : ObjectType<Book>
{
    protected override void Configure(IObjectTypeDescriptor<Book> descriptor)
    {
        descriptor
            .Field("author")
            .ResolveWith<BookResolvers>(r => r.GetAuthorAsync(default!, default!));
    }
}

public class BookResolvers
{
    public async Task<Author?> GetAuthorAsync(
        [Parent] Book book,
        AuthorDataLoader authorLoader)
    {
        return await authorLoader.LoadAsync(book.AuthorId);
    }
}
```

### GroupDataLoader

```csharp
// DataLoaders/BooksByAuthorDataLoader.cs
public class BooksByAuthorDataLoader : GroupedDataLoader<int, Book>
{
    private readonly IDbContextFactory<BookDbContext> _dbContextFactory;
    
    public BooksByAuthorDataLoader(
        IDbContextFactory<BookDbContext> dbContextFactory,
        IBatchScheduler batchScheduler,
        DataLoaderOptions? options = null)
        : base(batchScheduler, options)
    {
        _dbContextFactory = dbContextFactory;
    }
    
    protected override async Task<ILookup<int, Book>> LoadGroupedBatchAsync(
        IReadOnlyList<int> keys,
        CancellationToken cancellationToken)
    {
        await using var dbContext = await _dbContextFactory.CreateDbContextAsync();
        
        var books = await dbContext.Books
            .Where(b => keys.Contains(b.AuthorId))
            .ToListAsync(cancellationToken);
            
        return books.ToLookup(b => b.AuthorId);
    }
}
```

## 인증과 권한

### JWT 인증 설정

```csharp
// Program.cs
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]))
        };
    });

builder.Services
    .AddGraphQLServer()
    .AddAuthorization();
```

### 권한 부여

```csharp
public class Query
{
    // 인증된 사용자만 접근 가능
    [Authorize]
    public async Task<User> GetMyProfile(
        [Service] IUserService userService,
        ClaimsPrincipal claimsPrincipal)
    {
        var userId = claimsPrincipal.FindFirstValue(ClaimTypes.NameIdentifier);
        return await userService.GetUserAsync(userId);
    }
    
    // 특정 역할 필요
    [Authorize(Roles = "Admin")]
    public async Task<IEnumerable<User>> GetAllUsers(
        [Service] IUserService userService)
    {
        return await userService.GetAllUsersAsync();
    }
    
    // 정책 기반 권한
    [Authorize(Policy = "MinimumAge")]
    public async Task<IEnumerable<Book>> GetAdultBooks(
        [Service] IBookRepository repository)
    {
        return await repository.GetAdultBooksAsync();
    }
}
```

### 필드 레벨 권한

```csharp
public class UserType : ObjectType<User>
{
    protected override void Configure(IObjectTypeDescriptor<User> descriptor)
    {
        descriptor
            .Field(f => f.Email)
            .Authorize(); // 인증된 사용자만
            
        descriptor
            .Field(f => f.PhoneNumber)
            .Authorize(Roles = new[] { "Admin", "Self" }); // 관리자 또는 본인
            
        descriptor
            .Field(f => f.SocialSecurityNumber)
            .Authorize(Policy = "HighSecurity"); // 특별 정책
    }
}
```

## 성능 최적화

### 프로젝션 사용

```csharp
public class Query
{
    // 자동 프로젝션
    [UseProjection]
    public IQueryable<Book> GetBooks([Service] BookDbContext context)
    {
        return context.Books;
    }
}

// 생성되는 SQL은 요청된 필드만 SELECT
// query { books { title } }
// => SELECT b.Title FROM Books b
```

### 쿼리 복잡도 제한

```csharp
// Program.cs
builder.Services
    .AddGraphQLServer()
    .AddMaxComplexityRule(1000) // 최대 복잡도
    .AddMaxExecutionDepthRule(15) // 최대 깊이
    .ModifyRequestOptions(opt =>
    {
        opt.IncludeExceptionDetails = false;
        opt.ExecutionTimeout = TimeSpan.FromSeconds(30);
    });
```

### 지속 쿼리

```csharp
// Program.cs
builder.Services
    .AddGraphQLServer()
    .AddPersistedQueryPipeline()
    .UsePersistedQueryPipeline()
    .AddReadOnlyFileSystemQueryStorage("./persisted-queries");

// 클라이언트는 쿼리 해시만 전송
// {
//   "extensions": {
//     "persistedQuery": {
//       "sha256Hash": "ecf4edb46db40b5132295c0291d62fb65d6759a9eedfa4d5d612dd5ec54a6b38"
//     }
//   }
// }
```

## 테스트

### 통합 테스트

```csharp
public class GraphQLTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;
    
    public GraphQLTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory;
    }
    
    [Fact]
    public async Task GetBooks_Should_Return_Books()
    {
        // Arrange
        var client = _factory.CreateClient();
        var query = @"
            query {
                books {
                    id
                    title
                    author
                }
            }";
        
        var request = new
        {
            query = query
        };
        
        // Act
        var response = await client.PostAsJsonAsync("/graphql", request);
        var result = await response.Content.ReadFromJsonAsync<GraphQLResult>();
        
        // Assert
        Assert.NotNull(result.Data);
        Assert.Null(result.Errors);
        var books = result.Data.GetProperty("books");
        Assert.True(books.GetArrayLength() > 0);
    }
}
```

### 스키마 테스트

```csharp
[Fact]
public async Task Schema_Should_Be_Valid()
{
    // Arrange
    var schema = await new ServiceCollection()
        .AddSingleton<Query>()
        .AddGraphQL()
        .AddQueryType<Query>()
        .BuildSchemaAsync();
    
    // Act & Assert
    Assert.NotNull(schema);
    Assert.NotEmpty(schema.Types);
    Assert.Contains(schema.Types, t => t.Name == "Book");
}
```

## 고급 기능

### 스키마 스티칭

```csharp
// 여러 GraphQL 서비스 통합
builder.Services
    .AddGraphQLServer()
    .AddRemoteSchema("products")
    .AddRemoteSchema("users")
    .AddTypeExtensionsFromFile("./stitching.graphql");

// stitching.graphql
// extend type Product {
//   reviews: [Review!]!
// }
```

### 커스텀 스칼라 타입

```csharp
public class KoreanPhoneNumberType : ScalarType<string, StringValueNode>
{
    public KoreanPhoneNumberType() : base("KoreanPhoneNumber")
    {
        Description = "한국 전화번호 형식";
    }
    
    public override IValueNode ParseResult(object? resultValue)
    {
        if (resultValue is string s)
            return new StringValueNode(s);
            
        throw new SerializationException(
            "전화번호는 문자열이어야 합니다.", this);
    }
    
    protected override string ParseLiteral(StringValueNode valueSyntax)
    {
        var value = valueSyntax.Value;
        
        if (Regex.IsMatch(value, @"^01[0-9]-\d{4}-\d{4}$"))
            return value;
            
        throw new SerializationException(
            "올바른 전화번호 형식이 아닙니다. (예: 010-1234-5678)", this);
    }
}

// 등록
builder.Services
    .AddGraphQLServer()
    .AddType<KoreanPhoneNumberType>();
```

### 디렉티브

```csharp
public class UpperCaseDirectiveType : DirectiveType
{
    protected override void Configure(IDirectiveTypeDescriptor descriptor)
    {
        descriptor.Name("uppercase");
        descriptor.Location(DirectiveLocation.FieldDefinition);
        descriptor.Use(next => context =>
        {
            var result = next(context);
            
            if (context.Result is string s)
            {
                context.Result = s.ToUpper();
            }
            
            return result;
        });
    }
}

// 사용
public class BookType : ObjectType<Book>
{
    protected override void Configure(IObjectTypeDescriptor<Book> descriptor)
    {
        descriptor
            .Field(f => f.Title)
            .Directive("uppercase");
    }
}
```

## 모범 사례

### 스키마 설계 원칙

1. **명확한 네이밍**: 타입과 필드 이름은 직관적으로
2. **일관성 유지**: 네이밍 컨벤션 통일
3. **버저닝 전략**: 필드 deprecation 활용
4. **단일 책임**: 각 타입은 하나의 개념만 표현

### 성능 고려사항

1. **DataLoader 활용**: N+1 문제 방지
2. **프로젝션 사용**: 필요한 데이터만 조회
3. **페이징 구현**: 대량 데이터 처리
4. **캐싱 전략**: Redis 등 활용

### 보안 모범 사례

1. **인증/권한 검증**: 모든 민감한 작업에 적용
2. **입력 검증**: 모든 입력값 검증
3. **쿼리 복잡도 제한**: DoS 공격 방지
4. **에러 처리**: 상세 정보 노출 방지

## 마무리

Hot Chocolate는 ASP.NET Core에서 GraphQL API를 구축하는 데 필요한 모든 기능을 제공하는 강력한 프레임워크입니다. Code-First 접근 방식, 뛰어난 성능, 그리고 ASP.NET Core와의 완벽한 통합으로 엔터프라이즈급 GraphQL API를 쉽게 구축할 수 있습니다.

주요 장점:
- 타입 안정성과 IntelliSense 지원
- 자동 스키마 생성과 문서화
- 강력한 필터링, 정렬, 페이징 기능
- DataLoader를 통한 성능 최적화
- 실시간 구독 지원

GraphQL과 Hot Chocolate를 활용하면 유연하고 효율적인 API를 구축할 수 있으며, 클라이언트 중심의 데이터 페칭을 통해 더 나은 사용자 경험을 제공할 수 있습니다.