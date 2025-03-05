# 인증과 권한 부여 고급

## 개요

Blazor에서 고급 인증 및 권한 부여는 보안이 강화된 엔터프라이즈 애플리케이션 구축의 핵심입니다. 이 장에서는 JWT 토큰, OAuth 2.0/OpenID Connect, 역할 기반 접근 제어, 정책 기반 권한 부여 등을 학습합니다.

## 1. JWT 토큰 기반 인증

### 1.1 JWT 토큰 관리

```csharp
// Services/JwtTokenService.cs
public interface IJwtTokenService
{
    Task<TokenResult> GetTokenAsync();
    Task<TokenResult> RefreshTokenAsync();
    Task RevokeTokenAsync();
    bool IsTokenExpired();
    ClaimsPrincipal? GetPrincipalFromToken(string token);
}

public class JwtTokenService : IJwtTokenService
{
    private readonly HttpClient _httpClient;
    private readonly ILocalStorageService _localStorage;
    private readonly ILogger<JwtTokenService> _logger;
    private readonly JwtSecurityTokenHandler _tokenHandler;
    
    private const string AccessTokenKey = "access_token";
    private const string RefreshTokenKey = "refresh_token";
    private const string TokenExpiryKey = "token_expiry";
    
    public JwtTokenService(
        HttpClient httpClient,
        ILocalStorageService localStorage,
        ILogger<JwtTokenService> logger)
    {
        _httpClient = httpClient;
        _localStorage = localStorage;
        _logger = logger;
        _tokenHandler = new JwtSecurityTokenHandler();
    }
    
    public async Task<TokenResult> GetTokenAsync()
    {
        var token = await _localStorage.GetItemAsync<string>(AccessTokenKey);
        
        if (string.IsNullOrEmpty(token))
        {
            return TokenResult.Failed("No token found");
        }
        
        if (IsTokenExpired())
        {
            _logger.LogInformation("Token expired, attempting refresh");
            return await RefreshTokenAsync();
        }
        
        return TokenResult.Success(token);
    }
    
    public async Task<TokenResult> RefreshTokenAsync()
    {
        try
        {
            var refreshToken = await _localStorage.GetItemAsync<string>(RefreshTokenKey);
            
            if (string.IsNullOrEmpty(refreshToken))
            {
                return TokenResult.Failed("No refresh token available");
            }
            
            var response = await _httpClient.PostAsJsonAsync("api/auth/refresh", new
            {
                RefreshToken = refreshToken
            });
            
            if (response.IsSuccessStatusCode)
            {
                var tokenResponse = await response.Content.ReadFromJsonAsync<TokenResponse>();
                
                if (tokenResponse != null)
                {
                    await StoreTokensAsync(tokenResponse);
                    return TokenResult.Success(tokenResponse.AccessToken);
                }
            }
            
            return TokenResult.Failed("Failed to refresh token");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error refreshing token");
            return TokenResult.Failed(ex.Message);
        }
    }
    
    public async Task RevokeTokenAsync()
    {
        var refreshToken = await _localStorage.GetItemAsync<string>(RefreshTokenKey);
        
        if (!string.IsNullOrEmpty(refreshToken))
        {
            await _httpClient.PostAsJsonAsync("api/auth/revoke", new
            {
                RefreshToken = refreshToken
            });
        }
        
        await ClearTokensAsync();
    }
    
    public bool IsTokenExpired()
    {
        var expiryString = _localStorage.GetItemAsStringAsync(TokenExpiryKey).Result;
        
        if (DateTime.TryParse(expiryString, out var expiry))
        {
            return DateTime.UtcNow >= expiry.AddMinutes(-5); // 5분 여유
        }
        
        return true;
    }
    
    public ClaimsPrincipal? GetPrincipalFromToken(string token)
    {
        try
        {
            var validationParameters = new TokenValidationParameters
            {
                ValidateIssuer = false,
                ValidateAudience = false,
                ValidateLifetime = false,
                ValidateIssuerSigningKey = false,
                ClockSkew = TimeSpan.Zero
            };
            
            var principal = _tokenHandler.ValidateToken(token, validationParameters, out _);
            return principal;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error validating token");
            return null;
        }
    }
    
    private async Task StoreTokensAsync(TokenResponse response)
    {
        await _localStorage.SetItemAsync(AccessTokenKey, response.AccessToken);
        await _localStorage.SetItemAsync(RefreshTokenKey, response.RefreshToken);
        
        var token = _tokenHandler.ReadJwtToken(response.AccessToken);
        var expiry = token.ValidTo;
        await _localStorage.SetItemAsync(TokenExpiryKey, expiry.ToString("O"));
    }
    
    private async Task ClearTokensAsync()
    {
        await _localStorage.RemoveItemAsync(AccessTokenKey);
        await _localStorage.RemoveItemAsync(RefreshTokenKey);
        await _localStorage.RemoveItemAsync(TokenExpiryKey);
    }
}

// Models/TokenModels.cs
public class TokenResponse
{
    public string AccessToken { get; set; } = "";
    public string RefreshToken { get; set; } = "";
    public int ExpiresIn { get; set; }
    public string TokenType { get; set; } = "Bearer";
}

public class TokenResult
{
    public bool IsSuccess { get; }
    public string? Token { get; }
    public string? Error { get; }
    
    private TokenResult(bool isSuccess, string? token, string? error)
    {
        IsSuccess = isSuccess;
        Token = token;
        Error = error;
    }
    
    public static TokenResult Success(string token) => new(true, token, null);
    public static TokenResult Failed(string error) => new(false, null, error);
}
```

