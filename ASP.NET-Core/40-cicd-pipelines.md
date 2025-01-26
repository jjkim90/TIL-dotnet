# CI/CD 파이프라인

## 개요

CI/CD(Continuous Integration/Continuous Deployment)는 현대 소프트웨어 개발의 핵심 프랙티스입니다. ASP.NET Core 애플리케이션의 빌드, 테스트, 배포를 자동화하여 더 빠르고 안정적인 소프트웨어 딜리버리를 가능하게 합니다.

## CI/CD 기본 개념

### 지속적 통합 (Continuous Integration)

```yaml
# CI의 핵심 개념
1. 코드 변경사항을 자주 통합
2. 자동화된 빌드 및 테스트
3. 빠른 피드백 루프
4. 코드 품질 유지
```

### 지속적 배포 (Continuous Deployment)

```yaml
# CD의 핵심 개념
1. 검증된 코드의 자동 배포
2. 환경별 배포 전략
3. 롤백 메커니즘
4. 무중단 배포
```

### CI/CD의 이점

```csharp
// CI/CD 도입 효과
public class CICDBenefits
{
    public List<string> Benefits => new()
    {
        "빠른 피드백과 문제 조기 발견",
        "수동 작업 감소와 인적 오류 방지",
        "일관된 빌드 및 배포 프로세스",
        "더 빈번한 릴리즈 가능",
        "팀 협업 향상",
        "코드 품질 향상"
    };
}
```

## GitHub Actions 워크플로우

### 기본 워크플로우 구조

```yaml
# .github/workflows/ci-cd.yml
name: ASP.NET Core CI/CD

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]
  release:
    types: [ created ]

env:
  DOTNET_VERSION: '8.0.x'
  AZURE_WEBAPP_NAME: 'myapp-prod'
  AZURE_WEBAPP_PACKAGE_PATH: './publish'

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup .NET
      uses: actions/setup-dotnet@v4
      with:
        dotnet-version: ${{ env.DOTNET_VERSION }}
    
    - name: Restore dependencies
      run: dotnet restore
    
    - name: Build
      run: dotnet build --no-restore --configuration Release
    
    - name: Test
      run: dotnet test --no-build --verbosity normal --configuration Release
```

### 고급 빌드 및 테스트

```yaml
  advanced-build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        dotnet-version: ['6.0.x', '7.0.x', '8.0.x']
    
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0  # Shallow clones disabled for better analysis
    
    - name: Setup .NET ${{ matrix.dotnet-version }}
      uses: actions/setup-dotnet@v4
      with:
        dotnet-version: ${{ matrix.dotnet-version }}
    
    - name: Cache NuGet packages
      uses: actions/cache@v3
      with:
        path: ~/.nuget/packages
        key: ${{ runner.os }}-nuget-${{ hashFiles('**/packages.lock.json') }}
        restore-keys: |
          ${{ runner.os }}-nuget-
    
    - name: Restore dependencies
      run: dotnet restore
    
    - name: Build
      run: dotnet build --no-restore --configuration Release
    
    - name: Run unit tests with coverage
      run: |
        dotnet test --no-build --configuration Release \
          --collect:"XPlat Code Coverage" \
          --results-directory ./coverage
    
    - name: Generate coverage report
      uses: danielpalme/ReportGenerator-GitHub-Action@5
      with:
        reports: coverage/**/coverage.cobertura.xml
        targetdir: coveragereport
        reporttypes: HtmlInline_AzurePipelines;Cobertura
    
    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: ./coveragereport/Cobertura.xml
        flags: unittests
        name: codecov-umbrella
```

### 코드 품질 분석

