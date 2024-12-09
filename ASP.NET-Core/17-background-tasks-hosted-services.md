# 백그라운드 작업 - Hosted Services, Queues

## 백그라운드 작업 개요

백그라운드 작업은 웹 요청과 독립적으로 실행되는 작업으로, 장시간 실행되는 프로세스나 주기적인 작업을 처리하는 데 사용됩니다. ASP.NET Core는 IHostedService와 BackgroundService를 통해 백그라운드 작업을 지원합니다.

### 백그라운드 작업의 사용 사례
- **정기적인 데이터 동기화**: 외부 API와의 데이터 동기화
- **이메일 전송**: 대량 이메일 발송
- **파일 처리**: 업로드된 파일의 비동기 처리
- **캐시 갱신**: 캐시 데이터의 주기적인 업데이트
- **로그 정리**: 오래된 로그 파일 삭제

## IHostedService 인터페이스

### 기본 IHostedService 구현
```csharp
public interface IHostedService
{
    Task StartAsync(CancellationToken cancellationToken);
    Task StopAsync(CancellationToken cancellationToken);
}

// 기본 구현
public class TimedHostedService : IHostedService, IDisposable
{
    private readonly ILogger<TimedHostedService> _logger;
    private Timer _timer;
    
    public TimedHostedService(ILogger<TimedHostedService> logger)
    {
        _logger = logger;
    }
    
    public Task StartAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("Timed Hosted Service running.");
        
        _timer = new Timer(DoWork, null, TimeSpan.Zero, TimeSpan.FromSeconds(60));
        
        return Task.CompletedTask;
    }
    
    private void DoWork(object state)
    {
        _logger.LogInformation("Timed Hosted Service is working. {Time}", DateTimeOffset.Now);
    }
    
    public Task StopAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("Timed Hosted Service is stopping.");
        
        _timer?.Change(Timeout.Infinite, 0);
        
        return Task.CompletedTask;
    }
    
    public void Dispose()
    {
        _timer?.Dispose();
    }
}

// Program.cs에 등록
builder.Services.AddHostedService<TimedHostedService>();
```

### Scoped 서비스 사용
```csharp
public class ScopedBackgroundService : IHostedService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<ScopedBackgroundService> _logger;
    private Timer _timer;
    
    public ScopedBackgroundService(
        IServiceProvider serviceProvider,
        ILogger<ScopedBackgroundService> logger)
    {
        _serviceProvider = serviceProvider;
        _logger = logger;
    }
    
    public Task StartAsync(CancellationToken cancellationToken)
    {
        _timer = new Timer(DoWork, null, TimeSpan.Zero, TimeSpan.FromMinutes(5));
        return Task.CompletedTask;
    }
    
    private async void DoWork(object state)
    {
        using (var scope = _serviceProvider.CreateScope())
        {
            try
            {
                var dbContext = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
                var emailService = scope.ServiceProvider.GetRequiredService<IEmailService>();
                
                // 미발송 이메일 처리
                var pendingEmails = await dbContext.EmailQueue
                    .Where(e => !e.IsSent && e.RetryCount < 3)
                    .Take(100)
                    .ToListAsync();
                
                foreach (var email in pendingEmails)
                {
                    try
                    {
                        await emailService.SendAsync(email);
                        email.IsSent = true;
                        email.SentAt = DateTime.UtcNow;
                    }
                    catch (Exception ex)
                    {
                        _logger.LogError(ex, "Failed to send email {Id}", email.Id);
                        email.RetryCount++;
                        email.LastError = ex.Message;
                    }
                }
                
                await dbContext.SaveChangesAsync();
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error in email processing");
            }
        }
    }
    
    public Task StopAsync(CancellationToken cancellationToken)
    {
        _timer?.Change(Timeout.Infinite, 0);
        return Task.CompletedTask;
    }
}
```

## BackgroundService 추상 클래스

