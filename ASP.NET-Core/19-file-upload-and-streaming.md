# 파일 업로드와 스트리밍

## 파일 업로드 개요

ASP.NET Core는 다양한 파일 업로드 시나리오를 지원합니다. 작은 파일부터 대용량 파일까지, 단일 파일부터 다중 파일 업로드까지 처리할 수 있습니다.

### 파일 업로드 방식
- **버퍼링**: 전체 파일을 메모리나 디스크에 저장 후 처리
- **스트리밍**: 파일을 청크 단위로 읽으면서 처리
- **다중 파일**: 여러 파일 동시 업로드
- **청크 업로드**: 대용량 파일을 작은 조각으로 나누어 업로드

## 기본 파일 업로드

### 단일 파일 업로드
```csharp
[ApiController]
[Route("api/[controller]")]
public class FileUploadController : ControllerBase
{
    private readonly IWebHostEnvironment _environment;
    private readonly ILogger<FileUploadController> _logger;
    
    public FileUploadController(
        IWebHostEnvironment environment,
        ILogger<FileUploadController> logger)
    {
        _environment = environment;
        _logger = logger;
    }
    
    [HttpPost("upload")]
    public async Task<IActionResult> UploadFile(IFormFile file)
    {
        if (file == null || file.Length == 0)
        {
            return BadRequest("No file uploaded");
        }
        
        // 파일 크기 검증 (10MB)
        if (file.Length > 10 * 1024 * 1024)
        {
            return BadRequest("File size exceeds 10MB limit");
        }
        
        // 파일 확장자 검증
        var allowedExtensions = new[] { ".jpg", ".jpeg", ".png", ".gif", ".pdf" };
        var extension = Path.GetExtension(file.FileName).ToLowerInvariant();
        
        if (!allowedExtensions.Contains(extension))
        {
            return BadRequest("Invalid file type");
        }
        
        // 안전한 파일명 생성
        var fileName = $"{Guid.NewGuid()}{extension}";
        var uploadPath = Path.Combine(_environment.WebRootPath, "uploads");
        
        // 디렉토리가 없으면 생성
        if (!Directory.Exists(uploadPath))
        {
            Directory.CreateDirectory(uploadPath);
        }
        
        var filePath = Path.Combine(uploadPath, fileName);
        
        // 파일 저장
        using (var stream = new FileStream(filePath, FileMode.Create))
        {
            await file.CopyToAsync(stream);
        }
        
        _logger.LogInformation("File uploaded: {FileName}, Size: {Size}", fileName, file.Length);
        
        return Ok(new
        {
            fileName,
            originalName = file.FileName,
            size = file.Length,
            contentType = file.ContentType,
            url = $"/uploads/{fileName}"
        });
    }
}
```

### 다중 파일 업로드
```csharp
[HttpPost("upload-multiple")]
[RequestSizeLimit(100_000_000)] // 100MB 제한
public async Task<IActionResult> UploadMultipleFiles(List<IFormFile> files)
{
    if (files == null || !files.Any())
    {
        return BadRequest("No files uploaded");
    }
    
    var uploadResults = new List<FileUploadResult>();
    var uploadPath = Path.Combine(_environment.ContentRootPath, "uploads");
    
    foreach (var file in files)
    {
        try
        {
            var result = await ProcessFileAsync(file, uploadPath);
            uploadResults.Add(result);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error uploading file: {FileName}", file.FileName);
            uploadResults.Add(new FileUploadResult
            {
                FileName = file.FileName,
                Success = false,
                Error = ex.Message
            });
        }
    }
    
    return Ok(new
    {
        totalFiles = files.Count,
        successfulUploads = uploadResults.Count(r => r.Success),
        failedUploads = uploadResults.Count(r => !r.Success),
        results = uploadResults
    });
}

private async Task<FileUploadResult> ProcessFileAsync(IFormFile file, string uploadPath)
{
    // 파일 검증
    var validationResult = ValidateFile(file);
    if (!validationResult.IsValid)
    {
        return new FileUploadResult
        {
            FileName = file.FileName,
            Success = false,
            Error = validationResult.Error
        };
    }
    
    // 파일 저장
    var fileName = GenerateUniqueFileName(file.FileName);
    var filePath = Path.Combine(uploadPath, fileName);
    
    using (var stream = new FileStream(filePath, FileMode.Create))
    {
        await file.CopyToAsync(stream);
    }
    
    // 파일 정보 저장 (데이터베이스)
    var fileInfo = new FileInfo
    {
        Id = Guid.NewGuid(),
        OriginalName = file.FileName,
        StoredName = fileName,
        Size = file.Length,
        ContentType = file.ContentType,
        UploadedAt = DateTime.UtcNow
    };
    
    await SaveFileInfoAsync(fileInfo);
    
    return new FileUploadResult
    {
        FileName = file.FileName,
        StoredName = fileName,
        Size = file.Length,
        Success = true
    };
}
```

