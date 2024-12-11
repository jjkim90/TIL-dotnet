# 실시간 통신 - SignalR

## SignalR 개요

SignalR은 웹 애플리케이션에 실시간 기능을 추가할 수 있게 해주는 ASP.NET Core 라이브러리입니다. 서버와 클라이언트 간의 양방향 통신을 가능하게 하며, WebSockets를 비롯한 여러 전송 방식을 자동으로 관리합니다.

### SignalR의 주요 특징
- **자동 연결 관리**: 연결 끊김 시 자동 재연결
- **전송 방식 자동 선택**: WebSockets, Server-Sent Events, Long Polling
- **확장 가능한 메시징**: 여러 서버로 스케일 아웃 가능
- **강타입 허브**: C# 인터페이스를 통한 타입 안전성
- **연결 그룹 관리**: 사용자를 그룹으로 묶어 관리

## SignalR 설정

### 기본 설정
```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// SignalR 서비스 추가
builder.Services.AddSignalR(options =>
{
    // 클라이언트로부터 메시지를 받지 못한 경우 연결 종료 시간
    options.ClientTimeoutInterval = TimeSpan.FromSeconds(30);
    
    // 클라이언트에게 Keep-Alive 메시지 전송 간격
    options.KeepAliveInterval = TimeSpan.FromSeconds(15);
    
    // 메시지 버퍼 크기
    options.StreamBufferCapacity = 10;
    
    // 메시지 크기 제한 (null = 무제한)
    options.MaximumReceiveMessageSize = 32 * 1024; // 32KB
    
    // 병렬 호출 제한
    options.MaximumParallelInvocationsPerClient = 1;
});

// JSON 직렬화 옵션
builder.Services.AddSignalR()
    .AddJsonProtocol(options =>
    {
        options.PayloadSerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
        options.PayloadSerializerOptions.WriteIndented = false;
    });

var app = builder.Build();

// SignalR 허브 매핑
app.MapHub<ChatHub>("/hubs/chat");
app.MapHub<NotificationHub>("/hubs/notifications");

app.Run();
```

### CORS 설정
```csharp
// SignalR을 위한 CORS 설정
builder.Services.AddCors(options =>
{
    options.AddPolicy("SignalRPolicy", builder =>
    {
        builder.WithOrigins("https://localhost:3000", "https://myapp.com")
               .AllowAnyHeader()
               .AllowAnyMethod()
               .AllowCredentials(); // SignalR에는 필수
    });
});

app.UseCors("SignalRPolicy");
```

## Hub 구현

### 기본 Hub
```csharp
public class ChatHub : Hub
{
    private readonly ILogger<ChatHub> _logger;
    
    public ChatHub(ILogger<ChatHub> logger)
    {
        _logger = logger;
    }
    
    // 클라이언트가 호출할 수 있는 메서드
    public async Task SendMessage(string user, string message)
    {
        // 모든 연결된 클라이언트에게 메시지 전송
        await Clients.All.SendAsync("ReceiveMessage", user, message, DateTime.UtcNow);
        
        _logger.LogInformation("Message from {User}: {Message}", user, message);
    }
    
    // 특정 사용자에게 메시지 전송
    public async Task SendPrivateMessage(string userId, string message)
    {
        await Clients.User(userId).SendAsync("ReceivePrivateMessage", Context.UserIdentifier, message);
    }
    
    // 그룹에 메시지 전송
    public async Task SendMessageToGroup(string groupName, string message)
    {
        await Clients.Group(groupName).SendAsync("ReceiveGroupMessage", Context.UserIdentifier, message);
    }
    
    // 호출자를 제외한 모든 클라이언트에게 전송
    public async Task BroadcastMessage(string message)
    {
        await Clients.Others.SendAsync("ReceiveBroadcast", Context.UserIdentifier, message);
    }
    
    // 연결 수명 주기 이벤트
    public override async Task OnConnectedAsync()
    {
        _logger.LogInformation("Client connected: {ConnectionId}", Context.ConnectionId);
        
        // 연결된 사용자 수 업데이트
        await Clients.All.SendAsync("UserConnected", Context.ConnectionId);
        
        await base.OnConnectedAsync();
    }
    
    public override async Task OnDisconnectedAsync(Exception exception)
    {
        _logger.LogInformation("Client disconnected: {ConnectionId}", Context.ConnectionId);
        
        if (exception != null)
        {
            _logger.LogError(exception, "Client disconnected with error");
        }
        
        await Clients.All.SendAsync("UserDisconnected", Context.ConnectionId);
        
        await base.OnDisconnectedAsync(exception);
    }
}
```

