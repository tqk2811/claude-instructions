# Đọc log qua HTTP bằng header token (ASP.NET)

Dùng khi cần xem log của host đã deploy (IIS, VPS) mà không vào được máy chủ: mỗi host ghi log ra tệp theo ngày,
và mở hai endpoint chỉ đọc, bảo vệ bằng một khoá bí mật trong header. Đặt code trong project Shared để mọi host
(API, web, admin) dùng chung.

## Hợp đồng

- `GET /_logs`: liệt kê tệp `*.log` (tên, kích thước, giờ ghi cuối), mới nhất trước.
- `GET /_logs/{file}?tail=N`: trả N dòng cuối dạng `text/plain` (mặc định 500, kẹp 1..20000).
- Header `X-Log-Key` phải khớp cấu hình `LogReader:Key` của host đó.
- **Khoá cấu hình rỗng, thiếu header hoặc sai khoá đều trả 404**, không 401/403, để không lộ endpoint tồn tại.
  Không cấu hình khoá là endpoint coi như tắt.
- Endpoint gắn `.AllowAnonymous()`: nó tự kiểm khoá, không đi qua cookie/JWT của app (nếu app có fallback policy
  bắt đăng nhập thì thiếu dòng này là endpoint luôn bị chặn).

## Ghi log ra tệp (log4net)

- Mỗi host một thư mục `..\Logs\<tên project>\`, tệp `yyyy-MM-dd.log`, `RollingFileAppender` theo ngày, `utf-8`,
  `MinimalLock` (tiến trình khác vẫn mở đọc được), giới hạn kích thước/số bản.
- Đặt thư mục log NGOÀI thư mục publish (`..\Logs`) để deploy đè thư mục site không xoá log.
- `builder.Logging.AddLog4Net();` (gói `Microsoft.Extensions.Logging.Log4Net.AspNetCore`), giữ Console/Debug cho test.
- log4net tính đường dẫn tương đối từ `AppContext.BaseDirectory`, KHÔNG phải ContentRoot. Endpoint đọc log phải tính
  cùng gốc đó: `Path.GetFullPath(Path.Combine(AppContext.BaseDirectory, "..", "Logs", env.ApplicationName))`.
  Tên thư mục trong `log4net.config` phải trùng `ApplicationName` (tên assembly).
- IIS: tài khoản app pool phải có quyền ghi thư mục `..\Logs`.

```xml
<log4net>
  <appender name="RollingLogFileAppender" type="log4net.Appender.RollingFileAppender">
    <lockingModel type="log4net.Appender.FileAppender+MinimalLock"/>
    <file value="..\Logs\MyApp.Api\" />
    <datePattern value="yyyy-MM-dd'.log'"/>
    <staticLogFileName value="false"/>
    <encoding value="utf-8"/>
    <appendToFile value="true"/>
    <rollingStyle value="Date"/>
    <maxSizeRollBackups value="100"/>
    <maximumFileSize value="15MB"/>
    <layout type="log4net.Layout.PatternLayout">
      <conversionPattern value="[%date] [%5level] [%logger] [%thread] - %message %newline %exception" />
    </layout>
  </appender>
  <root>
    <level value="INFO"/>
    <appender-ref ref="RollingLogFileAppender"/>
  </root>
</log4net>
```

## Endpoint

```csharp
public static class LogReaderEndpointExtensions
{
    public const string KeyHeader = "X-Log-Key";
    public const string KeyConfigKey = "LogReader:Key";
    public const string RoutePrefix = "/_logs";
    private const int DefaultTail = 500;
    private const int MaxTail = 20000;

