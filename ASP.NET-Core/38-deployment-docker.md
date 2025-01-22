# ASP.NET Core 배포 - Docker

## 개요

Docker는 애플리케이션을 컨테이너로 패키징하여 어디서든 일관되게 실행할 수 있게 해주는 플랫폼입니다. ASP.NET Core 애플리케이션을 Docker로 배포하면 환경 독립성, 확장성, 그리고 배포 일관성을 확보할 수 있습니다.

## Docker 기본 개념

### 컨테이너 vs 가상 머신

```plaintext
가상 머신:
┌─────────────┬─────────────┬─────────────┐
│   App A     │   App B     │   App C     │
├─────────────┼─────────────┼─────────────┤
│  Guest OS   │  Guest OS   │  Guest OS   │
├─────────────┴─────────────┴─────────────┤
│              Hypervisor                  │
├──────────────────────────────────────────┤
│              Host OS                     │
├──────────────────────────────────────────┤
│              Hardware                    │
└──────────────────────────────────────────┘

Docker 컨테이너:
┌─────────────┬─────────────┬─────────────┐
│   App A     │   App B     │   App C     │
├─────────────┼─────────────┼─────────────┤
│ Container A │ Container B │ Container C │
├─────────────┴─────────────┴─────────────┤
│            Docker Engine                 │
├──────────────────────────────────────────┤
│              Host OS                     │
├──────────────────────────────────────────┤
│              Hardware                    │
└──────────────────────────────────────────┘
```

### Docker 핵심 구성 요소

```csharp
// Docker 개념 이해를 위한 의사 코드
public class DockerConcepts
{
    // 이미지: 읽기 전용 템플릿
    public class DockerImage
    {
        public string BaseOS { get; set; }
        public string Runtime { get; set; }
        public string Application { get; set; }
        public List<string> Layers { get; set; }
    }

    // 컨테이너: 이미지의 실행 인스턴스
    public class DockerContainer
    {
        public DockerImage Image { get; set; }
        public string State { get; set; } // Running, Stopped
        public Dictionary<string, string> Environment { get; set; }
        public List<int> ExposedPorts { get; set; }
    }

    // 레지스트리: 이미지 저장소
    public class DockerRegistry
    {
        public string RegistryUrl { get; set; }
        public List<DockerImage> Images { get; set; }
    }
}
```

## ASP.NET Core용 Dockerfile 작성

### 기본 Dockerfile

```dockerfile
# 기본 ASP.NET Core Dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["MyApp/MyApp.csproj", "MyApp/"]
RUN dotnet restore "MyApp/MyApp.csproj"
COPY . .
WORKDIR "/src/MyApp"
RUN dotnet build "MyApp.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "MyApp.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

### Dockerfile 명령어 설명

```dockerfile
# FROM: 베이스 이미지 지정
FROM mcr.microsoft.com/dotnet/aspnet:8.0

# WORKDIR: 작업 디렉토리 설정
WORKDIR /app

# COPY: 파일 복사
COPY . .

# RUN: 빌드 시 명령 실행
RUN dotnet restore

# EXPOSE: 포트 노출
EXPOSE 80

# ENV: 환경 변수 설정
ENV ASPNETCORE_ENVIRONMENT=Production

# ENTRYPOINT: 컨테이너 시작 명령
ENTRYPOINT ["dotnet", "MyApp.dll"]

# CMD: 기본 명령 (ENTRYPOINT와 함께 사용)
CMD ["--urls", "http://+:80"]
```

## 멀티 스테이지 빌드 최적화

### 최적화된 멀티 스테이지 Dockerfile

```dockerfile
# 1단계: 빌드 환경
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# 캐싱 최적화를 위해 프로젝트 파일만 먼저 복사
COPY ["MyApp/MyApp.csproj", "MyApp/"]
COPY ["MyApp.Core/MyApp.Core.csproj", "MyApp.Core/"]
COPY ["MyApp.Data/MyApp.Data.csproj", "MyApp.Data/"]

# NuGet 패키지 복원
RUN dotnet restore "MyApp/MyApp.csproj"

