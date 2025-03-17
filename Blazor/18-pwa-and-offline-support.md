# PWA와 오프라인 지원

## 개요

Progressive Web App (PWA)은 웹 기술을 사용하여 네이티브 앱과 유사한 사용자 경험을 제공합니다. 이 장에서는 Blazor를 사용한 PWA 구현, Service Worker, 캐싱 전략, 오프라인 데이터 동기화 등을 학습합니다.

## 1. PWA 기본 설정

### 1.1 Web App Manifest

```json
// wwwroot/manifest.json
{
  "name": "My Blazor PWA",
  "short_name": "BlazorPWA",
  "description": "A Progressive Web App built with Blazor",
  "start_url": "/",
  "scope": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#512BD4",
  "orientation": "portrait-primary",
  "icons": [
    {
      "src": "icon-72x72.png",
      "sizes": "72x72",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "icon-96x96.png",
      "sizes": "96x96",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "icon-128x128.png",
      "sizes": "128x128",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "icon-144x144.png",
      "sizes": "144x144",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "icon-152x152.png",
      "sizes": "152x152",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "icon-192x192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "icon-384x384.png",
      "sizes": "384x384",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "icon-512x512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "any maskable"
    }
  ],
  "related_applications": [],
  "prefer_related_applications": false,
  "categories": ["productivity", "utilities"],
  "shortcuts": [
    {
      "name": "Dashboard",
      "short_name": "Dashboard",
      "description": "View your dashboard",
      "url": "/dashboard",
      "icons": [{ "src": "/icon-96x96.png", "sizes": "96x96" }]
    },
    {
      "name": "Create New",
      "short_name": "New",
      "description": "Create a new item",
      "url": "/create",
      "icons": [{ "src": "/icon-96x96.png", "sizes": "96x96" }]
    }
  ],
  "screenshots": [
    {
      "src": "screenshot1.jpg",
      "sizes": "540x720",
      "type": "image/jpg"
    },
    {
      "src": "screenshot2.jpg",
      "sizes": "540x720",
      "type": "image/jpg"
    }
  ]
}
```

### 1.2 Service Worker 기본 구성

```javascript
// wwwroot/service-worker.js
const CACHE_NAME = 'blazor-pwa-v1';
const urlsToCache = [
    '/',
    '/index.html',
    '/css/app.css',
    '/css/bootstrap/bootstrap.min.css',
    '/icon-512x512.png',
    '/manifest.json'
];

// Install event - cache initial resources
self.addEventListener('install', event => {
    console.log('Service Worker installing...');
    
    event.waitUntil(
        caches.open(CACHE_NAME)
            .then(cache => {
                console.log('Opened cache');
                return cache.addAll(urlsToCache);
            })
            .then(() => self.skipWaiting())
    );
});

// Activate event - clean up old caches
self.addEventListener('activate', event => {
    console.log('Service Worker activating...');
    
    event.waitUntil(
        caches.keys().then(cacheNames => {
            return Promise.all(
                cacheNames.map(cacheName => {
                    if (cacheName !== CACHE_NAME) {
                        console.log('Deleting old cache:', cacheName);
                        return caches.delete(cacheName);
                    }
                })
            );
        }).then(() => self.clients.claim())
    );
});

// Fetch event - serve from cache, fallback to network
self.addEventListener('fetch', event => {
    // Skip non-GET requests
    if (event.request.method !== 'GET') {
        return;
    }
    
    event.respondWith(
        caches.match(event.request)
            .then(response => {
                // Cache hit - return response
                if (response) {
                    return response;
                }
                
                // Clone the request
                const fetchRequest = event.request.clone();
                
                return fetch(fetchRequest).then(response => {
                    // Check if valid response
                    if (!response || response.status !== 200 || response.type !== 'basic') {
                        return response;
                    }
                    
                    // Clone the response
                    const responseToCache = response.clone();
                    
                    caches.open(CACHE_NAME)
                        .then(cache => {
                            cache.put(event.request, responseToCache);
                        });
                    
                    return response;
                });
            })
            .catch(() => {
                // Offline fallback
                if (event.request.destination === 'document') {
                    return caches.match('/offline.html');
                }
            })
    );
});
```

## 2. 고급 Service Worker 패턴

