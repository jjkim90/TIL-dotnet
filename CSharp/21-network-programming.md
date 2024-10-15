# 네트워크 프로그래밍

## 네트워크 프로그래밍 기초

### 인터넷의 유래
- 1969년 ARPANET에서 시작
- TCP/IP 프로토콜의 표준화
- World Wide Web의 등장으로 대중화

### TCP/IP 스택
```
응용 계층 (Application Layer) - HTTP, FTP, SMTP
전송 계층 (Transport Layer) - TCP, UDP  
인터넷 계층 (Internet Layer) - IP
네트워크 인터페이스 계층 (Network Interface) - Ethernet, WiFi
```

### IP 주소
```csharp
// IP 주소 다루기
IPAddress ipAddress = IPAddress.Parse("192.168.1.1");
IPAddress localHost = IPAddress.Loopback;  // 127.0.0.1
IPAddress any = IPAddress.Any;  // 0.0.0.0

// 호스트 이름으로 IP 주소 얻기
IPHostEntry hostEntry = Dns.GetHostEntry("www.example.com");
foreach (IPAddress ip in hostEntry.AddressList)
{
    Console.WriteLine($"IP: {ip}");
}

// 로컬 컴퓨터의 IP 주소들
string hostName = Dns.GetHostName();
IPHostEntry localEntry = Dns.GetHostEntry(hostName);
foreach (IPAddress ip in localEntry.AddressList)
{
    if (ip.AddressFamily == AddressFamily.InterNetwork)  // IPv4
    {
        Console.WriteLine($"IPv4: {ip}");
    }
}
```

### 포트
```csharp
// 잘 알려진 포트
// HTTP: 80
// HTTPS: 443
// FTP: 21
// SSH: 22
// SMTP: 25

// IPEndPoint - IP 주소와 포트 조합
IPEndPoint endpoint = new IPEndPoint(IPAddress.Parse("192.168.1.1"), 8080);
Console.WriteLine($"Endpoint: {endpoint}");  // 192.168.1.1:8080
```

### TCP/IP 동작 과정
1. **연결 설정** (3-way handshake)
   - SYN → SYN-ACK → ACK
2. **데이터 전송**
   - 신뢰성 있는 순서대로 전송
3. **연결 종료** (4-way handshake)
   - FIN → ACK → FIN → ACK

## TcpListener와 TcpClient

### TCP 서버 (TcpListener)
```csharp
public class TcpServer
{
    private TcpListener listener;
    private bool isRunning;
    
    public async Task StartAsync(int port)
    {
        listener = new TcpListener(IPAddress.Any, port);
        listener.Start();
        isRunning = true;
        
        Console.WriteLine($"서버가 포트 {port}에서 시작되었습니다.");
        
        while (isRunning)
        {
            try
            {
                TcpClient client = await listener.AcceptTcpClientAsync();
                Console.WriteLine($"클라이언트 연결됨: {client.Client.RemoteEndPoint}");
                
                // 클라이언트 처리 (비동기)
                _ = Task.Run(() => HandleClientAsync(client));
            }
            catch (ObjectDisposedException)
            {
                break;  // 서버 종료
            }
        }
    }
    
    private async Task HandleClientAsync(TcpClient client)
    {
        try
        {
            using (client)
            using (NetworkStream stream = client.GetStream())
            {
                byte[] buffer = new byte[1024];
                
                while (client.Connected)
                {
                    int bytesRead = await stream.ReadAsync(buffer, 0, buffer.Length);
                    if (bytesRead == 0) break;  // 연결 종료
                    
                    string message = Encoding.UTF8.GetString(buffer, 0, bytesRead);
                    Console.WriteLine($"받은 메시지: {message}");
                    
                    // 에코 서버 - 받은 메시지를 그대로 전송
                    byte[] response = Encoding.UTF8.GetBytes($"Echo: {message}");
                    await stream.WriteAsync(response, 0, response.Length);
                }
            }
        }
        catch (Exception ex)
        {
            Console.WriteLine($"클라이언트 처리 오류: {ex.Message}");
        }
        
        Console.WriteLine("클라이언트 연결 종료");
    }
    
    public void Stop()
    {
        isRunning = false;
        listener?.Stop();
    }
}
```

