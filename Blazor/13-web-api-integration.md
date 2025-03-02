# Web API 통합

## 개요

Blazor 애플리케이션에서 Web API와의 효율적인 통합은 현대적인 웹 애플리케이션 개발의 핵심입니다. 이 장에서는 HttpClient 고급 사용법, RESTful API 패턴, 인터셉터, 오류 처리 등을 학습합니다.

## 1. HttpClient 고급 구성

### 1.1 타입 안전 HTTP 클라이언트

```csharp
// IApiClient.cs
public interface IApiClient
{
    Task<T?> GetAsync<T>(string endpoint);
    Task<TResponse?> PostAsync<TRequest, TResponse>(string endpoint, TRequest data);
    Task<TResponse?> PutAsync<TRequest, TResponse>(string endpoint, TRequest data);
    Task<bool> DeleteAsync(string endpoint);
    Task<ApiResult<T>> GetWithResultAsync<T>(string endpoint);
}

// ApiClient.cs
public class ApiClient : IApiClient
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<ApiClient> _logger;
    private readonly JsonSerializerOptions _jsonOptions;
    
    public ApiClient(HttpClient httpClient, ILogger<ApiClient> logger)
    {
        _httpClient = httpClient;
        _logger = logger;
        _jsonOptions = new JsonSerializerOptions
        {
            PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
            DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull,
            Converters = { new JsonStringEnumConverter() }
        };
    }
    
    public async Task<T?> GetAsync<T>(string endpoint)
    {
        try
        {
            var response = await _httpClient.GetAsync(endpoint);
            response.EnsureSuccessStatusCode();
            
            var json = await response.Content.ReadAsStringAsync();
            return JsonSerializer.Deserialize<T>(json, _jsonOptions);
        }
        catch (HttpRequestException ex)
        {
            _logger.LogError(ex, "HTTP request failed for {Endpoint}", endpoint);
            throw;
        }
        catch (TaskCanceledException ex)
        {
            _logger.LogError(ex, "Request timeout for {Endpoint}", endpoint);
            throw new TimeoutException($"Request to {endpoint} timed out", ex);
        }
    }
    
    public async Task<TResponse?> PostAsync<TRequest, TResponse>(string endpoint, TRequest data)
    {
        try
        {
            var json = JsonSerializer.Serialize(data, _jsonOptions);
            var content = new StringContent(json, Encoding.UTF8, "application/json");
            
            var response = await _httpClient.PostAsync(endpoint, content);
            response.EnsureSuccessStatusCode();
            
            var responseJson = await response.Content.ReadAsStringAsync();
            return JsonSerializer.Deserialize<TResponse>(responseJson, _jsonOptions);
        }
        catch (HttpRequestException ex)
        {
            _logger.LogError(ex, "POST request failed for {Endpoint}", endpoint);
            throw;
        }
    }
    
    public async Task<ApiResult<T>> GetWithResultAsync<T>(string endpoint)
    {
        try
        {
            var response = await _httpClient.GetAsync(endpoint);
            
            if (response.IsSuccessStatusCode)
            {
                var json = await response.Content.ReadAsStringAsync();
                var data = JsonSerializer.Deserialize<T>(json, _jsonOptions);
                return ApiResult<T>.Success(data);
            }
            
            var error = await ExtractError(response);
            return ApiResult<T>.Failure(error);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Request failed for {Endpoint}", endpoint);
            return ApiResult<T>.Failure(ex.Message);
        }
    }
    
    private async Task<string> ExtractError(HttpResponseMessage response)
    {
        try
        {
            var content = await response.Content.ReadAsStringAsync();
            var error = JsonSerializer.Deserialize<ApiError>(content, _jsonOptions);
            return error?.Message ?? response.ReasonPhrase ?? "Unknown error";
        }
        catch
        {
            return response.ReasonPhrase ?? "Unknown error";
        }
    }
}

// ApiResult.cs
public class ApiResult<T>
{
    public bool IsSuccess { get; }
    public T? Data { get; }
    public string? Error { get; }
    
    private ApiResult(bool isSuccess, T? data, string? error)
    {
        IsSuccess = isSuccess;
        Data = data;
        Error = error;
    }
    
    public static ApiResult<T> Success(T? data) => new(true, data, null);
    public static ApiResult<T> Failure(string error) => new(false, default, error);
}
```