### 2.1 캐싱 전략

```javascript
// wwwroot/service-worker-advanced.js
class CacheManager {
    constructor() {
        this.cacheNames = {
            static: 'static-v1',
            dynamic: 'dynamic-v1',
            api: 'api-v1'
        };
        
        this.cacheStrategies = {
            networkFirst: this.networkFirst.bind(this),
            cacheFirst: this.cacheFirst.bind(this),
            networkOnly: this.networkOnly.bind(this),
            cacheOnly: this.cacheOnly.bind(this),
            staleWhileRevalidate: this.staleWhileRevalidate.bind(this)
        };
    }
    
    // Network first, fallback to cache
    async networkFirst(request, cacheName) {
        try {
            const response = await fetch(request);
            if (response.ok) {
                const cache = await caches.open(cacheName);
                cache.put(request, response.clone());
            }
            return response;
        } catch (error) {
            const cached = await caches.match(request);
            return cached || new Response('Network error', { status: 503 });
        }
    }
    
    // Cache first, fallback to network
    async cacheFirst(request, cacheName) {
        const cached = await caches.match(request);
        if (cached) {
            return cached;
        }
        
        try {
            const response = await fetch(request);
            if (response.ok) {
                const cache = await caches.open(cacheName);
                cache.put(request, response.clone());
            }
            return response;
        } catch (error) {
            return new Response('Network error', { status: 503 });
        }
    }
    
    // Stale while revalidate
    async staleWhileRevalidate(request, cacheName) {
        const cached = await caches.match(request);
        
        const fetchPromise = fetch(request).then(response => {
            if (response.ok) {
                caches.open(cacheName).then(cache => {
                    cache.put(request, response.clone());
                });
            }
            return response;
        });
        
        return cached || fetchPromise;
    }
    
    // Network only
    async networkOnly(request) {
        return fetch(request);
    }
    
    // Cache only
    async cacheOnly(request) {
        const cached = await caches.match(request);
        return cached || new Response('Not found in cache', { status: 404 });
    }
    
    // Get strategy based on request
    getStrategy(request) {
        const url = new URL(request.url);
        
        // API calls - network first
        if (url.pathname.startsWith('/api/')) {
            return { strategy: 'networkFirst', cacheName: this.cacheNames.api };
        }
        
        // Static assets - cache first
        if (url.pathname.match(/\.(js|css|png|jpg|jpeg|svg|gif|woff2?)$/)) {
            return { strategy: 'cacheFirst', cacheName: this.cacheNames.static };
        }
        
        // HTML pages - stale while revalidate
        if (request.mode === 'navigate') {
            return { strategy: 'staleWhileRevalidate', cacheName: this.cacheNames.dynamic };
        }
        
        // Default - network first
        return { strategy: 'networkFirst', cacheName: this.cacheNames.dynamic };
    }
}

const cacheManager = new CacheManager();

// Fetch event handler
self.addEventListener('fetch', event => {
    const { strategy, cacheName } = cacheManager.getStrategy(event.request);
    const strategyFunction = cacheManager.cacheStrategies[strategy];
    
    event.respondWith(strategyFunction(event.request, cacheName));
});

// Background sync
self.addEventListener('sync', event => {
    if (event.tag === 'sync-data') {
        event.waitUntil(syncData());
    }
});

async function syncData() {
    try {
        const cache = await caches.open('offline-queue');
        const requests = await cache.keys();
        
        for (const request of requests) {
            try {
                const response = await fetch(request.clone());
                if (response.ok) {
                    await cache.delete(request);
                }
            } catch (error) {
                console.error('Sync failed for:', request.url);
            }
        }
    } catch (error) {
        console.error('Sync error:', error);
    }
}
```

### 2.2 Blazor PWA 서비스

