# 권한 부여 (Authorization) - Policy, Role, Claims

## 권한 부여 개요

권한 부여(Authorization)는 인증된 사용자가 무엇을 할 수 있는지 결정하는 프로세스입니다. ASP.NET Core는 Role, Claims, Policy 기반의 다양한 권한 부여 방식을 제공합니다.

### 인증 vs 권한 부여
- **인증 (Authentication)**: 사용자가 누구인지 확인하는 과정
- **권한 부여 (Authorization)**: 인증된 사용자가 무엇을 할 수 있는지 결정하는 과정

## 기본 권한 부여 설정

### Authorization 서비스 구성
```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// 인증 설정 (JWT 예제)
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
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
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:SecretKey"]))
        };
    });

// 권한 부여 설정
builder.Services.AddAuthorization(options =>
{
    // 기본 정책 설정
    options.FallbackPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();
        
    // 사용자 정의 정책 추가
    options.AddPolicy("RequireAdminRole", policy =>
        policy.RequireRole("Admin"));
});

var app = builder.Build();

// 미들웨어 순서 중요
app.UseAuthentication(); // 먼저 사용자 인증
app.UseAuthorization();  // 그 다음 권한 확인
```

### 기본 Authorize 특성 사용
```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    // 인증된 사용자만 접근 가능
    [Authorize]
    [HttpGet]
    public IActionResult GetProducts()
    {
        return Ok(new[] { "Product1", "Product2" });
    }
    
    // 익명 사용자도 접근 가능
    [AllowAnonymous]
    [HttpGet("public")]
    public IActionResult GetPublicProducts()
    {
        return Ok(new[] { "Public Product" });
    }
    
    // 전체 컨트롤러에 적용
    [Authorize]
    public class SecureController : ControllerBase
    {
        // 모든 액션이 인증 필요
    }
}
```

## Role 기반 권한 부여

### Role 관리 서비스
```csharp
public interface IRoleService
{
    Task<bool> CreateRoleAsync(string roleName, string description = null);
    Task<bool> DeleteRoleAsync(string roleName);
    Task<bool> RoleExistsAsync(string roleName);
    Task<IList<string>> GetAllRolesAsync();
    Task<bool> AddUserToRoleAsync(string userId, string roleName);
    Task<bool> RemoveUserFromRoleAsync(string userId, string roleName);
    Task<IList<string>> GetUserRolesAsync(string userId);
}

public class RoleService : IRoleService
{
    private readonly UserManager<ApplicationUser> _userManager;
    private readonly RoleManager<IdentityRole> _roleManager;
    
    public RoleService(
        UserManager<ApplicationUser> userManager,
        RoleManager<IdentityRole> roleManager)
    {
        _userManager = userManager;
        _roleManager = roleManager;
    }
    
    public async Task<bool> CreateRoleAsync(string roleName, string description = null)
    {
        if (await _roleManager.RoleExistsAsync(roleName))
        {
            return false;
        }
        
        var role = new IdentityRole(roleName);
        var result = await _roleManager.CreateAsync(role);
        return result.Succeeded;
    }
    
    public async Task<bool> AddUserToRoleAsync(string userId, string roleName)
    {
        var user = await _userManager.FindByIdAsync(userId);
        if (user == null) return false;
        
        if (!await _roleManager.RoleExistsAsync(roleName))
        {
            await CreateRoleAsync(roleName);
        }
        
        var result = await _userManager.AddToRoleAsync(user, roleName);
        return result.Succeeded;
    }
    
    public async Task<IList<string>> GetUserRolesAsync(string userId)
    {
        var user = await _userManager.FindByIdAsync(userId);
        if (user == null) return new List<string>();
        
        return await _userManager.GetRolesAsync(user);
    }
}
```

