# Web (MVC/Razor) gọi API bằng JWT: giữ token trong cookie

Dùng khi một web ASP.NET (MVC/Razor Pages) không tự xác thực mà gọi một API riêng bằng access token + refresh
token (mô hình BFF: trình duyệt chỉ nói chuyện với web, web gọi API). Đây là cơ chế chuẩn cho các project như vậy.

## Cơ chế

- **Cookie là nơi duy nhất giữ token.** Sau khi API đăng nhập trả cặp token, web `SignInAsync` cookie auth với
  `AuthenticationProperties.StoreTokens(...)` chứa access token, refresh token, hạn access token (ISO `"o"`, UTC).
  Vé cookie được mã hoá bằng Data Protection, HttpOnly, nên JavaScript và người dùng không đọc được token.
- **Không có kho token phía server** (không file, không `IDistributedCache`, không lock theo phiên). Web chạy
  nhiều instance hay khởi động lại vẫn chạy, chỉ cần key ring Data Protection được persist và dùng chung
  (`PersistKeysToFileSystem` ra thư mục ngoài thư mục publish, `SetApplicationName` cố định).
- **Principal trong cookie tối giản**: chỉ mã phiên (session id của API). Tên, email, vai, quyền KHÔNG nằm trong
  cookie vì sẽ cũ; mỗi request hỏi API `me` (cache ngắn theo mã phiên) rồi thay `HttpContext.User`.
- **Gọi API**: một `DelegatingHandler` (hoặc lớp bọc client, scoped) đọc access token từ vé cookie
  (`AuthenticateAsync` một lần mỗi request, cache vào `HttpContext.Items`) rồi gắn `Authorization: Bearer`.
  - Access token sắp hết hạn thì làm mới trước khi gửi.
  - API trả 401 thì làm mới ĐÚNG MỘT lần rồi gửi lại (thân upload multipart/stream không đệm được thì không
    gửi lại).
  - Làm mới xong, nếu `!Response.HasStarted` thì `SignInAsync` lại cùng principal với properties chứa cặp token
    mới để ghi đè cookie. Response đã bắt đầu thì bỏ qua ghi cookie: lần sau sẽ tự làm mới lại.
  - Làm mới bị 401/403 thì đăng xuất (xoá cookie) và đưa về trang đăng nhập; lỗi tạm (5xx, mạng) thì để lỗi bay
    lên, KHÔNG đăng xuất.
  - Lượt gọi làm mới dùng `CancellationToken` riêng có timeout, không dùng token của request.
- **Đăng xuất**: gọi API thu hồi phiên bằng token trong cookie (cố gắng, lỗi thì bỏ qua) rồi `SignOutAsync`.
- **Hạn cookie** "ghi nhớ đăng nhập" không dài hơn hạn refresh token.

## Điều kiện bắt buộc phía API: refresh token KHÔNG dùng một lần

- Làm mới hợp lệ thì trả access token mới; refresh token cũ VẪN dùng được tới khi hết hạn hoặc phiên bị thu hồi
  (có thể trả lại chính nó, hoặc phát token mới mà không vô hiệu token cũ).
- Mỗi lần làm mới vẫn tra DB: tài khoản còn sống/không bị khoá, phiên chưa bị thu hồi, chưa hết hạn; kéo dài hạn
  phiên theo hoạt động (sliding). Lấy vai/quyền từ DB, không tin bản sao trong token cũ.
- Thu hồi phiên (đăng xuất, đổi mật khẩu, admin khoá) phải làm cả access token lẫn refresh token chết ngay: API
  kiểm phiên ở mỗi request chứ không chỉ kiểm chữ ký JWT.
- Access token chỉ mang id người dùng, mã phiên, `jti`, hạn. Tên, email, vai, quyền KHÔNG nằm trong token:
  `OnTokenValidated` tra từ DB rồi thay claim, nên đổi vai/quyền có hiệu lực mà không chờ token hết hạn.
- Phép tra phiên + vai + quyền chạy ở MỌI request nên phải **cache ngắn** (`IMemoryCache`, vài chục giây, khoá theo
  mã phiên, `MemoryCache` riêng). Thu hồi phiên, đổi vai/quyền/mật khẩu thì vô hiệu ngay (tăng một bộ đếm thế hệ
  toàn cục) thay vì chờ hết TTL.