```csharp
// Services/PWAService.cs
public interface IPWAService
{
    Task<bool> IsOnlineAsync();
    Task<bool> IsPWAInstalledAsync();
    Task RequestInstallAsync();
    Task<ServiceWorkerRegistration?> GetRegistrationAsync();
    Task<bool> CheckForUpdatesAsync();
    Task UpdateServiceWorkerAsync();
    Task RegisterBackgroundSyncAsync(string tag);
    Task<bool> RequestNotificationPermissionAsync();
    Task ShowNotificationAsync(string title, NotificationOptions options);
}

public class PWAService : IPWAService
{
    private readonly IJSRuntime _jsRuntime;
    private readonly ILogger<PWAService> _logger;
    
    public PWAService(IJSRuntime jsRuntime, ILogger<PWAService> logger)
    {
        _jsRuntime = jsRuntime;
        _logger = logger;
    }
    
    public async Task<bool> IsOnlineAsync()
    {
        return await _jsRuntime.InvokeAsync<bool>("navigator.onLine");
    }
    
    public async Task<bool> IsPWAInstalledAsync()
    {
        try
        {
            var displayMode = await _jsRuntime.InvokeAsync<string>(
                "eval", "window.matchMedia('(display-mode: standalone)').matches ? 'standalone' : 'browser'");
            return displayMode == "standalone";
        }
        catch
        {
            return false;
        }
    }
    
    public async Task RequestInstallAsync()
    {
        await _jsRuntime.InvokeVoidAsync("blazorPWA.promptInstall");
    }
    
    public async Task<ServiceWorkerRegistration?> GetRegistrationAsync()
    {
        try
        {
            return await _jsRuntime.InvokeAsync<ServiceWorkerRegistration>(
                "navigator.serviceWorker.getRegistration");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to get service worker registration");
            return null;
        }
    }
    
    public async Task<bool> CheckForUpdatesAsync()
    {
        var registration = await GetRegistrationAsync();
        if (registration == null) return false;
        
        try
        {
            await _jsRuntime.InvokeVoidAsync("eval", "navigator.serviceWorker.registration.update()");
            return true;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to check for updates");
            return false;
        }
    }
    
    public async Task UpdateServiceWorkerAsync()
    {
        await _jsRuntime.InvokeVoidAsync("blazorPWA.updateServiceWorker");
    }
    
    public async Task RegisterBackgroundSyncAsync(string tag)
    {
        await _jsRuntime.InvokeVoidAsync("blazorPWA.registerBackgroundSync", tag);
    }
    
    public async Task<bool> RequestNotificationPermissionAsync()
    {
        var permission = await _jsRuntime.InvokeAsync<string>(
            "Notification.requestPermission");
        return permission == "granted";
    }
    
    public async Task ShowNotificationAsync(string title, NotificationOptions options)
    {
        await _jsRuntime.InvokeVoidAsync("blazorPWA.showNotification", title, options);
    }
}

public class ServiceWorkerRegistration
{
    public string Scope { get; set; } = "";
    public bool Active { get; set; }
    public bool Installing { get; set; }
    public bool Waiting { get; set; }
}

public class NotificationOptions
{
    public string Body { get; set; } = "";
    public string Icon { get; set; } = "";
    public string Badge { get; set; } = "";
    public string Tag { get; set; } = "";
    public bool RequireInteraction { get; set; }
    public object? Data { get; set; }
    public List<NotificationAction> Actions { get; set; } = new();
}

public class NotificationAction
{
    public string Action { get; set; } = "";
    public string Title { get; set; } = "";
    public string Icon { get; set; } = "";
}
```

## 3. 오프라인 데이터 관리

### 3.1 IndexedDB 통합

