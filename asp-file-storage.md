# Lưu và tải file người dùng upload (ASP.NET)

Áp dụng khi dựng hoặc sửa chức năng upload/download file trong project ASP.NET (ảnh, đính kèm, tài liệu có phân quyền).
Mục tiêu: một đường duy nhất phục vụ mọi file theo **id trong sổ DB**, không phục vụ thư mục upload bằng static file.

## Sổ file (bảng `UploadedFiles`)

Mỗi file một dòng, tối thiểu các cột:

| Cột | Ý nghĩa |
|---|---|
| `Id` | khoá **Guid** (dạng `D` chữ thường trong link `/file/{Id}/...`), sinh ở C# kiểu tuần tự (vd `SequentialGuidValueGenerator` của EF Core) để không phân mảnh index clustered; KHÔNG dùng int tự tăng (link dò được). Đổi khoá int → Guid cho dòng có sẵn thì cấp Guid NGẪU NHIÊN (`NEWID()`), không `NEWSEQUENTIALID()` (giá trị liền nhau đoán được từ một link đã lộ); bảng nhỏ chấp nhận phân mảnh, cần thì rebuild index. Down của migration đó làm hỏng mọi link `/file/{guid}/` đã phát hành, chỉ dùng ở dev |
| `StoragePath` | đường dẫn tương đối trên đĩa (`<guid>.<ext>`; file cũ giữ thư mục con), **unique**, không bao giờ lộ ra client |
| `IsPublic` | **bắt buộc truyền tường minh khi tạo dòng** (không default trong C# lẫn DB); **không bao giờ đổi `true` → `false` tại chỗ** (CDN/trình duyệt còn giữ bản cache): muốn chuyển thành riêng tư thì lưu Id mới rồi xoá Id cũ |
| `OriginalFileName` | tên gốc (đã làm sạch), dùng cho `Content-Disposition` và slug |
| `CompletedAt` | null = dòng vừa chèn, file đang ghi; mọi đường đọc (tải, tra theo link) bỏ qua dòng chưa xong |
| `SizeBytes`, `CreatedAt`, `DeletedAt`, `LastSeenInUseAt` | phục vụ ETag/Last-Modified và bộ dọn file mồ côi |

- KHÔNG có cột thư mục hay đuôi file riêng: đuôi lấy từ `StoragePath`, mà đuôi đó xác định từ magic bytes lúc lưu (không tin đuôi trong tên người dùng đặt).
- KHÔNG lưu cột `Url`: link chuẩn `/file/{Id}/{slug(OriginalFileName)}.{đuôi của StoragePath}` luôn **tính lại** bằng một hàm duy nhất (thiếu tên thì `file`). Các bảng nghiệp vụ lưu đúng chuỗi do hàm này trả lúc lưu, nên hàm phải tất định và ổn định: đổi luật slug là link đã lưu lệch link chuẩn (tải vẫn được nhờ 301, nhưng kiểm-khớp-trước-xoá không còn khớp).
  - Hàm slug tên file phải **đóng băng**: một bản riêng, private trong lớp dựng link, kèm test đầu ra cố định (chữ có dấu, ký tự lạ, tên dài). KHÔNG gọi hàm slug dùng chung của trang nội dung: ai đó sửa luật slug trang là âm thầm làm lệch mọi link file đã lưu.
  - **Why**: cột `Url` là dữ liệu suy ra được, lại cần trạng thái giả (`/file/pending/...`) cho dòng chưa có Id và một unique index thừa; tính lại thì không bao giờ lệch giữa sổ và link.
- Hàm lưu (`SaveAsync(stream, ext, bool isPublic, string? originalName, ct)`) đặt `isPublic` là tham số không có giá trị mặc định, để compiler bắt mọi chỗ quên.
  - **Why**: file riêng tư bị lưu nhầm thành công khai là lộ dữ liệu; default `true` che mất lỗi đó.

## Lưu file

1. Chèn dòng sổ trước (scope DB riêng, `CompletedAt` null) để có `Id`; DB lỗi thì upload lỗi, không ghi bù "best-effort".
2. Ghi `<guid>.<ext>` bằng `FileMode.CreateNew` vào một thư mục phẳng duy nhất, không chia thư mục con, chung cho cả file công khai lẫn riêng tư. Thư mục này nằm ngoài `wwwroot` và KHÔNG được phục vụ bằng static file: quyền truy cập chỉ do endpoint `/file` quyết định theo `IsPublic`.
   - Một thư mục gốc duy nhất cho mọi file, không tách thư mục riêng tư: được là nhờ không host nào phục vụ thư mục này bằng static file. Còn static `/uploads` thì phải gỡ trước (xem mục link cũ), không thì ai đoán được tên là tải được file riêng tư.
3. Cập nhật `SizeBytes`, `CompletedAt`. Ghi file lỗi thì gỡ dòng.
- Mọi nơi upload (kể cả host web admin) đều đi qua hàm lưu này ở host có DB (gọi API), không host nào tự ghi thẳng ra thư mục upload: ghi thẳng thì không có dòng sổ, không có link `/file/...`, và bộ dọn phải đoán quyền của file.
- Nội dung của một `Id` **không bao giờ đổi**: thay ảnh là lưu file mới (Id mới) rồi xoá file cũ. Đây là điều kiện để cache `immutable`.

## Tải file: `GET/HEAD /file/{id}/{name}` (chỉ host có DB, vd API)

- Tra dòng theo `id`. Không có, `DeletedAt` có giá trị, dòng chưa hoàn tất, file mất trên đĩa → `404` kèm `no-store`.
- Đường dẫn vật lý = thư mục upload + `StoragePath`, qua hàm chống path traversal (`Path.GetFullPath` rồi kiểm nằm dưới gốc).
- **File công khai**: `Cache-Control: public, max-age=31536000, s-maxage=<CDN ngắn, vd 86400>, immutable`, `CDN-Cache-Control` tương ứng; ETag mạnh `"{id}-{size}"`, Last-Modified (đổi đúng sang UTC); `PhysicalFile(..., enableRangeProcessing: true)` để framework lo 304/206.
- **File riêng tư**: bắt buộc query `exp` (unix giây) + `sig` = base64url(HMAC-SHA256(key, "{id}.{exp}")), so bằng `CryptographicOperations.FixedTimeEquals`; sai/hết hạn, hoặc `exp` xa quá hạn tối đa (vd now + 1 ngày; hàm ký cũng không ký dài hơn) → `404`. Trả `Cache-Control: private, no-store`, `CDN-Cache-Control: no-store`, `Cloudflare-CDN-Cache-Control: no-store`, `X-Robots-Tag: noindex`, không ETag.
  - Kiểm chữ ký TRƯỚC mọi nhánh khác, và mọi phản hồi của file riêng tư đều `no-store`.
  - Sai tên thì `404` (không `301`), kể cả chữ ký đúng: link do server ký luôn mang tên chuẩn tính từ dòng sổ (đừng ký theo Url cũ đang lưu ở bảng nghiệp vụ), nên sai tên chỉ có thể là link chế/chép hỏng; 301 thì phải chép `exp&sig` sang `Location`.
  - Khoá ký đọc từ cấu hình (base64, ≥ 32 byte); thiếu khoá thì chỉ Development/test được tự sinh, môi trường khác không cho khởi động.
- Tên trong URL chỉ để trang trí: file công khai sai tên → `301` về link chuẩn tính lại. URL luôn có đuôi file (CDN như Cloudflare mặc định cache theo đuôi).
- `Content-Type` theo đuôi (`FileExtensionContentTypeProvider`), `X-Content-Type-Options: nosniff`; ảnh/video `inline`, còn lại `attachment`; tên qua `ContentDispositionHeaderValue.SetHttpFileName` (có `filename*` UTF-8); mọi thứ không phải ảnh raster (kể cả SVG) thêm `Content-Security-Policy: sandbox`.
- Không có `Set-Cookie`/session trên route này (CDN không cache phản hồi có cookie).

## Cấp link riêng tư: ký lúc bấm

- Trang HTML KHÔNG nhúng link đã ký. Trang chỉ hiện link không bí mật (vd `/<trang>/tai-lieu/{rid}`); action đó kiểm quyền rồi `302` sang `/file/{id}/...?exp&sig` vừa ký (TTL ngắn, ~15 phút), phản hồi `no-store`.
  - **Why**: nhúng link ký vào HTML thì link hết hạn khi tab mở lâu, và trang bị cache (CDN "Cache Everything", output cache) là lộ link riêng của người này cho người khác.
- Link ký không gắn với phiên: ai cầm link còn hạn cũng tải được. Đó là đánh đổi có chủ đích; cần chặt hơn thì ràng buộc user id vào chữ ký và đặt route sau đăng nhập.

## Host web khác (UI) phục vụ cùng domain

- Proxy `/file/**` về host có DB: chỉ GET/HEAD, chỉ chuyển `Range`/`If-None-Match`/`If-Modified-Since`/`If-Range` và query; không chuyển cookie/Authorization; stream body (`ResponseHeadersRead`), không tự theo redirect, không giải nén; chép header theo **danh sách cho phép**; đặt trước static files/auth/session; không bật response compression cho route này (phá Range/ETag).
- DB lưu link tương đối (`/file/...`), không lưu host: đổi domain không phải sửa dữ liệu.

## Link cũ `/uploads/...` (dữ liệu trước khi chuyển sang `/file`)

- KHÔNG viết lại link cũ trong các bảng nghiệp vụ. Gỡ static `/uploads`, thay bằng route `GET/HEAD /uploads/{**path}` ở host có DB (host UI proxy cả `/uploads/**` như `/file/**`, không tự theo redirect, `Location` tương đối giữ nguyên): tra dòng theo `StoragePath` = phần sau `/uploads/`, rồi `301` sang link chuẩn `/file/{Id}/...`, cache được (`public, max-age=86400`).
- Chỉ dòng `IsPublic`, đã `CompletedAt`, chưa `DeletedAt` mới được 301; còn lại (riêng tư, không có dòng, file chưa ghi xong) đều `404` `no-store`, không phân biệt lý do. File riêng tư cũ không được lộ Id qua đường này.
- Chuẩn hoá path trước khi tra: `\` → `/`, gộp `//`, bỏ đoạn `.`; có đoạn `..`, ký tự điều khiển, `:` hoặc path rỗng → `404` ngay (chống path traversal; `StoragePath` trong sổ luôn ở dạng chuẩn này).
- File cũ nằm trên đĩa mà chưa có dòng sổ thì `404` cho tới khi bộ dọn ghi bù dòng; chạy một vòng bộ dọn (DryRun cũng được) ngay sau deploy.

## Xoá file và bộ dọn file mồ côi

- Hàm xoá theo link chỉ xoá khi link đưa vào **khớp từng ký tự** với link chuẩn tính lại của dòng sổ (không xoá chỉ theo id).
  - **Why**: link `/file/{id}/{tên}` tra file theo id, còn chỗ kiểm "còn dùng không" so nguyên chuỗi. Client gửi `/file/<id người khác>/x.webp` rồi kích hoạt "dọn ảnh cũ" là xoá được file của người khác, kể cả file riêng tư.
- Link do client gửi lên (ảnh đại diện, đính kèm...) phải được kiểm: id có trong sổ, link khớp đúng, đúng loại (công khai, là ảnh...). Đường "dọn file cũ" chỉ đụng file công khai của đúng chủ thể.
- Bộ dọn phải nhận tham chiếu cả dạng `/file/{id}/` trong cột lẫn trong HTML/JSON (kể cả `\/file\/` đã thoát), đọc snapshot sổ trước rồi mới quét đĩa, có thời gian ân hạn cho file vừa upload. Một dòng đang dùng khi Id của nó được trỏ tới HOẶC `/uploads/` + `StoragePath` còn nằm trong dữ liệu cũ. File tìm thấy trên đĩa mà chưa có dòng thì ghi bù dòng theo `StoragePath`; file phẳng ở gốc (dạng tên mới) mà mất dòng thì ghi là riêng tư, không đoán là công khai. Đổi cách lưu mà quên cập nhật bộ dọn là bộ dọn xoá nhầm file đang dùng: hai thay đổi phải deploy cùng lúc.

## CDN (Cloudflare)

- Header làm phần chính; cấu hình thêm Cache Rule cho `/file/` (Eligible for cache, Edge TTL theo header origin, Browser TTL "Respect origin") và bypass khi query có `sig=`.
- Giữ query string trong cache key (không bật "Ignore query string" cho `/file/`).
- File đã xoá còn ở edge tới hết `s-maxage`, ở trình duyệt tới hết `max-age`: chọn `s-maxage` ngắn nếu không có purge tự động. Không đổi `IsPublic` tại chỗ; muốn chuyển công khai → riêng tư thì lưu thành Id mới rồi xoá Id cũ.