# 소스 코드 복사 및 빌드
COPY . .
WORKDIR "/src/MyApp"
RUN dotnet build "MyApp.csproj" -c Release -o /app/build

# 2단계: 게시
FROM build AS publish
RUN dotnet publish "MyApp.csproj" -c Release -o /app/publish /p:UseAppHost=false

# 3단계: 런타임 이미지
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app

# 비루트 사용자로 실행
RUN addgroup -g 1000 -S appuser && adduser -u 1000 -S appuser -G appuser
USER appuser

# 게시된 앱 복사
COPY --from=publish --chown=appuser:appuser /app/publish .

# 헬스체크 추가
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost/health || exit 1

ENTRYPOINT ["dotnet", "MyApp.dll"]
```

### 빌드 시간 최적화 전략

```dockerfile
# .dockerignore 파일
**/bin
**/obj
**/.vs
**/.vscode
**/node_modules
**/dist
**/*.user
**/*.suo
**/logs
**/TestResults
.git
.gitignore
README.md
```

## 컨테이너 이미지 최적화

### 이미지 크기 최적화

```dockerfile
# Alpine Linux 기반 경량 이미지 사용
FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine AS base
WORKDIR /app
EXPOSE 80

# 필요한 패키지만 설치
RUN apk add --no-cache \
    ca-certificates \
    krb5-libs \
    libgcc \
    libintl \
    libssl1.1 \
    libstdc++ \
    tzdata

FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS build
WORKDIR /src

# 프로젝트 파일 복사 및 복원
COPY ["MyApp/MyApp.csproj", "MyApp/"]
RUN dotnet restore "MyApp/MyApp.csproj" -r linux-musl-x64

# 소스 복사 및 빌드
COPY . .
WORKDIR "/src/MyApp"
RUN dotnet publish "MyApp.csproj" \
    -c Release \
    -o /app/publish \
    -r linux-musl-x64 \
    --self-contained false \
    /p:PublishSingleFile=true \
    /p:PublishTrimmed=true

FROM base AS final
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["./MyApp"]
```

### 레이어 캐싱 최적화

```csharp
// Startup.cs - 정적 파일 캐싱 설정
public class Startup
{
    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        // 정적 파일 캐싱
        app.UseStaticFiles(new StaticFileOptions
        {
            OnPrepareResponse = ctx =>
            {
                // 캐시 헤더 설정
                ctx.Context.Response.Headers.Append(
                    "Cache-Control", "public,max-age=31536000");
            }
        });

        // 압축 활성화
        app.UseResponseCompression();

        app.UseRouting();
        app.UseEndpoints(endpoints =>
        {
            endpoints.MapControllers();
        });
    }
}
```

## Docker Compose 다중 컨테이너 관리

### 기본 docker-compose.yml

```yaml
version: '3.8'

services:
  webapp:
    build:
      context: .
      dockerfile: MyApp/Dockerfile
    ports:
      - "8080:80"
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - ConnectionStrings__DefaultConnection=Server=db;Database=MyAppDb;User=sa;Password=Your_password123;
    depends_on:
      - db
      - redis
    networks:
      - myapp-network

  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=Your_password123
    ports:
      - "1433:1433"
    volumes:
      - db-data:/var/opt/mssql
    networks:
      - myapp-network

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - myapp-network

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certs:/etc/nginx/certs:ro
    depends_on:
      - webapp
    networks:
      - myapp-network

volumes:
  db-data:
  redis-data:

networks:
  myapp-network:
    driver: bridge
```

### 고급 Docker Compose 설정

```yaml
version: '3.8'

services:
  webapp:
    build:
      context: .
      dockerfile: MyApp/Dockerfile
      args:
        - BUILD_VERSION=${BUILD_VERSION:-1.0.0}
    image: myregistry.azurecr.io/myapp:${TAG:-latest}
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    environment:
      - ASPNETCORE_ENVIRONMENT=${ENVIRONMENT:-Production}
      - ApplicationInsights__InstrumentationKey=${APP_INSIGHTS_KEY}
    env_file:
      - .env
    secrets:
      - db_password
      - jwt_secret
    networks:
      - frontend
      - backend
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