## 대용량 파일 업로드

### 스트리밍 업로드
```csharp
[HttpPost("upload-stream")]
[DisableFormValueModelBinding]
[RequestSizeLimit(2147483648)] // 2GB
public async Task<IActionResult> UploadLargeFile()
{
    if (!MultipartRequestHelper.IsMultipartContentType(Request.ContentType))
    {
        return BadRequest("Not a multipart request");
    }
    
    var boundary = MultipartRequestHelper.GetBoundary(
        MediaTypeHeaderValue.Parse(Request.ContentType),
        _defaultFormOptions.MultipartBoundaryLengthLimit);
    
    var reader = new MultipartReader(boundary, HttpContext.Request.Body);
    var section = await reader.ReadNextSectionAsync();
    
    while (section != null)
    {
        var hasContentDispositionHeader = ContentDispositionHeaderValue.TryParse(
            section.ContentDisposition, out var contentDisposition);
        
        if (hasContentDispositionHeader)
        {
            if (MultipartRequestHelper.HasFileContentDisposition(contentDisposition))
            {
                var fileName = contentDisposition.FileName.Value;
                var trustedFileNameForFileStorage = Path.GetRandomFileName();
                var streamedFileContent = await ProcessStreamedFile(
                    section, contentDisposition, ModelState,
                    _permittedExtensions, _fileSizeLimit);
                
                if (!ModelState.IsValid)
                {
                    return BadRequest(ModelState);
                }
                
                using (var targetStream = System.IO.File.Create(
                    Path.Combine(_targetFilePath, trustedFileNameForFileStorage)))
                {
                    await targetStream.WriteAsync(streamedFileContent);
                    
                    _logger.LogInformation(
                        "Uploaded file '{TrustedFileNameForFileStorage}' " +
                        "saved to '{TargetFilePath}' as {TrustedFileNameForFileStorage}",
                        fileName, _targetFilePath, trustedFileNameForFileStorage);
                }
            }
        }
        
        section = await reader.ReadNextSectionAsync();
    }
    
    return Created(nameof(StreamingController), null);
}

// 스트리밍 처리 헬퍼
private static async Task<byte[]> ProcessStreamedFile(
    MultipartSection section,
    ContentDispositionHeaderValue contentDisposition,
    ModelStateDictionary modelState,
    string[] permittedExtensions,
    long sizeLimit)
{
    try
    {
        using var memoryStream = new MemoryStream();
        await section.Body.CopyToAsync(memoryStream);
        
        if (memoryStream.Length == 0)
        {
            modelState.AddModelError("File", "The file is empty.");
        }
        else if (memoryStream.Length > sizeLimit)
        {
            var megabyteSizeLimit = sizeLimit / 1048576;
            modelState.AddModelError("File",
                $"The file exceeds {megabyteSizeLimit:N1} MB.");
        }
        else if (!IsValidFileExtensionAndSignature(
            contentDisposition.FileName.Value, memoryStream, permittedExtensions))
        {
            modelState.AddModelError("File",
                "The file type isn't permitted or the file's " +
                "signature doesn't match the file's extension.");
        }
        else
        {
            return memoryStream.ToArray();
        }
    }
    catch (Exception ex)
    {
        modelState.AddModelError("File", "The upload failed.");
    }
    
    return Array.Empty<byte>();
}
```

### DisableFormValueModelBinding 특성
```csharp
[AttributeUsage(AttributeTargets.Class | AttributeTargets.Method)]
public class DisableFormValueModelBindingAttribute : Attribute, IResourceFilter
{
    public void OnResourceExecuting(ResourceExecutingContext context)
    {
        var factories = context.ValueProviderFactories;
        factories.RemoveType<FormValueProviderFactory>();
        factories.RemoveType<FormFileValueProviderFactory>();
        factories.RemoveType<JQueryFormValueProviderFactory>();
    }
    
    public void OnResourceExecuted(ResourceExecutedContext context)
    {
    }
}
```

## 파일 다운로드와 스트리밍