### Role 기반 컨트롤러
```csharp
[ApiController]
[Route("api/[controller]")]
public class AdminController : ControllerBase
{
    private readonly IUserService _userService;
    
    public AdminController(IUserService userService)
    {
        _userService = userService;
    }
    
    // 단일 Role 요구
    [Authorize(Roles = "Admin")]
    [HttpGet("users")]
    public async Task<IActionResult> GetAllUsers()
    {
        var users = await _userService.GetAllUsersAsync();
        return Ok(users);
    }
    
    // 여러 Role 중 하나 (OR 조건)
    [Authorize(Roles = "Admin,Manager")]
    [HttpGet("reports")]
    public async Task<IActionResult> GetReports()
    {
        return Ok("Admin or Manager can see reports");
    }
    
    // 여러 Role 모두 필요 (AND 조건)
    [Authorize(Roles = "Admin")]
    [Authorize(Roles = "SuperUser")]
    [HttpDelete("system/{id}")]
    public async Task<IActionResult> DeleteSystemResource(int id)
    {
        // Admin이면서 SuperUser인 사용자만 접근 가능
        return Ok($"System resource {id} deleted");
    }
}

// 계층적 Role 구조
[ApiController]
[Route("api/[controller]")]
public class HierarchicalController : ControllerBase
{
    // CEO > Director > Manager > Employee
    [Authorize(Roles = "CEO,Director,Manager")]
    [HttpGet("company-reports")]
    public IActionResult GetCompanyReports()
    {
        var userRole = User.FindFirst(ClaimTypes.Role)?.Value;
        
        return userRole switch
        {
            "CEO" => Ok("Full company reports"),
            "Director" => Ok("Department reports"),
            "Manager" => Ok("Team reports"),
            _ => Forbid()
        };
    }
}
```

### JWT에 Role 정보 포함
```csharp
public class JwtTokenService
{
    private readonly IConfiguration _configuration;
    private readonly UserManager<ApplicationUser> _userManager;
    
    public JwtTokenService(
        IConfiguration configuration,
        UserManager<ApplicationUser> userManager)
    {
        _configuration = configuration;
        _userManager = userManager;
    }
    
    public async Task<string> GenerateTokenAsync(ApplicationUser user)
    {
        var roles = await _userManager.GetRolesAsync(user);
        
        var claims = new List<Claim>
        {
            new Claim(ClaimTypes.NameIdentifier, user.Id),
            new Claim(ClaimTypes.Name, user.UserName),
            new Claim(ClaimTypes.Email, user.Email)
        };
        
        // Role을 Claim으로 추가
        foreach (var role in roles)
        {
            claims.Add(new Claim(ClaimTypes.Role, role));
        }
        
        var key = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(_configuration["Jwt:SecretKey"]));
        var credentials = new SigningCredentials(
            key, SecurityAlgorithms.HmacSha256);
        
        var token = new JwtSecurityToken(
            issuer: _configuration["Jwt:Issuer"],
            audience: _configuration["Jwt:Audience"],
            claims: claims,
            expires: DateTime.UtcNow.AddHours(1),
            signingCredentials: credentials
        );
        
        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

## Claims 기반 권한 부여

### Claim 이해하기
```csharp
// Claim은 사용자의 속성이나 권한을 나타내는 이름-값 쌍
public class ClaimExamples
{
    public void DemonstrateClaimTypes()
    {
        var claims = new List<Claim>
        {
            // 표준 Claim 타입
            new Claim(ClaimTypes.Name, "John Doe"),
            new Claim(ClaimTypes.Email, "john@example.com"),
            new Claim(ClaimTypes.DateOfBirth, "1990-01-01"),
            new Claim(ClaimTypes.Country, "USA"),
            
            // 사용자 정의 Claim
            new Claim("EmployeeId", "EMP001"),
            new Claim("Department", "Engineering"),
            new Claim("SecurityClearance", "Level3"),
            new Claim("CanApprove", "true"),
            
            // 동일한 타입의 여러 Claim
            new Claim("Permission", "CanRead"),
            new Claim("Permission", "CanWrite"),
            new Claim("Permission", "CanDelete")
        };
        
        var identity = new ClaimsIdentity(claims, "Cookie");
        var principal = new ClaimsPrincipal(identity);
    }
}
```

### Claim 기반 정책 설정
```csharp
// Program.cs
builder.Services.AddAuthorization(options =>
{
    // 특정 Claim 존재 확인
    options.AddPolicy("EmployeeOnly", policy =>
        policy.RequireClaim("EmployeeId"));
    
    // 특정 Claim 값 확인
    options.AddPolicy("EngineeringDepartment", policy =>
        policy.RequireClaim("Department", "Engineering"));
    
    // 여러 Claim 값 중 하나 (OR 조건)
    options.AddPolicy("TechDepartments", policy =>
        policy.RequireClaim("Department", "Engineering", "IT", "QA"));
    
    // 복잡한 Claim 조건
    options.AddPolicy("SeniorEngineer", policy =>
    {
        policy.RequireClaim("Department", "Engineering");
        policy.RequireClaim("YearsOfExperience");
        policy.RequireAssertion(context =>
        {
            var experienceClaim = context.User.FindFirst("YearsOfExperience");
            if (experienceClaim != null && 
                int.TryParse(experienceClaim.Value, out int years))
            {
                return years >= 5;
            }
            return false;
        });
    });
    
    // 여러 조건 조합
    options.AddPolicy("CanApproveExpenses", policy =>
    {
        policy.RequireAuthenticatedUser();
        policy.RequireRole("Manager", "Director");
        policy.RequireClaim("CanApprove", "true");
    });
});
```

### Claim 기반 컨트롤러
```csharp
[ApiController]
[Route("api/[controller]")]
public class EmployeeController : ControllerBase
{
    private readonly IEmployeeService _employeeService;
    