```csharp
// Services/OfflineDataService.cs
public interface IOfflineDataService
{
    Task<bool> IsAvailableAsync();
    Task SaveDataAsync<T>(string storeName, string key, T data);
    Task<T?> GetDataAsync<T>(string storeName, string key);
    Task<List<T>> GetAllDataAsync<T>(string storeName);
    Task DeleteDataAsync(string storeName, string key);
    Task ClearStoreAsync(string storeName);
    Task<bool> SyncPendingChangesAsync();
}

public class OfflineDataService : IOfflineDataService
{
    private readonly IJSRuntime _jsRuntime;
    private readonly ILogger<OfflineDataService> _logger;
    private readonly string _dbName = "BlazorPWADB";
    private readonly int _dbVersion = 1;
    
    public OfflineDataService(IJSRuntime jsRuntime, ILogger<OfflineDataService> logger)
    {
        _jsRuntime = jsRuntime;
        _logger = logger;
    }
    
    public async Task<bool> IsAvailableAsync()
    {
        return await _jsRuntime.InvokeAsync<bool>("blazorPWA.indexedDB.isAvailable");
    }
    
    public async Task SaveDataAsync<T>(string storeName, string key, T data)
    {
        try
        {
            var wrapper = new OfflineDataWrapper<T>
            {
                Key = key,
                Data = data,
                Timestamp = DateTime.UtcNow,
                IsSynced = false,
                Version = 1
            };
            
            await _jsRuntime.InvokeVoidAsync(
                "blazorPWA.indexedDB.saveData",
                _dbName, storeName, key, wrapper);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to save offline data");
            throw;
        }
    }
    
    public async Task<T?> GetDataAsync<T>(string storeName, string key)
    {
        try
        {
            var wrapper = await _jsRuntime.InvokeAsync<OfflineDataWrapper<T>?>(
                "blazorPWA.indexedDB.getData",
                _dbName, storeName, key);
            
            return wrapper != null ? wrapper.Data : default;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to get offline data");
            return default;
        }
    }
    
    public async Task<List<T>> GetAllDataAsync<T>(string storeName)
    {
        try
        {
            var wrappers = await _jsRuntime.InvokeAsync<List<OfflineDataWrapper<T>>>(
                "blazorPWA.indexedDB.getAllData",
                _dbName, storeName);
            
            return wrappers?.Select(w => w.Data).ToList() ?? new List<T>();
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to get all offline data");
            return new List<T>();
        }
    }
    
    public async Task<bool> SyncPendingChangesAsync()
    {
        try
        {
            var pendingChanges = await _jsRuntime.InvokeAsync<List<PendingChange>>(
                "blazorPWA.indexedDB.getPendingChanges", _dbName);
            
            foreach (var change in pendingChanges)
            {
                try
                {
                    // Process each pending change
                    await ProcessPendingChange(change);
                    
                    // Mark as synced
                    await _jsRuntime.InvokeVoidAsync(
                        "blazorPWA.indexedDB.markAsSynced",
                        _dbName, change.StoreName, change.Key);
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex, "Failed to sync change: {Key}", change.Key);
                }
            }
            
            return true;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to sync pending changes");
            return false;
        }
    }
    
    private async Task ProcessPendingChange(PendingChange change)
    {
        // Implement actual sync logic based on change type
        switch (change.Operation)
        {
            case "create":
            case "update":
                // Send to server
                break;
            case "delete":
                // Delete from server
                break;
        }
    }
}

public class OfflineDataWrapper<T>
{
    public string Key { get; set; } = "";
    public T Data { get; set; } = default!;
    public DateTime Timestamp { get; set; }
    public bool IsSynced { get; set; }
    public int Version { get; set; }
}

public class PendingChange
{
    public string StoreName { get; set; } = "";
    public string Key { get; set; } = "";
    public string Operation { get; set; } = "";
    public object? Data { get; set; }
    public DateTime Timestamp { get; set; }
}
```

### 3.2 오프라인 큐 관리