### 파일 다운로드
```csharp
[HttpGet("download/{id}")]
public async Task<IActionResult> DownloadFile(Guid id)
{
    var fileInfo = await GetFileInfoAsync(id);
    
    if (fileInfo == null)
    {
        return NotFound();
    }
    
    var filePath = Path.Combine(_uploadPath, fileInfo.StoredName);
    
    if (!System.IO.File.Exists(filePath))
    {
        return NotFound();
    }
    
    var memory = new MemoryStream();
    using (var stream = new FileStream(filePath, FileMode.Open))
    {
        await stream.CopyToAsync(memory);
    }
    memory.Position = 0;
    
    return File(memory, fileInfo.ContentType, fileInfo.OriginalName);
}

// 스트리밍 다운로드
[HttpGet("stream/{id}")]
public async Task<IActionResult> StreamFile(Guid id)
{
    var fileInfo = await GetFileInfoAsync(id);
    
    if (fileInfo == null)
    {
        return NotFound();
    }
    
    var filePath = Path.Combine(_uploadPath, fileInfo.StoredName);
    
    if (!System.IO.File.Exists(filePath))
    {
        return NotFound();
    }
    
    var stream = new FileStream(filePath, FileMode.Open, FileAccess.Read);
    
    // FileStreamResult는 자동으로 스트림을 dispose함
    return new FileStreamResult(stream, fileInfo.ContentType)
    {
        FileDownloadName = fileInfo.OriginalName,
        EnableRangeProcessing = true // 부분 다운로드 지원
    };
}
```

### 비디오 스트리밍
```csharp
[HttpGet("video/{id}")]
public async Task<IActionResult> StreamVideo(Guid id)
{
    var videoInfo = await GetVideoInfoAsync(id);
    
    if (videoInfo == null)
    {
        return NotFound();
    }
    
    var videoPath = Path.Combine(_videoPath, videoInfo.FileName);
    
    if (!System.IO.File.Exists(videoPath))
    {
        return NotFound();
    }
    
    // Range 헤더 처리
    var enableRangeProcessing = !string.IsNullOrEmpty(Request.Headers.Range);
    
    if (enableRangeProcessing)
    {
        return PhysicalFile(videoPath, "video/mp4", enableRangeProcessing: true);
    }
    
    // 전체 파일 스트리밍
    var stream = new FileStream(videoPath, FileMode.Open, FileAccess.Read, FileShare.Read);
    return new FileStreamResult(stream, "video/mp4")
    {
        EnableRangeProcessing = true
    };
}

// 적응형 비트레이트 스트리밍 (HLS)
[HttpGet("video/{id}/playlist.m3u8")]
public async Task<IActionResult> GetHlsPlaylist(Guid id)
{
    var videoInfo = await GetVideoInfoAsync(id);
    
    if (videoInfo == null)
    {
        return NotFound();
    }
    
    var playlist = GenerateHlsPlaylist(videoInfo);
    
    return Content(playlist, "application/vnd.apple.mpegurl");
}

[HttpGet("video/{id}/segment-{segmentNumber}.ts")]
public async Task<IActionResult> GetHlsSegment(Guid id, int segmentNumber)
{
    var segmentPath = Path.Combine(_videoPath, $"{id}", $"segment-{segmentNumber}.ts");
    
    if (!System.IO.File.Exists(segmentPath))
    {
        return NotFound();
    }
    
    return PhysicalFile(segmentPath, "video/mp2t");
}
```

## 이미지 처리

