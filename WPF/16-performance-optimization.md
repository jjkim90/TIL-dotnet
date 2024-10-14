# 성능 최적화

## WPF 성능 기초

WPF 애플리케이션의 성능을 최적화하기 위한 핵심 개념과 기법들을 다룹니다.

### 렌더링 계층
```csharp
public static class RenderCapability
{
    public static int Tier => RenderCapability.Tier >> 16;
    
    public static void CheckRenderingTier()
    {
        int renderingTier = RenderCapability.Tier >> 16;
        
        switch (renderingTier)
        {
            case 0:
                // 소프트웨어 렌더링
                Console.WriteLine("Software rendering");
                break;
            case 1:
                // 부분 하드웨어 가속
                Console.WriteLine("Partial hardware acceleration");
                break;
            case 2:
                // 전체 하드웨어 가속
                Console.WriteLine("Full hardware acceleration");
                break;
        }
    }
}
```

## UI 가상화

### VirtualizingStackPanel
```xml
<!-- 기본 가상화 -->
<ListBox ItemsSource="{Binding LargeCollection}"
         VirtualizingPanel.IsVirtualizing="True"
         VirtualizingPanel.VirtualizationMode="Recycling"
         ScrollViewer.CanContentScroll="True">
    <ListBox.ItemsPanel>
        <ItemsPanelTemplate>
            <VirtualizingStackPanel/>
        </ItemsPanelTemplate>
    </ListBox.ItemsPanel>
</ListBox>

<!-- 데이터 가상화 -->
<DataGrid ItemsSource="{Binding LargeDataSet}"
          EnableRowVirtualization="True"
          EnableColumnVirtualization="True"
          VirtualizingPanel.IsVirtualizingWhenGrouping="True">
</DataGrid>
```

### 계층적 가상화
```xml
<TreeView VirtualizingPanel.IsVirtualizing="True"
          VirtualizingPanel.VirtualizationMode="Recycling">
    <TreeView.ItemsPanel>
        <ItemsPanelTemplate>
            <VirtualizingStackPanel/>
        </ItemsPanelTemplate>
    </TreeView.ItemsPanel>
    
    <TreeView.ItemContainerStyle>
        <Style TargetType="TreeViewItem">
            <Setter Property="ItemsPanel">
                <Setter.Value>
                    <ItemsPanelTemplate>
                        <VirtualizingStackPanel/>
                    </ItemsPanelTemplate>
                </Setter.Value>
            </Setter>
        </Style>
    </TreeView.ItemContainerStyle>
</TreeView>
```

### 사용자 정의 가상화 컬렉션
```csharp
public class VirtualizingCollection<T> : IList, INotifyCollectionChanged
{
    private readonly int _pageSize;
    private readonly Dictionary<int, IList<T>> _pages = new();
    private readonly Func<int, int, Task<IList<T>>> _fetchPage;
    private int _count;
    
    public VirtualizingCollection(int count, int pageSize, 
                                 Func<int, int, Task<IList<T>>> fetchPage)
    {
        _count = count;
        _pageSize = pageSize;
        _fetchPage = fetchPage;
    }
    
    public object this[int index]
    {
        get
        {
            var pageIndex = index / _pageSize;
            var pageOffset = index % _pageSize;
            
            if (!_pages.ContainsKey(pageIndex))
            {
                LoadPage(pageIndex);
                return null; // 로딩 중 표시
            }
            
            return _pages[pageIndex][pageOffset];
        }
        set => throw new NotSupportedException();
    }
    
    private async void LoadPage(int pageIndex)
    {
        if (_pages.ContainsKey(pageIndex))
            return;
        
        var startIndex = pageIndex * _pageSize;
        var items = await _fetchPage(startIndex, _pageSize);
        
        _pages[pageIndex] = items;
        
        // 페이지 로드 완료 알림
        var args = new NotifyCollectionChangedEventArgs(
            NotifyCollectionChangedAction.Reset);
        CollectionChanged?.Invoke(this, args);
    }
    
    public int Count => _count;
    public bool IsFixedSize => false;
    public bool IsReadOnly => true;
    public bool IsSynchronized => false;
    public object SyncRoot => this;
    
    public event NotifyCollectionChangedEventHandler CollectionChanged;
    
    // IList 구현 생략...
}
```

## 렌더링 최적화

