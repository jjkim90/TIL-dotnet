# 보안 모범 사례

## 개요

Blazor 애플리케이션의 보안은 사용자 데이터를 보호하고 신뢰할 수 있는 서비스를 제공하기 위한 필수 요소입니다. 이 장에서는 XSS 방지, CSRF 보호, 안전한 데이터 처리, HTTPS 강제 등 Blazor 애플리케이션의 보안 모범 사례를 학습합니다.

## 1. Cross-Site Scripting (XSS) 방지

### 1.1 입력 검증 및 Sanitization

```csharp
// Services/InputSanitizer.cs
public interface IInputSanitizer
{
    string SanitizeHtml(string input);
    string SanitizeForDisplay(string input);
    string SanitizeForAttribute(string input);
    bool ContainsDangerousContent(string input);
}

public class InputSanitizer : IInputSanitizer
{
    private readonly IHtmlSanitizer _htmlSanitizer;
    private readonly ILogger<InputSanitizer> _logger;
    
    // Dangerous patterns
    private readonly string[] _dangerousPatterns = new[]
    {
        "<script", "javascript:", "onload=", "onerror=", "onclick=",
        "onmouseover=", "onfocus=", "eval(", "expression(", "vbscript:",
        "<iframe", "<object", "<embed", "<applet"
    };
    
    public InputSanitizer(IHtmlSanitizer htmlSanitizer, ILogger<InputSanitizer> logger)
    {
        _htmlSanitizer = htmlSanitizer;
        _logger = logger;
    }
    
    public string SanitizeHtml(string input)
    {
        if (string.IsNullOrEmpty(input))
            return string.Empty;
        
        // Configure allowed tags and attributes
        var config = new HtmlSanitizerConfiguration
        {
            AllowedTags = new HashSet<string> { "p", "br", "strong", "em", "u", "a", "ul", "ol", "li" },
            AllowedAttributes = new Dictionary<string, HashSet<string>>
            {
                ["a"] = new HashSet<string> { "href", "title" }
            },
            AllowedSchemes = new HashSet<string> { "http", "https", "mailto" }
        };
        
        var sanitized = _htmlSanitizer.Sanitize(input, config);
        
        if (sanitized != input)
        {
            _logger.LogWarning("Potentially dangerous content was sanitized");
        }
        
        return sanitized;
    }
    
    public string SanitizeForDisplay(string input)
    {
        if (string.IsNullOrEmpty(input))
            return string.Empty;
        
        // HTML encode for safe display
        return System.Net.WebUtility.HtmlEncode(input);
    }
    
    public string SanitizeForAttribute(string input)
    {
        if (string.IsNullOrEmpty(input))
            return string.Empty;
        
        // Encode for use in HTML attributes
        return System.Net.WebUtility.HtmlEncode(input)
            .Replace("'", "&#39;")
            .Replace("\"", "&quot;");
    }
    
    public bool ContainsDangerousContent(string input)
    {
        if (string.IsNullOrEmpty(input))
            return false;
        
        var lowerInput = input.ToLowerInvariant();
        
        foreach (var pattern in _dangerousPatterns)
        {
            if (lowerInput.Contains(pattern))
            {
                _logger.LogWarning("Dangerous pattern detected: {Pattern}", pattern);
                return true;
            }
        }
        
        return false;
    }
}

// Components/SafeHtmlDisplay.razor
@inject IInputSanitizer Sanitizer

@if (TrustContent)
{
    <div>@((MarkupString)SanitizedContent)</div>
}
else
{
    <div>@DisplayContent</div>
}

@code {
    [Parameter] public string? Content { get; set; }
    [Parameter] public bool TrustContent { get; set; } = false;
    
    private string SanitizedContent => Sanitizer.SanitizeHtml(Content ?? "");
    private string DisplayContent => Sanitizer.SanitizeForDisplay(Content ?? "");
}
```

### 1.2 Content Security Policy (CSP)