### 이미지 업로드 및 리사이징
```csharp
public class ImageUploadService
{
    private readonly IWebHostEnvironment _environment;
    private readonly ILogger<ImageUploadService> _logger;
    
    public async Task<ImageUploadResult> UploadAndProcessImageAsync(IFormFile file)
    {
        // 이미지 파일 검증
        if (!IsImageFile(file))
        {
            throw new InvalidOperationException("File is not a valid image");
        }
        
        var uploadPath = Path.Combine(_environment.WebRootPath, "images");
        var fileName = $"{Guid.NewGuid()}{Path.GetExtension(file.FileName)}";
        
        // 원본 이미지 저장
        var originalPath = Path.Combine(uploadPath, "original", fileName);
        Directory.CreateDirectory(Path.GetDirectoryName(originalPath));
        
        using (var stream = new FileStream(originalPath, FileMode.Create))
        {
            await file.CopyToAsync(stream);
        }
        
        // 썸네일 생성
        var thumbnailPath = await CreateThumbnailAsync(originalPath, fileName, uploadPath);
        
        // 다양한 크기 생성
        var sizes = new[] { 
            (Width: 1920, Height: 1080, Name: "large"),
            (Width: 1280, Height: 720, Name: "medium"),
            (Width: 640, Height: 360, Name: "small")
        };
        
        var resizedImages = new Dictionary<string, string>();
        
        foreach (var size in sizes)
        {
            var resizedPath = await ResizeImageAsync(
                originalPath, fileName, uploadPath, size.Width, size.Height, size.Name);
            resizedImages[size.Name] = resizedPath;
        }
        
        return new ImageUploadResult
        {
            OriginalUrl = $"/images/original/{fileName}",
            ThumbnailUrl = thumbnailPath,
            ResizedUrls = resizedImages,
            FileName = fileName,
            Size = file.Length
        };
    }
    
    private async Task<string> CreateThumbnailAsync(
        string originalPath, string fileName, string uploadPath)
    {
        using var image = await Image.LoadAsync(originalPath);
        
        // 썸네일 크기 계산 (가로세로 비율 유지)
        var (width, height) = CalculateThumbnailSize(image.Width, image.Height, 200);
        
        image.Mutate(x => x.Resize(width, height));
        
        var thumbnailDir = Path.Combine(uploadPath, "thumbnails");
        Directory.CreateDirectory(thumbnailDir);
        
        var thumbnailPath = Path.Combine(thumbnailDir, fileName);
        await image.SaveAsync(thumbnailPath);
        
        return $"/images/thumbnails/{fileName}";
    }
    
    private async Task<string> ResizeImageAsync(
        string originalPath, string fileName, string uploadPath,
        int maxWidth, int maxHeight, string sizeName)
    {
        using var image = await Image.LoadAsync(originalPath);
        
        // 비율 유지하면서 리사이징
        var (width, height) = CalculateAspectRatio(
            image.Width, image.Height, maxWidth, maxHeight);
        
        image.Mutate(x => x
            .Resize(width, height)
            .Apply(img => OptimizeImage(img, sizeName)));
        
        var sizeDir = Path.Combine(uploadPath, sizeName);
        Directory.CreateDirectory(sizeDir);
        
        var resizedPath = Path.Combine(sizeDir, fileName);
        
        // 형식에 따라 품질 설정
        var encoder = GetEncoder(Path.GetExtension(fileName));
        await image.SaveAsync(resizedPath, encoder);
        
        return $"/images/{sizeName}/{fileName}";
    }
    
    private IImageEncoder GetEncoder(string extension)
    {
        return extension.ToLower() switch
        {
            ".jpg" or ".jpeg" => new JpegEncoder { Quality = 85 },
            ".png" => new PngEncoder { CompressionLevel = PngCompressionLevel.BestCompression },
            ".webp" => new WebpEncoder { Quality = 85 },
            _ => new JpegEncoder { Quality = 85 }
        };
    }
}
```

## 청크 업로드