### BitmapCache 사용
```xml
<!-- 복잡한 시각 요소 캐싱 -->
<Border Width="200" Height="200">
    <Border.CacheMode>
        <BitmapCache EnableClearType="False" 
                     RenderAtScale="1.0"
                     SnapsToDevicePixels="True"/>
    </Border.CacheMode>
    
    <Border.Background>
        <DrawingBrush TileMode="Tile" Viewport="0,0,50,50" ViewportUnits="Absolute">
            <DrawingBrush.Drawing>
                <GeometryDrawing Brush="LightBlue">
                    <GeometryDrawing.Geometry>
                        <PathGeometry>
                            <PathFigure StartPoint="0,0">
                                <LineSegment Point="25,25"/>
                                <LineSegment Point="50,0"/>
                                <LineSegment Point="25,50"/>
                                <LineSegment Point="0,0"/>
                            </PathFigure>
                        </PathGeometry>
                    </GeometryDrawing.Geometry>
                </GeometryDrawing>
            </DrawingBrush.Drawing>
        </DrawingBrush>
    </Border.Background>
</Border>
```

### 렌더링 옵션 설정
```csharp
public static class PerformanceOptimizer
{
    public static void OptimizeForPerformance(Visual visual)
    {
        // 엣지 앤티앨리어싱 비활성화
        RenderOptions.SetEdgeMode(visual, EdgeMode.Aliased);
        
        // 비트맵 스케일링 모드 설정
        RenderOptions.SetBitmapScalingMode(visual, BitmapScalingMode.LowQuality);
        
        // 클리어타입 비활성화
        RenderOptions.SetClearTypeHint(visual, ClearTypeHint.Disabled);
        
        // 캐싱 힌트 설정
        RenderOptions.SetCachingHint(visual, CachingHint.Cache);
        RenderOptions.SetCacheInvalidationThresholdMinimum(visual, 0.5);
        RenderOptions.SetCacheInvalidationThresholdMaximum(visual, 2.0);
    }
    
    public static void OptimizeImage(Image image)
    {
        // 디코딩 크기 제한
        if (image.Source is BitmapSource bitmapSource)
        {
            var bitmap = new BitmapImage();
            bitmap.BeginInit();
            bitmap.DecodePixelWidth = 200;  // 표시 크기에 맞춤
            bitmap.UriSource = new Uri(bitmapSource.ToString());
            bitmap.EndInit();
            bitmap.Freeze(); // 성능 향상을 위해 동결
            
            image.Source = bitmap;
        }
    }
}
```

## 데이터 바인딩 최적화

### 바인딩 모드 최적화
```xml
<!-- OneTime 바인딩 사용 -->
<TextBlock Text="{Binding StaticProperty, Mode=OneTime}"/>

<!-- OneWay 바인딩 (읽기 전용) -->
<TextBlock Text="{Binding ReadOnlyProperty, Mode=OneWay}"/>

<!-- UpdateSourceTrigger 최적화 -->
<TextBox Text="{Binding SearchText, 
                UpdateSourceTrigger=LostFocus,
                Delay=500}"/>

<!-- 바인딩 비활성화 -->
<ItemsControl ItemsSource="{x:Null}">
    <!-- 디자인 타임에만 사용 -->
</ItemsControl>
```

### 가상 프록시 패턴
```csharp
public class LazyLoadingViewModel : ViewModelBase
{
    private ObservableCollection<PersonProxy> _people;
    
    public ObservableCollection<PersonProxy> People
    {
        get
        {
            if (_people == null)
            {
                _people = new ObservableCollection<PersonProxy>(
                    DataService.GetPersonIds()
                        .Select(id => new PersonProxy(id)));
            }
            return _people;
        }
    }
}

public class PersonProxy : ViewModelBase
{
    private readonly int _id;
    private Person _person;
    private bool _isLoaded;
    
    public PersonProxy(int id)
    {
        _id = id;
    }
    
    public string Name
    {
        get
        {
            EnsureLoaded();
            return _person?.Name ?? "Loading...";
        }
    }
    
    public string Department
    {
        get
        {
            EnsureLoaded();
            return _person?.Department ?? "";
        }
    }
    
    private void EnsureLoaded()
    {
        if (!_isLoaded)
        {
            _person = DataService.LoadPerson(_id);
            _isLoaded = true;
            
            OnPropertyChanged(nameof(Name));
            OnPropertyChanged(nameof(Department));
        }
    }
}
```