```csharp
// Services/OfflineQueueService.cs
public interface IOfflineQueueService
{
    Task EnqueueAsync<T>(OfflineRequest<T> request) where T : class;
    Task<List<OfflineRequest<object>>> GetPendingRequestsAsync();
    Task ProcessQueueAsync();
    Task<bool> IsOnlineAsync();
    event EventHandler<OnlineStatusChangedEventArgs> OnlineStatusChanged;
}

public class OfflineQueueService : IOfflineQueueService, IDisposable
{
    private readonly IOfflineDataService _offlineData;
    private readonly IPWAService _pwaService;
    private readonly IHttpClientFactory _httpClientFactory;
    private readonly ILogger<OfflineQueueService> _logger;
    private Timer? _syncTimer;
    private bool _isOnline;
    
    public event EventHandler<OnlineStatusChangedEventArgs>? OnlineStatusChanged;
    
    public OfflineQueueService(
        IOfflineDataService offlineData,
        IPWAService pwaService,
        IHttpClientFactory httpClientFactory,
        ILogger<OfflineQueueService> logger)
    {
        _offlineData = offlineData;
        _pwaService = pwaService;
        _httpClientFactory = httpClientFactory;
        _logger = logger;
        
        StartMonitoring();
    }
    
    private void StartMonitoring()
    {
        _syncTimer = new Timer(async _ => await CheckOnlineStatus(), 
            null, TimeSpan.Zero, TimeSpan.FromSeconds(30));
    }
    
    private async Task CheckOnlineStatus()
    {
        var isOnline = await _pwaService.IsOnlineAsync();
        
        if (isOnline != _isOnline)
        {
            _isOnline = isOnline;
            OnlineStatusChanged?.Invoke(this, new OnlineStatusChangedEventArgs(isOnline));
            
            if (isOnline)
            {
                await ProcessQueueAsync();
            }
        }
    }
    
    public async Task EnqueueAsync<T>(OfflineRequest<T> request) where T : class
    {
        request.Id = Guid.NewGuid().ToString();
        request.CreatedAt = DateTime.UtcNow;
        request.Status = RequestStatus.Pending;
        
        await _offlineData.SaveDataAsync("offline-queue", request.Id, request);
        
        _logger.LogInformation("Request queued for offline processing: {Id}", request.Id);
        
        // Try to process immediately if online
        if (await IsOnlineAsync())
        {
            await ProcessQueueAsync();
        }
    }
    
    public async Task<List<OfflineRequest<object>>> GetPendingRequestsAsync()
    {
        return await _offlineData.GetAllDataAsync<OfflineRequest<object>>("offline-queue");
    }
    
    public async Task ProcessQueueAsync()
    {
        if (!await IsOnlineAsync())
        {
            _logger.LogInformation("Cannot process queue - offline");
            return;
        }
        
        var pendingRequests = await GetPendingRequestsAsync();
        var httpClient = _httpClientFactory.CreateClient();
        
        foreach (var request in pendingRequests.Where(r => r.Status == RequestStatus.Pending))
        {
            try
            {
                // Update status
                request.Status = RequestStatus.Processing;
                await _offlineData.SaveDataAsync("offline-queue", request.Id, request);
                
                // Process request
                var response = await ProcessRequestAsync(httpClient, request);
                
                if (response.IsSuccessStatusCode)
                {
                    // Success - remove from queue
                    await _offlineData.DeleteDataAsync("offline-queue", request.Id);
                    _logger.LogInformation("Offline request processed successfully: {Id}", request.Id);
                }
                else
                {
                    // Failed - update retry count
                    request.RetryCount++;
                    request.Status = RequestStatus.Failed;
                    request.LastError = $"HTTP {response.StatusCode}";
                    await _offlineData.SaveDataAsync("offline-queue", request.Id, request);
                    
                    _logger.LogWarning("Offline request failed: {Id}, Status: {Status}", 
                        request.Id, response.StatusCode);
                }
            }
            catch (Exception ex)
            {
                request.RetryCount++;
                request.Status = RequestStatus.Failed;
                request.LastError = ex.Message;
                await _offlineData.SaveDataAsync("offline-queue", request.Id, request);
                
                _logger.LogError(ex, "Error processing offline request: {Id}", request.Id);
            }
        }
    }
    
    private async Task<HttpResponseMessage> ProcessRequestAsync(
        HttpClient httpClient, OfflineRequest<object> request)
    {
        var httpRequest = new HttpRequestMessage(request.Method, request.Url);
        
        if (request.Headers != null)
        {
            foreach (var header in request.Headers)
            {
                httpRequest.Headers.Add(header.Key, header.Value);
            }
        }
        
        if (request.Body != null && 
            (request.Method == HttpMethod.Post || 
             request.Method == HttpMethod.Put || 
             request.Method == HttpMethod.Patch))
        {
            httpRequest.Content = new StringContent(
                JsonSerializer.Serialize(request.Body),
                Encoding.UTF8,
                "application/json");
        }
        
        return await httpClient.SendAsync(httpRequest);
    }
    
    public async Task<bool> IsOnlineAsync()
    {
        return await _pwaService.IsOnlineAsync();
    }
    
    public void Dispose()
    {
        _syncTimer?.Dispose();
    }
}

public class OfflineRequest<T> where T : class
{
    public string Id { get; set; } = "";
    public string Url { get; set; } = "";
    public HttpMethod Method { get; set; } = HttpMethod.Get;
    public Dictionary<string, string>? Headers { get; set; }
    public T? Body { get; set; }
    public DateTime CreatedAt { get; set; }
    public RequestStatus Status { get; set; }
    public int RetryCount { get; set; }
    public string? LastError { get; set; }
}

public enum RequestStatus
{
    Pending,
    Processing,
    Completed,
    Failed
}

public class OnlineStatusChangedEventArgs : EventArgs
{
    public bool IsOnline { get; }
    
    public OnlineStatusChangedEventArgs(bool isOnline)
    {
        IsOnline = isOnline;
    }
}
```

