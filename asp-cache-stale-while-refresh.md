# Cache kiểu StaleWhileRefreshCache (dữ liệu public, dùng chung)

Dùng cho dữ liệu nhiều người gọi trùng một khoá cùng lúc. Chốt yêu cầu và chọn kiểu cache trước ở
`~/.claude/asp-cache.md`.

## Lớp cache dùng chung (project Shared, cả API lẫn web tham chiếu được)

Mã nguồn đầy đủ ở mục [Mã nguồn lớp cache dùng chung](#mã-nguồn-lớp-cache-dùng-chung) cuối file: tự viết, không dùng thư viện ngoài,
chỉ dựa trên `SemaphoreSlim`, `Interlocked`/`volatile`, `TimeProvider`, `ConcurrentDictionary`.

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
- `CacheRegistry` (singleton) + enum `CacheGroup` theo vùng dữ liệu (vd Products, Orders, Settings).
  - Mỗi cache khai **mọi nhóm nó phụ thuộc** lúc tạo: `Create<T>(ttl, params CacheGroup[])`. Bên ghi chỉ cần xoá
    nhóm của chính nó, không phải nhớ ai đang đọc dữ liệu của mình.
  - Chỉ gọi `Create`/`CreateKeyed` trong ctor của singleton; gọi theo request là rò bộ nhớ (registry giữ tham chiếu
    mạnh). Log cảnh báo khi một nhóm vượt ~100 cache.
- `load` phải tự đọc dữ liệu BÊN TRONG lambda. Dùng giá trị tính sẵn bên ngoài thì invalidate mất tác dụng.
- Không mutate DTO lấy từ cache: mọi request dùng chung một instance. Ghi cảnh báo này trong class giữ cache.

## Tầng API

- Đặt cache ở **controller**, qua một singleton (vd `PublicReadCache`) giữ các field cache. Lambda gọi service của
  chính request nên không cần `IServiceScopeFactory`. Không làm decorator khi interface service trộn hàm quản trị với
  hàm public (phải forward hàng chục hàm).
- Chỉ cache: tham số mặc định, trang 1, không từ khoá/bộ lọc. Clamp tham số trước rồi mới làm khoá. Chuẩn hoá
  khoá (trim, lower) thì truyền ĐÚNG chuỗi đã chuẩn hoá vào service.
- Endpoint có trường riêng theo người xem (trạng thái của chính người đó, quyền...) chỉ cache nhánh khách
  (`viewer == null`). Có test khẳng định bản khách không mang trường của người xem.
- Xoá cache đặt trong **service** (phủ cả đường quản trị lẫn đường người dùng tự sửa), qua `ICacheInvalidator`:
  - Tham số ctor tuỳ chọn `ICacheInvalidator? cache = null` → bản no-op, để test cũ không vỡ.
  - Gọi SAU khi lưu/commit transaction thành công, không gọi ở nhánh thất bại. Ghi hàng loạt: gọi một lần cuối.
  - Nhiều hàm ghi cùng đi qua một hàm chung thì đặt ở hàm chung đó.
- `ICacheInvalidationNotifier`: hook gọi sau mỗi lần xoá, để báo sang tầng khác. Hợp đồng: chạy trong request ghi,
  CHỈ được đẩy vào queue rồi trả về ngay, không IO đồng bộ, không ném.

## Tầng web gọi API

- **Loader cache phải gọi API ẩn danh.** Handler gắn bearer của phiên vào mọi lời gọi; cache lại chia lỗi cho lượt
  chờ, nên phiên hết hạn của một người làm khách cùng lúc cũng nhận lỗi. Dùng cờ `AsyncLocal` (vd
  `AnonymousApiScope.RunAsync`) để handler bỏ qua bearer trong loader. `RunAsync` là hàm `async` nên cờ không rò ra
  ngoài caller.
- Endpoint không phụ thuộc người xem: cache cho mọi người, kể cả người đã đăng nhập. Endpoint phụ thuộc người xem:
  chỉ cache khi request không có phiên.
- Thành phần dùng chung giữa nhiều app (vd reader cấu hình site): nhận cache qua tham số ctor tuỳ chọn. App có
  đăng ký cache thì đọc cache trước, app không đăng ký thì gọi API thẳng. Vẫn giữ bản last-known-good khi API lỗi.
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

## Test bắt buộc

- Lớp cache: còn hạn, hết hạn, đồng thời một lượt nạp, stale khi đang nạp, invalidate lúc đang nạp, lỗi có/không
  bản cũ, backoff, cancel khi giữ khoá.
- Mỗi service ghi: thành công thì xoá đúng nhóm, thất bại thì không (fake `ICacheInvalidator`).
- Controller: người xem đăng nhập / trang > 1 / có bộ lọc thì không dùng cache.
- Endpoint xoá: non-loopback, có header proxy, sai khoá, thiếu cấu hình đều bị chặn.
- E2E dùng chung một host giữa các test: xoá toàn bộ cache (`InvalidateAll`) đầu mỗi lượt, nếu không test sau thấy
  dữ liệu của test trước.

## Mã nguồn lớp cache dùng chung

Bản đã chạy thật và qua review. Chép vào project Shared, đổi namespace `MyApp` cho khớp project; enum `CacheGroup`
liệt kê theo vùng dữ liệu của từng project. Phụ thuộc: `Microsoft.Extensions.Logging.Abstractions`,
`Microsoft.Extensions.DependencyInjection.Abstractions` (đã có sẵn nếu project tham chiếu ASP.NET Core).

### `Enums/CacheGroup.cs`

```csharp
namespace MyApp.Shared.Caching.Enums;

/// <summary>Nhóm cache để xoá theo nhóm khi dữ liệu nguồn đổi (<see cref="CacheRegistry.Invalidate"/>).</summary>
public enum CacheGroup
{
    // Mỗi project tự liệt kê theo vùng dữ liệu của mình.
    Products,
    Orders,
    Customers,
    Settings,
}
```

### `Interfaces/IInvalidatableCache.cs`

```csharp
namespace MyApp.Shared.Caching.Interfaces;

/// <summary>Cache mà <see cref="CacheRegistry"/> có thể xoá theo nhóm.</summary>
public interface IInvalidatableCache
{
    /// <summary>Đánh dấu mọi bản đang nhớ là cũ; lượt sau chờ bản mới, lượt nạp đang bay không được ghi đè.</summary>
    void Invalidate();
}
```

### `Interfaces/ICacheInvalidator.cs`

```csharp
using MyApp.Shared.Caching.Enums;

namespace MyApp.Shared.Caching.Interfaces;

/// <summary>Cổng xoá cache theo nhóm cho tầng service: gọi SAU khi dữ liệu nguồn đã lưu/commit thành công.</summary>
public interface ICacheInvalidator
{
    /// <summary>Đánh dấu cũ mọi cache thuộc các nhóm đã chọn.</summary>
    void Invalidate(params CacheGroup[] groups);
}
```

### `Interfaces/ICacheInvalidationNotifier.cs`

```csharp
using MyApp.Shared.Caching.Enums;

namespace MyApp.Shared.Caching.Interfaces;

/// <summary>
/// Bên muốn biết khi nhóm cache bị xoá (vd báo sang tiến trình khác).
/// <para><b>Hợp đồng:</b> <see cref="OnInvalidated"/> chạy ngay trong luồng request đang ghi, nên CHỈ được đẩy việc vào
/// queue/Channel rồi trả về ngay; không gọi mạng hay IO đồng bộ, và không được ném exception.</para>
/// </summary>
public interface ICacheInvalidationNotifier
{
    /// <summary>Gọi sau khi registry đã xoá xong các nhóm.</summary>
    void OnInvalidated(IReadOnlyList<CacheGroup> groups);
}
```

### `StaleWhileRefreshCache.cs`

```csharp
using MyApp.Shared.Caching.Interfaces;
using Microsoft.Extensions.Logging;
using System.Runtime.ExceptionServices;

namespace MyApp.Shared.Caching;

/// <summary>
/// Cache một giá trị kèm hạn, làm mới bởi đúng MỘT lượt một lúc.
/// <list type="bullet">
///   <item>Giá trị và hạn nằm chung một record bất biến gán một lần vào field <c>volatile</c>: người đọc ngoài
///   khoá không bao giờ thấy hạn mới đi với giá trị cũ.</item>
///   <item>Hết hạn TTL mà lượt khác đang làm mới thì trả NGAY bản cũ, không xếp hàng. Chỉ lần nạp đầu, hoặc
///   sau <see cref="Invalidate"/>, mới phải chờ.</item>
///   <item><see cref="Invalidate"/> tăng số phiên bản: lượt nạp bắt đầu TRƯỚC đó không được ghi đè kết quả cũ lên.</item>
///   <item>Nạp lỗi mà có bản cũ (chưa bị invalidate) thì log warning, trả bản cũ và gia hạn bản cũ thêm
///   <c>min(ttl, 10s)</c> (backoff, tránh mỗi request lại chịu một lần timeout + ngập log). Chưa có bản cũ thì ném lỗi;
///   lỗi được nhớ <see cref="FailureHoldFor"/> để các lượt đang xếp hàng chờ khoá nhận lại đúng lỗi đó thay vì
///   mỗi lượt tự nạp lại (tránh nối đuôi N x timeout khi backend chết).</item>
///   <item>Lượt làm mới khi ĐÃ có bản cũ chưa invalidate chạy với <see cref="CancellationToken.None"/> (client ngắt
///   không huỷ lượt làm mới của cả nhóm); lần nạp đầu / sau Invalidate dùng token của request.</item>
///   <item><b>Hàm <c>load</c> phải tự đọc dữ liệu bên trong lambda</b>, không dùng giá trị tính sẵn bên ngoài;
///   nếu không thì Invalidate không an toàn (lượt nạp sau vẫn ghi lại dữ liệu cũ).</item>
/// </list>
/// </summary>
/// <typeparam name="T">Giá trị nhớ.</typeparam>
public sealed class StaleWhileRefreshCache<T> : IInvalidatableCache
{
    private sealed record Entry(T Value, DateTimeOffset NextRefreshAt, bool Invalidated = false);

    /// <summary>Thời gian nhớ lỗi nạp cho các lượt đang chờ khoá.</summary>
    public static readonly TimeSpan FailureHoldFor = TimeSpan.FromSeconds(2);

    /// <summary>Trần thời gian gia hạn bản cũ sau khi nạp lỗi.</summary>
    public static readonly TimeSpan MaxStaleBackoff = TimeSpan.FromSeconds(10);

    private sealed record Failure(ExceptionDispatchInfo Error, DateTimeOffset Until, long Version, long Seq);

    private readonly SemaphoreSlim _lock = new(1, 1);
    private readonly TimeProvider _timeProvider;
    private readonly TimeSpan _ttl;
    private readonly ILogger? _logger;

    private volatile Entry? _entry;
    private volatile Failure? _failure;
    private long _version;
    private long _failureSeq;

    /// <param name="timeProvider">Nguồn thời gian.</param>
    /// <param name="ttl">Thời gian nhớ của lần <c>GetAsync(load, ct)</c> không tự chỉ định khoảng nhớ.</param>
    /// <param name="logger">Tuỳ chọn; dùng để log warning khi nạp lỗi mà trả bản cũ.</param>
    public StaleWhileRefreshCache(TimeProvider timeProvider, TimeSpan ttl, ILogger? logger = null)
    {
        _timeProvider = timeProvider;
        _ttl = ttl;
        _logger = logger;
    }

    /// <summary>Giá trị còn hạn, hoặc nạp mới qua <paramref name="load"/> (ném lỗi thì không nhớ).</summary>
    public Task<T> GetAsync(Func<CancellationToken, Task<T>> load, CancellationToken cancellationToken = default)
        => GetWithKeepAsync(async ct => (await load(ct).ConfigureAwait(false), (TimeSpan?)_ttl), cancellationToken);

    /// <summary>
    /// Như trên nhưng <paramref name="load"/> tự trả khoảng nhớ; <see langword="null"/> nghĩa là chỉ trả cho lượt
    /// này, không nhớ cho ai.
    /// </summary>
    public async Task<T> GetWithKeepAsync(
        Func<CancellationToken, Task<(T Value, TimeSpan? KeepFor)>> load,
        CancellationToken cancellationToken = default)
    {
        var entry = _entry;
        if (entry is not null && _timeProvider.GetUtcNow() < entry.NextRefreshAt)
            return entry.Value;

        // Mốc để biết có lượt nào vừa nạp lỗi trong lúc mình xếp hàng chờ khoá.
        long failureSeqBefore = Interlocked.Read(ref _failureSeq);

        if (entry is { Invalidated: false })
        {
            // Bản cũ chỉ hết hạn TTL: lượt khác đang làm mới thì dùng tạm, không chờ.
            // Bản đã Invalidate thì KHÔNG dùng tạm: chờ như lần nạp đầu.
            if (!_lock.Wait(0, CancellationToken.None))
                return entry.Value;
        }
        else
        {
            await _lock.WaitAsync(cancellationToken).ConfigureAwait(false);
        }

        try
        {
            entry = _entry;
            if (entry is not null && _timeProvider.GetUtcNow() < entry.NextRefreshAt)
                return entry.Value;

            var version = Interlocked.Read(ref _version);
            if (_failure is { } failed
                && failed.Seq != failureSeqBefore
                && failed.Version == version
                && _timeProvider.GetUtcNow() < failed.Until
                && entry is not { Invalidated: false })
            {
                // Lượt trước mình vừa nạp lỗi trong lúc mình chờ: nhận lại đúng lỗi đó, không tự nạp thêm.
                failed.Error.Throw();
            }

            // Đã có bản cũ thì lượt làm mới không phụ thuộc client nào; chưa có thì theo token của request.
            var loadToken = entry is { Invalidated: false } ? CancellationToken.None : cancellationToken;

            (T Value, TimeSpan? KeepFor) loaded;
            try
            {
                loaded = await load(loadToken).ConfigureAwait(false);
            }
            catch (Exception ex) when (!loadToken.IsCancellationRequested)
            {
                if (entry is { Invalidated: false })
                {
                    _logger?.LogWarning(ex, "Nạp {CacheValue} lỗi, dùng lại bản cũ.", typeof(T).Name);
                    ExtendStale(entry, version);
                    return entry.Value;
                }

                _failure = new Failure(ExceptionDispatchInfo.Capture(ex), _timeProvider.GetUtcNow() + FailureHoldFor, version,
                    Interlocked.Increment(ref _failureSeq));
                throw;
            }

            _failure = null;
            if (loaded.KeepFor is { } keep && Interlocked.Read(ref _version) == version)
            {
                _entry = new Entry(loaded.Value, _timeProvider.GetUtcNow() + keep);
                // Invalidate chen giữa lệnh kiểm số và lệnh gán: đánh dấu lại để lần đó không bị mất.
                if (Interlocked.Read(ref _version) != version)
                    MarkInvalidated();
            }
            return loaded.Value;
        }
        finally
        {
            _lock.Release();
        }
    }

    /// <summary>Gia hạn bản cũ thêm min(ttl, 10s) bằng CAS, chỉ khi chưa có Invalidate nào chen vào (không đè bản đã invalidate).</summary>
    private void ExtendStale(Entry entry, long version)
    {
        if (Interlocked.Read(ref _version) != version)
            return;
        var extended = entry with { NextRefreshAt = _timeProvider.GetUtcNow() + (_ttl < MaxStaleBackoff ? _ttl : MaxStaleBackoff) };
#pragma warning disable CS0420
        Interlocked.CompareExchange(ref _entry, extended, entry);
#pragma warning restore CS0420
        // Invalidate chen giữa CAS và lệnh kiểm: đánh dấu lại.
        if (Interlocked.Read(ref _version) != version)
            MarkInvalidated();
    }

    /// <summary>Bản đang nhớ (kể cả đã hết hạn), không hỏi lại gì; <see langword="false"/> khi chưa nạp lần nào.</summary>
    public bool TryPeek(out T value)
    {
        var entry = _entry;
        value = entry is null ? default! : entry.Value;
        return entry is not null;
    }

    /// <inheritdoc />
    public void Invalidate()
    {
        Interlocked.Increment(ref _version);
        MarkInvalidated();
    }

    /// <summary>Đặt bản đang nhớ thành hết hạn + đã invalidate; CAS để không đè mất bản mới một lượt nạp vừa ghi.</summary>
    private void MarkInvalidated()
    {
        while (true)
        {
            var entry = _entry;
            if (entry is null || entry.Invalidated)
                return;
            var marked = entry with { NextRefreshAt = DateTimeOffset.MinValue, Invalidated = true };
#pragma warning disable CS0420 // Interlocked tự có rào bộ nhớ, ref tới field volatile là an toàn.
            if (ReferenceEquals(Interlocked.CompareExchange(ref _entry, marked, entry), entry))
                return;
#pragma warning restore CS0420
        }
    }
}
```

### `KeyedStaleWhileRefreshCache.cs`

```csharp
using System.Collections.Concurrent;
using MyApp.Shared.Caching.Interfaces;
using Microsoft.Extensions.Logging;

namespace MyApp.Shared.Caching;

/// <summary>
/// Cache theo khoá: mỗi khoá một <see cref="StaleWhileRefreshCache{T}"/> riêng (hai khoá khác nhau không chờ nhau).
/// Giữ tối đa <c>maxEntries</c> khoá; vượt thì bỏ khoá ít được dùng gần nhất. <see cref="InvalidateAll"/> chặn cả
/// lượt nạp đang bay ghi đè (do từng cache con tự có số phiên bản). Hàm <c>load</c> phải tự đọc dữ liệu bên trong
/// lambda (không dùng giá trị tính sẵn) thì Invalidate mới an toàn.
/// </summary>
public sealed class KeyedStaleWhileRefreshCache<TKey, T> : IInvalidatableCache
    where TKey : notnull
{
    private sealed class Slot(StaleWhileRefreshCache<T> cache)
    {
        public StaleWhileRefreshCache<T> Cache { get; } = cache;
        public long LastUsed;
    }

    private readonly ConcurrentDictionary<TKey, Slot> _slots = new();
    private readonly TimeProvider _timeProvider;
    private readonly TimeSpan _ttl;
    private readonly int _maxEntries;
    private readonly ILogger? _logger;
    private long _tick;

    public KeyedStaleWhileRefreshCache(TimeProvider timeProvider, TimeSpan ttl, int maxEntries, ILogger? logger = null)
    {
        ArgumentOutOfRangeException.ThrowIfLessThan(maxEntries, 1);
        _timeProvider = timeProvider;
        _ttl = ttl;
        _maxEntries = maxEntries;
        _logger = logger;
    }

    /// <summary>Số khoá đang giữ.</summary>
    public int Count => _slots.Count;

    /// <summary>Giá trị của <paramref name="key"/>, còn hạn hoặc nạp mới qua <paramref name="load"/>.</summary>
    public Task<T> GetAsync(TKey key, Func<CancellationToken, Task<T>> load, CancellationToken cancellationToken = default)
    {
        if (!_slots.TryGetValue(key, out var slot))
        {
            if (_slots.Count >= _maxEntries)
                EvictOldest();
            // LastUsed đặt ngay trong factory để slot mới không bị coi là cũ nhất (0) rồi bị đuổi ngay.
            slot = _slots.GetOrAdd(key, _ => new Slot(new StaleWhileRefreshCache<T>(_timeProvider, _ttl, _logger))
            {
                LastUsed = Interlocked.Increment(ref _tick),
            });
        }
        slot.LastUsed = Interlocked.Increment(ref _tick);
        return slot.Cache.GetAsync(load, cancellationToken);
    }

    /// <summary>Đánh dấu cũ riêng một khoá (khoá chưa có thì bỏ qua).</summary>
    public void Invalidate(TKey key)
    {
        if (_slots.TryGetValue(key, out var slot))
            slot.Cache.Invalidate();
    }

    /// <summary>Đánh dấu cũ mọi khoá đang giữ.</summary>
    public void InvalidateAll()
    {
        foreach (var slot in _slots.Values)
            slot.Cache.Invalidate();
    }

    void IInvalidatableCache.Invalidate() => InvalidateAll();

    /// <summary>Bỏ khoá dùng lần cuối lâu nhất. Khoá bị bỏ đang nạp dở thì lượt đó vẫn xong, chỉ là kết quả không còn được nhớ.</summary>
    private void EvictOldest()
    {
        TKey? oldestKey = default;
        var oldest = long.MaxValue;
        var found = false;
        foreach (var pair in _slots)
        {
            if (pair.Value.LastUsed < oldest)
            {
                oldest = pair.Value.LastUsed;
                oldestKey = pair.Key;
                found = true;
            }
        }
        if (found)
            _slots.TryRemove(oldestKey!, out _);
    }
}
```

### `CacheRegistry.cs`

```csharp
using MyApp.Shared.Caching.Enums;
using MyApp.Shared.Caching.Interfaces;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Logging.Abstractions;

namespace MyApp.Shared.Caching;

/// <summary>
/// Sổ đăng ký cache theo nhóm (singleton): mọi cache tạo qua đây được xoá theo nhóm khi dữ liệu nguồn đổi.
/// <para>Chỉ gọi <c>Create</c> / <c>CreateKeyed</c> MỘT LẦN lúc khởi tạo singleton (constructor, field): registry giữ
/// tham chiếu mãi mãi, gọi theo từng request sẽ rò bộ nhớ.</para>
/// </summary>
public sealed class CacheRegistry
{
    private readonly TimeProvider _timeProvider;
    private readonly ILogger _logger;
    private const int WarnGroupSize = 100;
    private readonly object _gate = new();
    private readonly Dictionary<CacheGroup, List<IInvalidatableCache>> _groups = new();

    public CacheRegistry(TimeProvider timeProvider, ILoggerFactory? loggerFactory = null)
    {
        _timeProvider = timeProvider;
        _logger = (loggerFactory ?? NullLoggerFactory.Instance).CreateLogger<CacheRegistry>();
    }

    /// <summary>Tạo cache một giá trị gắn vào <paramref name="group"/>.</summary>
    public StaleWhileRefreshCache<T> Create<T>(CacheGroup group, TimeSpan ttl)
    {
        var cache = new StaleWhileRefreshCache<T>(_timeProvider, ttl, _logger);
        Register(group, cache);
        return cache;
    }

    /// <summary>Tạo cache theo khoá gắn vào <paramref name="group"/>, giữ tối đa <paramref name="maxEntries"/> khoá.</summary>
    public KeyedStaleWhileRefreshCache<TKey, T> CreateKeyed<TKey, T>(CacheGroup group, TimeSpan ttl, int maxEntries)
        where TKey : notnull
    {
        var cache = new KeyedStaleWhileRefreshCache<TKey, T>(_timeProvider, ttl, maxEntries, _logger);
        Register(group, cache);
        return cache;
    }

    /// <summary>
    /// Tạo cache một giá trị gắn vào NHIỀU nhóm: xoá bất kỳ nhóm nào trong <paramref name="groups"/> đều đánh dấu cache cũ.
    /// Dùng khi dữ liệu trả về ghép từ nhiều nguồn (vd trang danh bạ phụ thuộc nhóm người dùng và danh mục).
    /// </summary>
    public StaleWhileRefreshCache<T> Create<T>(TimeSpan ttl, params CacheGroup[] groups)
    {
        EnsureGroups(groups);
        var cache = new StaleWhileRefreshCache<T>(_timeProvider, ttl, _logger);
        foreach (var group in groups.Distinct())
            Register(group, cache);
        return cache;
    }

    /// <summary>Như <see cref="Create{T}(TimeSpan, CacheGroup[])"/> nhưng cache theo khoá, giữ tối đa <paramref name="maxEntries"/> khoá.</summary>
    public KeyedStaleWhileRefreshCache<TKey, T> CreateKeyed<TKey, T>(TimeSpan ttl, int maxEntries, params CacheGroup[] groups)
        where TKey : notnull
    {
        EnsureGroups(groups);
        var cache = new KeyedStaleWhileRefreshCache<TKey, T>(_timeProvider, ttl, maxEntries, _logger);
        foreach (var group in groups.Distinct())
            Register(group, cache);
        return cache;
    }

    /// <summary>Xoá mọi cache thuộc các nhóm đã chọn; nhóm khác không bị đụng.</summary>
    public void Invalidate(params CacheGroup[] groups)
    {
        foreach (var group in groups)
        {
            IInvalidatableCache[] caches;
            lock (_gate)
                caches = _groups.TryGetValue(group, out var list) ? list.ToArray() : [];
            foreach (var cache in caches)
                cache.Invalidate();
        }
    }

    /// <summary>Xoá mọi cache của mọi nhóm.</summary>
    public void InvalidateAll() => Invalidate(Enum.GetValues<CacheGroup>());

    private static void EnsureGroups(CacheGroup[] groups)
    {
        if (groups is null || groups.Length == 0)
            throw new ArgumentException("Cần ít nhất một nhóm cache.", nameof(groups));
    }

    private void Register(CacheGroup group, IInvalidatableCache cache)
    {
        lock (_gate)
        {
            if (!_groups.TryGetValue(group, out var list))
                _groups[group] = list = [];
            list.Add(cache);
            if (list.Count == WarnGroupSize + 1)
                _logger.LogWarning("Nhóm cache {Group} vượt {Max} cache; Create/CreateKeyed có đang bị gọi theo request?", group, WarnGroupSize);
        }
    }
}
```

### `CacheInvalidator.cs`

```csharp
using MyApp.Shared.Caching.Enums;
using MyApp.Shared.Caching.Interfaces;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Logging.Abstractions;

namespace MyApp.Shared.Caching;

/// <summary>
/// Singleton: xoá cache trong <see cref="CacheRegistry"/> trước, rồi báo từng <see cref="ICacheInvalidationNotifier"/>.
/// Notifier lỗi chỉ ghi log cảnh báo, không làm hỏng lượt ghi đã thành công của service.
/// </summary>
public sealed class CacheInvalidator : ICacheInvalidator
{
    private readonly CacheRegistry _registry;
    private readonly ICacheInvalidationNotifier[] _notifiers;
    private readonly ILogger _logger;

    public CacheInvalidator(CacheRegistry registry, IEnumerable<ICacheInvalidationNotifier> notifiers, ILoggerFactory? loggerFactory = null)
    {
        _registry = registry;
        _notifiers = notifiers.ToArray();
        _logger = (loggerFactory ?? NullLoggerFactory.Instance).CreateLogger<CacheInvalidator>();
    }

    public void Invalidate(params CacheGroup[] groups)
    {
        if (groups.Length == 0) return;
        _registry.Invalidate(groups);
        foreach (var notifier in _notifiers)
        {
            try { notifier.OnInvalidated(groups); }
            catch (Exception ex) { _logger.LogWarning(ex, "Notifier {Notifier} lỗi khi báo xoá cache", notifier.GetType().Name); }
        }
    }
}
```

### `NullCacheInvalidator.cs`

```csharp
using MyApp.Shared.Caching.Enums;
using MyApp.Shared.Caching.Interfaces;

namespace MyApp.Shared.Caching;

/// <summary>Bản không làm gì: mặc định khi service dựng ngoài DI (test, công cụ) không có cache để xoá.</summary>
public sealed class NullCacheInvalidator : ICacheInvalidator
{
    public static readonly NullCacheInvalidator Instance = new();

    private NullCacheInvalidator() { }

    public void Invalidate(params CacheGroup[] groups) { }
}
```

### `Extensions/AppCachingServiceCollectionExtensions.cs`

```csharp
using MyApp.Shared.Caching.Interfaces;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.DependencyInjection.Extensions;

namespace MyApp.Shared.Caching.Extensions;

public static class AppCachingServiceCollectionExtensions
{
    /// <summary>Đăng ký <see cref="TimeProvider"/> hệ thống (nếu chưa có) và <see cref="CacheRegistry"/> singleton.</summary>
    public static IServiceCollection AddAppCaching(this IServiceCollection services)
    {
        services.TryAddSingleton(TimeProvider.System);
        services.TryAddSingleton<CacheRegistry>();
        services.TryAddSingleton<ICacheInvalidator, CacheInvalidator>();
        return services;
    }
}
```