### 1.2 타입별 HTTP 클라이언트

```csharp
// Program.cs
builder.Services.AddHttpClient<IUserService, UserService>(client =>
{
    client.BaseAddress = new Uri("https://api.example.com/");
    client.DefaultRequestHeaders.Add("Accept", "application/json");
    client.Timeout = TimeSpan.FromSeconds(30);
});

builder.Services.AddHttpClient<IProductService, ProductService>(client =>
{
    client.BaseAddress = new Uri("https://api.example.com/");
})
.AddPolicyHandler(GetRetryPolicy())
.AddPolicyHandler(GetCircuitBreakerPolicy());

// Polly 정책
static IAsyncPolicy<HttpResponseMessage> GetRetryPolicy()
{
    return HttpPolicyExtensions
        .HandleTransientHttpError()
        .OrResult(msg => !msg.IsSuccessStatusCode)
        .WaitAndRetryAsync(
            3,
            retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)),
            onRetry: (outcome, timespan, retryCount, context) =>
            {
                Console.WriteLine($"Retry {retryCount} after {timespan} seconds");
            });
}

static IAsyncPolicy<HttpResponseMessage> GetCircuitBreakerPolicy()
{
    return HttpPolicyExtensions
        .HandleTransientHttpError()
        .CircuitBreakerAsync(
            5,
            TimeSpan.FromSeconds(30),
            onBreak: (result, timespan) => Console.WriteLine($"Circuit breaker opened for {timespan}"),
            onReset: () => Console.WriteLine("Circuit breaker reset"));
}
```

## 2. RESTful API 패턴

### 2.1 리소스 기반 서비스

```csharp
// IResourceService.cs
public interface IResourceService<TEntity, TKey> where TEntity : class
{
    Task<IEnumerable<TEntity>> GetAllAsync(QueryParameters? parameters = null);
    Task<TEntity?> GetByIdAsync(TKey id);
    Task<TEntity> CreateAsync(TEntity entity);
    Task<TEntity> UpdateAsync(TKey id, TEntity entity);
    Task<bool> DeleteAsync(TKey id);
    Task<PagedResult<TEntity>> GetPagedAsync(PagingParameters parameters);
}

// BaseResourceService.cs
public abstract class BaseResourceService<TEntity, TKey> : IResourceService<TEntity, TKey> 
    where TEntity : class
{
    protected readonly HttpClient HttpClient;
    protected readonly ILogger Logger;
    protected readonly string ResourceEndpoint;
    
    protected BaseResourceService(HttpClient httpClient, ILogger logger, string resourceEndpoint)
    {
        HttpClient = httpClient;
        Logger = logger;
        ResourceEndpoint = resourceEndpoint;
    }
    
    public virtual async Task<IEnumerable<TEntity>> GetAllAsync(QueryParameters? parameters = null)
    {
        var query = parameters?.ToQueryString() ?? "";
        var response = await HttpClient.GetFromJsonAsync<List<TEntity>>(
            $"{ResourceEndpoint}{query}");
        return response ?? Enumerable.Empty<TEntity>();
    }
    
    public virtual async Task<TEntity?> GetByIdAsync(TKey id)
    {
        try
        {
            return await HttpClient.GetFromJsonAsync<TEntity>($"{ResourceEndpoint}/{id}");
        }
        catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.NotFound)
        {
            return null;
        }
    }
    
    public virtual async Task<TEntity> CreateAsync(TEntity entity)
    {
        var response = await HttpClient.PostAsJsonAsync(ResourceEndpoint, entity);
        response.EnsureSuccessStatusCode();
        
        var created = await response.Content.ReadFromJsonAsync<TEntity>();
        return created ?? throw new InvalidOperationException("Failed to create resource");
    }
    
    public virtual async Task<TEntity> UpdateAsync(TKey id, TEntity entity)
    {
        var response = await HttpClient.PutAsJsonAsync($"{ResourceEndpoint}/{id}", entity);
        response.EnsureSuccessStatusCode();
        
        var updated = await response.Content.ReadFromJsonAsync<TEntity>();
        return updated ?? throw new InvalidOperationException("Failed to update resource");
    }
    
    public virtual async Task<bool> DeleteAsync(TKey id)
    {
        var response = await HttpClient.DeleteAsync($"{ResourceEndpoint}/{id}");
        return response.IsSuccessStatusCode;
    }
    
    public virtual async Task<PagedResult<TEntity>> GetPagedAsync(PagingParameters parameters)
    {
        var query = parameters.ToQueryString();
        var response = await HttpClient.GetAsync($"{ResourceEndpoint}/paged{query}");
        response.EnsureSuccessStatusCode();
        
        // Extract pagination headers
        var totalCount = int.Parse(
            response.Headers.GetValues("X-Total-Count").FirstOrDefault() ?? "0");
        
        var items = await response.Content.ReadFromJsonAsync<List<TEntity>>();
        
        return new PagedResult<TEntity>
        {
            Items = items ?? new List<TEntity>(),
            TotalCount = totalCount,
            PageNumber = parameters.PageNumber,
            PageSize = parameters.PageSize
        };
    }
}

// UserService.cs
public class UserService : BaseResourceService<User, int>
{
    public UserService(HttpClient httpClient, ILogger<UserService> logger)
        : base(httpClient, logger, "api/users")
    {
    }
    
    public async Task<User?> GetByEmailAsync(string email)
    {
        return await HttpClient.GetFromJsonAsync<User>(
            $"{ResourceEndpoint}/by-email/{Uri.EscapeDataString(email)}");
    }
    
    public async Task<bool> CheckEmailAvailabilityAsync(string email)
    {
        var response = await HttpClient.GetAsync(
            $"{ResourceEndpoint}/check-email/{Uri.EscapeDataString(email)}");
        return response.IsSuccessStatusCode;
    }
}
```

