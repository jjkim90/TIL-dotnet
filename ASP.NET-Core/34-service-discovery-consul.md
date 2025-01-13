# Service Discovery - Consul

## Service Discovery 소개

Service Discovery는 마이크로서비스 아키텍처에서 서비스들이 서로를 동적으로 찾고 통신할 수 있게 하는 메커니즘입니다. 서비스의 위치(IP, 포트)가 동적으로 변경되는 클라우드 환경에서 필수적인 패턴입니다.

### Service Discovery의 필요성

1. **동적 환경**: 컨테이너와 클라우드 환경에서 서비스 인스턴스가 동적으로 생성/삭제
2. **자동 확장**: 부하에 따라 서비스 인스턴스가 자동으로 증감
3. **장애 복구**: 실패한 인스턴스를 감지하고 트래픽을 건강한 인스턴스로 라우팅
4. **구성 관리**: 서비스 위치 정보를 하드코딩하지 않고 동적으로 관리

### Service Discovery 패턴

1. **클라이언트 사이드 디스커버리**: 클라이언트가 직접 서비스 레지스트리를 조회
2. **서버 사이드 디스커버리**: 로드 밸런서가 서비스 레지스트리를 조회

## Consul 소개

Consul은 HashiCorp에서 개발한 서비스 메시 솔루션으로, Service Discovery, 헬스 체킹, Key/Value 저장소, 다중 데이터센터 지원 등의 기능을 제공합니다.

### Consul의 주요 기능

- **Service Discovery**: 서비스 등록과 검색
- **Health Checking**: 서비스 상태 모니터링
- **Key/Value Store**: 동적 구성 저장
- **Secure Service Communication**: mTLS를 통한 서비스 간 보안 통신
- **Multi Datacenter**: 여러 데이터센터 간 서비스 검색

## Consul 설치 및 실행

### Docker를 사용한 Consul 실행

```bash
# 개발 모드로 Consul 실행
docker run -d --name=consul -p 8500:8500 -p 8600:8600/udp consul:latest agent -dev -ui -client=0.0.0.0

# Consul UI 접속: http://localhost:8500
```

### Docker Compose 설정

```yaml
# docker-compose.yml
version: '3.8'

services:
  consul:
    image: consul:latest
    container_name: consul
    ports:
      - "8500:8500"
      - "8600:8600/udp"
    command: agent -dev -ui -client=0.0.0.0
    networks:
      - microservices
    
  consul-server:
    image: consul:latest
    container_name: consul-server
    ports:
      - "8501:8500"
    command: agent -server -bootstrap-expect=1 -ui -client=0.0.0.0
    networks:
      - microservices
      
networks:
  microservices:
    driver: bridge
```

## ASP.NET Core와 Consul 통합

### NuGet 패키지 설치

```bash
dotnet add package Consul
dotnet add package Consul.AspNetCore
```

### 서비스 등록

