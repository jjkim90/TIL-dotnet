# 실시간 데이터 동기화

## 개요

Blazor에서 실시간 데이터 동기화는 SignalR을 통해 구현됩니다. 이 장에서는 SignalR 통합, 실시간 업데이트 패턴, 그리고 효율적인 데이터 동기화 전략을 학습합니다.

## 1. SignalR 기본 통합

### 1.1 SignalR Hub 구성

```csharp
// Hubs/ChatHub.cs
public class ChatHub : Hub
{
    private static readonly Dictionary<string, UserConnection> _connections = new();
    
    public override async Task OnConnectedAsync()
    {
        await Clients.All.SendAsync("UserConnected", Context.ConnectionId);
        await base.OnConnectedAsync();
    }
    
    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        if (_connections.TryGetValue(Context.ConnectionId, out var userConnection))
        {
            _connections.Remove(Context.ConnectionId);
            await Clients.All.SendAsync("UserDisconnected", userConnection.Username);
        }
        await base.OnDisconnectedAsync(exception);
    }
    
    public async Task JoinChat(string username)
    {
        _connections[Context.ConnectionId] = new UserConnection
        {
            Username = username,
            ConnectionId = Context.ConnectionId
        };
        
        await Clients.Others.SendAsync("UserJoined", username);
    }
    
    public async Task SendMessage(string message)
    {
        if (_connections.TryGetValue(Context.ConnectionId, out var userConnection))
        {
            await Clients.All.SendAsync("ReceiveMessage", userConnection.Username, message);
        }
    }
}

public class UserConnection
{
    public string Username { get; set; } = "";
    public string ConnectionId { get; set; } = "";
}

// Program.cs
builder.Services.AddSignalR();
app.MapHub<ChatHub>("/chathub");
```

### 1.2 Blazor 클라이언트 구현

```csharp
// Chat.razor
@page "/chat"
@using Microsoft.AspNetCore.SignalR.Client
@implements IAsyncDisposable

<h3>Real-time Chat</h3>

<div class="chat-container">
    @if (!IsConnected)
    {
        <div class="login">
            <input @bind="username" placeholder="Enter username" />
            <button @onclick="Connect">Connect</button>
        </div>
    }
    else
    {
        <div class="chat-messages">
            @foreach (var msg in messages)
            {
                <div class="message">
                    <strong>@msg.Username:</strong> @msg.Text
                    <small>@msg.Timestamp.ToString("HH:mm:ss")</small>
                </div>
            }
        </div>
        
        <div class="chat-input">
            <input @bind="messageInput" @onkeypress="@(async (e) => { if (e.Key == "Enter") await SendMessage(); })" />
            <button @onclick="SendMessage">Send</button>
        </div>
    }
</div>

@code {
    private HubConnection? hubConnection;
    private List<ChatMessage> messages = new();
    private string username = "";
    private string messageInput = "";
    
    private bool IsConnected => hubConnection?.State == HubConnectionState.Connected;
    
    private async Task Connect()
    {
        if (string.IsNullOrWhiteSpace(username)) return;
        
        hubConnection = new HubConnectionBuilder()
            .WithUrl(Navigation.ToAbsoluteUri("/chathub"))
            .WithAutomaticReconnect()
            .Build();
        
        hubConnection.On<string, string>("ReceiveMessage", (user, message) =>
        {
            messages.Add(new ChatMessage
            {
                Username = user,
                Text = message,
                Timestamp = DateTime.Now
            });
            InvokeAsync(StateHasChanged);
        });
        
        hubConnection.On<string>("UserJoined", (user) =>
        {
            messages.Add(new ChatMessage
            {
                Username = "System",
                Text = $"{user} joined the chat",
                Timestamp = DateTime.Now
            });
            InvokeAsync(StateHasChanged);
        });
        
        await hubConnection.StartAsync();
        await hubConnection.SendAsync("JoinChat", username);
    }
    
    private async Task SendMessage()
    {
        if (!string.IsNullOrWhiteSpace(messageInput) && IsConnected)
        {
            await hubConnection!.SendAsync("SendMessage", messageInput);
            messageInput = "";
        }
    }
    
    public async ValueTask DisposeAsync()
    {
        if (hubConnection != null)
        {
            await hubConnection.DisposeAsync();
        }
    }
    
    private class ChatMessage
    {
        public string Username { get; set; } = "";
        public string Text { get; set; } = "";
        public DateTime Timestamp { get; set; }
    }
}
```