### 컬렉션 업데이트 최적화
```csharp
public class OptimizedCollectionViewModel : ViewModelBase
{
    private readonly ObservableCollection<Item> _items = new();
    private readonly DispatcherTimer _batchTimer;
    private readonly List<Item> _pendingItems = new();
    
    public OptimizedCollectionViewModel()
    {
        _batchTimer = new DispatcherTimer
        {
            Interval = TimeSpan.FromMilliseconds(100)
        };
        _batchTimer.Tick += OnBatchTimerTick;
    }
    
    public ObservableCollection<Item> Items => _items;
    
    public void AddItem(Item item)
    {
        _pendingItems.Add(item);
        _batchTimer.Start();
    }
    
    private void OnBatchTimerTick(object sender, EventArgs e)
    {
        _batchTimer.Stop();
        
        // 일괄 업데이트
        using (Items.DeferRefresh())
        {
            foreach (var item in _pendingItems)
            {
                _items.Add(item);
            }
        }
        
        _pendingItems.Clear();
    }
    
    public void UpdateMultipleItems(IEnumerable<Item> items)
    {
        // CollectionChanged 이벤트 일시 중지
        var collection = Items as INotifyCollectionChanged;
        
        // 대량 업데이트
        _items.Clear();
        foreach (var item in items)
        {
            _items.Add(item);
        }
    }
}
```

## 메모리 최적화

### Weak References 사용
```csharp
public class WeakEventManager<TEventArgs> where TEventArgs : EventArgs
{
    private readonly List<WeakReference> _handlers = new();
    
    public void AddHandler(EventHandler<TEventArgs> handler)
    {
        _handlers.Add(new WeakReference(handler));
        CleanupHandlers();
    }
    
    public void RemoveHandler(EventHandler<TEventArgs> handler)
    {
        _handlers.RemoveAll(wr => !wr.IsAlive || wr.Target.Equals(handler));
    }
    
    public void RaiseEvent(object sender, TEventArgs args)
    {
        foreach (var weakRef in _handlers.ToList())
        {
            if (weakRef.Target is EventHandler<TEventArgs> handler)
            {
                handler(sender, args);
            }
        }
        
        CleanupHandlers();
    }
    
    private void CleanupHandlers()
    {
        _handlers.RemoveAll(wr => !wr.IsAlive);
    }
}
```

### 리소스 관리
```csharp
public class ResourceOptimizer
{
    private static readonly Dictionary<string, WeakReference> _imageCache = new();
    
    public static BitmapImage GetOptimizedImage(string path)
    {
        if (_imageCache.TryGetValue(path, out var weakRef) && 
            weakRef.IsAlive && 
            weakRef.Target is BitmapImage cachedImage)
        {
            return cachedImage;
        }
        
        var bitmap = new BitmapImage();
        bitmap.BeginInit();
        bitmap.CacheOption = BitmapCacheOption.OnLoad;
        bitmap.CreateOptions = BitmapCreateOptions.DelayCreation;
        bitmap.UriSource = new Uri(path, UriKind.RelativeOrAbsolute);
        
        // 메모리 사용량 감소를 위한 크기 제한
        if (GetFileSize(path) > 1024 * 1024) // 1MB 이상
        {
            bitmap.DecodePixelWidth = 800;
        }
        
        bitmap.EndInit();
        bitmap.Freeze(); // 스레드 간 공유 가능
        
        _imageCache[path] = new WeakReference(bitmap);
        
        return bitmap;
    }
    
    public static void ClearUnusedResources()
    {
        // 사용하지 않는 리소스 정리
        var keysToRemove = _imageCache
            .Where(kvp => !kvp.Value.IsAlive)
            .Select(kvp => kvp.Key)
            .ToList();
        
        foreach (var key in keysToRemove)
        {
            _imageCache.Remove(key);
        }
        
        // 가비지 컬렉션 강제 실행 (주의해서 사용)
        GC.Collect();
        GC.WaitForPendingFinalizers();
        GC.Collect();
    }
}
```

## 스레드 최적화

