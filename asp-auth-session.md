# Web (MVC/Razor) gọi API bằng session token mờ (opaque) giữ trong cookie

Dùng khi một web ASP.NET (MVC/Razor Pages) không tự xác thực mà gọi một API riêng (mô hình BFF: trình duyệt chỉ
nói chuyện với web, web gọi API). Đây là cơ chế chuẩn cho các project như vậy. Thư viện NuGet public
`TqkLibrary.Asp.ToolKit.Auth.Api` (API: `AddTqkSessionToken`, `ISessionAccessSource`, `SessionAccessCache`,
`SessionTokenCodec`, `SlidingSessionPolicy`) + `TqkLibrary.Asp.ToolKit.Auth.Web` (Web: `AddTqkWebAuth`,
`AddTqkSessionBearer`, `UseTqkWebSession`, `WebSessionManager.SignInAsync(AuthSession)`), từ bản 2.0.0, đã cài sẵn phần
lớn cơ chế dưới đây — dùng nó thay vì tự viết, đọc `README-vi.md` của hai gói để biết API đúng. File này mô tả cơ chế
chung.

## Tóm tắt

- KHÔNG có access token, refresh token, JWT giữa Web và API. Chỉ một credential: **session token mờ**
  `{sid}.{secret}`, sống suốt đời phiên, không bao giờ xoay vòng.
- Web giữ token trong cookie auth (Data Protection); mỗi lời gọi API gắn `Authorization: Bearer {sid}.{secret}`.
- API tra phiên trong DB (qua cache ngắn) ở mỗi request; API trả 401 kèm `WWW-Authenticate: Bearer` thì Web xoá cookie
  và đưa về đăng nhập (chi tiết ở mục Web, quy tắc 401).
- **Why**: (2026-11) với access + refresh, Web làm mới chủ động trước mỗi lời gọi; khi API auth chậm, MỌI lời gọi
  API phải chờ trọn timeout của lượt làm mới. Thêm vào đó là logic làm mới, đua ghi đè cookie giữa các AJAX song
  song, lỗi "response đã bắt đầu nên không ghi cookie được", gửi lại thân request sau 401. Bỏ refresh thì tất cả
  biến mất; việc kiểm phiên đằng nào cũng tra phiên mỗi request (qua cache ngắn: thu hồi có hiệu lực ngay trên instance
  vừa thu hồi, các instance khác trễ tối đa bằng TTL cache) nên JWT không đem lại gì.

## Token

- `sid`: ngẫu nhiên (`Guid`), là khoá chính của bảng phiên. `secret`: 256 bit từ `RandomNumberGenerator`, mã
  base64url.
- DB chỉ lưu `SHA-256(ASCII("tqk-session:v1:" + secret))` (tiền tố tách miền, để hash này không lẫn với SHA-256 trần của
  secret ở chỗ khác; hằng `SessionTokenCodec.HashPrefix`, seed/test tạo hash bằng `SessionTokenCodec.Hash`). Hash nhanh là đủ vì secret có entropy cao (khác mật khẩu, không cần
  bcrypt/Argon2). So sánh bằng `CryptographicOperations.FixedTimeEquals`.
- `sid` trong DB phải là dạng chuẩn. `ISessionAccessSource` trả `SessionAccessRecord.SessionId` (bắt buộc) ĐÚNG NHƯ ĐÃ
  LƯU; handler so ordinal với `sid` trong token, khác thì coi như không có phiên (không cache). Chặn alias `sid` qua
  `Guid.TryParse` lỏng hoặc collation SQL không phân biệt hoa thường (cùng một phiên có nhiều cách viết, lách
  `InvalidateSession` và giới hạn theo phiên).
- Không bao giờ log token. Log request phải che header `Authorization`.

```csharp
public static (string Token, byte[] SecretHash) CreateSessionToken(Guid sessionId)
{
    var secret = Base64Url.EncodeToString(RandomNumberGenerator.GetBytes(32));
    return ($"{sessionId:N}.{secret}", SHA256.HashData(Encoding.ASCII.GetBytes("tqk-session:v1:" + secret)));
}
```
`Base64Url` (`System.Buffers.Text`) chỉ có từ .NET 9; target thấp hơn dùng `WebEncoders.Base64UrlEncode`.

## Web

- **Cookie chỉ giữ dữ liệu bất biến**: token, `sid`, `UserId`. KHÔNG bao giờ tên, email, vai, quyền: chúng đổi
  được, nằm trong cookie là dữ liệu cũ và thành lỗ hổng quyền. Web lấy chúng từ API `me` (cache ngắn theo phiên)
  rồi thay `HttpContext.User`.
