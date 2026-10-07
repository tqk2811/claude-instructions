- Nếu cần đọc source code của các thư viện riêng của user, tên thư viện và thư mục source ghi ở `~/.claude/local.md`.
- Nếu cần dùng `record`, `init`, `required`... mà target framework không hỗ trợ sẵn (vd .NET Framework, netstandard2.0 thiếu các polyfill type như `IsExternalInit`, `RequiredMemberAttribute`), thì cài gói polyfill NuGet ghi ở `~/.claude/local.md`.
- Với model/data class: ưu tiên dùng `required` cho property bắt buộc và bật nullable reference type (property optional khai báo nullable `T?`). Nếu target framework thiếu polyfill cho `required` thì cài gói polyfill ở bullet trên.
- Repo/solution có TRÊN 2 project (.csproj) thì tạo `Directory.Build.rsp` ở gốc repo (cạnh .sln) để giới hạn MSBuild worker node — tránh VSCode/dotnet build sinh số process dotnet.exe = số core CPU và treo 15 phút do node reuse. Nội dung:
  ```
  -maxCpuCount:2
  -nodeReuse:false
  -p:UseSharedCompilation=false
  ```
  Dòng cuối tắt server biên dịch dùng chung (`VBCSCompiler.exe`): không có nó, server này ở lại chạy ngầm sau build và ăn nhiều CPU mỗi lần build lại. Đổi lại build chậm hơn một chút.

## Ưu tiên hướng đối tượng (OOP) hết mức có thể — hạn chế `static`

- Mặc định viết **class có trạng thái**: thứ gì dùng lại nhiều lần trong một luồng (phiên làm việc, kết nối, tiến trình con, thư mục làm việc, cấu hình) thì đưa vào **field `readonly` gán trong constructor**, đừng truyền đi truyền lại qua từng tham số hàm.
- **Dấu hiệu sai cần sửa**: một `static` method nhận cùng một đối tượng làm tham số đầu ở nhiều hàm liên tiếp (`DoA(session, ...)`, `DoB(session, ...)`) — đó là một class đang bị viết thành thủ tục. Cho đối tượng đó vào constructor rồi bỏ tham số đi.
- `static` chỉ dùng cho: hằng/bảng tra cứu `private static readonly`, hàm thuần tuý không phụ thuộc trạng thái (format chuỗi, tính toán nhỏ), extension method, và entry point.
- Đối tượng nắm tài nguyên (`IDisposable`/`IAsyncDisposable`) phải nói rõ **ai sở hữu**: tự tạo thì tự dispose; nhận từ ngoài vào thì KHÔNG dispose hộ. Ghi rõ quy ước này trong doc của class (thường làm bằng cờ `_ownsX` gán ở constructor).
- Ưu tiên **composition hơn kế thừa**: chứa đối tượng cộng tác thay vì kế thừa lớp cơ sở chỉ để mượn vài thành viên — kế thừa còn kéo theo mọi thành viên `public` của lớp cha ra API của mình (đó có thể là thứ đang muốn giấu).
- Trạng thái chỉ dùng trong nội bộ để `private`/`internal`; đừng để `public` cho tiện gọi.

## Quy ước tổ chức namespace theo C# type (trong feature)
- Giữ namespace theo feature (Native/Packet/Flow/Redirect/Pipeline...), nhưng tách type theo **kind** vào sub-namespace con của chính feature đó:
  - `interface` → `<Feature>.Interfaces`
  - `enum` → `<Feature>.Enums`
  - class dữ liệu/POCO + `struct` dữ liệu → `<Feature>.Models`
  - class tiện ích/utility dùng chung thật sự (helper) → `<Feature>.Helpers`; class chứa **extension methods** → `<Feature>.Extensions`. Ví dụ `Native.Helpers.AddressHelper`, `Demo.CommandHelpers.Extensions.ConnectSourceExtensions`. CHỈ áp cho helper/extension chung — đừng gom mọi `static class` (P/Invoke binding, parser lõi, service, runner... KHÔNG phải helper).
  - phần còn lại (service/engine class, `delegate`, P/Invoke, parser lõi, runner, options/config có hành vi) **giữ ở namespace feature gốc**
  - Ví dụ: `Pipeline.Interfaces.IPacketMiddleware`, `Redirect.Enums.RedirectProtocol`, `Native.Models.WinDivertAddress`; còn `ProcessRedirector`, `PacketPump`, `WinDivertHandle` ở gốc.
  - Type **nested** đơn giản (struct marshalling/POCO nhỏ) để chung file class cha cũng được; nếu **phức tạp** (có logic/nhiều thành viên) thì tách ra file riêng tên `<ClassCha>.<TypeBenTrong>.cs`, class cha khai báo `partial`.
  - File có nhiều kind → tách thành nhiều file (mỗi file 1 namespace, file-scoped namespace).
  - **Mỗi type (class/struct/interface/enum/delegate) nằm trong một file .cs riêng**, tên file = tên type. File đang chứa nhiều type thì tách mỗi type ra một file.