- **Why**: cookie nằm ở trình duyệt nên không đồng bộ được. Nhiều AJAX song song cùng cầm refresh token cũ; nếu
  API xoay token kiểu dùng một lần (phát hiện dùng lại → 409 hoặc thu hồi phiên) thì request thua sẽ làm người
  dùng bị đăng xuất oan, và lock theo user/phiên phía server không cứu được vì cookie của request thua vẫn giữ
  token cũ. Muốn giữ xoay một lần thì phải có kho token phía server — tức bỏ cơ chế này.
- Đánh đổi chấp nhận: lộ refresh token thì kẻ trộm dùng được tới khi phiên bị thu hồi/hết hạn (không tự phát hiện
  dùng lại). Bù lại bằng: cookie HttpOnly + Secure + SameSite, danh sách phiên cho người dùng/admin tự thu hồi.

## Bẫy

- Hạn chờ cho lời gọi `me` phải tính cả lượt làm mới token và lúc API khởi động nguội (IIS ngủ sau 20 phút):
  hạn quá ngắn (vài giây) làm request chạy "không có quyền" thất thường. Cho app pool API `AlwaysRunning` +
  `idleTimeout = 0` + `preloadEnabled`.
- Huỷ lượt làm mới giữa chừng (dùng chung `CancellationToken` của request) khi API vẫn xoay token phía server là
  mất cặp token mới; với API không xoay một lần thì vô hại, nhưng vẫn nên tách token.

## Code mẫu

Tên lớp/khoá chỉ là ví dụ (`MyApp`, `IMyApi`); khuôn lỗi, client API typed theo project.

### Web: ghi cặp token vào cookie

```csharp
public static class WebAuth
{
    public const string Scheme = CookieAuthenticationDefaults.AuthenticationScheme;
    public const string SessionIdClaim = "myapp_sid";
    public const string AccessTokenName = "access_token";
    public const string RefreshTokenName = "refresh_token";
    public const string AccessExpiresName = "access_expires_at";

    /// <summary>Principal chỉ mang mã phiên; id/tên/vai/quyền lấy từ API mỗi request.</summary>
    public static ClaimsPrincipal SessionPrincipal(string sessionId)
        => new(new ClaimsIdentity([new Claim(SessionIdClaim, sessionId)], Scheme, ClaimTypes.Name, ClaimTypes.Role));

    public static void StoreTokens(AuthenticationProperties props, AuthTokensVM t)
        => props.StoreTokens(
        [
            new AuthenticationToken { Name = AccessTokenName, Value = t.AccessToken },
            new AuthenticationToken { Name = RefreshTokenName, Value = t.RefreshToken },
            new AuthenticationToken
            {
                Name = AccessExpiresName,
                Value = DateTime.SpecifyKind(t.AccessExpiresAt, DateTimeKind.Utc).ToString("o"),
            },
        ]);

    public static Task SignInAsync(HttpContext http, AuthTokensVM t, bool remember)
    {
        var props = new AuthenticationProperties
        {
            IsPersistent = remember,
            // Không sống lâu hơn refresh token.
            ExpiresUtc = remember ? new DateTimeOffset(DateTime.SpecifyKind(t.RefreshExpiresAt, DateTimeKind.Utc)) : null,
        };
        StoreTokens(props, t);
        return http.SignInAsync(Scheme, SessionPrincipal(t.SessionId), props);
    }
}
```

`Program.cs` của Web:

```csharp
builder.Services.AddDataProtection()
    .SetApplicationName("MyApp.Web")
    .PersistKeysToFileSystem(new DirectoryInfo(
        Path.Combine(builder.Environment.ContentRootPath, "..", "Data", "keys")));

builder.Services.AddAuthentication(WebAuth.Scheme).AddCookie(o =>
{
    o.Cookie.Name = "myapp_auth";
    o.Cookie.HttpOnly = true;
    o.Cookie.SecurePolicy = CookieSecurePolicy.Always;
    o.Cookie.SameSite = SameSiteMode.Lax;
    o.LoginPath = "/account/login";
});

builder.Services.AddHttpContextAccessor();
builder.Services.AddTransient<BearerFromCookieHandler>();
builder.Services.AddHttpClient<IMyApi, MyApiClient>(c => c.BaseAddress = new(apiUrl))
    .AddHttpMessageHandler<BearerFromCookieHandler>();
```

