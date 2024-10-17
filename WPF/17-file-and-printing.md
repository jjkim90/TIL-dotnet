# 파일과 인쇄

## 파일 대화상자

WPF에서 파일을 선택하고 저장하기 위한 대화상자를 제공합니다.

### OpenFileDialog
```csharp
using Microsoft.Win32;

public class FileService
{
    public string OpenFile()
    {
        var dialog = new OpenFileDialog
        {
            Title = "파일 열기",
            Filter = "텍스트 파일 (*.txt)|*.txt|모든 파일 (*.*)|*.*",
            DefaultExt = ".txt",
            InitialDirectory = Environment.GetFolderPath(Environment.SpecialFolder.MyDocuments)
        };
        
        if (dialog.ShowDialog() == true)
        {
            return dialog.FileName;
        }
        
        return null;
    }
    
    public string[] OpenMultipleFiles()
    {
        var dialog = new OpenFileDialog
        {
            Title = "여러 파일 선택",
            Multiselect = true,
            Filter = "이미지 파일|*.jpg;*.jpeg;*.png;*.gif;*.bmp|모든 파일|*.*"
        };
        
        if (dialog.ShowDialog() == true)
        {
            return dialog.FileNames;
        }
        
        return Array.Empty<string>();
    }
}
```

### SaveFileDialog
```csharp
public class SaveService
{
    public string SaveFile(string defaultFileName = "")
    {
        var dialog = new SaveFileDialog
        {
            Title = "파일 저장",
            FileName = defaultFileName,
            DefaultExt = ".txt",
            Filter = "텍스트 파일 (*.txt)|*.txt|CSV 파일 (*.csv)|*.csv|모든 파일 (*.*)|*.*",
            OverwritePrompt = true,
            AddExtension = true
        };
        
        if (dialog.ShowDialog() == true)
        {
            return dialog.FileName;
        }
        
        return null;
    }
    
    // 특정 형식으로 저장
    public async Task<bool> SaveDocumentAsync(FlowDocument document)
    {
        var dialog = new SaveFileDialog
        {
            Filter = "Rich Text Format (*.rtf)|*.rtf|" +
                    "XAML Document (*.xaml)|*.xaml|" +
                    "Plain Text (*.txt)|*.txt",
            DefaultExt = ".rtf"
        };
        
        if (dialog.ShowDialog() == true)
        {
            try
            {
                using (var stream = new FileStream(dialog.FileName, FileMode.Create))
                {
                    var range = new TextRange(document.ContentStart, document.ContentEnd);
                    
                    string dataFormat = Path.GetExtension(dialog.FileName).ToLower() switch
                    {
                        ".rtf" => DataFormats.Rtf,
                        ".xaml" => DataFormats.Xaml,
                        _ => DataFormats.Text
                    };
                    
                    range.Save(stream, dataFormat);
                }
                
                return true;
            }
            catch (Exception ex)
            {
                MessageBox.Show($"저장 실패: {ex.Message}", "오류", 
                               MessageBoxButton.OK, MessageBoxImage.Error);
                return false;
            }
        }
        
        return false;
    }
}
```

## 파일 처리

### 텍스트 파일 읽기/쓰기
```csharp
public class TextFileHandler
{
    public async Task<string> ReadTextFileAsync(string filePath)
    {
        try
        {
            using (var reader = new StreamReader(filePath, Encoding.UTF8))
            {
                return await reader.ReadToEndAsync();
            }
        }
        catch (Exception ex)
        {
            throw new IOException($"파일 읽기 실패: {filePath}", ex);
        }
    }
    
    public async Task WriteTextFileAsync(string filePath, string content)
    {
        try
        {
            using (var writer = new StreamWriter(filePath, false, Encoding.UTF8))
            {
                await writer.WriteAsync(content);
            }
        }
        catch (Exception ex)
        {
            throw new IOException($"파일 쓰기 실패: {filePath}", ex);
        }
    }
    
    // 대용량 파일 처리
    public async IAsyncEnumerable<string> ReadLinesAsync(string filePath)
    {
        using var reader = new StreamReader(filePath);
        string line;
        
        while ((line = await reader.ReadLineAsync()) != null)
        {
            yield return line;
        }
    }
}
```