### 청크 업로드 구현
```csharp
[ApiController]
[Route("api/[controller]")]
public class ChunkUploadController : ControllerBase
{
    private readonly IChunkUploadService _uploadService;
    
    [HttpPost("initialize")]
    public async Task<IActionResult> InitializeUpload([FromBody] InitializeUploadRequest request)
    {
        var uploadId = await _uploadService.InitializeUploadAsync(
            request.FileName,
            request.FileSize,
            request.ChunkSize);
        
        return Ok(new { uploadId });
    }
    
    [HttpPost("chunk")]
    public async Task<IActionResult> UploadChunk([FromForm] ChunkUploadRequest request)
    {
        if (request.File == null || request.File.Length == 0)
        {
            return BadRequest("No chunk data");
        }
        
        var result = await _uploadService.SaveChunkAsync(
            request.UploadId,
            request.ChunkIndex,
            request.File);
        
        if (!result.Success)
        {
            return BadRequest(result.Error);
        }
        
        return Ok(new
        {
            chunkIndex = request.ChunkIndex,
            uploaded = result.UploadedBytes,
            remaining = result.RemainingBytes
        });
    }
    
    [HttpPost("complete")]
    public async Task<IActionResult> CompleteUpload([FromBody] CompleteUploadRequest request)
    {
        var result = await _uploadService.CompleteUploadAsync(request.UploadId);
        
        if (!result.Success)
        {
            return BadRequest(result.Error);
        }
        
        return Ok(new
        {
            fileName = result.FileName,
            fileSize = result.FileSize,
            url = result.FileUrl
        });
    }
}

// 청크 업로드 서비스
public class ChunkUploadService : IChunkUploadService
{
    private readonly IMemoryCache _cache;
    private readonly string _tempPath;
    
    public async Task<string> InitializeUploadAsync(
        string fileName, long fileSize, int chunkSize)
    {
        var uploadId = Guid.NewGuid().ToString();
        var uploadInfo = new ChunkUploadInfo
        {
            UploadId = uploadId,
            FileName = fileName,
            FileSize = fileSize,
            ChunkSize = chunkSize,
            TotalChunks = (int)Math.Ceiling((double)fileSize / chunkSize),
            UploadedChunks = new HashSet<int>(),
            StartTime = DateTime.UtcNow
        };
        
        // 임시 디렉토리 생성
        var tempDir = Path.Combine(_tempPath, uploadId);
        Directory.CreateDirectory(tempDir);
        
        // 캐시에 업로드 정보 저장
        _cache.Set($"upload_{uploadId}", uploadInfo, TimeSpan.FromHours(24));
        
        return uploadId;
    }
    
    public async Task<ChunkUploadResult> SaveChunkAsync(
        string uploadId, int chunkIndex, IFormFile chunk)
    {
        if (!_cache.TryGetValue($"upload_{uploadId}", out ChunkUploadInfo uploadInfo))
        {
            return new ChunkUploadResult { Success = false, Error = "Upload not found" };
        }
        
        // 청크 저장
        var chunkPath = Path.Combine(_tempPath, uploadId, $"chunk_{chunkIndex}");
        
        using (var stream = new FileStream(chunkPath, FileMode.Create))
        {
            await chunk.CopyToAsync(stream);
        }
        
        // 업로드 정보 업데이트
        lock (uploadInfo.UploadedChunks)
        {
            uploadInfo.UploadedChunks.Add(chunkIndex);
        }
        
        var uploadedBytes = uploadInfo.UploadedChunks.Count * uploadInfo.ChunkSize;
        var remainingBytes = uploadInfo.FileSize - uploadedBytes;
        
        return new ChunkUploadResult
        {
            Success = true,
            UploadedBytes = uploadedBytes,
            RemainingBytes = Math.Max(0, remainingBytes)
        };
    }
    
    public async Task<CompleteUploadResult> CompleteUploadAsync(string uploadId)
    {
        if (!_cache.TryGetValue($"upload_{uploadId}", out ChunkUploadInfo uploadInfo))
        {
            return new CompleteUploadResult { Success = false, Error = "Upload not found" };
        }
        
        // 모든 청크가 업로드되었는지 확인
        if (uploadInfo.UploadedChunks.Count != uploadInfo.TotalChunks)
        {
            return new CompleteUploadResult
            {
                Success = false,
                Error = $"Missing chunks. Expected: {uploadInfo.TotalChunks}, Received: {uploadInfo.UploadedChunks.Count}"
            };
        }
        
        // 청크 병합
        var finalPath = Path.Combine(_uploadPath, $"{Guid.NewGuid()}{Path.GetExtension(uploadInfo.FileName)}");
        
        using (var finalStream = new FileStream(finalPath, FileMode.Create))
        {
            for (int i = 0; i < uploadInfo.TotalChunks; i++)
            {
                var chunkPath = Path.Combine(_tempPath, uploadId, $"chunk_{i}");
                
                using (var chunkStream = new FileStream(chunkPath, FileMode.Open))
                {
                    await chunkStream.CopyToAsync(finalStream);
                }
            }
        }
        
        // 임시 파일 정리
        Directory.Delete(Path.Combine(_tempPath, uploadId), true);
        
        // 캐시에서 제거
        _cache.Remove($"upload_{uploadId}");
        
        return new CompleteUploadResult
        {
            Success = true,
            FileName = uploadInfo.FileName,
            FileSize = uploadInfo.FileSize,
            FileUrl = $"/files/{Path.GetFileName(finalPath)}"
        };
    }
}
```

## 파일 보안