    public EmployeeController(IEmployeeService employeeService)
    {
        _employeeService = employeeService;
    }
    
    [Authorize(Policy = "EmployeeOnly")]
    [HttpGet("profile")]
    public IActionResult GetMyProfile()
    {
        var employeeId = User.FindFirst("EmployeeId")?.Value;
        var department = User.FindFirst("Department")?.Value;
        
        return Ok(new
        {
            EmployeeId = employeeId,
            Department = department,
            Name = User.Identity.Name,
            Email = User.FindFirst(ClaimTypes.Email)?.Value
        });
    }
    
    [Authorize(Policy = "EngineeringDepartment")]
    [HttpGet("engineering-resources")]
    public async Task<IActionResult> GetEngineeringResources()
    {
        var resources = await _employeeService.GetDepartmentResourcesAsync("Engineering");
        return Ok(resources);
    }
    
    [Authorize(Policy = "SeniorEngineer")]
    [HttpPost("code-review/{pullRequestId}")]
    public async Task<IActionResult> ApproveCodeReview(int pullRequestId)
    {
        var employeeId = User.FindFirst("EmployeeId")?.Value;
        await _employeeService.ApproveCodeReviewAsync(pullRequestId, employeeId);
        return Ok(new { message = "Code review approved" });
    }
}
```

## Policy 기반 권한 부여

### 사용자 정의 요구사항 및 핸들러
```csharp
// 나이 제한 요구사항
public class MinimumAgeRequirement : IAuthorizationRequirement
{
    public int MinimumAge { get; }
    
    public MinimumAgeRequirement(int minimumAge)
    {
        MinimumAge = minimumAge;
    }
}

// 나이 제한 핸들러
public class MinimumAgeHandler : AuthorizationHandler<MinimumAgeRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        MinimumAgeRequirement requirement)
    {
        var dateOfBirthClaim = context.User.FindFirst(
            c => c.Type == ClaimTypes.DateOfBirth);
        
        if (dateOfBirthClaim == null)
        {
            return Task.CompletedTask;
        }
        
        var dateOfBirth = Convert.ToDateTime(dateOfBirthClaim.Value);
        var age = DateTime.Today.Year - dateOfBirth.Year;
        
        if (dateOfBirth > DateTime.Today.AddYears(-age))
        {
            age--;
        }
        
        if (age >= requirement.MinimumAge)
        {
            context.Succeed(requirement);
        }
        
        return Task.CompletedTask;
    }
}