### 기본 BackgroundService 구현
```csharp
public class LongRunningService : BackgroundService
{
    private readonly ILogger<LongRunningService> _logger;
    
    public LongRunningService(ILogger<LongRunningService> logger)
    {
        _logger = logger;
    }
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Long Running Service started");
        
        while (!stoppingToken.IsCancellationRequested)
        {
            _logger.LogInformation("Long Running Service is working. {Time}", DateTimeOffset.Now);
            
            try
            {
                await DoWorkAsync(stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error occurred executing work");
            }
            
            await Task.Delay(TimeSpan.FromMinutes(1), stoppingToken);
        }
        
        _logger.LogInformation("Long Running Service is stopping");
    }
    
    private async Task DoWorkAsync(CancellationToken cancellationToken)
    {
        // 실제 작업 수행
        await Task.Delay(1000, cancellationToken);
    }
    
    public override async Task StopAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("Long Running Service is stopping gracefully");
        await base.StopAsync(cancellationToken);
    }
}
```

### 주기적인 데이터 정리 서비스
```csharp
public class DataCleanupService : BackgroundService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<DataCleanupService> _logger;
    private readonly IConfiguration _configuration;
    
    public DataCleanupService(
        IServiceProvider serviceProvider,
        ILogger<DataCleanupService> logger,
        IConfiguration configuration)
    {
        _serviceProvider = serviceProvider;
        _logger = logger;
        _configuration = configuration;
    }
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        var cleanupInterval = _configuration.GetValue<int>("Cleanup:IntervalHours", 24);
        
        while (!stoppingToken.IsCancellationRequested)
        {
            _logger.LogInformation("Starting data cleanup at {Time}", DateTimeOffset.Now);
            
            try
            {
                await CleanupOldDataAsync(stoppingToken);
                await CleanupTempFilesAsync(stoppingToken);
                await CleanupExpiredTokensAsync(stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error during cleanup");
            }
            
            await Task.Delay(TimeSpan.FromHours(cleanupInterval), stoppingToken);
        }
    }
    
    private async Task CleanupOldDataAsync(CancellationToken cancellationToken)
    {
        using var scope = _serviceProvider.CreateScope();
        var dbContext = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
        
        var cutoffDate = DateTime.UtcNow.AddDays(-90);
        
        // 오래된 로그 삭제
        var oldLogs = await dbContext.AuditLogs
            .Where(log => log.CreatedAt < cutoffDate)
            .ToListAsync(cancellationToken);
        
        if (oldLogs.Any())
        {
            dbContext.AuditLogs.RemoveRange(oldLogs);
            await dbContext.SaveChangesAsync(cancellationToken);
            _logger.LogInformation("Deleted {Count} old audit logs", oldLogs.Count);
        }
    }
    
    private async Task CleanupTempFilesAsync(CancellationToken cancellationToken)
    {
        var tempPath = Path.Combine(Directory.GetCurrentDirectory(), "temp");
        
        if (Directory.Exists(tempPath))
        {
            var cutoffDate = DateTime.UtcNow.AddHours(-24);
            var files = Directory.GetFiles(tempPath);
            var deletedCount = 0;
            
            foreach (var file in files)
            {
                if (cancellationToken.IsCancellationRequested)
                    break;
                
                var fileInfo = new FileInfo(file);
                if (fileInfo.CreationTime < cutoffDate)
                {
                    try
                    {
                        File.Delete(file);
                        deletedCount++;
                    }
                    catch (Exception ex)
                    {
                        _logger.LogWarning(ex, "Failed to delete file {File}", file);
                    }
                }
            }
            
            _logger.LogInformation("Deleted {Count} temporary files", deletedCount);
        }
    }
    
    private async Task CleanupExpiredTokensAsync(CancellationToken cancellationToken)
    {
        using var scope = _serviceProvider.CreateScope();
        var dbContext = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
        
        var expiredTokens = await dbContext.RefreshTokens
            .Where(token => token.ExpiresAt < DateTime.UtcNow)
            .ToListAsync(cancellationToken);
        
        if (expiredTokens.Any())
        {
            dbContext.RefreshTokens.RemoveRange(expiredTokens);
            await dbContext.SaveChangesAsync(cancellationToken);
            _logger.LogInformation("Deleted {Count} expired tokens", expiredTokens.Count);
        }
    }
}
```