### 1.2 커스텀 AuthenticationStateProvider

```csharp
// Authentication/JwtAuthenticationStateProvider.cs
public class JwtAuthenticationStateProvider : AuthenticationStateProvider
{
    private readonly IJwtTokenService _tokenService;
    private readonly ILogger<JwtAuthenticationStateProvider> _logger;
    
    public JwtAuthenticationStateProvider(
        IJwtTokenService tokenService,
        ILogger<JwtAuthenticationStateProvider> logger)
    {
        _tokenService = tokenService;
        _logger = logger;
    }
    
    public override async Task<AuthenticationState> GetAuthenticationStateAsync()
    {
        try
        {
            var tokenResult = await _tokenService.GetTokenAsync();
            
            if (!tokenResult.IsSuccess || string.IsNullOrEmpty(tokenResult.Token))
            {
                return new AuthenticationState(new ClaimsPrincipal(new ClaimsIdentity()));
            }
            
            var principal = _tokenService.GetPrincipalFromToken(tokenResult.Token);
            
            if (principal == null)
            {
                return new AuthenticationState(new ClaimsPrincipal(new ClaimsIdentity()));
            }
            
            return new AuthenticationState(principal);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error getting authentication state");
            return new AuthenticationState(new ClaimsPrincipal(new ClaimsIdentity()));
        }
    }
    
    public async Task LoginAsync(LoginRequest request)
    {
        var response = await _httpClient.PostAsJsonAsync("api/auth/login", request);
        
        if (response.IsSuccessStatusCode)
        {
            var tokenResponse = await response.Content.ReadFromJsonAsync<TokenResponse>();
            
            if (tokenResponse != null)
            {
                await _tokenService.StoreTokensAsync(tokenResponse);
                NotifyAuthenticationStateChanged(GetAuthenticationStateAsync());
            }
        }
        else
        {
            throw new AuthenticationException("Login failed");
        }
    }
    
    public async Task LogoutAsync()
    {
        await _tokenService.RevokeTokenAsync();
        NotifyAuthenticationStateChanged(
            Task.FromResult(new AuthenticationState(
                new ClaimsPrincipal(new ClaimsIdentity()))));
    }
}
```

## 2. OAuth 2.0 / OpenID Connect

### 2.1 외부 인증 공급자 통합