### TCP 클라이언트 (TcpClient)
```csharp
public class TcpClientExample
{
    public async Task ConnectAsync(string host, int port)
    {
        try
        {
            using (TcpClient client = new TcpClient())
            {
                await client.ConnectAsync(host, port);
                Console.WriteLine($"서버 {host}:{port}에 연결됨");
                
                NetworkStream stream = client.GetStream();
                
                // 메시지 전송
                string message = "Hello, Server!";
                byte[] data = Encoding.UTF8.GetBytes(message);
                await stream.WriteAsync(data, 0, data.Length);
                
                // 응답 받기
                byte[] buffer = new byte[1024];
                int bytesRead = await stream.ReadAsync(buffer, 0, buffer.Length);
                string response = Encoding.UTF8.GetString(buffer, 0, bytesRead);
                Console.WriteLine($"서버 응답: {response}");
            }
        }
        catch (SocketException ex)
        {
            Console.WriteLine($"소켓 오류: {ex.Message}");
        }
    }
}
```

## 흐르는 패킷

### StreamReader/Writer 사용
```csharp
public class StreamBasedServer
{
    public async Task HandleClientWithStreamAsync(TcpClient client)
    {
        using (NetworkStream netStream = client.GetStream())
        using (StreamReader reader = new StreamReader(netStream, Encoding.UTF8))
        using (StreamWriter writer = new StreamWriter(netStream, Encoding.UTF8))
        {
            writer.AutoFlush = true;  // 자동 플러시
            
            while (client.Connected)
            {
                string line = await reader.ReadLineAsync();
                if (line == null) break;
                
                Console.WriteLine($"받음: {line}");
                
                // 처리 후 응답
                await writer.WriteLineAsync($"처리됨: {line}");
            }
        }
    }
}
```

### 바이너리 데이터 전송
```csharp
public class BinaryProtocol
{
    public async Task SendMessageAsync(NetworkStream stream, byte[] data)
    {
        // 먼저 데이터 길이 전송 (4바이트)
        byte[] lengthBytes = BitConverter.GetBytes(data.Length);
        await stream.WriteAsync(lengthBytes, 0, 4);
        
        // 실제 데이터 전송
        await stream.WriteAsync(data, 0, data.Length);
    }
    
    public async Task<byte[]> ReceiveMessageAsync(NetworkStream stream)
    {
        // 길이 읽기
        byte[] lengthBytes = new byte[4];
        await stream.ReadAsync(lengthBytes, 0, 4);
        int length = BitConverter.ToInt32(lengthBytes, 0);
        
        // 데이터 읽기
        byte[] data = new byte[length];
        int totalRead = 0;
        
        while (totalRead < length)
        {
            int read = await stream.ReadAsync(data, totalRead, length - totalRead);
            if (read == 0) throw new EndOfStreamException();
            totalRead += read;
        }
        
        return data;
    }
}
```

## 프로토콜 설계

### 메시지 프로토콜 정의
```csharp
// 메시지 타입
public enum MessageType : byte
{
    Connect = 0x01,
    Disconnect = 0x02,
    Data = 0x03,
    FileTransfer = 0x04,
    Heartbeat = 0x05
}

// 메시지 구조
public class Message
{
    public MessageType Type { get; set; }
    public int Length { get; set; }
    public byte[] Data { get; set; }
    
    public byte[] ToBytes()
    {
        using (MemoryStream ms = new MemoryStream())
        using (BinaryWriter writer = new BinaryWriter(ms))
        {
            writer.Write((byte)Type);
            writer.Write(Length);
            writer.Write(Data);
            return ms.ToArray();
        }
    }
    
    public static Message FromBytes(byte[] bytes)
    {
        using (MemoryStream ms = new MemoryStream(bytes))
        using (BinaryReader reader = new BinaryReader(ms))
        {
            return new Message
            {
                Type = (MessageType)reader.ReadByte(),
                Length = reader.ReadInt32(),
                Data = reader.ReadBytes(reader.ReadInt32())
            };
        }
    }
}
```