### 강타입 Hub
```csharp
// 클라이언트 메서드 인터페이스
public interface IChatClient
{
    Task ReceiveMessage(string user, string message, DateTime timestamp);
    Task ReceivePrivateMessage(string fromUser, string message);
    Task UserJoined(string userId, string userName);
    Task UserLeft(string userId);
    Task UpdateUserList(List<UserInfo> users);
}

// 강타입 Hub
public class StronglyTypedChatHub : Hub<IChatClient>
{
    private readonly IUserService _userService;
    
    public StronglyTypedChatHub(IUserService userService)
    {
        _userService = userService;
    }
    
    public async Task SendMessage(string message)
    {
        var user = await _userService.GetUserAsync(Context.UserIdentifier);
        
        // 타입 안전한 메서드 호출
        await Clients.All.ReceiveMessage(user.Name, message, DateTime.UtcNow);
    }
    
    public async Task JoinRoom(string roomName)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, roomName);
        
        var user = await _userService.GetUserAsync(Context.UserIdentifier);
        await Clients.Group(roomName).UserJoined(user.Id, user.Name);
    }
    
    public async Task LeaveRoom(string roomName)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, roomName);
        
        var user = await _userService.GetUserAsync(Context.UserIdentifier);
        await Clients.Group(roomName).UserLeft(user.Id);
    }
    
    public override async Task OnConnectedAsync()
    {
        var users = await _userService.GetOnlineUsersAsync();
        await Clients.Caller.UpdateUserList(users);
        
        await base.OnConnectedAsync();
    }
}
```

## 인증과 권한 부여

### Hub에서 인증 사용
```csharp
[Authorize]
public class SecureHub : Hub
{
    private readonly IUserConnectionManager _connectionManager;
    
    public SecureHub(IUserConnectionManager connectionManager)
    {
        _connectionManager = connectionManager;
    }
    
    public override async Task OnConnectedAsync()
    {
        var userId = Context.UserIdentifier; // 인증된 사용자 ID
        var connectionId = Context.ConnectionId;
        
        // 사용자-연결 매핑 저장
        await _connectionManager.AddConnectionAsync(userId, connectionId);
        
        // 사용자의 모든 연결에 알림
        await Clients.User(userId).SendAsync("DeviceConnected", connectionId);
        
        await base.OnConnectedAsync();
    }
    
    public override async Task OnDisconnectedAsync(Exception exception)
    {
        var userId = Context.UserIdentifier;
        var connectionId = Context.ConnectionId;
        
        await _connectionManager.RemoveConnectionAsync(userId, connectionId);
        
        await base.OnDisconnectedAsync(exception);
    }
    
    [Authorize(Roles = "Admin")]
    public async Task AdminBroadcast(string message)
    {
        await Clients.All.SendAsync("AdminMessage", message);
    }
    
    [Authorize(Policy = "PremiumUser")]
    public async Task PremiumFeature(string data)
    {
        // Premium 사용자만 사용 가능한 기능
        await Clients.Caller.SendAsync("PremiumResponse", data);
    }
}

// 사용자 연결 관리 서비스
public interface IUserConnectionManager
{
    Task AddConnectionAsync(string userId, string connectionId);
    Task RemoveConnectionAsync(string userId, string connectionId);
    Task<IEnumerable<string>> GetConnectionsAsync(string userId);
    Task<string> GetUserIdAsync(string connectionId);
}

public class UserConnectionManager : IUserConnectionManager
{
    private readonly IConnectionMultiplexer _redis;
    
    public UserConnectionManager(IConnectionMultiplexer redis)
    {
        _redis = redis;
    }
    
    public async Task AddConnectionAsync(string userId, string connectionId)
    {
        var db = _redis.GetDatabase();
        
        // 사용자 -> 연결 매핑
        await db.SetAddAsync($"user:{userId}:connections", connectionId);
        
        // 연결 -> 사용자 매핑
        await db.StringSetAsync($"connection:{connectionId}:user", userId);
    }
    
    public async Task RemoveConnectionAsync(string userId, string connectionId)
    {
        var db = _redis.GetDatabase();
        
        await db.SetRemoveAsync($"user:{userId}:connections", connectionId);
        await db.KeyDeleteAsync($"connection:{connectionId}:user");
    }
    
    public async Task<IEnumerable<string>> GetConnectionsAsync(string userId)
    {
        var db = _redis.GetDatabase();
        var connections = await db.SetMembersAsync($"user:{userId}:connections");
        
        return connections.Select(c => c.ToString());
    }
}
```