secrets:
  db_password:
    external: true
  jwt_secret:
    external: true

networks:
  frontend:
    driver: overlay
  backend:
    driver: overlay
    internal: true
```

## 환경 구성 및 시크릿 관리

### 환경별 설정

```csharp
// Program.cs - 환경별 구성
public class Program
{
    public static void Main(string[] args)
    {
        CreateHostBuilder(args).Build().Run();
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureAppConfiguration((context, config) =>
            {
                var env = context.HostingEnvironment;
                
                config
                    .SetBasePath(Directory.GetCurrentDirectory())
                    .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
                    .AddJsonFile($"appsettings.{env.EnvironmentName}.json", optional: true)
                    .AddEnvironmentVariables()
                    .AddCommandLine(args);

                // Docker 시크릿 추가
                if (env.IsProduction())
                {
                    config.AddDockerSecrets("/run/secrets");
                }
            })
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.UseStartup<Startup>();
            });
}

// Docker 시크릿 구성 제공자
public static class DockerSecretsConfigurationExtensions
{
    public static IConfigurationBuilder AddDockerSecrets(
        this IConfigurationBuilder builder, 
        string secretsPath)
    {
        if (Directory.Exists(secretsPath))
        {
            foreach (var file in Directory.GetFiles(secretsPath))
            {
                var secretName = Path.GetFileName(file);
                var secretValue = File.ReadAllText(file).Trim();
                
                builder.AddInMemoryCollection(new[]
                {
                    new KeyValuePair<string, string>(secretName, secretValue)
                });
            }
        }
        
        return builder;
    }
}
```

### 시크릿 관리 예제

```yaml
# docker-compose.override.yml - 개발 환경
version: '3.8'

services:
  webapp:
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__DefaultConnection=Server=localhost;Database=MyAppDb;Integrated Security=true;
    volumes:
      - ./src:/app
      - ~/.aspnet/https:/https:ro
    ports:
      - "5000:80"
      - "5001:443"
```

```bash
# Docker 시크릿 생성
echo "SuperSecretPassword123!" | docker secret create db_password -
echo "MyJwtSecretKey123!" | docker secret create jwt_secret -

# 시크릿 사용
docker service create \
  --name myapp \
  --secret db_password \
  --secret jwt_secret \
  myregistry.azurecr.io/myapp:latest
```

## 헬스체크 구현

### 헬스체크 엔드포인트

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
            await _connection.OpenAsync();
            
            var command = _connection.CreateCommand();
            command.CommandText = "SELECT 1";
            await command.ExecuteScalarAsync();
            
            return HealthCheckResult.Healthy("Database is accessible");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy(
                "Database is not accessible", 
                exception: ex);
        }
        finally
        {
            await _connection.CloseAsync();
        }
    }
}

// Startup.cs
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddHealthChecks()
            .AddCheck<DatabaseHealthCheck>("database")
            .AddRedis(
                Configuration.GetConnectionString("Redis"),
                name: "redis",
                tags: new[] { "cache" })
            .AddUrlGroup(
                new Uri("https://api.external.com/health"),
                name: "external-api",
                tags: new[] { "external" });

        services.AddHealthChecksUI()
            .AddInMemoryStorage();
    }

    public void Configure(IApplicationBuilder app)
    {
        app.UseRouting();

        app.UseEndpoints(endpoints =>
        {
            endpoints.MapHealthChecks("/health", new HealthCheckOptions
            {
                ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
            });

            endpoints.MapHealthChecks("/health/ready", new HealthCheckOptions
            {
                Predicate = check => check.Tags.Contains("ready")
            });

            endpoints.MapHealthChecks("/health/live", new HealthCheckOptions
            {
                Predicate = _ => false
            });

            endpoints.MapHealthChecksUI();
        });
    }
}
```

### Docker 헬스체크 통합

```dockerfile
# Dockerfile with health check
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app
COPY --from=publish /app/publish .

# 헬스체크 도구 설치
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

# 헬스체크 설정
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost/health || exit 1

ENTRYPOINT ["dotnet", "MyApp.dll"]
```