### 비동기 작업 최적화
```csharp
public class AsyncDataLoader : ViewModelBase
{
    private readonly SemaphoreSlim _loadingSemaphore = new(3); // 동시 3개 제한
    private CancellationTokenSource _cancellationTokenSource;
    
    public async Task LoadDataAsync()
    {
        _cancellationTokenSource?.Cancel();
        _cancellationTokenSource = new CancellationTokenSource();
        var token = _cancellationTokenSource.Token;
        
        await _loadingSemaphore.WaitAsync(token);
        try
        {
            // 백그라운드에서 데이터 로드
            var data = await Task.Run(() => LoadDataFromDatabase(), token);
            
            // UI 스레드에서 업데이트
            await Application.Current.Dispatcher.InvokeAsync(() =>
            {
                UpdateUI(data);
            }, DispatcherPriority.DataBind);
        }
        finally
        {
            _loadingSemaphore.Release();
        }
    }
    
    private void UpdateUI(object data)
    {
        // 우선순위별 업데이트
        Dispatcher.CurrentDispatcher.BeginInvoke(
            DispatcherPriority.Background,
            new Action(() => UpdateLowPriorityData(data)));
        
        Dispatcher.CurrentDispatcher.BeginInvoke(
            DispatcherPriority.Normal,
            new Action(() => UpdateNormalPriorityData(data)));
        
        Dispatcher.CurrentDispatcher.BeginInvoke(
            DispatcherPriority.Send,
            new Action(() => UpdateHighPriorityData(data)));
    }
}
```

### BackgroundWorker 대체
```csharp
public class ProgressiveLoader : ViewModelBase
{
    private readonly Progress<int> _progress;
    private double _progressValue;
    
    public double ProgressValue
    {
        get => _progressValue;
        set => SetProperty(ref _progressValue, value);
    }
    
    public ProgressiveLoader()
    {
        _progress = new Progress<int>(percent =>
        {
            ProgressValue = percent;
        });
    }
    
    public async Task LoadLargeDataSetAsync()
    {
        var items = new List<DataItem>();
        
        await Task.Run(() =>
        {
            var totalItems = 10000;
            
            for (int i = 0; i < totalItems; i++)
            {
                // 데이터 로드
                var item = LoadDataItem(i);
                items.Add(item);
                
                // 진행률 보고
                if (i % 100 == 0)
                {
                    _progress.Report((i * 100) / totalItems);
                }
            }
        });
        
        // UI 업데이트
        await Application.Current.Dispatcher.InvokeAsync(() =>
        {
            UpdateCollection(items);
        });
    }
}
```

## 애니메이션 최적화

### 하드웨어 가속 애니메이션
```xml
<!-- GPU 가속 Transform 애니메이션 -->
<Rectangle Width="100" Height="100" Fill="Blue">
    <Rectangle.RenderTransform>
        <TransformGroup>
            <TranslateTransform x:Name="translateTransform"/>
            <ScaleTransform x:Name="scaleTransform"/>
            <RotateTransform x:Name="rotateTransform"/>
        </TransformGroup>
    </Rectangle.RenderTransform>
    
    <Rectangle.Triggers>
        <EventTrigger RoutedEvent="Loaded">
            <BeginStoryboard>
                <Storyboard>
                    <!-- GPU에서 처리됨 -->
                    <DoubleAnimation Storyboard.TargetName="translateTransform"
                                   Storyboard.TargetProperty="X"
                                   From="0" To="200" Duration="0:0:2"
                                   RepeatBehavior="Forever"/>
                </Storyboard>
            </BeginStoryboard>
        </EventTrigger>
    </Rectangle.Triggers>
</Rectangle>

<!-- 피해야 할 속성 애니메이션 -->
<!-- Width, Height, Margin 등은 레이아웃 재계산 발생 -->
```

### CompositionTarget 최적화
```csharp
public class OptimizedAnimation
{
    private long _lastRenderTime;
    private readonly int _targetFps = 30;
    private readonly long _frameInterval;
    
    public OptimizedAnimation()
    {
        _frameInterval = TimeSpan.FromSeconds(1.0 / _targetFps).Ticks;
        CompositionTarget.Rendering += OnRendering;
    }
    
    private void OnRendering(object sender, EventArgs e)
    {
        var renderingEventArgs = e as RenderingEventArgs;
        var currentTime = renderingEventArgs.RenderingTime.Ticks;
        
        // 프레임 레이트 제한
        if (currentTime - _lastRenderTime < _frameInterval)
            return;
        
        _lastRenderTime = currentTime;
        
        // 애니메이션 업데이트
        UpdateAnimation();
    }
    
    private void UpdateAnimation()
    {
        // 애니메이션 로직
    }
    
    public void Stop()
    {
        CompositionTarget.Rendering -= OnRendering;
    }
}
```