## 그룹 관리

### 동적 그룹 관리
```csharp
public class RoomHub : Hub
{
    private readonly IRoomService _roomService;
    private readonly ILogger<RoomHub> _logger;
    
    public RoomHub(IRoomService roomService, ILogger<RoomHub> logger)
    {
        _roomService = roomService;
        _logger = logger;
    }
    
    public async Task CreateRoom(string roomName, RoomOptions options)
    {
        var userId = Context.UserIdentifier;
        var room = await _roomService.CreateRoomAsync(roomName, userId, options);
        
        // 생성자를 자동으로 룸에 추가
        await Groups.AddToGroupAsync(Context.ConnectionId, room.Id);
        
        await Clients.Caller.SendAsync("RoomCreated", room);
        _logger.LogInformation("Room {RoomName} created by {UserId}", roomName, userId);
    }
    
    public async Task JoinRoom(string roomId, string password = null)
    {
        var room = await _roomService.GetRoomAsync(roomId);
        
        if (room == null)
        {
            await Clients.Caller.SendAsync("Error", "Room not found");
            return;
        }
        
        if (room.RequiresPassword && !await _roomService.VerifyPasswordAsync(roomId, password))
        {
            await Clients.Caller.SendAsync("Error", "Invalid password");
            return;
        }
        
        await Groups.AddToGroupAsync(Context.ConnectionId, roomId);
        await _roomService.AddUserToRoomAsync(roomId, Context.UserIdentifier);
        
        // 룸의 다른 사용자들에게 알림
        await Clients.OthersInGroup(roomId).SendAsync("UserJoinedRoom", new
        {
            UserId = Context.UserIdentifier,
            RoomId = roomId,
            JoinedAt = DateTime.UtcNow
        });
        
        // 새로 참가한 사용자에게 룸 정보 전송
        var roomInfo = await _roomService.GetRoomInfoAsync(roomId);
        await Clients.Caller.SendAsync("JoinedRoom", roomInfo);
    }
    
    public async Task LeaveRoom(string roomId)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, roomId);
        await _roomService.RemoveUserFromRoomAsync(roomId, Context.UserIdentifier);
        
        await Clients.Group(roomId).SendAsync("UserLeftRoom", new
        {
            UserId = Context.UserIdentifier,
            RoomId = roomId,
            LeftAt = DateTime.UtcNow
        });
        
        // 빈 룸 자동 삭제
        if (await _roomService.IsRoomEmptyAsync(roomId))
        {
            await _roomService.DeleteRoomAsync(roomId);
            _logger.LogInformation("Empty room {RoomId} deleted", roomId);
        }
    }
    
    public async Task SendMessageToRoom(string roomId, string message)
    {
        // 사용자가 룸에 속해있는지 확인
        if (!await _roomService.IsUserInRoomAsync(roomId, Context.UserIdentifier))
        {
            await Clients.Caller.SendAsync("Error", "You are not in this room");
            return;
        }
        
        var messageData = new RoomMessage
        {
            Id = Guid.NewGuid().ToString(),
            RoomId = roomId,
            UserId = Context.UserIdentifier,
            Message = message,
            Timestamp = DateTime.UtcNow
        };
        
        await _roomService.SaveMessageAsync(messageData);
        await Clients.Group(roomId).SendAsync("RoomMessage", messageData);
    }
}
```

## 실시간 알림 시스템