## 4. PWA UI 컴포넌트

### 4.1 설치 프롬프트

```csharp
// Components/InstallPrompt.razor
@inject IPWAService PWAService
@inject IJSRuntime JS

@if (showInstallPrompt)
{
    <div class="install-prompt">
        <div class="install-content">
            <h4>Install Our App</h4>
            <p>Get a better experience by installing our app on your device!</p>
            <div class="install-buttons">
                <button class="btn btn-primary" @onclick="InstallApp">Install</button>
                <button class="btn btn-secondary" @onclick="DismissPrompt">Not Now</button>
            </div>
        </div>
    </div>
}

@code {
    private bool showInstallPrompt = false;
    private DotNetObjectReference<InstallPrompt>? objRef;
    
    protected override async Task OnInitializedAsync()
    {
        objRef = DotNetObjectReference.Create(this);
        
        // Register for install prompt events
        await JS.InvokeVoidAsync("blazorPWA.registerInstallPrompt", objRef);
        
        // Check if already installed
        var isInstalled = await PWAService.IsPWAInstalledAsync();
        if (!isInstalled)
        {
            // Check if prompt was dismissed recently
            var dismissedDate = await GetDismissedDateAsync();
            if (dismissedDate == null || 
                DateTime.UtcNow - dismissedDate.Value > TimeSpan.FromDays(7))
            {
                showInstallPrompt = true;
            }
        }
    }
    
    [JSInvokable]
    public void OnBeforeInstallPrompt()
    {
        showInstallPrompt = true;
        StateHasChanged();
    }
    
    private async Task InstallApp()
    {
        showInstallPrompt = false;
        await PWAService.RequestInstallAsync();
    }
    
    private async Task DismissPrompt()
    {
        showInstallPrompt = false;
        await SaveDismissedDateAsync(DateTime.UtcNow);
    }
    
    private async Task<DateTime?> GetDismissedDateAsync()
    {
        var dateString = await JS.InvokeAsync<string?>("localStorage.getItem", "install-prompt-dismissed");
        return DateTime.TryParse(dateString, out var date) ? date : null;
    }
    
    private async Task SaveDismissedDateAsync(DateTime date)
    {
        await JS.InvokeVoidAsync("localStorage.setItem", 
            "install-prompt-dismissed", date.ToString("O"));
    }
    
    public void Dispose()
    {
        objRef?.Dispose();
    }
}
```

### 4.2 오프라인 상태 표시