```csharp
// Configuration/ConsulConfig.cs
public class ConsulConfig
{
    public string ServiceId { get; set; }
    public string ServiceName { get; set; }
    public string ServiceHost { get; set; }
    public int ServicePort { get; set; }
    public string ConsulAddress { get; set; }
    public string HealthCheckEndpoint { get; set; }
    public int HealthCheckInterval { get; set; } = 10;
    public int HealthCheckTimeout { get; set; } = 5;
    public int DeregisterCriticalServiceAfter { get; set; } = 30;
}

// Extensions/ConsulExtensions.cs
public static class ConsulExtensions
{
    public static IApplicationBuilder UseConsul(this IApplicationBuilder app, IHostApplicationLifetime lifetime)
    {
        var consulConfig = app.ApplicationServices.GetRequiredService<IOptions<ConsulConfig>>().Value;
        var consulClient = app.ApplicationServices.GetRequiredService<IConsulClient>();
        var logger = app.ApplicationServices.GetRequiredService<ILogger<IApplicationBuilder>>();
        
        var registration = new AgentServiceRegistration
        {
            ID = consulConfig.ServiceId,
            Name = consulConfig.ServiceName,
            Address = consulConfig.ServiceHost,
            Port = consulConfig.ServicePort,
            Tags = new[] { "api", "microservice", Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT") },
            Check = new AgentServiceCheck
            {
                HTTP = $"http://{consulConfig.ServiceHost}:{consulConfig.ServicePort}{consulConfig.HealthCheckEndpoint}",
                Interval = TimeSpan.FromSeconds(consulConfig.HealthCheckInterval),
                Timeout = TimeSpan.FromSeconds(consulConfig.HealthCheckTimeout),
                DeregisterCriticalServiceAfter = TimeSpan.FromSeconds(consulConfig.DeregisterCriticalServiceAfter)
            }
        };
        
        logger.LogInformation("Registering service with Consul");
        consulClient.Agent.ServiceRegister(registration).ConfigureAwait(false).GetAwaiter().GetResult();
        
        lifetime.ApplicationStopping.Register(() =>
        {
            logger.LogInformation("Deregistering service from Consul");
            consulClient.Agent.ServiceDeregister(registration.ID).ConfigureAwait(false).GetAwaiter().GetResult();
        });
        
        return app;
    }
}

// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Consul 설정
builder.Services.Configure<ConsulConfig>(builder.Configuration.GetSection("Consul"));
builder.Services.AddSingleton<IConsulClient>(p => new ConsulClient(consulConfig =>
{
    var config = p.GetRequiredService<IOptions<ConsulConfig>>().Value;
    consulConfig.Address = new Uri(config.ConsulAddress);
}));

// Health Check 추가
builder.Services.AddHealthChecks();

var app = builder.Build();

// Health Check 엔드포인트
app.MapHealthChecks("/health");

// Consul 등록
app.UseConsul(app.Lifetime);

app.Run();
```

### appsettings.json 설정

```json
{
  "Consul": {
    "ServiceId": "product-service-01",
    "ServiceName": "product-service",
    "ServiceHost": "localhost",
    "ServicePort": 5001,
    "ConsulAddress": "http://localhost:8500",
    "HealthCheckEndpoint": "/health",
    "HealthCheckInterval": 10,
    "HealthCheckTimeout": 5,
    "DeregisterCriticalServiceAfter": 30
  }
}
```

## 서비스 검색

### 기본 서비스 검색

```csharp
// Services/IServiceDiscovery.cs
public interface IServiceDiscovery
{
    Task<List<ServiceEndpoint>> GetServiceEndpointsAsync(string serviceName);
    Task<ServiceEndpoint> GetServiceEndpointAsync(string serviceName);
}

public class ServiceEndpoint
{
    public string ServiceId { get; set; }
    public string Address { get; set; }
    public int Port { get; set; }
    public string[] Tags { get; set; }
}

// Services/ConsulServiceDiscovery.cs
public class ConsulServiceDiscovery : IServiceDiscovery
{
    private readonly IConsulClient _consulClient;
    private readonly ILogger<ConsulServiceDiscovery> _logger;
    
    public ConsulServiceDiscovery(IConsulClient consulClient, ILogger<ConsulServiceDiscovery> logger)
    {
        _consulClient = consulClient;
        _logger = logger;
    }
    
    public async Task<List<ServiceEndpoint>> GetServiceEndpointsAsync(string serviceName)
    {
        var services = await _consulClient.Health.Service(serviceName, "", true);
        
        return services.Response
            .Where(s => s.Service != null)
            .Select(s => new ServiceEndpoint
            {
                ServiceId = s.Service.ID,
                Address = s.Service.Address,
                Port = s.Service.Port,
                Tags = s.Service.Tags
            })
            .ToList();
    }
    
    public async Task<ServiceEndpoint> GetServiceEndpointAsync(string serviceName)
    {
        var endpoints = await GetServiceEndpointsAsync(serviceName);
        
        if (!endpoints.Any())
        {
            throw new ServiceNotFoundException($"Service '{serviceName}' not found");
        }
        
        // 간단한 랜덤 선택 (실제로는 더 복잡한 로드 밸런싱 로직 필요)
        var random = new Random();
        return endpoints[random.Next(endpoints.Count)];
    }
}
```

### HttpClient 통합