```csharp
// Middleware/ContentSecurityPolicyMiddleware.cs
public class ContentSecurityPolicyMiddleware
{
    private readonly RequestDelegate _next;
    private readonly CspOptions _options;
    
    public ContentSecurityPolicyMiddleware(RequestDelegate next, CspOptions options)
    {
        _next = next;
        _options = options;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        // Generate nonce for inline scripts
        var nonce = GenerateNonce();
        context.Items["CSP-Nonce"] = nonce;
        
        // Build CSP header
        var csp = BuildCspHeader(nonce);
        
        // Add CSP header
        context.Response.Headers.Add("Content-Security-Policy", csp);
        
        // Add other security headers
        AddSecurityHeaders(context.Response.Headers);
        
        await _next(context);
    }
    
    private string BuildCspHeader(string nonce)
    {
        var directives = new List<string>
        {
            $"default-src {string.Join(" ", _options.DefaultSrc)}",
            $"script-src {string.Join(" ", _options.ScriptSrc)} 'nonce-{nonce}'",
            $"style-src {string.Join(" ", _options.StyleSrc)} 'nonce-{nonce}'",
            $"img-src {string.Join(" ", _options.ImgSrc)}",
            $"connect-src {string.Join(" ", _options.ConnectSrc)}",
            $"font-src {string.Join(" ", _options.FontSrc)}",
            $"object-src 'none'",
            $"media-src {string.Join(" ", _options.MediaSrc)}",
            $"frame-src {string.Join(" ", _options.FrameSrc)}",
            "frame-ancestors 'none'",
            "base-uri 'self'",
            "form-action 'self'",
            "upgrade-insecure-requests"
        };
        
        if (_options.ReportUri != null)
        {
            directives.Add($"report-uri {_options.ReportUri}");
        }
        
        return string.Join("; ", directives);
    }
    
    private void AddSecurityHeaders(IHeaderDictionary headers)
    {
        headers.Add("X-Content-Type-Options", "nosniff");
        headers.Add("X-Frame-Options", "DENY");
        headers.Add("X-XSS-Protection", "1; mode=block");
        headers.Add("Referrer-Policy", "strict-origin-when-cross-origin");
        headers.Add("Permissions-Policy", "geolocation=(), microphone=(), camera=()");
    }
    
    private string GenerateNonce()
    {
        var bytes = new byte[16];
        using var rng = RandomNumberGenerator.Create();
        rng.GetBytes(bytes);
        return Convert.ToBase64String(bytes);
    }
}

// Configuration/CspOptions.cs
public class CspOptions
{
    public List<string> DefaultSrc { get; set; } = new() { "'self'" };
    public List<string> ScriptSrc { get; set; } = new() { "'self'" };
    public List<string> StyleSrc { get; set; } = new() { "'self'" };
    public List<string> ImgSrc { get; set; } = new() { "'self'", "data:", "https:" };
    public List<string> ConnectSrc { get; set; } = new() { "'self'" };
    public List<string> FontSrc { get; set; } = new() { "'self'" };
    public List<string> MediaSrc { get; set; } = new() { "'self'" };
    public List<string> FrameSrc { get; set; } = new() { "'none'" };
    public string? ReportUri { get; set; }
}
```

## 2. Cross-Site Request Forgery (CSRF) 보호

### 2.1 Anti-Forgery 토큰

