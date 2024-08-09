# 파일 다루기

## 파일 정보와 디렉터리 정보

### FileInfo와 DirectoryInfo 클래스
```csharp
// FileInfo - 파일 정보
FileInfo fileInfo = new FileInfo(@"C:\temp\test.txt");
Console.WriteLine($"Name: {fileInfo.Name}");
Console.WriteLine($"Full Path: {fileInfo.FullName}");
Console.WriteLine($"Extension: {fileInfo.Extension}");
Console.WriteLine($"Size: {fileInfo.Length} bytes");
Console.WriteLine($"Created: {fileInfo.CreationTime}");
Console.WriteLine($"Modified: {fileInfo.LastWriteTime}");
Console.WriteLine($"Attributes: {fileInfo.Attributes}");

// DirectoryInfo - 디렉터리 정보
DirectoryInfo dirInfo = new DirectoryInfo(@"C:\temp");
Console.WriteLine($"Name: {dirInfo.Name}");
Console.WriteLine($"Parent: {dirInfo.Parent?.Name}");
Console.WriteLine($"Root: {dirInfo.Root}");
Console.WriteLine($"Exists: {dirInfo.Exists}");
```

### File과 Directory 정적 클래스
```csharp
// File 클래스 - 파일 작업
if (File.Exists(@"C:\temp\test.txt"))
{
    File.Copy(@"C:\temp\test.txt", @"C:\temp\backup.txt", true);
    File.Move(@"C:\temp\test.txt", @"C:\temp\moved.txt");
    File.Delete(@"C:\temp\backup.txt");
}

// Directory 클래스 - 디렉터리 작업
if (!Directory.Exists(@"C:\temp\subfolder"))
{
    Directory.CreateDirectory(@"C:\temp\subfolder");
}

string[] files = Directory.GetFiles(@"C:\temp", "*.txt");
string[] dirs = Directory.GetDirectories(@"C:\temp");
```

### 예제: 디렉터리/파일 정보 조회
```csharp
public class FileExplorer
{
    public static void ListDirectory(string path, string indent = "")
    {
        DirectoryInfo dir = new DirectoryInfo(path);
        
        // 현재 디렉터리 출력
        Console.WriteLine($"{indent}[{dir.Name}]");
        
        // 파일 목록
        foreach (FileInfo file in dir.GetFiles())
        {
            Console.WriteLine($"{indent}  {file.Name} ({file.Length:N0} bytes)");
        }
        
        // 하위 디렉터리 재귀 탐색
        foreach (DirectoryInfo subDir in dir.GetDirectories())
        {
            ListDirectory(subDir.FullName, indent + "  ");
        }
    }
    
    public static long CalculateDirectorySize(string path)
    {
        DirectoryInfo dir = new DirectoryInfo(path);
        return dir.GetFiles("*", SearchOption.AllDirectories)
                  .Sum(file => file.Length);
    }
}
```

### 예제: 파일/디렉터리 생성
```csharp
public class FileManager
{
    public static void CreateFileStructure()
    {
        string basePath = @"C:\temp\project";
        
        // 디렉터리 구조 생성
        Directory.CreateDirectory(Path.Combine(basePath, "src"));
        Directory.CreateDirectory(Path.Combine(basePath, "docs"));
        Directory.CreateDirectory(Path.Combine(basePath, "tests"));
        
        // 파일 생성
        File.WriteAllText(Path.Combine(basePath, "README.md"), "# My Project");
        File.WriteAllText(Path.Combine(basePath, ".gitignore"), "bin/\nobj/");
        
        // 파일 속성 변경
        FileInfo readme = new FileInfo(Path.Combine(basePath, "README.md"));
        readme.Attributes = FileAttributes.ReadOnly;
    }
}
```

## Stream 클래스