- Cookie auth của ASP.NET đã mã hoá + ký bằng Data Protection, HttpOnly + Secure + SameSite. **Persist key ring
  Data Protection** (thư mục ngoài thư mục publish, `SetApplicationName` cố định, dùng chung giữa các instance):
  không persist thì mỗi lần restart là mọi người bị đăng xuất.
- **Hạn cookie** = trần TUYỆT ĐỐI của phiên do API trả trong response đăng nhập (`ExpiresUtc`). Cookie không cấp
  lại: `SlidingExpiration = false`, đặt SAU lambda cấu hình của host. Hạn trượt do API quản; web không tự tính hạn.
- Ô "Ghi nhớ đăng nhập": tick thì cookie persistent, không tick thì cookie phiên; API dùng lifetime tương ứng.
- **Gọi API**: một `DelegatingHandler` (`SessionBearerHandler`) chỉ đọc token từ vé cookie và gắn header, gửi đúng một
  lần. Không làm mới chủ động, không gửi lại request.
- **Quy tắc 401** (Web chỉ kết thúc phiên khi chắc phiên đã chết):
  - API trả 401 KÈM `WWW-Authenticate: Bearer` thì ném `SessionExpiredException`, middleware/filter xoá cookie và
    chuyển về trang đăng nhập (AJAX nhận JSON 401 kèm `redirect`). Handler xác thực của API đặt header này TRƯỚC
    `OnChallenge`; hook `OnChallenge` được ghi body riêng nhưng không được xoá header.
  - 401 KHÔNG có header đó là 401 nghiệp vụ (vd sai mã 2FA ở trang `[Authorize]`): trả nguyên, user vẫn đăng nhập.
    403 và mọi status khác không bao giờ tính.
  - Phía API: 401 nghiệp vụ trả bằng `Unauthorized()` / `Results.Unauthorized()` / `Problem(statusCode: 401)`, KHÔNG
    dùng `Challenge()` / `ChallengeAsync()` (nó thêm `WWW-Authenticate: Bearer` và làm phiên web kết thúc). Scheme
    session-token phải là challenge scheme của endpoint Web gọi (scheme Bearer khác như JwtBearer cũng sẽ kết thúc phiên).
  - Option `TreatAnyUnauthorizedAsSessionExpired` (mặc định `false`): `true` cho API không xây trên `Auth.Api`, không
    gửi challenge header; lúc đó mọi 401 kết thúc phiên. Chỉ áp dụng cho `SessionBearerHandler`.
  - RIÊNG `me`: MỌI 401 là phiên hết, có hay không có header (`me` không có 401 nghiệp vụ, proxy có thể cắt
    `WWW-Authenticate`, nếu không user đăng nhập mãi mà không có quyền). `CurrentUserMiddleware` làm việc này;
    `WebAuthOptions.IsSessionRejectedException` là lưới an toàn cho exception riêng của app (mặc định
    `SessionChallenge.IsSessionRejectedException`: có `HttpRequestException` 401 trong chuỗi `InnerException` /
    `AggregateException.InnerExceptions`, tối đa 16 node; chỉ được trả `true` cho 401). Timeout/huỷ
    (`OperationCanceledException`, `TimeoutException`) không bao giờ đăng xuất, predicate không được gọi. Rủi ro:
    gateway trả 401 hàng loạt sẽ đăng xuất mọi người; `me` gọi dịch vụ khác bằng credential riêng thì phải tự đổi lỗi đó.
  - Code gọi API không qua handler (reverse proxy/forwarder, `HttpClient` tự dựng) dùng `SessionChallenge` (public):
    `SessionChallenge.EndsSession(response, webAuthOptions)` để áp đúng luật trên.
- Phiên chết chỉ được phát hiện khi request có gọi API (kể cả `me`). Trang không gọi API thì vẫn hiển thị tới
  lượt gọi kế tiếp. Đây là đánh đổi có chủ đích; trang hiển thị dữ liệu nhạy cảm phải gọi API (hoặc `me`) để
  phiên đã thu hồi không còn xem được.
- `me` lỗi tạm (mạng/5xx/timeout) thì KHÔNG đăng xuất: chạy tiếp với principal chỉ có phiên (không quyền).
- **Đăng xuất**: gọi API thu hồi phiên (cố gắng, lỗi thì bỏ qua) rồi `SignOutAsync`.
- Mỗi web một tên cookie riêng, kể cả khi nhiều web dùng chung một API.