### 바이너리 파일 처리
```csharp
public class BinaryFileHandler
{
    public async Task<byte[]> ReadBinaryFileAsync(string filePath)
    {
        return await File.ReadAllBytesAsync(filePath);
    }
    
    public async Task WriteBinaryFileAsync(string filePath, byte[] data)
    {
        await File.WriteAllBytesAsync(filePath, data);
    }
    
    // 스트림 기반 처리
    public async Task ProcessLargeFileAsync(string inputPath, string outputPath)
    {
        const int bufferSize = 4096;
        var buffer = new byte[bufferSize];
        
        using (var input = new FileStream(inputPath, FileMode.Open, FileAccess.Read))
        using (var output = new FileStream(outputPath, FileMode.Create, FileAccess.Write))
        {
            int bytesRead;
            while ((bytesRead = await input.ReadAsync(buffer, 0, bufferSize)) > 0)
            {
                // 데이터 처리
                ProcessBuffer(buffer, bytesRead);
                
                // 출력
                await output.WriteAsync(buffer, 0, bytesRead);
            }
        }
    }
    
    private void ProcessBuffer(byte[] buffer, int count)
    {
        // 바이트 데이터 처리 로직
    }
}
```

### 파일 감시 (FileSystemWatcher)
```csharp
public class FileWatcherService : IDisposable
{
    private readonly FileSystemWatcher _watcher;
    public event EventHandler<FileSystemEventArgs> FileChanged;
    public event EventHandler<RenamedEventArgs> FileRenamed;
    
    public FileWatcherService(string path, string filter = "*.*")
    {
        _watcher = new FileSystemWatcher
        {
            Path = path,
            Filter = filter,
            NotifyFilter = NotifyFilters.LastWrite | 
                          NotifyFilters.FileName | 
                          NotifyFilters.DirectoryName | 
                          NotifyFilters.Size
        };
        
        _watcher.Changed += OnChanged;
        _watcher.Created += OnChanged;
        _watcher.Deleted += OnChanged;
        _watcher.Renamed += OnRenamed;
    }
    
    public void Start()
    {
        _watcher.EnableRaisingEvents = true;
    }
    
    public void Stop()
    {
        _watcher.EnableRaisingEvents = false;
    }
    
    private void OnChanged(object sender, FileSystemEventArgs e)
    {
        // UI 스레드에서 이벤트 발생
        Application.Current.Dispatcher.Invoke(() =>
        {
            FileChanged?.Invoke(this, e);
        });
    }
    
    private void OnRenamed(object sender, RenamedEventArgs e)
    {
        Application.Current.Dispatcher.Invoke(() =>
        {
            FileRenamed?.Invoke(this, e);
        });
    }
    
    public void Dispose()
    {
        _watcher?.Dispose();
    }
}
```

## 문서 처리