### Stream 기본 개념
```csharp
// Stream은 추상 클래스
// 주요 구현체: FileStream, MemoryStream, NetworkStream

// 기본 Stream 작업
using (FileStream fs = new FileStream(@"C:\temp\data.bin", FileMode.Create))
{
    byte[] data = { 0x48, 0x65, 0x6C, 0x6C, 0x6F };  // "Hello"
    fs.Write(data, 0, data.Length);
    
    // 위치 이동
    fs.Position = 0;
    
    // 읽기
    byte[] buffer = new byte[5];
    int bytesRead = fs.Read(buffer, 0, buffer.Length);
}
```

### MemoryStream
```csharp
// 메모리 상에서 스트림 작업
using (MemoryStream ms = new MemoryStream())
{
    // 쓰기
    byte[] data = Encoding.UTF8.GetBytes("Hello, Memory Stream!");
    ms.Write(data, 0, data.Length);
    
    // 읽기를 위해 위치 리셋
    ms.Position = 0;
    
    // 읽기
    byte[] buffer = new byte[ms.Length];
    ms.Read(buffer, 0, buffer.Length);
    string text = Encoding.UTF8.GetString(buffer);
    
    // byte 배열로 변환
    byte[] result = ms.ToArray();
}
```

### BufferedStream
```csharp
// 버퍼링을 통한 성능 향상
using (FileStream fs = new FileStream(@"C:\temp\large.dat", FileMode.Create))
using (BufferedStream bs = new BufferedStream(fs, 65536))  // 64KB 버퍼
{
    for (int i = 0; i < 1000000; i++)
    {
        byte[] data = BitConverter.GetBytes(i);
        bs.Write(data, 0, data.Length);
    }
}  // using 블록을 벗어나면 자동으로 Flush 및 Close
```

## using 선언 (C# 8.0+)

```csharp
// 기존 using 문
using (var file = new FileStream("data.txt", FileMode.Create))
{
    // 파일 작업
}  // 여기서 Dispose 호출

// using 선언 (C# 8.0+)
void ProcessFile()
{
    using var file = new FileStream("data.txt", FileMode.Create);
    using var writer = new StreamWriter(file);
    
    writer.WriteLine("Hello");
    // 메서드 끝에서 자동으로 Dispose
}

// 여러 리소스 관리
void CopyFile(string source, string destination)
{
    using var sourceStream = File.OpenRead(source);
    using var destStream = File.Create(destination);
    
    sourceStream.CopyTo(destStream);
    // 메서드 끝에서 두 스트림 모두 Dispose
}
```

## BinaryWriter/BinaryReader

### 이진 데이터 쓰기
```csharp
public class BinaryDataWriter
{
    public static void WriteBinaryData(string filename)
    {
        using (FileStream fs = new FileStream(filename, FileMode.Create))
        using (BinaryWriter writer = new BinaryWriter(fs))
        {
            // 다양한 타입 쓰기
            writer.Write(true);          // bool
            writer.Write((byte)255);     // byte
            writer.Write('A');           // char
            writer.Write(123.456);       // double
            writer.Write(42);            // int
            writer.Write("Hello Binary"); // string
            
            // 배열 쓰기
            byte[] data = { 1, 2, 3, 4, 5 };
            writer.Write(data.Length);   // 길이 먼저
            writer.Write(data);          // 데이터
        }
    }
}
```

### 이진 데이터 읽기
```csharp
public class BinaryDataReader
{
    public static void ReadBinaryData(string filename)
    {
        using (FileStream fs = new FileStream(filename, FileMode.Open))
        using (BinaryReader reader = new BinaryReader(fs))
        {
            // 쓴 순서대로 읽기
            bool boolValue = reader.ReadBoolean();
            byte byteValue = reader.ReadByte();
            char charValue = reader.ReadChar();
            double doubleValue = reader.ReadDouble();
            int intValue = reader.ReadInt32();
            string stringValue = reader.ReadString();
            
            // 배열 읽기
            int length = reader.ReadInt32();
            byte[] data = reader.ReadBytes(length);
            
            Console.WriteLine($"Bool: {boolValue}");
            Console.WriteLine($"String: {stringValue}");
        }
    }
}
```

## StreamWriter/StreamReader