## Queue 기반 백그라운드 작업

### In-Memory Queue 구현
```csharp
public interface IBackgroundTaskQueue
{
    ValueTask QueueBackgroundWorkItemAsync(Func<CancellationToken, ValueTask> workItem);
    ValueTask<Func<CancellationToken, ValueTask>> DequeueAsync(CancellationToken cancellationToken);
}

public class BackgroundTaskQueue : IBackgroundTaskQueue
{
    private readonly Channel<Func<CancellationToken, ValueTask>> _queue;
    
    public BackgroundTaskQueue(int capacity)
    {
        var options = new BoundedChannelOptions(capacity)
        {
            FullMode = BoundedChannelFullMode.Wait
        };
        _queue = Channel.CreateBounded<Func<CancellationToken, ValueTask>>(options);
    }
    
    public async ValueTask QueueBackgroundWorkItemAsync(
        Func<CancellationToken, ValueTask> workItem)
    {
        if (workItem == null)
        {
            throw new ArgumentNullException(nameof(workItem));
        }
        
        await _queue.Writer.WriteAsync(workItem);
    }
    
    public async ValueTask<Func<CancellationToken, ValueTask>> DequeueAsync(
        CancellationToken cancellationToken)
    {
        var workItem = await _queue.Reader.ReadAsync(cancellationToken);
        return workItem;
    }
}

// Queue 처리 서비스
public class QueuedHostedService : BackgroundService
{
    private readonly ILogger<QueuedHostedService> _logger;
    private readonly IBackgroundTaskQueue _taskQueue;
    
    public QueuedHostedService(
        IBackgroundTaskQueue taskQueue,
        ILogger<QueuedHostedService> logger)
    {
        _taskQueue = taskQueue;
        _logger = logger;
    }
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Queued Hosted Service is running.");
        
        await BackgroundProcessing(stoppingToken);
    }
    
    private async Task BackgroundProcessing(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var workItem = await _taskQueue.DequeueAsync(stoppingToken);
            
            try
            {
                await workItem(stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error occurred executing work item");
            }
        }
    }
}

// Program.cs 등록
builder.Services.AddSingleton<IBackgroundTaskQueue>(ctx =>
    new BackgroundTaskQueue(100));
builder.Services.AddHostedService<QueuedHostedService>();
```

### Queue 사용 예제
```csharp
[ApiController]
[Route("api/[controller]")]
public class OrderController : ControllerBase
{
    private readonly IBackgroundTaskQueue _queue;
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<OrderController> _logger;
    
    public OrderController(
        IBackgroundTaskQueue queue,
        IServiceProvider serviceProvider,
        ILogger<OrderController> logger)
    {
        _queue = queue;
        _serviceProvider = serviceProvider;
        _logger = logger;
    }
    
    [HttpPost]
    public async Task<IActionResult> CreateOrder([FromBody] CreateOrderDto dto)
    {
        // 주문 생성 (빠른 응답)
        var orderId = Guid.NewGuid();
        
        // 백그라운드 작업 큐에 추가
        await _queue.QueueBackgroundWorkItemAsync(async token =>
        {
            using var scope = _serviceProvider.CreateScope();
            var orderService = scope.ServiceProvider.GetRequiredService<IOrderService>();
            var emailService = scope.ServiceProvider.GetRequiredService<IEmailService>();
            
            try
            {
                // 주문 처리
                await orderService.ProcessOrderAsync(orderId, dto, token);
                
                // 확인 이메일 발송
                await emailService.SendOrderConfirmationAsync(dto.CustomerEmail, orderId);
                
                _logger.LogInformation("Order {OrderId} processed successfully", orderId);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to process order {OrderId}", orderId);
                // 실패 처리 로직
                await orderService.MarkOrderAsFailedAsync(orderId, ex.Message);
            }
        });
        
        return Accepted(new { orderId });
    }
}
```

## 고급 백그라운드 작업 패턴