### 2.2 OData 지원

```csharp
// ODataService.cs
public class ODataService<T> where T : class
{
    private readonly HttpClient _httpClient;
    private readonly string _entitySet;
    
    public ODataService(HttpClient httpClient, string entitySet)
    {
        _httpClient = httpClient;
        _entitySet = entitySet;
    }
    
    public async Task<ODataResponse<T>> QueryAsync(ODataQuery query)
    {
        var queryString = query.Build();
        var response = await _httpClient.GetFromJsonAsync<ODataResponse<T>>(
            $"{_entitySet}{queryString}");
        
        return response ?? new ODataResponse<T>();
    }
}

// ODataQuery.cs
public class ODataQuery
{
    private readonly List<string> _queryParts = new();
    
    public ODataQuery Select(params string[] fields)
    {
        _queryParts.Add($"$select={string.Join(",", fields)}");
        return this;
    }
    
    public ODataQuery Filter(string filter)
    {
        _queryParts.Add($"$filter={Uri.EscapeDataString(filter)}");
        return this;
    }
    
    public ODataQuery OrderBy(string field, bool descending = false)
    {
        var order = descending ? "desc" : "asc";
        _queryParts.Add($"$orderby={field} {order}");
        return this;
    }
    
    public ODataQuery Skip(int count)
    {
        _queryParts.Add($"$skip={count}");
        return this;
    }
    
    public ODataQuery Top(int count)
    {
        _queryParts.Add($"$top={count}");
        return this;
    }
    
    public ODataQuery Expand(params string[] navigations)
    {
        _queryParts.Add($"$expand={string.Join(",", navigations)}");
        return this;
    }
    
    public ODataQuery Count()
    {
        _queryParts.Add("$count=true");
        return this;
    }
    
    public string Build()
    {
        return _queryParts.Any() ? "?" + string.Join("&", _queryParts) : "";
    }
}

// Usage
var query = new ODataQuery()
    .Select("id", "name", "email")
    .Filter("startswith(name, 'John')")
    .OrderBy("createdDate", descending: true)
    .Skip(0)
    .Top(10)
    .Expand("orders");
```

## 3. HTTP 인터셉터

