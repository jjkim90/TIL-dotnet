# 가상화와 대용량 데이터

## 개요

Blazor에서 대용량 데이터를 효율적으로 처리하기 위해서는 가상화 기술이 필수적입니다. 이 장에서는 Virtualize 컴포넌트, 무한 스크롤, 페이징 최적화 등 대용량 데이터 처리 기법을 학습합니다.

## 1. Virtualize 컴포넌트

### 1.1 기본 가상화

```csharp
// BasicVirtualization.razor
@page "/virtualization/basic"

<h3>Basic Virtualization</h3>

<div class="virtual-container" style="height: 500px; overflow-y: auto;">
    <Virtualize Items="items" Context="item">
        <div class="item-row">
            <span>@item.Id</span>
            <span>@item.Name</span>
            <span>@item.Description</span>
        </div>
    </Virtualize>
</div>

@code {
    private List<DataItem> items = GenerateItems(10000);
    
    private static List<DataItem> GenerateItems(int count)
    {
        return Enumerable.Range(1, count)
            .Select(i => new DataItem
            {
                Id = i,
                Name = $"Item {i}",
                Description = $"Description for item {i}"
            })
            .ToList();
    }
    
    private class DataItem
    {
        public int Id { get; set; }
        public string Name { get; set; } = "";
        public string Description { get; set; } = "";
    }
}
```

### 1.2 고급 가상화 옵션

```csharp
// AdvancedVirtualization.razor
@page "/virtualization/advanced"

<h3>Advanced Virtualization</h3>

<div class="controls">
    <button @onclick="RefreshData">Refresh</button>
    <label>
        Item Height: 
        <input type="number" @bind="itemHeight" @bind:event="oninput" />
    </label>
</div>

<div class="virtual-container" style="height: 600px; overflow-y: auto;">
    <Virtualize @ref="virtualizeComponent" 
                Items="items" 
                ItemSize="@itemHeight"
                OverscanCount="5"
                Context="item">
        <ItemContent>
            <div class="item-card" style="height: @(itemHeight)px;">
                <h4>@item.Title</h4>
                <p>@item.Content</p>
                <small>Created: @item.CreatedAt.ToString("g")</small>
            </div>
        </ItemContent>
        <Placeholder>
            <div class="item-placeholder" style="height: @(itemHeight)px;">
                <div class="skeleton-text"></div>
                <div class="skeleton-text short"></div>
            </div>
        </Placeholder>
    </Virtualize>
</div>

@code {
    private Virtualize<ComplexItem>? virtualizeComponent;
    private List<ComplexItem> items = new();
    private float itemHeight = 100;
    
    protected override void OnInitialized()
    {
        RefreshData();
    }
    
    private void RefreshData()
    {
        items = GenerateComplexItems(5000);
        virtualizeComponent?.RefreshDataAsync();
    }
    
    private List<ComplexItem> GenerateComplexItems(int count)
    {
        var random = new Random();
        return Enumerable.Range(1, count)
            .Select(i => new ComplexItem
            {
                Id = i,
                Title = $"Complex Item {i}",
                Content = GenerateRandomContent(random.Next(50, 200)),
                CreatedAt = DateTime.Now.AddDays(-random.Next(0, 365))
            })
            .ToList();
    }
    
    private string GenerateRandomContent(int length)
    {
        const string chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789 ";
        var random = new Random();
        return new string(Enumerable.Repeat(chars, length)
            .Select(s => s[random.Next(s.Length)]).ToArray());
    }
    
    private class ComplexItem
    {
        public int Id { get; set; }
        public string Title { get; set; } = "";
        public string Content { get; set; } = "";
        public DateTime CreatedAt { get; set; }
    }
}
```

## 2. 무한 스크롤

### 2.1 ItemsProvider 패턴