## 파일 전송 프로토콜

### 파일 전송 서버
```csharp
public class FileTransferServer
{
    private readonly string saveDirectory;
    
    public FileTransferServer(string saveDirectory)
    {
        this.saveDirectory = saveDirectory;
        Directory.CreateDirectory(saveDirectory);
    }
    
    public async Task HandleFileUploadAsync(NetworkStream stream)
    {
        using (BinaryReader reader = new BinaryReader(stream))
        using (BinaryWriter writer = new BinaryWriter(stream))
        {
            // 파일 정보 읽기
            string fileName = reader.ReadString();
            long fileSize = reader.ReadInt64();
            
            Console.WriteLine($"파일 수신 시작: {fileName} ({fileSize:N0} bytes)");
            
            // 파일 저장
            string filePath = Path.Combine(saveDirectory, fileName);
            using (FileStream fileStream = new FileStream(filePath, FileMode.Create))
            {
                byte[] buffer = new byte[8192];
                long totalReceived = 0;
                
                while (totalReceived < fileSize)
                {
                    int toRead = (int)Math.Min(buffer.Length, fileSize - totalReceived);
                    int read = reader.Read(buffer, 0, toRead);
                    
                    if (read == 0) throw new EndOfStreamException();
                    
                    await fileStream.WriteAsync(buffer, 0, read);
                    totalReceived += read;
                    
                    // 진행률 전송
                    int progress = (int)((totalReceived * 100) / fileSize);
                    writer.Write(progress);
                    writer.Flush();
                }
            }
            
            Console.WriteLine($"파일 수신 완료: {fileName}");
            
            // 완료 응답
            writer.Write(true);  // 성공
            writer.Write($"파일 '{fileName}' 업로드 완료");
        }
    }
}
```

### 파일 전송 클라이언트
```csharp
public class FileTransferClient
{
    public async Task UploadFileAsync(string serverHost, int port, string filePath)
    {
        FileInfo fileInfo = new FileInfo(filePath);
        
        using (TcpClient client = new TcpClient())
        {
            await client.ConnectAsync(serverHost, port);
            
            using (NetworkStream stream = client.GetStream())
            using (BinaryWriter writer = new BinaryWriter(stream))
            using (BinaryReader reader = new BinaryReader(stream))
            {
                // 파일 정보 전송
                writer.Write(fileInfo.Name);
                writer.Write(fileInfo.Length);
                
                // 파일 데이터 전송
                using (FileStream fileStream = File.OpenRead(filePath))
                {
                    byte[] buffer = new byte[8192];
                    int read;
                    
                    while ((read = await fileStream.ReadAsync(buffer, 0, buffer.Length)) > 0)
                    {
                        writer.Write(buffer, 0, read);
                        writer.Flush();
                        
                        // 진행률 수신
                        int progress = reader.ReadInt32();
                        Console.WriteLine($"업로드 진행률: {progress}%");
                    }
                }
                
                // 완료 응답 수신
                bool success = reader.ReadBoolean();
                string message = reader.ReadString();
                Console.WriteLine($"서버 응답: {message}");
            }
        }
    }
}
```

## UDP 통신