```csharp
// Services/AntiForgeryService.cs
public interface IAntiForgeryService
{
    Task<string> GetTokenAsync();
    Task<bool> ValidateTokenAsync(string token);
    void ConfigureHttpClient(HttpClient client);
}

public class AntiForgeryService : IAntiForgeryService
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<AntiForgeryService> _logger;
    private string? _cachedToken;
    private DateTime _tokenExpiry;
    
    public AntiForgeryService(HttpClient httpClient, ILogger<AntiForgeryService> logger)
    {
        _httpClient = httpClient;
        _logger = logger;
    }
    
    public async Task<string> GetTokenAsync()
    {
        // Check cache
        if (!string.IsNullOrEmpty(_cachedToken) && DateTime.UtcNow < _tokenExpiry)
        {
            return _cachedToken;
        }
        
        try
        {
            var response = await _httpClient.GetAsync("api/antiforgery/token");
            response.EnsureSuccessStatusCode();
            
            var tokenResponse = await response.Content.ReadFromJsonAsync<AntiForgeryTokenResponse>();
            
            if (tokenResponse != null)
            {
                _cachedToken = tokenResponse.Token;
                _tokenExpiry = DateTime.UtcNow.AddMinutes(tokenResponse.ExpiryMinutes - 1);
                return _cachedToken;
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to get anti-forgery token");
        }
        
        throw new SecurityException("Unable to obtain anti-forgery token");
    }
    
    public async Task<bool> ValidateTokenAsync(string token)
    {
        try
        {
            var response = await _httpClient.PostAsJsonAsync("api/antiforgery/validate", new { token });
            return response.IsSuccessStatusCode;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Token validation failed");
            return false;
        }
    }
    
    public void ConfigureHttpClient(HttpClient client)
    {
        client.DefaultRequestHeaders.Add("X-CSRF-TOKEN", _cachedToken ?? "");
    }
}

// Components/SecureForm.razor
@inject IAntiForgeryService AntiForgery

<EditForm Model="@Model" OnValidSubmit="HandleSubmit">
    <input type="hidden" name="__RequestVerificationToken" value="@antiForgeryToken" />
    
    @ChildContent
    
    <button type="submit" disabled="@isSubmitting">@SubmitText</button>
</EditForm>

@code {
    [Parameter] public object Model { get; set; } = new();
    [Parameter] public EventCallback<object> OnSubmit { get; set; }
    [Parameter] public RenderFragment? ChildContent { get; set; }
    [Parameter] public string SubmitText { get; set; } = "Submit";
    
    private string antiForgeryToken = "";
    private bool isSubmitting = false;
    
    protected override async Task OnInitializedAsync()
    {
        antiForgeryToken = await AntiForgery.GetTokenAsync();
    }
    
    private async Task HandleSubmit()
    {
        isSubmitting = true;
        
        try
        {
            // Validate token before submission
            var isValid = await AntiForgery.ValidateTokenAsync(antiForgeryToken);
            
            if (!isValid)
            {
                // Refresh token and retry
                antiForgeryToken = await AntiForgery.GetTokenAsync();
                isValid = await AntiForgery.ValidateTokenAsync(antiForgeryToken);
            }
            
            if (isValid)
            {
                await OnSubmit.InvokeAsync(Model);
            }
            else
            {
                throw new SecurityException("Invalid anti-forgery token");
            }
        }
        finally
        {
            isSubmitting = false;
        }
    }
}
```

## 3. 안전한 데이터 처리

### 3.1 암호화 서비스