## 2. 고급 SignalR 패턴

### 2.1 그룹 관리와 타겟 통신

```csharp
// CollaborationHub.cs
public class CollaborationHub : Hub<ICollaborationClient>
{
    private readonly ICollaborationService _collaborationService;
    
    public CollaborationHub(ICollaborationService collaborationService)
    {
        _collaborationService = collaborationService;
    }
    
    public async Task JoinDocument(string documentId)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, $"doc-{documentId}");
        
        var document = await _collaborationService.GetDocumentAsync(documentId);
        await Clients.Caller.DocumentLoaded(document);
        
        await Clients.OthersInGroup($"doc-{documentId}")
            .UserJoinedDocument(Context.UserIdentifier ?? "Anonymous");
    }
    
    public async Task UpdateDocument(string documentId, DocumentUpdate update)
    {
        await _collaborationService.ApplyUpdateAsync(documentId, update);
        
        await Clients.OthersInGroup($"doc-{documentId}")
            .DocumentUpdated(update);
    }
    
    public async Task SendCursorPosition(string documentId, CursorPosition position)
    {
        await Clients.OthersInGroup($"doc-{documentId}")
            .CursorMoved(Context.UserIdentifier ?? "Anonymous", position);
    }
}

public interface ICollaborationClient
{
    Task DocumentLoaded(Document document);
    Task DocumentUpdated(DocumentUpdate update);
    Task UserJoinedDocument(string userId);
    Task UserLeftDocument(string userId);
    Task CursorMoved(string userId, CursorPosition position);
}

// CollaborativeEditor.razor
@page "/editor/{DocumentId}"
@implements IAsyncDisposable

<div class="collaborative-editor">
    <div class="editor-header">
        <h3>Document: @DocumentId</h3>
        <div class="active-users">
            @foreach (var user in activeUsers)
            {
                <span class="user-badge" style="background-color: @user.Color">
                    @user.Name
                </span>
            }
        </div>
    </div>
    
    <div class="editor-content">
        <textarea @bind="content" @oninput="HandleInput" 
                  @onselectionchange="HandleSelectionChange"></textarea>
        
        @foreach (var cursor in otherCursors)
        {
            <div class="cursor" style="left: @(cursor.X)px; top: @(cursor.Y)px; 
                                       background-color: @cursor.Color">
                @cursor.Username
            </div>
        }
    </div>
</div>

@code {
    [Parameter] public string DocumentId { get; set; } = "";
    
    private HubConnection? hubConnection;
    private string content = "";
    private List<ActiveUser> activeUsers = new();
    private List<RemoteCursor> otherCursors = new();
    
    protected override async Task OnInitializedAsync()
    {
        hubConnection = new HubConnectionBuilder()
            .WithUrl(Navigation.ToAbsoluteUri("/collaboration"))
            .Build();
        
        hubConnection.On<Document>("DocumentLoaded", document =>
        {
            content = document.Content;
            InvokeAsync(StateHasChanged);
        });
        
        hubConnection.On<DocumentUpdate>("DocumentUpdated", update =>
        {
            ApplyUpdate(update);
            InvokeAsync(StateHasChanged);
        });
        
        hubConnection.On<string, CursorPosition>("CursorMoved", (userId, position) =>
        {
            UpdateCursor(userId, position);
            InvokeAsync(StateHasChanged);
        });
        
        await hubConnection.StartAsync();
        await hubConnection.SendAsync("JoinDocument", DocumentId);
    }
    
    private async Task HandleInput(ChangeEventArgs e)
    {
        var newContent = e.Value?.ToString() ?? "";
        var update = CalculateUpdate(content, newContent);
        content = newContent;
        
        if (hubConnection != null)
        {
            await hubConnection.SendAsync("UpdateDocument", DocumentId, update);
        }
    }
    
    private DocumentUpdate CalculateUpdate(string oldContent, string newContent)
    {
        // 실제 구현에서는 Operational Transform 알고리즘 사용
        return new DocumentUpdate
        {
            Operation = "replace",
            Position = 0,
            OldText = oldContent,
            NewText = newContent
        };
    }
    
    private void ApplyUpdate(DocumentUpdate update)
    {
        // Operational Transform 적용
        content = update.NewText;
    }
    
    public async ValueTask DisposeAsync()
    {
        if (hubConnection != null)
        {
            await hubConnection.DisposeAsync();
        }
    }
}
```