```yaml
  code-quality:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup .NET
      uses: actions/setup-dotnet@v4
      with:
        dotnet-version: ${{ env.DOTNET_VERSION }}
    
    - name: Install SonarScanner
      run: |
        dotnet tool install --global dotnet-sonarscanner
        dotnet tool install --global dotnet-coverage
    
    - name: SonarCloud Scan
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      run: |
        dotnet sonarscanner begin \
          /k:"myorg_myapp" \
          /o:"myorg" \
          /d:sonar.host.url="https://sonarcloud.io" \
          /d:sonar.login="${{ secrets.SONAR_TOKEN }}" \
          /d:sonar.cs.vscoveragexml.reportsPaths=coverage.xml
        
        dotnet build --no-incremental
        dotnet-coverage collect 'dotnet test' -f xml -o 'coverage.xml'
        
        dotnet sonarscanner end /d:sonar.login="${{ secrets.SONAR_TOKEN }}"
    
    - name: Run Security Scan
      run: |
        dotnet tool install --global security-scan
        security-scan -p ./src/MyApp.csproj
```

## Azure DevOps 파이프라인

### 빌드 파이프라인

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
    - main
    - develop
    - feature/*
  paths:
    exclude:
    - docs/*
    - README.md

pool:
  vmImage: 'ubuntu-latest'

variables:
  buildConfiguration: 'Release'
  dotnetVersion: '8.0.x'
  
stages:
- stage: Build
  displayName: 'Build and Test'
  jobs:
  - job: BuildJob
    displayName: 'Build Job'
    steps:
    - task: UseDotNet@2
      displayName: 'Use .NET SDK'
      inputs:
        packageType: 'sdk'
        version: $(dotnetVersion)
    
    - task: DotNetCoreCLI@2
      displayName: 'Restore packages'
      inputs:
        command: 'restore'
        projects: '**/*.csproj'
        feedsToUse: 'select'
        vstsFeed: 'my-feed'
    
    - task: DotNetCoreCLI@2
      displayName: 'Build solution'
      inputs:
        command: 'build'
        projects: '**/*.csproj'
        arguments: '--configuration $(buildConfiguration) --no-restore'
    
    - task: DotNetCoreCLI@2
      displayName: 'Run tests'
      inputs:
        command: 'test'
        projects: '**/*Tests.csproj'
        arguments: '--configuration $(buildConfiguration) --no-build --collect:"XPlat Code Coverage"'
        publishTestResults: true
    
    - task: PublishCodeCoverageResults@1
      displayName: 'Publish code coverage'
      inputs:
        codeCoverageTool: 'Cobertura'
        summaryFileLocation: '$(Agent.TempDirectory)/**/coverage.cobertura.xml'
```

### 릴리즈 파이프라인

```yaml
- stage: Package
  displayName: 'Package Application'
  dependsOn: Build
  jobs:
  - job: PackageJob
    displayName: 'Create Deployment Package'
    steps:
    - task: DotNetCoreCLI@2
      displayName: 'Publish Web App'
      inputs:
        command: 'publish'
        publishWebProjects: true
        arguments: '--configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)'
        zipAfterPublish: true
    
    - task: PublishPipelineArtifact@1
      displayName: 'Publish Artifact'
      inputs:
        targetPath: '$(Build.ArtifactStagingDirectory)'
        artifact: 'webapp'
        publishLocation: 'pipeline'

- stage: DeployDev
  displayName: 'Deploy to Development'
  dependsOn: Package
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/develop'))
  jobs:
  - deployment: DeployToDev
    displayName: 'Deploy to Dev Environment'
    environment: 'Development'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureWebApp@1
            displayName: 'Deploy to Azure Web App'
            inputs:
              azureSubscription: 'Azure-Dev'
              appType: 'webAppLinux'
              appName: 'myapp-dev'
              package: '$(Pipeline.Workspace)/webapp/**/*.zip'