- LƯU Ý đặt tên namespace: tránh segment trùng tên type BCL trong cây namespace. VD `MyLib.Network.Dns` sẽ **che khuất** `System.Net.Dns` ở mọi file trong cây `MyLib.Network.*` (gây lỗi `Dns.GetHostAddresses`...). Dùng tên khác như `SecureDns`.
`

## `Directory.Build.rsp` là cấu hình build CỤC BỘ của máy — KHÔNG commit
- File `Directory.Build.rsp` ở gốc repo là MSBuild response file, msbuild/dotnet tự nạp mỗi lần build; thường chứa tinh chỉnh riêng của máy như `-maxCpuCount:2`, `-nodeReuse:false`. Commit vào repo sẽ ÉP mọi máy khác build theo cấu hình đó (vd giới hạn CPU) — không mong muốn.
- Xử lý: thêm `Directory.Build.rsp` vào `.gitignore`; nếu lỡ `git add -A` thì `git restore --staged Directory.Build.rsp` trước khi commit. ĐỪNG nhầm với `Directory.Build.props`/`Directory.Build.targets` (2 file này LÀ cấu hình dự án dùng chung, PHẢI commit).
- Hệ quả: `git clone`/`git worktree add` sang thư mục mới sẽ KHÔNG có file này. Tạo clone/worktree để build hoặc giao agent build thì chép ngay `Directory.Build.rsp` từ repo gốc sang đúng vị trí tương ứng (cạnh .sln). Build xong mà còn `VBCSCompiler.exe` chạy ngầm thì kill nó.
  - **Why**: agent build trong clone thiếu file này làm .NET chiếm nhiều CPU, user phải tự phát hiện.

## Ẩn đường dẫn build cho bản Release

Bản Release mặc định LỘ đường dẫn máy build, URL git repo và commit hash ra sản phẩm giao đi,
qua 3 cơ chế độc lập nhau. BẮT BUỘC xử lý cho mọi project C# — chi tiết cách làm và cách kiểm chứng
ở `~/.claude/releaseCsharp.md`, đọc file đó trước khi build/pack/giao bản Release hoặc khi dựng project C# mới.

## Dữ liệu đi kèm giá trị enum: gắn **attribute** lên từng giá trị, đừng nuôi bảng tra rời

- Khi mỗi giá trị enum cần dữ liệu kèm theo (id thật của API, tên hiển thị, mã số, đơn vị...), khai báo một attribute `[AttributeUsage(AttributeTargets.Field)]` rồi gắn NGAY tại giá trị đó. Đừng để `Dictionary<TEnum, string>` ở lớp helper: hai chỗ tách rời thì thêm giá trị enum mà quên điền bảng là chuyện sớm muộn, và người đọc enum không thấy được dữ liệu.
- Thuộc tính suy ra được thì cho attribute **tự tính**, đừng nhận thêm tham số (vd `ResourceName => "models/" + Id`) — chép cùng một giá trị vào hai tham số là sớm muộn cũng lệch.
- **Đọc bằng reflection thì gom MỘT LẦN** vào `private static readonly IReadOnlyDictionary<TEnum, TAttr>` dựng lúc khởi tạo lớp; gọi `GetCustomAttribute` mỗi lần vừa chậm vừa thừa (bảng không đổi trong một lần chạy):
  ```csharp
  private static readonly IReadOnlyDictionary<TEnum, TAttr> _infos = BuildInfos();

  private static IReadOnlyDictionary<TEnum, TAttr> BuildInfos()
      => Enum.GetValues<TEnum>().ToDictionary(
          x => x,
          x => typeof(TEnum).GetField(x.ToString())!.GetCustomAttribute<TAttr>()
               ?? throw new InvalidOperationException($"'{x}' chưa gắn attribute."));
  ```
- **Thiếu attribute phải NÉM, không im lặng trả rỗng** — trả chuỗi rỗng thì lỗi trôi xuống tận request gửi đi. Kèm luôn một unit test duyệt `Enum.GetValues<T>()` khẳng định giá trị nào cũng có attribute và dữ liệu không rỗng; đây là thứ compiler không bắt hộ được.
- **BẪY ĐẶT TÊN — `CS1614`**: attribute cho enum `Foo` mà đặt tên `FooAttribute` thì `[Foo(...)]` nhập nhằng giữa kiểu `Foo` và `FooAttribute`, compiler bắt phải viết `[@Foo]` hoặc `[FooAttribute]`. Đặt tên khác đi ngay từ đầu (`FooInfoAttribute` ⇒ dùng `[FooInfo(...)]`).
- Giá trị nằm ngoài enum (ép kiểu từ số lạ trong file config cũ) thì ném `ArgumentOutOfRangeException` nêu rõ giá trị — đừng để `KeyNotFoundException` trần trụi.
- **KHÔNG phải dữ liệu nào cũng nên vào attribute.** Chỉ đưa vào thứ *nội tại* của giá trị (id API, mã, tên chính thức). Chữ hiển thị của giao diện thì để lại tầng UI: nó là bản dịch/copy, có thể đổi theo ngữ cảnh (cùng một giá trị hiện tên khác nhau ở hai màn), và nhét vào enum là kéo ngôn ngữ UI xuống tầng domain.
- Với bảng `enum → nhãn UI` (danh sách ComboBox, hàm `Describe`) thì chốt chặn là **test phủ đủ giá trị**: duyệt `Enum.GetValues<T>()` khẳng định giá trị nào cũng có nhãn. Không có test thì thêm giá trị enum mà quên nhãn sẽ *im lặng* — ComboBox thiếu lựa chọn, còn `switch` có nhánh `_ => x.ToString()` thì hiện tên hằng tiếng Anh giữa giao diện tiếng Việt. Cả hai đều không ném lỗi, không log.