```csharp
// Services/EncryptionService.cs
public interface IEncryptionService
{
    string Encrypt(string plainText);
    string Decrypt(string cipherText);
    byte[] EncryptData(byte[] data);
    byte[] DecryptData(byte[] encryptedData);
    string HashPassword(string password);
    bool VerifyPassword(string password, string hash);
}

public class EncryptionService : IEncryptionService
{
    private readonly IDataProtector _dataProtector;
    private readonly IConfiguration _configuration;
    
    public EncryptionService(
        IDataProtectionProvider dataProtectionProvider,
        IConfiguration configuration)
    {
        _dataProtector = dataProtectionProvider.CreateProtector("BlazorApp.Encryption");
        _configuration = configuration;
    }
    
    public string Encrypt(string plainText)
    {
        if (string.IsNullOrEmpty(plainText))
            return string.Empty;
        
        return _dataProtector.Protect(plainText);
    }
    
    public string Decrypt(string cipherText)
    {
        if (string.IsNullOrEmpty(cipherText))
            return string.Empty;
        
        try
        {
            return _dataProtector.Unprotect(cipherText);
        }
        catch (CryptographicException)
        {
            throw new SecurityException("Failed to decrypt data");
        }
    }
    
    public byte[] EncryptData(byte[] data)
    {
        using var aes = Aes.Create();
        aes.Key = GetEncryptionKey();
        aes.GenerateIV();
        
        using var encryptor = aes.CreateEncryptor();
        using var ms = new MemoryStream();
        
        // Write IV to the beginning
        ms.Write(aes.IV, 0, aes.IV.Length);
        
        using (var cs = new CryptoStream(ms, encryptor, CryptoStreamMode.Write))
        {
            cs.Write(data, 0, data.Length);
        }
        
        return ms.ToArray();
    }
    
    public byte[] DecryptData(byte[] encryptedData)
    {
        using var aes = Aes.Create();
        aes.Key = GetEncryptionKey();
        
        // Extract IV from the beginning
        var iv = new byte[aes.IV.Length];
        Array.Copy(encryptedData, 0, iv, 0, iv.Length);
        aes.IV = iv;
        
        using var decryptor = aes.CreateDecryptor();
        using var ms = new MemoryStream(encryptedData, iv.Length, encryptedData.Length - iv.Length);
        using var cs = new CryptoStream(ms, decryptor, CryptoStreamMode.Read);
        using var result = new MemoryStream();
        
        cs.CopyTo(result);
        return result.ToArray();
    }
    
    public string HashPassword(string password)
    {
        // Use Argon2 for password hashing
        var salt = new byte[16];
        using (var rng = RandomNumberGenerator.Create())
        {
            rng.GetBytes(salt);
        }
        
        var argon2 = new Argon2id(Encoding.UTF8.GetBytes(password))
        {
            Salt = salt,
            DegreeOfParallelism = 8,
            MemorySize = 65536,
            Iterations = 4
        };
        
        var hash = argon2.GetBytes(32);
        
        // Combine salt and hash
        var result = new byte[salt.Length + hash.Length];
        Array.Copy(salt, 0, result, 0, salt.Length);
        Array.Copy(hash, 0, result, salt.Length, hash.Length);
        
        return Convert.ToBase64String(result);
    }
    
    public bool VerifyPassword(string password, string hash)
    {
        try
        {
            var hashBytes = Convert.FromBase64String(hash);
            
            // Extract salt
            var salt = new byte[16];
            Array.Copy(hashBytes, 0, salt, 0, salt.Length);
            
            // Hash the input password with the same salt
            var argon2 = new Argon2id(Encoding.UTF8.GetBytes(password))
            {
                Salt = salt,
                DegreeOfParallelism = 8,
                MemorySize = 65536,
                Iterations = 4
            };
            
            var computedHash = argon2.GetBytes(32);
            
            // Compare hashes
            for (int i = 0; i < computedHash.Length; i++)
            {
                if (hashBytes[salt.Length + i] != computedHash[i])
                    return false;
            }
            
            return true;
        }
        catch
        {
            return false;
        }
    }
    
    private byte[] GetEncryptionKey()
    {
        var key = _configuration["Encryption:Key"];
        if (string.IsNullOrEmpty(key))
            throw new InvalidOperationException("Encryption key not configured");
        
        // Ensure key is 256 bits
        using var sha256 = SHA256.Create();
        return sha256.ComputeHash(Encoding.UTF8.GetBytes(key));
    }
}
```

### 3.2 안전한 데이터 저장