```

## 컨테이너 이미지 빌드 및 푸시

### Docker 빌드 워크플로우

```yaml
# GitHub Actions Docker 워크플로우
  docker-build:
    runs-on: ubuntu-latest
    needs: build-and-test
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    
    - name: Log in to Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}
    
    - name: Log in to Azure Container Registry
      uses: docker/login-action@v3
      with:
        registry: myacr.azurecr.io
        username: ${{ secrets.ACR_USERNAME }}
        password: ${{ secrets.ACR_PASSWORD }}
    
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: |
          myorg/myapp
          myacr.azurecr.io/myapp
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=semver,pattern={{version}}
          type=semver,pattern={{major}}.{{minor}}
          type=sha,prefix={{branch}}-
    
    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        context: .
        file: ./Dockerfile
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
        platforms: linux/amd64,linux/arm64
```

### 멀티스테이지 Dockerfile

```dockerfile
# Dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Copy csproj and restore dependencies
COPY ["src/MyApp/MyApp.csproj", "MyApp/"]
COPY ["src/MyApp.Core/MyApp.Core.csproj", "MyApp.Core/"]
RUN dotnet restore "MyApp/MyApp.csproj"

# Copy source code and build
COPY src/ .
WORKDIR "/src/MyApp"
RUN dotnet build "MyApp.csproj" -c Release -o /app/build

# Run tests
FROM build AS test
WORKDIR /src
RUN dotnet test "MyApp.Tests/MyApp.Tests.csproj" -c Release

# Publish
FROM build AS publish
RUN dotnet publish "MyApp.csproj" -c Release -o /app/publish /p:UseAppHost=false

# Final stage
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app
EXPOSE 80
EXPOSE 443

# Install health check dependencies
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

# Copy published app
COPY --from=publish /app/publish .

# Add non-root user
RUN useradd -m -u 1001 appuser && chown -R appuser:appuser /app
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost/health || exit 1

ENTRYPOINT ["dotnet", "MyApp.dll"]
```

## 환경별 배포 전략

### 환경 구성 관리

```csharp
// appsettings.json 환경별 구성
public class EnvironmentConfig
{
    public static IConfiguration GetConfiguration(string environment)
    {
        var builder = new ConfigurationBuilder()
            .SetBasePath(Directory.GetCurrentDirectory())
            .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
            .AddJsonFile($"appsettings.{environment}.json", optional: true)
            .AddEnvironmentVariables()
            .AddAzureKeyVault(
                $"https://myapp-{environment}-kv.vault.azure.net/",
                new DefaultAzureCredential());
        
        return builder.Build();
    }
}
```

### 배포 스크립트

```yaml
# 환경별 배포 워크플로우
  deploy:
    runs-on: ubuntu-latest
    needs: docker-build
    strategy:
      matrix:
        environment: [dev, staging, production]
    
    steps:
    - name: Deploy to ${{ matrix.environment }}
      uses: azure/webapps-deploy@v2
      with:
        app-name: myapp-${{ matrix.environment }}
        publish-profile: ${{ secrets[format('AZURE_WEBAPP_PUBLISH_PROFILE_{0}', matrix.environment)] }}
        images: myacr.azurecr.io/myapp:${{ github.sha }}
    
    - name: Run smoke tests
      run: |
        npm install -g newman
        newman run postman/smoke-tests.json \
          --env-var "baseUrl=https://myapp-${{ matrix.environment }}.azurewebsites.net"
```

## Infrastructure as Code (IaC)

### Terraform을 사용한 인프라 프로비저닝

```hcl
# main.tf
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features {}
}

# Resource Group
resource "azurerm_resource_group" "main" {
  name     = "${var.project_name}-${var.environment}-rg"
  location = var.location
  
  tags = {
    Environment = var.environment
    Project     = var.project_name
    ManagedBy   = "Terraform"
  }
}

# App Service Plan
resource "azurerm_service_plan" "main" {
  name                = "${var.project_name}-${var.environment}-asp"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  os_type             = "Linux"
  sku_name            = var.app_service_sku
}

