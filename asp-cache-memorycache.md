# Cache kiểu `IMemoryCache` (dữ liệu theo từng người dùng)

Dùng cho dữ liệu mỗi khoá chỉ vài lượt gọi cùng lúc: vai trò, quyền, hạn mức, tính năng bật cho một user. Chốt yêu
cầu và chọn kiểu cache trước ở `~/.claude/asp-cache.md`; dữ liệu public dùng chung thì KHÔNG dùng file này.

## Khuôn

- Bọc trong một class riêng (vd `UserRoleCache`), đăng ký singleton hoặc scoped, nhận `IMemoryCache` qua ctor
  (`services.AddMemoryCache()`). Code nghiệp vụ gọi class này, không gọi `IMemoryCache` trực tiếp.
- Đọc: `TryGetValue(key, out var cached)` có thì trả; không có thì nạp từ DB/API rồi `Set(key, value, ttl)`.
- Khoá có tiền tố theo loại dữ liệu + id: `$"user-roles:{userId}"`. Gom việc dựng khoá vào một hàm `KeyOf(...)`
  để bên đọc và bên xoá dùng đúng một chuỗi.
- TTL ngắn (vài chục giây tới vài phút) làm lưới an toàn; đúng đắn dựa vào việc xoá khi ghi, không dựa vào TTL.
- Giá trị lưu là bất biến (`IReadOnlyList<T>`, record). Không trả list/DTO có thể bị bên gọi sửa: mọi request
  dùng chung một instance.

## Xoá khi ghi

- Hàm ghi đổi dữ liệu của user (đổi vai trò, cấp/thu quyền, đổi hạn mức) gọi `Remove(KeyOf(userId))` SAU khi
  lưu thành công. Thiếu bước này thì người bị thu quyền vẫn còn quyền tới hết TTL: đây là lỗ hổng quyền, không
  phải lỗi hiển thị.
- Ghi ảnh hưởng nhiều user (đổi định nghĩa một vai trò) thì không liệt kê nổi khoá: dùng
  `CancellationChangeToken` chung cho nhóm đó, hoặc chấp nhận TTL thật ngắn và nói rõ với user.
- Nhiều instance: `Remove` chỉ xoá ở tiến trình hiện tại. Dữ liệu quyền thì phải có kênh xoá liên tiến trình
  hoặc không cache.

## Bẫy

- `GetOrCreate`/`GetOrCreateAsync` không gộp lượt nạp trùng; với dữ liệu theo user thì chấp nhận được, đừng dùng
  nó như bằng chứng là đã chống cache stampede.
- Đặt `SizeLimit` cho `MemoryCache` thì MỌI lần `Set` phải khai `Size`, không thì ném exception. Bật `SizeLimit`
  khi số user lớn để cache không phình vô hạn.
- Không cache kết quả của request đang có lỗi tạm (timeout, 5xx): lần sau phải nạp lại.

## Test bắt buộc

- Lần gọi thứ hai trong TTL không chạm DB/API (đếm số lần nạp bằng fake).
- Sau hàm ghi thành công, lần đọc kế tiếp thấy dữ liệu mới; ghi thất bại thì không xoá.
- Hai user khác nhau không đọc nhầm cache của nhau.
