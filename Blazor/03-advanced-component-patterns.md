# 고급 컴포넌트 패턴

## 개요

Blazor의 고급 컴포넌트 패턴을 활용하면 재사용 가능하고 유연한 컴포넌트를 만들 수 있습니다. RenderFragment, Templated Components, Generic Components 등의 패턴을 통해 더욱 강력하고 확장 가능한 애플리케이션을 구축할 수 있습니다.

## 1. RenderFragment 심화

### 1.1 RenderFragment 기본 개념

RenderFragment는 Blazor의 렌더링 가능한 콘텐츠 조각을 나타내는 델리게이트입니다.

```csharp
// Card.razor
<div class="card">
    <div class="card-header">
        @HeaderContent
    </div>
    <div class="card-body">
        @BodyContent
    </div>
    @if (FooterContent != null)
    {
        <div class="card-footer">
            @FooterContent
        </div>
    }
</div>

@code {
    [Parameter] public RenderFragment HeaderContent { get; set; } = default!;
    [Parameter] public RenderFragment BodyContent { get; set; } = default!;
    [Parameter] public RenderFragment? FooterContent { get; set; }
}

// 사용 예제
<Card>
    <HeaderContent>
        <h3>제목</h3>
    </HeaderContent>
    <BodyContent>
        <p>카드 본문 내용입니다.</p>
    </BodyContent>
    <FooterContent>
        <button class="btn btn-primary">액션</button>
    </FooterContent>
</Card>
```

### 1.2 동적 RenderFragment 생성

```csharp
@page "/dynamic-render"

<h3>Dynamic RenderFragment Example</h3>

<div class="content-area">
    @dynamicContent
</div>

<button @onclick="ChangeToBold">Bold Text</button>
<button @onclick="ChangeToItalic">Italic Text</button>
<button @onclick="ChangeToList">List View</button>

@code {
    private RenderFragment dynamicContent = builder => { };
    private string[] items = { "Item 1", "Item 2", "Item 3" };

    protected override void OnInitialized()
    {
        dynamicContent = CreateDefaultContent();
    }

    private RenderFragment CreateDefaultContent()
    {
        return builder =>
        {
            builder.OpenElement(0, "p");
            builder.AddContent(1, "기본 콘텐츠입니다.");
            builder.CloseElement();
        };
    }

    private void ChangeToBold()
    {
        dynamicContent = builder =>
        {
            builder.OpenElement(0, "strong");
            builder.AddContent(1, "굵은 텍스트로 표시됩니다.");
            builder.CloseElement();
        };
    }

    private void ChangeToItalic()
    {
        dynamicContent = builder =>
        {
            builder.OpenElement(0, "em");
            builder.AddContent(1, "이탤릭체로 표시됩니다.");
            builder.CloseElement();
        };
    }

    private void ChangeToList()
    {
        dynamicContent = builder =>
        {
            var seq = 0;
            builder.OpenElement(seq++, "ul");
            
            foreach (var item in items)
            {
                builder.OpenElement(seq++, "li");
                builder.AddContent(seq++, item);
                builder.CloseElement();
            }
            
            builder.CloseElement();
        };
    }
}
```

### 1.3 조건부 RenderFragment

```csharp
// ConditionalContent.razor
@if (ShowContent)
{
    @Content
}
else if (AlternativeContent != null)
{
    @AlternativeContent
}
else
{
    <p class="text-muted">콘텐츠가 없습니다.</p>
}

@code {
    [Parameter] public bool ShowContent { get; set; }
    [Parameter] public RenderFragment Content { get; set; } = default!;
    [Parameter] public RenderFragment? AlternativeContent { get; set; }
}

// LayoutSelector.razor
<div class="layout-container">
    @switch(LayoutType)
    {
        case "grid":
            @GridLayout
            break;
        case "list":
            @ListLayout
            break;
        case "card":
            @CardLayout
            break;
        default:
            @DefaultLayout
            break;
    }
</div>

@code {
    [Parameter] public string LayoutType { get; set; } = "default";
    [Parameter] public RenderFragment GridLayout { get; set; } = default!;
    [Parameter] public RenderFragment ListLayout { get; set; } = default!;
    [Parameter] public RenderFragment CardLayout { get; set; } = default!;
    [Parameter] public RenderFragment DefaultLayout { get; set; } = default!;
}
```

## 2. Templated Components