### 파일 검증
```csharp
public class FileValidationService
{
    private readonly Dictionary<string, byte[]> _fileSignatures = new()
    {
        { ".jpg", new byte[] { 0xFF, 0xD8, 0xFF } },
        { ".jpeg", new byte[] { 0xFF, 0xD8, 0xFF } },
        { ".png", new byte[] { 0x89, 0x50, 0x4E, 0x47 } },
        { ".gif", new byte[] { 0x47, 0x49, 0x46 } },
        { ".pdf", new byte[] { 0x25, 0x50, 0x44, 0x46 } },
        { ".zip", new byte[] { 0x50, 0x4B, 0x03, 0x04 } },
        { ".docx", new byte[] { 0x50, 0x4B, 0x03, 0x04 } }
    };
    
    public ValidationResult ValidateFile(IFormFile file, FileUploadOptions options)
    {
        // 파일 크기 검증
        if (file.Length > options.MaxFileSize)
        {
            return ValidationResult.Fail($"File size exceeds maximum allowed size of {options.MaxFileSize / 1024 / 1024}MB");
        }
        
        // 파일 확장자 검증
        var extension = Path.GetExtension(file.FileName).ToLowerInvariant();
        if (!options.AllowedExtensions.Contains(extension))
        {
            return ValidationResult.Fail($"File extension '{extension}' is not allowed");
        }
        
        // MIME 타입 검증
        if (!options.AllowedMimeTypes.Contains(file.ContentType))
        {
            return ValidationResult.Fail($"File content type '{file.ContentType}' is not allowed");
        }
        
        // 파일 시그니처 검증
        if (!ValidateFileSignature(file, extension))
        {
            return ValidationResult.Fail("File content does not match its extension");
        }
        
        // 파일명 검증
        if (!IsValidFileName(file.FileName))
        {
            return ValidationResult.Fail("Invalid file name");
        }
        
        return ValidationResult.Success();
    }
    
    private bool ValidateFileSignature(IFormFile file, string extension)
    {
        if (!_fileSignatures.TryGetValue(extension, out var expectedSignature))
        {
            return true; // 시그니처가 정의되지 않은 파일 타입은 통과
        }
        
        using var reader = new BinaryReader(file.OpenReadStream());
        var headerBytes = reader.ReadBytes(expectedSignature.Length);
        
        return headerBytes.SequenceEqual(expectedSignature);
    }
    
    private bool IsValidFileName(string fileName)
    {
        // 위험한 문자 검사
        var invalidChars = Path.GetInvalidFileNameChars();
        if (fileName.IndexOfAny(invalidChars) >= 0)
        {
            return false;
        }
        
        // 경로 탐색 공격 방지
        if (fileName.Contains("..") || fileName.Contains("/") || fileName.Contains("\\"))
        {
            return false;
        }
        
        // 예약된 파일명 검사
        var reservedNames = new[] { "CON", "PRN", "AUX", "NUL", "COM1", "LPT1" };
        var nameWithoutExtension = Path.GetFileNameWithoutExtension(fileName).ToUpper();
        
        return !reservedNames.Contains(nameWithoutExtension);
    }
}

// 바이러스 검사 서비스
public class AntivirusService
{
    private readonly HttpClient _httpClient;
    
    public async Task<ScanResult> ScanFileAsync(Stream fileStream, string fileName)
    {
        // Windows Defender API 사용 예제
        try
        {
            // 임시 파일로 저장
            var tempFile = Path.GetTempFileName();
            using (var fs = File.Create(tempFile))
            {
                await fileStream.CopyToAsync(fs);
            }
            
            // Windows Defender 스캔
            var scanResult = await RunWindowsDefenderScan(tempFile);
            
            // 임시 파일 삭제
            File.Delete(tempFile);
            
            return scanResult;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Antivirus scan failed");
            return new ScanResult { IsSafe = false, Error = "Scan failed" };
        }
    }
    
    private async Task<ScanResult> RunWindowsDefenderScan(string filePath)
    {
        var processInfo = new ProcessStartInfo
        {
            FileName = "MpCmdRun.exe",
            Arguments = $"-Scan -ScanType 3 -File \"{filePath}\"",
            UseShellExecute = false,
            RedirectStandardOutput = true,
            CreateNoWindow = true
        };
        
        using var process = Process.Start(processInfo);
        await process.WaitForExitAsync();
        
        return new ScanResult
        {
            IsSafe = process.ExitCode == 0,
            Error = process.ExitCode != 0 ? "Threat detected" : null
        };
    }
}
```

## 클라우드 스토리지 통합