### 3.1 요청/응답 인터셉터

```csharp
// AuthorizationMessageHandler.cs
public class AuthorizationMessageHandler : DelegatingHandler
{
    private readonly IAccessTokenProvider _tokenProvider;
    private readonly NavigationManager _navigation;
    
    public AuthorizationMessageHandler(
        IAccessTokenProvider tokenProvider,
        NavigationManager navigation)
    {
        _tokenProvider = tokenProvider;
        _navigation = navigation;
    }
    
    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request,
        CancellationToken cancellationToken)
    {
        // Add authorization header
        var tokenResult = await _tokenProvider.RequestAccessToken();
        
        if (tokenResult.TryGetToken(out var token))
        {
            request.Headers.Authorization = 
                new AuthenticationHeaderValue("Bearer", token.Value);
        }
        
        // Log request
        Console.WriteLine($"Request: {request.Method} {request.RequestUri}");
        
        var response = await base.SendAsync(request, cancellationToken);
        
        // Handle 401 responses
        if (response.StatusCode == HttpStatusCode.Unauthorized)
        {
            _navigation.NavigateTo("authentication/login");
        }
        
        return response;
    }
}

// LoggingHandler.cs
public class LoggingHandler : DelegatingHandler
{
    private readonly ILogger<LoggingHandler> _logger;
    
    public LoggingHandler(ILogger<LoggingHandler> logger)
    {
        _logger = logger;
    }
    
    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request,
        CancellationToken cancellationToken)
    {
        var requestId = Guid.NewGuid().ToString("N");
        
        _logger.LogInformation(
            "Sending request [{RequestId}]: {Method} {Uri}",
            requestId, request.Method, request.RequestUri);
        
        var stopwatch = Stopwatch.StartNew();
        
        try
        {
            var response = await base.SendAsync(request, cancellationToken);
            
            stopwatch.Stop();
            
            _logger.LogInformation(
                "Received response [{RequestId}]: {StatusCode} in {ElapsedMs}ms",
                requestId, response.StatusCode, stopwatch.ElapsedMilliseconds);
            
            return response;
        }
        catch (Exception ex)
        {
            stopwatch.Stop();
            
            _logger.LogError(ex,
                "Request failed [{RequestId}] after {ElapsedMs}ms",
                requestId, stopwatch.ElapsedMilliseconds);
            
            throw;
        }
    }
}
```

### 3.2 캐싱 인터셉터

```csharp
// CachingHandler.cs
public class CachingHandler : DelegatingHandler
{
    private readonly IMemoryCache _cache;
    private readonly ICachePolicy _cachePolicy;
    
    public CachingHandler(IMemoryCache cache, ICachePolicy cachePolicy)
    {
        _cache = cache;
        _cachePolicy = cachePolicy;
    }
    
    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request,
        CancellationToken cancellationToken)
    {
        // Only cache GET requests
        if (request.Method != HttpMethod.Get)
        {
            return await base.SendAsync(request, cancellationToken);
        }
        
        var cacheKey = GetCacheKey(request);
        
        // Try to get from cache
        if (_cache.TryGetValue<CachedResponse>(cacheKey, out var cachedResponse))
        {
            return CreateResponseFromCache(cachedResponse);
        }
        
        // Send request
        var response = await base.SendAsync(request, cancellationToken);
        
        // Cache successful responses
        if (response.IsSuccessStatusCode && _cachePolicy.ShouldCache(request, response))
        {
            await CacheResponse(cacheKey, response);
        }
        
        return response;
    }
    
    private string GetCacheKey(HttpRequestMessage request)
    {
        return $"{request.Method}:{request.RequestUri}";
    }
    
    private HttpResponseMessage CreateResponseFromCache(CachedResponse cached)
    {
        var response = new HttpResponseMessage(cached.StatusCode)
        {
            Content = new StringContent(cached.Content, Encoding.UTF8, cached.MediaType),
            ReasonPhrase = cached.ReasonPhrase
        };
        
        foreach (var header in cached.Headers)
        {
            response.Headers.TryAddWithoutValidation(header.Key, header.Value);
        }
        
        return response;
    }
    
    private async Task CacheResponse(string key, HttpResponseMessage response)
    {
        var content = await response.Content.ReadAsStringAsync();
        var cached = new CachedResponse
        {
            StatusCode = response.StatusCode,
            ReasonPhrase = response.ReasonPhrase,
            Content = content,
            MediaType = response.Content.Headers.ContentType?.MediaType ?? "application/json",
            Headers = response.Headers.ToDictionary(h => h.Key, h => h.Value.ToList())
        };
        
        var cacheOptions = _cachePolicy.GetCacheOptions(response);
        _cache.Set(key, cached, cacheOptions);
    }
}
```