### UDP 서버
```csharp
public class UdpServer
{
    private UdpClient udpClient;
    private bool isRunning;
    
    public async Task StartAsync(int port)
    {
        udpClient = new UdpClient(port);
        isRunning = true;
        
        Console.WriteLine($"UDP 서버가 포트 {port}에서 시작되었습니다.");
        
        while (isRunning)
        {
            try
            {
                UdpReceiveResult result = await udpClient.ReceiveAsync();
                string message = Encoding.UTF8.GetString(result.Buffer);
                
                Console.WriteLine($"받음 [{result.RemoteEndPoint}]: {message}");
                
                // 응답 전송
                byte[] response = Encoding.UTF8.GetBytes($"Echo: {message}");
                await udpClient.SendAsync(response, response.Length, result.RemoteEndPoint);
            }
            catch (ObjectDisposedException)
            {
                break;
            }
        }
    }
    
    public void Stop()
    {
        isRunning = false;
        udpClient?.Close();
    }
}
```

### UDP 클라이언트
```csharp
public class UdpClientExample
{
    public async Task SendMessageAsync(string host, int port, string message)
    {
        using (UdpClient client = new UdpClient())
        {
            byte[] data = Encoding.UTF8.GetBytes(message);
            
            // 메시지 전송
            await client.SendAsync(data, data.Length, host, port);
            Console.WriteLine($"전송: {message}");
            
            // 응답 수신 (타임아웃 설정)
            client.Client.ReceiveTimeout = 5000;  // 5초
            
            try
            {
                UdpReceiveResult result = await client.ReceiveAsync();
                string response = Encoding.UTF8.GetString(result.Buffer);
                Console.WriteLine($"응답: {response}");
            }
            catch (SocketException)
            {
                Console.WriteLine("응답 타임아웃");
            }
        }
    }
}
```

## HTTP 통신

### HttpClient 사용
```csharp
public class HttpClientExample
{
    private readonly HttpClient httpClient;
    
    public HttpClientExample()
    {
        httpClient = new HttpClient();
        httpClient.DefaultRequestHeaders.Add("User-Agent", "MyApp/1.0");
    }
    
    public async Task<string> GetAsync(string url)
    {
        try
        {
            HttpResponseMessage response = await httpClient.GetAsync(url);
            response.EnsureSuccessStatusCode();
            
            string content = await response.Content.ReadAsStringAsync();
            return content;
        }
        catch (HttpRequestException ex)
        {
            Console.WriteLine($"HTTP 오류: {ex.Message}");
            return null;
        }
    }
    
    public async Task<string> PostJsonAsync(string url, object data)
    {
        string json = JsonSerializer.Serialize(data);
        StringContent content = new StringContent(json, Encoding.UTF8, "application/json");
        
        HttpResponseMessage response = await httpClient.PostAsync(url, content);
        response.EnsureSuccessStatusCode();
        
        return await response.Content.ReadAsStringAsync();
    }
    
    public async Task DownloadFileAsync(string url, string filePath)
    {
        using (HttpResponseMessage response = await httpClient.GetAsync(url))
        {
            response.EnsureSuccessStatusCode();
            
            using (Stream contentStream = await response.Content.ReadAsStreamAsync())
            using (FileStream fileStream = new FileStream(filePath, FileMode.Create))
            {
                await contentStream.CopyToAsync(fileStream);
            }
        }
    }
}
```

### WebSocket 통신
```csharp
public class WebSocketClient
{
    private ClientWebSocket webSocket;
    
    public async Task ConnectAsync(string uri)
    {
        webSocket = new ClientWebSocket();
        await webSocket.ConnectAsync(new Uri(uri), CancellationToken.None);
        Console.WriteLine("WebSocket 연결됨");
        
        // 수신 태스크 시작
        _ = Task.Run(ReceiveLoop);
    }
    
    public async Task SendMessageAsync(string message)
    {
        byte[] buffer = Encoding.UTF8.GetBytes(message);
        await webSocket.SendAsync(
            new ArraySegment<byte>(buffer),
            WebSocketMessageType.Text,
            true,
            CancellationToken.None);
    }
    
    private async Task ReceiveLoop()
    {
        byte[] buffer = new byte[1024];
        
        while (webSocket.State == WebSocketState.Open)
        {
            try
            {
                WebSocketReceiveResult result = await webSocket.ReceiveAsync(
                    new ArraySegment<byte>(buffer),
                    CancellationToken.None);
                
                if (result.MessageType == WebSocketMessageType.Text)
                {
                    string message = Encoding.UTF8.GetString(buffer, 0, result.Count);
                    Console.WriteLine($"수신: {message}");
                }
                else if (result.MessageType == WebSocketMessageType.Close)
                {
                    await webSocket.CloseAsync(
                        WebSocketCloseStatus.NormalClosure,
                        "Closing",
                        CancellationToken.None);
                }
            }
            catch (Exception ex)
            {
                Console.WriteLine($"수신 오류: {ex.Message}");
                break;
            }
        }
    }
}
```