# App Service
resource "azurerm_linux_web_app" "main" {
  name                = "${var.project_name}-${var.environment}"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  service_plan_id     = azurerm_service_plan.main.id

  site_config {
    always_on = var.environment == "production" ? true : false
    
    application_stack {
      dotnet_version = "8.0"
    }
    
    health_check_path = "/health"
    
    app_command_line = ""
  }

  app_settings = {
    "ASPNETCORE_ENVIRONMENT"                = var.aspnetcore_environment
    "ApplicationInsights__InstrumentationKey" = azurerm_application_insights.main.instrumentation_key
  }

  connection_string {
    name  = "DefaultConnection"
    type  = "SQLAzure"
    value = "Server=tcp:${azurerm_mssql_server.main.fully_qualified_domain_name},1433;Initial Catalog=${azurerm_mssql_database.main.name};Persist Security Info=False;User ID=${var.db_admin_login};Password=${var.db_admin_password};MultipleActiveResultSets=False;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;"
  }
}

# Application Insights
resource "azurerm_application_insights" "main" {
  name                = "${var.project_name}-${var.environment}-ai"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  application_type    = "web"
}
```

### Bicep을 사용한 Azure 리소스 배포

```bicep
// main.bicep
param projectName string
param environment string
param location string = resourceGroup().location

@allowed([
  'F1'
  'B1'
  'B2'
  'S1'
  'P1V2'
])
param appServicePlanSku string = 'B1'

// App Service Plan
resource appServicePlan 'Microsoft.Web/serverfarms@2022-03-01' = {
  name: '${projectName}-${environment}-asp'
  location: location
  sku: {
    name: appServicePlanSku
  }
  kind: 'linux'
  properties: {
    reserved: true
  }
}

// Web App
resource webApp 'Microsoft.Web/sites@2022-03-01' = {
  name: '${projectName}-${environment}'
  location: location
  properties: {
    serverFarmId: appServicePlan.id
    siteConfig: {
      linuxFxVersion: 'DOTNETCORE|8.0'
      alwaysOn: environment == 'production'
      healthCheckPath: '/health'
      appSettings: [
        {
          name: 'ASPNETCORE_ENVIRONMENT'
          value: environment
        }
        {
          name: 'ApplicationInsights:InstrumentationKey'
          value: appInsights.properties.InstrumentationKey
        }
      ]
    }
  }
}

// Application Insights
resource appInsights 'Microsoft.Insights/components@2020-02-02' = {
  name: '${projectName}-${environment}-ai'
  location: location
  kind: 'web'
  properties: {
    Application_Type: 'web'
  }
}
```

## 보안 스캔 및 품질 게이트

### 보안 취약점 스캔

```yaml
# 보안 스캔 워크플로우
  security-scan:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: 'fs'
        scan-ref: '.'
        format: 'sarif'
        output: 'trivy-results.sarif'
        severity: 'CRITICAL,HIGH'
    
    - name: Upload Trivy scan results
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'
    
    - name: OWASP Dependency Check
      uses: dependency-check/Dependency-Check_Action@main
      with:
        project: 'MyApp'
        path: '.'
        format: 'HTML'
        args: >
          --enableRetired
          --enableExperimental
    
    - name: Upload dependency check results
      uses: actions/upload-artifact@v3
      with:
        name: dependency-check-report
        path: reports/
```

### 품질 게이트 구현

```csharp
// 품질 게이트 설정
public class QualityGate
{
    public class Thresholds
    {
        public double CodeCoverage { get; set; } = 80.0;
        public int CriticalIssues { get; set; } = 0;
        public int MajorIssues { get; set; } = 5;
        public double TechnicalDebt { get; set; } = 5.0; // days
        public double Maintainability { get; set; } = 3.0; // A-E scale
    }
    