### 2.2 스트리밍 데이터

```csharp
// StreamingHub.cs
public class StreamingHub : Hub
{
    public async IAsyncEnumerable<StockPrice> StreamStockPrices(
        [EnumeratorCancellation] CancellationToken cancellationToken)
    {
        var random = new Random();
        var stocks = new[] { "AAPL", "GOOGL", "MSFT", "AMZN" };
        
        while (!cancellationToken.IsCancellationRequested)
        {
            foreach (var symbol in stocks)
            {
                yield return new StockPrice
                {
                    Symbol = symbol,
                    Price = 100 + random.Next(-5, 6),
                    Timestamp = DateTime.UtcNow
                };
            }
            
            await Task.Delay(1000, cancellationToken);
        }
    }
    
    public ChannelReader<Notification> StreamNotifications(CancellationToken cancellationToken)
    {
        var channel = Channel.CreateUnbounded<Notification>();
        
        _ = WriteNotificationsAsync(channel.Writer, cancellationToken);
        
        return channel.Reader;
    }
    
    private async Task WriteNotificationsAsync(
        ChannelWriter<Notification> writer,
        CancellationToken cancellationToken)
    {
        var notificationId = 0;
        
        while (!cancellationToken.IsCancellationRequested)
        {
            await writer.WriteAsync(new Notification
            {
                Id = ++notificationId,
                Message = $"Notification #{notificationId}",
                Timestamp = DateTime.UtcNow
            });
            
            await Task.Delay(Random.Shared.Next(2000, 5000), cancellationToken);
        }
        
        writer.TryComplete();
    }
}

// StreamingData.razor
@page "/streaming"
@implements IAsyncDisposable

<h3>Streaming Data</h3>

<div class="streaming-container">
    <h4>Stock Prices</h4>
    <table class="table">
        <thead>
            <tr>
                <th>Symbol</th>
                <th>Price</th>
                <th>Time</th>
            </tr>
        </thead>
        <tbody>
            @foreach (var stock in stockPrices.Values)
            {
                <tr class="@(stock.PriceChange > 0 ? "price-up" : "price-down")">
                    <td>@stock.Symbol</td>
                    <td>$@stock.Price</td>
                    <td>@stock.Timestamp.ToString("HH:mm:ss")</td>
                </tr>
            }
        </tbody>
    </table>
    
    <h4>Notifications</h4>
    <div class="notifications">
        @foreach (var notification in notifications.TakeLast(5))
        {
            <div class="notification">
                @notification.Message - @notification.Timestamp.ToString("HH:mm:ss")
            </div>
        }
    </div>
</div>

@code {
    private HubConnection? hubConnection;
    private Dictionary<string, StockPriceDisplay> stockPrices = new();
    private List<Notification> notifications = new();
    private CancellationTokenSource? cts;
    
    protected override async Task OnInitializedAsync()
    {
        hubConnection = new HubConnectionBuilder()
            .WithUrl(Navigation.ToAbsoluteUri("/streaming"))
            .Build();
        
        await hubConnection.StartAsync();
        
        cts = new CancellationTokenSource();
        
        // 스트리밍 시작
        await foreach (var price in hubConnection.StreamAsync<StockPrice>(
            "StreamStockPrices", cts.Token))
        {
            UpdateStockPrice(price);
            StateHasChanged();
        }
    }
    
    private void UpdateStockPrice(StockPrice newPrice)
    {
        if (stockPrices.TryGetValue(newPrice.Symbol, out var existing))
        {
            existing.PriceChange = newPrice.Price - existing.Price;
        }
        
        stockPrices[newPrice.Symbol] = new StockPriceDisplay
        {
            Symbol = newPrice.Symbol,
            Price = newPrice.Price,
            Timestamp = newPrice.Timestamp,
            PriceChange = existing?.PriceChange ?? 0
        };
    }
    
    public async ValueTask DisposeAsync()
    {
        cts?.Cancel();
        if (hubConnection != null)
        {
            await hubConnection.DisposeAsync();
        }
    }
    
    private class StockPriceDisplay : StockPrice
    {
        public decimal PriceChange { get; set; }
    }
}
```