### FlowDocument 작업
```csharp
public class DocumentHandler
{
    public FlowDocument CreateDocument(string title, string content)
    {
        var document = new FlowDocument();
        
        // 제목 추가
        var titleParagraph = new Paragraph(new Run(title))
        {
            FontSize = 24,
            FontWeight = FontWeights.Bold,
            Margin = new Thickness(0, 0, 0, 10)
        };
        document.Blocks.Add(titleParagraph);
        
        // 내용 추가
        var contentParagraph = new Paragraph(new Run(content))
        {
            FontSize = 12,
            LineHeight = 20
        };
        document.Blocks.Add(contentParagraph);
        
        // 리스트 추가
        var list = new List
        {
            MarkerStyle = TextMarkerStyle.Decimal
        };
        
        list.ListItems.Add(new ListItem(new Paragraph(new Run("첫 번째 항목"))));
        list.ListItems.Add(new ListItem(new Paragraph(new Run("두 번째 항목"))));
        list.ListItems.Add(new ListItem(new Paragraph(new Run("세 번째 항목"))));
        
        document.Blocks.Add(list);
        
        // 테이블 추가
        var table = CreateSampleTable();
        document.Blocks.Add(table);
        
        return document;
    }
    
    private Table CreateSampleTable()
    {
        var table = new Table();
        
        // 컬럼 정의
        table.Columns.Add(new TableColumn { Width = new GridLength(100) });
        table.Columns.Add(new TableColumn { Width = new GridLength(200) });
        table.Columns.Add(new TableColumn { Width = new GridLength(100) });
        
        // 헤더 행
        var headerGroup = new TableRowGroup();
        var headerRow = new TableRow();
        
        headerRow.Cells.Add(new TableCell(new Paragraph(new Run("번호"))) 
        { 
            Background = Brushes.LightGray,
            FontWeight = FontWeights.Bold
        });
        headerRow.Cells.Add(new TableCell(new Paragraph(new Run("이름"))) 
        { 
            Background = Brushes.LightGray,
            FontWeight = FontWeights.Bold
        });
        headerRow.Cells.Add(new TableCell(new Paragraph(new Run("점수"))) 
        { 
            Background = Brushes.LightGray,
            FontWeight = FontWeights.Bold
        });
        
        headerGroup.Rows.Add(headerRow);
        table.RowGroups.Add(headerGroup);
        
        // 데이터 행
        var dataGroup = new TableRowGroup();
        for (int i = 1; i <= 3; i++)
        {
            var row = new TableRow();
            row.Cells.Add(new TableCell(new Paragraph(new Run(i.ToString()))));
            row.Cells.Add(new TableCell(new Paragraph(new Run($"학생 {i}"))));
            row.Cells.Add(new TableCell(new Paragraph(new Run($"{80 + i * 5}"))));
            dataGroup.Rows.Add(row);
        }
        
        table.RowGroups.Add(dataGroup);
        
        return table;
    }
}
```

### RichTextBox 파일 처리
```csharp
public class RichTextBoxFileHandler
{
    private readonly RichTextBox _richTextBox;
    
    public RichTextBoxFileHandler(RichTextBox richTextBox)
    {
        _richTextBox = richTextBox;
    }
    
    public void LoadFile(string filePath)
    {
        var range = new TextRange(_richTextBox.Document.ContentStart, 
                                 _richTextBox.Document.ContentEnd);
        
        using (var stream = new FileStream(filePath, FileMode.Open))
        {
            string dataFormat = Path.GetExtension(filePath).ToLower() switch
            {
                ".rtf" => DataFormats.Rtf,
                ".xaml" => DataFormats.Xaml,
                ".txt" => DataFormats.Text,
                _ => DataFormats.Text
            };
            
            range.Load(stream, dataFormat);
        }
    }
    
    public void SaveFile(string filePath, string format = DataFormats.Rtf)
    {
        var range = new TextRange(_richTextBox.Document.ContentStart, 
                                 _richTextBox.Document.ContentEnd);
        
        using (var stream = new FileStream(filePath, FileMode.Create))
        {
            range.Save(stream, format);
        }
    }
    
    // 이미지 포함된 문서 처리
    public void InsertImage(string imagePath)
    {
        var image = new Image
        {
            Source = new BitmapImage(new Uri(imagePath)),
            Width = 300,
            Height = 200
        };
        
        var container = new InlineUIContainer(image);
        var paragraph = new Paragraph(container);
        
        _richTextBox.Document.Blocks.Add(paragraph);
    }
}
```

## 인쇄 기본

### PrintDialog 사용
```csharp
public class PrintService
{
    public void PrintDocument(FlowDocument document)
    {
        var printDialog = new PrintDialog();
        
        if (printDialog.ShowDialog() == true)
        {
            // 문서 크기 설정
            document.PageHeight = printDialog.PrintableAreaHeight;
            document.PageWidth = printDialog.PrintableAreaWidth;
            document.PagePadding = new Thickness(50);
            document.ColumnGap = 0;
            document.ColumnWidth = printDialog.PrintableAreaWidth;
            
            // 인쇄
            IDocumentPaginatorSource paginator = document;
            printDialog.PrintDocument(paginator.DocumentPaginator, "문서 인쇄");
        }
    }
    
    public void PrintVisual(Visual visual, string description)
    {
        var printDialog = new PrintDialog();
        
        if (printDialog.ShowDialog() == true)
        {
            printDialog.PrintVisual(visual, description);
        }
    }
}
```