```csharp
// InfiniteScroll.razor
@page "/virtualization/infinite"

<h3>Infinite Scroll</h3>

<div class="infinite-container" style="height: 600px; overflow-y: auto;">
    <Virtualize ItemsProvider="LoadItems" Context="item">
        <ItemContent>
            <div class="post-card">
                <h4>@item.Title</h4>
                <p>@item.Body</p>
                <div class="post-meta">
                    <span>By @item.Author</span>
                    <span>@item.PublishedAt.ToString("MMM dd, yyyy")</span>
                </div>
            </div>
        </ItemContent>
        <Placeholder>
            <div class="post-placeholder">
                <div class="skeleton-header"></div>
                <div class="skeleton-body"></div>
            </div>
        </Placeholder>
    </Virtualize>
</div>

<div class="status">
    Total loaded: @totalLoaded | Has more: @hasMore
</div>

@code {
    private int totalLoaded = 0;
    private bool hasMore = true;
    private readonly int pageSize = 20;
    
    private async ValueTask<ItemsProviderResult<Post>> LoadItems(ItemsProviderRequest request)
    {
        // 실제 API 호출 시뮬레이션
        await Task.Delay(500);
        
        var startIndex = request.StartIndex;
        var requestedCount = Math.Min(request.Count, pageSize);
        
        // 최대 1000개까지만 로드
        if (startIndex >= 1000)
        {
            hasMore = false;
            return new ItemsProviderResult<Post>(
                Enumerable.Empty<Post>(), 
                1000);
        }
        
        var posts = GeneratePosts(startIndex, requestedCount);
        totalLoaded = startIndex + posts.Count;
        
        // 총 개수는 알 수 없는 경우 (진짜 무한 스크롤)
        return new ItemsProviderResult<Post>(posts, totalCount: null);
    }
    
    private List<Post> GeneratePosts(int startIndex, int count)
    {
        return Enumerable.Range(startIndex, count)
            .Select(i => new Post
            {
                Id = i,
                Title = $"Post Title {i + 1}",
                Body = $"This is the body of post {i + 1}. It contains some interesting content that users might want to read.",
                Author = $"Author{(i % 5) + 1}",
                PublishedAt = DateTime.Now.AddDays(-i)
            })
            .ToList();
    }
    
    private class Post
    {
        public int Id { get; set; }
        public string Title { get; set; } = "";
        public string Body { get; set; } = "";
        public string Author { get; set; } = "";
        public DateTime PublishedAt { get; set; }
    }
}
```

### 2.2 커스텀 무한 스크롤

```csharp
// CustomInfiniteScroll.razor
@page "/virtualization/custom-infinite"
@implements IAsyncDisposable

<h3>Custom Infinite Scroll</h3>

<div @ref="scrollContainer" 
     class="custom-scroll-container" 
     style="height: 600px; overflow-y: auto;"
     @onscroll="HandleScroll">
    
    @foreach (var item in displayedItems)
    {
        <div class="data-item" data-id="@item.Id">
            <h4>@item.Title</h4>
            <p>@item.Description</p>
            <img src="@item.ImageUrl" loading="lazy" />
        </div>
    }
    
    @if (isLoading)
    {
        <div class="loading-indicator">
            <span>Loading more items...</span>
        </div>
    }
    
    @if (!hasMoreData && displayedItems.Any())
    {
        <div class="end-indicator">
            <span>No more items to load</span>
        </div>
    }
</div>

@code {
    private ElementReference scrollContainer;
    private List<MediaItem> displayedItems = new();
    private bool isLoading = false;
    private bool hasMoreData = true;
    private int currentPage = 0;
    private readonly int itemsPerPage = 20;
    private readonly int scrollThreshold = 200; // pixels from bottom
    
    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            await LoadMoreItems();
        }
    }
    
    private async Task HandleScroll()
    {
        if (isLoading || !hasMoreData) return;
        
        var scrollInfo = await GetScrollInfo();
        
        if (scrollInfo.ScrollTop + scrollInfo.ClientHeight >= 
            scrollInfo.ScrollHeight - scrollThreshold)
        {
            await LoadMoreItems();
        }
    }
    
    private async Task<ScrollInfo> GetScrollInfo()
    {
        // JavaScript interop to get scroll position
        return await JS.InvokeAsync<ScrollInfo>("getScrollInfo", scrollContainer);
    }
    
    private async Task LoadMoreItems()
    {
        if (isLoading) return;
        
        isLoading = true;
        StateHasChanged();
        
        try
        {
            // API 호출 시뮬레이션
            await Task.Delay(1000);
            
            var newItems = await FetchItems(currentPage, itemsPerPage);
            
            if (newItems.Any())
            {
                displayedItems.AddRange(newItems);
                currentPage++;
            }
            else
            {
                hasMoreData = false;
            }
        }
        finally
        {
            isLoading = false;
            StateHasChanged();
        }
    }
    
    private async Task<List<MediaItem>> FetchItems(int page, int pageSize)
    {
        // 실제 API 호출 대신 시뮬레이션
        await Task.CompletedTask;
        
        if (page >= 10) return new List<MediaItem>(); // 10페이지까지만
        
        return Enumerable.Range(page * pageSize, pageSize)
            .Select(i => new MediaItem
            {
                Id = i,
                Title = $"Media Item {i + 1}",
                Description = $"Description for media item {i + 1}",
                ImageUrl = $"https://picsum.photos/300/200?random={i}"
            })
            .ToList();
    }
    
    public async ValueTask DisposeAsync()
    {
        // Cleanup
    }
    
    private class MediaItem
    {
        public int Id { get; set; }
        public string Title { get; set; } = "";
        public string Description { get; set; } = "";
        public string ImageUrl { get; set; } = "";
    }
    
    private class ScrollInfo
    {
        public double ScrollTop { get; set; }
        public double ScrollHeight { get; set; }
        public double ClientHeight { get; set; }
    }
}
```

