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
- API tra phiên trong DB (qua cache ngắn) ở mỗi request; API trả 401 thì Web xoá cookie và đưa về đăng nhập.
- **Why**: (2026-11) với access + refresh, Web làm mới chủ động trước mỗi lời gọi; khi API auth chậm, MỌI lời gọi
  API phải chờ trọn timeout của lượt làm mới. Thêm vào đó là logic làm mới, đua ghi đè cookie giữa các AJAX song
  song, lỗi "response đã bắt đầu nên không ghi cookie được", gửi lại thân request sau 401. Bỏ refresh thì tất cả
  biến mất; việc kiểm phiên đằng nào cũng tra phiên mỗi request (qua cache ngắn: thu hồi có hiệu lực ngay trên instance
  vừa thu hồi, các instance khác trễ tối đa bằng TTL cache) nên JWT không đem lại gì.

## Token

- `sid`: ngẫu nhiên (`Guid`), là khoá chính của bảng phiên. `secret`: 256 bit từ `RandomNumberGenerator`, mã
  base64url.
- DB chỉ lưu `SHA-256(secret)`. Hash nhanh là đủ vì secret có entropy cao (khác mật khẩu, không cần
  bcrypt/Argon2). So sánh bằng `CryptographicOperations.FixedTimeEquals`.
- Không bao giờ log token. Log request phải che header `Authorization`.

```csharp
public static (string Token, byte[] SecretHash) CreateSessionToken(Guid sessionId)
{
    var secret = Base64Url.EncodeToString(RandomNumberGenerator.GetBytes(32));
    return ($"{sessionId:N}.{secret}", SHA256.HashData(Encoding.ASCII.GetBytes(secret)));
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
- **Gọi API**: một `DelegatingHandler` chỉ đọc token từ vé cookie và gắn header, gửi đúng một lần. API trả 401 →
  phiên hết → ném `SessionExpiredException`, middleware/filter xoá cookie và chuyển về trang đăng nhập (AJAX nhận
  JSON 401 kèm `redirect`). Không làm mới chủ động, không gửi lại request.
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

## DBSC (tuỳ chọn, về sau)

Device Bound Session Credentials: thêm một cookie gắn thiết bị riêng, sống ngắn (~10 phút), trình duyệt tự làm mới
bằng cách ký challenge bằng khoá trong TPM (challenge là nonce có ký, server không cần lưu). Server lưu public key
của thiết bị theo phiên (`DevicePublicKey`). Trình duyệt không hỗ trợ thì lùi về session token thường. Đây không
phải định danh thiết bị: mỗi phiên/site một khoá mới.