## 3. 실시간 업데이트 패턴

### 3.1 낙관적 업데이트

```csharp
// OptimisticUpdate.razor
@page "/optimistic"

<h3>Optimistic Updates</h3>

<div class="todo-list">
    @foreach (var todo in todos)
    {
        <div class="todo-item @(todo.IsPending ? "pending" : "")">
            <input type="checkbox" checked="@todo.IsCompleted" 
                   @onchange="@(() => ToggleTodo(todo))" />
            <span class="@(todo.IsCompleted ? "completed" : "")">@todo.Title</span>
            @if (todo.IsPending)
            {
                <span class="pending-indicator">Saving...</span>
            }
        </div>
    }
</div>

@code {
    private List<TodoItem> todos = new();
    private HubConnection? hubConnection;
    
    protected override async Task OnInitializedAsync()
    {
        hubConnection = new HubConnectionBuilder()
            .WithUrl(Navigation.ToAbsoluteUri("/todohub"))
            .Build();
        
        hubConnection.On<TodoUpdate>("TodoUpdated", update =>
        {
            ApplyServerUpdate(update);
            InvokeAsync(StateHasChanged);
        });
        
        await hubConnection.StartAsync();
        await LoadTodos();
    }
    
    private async Task ToggleTodo(TodoItem todo)
    {
        // 낙관적 업데이트 - UI 즉시 반영
        todo.IsCompleted = !todo.IsCompleted;
        todo.IsPending = true;
        StateHasChanged();
        
        try
        {
            // 서버에 업데이트 전송
            await hubConnection!.SendAsync("UpdateTodo", new TodoUpdate
            {
                Id = todo.Id,
                IsCompleted = todo.IsCompleted,
                ClientId = Guid.NewGuid().ToString()
            });
        }
        catch
        {
            // 실패 시 롤백
            todo.IsCompleted = !todo.IsCompleted;
            todo.IsPending = false;
            StateHasChanged();
        }
    }
    
    private void ApplyServerUpdate(TodoUpdate update)
    {
        var todo = todos.FirstOrDefault(t => t.Id == update.Id);
        if (todo != null)
        {
            todo.IsCompleted = update.IsCompleted;
            todo.IsPending = false;
        }
    }
    
    private class TodoItem
    {
        public string Id { get; set; } = "";
        public string Title { get; set; } = "";
        public bool IsCompleted { get; set; }
        public bool IsPending { get; set; }
    }
}
```

### 3.2 충돌 해결

