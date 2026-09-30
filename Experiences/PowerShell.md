- **KHÔNG dùng toán tử `>` của PowerShell để hứng output NHỊ PHÂN** (vd `adb exec-out screencap -p > f.png`, `curl ... > f.zip`). PS 5.1 coi `>` là ghi **TEXT UTF-16LE** → file hỏng (magic bytes ra `ff fe ...` thay vì `89 50 4e 47`), báo lỗi kiểu "not a valid PNG". Dùng **Bash** cho các lệnh này, hoặc `Start-Process -RedirectStandardOutput`, hoặc để chính công cụ ghi file (`adb shell screencap -p /sdcard/f.png` rồi `adb pull`).
- **`Select-String -SimpleMatch "a|b|c"`**: `-SimpleMatch` TẮT regex nên `|` bị coi là ký tự thường → tìm cả chuỗi `"a|b|c"` nguyên văn → **0 kết quả**, dễ kết luận nhầm "không tồn tại". Muốn alternation thì BỎ `-SimpleMatch`; chỉ dùng `-SimpleMatch` khi pattern là chuỗi literal có ký tự đặc biệt regex (`.`, `[`, `(`...).
- **KHÔNG nhét here-string `@'...'@` vào lệnh một dòng có `;` hoặc block `if (...) { ... }`** (vd `git add ...; if ($?) { git commit -m @'...'@ }`). Parser cắt nhầm, nội dung here-string lọt sang lệnh ĐẦU làm đối số → lỗi kiểu `error: pathspec 'the' did not match any file(s)`. Cách chắc ăn cho message nhiều dòng: Write ra file rồi `git commit -F <file>` (hoặc để here-string đứng MỘT MÌNH trong 1 lệnh, không chain). Luật này áp cho CẢ HAI đầu: nối `; lệnh-tiếp-theo` ngay SAU dòng đóng `'@` cũng vỡ y hệt (dòng đóng phải là `'@` và không có gì khác trên dòng đó) — triệu chứng đánh lừa vì lỗi lại chỉ vào một mẩu chữ nằm giữa message, nhìn như message có ký tự đặc biệt.
- **Đo RAM tiến trình: 3 counter KHÁC NHAU, đừng so chéo** (đã kết luận nhầm "tối ưu làm RAM tăng" vì so `WorkingSetSize` với số Task Manager):
  - `Win32_Process.PrivatePageCount` = **Private Bytes / Commit**: bộ nhớ ảo commit RIÊNG cho process (kể cả phần đã đẩy ra pagefile). **KHÔNG** gồm file-backed.
  - `Win32_Process.WorkingSetSize` = **Working Set ĐẦY ĐỦ**: mọi page vật lý đang map — private + DLL dùng chung + **file-backed/memory-mapped**.
  - Cột **"Memory" trong Task Manager** = **active private working set**: chỉ phần private ĐANG ở RAM, đã LOẠI shared và file-backed → luôn NHỎ HƠN 2 cái trên.
  ⇒ Khi chuyển dữ liệu sang **memory-mapped file**, `WorkingSetSize` có thể TĂNG (page memmap vừa ghi đang thường trú) trong khi Task Manager GIẢM mạnh — vì page file-backed OS thu hồi được, không tính là RAM của process. **So sánh trước/sau phải dùng CÙNG một counter.**