## API

- **AuthenticationHandler tự viết** (scheme riêng, không `AddJwtBearer`), theo thứ tự:
  1. Tách token: giới hạn độ dài và bộ ký tự, sai dạng → `invalid_token`.
  2. Nạp bản ghi phiên qua cache trong process (~10–30 giây, khoá theo `sid`).
  3. Không có phiên → vẫn băm secret và so `FixedTimeEquals` với một hash giả cố định cùng độ dài, rồi trả
     `invalid_token` (thời gian phản hồi bằng ca có phiên).
  4. Kiểm secret TRƯỚC (`FixedTimeEquals`). Sai → `invalid_token`.
  5. Rồi mới tới trạng thái user (bị khoá) → thu hồi (`RevokedAt`; được trả lý do cụ thể vì đã qua bước kiểm
     secret) → hết hạn (`ExpiresAt`) / trần tuyệt đối `CreatedAt + MaxSessionLifetime`.
  6. Dựng principal: user id, `sid`. Vai/quyền tra từ DB qua cache, không lấy từ token.
- **Sliding expiration ở API**, ghi có điều tiết: mỗi phiên tối đa một lần ghi mỗi khoảng (5 phút–1 giờ), đặt
  `LastAccessAt = now`, `ExpiresAt = min(now + lifetime, CreatedAt + MaxSessionLifetime)`. Lượt ghi dùng timeout
  riêng, KHÔNG dùng `RequestAborted`; ghi lỗi chỉ log warning, request không bao giờ hỏng vì nó.
- **Bảng phiên**: `Id`, `UserId`, `TokenHash`, `CreatedAt`, `LastAccessAt`, `ExpiresAt`, `IsPersistent`,
  `RevokedAt`, `RevokedReason`, `Ip`, `UserAgent`, `DevicePublicKey`/`DeviceKeyAlgorithm` (nullable, dành cho DBSC).
  Index `ExpiresAt`.
- **Dọn dẹp**: background job trong app, mỗi ngày xoá phiên hết hạn/thu hồi cũ hơn thời gian giữ (~30 ngày), theo
  lô (vd 1000 dòng/lượt).
- **Thu hồi** (đăng xuất, đổi mật khẩu, admin thu hồi): đặt `RevokedAt` + `RevokedReason` rồi vô hiệu cache của
  phiên (đổi vai/quyền/khoá user thì vô hiệu theo user). Nhiều instance: instance khác trễ tối đa bằng TTL cache;
  muốn admin thu hồi nhanh hơn thì để TTL ngắn hơn.
- Đổi tên/vai/quyền có hiệu lực sau TTL cache (Web `me` + API), trừ khi vô hiệu cache theo user.

## Bẫy

- **Chuyển từ access + refresh sang session token**: nếu refresh token cũ có cùng dạng `{sid}.{secret}` và cùng
  kiểu hash, đổi tên cột hash là biến mọi refresh token cũ thành session token hợp lệ (thường sống lâu hơn dự
  kiến). Migration phải hết hạn/thu hồi toàn bộ phiên đang sống, bắt mọi người đăng nhập lại.
- Cookie cũ (do bản cũ cấp, không có session token) phải bị từ chối và đăng xuất ở request đầu tiên.
- Timeout của HttpClient gọi `me` phải lớn hơn thời gian API khởi động nguội (IIS ngủ sau 20 phút), hoặc giữ API
  luôn ấm: app pool API `AlwaysRunning` + `idleTimeout = 0` + `preloadEnabled`.

## Phạm vi

- Token giữa các service / máy (secret của node, khoá dịch vụ…) là cơ chế riêng, giữ nguyên.
- Client ngoài (app mobile, bên thứ ba) mới là chỗ access + refresh / OAuth còn hợp lý.

## DBSC (tuỳ chọn, mặc định tắt)

Device Bound Session Credentials (đặc tả W3C draft, còn có thể đổi; Chrome trên Windows có TPM, nơi khác bỏ qua):
thêm một **cookie proof** gắn thiết bị, sống ngắn (mặc định 10 phút). Trình duyệt tự làm mới nó bằng cách ký challenge
bằng khoá trong TPM. Cookie chính bị chép sang máy khác thì hết dùng khi proof hết hạn. Trình duyệt không hỗ trợ bỏ qua
các header và lùi về session token thường. Đây không phải định danh thiết bị: mỗi phiên/site một khoá mới.