// 작업 시간 제한 요구사항
public class WorkingHoursRequirement : IAuthorizationRequirement
{
    public TimeSpan StartTime { get; }
    public TimeSpan EndTime { get; }
    
    public WorkingHoursRequirement(string startTime, string endTime)
    {
        StartTime = TimeSpan.Parse(startTime);
        EndTime = TimeSpan.Parse(endTime);
    }
}

// 작업 시간 제한 핸들러
public class WorkingHoursHandler : AuthorizationHandler<WorkingHoursRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        WorkingHoursRequirement requirement)
    {
        var currentTime = DateTime.Now.TimeOfDay;
        
        // 관리자는 시간 제한 없음
        if (context.User.IsInRole("Admin"))
        {
            context.Succeed(requirement);
            return Task.CompletedTask;
        }
        
        if (currentTime >= requirement.StartTime && 
            currentTime <= requirement.EndTime)
        {
            context.Succeed(requirement);
        }
        
        return Task.CompletedTask;
    }
}
```

### Policy 등록 및 사용
```csharp
// Program.cs
builder.Services.AddSingleton<IAuthorizationHandler, MinimumAgeHandler>();
builder.Services.AddSingleton<IAuthorizationHandler, WorkingHoursHandler>();

builder.Services.AddAuthorization(options =>
{
    // 나이 제한 정책
    options.AddPolicy("AtLeast21", policy =>
        policy.Requirements.Add(new MinimumAgeRequirement(21)));
    
    // 작업 시간 제한 정책
    options.AddPolicy("BusinessHours", policy =>
        policy.Requirements.Add(new WorkingHoursRequirement("09:00", "18:00")));
    
    // 복합 정책
    options.AddPolicy("PremiumContent", policy =>
    {
        policy.RequireAuthenticatedUser();
        policy.RequireClaim("Subscription", "Premium");
        policy.Requirements.Add(new MinimumAgeRequirement(18));
    });
});

// 컨트롤러에서 사용
[ApiController]
[Route("api/[controller]")]
public class ContentController : ControllerBase
{
    [Authorize(Policy = "AtLeast21")]
    [HttpGet("adult-content")]
    public IActionResult GetAdultContent()
    {
        return Ok("Content for 21+ users");
    }
    
    [Authorize(Policy = "BusinessHours")]
    [HttpPost("submit-report")]
    public IActionResult SubmitReport([FromBody] ReportDto report)
    {
        return Ok("Report submitted during business hours");
    }
    
    [Authorize(Policy = "PremiumContent")]
    [HttpGet("premium")]
    public IActionResult GetPremiumContent()
    {
        return Ok("Premium content for adult premium subscribers");
    }
}
```

## Resource 기반 권한 부여

### Resource 기반 핸들러
```csharp
// Document 모델
public class Document
{
    public int Id { get; set; }
    public string Title { get; set; }
    public string Content { get; set; }
    public string OwnerId { get; set; }
    public List<string> SharedWithUserIds { get; set; } = new();
    public DateTime CreatedAt { get; set; }
    public bool IsPublic { get; set; }
}

// Operations 정의
public static class DocumentOperations
{
    public static OperationAuthorizationRequirement Create =
        new OperationAuthorizationRequirement { Name = nameof(Create) };
    public static OperationAuthorizationRequirement Read =
        new OperationAuthorizationRequirement { Name = nameof(Read) };
    public static OperationAuthorizationRequirement Update =
        new OperationAuthorizationRequirement { Name = nameof(Update) };
    public static OperationAuthorizationRequirement Delete =
        new OperationAuthorizationRequirement { Name = nameof(Delete) };
}