- Khi tạo/sửa file PowerShell (`.ps1`) có ký tự tiếng Việt (hoặc chạy trên Windows PowerShell 5.1), luôn lưu file dưới dạng **UTF-8 có BOM** (`EF BB BF`). File không BOM bị PS 5.1 đọc theo codepage ANSI → comment tiếng Việt giải mã sai → lỗi parser kiểu `Missing closing '}'`. Tool Write tạo UTF-8 không BOM, nên sau khi Write phải convert lại: `[System.IO.File]::WriteAllText($p,[System.IO.File]::ReadAllText($p,[System.Text.UTF8Encoding]::new($false)),[System.Text.UTF8Encoding]::new($true))`.
  - **Hậu quả tệ hơn lỗi parser: script CHẠY BÌNH THƯỜNG mà dữ liệu ra mojibake.** Chuỗi tiếng Việt trong script đã hỏng ngay lúc PS nạp file, nên `[Text.Encoding]::UTF8.GetBytes($s)` mã hoá đúng… cái đã hỏng. Script seed dữ liệu thử kiểu này ghi thẳng `Cá»•ng chÃ­nh` vào database, không lỗi, không cảnh báo — rồi mọi phép kiểm sau đó "MISS" và mất hàng giờ đi tìm bug trong ứng dụng (2026-07-29: đúng vết xe đổ của `Experiences/sqlcmd.md`).
  - **Luật rút ra: script kiểm thử có chữ tiếng Việt thì viết bằng bash + curl**, đừng dùng PS 5.1. Nếu buộc phải dùng PS, đưa payload qua **file** do tool Write tạo, đừng để chuỗi nằm trong .ps1.- **Hạ RAM của một process đang chạy mà không kill nó** (hữu ích khi process map file lớn: memmap replay buffer, DB, video): gọi `EmptyWorkingSet` từ `psapi.dll` — đẩy toàn bộ working set sang **standby list**, Windows lập tức tính là "Available".
  ```powershell
  Add-Type -Namespace Win32 -Name Mem -MemberDefinition '
  [DllImport("psapi.dll", SetLastError=true)]
  public static extern bool EmptyWorkingSet(IntPtr hProcess);'
  [Win32.Mem]::EmptyWorkingSet((Get-Process -Id $pid).Handle)
  ```
  - **An toàn**: page bẩn được ghi xuống đĩa TRƯỚC khi bỏ khỏi working set ⇒ không mất dữ liệu. Chỉ tốn re-fault khi process cần lại.
  - Đo thật (train PyTorch + memmap 25 GB): WorkingSet 18.603 → 256 MB, Available 0 → 9.929 MB, tốc độ tụt ~9% rồi bò lên lại.
  - Working set **PHÌNH LẠI** dần khi process chạm lại các page đó → phải gọi lặp nếu muốn giữ RAM trống.
  - **Chẩn đoán trước khi làm**: nếu `WS Private` ≪ `Working Set` (vd 783 MB vs 18 GB) thì phần chênh là **file-backed, OS tự thu hồi được** — không phải leak, và EmptyWorkingSet chỉ *đẩy sớm* việc OS sẽ tự làm. Nếu `WS Private` mới là phần lớn thì EmptyWorkingSet vô dụng, phải sửa code.

## Sửa file UTF-8 bằng `Get-Content -Raw` + `Set-Content` làm hỏng tiếng Việt