## 측정 및 프로파일링

### 성능 측정 도구
```csharp
public class PerformanceMonitor
{
    private readonly Stopwatch _stopwatch = new();
    private readonly Dictionary<string, List<long>> _measurements = new();
    
    public IDisposable MeasureBlock(string operationName)
    {
        return new PerformanceMeasurement(this, operationName);
    }
    
    private class PerformanceMeasurement : IDisposable
    {
        private readonly PerformanceMonitor _monitor;
        private readonly string _operationName;
        private readonly Stopwatch _stopwatch;
        
        public PerformanceMeasurement(PerformanceMonitor monitor, string operationName)
        {
            _monitor = monitor;
            _operationName = operationName;
            _stopwatch = Stopwatch.StartNew();
        }
        
        public void Dispose()
        {
            _stopwatch.Stop();
            _monitor.RecordMeasurement(_operationName, _stopwatch.ElapsedMilliseconds);
        }
    }
    
    private void RecordMeasurement(string operationName, long elapsedMs)
    {
        if (!_measurements.ContainsKey(operationName))
            _measurements[operationName] = new List<long>();
        
        _measurements[operationName].Add(elapsedMs);
        
        // 평균 계산 및 로깅
        var average = _measurements[operationName].Average();
        if (average > 16) // 60 FPS 기준
        {
            Debug.WriteLine($"Performance warning: {operationName} took {elapsedMs}ms (avg: {average:F2}ms)");
        }
    }
    
    public void GenerateReport()
    {
        foreach (var kvp in _measurements)
        {
            var stats = new
            {
                Operation = kvp.Key,
                Count = kvp.Value.Count,
                Average = kvp.Value.Average(),
                Min = kvp.Value.Min(),
                Max = kvp.Value.Max(),
                Total = kvp.Value.Sum()
            };
            
            Debug.WriteLine($"{stats.Operation}: Avg={stats.Average:F2}ms, " +
                          $"Min={stats.Min}ms, Max={stats.Max}ms, " +
                          $"Total={stats.Total}ms ({stats.Count} calls)");
        }
    }
}
```

### Visual Tree 분석
```csharp
public static class VisualTreeAnalyzer
{
    public static void AnalyzeVisualTree(Visual root)
    {
        var stats = new VisualTreeStats();
        AnalyzeVisualTreeRecursive(root, stats, 0);
        
        Console.WriteLine($"Total visuals: {stats.TotalVisuals}");
        Console.WriteLine($"Max depth: {stats.MaxDepth}");
        Console.WriteLine($"Controls by type:");
        
        foreach (var kvp in stats.TypeCounts.OrderByDescending(x => x.Value))
        {
            Console.WriteLine($"  {kvp.Key}: {kvp.Value}");
        }
    }
    
    private static void AnalyzeVisualTreeRecursive(DependencyObject element, 
                                                   VisualTreeStats stats, 
                                                   int depth)
    {
        stats.TotalVisuals++;
        stats.MaxDepth = Math.Max(stats.MaxDepth, depth);
        
        var typeName = element.GetType().Name;
        if (!stats.TypeCounts.ContainsKey(typeName))
            stats.TypeCounts[typeName] = 0;
        stats.TypeCounts[typeName]++;
        
        // 복잡도 경고
        if (depth > 15)
        {
            Debug.WriteLine($"Warning: Deep visual tree at depth {depth}");
        }
        
        int childCount = VisualTreeHelper.GetChildrenCount(element);
        for (int i = 0; i < childCount; i++)
        {
            var child = VisualTreeHelper.GetChild(element, i);
            AnalyzeVisualTreeRecursive(child, stats, depth + 1);
        }
    }
    
    private class VisualTreeStats
    {
        public int TotalVisuals { get; set; }
        public int MaxDepth { get; set; }
        public Dictionary<string, int> TypeCounts { get; } = new();
    }
}
```

## 실전 최적화 예제