### 우선순위 큐
```csharp
public interface IPriorityBackgroundTaskQueue
{
    ValueTask EnqueueAsync(BackgroundWorkItem workItem);
    ValueTask<BackgroundWorkItem> DequeueAsync(CancellationToken cancellationToken);
}

public class BackgroundWorkItem
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public Func<CancellationToken, ValueTask> WorkItem { get; set; }
    public WorkPriority Priority { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
}

public enum WorkPriority
{
    Low = 0,
    Normal = 1,
    High = 2,
    Critical = 3
}

public class PriorityBackgroundTaskQueue : IPriorityBackgroundTaskQueue
{
    private readonly SortedSet<BackgroundWorkItem> _queue;
    private readonly SemaphoreSlim _signal;
    
    public PriorityBackgroundTaskQueue()
    {
        _queue = new SortedSet<BackgroundWorkItem>(new WorkItemComparer());
        _signal = new SemaphoreSlim(0);
    }
    
    public async ValueTask EnqueueAsync(BackgroundWorkItem workItem)
    {
        if (workItem == null)
            throw new ArgumentNullException(nameof(workItem));
        
        lock (_queue)
        {
            _queue.Add(workItem);
        }
        
        _signal.Release();
    }
    
    public async ValueTask<BackgroundWorkItem> DequeueAsync(
        CancellationToken cancellationToken)
    {
        await _signal.WaitAsync(cancellationToken);
        
        lock (_queue)
        {
            var workItem = _queue.Max;
            _queue.Remove(workItem);
            return workItem;
        }
    }
    
    private class WorkItemComparer : IComparer<BackgroundWorkItem>
    {
        public int Compare(BackgroundWorkItem x, BackgroundWorkItem y)
        {
            // 우선순위가 높을수록, 생성 시간이 빠를수록 우선
            var priorityComparison = y.Priority.CompareTo(x.Priority);
            
            if (priorityComparison != 0)
                return priorityComparison;
            
            return x.CreatedAt.CompareTo(y.CreatedAt);
        }
    }
}
```

### 재시도 메커니즘
```csharp
public class RetryableBackgroundService : BackgroundService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<RetryableBackgroundService> _logger;
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            using var scope = _serviceProvider.CreateScope();
            var workItemRepository = scope.ServiceProvider
                .GetRequiredService<IWorkItemRepository>();
            
            // 실패한 작업 조회
            var failedItems = await workItemRepository.GetFailedItemsAsync();
            
            foreach (var item in failedItems)
            {
                if (stoppingToken.IsCancellationRequested)
                    break;
                
                await ProcessWithRetryAsync(item, stoppingToken);
            }
            
            await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
        }
    }
    
    private async Task ProcessWithRetryAsync(
        WorkItem item, 
        CancellationToken cancellationToken)
    {
        var retryPolicy = Policy
            .Handle<Exception>()
            .WaitAndRetryAsync(
                3,
                retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)),
                onRetry: (exception, timeSpan, retry, ctx) =>
                {
                    _logger.LogWarning(
                        "Retry {Retry} after {Delay}s for work item {Id}",
                        retry, timeSpan.TotalSeconds, item.Id);
                });
        
        try
        {
            await retryPolicy.ExecuteAsync(async () =>
            {
                await ProcessWorkItemAsync(item, cancellationToken);
            });
            
            item.Status = WorkItemStatus.Completed;
            item.CompletedAt = DateTime.UtcNow;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to process work item {Id} after retries", item.Id);
            
            item.Status = WorkItemStatus.Failed;
            item.LastError = ex.Message;
            item.RetryCount++;
            
            if (item.RetryCount >= 5)
            {
                item.Status = WorkItemStatus.DeadLetter;
            }
        }
        
        await UpdateWorkItemAsync(item);
    }
}
```