// Document Authorization Handler
public class DocumentAuthorizationHandler :
    AuthorizationHandler<OperationAuthorizationRequirement, Document>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        OperationAuthorizationRequirement requirement,
        Document resource)
    {
        var userId = context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        
        if (string.IsNullOrEmpty(userId))
        {
            return Task.CompletedTask;
        }
        
        // 관리자는 모든 작업 가능
        if (context.User.IsInRole("Admin"))
        {
            context.Succeed(requirement);
            return Task.CompletedTask;
        }
        
        // 소유자는 모든 작업 가능
        if (resource.OwnerId == userId)
        {
            context.Succeed(requirement);
            return Task.CompletedTask;
        }
        
        // 읽기 작업에 대한 권한 확인
        if (requirement.Name == DocumentOperations.Read.Name)
        {
            // 공개 문서는 누구나 읽기 가능
            if (resource.IsPublic)
            {
                context.Succeed(requirement);
                return Task.CompletedTask;
            }
            
            // 공유된 사용자는 읽기 가능
            if (resource.SharedWithUserIds.Contains(userId))
            {
                context.Succeed(requirement);
                return Task.CompletedTask;
            }
        }
        
        return Task.CompletedTask;
    }
}
```

### Resource 기반 권한 부여 사용
```csharp
[ApiController]
[Route("api/[controller]")]
public class DocumentsController : ControllerBase
{
    private readonly IAuthorizationService _authorizationService;
    private readonly IDocumentRepository _documentRepository;
    
    public DocumentsController(
        IAuthorizationService authorizationService,
        IDocumentRepository documentRepository)
    {
        _authorizationService = authorizationService;
        _documentRepository = documentRepository;
    }
    
    [HttpGet("{id}")]
    public async Task<IActionResult> GetDocument(int id)
    {
        var document = await _documentRepository.GetByIdAsync(id);
        
        if (document == null)
        {
            return NotFound();
        }
        
        var authResult = await _authorizationService.AuthorizeAsync(
            User, document, DocumentOperations.Read);
        
        if (!authResult.Succeeded)
        {
            return Forbid();
        }
        
        return Ok(document);
    }
    
    [HttpPut("{id}")]
    public async Task<IActionResult> UpdateDocument(int id, [FromBody] DocumentUpdateDto dto)
    {
        var document = await _documentRepository.GetByIdAsync(id);
        
        if (document == null)
        {
            return NotFound();
        }
        
        var authResult = await _authorizationService.AuthorizeAsync(
            User, document, DocumentOperations.Update);
        
        if (!authResult.Succeeded)
        {
            return Forbid();
        }
        
        document.Title = dto.Title;
        document.Content = dto.Content;
        await _documentRepository.UpdateAsync(document);
        
        return Ok(document);
    }
    
    [HttpDelete("{id}")]
    public async Task<IActionResult> DeleteDocument(int id)
    {
        var document = await _documentRepository.GetByIdAsync(id);
        
        if (document == null)
        {
            return NotFound();
        }
        
        var authResult = await _authorizationService.AuthorizeAsync(
            User, document, DocumentOperations.Delete);
        
        if (!authResult.Succeeded)
        {
            return Forbid();
        }
        
        await _documentRepository.DeleteAsync(id);
        
        return NoContent();
    }
    
    [HttpPost("{id}/share")]
    public async Task<IActionResult> ShareDocument(int id, [FromBody] ShareDocumentDto dto)
    {
        var document = await _documentRepository.GetByIdAsync(id);
        
        if (document == null)
        {
            return NotFound();
        }
        
        // 소유자만 공유 가능
        var authResult = await _authorizationService.AuthorizeAsync(
            User, document, DocumentOperations.Update);
        
        if (!authResult.Succeeded)
        {
            return Forbid();
        }
        
        document.SharedWithUserIds.AddRange(dto.UserIds);
        await _documentRepository.UpdateAsync(document);
        
        return Ok(new { message = "Document shared successfully" });
    }
}
```

## 사용자 정의 Authorization

### 사용자 정의 Authorization Attribute
```csharp
// Permission Attribute
public class RequirePermissionAttribute : TypeFilterAttribute
{
    public RequirePermissionAttribute(string permission) 
        : base(typeof(PermissionAuthorizationFilter))
    {
        Arguments = new object[] { permission };
    }
}