```csharp
// Components/OfflineIndicator.razor
@inject IPWAService PWAService
@inject IOfflineQueueService OfflineQueue
@implements IDisposable

@if (!isOnline)
{
    <div class="offline-indicator">
        <i class="fas fa-wifi-slash"></i>
        <span>Offline Mode</span>
        @if (pendingCount > 0)
        {
            <span class="pending-badge">@pendingCount pending</span>
        }
    </div>
}

@if (showSyncStatus)
{
    <div class="sync-status @(syncSuccess ? "success" : "error")">
        @syncMessage
    </div>
}

@code {
    private bool isOnline = true;
    private int pendingCount = 0;
    private bool showSyncStatus = false;
    private bool syncSuccess = false;
    private string syncMessage = "";
    private Timer? statusTimer;
    
    protected override async Task OnInitializedAsync()
    {
        // Subscribe to online status changes
        OfflineQueue.OnlineStatusChanged += OnOnlineStatusChanged;
        
        // Check initial status
        isOnline = await PWAService.IsOnlineAsync();
        await UpdatePendingCount();
        
        // Start periodic status check
        statusTimer = new Timer(async _ => 
        {
            await UpdatePendingCount();
            await InvokeAsync(StateHasChanged);
        }, null, TimeSpan.Zero, TimeSpan.FromSeconds(10));
    }
    
    private async void OnOnlineStatusChanged(object? sender, OnlineStatusChangedEventArgs e)
    {
        isOnline = e.IsOnline;
        
        if (isOnline)
        {
            await ShowSyncStatus("Syncing offline data...", false);
            
            var success = await OfflineQueue.ProcessQueueAsync();
            await ShowSyncStatus(
                success ? "Data synced successfully" : "Sync failed - will retry",
                success);
        }
        
        await InvokeAsync(StateHasChanged);
    }
    
    private async Task UpdatePendingCount()
    {
        var pending = await OfflineQueue.GetPendingRequestsAsync();
        pendingCount = pending.Count;
    }
    
    private async Task ShowSyncStatus(string message, bool success)
    {
        syncMessage = message;
        syncSuccess = success;
        showSyncStatus = true;
        await InvokeAsync(StateHasChanged);
        
        // Hide after 3 seconds
        _ = Task.Delay(3000).ContinueWith(_ => 
        {
            showSyncStatus = false;
            InvokeAsync(StateHasChanged);
        });
    }
    
    public void Dispose()
    {
        OfflineQueue.OnlineStatusChanged -= OnOnlineStatusChanged;
        statusTimer?.Dispose();
    }
}
```

### 4.3 업데이트 알림

```csharp
// Components/UpdateNotification.razor
@inject IPWAService PWAService
@inject IJSRuntime JS

@if (updateAvailable)
{
    <div class="update-notification">
        <div class="update-content">
            <i class="fas fa-download"></i>
            <span>A new version is available!</span>
            <button class="btn btn-sm btn-primary" @onclick="UpdateApp">Update Now</button>
            <button class="btn btn-sm btn-link" @onclick="DismissUpdate">Later</button>
        </div>
    </div>
}

@code {
    private bool updateAvailable = false;
    private Timer? updateCheckTimer;
    
    protected override async Task OnInitializedAsync()
    {
        // Register for service worker update events
        await JS.InvokeVoidAsync("blazorPWA.onUpdateAvailable", 
            DotNetObjectReference.Create(this));
        
        // Check for updates periodically
        updateCheckTimer = new Timer(async _ => 
        {
            await CheckForUpdates();
        }, null, TimeSpan.FromMinutes(5), TimeSpan.FromMinutes(30));
    }
    
    [JSInvokable]
    public void OnUpdateAvailable()
    {
        updateAvailable = true;
        StateHasChanged();
    }
    
    private async Task CheckForUpdates()
    {
        var hasUpdate = await PWAService.CheckForUpdatesAsync();
        if (hasUpdate)
        {
            updateAvailable = true;
            await InvokeAsync(StateHasChanged);
        }
    }
    
    private async Task UpdateApp()
    {
        await PWAService.UpdateServiceWorkerAsync();
        // Reload the app
        await JS.InvokeVoidAsync("location.reload");
    }
    
    private void DismissUpdate()
    {
        updateAvailable = false;
    }
    
    public void Dispose()
    {
        updateCheckTimer?.Dispose();
    }
}
```

## 5. JavaScript 통합

