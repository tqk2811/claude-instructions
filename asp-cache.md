# Cache dữ liệu trong ASP.NET (API + web gọi API)

Khuôn đã dùng thật cho hệ "API backend + web Razor gọi API qua client typed". Áp dụng khi được yêu cầu
"tăng tốc / thêm cache" cho dữ liệu đọc nhiều, ít đổi. Đọc kèm `~/.claude/asp.md` và `~/.claude/asp-architecture.md`.

## Chốt với user TRƯỚC khi code

- Tầng nào được cache (API, web public, web admin). Mặc định **không cache app admin**: admin sửa xong phải thấy ngay.
- TTL chấp nhận được ở từng tầng (vd API 10 phút cho dữ liệu đứng yên, 2 phút cho danh sách; web 60 giây).
- Bộ đếm (lượt xem, click) có được trễ theo TTL không. Thường là có: KHÔNG xoá cache theo lượt xem, nếu không
  cache bị xoá liên tục.
- **Không làm nút "xoá cache" cho người dùng bấm** trừ khi user đòi. Cache chỉ xoá tự động khi dữ liệu liên quan đổi.
- Chạy một hay nhiều instance. Một instance: cache trong tiến trình là đủ. Nhiều instance: cần kênh xoá liên tiến
  trình (pub/sub, Redis) hoặc chấp nhận trễ theo TTL.

## Lớp cache dùng chung (project Shared, cả API lẫn web tham chiếu được)

- `StaleWhileRefreshCache<T>`: một giá trị + hạn, singleton.
  - Giá trị và hạn nằm chung một record bất biến gán vào field `volatile`: người đọc không cần khoá.
  - Hết hạn: chỉ MỘT lượt nạp lại (SemaphoreSlim). Đã có bản cũ thì lượt khác `Wait(0)` hụt khoá trả ngay bản cũ,
    không xếp hàng sau lượt đang treo.
  - `Invalidate()` tăng số phiên bản + đánh dấu bản đang giữ bằng CAS. Lượt nạp bắt đầu TRƯỚC lúc invalidate
    không được ghi kết quả lên. Sau invalidate, các lượt CHỜ bản mới chứ không nhận bản cũ.
  - Nạp lỗi mà có bản cũ: log warning, trả bản cũ, **gia hạn bản cũ thêm min(ttl, ~10s)** (CAS, chỉ khi version
    chưa đổi). Không gia hạn thì backend chết là mọi request đều chịu timeout và log ngập.
  - Nạp lỗi mà chưa có bản cũ: nhớ lỗi ~2s (kèm số thứ tự) cho các lượt ĐANG xếp hàng lúc đó, ném lại bằng
    `ExceptionDispatchInfo` (giữ stack). Không có bước này thì N lượt chờ nạp nối đuôi N×timeout. Lượt đến sau
    khi lỗi đã ghi vẫn tự nạp lại. Nạp thành công thì xoá lỗi đã nhớ.
  - Lượt làm mới khi đã có bản cũ chạy `load` với `CancellationToken.None` (client ngắt không huỷ lượt làm mới);
    lần nạp đầu dùng token của request. Filter `catch ... when (!loadToken.IsCancellationRequested)`, không dùng
    token của request.
  - Dùng `TimeProvider` để test được hạn.
- `KeyedStaleWhileRefreshCache<TKey,T>`: mỗi key một cache con, có trần `maxEntries` (bỏ key lâu không dùng;
  đặt `LastUsed` ngay trong factory để slot mới không bị đuổi ngay). Cache cả kết quả `null` (404), ném 404 ở ngoài.
- `CacheRegistry` (singleton) + enum `CacheGroup` theo vùng dữ liệu (News, Events, Members...).
  - Mỗi cache khai **mọi nhóm nó phụ thuộc** lúc tạo: `Create<T>(ttl, params CacheGroup[])`. Bên ghi chỉ cần xoá
    nhóm của chính nó, không phải nhớ ai đang đọc dữ liệu của mình.
  - Chỉ gọi `Create`/`CreateKeyed` trong ctor của singleton; gọi theo request là rò bộ nhớ (registry giữ tham chiếu
    mạnh). Log cảnh báo khi một nhóm vượt ~100 cache.
- `load` phải tự đọc dữ liệu BÊN TRONG lambda. Dùng giá trị tính sẵn bên ngoài thì invalidate mất tác dụng.
- Không mutate DTO lấy từ cache: mọi request dùng chung một instance. Ghi cảnh báo này trong class giữ cache.

## Tầng API

- Đặt cache ở **controller**, qua một singleton (vd `PublicReadCache`) giữ các field cache. Lambda gọi service của
  chính request nên không cần `IServiceScopeFactory`. Không làm decorator khi interface service trộn hàm admin với
  hàm public (phải forward hàng chục hàm).