### 고급 인쇄 설정
```csharp
public class AdvancedPrintService
{
    public void PrintWithSettings(FlowDocument document)
    {
        var printDialog = new PrintDialog();
        
        // 인쇄 티켓 설정
        var printTicket = printDialog.PrintTicket;
        printTicket.PageOrientation = PageOrientation.Portrait;
        printTicket.PageMediaSize = new PageMediaSize(PageMediaSizeName.ISOA4);
        printTicket.CopyCount = 1;
        
        if (printDialog.ShowDialog() == true)
        {
            // 인쇄 큐 가져오기
            var printQueue = printDialog.PrintQueue;
            
            // XPS 문서 생성
            var xpsDocument = CreateXpsDocument(document);
            var xpsWriter = PrintQueue.CreateXpsDocumentWriter(printQueue);
            
            // 비동기 인쇄
            xpsWriter.WriteAsync(xpsDocument.GetFixedDocumentSequence());
        }
    }
    
    private XpsDocument CreateXpsDocument(FlowDocument flowDocument)
    {
        var tempFile = Path.GetTempFileName();
        
        using (var xpsDocument = new XpsDocument(tempFile, FileAccess.ReadWrite))
        {
            var writer = XpsDocument.CreateXpsDocumentWriter(xpsDocument);
            writer.Write(((IDocumentPaginatorSource)flowDocument).DocumentPaginator);
            return xpsDocument;
        }
    }
}
```

## 사용자 정의 인쇄

### DocumentPaginator 구현
```csharp
public class CustomDocumentPaginator : DocumentPaginator
{
    private readonly List<object> _items;
    private readonly Size _pageSize;
    private readonly Thickness _margin;
    private List<DocumentPage> _pages;
    
    public CustomDocumentPaginator(List<object> items, Size pageSize, Thickness margin)
    {
        _items = items;
        _pageSize = pageSize;
        _margin = margin;
        PaginateData();
    }
    
    private void PaginateData()
    {
        _pages = new List<DocumentPage>();
        
        const int itemsPerPage = 20;
        int pageCount = (int)Math.Ceiling(_items.Count / (double)itemsPerPage);
        
        for (int pageIndex = 0; pageIndex < pageCount; pageIndex++)
        {
            var page = CreatePage(pageIndex, itemsPerPage);
            _pages.Add(page);
        }
    }
    
    private DocumentPage CreatePage(int pageIndex, int itemsPerPage)
    {
        var visual = new DrawingVisual();
        
        using (var dc = visual.RenderOpen())
        {
            // 헤더 그리기
            DrawHeader(dc, pageIndex + 1);
            
            // 내용 그리기
            double y = _margin.Top + 50;
            int startIndex = pageIndex * itemsPerPage;
            int endIndex = Math.Min(startIndex + itemsPerPage, _items.Count);
            
            for (int i = startIndex; i < endIndex; i++)
            {
                DrawItem(dc, _items[i], y);
                y += 25;
            }
            
            // 푸터 그리기
            DrawFooter(dc, pageIndex + 1, PageCount);
        }
        
        return new DocumentPage(visual, _pageSize, 
            new Rect(new Point(), _pageSize), 
            new Rect(new Point(_margin.Left, _margin.Top), 
            new Size(_pageSize.Width - _margin.Left - _margin.Right,
                    _pageSize.Height - _margin.Top - _margin.Bottom)));
    }
    
    private void DrawHeader(DrawingContext dc, int pageNumber)
    {
        var headerText = new FormattedText(
            $"보고서 - {DateTime.Now:yyyy-MM-dd}",
            CultureInfo.CurrentCulture,
            FlowDirection.LeftToRight,
            new Typeface("Arial"),
            16,
            Brushes.Black,
            VisualTreeHelper.GetDpi(new Visual()).PixelsPerDip);
        
        dc.DrawText(headerText, new Point(_margin.Left, _margin.Top));
        
        // 구분선
        dc.DrawLine(new Pen(Brushes.Black, 1), 
            new Point(_margin.Left, _margin.Top + 30),
            new Point(_pageSize.Width - _margin.Right, _margin.Top + 30));
    }
    
    private void DrawItem(DrawingContext dc, object item, double y)
    {
        var text = new FormattedText(
            item.ToString(),
            CultureInfo.CurrentCulture,
            FlowDirection.LeftToRight,
            new Typeface("Arial"),
            12,
            Brushes.Black,
            VisualTreeHelper.GetDpi(new Visual()).PixelsPerDip);
        
        dc.DrawText(text, new Point(_margin.Left, y));
    }
    
    private void DrawFooter(DrawingContext dc, int currentPage, int totalPages)
    {
        var footerText = new FormattedText(
            $"페이지 {currentPage} / {totalPages}",
            CultureInfo.CurrentCulture,
            FlowDirection.LeftToRight,
            new Typeface("Arial"),
            10,
            Brushes.Gray,
            VisualTreeHelper.GetDpi(new Visual()).PixelsPerDip);
        
        var x = (_pageSize.Width - footerText.Width) / 2;
        var y = _pageSize.Height - _margin.Bottom - 20;
        
        dc.DrawText(footerText, new Point(x, y));
    }
    
    public override DocumentPage GetPage(int pageNumber)
    {
        return _pages[pageNumber];
    }
    
    public override bool IsPageCountValid => true;
    public override int PageCount => _pages.Count;
    public override Size PageSize { get => _pageSize; set { } }
    public override IDocumentPaginatorSource Source => null;
}
```