```csharp
// HttpClient/ConsulHttpClientHandler.cs
public class ConsulHttpClientHandler : DelegatingHandler
{
    private readonly IServiceDiscovery _serviceDiscovery;
    private readonly ILogger<ConsulHttpClientHandler> _logger;
    
    public ConsulHttpClientHandler(
        IServiceDiscovery serviceDiscovery,
        ILogger<ConsulHttpClientHandler> logger)
    {
        _serviceDiscovery = serviceDiscovery;
        _logger = logger;
    }
    
    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request,
        CancellationToken cancellationToken)
    {
        var currentUri = request.RequestUri;
        var serviceName = currentUri.Host;
        
        try
        {
            var endpoint = await _serviceDiscovery.GetServiceEndpointAsync(serviceName);
            var newUri = new UriBuilder(currentUri)
            {
                Host = endpoint.Address,
                Port = endpoint.Port
            }.Uri;
            
            request.RequestUri = newUri;
            
            _logger.LogDebug($"Routing request to {newUri}");
            
            return await base.SendAsync(request, cancellationToken);
        }
        catch (ServiceNotFoundException)
        {
            _logger.LogError($"Service '{serviceName}' not found in Consul");
            throw;
        }
    }
}

// DI 등록
builder.Services.AddScoped<IServiceDiscovery, ConsulServiceDiscovery>();
builder.Services.AddTransient<ConsulHttpClientHandler>();

builder.Services.AddHttpClient("consul-client")
    .AddHttpMessageHandler<ConsulHttpClientHandler>();
```

## Health Checking

### 상세한 Health Check 구현

```csharp
// HealthChecks/DatabaseHealthCheck.cs
public class DatabaseHealthCheck : IHealthCheck
{
    private readonly IDbConnection _connection;
    
    public DatabaseHealthCheck(IDbConnection connection)
    {
        _connection = connection;
    }
    
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            _connection.Open();
            var command = _connection.CreateCommand();
            command.CommandText = "SELECT 1";
            await command.ExecuteScalarAsync();
            
            return HealthCheckResult.Healthy("Database is accessible");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Database is not accessible", ex);
        }
        finally
        {
            _connection.Close();
        }
    }
}

// Program.cs
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy())
    .AddCheck<DatabaseHealthCheck>("database")
    .AddCheck("external-service", async () =>
    {
        using var httpClient = new HttpClient();
        try
        {
            var response = await httpClient.GetAsync("https://api.external.com/health");
            if (response.IsSuccessStatusCode)
                return HealthCheckResult.Healthy();
            
            return HealthCheckResult.Unhealthy($"External service returned {response.StatusCode}");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("External service is not accessible", ex);
        }
    });
```

### Health Check 응답 커스터마이징

```csharp
// HealthChecks/HealthCheckResponseWriter.cs
public static class HealthCheckResponseWriter
{
    public static Task WriteResponse(HttpContext context, HealthReport report)
    {
        context.Response.ContentType = "application/json";
        
        var response = new
        {
            status = report.Status.ToString(),
            checks = report.Entries.Select(x => new
            {
                name = x.Key,
                status = x.Value.Status.ToString(),
                description = x.Value.Description,
                duration = x.Value.Duration.TotalMilliseconds,
                exception = x.Value.Exception?.Message,
                data = x.Value.Data
            }),
            totalDuration = report.TotalDuration.TotalMilliseconds
        };
        
        var json = JsonSerializer.Serialize(response, new JsonSerializerOptions
        {
            PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
            WriteIndented = true
        });
        
        return context.Response.WriteAsync(json);
    }
}

// Program.cs
app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = HealthCheckResponseWriter.WriteResponse
});
```

## Key/Value Store

### 구성 저장 및 조회

```csharp
// Services/IConsulConfigurationService.cs
public interface IConsulConfigurationService
{
    Task<T> GetConfigurationAsync<T>(string key);
    Task SetConfigurationAsync<T>(string key, T value);
    Task<bool> DeleteConfigurationAsync(string key);
}

// Services/ConsulConfigurationService.cs
public class ConsulConfigurationService : IConsulConfigurationService
{
    private readonly IConsulClient _consulClient;
    private readonly JsonSerializerOptions _jsonOptions;
    
    public ConsulConfigurationService(IConsulClient consulClient)
    {
        _consulClient = consulClient;
        _jsonOptions = new JsonSerializerOptions
        {
            PropertyNamingPolicy = JsonNamingPolicy.CamelCase
        };
    }
    
    public async Task<T> GetConfigurationAsync<T>(string key)
    {
        var getPair = await _consulClient.KV.Get(key);
        
        if (getPair.Response == null)
        {
            return default(T);
        }
        
        var value = Encoding.UTF8.GetString(getPair.Response.Value);
        return JsonSerializer.Deserialize<T>(value, _jsonOptions);
    }
    
    public async Task SetConfigurationAsync<T>(string key, T value)
    {
        var json = JsonSerializer.Serialize(value, _jsonOptions);
        var putPair = new KVPair(key)
        {
            Value = Encoding.UTF8.GetBytes(json)
        };
        
        await _consulClient.KV.Put(putPair);
    }
    
    public async Task<bool> DeleteConfigurationAsync(string key)
    {
        var result = await _consulClient.KV.Delete(key);
        return result.Response;
    }
}
```

