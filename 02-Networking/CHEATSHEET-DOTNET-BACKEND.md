# Networking Cheat Sheet for .NET Backend Engineers

## Quick Reference - HTTP in .NET

### HttpClient Best Practices
```csharp
// WRONG - Creates socket exhaustion
var client = new HttpClient();
var result = await client.GetAsync(url);

// RIGHT - Use IHttpClientFactory
public class MyService
{
    private readonly HttpClient _client;

    public MyService(IHttpClientFactory factory)
    {
        _client = factory.CreateClient("MyApi");
    }
}

// Registration in Program.cs
builder.Services.AddHttpClient("MyApi", client =>
{
    client.BaseAddress = new Uri("https://api.example.com");
    client.Timeout = TimeSpan.FromSeconds(30);
    client.DefaultRequestHeaders.Add("User-Agent", "MyApp/1.0");
});
```

### Configuring HttpClient
```csharp
// With Polly for resilience
builder.Services.AddHttpClient("MyApi")
    .AddTransientHttpErrorPolicy(p =>
        p.WaitAndRetryAsync(3, _ => TimeSpan.FromMilliseconds(300)))
    .AddTransientHttpErrorPolicy(p =>
        p.CircuitBreakerAsync(5, TimeSpan.FromSeconds(30)));

// With custom handler
builder.Services.AddHttpClient("MyApi")
    .ConfigurePrimaryHttpMessageHandler(() => new HttpClientHandler
    {
        MaxConnectionsPerServer = 10,
        UseProxy = false
    });
```

### Making HTTP Requests
```csharp
// GET request
var response = await _client.GetAsync("/api/users");
var users = await response.Content.ReadFromJsonAsync<List<User>>();

// POST request
var user = new User { Name = "John" };
var response = await _client.PostAsJsonAsync("/api/users", user);

// With headers
var request = new HttpRequestMessage(HttpMethod.Get, "/api/data");
request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", token);
var response = await _client.SendAsync(request);
```

---

## Kestrel Configuration

### Basic Setup
```csharp
// Program.cs
builder.WebHost.ConfigureKestrel(options =>
{
    options.ListenAnyIP(5000); // HTTP
    options.ListenAnyIP(5001, listenOptions =>
    {
        listenOptions.UseHttps("certificate.pfx", "password");
    });
});
```

### appsettings.json Configuration
```json
{
  "Kestrel": {
    "Endpoints": {
      "Http": {
        "Url": "http://0.0.0.0:5000"
      },
      "Https": {
        "Url": "https://0.0.0.0:5001",
        "Certificate": {
          "Path": "certificate.pfx",
          "Password": "password"
        }
      }
    },
    "Limits": {
      "MaxConcurrentConnections": 100,
      "MaxRequestBodySize": 10485760,
      "RequestHeadersTimeout": "00:00:30"
    }
  }
}
```

### URL Bindings
```bash
# Via command line
dotnet run --urls "http://0.0.0.0:5000;https://0.0.0.0:5001"

# Via environment variable
export ASPNETCORE_URLS="http://+:5000;https://+:5001"
```

---

## TCP/UDP in .NET

### TCP Server
```csharp
var listener = new TcpListener(IPAddress.Any, 8080);
listener.Start();

while (true)
{
    var client = await listener.AcceptTcpClientAsync();
    _ = HandleClientAsync(client);
}

async Task HandleClientAsync(TcpClient client)
{
    using var stream = client.GetStream();
    var buffer = new byte[1024];
    int bytesRead = await stream.ReadAsync(buffer);
    // Process data
    await stream.WriteAsync(response);
    client.Close();
}
```

### TCP Client
```csharp
using var client = new TcpClient();
await client.ConnectAsync("localhost", 8080);
using var stream = client.GetStream();

await stream.WriteAsync(Encoding.UTF8.GetBytes("Hello"));
var buffer = new byte[1024];
int bytesRead = await stream.ReadAsync(buffer);
var response = Encoding.UTF8.GetString(buffer, 0, bytesRead);
```

### UDP
```csharp
// Server
using var udp = new UdpClient(8080);
var result = await udp.ReceiveAsync();
var message = Encoding.UTF8.GetString(result.Buffer);

// Client
using var udp = new UdpClient();
var data = Encoding.UTF8.GetBytes("Hello");
await udp.SendAsync(data, data.Length, "localhost", 8080);
```

---

## SignalR (Real-time Communication)

### Hub Definition
```csharp
public class ChatHub : Hub
{
    public async Task SendMessage(string user, string message)
    {
        await Clients.All.SendAsync("ReceiveMessage", user, message);
    }

    public async Task JoinGroup(string groupName)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, groupName);
    }
}
```

### Configuration
```csharp
// Program.cs
builder.Services.AddSignalR()
    .AddJsonProtocol(options =>
    {
        options.PayloadSerializerOptions.PropertyNamingPolicy = null;
    });

app.MapHub<ChatHub>("/chatHub");
```