### 인쇄 미리보기
```csharp
public partial class PrintPreviewWindow : Window
{
    private readonly DocumentPaginator _paginator;
    
    public PrintPreviewWindow(DocumentPaginator paginator)
    {
        InitializeComponent();
        _paginator = paginator;
        
        // DocumentViewer에 설정
        var fixedDocument = new FixedDocument();
        
        for (int i = 0; i < _paginator.PageCount; i++)
        {
            var page = _paginator.GetPage(i);
            var fixedPage = CreateFixedPage(page);
            
            var pageContent = new PageContent();
            ((IAddChild)pageContent).AddChild(fixedPage);
            fixedDocument.Pages.Add(pageContent);
        }
        
        documentViewer.Document = fixedDocument;
    }
    
    private FixedPage CreateFixedPage(DocumentPage documentPage)
    {
        var fixedPage = new FixedPage
        {
            Width = documentPage.Size.Width,
            Height = documentPage.Size.Height
        };
        
        // Visual을 UIElement로 변환
        var container = new Border
        {
            Child = new VisualHost { Visual = documentPage.Visual }
        };
        
        fixedPage.Children.Add(container);
        
        return fixedPage;
    }
    
    private void PrintButton_Click(object sender, RoutedEventArgs e)
    {
        var printDialog = new PrintDialog();
        if (printDialog.ShowDialog() == true)
        {
            printDialog.PrintDocument(_paginator, "미리보기 인쇄");
        }
    }
}

// Visual 호스팅을 위한 헬퍼 클래스
public class VisualHost : FrameworkElement
{
    private Visual _visual;
    
    public Visual Visual
    {
        get => _visual;
        set
        {
            _visual = value;
            AddVisualChild(_visual);
            AddLogicalChild(_visual);
        }
    }
    
    protected override int VisualChildrenCount => _visual != null ? 1 : 0;
    
    protected override Visual GetVisualChild(int index)
    {
        if (index != 0)
            throw new ArgumentOutOfRangeException();
        return _visual;
    }
}
```

## XPS 문서 처리

### XPS 문서 생성
```csharp
public class XpsDocumentService
{
    public void CreateXpsDocument(string filePath, FlowDocument flowDocument)
    {
        using (var package = Package.Open(filePath, FileMode.Create))
        {
            using (var xpsDocument = new XpsDocument(package))
            {
                var writer = XpsDocument.CreateXpsDocumentWriter(xpsDocument);
                writer.Write(((IDocumentPaginatorSource)flowDocument).DocumentPaginator);
            }
        }
    }
    
    public void CreateXpsFromVisuals(string filePath, List<Visual> visuals)
    {
        using (var package = Package.Open(filePath, FileMode.Create))
        {
            using (var xpsDocument = new XpsDocument(package))
            {
                var writer = XpsDocument.CreateXpsDocumentWriter(xpsDocument);
                var fixedDocument = new FixedDocument();
                
                foreach (var visual in visuals)
                {
                    var fixedPage = new FixedPage
                    {
                        Width = 816,  // A4 width in pixels at 96 DPI
                        Height = 1056 // A4 height in pixels at 96 DPI
                    };
                    
                    // Visual을 캔버스에 추가
                    var canvas = new Canvas();
                    canvas.Children.Add(new VisualHost { Visual = visual });
                    fixedPage.Children.Add(canvas);
                    
                    var pageContent = new PageContent();
                    ((IAddChild)pageContent).AddChild(fixedPage);
                    fixedDocument.Pages.Add(pageContent);
                }
                
                writer.Write(fixedDocument);
            }
        }
    }
}
```