```csharp
// Services/OAuthService.cs
public interface IOAuthService
{
    Task<string> GetAuthorizationUrlAsync(string provider);
    Task<AuthenticationResult> HandleCallbackAsync(string provider, string code, string state);
    Task<UserInfo> GetUserInfoAsync(string provider, string accessToken);
}

public class OAuthService : IOAuthService
{
    private readonly HttpClient _httpClient;
    private readonly IConfiguration _configuration;
    private readonly Dictionary<string, OAuthProvider> _providers;
    
    public OAuthService(HttpClient httpClient, IConfiguration configuration)
    {
        _httpClient = httpClient;
        _configuration = configuration;
        _providers = LoadProviders();
    }
    
    public Task<string> GetAuthorizationUrlAsync(string provider)
    {
        if (!_providers.TryGetValue(provider, out var config))
        {
            throw new ArgumentException($"Unknown provider: {provider}");
        }
        
        var state = GenerateState();
        var nonce = GenerateNonce();
        
        var parameters = new Dictionary<string, string>
        {
            ["client_id"] = config.ClientId,
            ["redirect_uri"] = config.RedirectUri,
            ["response_type"] = "code",
            ["scope"] = string.Join(" ", config.Scopes),
            ["state"] = state,
            ["nonce"] = nonce
        };
        
        if (config.ResponseMode != null)
        {
            parameters["response_mode"] = config.ResponseMode;
        }
        
        var query = string.Join("&", parameters.Select(p => 
            $"{p.Key}={Uri.EscapeDataString(p.Value)}"));
        
        return Task.FromResult($"{config.AuthorizationEndpoint}?{query}");
    }
    
    public async Task<AuthenticationResult> HandleCallbackAsync(
        string provider, string code, string state)
    {
        if (!_providers.TryGetValue(provider, out var config))
        {
            throw new ArgumentException($"Unknown provider: {provider}");
        }
        
        // Validate state
        if (!ValidateState(state))
        {
            throw new SecurityException("Invalid state parameter");
        }
        
        // Exchange code for tokens
        var tokenRequest = new FormUrlEncodedContent(new Dictionary<string, string>
        {
            ["grant_type"] = "authorization_code",
            ["code"] = code,
            ["redirect_uri"] = config.RedirectUri,
            ["client_id"] = config.ClientId,
            ["client_secret"] = config.ClientSecret
        });
        
        var response = await _httpClient.PostAsync(config.TokenEndpoint, tokenRequest);
        response.EnsureSuccessStatusCode();
        
        var tokenResponse = await response.Content.ReadFromJsonAsync<OAuthTokenResponse>();
        
        if (tokenResponse == null)
        {
            throw new InvalidOperationException("Failed to obtain tokens");
        }
        
        // Validate ID token if present
        if (!string.IsNullOrEmpty(tokenResponse.IdToken))
        {
            var principal = await ValidateIdTokenAsync(tokenResponse.IdToken, config);
            
            return new AuthenticationResult
            {
                IsSuccess = true,
                Principal = principal,
                AccessToken = tokenResponse.AccessToken,
                RefreshToken = tokenResponse.RefreshToken,
                ExpiresIn = tokenResponse.ExpiresIn
            };
        }
        
        // Get user info using access token
        var userInfo = await GetUserInfoAsync(provider, tokenResponse.AccessToken);
        
        return new AuthenticationResult
        {
            IsSuccess = true,
            Principal = CreatePrincipal(userInfo),
            AccessToken = tokenResponse.AccessToken,
            RefreshToken = tokenResponse.RefreshToken,
            ExpiresIn = tokenResponse.ExpiresIn
        };
    }
    
    public async Task<UserInfo> GetUserInfoAsync(string provider, string accessToken)
    {
        if (!_providers.TryGetValue(provider, out var config))
        {
            throw new ArgumentException($"Unknown provider: {provider}");
        }
        
        var request = new HttpRequestMessage(HttpMethod.Get, config.UserInfoEndpoint);
        request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", accessToken);
        
        var response = await _httpClient.SendAsync(request);
        response.EnsureSuccessStatusCode();
        
        return await response.Content.ReadFromJsonAsync<UserInfo>() 
            ?? throw new InvalidOperationException("Failed to get user info");
    }
    
    private Dictionary<string, OAuthProvider> LoadProviders()
    {
        var providers = new Dictionary<string, OAuthProvider>();
        
        var section = _configuration.GetSection("Authentication:Providers");
        
        foreach (var provider in section.GetChildren())
        {
            var config = new OAuthProvider
            {
                Name = provider.Key,
                ClientId = provider["ClientId"] ?? "",
                ClientSecret = provider["ClientSecret"] ?? "",
                AuthorizationEndpoint = provider["AuthorizationEndpoint"] ?? "",
                TokenEndpoint = provider["TokenEndpoint"] ?? "",
                UserInfoEndpoint = provider["UserInfoEndpoint"] ?? "",
                RedirectUri = provider["RedirectUri"] ?? "",
                Scopes = provider.GetSection("Scopes").Get<string[]>() ?? Array.Empty<string>(),
                ResponseMode = provider["ResponseMode"]
            };
            
            providers[provider.Key] = config;
        }
        
        return providers;
    }
}
```

