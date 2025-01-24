# Kubernetes 배포

## 개요

Kubernetes는 컨테이너화된 애플리케이션의 배포, 확장 및 관리를 자동화하는 오픈소스 플랫폼입니다. ASP.NET Core 애플리케이션을 Kubernetes에 배포하면 높은 가용성, 자동 확장, 자가 치유 등의 이점을 얻을 수 있습니다.

## Kubernetes 기본 개념

### 아키텍처 구성 요소

Kubernetes 클러스터는 마스터 노드와 워커 노드로 구성됩니다:

```yaml
# Kubernetes 아키텍처
Master Node:
  - API Server: 클러스터 관리 API 제공
  - Controller Manager: 리소스 상태 관리
  - Scheduler: Pod 배치 결정
  - etcd: 클러스터 데이터 저장소

Worker Node:
  - kubelet: 컨테이너 실행 관리
  - kube-proxy: 네트워크 프록시
  - Container Runtime: Docker/containerd
```

### 핵심 리소스

```yaml
# Pod: 가장 작은 배포 단위
apiVersion: v1
kind: Pod
metadata:
  name: aspnetcore-pod
spec:
  containers:
  - name: aspnetcore-app
    image: myapp:latest
    ports:
    - containerPort: 80
```

## ASP.NET Core 애플리케이션 준비

### Dockerfile 작성

```dockerfile
# Multi-stage build for ASP.NET Core
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Copy project files
COPY ["MyApp.csproj", "./"]
RUN dotnet restore "MyApp.csproj"

# Copy source code and build
COPY . .
RUN dotnet build "MyApp.csproj" -c Release -o /app/build
RUN dotnet publish "MyApp.csproj" -c Release -o /app/publish

# Runtime image
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app
EXPOSE 80
EXPOSE 443

# Install curl for health checks
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

# Copy published app
COPY --from=build /app/publish .

# Set environment variable
ENV ASPNETCORE_URLS=http://+:80

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:80/health || exit 1

ENTRYPOINT ["dotnet", "MyApp.dll"]
```

### Program.cs 설정

```csharp
var builder = WebApplication.CreateBuilder(args);

// Kubernetes 환경을 위한 설정
builder.WebHost.ConfigureKestrel(serverOptions =>
{
    serverOptions.ListenAnyIP(80);
    if (builder.Environment.IsProduction())
    {
        serverOptions.ListenAnyIP(443, listenOptions =>
        {
            listenOptions.UseHttps();
        });
    }
});

// 서비스 추가
builder.Services.AddControllers();
builder.Services.AddHealthChecks();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Kubernetes 환경 변수 읽기
builder.Configuration.AddEnvironmentVariables();

var app = builder.Build();

// 미들웨어 파이프라인
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

// Health check 엔드포인트
app.MapHealthChecks("/health");
app.MapHealthChecks("/ready");

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

## Deployment 생성

### 기본 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aspnetcore-deployment
  labels:
    app: aspnetcore
spec:
  replicas: 3
  selector:
    matchLabels:
      app: aspnetcore
  template:
    metadata:
      labels:
        app: aspnetcore
    spec:
      containers:
      - name: aspnetcore
        image: myregistry.azurecr.io/aspnetcore-app:v1.0.0
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        env:
        - name: ASPNETCORE_ENVIRONMENT
          value: "Production"
        - name: ConnectionStrings__DefaultConnection
          valueFrom:
            secretKeyRef:
              name: aspnetcore-secrets
              key: connection-string
```

### Rolling Update 전략

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aspnetcore-deployment
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 1
  template:
    spec:
      containers:
      - name: aspnetcore
        image: myregistry.azurecr.io/aspnetcore-app:v2.0.0
        readinessProbe:
          httpGet:
            path: /ready
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
```

## Service 구성

### ClusterIP Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: aspnetcore-service
spec:
  selector:
    app: aspnetcore
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
```

### LoadBalancer Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: aspnetcore-loadbalancer
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
spec:
  type: LoadBalancer
  selector:
    app: aspnetcore
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    name: http
  - port: 443
    targetPort: 443
    protocol: TCP
    name: https
```

## ConfigMap과 Secret 관리

### ConfigMap 생성

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aspnetcore-config
data:
  appsettings.Production.json: |
    {
      "Logging": {
        "LogLevel": {
          "Default": "Information",
          "Microsoft.AspNetCore": "Warning"
        }
      },
      "AppSettings": {
        "Title": "Production App",
        "MaxUploadSize": "10485760",
        "CacheExpiration": "3600"
      }
    }
  logging.json: |
    {
      "Serilog": {
        "MinimumLevel": "Information",
        "WriteTo": [
          {
            "Name": "Console"
          }
        ]
      }
    }
```