### Cron 기반 스케줄링
```csharp
public interface ICronJob
{
    string CronExpression { get; }
    Task ExecuteAsync(CancellationToken cancellationToken);
}

public class DailyReportJob : ICronJob
{
    private readonly IReportService _reportService;
    private readonly IEmailService _emailService;
    
    public string CronExpression => "0 0 9 * * ?"; // 매일 오전 9시
    
    public DailyReportJob(IReportService reportService, IEmailService emailService)
    {
        _reportService = reportService;
        _emailService = emailService;
    }
    
    public async Task ExecuteAsync(CancellationToken cancellationToken)
    {
        var report = await _reportService.GenerateDailyReportAsync();
        await _emailService.SendReportAsync(report);
    }
}

public class CronJobService : BackgroundService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<CronJobService> _logger;
    private readonly List<(Type jobType, CronExpression expression)> _cronJobs;
    
    public CronJobService(
        IServiceProvider serviceProvider,
        ILogger<CronJobService> logger)
    {
        _serviceProvider = serviceProvider;
        _logger = logger;
        _cronJobs = new List<(Type, CronExpression)>();
        
        // Cron 작업 등록
        RegisterCronJobs();
    }
    
    private void RegisterCronJobs()
    {
        var jobTypes = Assembly.GetExecutingAssembly()
            .GetTypes()
            .Where(t => t.IsClass && !t.IsAbstract && typeof(ICronJob).IsAssignableFrom(t));
        
        foreach (var jobType in jobTypes)
        {
            using var scope = _serviceProvider.CreateScope();
            var job = (ICronJob)scope.ServiceProvider.GetRequiredService(jobType);
            var expression = CronExpression.Parse(job.CronExpression);
            _cronJobs.Add((jobType, expression));
        }
    }
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var now = DateTime.UtcNow;
            var nextMinute = now.AddMinutes(1).Date.AddHours(now.Hour).AddMinutes(now.Minute);
            var delay = nextMinute - now;
            
            await Task.Delay(delay, stoppingToken);
            
            foreach (var (jobType, expression) in _cronJobs)
            {
                var nextOccurrence = expression.GetNextOccurrence(DateTime.UtcNow);
                
                if (nextOccurrence.HasValue && 
                    Math.Abs((nextOccurrence.Value - DateTime.UtcNow).TotalSeconds) < 30)
                {
                    _ = Task.Run(async () =>
                    {
                        using var scope = _serviceProvider.CreateScope();
                        var job = (ICronJob)scope.ServiceProvider.GetRequiredService(jobType);
                        
                        try
                        {
                            _logger.LogInformation("Executing cron job {JobType}", jobType.Name);
                            await job.ExecuteAsync(stoppingToken);
                        }
                        catch (Exception ex)
                        {
                            _logger.LogError(ex, "Error executing cron job {JobType}", jobType.Name);
                        }
                    }, stoppingToken);
                }
            }
        }
    }
}
```

## 분산 작업 큐