### 2.2 소셜 로그인 컴포넌트

```csharp
// Components/SocialLogin.razor
@inject IOAuthService OAuthService
@inject NavigationManager Navigation

<div class="social-login">
    <h4>Sign in with</h4>
    
    <button class="btn-social google" @onclick="() => LoginWithProvider(\"google\")">
        <i class="fab fa-google"></i> Google
    </button>
    
    <button class="btn-social facebook" @onclick="() => LoginWithProvider(\"facebook\")">
        <i class="fab fa-facebook"></i> Facebook
    </button>
    
    <button class="btn-social github" @onclick="() => LoginWithProvider(\"github\")">
        <i class="fab fa-github"></i> GitHub
    </button>
    
    <button class="btn-social microsoft" @onclick="() => LoginWithProvider(\"microsoft\")">
        <i class="fab fa-microsoft"></i> Microsoft
    </button>
</div>

@code {
    private async Task LoginWithProvider(string provider)
    {
        try
        {
            var authUrl = await OAuthService.GetAuthorizationUrlAsync(provider);
            Navigation.NavigateTo(authUrl, forceLoad: true);
        }
        catch (Exception ex)
        {
            // Handle error
            Logger.LogError(ex, "Failed to initiate OAuth login for {Provider}", provider);
        }
    }
}

// Pages/Authentication/OAuthCallback.razor
@page "/authentication/oauth/callback"
@inject IOAuthService OAuthService
@inject IJwtTokenService TokenService
@inject AuthenticationStateProvider AuthStateProvider
@inject NavigationManager Navigation

<h3>Processing login...</h3>

@code {
    [Parameter]
    [SupplyParameterFromQuery]
    public string? Code { get; set; }
    
    [Parameter]
    [SupplyParameterFromQuery]
    public string? State { get; set; }
    
    [Parameter]
    [SupplyParameterFromQuery]
    public string? Provider { get; set; }
    
    protected override async Task OnInitializedAsync()
    {
        if (string.IsNullOrEmpty(Code) || string.IsNullOrEmpty(State) || string.IsNullOrEmpty(Provider))
        {
            Navigation.NavigateTo("/authentication/login?error=invalid_request");
            return;
        }
        
        try
        {
            var result = await OAuthService.HandleCallbackAsync(Provider, Code, State);
            
            if (result.IsSuccess)
            {
                // Exchange OAuth token for application JWT
                var appToken = await ExchangeForApplicationToken(result);
                
                // Store token and update authentication state
                await TokenService.StoreTokensAsync(appToken);
                await ((JwtAuthenticationStateProvider)AuthStateProvider)
                    .NotifyAuthenticationStateChanged();
                
                Navigation.NavigateTo("/");
            }
            else
            {
                Navigation.NavigateTo($"/authentication/login?error={result.Error}");
            }
        }
        catch (Exception ex)
        {
            Logger.LogError(ex, "OAuth callback error");
            Navigation.NavigateTo("/authentication/login?error=callback_error");
        }
    }
}
```

## 3. 역할 기반 접근 제어 (RBAC)

### 3.1 계층적 역할 시스템