## 컨테이너 레지스트리 관리

### Azure Container Registry 사용

```bash
# ACR 로그인
az acr login --name myregistry

# 이미지 태깅
docker tag myapp:latest myregistry.azurecr.io/myapp:latest
docker tag myapp:latest myregistry.azurecr.io/myapp:v1.0.0

# 이미지 푸시
docker push myregistry.azurecr.io/myapp:latest
docker push myregistry.azurecr.io/myapp:v1.0.0
```

### 레지스트리 자동화 스크립트

```csharp
// RegistryManager.cs
public class ContainerRegistryManager
{
    private readonly IConfiguration _configuration;
    private readonly ILogger<ContainerRegistryManager> _logger;

    public ContainerRegistryManager(
        IConfiguration configuration,
        ILogger<ContainerRegistryManager> logger)
    {
        _configuration = configuration;
        _logger = logger;
    }

    public async Task<string> BuildAndPushImageAsync(
        string dockerfilePath,
        string imageName,
        string tag)
    {
        var registry = _configuration["ContainerRegistry:Url"];
        var fullImageName = $"{registry}/{imageName}:{tag}";

        try
        {
            // Docker 빌드
            var buildProcess = new Process
            {
                StartInfo = new ProcessStartInfo
                {
                    FileName = "docker",
                    Arguments = $"build -t {fullImageName} -f {dockerfilePath} .",
                    RedirectStandardOutput = true,
                    RedirectStandardError = true,
                    UseShellExecute = false
                }
            };

            buildProcess.Start();
            await buildProcess.WaitForExitAsync();

            if (buildProcess.ExitCode != 0)
            {
                var error = await buildProcess.StandardError.ReadToEndAsync();
                throw new Exception($"Docker build failed: {error}");
            }

            // Docker 푸시
            var pushProcess = new Process
            {
                StartInfo = new ProcessStartInfo
                {
                    FileName = "docker",
                    Arguments = $"push {fullImageName}",
                    RedirectStandardOutput = true,
                    RedirectStandardError = true,
                    UseShellExecute = false
                }
            };

            pushProcess.Start();
            await pushProcess.WaitForExitAsync();

            if (pushProcess.ExitCode != 0)
            {
                var error = await pushProcess.StandardError.ReadToEndAsync();
                throw new Exception($"Docker push failed: {error}");
            }

            _logger.LogInformation($"Successfully pushed image: {fullImageName}");
            return fullImageName;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to build and push Docker image");
            throw;
        }
    }
}
```

## Docker 네트워킹과 볼륨

### 커스텀 네트워크 구성

```yaml
# docker-compose.networks.yml
version: '3.8'

networks:
  frontend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
          gateway: 172.20.0.1

  backend:
    driver: bridge
    internal: true
    ipam:
      config:
        - subnet: 172.21.0.0/16

services:
  webapp:
    networks:
      frontend:
        ipv4_address: 172.20.0.10
      backend:
    
  database:
    networks:
      backend:
        ipv4_address: 172.21.0.10
```

### 볼륨 관리

```yaml
# docker-compose.volumes.yml
version: '3.8'

services:
  webapp:
    volumes:
      # 명명된 볼륨
      - app-data:/app/data
      # 바인드 마운트
      - ./logs:/app/logs
      # 임시 볼륨
      - /app/temp

  database:
    volumes:
      - type: volume
        source: db-data
        target: /var/opt/mssql
        volume:
          nocopy: true
      # 백업 볼륨
      - type: bind
        source: ./backup
        target: /backup
        read_only: false

volumes:
  app-data:
    driver: local
    driver_opts:
      type: none
      device: /data/app
      o: bind
  
  db-data:
    external: true
    name: production_db_data
```

### 볼륨 백업 스크립트