### Redis 기반 작업 큐
```csharp
public interface IDistributedTaskQueue
{
    Task<string> EnqueueAsync<T>(T message, string queue = "default") where T : class;
    Task<T> DequeueAsync<T>(string queue = "default", CancellationToken cancellationToken = default) where T : class;
}

public class RedisTaskQueue : IDistributedTaskQueue
{
    private readonly IConnectionMultiplexer _redis;
    private readonly ILogger<RedisTaskQueue> _logger;
    
    public RedisTaskQueue(IConnectionMultiplexer redis, ILogger<RedisTaskQueue> logger)
    {
        _redis = redis;
        _logger = logger;
    }
    
    public async Task<string> EnqueueAsync<T>(T message, string queue = "default") where T : class
    {
        var db = _redis.GetDatabase();
        var messageId = Guid.NewGuid().ToString();
        
        var messageData = new RedisMessage<T>
        {
            Id = messageId,
            Data = message,
            EnqueuedAt = DateTime.UtcNow,
            Queue = queue
        };
        
        var json = JsonSerializer.Serialize(messageData);
        await db.ListRightPushAsync($"queue:{queue}", json);
        
        // Pub/Sub로 알림
        await _redis.GetSubscriber().PublishAsync($"queue:{queue}:notify", messageId);
        
        _logger.LogInformation("Message {MessageId} enqueued to {Queue}", messageId, queue);
        
        return messageId;
    }
    
    public async Task<T> DequeueAsync<T>(string queue = "default", CancellationToken cancellationToken = default) where T : class
    {
        var db = _redis.GetDatabase();
        var subscriber = _redis.GetSubscriber();
        
        // Blocking pop with timeout
        while (!cancellationToken.IsCancellationRequested)
        {
            var result = await db.ListLeftPopAsync($"queue:{queue}");
            
            if (!result.IsNullOrEmpty)
            {
                var messageData = JsonSerializer.Deserialize<RedisMessage<T>>(result);
                _logger.LogInformation("Message {MessageId} dequeued from {Queue}", messageData.Id, queue);
                return messageData.Data;
            }
            
            // Pub/Sub로 새 메시지 대기
            var tcs = new TaskCompletionSource<bool>();
            
            await subscriber.SubscribeAsync($"queue:{queue}:notify", (channel, value) =>
            {
                tcs.TrySetResult(true);
            });
            
            try
            {
                await Task.WhenAny(
                    tcs.Task,
                    Task.Delay(TimeSpan.FromSeconds(30), cancellationToken)
                );
            }
            finally
            {
                await subscriber.UnsubscribeAsync($"queue:{queue}:notify");
            }
        }
        
        return null;
    }
}

public class RedisMessage<T>
{
    public string Id { get; set; }
    public T Data { get; set; }
    public DateTime EnqueuedAt { get; set; }
    public string Queue { get; set; }
}
```

### 분산 작업 처리 서비스
```csharp
public class DistributedWorkerService : BackgroundService
{
    private readonly IDistributedTaskQueue _taskQueue;
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<DistributedWorkerService> _logger;
    private readonly string _workerId;
    
    public DistributedWorkerService(
        IDistributedTaskQueue taskQueue,
        IServiceProvider serviceProvider,
        ILogger<DistributedWorkerService> logger)
    {
        _taskQueue = taskQueue;
        _serviceProvider = serviceProvider;
        _logger = logger;
        _workerId = Guid.NewGuid().ToString();
    }
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Distributed Worker {WorkerId} started", _workerId);
        
        // 여러 큐를 병렬로 처리
        var tasks = new[]
        {
            ProcessQueueAsync("high-priority", stoppingToken),
            ProcessQueueAsync("default", stoppingToken),
            ProcessQueueAsync("low-priority", stoppingToken)
        };
        
        await Task.WhenAll(tasks);
    }
    
    private async Task ProcessQueueAsync(string queueName, CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                var message = await _taskQueue.DequeueAsync<WorkMessage>(queueName, stoppingToken);
                
                if (message != null)
                {
                    await ProcessMessageAsync(message, stoppingToken);
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error processing message from queue {Queue}", queueName);
                await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
            }
        }
    }
    
    private async Task ProcessMessageAsync(WorkMessage message, CancellationToken cancellationToken)
    {
        using var scope = _serviceProvider.CreateScope();
        
        _logger.LogInformation(
            "Worker {WorkerId} processing message {MessageId} of type {Type}",
            _workerId, message.Id, message.Type);
        
        try
        {
            switch (message.Type)
            {
                case "SendEmail":
                    var emailService = scope.ServiceProvider.GetRequiredService<IEmailService>();
                    await emailService.SendAsync(message.GetData<EmailMessage>());
                    break;
                    
                case "ProcessImage":
                    var imageService = scope.ServiceProvider.GetRequiredService<IImageService>();
                    await imageService.ProcessAsync(message.GetData<ImageProcessingRequest>());
                    break;
                    
                case "GenerateReport":
                    var reportService = scope.ServiceProvider.GetRequiredService<IReportService>();
                    await reportService.GenerateAsync(message.GetData<ReportRequest>());
                    break;
                    
                default:
                    _logger.LogWarning("Unknown message type: {Type}", message.Type);
                    break;
            }
            
            _logger.LogInformation("Message {MessageId} processed successfully", message.Id);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to process message {MessageId}", message.Id);
            
            // Dead letter queue로 이동
            await _taskQueue.EnqueueAsync(
                new DeadLetterMessage
                {
                    OriginalMessage = message,
                    Error = ex.Message,
                    FailedAt = DateTime.UtcNow,
                    WorkerId = _workerId
                },
                "dead-letter");
        }
    }
}

public class WorkMessage
{
    public string Id { get; set; } = Guid.NewGuid().ToString();
    public string Type { get; set; }
    public JsonElement Data { get; set; }
    
    public T GetData<T>()
    {
        return JsonSerializer.Deserialize<T>(Data.GetRawText());
    }
}
```