### Secret 생성

```bash
# Secret 생성
kubectl create secret generic aspnetcore-secrets \
  --from-literal=connection-string="Server=myserver;Database=mydb;User Id=sa;Password=MyPass123!" \
  --from-literal=jwt-key="MySuper$ecretKey123456789012345678901234567890"
```

### Deployment에서 ConfigMap과 Secret 사용

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aspnetcore-deployment
spec:
  template:
    spec:
      containers:
      - name: aspnetcore
        image: myregistry.azurecr.io/aspnetcore-app:v1.0.0
        volumeMounts:
        - name: config-volume
          mountPath: /app/config
          readOnly: true
        - name: secret-volume
          mountPath: /app/secrets
          readOnly: true
        env:
        - name: ASPNETCORE_ENVIRONMENT
          value: "Production"
        envFrom:
        - configMapRef:
            name: aspnetcore-env-config
        - secretRef:
            name: aspnetcore-env-secrets
      volumes:
      - name: config-volume
        configMap:
          name: aspnetcore-config
      - name: secret-volume
        secret:
          secretName: aspnetcore-secrets
```

### ASP.NET Core에서 설정 읽기

```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        // ConfigMap에서 설정 읽기
        var configPath = "/app/config/appsettings.Production.json";
        if (File.Exists(configPath))
        {
            var config = new ConfigurationBuilder()
                .AddJsonFile(configPath)
                .Build();
            
            services.Configure<AppSettings>(config.GetSection("AppSettings"));
        }

        // Secret에서 값 읽기
        var connectionString = Environment.GetEnvironmentVariable("ConnectionStrings__DefaultConnection");
        services.AddDbContext<ApplicationDbContext>(options =>
            options.UseSqlServer(connectionString));
    }
}
```

## Health Checks와 Probes

### Health Check 구현

```csharp
public class DatabaseHealthCheck : IHealthCheck
{
    private readonly ApplicationDbContext _context;

    public DatabaseHealthCheck(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            await _context.Database.CanConnectAsync(cancellationToken);
            return HealthCheckResult.Healthy("Database is accessible");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Database is not accessible", ex);
        }
    }
}

// Program.cs
builder.Services.AddHealthChecks()
    .AddDbContextCheck<ApplicationDbContext>()
    .AddUrlGroup(new Uri("https://api.dependency.com/health"), "External API")
    .AddCheck<DatabaseHealthCheck>("database");

app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.MapHealthChecks("/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});
```

### Kubernetes Probes 설정

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aspnetcore-deployment
spec:
  template:
    spec:
      containers:
      - name: aspnetcore
        image: myregistry.azurecr.io/aspnetcore-app:v1.0.0
        ports:
        - containerPort: 80
        livenessProbe:
          httpGet:
            path: /health
            port: 80
            httpHeaders:
            - name: X-Health-Check
              value: liveness
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3
        startupProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 0
          periodSeconds: 10
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 30
```

## Horizontal Pod Autoscaling (HPA)

### HPA 설정

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: aspnetcore-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: aspnetcore-deployment
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max
```

### 사용자 정의 메트릭 사용

```csharp
// Prometheus 메트릭 설정
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddSingleton<IMetrics>(new MetricsBuilder()
            .OutputMetrics.AsPrometheusPlainText()
            .Build());
    }

    public void Configure(IApplicationBuilder app, IMetrics metrics)
    {
        app.UseMetricsAllMiddleware();
        app.UseMetricsActiveRequestMiddleware();
        app.UseMetricsErrorTrackingMiddleware();
        app.UseMetricsPostAndPutSizeTrackingMiddleware();
        app.UseMetricsRequestTrackingMiddleware();
        app.UseMetricsResponseTrackingMiddleware();
    }
}
```

## Ingress Controller와 라우팅

### NGINX Ingress Controller 설정

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: aspnetcore-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    - app.example.com
    secretName: aspnetcore-tls
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: aspnetcore-api-service
            port:
              number: 80
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: aspnetcore-web-service
            port:
              number: 80
```

### Path 기반 라우팅

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: aspnetcore-path-ingress
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api/v1
        pathType: Prefix
        backend:
          service:
            name: api-v1-service
            port:
              number: 80
      - path: /api/v2
        pathType: Prefix
        backend:
          service:
            name: api-v2-service
            port:
              number: 80
      - path: /admin
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

## Persistent Volumes와 StatefulSets

### PersistentVolume과 PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: aspnetcore-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: managed-premium
  csi:
    driver: disk.csi.azure.com
    volumeHandle: /subscriptions/xxx/resourceGroups/xxx/providers/Microsoft.Compute/disks/xxx
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: aspnetcore-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: managed-premium
  resources:
    requests:
      storage: 10Gi
