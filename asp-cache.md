# Cache dữ liệu trong ASP.NET (API + web gọi API)

Khuôn cho hệ "API backend + web Razor gọi API qua client typed". Áp dụng khi được yêu cầu
"tăng tốc / thêm cache" cho dữ liệu đọc nhiều, ít đổi. Đọc kèm `~/.claude/asp.md` và `~/.claude/asp-architecture.md`.

## Chốt với user TRƯỚC khi code

- Tầng/app nào được cache. App nào người dùng sửa dữ liệu xong cần thấy ngay (thường là app quản trị) thì hỏi
  có loại khỏi cache không.
- TTL chấp nhận được ở từng tầng (vd tầng API vài phút, tầng web vài chục giây).
- Bộ đếm ghi liên tục (lượt xem, lượt bấm) có được trễ theo TTL không. Nếu được thì KHÔNG xoá cache theo bộ đếm,
  nếu không cache bị xoá liên tục.
- Có cần cơ chế xoá cache thủ công không, hay chỉ xoá tự động khi dữ liệu liên quan đổi.
- Chạy một hay nhiều instance. Một instance: cache trong tiến trình là đủ. Nhiều instance: cần kênh xoá liên tiến
  trình (pub/sub, Redis) hoặc chấp nhận trễ theo TTL.

## Chọn kiểu cache: `IMemoryCache` hay `StaleWhileRefreshCache`

Chọn theo số lượt gọi TRÙNG MỘT KHOÁ cùng lúc, không theo tầng API hay web. Chọn xong thì đọc file của kiểu đó.

| Loại dữ liệu | Lượt gọi trùng khoá | Dùng | Đọc |
|---|---|---|---|
| Public, dùng chung (trang chủ, danh mục, cài đặt site, endpoint ẩn danh) | Rất nhiều | `StaleWhileRefreshCache` | `~/.claude/asp-cache-stale-while-refresh.md` |
| Theo từng người dùng (vai trò, quyền, hạn mức, tính năng bật cho user) | Vài lượt | `IMemoryCache` | `~/.claude/asp-cache-memorycache.md` |

- `IMemoryCache` an toàn đa luồng (không hỏng dữ liệu). Nó yếu ở **lúc hết hạn**:
  - **Cache stampede**: mục hết hạn thì mọi request đang tới đều không thấy dữ liệu và cùng gọi DB/API. Khoá có
    hàng trăm khách thì DB bị dồn tải đột ngột. `GetOrCreateAsync` KHÔNG gộp các lượt nạp trùng.
  - Hết hạn là mất bản cũ: request phải chờ nạp. Backend lỗi đúng lúc đó thì mọi người cùng nhận lỗi.
  - Không có xoá theo nhóm dữ liệu khi ghi; phải tự dựng bằng `CancellationChangeToken`.
- Dữ liệu theo người dùng thì mỗi khoá chỉ một người gọi, nên những điểm yếu trên gần như không lộ. Lúc đó
  `IMemoryCache` gọn hơn và tự dọn bớt khi bộ nhớ chật (`SizeLimit`/compaction).
- Hai kiểu cùng tồn tại là lựa chọn cho tiện, không bắt buộc. Muốn một kiểu duy nhất:
  - Dùng `StaleWhileRefreshCache` cho mọi chỗ: đúng nhưng hơi nặng tay cho cache theo người dùng.
  - Dùng `HybridCache` (.NET 9+): có gộp lượt nạp trùng và xoá theo tag. Nhưng không trả bản cũ khi nạp lỗi
    hay trong lúc nạp lại, những chỗ đó vẫn phải tự làm.

## File tĩnh

- `UseStaticFiles` + `OnPrepareResponse`: URL có `?v=` (`asp-append-version`) → `public,max-age=31536000,immutable`;
  không version → vài giờ. Thư mục upload chỉ để `immutable` khi tên tệp sinh mới mỗi lần ghi (Guid).