## 3. 페이징 최적화

### 3.1 서버 사이드 페이징

```csharp
// ServerSidePaging.razor
@page "/paging/server"
@inject HttpClient Http

<h3>Server-Side Paging</h3>

<div class="paging-controls">
    <input @bind="searchTerm" @bind:event="oninput" 
           @onkeypress="@(async (e) => { if (e.Key == "Enter") await Search(); })"
           placeholder="Search..." />
    <select @bind="pageSize" @bind:event="onchange">
        <option value="10">10</option>
        <option value="25">25</option>
        <option value="50">50</option>
        <option value="100">100</option>
    </select>
    <select @bind="sortBy" @bind:event="onchange">
        <option value="name">Name</option>
        <option value="date">Date</option>
        <option value="size">Size</option>
    </select>
</div>

<table class="table">
    <thead>
        <tr>
            <th @onclick="@(() => Sort("name"))">
                Name @GetSortIndicator("name")
            </th>
            <th @onclick="@(() => Sort("date"))">
                Date @GetSortIndicator("date")
            </th>
            <th @onclick="@(() => Sort("size"))">
                Size @GetSortIndicator("size")
            </th>
        </tr>
    </thead>
    <tbody>
        @if (isLoading)
        {
            <tr>
                <td colspan="3">Loading...</td>
            </tr>
        }
        else
        {
            @foreach (var item in pagedData.Items)
            {
                <tr>
                    <td>@item.Name</td>
                    <td>@item.Date.ToString("yyyy-MM-dd")</td>
                    <td>@FormatFileSize(item.Size)</td>
                </tr>
            }
        }
    </tbody>
</table>

<div class="pagination">
    <button @onclick="FirstPage" disabled="@(currentPage == 1)">First</button>
    <button @onclick="PreviousPage" disabled="@(currentPage == 1)">Previous</button>
    
    @foreach (var page in GetPageNumbers())
    {
        <button @onclick="@(() => GoToPage(page))" 
                class="@(page == currentPage ? "active" : "")">
            @page
        </button>
    }
    
    <button @onclick="NextPage" disabled="@(currentPage == totalPages)">Next</button>
    <button @onclick="LastPage" disabled="@(currentPage == totalPages)">Last</button>
</div>

<div class="paging-info">
    Showing @((currentPage - 1) * pageSize + 1) to 
    @(Math.Min(currentPage * pageSize, pagedData.TotalCount)) of 
    @pagedData.TotalCount items
</div>

@code {
    private PagedResult<FileItem> pagedData = new();
    private string searchTerm = "";
    private int currentPage = 1;
    private int pageSize = 25;
    private int totalPages = 1;
    private string sortBy = "name";
    private bool sortDescending = false;
    private bool isLoading = false;
    
    protected override async Task OnInitializedAsync()
    {
        await LoadData();
    }
    
    private async Task LoadData()
    {
        isLoading = true;
        
        var query = new PagingQuery
        {
            Page = currentPage,
            PageSize = pageSize,
            SearchTerm = searchTerm,
            SortBy = sortBy,
            SortDescending = sortDescending
        };
        
        pagedData = await FetchPagedData(query);
        totalPages = (int)Math.Ceiling((double)pagedData.TotalCount / pageSize);
        
        isLoading = false;
    }
    
    private async Task<PagedResult<FileItem>> FetchPagedData(PagingQuery query)
    {
        // 실제 API 호출 시뮬레이션
        await Task.Delay(300);
        
        var allItems = GenerateFileItems(1000);
        
        // 검색
        if (!string.IsNullOrWhiteSpace(query.SearchTerm))
        {
            allItems = allItems.Where(i => 
                i.Name.Contains(query.SearchTerm, StringComparison.OrdinalIgnoreCase))
                .ToList();
        }
        
        // 정렬
        allItems = query.SortBy switch
        {
            "name" => query.SortDescending 
                ? allItems.OrderByDescending(i => i.Name).ToList()
                : allItems.OrderBy(i => i.Name).ToList(),
            "date" => query.SortDescending
                ? allItems.OrderByDescending(i => i.Date).ToList()
                : allItems.OrderBy(i => i.Date).ToList(),
            "size" => query.SortDescending
                ? allItems.OrderByDescending(i => i.Size).ToList()
                : allItems.OrderBy(i => i.Size).ToList(),
            _ => allItems
        };
        
        // 페이징
        var items = allItems
            .Skip((query.Page - 1) * query.PageSize)
            .Take(query.PageSize)
            .ToList();
        
        return new PagedResult<FileItem>
        {
            Items = items,
            TotalCount = allItems.Count,
            Page = query.Page,
            PageSize = query.PageSize
        };
    }
    
    private List<FileItem> GenerateFileItems(int count)
    {
        var random = new Random();
        return Enumerable.Range(1, count)
            .Select(i => new FileItem
            {
                Id = i,
                Name = $"File_{i:D4}.txt",
                Date = DateTime.Now.AddDays(-random.Next(0, 365)),
                Size = random.Next(1024, 1024 * 1024 * 10)
            })
            .ToList();
    }
    
    private async Task Sort(string column)
    {
        if (sortBy == column)
        {
            sortDescending = !sortDescending;
        }
        else
        {
            sortBy = column;
            sortDescending = false;
        }
        
        currentPage = 1;
        await LoadData();
    }
    
    private string GetSortIndicator(string column)
    {
        if (sortBy != column) return "";
        return sortDescending ? "▼" : "▲";
    }
    
    private List<int> GetPageNumbers()
    {
        var pages = new List<int>();
        var start = Math.Max(1, currentPage - 2);
        var end = Math.Min(totalPages, currentPage + 2);
        
        for (int i = start; i <= end; i++)
        {
            pages.Add(i);
        }
        
        return pages;
    }
    
    private async Task Search()
    {
        currentPage = 1;
        await LoadData();
    }
    
    private async Task GoToPage(int page)
    {
        currentPage = page;
        await LoadData();
    }
    
    private async Task FirstPage() => await GoToPage(1);
    private async Task LastPage() => await GoToPage(totalPages);
    private async Task NextPage() => await GoToPage(Math.Min(currentPage + 1, totalPages));
    private async Task PreviousPage() => await GoToPage(Math.Max(currentPage - 1, 1));
    
    private string FormatFileSize(long bytes)
    {
        string[] sizes = { "B", "KB", "MB", "GB" };
        int order = 0;
        double size = bytes;
        
        while (size >= 1024 && order < sizes.Length - 1)
        {
            order++;
            size /= 1024;
        }
        
        return $"{size:0.##} {sizes[order]}";
    }
    
    private class FileItem
    {
        public int Id { get; set; }
        public string Name { get; set; } = "";
        public DateTime Date { get; set; }
        public long Size { get; set; }
    }
    
    private class PagedResult<T>
    {
        public List<T> Items { get; set; } = new();
        public int TotalCount { get; set; }
        public int Page { get; set; }
        public int PageSize { get; set; }
    }
    
    private class PagingQuery
    {
        public int Page { get; set; }
        public int PageSize { get; set; }
        public string SearchTerm { get; set; } = "";
        public string SortBy { get; set; } = "";
        public bool SortDescending { get; set; }
    }
}
```