### 대용량 데이터 그리드
```csharp
public class OptimizedDataGrid : DataGrid
{
    protected override void OnItemsSourceChanged(
        IEnumerable oldValue, IEnumerable newValue)
    {
        // 가상화 설정 확인
        SetValue(VirtualizingPanel.IsVirtualizingProperty, true);
        SetValue(VirtualizingPanel.VirtualizationModeProperty, 
                VirtualizationMode.Recycling);
        
        // 대용량 데이터셋 최적화
        if (newValue is ICollection collection && collection.Count > 1000)
        {
            EnableRowVirtualization = true;
            EnableColumnVirtualization = true;
            
            // 스크롤 성능 향상
            SetValue(ScrollViewer.IsDeferredScrollingEnabledProperty, true);
            
            // 선택 모드 최적화
            if (SelectionMode == DataGridSelectionMode.Extended)
            {
                SelectionMode = DataGridSelectionMode.Single;
            }
        }
        
        base.OnItemsSourceChanged(oldValue, newValue);
    }
    
    protected override void OnAutoGeneratingColumn(
        DataGridAutoGeneratingColumnEventArgs e)
    {
        base.OnAutoGeneratingColumn(e);
        
        // 컬럼 너비 최적화
        if (e.Column is DataGridTextColumn textColumn)
        {
            // 자동 크기 조정 비활성화
            textColumn.Width = new DataGridLength(100);
            
            // 텍스트 트리밍
            var style = new Style(typeof(TextBlock));
            style.Setters.Add(new Setter(TextBlock.TextTrimmingProperty, 
                                       TextTrimming.CharacterEllipsis));
            textColumn.ElementStyle = style;
        }
    }
}
```

### 이미지 갤러리 최적화
```csharp
public class OptimizedImageGallery : ItemsControl
{
    private readonly ObjectPool<Image> _imagePool;
    private readonly ConcurrentDictionary<string, BitmapImage> _imageCache;
    
    public OptimizedImageGallery()
    {
        _imagePool = new ObjectPool<Image>(() => new Image());
        _imageCache = new ConcurrentDictionary<string, BitmapImage>();
        
        // 가상화 패널 설정
        var factory = new FrameworkElementFactory(typeof(VirtualizingWrapPanel));
        factory.SetValue(VirtualizingWrapPanel.OrientationProperty, Orientation.Horizontal);
        ItemsPanel = new ItemsPanelTemplate(factory);
        
        // 스크롤 가상화
        SetValue(ScrollViewer.CanContentScrollProperty, true);
        SetValue(VirtualizingPanel.IsVirtualizingProperty, true);
    }
    
    protected override DependencyObject GetContainerForItemOverride()
    {
        return new OptimizedImageContainer(_imagePool, _imageCache);
    }
}

public class OptimizedImageContainer : ContentControl
{
    private readonly ObjectPool<Image> _imagePool;
    private readonly ConcurrentDictionary<string, BitmapImage> _imageCache;
    private Image _currentImage;
    
    public OptimizedImageContainer(ObjectPool<Image> imagePool,
                                  ConcurrentDictionary<string, BitmapImage> imageCache)
    {
        _imagePool = imagePool;
        _imageCache = imageCache;
    }
    
    protected override void OnContentChanged(object oldContent, object newContent)
    {
        base.OnContentChanged(oldContent, newContent);
        
        if (oldContent != null && _currentImage != null)
        {
            // 이미지 풀에 반환
            _currentImage.Source = null;
            _imagePool.Return(_currentImage);
            _currentImage = null;
        }
        
        if (newContent is string imagePath)
        {
            _currentImage = _imagePool.Rent();
            LoadImageAsync(imagePath);
        }
    }
    
    private async void LoadImageAsync(string imagePath)
    {
        // 캐시 확인
        if (_imageCache.TryGetValue(imagePath, out var cachedImage))
        {
            _currentImage.Source = cachedImage;
            Content = _currentImage;
            return;
        }
        
        // 비동기 로드
        var bitmap = await Task.Run(() =>
        {
            var bmp = new BitmapImage();
            bmp.BeginInit();
            bmp.CacheOption = BitmapCacheOption.OnLoad;
            bmp.DecodePixelWidth = 200; // 썸네일 크기
            bmp.UriSource = new Uri(imagePath);
            bmp.EndInit();
            bmp.Freeze();
            return bmp;
        });
        
        _imageCache.TryAdd(imagePath, bitmap);
        
        if (_currentImage != null)
        {
            _currentImage.Source = bitmap;
            Content = _currentImage;
        }
    }
}

// 객체 풀 구현
public class ObjectPool<T> where T : class
{
    private readonly ConcurrentBag<T> _objects = new();
    private readonly Func<T> _objectGenerator;
    
    public ObjectPool(Func<T> objectGenerator)
    {
        _objectGenerator = objectGenerator;
    }
    
    public T Rent()
    {
        return _objects.TryTake(out T item) ? item : _objectGenerator();
    }
    
    public void Return(T item)
    {
        _objects.Add(item);
    }
}
```