## 4. 오류 처리와 재시도

### 4.1 전역 오류 처리

```csharp
// GlobalErrorHandler.cs
public class GlobalErrorHandler : DelegatingHandler
{
    private readonly ILogger<GlobalErrorHandler> _logger;
    private readonly INotificationService _notifications;
    
    public GlobalErrorHandler(
        ILogger<GlobalErrorHandler> logger,
        INotificationService notifications)
    {
        _logger = logger;
        _notifications = notifications;
    }
    
    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request,
        CancellationToken cancellationToken)
    {
        try
        {
            var response = await base.SendAsync(request, cancellationToken);
            
            if (!response.IsSuccessStatusCode)
            {
                await HandleErrorResponse(request, response);
            }
            
            return response;
        }
        catch (TaskCanceledException ex)
        {
            _logger.LogWarning(ex, "Request timeout: {Uri}", request.RequestUri);
            await _notifications.ShowError("Request timed out. Please try again.");
            throw;
        }
        catch (HttpRequestException ex)
        {
            _logger.LogError(ex, "HTTP error: {Uri}", request.RequestUri);
            await _notifications.ShowError("Network error occurred. Please check your connection.");
            throw;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unexpected error: {Uri}", request.RequestUri);
            await _notifications.ShowError("An unexpected error occurred.");
            throw;
        }
    }
    
    private async Task HandleErrorResponse(HttpRequestMessage request, HttpResponseMessage response)
    {
        var content = await response.Content.ReadAsStringAsync();
        
        var errorMessage = response.StatusCode switch
        {
            HttpStatusCode.BadRequest => ExtractValidationErrors(content),
            HttpStatusCode.Unauthorized => "Authentication required",
            HttpStatusCode.Forbidden => "You don't have permission to access this resource",
            HttpStatusCode.NotFound => "Resource not found",
            HttpStatusCode.Conflict => "Operation conflicts with existing data",
            HttpStatusCode.InternalServerError => "Server error occurred",
            _ => $"Request failed with status {response.StatusCode}"
        };
        
        _logger.LogWarning(
            "API error: {Method} {Uri} returned {StatusCode}. Message: {Message}",
            request.Method, request.RequestUri, response.StatusCode, errorMessage);
        
        await _notifications.ShowError(errorMessage);
    }
    
    private string ExtractValidationErrors(string content)
    {
        try
        {
            var errors = JsonSerializer.Deserialize<ValidationProblemDetails>(content);
            if (errors?.Errors != null)
            {
                var messages = errors.Errors
                    .SelectMany(e => e.Value)
                    .Take(3);
                return string.Join(". ", messages);
            }
        }
        catch { }
        
        return "Validation error occurred";
    }
}
```

### 4.2 재시도 전략