### 2.1 기본 Templated Component

```csharp
// DataList.razor
@typeparam TItem

<div class="data-list">
    @if (Items != null && Items.Any())
    {
        @foreach (var item in Items)
        {
            <div class="list-item">
                @ItemTemplate(item)
            </div>
        }
    }
    else
    {
        @if (EmptyTemplate != null)
        {
            @EmptyTemplate
        }
        else
        {
            <p>데이터가 없습니다.</p>
        }
    }
</div>

@code {
    [Parameter] public IEnumerable<TItem>? Items { get; set; }
    [Parameter] public RenderFragment<TItem> ItemTemplate { get; set; } = default!;
    [Parameter] public RenderFragment? EmptyTemplate { get; set; }
}

// 사용 예제
<DataList Items="products" Context="product">
    <ItemTemplate>
        <div class="product-card">
            <h4>@product.Name</h4>
            <p>가격: @product.Price.ToString("C")</p>
        </div>
    </ItemTemplate>
    <EmptyTemplate>
        <div class="alert alert-info">
            등록된 상품이 없습니다.
        </div>
    </EmptyTemplate>
</DataList>
```

### 2.2 고급 Templated Component

```csharp
// AdvancedTable.razor
@typeparam TItem

<table class="table">
    @if (ShowHeader)
    {
        <thead>
            <tr>
                @HeaderTemplate
            </tr>
        </thead>
    }
    <tbody>
        @if (Items != null && Items.Any())
        {
            @foreach (var (item, index) in Items.Select((item, idx) => (item, idx)))
            {
                <tr class="@(RowClass?.Invoke(item, index))"
                    @onclick="@(() => OnRowClick.InvokeAsync(item))">
                    @RowTemplate((item, index))
                </tr>
            }
        }
        else
        {
            <tr>
                <td colspan="100">
                    @if (NoDataTemplate != null)
                    {
                        @NoDataTemplate
                    }
                    else
                    {
                        <span>데이터가 없습니다.</span>
                    }
                </td>
            </tr>
        }
    </tbody>
    @if (FooterTemplate != null)
    {
        <tfoot>
            <tr>
                @FooterTemplate
            </tr>
        </tfoot>
    }
</table>

@code {
    [Parameter] public IEnumerable<TItem>? Items { get; set; }
    [Parameter] public bool ShowHeader { get; set; } = true;
    [Parameter] public RenderFragment HeaderTemplate { get; set; } = default!;
    [Parameter] public RenderFragment<(TItem Item, int Index)> RowTemplate { get; set; } = default!;
    [Parameter] public RenderFragment? FooterTemplate { get; set; }
    [Parameter] public RenderFragment? NoDataTemplate { get; set; }
    [Parameter] public Func<TItem, int, string>? RowClass { get; set; }
    [Parameter] public EventCallback<TItem> OnRowClick { get; set; }
}

// 사용 예제
<AdvancedTable Items="employees" Context="data">
    <HeaderTemplate>
        <th>번호</th>
        <th>이름</th>
        <th>부서</th>
        <th>급여</th>
    </HeaderTemplate>
    <RowTemplate>
        <td>@(data.Index + 1)</td>
        <td>@data.Item.Name</td>
        <td>@data.Item.Department</td>
        <td>@data.Item.Salary.ToString("C")</td>
    </RowTemplate>
    <FooterTemplate>
        <td colspan="3">총계</td>
        <td>@employees.Sum(e => e.Salary).ToString("C")</td>
    </FooterTemplate>
</AdvancedTable>
```

### 2.3 다중 템플릿 컴포넌트

```csharp
// MultiTemplateList.razor
@typeparam TItem

<div class="multi-template-list">
    @foreach (var item in Items)
    {
        @if (ItemSelector != null)
        {
            var templateName = ItemSelector(item);
            
            @switch (templateName)
            {
                case "primary":
                    @PrimaryTemplate(item)
                    break;
                case "secondary":
                    @SecondaryTemplate(item)
                    break;
                case "special":
                    @if (SpecialTemplate != null)
                    {
                        @SpecialTemplate(item)
                    }
                    else
                    {
                        @DefaultTemplate(item)
                    }
                    break;
                default:
                    @DefaultTemplate(item)
                    break;
            }
        }
        else
        {
            @DefaultTemplate(item)
        }
    }
</div>

@code {
    [Parameter] public IEnumerable<TItem> Items { get; set; } = Enumerable.Empty<TItem>();
    [Parameter] public Func<TItem, string>? ItemSelector { get; set; }
    [Parameter] public RenderFragment<TItem> PrimaryTemplate { get; set; } = default!;
    [Parameter] public RenderFragment<TItem> SecondaryTemplate { get; set; } = default!;
    [Parameter] public RenderFragment<TItem>? SpecialTemplate { get; set; }
    [Parameter] public RenderFragment<TItem> DefaultTemplate { get; set; } = default!;
}
```