### Web: gắn bearer, làm mới khi cần, ghi lại cookie

```csharp
public sealed class BearerFromCookieHandler(IHttpContextAccessor accessor, IServiceProvider services) : DelegatingHandler
{
    private static readonly object TicketKey = new();
    private static readonly TimeSpan RefreshSkew = TimeSpan.FromSeconds(60);
    private static readonly TimeSpan RefreshTimeout = TimeSpan.FromSeconds(30);

    protected override async Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken ct)
    {
        var http = accessor.HttpContext;
        if (http is null || request.Headers.Authorization is not null || IsAnonymousAuthCall(request))
            return await base.SendAsync(request, ct);

        var ticket = await GetTicketAsync(http);
        if (ticket?.Properties.GetTokenValue(WebAuth.AccessTokenName) is not { } access)
            return await base.SendAsync(request, ct);   // khách

        if (IsExpiring(ticket.Properties))
            access = await RefreshAsync(http, ticket);

        // Thân upload (multipart/stream) không đệm được nên không gửi lại.
        var replayable = request.Content is not (MultipartContent or StreamContent);
        if (replayable && request.Content is not null)
            await request.Content.LoadIntoBufferAsync(ct);

        request.Headers.Authorization = new("Bearer", access);
        var response = await base.SendAsync(request, ct);
        if (response.StatusCode != HttpStatusCode.Unauthorized || !replayable)
            return response;

        response.Dispose();
        request.Headers.Authorization = new("Bearer", await RefreshAsync(http, ticket));
        return await base.SendAsync(request, ct);
    }

    /// <summary>Đọc vé cookie một lần mỗi request; bản sau khi làm mới cũng nằm ở đây.</summary>
    private static async Task<AuthenticationTicket?> GetTicketAsync(HttpContext http)
    {
        if (http.Items.TryGetValue(TicketKey, out var cached))
            return (AuthenticationTicket?)cached;
        var result = await http.AuthenticateAsync(WebAuth.Scheme);
        http.Items[TicketKey] = result.Ticket;
        return result.Ticket;
    }

    private async Task<string> RefreshAsync(HttpContext http, AuthenticationTicket ticket)
    {
        var refresh = ticket.Properties.GetTokenValue(WebAuth.RefreshTokenName)
            ?? throw new SessionExpiredException();

        // Không dùng ct của request: huỷ giữa chừng là mất cặp token mới.
        using var cts = new CancellationTokenSource(RefreshTimeout);
        AuthTokensVM tokens;
        try
        {
            // Client KHÔNG đi qua handler này (tránh đệ quy).
            var api = services.GetRequiredService<IMyApiAnonymous>();
            tokens = await api.RefreshAsync(new RefreshTokenRequestVM(refresh), cts.Token);
        }
        catch (ApiException ex) when (ex.StatusCode is HttpStatusCode.Unauthorized or HttpStatusCode.Forbidden)
        {
            // Middleware/filter bắt: SignOut + về trang đăng nhập. Lỗi tạm (5xx, mạng) để bay lên, KHÔNG đăng xuất.
            throw new SessionExpiredException();
        }

        WebAuth.StoreTokens(ticket.Properties, tokens);
        http.Items[TicketKey] = ticket;
        // Response đã gửi header thì không ghi cookie được; refresh token không đổi nên lần sau tự làm mới lại.
        if (!http.Response.HasStarted)
            await http.SignInAsync(WebAuth.Scheme, ticket.Principal, ticket.Properties);
        return tokens.AccessToken;
    }

    private static bool IsExpiring(AuthenticationProperties p)
        => !DateTime.TryParse(p.GetTokenValue(WebAuth.AccessExpiresName), null, DateTimeStyles.RoundtripKind, out var exp)
           || exp - DateTime.UtcNow < RefreshSkew;

    private static bool IsAnonymousAuthCall(HttpRequestMessage r)
    {
        var path = r.RequestUri?.AbsolutePath ?? string.Empty;
        return path.EndsWith("/auth/login", StringComparison.OrdinalIgnoreCase)
            || path.EndsWith("/auth/refresh", StringComparison.OrdinalIgnoreCase);
    }
}
```