    public static IEndpointRouteBuilder MapLogReader(this IEndpointRouteBuilder endpoints)
    {
        var configuration = endpoints.ServiceProvider.GetRequiredService<IConfiguration>();
        var environment = endpoints.ServiceProvider.GetRequiredService<IWebHostEnvironment>();
        // log4net tính đường dẫn tương đối từ thư mục của AppDomain, không phải ContentRoot.
        var logDirectory = Path.GetFullPath(Path.Combine(AppContext.BaseDirectory, "..", "Logs", environment.ApplicationName));

        endpoints.MapGet(RoutePrefix, (HttpContext context) =>
        {
            if (!IsAuthorized(context, configuration)) return Results.NotFound();
            if (!Directory.Exists(logDirectory)) return Results.Ok(Array.Empty<object>());
            var files = new DirectoryInfo(logDirectory).GetFiles("*.log")
                .OrderByDescending(x => x.LastWriteTimeUtc)
                .Select(x => new { name = x.Name, size = x.Length, lastWriteUtc = x.LastWriteTimeUtc })
                .ToList();
            return Results.Ok(files);
        }).AllowAnonymous();

        endpoints.MapGet(RoutePrefix + "/{file}", (HttpContext context, string file, int? tail) =>
        {
            if (!IsAuthorized(context, configuration)) return Results.NotFound();
            // Chỉ nhận tên tệp trần *.log nằm ngay trong thư mục log (chặn "..", đường dẫn tuyệt đối).
            if (Path.GetFileName(file) != file || !file.EndsWith(".log", StringComparison.OrdinalIgnoreCase)) return Results.NotFound();
            var path = Path.Combine(logDirectory, file);
            if (!File.Exists(path)) return Results.NotFound();
            var lineCount = Math.Clamp(tail ?? DefaultTail, 1, MaxTail);
            // Tệp có thể bị xoá giữa File.Exists và lúc mở.
            try { return Results.Text(ReadTail(path, lineCount), "text/plain; charset=utf-8"); }
            catch (FileNotFoundException) { return Results.NotFound(); }
        }).AllowAnonymous();

        return endpoints;
    }

    private static bool IsAuthorized(HttpContext context, IConfiguration configuration)
    {
        var expected = configuration[KeyConfigKey];
        if (string.IsNullOrEmpty(expected)) return false;
        var actual = context.Request.Headers[KeyHeader].ToString();
        // So hash hai bên: FixedTimeEquals trả false ngay khi khác độ dài, so thẳng chuỗi là lộ độ dài khoá.
        return CryptographicOperations.FixedTimeEquals(
            SHA256.HashData(Encoding.UTF8.GetBytes(actual)),
            SHA256.HashData(Encoding.UTF8.GetBytes(expected)));
    }

    private static string ReadTail(string path, int lineCount)
    {
        // FileShare.ReadWrite: log4net (MinimalLock) vẫn ghi tiếp được trong lúc đọc.
        using var stream = new FileStream(path, FileMode.Open, FileAccess.Read, FileShare.ReadWrite | FileShare.Delete);
        using var reader = new StreamReader(stream, Encoding.UTF8);
        var lines = new Queue<string>(lineCount);
        string? line;
        while ((line = reader.ReadLine()) != null)
        {
            if (lines.Count == lineCount) lines.Dequeue();
            lines.Enqueue(line);
        }
        return string.Join(Environment.NewLine, lines);
    }
}
```

Gắn vào từng host: `app.MapLogReader();` (sau `UseRouting`/`UseAuthorization` như các endpoint khác).

## Cấu hình và dùng

- `appsettings.json` của từng host: `"LogReader": { "Key": "<chuỗi ngẫu nhiên >= 32 byte>" }`
  (sinh bằng `openssl rand -hex 32`). Mỗi host một khoá riêng thì lộ một khoá chỉ lộ log một host.
- Ghi URL các host và cách đọc vào memory local của project để lần sau tra nhanh.
- Đọc:
  ```bash
  curl -s -H "X-Log-Key: <key>" https://api.example.com/_logs
  curl -s -H "X-Log-Key: <key>" "https://api.example.com/_logs/2026-01-31.log?tail=300"
  ```
  Tìm lỗi: thêm `| grep -n -A5 ERROR`. Log dài thì ghi ra tệp trong scratchpad rồi mới lọc, đừng in hết vào phiên.

## Lưu ý

- Log có thể chứa dữ liệu cá nhân, token, câu SQL: khoá phải mạnh, chỉ đi qua HTTPS, không dán khoá vào chat công khai.
- Không mở endpoint ghi/xoá log qua HTTP.
- Đọc cả tệp để lấy N dòng cuối là O(kích thước tệp); tệp đã giới hạn 15MB nên chấp nhận được. Tệp lớn hơn thì đọc
  ngược từ cuối.
- Test: thiếu cấu hình / sai khoá / thiếu header → 404; `file` chứa `..` hoặc không đuôi `.log` → 404; tail kẹp đúng.