```csharp
// Services/RoleHierarchyService.cs
public interface IRoleHierarchyService
{
    Task<IEnumerable<string>> GetEffectiveRolesAsync(string userId);
    Task<bool> IsInRoleAsync(string userId, string role);
    Task<bool> HasPermissionAsync(string userId, string permission);
}

public class RoleHierarchyService : IRoleHierarchyService
{
    private readonly IUserRoleRepository _userRoleRepository;
    private readonly IRoleRepository _roleRepository;
    private readonly IMemoryCache _cache;
    
    private readonly Dictionary<string, string[]> _roleHierarchy = new()
    {
        ["SuperAdmin"] = new[] { "Admin", "Manager", "User" },
        ["Admin"] = new[] { "Manager", "User" },
        ["Manager"] = new[] { "User" },
        ["User"] = Array.Empty<string>()
    };
    
    public RoleHierarchyService(
        IUserRoleRepository userRoleRepository,
        IRoleRepository roleRepository,
        IMemoryCache cache)
    {
        _userRoleRepository = userRoleRepository;
        _roleRepository = roleRepository;
        _cache = cache;
    }
    
    public async Task<IEnumerable<string>> GetEffectiveRolesAsync(string userId)
    {
        var cacheKey = $"user_roles_{userId}";
        
        if (_cache.TryGetValue<string[]>(cacheKey, out var cachedRoles))
        {
            return cachedRoles;
        }
        
        var directRoles = await _userRoleRepository.GetUserRolesAsync(userId);
        var effectiveRoles = new HashSet<string>(directRoles);
        
        // Add inherited roles
        foreach (var role in directRoles)
        {
            if (_roleHierarchy.TryGetValue(role, out var inheritedRoles))
            {
                foreach (var inheritedRole in inheritedRoles)
                {
                    effectiveRoles.Add(inheritedRole);
                }
            }
        }
        
        var result = effectiveRoles.ToArray();
        _cache.Set(cacheKey, result, TimeSpan.FromMinutes(5));
        
        return result;
    }
    
    public async Task<bool> IsInRoleAsync(string userId, string role)
    {
        var effectiveRoles = await GetEffectiveRolesAsync(userId);
        return effectiveRoles.Contains(role);
    }
    
    public async Task<bool> HasPermissionAsync(string userId, string permission)
    {
        var effectiveRoles = await GetEffectiveRolesAsync(userId);
        
        foreach (var role in effectiveRoles)
        {
            var rolePermissions = await GetRolePermissionsAsync(role);
            if (rolePermissions.Contains(permission))
            {
                return true;
            }
        }
        
        return false;
    }
    
    private async Task<IEnumerable<string>> GetRolePermissionsAsync(string role)
    {
        var cacheKey = $"role_permissions_{role}";
        
        if (_cache.TryGetValue<string[]>(cacheKey, out var cachedPermissions))
        {
            return cachedPermissions;
        }
        
        var permissions = await _roleRepository.GetRolePermissionsAsync(role);
        var result = permissions.ToArray();
        
        _cache.Set(cacheKey, result, TimeSpan.FromMinutes(10));
        
        return result;
    }
}

// Components/RoleBasedView.razor
@inject IRoleHierarchyService RoleService
@inject AuthenticationStateProvider AuthStateProvider

@if (hasAccess)
{
    @ChildContent
}
else if (ShowUnauthorized)
{
    <div class="alert alert-warning">
        You don't have permission to view this content.
    </div>
}

@code {
    [Parameter] public RenderFragment? ChildContent { get; set; }
    [Parameter] public string? RequiredRole { get; set; }
    [Parameter] public string? RequiredPermission { get; set; }
    [Parameter] public bool ShowUnauthorized { get; set; } = false;
    
    private bool hasAccess = false;
    
    protected override async Task OnParametersSetAsync()
    {
        var authState = await AuthStateProvider.GetAuthenticationStateAsync();
        var user = authState.User;
        
        if (!user.Identity?.IsAuthenticated ?? true)
        {
            hasAccess = false;
            return;
        }
        
        var userId = user.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        
        if (string.IsNullOrEmpty(userId))
        {
            hasAccess = false;
            return;
        }
        
        if (!string.IsNullOrEmpty(RequiredRole))
        {
            hasAccess = await RoleService.IsInRoleAsync(userId, RequiredRole);
        }
        else if (!string.IsNullOrEmpty(RequiredPermission))
        {
            hasAccess = await RoleService.HasPermissionAsync(userId, RequiredPermission);
        }
        else
        {
            hasAccess = true;
        }
    }
}
```