## 상태 모니터링과 관리

### 백그라운드 작업 상태 추적
```csharp
public interface IBackgroundJobTracker
{
    Task<string> RegisterJobAsync(string jobName, string description);
    Task UpdateJobStatusAsync(string jobId, JobStatus status, string message = null);
    Task<IEnumerable<JobInfo>> GetActiveJobsAsync();
    Task<JobInfo> GetJobAsync(string jobId);
}

public class BackgroundJobTracker : IBackgroundJobTracker
{
    private readonly IMemoryCache _cache;
    private readonly ILogger<BackgroundJobTracker> _logger;
    
    public async Task<string> RegisterJobAsync(string jobName, string description)
    {
        var jobInfo = new JobInfo
        {
            Id = Guid.NewGuid().ToString(),
            Name = jobName,
            Description = description,
            Status = JobStatus.Running,
            StartedAt = DateTime.UtcNow,
            LastHeartbeat = DateTime.UtcNow
        };
        
        _cache.Set($"job:{jobInfo.Id}", jobInfo, TimeSpan.FromHours(24));
        
        _logger.LogInformation("Job {JobId} registered: {JobName}", jobInfo.Id, jobName);
        
        return jobInfo.Id;
    }
    
    public async Task UpdateJobStatusAsync(string jobId, JobStatus status, string message = null)
    {
        if (_cache.TryGetValue<JobInfo>($"job:{jobId}", out var jobInfo))
        {
            jobInfo.Status = status;
            jobInfo.LastHeartbeat = DateTime.UtcNow;
            jobInfo.Message = message;
            
            if (status == JobStatus.Completed || status == JobStatus.Failed)
            {
                jobInfo.CompletedAt = DateTime.UtcNow;
            }
            
            _cache.Set($"job:{jobId}", jobInfo, TimeSpan.FromHours(24));
        }
    }
    
    public async Task<IEnumerable<JobInfo>> GetActiveJobsAsync()
    {
        // 실제 구현에서는 영구 저장소 사용 권장
        var jobs = new List<JobInfo>();
        
        // 캐시에서 모든 작업 조회 (예제)
        return jobs.Where(j => j.Status == JobStatus.Running);
    }
}

public enum JobStatus
{
    Running,
    Completed,
    Failed,
    Cancelled
}

public class JobInfo
{
    public string Id { get; set; }
    public string Name { get; set; }
    public string Description { get; set; }
    public JobStatus Status { get; set; }
    public DateTime StartedAt { get; set; }
    public DateTime? CompletedAt { get; set; }
    public DateTime LastHeartbeat { get; set; }
    public string Message { get; set; }
    public Dictionary<string, object> Metadata { get; set; } = new();
}

// 모니터링 엔드포인트
[ApiController]
[Route("api/admin/jobs")]
[Authorize(Roles = "Admin")]
public class JobMonitoringController : ControllerBase
{
    private readonly IBackgroundJobTracker _jobTracker;
    
    [HttpGet]
    public async Task<IActionResult> GetActiveJobs()
    {
        var jobs = await _jobTracker.GetActiveJobsAsync();
        return Ok(jobs);
    }
    
    [HttpGet("{jobId}")]
    public async Task<IActionResult> GetJob(string jobId)
    {
        var job = await _jobTracker.GetJobAsync(jobId);
        
        if (job == null)
            return NotFound();
        
        return Ok(job);
    }
}
```