// Permission Authorization Filter
public class PermissionAuthorizationFilter : IAsyncAuthorizationFilter
{
    private readonly string _permission;
    private readonly IPermissionService _permissionService;
    
    public PermissionAuthorizationFilter(
        string permission,
        IPermissionService permissionService)
    {
        _permission = permission;
        _permissionService = permissionService;
    }
    
    public async Task OnAuthorizationAsync(AuthorizationFilterContext context)
    {
        var user = context.HttpContext.User;
        
        if (!user.Identity.IsAuthenticated)
        {
            context.Result = new UnauthorizedResult();
            return;
        }
        
        var userId = user.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        var hasPermission = await _permissionService.UserHasPermissionAsync(
            userId, _permission);
        
        if (!hasPermission)
        {
            context.Result = new ObjectResult(new
            {
                error = "Forbidden",
                message = $"Permission '{_permission}' is required"
            })
            {
                StatusCode = 403
            };
        }
    }
}

// Permission Service
public interface IPermissionService
{
    Task<bool> UserHasPermissionAsync(string userId, string permission);
    Task<List<string>> GetUserPermissionsAsync(string userId);
    Task GrantPermissionAsync(string userId, string permission);
    Task RevokePermissionAsync(string userId, string permission);
}

public class PermissionService : IPermissionService
{
    private readonly ApplicationDbContext _context;
    private readonly IMemoryCache _cache;
    
    public PermissionService(ApplicationDbContext context, IMemoryCache cache)
    {
        _context = context;
        _cache = cache;
    }
    
    public async Task<bool> UserHasPermissionAsync(string userId, string permission)
    {
        var cacheKey = $"permission_{userId}_{permission}";
        
        if (_cache.TryGetValue<bool>(cacheKey, out var cachedResult))
        {
            return cachedResult;
        }
        
        // 직접 권한 확인
        var hasDirectPermission = await _context.UserPermissions
            .AnyAsync(up => up.UserId == userId && up.Permission == permission);
        
        if (hasDirectPermission)
        {
            _cache.Set(cacheKey, true, TimeSpan.FromMinutes(5));
            return true;
        }
        
        // Role 기반 권한 확인
        var hasRolePermission = await (
            from ur in _context.UserRoles
            join rp in _context.RolePermissions on ur.RoleId equals rp.RoleId
            where ur.UserId == userId && rp.Permission == permission
            select rp
        ).AnyAsync();
        
        _cache.Set(cacheKey, hasRolePermission, TimeSpan.FromMinutes(5));
        return hasRolePermission;
    }
}
```

### 사용자 정의 Authorization 사용
```csharp
[ApiController]
[Route("api/[controller]")]
public class ReportsController : ControllerBase
{
    private readonly IReportService _reportService;
    
    public ReportsController(IReportService reportService)
    {
        _reportService = reportService;
    }
    
    [RequirePermission("Reports.View")]
    [HttpGet]
    public async Task<IActionResult> GetReports()
    {
        var reports = await _reportService.GetAllReportsAsync();
        return Ok(reports);
    }
    
    [RequirePermission("Reports.Create")]
    [HttpPost]
    public async Task<IActionResult> CreateReport([FromBody] CreateReportDto dto)
    {
        var report = await _reportService.CreateReportAsync(dto);
        return CreatedAtAction(nameof(GetReport), new { id = report.Id }, report);
    }
    
    [RequirePermission("Reports.Export")]
    [HttpGet("{id}/export")]
    public async Task<IActionResult> ExportReport(int id)
    {
        var file = await _reportService.ExportReportAsync(id);
        return File(file.Content, file.ContentType, file.FileName);
    }
}
```

## 고급 시나리오

### 계층적 권한 시스템
```csharp
// 조직 계층 요구사항
public class OrganizationLevelRequirement : IAuthorizationRequirement
{
    public string MinimumLevel { get; }
    
    public OrganizationLevelRequirement(string minimumLevel)
    {
        MinimumLevel = minimumLevel;
    }
}