`Get-Content` trong PS 5.1 đọc file UTF-8 **không BOM** theo codepage ANSI, `Set-Content -Encoding utf8`
lại ghi ra UTF-8 → mỗi vòng đọc-ghi **double-encode** một lần: `Không` → `KhÃ´ng` → `KhÃƒÂ´ng`...
Nguy hiểm vì file vẫn hợp lệ về cú pháp (XML/JSON/C# vẫn build được), chỉ có chữ tiếng Việt trong
comment/chuỗi bị hỏng — rất dễ lọt vào commit mà không ai thấy.

Dùng .NET API, chỉ định encoding cả hai chiều:
```powershell
$raw = [System.IO.File]::ReadAllText($p, [System.Text.UTF8Encoding]::new($false))
[System.IO.File]::WriteAllText($p, ($raw -replace 'A','B'), [System.Text.UTF8Encoding]::new($false))
```
Hoặc dùng tool Edit/Write của Claude Code thay vì viết vòng lặp PowerShell.

Phát hiện file đã hỏng (mojibake là các byte `C3 83`/`C3 85` liền nhau):
```bash
grep -rlP '[\xC3][\x83-\x85]' --include="*.cs" --include="*.props" .
```

## `Add-Type -TypeDefinition` trong Windows PowerShell 5.1 chỉ hiểu C# 5

`powershell.exe` (5.1) biên dịch qua CodeDom compiler **C# 5** — KHÔNG phải Roslyn. Mọi cú pháp C# 6+
đều lỗi cú pháp khó hiểu (`{ or ; expected`, `Invalid token '=' in class...`, `Method must have a
return type`), dù code hoàn toàn hợp lệ với `dotnet build`:

- expression-bodied member: `public bool X { get => _x; set { ... } }`, `void F() => ...`
- auto-property initializer: `public List<T> Items { get; } = new List<T>();`
- null-conditional: `PropertyChanged?.Invoke(...)`
- string interpolation `$"..."`, `nameof`, out-var `int.TryParse(s, out var n)`

Viết bằng cú pháp cũ: `get { return _x; }`, gán khởi tạo trong constructor, và tạo biến tạm cho
event (`var h = PropertyChanged; if (h != null) h(this, ...);`).

`pwsh` (PowerShell 7+) dùng Roslyn nên KHÔNG bị — nếu cần C# hiện đại thì chạy bằng `pwsh`, hoặc
tách hẳn ra file `.cs` và build bằng `dotnet`.

Bẫy kèm theo: khi `Add-Type` fail, script vẫn chạy tiếp và mọi kiểm chứng phía sau cho kết quả sai
lệch (`New-Object` trả null → binding báo `PathError`, đếm ra `0/0`) — dễ kết luận nhầm là logic sai
thay vì compile lỗi. Đặt `$ErrorActionPreference='Stop'` hoặc kiểm tra type đã load trước khi tin số liệu.

## `-replace` KHÔNG phân biệt hoa thường — refactor định danh phải dùng `-creplace`

`$text -replace 'Accounts','Profiles'` khớp cả `accounts`, `ACCOUNTS`… vì `-replace` mặc định là
case-insensitive (như `-imatch`). Dùng nó để đổi tên định danh trong code sẽ **viết hoa cả biến
local**: `IReadOnlyList<T> accounts` biến thành `Profiles`, trùng tên property và gây lỗi biên dịch
khó truy (`'IReadOnlyList<T>' does not contain a definition for 'Clear'`).

Luôn dùng `-creplace` (case-sensitive) cho mọi phép thay tên type/biến, và chạy phép dài trước phép
ngắn (`VeoAccountPool` trước `VeoAccount`) để không chồng lấn.

## `@( @('a','b') )` bị làm phẳng — bảng phép thay dạng mảng-lồng phải ép kiểu

PowerShell unroll mảng lồng một phần tử: `@( @('tim','thay') )` cho ra mảng **2 chuỗi**, không phải
mảng chứa 1 cặp. Khi đó `foreach ($pair in $list) { $t -creplace $pair[0], $pair[1] }` lấy `$pair[0]`
là **ký tự đầu** của chuỗi và `$pair[1]` là ký tự thứ hai ⇒ thay từng chữ cái trên toàn file
(`Provider` → `rrovider`, `Pool` → `oool`). File hỏng im lặng, build vẫn qua nếu chưa biên dịch lại.

Cách an toàn: `,@('tim','thay')` (dấu phẩy đầu), hoặc `[object[]]`/`ArrayList`, hoặc bỏ hẳn mảng lồng
— dùng `$t.Replace($from,$to)` (literal, không regex) gọi riêng từng cặp. Sau mọi phép thay hàng loạt
phải `git diff` kiểm tra trước khi tin là đúng.

## UIAutomation: cửa sổ modal có `Owner` KHÔNG nằm ở `RootElement.Children`

Kịch bản kiểm thử UI hay viết: tìm cửa sổ theo `ProcessId` ở `TreeScope::Children` của
`RootElement`, rồi lọc theo `Current.Name`. Cách này thấy cửa sổ chính, nhưng **không thấy hộp thoại
modal đã đặt `Owner`** (WPF `ShowDialog()` với `window.Owner = ...`): cây UIAutomation gắn nó vào
dưới cửa sổ chủ, không phải dưới desktop. Kết quả là script báo "cửa sổ không mở được" trong khi nó
đang hiện rành rành trên màn hình — rồi mất thời gian đi tìm lỗi XAML không hề tồn tại.

Cách kiểm chứng đúng: duyệt `TreeScope::Descendants` của **cửa sổ chủ** và tìm theo tên control
(nút, checkbox, tiêu đề nhóm) thay vì tìm theo title cửa sổ:

```powershell
$textCond = New-Object System.Windows.Automation.PropertyCondition(
    [System.Windows.Automation.AutomationElement]::ControlTypeProperty,
    [System.Windows.Automation.ControlType]::Text)
$owner.FindAll([System.Windows.Automation.TreeScope]::Descendants, $textCond) |
    ForEach-Object { $_.Current.Name }
```

Đổ hết text của cây ra như trên cũng là cách nghiệm thu bố cục rẻ nhất khi không chụp được màn hình:
thấy đủ nhãn nút, tiêu đề cột và nội dung dòng đầu là đủ kết luận cửa sổ render đúng.

## KHÔNG kill tiến trình theo bộ lọc "cửa sổ rỗng" — nó giết cả trình duyệt của người dùng

`Get-Process chrome | Where-Object { $_.MainWindowTitle -eq "" } | Stop-Process -Force` **làm sập
mọi tab Chrome người dùng đang mở**. Chrome/Edge/VSCode chạy nhiều tiến trình con (renderer, GPU,
utility) và *chỉ tiến trình cửa sổ chính mới có `MainWindowTitle`* — toàn bộ tiến trình con đều lọt
bộ lọc. Đã vấp thật khi dọn Chrome headless của chính mình (2026-08-08).

Cách đúng: chỉ kill đúng PID mình tạo ra.

```powershell
$p = Start-Process -FilePath $exe -ArgumentList $args -PassThru -WindowStyle Hidden
Set-Content "$dir\app.pid" $p.Id      # lưu lại nếu cần kill ở lệnh sau
Stop-Process -Id $p.Id -Force -ErrorAction SilentlyContinue
```

Với trình duyệt còn phải thêm `--user-data-dir=<thư mục riêng>`: thiếu nó, Chrome thấy đã có phiên
nên chuyển yêu cầu sang tiến trình cũ rồi tự thoát — vừa không kiểm soát được tiến trình mình vừa
tạo, vừa đụng thẳng vào cửa sổ người ta đang dùng.

Cùng ý đó, lọc tiến trình dev server nên bám vào `CommandLine` chứ không bám tên ảnh:
`Get-CimInstance Win32_Process -Filter "Name='dotnet.exe'" | Where-Object { $_.CommandLine -like "*<TenProject>*" }`.

## `Invoke-RestMethod -SkipCertificateCheck` KHÔNG có trong Windows PowerShell 5.1

Tham số đó chỉ có từ PowerShell 7. Ở 5.1 nó ném `ParameterBindingException`, **nhưng phần còn lại
của dòng lệnh vẫn chạy tiếp** — nên kết quả in ra trông y hệt "API trả về rỗng" chứ không giống một
lỗi cú pháp, rất dễ đi sửa nhầm sang phía server. Với endpoint HTTPS chứng chỉ dev thì dùng `curl -k`
(Git Bash) hoặc gọi qua cổng HTTP.

## `Get-ChildItem ... | Remove-Item -Force` có thể bị chặn ở mức sandbox

Đường ống mà bộ lọc không khớp file nào dễ bị diễn giải thành đường dẫn `*` và bị chặn với
`system path '*' is blocked`. Duyệt tên tường minh thì chắc chắn hơn:

```powershell
foreach ($i in 1..7) {
    $f = Join-Path $dir "zztest-logo-$i.png"
    if (Test-Path $f) { Remove-Item -LiteralPath $f -Force }
}
```

## Truy vấn SQL Server không cần module `SqlServer`

`Invoke-Sqlcmd` đòi module riêng. Không có thì gọi thẳng ADO.NET — nằm sẵn trong .NET Framework nên
PS 5.1 dùng được ngay, tiện cho việc seed/dọn dữ liệu thử:

```powershell
$conn = New-Object System.Data.SqlClient.SqlConnection "Data Source=.;Initial Catalog=Db;Integrated Security=True;Encrypt=False;"
$conn.Open()
$cmd = $conn.CreateCommand(); $cmd.CommandText = "SELECT COUNT(*) FROM T"
$cmd.ExecuteScalar()
$conn.Close()
```

Lưu ý đi kèm: chuỗi tiếng Việt trong `.ps1` vẫn dính bẫy encoding ở mục trên — payload có dấu thì
đưa qua file hoặc dùng bash.

## `$PSScriptRoot` RỖNG khi gọi từ dòng lệnh — đừng dùng nó để dựng tham số cho script

`$PSScriptRoot` chỉ có giá trị **bên trong** một file `.ps1` đang chạy. Gõ ở dòng lệnh (hoặc trong
lệnh một dòng do tool sinh ra) nó là chuỗi rỗng, nên `.\Script.ps1 -LogFile (Join-Path $PSScriptRoot 'x.log')`
chết ngay ở bước **bind tham số**:

```
Join-Path : Cannot bind argument to parameter 'Path' because it is an empty string.
```

Bẫy nằm ở chỗ script **chưa hề chạy một dòng nào** — dễ tưởng script lỗi và đi sửa nhầm bên trong nó.
Truyền đường dẫn **tuyệt đối** khi gọi từ ngoài; để `$PSScriptRoot` cho giá trị mặc định của `param()`
bên trong script (ở đó nó đúng). Cần thư mục script tại dòng lệnh thì dùng `$PWD` hoặc
`Split-Path (Resolve-Path .\Script.ps1)`.

## Commit message dài đưa qua **file** (`git commit -F`), đừng đưa qua here-string

Here-string `@'...'@` truyền thẳng cho `git commit -m` chạy được với phần lớn nội dung, nhưng đủ để
vỡ khi trong thân có **dấu nháy kép**: PowerShell không nhận đó là here-string nữa, chuỗi bị cắt tại
dấu nháy đầu tiên và phần còn lại của message đi vào git thành **pathspec**. Triệu chứng đọc ra là
một câu hoàn toàn không liên quan:

```
error: pathspec 'sponsor button would be a write to the database in the middle ... ' did not match any file(s) known to git
```

Bẫy nằm ở chỗ commit **không** được tạo, mà thông báo lại nói về đường dẫn file — dễ tưởng mình
stage nhầm và đi sửa `git add`. Cách chắc chắn: `Write` message ra một file trong scratchpad rồi
`git commit -F <đường dẫn tuyệt đối>`. Nó miễn nhiễm với mọi ký tự trong thân message (nháy đơn,
nháy kép, backtick, `$`) và còn đọc lại được nguyên văn trước khi commit.

## `-like` coi `[...]` là LỚP KÝ TỰ — kiểm chuỗi có ngoặc vuông phải dùng `-eq`

`-like`/`-notlike` là wildcard, và trong wildcard `[abc]` nghĩa là "một ký tự thuộc tập". Nên
`$ver -like '*@[9.0.2]'` KHÔNG kiểm "kết thúc bằng `@[9.0.2]`" mà là "`@` rồi MỘT ký tự thuộc
`{9, ., 0, 2}`" → mọi giá trị đúng đều bị báo **sai**. Đã vấp khi verify dependency version của
nupkg (`[9.0.2]`): 28/28 gói đúng nhưng script báo 28 gói sai.

So sánh chuỗi cố định thì dùng `-eq` (hoặc `.Contains()`/`-clike` nếu cần). Cần literal trong wildcard
thì escape bằng backtick: `'*@`[9.0.2`]'`. Cùng họ bẫy này: `-match` là **regex**, `[9.0.2]` ở đó cũng
là lớp ký tự.

## Dựng ảnh PNG thử nghiệm bằng System.Drawing

Cần vài file ảnh để thử luồng upload/hiển thị mà không muốn tải từ đâu về:

```powershell
Add-Type -AssemblyName System.Drawing
$bmp = New-Object System.Drawing.Bitmap 420, 120
$g = [System.Drawing.Graphics]::FromImage($bmp)
$g.Clear([System.Drawing.Color]::Transparent)
$g.DrawString("LOGO", (New-Object System.Drawing.Font "Segoe UI", 34, ([System.Drawing.FontStyle]::Bold)),
              [System.Drawing.Brushes]::Black, 0, 0)
$g.Dispose()
$bmp.Save("$dir\logo.png", [System.Drawing.Imaging.ImageFormat]::Png)
$bmp.Dispose()
```

## Tên biến PowerShell KHÔNG phân biệt hoa/thường — `$S` và `$s` là cùng một biến

```powershell
$S = "C:\deploy"
foreach ($n in 'api','web') {
    $s = (Get-ChildItem "$S\$n" -Recurse -File | Measure-Object Length -Sum).Sum   # ← ghi đè $S
    "{0} {1:N1} MB" -f $n, ($s/1MB)
}
```

Vòng lặp đầu chạy đúng, vòng thứ hai `$S` đã biến thành **con số** (tổng byte) nên `"$S\$n"` ra
`"130254027\web"` → `Get-ChildItem` báo `PathNotFound` với một đường dẫn vô nghĩa mà chẳng dòng nào
trong script tạo ra. Rất dễ đi nghi ngờ nội suy chuỗi hoặc quyền truy cập.

Cùng họ bẫy này: `$Path`/`$path`, `$Root`/`$root`, `$Name`/`$name`. Quy tắc: trong một scope chỉ dùng
**một** cách viết cho mỗi tên, và đặt tên biến "khung" (thư mục gốc, cấu hình) dài rõ ràng
(`$deployRoot`) thay vì một chữ cái để không đụng biến tạm.

## Biến tự động bị gán đè: `$input`, `$pwd`, `$profile`, `$args`, `$error`, `$host`, `$matches`

Đặt tên biến local trùng biến tự động là **hợp lệ về cú pháp**, chạy vẫn ra kết quả, nên không có
gì báo cho biết — hỏng thì hỏng ngầm ở chỗ khác:

- `$input` = enumerator của pipeline. Gán đè (`$input = [IO.File]::OpenRead($p)`) làm mọi hàm nhận
  dữ liệu qua pipeline trong cùng scope đọc ra rác.
- `$pwd` = thư mục hiện tại. Đè nó thì `Get-ChildItem`/`Join-Path` dùng đường dẫn tương đối đi lạc.
- `$profile` = đường dẫn file profile của PowerShell (rất hay bị đè khi làm việc với publish profile).
- Còn `$args`, `$error`, `$host`, `$matches`, `$psitem`, `$this`.

Cách bắt: **PSScriptAnalyzer** báo `PSAvoidAssignmentToAutomaticVariable`, và trong VS Code nó chạy
sẵn — mở file `.ps1` ra là thấy cảnh báo ngay, rẻ hơn nhiều so với đi tìm lỗi lúc chạy. Đặt tên
theo nghĩa cụ thể (`$source`/`$destination` thay `$input`, `$profileInfo` thay `$profile`,
`$decoded` thay `$pwd`) là hết.

## `$mang[-1]` khi mảng chỉ có MỘT phần tử trả về ký tự cuối của chuỗi

Lệnh lọc trả về một phần tử duy nhất thì PowerShell **không** gói nó thành mảng — nó trả thẳng chuỗi.
Lấy `[-1]` trên chuỗi là lấy **ký tự** cuối:

```powershell
$names = $lines | ForEach-Object { ($_ -split '\s+')[-1] } | Where-Object { $_ -match '\.txt$' }
$last = $names[-1]      # chi co 1 file -> $names la String -> $last = "t"
```

Triệu chứng: đường dẫn ghép ra thành `.../logs/t` rồi FTP/file system báo **550 / not found** với một cái
tên vô nghĩa dài đúng một ký tự. Rất dễ đi nghi ngờ quyền truy cập hoặc lỗi mạng.

Chữa: luôn ép thành mảng khi cần index — `@($names)[-1]`, hoặc dùng `Select-Object -Last 1`
(an toàn với mọi số lượng phần tử). Cùng họ bẫy này: `$x.Count` trên một phần tử trả về độ dài chuỗi
chứ không phải 1.

## Xoá thư mục có path vượt MAX_PATH (node_modules, .pnpm...) — dùng `robocopy /MIR`

`Remove-Item -Recurse -Force` và `rm -rf` đều **chết giữa chừng** khi cây thư mục chứa path dài hơn 260 ký tự
(điển hình: `node_modules/.pnpm/<pkg>@<ver>_<hash>/node_modules/next/dist/esm/client/.../LeftRightDialogHeader/`).
Triệu chứng: lỗi `Filename too long` / `PathTooLongException`, xoá được một phần rồi bỏ dở, chạy lại vẫn kẹt.

Cách xoá chạy được: mirror một thư mục RỖNG đè lên target (robocopy dùng API path dài), rồi mới xoá vỏ:

```powershell
$empty = Join-Path $env:TEMP "empty_mirror"
New-Item -ItemType Directory $empty -Force | Out-Null
robocopy $empty $target /MIR /NFL /NDL /NJH /NJS /NC /NS /NP | Out-Null
if ($LASTEXITCODE -lt 8) { Remove-Item $target -Recurse -Force; Remove-Item $empty -Recurse -Force }
```

**BẪY exit code**: robocopy KHÔNG theo quy ước 0 = thành công. `0` = không có gì để làm, `1` = đã copy,
`2` = phát hiện file/thư mục thừa (khi /MIR xoá sạch target thì thường trả **2** — đây là THÀNH CÔNG).
Chỉ `>= 8` mới là lỗi thật. Đừng kiểm `if ($LASTEXITCODE -eq 0)`, cũng đừng hoảng khi tool báo "Exit code 2".

## Dừng console cho người dùng đọc (pause) — 3 bẫy

Script chạy bằng bấm đôi chuột hoặc tự nâng quyền qua UAC sẽ mở cửa sổ riêng và **đóng ngay** khi
chạy xong, không kịp đọc gì. Muốn nó dừng lại thì phải tránh đủ 3 bẫy sau.

**Bẫy 1 — `ReadKey` đọc bộ đệm bàn phím của console, KHÔNG đọc stdin.** Nên
`$Host.UI.RawUI.ReadKey('NoEcho,IncludeKeyDown')` sẽ **treo vô hạn** khi script được gọi tự động
(`< /dev/null`, `echo x | powershell -File ...`, tác vụ định kỳ, agent). Tệ hơn:
`[Environment]::UserInteractive` trong mấy ca đó vẫn trả **True** nên kiểm mỗi nó là chưa đủ. Phải
kiểm thêm `[Console]::IsInputRedirected` và bỏ qua pause khi nó True:

```powershell
if (-not [Environment]::UserInteractive) { return }
if ([Console]::IsInputRedirected) { return }   # <- cái này mới cứu được
try   { $null = $Host.UI.RawUI.ReadKey('NoEcho,IncludeKeyDown') }
catch { Read-Host 'Nhan Enter de dong' | Out-Null }   # ISE/VS Code khong co RawUI -> nem loi
```

Cách kiểm chứng nhanh 2 chiều: `timeout 25 powershell -File x.ps1 < /dev/null` (trả 124 là đang treo)
và `Start-Process powershell -ArgumentList ... -PassThru` rồi `Start-Sleep 6; $p.HasExited` (còn sống
tức là đang chờ bấm phím đúng như mong muốn).

**Bẫy 2 — `exit` VẪN chạy khối `finally`.** Đã thử: hàm gọi `exit 3` bên trong `try` thì `finally` in
ra bình thường rồi mới thoát với đúng exit code. Hữu ích (đặt pause trong `finally` thì lỗi giữa
chừng vẫn dừng cho đọc) nhưng cũng là bẫy: script tự nâng quyền kiểu
`Start-Process -Verb RunAs; exit 0` sẽ khiến **cả hai** cửa sổ cùng chờ bấm phím. Chữa bằng một cờ
`$Global:` đặt ngay trước `exit`, hàm pause thấy cờ thì thoát luôn — đã kiểm: biến `$Global:` đặt
trong hàm vẫn đọc được từ `finally` của script.

**Bẫy 3 — `$ErrorActionPreference = 'Stop'`** làm lỗi giữa chừng nhảy thẳng ra ngoài, bỏ qua mọi dòng
pause viết ở cuối file. Đúng lúc cần đọc lỗi nhất thì cửa sổ đóng. Luôn bọc thân script bằng
`try { ... } catch { in loi } finally { pause }`.

## `.Length` của `Get-Content -Raw` KHÔNG phải số ký tự — đừng lấy nó đi so bản

Cùng gốc với mục trên: `Get-Content -Raw` đọc file UTF-8 **không BOM** theo codepage ANSI, nên mỗi
byte thành một "ký tự". Với văn bản tiếng Việt, `.Length` vì thế phình lên ~30% so với số ký tự
thật — `9782` byte hiện thành `9780`, trong khi chuỗi thật chỉ `7527` ký tự.

Bẫy thật sự là lúc **so một file trên đĩa với một chuỗi lấy từ nơi khác** (database qua `psql`, API
qua `Invoke-RestMethod`, `git show`) — những nguồn đó trả về chuỗi đã giải mã UTF-8 đúng. So hai con
số `.Length` sẽ kết luận "hai bên lệch nhau" trong khi chúng giống hệt, và cái kết luận sai đó nghe
rất thuyết phục vì có số kèm theo.

Cách so đúng — đọc file bằng .NET với encoding nói rõ:
```powershell
$file = [System.IO.File]::ReadAllText($p, [System.Text.UTF8Encoding]::new($false))
```
Hoặc so nội dung chứ đừng so độ dài: ghi chuỗi kia ra tệp UTF-8 bằng
`[System.IO.File]::WriteAllText(...)` rồi `diff` hai tệp. Nhớ chuẩn hoá `\r\n` và dòng trống cuối
tệp trước khi kết luận: chuỗi lưu trong database thường đã bị trim mất `\n` cuối, nên `diff` báo
`\ No newline at end of file` là chuyện bình thường, không phải nội dung khác nhau.