### XPS 문서 읽기
```csharp
public class XpsReader
{
    public FixedDocumentSequence ReadXpsDocument(string filePath)
    {
        using (var xpsDocument = new XpsDocument(filePath, FileAccess.Read))
        {
            return xpsDocument.GetFixedDocumentSequence();
        }
    }
    
    public void DisplayXpsDocument(string filePath, DocumentViewer viewer)
    {
        try
        {
            var xpsDocument = new XpsDocument(filePath, FileAccess.Read);
            var sequence = xpsDocument.GetFixedDocumentSequence();
            viewer.Document = sequence;
            
            // XPS 문서는 나중에 닫아야 함
            viewer.Tag = xpsDocument;
        }
        catch (Exception ex)
        {
            MessageBox.Show($"XPS 문서 열기 실패: {ex.Message}");
        }
    }
}
```

## PDF 생성 (iTextSharp 사용)

### 기본 PDF 생성
```csharp
using iTextSharp.text;
using iTextSharp.text.pdf;

public class PdfGenerator
{
    public void CreateSimplePdf(string filePath, string content)
    {
        using (var document = new Document(PageSize.A4))
        {
            using (var writer = PdfWriter.GetInstance(
                document, new FileStream(filePath, FileMode.Create)))
            {
                document.Open();
                
                // 한글 폰트 설정
                var baseFont = BaseFont.CreateFont(
                    @"C:\Windows\Fonts\malgun.ttf", 
                    BaseFont.IDENTITY_H, 
                    BaseFont.EMBEDDED);
                
                var font = new Font(baseFont, 12);
                var titleFont = new Font(baseFont, 18, Font.BOLD);
                
                // 제목 추가
                var title = new Paragraph("PDF 문서 제목", titleFont);
                title.Alignment = Element.ALIGN_CENTER;
                document.Add(title);
                
                document.Add(new Paragraph("\n"));
                
                // 내용 추가
                document.Add(new Paragraph(content, font));
                
                // 테이블 추가
                var table = CreateSampleTable(font);
                document.Add(table);
                
                document.Close();
            }
        }
    }
    
    private PdfPTable CreateSampleTable(Font font)
    {
        var table = new PdfPTable(3);
        table.WidthPercentage = 100;
        table.SetWidths(new float[] { 1f, 2f, 1f });
        
        // 헤더
        table.AddCell(new PdfPCell(new Phrase("번호", font)) 
        { 
            BackgroundColor = BaseColor.LIGHT_GRAY 
        });
        table.AddCell(new PdfPCell(new Phrase("이름", font)) 
        { 
            BackgroundColor = BaseColor.LIGHT_GRAY 
        });
        table.AddCell(new PdfPCell(new Phrase("점수", font)) 
        { 
            BackgroundColor = BaseColor.LIGHT_GRAY 
        });
        
        // 데이터
        for (int i = 1; i <= 5; i++)
        {
            table.AddCell(new Phrase(i.ToString(), font));
            table.AddCell(new Phrase($"학생 {i}", font));
            table.AddCell(new Phrase((80 + i * 2).ToString(), font));
        }
        
        return table;
    }
}
```