### 동적 구성 관리

```csharp
// Configuration/ConsulConfigurationProvider.cs
public class ConsulConfigurationProvider : ConfigurationProvider
{
    private readonly IConsulClient _consulClient;
    private readonly string _prefix;
    private readonly bool _watchForChanges;
    
    public ConsulConfigurationProvider(
        IConsulClient consulClient, 
        string prefix, 
        bool watchForChanges)
    {
        _consulClient = consulClient;
        _prefix = prefix;
        _watchForChanges = watchForChanges;
    }
    
    public override void Load()
    {
        LoadAsync().ConfigureAwait(false).GetAwaiter().GetResult();
        
        if (_watchForChanges)
        {
            WatchForChanges();
        }
    }
    
    private async Task LoadAsync()
    {
        var kvPairs = await _consulClient.KV.List(_prefix);
        
        if (kvPairs.Response == null)
            return;
            
        Data.Clear();
        
        foreach (var kvPair in kvPairs.Response)
        {
            var key = kvPair.Key.Substring(_prefix.Length + 1)
                .Replace('/', ':');
            var value = Encoding.UTF8.GetString(kvPair.Value);
            
            Data[key] = value;
        }
    }
    
    private void WatchForChanges()
    {
        Task.Run(async () =>
        {
            var lastIndex = 0UL;
            
            while (true)
            {
                try
                {
                    var queryOptions = new QueryOptions
                    {
                        WaitIndex = lastIndex
                    };
                    
                    var result = await _consulClient.KV.List(_prefix, queryOptions);
                    
                    if (result.LastIndex != lastIndex)
                    {
                        lastIndex = result.LastIndex;
                        await LoadAsync();
                        OnReload();
                    }
                }
                catch (Exception ex)
                {
                    // 로깅
                    await Task.Delay(5000); // 에러 시 5초 대기
                }
            }
        });
    }
}

// Configuration/ConsulConfigurationSource.cs
public class ConsulConfigurationSource : IConfigurationSource
{
    public string ConsulAddress { get; set; }
    public string Prefix { get; set; }
    public bool WatchForChanges { get; set; }
    
    public IConfigurationProvider Build(IConfigurationBuilder builder)
    {
        var consulClient = new ConsulClient(config =>
        {
            config.Address = new Uri(ConsulAddress);
        });
        
        return new ConsulConfigurationProvider(consulClient, Prefix, WatchForChanges);
    }
}
```

## 부하 분산 전략

### 라운드 로빈 부하 분산

```csharp
// LoadBalancing/ILoadBalancer.cs
public interface ILoadBalancer
{
    Task<ServiceEndpoint> NextAsync(string serviceName);
}

// LoadBalancing/RoundRobinLoadBalancer.cs
public class RoundRobinLoadBalancer : ILoadBalancer
{
    private readonly IServiceDiscovery _serviceDiscovery;
    private readonly Dictionary<string, int> _counters = new();
    private readonly SemaphoreSlim _semaphore = new(1, 1);
    
    public RoundRobinLoadBalancer(IServiceDiscovery serviceDiscovery)
    {
        _serviceDiscovery = serviceDiscovery;
    }
    
    public async Task<ServiceEndpoint> NextAsync(string serviceName)
    {
        var endpoints = await _serviceDiscovery.GetServiceEndpointsAsync(serviceName);
        
        if (!endpoints.Any())
        {
            throw new ServiceNotFoundException($"Service '{serviceName}' not found");
        }
        
        await _semaphore.WaitAsync();
        try
        {
            if (!_counters.ContainsKey(serviceName))
            {
                _counters[serviceName] = 0;
            }
            
            var index = _counters[serviceName] % endpoints.Count;
            _counters[serviceName]++;
            
            return endpoints[index];
        }
        finally
        {
            _semaphore.Release();
        }
    }
}
```