```csharp
// ConflictResolution.razor
@page "/conflict-resolution"

<h3>Collaborative Document with Conflict Resolution</h3>

<div class="document-editor">
    <div class="version-info">
        Version: @currentVersion | Base Version: @baseVersion
    </div>
    
    <textarea @bind="localContent" @oninput="HandleLocalEdit"></textarea>
    
    @if (hasConflict)
    {
        <div class="conflict-resolution">
            <h4>Conflict Detected!</h4>
            <div class="conflict-options">
                <div>
                    <h5>Your Version:</h5>
                    <pre>@localContent</pre>
                </div>
                <div>
                    <h5>Server Version:</h5>
                    <pre>@serverContent</pre>
                </div>
                <div>
                    <h5>Merged Version:</h5>
                    <pre>@mergedContent</pre>
                </div>
            </div>
            <button @onclick="AcceptLocal">Keep Mine</button>
            <button @onclick="AcceptServer">Take Theirs</button>
            <button @onclick="AcceptMerged">Accept Merge</button>
        </div>
    }
</div>

@code {
    private string localContent = "";
    private string serverContent = "";
    private string mergedContent = "";
    private string baseVersion = "";
    private string currentVersion = "";
    private bool hasConflict = false;
    private HubConnection? hubConnection;
    
    protected override async Task OnInitializedAsync()
    {
        hubConnection = new HubConnectionBuilder()
            .WithUrl(Navigation.ToAbsoluteUri("/documenthub"))
            .Build();
        
        hubConnection.On<DocumentState>("DocumentUpdated", state =>
        {
            HandleServerUpdate(state);
        });
        
        await hubConnection.StartAsync();
    }
    
    private void HandleServerUpdate(DocumentState state)
    {
        if (state.Version != currentVersion)
        {
            if (HasLocalChanges())
            {
                // 충돌 감지
                serverContent = state.Content;
                mergedContent = AttemptAutoMerge(baseVersion, localContent, serverContent);
                hasConflict = true;
            }
            else
            {
                // 충돌 없음, 서버 버전 적용
                localContent = state.Content;
                currentVersion = state.Version;
                baseVersion = state.Version;
            }
        }
        
        InvokeAsync(StateHasChanged);
    }
    
    private string AttemptAutoMerge(string baseText, string localText, string serverText)
    {
        // 간단한 3-way merge 구현
        // 실제로는 더 복잡한 알고리즘 사용
        var baseLines = baseText.Split('\n');
        var localLines = localText.Split('\n');
        var serverLines = serverText.Split('\n');
        
        var merged = new List<string>();
        
        for (int i = 0; i < Math.Max(localLines.Length, serverLines.Length); i++)
        {
            var baseLine = i < baseLines.Length ? baseLines[i] : "";
            var localLine = i < localLines.Length ? localLines[i] : "";
            var serverLine = i < serverLines.Length ? serverLines[i] : "";
            
            if (localLine == serverLine)
            {
                merged.Add(localLine);
            }
            else if (localLine == baseLine)
            {
                merged.Add(serverLine);
            }
            else if (serverLine == baseLine)
            {
                merged.Add(localLine);
            }
            else
            {
                // 실제 충돌
                merged.Add($"<<<<<<< LOCAL\n{localLine}\n=======\n{serverLine}\n>>>>>>> SERVER");
            }
        }
        
        return string.Join('\n', merged);
    }
    
    private bool HasLocalChanges()
    {
        return localContent != baseVersion;
    }
}
```

## 4. 성능 최적화

### 4.1 메시지 배칭과 압축