### WPF Visual을 PDF로 변환
```csharp
public class WpfToPdfConverter
{
    public void ConvertVisualToPdf(Visual visual, string pdfPath)
    {
        // Visual을 비트맵으로 렌더링
        var renderBitmap = new RenderTargetBitmap(
            (int)((FrameworkElement)visual).ActualWidth,
            (int)((FrameworkElement)visual).ActualHeight,
            96, 96, PixelFormats.Pbgra32);
        
        renderBitmap.Render(visual);
        
        // 비트맵을 PNG로 인코딩
        var encoder = new PngBitmapEncoder();
        encoder.Frames.Add(BitmapFrame.Create(renderBitmap));
        
        using (var stream = new MemoryStream())
        {
            encoder.Save(stream);
            
            // PDF 생성
            using (var document = new Document())
            {
                using (var writer = PdfWriter.GetInstance(
                    document, new FileStream(pdfPath, FileMode.Create)))
                {
                    document.Open();
                    
                    var image = Image.GetInstance(stream.ToArray());
                    image.ScaleToFit(document.PageSize.Width - 50, 
                                     document.PageSize.Height - 50);
                    image.Alignment = Element.ALIGN_CENTER;
                    
                    document.Add(image);
                    document.Close();
                }
            }
        }
    }
}
```

## 파일 드래그 앤 드롭

### 드래그 앤 드롭 처리
```csharp
public partial class FileDropWindow : Window
{
    public FileDropWindow()
    {
        InitializeComponent();
        
        // 드롭 활성화
        AllowDrop = true;
        
        // 이벤트 등록
        PreviewDragOver += OnPreviewDragOver;
        Drop += OnDrop;
    }
    
    private void OnPreviewDragOver(object sender, DragEventArgs e)
    {
        e.Handled = true;
        
        // 파일 드롭 가능 여부 확인
        if (e.Data.GetDataPresent(DataFormats.FileDrop))
        {
            e.Effects = DragDropEffects.Copy;
        }
        else
        {
            e.Effects = DragDropEffects.None;
        }
    }
    
    private void OnDrop(object sender, DragEventArgs e)
    {
        if (e.Data.GetDataPresent(DataFormats.FileDrop))
        {
            var files = (string[])e.Data.GetData(DataFormats.FileDrop);
            ProcessDroppedFiles(files);
        }
    }
    
    private void ProcessDroppedFiles(string[] files)
    {
        foreach (var file in files)
        {
            if (File.Exists(file))
            {
                // 파일 처리
                var fileInfo = new FileInfo(file);
                AddFileToList(fileInfo);
            }
            else if (Directory.Exists(file))
            {
                // 디렉토리 처리
                ProcessDirectory(file);
            }
        }
    }
    
    private void ProcessDirectory(string directoryPath)
    {
        var files = Directory.GetFiles(directoryPath, "*.*", 
                                      SearchOption.AllDirectories);
        
        foreach (var file in files)
        {
            AddFileToList(new FileInfo(file));
        }
    }
    
    private void AddFileToList(FileInfo fileInfo)
    {
        // UI 업데이트
        Dispatcher.Invoke(() =>
        {
            fileListBox.Items.Add(new
            {
                Name = fileInfo.Name,
                Size = FormatFileSize(fileInfo.Length),
                Type = fileInfo.Extension,
                Path = fileInfo.FullName
            });
        });
    }
    
    private string FormatFileSize(long bytes)
    {
        string[] sizes = { "B", "KB", "MB", "GB", "TB" };
        double len = bytes;
        int order = 0;
        
        while (len >= 1024 && order < sizes.Length - 1)
        {
            order++;
            len = len / 1024;
        }
        
        return $"{len:0.##} {sizes[order]}";
    }
}
```

## 실전 예제: 문서 편집기