```csharp
// RetryService.cs
public class RetryService
{
    private readonly ILogger<RetryService> _logger;
    
    public RetryService(ILogger<RetryService> logger)
    {
        _logger = logger;
    }
    
    public async Task<T> ExecuteWithRetryAsync<T>(
        Func<Task<T>> operation,
        RetryPolicy policy,
        CancellationToken cancellationToken = default)
    {
        var attempt = 0;
        var exceptions = new List<Exception>();
        
        while (attempt < policy.MaxAttempts)
        {
            try
            {
                attempt++;
                return await operation();
            }
            catch (Exception ex) when (policy.ShouldRetry(ex) && attempt < policy.MaxAttempts)
            {
                exceptions.Add(ex);
                
                var delay = policy.GetDelay(attempt);
                _logger.LogWarning(
                    "Operation failed on attempt {Attempt}/{MaxAttempts}. Retrying in {Delay}ms",
                    attempt, policy.MaxAttempts, delay.TotalMilliseconds);
                
                await Task.Delay(delay, cancellationToken);
            }
        }
        
        throw new AggregateException(
            $"Operation failed after {policy.MaxAttempts} attempts",
            exceptions);
    }
}

// RetryPolicy.cs
public class RetryPolicy
{
    public int MaxAttempts { get; set; } = 3;
    public TimeSpan BaseDelay { get; set; } = TimeSpan.FromSeconds(1);
    public RetryStrategy Strategy { get; set; } = RetryStrategy.ExponentialBackoff;
    public Func<Exception, bool> ShouldRetry { get; set; } = _ => true;
    
    public TimeSpan GetDelay(int attempt)
    {
        return Strategy switch
        {
            RetryStrategy.Fixed => BaseDelay,
            RetryStrategy.Linear => TimeSpan.FromMilliseconds(BaseDelay.TotalMilliseconds * attempt),
            RetryStrategy.ExponentialBackoff => TimeSpan.FromMilliseconds(
                BaseDelay.TotalMilliseconds * Math.Pow(2, attempt - 1)),
            _ => BaseDelay
        };
    }
    
    public static RetryPolicy Default => new()
    {
        MaxAttempts = 3,
        BaseDelay = TimeSpan.FromSeconds(1),
        Strategy = RetryStrategy.ExponentialBackoff,
        ShouldRetry = ex => ex is HttpRequestException or TaskCanceledException
    };
}

public enum RetryStrategy
{
    Fixed,
    Linear,
    ExponentialBackoff
}
```

## 5. 파일 업로드/다운로드

### 5.1 파일 업로드

```csharp
// FileUploadService.cs
public class FileUploadService
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<FileUploadService> _logger;
    
    public FileUploadService(HttpClient httpClient, ILogger<FileUploadService> logger)
    {
        _httpClient = httpClient;
        _logger = logger;
    }
    
    public async Task<FileUploadResult> UploadAsync(
        IBrowserFile file,
        IProgress<int>? progress = null,
        CancellationToken cancellationToken = default)
    {
        try
        {
            using var content = new MultipartFormDataContent();
            using var fileContent = new StreamContent(file.OpenReadStream(maxAllowedSize: 10_000_000));
            
            fileContent.Headers.ContentType = new MediaTypeHeaderValue(file.ContentType);
            content.Add(fileContent, "file", file.Name);
            
            // Add metadata
            content.Add(new StringContent(file.Size.ToString()), "size");
            content.Add(new StringContent(file.ContentType), "contentType");
            
            // Upload with progress
            var response = await UploadWithProgressAsync(
                "api/files/upload",
                content,
                progress,
                cancellationToken);
            
            response.EnsureSuccessStatusCode();
            
            var result = await response.Content.ReadFromJsonAsync<FileUploadResult>();
            return result ?? throw new InvalidOperationException("Invalid response");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "File upload failed: {FileName}", file.Name);
            throw;
        }
    }
    
    private async Task<HttpResponseMessage> UploadWithProgressAsync(
        string uri,
        HttpContent content,
        IProgress<int>? progress,
        CancellationToken cancellationToken)
    {
        using var request = new HttpRequestMessage(HttpMethod.Post, uri) { Content = content };
        
        if (progress != null)
        {
            var totalBytes = content.Headers.ContentLength ?? 0;
            var uploadedBytes = 0L;
            
            // Create progress tracking stream
            var progressContent = new ProgressStreamContent(content, async (bytes) =>
            {
                uploadedBytes += bytes;
                var percentage = totalBytes > 0 
                    ? (int)((uploadedBytes * 100) / totalBytes)
                    : 0;
                progress.Report(percentage);
            });
            
            request.Content = progressContent;
        }
        
        return await _httpClient.SendAsync(request, cancellationToken);
    }
}

// FileUploadComponent.razor
@page "/upload"

<h3>File Upload</h3>

<InputFile OnChange="HandleFileSelected" multiple />

@if (uploads.Any())
{
    <div class="upload-list">
        @foreach (var upload in uploads)
        {
            <div class="upload-item">
                <span>@upload.FileName</span>
                <div class="progress">
                    <div class="progress-bar" style="width: @upload.Progress%"></div>
                </div>
                <span>@upload.Status</span>
            </div>
        }
    </div>
}

@code {
    private List<UploadInfo> uploads = new();
    
    private async Task HandleFileSelected(InputFileChangeEventArgs e)
    {
        var files = e.GetMultipleFiles();
        
        foreach (var file in files)
        {
            var upload = new UploadInfo { FileName = file.Name };
            uploads.Add(upload);
            
            _ = UploadFileAsync(file, upload);
        }
    }
    
    private async Task UploadFileAsync(IBrowserFile file, UploadInfo upload)
    {
        try
        {
            var progress = new Progress<int>(percent =>
            {
                upload.Progress = percent;
                InvokeAsync(StateHasChanged);
            });
            
            upload.Status = "Uploading";
            var result = await FileUploadService.UploadAsync(file, progress);
            
            upload.Status = "Completed";
            upload.FileId = result.FileId;
        }
        catch (Exception ex)
        {
            upload.Status = $"Failed: {ex.Message}";
        }
        
        await InvokeAsync(StateHasChanged);
    }
}
```