- **API là nguồn sự thật duy nhất của khoá thiết bị**: lưu `DevicePublicKey` (JWK public chuẩn hoá) và
  `DeviceKeyAlgorithm` (`ES256`/`RS256`) theo phiên, không bao giờ thay khoá đã gắn. Khoá KHÔNG nằm trong cookie ticket
  của Web. Đăng ký bằng một câu lệnh nguyên tử (chỉ phiên còn sống, chưa có khoá, trong `DeviceKeyRegistrationWindow`
  kể từ lúc tạo phiên, mặc định 10 phút) để token bị lộ không gắn lại phiên sang thiết bị của kẻ tấn công.
- `me` trả `deviceKeyJwk` + `deviceKeyAlgorithm` (null khi chưa gắn) và có thể trả `deviceBindingMode`
  (`ReportOnly`/`Reject`/`SignOut`, người dùng chọn). Web quyết định "đã gắn" dựa vào `deviceKeyJwk`, không đọc
  `isDeviceBound`. Thiếu `deviceKeyAlgorithm` thì Web suy từ `kty` (`EC` thành `ES256`, `RSA` thành `RS256`).
- Web giữ một `DeviceBindingCache` nhỏ trong bộ nhớ từng instance: phiên đã biết là gắn khoá thì luôn coi là gắn khoá.
  Câu trả lời "chưa gắn" của `me` chỉ được tin trong negative cache `UnboundRecheckInterval` (mặc định 30 giây); quá
  khoảng đó, hoặc chưa biết gì (instance mới), thì hỏi lại `me`, kể cả path `SkipCurrentUser`.
- Luồng: đăng nhập (chỉ HTTPS) gửi `Secure-Session-Registration`, trình duyệt POST JWT ký (`Secure-Session-Response`)
  tới `RegistrationPath`, Web kiểm challenge + proof-of-possession rồi gọi `IDeviceKeyApiClient.RegisterDeviceKeyAsync`
  và đặt cookie proof. Làm mới: POST `RefreshPath`, Web trả 403 + `Secure-Session-Challenge`, kiểm challenge đã ký bằng
  khoá lấy từ binding cache/`me`, cấp proof mới. Trình duyệt báo bỏ qua refresh qua `Secure-Session-Skipped` (chỉ
  log); request từ trình duyệt mang `Sec-Secure-Session-Id`. KHÔNG dùng tên cũ `Sec-Session-*`.
- Challenge và cookie proof không lưu trạng thái (Data Protection, purpose riêng, hạn nằm trong payload), nhiều
  instance chỉ cần dùng chung key ring. Proof mang session id + thumbprint RFC 7638 của khoá. Chống replay challenge
  chỉ theo bộ nhớ từng instance.
- Proof lỗi (`Missing`/`Expired`/`WrongSession`/`Invalid`) thì `DeviceProofFailureHandler` (singleton, kế thừa và đăng
  ký được) trả `DeviceProofOutcome`: `Continue` (= `ReportOnly`: chạy tiếp, chỉ đếm counter + log Debug), `Reject`
  (mặc định: chạy như khách, KHÔNG gửi session token, không đăng xuất, proof làm mới xong là chạy lại bình thường),
  `SignOut` (thu hồi phiên trên API cố gắng + xoá cả hai cookie), `Handled` (chỉ handler tự viết: handler đã tự ghi
  response, pipeline dừng). Handler mặc định trả `context.DefaultMode`: `DeviceBound.FailureMode` cố định, hoặc lựa
  chọn của người dùng khi `AllowUserChoice = true` (giới hạn bởi `AllowedUserModes`, giá trị lạ thì dùng `FailureMode`).
  Triển khai: chạy `ReportOnly` trước, theo dõi counter `tqk.auth.dbsc.proof_failures`, rồi mới chuyển `Reject`.
- Rủi ro: trình duyệt mất phiên DBSC thì phiên đã gắn khoá phải đăng nhập lại (với `Reject`/`SignOut`); proof bị trộm
  dùng được tới khi hết hạn nên giữ `ProofLifetime` ngắn; `ExcludedPaths` bỏ hẳn việc kiểm proof; đăng xuất lúc bị
  `Reject` xoá cookie nhưng không thu hồi phiên API (request là khách, không gửi token); instance mới + API sập thì
  coi phiên là chưa gắn tới khi `me` trả lời lại.
- Chi tiết option, hợp đồng và bẫy: `README-vi.md` của `Auth.Web` (mục DBSC) và `Auth.Api` (mục Gắn phiên với thiết bị).