## 4. 정책 기반 권한 부여

### 4.1 커스텀 권한 부여 정책

```csharp
// Authorization/Requirements/MinimumAgeRequirement.cs
public class MinimumAgeRequirement : IAuthorizationRequirement
{
    public int MinimumAge { get; }
    
    public MinimumAgeRequirement(int minimumAge)
    {
        MinimumAge = minimumAge;
    }
}

public class MinimumAgeHandler : AuthorizationHandler<MinimumAgeRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        MinimumAgeRequirement requirement)
    {
        var dateOfBirthClaim = context.User.FindFirst(c => c.Type == "DateOfBirth");
        
        if (dateOfBirthClaim != null && 
            DateTime.TryParse(dateOfBirthClaim.Value, out var dateOfBirth))
        {
            var age = DateTime.Today.Year - dateOfBirth.Year;
            if (dateOfBirth.Date > DateTime.Today.AddYears(-age)) age--;
            
            if (age >= requirement.MinimumAge)
            {
                context.Succeed(requirement);
            }
        }
        
        return Task.CompletedTask;
    }
}

// Authorization/Requirements/ResourceOwnerRequirement.cs
public class ResourceOwnerRequirement : IAuthorizationRequirement { }

public class ResourceOwnerHandler : AuthorizationHandler<ResourceOwnerRequirement, IResource>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        ResourceOwnerRequirement requirement,
        IResource resource)
    {
        var userId = context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        
        if (!string.IsNullOrEmpty(userId) && resource.OwnerId == userId)
        {
            context.Succeed(requirement);
        }
        
        return Task.CompletedTask;
    }
}

// Authorization/PolicyProvider.cs
public class DynamicPolicyProvider : IAuthorizationPolicyProvider
{
    private readonly DefaultAuthorizationPolicyProvider _fallbackPolicyProvider;
    
    public DynamicPolicyProvider(IOptions<AuthorizationOptions> options)
    {
        _fallbackPolicyProvider = new DefaultAuthorizationPolicyProvider(options);
    }
    
    public Task<AuthorizationPolicy?> GetPolicyAsync(string policyName)
    {
        if (policyName.StartsWith("MinAge:"))
        {
            var age = int.Parse(policyName.Substring(7));
            var policy = new AuthorizationPolicyBuilder()
                .RequireAuthenticatedUser()
                .AddRequirements(new MinimumAgeRequirement(age))
                .Build();
            
            return Task.FromResult<AuthorizationPolicy?>(policy);
        }
        
        if (policyName.StartsWith("Permission:"))
        {
            var permission = policyName.Substring(11);
            var policy = new AuthorizationPolicyBuilder()
                .RequireAuthenticatedUser()
                .AddRequirements(new PermissionRequirement(permission))
                .Build();
            
            return Task.FromResult<AuthorizationPolicy?>(policy);
        }
        
        return _fallbackPolicyProvider.GetPolicyAsync(policyName);
    }
    
    public Task<AuthorizationPolicy> GetDefaultPolicyAsync() =>
        _fallbackPolicyProvider.GetDefaultPolicyAsync();
    
    public Task<AuthorizationPolicy?> GetFallbackPolicyAsync() =>
        _fallbackPolicyProvider.GetFallbackPolicyAsync();
}
```

### 4.2 정책 기반 컴포넌트