Middleware dựng `HttpContext.User` từ API (đặt sau `UseAuthentication`, trước `UseAuthorization`): đọc
`SessionIdClaim`, hỏi `GET me` (cache `IMemoryCache` ngắn theo mã phiên), dựng principal có id/tên/email/vai/quyền;
`SessionExpiredException` hoặc 401 thì `SignOutAsync`; API lỗi tạm thì giữ principal chỉ có mã phiên (không quyền).
Hạn chờ cho `me` đủ dài để chứa cả lượt làm mới token.

### API: access token gọn

```csharp
public (string Token, DateTime ExpiresAt) CreateAccessToken(string userId, string sessionId, DateTime utcNow)
{
    var expires = utcNow.Add(_options.AccessTokenLifetime);   // vd 15 phút
    Claim[] claims =
    [
        new(JwtRegisteredClaimNames.Sub, userId),
        new(AppClaimTypes.SessionId, sessionId),
        new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString("N")),
    ];
    var token = new JwtSecurityToken(_options.Issuer, _options.Audience, claims, utcNow, expires, _signingCredentials);
    return (new JwtSecurityTokenHandler().WriteToken(token), expires);
}
```

### API: làm mới không dùng một lần

```csharp
public async Task<ServiceResult<AuthTokensVM>> RefreshAsync(string refreshToken, CancellationToken ct)
{
    if (!RefreshTokenCodec.TryParse(refreshToken, out var sessionId, out var secret))
        return ServiceResult.Unauthorized(SessionEnded);

    var now = _clock.GetUtcNow().UtcDateTime;
    var session = await _sessions.GetAsync(sessionId, ct);
    if (session is null || !session.IsAlive(now)
        || !RefreshTokenCodec.HashEquals(RefreshTokenCodec.Hash(secret), session.RefreshTokenHash))
        return ServiceResult.Unauthorized(SessionEnded);

    var user = await _accounts.GetAccessAsync(session.UserId, ct);
    if (user is null || user.IsDeleted || user.IsLocked)
        return ServiceResult.Unauthorized(SessionEnded);

    // Giữ nguyên refresh token, chỉ kéo dài hạn phiên: request song song cùng làm mới đều thành công.
    var refreshExpires = now.Add(_options.RefreshTokenLifetime);
    await _sessions.ExtendAsync(sessionId, refreshExpires, now, ct);

    var (access, accessExpires) = _tokens.CreateAccessToken(user.UserId, sessionId, now);
    return ServiceResult<AuthTokensVM>.Ok(new(access, accessExpires, refreshToken, refreshExpires, sessionId));
}
```

### API: kiểm phiên mỗi request, cache ngắn

```csharp
/// <summary>
/// Kết quả tra phiên + vai + quyền, cache ngắn theo mã phiên. Mọi lượt vô hiệu tăng MỘT bộ đếm thế hệ (field, tăng
/// đơn điệu); mục ghi kèm thế hệ đọc TRƯỚC khi nạp, nên lượt nạp đang bay gặp vô hiệu sẽ bị bỏ ở lần đọc sau.
/// </summary>
public sealed class SessionAccessCache(TimeProvider clock) : IDisposable
{
    public static readonly TimeSpan Lifetime = TimeSpan.FromSeconds(30);

    // Cache RIÊNG: IMemoryCache chung của app mà bị đặt SizeLimit thì Set không kèm Size sẽ ném, hỏng xác thực.
    private readonly MemoryCache _cache = new(new MemoryCacheOptions());
    private long _generation;

    private sealed record Entry(UserAccessRecordVM Access, DateTimeOffset ExpiresAt, long Generation);

    public async Task<UserAccessRecordVM?> GetAsync(string sessionId, string userId,
        Func<CancellationToken, Task<UserAccessRecordVM?>> load, CancellationToken ct)
    {
        var generation = Interlocked.Read(ref _generation);
        if (_cache.TryGetValue(sessionId, out Entry? hit) && hit!.Access.UserId == userId
            && hit.Generation == generation && hit.ExpiresAt > clock.GetUtcNow())
            return hit.Access;

        var access = await load(ct);
        if (access is null)
            _cache.Remove(sessionId);   // phiên chết: không cache, lần sau tra lại
        else
            _cache.Set(sessionId, new Entry(access, clock.GetUtcNow().Add(Lifetime), generation), Lifetime);
        return access;
    }

    // Ghi hiếm nên bỏ cả cache 30 giây mỗi lần vô hiệu gần như không tốn gì; tách tên để nơi gọi nói rõ ý định.
    public void InvalidateSession(string sessionId) => InvalidateAll();
    public void InvalidateUser(string userId) => InvalidateAll();
    public void InvalidateAll() => Interlocked.Increment(ref _generation);

    public void Dispose() => _cache.Dispose();
}
```