## 3. Generic Components

### 3.1 기본 Generic Component

```csharp
// GenericSelector.razor
@typeparam TValue where TValue : notnull

<div class="selector">
    <label>@Label</label>
    <select @bind="SelectedValue" class="form-control">
        <option value="">선택하세요</option>
        @foreach (var item in Items)
        {
            <option value="@GetValue(item)">@GetDisplay(item)</option>
        }
    </select>
</div>

@code {
    [Parameter] public string Label { get; set; } = "";
    [Parameter] public IEnumerable<TValue> Items { get; set; } = Enumerable.Empty<TValue>();
    [Parameter] public Func<TValue, string> GetDisplay { get; set; } = item => item.ToString() ?? "";
    [Parameter] public Func<TValue, object> GetValue { get; set; } = item => item;
    
    private TValue? _selectedValue;
    
    [Parameter]
    public TValue? SelectedValue
    {
        get => _selectedValue;
        set
        {
            if (!EqualityComparer<TValue>.Default.Equals(_selectedValue, value))
            {
                _selectedValue = value;
                SelectedValueChanged.InvokeAsync(value);
            }
        }
    }
    
    [Parameter] public EventCallback<TValue?> SelectedValueChanged { get; set; }
}
```

### 3.2 제약 조건이 있는 Generic Component

```csharp
// EntityEditor.razor
@typeparam TEntity where TEntity : class, IEntity, new()
@implements IDisposable

<EditForm Model="Entity" OnValidSubmit="HandleValidSubmit">
    <DataAnnotationsValidator />
    
    <div class="form-group">
        @foreach (var property in GetEditableProperties())
        {
            <div class="mb-3">
                <label>@property.DisplayName</label>
                @CreateInputForProperty(property)
                <ValidationMessage For="@(() => property.GetValue(Entity))" />
            </div>
        }
    </div>
    
    <div class="form-actions">
        <button type="submit" class="btn btn-primary">저장</button>
        <button type="button" class="btn btn-secondary" @onclick="Cancel">취소</button>
    </div>
</EditForm>

@code {
    [Parameter] public TEntity Entity { get; set; } = new();
    [Parameter] public EventCallback<TEntity> OnSave { get; set; }
    [Parameter] public EventCallback OnCancel { get; set; }
    
    private PropertyInfo[] editableProperties = Array.Empty<PropertyInfo>();
    
    protected override void OnInitialized()
    {
        editableProperties = typeof(TEntity)
            .GetProperties(BindingFlags.Public | BindingFlags.Instance)
            .Where(p => p.CanWrite && p.GetCustomAttribute<EditableAttribute>()?.AllowEdit != false)
            .ToArray();
    }
    
    private IEnumerable<(PropertyInfo Property, string DisplayName)> GetEditableProperties()
    {
        return editableProperties.Select(p => 
        {
            var displayAttr = p.GetCustomAttribute<DisplayAttribute>();
            var displayName = displayAttr?.Name ?? p.Name;
            return (p, displayName);
        });
    }
    
    private RenderFragment CreateInputForProperty(PropertyInfo property)
    {
        return builder =>
        {
            var propertyType = property.PropertyType;
            var value = property.GetValue(Entity);
            
            if (propertyType == typeof(string))
            {
                builder.OpenComponent<InputText>(0);
                builder.AddAttribute(1, "class", "form-control");
                builder.AddAttribute(2, "Value", value);
                builder.AddAttribute(3, "ValueChanged", EventCallback.Factory.Create<string>(
                    this, newValue => property.SetValue(Entity, newValue)));
                builder.CloseComponent();
            }
            else if (propertyType == typeof(int) || propertyType == typeof(int?))
            {
                builder.OpenComponent<InputNumber<int>>(0);
                builder.AddAttribute(1, "class", "form-control");
                builder.AddAttribute(2, "Value", value);
                builder.AddAttribute(3, "ValueChanged", EventCallback.Factory.Create<int>(
                    this, newValue => property.SetValue(Entity, newValue)));
                builder.CloseComponent();
            }
            else if (propertyType == typeof(DateTime) || propertyType == typeof(DateTime?))
            {
                builder.OpenComponent<InputDate<DateTime>>(0);
                builder.AddAttribute(1, "class", "form-control");
                builder.AddAttribute(2, "Value", value);
                builder.AddAttribute(3, "ValueChanged", EventCallback.Factory.Create<DateTime>(
                    this, newValue => property.SetValue(Entity, newValue)));
                builder.CloseComponent();
            }
            else if (propertyType == typeof(bool))
            {
                builder.OpenComponent<InputCheckbox>(0);
                builder.AddAttribute(1, "class", "form-check-input");
                builder.AddAttribute(2, "Value", value);
                builder.AddAttribute(3, "ValueChanged", EventCallback.Factory.Create<bool>(
                    this, newValue => property.SetValue(Entity, newValue)));
                builder.CloseComponent();
            }
        };
    }
    
    private async Task HandleValidSubmit()
    {
        await OnSave.InvokeAsync(Entity);
    }
    
    private async Task Cancel()
    {
        await OnCancel.InvokeAsync();
    }
    
    public void Dispose()
    {
        // 리소스 정리
    }
}

// 엔티티 인터페이스
public interface IEntity
{
    int Id { get; set; }
    DateTime CreatedAt { get; set; }
    DateTime? UpdatedAt { get; set; }
}
```