```

### StatefulSet 구성

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: aspnetcore-statefulset
spec:
  serviceName: aspnetcore-stateful-service
  replicas: 3
  selector:
    matchLabels:
      app: aspnetcore-stateful
  template:
    metadata:
      labels:
        app: aspnetcore-stateful
    spec:
      containers:
      - name: aspnetcore
        image: myregistry.azurecr.io/aspnetcore-app:v1.0.0
        ports:
        - containerPort: 80
        volumeMounts:
        - name: data
          mountPath: /app/data
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "managed-premium"
      resources:
        requests:
          storage: 10Gi
```

### ASP.NET Core에서 Persistent Storage 사용

```csharp
public class FileStorageService
{
    private readonly string _dataPath;
    private readonly ILogger<FileStorageService> _logger;

    public FileStorageService(IConfiguration configuration, ILogger<FileStorageService> logger)
    {
        _dataPath = configuration["Storage:DataPath"] ?? "/app/data";
        _logger = logger;
        
        // Pod 이름을 사용하여 고유 디렉토리 생성
        var podName = Environment.GetEnvironmentVariable("POD_NAME");
        if (!string.IsNullOrEmpty(podName))
        {
            _dataPath = Path.Combine(_dataPath, podName);
        }
        
        Directory.CreateDirectory(_dataPath);
    }

    public async Task SaveFileAsync(string fileName, Stream fileStream)
    {
        var filePath = Path.Combine(_dataPath, fileName);
        
        using (var fileStreamOutput = new FileStream(filePath, FileMode.Create))
        {
            await fileStream.CopyToAsync(fileStreamOutput);
        }
        
        _logger.LogInformation($"File saved: {filePath}");
    }

    public async Task<Stream> GetFileAsync(string fileName)
    {
        var filePath = Path.Combine(_dataPath, fileName);
        
        if (!File.Exists(filePath))
        {
            throw new FileNotFoundException($"File not found: {fileName}");
        }
        
        return new FileStream(filePath, FileMode.Open, FileAccess.Read);
    }
}
```

## Helm Charts

### Chart 구조

```bash
aspnetcore-chart/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── hpa.yaml
│   └── _helpers.tpl
└── charts/
```

### Chart.yaml

```yaml
apiVersion: v2
name: aspnetcore-app
description: A Helm chart for ASP.NET Core application
type: application
version: 1.0.0
appVersion: "1.0.0"
keywords:
  - aspnetcore
  - dotnet
  - api
maintainers:
  - name: Your Name
    email: your.email@example.com
```

### values.yaml

```yaml
replicaCount: 3

image:
  repository: myregistry.azurecr.io/aspnetcore-app
  pullPolicy: IfNotPresent
  tag: ""

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  annotations: {}
  name: ""

podAnnotations: {}

podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 2000

securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  runAsUser: 1000

service:
  type: ClusterIP
  port: 80
  targetPort: 80

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: api.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: aspnetcore-tls
      hosts:
        - api.example.com

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

nodeSelector: {}

tolerations: []

affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - aspnetcore
        topologyKey: kubernetes.io/hostname

config:
  ASPNETCORE_ENVIRONMENT: Production
  Logging__LogLevel__Default: Information

secrets:
  ConnectionStrings__DefaultConnection: ""
  JWT__Key: ""
```