```csharp
// Services/SecureStorageService.cs
public interface ISecureStorageService
{
    Task SetSecureDataAsync<T>(string key, T data) where T : class;
    Task<T?> GetSecureDataAsync<T>(string key) where T : class;
    Task RemoveSecureDataAsync(string key);
    Task ClearAllSecureDataAsync();
}

public class SecureStorageService : ISecureStorageService
{
    private readonly ILocalStorageService _localStorage;
    private readonly IEncryptionService _encryption;
    private readonly ILogger<SecureStorageService> _logger;
    
    private const string SecurePrefix = "secure_";
    
    public SecureStorageService(
        ILocalStorageService localStorage,
        IEncryptionService encryption,
        ILogger<SecureStorageService> logger)
    {
        _localStorage = localStorage;
        _encryption = encryption;
        _logger = logger;
    }
    
    public async Task SetSecureDataAsync<T>(string key, T data) where T : class
    {
        try
        {
            var json = JsonSerializer.Serialize(data);
            var encrypted = _encryption.Encrypt(json);
            
            // Add metadata
            var secureData = new SecureDataWrapper
            {
                Data = encrypted,
                Type = typeof(T).FullName ?? "",
                Timestamp = DateTime.UtcNow,
                Version = "1.0"
            };
            
            await _localStorage.SetItemAsync($"{SecurePrefix}{key}", secureData);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to store secure data for key: {Key}", key);
            throw;
        }
    }
    
    public async Task<T?> GetSecureDataAsync<T>(string key) where T : class
    {
        try
        {
            var wrapper = await _localStorage.GetItemAsync<SecureDataWrapper>($"{SecurePrefix}{key}");
            
            if (wrapper == null)
                return null;
            
            // Check expiration
            if (wrapper.ExpiresAt.HasValue && wrapper.ExpiresAt.Value < DateTime.UtcNow)
            {
                await RemoveSecureDataAsync(key);
                return null;
            }
            
            var decrypted = _encryption.Decrypt(wrapper.Data);
            return JsonSerializer.Deserialize<T>(decrypted);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to retrieve secure data for key: {Key}", key);
            return null;
        }
    }
    
    public async Task RemoveSecureDataAsync(string key)
    {
        await _localStorage.RemoveItemAsync($"{SecurePrefix}{key}");
    }
    
    public async Task ClearAllSecureDataAsync()
    {
        var keys = await _localStorage.KeysAsync();
        var secureKeys = keys.Where(k => k.StartsWith(SecurePrefix));
        
        foreach (var key in secureKeys)
        {
            await _localStorage.RemoveItemAsync(key);
        }
    }
    
    private class SecureDataWrapper
    {
        public string Data { get; set; } = "";
        public string Type { get; set; } = "";
        public DateTime Timestamp { get; set; }
        public DateTime? ExpiresAt { get; set; }
        public string Version { get; set; } = "";
    }
}
```

## 4. 안전한 통신

### 4.1 HTTPS 강제 및 HSTS

```csharp
// Middleware/HttpsEnforcementMiddleware.cs
public class HttpsEnforcementMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<HttpsEnforcementMiddleware> _logger;
    private readonly HttpsEnforcementOptions _options;
    
    public HttpsEnforcementMiddleware(
        RequestDelegate next,
        ILogger<HttpsEnforcementMiddleware> logger,
        IOptions<HttpsEnforcementOptions> options)
    {
        _next = next;
        _logger = logger;
        _options = options.Value;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        // Check if already HTTPS
        if (context.Request.IsHttps)
        {
            // Add HSTS header
            AddHstsHeader(context.Response);
            await _next(context);
            return;
        }
        
        // Check if upgrade is allowed
        if (!_options.AllowHttpsRedirect)
        {
            context.Response.StatusCode = 403;
            await context.Response.WriteAsync("HTTPS Required");
            return;
        }
        
        // Redirect to HTTPS
        var httpsUrl = BuildHttpsUrl(context.Request);
        _logger.LogInformation("Redirecting to HTTPS: {Url}", httpsUrl);
        
        context.Response.StatusCode = 301;
        context.Response.Headers["Location"] = httpsUrl;
    }
    
    private void AddHstsHeader(HttpResponse response)
    {
        var hstsValue = $"max-age={_options.HstsMaxAge.TotalSeconds}";
        
        if (_options.IncludeSubDomains)
        {
            hstsValue += "; includeSubDomains";
        }
        
        if (_options.Preload)
        {
            hstsValue += "; preload";
        }
        
        response.Headers["Strict-Transport-Security"] = hstsValue;
    }
    
    private string BuildHttpsUrl(HttpRequest request)
    {
        var host = request.Host.Value;
        var pathBase = request.PathBase.Value;
        var path = request.Path.Value;
        var queryString = request.QueryString.Value;
        
        return $"https://{host}{pathBase}{path}{queryString}";
    }
}

public class HttpsEnforcementOptions
{
    public bool AllowHttpsRedirect { get; set; } = true;
    public TimeSpan HstsMaxAge { get; set; } = TimeSpan.FromDays(365);
    public bool IncludeSubDomains { get; set; } = true;
    public bool Preload { get; set; } = false;
}
```