### Azure Blob Storage
```csharp
public class AzureBlobStorageService : ICloudStorageService
{
    private readonly BlobServiceClient _blobServiceClient;
    private readonly ILogger<AzureBlobStorageService> _logger;
    
    public AzureBlobStorageService(
        BlobServiceClient blobServiceClient,
        ILogger<AzureBlobStorageService> logger)
    {
        _blobServiceClient = blobServiceClient;
        _logger = logger;
    }
    
    public async Task<CloudUploadResult> UploadAsync(
        Stream fileStream, 
        string fileName, 
        string containerName = "uploads")
    {
        try
        {
            var containerClient = _blobServiceClient.GetBlobContainerClient(containerName);
            await containerClient.CreateIfNotExistsAsync(PublicAccessType.None);
            
            var blobName = $"{Guid.NewGuid()}/{fileName}";
            var blobClient = containerClient.GetBlobClient(blobName);
            
            // 업로드 옵션
            var uploadOptions = new BlobUploadOptions
            {
                HttpHeaders = new BlobHttpHeaders
                {
                    ContentType = GetContentType(fileName),
                    CacheControl = "public, max-age=3600"
                },
                TransferOptions = new StorageTransferOptions
                {
                    MaximumConcurrency = 2,
                    MaximumTransferSize = 4 * 1024 * 1024 // 4MB chunks
                }
            };
            
            // 진행률 추적
            var progressHandler = new Progress<long>(bytesTransferred =>
            {
                _logger.LogInformation("Upload progress: {Bytes} bytes", bytesTransferred);
            });
            
            uploadOptions.ProgressHandler = progressHandler;
            
            // 업로드
            var response = await blobClient.UploadAsync(fileStream, uploadOptions);
            
            // SAS URL 생성 (1시간 유효)
            var sasUrl = GenerateSasUrl(blobClient, TimeSpan.FromHours(1));
            
            return new CloudUploadResult
            {
                Success = true,
                BlobName = blobName,
                Url = blobClient.Uri.ToString(),
                SasUrl = sasUrl,
                ETag = response.Value.ETag.ToString()
            };
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to upload to Azure Blob Storage");
            return new CloudUploadResult
            {
                Success = false,
                Error = ex.Message
            };
        }
    }
    
    public async Task<Stream> DownloadAsync(string blobName, string containerName = "uploads")
    {
        var containerClient = _blobServiceClient.GetBlobContainerClient(containerName);
        var blobClient = containerClient.GetBlobClient(blobName);
        
        var response = await blobClient.DownloadStreamingAsync();
        return response.Value.Content;
    }
    
    private string GenerateSasUrl(BlobClient blobClient, TimeSpan validFor)
    {
        if (!blobClient.CanGenerateSasUri)
        {
            return null;
        }
        
        var sasBuilder = new BlobSasBuilder
        {
            BlobContainerName = blobClient.BlobContainerName,
            BlobName = blobClient.Name,
            Resource = "b",
            StartsOn = DateTimeOffset.UtcNow,
            ExpiresOn = DateTimeOffset.UtcNow.Add(validFor)
        };
        
        sasBuilder.SetPermissions(BlobSasPermissions.Read);
        
        return blobClient.GenerateSasUri(sasBuilder).ToString();
    }
}
```

### AWS S3
```csharp
public class S3StorageService : ICloudStorageService
{
    private readonly IAmazonS3 _s3Client;
    private readonly string _bucketName;
    
    public S3StorageService(IAmazonS3 s3Client, IConfiguration configuration)
    {
        _s3Client = s3Client;
        _bucketName = configuration["AWS:S3:BucketName"];
    }
    
    public async Task<CloudUploadResult> UploadAsync(
        Stream fileStream,
        string fileName,
        string folder = "uploads")
    {
        var key = $"{folder}/{Guid.NewGuid()}/{fileName}";
        
        var uploadRequest = new PutObjectRequest
        {
            BucketName = _bucketName,
            Key = key,
            InputStream = fileStream,
            ContentType = GetContentType(fileName),
            ServerSideEncryptionMethod = ServerSideEncryptionMethod.AES256,
            StorageClass = S3StorageClass.Standard
        };
        
        // 메타데이터 추가
        uploadRequest.Metadata.Add("uploaded-by", "api");
        uploadRequest.Metadata.Add("upload-date", DateTime.UtcNow.ToString("O"));
        
        var response = await _s3Client.PutObjectAsync(uploadRequest);
        
        // Pre-signed URL 생성 (1시간 유효)
        var urlRequest = new GetPreSignedUrlRequest
        {
            BucketName = _bucketName,
            Key = key,
            Verb = HttpVerb.GET,
            Expires = DateTime.UtcNow.AddHours(1)
        };
        
        var presignedUrl = await _s3Client.GetPreSignedURLAsync(urlRequest);
        
        return new CloudUploadResult
        {
            Success = true,
            Key = key,
            Url = $"https://{_bucketName}.s3.amazonaws.com/{key}",
            PresignedUrl = presignedUrl,
            ETag = response.ETag
        };
    }
    
    // Multipart 업로드 (대용량 파일)
    public async Task<string> UploadLargeFileAsync(
        string filePath,
        string key,
        IProgress<long> progress = null)
    {
        var fileInfo = new FileInfo(filePath);
        var partSize = 5 * 1024 * 1024; // 5MB parts
        
        var initiateRequest = new InitiateMultipartUploadRequest
        {
            BucketName = _bucketName,
            Key = key,
            ContentType = GetContentType(filePath)
        };
        
        var initResponse = await _s3Client.InitiateMultipartUploadAsync(initiateRequest);
        var uploadId = initResponse.UploadId;
        
        try
        {
            var parts = new List<PartETag>();
            var filePosition = 0L;
            
            for (int i = 1; filePosition < fileInfo.Length; i++)
            {
                var size = Math.Min(partSize, fileInfo.Length - filePosition);
                
                var uploadRequest = new UploadPartRequest
                {
                    BucketName = _bucketName,
                    Key = key,
                    UploadId = uploadId,
                    PartNumber = i,
                    PartSize = size,
                    FilePosition = filePosition,
                    FilePath = filePath
                };
                
                var response = await _s3Client.UploadPartAsync(uploadRequest);
                parts.Add(new PartETag(i, response.ETag));
                
                filePosition += size;
                progress?.Report(filePosition);
            }
            
            var completeRequest = new CompleteMultipartUploadRequest
            {
                BucketName = _bucketName,
                Key = key,
                UploadId = uploadId,
                PartETags = parts
            };
            
            await _s3Client.CompleteMultipartUploadAsync(completeRequest);
            
            return key;
        }
        catch
        {
            // 실패 시 업로드 취소
            await _s3Client.AbortMultipartUploadAsync(new AbortMultipartUploadRequest
            {
                BucketName = _bucketName,
                Key = key,
                UploadId = uploadId
            });
            
            throw;
        }
    }
}
```