```csharp
// Components/PolicyAuthorizeView.razor
@inject IAuthorizationService AuthorizationService
@inject AuthenticationStateProvider AuthStateProvider

@if (isAuthorized)
{
    @Authorized
}
else if (isAuthenticating)
{
    @Authorizing
}
else
{
    @NotAuthorized
}

@code {
    [Parameter] public string Policy { get; set; } = "";
    [Parameter] public object? Resource { get; set; }
    [Parameter] public RenderFragment? Authorized { get; set; }
    [Parameter] public RenderFragment? NotAuthorized { get; set; }
    [Parameter] public RenderFragment? Authorizing { get; set; }
    
    private bool isAuthorized = false;
    private bool isAuthenticating = true;
    
    protected override async Task OnParametersSetAsync()
    {
        isAuthenticating = true;
        
        var authState = await AuthStateProvider.GetAuthenticationStateAsync();
        var user = authState.User;
        
        if (!user.Identity?.IsAuthenticated ?? true)
        {
            isAuthorized = false;
            isAuthenticating = false;
            return;
        }
        
        var authResult = Resource != null
            ? await AuthorizationService.AuthorizeAsync(user, Resource, Policy)
            : await AuthorizationService.AuthorizeAsync(user, Policy);
        
        isAuthorized = authResult.Succeeded;
        isAuthenticating = false;
    }
}

// Usage example
<PolicyAuthorizeView Policy="MinAge:18">
    <Authorized>
        <p>You are old enough to view this content.</p>
    </Authorized>
    <NotAuthorized>
        <p>You must be 18 or older to view this content.</p>
    </NotAuthorized>
</PolicyAuthorizeView>

<PolicyAuthorizeView Policy="Permission:DeletePost" Resource="@post">
    <Authorized>
        <button @onclick="DeletePost">Delete Post</button>
    </Authorized>
</PolicyAuthorizeView>
```

## 5. 다단계 인증 (MFA)

### 5.1 TOTP 기반 2FA