### 3.3 복잡한 Generic Component

```csharp
// AsyncDataGrid.razor
@typeparam TItem where TItem : class
@typeparam TKey where TKey : IEquatable<TKey>

<div class="async-data-grid">
    @if (IsLoading)
    {
        <div class="loading-indicator">
            <span>데이터를 불러오는 중...</span>
        </div>
    }
    
    <div class="grid-toolbar">
        <input type="text" @bind="searchQuery" @bind:event="oninput" 
               placeholder="검색..." class="form-control" />
        <button @onclick="RefreshData" class="btn btn-sm btn-outline-primary">
            새로고침
        </button>
    </div>
    
    <div class="grid-container">
        @if (pagedItems != null && pagedItems.Any())
        {
            <table class="table">
                <thead>
                    <tr>
                        @foreach (var column in Columns)
                        {
                            <th @onclick="@(() => SortByColumn(column))">
                                @column.Title
                                @if (sortColumn == column)
                                {
                                    <span>@(sortAscending ? "▲" : "▼")</span>
                                }
                            </th>
                        }
                        @if (ShowActions)
                        {
                            <th>액션</th>
                        }
                    </tr>
                </thead>
                <tbody>
                    @foreach (var item in pagedItems)
                    {
                        <tr @key="GetItemKey(item)" 
                            class="@(selectedItems.Contains(item) ? "selected" : "")">
                            @foreach (var column in Columns)
                            {
                                <td>@column.CellTemplate(item)</td>
                            }
                            @if (ShowActions)
                            {
                                <td>
                                    @ActionsTemplate(item)
                                </td>
                            }
                        </tr>
                    }
                </tbody>
            </table>
            
            <Pagination CurrentPage="currentPage" 
                       TotalPages="totalPages" 
                       OnPageChanged="ChangePage" />
        }
        else if (!IsLoading)
        {
            <div class="no-data">
                @NoDataTemplate
            </div>
        }
    </div>
</div>

@code {
    [Parameter] public Func<DataRequest, Task<DataResponse<TItem>>> DataSource { get; set; } = default!;
    [Parameter] public List<ColumnDefinition<TItem>> Columns { get; set; } = new();
    [Parameter] public Func<TItem, TKey> GetItemKey { get; set; } = default!;
    [Parameter] public bool ShowActions { get; set; }
    [Parameter] public RenderFragment<TItem> ActionsTemplate { get; set; } = default!;
    [Parameter] public RenderFragment NoDataTemplate { get; set; } = @<text>데이터가 없습니다.</text>;
    [Parameter] public int PageSize { get; set; } = 10;
    
    private bool IsLoading = false;
    private string searchQuery = "";
    private List<TItem>? allItems;
    private List<TItem>? pagedItems;
    private HashSet<TItem> selectedItems = new();
    private ColumnDefinition<TItem>? sortColumn;
    private bool sortAscending = true;
    private int currentPage = 1;
    private int totalPages = 1;
    
    private CancellationTokenSource? loadCts;
    
    protected override async Task OnInitializedAsync()
    {
        await LoadDataAsync();
    }
    
    private async Task LoadDataAsync()
    {
        loadCts?.Cancel();
        loadCts = new CancellationTokenSource();
        
        IsLoading = true;
        try
        {
            var request = new DataRequest
            {
                SearchQuery = searchQuery,
                SortColumn = sortColumn?.PropertyName,
                SortAscending = sortAscending,
                Page = currentPage,
                PageSize = PageSize
            };
            
            var response = await DataSource(request);
            allItems = response.Items.ToList();
            totalPages = response.TotalPages;
            
            ApplyLocalFiltersAndPaging();
        }
        catch (OperationCanceledException)
        {
            // 취소됨
        }
        finally
        {
            IsLoading = false;
        }
    }
    
    private void ApplyLocalFiltersAndPaging()
    {
        if (allItems == null) return;
        
        var filtered = allItems.AsEnumerable();
        
        // 로컬 검색
        if (!string.IsNullOrWhiteSpace(searchQuery))
        {
            filtered = filtered.Where(item => 
                Columns.Any(col => col.SearchPredicate?.Invoke(item, searchQuery) ?? false));
        }
        
        // 로컬 정렬
        if (sortColumn != null && sortColumn.SortExpression != null)
        {
            filtered = sortAscending 
                ? filtered.OrderBy(sortColumn.SortExpression.Compile())
                : filtered.OrderByDescending(sortColumn.SortExpression.Compile());
        }
        
        // 페이징
        pagedItems = filtered
            .Skip((currentPage - 1) * PageSize)
            .Take(PageSize)
            .ToList();
    }
    
    private async Task SortByColumn(ColumnDefinition<TItem> column)
    {
        if (!column.Sortable) return;
        
        if (sortColumn == column)
        {
            sortAscending = !sortAscending;
        }
        else
        {
            sortColumn = column;
            sortAscending = true;
        }
        
        await LoadDataAsync();
    }
    
    private async Task ChangePage(int page)
    {
        currentPage = page;
        await LoadDataAsync();
    }
    
    private async Task RefreshData()
    {
        currentPage = 1;
        await LoadDataAsync();
    }
    
    protected override async Task OnParametersSetAsync()
    {
        // 검색어 변경 시 디바운싱
        await Task.Delay(300);
        await LoadDataAsync();
    }
    
    public class ColumnDefinition<T>
    {
        public string Title { get; set; } = "";
        public string PropertyName { get; set; } = "";
        public RenderFragment<T> CellTemplate { get; set; } = default!;
        public bool Sortable { get; set; } = true;
        public Expression<Func<T, object>>? SortExpression { get; set; }
        public Func<T, string, bool>? SearchPredicate { get; set; }
    }
    
    public class DataRequest
    {
        public string? SearchQuery { get; set; }
        public string? SortColumn { get; set; }
        public bool SortAscending { get; set; } = true;
        public int Page { get; set; } = 1;
        public int PageSize { get; set; } = 10;
    }
    
    public class DataResponse<T>
    {
        public IEnumerable<T> Items { get; set; } = Enumerable.Empty<T>();
        public int TotalCount { get; set; }
        public int TotalPages { get; set; }
    }
}
```