## 성능 최적화

### 병렬 업로드
```csharp
public class ParallelUploadService
{
    private readonly SemaphoreSlim _semaphore;
    private readonly ICloudStorageService _storageService;
    
    public ParallelUploadService(ICloudStorageService storageService)
    {
        _storageService = storageService;
        _semaphore = new SemaphoreSlim(4); // 최대 4개 동시 업로드
    }
    
    public async Task<IEnumerable<UploadResult>> UploadMultipleAsync(
        IEnumerable<IFormFile> files)
    {
        var tasks = files.Select(async file =>
        {
            await _semaphore.WaitAsync();
            try
            {
                return await UploadSingleAsync(file);
            }
            finally
            {
                _semaphore.Release();
            }
        });
        
        return await Task.WhenAll(tasks);
    }
    
    // 청크 병렬 업로드
    public async Task<string> UploadInParallelChunksAsync(
        Stream fileStream,
        string fileName,
        int chunkSize = 5 * 1024 * 1024) // 5MB chunks
    {
        var uploadId = Guid.NewGuid().ToString();
        var chunks = new ConcurrentBag<ChunkInfo>();
        var uploadTasks = new List<Task>();
        
        var buffer = new byte[chunkSize];
        var chunkIndex = 0;
        
        while (true)
        {
            var bytesRead = await fileStream.ReadAsync(buffer, 0, chunkSize);
            if (bytesRead == 0) break;
            
            var chunk = new byte[bytesRead];
            Array.Copy(buffer, chunk, bytesRead);
            
            var currentIndex = chunkIndex++;
            var uploadTask = Task.Run(async () =>
            {
                var chunkId = await UploadChunkAsync(uploadId, currentIndex, chunk);
                chunks.Add(new ChunkInfo { Index = currentIndex, Id = chunkId });
            });
            
            uploadTasks.Add(uploadTask);
            
            // 동시 업로드 수 제한
            if (uploadTasks.Count >= 4)
            {
                await Task.WhenAny(uploadTasks);
                uploadTasks.RemoveAll(t => t.IsCompleted);
            }
        }
        
        await Task.WhenAll(uploadTasks);
        
        // 청크 병합
        return await MergeChunksAsync(uploadId, chunks.OrderBy(c => c.Index));
    }
}
```

## 모범 사례

### 파일 업로드 구성
```csharp
// Program.cs
builder.Services.Configure<FormOptions>(options =>
{
    options.ValueLengthLimit = int.MaxValue;
    options.MultipartBodyLengthLimit = int.MaxValue;
    options.MultipartHeadersLengthLimit = int.MaxValue;
});

builder.Services.Configure<IISServerOptions>(options =>
{
    options.MaxRequestBodySize = int.MaxValue;
});

// Kestrel 구성
builder.WebHost.ConfigureKestrel(serverOptions =>
{
    serverOptions.Limits.MaxRequestBodySize = int.MaxValue;
    serverOptions.Limits.MaxRequestLineSize = 8192;
    serverOptions.Limits.MaxRequestHeadersTotalSize = 32768;
});

// 파일 업로드 옵션
builder.Services.Configure<FileUploadOptions>(options =>
{
    options.MaxFileSize = 100 * 1024 * 1024; // 100MB
    options.AllowedExtensions = new[] { ".jpg", ".jpeg", ".png", ".gif", ".pdf", ".doc", ".docx" };
    options.StoragePath = Path.Combine(builder.Environment.ContentRootPath, "uploads");
    options.EnableVirusScan = true;
    options.EnableImageOptimization = true;
});
```

파일 업로드와 스트리밍은 웹 애플리케이션의 중요한 기능입니다. 보안, 성능, 사용성을 모두 고려하여 구현해야 합니다.