```csharp
// VolumeBackupService.cs
public class VolumeBackupService : BackgroundService
{
    private readonly ILogger<VolumeBackupService> _logger;
    private readonly IConfiguration _configuration;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await BackupVolumesAsync();
                await Task.Delay(TimeSpan.FromHours(24), stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Volume backup failed");
            }
        }
    }

    private async Task BackupVolumesAsync()
    {
        var volumes = _configuration.GetSection("Docker:Volumes").Get<string[]>();
        
        foreach (var volume in volumes)
        {
            var backupPath = $"/backup/{volume}-{DateTime.UtcNow:yyyyMMdd-HHmmss}.tar";
            
            var process = new Process
            {
                StartInfo = new ProcessStartInfo
                {
                    FileName = "docker",
                    Arguments = $"run --rm -v {volume}:/source -v backup:/backup alpine tar -czf {backupPath} -C /source .",
                    UseShellExecute = false
                }
            };

            process.Start();
            await process.WaitForExitAsync();
            
            _logger.LogInformation($"Volume {volume} backed up to {backupPath}");
        }
    }
}
```

## 프로덕션 배포 모범 사례

### 보안 강화된 Dockerfile

```dockerfile
# 프로덕션용 보안 강화 Dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine AS base

# 보안 업데이트
RUN apk update && apk upgrade && rm -rf /var/cache/apk/*

# 비루트 사용자 생성
RUN addgroup -g 1001 -S appuser && adduser -u 1001 -S appuser -G appuser

# 작업 디렉토리 설정
WORKDIR /app
RUN chown -R appuser:appuser /app

# 빌드 스테이지
FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS build
WORKDIR /src

# 의존성 캐싱을 위한 레이어 분리
COPY ["MyApp/MyApp.csproj", "MyApp/"]
RUN dotnet restore "MyApp/MyApp.csproj"

# 소스 복사 및 빌드
COPY . .
WORKDIR "/src/MyApp"
RUN dotnet publish "MyApp.csproj" \
    -c Release \
    -o /app/publish \
    --no-restore \
    /p:PublishTrimmed=true \
    /p:PublishSingleFile=true

# 런타임 스테이지
FROM base AS final

# 필요한 런타임 의존성만 설치
RUN apk add --no-cache \
    ca-certificates \
    tzdata

# 사용자 전환
USER appuser

# 애플리케이션 복사
COPY --from=build --chown=appuser:appuser /app/publish .

# 읽기 전용 파일 시스템
RUN chmod -R 555 /app

# 환경 변수
ENV ASPNETCORE_URLS=http://+:8080 \
    DOTNET_RUNNING_IN_CONTAINER=true \
    DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=false

# 포트 노출
EXPOSE 8080

# 헬스체크
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1

ENTRYPOINT ["./MyApp"]
```

### 프로덕션 Docker Compose

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  webapp:
    image: ${REGISTRY}/myapp:${VERSION}
    deploy:
      mode: replicated
      replicas: 3
      placement:
        constraints:
          - node.role == worker
      resources:
        limits:
          cpus: '1.0'
          memory: 1G
        reservations:
          cpus: '0.5'
          memory: 512M
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
        monitor: 60s
        max_failure_ratio: 0.3
      restart_policy:
        condition: any
        delay: 5s
        max_attempts: 3
        window: 120s
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - ApplicationInsights__InstrumentationKey=${APP_INSIGHTS_KEY}
    secrets:
      - source: db_connection
        target: ConnectionStrings__DefaultConnection
      - source: jwt_key
        target: Security__JwtKey
    networks:
      - overlay_net
    logging:
      driver: "fluentd"
      options:
        fluentd-address: "localhost:24224"
        tag: "webapp.{{.Name}}"

  nginx:
    image: nginx:stable-alpine
    deploy:
      mode: global
      placement:
        constraints:
          - node.role == manager
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/certs:/etc/nginx/certs:ro
    networks:
      - overlay_net

networks:
  overlay_net:
    driver: overlay
    attachable: true
    encrypted: true

secrets:
  db_connection:
    external: true
  jwt_key:
    external: true