### templates/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "aspnetcore-app.fullname" . }}
  labels:
    {{- include "aspnetcore-app.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "aspnetcore-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        checksum/secret: {{ include (print $.Template.BasePath "/secret.yaml") . | sha256sum }}
      {{- with .Values.podAnnotations }}
        {{- toYaml . | nindent 8 }}
      {{- end }}
      labels:
        {{- include "aspnetcore-app.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "aspnetcore-app.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          envFrom:
            - configMapRef:
                name: {{ include "aspnetcore-app.fullname" . }}
            - secretRef:
                name: {{ include "aspnetcore-app.fullname" . }}
          volumeMounts:
            - name: temp
              mountPath: /tmp
            - name: cache
              mountPath: /app/.aspnet
      volumes:
        - name: temp
          emptyDir: {}
        - name: cache
          emptyDir: {}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

### Helm 배포 명령

```bash
# Chart 설치
helm install aspnetcore-app ./aspnetcore-chart \
  --namespace production \
  --create-namespace \
  --values ./values-prod.yaml

# Chart 업그레이드
helm upgrade aspnetcore-app ./aspnetcore-chart \
  --namespace production \
  --values ./values-prod.yaml \
  --set image.tag=v2.0.0

# Rollback
helm rollback aspnetcore-app 1 --namespace production

# 상태 확인
helm status aspnetcore-app --namespace production
```

## Service Mesh 통합 (Istio)

### Istio 설치 및 네임스페이스 설정

```bash
# Istio 설치
istioctl install --set profile=demo -y

# 네임스페이스에 자동 사이드카 주입 활성화
kubectl label namespace production istio-injection=enabled
```

### VirtualService와 DestinationRule

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: aspnetcore-vs
spec:
  hosts:
  - api.example.com
  gateways:
  - aspnetcore-gateway
  http:
  - match:
    - uri:
        prefix: /v1
    route:
    - destination:
        host: aspnetcore-service
        subset: v1
      weight: 80
    - destination:
        host: aspnetcore-service
        subset: v2
      weight: 20
  - match:
    - uri:
        prefix: /v2
    route:
    - destination:
        host: aspnetcore-service
        subset: v2
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: aspnetcore-dr
spec:
  host: aspnetcore-service
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 50
        h2MaxRequests: 100
    loadBalancer:
      simple: LEAST_CONN
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
```

### Circuit Breaker와 Retry

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: aspnetcore-resilience
spec:
  hosts:
  - aspnetcore-service
  http:
  - timeout: 30s
    retries:
      attempts: 3
      perTryTimeout: 10s
      retryOn: 5xx,reset,connect-failure,refused-stream
    fault:
      delay:
        percentage:
          value: 0.1
        fixedDelay: 5s
    route:
    - destination:
        host: aspnetcore-service
```

### mTLS 설정

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: aspnetcore-mtls
  namespace: production
spec:
  host: "*.production.svc.cluster.local"
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
```

## 프로덕션 모범 사례

### 보안 강화

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: aspnetcore-secure
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: aspnetcore
    image: myapp:latest
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /app/.aspnet
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
```

### NetworkPolicy 설정

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: aspnetcore-network-policy
spec:
  podSelector:
    matchLabels:
      app: aspnetcore
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: database
    ports:
    - protocol: TCP
      port: 1433
  - to:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```

### 리소스 최적화

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
spec:
  hard:
    requests.cpu: "100"
    requests.memory: 200Gi
    limits.cpu: "200"
    limits.memory: 400Gi
    persistentvolumeclaims: "10"
    services.loadbalancers: "2"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: production-limits
spec:
  limits:
  - max:
      cpu: "2"
      memory: 4Gi
    min:
      cpu: 100m
      memory: 128Mi
    default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 250m
      memory: 256Mi
    type: Container
```

### 모니터링과 로깅

```csharp
// Prometheus 메트릭 설정
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddMetrics();
        services.AddMetricsTrackingMiddleware();
        services.AddMetricsReportingHostedService();
        
        // OpenTelemetry 설정
        services.AddOpenTelemetryTracing(builder =>
        {
            builder
                .SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("aspnetcore-app"))
                .AddAspNetCoreInstrumentation()
                .AddHttpClientInstrumentation()
                .AddSqlClientInstrumentation()
                .AddJaegerExporter(options =>
                {
                    options.AgentHost = Environment.GetEnvironmentVariable("JAEGER_AGENT_HOST") ?? "jaeger-agent";
                    options.AgentPort = 6831;
                });
        });
    }
}
```

### 배포 체크리스트

```yaml
# 프로덕션 배포 전 확인 사항
checklist:
  security:
    - [ ] Security context 설정
    - [ ] Network policies 구성
    - [ ] RBAC 권한 최소화
    - [ ] Secret 암호화
    - [ ] 이미지 취약점 스캔
  
  reliability:
    - [ ] Health checks 구현
    - [ ] Readiness/Liveness probes 설정
    - [ ] Pod disruption budget 설정
    - [ ] 리소스 요청/제한 설정
    - [ ] HPA 구성
  
  observability:
    - [ ] 로깅 구성
    - [ ] 메트릭 수집
    - [ ] 분산 추적 설정
    - [ ] 알림 규칙 구성
  
  performance:
    - [ ] 이미지 크기 최적화
    - [ ] 시작 시간 최적화
    - [ ] 캐싱 전략 구현
    - [ ] 연결 풀링 설정
```

## 마무리

Kubernetes는 ASP.NET Core 애플리케이션을 대규모로 배포하고 관리하는 데 필요한 모든 기능을 제공합니다. 컨테이너화, 오케스트레이션, 자동 확장, 자가 치유 등의 기능을 통해 안정적이고 확장 가능한 애플리케이션을 구축할 수 있습니다. Helm charts, Service mesh, 모니터링 도구들을 활용하여 프로덕션 환경에서 효과적으로 운영할 수 있습니다.