## 4. 컴포넌트 컴포지션 패턴

### 4.1 Compound Components

```csharp
// Tabs.razor (부모 컴포넌트)
@implements ITabContainer

<CascadingValue Value="this">
    <div class="tabs-container">
        <ul class="nav nav-tabs">
            @foreach (var tab in tabs)
            {
                <li class="nav-item">
                    <button class="nav-link @(tab == activeTab ? "active" : "")"
                            @onclick="@(() => SetActiveTab(tab))">
                        @tab.Title
                    </button>
                </li>
            }
        </ul>
        <div class="tab-content">
            @ChildContent
        </div>
    </div>
</CascadingValue>

@code {
    [Parameter] public RenderFragment ChildContent { get; set; } = default!;
    
    private List<TabPanel> tabs = new();
    private TabPanel? activeTab;
    
    public void AddTab(TabPanel tab)
    {
        tabs.Add(tab);
        if (tabs.Count == 1)
        {
            SetActiveTab(tab);
        }
        StateHasChanged();
    }
    
    public void RemoveTab(TabPanel tab)
    {
        tabs.Remove(tab);
        if (activeTab == tab && tabs.Any())
        {
            SetActiveTab(tabs.First());
        }
        StateHasChanged();
    }
    
    public bool IsTabActive(TabPanel tab) => activeTab == tab;
    
    private void SetActiveTab(TabPanel tab)
    {
        activeTab = tab;
        StateHasChanged();
    }
}

// TabPanel.razor (자식 컴포넌트)
@implements IDisposable

@if (TabContainer?.IsTabActive(this) == true)
{
    <div class="tab-panel">
        @ChildContent
    </div>
}

@code {
    [CascadingParameter] private ITabContainer? TabContainer { get; set; }
    [Parameter] public string Title { get; set; } = "Tab";
    [Parameter] public RenderFragment ChildContent { get; set; } = default!;
    
    protected override void OnInitialized()
    {
        TabContainer?.AddTab(this);
    }
    
    public void Dispose()
    {
        TabContainer?.RemoveTab(this);
    }
}

// ITabContainer.cs
public interface ITabContainer
{
    void AddTab(TabPanel tab);
    void RemoveTab(TabPanel tab);
    bool IsTabActive(TabPanel tab);
}

// 사용 예제
<Tabs>
    <TabPanel Title="개요">
        <h3>제품 개요</h3>
        <p>제품에 대한 설명...</p>
    </TabPanel>
    <TabPanel Title="사양">
        <h3>기술 사양</h3>
        <ul>
            <li>크기: 100x50mm</li>
            <li>무게: 200g</li>
        </ul>
    </TabPanel>
    <TabPanel Title="리뷰">
        <h3>고객 리뷰</h3>
        <ReviewList ProductId="@productId" />
    </TabPanel>
</Tabs>
```