### 텍스트 파일 쓰기
```csharp
// StreamWriter 사용
using (StreamWriter writer = new StreamWriter(@"C:\temp\text.txt"))
{
    writer.WriteLine("첫 번째 줄");
    writer.WriteLine("두 번째 줄");
    writer.Write("줄바꿈 없이");
    writer.Write(" 이어서 쓰기");
}

// File 클래스 헬퍼 메서드
string[] lines = { "Line 1", "Line 2", "Line 3" };
File.WriteAllLines(@"C:\temp\lines.txt", lines);
File.WriteAllText(@"C:\temp\text.txt", "전체 텍스트");

// 추가 모드
using (StreamWriter writer = new StreamWriter(@"C:\temp\log.txt", append: true))
{
    writer.WriteLine($"[{DateTime.Now}] 로그 메시지");
}
```

### 텍스트 파일 읽기
```csharp
// StreamReader 사용
using (StreamReader reader = new StreamReader(@"C:\temp\text.txt"))
{
    // 한 줄씩 읽기
    string line;
    while ((line = reader.ReadLine()) != null)
    {
        Console.WriteLine(line);
    }
    
    // 또는 전체 읽기
    reader.BaseStream.Position = 0;  // 처음으로 되돌리기
    string allText = reader.ReadToEnd();
}

// File 클래스 헬퍼 메서드
string[] allLines = File.ReadAllLines(@"C:\temp\text.txt");
string fullText = File.ReadAllText(@"C:\temp\text.txt");

// 대용량 파일 처리
foreach (string line in File.ReadLines(@"C:\temp\largefile.txt"))
{
    ProcessLine(line);  // 메모리 효율적
}
```

## 객체 직렬화

### Binary Serialization (레거시)
```csharp
[Serializable]
public class Person
{
    public string Name { get; set; }
    public int Age { get; set; }
    [NonSerialized]
    public string Password;  // 직렬화 제외
}

// 이진 직렬화 (현재는 보안상 권장하지 않음)
public static void BinarySerialize(Person person, string filename)
{
    using (FileStream fs = new FileStream(filename, FileMode.Create))
    {
        BinaryFormatter formatter = new BinaryFormatter();
        formatter.Serialize(fs, person);
    }
}
```

### JSON 직렬화
```csharp
using System.Text.Json;

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
    public DateTime Created { get; set; }
}

// JSON 직렬화
public static void JsonSerialize()
{
    var product = new Product
    {
        Id = 1,
        Name = "Laptop",
        Price = 999.99m,
        Created = DateTime.Now
    };
    
    // 직렬화
    string json = JsonSerializer.Serialize(product, new JsonSerializerOptions
    {
        WriteIndented = true
    });
    File.WriteAllText("product.json", json);
    
    // 역직렬화
    string jsonText = File.ReadAllText("product.json");
    Product deserializedProduct = JsonSerializer.Deserialize<Product>(jsonText);
}
```

### XML 직렬화
```csharp
using System.Xml.Serialization;

[XmlRoot("Product")]
public class ProductXml
{
    [XmlAttribute("id")]
    public int Id { get; set; }
    
    [XmlElement("ProductName")]
    public string Name { get; set; }
    
    [XmlElement("Price")]
    public decimal Price { get; set; }
    
    [XmlIgnore]
    public string InternalCode { get; set; }
}

// XML 직렬화
public static void XmlSerialize()
{
    var product = new ProductXml
    {
        Id = 1,
        Name = "Mouse",
        Price = 29.99m
    };
    
    XmlSerializer serializer = new XmlSerializer(typeof(ProductXml));
    
    // 직렬화
    using (FileStream fs = new FileStream("product.xml", FileMode.Create))
    {
        serializer.Serialize(fs, product);
    }
    
    // 역직렬화
    using (FileStream fs = new FileStream("product.xml", FileMode.Open))
    {
        ProductXml deserialized = (ProductXml)serializer.Deserialize(fs);
    }
}
```

## 파일 작업 실전 예제