**Bẫy đã gặp**: đừng giữ số phiên bản theo người dùng trong `IMemoryCache` (`cache.Set(userKey, version + 1)`). Mục
phiên bản bị evict (hết hạn, `Compact` khi thiếu bộ nhớ) thì lần tăng sau đặt lại `1`, khớp đúng số mà lượt nạp đang
bay đã đọc, nên quyền/phiên cũ sống thêm một chu kỳ. Cần chi tiết theo người thì giữ số trong
`ConcurrentDictionary<string,long>` tăng đơn điệu, không để cache evict.

```csharp
builder.Services.AddSingleton<SessionAccessCache>();
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme).AddJwtBearer(o =>
{
    o.MapInboundClaims = false;
    o.TokenValidationParameters = new()
    {
        // issuer, audience, signing key, ClockSkew nhỏ...
        NameClaimType = ClaimTypes.Name,
        RoleClaimType = ClaimTypes.Role,
    };
    o.Events = new JwtBearerEvents
    {
        OnTokenValidated = async ctx =>
        {
            var sid = ctx.Principal?.FindFirstValue(AppClaimTypes.SessionId);
            var uid = ctx.Principal?.FindFirstValue(JwtRegisteredClaimNames.Sub);
            if (string.IsNullOrEmpty(sid) || string.IsNullOrEmpty(uid))
            {
                ctx.Fail("Token không mang mã phiên.");
                return;
            }

            var sp = ctx.HttpContext.RequestServices;
            var access = await sp.GetRequiredService<SessionAccessCache>().GetAsync(sid, uid,
                c => sp.GetRequiredService<IAuthService>().ValidateSessionAsync(sid, uid, c),
                ctx.HttpContext.RequestAborted);
            if (access is null)
            {
                ctx.Fail("Phiên đã bị thu hồi hoặc hết hạn.");
                return;
            }

            // Định danh lấy từ token, phần còn lại từ DB.
            var claims = new List<Claim>
            {
                new(ClaimTypes.NameIdentifier, uid),
                new(AppClaimTypes.SessionId, sid),
                new(ClaimTypes.Name, access.UserName),
            };
            if (!string.IsNullOrEmpty(access.Email))
                claims.Add(new(ClaimTypes.Email, access.Email));
            claims.AddRange(access.Roles.Select(r => new Claim(ClaimTypes.Role, r)));
            claims.AddRange(access.Permissions.Select(p => new Claim(AppClaimTypes.Permission, p)));
            ctx.Principal = new(new ClaimsIdentity(claims, JwtBearerDefaults.AuthenticationScheme,
                ClaimTypes.Name, ClaimTypes.Role));
        },
    };
});
```

Chỗ ghi phải gọi vô hiệu cache (trong service ghi, sau khi lưu thành công):
- Đăng xuất / thu hồi một phiên → `InvalidateSession`.
- Đổi mật khẩu, khoá/xoá tài khoản, đổi vai của một người → thu hồi phiên trong DB (nếu cần) rồi `InvalidateUser`.
- Sửa quyền của một vai → `InvalidateAll`.
- Nhiều instance API thì `IMemoryCache` không đồng bộ: chấp nhận trễ tối đa `Lifetime`, hoặc phát tín hiệu vô
  hiệu qua kênh chung (Redis pub/sub).