### 4.2 API 보안

```csharp
// Services/SecureApiClient.cs
public class SecureApiClient
{
    private readonly HttpClient _httpClient;
    private readonly IEncryptionService _encryption;
    private readonly IAntiForgeryService _antiForgery;
    private readonly ILogger<SecureApiClient> _logger;
    
    public SecureApiClient(
        HttpClient httpClient,
        IEncryptionService encryption,
        IAntiForgeryService antiForgery,
        ILogger<SecureApiClient> logger)
    {
        _httpClient = httpClient;
        _encryption = encryption;
        _antiForgery = antiForgery;
        _logger = logger;
        
        ConfigureHttpClient();
    }
    
    private void ConfigureHttpClient()
    {
        // Force HTTPS
        if (_httpClient.BaseAddress?.Scheme != "https")
        {
            throw new InvalidOperationException("API must use HTTPS");
        }
        
        // Set security headers
        _httpClient.DefaultRequestHeaders.Add("X-Requested-With", "XMLHttpRequest");
        _httpClient.DefaultRequestHeaders.Add("X-Content-Type-Options", "nosniff");
        
        // Configure timeout
        _httpClient.Timeout = TimeSpan.FromSeconds(30);
    }
    
    public async Task<T?> SecureGetAsync<T>(string endpoint)
    {
        var request = new HttpRequestMessage(HttpMethod.Get, endpoint);
        
        // Add anti-forgery token
        var token = await _antiForgery.GetTokenAsync();
        request.Headers.Add("X-CSRF-Token", token);
        
        var response = await _httpClient.SendAsync(request);
        response.EnsureSuccessStatusCode();
        
        var content = await response.Content.ReadAsStringAsync();
        return JsonSerializer.Deserialize<T>(content);
    }
    
    public async Task<TResponse?> SecurePostAsync<TRequest, TResponse>(
        string endpoint, TRequest data, bool encrypt = false)
    {
        var json = JsonSerializer.Serialize(data);
        
        if (encrypt)
        {
            json = _encryption.Encrypt(json);
        }
        
        var content = new StringContent(json, Encoding.UTF8, "application/json");
        
        var request = new HttpRequestMessage(HttpMethod.Post, endpoint)
        {
            Content = content
        };
        
        // Add security headers
        var token = await _antiForgery.GetTokenAsync();
        request.Headers.Add("X-CSRF-Token", token);
        
        if (encrypt)
        {
            request.Headers.Add("X-Encrypted-Content", "true");
        }
        
        var response = await _httpClient.SendAsync(request);
        response.EnsureSuccessStatusCode();
        
        var responseContent = await response.Content.ReadAsStringAsync();
        
        if (response.Headers.Contains("X-Encrypted-Content"))
        {
            responseContent = _encryption.Decrypt(responseContent);
        }
        
        return JsonSerializer.Deserialize<TResponse>(responseContent);
    }
}
```

## 5. 감사 및 모니터링

### 5.1 보안 이벤트 로깅