### 실시간 차트 최적화
```csharp
public class OptimizedRealtimeChart : FrameworkElement
{
    private readonly List<double> _dataPoints = new();
    private readonly int _maxPoints = 1000;
    private WriteableBitmap _bitmap;
    private Int32Rect _dirtyRect;
    private readonly object _lockObject = new();
    
    protected override void OnRenderSizeChanged(SizeChangedInfo sizeInfo)
    {
        base.OnRenderSizeChanged(sizeInfo);
        
        // 비트맵 재생성
        _bitmap = new WriteableBitmap(
            (int)sizeInfo.NewSize.Width,
            (int)sizeInfo.NewSize.Height,
            96, 96, PixelFormats.Pbgra32, null);
        
        InvalidateVisual();
    }
    
    public void AddDataPoint(double value)
    {
        lock (_lockObject)
        {
            _dataPoints.Add(value);
            
            // 최대 포인트 수 제한
            if (_dataPoints.Count > _maxPoints)
            {
                _dataPoints.RemoveAt(0);
            }
        }
        
        // 비동기 렌더링
        Dispatcher.BeginInvoke(DispatcherPriority.Render, new Action(UpdateBitmap));
    }
    
    private void UpdateBitmap()
    {
        if (_bitmap == null) return;
        
        _bitmap.Lock();
        
        try
        {
            // 부분 업데이트만 수행
            unsafe
            {
                var buffer = (byte*)_bitmap.BackBuffer;
                var stride = _bitmap.BackBufferStride;
                var width = _bitmap.PixelWidth;
                var height = _bitmap.PixelHeight;
                
                // 이전 프레임의 오른쪽 부분만 지우기
                var clearStartX = width - 10;
                for (int y = 0; y < height; y++)
                {
                    for (int x = clearStartX; x < width; x++)
                    {
                        var offset = y * stride + x * 4;
                        buffer[offset] = 255;     // B
                        buffer[offset + 1] = 255; // G
                        buffer[offset + 2] = 255; // R
                        buffer[offset + 3] = 255; // A
                    }
                }
                
                // 새 데이터 포인트 그리기
                lock (_lockObject)
                {
                    if (_dataPoints.Count > 1)
                    {
                        var pointWidth = (double)width / _maxPoints;
                        var lastIndex = _dataPoints.Count - 1;
                        
                        // 마지막 선분만 그리기
                        var x1 = (int)((lastIndex - 1) * pointWidth);
                        var y1 = (int)(height - _dataPoints[lastIndex - 1] * height);
                        var x2 = (int)(lastIndex * pointWidth);
                        var y2 = (int)(height - _dataPoints[lastIndex] * height);
                        
                        DrawLine(buffer, stride, width, height, x1, y1, x2, y2);
                        
                        _dirtyRect = new Int32Rect(x1, Math.Min(y1, y2), 
                                                   x2 - x1 + 1, 
                                                   Math.Abs(y2 - y1) + 1);
                    }
                }
            }
            
            _bitmap.AddDirtyRect(_dirtyRect);
        }
        finally
        {
            _bitmap.Unlock();
        }
        
        InvalidateVisual();
    }
    
    private unsafe void DrawLine(byte* buffer, int stride, int width, int height,
                                int x1, int y1, int x2, int y2)
    {
        // Bresenham's line algorithm
        int dx = Math.Abs(x2 - x1);
        int dy = Math.Abs(y2 - y1);
        int sx = x1 < x2 ? 1 : -1;
        int sy = y1 < y2 ? 1 : -1;
        int err = dx - dy;
        
        while (true)
        {
            if (x1 >= 0 && x1 < width && y1 >= 0 && y1 < height)
            {
                var offset = y1 * stride + x1 * 4;
                buffer[offset] = 0;       // B
                buffer[offset + 1] = 0;   // G
                buffer[offset + 2] = 255; // R
                buffer[offset + 3] = 255; // A
            }
            
            if (x1 == x2 && y1 == y2) break;
            
            int e2 = 2 * err;
            if (e2 > -dy)
            {
                err -= dy;
                x1 += sx;
            }
            if (e2 < dx)
            {
                err += dx;
                y1 += sy;
            }
        }
    }
    
    protected override void OnRender(DrawingContext drawingContext)
    {
        if (_bitmap != null)
        {
            drawingContext.DrawImage(_bitmap, 
                new Rect(0, 0, ActualWidth, ActualHeight));
        }
    }
}
```