---

## gRPC in .NET

### Proto Definition
```protobuf
syntax = "proto3";

service Greeter {
    rpc SayHello (HelloRequest) returns (HelloReply);
}

message HelloRequest { string name = 1; }
message HelloReply { string message = 1; }
```

### Service Implementation
```csharp
public class GreeterService : Greeter.GreeterBase
{
    public override Task<HelloReply> SayHello(HelloRequest request,
        ServerCallContext context)
    {
        return Task.FromResult(new HelloReply
        {
            Message = $"Hello {request.Name}"
        });
    }
}
```

### Client
```csharp
var channel = GrpcChannel.ForAddress("https://localhost:5001");
var client = new Greeter.GreeterClient(channel);
var reply = await client.SayHelloAsync(new HelloRequest { Name = "World" });
```

---

## Interview Q&A

### Q1: What is socket exhaustion and how do you prevent it?
**A:** When HttpClient instances aren't reused, sockets remain in TIME_WAIT state, eventually exhausting available ports.
Prevention:
- Use `IHttpClientFactory` (manages connection pooling)
- Reuse static HttpClient instances (with DNS refresh handling)
- Never create HttpClient in using blocks for repeated calls

### Q2: How does Kestrel handle concurrent connections?
**A:** Kestrel uses async I/O with libuv/SocketsHttpHandler. It can handle thousands of concurrent connections with minimal threads. Configure limits via `MaxConcurrentConnections` and connection middleware.

### Q3: What is the difference between HTTP/1.1, HTTP/2, and HTTP/3?
**A:**
- **HTTP/1.1**: Text-based, one request per connection (pipelining rarely used)
- **HTTP/2**: Binary, multiplexed streams over single connection, header compression
- **HTTP/3**: Uses QUIC (UDP-based), faster connection establishment, better for mobile

### Q4: How do you implement retry logic for HTTP calls?
**A:**
```csharp
// Using Polly
builder.Services.AddHttpClient("MyApi")
    .AddTransientHttpErrorPolicy(p =>
        p.WaitAndRetryAsync(3,
            retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt))));
```

### Q5: What ports does a .NET application need?
**A:**
- Development: 5000 (HTTP), 5001 (HTTPS)
- Production: 80/443 via reverse proxy
- gRPC: Typically 5001 or custom
- SignalR: Same as web (WebSocket upgrade on 80/443)

### Q6: How do you configure CORS in .NET?
**A:**
```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowSpecific", policy =>
    {
        policy.WithOrigins("https://example.com")
              .AllowAnyMethod()
              .AllowAnyHeader()
              .AllowCredentials();
    });
});

app.UseCors("AllowSpecific");
```

### Q7: How do you handle timeouts in HTTP calls?
**A:**
```csharp
// Client-side timeout
builder.Services.AddHttpClient("MyApi", client =>
{
    client.Timeout = TimeSpan.FromSeconds(30);
});

// Per-request timeout with CancellationToken
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(10));
var response = await client.GetAsync(url, cts.Token);
```

### Q8: What is the X-Forwarded-For header and why is it important?
**A:** When behind a reverse proxy, the original client IP is lost. `X-Forwarded-For` contains the original IP chain.
```csharp
// Configure forwarded headers
builder.Services.Configure<ForwardedHeadersOptions>(options =>
{
    options.ForwardedHeaders = ForwardedHeaders.XForwardedFor |
                               ForwardedHeaders.XForwardedProto;
});
app.UseForwardedHeaders();
```

---

## Network Diagnostics in .NET

### DNS Resolution
```csharp
var addresses = await Dns.GetHostAddressesAsync("example.com");
foreach (var ip in addresses)
    Console.WriteLine(ip);
```

### Check Port Availability
```csharp
bool IsPortOpen(string host, int port, int timeout = 1000)
{
    try
    {
        using var client = new TcpClient();
        var result = client.BeginConnect(host, port, null, null);
        bool success = result.AsyncWaitHandle.WaitOne(timeout);
        client.EndConnect(result);
        return success;
    }
    catch { return false; }
}
```

### Network Interface Info
```csharp
foreach (var ni in NetworkInterface.GetAllNetworkInterfaces())
{
    Console.WriteLine($"{ni.Name}: {ni.OperationalStatus}");
    var props = ni.GetIPProperties();
    foreach (var addr in props.UnicastAddresses)
        Console.WriteLine($"  {addr.Address}");
}
```

---

## Best Practices

1. **Always use IHttpClientFactory** - Prevents socket exhaustion
2. **Configure timeouts** - Both connection and request timeouts
3. **Implement retry with backoff** - Use Polly for transient errors
4. **Use circuit breakers** - Prevent cascading failures
5. **Enable HTTP/2** - Better performance for multiple requests
6. **Configure CORS properly** - Don't use AllowAnyOrigin with credentials
7. **Handle forwarded headers** - When behind proxy/load balancer
8. **Log network calls** - Include timing, status codes, errors