// 조직 계층 핸들러
public class OrganizationLevelHandler : AuthorizationHandler<OrganizationLevelRequirement>
{
    private readonly IOrganizationService _organizationService;
    
    private readonly Dictionary<string, int> _levelHierarchy = new()
    {
        { "CEO", 5 },
        { "Director", 4 },
        { "Manager", 3 },
        { "TeamLead", 2 },
        { "Employee", 1 }
    };
    
    public OrganizationLevelHandler(IOrganizationService organizationService)
    {
        _organizationService = organizationService;
    }
    
    protected override async Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        OrganizationLevelRequirement requirement)
    {
        var userId = context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        
        if (string.IsNullOrEmpty(userId))
        {
            return;
        }
        
        var userLevel = await _organizationService.GetUserLevelAsync(userId);
        
        if (_levelHierarchy.TryGetValue(userLevel, out var userLevelValue) &&
            _levelHierarchy.TryGetValue(requirement.MinimumLevel, out var requiredLevelValue))
        {
            if (userLevelValue >= requiredLevelValue)
            {
                context.Succeed(requirement);
            }
        }
    }
}
```

### Multi-tenant Authorization
```csharp
// Tenant 요구사항
public class TenantAuthorizationRequirement : IAuthorizationRequirement { }

// Tenant 핸들러
public class TenantAuthorizationHandler : AuthorizationHandler<TenantAuthorizationRequirement>
{
    private readonly IHttpContextAccessor _httpContextAccessor;
    
    public TenantAuthorizationHandler(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }
    
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        TenantAuthorizationRequirement requirement)
    {
        var httpContext = _httpContextAccessor.HttpContext;
        var userTenantId = context.User.FindFirst("TenantId")?.Value;
        
        // 헤더에서 요청된 TenantId 추출
        var requestedTenantId = httpContext.Request.Headers["X-Tenant-Id"].FirstOrDefault();
        
        // Route에서 추출
        if (string.IsNullOrEmpty(requestedTenantId))
        {
            requestedTenantId = httpContext.Request.RouteValues["tenantId"]?.ToString();
        }
        
        // SuperAdmin은 모든 테넌트 접근 가능
        if (context.User.IsInRole("SuperAdmin"))
        {
            context.Succeed(requirement);
            return Task.CompletedTask;
        }
        
        // 사용자가 요청한 테넌트에 속하는지 확인
        if (!string.IsNullOrEmpty(userTenantId) && 
            userTenantId == requestedTenantId)
        {
            context.Succeed(requirement);
        }
        
        return Task.CompletedTask;
    }
}

// Multi-tenant Controller
[ApiController]
[Route("api/tenants/{tenantId}/[controller]")]
[Authorize(Policy = "TenantAccess")]
public class TenantResourcesController : ControllerBase
{
    [HttpGet("data")]
    public async Task<IActionResult> GetTenantData(string tenantId)
    {
        // tenantId는 이미 권한 부여로 검증됨
        return Ok($"Data for tenant: {tenantId}");
    }
}
```

## 모범 사례

### Authorization 캐싱
```csharp
public class CachedAuthorizationHandler : IAuthorizationHandler
{
    private readonly IMemoryCache _cache;
    private readonly ILogger<CachedAuthorizationHandler> _logger;
    
    public CachedAuthorizationHandler(
        IMemoryCache cache,
        ILogger<CachedAuthorizationHandler> logger)
    {
        _cache = cache;
        _logger = logger;
    }
    
    public async Task HandleAsync(AuthorizationHandlerContext context)
    {
        // 캐시 키 생성
        var user = context.User;
        var requirements = string.Join(",", 
            context.Requirements.Select(r => r.GetType().Name));
        var cacheKey = $"auth_{user.Identity.Name}_{requirements}";
        
        // 캐시 확인
        if (_cache.TryGetValue<bool>(cacheKey, out var cachedResult))
        {
            if (cachedResult)
            {
                context.Succeed(context.Requirements.First());
            }
            return;
        }
        
        // 실제 권한 확인은 다른 핸들러에게 위임
        await Task.CompletedTask;
    }
}
```

### Authorization 로깅 및 감사
```csharp
public class AuditAuthorizationHandler : IAuthorizationHandler
{
    private readonly IAuditService _auditService;
    