### 알림 Hub
```csharp
public class NotificationHub : Hub<INotificationClient>
{
    private readonly INotificationService _notificationService;
    private readonly IUserConnectionManager _connectionManager;
    
    public NotificationHub(
        INotificationService notificationService,
        IUserConnectionManager connectionManager)
    {
        _notificationService = notificationService;
        _connectionManager = connectionManager;
    }
    
    public override async Task OnConnectedAsync()
    {
        var userId = Context.UserIdentifier;
        
        // 미읽은 알림 전송
        var unreadNotifications = await _notificationService.GetUnreadNotificationsAsync(userId);
        
        if (unreadNotifications.Any())
        {
            await Clients.Caller.ReceiveNotifications(unreadNotifications);
        }
        
        // 온라인 상태 업데이트
        await _notificationService.SetUserOnlineAsync(userId, true);
        await base.OnConnectedAsync();
    }
    
    public override async Task OnDisconnectedAsync(Exception exception)
    {
        var userId = Context.UserIdentifier;
        var connections = await _connectionManager.GetConnectionsAsync(userId);
        
        // 마지막 연결이 끊어진 경우에만 오프라인으로 표시
        if (!connections.Any(c => c != Context.ConnectionId))
        {
            await _notificationService.SetUserOnlineAsync(userId, false);
        }
        
        await base.OnDisconnectedAsync(exception);
    }
    
    public async Task MarkAsRead(string notificationId)
    {
        await _notificationService.MarkAsReadAsync(notificationId, Context.UserIdentifier);
        await Clients.Caller.NotificationRead(notificationId);
    }
    
    public async Task MarkAllAsRead()
    {
        await _notificationService.MarkAllAsReadAsync(Context.UserIdentifier);
        await Clients.Caller.AllNotificationsRead();
    }
}

// 알림 클라이언트 인터페이스
public interface INotificationClient
{
    Task ReceiveNotification(Notification notification);
    Task ReceiveNotifications(IEnumerable<Notification> notifications);
    Task NotificationRead(string notificationId);
    Task AllNotificationsRead();
}

// 알림 서비스
public class NotificationService : INotificationService
{
    private readonly IHubContext<NotificationHub, INotificationClient> _hubContext;
    private readonly INotificationRepository _repository;
    
    public NotificationService(
        IHubContext<NotificationHub, INotificationClient> hubContext,
        INotificationRepository repository)
    {
        _hubContext = hubContext;
        _repository = repository;
    }
    
    public async Task SendNotificationAsync(string userId, string title, string message, NotificationType type)
    {
        var notification = new Notification
        {
            Id = Guid.NewGuid().ToString(),
            UserId = userId,
            Title = title,
            Message = message,
            Type = type,
            CreatedAt = DateTime.UtcNow,
            IsRead = false
        };
        
        await _repository.SaveNotificationAsync(notification);
        
        // 실시간으로 사용자에게 전송
        await _hubContext.Clients.User(userId).ReceiveNotification(notification);
    }
    
    public async Task SendBulkNotificationAsync(IEnumerable<string> userIds, string title, string message)
    {
        var tasks = userIds.Select(userId => SendNotificationAsync(userId, title, message, NotificationType.System));
        await Task.WhenAll(tasks);
    }
}
```

## 스트리밍