    public static async Task<bool> EvaluateAsync(QualityMetrics metrics, Thresholds thresholds)
    {
        var failures = new List<string>();
        
        if (metrics.CodeCoverage < thresholds.CodeCoverage)
            failures.Add($"Code coverage {metrics.CodeCoverage}% is below threshold {thresholds.CodeCoverage}%");
        
        if (metrics.CriticalIssues > thresholds.CriticalIssues)
            failures.Add($"Critical issues {metrics.CriticalIssues} exceeds threshold {thresholds.CriticalIssues}");
        
        if (metrics.MajorIssues > thresholds.MajorIssues)
            failures.Add($"Major issues {metrics.MajorIssues} exceeds threshold {thresholds.MajorIssues}");
        
        if (failures.Any())
        {
            Console.WriteLine("Quality gate failed:");
            failures.ForEach(f => Console.WriteLine($"  - {f}"));
            return false;
        }
        
        return true;
    }
}
```

## 릴리즈 관리 전략

### 블루-그린 배포

```yaml
# 블루-그린 배포 스크립트
  blue-green-deploy:
    runs-on: ubuntu-latest
    
    steps:
    - name: Deploy to Green environment
      run: |
        az webapp deployment slot create \
          --name myapp-prod \
          --resource-group myapp-rg \
          --slot green
        
        az webapp deployment slot swap \
          --name myapp-prod \
          --resource-group myapp-rg \
          --slot green \
          --target-slot production
    
    - name: Health check on Green
      run: |
        for i in {1..10}; do
          response=$(curl -s -o /dev/null -w "%{http_code}" https://myapp-prod-green.azurewebsites.net/health)
          if [ $response -eq 200 ]; then
            echo "Health check passed"
            break
          fi
          echo "Health check attempt $i failed"
          sleep 30
        done
    
    - name: Swap slots
      if: success()
      run: |
        az webapp deployment slot swap \
          --name myapp-prod \
          --resource-group myapp-rg \
          --slot green
```

### 카나리 릴리즈

```csharp
// 카나리 릴리즈 구성
public class CanaryReleaseMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IConfiguration _configuration;
    
    public CanaryReleaseMiddleware(RequestDelegate next, IConfiguration configuration)
    {
        _next = next;
        _configuration = configuration;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        var canaryPercentage = _configuration.GetValue<int>("CanaryRelease:Percentage");
        var isCanaryUser = DetermineCanaryUser(context, canaryPercentage);
        
        if (isCanaryUser)
        {
            context.Items["IsCanaryRelease"] = true;
            context.Response.Headers.Add("X-Canary-Release", "true");
        }
        
        await _next(context);
    }
    
    private bool DetermineCanaryUser(HttpContext context, int percentage)
    {
        // 사용자 ID 기반 일관된 라우팅
        var userId = context.User?.FindFirst("sub")?.Value ?? context.Connection.Id;
        var hash = userId.GetHashCode();
        return Math.Abs(hash) % 100 < percentage;
    }
}
```

### Feature Flags

```csharp
// Feature Flag 서비스
public interface IFeatureFlagService
{
    Task<bool> IsEnabledAsync(string feature, ClaimsPrincipal user = null);
}

public class FeatureFlagService : IFeatureFlagService
{
    private readonly IConfiguration _configuration;
    private readonly ILogger<FeatureFlagService> _logger;
    
    public FeatureFlagService(IConfiguration configuration, ILogger<FeatureFlagService> logger)
    {
        _configuration = configuration;
        _logger = logger;
    }
    
    public async Task<bool> IsEnabledAsync(string feature, ClaimsPrincipal user = null)
    {
        try
        {
            var featureConfig = _configuration.GetSection($"Features:{feature}");
            
            if (!featureConfig.Exists())
                return false;
            
            var enabled = featureConfig.GetValue<bool>("Enabled");
            if (!enabled)
                return false;
            
            // 사용자 그룹 확인
            var allowedGroups = featureConfig.GetSection("AllowedGroups").Get<string[]>();
            if (allowedGroups?.Any() == true && user != null)
            {
                var userGroups = user.Claims
                    .Where(c => c.Type == "groups")
                    .Select(c => c.Value);
                
                return allowedGroups.Intersect(userGroups).Any();
            }
            
            // 롤아웃 퍼센티지 확인
            var rolloutPercentage = featureConfig.GetValue<int>("RolloutPercentage");
            if (rolloutPercentage > 0 && rolloutPercentage < 100)
            {
                var userId = user?.FindFirst("sub")?.Value ?? "anonymous";
                var hash = userId.GetHashCode();
                return Math.Abs(hash) % 100 < rolloutPercentage;
            }
            
            return enabled;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error checking feature flag {Feature}", feature);
            return false;
        }
    }
}
```

## 모니터링 및 롤백

### 배포 모니터링

```csharp
// 배포 상태 모니터링
public class DeploymentMonitor
{
    private readonly ILogger<DeploymentMonitor> _logger;
    private readonly ITelemetryClient _telemetryClient;
    private readonly IHealthCheckService _healthCheckService;
    