## 최적화 체크리스트

### 일반 최적화
```csharp
public static class OptimizationChecklist
{
    public static void ApplyGeneralOptimizations(Window window)
    {
        // 1. 하드웨어 가속 확인
        if (RenderCapability.Tier >> 16 < 2)
        {
            // 소프트웨어 렌더링 최적화 적용
            ApplySoftwareRenderingOptimizations(window);
        }
        
        // 2. 창 투명도 제거
        if (window.AllowsTransparency)
        {
            window.AllowsTransparency = false;
            window.WindowStyle = WindowStyle.SingleBorderWindow;
        }
        
        // 3. 애니메이션 최적화
        Timeline.DesiredFrameRateProperty.OverrideMetadata(
            typeof(Timeline),
            new FrameworkPropertyMetadata(30));
        
        // 4. 텍스트 렌더링 최적화
        TextOptions.SetTextFormattingMode(window, TextFormattingMode.Display);
        TextOptions.SetTextRenderingMode(window, TextRenderingMode.ClearType);
        
        // 5. 레이아웃 반올림
        window.UseLayoutRounding = true;
        window.SnapsToDevicePixels = true;
    }
    
    public static void OptimizeDataBinding(FrameworkElement element)
    {
        // 1. 불필요한 양방향 바인딩 제거
        var bindings = element.GetBindings();
        foreach (var binding in bindings)
        {
            if (binding.Mode == BindingMode.TwoWay && 
                !binding.Source.GetType().GetProperty(binding.Path.Path).CanWrite)
            {
                binding.Mode = BindingMode.OneWay;
            }
        }
        
        // 2. 업데이트 트리거 최적화
        foreach (var binding in bindings)
        {
            if (binding.UpdateSourceTrigger == UpdateSourceTrigger.PropertyChanged)
            {
                // 텍스트 입력이 아닌 경우 LostFocus로 변경
                binding.UpdateSourceTrigger = UpdateSourceTrigger.LostFocus;
            }
        }
    }
}
```

### 메모리 프로파일링
```csharp
public class MemoryProfiler
{
    private readonly Timer _timer;
    private long _lastTotalMemory;
    
    public MemoryProfiler()
    {
        _timer = new Timer(CheckMemory, null, TimeSpan.Zero, TimeSpan.FromSeconds(10));
    }
    
    private void CheckMemory(object state)
    {
        var currentMemory = GC.GetTotalMemory(false);
        var delta = currentMemory - _lastTotalMemory;
        
        if (Math.Abs(delta) > 1024 * 1024) // 1MB 변화
        {
            Debug.WriteLine($"Memory: {currentMemory / 1024 / 1024}MB " +
                          $"(Change: {delta / 1024 / 1024:+#;-#;0}MB)");
            
            // 메모리 증가 경고
            if (delta > 10 * 1024 * 1024) // 10MB 증가
            {
                Debug.WriteLine("WARNING: Large memory increase detected!");
                
                // 상세 정보 수집
                for (int gen = 0; gen <= GC.MaxGeneration; gen++)
                {
                    Debug.WriteLine($"Gen {gen} collections: {GC.CollectionCount(gen)}");
                }
            }
        }
        
        _lastTotalMemory = currentMemory;
    }
}
```

## 핵심 개념 정리
- **UI 가상화**: VirtualizingStackPanel을 통한 대용량 데이터 처리
- **렌더링 최적화**: BitmapCache, RenderOptions 설정
- **바인딩 최적화**: Mode, UpdateSourceTrigger, 가상 프록시
- **메모리 최적화**: WeakReference, 리소스 재사용
- **스레드 최적화**: 비동기 작업, Dispatcher 우선순위
- **애니메이션 최적화**: GPU 가속, Transform 애니메이션
- **측정 도구**: 성능 모니터링, Visual Tree 분석
- **객체 풀링**: 리소스 재사용을 통한 메모리 절약
- **비트맵 최적화**: WriteableBitmap, 부분 업데이트
- **프로파일링**: 메모리 및 성능 측정