```

### 모니터링과 로깅

```csharp
// Startup.cs - 프로덕션 로깅 구성
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        // Application Insights
        services.AddApplicationInsightsTelemetry();
        
        // 구조화된 로깅
        services.AddLogging(builder =>
        {
            builder.AddConsole(options =>
            {
                options.Format = ConsoleLoggerFormat.Systemd;
                options.TimestampFormat = "yyyy-MM-dd HH:mm:ss ";
            });
            
            builder.AddJsonConsole(options =>
            {
                options.UseUtcTimestamp = true;
                options.IncludeScopes = true;
                options.JsonWriterOptions = new JsonWriterOptions
                {
                    Indented = false
                };
            });
        });

        // 메트릭 수집
        services.AddSingleton<IMetrics>(provider =>
        {
            var metrics = new MetricsBuilder()
                .Report.ToInfluxDb(
                    options =>
                    {
                        options.InfluxDb.BaseUri = new Uri("http://influxdb:8086");
                        options.InfluxDb.Database = "myapp_metrics";
                    })
                .Build();

            return metrics;
        });
    }

    public void Configure(IApplicationBuilder app)
    {
        // 요청 로깅
        app.UseSerilogRequestLogging(options =>
        {
            options.MessageTemplate = "HTTP {RequestMethod} {RequestPath} responded {StatusCode} in {Elapsed:0.0000} ms";
            options.EnrichDiagnosticContext = (diagnosticContext, httpContext) =>
            {
                diagnosticContext.Set("RequestHost", httpContext.Request.Host.Value);
                diagnosticContext.Set("UserAgent", httpContext.Request.Headers["User-Agent"]);
                diagnosticContext.Set("RemoteIP", httpContext.Connection.RemoteIpAddress);
            };
        });
    }
}
```

### 자동 배포 스크립트

```bash
#!/bin/bash
# deploy.sh - 프로덕션 배포 스크립트

set -e

# 환경 변수
REGISTRY="myregistry.azurecr.io"
APP_NAME="myapp"
VERSION="${BUILD_VERSION:-latest}"
ENVIRONMENT="${DEPLOY_ENV:-production}"

# 색상 정의
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

echo -e "${YELLOW}Starting deployment of $APP_NAME:$VERSION to $ENVIRONMENT${NC}"

# 1. 이미지 빌드
echo -e "${GREEN}Building Docker image...${NC}"
docker build -t $REGISTRY/$APP_NAME:$VERSION -f Dockerfile .

# 2. 이미지 스캔
echo -e "${GREEN}Scanning image for vulnerabilities...${NC}"
docker scan $REGISTRY/$APP_NAME:$VERSION

# 3. 레지스트리 푸시
echo -e "${GREEN}Pushing to registry...${NC}"
docker push $REGISTRY/$APP_NAME:$VERSION

# 4. 배포
echo -e "${GREEN}Deploying to $ENVIRONMENT...${NC}"
docker stack deploy -c docker-compose.prod.yml $APP_NAME

# 5. 헬스체크 대기
echo -e "${GREEN}Waiting for health checks...${NC}"
sleep 30

# 6. 배포 검증
HEALTH_STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost/health)
if [ $HEALTH_STATUS -eq 200 ]; then
    echo -e "${GREEN}Deployment successful!${NC}"
else
    echo -e "${RED}Deployment failed! Health check returned $HEALTH_STATUS${NC}"
    exit 1
fi
```

## 성능 최적화

### 이미지 레이어 최적화

```dockerfile
# 최적화된 레이어 구조
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build

# 1. 도구 설치 (거의 변경되지 않음)
RUN dotnet tool install -g dotnet-ef

# 2. 패키지 복원 (의존성 변경 시에만)
WORKDIR /src
COPY ["*.sln", "./"]
COPY ["*/*.csproj", "./"]
RUN for file in $(ls *.csproj); do mkdir -p ${file%.*}/ && mv $file ${file%.*}/; done
RUN dotnet restore