```csharp
// Services/SecurityAuditService.cs
public interface ISecurityAuditService
{
    Task LogSecurityEventAsync(SecurityEvent securityEvent);
    Task<IEnumerable<SecurityEvent>> GetSecurityEventsAsync(SecurityEventQuery query);
    Task<SecurityMetrics> GetSecurityMetricsAsync(DateTime from, DateTime to);
}

public class SecurityAuditService : ISecurityAuditService
{
    private readonly ILogger<SecurityAuditService> _logger;
    private readonly IDbContext _dbContext;
    private readonly INotificationService _notifications;
    
    public SecurityAuditService(
        ILogger<SecurityAuditService> logger,
        IDbContext dbContext,
        INotificationService notifications)
    {
        _logger = logger;
        _dbContext = dbContext;
        _notifications = notifications;
    }
    
    public async Task LogSecurityEventAsync(SecurityEvent securityEvent)
    {
        // Enrich event with context
        securityEvent.Timestamp = DateTime.UtcNow;
        securityEvent.Id = Guid.NewGuid().ToString();
        
        // Log to structured logging
        _logger.LogWarning("Security Event: {EventType} - {Description}",
            securityEvent.EventType, securityEvent.Description);
        
        // Store in database
        await _dbContext.SecurityEvents.AddAsync(securityEvent);
        await _dbContext.SaveChangesAsync();
        
        // Check for critical events
        if (securityEvent.Severity == SecuritySeverity.Critical)
        {
            await NotifyCriticalEvent(securityEvent);
        }
        
        // Check for patterns
        await CheckSecurityPatterns(securityEvent);
    }
    
    private async Task CheckSecurityPatterns(SecurityEvent newEvent)
    {
        // Check for brute force attempts
        if (newEvent.EventType == "FailedLogin")
        {
            var recentFailures = await _dbContext.SecurityEvents
                .Where(e => e.EventType == "FailedLogin" &&
                           e.UserId == newEvent.UserId &&
                           e.Timestamp > DateTime.UtcNow.AddMinutes(-15))
                .CountAsync();
            
            if (recentFailures >= 5)
            {
                await LogSecurityEventAsync(new SecurityEvent
                {
                    EventType = "BruteForceDetected",
                    Description = $"Multiple failed login attempts for user {newEvent.UserId}",
                    Severity = SecuritySeverity.High,
                    UserId = newEvent.UserId,
                    IpAddress = newEvent.IpAddress
                });
            }
        }
        
        // Check for suspicious data access
        if (newEvent.EventType == "DataAccess")
        {
            var recentAccess = await _dbContext.SecurityEvents
                .Where(e => e.EventType == "DataAccess" &&
                           e.UserId == newEvent.UserId &&
                           e.Timestamp > DateTime.UtcNow.AddMinutes(-5))
                .CountAsync();
            
            if (recentAccess >= 100)
            {
                await LogSecurityEventAsync(new SecurityEvent
                {
                    EventType = "SuspiciousDataAccess",
                    Description = $"Unusual data access pattern detected for user {newEvent.UserId}",
                    Severity = SecuritySeverity.Medium,
                    UserId = newEvent.UserId
                });
            }
        }
    }
    
    private async Task NotifyCriticalEvent(SecurityEvent securityEvent)
    {
        await _notifications.SendSecurityAlertAsync(new SecurityAlert
        {
            Title = $"Critical Security Event: {securityEvent.EventType}",
            Message = securityEvent.Description,
            Severity = "Critical",
            Timestamp = securityEvent.Timestamp,
            Details = new
            {
                securityEvent.UserId,
                securityEvent.IpAddress,
                securityEvent.UserAgent,
                securityEvent.AdditionalData
            }
        });
    }
}

// Models/SecurityEvent.cs
public class SecurityEvent
{
    public string Id { get; set; } = "";
    public string EventType { get; set; } = "";
    public string Description { get; set; } = "";
    public SecuritySeverity Severity { get; set; }
    public DateTime Timestamp { get; set; }
    public string? UserId { get; set; }
    public string? IpAddress { get; set; }
    public string? UserAgent { get; set; }
    public Dictionary<string, object> AdditionalData { get; set; } = new();
}

public enum SecuritySeverity
{
    Low,
    Medium,
    High,
    Critical
}
```