### 문서 편집기 구현
```csharp
public partial class DocumentEditor : Window
{
    private string _currentFilePath;
    private bool _isModified;
    
    public DocumentEditor()
    {
        InitializeComponent();
        InitializeCommands();
    }
    
    private void InitializeCommands()
    {
        // 명령 바인딩
        CommandBindings.Add(new CommandBinding(ApplicationCommands.New, NewDocument));
        CommandBindings.Add(new CommandBinding(ApplicationCommands.Open, OpenDocument));
        CommandBindings.Add(new CommandBinding(ApplicationCommands.Save, SaveDocument));
        CommandBindings.Add(new CommandBinding(ApplicationCommands.SaveAs, SaveAsDocument));
        CommandBindings.Add(new CommandBinding(ApplicationCommands.Print, PrintDocument));
        
        // 텍스트 변경 감지
        richTextBox.TextChanged += (s, e) => _isModified = true;
    }
    
    private void NewDocument(object sender, ExecutedRoutedEventArgs e)
    {
        if (CheckSaveChanges())
        {
            richTextBox.Document.Blocks.Clear();
            _currentFilePath = null;
            _isModified = false;
            UpdateTitle();
        }
    }
    
    private void OpenDocument(object sender, ExecutedRoutedEventArgs e)
    {
        if (!CheckSaveChanges()) return;
        
        var dialog = new OpenFileDialog
        {
            Filter = "Rich Text Format (*.rtf)|*.rtf|" +
                    "Text Files (*.txt)|*.txt|" +
                    "All Files (*.*)|*.*"
        };
        
        if (dialog.ShowDialog() == true)
        {
            try
            {
                var handler = new RichTextBoxFileHandler(richTextBox);
                handler.LoadFile(dialog.FileName);
                
                _currentFilePath = dialog.FileName;
                _isModified = false;
                UpdateTitle();
            }
            catch (Exception ex)
            {
                MessageBox.Show($"파일 열기 실패: {ex.Message}", "오류", 
                               MessageBoxButton.OK, MessageBoxImage.Error);
            }
        }
    }
    
    private void SaveDocument(object sender, ExecutedRoutedEventArgs e)
    {
        if (string.IsNullOrEmpty(_currentFilePath))
        {
            SaveAsDocument(sender, e);
        }
        else
        {
            SaveToFile(_currentFilePath);
        }
    }
    
    private void SaveAsDocument(object sender, ExecutedRoutedEventArgs e)
    {
        var dialog = new SaveFileDialog
        {
            Filter = "Rich Text Format (*.rtf)|*.rtf|" +
                    "Text Files (*.txt)|*.txt",
            DefaultExt = ".rtf"
        };
        
        if (dialog.ShowDialog() == true)
        {
            SaveToFile(dialog.FileName);
            _currentFilePath = dialog.FileName;
            UpdateTitle();
        }
    }
    
    private void SaveToFile(string filePath)
    {
        try
        {
            var handler = new RichTextBoxFileHandler(richTextBox);
            var format = Path.GetExtension(filePath).ToLower() == ".rtf" 
                ? DataFormats.Rtf : DataFormats.Text;
            
            handler.SaveFile(filePath, format);
            _isModified = false;
        }
        catch (Exception ex)
        {
            MessageBox.Show($"파일 저장 실패: {ex.Message}", "오류", 
                           MessageBoxButton.OK, MessageBoxImage.Error);
        }
    }
    
    private void PrintDocument(object sender, ExecutedRoutedEventArgs e)
    {
        var printService = new PrintService();
        printService.PrintDocument(richTextBox.Document);
    }
    
    private bool CheckSaveChanges()
    {
        if (_isModified)
        {
            var result = MessageBox.Show(
                "변경사항을 저장하시겠습니까?", 
                "저장 확인", 
                MessageBoxButton.YesNoCancel,
                MessageBoxImage.Question);
            
            switch (result)
            {
                case MessageBoxResult.Yes:
                    SaveDocument(null, null);
                    return !_isModified;
                case MessageBoxResult.No:
                    return true;
                case MessageBoxResult.Cancel:
                    return false;
            }
        }
        
        return true;
    }
    
    private void UpdateTitle()
    {
        var fileName = string.IsNullOrEmpty(_currentFilePath) 
            ? "제목 없음" : Path.GetFileName(_currentFilePath);
        
        Title = $"문서 편집기 - {fileName}{(_isModified ? "*" : "")}";
    }
}
```

## 핵심 개념 정리
- **파일 대화상자**: OpenFileDialog, SaveFileDialog
- **파일 처리**: 텍스트/바이너리 파일 읽기/쓰기
- **FileSystemWatcher**: 파일 시스템 변경 감시
- **FlowDocument**: 유동적 문서 형식
- **인쇄**: PrintDialog, DocumentPaginator
- **사용자 정의 인쇄**: 커스텀 DocumentPaginator 구현
- **XPS 문서**: XPS 형식 생성 및 읽기
- **PDF 생성**: iTextSharp를 통한 PDF 생성
- **드래그 앤 드롭**: 파일 드롭 처리
- **문서 편집기**: RichTextBox 기반 편집기 구현