### 서버 -> 클라이언트 스트리밍
```csharp
public class StreamingHub : Hub
{
    public async IAsyncEnumerable<StockPrice> StreamStockPrices(
        string symbol,
        [EnumeratorCancellation] CancellationToken cancellationToken)
    {
        var random = new Random();
        var basePrice = 100.0;
        
        while (!cancellationToken.IsCancellationRequested)
        {
            // 가격 변동 시뮬레이션
            var change = (random.NextDouble() - 0.5) * 2; // -1 to 1
            basePrice = Math.Max(0.01, basePrice + change);
            
            yield return new StockPrice
            {
                Symbol = symbol,
                Price = Math.Round(basePrice, 2),
                Timestamp = DateTime.UtcNow,
                Volume = random.Next(1000000, 5000000)
            };
            
            await Task.Delay(1000, cancellationToken); // 1초마다 업데이트
        }
    }
    
    public async IAsyncEnumerable<LogEntry> StreamLogs(
        LogLevel minLevel,
        [EnumeratorCancellation] CancellationToken cancellationToken)
    {
        var logQueue = GetLogQueue(); // 로그 큐 구현
        
        await foreach (var log in logQueue.Reader.ReadAllAsync(cancellationToken))
        {
            if (log.Level >= minLevel)
            {
                yield return log;
            }
        }
    }
}

// 클라이언트 -> 서버 스트리밍
public class DataProcessingHub : Hub
{
    private readonly IDataProcessor _processor;
    
    public async Task<ProcessingResult> ProcessDataStream(
        IAsyncEnumerable<DataChunk> dataStream,
        CancellationToken cancellationToken)
    {
        var processedCount = 0;
        var errors = new List<string>();
        
        await foreach (var chunk in dataStream.WithCancellation(cancellationToken))
        {
            try
            {
                await _processor.ProcessChunkAsync(chunk);
                processedCount++;
                
                // 진행 상황 업데이트
                await Clients.Caller.SendAsync("ProcessingProgress", new
                {
                    ProcessedChunks = processedCount,
                    CurrentChunkId = chunk.Id
                });
            }
            catch (Exception ex)
            {
                errors.Add($"Error processing chunk {chunk.Id}: {ex.Message}");
            }
        }
        
        return new ProcessingResult
        {
            TotalProcessed = processedCount,
            Errors = errors,
            CompletedAt = DateTime.UtcNow
        };
    }
}
```

## 확장성과 성능

### Redis Backplane 설정
```csharp
// Program.cs - Redis를 사용한 SignalR 스케일 아웃
builder.Services.AddSignalR()
    .AddStackExchangeRedis(options =>
    {
        options.Configuration = builder.Configuration.GetConnectionString("Redis");
        
        // Redis 설정 옵션
        options.Configuration = new ConfigurationOptions
        {
            EndPoints = { "localhost:6379" },
            Password = "your_password",
            Ssl = true,
            AbortOnConnectFail = false,
            ConnectRetry = 3,
            ConnectTimeout = 5000,
            SyncTimeout = 5000
        };
    });

// 또는 Azure SignalR Service 사용
builder.Services.AddSignalR()
    .AddAzureSignalR(options =>
    {
        options.ConnectionString = builder.Configuration["Azure:SignalR:ConnectionString"];
        options.ServerStickyMode = ServerStickyMode.Required;
    });
```

### 성능 최적화
```csharp
public class OptimizedHub : Hub
{
    private readonly IMemoryCache _cache;
    private readonly SemaphoreSlim _semaphore;
    
    public OptimizedHub(IMemoryCache cache)
    {
        _cache = cache;
        _semaphore = new SemaphoreSlim(1, 1);
    }
    
    // 메시지 배치 처리
    private readonly List<Message> _pendingMessages = new();
    private Timer _flushTimer;
    
    public async Task QueueMessage(string message)
    {
        await _semaphore.WaitAsync();
        try
        {
            _pendingMessages.Add(new Message 
            { 
                Text = message, 
                UserId = Context.UserIdentifier,
                Timestamp = DateTime.UtcNow 
            });
            
            // 타이머가 없으면 생성
            if (_flushTimer == null)
            {
                _flushTimer = new Timer(async _ => await FlushMessages(), null, 100, Timeout.Infinite);
            }
            
            // 메시지가 100개 이상이면 즉시 전송
            if (_pendingMessages.Count >= 100)
            {
                await FlushMessages();
            }
        }
        finally
        {
            _semaphore.Release();
        }
    }
    
    private async Task FlushMessages()
    {
        await _semaphore.WaitAsync();
        try
        {
            if (_pendingMessages.Any())
            {
                var messagesToSend = _pendingMessages.ToList();
                _pendingMessages.Clear();
                
                await Clients.All.SendAsync("ReceiveMessages", messagesToSend);
            }
            
            _flushTimer?.Dispose();
            _flushTimer = null;
        }
        finally
        {
            _semaphore.Release();
        }
    }
    
    // 결과 캐싱
    public async Task<CachedData> GetCachedData(string key)
    {
        return await _cache.GetOrCreateAsync($"hub_data_{key}", async entry =>
        {
            entry.SlidingExpiration = TimeSpan.FromMinutes(5);
            
            var data = await FetchExpensiveData(key);
            return data;
        });
    }
}
```