### 가중치 기반 부하 분산

```csharp
// LoadBalancing/WeightedLoadBalancer.cs
public class WeightedLoadBalancer : ILoadBalancer
{
    private readonly IServiceDiscovery _serviceDiscovery;
    private readonly Dictionary<string, List<WeightedEndpoint>> _weightedEndpoints = new();
    
    public async Task<ServiceEndpoint> NextAsync(string serviceName)
    {
        if (!_weightedEndpoints.ContainsKey(serviceName))
        {
            await RefreshEndpoints(serviceName);
        }
        
        var endpoints = _weightedEndpoints[serviceName];
        var totalWeight = endpoints.Sum(e => e.Weight);
        var randomValue = new Random().Next(totalWeight);
        
        var currentWeight = 0;
        foreach (var endpoint in endpoints)
        {
            currentWeight += endpoint.Weight;
            if (randomValue < currentWeight)
            {
                return endpoint.Endpoint;
            }
        }
        
        return endpoints.Last().Endpoint;
    }
    
    private async Task RefreshEndpoints(string serviceName)
    {
        var endpoints = await _serviceDiscovery.GetServiceEndpointsAsync(serviceName);
        
        _weightedEndpoints[serviceName] = endpoints.Select(e => new WeightedEndpoint
        {
            Endpoint = e,
            Weight = GetWeight(e.Tags)
        }).ToList();
    }
    
    private int GetWeight(string[] tags)
    {
        // 태그에서 가중치 추출 (예: "weight:10")
        var weightTag = tags?.FirstOrDefault(t => t.StartsWith("weight:"));
        if (weightTag != null && int.TryParse(weightTag.Split(':')[1], out var weight))
        {
            return weight;
        }
        
        return 1; // 기본 가중치
    }
    
    private class WeightedEndpoint
    {
        public ServiceEndpoint Endpoint { get; set; }
        public int Weight { get; set; }
    }
}
```

## 서비스 메시 패턴

### Sidecar 프록시 구현

```csharp
// Sidecar/SidecarProxy.cs
public class SidecarProxy
{
    private readonly IServiceDiscovery _serviceDiscovery;
    private readonly IHttpClientFactory _httpClientFactory;
    private readonly ILogger<SidecarProxy> _logger;
    
    public SidecarProxy(
        IServiceDiscovery serviceDiscovery,
        IHttpClientFactory httpClientFactory,
        ILogger<SidecarProxy> logger)
    {
        _serviceDiscovery = serviceDiscovery;
        _httpClientFactory = httpClientFactory;
        _logger = logger;
    }
    
    public async Task<HttpResponseMessage> ForwardRequestAsync(
        string serviceName,
        HttpRequestMessage request)
    {
        var endpoint = await _serviceDiscovery.GetServiceEndpointAsync(serviceName);
        var httpClient = _httpClientFactory.CreateClient();
        
        // 요청 URL 변경
        var targetUri = new UriBuilder(request.RequestUri)
        {
            Host = endpoint.Address,
            Port = endpoint.Port
        }.Uri;
        
        var newRequest = new HttpRequestMessage(request.Method, targetUri)
        {
            Content = request.Content
        };
        
        // 헤더 복사
        foreach (var header in request.Headers)
        {
            newRequest.Headers.TryAddWithoutValidation(header.Key, header.Value);
        }
        
        // 추가 헤더
        newRequest.Headers.Add("X-Forwarded-Host", request.RequestUri.Host);
        newRequest.Headers.Add("X-Forwarded-Proto", request.RequestUri.Scheme);
        
        _logger.LogInformation($"Forwarding request to {targetUri}");
        
        return await httpClient.SendAsync(newRequest);
    }
}
```

## 서비스 간 보안 통신

### mTLS 설정