## 실전 예제: 채팅 서버

```csharp
public class ChatServer
{
    private readonly List<ClientHandler> clients = new List<ClientHandler>();
    private readonly object lockObj = new object();
    
    public async Task StartAsync(int port)
    {
        TcpListener listener = new TcpListener(IPAddress.Any, port);
        listener.Start();
        
        Console.WriteLine($"채팅 서버가 포트 {port}에서 시작되었습니다.");
        
        while (true)
        {
            TcpClient tcpClient = await listener.AcceptTcpClientAsync();
            ClientHandler client = new ClientHandler(tcpClient, this);
            
            lock (lockObj)
            {
                clients.Add(client);
            }
            
            _ = Task.Run(client.HandleAsync);
        }
    }
    
    public void RemoveClient(ClientHandler client)
    {
        lock (lockObj)
        {
            clients.Remove(client);
        }
    }
    
    public async Task BroadcastAsync(string message, ClientHandler sender = null)
    {
        List<ClientHandler> clientsCopy;
        lock (lockObj)
        {
            clientsCopy = new List<ClientHandler>(clients);
        }
        
        foreach (var client in clientsCopy)
        {
            if (client != sender)
            {
                await client.SendMessageAsync(message);
            }
        }
    }
}

public class ClientHandler
{
    private readonly TcpClient tcpClient;
    private readonly ChatServer server;
    private StreamReader reader;
    private StreamWriter writer;
    private string userName;
    
    public ClientHandler(TcpClient tcpClient, ChatServer server)
    {
        this.tcpClient = tcpClient;
        this.server = server;
    }
    
    public async Task HandleAsync()
    {
        try
        {
            NetworkStream stream = tcpClient.GetStream();
            reader = new StreamReader(stream);
            writer = new StreamWriter(stream) { AutoFlush = true };
            
            // 사용자 이름 요청
            await writer.WriteLineAsync("이름을 입력하세요:");
            userName = await reader.ReadLineAsync();
            
            await server.BroadcastAsync($"{userName}님이 입장했습니다.", this);
            
            // 메시지 수신 루프
            string message;
            while ((message = await reader.ReadLineAsync()) != null)
            {
                await server.BroadcastAsync($"{userName}: {message}", this);
            }
        }
        catch (Exception ex)
        {
            Console.WriteLine($"클라이언트 오류: {ex.Message}");
        }
        finally
        {
            server.RemoveClient(this);
            await server.BroadcastAsync($"{userName}님이 퇴장했습니다.", this);
            tcpClient.Close();
        }
    }
    
    public async Task SendMessageAsync(string message)
    {
        try
        {
            await writer.WriteLineAsync(message);
        }
        catch
        {
            // 전송 실패 시 무시
        }
    }
}
```

## 핵심 개념 정리
- **TCP**: 신뢰성 있는 연결 지향 프로토콜
- **UDP**: 빠르지만 신뢰성 없는 프로토콜
- **소켓**: 네트워크 통신의 끝점
- **프로토콜**: 통신 규약과 메시지 형식
- **비동기 처리**: async/await로 효율적인 네트워크 프로그래밍