### 5.2 파일 다운로드

```csharp
// FileDownloadService.cs
public class FileDownloadService
{
    private readonly HttpClient _httpClient;
    private readonly IJSRuntime _js;
    
    public FileDownloadService(HttpClient httpClient, IJSRuntime js)
    {
        _httpClient = httpClient;
        _js = js;
    }
    
    public async Task DownloadFileAsync(
        string fileId,
        string fileName,
        IProgress<int>? progress = null)
    {
        var response = await _httpClient.GetAsync(
            $"api/files/{fileId}/download",
            HttpCompletionOption.ResponseHeadersRead);
        
        response.EnsureSuccessStatusCode();
        
        var totalBytes = response.Content.Headers.ContentLength ?? 0;
        var buffer = new byte[8192];
        var downloadedBytes = 0L;
        
        using var stream = await response.Content.ReadAsStreamAsync();
        using var memoryStream = new MemoryStream();
        
        int bytesRead;
        while ((bytesRead = await stream.ReadAsync(buffer, 0, buffer.Length)) > 0)
        {
            await memoryStream.WriteAsync(buffer, 0, bytesRead);
            downloadedBytes += bytesRead;
            
            if (progress != null && totalBytes > 0)
            {
                var percentage = (int)((downloadedBytes * 100) / totalBytes);
                progress.Report(percentage);
            }
        }
        
        var fileBytes = memoryStream.ToArray();
        await SaveFileAsync(fileBytes, fileName, response.Content.Headers.ContentType?.MediaType);
    }
    
    private async Task SaveFileAsync(byte[] fileBytes, string fileName, string? contentType)
    {
        // Use JavaScript interop to trigger browser download
        await _js.InvokeVoidAsync(
            "downloadFileFromBytes",
            fileName,
            contentType ?? "application/octet-stream",
            fileBytes);
    }
}

// wwwroot/js/download.js
window.downloadFileFromBytes = (fileName, contentType, bytes) => {
    const blob = new Blob([bytes], { type: contentType });
    const url = URL.createObjectURL(blob);
    
    const link = document.createElement('a');
    link.href = url;
    link.download = fileName;
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
    
    URL.revokeObjectURL(url);
};
```

## 마무리

Blazor에서 Web API와의 효율적인 통합은 현대적인 웹 애플리케이션 개발의 핵심입니다. 타입 안전 HTTP 클라이언트, RESTful 패턴, 인터셉터를 통한 공통 처리, 효과적인 오류 처리와 재시도 전략을 구현하면 견고하고 유지보수가 쉬운 애플리케이션을 구축할 수 있습니다. 파일 업로드/다운로드와 같은 고급 시나리오도 적절한 패턴을 적용하면 효과적으로 처리할 수 있습니다.