    public AuditAuthorizationHandler(IAuditService auditService)
    {
        _auditService = auditService;
    }
    
    public async Task HandleAsync(AuthorizationHandlerContext context)
    {
        // 모든 권한 확인 시도 기록
        foreach (var requirement in context.PendingRequirements.ToList())
        {
            await _auditService.LogAuthorizationAttemptAsync(new AuthorizationAudit
            {
                UserId = context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value,
                UserName = context.User.Identity?.Name,
                Requirement = requirement.GetType().Name,
                Resource = context.Resource?.GetType().Name,
                Timestamp = DateTime.UtcNow,
                Success = context.HasSucceeded
            });
        }
    }
}
```

### 테스트 가능한 Authorization
```csharp
[TestClass]
public class DocumentAuthorizationHandlerTests
{
    private DocumentAuthorizationHandler _handler;
    private ClaimsPrincipal _owner;
    private ClaimsPrincipal _sharedUser;
    private ClaimsPrincipal _admin;
    private Document _privateDocument;
    
    [TestInitialize]
    public void Setup()
    {
        _handler = new DocumentAuthorizationHandler();
        
        _owner = new ClaimsPrincipal(new ClaimsIdentity(new[]
        {
            new Claim(ClaimTypes.NameIdentifier, "owner123"),
            new Claim(ClaimTypes.Name, "Document Owner")
        }, "Test"));
        
        _sharedUser = new ClaimsPrincipal(new ClaimsIdentity(new[]
        {
            new Claim(ClaimTypes.NameIdentifier, "shared456"),
            new Claim(ClaimTypes.Name, "Shared User")
        }, "Test"));
        
        _admin = new ClaimsPrincipal(new ClaimsIdentity(new[]
        {
            new Claim(ClaimTypes.NameIdentifier, "admin789"),
            new Claim(ClaimTypes.Name, "Admin User"),
            new Claim(ClaimTypes.Role, "Admin")
        }, "Test"));
        
        _privateDocument = new Document
        {
            Id = 1,
            Title = "Private Document",
            OwnerId = "owner123",
            SharedWithUserIds = new List<string> { "shared456" },
            IsPublic = false
        };
    }
    
    [TestMethod]
    public async Task Owner_Should_Have_All_Permissions()
    {
        // Arrange
        var requirement = DocumentOperations.Delete;
        var context = new AuthorizationHandlerContext(
            new[] { requirement }, _owner, _privateDocument);
        
        // Act
        await _handler.HandleAsync(context);
        
        // Assert
        Assert.IsTrue(context.HasSucceeded);
    }
    
    [TestMethod]
    public async Task SharedUser_Should_Only_Read()
    {
        // Read permission - should succeed
        var readContext = new AuthorizationHandlerContext(
            new[] { DocumentOperations.Read }, _sharedUser, _privateDocument);
        await _handler.HandleAsync(readContext);
        Assert.IsTrue(readContext.HasSucceeded);
        
        // Update permission - should fail
        var updateContext = new AuthorizationHandlerContext(
            new[] { DocumentOperations.Update }, _sharedUser, _privateDocument);
        await _handler.HandleAsync(updateContext);
        Assert.IsFalse(updateContext.HasSucceeded);
    }
    
    [TestMethod]
    public async Task Admin_Should_Have_All_Permissions()
    {
        // Arrange
        var requirement = DocumentOperations.Delete;
        var context = new AuthorizationHandlerContext(
            new[] { requirement }, _admin, _privateDocument);
        
        // Act
        await _handler.HandleAsync(context);
        
        // Assert
        Assert.IsTrue(context.HasSucceeded);
    }
}
```

권한 부여는 애플리케이션 보안의 핵심 요소입니다. Role, Claims, Policy 기반의 다양한 접근 방식을 조합하여 유연하고 확장 가능한 권한 시스템을 구축할 수 있습니다.