```csharp
// Security/ConsulCertificateManager.cs
public class ConsulCertificateManager
{
    private readonly IConsulClient _consulClient;
    
    public ConsulCertificateManager(IConsulClient consulClient)
    {
        _consulClient = consulClient;
    }
    
    public async Task<X509Certificate2> GetServiceCertificateAsync(string serviceName)
    {
        // Consul Connect CA에서 인증서 가져오기
        var connectCA = await _consulClient.Agent.ConnectCALeaf(serviceName);
        
        if (connectCA.Response != null)
        {
            var certPem = connectCA.Response.CertPEM;
            var keyPem = connectCA.Response.PrivateKeyPEM;
            
            return CreateCertificateFromPem(certPem, keyPem);
        }
        
        throw new InvalidOperationException($"Unable to get certificate for service {serviceName}");
    }
    
    private X509Certificate2 CreateCertificateFromPem(string certPem, string keyPem)
    {
        var cert = X509Certificate2.CreateFromPem(certPem, keyPem);
        return new X509Certificate2(cert.Export(X509ContentType.Pfx));
    }
}

// HttpClient 설정
builder.Services.AddHttpClient("secure-client")
    .ConfigurePrimaryHttpMessageHandler(serviceProvider =>
    {
        var certManager = serviceProvider.GetRequiredService<ConsulCertificateManager>();
        var cert = certManager.GetServiceCertificateAsync("my-service")
            .ConfigureAwait(false).GetAwaiter().GetResult();
            
        var handler = new HttpClientHandler();
        handler.ClientCertificates.Add(cert);
        handler.ServerCertificateCustomValidationCallback = 
            (httpRequestMessage, cert, certChain, policyErrors) => true;
            
        return handler;
    });
```

## 모니터링과 추적

### 서비스 메트릭 수집

```csharp
// Monitoring/ConsulMetricsCollector.cs
public class ConsulMetricsCollector : BackgroundService
{
    private readonly IConsulClient _consulClient;
    private readonly IMetrics _metrics;
    private readonly ILogger<ConsulMetricsCollector> _logger;
    
    public ConsulMetricsCollector(
        IConsulClient consulClient,
        IMetrics metrics,
        ILogger<ConsulMetricsCollector> logger)
    {
        _consulClient = consulClient;
        _metrics = metrics;
        _logger = logger;
    }
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                // 모든 서비스 상태 수집
                var services = await _consulClient.Agent.Services();
                
                foreach (var service in services.Response.Values)
                {
                    var health = await _consulClient.Health.Service(service.Service, "", false);
                    
                    var healthyCount = health.Response.Count(h => h.Checks.All(c => c.Status == HealthStatus.Passing));
                    var unhealthyCount = health.Response.Count - healthyCount;
                    
                    _metrics.Measure.Gauge.SetValue(
                        new GaugeOptions
                        {
                            Name = "consul_service_instances_healthy",
                            Tags = new MetricTags("service", service.Service)
                        },
                        healthyCount);
                        
                    _metrics.Measure.Gauge.SetValue(
                        new GaugeOptions
                        {
                            Name = "consul_service_instances_unhealthy",
                            Tags = new MetricTags("service", service.Service)
                        },
                        unhealthyCount);
                }
                
                await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error collecting Consul metrics");
                await Task.Delay(TimeSpan.FromSeconds(60), stoppingToken);
            }
        }
    }
}
```

### 분산 추적

```csharp
// Tracing/ConsulTracingMiddleware.cs
public class ConsulTracingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IServiceDiscovery _serviceDiscovery;
    
    public ConsulTracingMiddleware(
        RequestDelegate next,
        IServiceDiscovery serviceDiscovery)
    {
        _next = next;
        _serviceDiscovery = serviceDiscovery;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        var activity = Activity.Current;
        
        if (activity != null)
        {
            // Consul 서비스 정보를 추적 데이터에 추가
            var serviceName = context.Request.Headers["X-Service-Name"].FirstOrDefault();
            
            if (!string.IsNullOrEmpty(serviceName))
            {
                var endpoints = await _serviceDiscovery.GetServiceEndpointsAsync(serviceName);
                
                activity.SetTag("consul.service.name", serviceName);
                activity.SetTag("consul.service.instance_count", endpoints.Count);
                activity.SetTag("consul.datacenter", "dc1");
            }
        }
        
        await _next(context);
    }
}
```

## 마무리

Consul은 마이크로서비스 아키텍처에서 Service Discovery, 헬스 체킹, 구성 관리 등을 제공하는 강력한 도구입니다. ASP.NET Core와의 통합을 통해 동적이고 확장 가능한 마이크로서비스 시스템을 구축할 수 있습니다.