## 에러 처리와 재연결

### 클라이언트 재연결 처리
```csharp
public class ReconnectHub : Hub
{
    private readonly IConnectionStateManager _stateManager;
    
    public async Task ReconnectWithState(string lastMessageId)
    {
        var userId = Context.UserIdentifier;
        var connectionId = Context.ConnectionId;
        
        // 이전 상태 복원
        var state = await _stateManager.GetStateAsync(userId);
        if (state != null)
        {
            await Clients.Caller.SendAsync("StateRestored", state);
        }
        
        // 놓친 메시지 전송
        if (!string.IsNullOrEmpty(lastMessageId))
        {
            var missedMessages = await GetMissedMessages(userId, lastMessageId);
            
            if (missedMessages.Any())
            {
                await Clients.Caller.SendAsync("MissedMessages", missedMessages);
            }
        }
        
        await Clients.Caller.SendAsync("ReconnectComplete", new
        {
            ConnectionId = connectionId,
            ServerTime = DateTime.UtcNow
        });
    }
    
    // Hub 파이프라인에서 예외 처리
    public override async Task OnDisconnectedAsync(Exception exception)
    {
        if (exception != null)
        {
            _logger.LogError(exception, "Client disconnected with error");
            
            // 연결 상태 저장
            await _stateManager.SaveDisconnectionStateAsync(
                Context.UserIdentifier, 
                Context.ConnectionId,
                exception);
        }
        
        await base.OnDisconnectedAsync(exception);
    }
}

// Hub 필터로 전역 에러 처리
public class ErrorHandlingHubFilter : IHubFilter
{
    private readonly ILogger<ErrorHandlingHubFilter> _logger;
    
    public ErrorHandlingHubFilter(ILogger<ErrorHandlingHubFilter> logger)
    {
        _logger = logger;
    }
    
    public async ValueTask<object> InvokeMethodAsync(
        HubInvocationContext invocationContext,
        Func<HubInvocationContext, ValueTask<object>> next)
    {
        try
        {
            return await next(invocationContext);
        }
        catch (HubException)
        {
            // HubException은 클라이언트에 전달됨
            throw;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Hub method {Method} failed", invocationContext.HubMethodName);
            
            // 클라이언트에게 일반적인 에러 메시지 전송
            throw new HubException("An error occurred while processing your request.");
        }
    }
}

// 필터 등록
builder.Services.AddSignalR(options =>
{
    options.AddFilter<ErrorHandlingHubFilter>();
});
```

## 실전 예제: 실시간 대시보드

### 대시보드 Hub
```csharp
public class DashboardHub : Hub<IDashboardClient>
{
    private readonly IDashboardService _dashboardService;
    private readonly IMemoryCache _cache;
    private static readonly SemaphoreSlim _updateSemaphore = new(1, 1);
    
    public override async Task OnConnectedAsync()
    {
        // 대시보드 구독
        await Groups.AddToGroupAsync(Context.ConnectionId, "dashboard-viewers");
        
        // 초기 데이터 전송
        var dashboardData = await _dashboardService.GetCurrentDataAsync();
        await Clients.Caller.UpdateDashboard(dashboardData);
        
        await base.OnConnectedAsync();
    }
    
    public async Task SubscribeToMetric(string metricName)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, $"metric-{metricName}");
        
        // 최신 메트릭 데이터 전송
        var latestData = await _dashboardService.GetMetricDataAsync(metricName);
        await Clients.Caller.UpdateMetric(metricName, latestData);
    }
    
    public async Task UnsubscribeFromMetric(string metricName)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, $"metric-{metricName}");
    }
}

// 백그라운드 서비스로 실시간 업데이트
public class DashboardUpdateService : BackgroundService
{
    private readonly IHubContext<DashboardHub, IDashboardClient> _hubContext;
    private readonly IDashboardService _dashboardService;
    private readonly ILogger<DashboardUpdateService> _logger;
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                // 메트릭 수집
                var metrics = await _dashboardService.CollectMetricsAsync();
                
                // 각 메트릭별로 구독자에게 전송
                foreach (var metric in metrics)
                {
                    await _hubContext.Clients
                        .Group($"metric-{metric.Name}")
                        .UpdateMetric(metric.Name, metric.Data);
                }
                
                // 전체 대시보드 업데이트 (5초마다)
                if (DateTime.UtcNow.Second % 5 == 0)
                {
                    var dashboardData = await _dashboardService.GetCurrentDataAsync();
                    await _hubContext.Clients
                        .Group("dashboard-viewers")
                        .UpdateDashboard(dashboardData);
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error updating dashboard");
            }
            
            await Task.Delay(1000, stoppingToken); // 1초마다 업데이트
        }
    }
}

// 클라이언트 인터페이스
public interface IDashboardClient
{
    Task UpdateDashboard(DashboardData data);
    Task UpdateMetric(string metricName, MetricData data);
    Task Alert(AlertData alert);
}
```