## 4. 대용량 데이터 최적화

### 4.1 데이터 청크 로딩

```csharp
// ChunkLoading.razor
@page "/data/chunks"

<h3>Chunk Loading for Large Data</h3>

<div class="chunk-progress">
    <div class="progress-bar" style="width: @($"{loadProgress}%")"></div>
    <span>@loadedChunks / @totalChunks chunks loaded</span>
</div>

<div class="data-grid">
    @foreach (var item in loadedData)
    {
        <DataCard Item="item" />
    }
</div>

@if (isLoadingChunk)
{
    <div class="chunk-loading">Loading chunk @(loadedChunks + 1)...</div>
}

@code {
    private List<LargeDataItem> loadedData = new();
    private bool isLoadingChunk = false;
    private int loadedChunks = 0;
    private int totalChunks = 20;
    private int chunkSize = 500;
    private double loadProgress => (double)loadedChunks / totalChunks * 100;
    
    protected override async Task OnInitializedAsync()
    {
        await LoadDataInChunks();
    }
    
    private async Task LoadDataInChunks()
    {
        for (int chunk = 0; chunk < totalChunks; chunk++)
        {
            isLoadingChunk = true;
            StateHasChanged();
            
            var chunkData = await LoadChunk(chunk);
            loadedData.AddRange(chunkData);
            loadedChunks++;
            
            isLoadingChunk = false;
            StateHasChanged();
            
            // Give UI time to update
            await Task.Delay(50);
        }
    }
    
    private async Task<List<LargeDataItem>> LoadChunk(int chunkIndex)
    {
        // Simulate API call
        await Task.Delay(200);
        
        var startId = chunkIndex * chunkSize;
        return Enumerable.Range(startId, chunkSize)
            .Select(i => new LargeDataItem
            {
                Id = i,
                Data = GenerateLargeData(i),
                Timestamp = DateTime.Now
            })
            .ToList();
    }
    
    private string GenerateLargeData(int seed)
    {
        // Generate some data based on seed
        return $"Large data content for item {seed}";
    }
    
    private class LargeDataItem
    {
        public int Id { get; set; }
        public string Data { get; set; } = "";
        public DateTime Timestamp { get; set; }
    }
}

// DataCard.razor
<div class="data-card">
    <h5>Item #@Item.Id</h5>
    <p>@Item.Data</p>
    <small>@Item.Timestamp.ToString("HH:mm:ss")</small>
</div>

@code {
    [Parameter] public LargeDataItem Item { get; set; } = default!;
}
```