# 3. 소스 복사 및 빌드 (자주 변경됨)
COPY . .
RUN dotnet publish -c Release -o /app/publish
```

### 컨테이너 리소스 최적화

```csharp
// Program.cs - 리소스 최적화
public class Program
{
    public static void Main(string[] args)
    {
        // GC 설정 최적화
        GCSettings.LatencyMode = GCLatencyMode.SustainedLowLatency;
        
        CreateHostBuilder(args).Build().Run();
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.ConfigureKestrel(serverOptions =>
                {
                    // 연결 제한
                    serverOptions.Limits.MaxConcurrentConnections = 100;
                    serverOptions.Limits.MaxConcurrentUpgradedConnections = 100;
                    
                    // 요청 크기 제한
                    serverOptions.Limits.MaxRequestBodySize = 10 * 1024 * 1024; // 10MB
                    
                    // Keep-Alive 타임아웃
                    serverOptions.Limits.KeepAliveTimeout = TimeSpan.FromMinutes(2);
                    
                    // 스레드 풀 설정
                    serverOptions.Limits.MinRequestBodyDataRate = null;
                });
                
                webBuilder.UseStartup<Startup>();
            })
            .UseDefaultServiceProvider(options =>
            {
                options.ValidateScopes = false;
                options.ValidateOnBuild = false;
            });
}
```

## 문제 해결 및 디버깅

### 컨테이너 디버깅 설정

```dockerfile
# 디버그용 Dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS debug
WORKDIR /app
EXPOSE 80
EXPOSE 443

# 디버거 설치
RUN apt-get update && apt-get install -y procps
RUN dotnet tool install -g dotnet-trace
RUN dotnet tool install -g dotnet-dump
RUN dotnet tool install -g dotnet-counters

ENV PATH="${PATH}:/root/.dotnet/tools"
ENV ASPNETCORE_ENVIRONMENT=Development
ENV DOTNET_USE_POLLING_FILE_WATCHER=true

ENTRYPOINT ["dotnet", "watch", "run", "--urls=http://+:80"]
```

### 로그 수집 및 분석

```yaml
# docker-compose.logging.yml
version: '3.8'

services:
  webapp:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
        labels: "app=myapp,env=production"

  fluentd:
    image: fluent/fluentd:v1.16-debian
    volumes:
      - ./fluentd/fluent.conf:/fluentd/etc/fluent.conf
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
    ports:
      - "24224:24224"
    environment:
      - FLUENTD_CONF=fluent.conf
```

### 컨테이너 메트릭 모니터링

```csharp
// MetricsMiddleware.cs
public class MetricsMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IMetrics _metrics;

    public MetricsMiddleware(RequestDelegate next, IMetrics metrics)
    {
        _next = next;
        _metrics = metrics;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var stopwatch = Stopwatch.StartNew();

        try
        {
            await _next(context);
        }
        finally
        {
            stopwatch.Stop();

            var tags = new MetricTags(
                new[] { "method", "endpoint", "status" },
                new[] { context.Request.Method, context.Request.Path, context.Response.StatusCode.ToString() }
            );

            _metrics.Measure.Timer.Time(tags, stopwatch.Elapsed);
            _metrics.Measure.Counter.Increment(MetricsRegistry.RequestCounter, tags);
        }
    }
}
```

## 마무리

Docker를 사용한 ASP.NET Core 애플리케이션 배포는 현대적인 클라우드 네이티브 개발의 핵심입니다. 이 문서에서 다룬 주요 내용은:

1. **Docker 기초**: 컨테이너와 이미지의 개념 이해
2. **Dockerfile 작성**: 효율적인 멀티스테이지 빌드
3. **이미지 최적화**: 크기와 빌드 시간 최소화
4. **Docker Compose**: 다중 컨테이너 오케스트레이션
5. **보안과 시크릿**: 안전한 구성 관리
6. **헬스체크**: 컨테이너 상태 모니터링
7. **레지스트리 관리**: 이미지 저장소 운영
8. **네트워킹과 볼륨**: 데이터 영속성과 통신
9. **프로덕션 배포**: 모범 사례와 자동화
10. **모니터링과 디버깅**: 운영 환경 관리

Docker를 통해 일관된 배포 환경을 구축하고, 확장 가능한 마이크로서비스 아키텍처를 구현할 수 있습니다. 컨테이너 기술을 마스터하여 더 나은 DevOps 문화를 만들어가세요!