- Chỉ cache: tham số mặc định, trang 1, không từ khoá/bộ lọc. Clamp tham số trước rồi mới làm khoá. Chuẩn hoá
  khoá (trim, lower) thì truyền ĐÚNG chuỗi đã chuẩn hoá vào service.
- Endpoint có dữ liệu theo người xem (đã đăng ký chưa, có phải hội viên không...) chỉ cache nhánh khách
  (`viewer == null`). Có test khẳng định bản khách không mang trường của người xem.
- Xoá cache đặt trong **service** (phủ cả đường admin lẫn đường người dùng tự sửa), qua `ICacheInvalidator`:
  - Tham số ctor tuỳ chọn `ICacheInvalidator? cache = null` → bản no-op, để test cũ không vỡ.
  - Gọi SAU khi lưu/commit transaction thành công, không gọi ở nhánh thất bại. Import hàng loạt: gọi một lần cuối.
  - Hàm gián tiếp (Approve đi qua ChangeStatus...) thì đặt ở hàm chung.
- `ICacheInvalidationNotifier`: hook gọi sau mỗi lần xoá, để báo sang tầng khác. Hợp đồng: chạy trong request ghi,
  CHỈ được đẩy vào queue rồi trả về ngay, không IO đồng bộ, không ném.

## Tầng web gọi API

- **Loader cache phải gọi API ẩn danh.** Handler gắn bearer của phiên vào mọi lời gọi; cache lại chia lỗi cho lượt
  chờ, nên phiên hết hạn của một người làm khách cùng lúc cũng nhận lỗi. Dùng cờ `AsyncLocal` (vd
  `AnonymousApiScope.RunAsync`) để handler bỏ qua bearer trong loader. `RunAsync` là hàm `async` nên cờ không rò ra
  ngoài caller.
- Endpoint không phụ thuộc người xem: cache cho mọi người, kể cả người đã đăng nhập. Endpoint phụ thuộc người xem:
  chỉ cache khi request không có phiên.
- Reader cấu hình site: đọc cache trước, giữ bản last-known-good khi API lỗi. App admin không đăng ký cache thì
  reader gọi API thẳng (tham số ctor tuỳ chọn).
- Endpoint nội bộ nhận lệnh xoá (vd `POST /_internal/cache/invalidate`, body `{"groups":[...]}`):
  - Middleware short-circuit đặt TRƯỚC `UseForwardedHeaders`, `UseHttpsRedirection`, `UseStatusCodePages`.
  - Chỉ nhận `Connection.RemoteIpAddress` loopback (IP thô) VÀ không có header proxy (`X-Forwarded-For`,
    `Forwarded`, `X-Real-IP`, `CF-Connecting-IP`, `X-Original-For` — IIS chuyển XFF sang header này).
  - Khoá trong header, so SHA256 hai bên bằng `CryptographicOperations.FixedTimeEquals` (không lộ độ dài).
  - Khoá cấu hình rỗng → endpoint tắt. Mọi trường hợp không hợp lệ trả 404. Giới hạn body (đếm cả chunked).
  - Gọi `CacheRegistry` trực tiếp, không gọi `ICacheInvalidator` (tránh vòng báo).
- Phía API gửi lệnh: `Channel` bounded `DropOldest`, `BackgroundService` gom ~200ms rồi POST qua `IHttpClientFactory`
  (timeout ngắn). Retry 3 lần với 5xx/exception; 401/403/404 là lỗi cấu hình → không retry, log Error. Lúc khởi động
  log bật/tắt, cảnh báo nếu URL không phải loopback. Thiếu cấu hình thì không đăng ký gì; TTL ở web là lưới an toàn.

## File tĩnh

- `UseStaticFiles` + `OnPrepareResponse`: URL có `?v=` (`asp-append-version`) → `public,max-age=31536000,immutable`;
  không version → vài giờ. Thư mục upload chỉ để `immutable` khi tên tệp sinh mới mỗi lần ghi (Guid).

## Test bắt buộc

- Lớp cache: còn hạn, hết hạn, đồng thời một lượt nạp, stale khi đang nạp, invalidate lúc đang nạp, lỗi có/không
  bản cũ, backoff, cancel khi giữ khoá.
- Mỗi service ghi: thành công thì xoá đúng nhóm, thất bại thì không (fake `ICacheInvalidator`).
- Controller: người xem đăng nhập / trang > 1 / có bộ lọc thì không dùng cache.
- Endpoint xoá: non-loopback, có header proxy, sai khoá, thiếu cấu hình đều bị chặn.
- E2E dùng chung một host giữa các test: xoá toàn bộ cache (`InvalidateAll`) đầu mỗi lượt, nếu không test sau thấy
  dữ liệu của test trước.