### 4.2 메모리 효율적 처리

```csharp
// MemoryEfficientData.razor
@page "/data/memory-efficient"
@implements IDisposable

<h3>Memory Efficient Data Processing</h3>

<div class="controls">
    <button @onclick="ProcessLargeFile">Process Large File</button>
    <button @onclick="CancelProcessing" disabled="@(!isProcessing)">Cancel</button>
    <button @onclick="ClearData">Clear Data</button>
</div>

<div class="stats">
    <p>Processed: @processedCount items</p>
    <p>Memory: @($"{GC.GetTotalMemory(false) / (1024 * 1024):N2}") MB</p>
    <p>Gen 0: @GC.CollectionCount(0), Gen 1: @GC.CollectionCount(1), Gen 2: @GC.CollectionCount(2)</p>
</div>

<div class="results">
    @foreach (var result in processedResults.TakeLast(10))
    {
        <div>@result</div>
    }
</div>

@code {
    private CancellationTokenSource? cts;
    private bool isProcessing = false;
    private int processedCount = 0;
    private List<string> processedResults = new();
    private Timer? gcTimer;
    
    protected override void OnInitialized()
    {
        // Periodic GC stats update
        gcTimer = new Timer(_ => InvokeAsync(StateHasChanged), 
            null, TimeSpan.Zero, TimeSpan.FromSeconds(1));
    }
    
    private async Task ProcessLargeFile()
    {
        isProcessing = true;
        processedCount = 0;
        cts = new CancellationTokenSource();
        
        try
        {
            await foreach (var batch in ReadLargeDataAsync(cts.Token))
            {
                ProcessBatch(batch);
                
                // Force GC periodically to keep memory usage low
                if (processedCount % 10000 == 0)
                {
                    GC.Collect(2, GCCollectionMode.Forced);
                    GC.WaitForPendingFinalizers();
                    GC.Collect(2, GCCollectionMode.Forced);
                }
                
                StateHasChanged();
            }
        }
        catch (OperationCanceledException)
        {
            processedResults.Add("Processing cancelled");
        }
        finally
        {
            isProcessing = false;
            cts?.Dispose();
        }
    }
    
    private async IAsyncEnumerable<DataBatch> ReadLargeDataAsync(
        [EnumeratorCancellation] CancellationToken cancellationToken)
    {
        const int batchSize = 1000;
        var buffer = new DataRecord[batchSize];
        var bufferIndex = 0;
        
        // Simulate reading from large file/database
        for (int i = 0; i < 1000000; i++)
        {
            cancellationToken.ThrowIfCancellationRequested();
            
            buffer[bufferIndex++] = new DataRecord
            {
                Id = i,
                Value = Random.Shared.Next(1000)
            };
            
            if (bufferIndex >= batchSize)
            {
                yield return new DataBatch
                {
                    Records = buffer.ToArray(),
                    BatchNumber = i / batchSize
                };
                
                bufferIndex = 0;
                await Task.Delay(10, cancellationToken);
            }
        }
        
        // Yield remaining items
        if (bufferIndex > 0)
        {
            yield return new DataBatch
            {
                Records = buffer.Take(bufferIndex).ToArray(),
                BatchNumber = -1
            };
        }
    }
    
    private void ProcessBatch(DataBatch batch)
    {
        // Process without keeping references
        var sum = batch.Records.Sum(r => r.Value);
        var avg = sum / batch.Records.Length;
        
        processedCount += batch.Records.Length;
        processedResults.Add($"Batch {batch.BatchNumber}: Avg = {avg}");
        
        // Limit results to prevent memory growth
        if (processedResults.Count > 100)
        {
            processedResults.RemoveRange(0, processedResults.Count - 100);
        }
    }
    
    private void CancelProcessing()
    {
        cts?.Cancel();
    }
    
    private void ClearData()
    {
        processedResults.Clear();
        processedCount = 0;
        GC.Collect();
    }
    
    public void Dispose()
    {
        gcTimer?.Dispose();
        cts?.Dispose();
    }
    
    private struct DataRecord
    {
        public int Id { get; set; }
        public int Value { get; set; }
    }
    
    private class DataBatch
    {
        public DataRecord[] Records { get; set; } = Array.Empty<DataRecord>();
        public int BatchNumber { get; set; }
    }
}
```

## 마무리

Blazor에서 대용량 데이터를 효율적으로 처리하기 위해서는 가상화, 페이징, 청크 로딩 등의 기술을 적절히 활용해야 합니다. Virtualize 컴포넌트는 기본적인 가상화를 쉽게 구현할 수 있게 해주며, ItemsProvider 패턴을 통해 무한 스크롤도 구현할 수 있습니다. 메모리 효율성을 위해서는 데이터를 청크 단위로 처리하고, 필요시 가비지 컬렉션을 적절히 활용하는 것이 중요합니다.