# Ẩn đường dẫn máy build & thông tin git khỏi bản Release C#

Áp dụng BẮT BUỘC cho mọi project C#. Đọc file này khi build/pack/giao bản Release,
hoặc khi dựng project C# mới.

## Ba chỗ lộ, độc lập nhau

Mặc định build C# làm LỘ đường dẫn máy dev VÀ thông tin git repo ra sản phẩm giao đi, ở BA chỗ ĐỘC LẬP nhau
(bịt chỗ này không bịt được chỗ kia — phải xử lý cả ba):
- **Mọi file DLL** nhúng đường dẫn tuyệt đối trỏ tới file `.pdb` của nó (vd `C:\src\...\obj\Release\net8.0\X.pdb`) — kể cả bản Release.
- **.NET SDK 8+ tự bật SourceLink** cho mọi repo có git remote, nhúng vào PDB một map JSON dạng
  `{"documents":{"<đường dẫn tuyệt đối>":"<URL repo>/-/raw/<commit hash>/*"}}` — lộ CẢ đường dẫn máy build LẪN URL repo và commit hash.
  `PathMap` KHÔNG xử lý được phần này, phải tắt SourceLink riêng.
- **SDK tự nối commit hash vào `AssemblyInformationalVersion`** thành `"1.0.0+<40 ký tự sha>"`, ai cũng đọc được bằng
  `[Diagnostics.FileVersionInfo]::GetVersionInfo(dll).ProductVersion`. Cơ chế này ĐỘC LẬP với SourceLink — tắt SourceLink
  vẫn còn nguyên, phải tắt bằng `IncludeSourceRevisionInInformationalVersion=false`. Đây là chỗ DỄ SÓT NHẤT vì
  quét chuỗi thô trong DLL vẫn thấy sha nhưng dễ tưởng là dữ liệu khác.

## Cách xử lý

KHÔNG sửa từng `.csproj` (dễ sót project mới thêm). Tạo 2 file MSBuild ở thư mục gốc chứa các project,
MSBuild tự import cho mọi project nằm dưới:

`Directory.Build.props` — áp cho MỌI config:
```xml
<Project>
	<PropertyGroup>
		<PathMap>$(MSBuildProjectDirectory)=$(MSBuildProjectName)</PathMap>
	</PropertyGroup>
</Project>
```

`Directory.Build.targets` — chỉ siết ở Release (bản đem giao):
```xml
<Project>
	<PropertyGroup Condition="$(Configuration.Contains('Release'))">
		<DebugType>portable</DebugType>
		<EnableSourceLink>false</EnableSourceLink>
		<IncludeSourceRevisionInInformationalVersion>false</IncludeSourceRevisionInInformationalVersion>
	</PropertyGroup>
</Project>
```

## Lưu ý quan trọng

- **Điều kiện `$(Configuration)` phải đặt ở `.targets`, KHÔNG đặt ở `.props`.** `Directory.Build.props` được import rất sớm
  (bên trong `Microsoft.Common.props`), lúc đó `$(Configuration)` có thể chưa có giá trị nên điều kiện chạy sai âm thầm.
  `Directory.Build.targets` import ở cuối nên luôn đáng tin.
- `Condition="$(Configuration.Contains('Release'))"` bắt được cả config tuỳ biến kiểu `ReleaseClientA`, `ReleaseProd`.
- **`PathMap` KHÔNG làm mất tên file + số dòng trong stack trace** — chỉ đổi tiền tố. Document trong PDB thành
  `<TênProject>\<file>.cs` thay vì `C:\src\...\<file>.cs`. Vẫn debug được bình thường.
- Chọn `DebugType`: `portable` = có file `.pdb` rời (mặc định, giữ số dòng khi kèm pdb);
  `embedded` = nhúng vào DLL, không có file `.pdb` rời để lỡ tay phát tán; `none` = không có debug info, stack trace mất số dòng.
- Các thư viện riêng của user (danh sách ở `~/.claude/local.md`) bản gốc đều đã có `PathMap` sẵn trong `ProjectBuildProperties.targets`.
  Khi CLONE/vendor source các lib này vào project khác, PHẢI mang theo `PathMap` — nếu viết lại `.csproj` từ đầu mà quên thì bản clone
  lộ đường dẫn trong khi package NuGet gốc vốn sạch.

## Cách kiểm chứng

Sau khi build (xoá sạch `bin`/`obj` trước rồi build lại):

1. Quét binary tìm chuỗi đường dẫn gốc — đọc BYTE và thử cả UTF8 lẫn Unicode encoding
   (`[IO.File]::ReadAllBytes` rồi `$enc.GetString()`), vì chuỗi trong PDB/DLL không luôn là UTF8.
   Nhớ tìm cả dạng backslash nhân đôi `C:\\src\\...` (SourceLink lưu dạng JSON escape).
2. Kiểm tra chính xác hơn: đọc bảng document trong PDB bằng `System.Reflection.Metadata`:
   `MetadataReaderProvider.FromPortablePdbStream(fs)` rồi duyệt `md.Documents` và in `md.GetString(doc.Name)`.
   Đây là cách duy nhất thấy đúng đường dẫn source mà debugger sẽ dùng — quét chuỗi thô KHÔNG thấy được vì
   tên document trong portable PDB được lưu ở dạng nén/cắt khúc.
3. Kiểm tra commit hash: `[Diagnostics.FileVersionInfo]::GetVersionInfo($dll).ProductVersion` — phải là `1.0.0` trơn,
   KHÔNG có phần `+<sha>` phía sau.
4. Nhân tiện soát luôn các thứ hay lộ kèm (dùng cùng vòng quét byte ở bước 1): username máy (`$env:USERNAME`),
   tên máy (`$env:COMPUTERNAME`), `C:\Users\<ai đó>`, đường dẫn cache `.nuget\packages`, và connection string
   (`Data Source=|Initial Catalog=|User ID=|Password=`) bị hardcode trong binary.
5. Đọc custom attribute mức assembly (duyệt `md.CustomAttributes` lọc `ca.Parent.Kind == HandleKind.AssemblyDefinition`)
   để xem còn metadata nào lộ không. Lưu ý `AssemblyConfigurationAttribute` mang tên config build (vd `ReleaseKhachHangA`),
   có thể lộ thông tin nội bộ — tắt bằng `GenerateAssemblyConfigurationAttribute=false` nếu cần.

## Ngoài phạm vi build nhưng luôn kiểm tra cùng lúc

**File `appsettings*.json` được copy vào thư mục output** thường chứa mật khẩu/token plaintext và đi kèm mọi bản binary
giao đi; đồng thời chúng hay bị commit vào git (`.gitignore` thường chỉ loại `appsettings.Development.json` mà bỏ sót
bản Production/mặc định). Quét bằng cách liệt kê TÊN key khớp
`^(pass|password|pwd|secret|clientsecret|apikey|token|privatekey)$` và giá trị khớp `Password=|Pwd=` — in tên key và
độ dài giá trị, KHÔNG in giá trị ra output.