```csharp
// BatchingHub.cs
public class BatchingHub : Hub
{
    private readonly IMessageBatcher _batcher;
    
    public BatchingHub(IMessageBatcher batcher)
    {
        _batcher = batcher;
    }
    
    public async Task SendBatchedUpdate(UpdateMessage message)
    {
        await _batcher.QueueMessage(Context.ConnectionId, message);
    }
}

public class MessageBatcher : IMessageBatcher, IHostedService
{
    private readonly IHubContext<BatchingHub> _hubContext;
    private readonly Dictionary<string, List<UpdateMessage>> _messageQueues = new();
    private Timer? _flushTimer;
    
    public MessageBatcher(IHubContext<BatchingHub> hubContext)
    {
        _hubContext = hubContext;
    }
    
    public Task QueueMessage(string connectionId, UpdateMessage message)
    {
        lock (_messageQueues)
        {
            if (!_messageQueues.ContainsKey(connectionId))
            {
                _messageQueues[connectionId] = new List<UpdateMessage>();
            }
            
            _messageQueues[connectionId].Add(message);
        }
        
        return Task.CompletedTask;
    }
    
    public Task StartAsync(CancellationToken cancellationToken)
    {
        _flushTimer = new Timer(FlushBatches, null, 
            TimeSpan.Zero, TimeSpan.FromMilliseconds(100));
        return Task.CompletedTask;
    }
    
    private void FlushBatches(object? state)
    {
        Dictionary<string, List<UpdateMessage>> toSend;
        
        lock (_messageQueues)
        {
            toSend = new Dictionary<string, List<UpdateMessage>>(_messageQueues);
            _messageQueues.Clear();
        }
        
        foreach (var (connectionId, messages) in toSend)
        {
            if (messages.Any())
            {
                var batch = new BatchedUpdate
                {
                    Messages = messages,
                    Timestamp = DateTime.UtcNow
                };
                
                _hubContext.Clients.Client(connectionId)
                    .SendAsync("ReceiveBatch", batch);
            }
        }
    }
    
    public Task StopAsync(CancellationToken cancellationToken)
    {
        _flushTimer?.Dispose();
        return Task.CompletedTask;
    }
}
```

### 4.2 선택적 구독

```csharp
// SelectiveSubscription.razor
@page "/selective"

<h3>Selective Data Subscription</h3>

<div class="subscription-manager">
    <h4>Available Channels:</h4>
    @foreach (var channel in availableChannels)
    {
        <label>
            <input type="checkbox" 
                   checked="@subscribedChannels.Contains(channel)"
                   @onchange="@(() => ToggleSubscription(channel))" />
            @channel
        </label>
    }
</div>

<div class="data-display">
    @foreach (var data in receivedData.TakeLast(10))
    {
        <div class="data-item">
            <span class="channel">[@data.Channel]</span>
            <span class="value">@data.Value</span>
            <span class="time">@data.Timestamp.ToString("HH:mm:ss")</span>
        </div>
    }
</div>

@code {
    private HubConnection? hubConnection;
    private List<string> availableChannels = new() { "stocks", "news", "weather", "sports" };
    private HashSet<string> subscribedChannels = new();
    private List<ChannelData> receivedData = new();
    
    protected override async Task OnInitializedAsync()
    {
        hubConnection = new HubConnectionBuilder()
            .WithUrl(Navigation.ToAbsoluteUri("/datahub"))
            .Build();
        
        hubConnection.On<ChannelData>("ReceiveData", data =>
        {
            if (subscribedChannels.Contains(data.Channel))
            {
                receivedData.Add(data);
                InvokeAsync(StateHasChanged);
            }
        });
        
        await hubConnection.StartAsync();
    }
    
    private async Task ToggleSubscription(string channel)
    {
        if (subscribedChannels.Contains(channel))
        {
            subscribedChannels.Remove(channel);
            await hubConnection!.SendAsync("Unsubscribe", channel);
        }
        else
        {
            subscribedChannels.Add(channel);
            await hubConnection!.SendAsync("Subscribe", channel);
        }
    }
    
    private class ChannelData
    {
        public string Channel { get; set; } = "";
        public object Value { get; set; } = new();
        public DateTime Timestamp { get; set; }
    }
}
```

## 마무리

SignalR과 Blazor의 통합은 강력한 실시간 웹 애플리케이션을 구축할 수 있게 해줍니다. 양방향 통신, 스트리밍, 그룹 관리 등의 기능을 활용하여 채팅, 협업 도구, 실시간 대시보드 등 다양한 시나리오를 구현할 수 있습니다. 성능 최적화를 위해 메시지 배칭, 선택적 구독, 효율적인 충돌 해결 전략을 적용하는 것이 중요합니다.