### 4.2 Provider Pattern

```csharp
// ThemeProvider.razor
<CascadingValue Value="themeService" IsFixed="true">
    <div class="theme-@themeService.CurrentTheme">
        @ChildContent
    </div>
</CascadingValue>

@code {
    [Parameter] public RenderFragment ChildContent { get; set; } = default!;
    
    private ThemeService themeService = new();
    
    protected override void OnInitialized()
    {
        themeService.ThemeChanged += OnThemeChanged;
    }
    
    private void OnThemeChanged()
    {
        InvokeAsync(StateHasChanged);
    }
    
    public void Dispose()
    {
        themeService.ThemeChanged -= OnThemeChanged;
    }
}

// ThemedButton.razor
<button class="btn btn-@Theme" @attributes="AdditionalAttributes">
    @ChildContent
</button>

@code {
    [CascadingParameter] private ThemeService? ThemeService { get; set; }
    [Parameter] public RenderFragment? ChildContent { get; set; }
    [Parameter(CaptureUnmatchedValues = true)] 
    public Dictionary<string, object>? AdditionalAttributes { get; set; }
    
    private string Theme => ThemeService?.CurrentTheme ?? "default";
}
```

## 5. 실전 고급 패턴

### 5.1 Form Builder Pattern

```csharp
// FormBuilder.razor
@typeparam TModel where TModel : class, new()

<EditForm Model="Model" OnValidSubmit="HandleSubmit">
    <DataAnnotationsValidator />
    
    @foreach (var field in fields)
    {
        <div class="form-group">
            @field.Render()
        </div>
    }
    
    <div class="form-actions">
        <button type="submit" class="btn btn-primary">제출</button>
    </div>
</EditForm>

@code {
    [Parameter] public TModel Model { get; set; } = new();
    [Parameter] public EventCallback<TModel> OnSubmit { get; set; }
    
    private List<IFormField> fields = new();
    
    public FormBuilder<TModel> AddTextField(Expression<Func<TModel, string>> property, string label)
    {
        fields.Add(new TextField<TModel>(Model, property, label));
        return this;
    }
    
    public FormBuilder<TModel> AddNumberField<TValue>(
        Expression<Func<TModel, TValue>> property, 
        string label) where TValue : struct
    {
        fields.Add(new NumberField<TModel, TValue>(Model, property, label));
        return this;
    }
    
    public FormBuilder<TModel> AddSelectField<TValue>(
        Expression<Func<TModel, TValue>> property,
        string label,
        IEnumerable<SelectOption<TValue>> options)
    {
        fields.Add(new SelectField<TModel, TValue>(Model, property, label, options));
        return this;
    }
    
    private async Task HandleSubmit()
    {
        await OnSubmit.InvokeAsync(Model);
    }
    
    private interface IFormField
    {
        RenderFragment Render();
    }
    
    private class TextField<T> : IFormField where T : class
    {
        private readonly T model;
        private readonly Expression<Func<T, string>> property;
        private readonly string label;
        
        public TextField(T model, Expression<Func<T, string>> property, string label)
        {
            this.model = model;
            this.property = property;
            this.label = label;
        }
        
        public RenderFragment Render() => builder =>
        {
            builder.OpenElement(0, "label");
            builder.AddContent(1, label);
            builder.CloseElement();
            
            builder.OpenComponent<InputText>(2);
            builder.AddAttribute(3, "class", "form-control");
            builder.AddAttribute(4, "Value", property.Compile()(model));
            builder.AddAttribute(5, "ValueExpression", property);
            builder.AddAttribute(6, "ValueChanged", 
                EventCallback.Factory.Create<string>(model, value =>
                {
                    var memberExpression = (MemberExpression)property.Body;
                    var propertyInfo = (PropertyInfo)memberExpression.Member;
                    propertyInfo.SetValue(model, value);
                }));
            builder.CloseComponent();
            
            builder.OpenComponent<ValidationMessage<string>>(7);
            builder.AddAttribute(8, "For", property);
            builder.CloseComponent();
        };
    }
}
```