```csharp
// Services/TwoFactorService.cs
public interface ITwoFactorService
{
    Task<SetupTwoFactorResult> SetupTwoFactorAsync(string userId);
    Task<bool> ValidateTwoFactorCodeAsync(string userId, string code);
    Task<bool> DisableTwoFactorAsync(string userId);
    Task<string[]> GenerateRecoveryCodesAsync(string userId);
}

public class TwoFactorService : ITwoFactorService
{
    private readonly IUserRepository _userRepository;
    private readonly IDataProtector _dataProtector;
    
    public TwoFactorService(
        IUserRepository userRepository,
        IDataProtectionProvider dataProtectionProvider)
    {
        _userRepository = userRepository;
        _dataProtector = dataProtectionProvider.CreateProtector("TwoFactorAuth");
    }
    
    public async Task<SetupTwoFactorResult> SetupTwoFactorAsync(string userId)
    {
        var user = await _userRepository.GetByIdAsync(userId);
        
        if (user == null)
        {
            throw new InvalidOperationException("User not found");
        }
        
        // Generate secret key
        var key = KeyGeneration.GenerateRandomKey(20);
        var base32Key = Base32Encoding.ToString(key);
        
        // Generate QR code URI
        var issuer = "YourApp";
        var email = user.Email;
        var uri = $"otpauth://totp/{issuer}:{email}?secret={base32Key}&issuer={issuer}";
        
        // Generate QR code
        var qrGenerator = new QRCodeGenerator();
        var qrCodeData = qrGenerator.CreateQrCode(uri, QRCodeGenerator.ECCLevel.Q);
        var qrCode = new PngByteQRCode(qrCodeData);
        var qrCodeImage = qrCode.GetGraphic(10);
        
        // Store encrypted secret
        var encryptedSecret = _dataProtector.Protect(base32Key);
        await _userRepository.UpdateTwoFactorSecretAsync(userId, encryptedSecret);
        
        return new SetupTwoFactorResult
        {
            Secret = base32Key,
            QrCodeUri = uri,
            QrCodeImage = Convert.ToBase64String(qrCodeImage)
        };
    }
    
    public async Task<bool> ValidateTwoFactorCodeAsync(string userId, string code)
    {
        var user = await _userRepository.GetByIdAsync(userId);
        
        if (user?.TwoFactorSecret == null)
        {
            return false;
        }
        
        var secret = _dataProtector.Unprotect(user.TwoFactorSecret);
        var totp = new Totp(Base32Encoding.ToBytes(secret));
        
        // Allow for time drift
        return totp.VerifyTotp(code, out _, new VerificationWindow(2, 2));
    }
    
    public async Task<string[]> GenerateRecoveryCodesAsync(string userId)
    {
        var codes = new string[8];
        
        for (int i = 0; i < codes.Length; i++)
        {
            codes[i] = GenerateRecoveryCode();
        }
        
        // Hash and store recovery codes
        var hashedCodes = codes.Select(c => HashRecoveryCode(c)).ToArray();
        await _userRepository.UpdateRecoveryCodesAsync(userId, hashedCodes);
        
        return codes;
    }
    
    private string GenerateRecoveryCode()
    {
        var bytes = new byte[6];
        using var rng = RandomNumberGenerator.Create();
        rng.GetBytes(bytes);
        return Convert.ToBase32String(bytes).Replace("=", "").Substring(0, 10);
    }
    
    private string HashRecoveryCode(string code)
    {
        using var sha256 = SHA256.Create();
        var bytes = Encoding.UTF8.GetBytes(code);
        var hash = sha256.ComputeHash(bytes);
        return Convert.ToBase64String(hash);
    }
}

// Components/TwoFactorSetup.razor
@page "/account/security/2fa"
@inject ITwoFactorService TwoFactorService

<h3>Two-Factor Authentication Setup</h3>

@if (setupResult != null)
{
    <div class="two-factor-setup">
        <div class="qr-code">
            <img src="data:image/png;base64,@setupResult.QrCodeImage" alt="QR Code" />
        </div>
        
        <div class="manual-entry">
            <p>Can't scan? Enter this code manually:</p>
            <code class="secret">@setupResult.Secret</code>
        </div>
        
        <div class="verification">
            <p>Enter the 6-digit code from your authenticator app:</p>
            <input @bind="verificationCode" maxlength="6" />
            <button @onclick="VerifyAndEnable">Verify and Enable</button>
        </div>
    </div>
}

@if (recoveryCodes != null)
{
    <div class="recovery-codes">
        <h4>Recovery Codes</h4>
        <p>Save these codes in a safe place. Each can be used once if you lose access to your authenticator.</p>
        <div class="codes">
            @foreach (var code in recoveryCodes)
            {
                <code>@code</code>
            }
        </div>
    </div>
}

@code {
    private SetupTwoFactorResult? setupResult;
    private string verificationCode = "";
    private string[]? recoveryCodes;
    
    protected override async Task OnInitializedAsync()
    {
        var authState = await AuthStateProvider.GetAuthenticationStateAsync();
        var userId = authState.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        
        if (!string.IsNullOrEmpty(userId))
        {
            setupResult = await TwoFactorService.SetupTwoFactorAsync(userId);
        }
    }
    
    private async Task VerifyAndEnable()
    {
        var authState = await AuthStateProvider.GetAuthenticationStateAsync();
        var userId = authState.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        
        if (string.IsNullOrEmpty(userId)) return;
        
        var isValid = await TwoFactorService.ValidateTwoFactorCodeAsync(userId, verificationCode);
        
        if (isValid)
        {
            recoveryCodes = await TwoFactorService.GenerateRecoveryCodesAsync(userId);
            // Enable 2FA for user
            await UserService.EnableTwoFactorAsync(userId);
        }
        else
        {
            // Show error
        }
    }
}
```

## 마무리

Blazor에서 고급 인증 및 권한 부여를 구현하면 안전하고 확장 가능한 애플리케이션을 구축할 수 있습니다. JWT 토큰 관리, OAuth 2.0/OpenID Connect 통합, 역할 및 정책 기반 권한 부여, 다단계 인증 등을 통해 엔터프라이즈 수준의 보안을 달성할 수 있습니다. 항상 보안 모범 사례를 따르고 정기적으로 보안 감사를 수행하는 것이 중요합니다.