```javascript
// wwwroot/js/blazor-pwa.js
window.blazorPWA = {
    installPrompt: null,
    
    // Install prompt handling
    registerInstallPrompt: function(dotNetRef) {
        window.addEventListener('beforeinstallprompt', (e) => {
            e.preventDefault();
            this.installPrompt = e;
            dotNetRef.invokeMethodAsync('OnBeforeInstallPrompt');
        });
    },
    
    promptInstall: async function() {
        if (this.installPrompt) {
            this.installPrompt.prompt();
            const result = await this.installPrompt.userChoice;
            console.log('Install prompt result:', result);
            this.installPrompt = null;
        }
    },
    
    // Service worker updates
    onUpdateAvailable: function(dotNetRef) {
        if ('serviceWorker' in navigator) {
            navigator.serviceWorker.ready.then(registration => {
                registration.addEventListener('updatefound', () => {
                    const newWorker = registration.installing;
                    newWorker.addEventListener('statechange', () => {
                        if (newWorker.state === 'installed' && navigator.serviceWorker.controller) {
                            dotNetRef.invokeMethodAsync('OnUpdateAvailable');
                        }
                    });
                });
            });
        }
    },
    
    updateServiceWorker: async function() {
        if ('serviceWorker' in navigator) {
            const registration = await navigator.serviceWorker.ready;
            registration.update();
        }
    },
    
    // Background sync
    registerBackgroundSync: async function(tag) {
        if ('serviceWorker' in navigator && 'SyncManager' in window) {
            const registration = await navigator.serviceWorker.ready;
            await registration.sync.register(tag);
        }
    },
    
    // Notifications
    showNotification: async function(title, options) {
        if ('Notification' in window && Notification.permission === 'granted') {
            const registration = await navigator.serviceWorker.ready;
            registration.showNotification(title, options);
        }
    },
    
    // IndexedDB operations
    indexedDB: {
        isAvailable: function() {
            return 'indexedDB' in window;
        },
        
        openDB: function(dbName, version, stores) {
            return new Promise((resolve, reject) => {
                const request = indexedDB.open(dbName, version);
                
                request.onupgradeneeded = (event) => {
                    const db = event.target.result;
                    stores.forEach(storeName => {
                        if (!db.objectStoreNames.contains(storeName)) {
                            db.createObjectStore(storeName, { keyPath: 'key' });
                        }
                    });
                };
                
                request.onsuccess = () => resolve(request.result);
                request.onerror = () => reject(request.error);
            });
        },
        
        saveData: async function(dbName, storeName, key, data) {
            const db = await this.openDB(dbName, 1, [storeName]);
            const transaction = db.transaction([storeName], 'readwrite');
            const store = transaction.objectStore(storeName);
            
            data.key = key;
            const request = store.put(data);
            
            return new Promise((resolve, reject) => {
                request.onsuccess = () => resolve();
                request.onerror = () => reject(request.error);
            });
        },
        
        getData: async function(dbName, storeName, key) {
            const db = await this.openDB(dbName, 1, [storeName]);
            const transaction = db.transaction([storeName], 'readonly');
            const store = transaction.objectStore(storeName);
            const request = store.get(key);
            
            return new Promise((resolve, reject) => {
                request.onsuccess = () => resolve(request.result);
                request.onerror = () => reject(request.error);
            });
        },
        
        getAllData: async function(dbName, storeName) {
            const db = await this.openDB(dbName, 1, [storeName]);
            const transaction = db.transaction([storeName], 'readonly');
            const store = transaction.objectStore(storeName);
            const request = store.getAll();
            
            return new Promise((resolve, reject) => {
                request.onsuccess = () => resolve(request.result);
                request.onerror = () => reject(request.error);
            });
        },
        
        deleteData: async function(dbName, storeName, key) {
            const db = await this.openDB(dbName, 1, [storeName]);
            const transaction = db.transaction([storeName], 'readwrite');
            const store = transaction.objectStore(storeName);
            const request = store.delete(key);
            
            return new Promise((resolve, reject) => {
                request.onsuccess = () => resolve();
                request.onerror = () => reject(request.error);
            });
        },
        
        getPendingChanges: async function(dbName) {
            const stores = ['offline-queue', 'pending-sync'];
            const pending = [];
            
            for (const storeName of stores) {
                const data = await this.getAllData(dbName, storeName);
                data.forEach(item => {
                    if (!item.isSynced) {
                        pending.push({
                            storeName: storeName,
                            key: item.key,
                            operation: item.operation || 'update',
                            data: item.data,
                            timestamp: item.timestamp
                        });
                    }
                });
            }
            
            return pending;
        }
    }
};
```

## 마무리

Progressive Web App은 Blazor 애플리케이션에 네이티브 앱과 유사한 경험을 제공합니다. Service Worker를 통한 오프라인 지원, IndexedDB를 활용한 로컬 데이터 저장, 백그라운드 동기화, 푸시 알림 등의 기능을 구현하여 사용자에게 끊김 없는 경험을 제공할 수 있습니다. 적절한 캐싱 전략과 오프라인 데이터 관리를 통해 네트워크 연결 상태에 관계없이 안정적인 서비스를 제공할 수 있습니다.