## 모범 사례

### 견고한 백그라운드 서비스
```csharp
public abstract class ResilientBackgroundService : BackgroundService
{
    private readonly ILogger _logger;
    private readonly int _maxRetries;
    private readonly TimeSpan _retryDelay;
    
    protected ResilientBackgroundService(ILogger logger, int maxRetries = 3, int retryDelaySeconds = 60)
    {
        _logger = logger;
        _maxRetries = maxRetries;
        _retryDelay = TimeSpan.FromSeconds(retryDelaySeconds);
    }
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        var retryCount = 0;
        
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await ExecuteInternalAsync(stoppingToken);
                retryCount = 0; // 성공 시 재시도 카운트 리셋
            }
            catch (Exception ex)
            {
                retryCount++;
                _logger.LogError(ex, "Error in background service. Retry {RetryCount}/{MaxRetries}", 
                    retryCount, _maxRetries);
                
                if (retryCount >= _maxRetries)
                {
                    _logger.LogCritical("Max retries reached. Service will stop.");
                    throw;
                }
                
                await Task.Delay(_retryDelay * retryCount, stoppingToken);
            }
        }
    }
    
    protected abstract Task ExecuteInternalAsync(CancellationToken stoppingToken);
}

// 구현 예제
public class ResilientEmailService : ResilientBackgroundService
{
    private readonly IServiceProvider _serviceProvider;
    
    public ResilientEmailService(
        IServiceProvider serviceProvider,
        ILogger<ResilientEmailService> logger) 
        : base(logger)
    {
        _serviceProvider = serviceProvider;
    }
    
    protected override async Task ExecuteInternalAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            using var scope = _serviceProvider.CreateScope();
            var emailService = scope.ServiceProvider.GetRequiredService<IEmailService>();
            
            await emailService.ProcessPendingEmailsAsync(stoppingToken);
            
            await Task.Delay(TimeSpan.FromMinutes(1), stoppingToken);
        }
    }
}
```

### 구성 가능한 백그라운드 서비스
```csharp
public class BackgroundServiceOptions
{
    public bool Enabled { get; set; } = true;
    public int IntervalSeconds { get; set; } = 60;
    public int BatchSize { get; set; } = 100;
    public int MaxConcurrency { get; set; } = 5;
}

public class ConfigurableBackgroundService : BackgroundService
{
    private readonly IOptionsMonitor<BackgroundServiceOptions> _options;
    private readonly ILogger<ConfigurableBackgroundService> _logger;
    
    public ConfigurableBackgroundService(
        IOptionsMonitor<BackgroundServiceOptions> options,
        ILogger<ConfigurableBackgroundService> logger)
    {
        _options = options;
        _logger = logger;
    }
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var options = _options.CurrentValue;
            
            if (!options.Enabled)
            {
                _logger.LogInformation("Service is disabled. Waiting...");
                await Task.Delay(TimeSpan.FromMinutes(1), stoppingToken);
                continue;
            }
            
            using var semaphore = new SemaphoreSlim(options.MaxConcurrency);
            var tasks = new List<Task>();
            
            for (int i = 0; i < options.BatchSize; i++)
            {
                await semaphore.WaitAsync(stoppingToken);
                
                tasks.Add(Task.Run(async () =>
                {
                    try
                    {
                        await ProcessItemAsync(stoppingToken);
                    }
                    finally
                    {
                        semaphore.Release();
                    }
                }, stoppingToken));
            }
            
            await Task.WhenAll(tasks);
            await Task.Delay(TimeSpan.FromSeconds(options.IntervalSeconds), stoppingToken);
        }
    }
    
    private async Task ProcessItemAsync(CancellationToken cancellationToken)
    {
        // 작업 처리 로직
        await Task.Delay(100, cancellationToken);
    }
}
```

백그라운드 작업은 웹 애플리케이션의 중요한 부분입니다. IHostedService와 BackgroundService를 활용하여 안정적이고 확장 가능한 백그라운드 처리 시스템을 구축할 수 있습니다.