    public async Task<DeploymentStatus> MonitorDeploymentAsync(string deploymentId, CancellationToken cancellationToken)
    {
        var startTime = DateTime.UtcNow;
        var timeout = TimeSpan.FromMinutes(30);
        
        while (DateTime.UtcNow - startTime < timeout)
        {
            var health = await _healthCheckService.CheckHealthAsync(cancellationToken);
            
            if (health.Status == HealthStatus.Healthy)
            {
                _logger.LogInformation("Deployment {DeploymentId} is healthy", deploymentId);
                
                // 메트릭 확인
                var metrics = await CheckDeploymentMetricsAsync(deploymentId);
                if (metrics.AreHealthy)
                {
                    return DeploymentStatus.Success;
                }
            }
            else if (health.Status == HealthStatus.Unhealthy)
            {
                _logger.LogError("Deployment {DeploymentId} is unhealthy", deploymentId);
                return DeploymentStatus.Failed;
            }
            
            await Task.Delay(TimeSpan.FromSeconds(30), cancellationToken);
        }
        
        return DeploymentStatus.Timeout;
    }
    
    private async Task<DeploymentMetrics> CheckDeploymentMetricsAsync(string deploymentId)
    {
        var metrics = new DeploymentMetrics
        {
            ErrorRate = await _telemetryClient.GetErrorRateAsync(TimeSpan.FromMinutes(5)),
            ResponseTime = await _telemetryClient.GetAverageResponseTimeAsync(TimeSpan.FromMinutes(5)),
            Throughput = await _telemetryClient.GetThroughputAsync(TimeSpan.FromMinutes(5))
        };
        
        metrics.AreHealthy = metrics.ErrorRate < 0.01 && 
                            metrics.ResponseTime < TimeSpan.FromSeconds(2) &&
                            metrics.Throughput > 100;
        
        return metrics;
    }
}
```

### 자동 롤백 구현

```yaml
# 자동 롤백 워크플로우
  monitor-and-rollback:
    runs-on: ubuntu-latest
    needs: deploy
    
    steps:
    - name: Monitor deployment health
      id: health-check
      run: |
        for i in {1..20}; do
          echo "Health check attempt $i"
          
          # API 헬스 체크
          api_health=$(curl -s -o /dev/null -w "%{http_code}" https://myapp.com/health)
          
          # 메트릭 확인
          error_rate=$(curl -s https://api.appinsights.io/v1/apps/$APP_ID/metrics/requests/failed \
            -H "X-Api-Key: $API_KEY" | jq '.value')
          
          if [ $api_health -eq 200 ] && [ $(echo "$error_rate < 0.01" | bc) -eq 1 ]; then
            echo "Deployment is healthy"
            echo "::set-output name=status::healthy"
            exit 0
          fi
          
          sleep 30
        done
        
        echo "::set-output name=status::unhealthy"
        exit 1
    
    - name: Rollback if unhealthy
      if: steps.health-check.outputs.status == 'unhealthy'
      run: |
        echo "Deployment unhealthy, initiating rollback"
        
        # Azure App Service 슬롯 스왑 롤백
        az webapp deployment slot swap \
          --name myapp \
          --resource-group myapp-rg \
          --slot staging \
          --target-slot production
        
        # 알림 전송
        curl -X POST $SLACK_WEBHOOK \
          -H 'Content-type: application/json' \
          --data '{"text":"🚨 Deployment rollback initiated for version ${{ github.sha }}"}'
```

### 롤백 전략

```csharp
// 롤백 관리자
public class RollbackManager
{
    private readonly IDeploymentService _deploymentService;
    private readonly ILogger<RollbackManager> _logger;
    private readonly INotificationService _notificationService;
    
    public async Task<RollbackResult> RollbackAsync(string deploymentId, RollbackReason reason)
    {
        try
        {
            _logger.LogWarning("Initiating rollback for deployment {DeploymentId}, reason: {Reason}", 
                deploymentId, reason);
            
            // 이전 버전 정보 가져오기
            var previousVersion = await _deploymentService.GetPreviousVersionAsync(deploymentId);
            if (previousVersion == null)
            {
                throw new InvalidOperationException("No previous version found for rollback");
            }
            
            // 롤백 실행
            var rollbackSteps = new[]
            {
                new RollbackStep("Swap deployment slots", async () => 
                    await _deploymentService.SwapSlotsAsync(previousVersion.SlotName)),
                
                new RollbackStep("Update traffic routing", async () => 
                    await _deploymentService.UpdateTrafficRoutingAsync(previousVersion.Version, 100)),
                
                new RollbackStep("Verify rollback health", async () => 
                    await VerifyRollbackHealthAsync(previousVersion)),
                
                new RollbackStep("Clean up failed deployment", async () => 
                    await _deploymentService.CleanupFailedDeploymentAsync(deploymentId))
            };
            
            foreach (var step in rollbackSteps)
            {
                try
                {
                    _logger.LogInformation("Executing rollback step: {StepName}", step.Name);
                    await step.ExecuteAsync();
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex, "Rollback step failed: {StepName}", step.Name);
                    throw new RollbackException($"Rollback failed at step: {step.Name}", ex);
                }
            }
            
            // 알림 전송
            await _notificationService.SendAsync(new RollbackNotification
            {
                DeploymentId = deploymentId,
                PreviousVersion = previousVersion.Version,
                Reason = reason,
                Status = "Completed",
                Timestamp = DateTime.UtcNow
            });
            
            return new RollbackResult
            {
                Success = true,
                RolledBackToVersion = previousVersion.Version,
                CompletedAt = DateTime.UtcNow
            };
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Rollback failed for deployment {DeploymentId}", deploymentId);
            
            await _notificationService.SendAsync(new RollbackNotification
            {
                DeploymentId = deploymentId,
                Reason = reason,
                Status = "Failed",
                Error = ex.Message,
                Timestamp = DateTime.UtcNow
            });
            
            return new RollbackResult
            {
                Success = false,
                Error = ex.Message
            };
        }
    }
    
    private async Task VerifyRollbackHealthAsync(PreviousVersion version)
    {
        var retries = 5;
        var delay = TimeSpan.FromSeconds(10);
        
        for (int i = 0; i < retries; i++)
        {
            var health = await _deploymentService.CheckHealthAsync(version.Endpoint);
            if (health.IsHealthy)
            {
                _logger.LogInformation("Rollback health check passed");
                return;
            }
            
            await Task.Delay(delay);
        }
        
        throw new RollbackException("Rollback health verification failed");
    }
}
```

## 결론

CI/CD 파이프라인은 현대적인 ASP.NET Core 애플리케이션 개발의 핵심입니다. 자동화된 빌드, 테스트, 배포 프로세스를 통해 더 빠르고 안정적인 소프트웨어 딜리버리가 가능합니다. GitHub Actions, Azure DevOps와 같은 도구를 활용하고, 컨테이너화, IaC, 보안 스캔, 모니터링 등의 베스트 프랙티스를 적용하여 견고한 CI/CD 파이프라인을 구축할 수 있습니다.