### 5.2 렌더링 최적화 패턴

```csharp
// VirtualScroll.razor
@typeparam TItem

<div class="virtual-scroll-container" @ref="containerElement" 
     style="height: @(ContainerHeight)px; overflow-y: auto;"
     @onscroll="OnScroll">
    <div style="height: @(totalHeight)px; position: relative;">
        @foreach (var item in visibleItems)
        {
            <div style="position: absolute; top: @(item.Top)px; height: @(ItemHeight)px;">
                @ItemTemplate(item.Item)
            </div>
        }
    </div>
</div>

@code {
    [Parameter] public IList<TItem> Items { get; set; } = new List<TItem>();
    [Parameter] public RenderFragment<TItem> ItemTemplate { get; set; } = default!;
    [Parameter] public int ItemHeight { get; set; } = 50;
    [Parameter] public int ContainerHeight { get; set; } = 400;
    [Parameter] public int OverscanCount { get; set; } = 3;
    
    private ElementReference containerElement;
    private List<(TItem Item, int Index, int Top)> visibleItems = new();
    private int totalHeight;
    private double scrollTop;
    private int startIndex;
    private int endIndex;
    
    protected override void OnParametersSet()
    {
        totalHeight = Items.Count * ItemHeight;
        CalculateVisibleItems();
    }
    
    private async Task OnScroll()
    {
        scrollTop = await JSRuntime.InvokeAsync<double>(
            "getScrollTop", containerElement);
        CalculateVisibleItems();
    }
    
    private void CalculateVisibleItems()
    {
        var viewportHeight = ContainerHeight;
        startIndex = Math.Max(0, (int)(scrollTop / ItemHeight) - OverscanCount);
        endIndex = Math.Min(Items.Count - 1, 
            (int)((scrollTop + viewportHeight) / ItemHeight) + OverscanCount);
        
        visibleItems.Clear();
        for (int i = startIndex; i <= endIndex; i++)
        {
            visibleItems.Add((Items[i], i, i * ItemHeight));
        }
    }
    
    [Inject] private IJSRuntime JSRuntime { get; set; } = default!;
}
```

## 마무리

Blazor의 고급 컴포넌트 패턴을 활용하면 재사용 가능하고 유지보수가 쉬운 컴포넌트를 만들 수 있습니다. RenderFragment를 통한 유연한 콘텐츠 렌더링, Templated Components로 데이터 표현의 자유도를 높이고, Generic Components로 타입 안전성을 확보할 수 있습니다.

이러한 패턴들을 적절히 조합하여 사용하면, 복잡한 UI 요구사항도 깔끔하고 효율적으로 구현할 수 있습니다. 특히 대규모 애플리케이션에서는 이러한 패턴들이 코드의 재사용성과 유지보수성을 크게 향상시킵니다.