### 5.2 보안 대시보드

```csharp
// Components/SecurityDashboard.razor
@page "/admin/security"
@attribute [Authorize(Roles = "Admin,SecurityAdmin")]
@inject ISecurityAuditService SecurityAudit

<h3>Security Dashboard</h3>

<div class="security-metrics">
    <div class="metric-card">
        <h4>Failed Login Attempts</h4>
        <div class="metric-value">@metrics.FailedLogins</div>
        <div class="metric-trend @(metrics.FailedLoginsTrend > 0 ? "up" : "down")">
            @metrics.FailedLoginsTrend%
        </div>
    </div>
    
    <div class="metric-card">
        <h4>Suspicious Activities</h4>
        <div class="metric-value">@metrics.SuspiciousActivities</div>
    </div>
    
    <div class="metric-card">
        <h4>Active Threats</h4>
        <div class="metric-value">@metrics.ActiveThreats</div>
    </div>
</div>

<div class="recent-events">
    <h4>Recent Security Events</h4>
    <table class="table">
        <thead>
            <tr>
                <th>Time</th>
                <th>Type</th>
                <th>Severity</th>
                <th>User</th>
                <th>IP Address</th>
                <th>Actions</th>
            </tr>
        </thead>
        <tbody>
            @foreach (var evt in recentEvents)
            {
                <tr class="@GetSeverityClass(evt.Severity)">
                    <td>@evt.Timestamp.ToString("yyyy-MM-dd HH:mm:ss")</td>
                    <td>@evt.EventType</td>
                    <td>
                        <span class="severity-badge severity-@evt.Severity.ToString().ToLower()">
                            @evt.Severity
                        </span>
                    </td>
                    <td>@(evt.UserId ?? "Anonymous")</td>
                    <td>@(evt.IpAddress ?? "N/A")</td>
                    <td>
                        <button @onclick="() => ViewDetails(evt)">Details</button>
                        @if (evt.Severity >= SecuritySeverity.High)
                        {
                            <button @onclick="() => BlockIp(evt.IpAddress)">Block IP</button>
                        }
                    </td>
                </tr>
            }
        </tbody>
    </table>
</div>

@code {
    private SecurityMetrics metrics = new();
    private List<SecurityEvent> recentEvents = new();
    private Timer? refreshTimer;
    
    protected override async Task OnInitializedAsync()
    {
        await LoadData();
        
        // Auto-refresh every 30 seconds
        refreshTimer = new Timer(async _ => 
        {
            await LoadData();
            await InvokeAsync(StateHasChanged);
        }, null, TimeSpan.FromSeconds(30), TimeSpan.FromSeconds(30));
    }
    
    private async Task LoadData()
    {
        var now = DateTime.UtcNow;
        var dayAgo = now.AddDays(-1);
        
        metrics = await SecurityAudit.GetSecurityMetricsAsync(dayAgo, now);
        
        recentEvents = (await SecurityAudit.GetSecurityEventsAsync(new SecurityEventQuery
        {
            From = now.AddHours(-1),
            To = now,
            PageSize = 50,
            OrderBy = "Timestamp",
            Descending = true
        })).ToList();
    }
    
    private string GetSeverityClass(SecuritySeverity severity)
    {
        return severity switch
        {
            SecuritySeverity.Critical => "table-danger",
            SecuritySeverity.High => "table-warning",
            SecuritySeverity.Medium => "table-info",
            _ => ""
        };
    }
    
    public void Dispose()
    {
        refreshTimer?.Dispose();
    }
}
```

## 마무리

Blazor 애플리케이션의 보안은 다층 방어 접근 방식을 통해 구현해야 합니다. XSS 방지, CSRF 보호, 안전한 데이터 처리, HTTPS 강제, 로깅 및 모니터링 등 모든 층에서 보안을 고려해야 합니다. 정기적인 보안 감사와 취약점 스캔을 통해 애플리케이션의 보안을 지속적으로 개선하는 것이 중요합니다.