### 로그 파일 관리
```csharp
public class FileLogger
{
    private readonly string logPath;
    private readonly object lockObj = new object();
    
    public FileLogger(string logPath)
    {
        this.logPath = logPath;
        Directory.CreateDirectory(Path.GetDirectoryName(logPath));
    }
    
    public void Log(string message, LogLevel level = LogLevel.Info)
    {
        string logEntry = $"[{DateTime.Now:yyyy-MM-dd HH:mm:ss}] [{level}] {message}";
        
        lock (lockObj)
        {
            File.AppendAllText(logPath, logEntry + Environment.NewLine);
        }
        
        // 로그 파일 크기 관리
        RotateLogIfNeeded();
    }
    
    private void RotateLogIfNeeded()
    {
        FileInfo fileInfo = new FileInfo(logPath);
        if (fileInfo.Exists && fileInfo.Length > 10 * 1024 * 1024)  // 10MB
        {
            string backupPath = Path.ChangeExtension(logPath, 
                $".{DateTime.Now:yyyyMMddHHmmss}.log");
            File.Move(logPath, backupPath);
        }
    }
}

public enum LogLevel
{
    Debug, Info, Warning, Error
}
```

### CSV 파일 처리
```csharp
public class CsvProcessor
{
    public static void WriteCsv<T>(string filename, IEnumerable<T> data)
    {
        using (StreamWriter writer = new StreamWriter(filename))
        {
            // 헤더 쓰기
            var properties = typeof(T).GetProperties();
            writer.WriteLine(string.Join(",", properties.Select(p => p.Name)));
            
            // 데이터 쓰기
            foreach (var item in data)
            {
                var values = properties.Select(p => p.GetValue(item)?.ToString() ?? "");
                writer.WriteLine(string.Join(",", values));
            }
        }
    }
    
    public static List<Dictionary<string, string>> ReadCsv(string filename)
    {
        var result = new List<Dictionary<string, string>>();
        
        using (StreamReader reader = new StreamReader(filename))
        {
            string headerLine = reader.ReadLine();
            if (headerLine == null) return result;
            
            string[] headers = headerLine.Split(',');
            
            string line;
            while ((line = reader.ReadLine()) != null)
            {
                string[] values = line.Split(',');
                var row = new Dictionary<string, string>();
                
                for (int i = 0; i < headers.Length && i < values.Length; i++)
                {
                    row[headers[i]] = values[i];
                }
                
                result.Add(row);
            }
        }
        
        return result;
    }
}
```

### 파일 감시자 (FileSystemWatcher)
```csharp
public class FileMonitor
{
    private FileSystemWatcher watcher;
    
    public void StartMonitoring(string path)
    {
        watcher = new FileSystemWatcher(path);
        
        // 감시할 변경 사항
        watcher.NotifyFilter = NotifyFilters.LastWrite 
                             | NotifyFilters.FileName 
                             | NotifyFilters.DirectoryName;
        
        // 필터
        watcher.Filter = "*.txt";
        
        // 이벤트 핸들러
        watcher.Created += OnCreated;
        watcher.Changed += OnChanged;
        watcher.Deleted += OnDeleted;
        watcher.Renamed += OnRenamed;
        
        // 감시 시작
        watcher.EnableRaisingEvents = true;
    }
    
    private void OnCreated(object sender, FileSystemEventArgs e)
    {
        Console.WriteLine($"File created: {e.FullPath}");
    }
    
    private void OnChanged(object sender, FileSystemEventArgs e)
    {
        Console.WriteLine($"File changed: {e.FullPath}");
    }
    
    private void OnDeleted(object sender, FileSystemEventArgs e)
    {
        Console.WriteLine($"File deleted: {e.FullPath}");
    }
    
    private void OnRenamed(object sender, RenamedEventArgs e)
    {
        Console.WriteLine($"File renamed: {e.OldFullPath} -> {e.FullPath}");
    }
}
```

## 핵심 개념 정리
- **FileInfo/DirectoryInfo**: 파일/디렉터리 정보 관리
- **Stream**: 바이트 단위 입출력의 추상화
- **BinaryWriter/Reader**: 이진 데이터 처리
- **StreamWriter/Reader**: 텍스트 데이터 처리
- **직렬화**: 객체를 저장/전송 가능한 형태로 변환