## 보안 고려사항

### 메시지 검증과 제한
```csharp
public class SecureMessageHub : Hub
{
    private readonly IMessageValidator _validator;
    private readonly IRateLimiter _rateLimiter;
    
    public async Task SendSecureMessage(string encryptedMessage)
    {
        // Rate limiting
        if (!await _rateLimiter.AllowRequestAsync(Context.UserIdentifier))
        {
            throw new HubException("Rate limit exceeded");
        }
        
        // 메시지 크기 제한
        if (encryptedMessage.Length > 10000)
        {
            throw new HubException("Message too large");
        }
        
        // 메시지 검증
        if (!_validator.IsValid(encryptedMessage))
        {
            throw new HubException("Invalid message format");
        }
        
        // 메시지 복호화 및 처리
        var decryptedMessage = await DecryptMessage(encryptedMessage);
        
        // XSS 방지를 위한 sanitization
        var sanitizedMessage = HtmlEncoder.Default.Encode(decryptedMessage);
        
        await Clients.Others.SendAsync("ReceiveSecureMessage", sanitizedMessage);
    }
}

// 연결별 Rate Limiting
public class ConnectionRateLimiter : IRateLimiter
{
    private readonly IMemoryCache _cache;
    private readonly int _maxRequests = 100;
    private readonly TimeSpan _window = TimeSpan.FromMinutes(1);
    
    public async Task<bool> AllowRequestAsync(string userId)
    {
        var key = $"rate_limit_{userId}";
        var count = await _cache.GetOrCreateAsync(key, async entry =>
        {
            entry.SlidingExpiration = _window;
            return 0;
        });
        
        if (count >= _maxRequests)
        {
            return false;
        }
        
        _cache.Set(key, count + 1, _window);
        return true;
    }
}
```

## 모니터링과 진단

### SignalR 메트릭 수집
```csharp
public class SignalRMetricsService : IHostedService
{
    private readonly IHubContext<MetricsHub> _hubContext;
    private readonly ILogger<SignalRMetricsService> _logger;
    private Timer _timer;
    
    public Task StartAsync(CancellationToken cancellationToken)
    {
        _timer = new Timer(CollectMetrics, null, TimeSpan.Zero, TimeSpan.FromSeconds(30));
        return Task.CompletedTask;
    }
    
    private async void CollectMetrics(object state)
    {
        try
        {
            var metrics = new SignalRMetrics
            {
                Timestamp = DateTime.UtcNow,
                ActiveConnections = GetActiveConnectionCount(),
                MessagesPerSecond = CalculateMessagesPerSecond(),
                AverageMessageSize = CalculateAverageMessageSize(),
                ErrorRate = CalculateErrorRate()
            };
            
            // 메트릭 저장
            await SaveMetrics(metrics);
            
            // 관리자에게 실시간 전송
            await _hubContext.Clients.Group("admins").SendAsync("MetricsUpdate", metrics);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to collect SignalR metrics");
        }
    }
    
    public Task StopAsync(CancellationToken cancellationToken)
    {
        _timer?.Change(Timeout.Infinite, 0);
        _timer?.Dispose();
        return Task.CompletedTask;
    }
}
```

SignalR은 실시간 웹 애플리케이션 구축을 위한 강력한 도구입니다. 적절한 설계와 구현을 통해 확장 가능하고 안정적인 실시간 기능을 제공할 수 있습니다.