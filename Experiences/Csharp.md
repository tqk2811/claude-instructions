# Kinh nghiệm xử lý lỗi C# / WPF

## EF Core: cùng một điều kiện lọc chạy hai nơi thì ngữ nghĩa so chuỗi KHÁC nhau
- `value.Contains(keyword)` trong LINQ-to-Entities → EF dịch thành `LIKE N'%keyword%'`, theo collation cột nên **thường KHÔNG phân biệt hoa thường**.
- Cùng dòng đó chạy in-memory trên object đã load → `StringComparison.Ordinal`, **CÓ phân biệt hoa thường**.
- Hậu quả: bản ghi bị lọc khỏi danh sách hiển thị nhưng lọt qua kiểm tra nghiệp vụ ở endpoint khác — trông như "ẩn rồi mà vẫn thao tác được". Chỉ lộ khi dữ liệu nguồn viết hoa thường không nhất quán.
- Cách tránh: helper dùng chung cho phía in-memory, nêu rõ ngữ nghĩa; phía EF để nguyên `Contains` (EF không dịch được overload có `StringComparison`) kèm chú thích là dựa vào collation.
  ```csharp
  private static bool ContainsKeyword(string value, string keyword)
      => !string.IsNullOrEmpty(keyword) && !string.IsNullOrEmpty(value)
         && value.IndexOf(keyword, StringComparison.OrdinalIgnoreCase) >= 0;
  ```
- Cùng loại bẫy: `==`, `StartsWith`, `EndsWith`, và dấu tiếng Việt (collation `..._CI_AI` coi "Thứ" == "Thu").

## EF Core: kiểm tra concurrency token TRƯỚC khi kết luận có race condition
- Thấy `đọc → so sánh → sửa → SaveChanges` thì đừng vội báo race. Nếu entity có `byte[] RowVersion` cấu hình `.IsRowVersion()` thì EF sinh `UPDATE ... WHERE RowVersion = @rv` → request thua khớp 0 dòng, ném `DbUpdateConcurrencyException`, không ghi đè được.
- Phải mở entity + `IEntityTypeConfiguration` ra xem trước khi kết luận.
- Cái thường vẫn hở: **nhiều `SaveChanges` rời nhau không transaction** — bước 1 xong, bước 2 lỗi → dữ liệu lệch. Gộp về một `SaveChanges` là EF tự bọc transaction ngầm.

## EF Core CLI: `migrations add --no-build` với startup-project CHƯA build lại → migration RỖNG, và `migrations remove --force` sau đó gỡ nhầm migration TRƯỚC + revert nó khỏi DB
- Đã vấp thật, mất một migration đã commit và một cột trong DB dev.
- Chuỗi sự việc: sửa entity → `dotnet build <DbProject>` → `dotnet ef migrations add X --startup-project <Api> --no-build`. EF nạp model từ **output của startup-project**, mà bản copy `<DbProject>.dll` trong `bin` của Api vẫn là bản cũ ⇒ model không có property mới ⇒ `Up()`/`Down()` **rỗng**, không cảnh báo gì.
- Phản xạ "gỡ ra làm lại" bằng `dotnet ef migrations remove --force` thì tệ hơn nhiều: `--force` cho phép **revert migration đã apply** — nó chạy `Down()` trên database — và với model cũ đó, thứ nó coi là "migration cuối" lại là migration **trước đó**. Kết quả: file migration đã commit bị xoá khỏi đĩa, cột của nó bị DROP khỏi DB dev, còn migration rỗng vừa tạo thì nằm nguyên.
- Luật:
  1. Trước `migrations add/remove`, build **startup-project** (không phải chỉ project chứa DbContext), hoặc bỏ hẳn `--no-build`.
  2. `Up()` rỗng = **model chưa tới**, không phải "không có gì để đổi". Mở file ra đọc trước khi làm gì tiếp.
  3. Migration rỗng thì **xoá tay hai file** `<stamp>_X.cs` + `.Designer.cs` rồi `git checkout --` cái snapshot, đừng gọi `migrations remove`.
  4. `--force` chỉ dùng khi biết chắc migration cuối là cái mình vừa tạo VÀ chấp nhận `Down()` chạy trên DB.
- Gỡ khi đã lỡ: `git checkout --` để lấy lại file migration bị xoá + `<App>DbContextModelSnapshot.cs`, xoá file rỗng, build lại startup-project, `migrations add` lại, rồi `database update` (nó tự apply lại cả cái vừa bị revert).

## EF Core: tên bảng thật nằm ở migration, không phải `ToTable(...)`
- `IdentityDbContext.OnModelCreating` đặt lại tên bảng Identity. Nếu `base.OnModelCreating(...)` được gọi SAU các `ApplyConfiguration(...)` thì `ToTable("AppUsers")` bị đè, bảng thật vẫn là `AspNetUsers` — dòng đó thành code chết gây hiểu nhầm.
- Nguồn sự thật: `CreateTable(name: "...")` trong file migration.

## WPF multi-target: lỗi `InitializeComponent does not exist` hàng loạt là RACE khi build song song
- Triệu chứng: project WPF multi-target (`net48;net8.0-windows;net10.0-windows`) build ra hàng loạt `error CS0103: The name 'InitializeComponent' does not exist` ở nhiều file `*.xaml.cs` **không liên quan tới file vừa sửa**; lỗi luôn kèm tên project tạm dạng `<Project>_xxxxxxxx_wpftmp.csproj` và chỉ dính **một** TFM.
- Nguyên nhân: WPF sinh project tạm `_wpftmp.csproj` để chạy pass markup-compile. Khi msbuild build nhiều TFM song song, các pass này giẫm lên nhau → pass sinh file `.g.cs` chưa xong đã bị pass khác đọc.
- KHÔNG phải lỗi code. Cách xác nhận nhanh, theo thứ tự:
  1. Build riêng đúng TFM báo lỗi: `dotnet build <proj> -f net8.0-windows7.0` → thường pass ngay.
  2. Build lại toàn bộ TFM tuần tự: `dotnet build <proj> -m:1` → pass.
- Chỉ khi build tuần tự VẪN lỗi thì mới là lỗi thật (thiếu `x:Class`, XAML sai namespace, file `.xaml` không có `Page` build action).
- Đừng vội `git stash` để so với HEAD: build sạch có thể pass chỉ nhờ khác thứ tự lịch song song, dễ kết luận nhầm là do code mới.
- Biến thể: `error MC1000: Unknown build error, 'Access to the path 'obj\...\Views\<View>.g.cs' is denied.'` — cùng gốc tranh file `.g.cs`, nhưng nguồn tranh chấp có thể là **Visual Studio đang mở solution** (design-time build / build nền của VS ghi cùng thư mục obj), lúc đó `-m:1` KHÔNG cứu được vì tiến trình tranh chấp nằm ngoài msbuild của mình. Chạy lại đúng lệnh cũ thường là qua ngay; cứ lỗi lại thì đóng VS (hoặc chờ VS build xong) rồi build lại. Đừng đi sửa XAML — file bị "denied" thường chẳng liên quan gì tới thay đổi vừa làm.
- Biến thể: `error CS2001: Source file 'obj\...\Views\<View>.g.cs' could not be found` hàng loạt (cũng kèm `_wpftmp.csproj`) — obj bị hỏng/stale sau lần build trước bị lock file (MSB3026 retry) hoặc bị chạy đè. Build lại thường không tự lành; xoá cả thư mục `obj` của project rồi build lại là hết.

## WPF: `dotnet msbuild -t:Compile` KHÔNG chạy markup-compile → "pass" GIẢ
- Khi app đang chạy khoá `bin\App.exe`, hay dùng `-t:Compile` để né bước copy. Nhưng target `Compile` chỉ chạy CSC trên các `.g.cs` **đã có sẵn trong obj** — nó KHÔNG chạy `MarkupCompilePass1/2`.
- Hậu quả: sửa XAML (hoặc thêm file `.xaml` mới) → `-t:Compile` báo thành công mà thực chất chưa hề biên dịch XAML mới; sai XAML chỉ lộ lúc runtime. Thêm UserControl mới thì nó lại lỗi ngược `CS0103: InitializeComponent does not exist` (chưa có g.cs) — dễ tưởng lỗi code.
- Cách né lock đúng: build đầy đủ nhưng đổi chỗ output — `dotnet build <proj> -p:BaseOutputPath=<thư mục tạm>\` (giữ nguyên obj để tận dụng incremental). ĐỪNG đổi luôn `BaseIntermediateOutputPath`: project tạm `_wpftmp.csproj` vẫn đọc obj gốc, gây lỗi lệch.
- Bẫy kèm theo: obj còn `<View>.g.i.cs` của file XAML **đã bị xoá** → `error CS1504: Source file ... could not be opened`. Xoá đúng mấy file `.g*.cs` mồ côi trong obj là xong (không cần clean cả obj).
- ĐỪNG dùng `-p:OutputPath` thay cho `-p:BaseOutputPath`: `OutputPath` là đường dẫn output **cuối cùng** (bỏ qua phần `<Configuration>\<TFM>\` mà `BaseOutputPath` tự nối), nên nhiều TFM/nhiều lần build ghi đè lẫn nhau vào cùng một thư mục. Hệ quả gặp thật: lần đầu chạy OK, các lần build sau cùng project ra hàng loạt `error CS2001: ... <View>.g.cs could not be found` (MarkupCompilePass tưởng đã up-to-date nên không sinh lại `.g.cs`), kèm `GenerateDepsFile task failed unexpectedly` do 3 TFM tranh nhau ghi `<Project>.deps.json` trong cùng thư mục. Chữa: `-t:Rebuild` cho project đó, và chuyển sang `-p:BaseOutputPath=<thư mục tạm>\` (hoặc mỗi TFM một thư mục riêng).

## WPF: DynamicResource KHÔNG hoạt động trên `DataGridColumn.Header`
- Triệu chứng: đặt `Header="{DynamicResource SomeKey}"` trên `DataGridTextColumn`/`DataGridTemplateColumn` → header hiển thị TRỐNG (không lỗi build, không exception).
- Nguyên nhân: `DataGridColumn` không nằm trong visual/logical tree của DataGrid (nó ở collection `Columns`), nên ResourceReference không resolve được tài nguyên (kể cả StaticResource cũng chỉ resolve 1 lần lúc load).
- Cách xử lý: nhét 1 element (đang trong tree qua header presenter) vào Header:
  ```xml
  <DataGridTextColumn Binding="{Binding Foo}">
      <DataGridTextColumn.Header><TextBlock Text="{DynamicResource SomeKey}"/></DataGridTextColumn.Header>
  </DataGridTextColumn>
  ```
  TextBlock nằm trong header presenter (thuộc visual tree) nên DynamicResource resolve & cập nhật live được. Áp dụng cho localization swap ResourceDictionary.
- MenuItem.Header / TabItem.Header / Label.Content / Button.Content / ToolTip / TextBox.Tag đều là FrameworkElement trong tree → DynamicResource hoạt động bình thường.

## WPF localization runtime-switch bằng swap ResourceDictionary
- Text tĩnh trong XAML: `{DynamicResource Str.Key}` → đổi ngôn ngữ live tức thì khi swap dictionary.
- Chuỗi dựng trong ViewModel/code-behind (MessageBox, format string, enum ToString baked vào property): KHÔNG tự đổi — phải raise event `LanguageChanged` rồi cho VM rebuild (reload collection / gán lại property). Enum trong DataGrid cell dùng converter tra key `Enum.<Type>.<Value>` và refresh khi reload data.
- File `.xaml` ResourceDictionary (không x:Class) đặt trong project với `UseWPF=true` được tự động include là Page (giống Themes/*.xaml), truy cập qua `pack://application:,,,/Folder/File.xaml`, không cần khai báo trong .csproj.

## OS culture khi chạy app để test
- Chạy exe qua **Bash tool** (Git Bash) có thể cho `CultureInfo.CurrentUICulture` = English (do biến môi trường LANG/LC_*).
- Chạy qua **PowerShell `Start-Process`** cho đúng culture Windows của user.
- → Khi test tính năng phụ thuộc culture (vd ngôn ngữ "System/Auto"), dùng PowerShell để phản ánh đúng môi trường thật.

## Chụp ảnh cửa sổ WPF để verify
- `CopyFromScreen` chụp theo tọa độ màn hình → dễ bị cửa sổ khác (browser…) đè lên; `SetForegroundWindow` thường bị Windows chặn không kéo được app lên trên.
- `PrintWindow(h, hdc, 0x2 /*PW_RENDERFULLCONTENT*/)` render nội dung cửa sổ bất kể bị che → chụp được vùng content, NHƯNG phần overlay title-bar/WindowChrome caption có thể ra trắng. Dùng kết hợp: PrintWindow để xem nội dung, CopyFromScreen khi app chắc chắn ở foreground.

## Thư viện concurrency nội bộ — `WorkQueue<T>.MaxRun` KHÔNG phải giới hạn cứng
- `WorkQueue<T>.RunNewWork()` (bản 1.0.1) kiểm tra `_Runnings.Count >= MaxRun` **ngoài vùng khoá**, rồi `StartQueue()` mới thêm vào `_Runnings` trong một khoá **khác** (`_lock_runnings`). Hai lời gọi điều phối đồng thời (từ `Add`, từ `WorkCompleted`, và từ nhánh đệ quy) cùng vượt qua kiểm tra ⇒ khởi động dư job.
- **Thực đo**: đặt `MaxRun = 2` vẫn quan sát được 3 job chạy song song.
- Hầu hết trường hợp vượt tạm thời 1 job là vô hại. Nhưng khi số luồng song song là **ràng buộc nghiệp vụ** (giới hạn phiên trình duyệt để tránh bị site ban, giới hạn kết nối tới API tính tiền theo concurrency, giới hạn tài nguyên phần cứng) thì phải tự chặn thêm.
- Cách chặn: một semaphore async bao quanh phần thân công việc, `WaitAsync` **trước** khi chiếm tài nguyên dùng chung (account/kết nối), `Release` trong `finally`. `SemaphoreSlim` không đổi được `CurrentCount` lúc chạy ⇒ nếu cần chỉnh giới hạn động thì viết semaphore riêng (lock + `Queue<TaskCompletionSource<bool>>`; khi Pump gặp waiter đã huỷ thì `TrySetResult` trả false, bỏ qua, không tiêu suất).
- Bài học chung: với thư viện hàng đợi/điều phối, **đừng mặc định coi tham số "max" là bất biến cứng** — viết một test đo đỉnh concurrency thật (đếm vào/ra quanh `Task.Delay`) rồi mới tin.

## HttpClient: `ResponseHeadersRead` + đọc body không token/timeout = treo vô hạn
- `HttpClient.Timeout` chỉ phủ tới lúc `SendAsync` trả về. Với `HttpCompletionOption.ResponseHeadersRead`, SendAsync trả về NGAY KHI CÓ HEADER → phần đọc body sau đó (`ReadAsStringAsync`/`ReadAsByteArrayAsync`/đọc stream) **không còn timeout nào bảo vệ**.
- Gặp kết nối nửa mở (proxy/mạng chết giữa chừng sau khi đã trả header, không FIN/RST) thì `ReadAsStringAsync()` **không token** treo vĩnh viễn: không exception, không log, thread/loop đứng im. App "chạy mà như chết", RAM bình thường; Ctrl+C phải chờ `HostOptions.ShutdownTimeout` (30s) vì await đó miễn nhiễm cancellation. Thực tế: BotCrawler Lcukos9h đơ 21:06→22:57 (22/07/2026).
- Triệu chứng nhận dạng qua log HttpClientFactory: vẫn thấy `Received HTTP response headers ... 200` + `End processing` (log này bắn lúc NHẬN HEADER) rồi im bặt — dễ tưởng request thành công.
- Quy tắc: mặc định dùng `ResponseContentRead` (buffer trọn body trong SendAsync, Timeout phủ hết); chỉ dùng `ResponseHeadersRead` khi thật sự cần streaming, và khi đó PHẢI tự bọc đọc body bằng CTS timeout + truyền CancellationToken vào mọi lệnh đọc.
- Đây cũng là lý do đừng tin catch quanh HTTP call là đủ: catch chỉ bắt được exception, không bắt được await không bao giờ trả về.

## MSTest 3.8+ / 4.x: analyzer MSTEST0037 bắt dùng assert chuyên biệt
- `Assert.AreEqual(0, list.Count)` → `Assert.IsEmpty(list)`; `Assert.AreEqual(n, list.Count)` → `Assert.HasCount(n, list)`; `Assert.IsTrue(a > b)` → `Assert.IsGreaterThan(b, a)`; `Assert.IsTrue(a <= b)` → `Assert.IsLessThanOrEqualTo(b, a)`.
- Lưu ý **thứ tự tham số ngược trực giác**: `IsGreaterThan(lowerBound, value)`, `IsLessThanOrEqualTo(upperBound, value)` — giá trị cần kiểm tra đứng SAU.

## WPF: đổi `ShowInTaskbar` lúc RUNTIME làm hiện cửa sổ trắng tên `HiddenWindow`
- WPF không đổi được `Window.ShowInTaskbar` sau khi cửa sổ đã hiện: setter **huỷ rồi tạo lại HWND**. Khi cờ = `false` mà cửa sổ không có Owner, WPF gắn nó vào window nội bộ tên `HiddenWindow` → lúc tạo lại HWND cái đó lòi ra thành **cửa sổ trắng rỗng** trên màn hình/Alt-Tab.
- Tạo lại HWND còn kéo theo: mất focus, mất `Topmost`/Owner, có thể phá trạng thái modal của `ShowDialog()`.
- Quy tắc: chỉ đặt `ShowInTaskbar` **trong XAML/ctor trước khi Show lần đầu**; cần "ẩn tạm" cửa sổ thì dùng `Opacity = 0` + `IsHitTestVisible = false` + đẩy `Left/Top` ra ngoài màn hình (vd `-32000`), KHÔNG động vào `ShowInTaskbar`.
- Bối cảnh gặp: hàm `FakeHide()` giấu form WPF trong lúc user pick đối tượng AutoCAD (không được dùng `Hide()` vì `Hide()` thoát vòng lặp modal → `ShowDialog()` trả về `false` ngay).

## WPF: `WindowStyle=None` + maximize = đáy cửa sổ chui xuống dưới taskbar
- Cửa sổ tự vẽ khung (`WindowStyle="None"`, kể cả khi đã dùng `WindowChrome`) lúc phóng to được Windows căng ra **toàn bộ màn hình** chứ không phải vùng làm việc, lại còn nống thêm bằng bề dày khung ⇒ phần đáy (status bar) nằm dưới taskbar, mép trái/phải cũng bị cắt.
- Fix chuẩn: hook `WM_GETMINMAXINFO` (0x0024) trong `OnSourceInitialized` qua `HwndSource.AddHook`, rồi ghi đè `MINMAXINFO.ptMaxPosition` = `rcWork - rcMonitor` và `ptMaxSize` = kích thước `rcWork` (lấy bằng `MonitorFromWindow(MONITOR_DEFAULTTONEAREST)` + `GetMonitorInfo`). Toạ độ trong message là **pixel vật lý**, trùng đơn vị `MONITORINFO` nên KHÔNG quy đổi DPI.
- ĐỪNG siết `ptMaxTrackSize`: nó chỉ giới hạn kéo tay, siết lại sẽ chặn luôn việc kéo cửa sổ trải rộng qua nhiều màn hình — không liên quan gì tới lỗi taskbar.
- Hệ quả kèm theo: sau khi ép đúng work area thì KHÔNG cần mẹo bù `Padding` ~7px lúc maximized nữa (mẹo đó chỉ để chữa triệu chứng nống khung); để lại sẽ thành viền thừa.

## WPF: máy nhiều màn hình — `CenterScreen` + `WindowState=Maximized` luôn mở ở MÀN HÌNH CHÍNH
- Triệu chứng: cửa sổ A đang ở màn phụ, thao tác trên đó mở cửa sổ B ⇒ B luôn hiện/phóng to ở màn hình chính.
- `WindowStartupLocation="CenterScreen"` KHÔNG phải "màn hình có chuột": WPF căn giữa theo màn hình đang chứa **handle của chính cửa sổ đó** tại lúc tạo — mà lúc ấy Windows còn đặt nó ở vị trí mặc định (thường là màn chính). Khai báo sẵn `WindowState="Maximized"` trong XAML thì càng chắc chắn dính màn chính: Windows phóng to theo màn hình chứa cửa sổ, trước khi có ai kịp dời nó đi.
- `Owner` + `CenterOwner` chữa được cho hộp thoại modal, nhưng KHÔNG dùng được cho cửa sổ ngang hàng (đặt Owner là kéo theo luôn quan hệ đóng/always-on-top).
- Cách chữa: tự tra màn hình đích rồi dời bằng Win32 trước khi cửa sổ hiện.
  1. Màn hình đích = `MonitorFromWindow(handle cửa sổ nguồn, MONITOR_DEFAULTTONEAREST)`; không có cửa sổ nguồn thì `GetCursorPos` + `MonitorFromPoint` (mở theo màn hình đang có con trỏ chuột).
  2. Dời trong `SourceInitialized` — handle đã có nhưng cửa sổ **chưa hiện**, nên không thấy nháy qua màn hình khác. Cửa sổ đã từng `Show` rồi `Hide` thì `SourceInitialized` KHÔNG bắn lại ⇒ phải kiểm tra `new WindowInteropHelper(w).Handle != IntPtr.Zero` để dời thẳng.
  3. Dời bằng `SetWindowPos` + toạ độ **pixel vật lý** (`GetMonitorInfo.rcWork`, `GetWindowRect`), ĐỪNG gán `Window.Left/Top`: đơn vị của WPF là DIP theo DPI của màn hình hiện tại, hai màn khác scale là tính sai chỗ.
  4. Muốn cửa sổ phóng to: tạm hạ `WindowState` về `Normal` trước `Show()`, dời xong rồi mới gán lại `Maximized` — làm ngược thứ tự là phóng to nhầm màn hình.
- Kiểm chứng được bằng UIAutomation + `Screen.FromHandle` (kéo cửa sổ nguồn sang màn phụ, để chuột ở màn chính, rồi so `DeviceName` của cửa sổ mới) — đừng test lúc mọi thứ đang ở màn chính, kết quả trùng nhau nên không phân biệt được đúng/sai.

## WPF: `Path` với `Stretch="None"` KHÔNG tự canh giữa hình trong ô
- `VerticalAlignment/HorizontalAlignment="Center"` chỉ canh cái **ô layout**, còn nét vẽ nằm đúng toạ độ trong `Data`. Nên `Data="M0,0 H10"` trong ô 10×10 cho vạch bám mép TRÊN (nửa nét còn bị cắt), không phải ở giữa — phải viết `M0,5 H10`.
- Chỉ lộ ra ở icon vẽ một nét (dấu trừ của nút minimize, dấu gạch ngang); icon phủ kín ô (X, khung vuông) trông vẫn cân nên dễ tưởng cả bộ đều đúng.
- Muốn khỏi tính tay thì bỏ `Stretch="None"` và để `Stretch="Uniform"`, nhưng khi đó nét sẽ bị co giãn theo ô.

## WPF: 4 cái bẫy khi style lại toàn bộ control (làm theme dark/light)
- **Style ngầm khớp CHÍNH XÁC kiểu**: `<Style TargetType="ListBox">` KHÔNG áp cho `ListView` (kế thừa cũng không tính). Phải khai báo riêng.
- **`ListView` + `GridView` gần như không style lại được**: khung lưới (hàng header, `GridViewRowPresenter`) đến từ *theme style* theo `DefaultStyleKey` mà `ListView.View` gán. Style ngầm có `Setter Template` sẽ đè theme style ⇒ **mất sạch cột**. Đổi sang `DataGrid` nhanh và an toàn hơn là cố dựng lại khung GridView.
- **Cột `DataGrid` không ăn style ngầm**: `DataGridTextColumn`/`DataGridCheckBoxColumn` dựng phần tử con bằng `DefaultElementStyle`/`DefaultEditingElementStyle` — là Style **tường minh, không `BasedOn`** ⇒ style ngầm bị bỏ qua, ô lúc sửa vẫn nền trắng chữ đen trên theme tối. Phải gán tay `ElementStyle=`/`EditingElementStyle=` cho từng cột.
- **`ToolBar`/`ToolBarTray`/`StatusBar` không đáng theme lại**: template Aero gắn cứng gradient xám; thay bằng `Border` + `StackPanel` tự style thì ít việc hơn nhiều.

## WPF: `TabItem` rò trạng thái VÀ property kế thừa xuống nội dung tab
- Nội dung trang là **logical child của `TabItem`** (dù về visual tree nó nằm trong `ContentPresenter` của `TabControl`). Hai hệ quả đều gây bug thật:
- **`TabItem.IsMouseOver` bật cả khi chuột ở trong nội dung tab** — trạng thái mouse-over là *reverse-inherit property*, lan lên theo `FrameworkObject.FrameworkParent` tức **logical parent trước**. Triệu chứng: `<Trigger Property="IsMouseOver">` trong template TabItem làm **header nhấp nháy** mỗi khi chuột đi từ control này sang control khác trong trang (WPF dựng lại chuỗi mouse-over: tắt chuỗi cũ rồi bật chuỗi mới). Fix: `<Trigger SourceName="tênBorderTrongTemplate" Property="IsMouseOver">` — bắt hover của chính phần header. Cùng bẫy này áp cho `IsKeyboardFocusWithin`.
- **Setter `Foreground`/`FontSize`/`FontWeight`/`Cursor` đặt thẳng lên `TabItem` kế thừa xuống cả trang nội dung**: chọn tab làm `FontWeight=SemiBold` là chữ trong trang đậm theo, `Cursor=Hand` là cả trang thành hình bàn tay. Fix: chỉ để `Setter Property="Template"` trên TabItem, mọi thứ khác đặt lên element trong template.
- Bẫy phụ khi làm header: `ContentPresenter` sinh `TextBlock` cho header chuỗi, mà **style ngầm của `TextBlock` áp được vào đó** và đè `Foreground` ⇒ trigger đổi màu chữ vô tác dụng. Dùng `TextBlock` tường minh + `Style="{x:Null}"` trong template (chấp nhận header chỉ là chuỗi).

## WPF: `async void` handler cho `Closing` — `await` xong đồng bộ là ném `InvalidOperationException`
- Mẫu hay dùng để hỏi "lưu trước khi đóng?": `Closing += async (_, e) => { e.Cancel = true; if (await ConfirmAsync()) { approved = true; window.Close(); } }`.
- Bẫy: nếu `ConfirmAsync()` chạy hết mà **không hề yield** (MessageBox là đồng bộ; nhánh "Không lưu" chỉ `return true`) thì continuation chạy **inline ngay trong handler** ⇒ `Close()` được gọi khi cửa sổ đang đóng ⇒ `System.InvalidOperationException: Cannot set Visibility to Visible or call Show, ShowDialog, Close, or WindowInteropHelper.EnsureHandle while a Window is closing.`
- Cực dễ tưởng đã đúng vì nhánh có I/O thật (lưu file) **await thật ⇒ chạy sau, không lỗi**; chỉ nhánh trả về ngay mới lộ. Đừng kết luận "có await là đã thoát khỏi handler".
- Fix: hoãn toàn bộ phần hỏi + `Close()` ra khỏi ngữ cảnh `Closing` bằng `_ = window.Dispatcher.InvokeAsync(async () => { ... })` (handler `Closing` để đồng bộ, chỉ `e.Cancel = true` rồi trả về).
- Cùng luật này áp cho `Show`/`ShowDialog`/`Visibility` gọi từ trong `Closing`/`Closed` của chính cửa sổ đó.

## WPF: subscribe event của ViewModel phải theo cặp `Loaded`/`Unloaded`, KHÔNG dựa vào `DataContextChanged`
- Mẫu sai hay gặp: code-behind gắn handler trong `DataContextChanged` nhưng gỡ trong `Unloaded`. `TabControl` (và mọi container đổi content: Frame, ContentPresenter, Expander...) **gỡ view khỏi visual tree khi rời tab ⇒ bắn `Unloaded`**, còn khi quay lại chỉ bắn `Loaded` — `DataContextChanged` KHÔNG bắn vì `TabItem` vẫn là logical child nên DataContext không hề đổi. Kết quả: handler mất vĩnh viễn, view "chết" lặng lẽ sau lần chuyển tab đầu tiên.
- Cực khó nhận ra khi trên cùng view có cả phần dùng `{Binding}`: binding tự re-attach nên vẫn cập nhật bình thường, chỉ phần vẽ tay bằng code-behind đứng im ⇒ dễ đổ oan cho nguồn dữ liệu.
- Đúng: `Loaded += Attach; Unloaded += Detach;` với cờ `_attached` chống subscribe trùng (Loaded có thể bắn nhiều lần), `DataContextChanged` chỉ đổi tham chiếu VM rồi `if (IsLoaded) Attach();`. Vẽ lại ngay trong `Loaded` để bù dữ liệu bỏ lỡ lúc bị unload.
- Lợi ích phụ: gỡ handler khi rời tab cũng chính là cách tránh leak và tránh vẽ lại vô ích khi view không hiển thị.

## Event C#: đối tượng dựng LƯỜI ngay trong handler sẽ BỎ LỠ chính lần phát đó
- `Handler?.Invoke(x)` chụp danh sách người nghe **tại thời điểm gọi**; ai `+=` trong lúc invocation đang chạy thì lần phát đó không tới tay họ (multicast delegate là immutable, `+=` tạo delegate MỚI).
- Bẫy thật hay gặp với DI singleton + navigation: `Session.SetProject(p)` phát `ProjectChanged` → handler của navigator resolve cửa sổ mới → DI dựng lần đầu các ViewModel con, chúng `+=` vào chính event đó rồi ngồi đợi. Kết quả: **lần mở đầu tiên sau khi bật app** màn hình rỗng, mở lần thứ hai lại đúng (VM đã tồn tại, đã nghe từ trước) ⇒ rất dễ đổ oan cho khâu lưu/đọc file.
- Chỉ lộ ở ViewModel **materialize collection** trong hàm reload. VM chỉ đọc `Session.Current` qua property getter thì vẫn đúng, vì binding tự đánh giá sau — nên trong cùng một cửa sổ có màn hỏng màn không, càng khó lần.
- Quy tắc: đăng ký event xong phải **đồng bộ ngay với trạng thái hiện tại** trong constructor (`_x.Changed += Reload; Reload();`), đừng coi event là nguồn khởi tạo. Hàm reload phải idempotent (clear rồi dựng lại) để gọi thừa cũng vô hại.

## Build fail "file is locked by <app>" — đổi RIÊNG BaseOutputPath thì được, đổi kèm BaseIntermediateOutputPath thì hỏng
- Triệu chứng: `error MSB3027/MSB3021: Could not copy ... Exceeded retry count of 10. The file is locked by: "App (PID)"` — app vừa build lần trước **vẫn đang chạy** (user tự mở, hoặc lần chạy thử trước chưa thoát).
- Cách chữa SAI mà tưởng khôn: build sang thư mục khác bằng `-p:BaseOutputPath=... -p:BaseIntermediateOutputPath=...`. Truyền qua dòng lệnh thì **mọi project trong solution dùng CHUNG một thư mục obj** ⇒ `project.assets.json` của project cuối cùng restore ghi đè các project khác (`error NETSDK1005: Assets file doesn't have a target for <TFM>`) và AssemblyInfo sinh trùng (`error CS0579: Duplicate ...Attribute`). Đống lỗi này KHÔNG liên quan gì tới code đang sửa — dễ mất thời gian đuổi lỗi ma.
- `-t:Compile` **chỉ cứu được project ĐANG sửa, không cứu được project tham chiếu tới nó**: chạy `dotnet build <ProjectDangSua>.csproj -t:Compile` thì CSC (kèm source generator, tức Razor `.cshtml` cũng được kiểm) chạy đủ và báo `Build succeeded`, vì bước copy sang `bin` nằm ở `CopyFilesToOutputDirectory` phía sau. Nhưng `dotnet build <ProjectTest>.csproj -t:Compile` vẫn vỡ y như cũ: project tham chiếu bị build **đầy đủ** (không kế thừa `-t:Compile`) nên vẫn đụng `bin` bị khoá ⇒ **không chạy được test** khi app đang bị debug. (Ghi chú cũ nói `-t:Compile` gây `MSB4057` là sai với project SDK-style `Microsoft.NET.Sdk.Web` — đã kiểm lại trên .NET 10, target `Compile` có tồn tại.) Còn `-p:SkipCopyBuildProduct=true` thì làm project phụ thuộc không tìm thấy metadata (`error CS0006`).
- Đúng: kiểm tra process rồi xử lý — `Get-Process <TênExe> -ErrorAction SilentlyContinue`. Nếu đang chạy thì **báo user đóng app** (đừng tự kill tiến trình của user), chờ rồi build lại. Chỉ dừng process khi chính mình vừa khởi động nó để chạy thử.
- Khi KHÔNG được phép đụng vào app đang chạy của user (đang debug trong Visual Studio) mà vẫn cần chạy thử bản mới: `dotnet publish <proj> -o <thư mục tạm>/pub -p:BaseOutputPath=<thư mục tạm>/mo/` — **chỉ `BaseOutputPath`, KHÔNG kèm `BaseIntermediateOutputPath`** (xem gạch đầu dòng trên để biết vì sao kèm vào là vỡ). Property truyền qua dòng lệnh lan sang mọi project tham chiếu nên toàn bộ đồ hình build tránh được `bin` đang bị khoá; `obj` giữ nguyên nên vẫn incremental. Đã dùng thật để chạy song song hai host ASP.NET trên cổng riêng trong lúc user vẫn đang debug bộ gốc.

## WPF: `Clear()` + thêm lại `ObservableCollection` NGAY TRONG setter của `SelectedItem` ⇒ danh sách NHÂN ĐÔI
- Mẫu sai rất hay gặp ở MVVM: `ListBox` bind `ItemsSource={Binding Items}` + `SelectedItem={Binding Selected, Mode=TwoWay}`, còn setter `Selected` gọi một hàm "đồng bộ lại cho chắc" kiểu `Items.Clear(); foreach (...) Items.Add(x);`.
- Setter đó chạy **ngay bên trong lúc `Selector` đang xử lý đổi selection**. `Clear()` phát `NotifyCollectionChangedAction.Reset` giữa chừng ⇒ `CollectionView`/`ItemContainerGenerator` lệch khỏi collection: UI hiện **gấp đôi số mục** (2 ảnh thành 4), lâu lâu **sai thứ tự**, dù collection thật vẫn đúng số lượng. Debug bằng `Items.Count` sẽ thấy "đúng" ⇒ rất dễ đổ oan cho tầng dữ liệu/model.
- Triệu chứng nhận dạng: bấm chọn mục **đang chọn sẵn** thì không sao (setter thoát sớm), bấm chọn mục **khác** mới nhân đôi.
- Fix: đồng bộ **tại chỗ theo diff** — xoá phần dôi ở cuối, rồi so từng vị trí, khác mới `Items[i] = x`, thiếu mới `Add`. Trường hợp danh sách không đổi (chính là lúc đổi selection) sẽ phát **0 sự kiện** nên không còn reentrancy.
- Luật chung: đừng bao giờ phát `Reset` (`Clear()`, gán lại `ItemsSource`) từ trong đường chạy của một sự kiện mà `ItemsControl` đang xử lý (SelectionChanged, CurrentChanged, CollectionChanged).

## ASP.NET + jQuery Validation: `step="0.5"` trên ô `decimal` chặn luôn giá trị HỢP LỆ
- Triệu chứng: ô `<input asp-for="X" type="number" step="0.5">` bind với `decimal` map cột `decimal(p,2)`; mở trang lên bấm Lưu **không sửa gì** cũng ra `Please enter a multiple of 0.5.` Trình duyệt hiển thị `1,00` (locale máy) làm rất dễ đổ oan cho dấu thập phân theo culture, đi sửa `RequestLocalization` mất công vô ích.
- Thông báo đó là của **jQuery Validation**, KHÔNG phải HTML5 (`jquery.validate.js`, `messages.step`). Luật `step` của nó kiểm tra **hai** điều kiện: `decimalPlaces(value) > decimalPlaces(step) || toInt(value) % toInt(step) !== 0`. Vế đầu mới là thủ phạm.
- `decimal` giữ nguyên scale khi ToString: giá trị đọc từ cột `decimal(9,2)` render ra chuỗi `"1.00"` → 2 chữ số thập phân > 1 chữ số của `step="0.5"` → **fail**, dù 1.00 chia hết cho 0.5. Nghĩa là ô hỏng với *mọi* giá trị lấy từ DB (`0.50`, `1.00`, `2.00`), không phải lỗi ngẫu nhiên.
- Cách nhận biết nhanh: lỗi xuất hiện ngay khi vừa nạp trang mà chưa gõ gì, và biến mất nếu xoá `_ValidationScriptsPartial`.
- Fix: `step="any"` — `normalizeAttributeRule` làm `Number("any")` = NaN ⇒ bỏ luôn rule step, HTML5 cũng không kiểm `stepMismatch` nữa. Rồi kiểm số chữ số thập phân ở **server** bằng `ValidationAttribute` riêng, thông báo lỗi rõ nghĩa hơn "multiple of".
- Kiểm scale đúng cách là so với chính giá trị đã làm tròn, đừng đọc `decimal.Scale`: `0.5000m` có `Scale = 4` nhưng bằng `0.50m`, còn `==` của decimal bỏ qua số 0 ở đuôi.
  ```csharp
  // hợp lệ khi số chữ số thập phân "có nghĩa" ≤ scale
  decimal.Round(d, scale, MidpointRounding.AwayFromZero) == d
  ```
- Bối cảnh vì sao phải chặn ở server: SQL Server ghi vào `decimal(p,s)` **làm tròn lặng lẽ**, nhập 0.333 vào cột scale 2 thì lưu ra 0.33 mà không lỗi gì — mở lại trang mới thấy số đã khác.

## ASP.NET Core: `AddDataAnnotationsLocalization()` KHÔNG dịch `ErrorMessage` của attribute tự viết
- Triệu chứng: form hiện thẳng **khoá resource** (`MyVm.SomeError`) thay vì câu tiếng Việt, trong khi `[Required]`/`[Range]`/`[RegularExpression]` trên **cùng property, cùng ViewModel** vẫn dịch đúng, và tra tay `IStringLocalizer["MyVm.SomeError"]` cũng ra đúng. Rất dễ đổ oan cho file resource/cache/đường dẫn và đi sửa nhầm chỗ.
- Nguyên nhân: `DataAnnotationsModelValidator.GetErrorMessage()` là một `switch` trên **kiểu attribute cụ thể** (Required, Range, StringLength, MinLength, MaxLength, Compare, RegularExpression, DataType…). Attribute không nằm trong danh sách rơi vào nhánh mặc định trả `null`, rồi `?? result.ErrorMessage` lấy chuỗi thô từ `FormatErrorMessage` — tức chính khoá resource. `IStringLocalizer` có được inject đầy đủ, chỉ là không bao giờ được gọi.
- Fix: attribute tự tra localizer trong `IsValid(object?, ValidationContext)` — MVC truyền `HttpContext.RequestServices` vào `ValidationContext`, nên `GetService` lấy được factory:
  ```csharp
  protected override ValidationResult? IsValid(object? value, ValidationContext ctx)
  {
      if (IsValid(value)) return ValidationResult.Success;
      string msg = FormatErrorMessage(ctx.DisplayName);
      if (!string.IsNullOrEmpty(ErrorMessage)
          && ctx.GetService(typeof(IStringLocalizerFactory)) is IStringLocalizerFactory f)
      {
          var s = f.Create(ctx.ObjectType)[ErrorMessage!, ctx.DisplayName, /* tham số riêng */ ];
          if (!s.ResourceNotFound) msg = s.Value;
      }
      return new ValidationResult(msg, ctx.MemberName is null ? null : new[] { ctx.MemberName });
  }
  ```
- Lợi thêm: tự tra thì **truyền được tham số riêng** (`{1}`, `{2}`…), thứ mà đường mặc định không cho — nó chỉ ghép `{0}` = display name, còn `{1}`/`{2}` chỉ có với vài attribute BCL được hard-code. Nhớ override cả `FormatErrorMessage(name)` thành `string.Format(..., name, thamSốRiêng)`, không thì nhánh dự phòng ném `FormatException` vì chuỗi có `{1}`.
- Cách xác minh nhanh không cần chạy cả app: dựng `WebApplication.CreateBuilder` với `ContentRootPath` trỏ vào project thật, đăng ký y hệt, rồi gọi `IObjectModelValidator.Validate(actionContext, new ValidationStateDictionary(), "Input", model)` và in `ModelState`. So kết quả của attribute tự viết với một attribute BCL trên cùng property là ra ngay thủ phạm.

## Đặt namespace TRÙNG tên một type ở project khác ⇒ CS0118 "is a namespace but is used like a type"
- Tình huống thật: tách một tool console tên `VeoFlow.AiStudioRecorder` (namespace mặc định = tên project) trong khi project thư viện đã có lớp `VeoFlow.Selenium.AiStudioRecorder`. Mọi file trong cây namespace mới không còn dùng được lớp đó: `new AiStudioRecorder(...)` báo `CS0118`.
- Nguyên nhân: C# tra tên từ namespace hiện tại đi ra ngoài. Đứng trong `VeoFlow.AiStudioRecorder`, cái tên `AiStudioRecorder` khớp **namespace** `VeoFlow.AiStudioRecorder` trước khi kịp xét `using` nào. Cùng gốc với bẫy "segment namespace trùng tên type BCL" (`...Dns` che `System.Net.Dns`), nhưng lần này type bị che nằm ở project của chính mình.
- Chữa: đổi **namespace**, giữ nguyên tên project/file chạy nếu tên đó là thứ người dùng gõ — `<RootNamespace>` trong .csproj tách hai thứ đó ra. Không cần đổi `AssemblyName`.
- Cách né từ đầu: tên project cho tool/exe nên khác tên lớp bên trong thư viện (thêm segment `Tools.`, `Cli.`…), hoặc đặt namespace tool ở một nhánh riêng ngay từ lúc tạo project.

## WPF: nút `IsCancel="True"` + `Closing` bị cancel ⇒ cửa sổ KẸT không đóng được nữa
- Tình huống thật: thêm hỏi "Lưu / Không lưu / Huỷ" vào `Closing` của một cửa sổ modal có nút "Đóng" đặt `IsCancel="True"`. Người dùng chọn "Huỷ" một lần là từ đó bấm "Đóng" hay Esc đều VÔ TÁC DỤNG, chỉ còn nút X của Windows đóng được.
- Nguyên nhân: nút `IsCancel` không gọi `Close()` mà gán `Window.DialogResult = false`, và setter đó có chốt `if (_dialogResult != value)` mới gọi `Close()`. Lần bấm đầu đã gán `false` rồi; `Closing` bị cancel nên cửa sổ còn sống nhưng `_dialogResult` vẫn là `false` ⇒ mọi lần gán sau đều rơi vào nhánh "không đổi", không ai gọi `Close()` nữa.
- Bẫy đi kèm: để CẢ `IsCancel="True"` lẫn `Click="OnCloseClick"` (handler gọi `Close()`) thì một cú bấm chạy hai đường đóng ⇒ hộp thoại hỏi lưu bật lên HAI LẦN.
- Chữa: bỏ `IsCancel`, giữ đúng MỘT đường đóng là `Click` → `Close()`, rồi tự bắt Esc bằng `protected override void OnPreviewKeyDown` (`e.Key == Key.Escape` ⇒ `e.Handled = true; Close();`). Mọi đường (nút, Esc, X) khi đó cùng đi qua `Closing` đúng một lần.
- Quy tắc chung: cửa sổ nào có `Closing` cancel được thì đừng đóng bằng cách gán `DialogResult` — luôn đóng bằng `Close()`.

## `Process` có RedirectStandardOutput/Error mà không đọc pipe ⇒ tiến trình con TREO vô hạn
- Tình huống thật: test dựng clip mẫu bằng `ffmpeg -f lavfi -i testsrc ...`, `ProcessStartInfo` bật `RedirectStandardOutput` + `RedirectStandardError` cho khỏi bẩn console, rồi `await process.WaitForExitAsync()`. Lệnh `dotnet test` chạy quá 10 phút không xong; `tasklist` thấy `ffmpeg.exe` còn sống, ngốn 250 MB.
- Nguyên nhân: pipe của Windows chỉ đệm được vài KB. ffmpeg ghi log tiến độ ra **stderr**, đầy buffer là nó đứng chờ có người đọc, còn phía mình thì đang chờ nó thoát — kẹt cả hai đầu. Redirect mà không đọc là tự dựng deadlock, và triệu chứng nhìn y hệt "công cụ ngoài chạy chậm".
- Chữa: đọc pipe SONG SONG với việc chờ, đừng đọc sau `WaitForExit`:
  ```csharp
  Task<string> stdout = process.StandardOutput.ReadToEndAsync();
  Task<string> stderr = process.StandardError.ReadToEndAsync();
  await process.WaitForExitAsync();
  await Task.WhenAll(stdout, stderr);
  ```
  Bản đồng bộ thì `ReadToEnd()` cả hai luồng TRƯỚC `WaitForExit()`. Hoặc dùng `OutputDataReceived`/`ErrorDataReceived` + `BeginOutputReadLine()`.
- Kèm theo với ffmpeg: luôn thêm `-nostdin`. Chạy trong test runner/dịch vụ thì không có bàn phím, mà ffmpeg mặc định vẫn nghe stdin để bắt phím `q`.
- Dấu hiệu nhận ra nhanh: lệnh treo + `tasklist` thấy tiến trình con còn sống và bộ nhớ đứng yên ⇒ gần như chắc chắn là deadlock pipe, không phải công việc nặng.

## WPF: `ScrollIntoView` ngay trong handler `CollectionChanged` ⇒ "An ItemsControl is inconsistent with its items source"
- Tình huống thật: ListBox hiện nhật ký, code-behind nghe `CollectionChanged` rồi gọi `list.ScrollIntoView(list.Items[^1])` để tự cuộn xuống dòng mới. Chạy êm cả buổi, tới lúc log dồn về hàng loạt (bấm Huỷ ⇒ mỗi việc đang chờ ghi một dòng) thì `InvalidOperationException` giết app.
- Nguyên nhân: khi handler của mình chạy, `ItemsControl` **chưa chắc đã xử lý xong** chính thay đổi vừa được báo — nó cũng là một người nghe khác của cùng sự kiện. `ScrollIntoView` buộc nó dựng/đọc lại container giữa chừng, thấy `Items` lệch với nguồn ⇒ ném. Collection tự marshal về UI thread (kiểu `DispatcherObservableCollection`) càng dễ dính vì sự kiện tới sau khi nguồn đã đổi thêm vài lần.
- Chữa: hoãn sang lượt dispatcher sau, và bọc `try/catch` vì tới lúc đó danh sách có thể lại đổi:
  ```csharp
  private void OnLogsChanged(object? s, NotifyCollectionChangedEventArgs e)
  {
      if (e.Action != NotifyCollectionChangedAction.Add) return;
      Dispatcher.BeginInvoke(ScrollToLast, DispatcherPriority.Background);
  }

  private void ScrollToLast()
  {
      try { if (List.Items.Count > 0) List.ScrollIntoView(List.Items[^1]); }
      catch (InvalidOperationException) { }
  }
  ```
- Quy tắc chung: trong handler `CollectionChanged`/`PropertyChanged` đừng gọi API buộc control đọc lại dữ liệu (`ScrollIntoView`, `UpdateLayout`, `Items[i]`, `ContainerFromIndex`). Đẩy sang `DispatcherPriority.Background` là đủ.

## Comment trong `.csproj` KHÔNG được chứa `--` ⇒ `MSB4025`, build chết ngay ở bước nạp project
- `.csproj` là XML, mà chuẩn XML cấm chuỗi `--` bên trong comment. Viết chú thích kiểu `<!-- cờ --proxy-server không nhận user/pass -->` là hỏng: `error MSB4025: The project file could not be loaded. An XML comment cannot contain '--'`.
- Rất dễ dính vì chú thích về **cờ dòng lệnh** (`--user-data-dir`, `--headless`, `--no-sandbox`) là thứ hay viết nhất trong csproj của project chạy CLI/trình duyệt.
- Cách viết: bỏ hai gạch (`cờ proxy-server`), hoặc dùng một gạch, hoặc thoát bằng `&#45;&#45;`. Cùng luật đó áp cho mọi file XML khác: `.props`, `.targets`, `.config`, `.xaml`.

## Thêm một PackageReference có thể làm vỡ restore của project KHÁC vì `NU1605` (downgrade)
- Triệu chứng: vừa thêm gói vào project A, `dotnet test`/`build` báo lỗi ở project B — `error NU1605: Detected package downgrade: X from 10.0.8 to 8.0.1`.
- Nguyên nhân: gói mới kéo theo một phụ thuộc bắc cầu bản cao (vd một gói nội bộ → `Microsoft.Extensions.Logging` 10.0.8 → `Microsoft.Extensions.DependencyInjection >= 10.0.8`), trong khi project B **khai báo trực tiếp** bản thấp hơn (8.0.1). NuGet coi khai báo trực tiếp là "hạ cấp" và báo LỖI, không phải cảnh báo.
- Cách xử lý đúng: nâng khai báo trực tiếp ở project B lên đúng nhánh (`10.0.8`). Kiểm tra trước bằng `ls ~/.nuget/packages/<gói>/<phiên bản>/lib` xem có thư mục khớp TFM đang dùng không — các gói `Microsoft.Extensions.*` 10.x vẫn có `net8.0`, nên KHÔNG phải nâng `TargetFramework` theo.
- Đừng chữa bằng cách hạ gói mới hay thêm `NoWarn=NU1605`: nó chỉ giấu việc runtime sẽ nạp bản nào.

## Cấu hình người dùng sửa được: truyền `Func<T>`, đừng truyền `T` đã lấy
- Triệu chứng: user sửa một giá trị trong màn Cấu hình, app vẫn chạy theo số cũ tới hết lượt — nhìn hệt như "app không lưu cấu hình".
- Nguyên nhân: đối tượng sống lâu (pool, scheduler, worker) nhận sẵn một options object dựng từ config lúc khởi tạo. Đó là **bản chụp**, và nó sống đúng bằng tuổi thọ đối tượng đó.
- Cách sửa: nhận `Func<TOptions>` rồi gọi lại ở mỗi lần cần. Giữ thêm quá tải nhận `TOptions` cho test (bọc bằng một hàm trả hằng, gọi `Validate()` ngay tại đó để vẫn ném sớm khi cấu hình sai).
- **Gọi factory TRƯỚC khi vào `lock`**: nó là code của tầng trên (đọc config, có khi đụng UI), giữ khoá trong lúc chạy code người khác là công thức deadlock.
- Đọc sống thì KHÔNG được ném lỗi giữa chừng vì số vô lý (min > max, interval <= 0): lúc dựng thì ném là đúng, nhưng lúc đang chạy thì phải **nắn** giá trị, vì cấu hình sai không đáng làm đổ cả lượt việc đang dở.

## Mốc thời gian tuyệt đối đã persist là một bản chụp cấu hình ẩn
- `DeadlineAt = now + KhoangLayTuCauHinh` rồi ghi xuống file/DB: giá trị cấu hình cũ đã **đông cứng** trong mốc đó và sống lâu hơn cả tiến trình sinh ra nó. Sửa cấu hình xong khởi động lại vẫn thấy hành vi cũ ⇒ rất dễ bị chẩn đoán nhầm thành "code không đọc config".
- Cách gỡ: lưu kèm mốc **bắt đầu** (`StartedAt`), rồi khi đọc thì kẹp lại `min(DeadlineAt, StartedAt + TranHienHanh)`. Chỉ kẹp NGẮN lại, đừng nới dài ra — nới dài là một kiểu bất ngờ khác.
- Bản ghi cũ không có `StartedAt` thì **giữ nguyên mốc đã ghi**, đừng kẹp bừa theo một mốc đoán: kẹp sai hướng là thả tài nguyên ra sớm rồi đâm lại đúng cái lỗi vừa gặp.
- Nhớ tìm cả **chỗ hiển thị**: UI đọc thẳng mốc thô sẽ hiện số khác với số hệ thống đang dùng. Tách phần tính vào một helper thuần tuý cho cả hai bên gọi chung.

## `rm -rf bin/obj` để gỡ khoá file: PHẢI liệt kê nội dung trước — thư mục output có thể chứa dữ liệu THẬT
- Tình huống: `git mv`/đổi tên thư mục project báo `Permission denied` vì IDE (Visual Studio) giữ handle trong `obj/`. Phản xạ "xoá bin/obj cho sạch rồi làm lại" gỡ được khoá thật — nhưng `bin\Debug\<TFM>\` là **thư mục làm việc của app**: mọi app dùng `AppContext.BaseDirectory` để lưu config/profile/dữ liệu người dùng đều đổ dữ liệu vào đúng đó.
- Hậu quả có thật: xoá mất profile Chrome đã đăng nhập, `config.json` chứa API key, thư mục project của người dùng. `rm` của Git Bash **KHÔNG đi qua Recycle Bin** ⇒ không undo được; máy không bật System Protection thì cũng không có shadow copy để khôi phục.
- Bắt buộc trước khi xoá: `ls bin/Debug/<TFM>` và soi có gì ngoài `*.dll/*.exe/*.pdb/*.json` của build không (thư mục lạ như `Profiles/`, `Logs/`, `projects/`, file `config.json` ở cấp gốc). Có thì **dời ra ngoài trước**, đừng xoá.
- Gỡ khoá mà không xoá: `dotnet build-server shutdown` (dừng node MSBuild/VBCSCompiler) thường đã đủ; hoặc chỉ xoá `obj/` — `git mv` chỉ vướng handle ở đó. **Chỉ xoá `bin/` khi đã kiểm tra nội dung.**

## .NET Android: `adb install` bản Debug tự tay bị crash "No assemblies found" (Fast Deployment)
- Triệu chứng: build `dotnet build -c Debug` cho project `net*-android`, lấy `*-Signed.apk` rồi `adb install -r` xong mở app là **crash ngay** (SIGABRT), logcat báo:
  `F monodroid: No assemblies found in '/data/user/0/<pkg>/files/.__override__/<abi>' or '<unavailable>'. Assuming this is part of Fast Deployment. Exiting...`
- Nguyên nhân: bản **Debug mặc định BẬT Fast Deployment** — assembly .NET (managed dll) KHÔNG nhúng vào APK; chúng được `dotnet build -t:Run`/`-t:Install` đẩy riêng sang `.__override__`. Cài APK bằng tay thì thiếu → runtime abort. (Native `.so` trong `lib/<abi>/` VẪN nạp bình thường — lỗi thuần tầng managed, đừng nhầm là lỗi thư viện native/P/Invoke.)
- Cách sửa (chọn 1): (a) rebuild `dotnet build -c Debug -p:EmbedAssembliesIntoApk=true` → APK tự chứa, `adb install -r` chạy được; (b) build `-c Release` (Release luôn nhúng assembly); (c) deploy đúng cách bằng `dotnet build -t:Run -c Debug` (tự cài + đẩy assembly + mở app) khi đã cắm máy.
- Phụ: tên activity .NET Android bị mã hoá `crc64<hash>.MainActivity` → ĐỪNG `am start` với wildcard (`Error type 3`); mở bằng `adb shell monkey -p <pkg> -c android.intent.category.LAUNCHER 1`, hoặc lấy tên thật `adb shell cmd package resolve-activity --brief <pkg>`.
- Phụ: nhiều máy (vd Pixel) khi `adb install` in `Incremental installation not allowed` rồi tự fallback `Performing Streamed Install → Success` — KHÔNG phải lỗi.

## WPF: `PreviewMouseRightButtonDown`/`MouseLeftButtonDown`… là routed event **Direct**, không Tunnel/Bubble
- Bối cảnh: viết attached behavior gắn `dataGrid.PreviewMouseRightButtonDown += ...` (để click phải chọn dòng dưới con trỏ), rồi test bằng cách `cell.RaiseEvent(new MouseButtonEventArgs(...){ RoutedEvent = UIElement.PreviewMouseRightButtonDownEvent })` — handler trên DataGrid **không hề được gọi**, dễ kết luận sai là "behavior không chạy" và đi sửa code đang đúng.
- Nguyên nhân: cặp `Mouse{Left,Right}ButtonDown/Up` và các `Preview*` của chúng được `EventManager.RegisterRoutedEvent` với `RoutingStrategy.Direct`. Khi có chuột thật, WPF nhận `PreviewMouseDown` (Tunnel) / `MouseDown` (Bubble) rồi **promote**: raise event Direct đó **trên từng element dọc route**, với `Source = e.OriginalSource` (element sâu nhất). Vì vậy handler trên ancestor CHẠY BÌNH THƯỜNG trong app thật, chỉ có RaiseEvent thủ công từ element con là không lan lên.
- Cách test đúng: raise **trên chính element có handler**, đặt `Source` là element sâu nhất để mô phỏng `OriginalSource`:
  `grid.RaiseEvent(new MouseButtonEventArgs(Mouse.PrimaryDevice, Environment.TickCount, MouseButton.Right){ RoutedEvent = UIElement.PreviewMouseRightButtonDownEvent, Source = textBlockTrongCell });`
- Suy ra: behavior loại này phải tìm ngược ancestor từ `e.OriginalSource` (không dùng `e.Source` — đã bị đặt lại theo element đang xử lý).
- Mẹo kiểm chứng UI không cần click tay: chạy app WPF trong process test, `Show()` window ở toạ độ ngoài màn hình, `ContextMenu.PlacementTarget = grid; IsOpen = true`, đọc container bằng `ItemContainerGenerator.ContainerFromIndex(i)` để xem `Header/Command/CommandParameter`, và chụp ảnh style bằng `RenderTargetBitmap.Render(menu)` → PNG.

## ASP.NET: hai host trả cùng một DTO thì PHẢI cấu hình serializer JSON giống nhau
- Bối cảnh: host Api dùng `AddControllers().AddNewtonsoftJson(... DefaultContractResolver + StringEnumConverter)` (PascalCase, enum ra tên); host Web (Razor Pages) chỉ gọi `AddRazorPages()` nên dùng mặc định **System.Text.Json với `JsonSerializerDefaults.Web`** (camelCase, enum ra số). Các handler của Web trả thẳng `ApiResult<T>` nhận từ Api ra cho JavaScript ⇒ **cùng một đối tượng đi ra hai host với hai định dạng khác nhau**.
- Hai triệu chứng sinh ra từ đúng một gốc, và **không cái nào có lỗi biên dịch**:
  + JS đọc `result.IsSuccessed` nhưng server trả `isSuccessed` ⇒ luôn `undefined` ⇒ nhánh `if (!result.IsSuccessed)` luôn đúng ⇒ báo "thất bại" ngay bước đầu dù server trả 200 và làm đúng việc.
  + `<option value="@kind">` của Razor render ra **tên** enum (`"Installer"`, vì `InputTagHelper`/`ToString()`), JS gửi lên dạng chuỗi; System.Text.Json không có `JsonStringEnumConverter` thì **không** đổi được chuỗi → tham số `[FromBody]` về **null** (không phải 400) ⇒ `NullReferenceException`/`ArgumentNullException` ở tầng dưới, người dùng chỉ thấy trang lỗi chung.
- Vì sao lọt: smoke test gọi **thẳng Api** nên thấy đúng hết; unit test không chạm tầng HTTP; trình duyệt là chỗ duy nhất lộ ra. Bug kiểu này sống sót qua cả một giai đoạn phát triển.
- Cách phát hiện rẻ nhất: gọi handler bằng PowerShell/curl rồi **in RAW body**, đừng `ConvertFrom-Json` xong kiểm tra thuộc tính — `ConvertFrom-Json` của PowerShell truy cập thuộc tính **không phân biệt hoa thường**, nên `$r.IsSuccessed` vẫn ra `True` trong khi JS thì `undefined`. Suýt kết luận sai vì chỗ này.
- Phòng ngừa: cấu hình JSON đặt cùng một chỗ dùng chung cho mọi host; hoặc tối thiểu là copy y nguyên khối `AddNewtonsoftJson(...)` sang host thứ hai. Kiểm tra `.csproj` — gói `Microsoft.AspNetCore.Mvc.NewtonsoftJson` có thể đã được tham chiếu sẵn mà **chưa hề gọi** `AddNewtonsoftJson()`, nhìn qua rất dễ tưởng đã cấu hình rồi.
- Thêm chốt `if (model is null) return BadRequest(...)` ở mọi handler `[FromBody]`: thân request hỏng thì trả 400 đúng nghĩa thay vì để null trôi xuống thành 500.

## Razor Pages: handler `[FromBody]` + client gửi FormData ⇒ tham số NULL, KHÔNG phải 415
- Ở controller MVC, content-type không đọc được thì `UnsupportedContentTypeFilter` trả **415**. Filter đó là action filter — Razor Pages **không chạy** action filter, nên handler page vẫn được gọi, chỉ ghi lỗi vào `ModelState` và đưa tham số `[FromBody]` = **null** ⇒ `NullReferenceException` ⇒ trang lỗi HTML 500. Phía JS thấy "server trả text/html thay vì JSON (HTTP 500)".
- Bẫy tên hàm: helper kiểu `app.postJson(url, data)` của một repo có thể thật ra gửi **FormData** (để kèm token antiforgery như form thường). Đọc thân helper trước khi viết handler, đừng suy từ cái tên.
- Sửa: handler nhận tham số rời (`string? draft, string? name`) — binder đọc được từ form; hoặc client gửi JSON thật (`fetch` + `Content-Type: application/json` + header antiforgery). Đừng trộn hai phía.
- Vì sao lọt: unit test không chạm tầng HTTP, và build không có cảnh báo gì — chỉ bấm nút trên trình duyệt mới lộ.

## `Progress<T>` KHÔNG tuần tự hoá handler — nó chạy trên thread pool và có thể vào SONG SONG chính nó
- `Progress<T>` chụp `SynchronizationContext.Current` **tại lúc `new`**. Có context (UI thread) thì mọi handler về đúng luồng đó ⇒ tuần tự. **Không có** context — đúng trường hợp đối tượng được dựng sau một chuỗi `await ... ConfigureAwait(false)`, tức là hầu hết code tầng service — thì mỗi lần `Report` là một lần `ThreadPool.QueueUserWorkItem` **độc lập** ⇒ hai bản báo liên tiếp có thể chạy **cùng lúc trên hai luồng**, dù bên gửi gọi `Report` tuần tự.
- Hệ quả điển hình: mọi phép **kiểm-rồi-làm** viết rời trong handler đều hở.
  ```csharp
  // SAI: hai luồng cùng thấy Queued rồi cùng chuyển
  if (scene.Status == SceneStatus.Queued)
      SceneStateMachine.Transition(scene, SceneStatus.Generating);
  ```
  Hai kết cục, tuỳ chỗ chen vào: (a) máy trạng thái ném "không chuyển được `X` → `X`"; (b) **không ném gì cả** mà cả hai cùng qua được (cả hai đọc trạng thái cũ trước khi ai kịp ghi) ⇒ bộ đếm `Attempts` tăng hai lần, việc "chỉ làm một lần" chạy hai lần. Ca (b) im lặng nên nguy hơn.
- 🔴 **Ném trong handler của `Progress<T>` là SẬP APP**: nó chạy trên thread pool, không nằm trong `try` của ai, không ai `await` nó ⇒ unhandled exception, không phải `UnobservedTaskException` (thứ này chỉ nuốt lỗi của `Task` bị bỏ rơi). Debugger dừng ở `LastBreakReason = ExceptionNotHandled`.
- Cách sửa: cho phần đổi trạng thái vào **một đối tượng có khoá**, handler chỉ gọi một thao tác nguyên tử (`TryMarkStarted()` trả `bool`). Nếu cùng một đối tượng đích được bọc ở nhiều nơi thì khoá phải bám theo **đích**, không phải theo bản bọc — `ConditionalWeakTable<TKey, object>` là chỗ giữ khoá gọn nhất (an toàn đa luồng sẵn, key yếu nên không rò), và an toàn hơn `lock (target)` vì không ai ngoài lớp đó khoá lên chính đối tượng được.
- **Viết test cho ca đua thì đừng dùng `Parallel.For`/`Task.Run` + `Barrier`**: cả hai lấy luồng từ thread pool và KHÔNG hứa chạy đủ số việc cùng lúc ⇒ barrier khoá chết cả bộ test. Dùng `new Thread(...)` thật, cho `IsBackground = true`, `Join(timeout)` rồi `Assert` timeout. Lặp ~100 vòng vì ca đua không phải lúc nào cũng trúng.
- Luôn **kiểm chứng test có răng**: tạm bỏ khoá đi, chạy lại, phải THẤY nó fail — test đa luồng viết sai rất hay pass vì lý do không liên quan.

## `MSB4018` ở `DefineStaticWebAssets`: xoá file cache trong `obj`, đừng đi tìm lỗi trong code

Triệu chứng: build một project ASP.NET Core chết với một đống stack trace `MSB4018` từ
`Microsoft.NET.Sdk.StaticWebAssets.targets`, đáy stack là `File.OpenRead` →
`DefineStaticWebAssetsCache.ReadOrCreateCache`. Không dòng nào chỉ vào code của mình, và build lại
bao nhiêu lần cũng y hệt.

Nguyên nhân: `obj/<Config>/<tfm>/staticwebassets.build.json.cache` bị **cụt** — thường do một lần
build trước bị giết giữa chừng (kill tiến trình, tắt máy, hết đĩa). SDK đọc file cache đó rồi ném,
chứ không tự dựng lại.

Cách sửa: xoá đúng file cache rồi build lại, không cần `dotnet clean` cả solution.

```powershell
Get-ChildItem "<proj>\obj\Debug\net10.0" -Filter "staticwebassets*.cache" |
    ForEach-Object { Remove-Item -LiteralPath $_.FullName -Force }
```

Lần vấp thật: file còn đúng **44 byte** sau khi `Stop-Process` mấy tiến trình `dotnet` đang chạy để
giải phóng dll cho lần build sau (2026-08-08).

## `dotnet build` với file `.slnx` — `.sln` có thể không còn tồn tại

Solution format mới (`.slnx`, XML) đã thay `.sln` ở một số repo. `dotnet build Foo.sln` khi chỉ có
`Foo.slnx` cho ra `MSB1009: Project file does not exist` — nghe hệt như đứng nhầm thư mục. Kiểm bằng
`Get-ChildItem -Filter "*.sln*"` trước khi kết luận.

## Ba tấm lưới bắt lỗi của app .NET/WPF — thiếu tấm nào là loại lỗi đó giết app hoặc biến mất im lặng
- `Application.DispatcherUnhandledException` — **chỉ** luồng UI. Chặn được (`e.Handled = true`), app chạy tiếp.
- `AppDomain.CurrentDomain.UnhandledException` — luồng thread pool / luồng thường. **KHÔNG chặn được**: .NET đã quyết định kết liễu tiến trình trước khi gọi tới đây, đặt cờ gì cũng vô ích. Việc duy nhất còn làm được là ghi lại. Thiếu tấm này thì app chết trắng, không để lại một dòng nào — rất hay bị nhầm thành "máy tự tắt app".
- `TaskScheduler.UnobservedTaskException` — `Task` lỗi mà không ai `await`, nổ **muộn** lúc GC thu hồi. Nhớ `e.SetObserved()`.
- 🔴 **Bẫy chết người khi log ở tấm thứ hai**: nếu logger ghi file kiểu bắn-đi-rồi-quên (`_ = WriteAsync(...)`, hàng đợi nền, `BackgroundService`) thì dòng log **chưa kịp chạm đĩa đã chết theo tiến trình** — đúng ca cần log nhất lại là ca mất log. Ở nhánh `IsTerminating` phải ghi **đồng bộ** (`File.AppendAllText`) trước, rồi mới gọi đường log bình thường.
- `OnStartup`/`Main` cũng phải có `try/catch` riêng: hỏng lúc dựng DI thì `DispatcherUnhandledException` có bắt được nhưng `e.Handled = true` để lại một tiến trình sống mà **không có cửa sổ nào** — phải `Shutdown(1)` cho dứt khoát.
- Chỗ cần `try/catch` của riêng mình (lưới app chỉ là phương án cuối, không thay được): handler của `Progress<T>`, `async void` event handler, `_ = SomethingAsync()`, và `Dispatcher.InvokeAsync(async () => ...)` bắn đi rồi quên — riêng cái cuối lỗi **không** chảy về `DispatcherUnhandledException` mà chui vào `DispatcherOperation` rồi biến mất, người dùng chỉ thấy "bấm không thấy gì xảy ra".
- Gói lại thành một helper (`Guarded.Run/Handler/FireAndForget(logger, context, body)`) thay vì chép `try/catch` khắp nơi: vừa ép mọi chỗ ghi log kèm ngữ cảnh, vừa **grep ra được** chỗ nào chưa bọc. Trong helper, `OperationCanceledException` nên ghi mức Debug — người dùng bấm Huỷ mà nhật ký đỏ lòm thì lần sau không ai đọc nhật ký nữa. Và bản thân lệnh ghi log cũng phải bọc `try/catch`: ném ở đó thì đúng thứ đang muốn chặn lại xảy ra.
- **KHÔNG** dùng helper đó bên trong chính `ILoggerProvider` — nó ghi lỗi bằng `ILogger`, mà `ILogger` lại chính là chỗ đang hỏng ⇒ đệ quy vô tận. Ở đó dùng `try/catch` trần.

## Kèm binary theo app là chưa đủ: thư viện bên thứ ba đọc **cấu hình toàn cục**, phải set nó ở entry point

- Ca thật (FFMpegCore, 2026-08-09): app đã đóng gói sẵn `ffmpeg.exe`/`ffprobe.exe` và code của mình luôn truyền `FFOptions { BinaryFolder = ... }`, nhưng một thư viện bên thứ ba lại gọi `FFProbe.AnalyseAsync(path, cancellationToken: ct)` **không kèm options** ⇒ nó rơi về `GlobalFFOptions.Current`, mặc định `BinaryFolder` rỗng = **chỉ tìm trong PATH**. Máy chưa cài ffmpeg thì tính năng hỏng sạch, kèm thông báo lạc hướng ("không phân tích được clip") dù binary vẫn nằm cạnh exe.
- Quy tắc: mỗi thư viện có **hai đường cấu hình** (per-call options và global/static) thì per-call chỉ che được code CỦA MÌNH. Code bên thứ ba, kể cả submodule tự viết, luôn đi đường global ⇒ **entry point phải set global một lần** (`GlobalFFOptions.Configure(...)`, `HttpClient.DefaultProxy`, `AppContext.SetSwitch`, `CultureInfo.DefaultThreadCurrentCulture`…).
- Chỗ đặt: `Main`/`OnStartup` của app, và **cả test** — test là một entry point khác, đặt trong lớp helper mà mọi ca test đi qua.
- Hàm `EnsureConfigured()` nên **idempotent** (cờ + `lock`) và **không ghi đè bằng giá trị rỗng**: dò không ra binary thì để nguyên mặc định, đừng xoá mất cấu hình người dùng đặt qua file config.
- Cách kiểm chứng đáng tin (nếu không thì máy dev có sẵn binary trong PATH sẽ pass giả): chạy test với PATH bị cắt còn tối thiểu — `$env:PATH = 'C:\Windows\System32;C:\Windows;C:\Program Files\dotnet'; dotnet test --no-build`. Với MSTest, nhớ nhìn cả cột **Skipped**: `Assert.Inconclusive` báo Skipped chứ không Failed, nên "Passed!" mà Skipped > 0 nghĩa là ca cần binary chưa hề chạy.

## Kiểm giao diện WPF từ PowerShell: hai cái bẫy làm kết luận SAI "app không phản hồi"

Cả hai đều khiến app **chạy đúng** mà mình đọc ra thành hỏng, rất tốn thời gian đuổi lỗi ma.

- **Cửa sổ mở bằng `ShowDialog()` có `Owner` KHÔNG nằm trong danh sách con của desktop.** Liệt kê
  `AutomationElement.RootElement.FindAll(TreeScope.Children, <PID>)` chỉ thấy cửa sổ chính ⇒ dễ kết luận
  "bấm nút không mở được dialog", rồi đi nghi ngờ binding command, `[RelayCommand]`, DataContext… trong khi
  dialog đã mở từ lâu. Cùng bẫy đó với `OpenFileDialog`/`MessageBox` (đều là cửa sổ có owner).
  Đúng: tìm bằng `FindAll(TreeScope.Descendants, ControlType.Window)` **từ cửa sổ chính**, hoặc quét
  toàn bộ `Button`/`Edit` theo `TreeScope.Descendants` rồi nhìn tên nút lạ xuất hiện.
- **PowerShell không DPI-aware ⇒ `GetWindowRect` trả kích thước ẢO.** Màn 150% thì cửa sổ WPF
  `Width="1100"` thật ra 1650px vật lý, nhưng `GetWindowRect` gọi từ PS trả về 1100 ⇒ bitmap tạo ra chỉ
  bằng 2/3 cửa sổ, `PrintWindow` vẽ vào đó thành ảnh **bị cắt mất mép phải/dưới**. Nhìn ảnh cắt đó rất
  giống lỗi layout tràn ngang, dễ đi sửa XAML vô ích. Chữa: gọi `SetProcessDPIAware()` **trước** khi đo,
  hoặc lấy kích thước từ `AutomationElement.Current.BoundingRectangle` (luôn là pixel vật lý).
- Cách xác nhận nhanh app có nhận input UIA hay không, trước khi nghi ngờ code: `TogglePattern.Toggle()`
  lên một CheckBox rồi đọc lại `ToggleState`. Đổi được nghĩa là đường UIA thông, lỗi nằm ở chỗ mình đọc
  kết quả chứ không phải ở app.
- Đừng dùng click chuột thật (`SetCursorPos` + `mouse_event`) để thay thế: cửa sổ app thường **bị cửa sổ
  khác che**, cú bấm rơi vào ứng dụng khác, và cú bấm đầu tiên vào cửa sổ chưa active chỉ để activate.
  `InvokePattern.Invoke()` không quan tâm che khuất nên đáng tin hơn hẳn.
- `Cookie đăng nhập ASP.NET Core ôm JWT: SlidingExpiration làm cookie sống lâu hơn token`: Mẫu "đăng nhập cookie ở web, gọi API bằng JWT cất trong claim" rất dễ rơi vào cảnh **còn đăng nhập nhưng mọi lời gọi API trả 401**. Dù lúc `SignInAsync` đã đặt `ExpiresUtc = jwt.ValidTo` cho cookie chết cùng token, `CookieAuthenticationOptions.SlidingExpiration = true` vẫn phá vỡ điều đó: qua nửa đời cookie, mỗi request lại gia hạn cookie thêm một quãng bằng quãng ban đầu (`CookieAuthenticationHandler.RequestRefresh`), trong khi JWT bên trong KHÔNG được ký lại nên chết đúng giờ cũ.
    + Triệu chứng: middleware xác thực thấy cookie hợp lệ nên cho vào trang; trang gọi API, client ném exception (vd `ApiException` với `StatusCode = 401`) và người dùng nhận **trang lỗi** thay vì trang đăng nhập. Không lộ ra lúc dev vì phiên mới luôn còn hạn.
    + Sửa gốc: đặt `AllowRefresh = false` trong `AuthenticationProperties` lúc `SignInAsync` — cookie chết đúng lúc token chết, middleware tự đưa về `LoginPath`.
    + Lưới an toàn (token bị thu hồi, đổi khóa ký, khởi động lại máy chủ): bắt riêng exception 401 ở middleware xử lý lỗi → `SignOutAsync` rồi chuyển hướng `/Account/Login?ReturnUrl=...`. **Bắt buộc đăng xuất**, không chỉ chuyển hướng: trang login thường tự đá người đang đăng nhập về dashboard → vòng lặp chuyển hướng. Và phải gọi `Response.Clear()` TRƯỚC `SignOutAsync`, gọi sau là cuốn luôn header `Set-Cookie` xóa cookie.
    + Chỉ mang `ReturnUrl` cho request GET: quay lại một đường POST sau khi đăng nhập là chạy lại thao tác ghi mà người dùng không hề bấm.
    + Cách dựng lại lỗi để kiểm chứng: đăng nhập bằng khóa hiện tại, rồi khởi động lại API với `Tokens__Key` khác — cookie vẫn còn hạn nhưng JWT bị từ chối, đúng tình huống thật.
- `App ASP.NET "treo" khi chạy trong Visual Studio thường là debugger đang dừng ở exception`: mọi request đứng im, curl timeout, không có log mới. Đừng đoán theo triệu chứng mạng — hỏi thẳng debugger qua MCP Visual Studio: `debugger_status` (trả `Mode = Break`, `LastBreakReason = ExceptionNotHandled`, file + dòng), `debugger_get_callstack`, `debugger_get_locals` (biến `$exception` cho biết kiểu và thông điệp). Nhanh hơn đọc log rất nhiều và chỉ đúng dòng ném.
- `NETSDK1152 lúc publish: hai file cùng tên theo project reference chảy vào cùng thư mục output`:
  Project thư viện (vd tầng DataBase) để `appsettings.json` với `CopyToOutputDirectory=PreserveNewest`
  cho design-time (`IDesignTimeDbContextFactory` đọc nó khi chạy `dotnet ef`). File đó **theo project
  reference** sang project web và đụng đúng tên với `appsettings.json` thật của web →
  `error NETSDK1152: Found multiple publish output files with the same relative path`.
    + Bẫy ở chỗ: `dotnet build`, F5, `dotnet run` đều **sạch trơn**; lỗi CHỈ xuất hiện lúc
      `dotnet publish` — tức đúng lúc sắp triển khai, thường là lúc gấp nhất.
    + Sửa: giữ `CopyToOutputDirectory` (design-time vẫn cần) nhưng thêm
      `<CopyToPublishDirectory>Never</CopyToPublishDirectory>` ở project thư viện.
    + ĐỪNG sửa bằng `ErrorOnDuplicatePublishOutputFiles=false` — nó chỉ tắt cảnh báo, còn file nào
      thắng thì tuỳ thứ tự build, có ngày publish ra bản mang connection string của máy dev.
- `Publish self-contained để chạy trên host không cài .NET`: `dotnet publish -c Release -r win-x64
  --self-contained true` nhét luôn runtime vào output (~105–125 MB/app cho ASP.NET Core 10). Dùng khi
  shared hosting/IIS chưa có ASP.NET Core Hosting Bundle đúng phiên bản.
    + SDK tự đổi `processPath` trong `web.config` sang `.\App.exe` nhưng **giữ nguyên**
      `arguments=".\App.dll"` nếu file nguồn có ghi. Đã kiểm thật: app vẫn khởi động và phục vụ
      HTTP 200 bình thường, không cần bỏ thuộc tính đó.
    + Vẫn cần module `AspNetCoreModuleV2` có sẵn trên IIS. Đặt `hostingModel="outofprocess"` thì
      module chỉ làm reverse proxy sang Kestrel nên ít kén phiên bản hơn `inprocess`.
- `dotnet run KHÔNG có launch profile = Environment Production → nuốt trọn appsettings.Production.json`:
  Khi project đã commit `appsettings.Production.json` (thường sau đợt làm triển khai), mọi lệnh
  `dotnet run --no-launch-profile` để smoke test đều đọc file đó, và hỏng theo **ba tầng nối tiếp**,
  mỗi tầng chỉ lộ ra sau khi sửa xong tầng trước:
    + `AllowedHosts` khóa vào tên miền thật → mọi request `localhost` trả **400 "Bad Request - Invalid
      Hostname"**. Thân lỗi trông y hệt trang lỗi của IIS/http.sys nên rất dễ đổ oan cho cổng bị chiếm;
      kiểm header `Server: Kestrel` là biết ngay chính app mình từ chối.
    + `ApiBaseUrl`/URL dịch vụ ngoài trỏ ra tên miền thật → `SocketException: No such host is known`.
    + `ConnectionStrings` trỏ vào DB hosting → app chạy nhưng đăng nhập/truy vấn nào cũng timeout.
  Sửa một phát: đặt `ASPNETCORE_ENVIRONMENT=Development` trước `dotnet run` (bỏ hẳn file Production)
  thay vì đè từng khóa bằng `-- --AllowedHosts "*" --ApiBaseUrl ...` — đè tay thì luôn sót khóa tiếp theo.

## `DateTime` gửi qua JSON: `Kind = Unspecified` bị TRỪ MÚI GIỜ LẦN THỨ HAI

- Tình huống thật: web đọc ngày người dùng chọn (giờ VN), quy về UTC bằng
  `DateTime.SpecifyKind(local, DateTimeKind.Unspecified).AddHours(-7)` rồi POST JSON sang API. Ngày lưu
  xuống database **lùi đúng một ngày** so với ô người dùng vừa chọn (chọn 13/08 → DB `12/08 10:00Z`
  thay vì `12/08 17:00Z`).
- Nguyên nhân: bộ chuyển JSON (và nhiều tầng khác: `ToUniversalTime()`, driver DB, `TimeZoneInfo`)
  coi `Unspecified` là **giờ của máy đang chạy**, nên khi cần một giá trị UTC nó trừ tiếp offset của
  máy — máy đặt UTC+7 thì trừ thêm 7 giờ nữa. `AddHours(-7)` của mình chỉ là lần trừ thứ nhất.
- Cách nhận ra: sai số đúng bằng offset của máy build/chạy, và **chỉ lộ trên máy có múi giờ khác UTC**.
  Máy CI chạy UTC sẽ cho kết quả đúng, nên test tự động rất dễ bỏ sót.
- Quy tắc: giá trị đã quy về UTC thì phải **đóng dấu `DateTimeKind.Utc` ngay tại chỗ tính**:
  `DateTime.SpecifyKind(local.AddHours(-7), DateTimeKind.Utc)`. Ranh giới truyền đi (JSON/HTTP/DB)
  chỉ an toàn với `Utc`; `Unspecified` là "tuỳ nơi diễn giải", còn `Local` thì phụ thuộc máy.
- Đối xứng ở chiều đọc: giá trị EF trả về có `Kind = Unspecified` (thực chất là UTC) nên trước khi
  đổi sang giờ địa phương phải `SpecifyKind(..., Utc)` — cùng một bẫy, ngược hướng.

## EF Core: entity con khoá `Guid` do code sinh, thêm vào **navigation collection** → EF hiểu là UPDATE chứ không INSERT

- Tình huống thật: entity cha đang được track (vừa `FirstOrDefaultAsync(...)` có `.Include(x => x.Children)`),
  thêm dòng con mới bằng `parent.Children.Add(new ChildDbo { Id = Guid.NewGuid(), ParentId = parent.Id, ... })`
  rồi `SaveChangesAsync()`. Kết quả: **`DbUpdateConcurrencyException`** —
  *"The database operation was expected to affect 1 row(s), but actually affected 0 row(s)"*.
- Nguyên nhân: EF quyết định Added hay Modified theo **khoá đã có giá trị hay chưa**. Với khoá
  store-generated (`int identity`) thì khoá còn `0` ⇒ Added, nên cách viết trên chạy đúng và ai cũng
  quen tay. Với khoá `Guid` **do code gán** (`ValueGenerated.Never`), khoá đã khác `Guid.Empty` ngay
  từ lúc khởi tạo ⇒ EF track nó như một dòng **đã tồn tại** (quy tắc của `Attach` cho cả graph) và
  sinh `UPDATE ... WHERE Id = @id` cho một dòng chưa hề có trong bảng ⇒ 0 dòng ảnh hưởng.
- Chữa: gọi thẳng **`dbContext.Children.Add(child)`** (hoặc `AddRange`) — `Add` ép state Added bất kể
  khoá. Gán sẵn `ParentId` là đủ, không cần thêm vào navigation.
- Cùng họ bẫy: thay cả danh sách con thì **chỉ `RemoveRange(cũ)`**, đừng gọi thêm `parent.Children.Clear()`,
  và đừng suy ra "đã dọn xong chưa" bằng `parent.Children.Count` — mấy dòng đã `Remove` vẫn nằm trong
  collection ở trạng thái Deleted cho tới lúc `SaveChanges`. Dùng một cờ `bool` tường minh.
- **Vì sao mất thời gian**: `DbUpdateConcurrencyException` là kiểu ngoại lệ mà mọi người đọc thành
  "có người khác vừa sửa dòng này", nên nhánh `catch` thường đã có sẵn một câu thông báo lạc quan
  kiểu "dữ liệu vừa thay đổi, mở lại rồi lưu lần nữa" — bấm lại bao nhiêu lần cũng y hệt. Hai luật rút ra:
  1. `catch (DbUpdateConcurrencyException ex)` phải **log kèm `ex`**, đừng chỉ log một dòng chữ:
     EF ném đúng kiểu đó cho *mọi* trường hợp "số dòng ảnh hưởng khác kỳ vọng", kể cả câu lệnh do
     chính mình dựng sai. Không có stack trace thì hai chuyện đó nhìn giống hệt nhau.
  2. Khoanh vùng bằng cách **tách từng thao tác**: chỉ-DELETE, chỉ-INSERT, DELETE+INSERT. Ở ca này
     chỉ-DELETE chạy được còn chỉ-INSERT hỏng — đủ để chỉ thẳng vào chỗ thêm dòng, thay vì đi ngờ
     concurrency token của bảng cha (`rowversion`) như phản xạ đầu tiên.

## `HttpClient` không tự gửi `User-Agent` → IIS/WAF trả 403, và nhìn cứ như lỗi bên server đích

- Tình huống thật (13/08/2026): web gọi API cùng hosting, **mọi** lời gọi ném
  `ApiException` với 403. Gọi đúng URL đó bằng trình duyệt hay `Invoke-WebRequest` thì **200**, nên
  mất một lúc để đi soi API, soi DB, soi chuỗi kết nối — trong khi API hoàn toàn khoẻ.
- Nguyên nhân: `System.Net.Http.HttpClient` **không đặt `User-Agent` mặc định**. IIS/Plesk (và phần
  lớn WAF, ModSecurity CRS rule 920320, Cloudflare) chặn request thiếu header này. 403 trả về là
  **trang HTML của IIS**, request chưa vào tới ứng dụng nên log của ứng dụng đích sạch trơn.
- Vì sao dễ chẩn đoán nhầm: mọi công cụ kiểm tra bằng tay đều tự gửi `User-Agent`
  (`curl`, `Invoke-WebRequest`, `HttpWebRequest`, trình duyệt) — chỉ `HttpClient` là không. Nên
  "gọi tay thì được, code thì 403" là **dấu hiệu đặc trưng** của chính lỗi này, không phải dấu hiệu
  của tường lửa theo IP.
- Cách tái hiện trong 10 giây, chạy được từ máy bất kỳ:
  ```powershell
  Add-Type -AssemblyName System.Net.Http
  $h = New-Object System.Net.Http.HttpClient
  $h.GetAsync('https://host/endpoint').Result.StatusCode   # 403
  $h2 = New-Object System.Net.Http.HttpClient
  $h2.DefaultRequestHeaders.Add('User-Agent','test')
  $h2.GetAsync('https://host/endpoint').Result.StatusCode  # 200
  ```
- Chữa: đặt một lần lúc đăng ký client, đừng đặt rải rác ở từng lời gọi.
  ```csharp
  client.DefaultRequestHeaders.UserAgent.ParseAdd($"MyApp/{version}");
  ```
- **Luật rút ra**: mọi `HttpClient` gọi ra ngoài process đều phải có `User-Agent` tường minh, kể cả
  khi hôm nay chạy được — nó sẽ chết vào đúng ngày hosting bật WAF, và triệu chứng lúc đó
  (403 hàng loạt, ứng dụng đích không có log) trỏ sai hướng.
- Điều tra kiểu này: **đừng tin log ở đầu gọi**. `ApiException` chỉ nói "status không thành công";
  phải lấy cho được **status code + body thật**. Ở đây body là HTML `Server Error 403` của IIS —
  chỉ riêng việc nó không phải JSON đã đủ kết luận request chết trước khi tới ứng dụng.

## Sửa `.cshtml` khi app đang chạy `dotnet run` thì KHÔNG có tác dụng — phải restart

Razor view được **biên dịch lúc build** (`Microsoft.AspNetCore.Mvc.Razor.RuntimeCompilation` không
bật theo mặc định). Sửa `.cshtml` rồi F5 lại trang thì server vẫn trả **markup cũ**, trong khi file
`wwwroot/*.css`/`*.js` lại cập nhật ngay vì chúng do static file middleware đọc thẳng từ đĩa.

Cái bẫy nằm ở chỗ hai thứ đó **lệch pha nhau**: thêm một class mới vào `.cshtml` rồi viết quy tắc CSS
cho class đó, CSS mới lên ngay còn class thì chưa có → mọi quy tắc mới "không có tác dụng". Rất dễ
kết luận sai là selector viết sai, độ ưu tiên thấp, hay trình duyệt cache CSS, rồi sửa CSS thêm mấy
lượt (đã vấp: mất 3 lượt chụp ảnh + 2 lượt đo bằng CDP cho đúng một class chưa được biên dịch).

Cách kiểm 5 giây, làm TRƯỚC khi nghi ngờ CSS: xem class đó có thật trong HTML server trả về hay chưa.

```bash
curl -s http://localhost:5291/duong-dan | grep -o 'class="[^"]*ten-class[^"]*"'
```

Không thấy → restart app (`dotnet run` lại), đừng sửa CSS thêm dòng nào. Còn nếu đang lặp nhanh trên
view thì bật runtime compilation cho môi trường Development:
`services.AddRazorPages().AddRazorRuntimeCompilation()` + package
`Microsoft.AspNetCore.Mvc.Razor.RuntimeCompilation`.

## Tự động hoá GUI app khác bằng click chuột THẬT: "app chặn click ảo" thường là chẩn đoán nhầm của lệch DPI
- Bối cảnh: điều khiển GUI của app khác (Hotspot Shield) bằng cách chiếm chuột hệ thống — `SetCursorPos(x,y)` + `mouse_event(MOUSEEVENTF_LEFTDOWN/UP)`. Đây là input mức hardware, được hit-test bình thường ⇒ **hầu hết app đều nhận**, kể cả app "được cho là chặn click ảo". Chỉ app tự cài `WH_MOUSE_LL` rồi loại bỏ event có cờ `LLMHF_INJECTED` mới chặn được — hiếm. Đừng vội kết luận "app chặn" khi click không ăn.
- **Thủ phạm thật hay gặp: DPI.** Nếu process **không DPI-aware** mà màn hình scale ≠ 100% (vd 150%), thì `SetCursorPos`/`GetCursorPos` chạy trong toạ độ **ảo (đã ÷ scale)**, còn `DwmGetWindowAttribute(EXTENDED_FRAME_BOUNDS)` (rect cửa sổ) và ảnh chụp (WinRT Graphics Capture / màn hình) là **pixel vật lý**. Trộn hai hệ ⇒ click rơi lệch đúng bằng tỉ lệ scale. Cách phát hiện: chụp lại thấy con trỏ đáp ở chỗ khác toạ độ đã lệnh; lấy (vị trí thật)/(toạ độ lệnh) ra ~1.5 là đang 150%.
- **Sửa:** cho process DPI-aware NGAY ĐẦU `Main`, trước mọi thao tác toạ độ:
  ```csharp
  [DllImport("user32.dll")] static extern bool SetProcessDpiAwarenessContext(IntPtr v);
  [DllImport("user32.dll")] static extern bool SetProcessDPIAware();
  static readonly IntPtr PER_MONITOR_AWARE_V2 = (IntPtr)(-4);
  // gọi: if (!SetProcessDpiAwarenessContext(PER_MONITOR_AWARE_V2)) SetProcessDPIAware();
  ```
  Xong thì `SetCursorPos`, rect cửa sổ (DWM) và ảnh chụp cùng một hệ pixel vật lý ⇒ `screen = windowRect.TopLeft + (pixel trong ảnh)` click trúng.
- **Đưa cửa sổ đích lên trước:** `SetForegroundWindow` trần bị khoá foreground của Windows chặn khi gọi từ process nền (chỉ nháy taskbar). Dùng thủ thuật `AttachThreadInput(curThread, targetThread, true)` + `BringWindowToTop` + `SetForegroundWindow` + `ShowWindow(SW_SHOW)` rồi detach. Và nếu **console của chính app** che cửa sổ đích (hay che vì nằm giữa màn hình) thì ẩn nó: `GetConsoleWindow()` + `ShowWindow(h, SW_HIDE)`.
- **Cửa sổ WPF của app khác:** `PrintWindow` cổ điển hay ra đen; **WinRT Graphics Capture** (`InitWindow(hwnd)`) bắt được, nhưng frame về **bất đồng bộ** nên phải poll `Capture()` vài lần (mỗi lần ~200ms) mới có bitmap.
- Cửa sổ vừa được active thường "nuốt" cú tương tác đầu ⇒ sau khi foreground nên delay (~1–1.5s) rồi mới click; di chuột theo nhiều bước (nhân hoá) cũng giúp qua vài app chống bot.

## Kestrel: "Failed to bind to address ... address already in use"

- Thủ phạm chiếm port **thường không phải project khác trong solution**. Trước khi đổ lỗi cho cấu hình, xác định chủ sở hữu port bằng PID:
  ```powershell
  netstat -ano -p tcp | Select-String ":8080 .*LISTENING"
  Get-CimInstance Win32_Process -Filter "ProcessId=<PID>" | Select ProcessId,Name,CommandLine
  ```
  Ca thật gặp: `NVIDIA Broadcast.exe` giữ `127.0.0.1:8080` (nó có web UI kiểu Express, `curl http://localhost:8080/` trả `Cannot GET /`). Các app desktop hay chiếm port "đẹp" 8080/3000/5000/8888.
- Khi VS đang **Break** vì exception này, lấy đúng thông điệp qua Immediate/`debugger_evaluate`: `$exception.Message` — nhanh hơn đọc Output.
- Đổi port thì phải quét cả `localhost:<port>` lẫn mention trần trong docs; **giữ nguyên** reference container/docker (`svc-name:8080`, `"8080:8080"`, `ASPNETCORE_URLS=http://+:8080`) vì đó là port trong container, không đụng máy local.
- Dev server (Vite/webpack) và host ASP.NET phục vụ chính SPA đó **phải khác port**; đặt trùng thì chỉ nổ vào lúc chạy song song, build vẫn xanh nên rất dễ lọt.
- Chạy thử `.exe` đã build để kiểm port: nhớ `$env:ASPNETCORE_ENVIRONMENT = "Development"`, không thì nó đọc appsettings.json (connection string production) và chết **trước** `app.Run()` — nhìn giống hệt lỗi bind nhưng không phải.

## NETSDK1152 "multiple publish output files with the same relative path" — do class library đặt tên file trùng app host

- **Triệu chứng:** `dotnet publish` app host báo `NETSDK1152` liệt kê 2 đường dẫn `appsettings.json` — một của app, một của project library được tham chiếu.
- **Nguyên nhân:** `<None Update="x.json"><CopyToOutputDirectory>` của project A **chảy sang thư mục output/publish của mọi project tham chiếu A**. Trùng tên ⇒ trùng đường dẫn tương đối. Lúc `build` MSBuild đè âm thầm (không xác định file nào thắng — bug tiềm ẩn nguy hiểm hơn); chỉ `publish` mới ném lỗi.
- **Ca hay gặp:** project Database giữ `appsettings.json` riêng cho `IDesignTimeDbContextFactory` (`dotnet ef` cần file cạnh assembly vì factory dùng `AppContext.BaseDirectory`).
- **Cách sửa:** ĐỔI TÊN file trong library sang tên không thể trùng (vd `efdesign.json`, `efdesign.Development.json`) rồi sửa factory đọc tên mới. MSBuild KHÔNG có cờ per-item để chặn item chảy sang project tham chiếu — `CopyToPublishDirectory=Never` chỉ chặn ở bước publish, build vẫn đè nhau.
- Thêm `<CopyToPublishDirectory>Never</CopyToPublishDirectory>` cho file design-time: connection string dev (thường có `sa`/password) không lọt vào bản giao đi.
- Nhớ cập nhật `.gitignore` nếu file cũ được ignore theo tên (`**/appsettings.Development.json` không còn khớp tên mới).

## Project ASP.NET thiếu `launchSettings.json` ⇒ F5 chạy ở **Production**, có thể migrate nhầm database thật

- **Triệu chứng:** F5 project API trong Visual Studio thì ném `SqlException: Login failed for user '<user production>'` ngay lúc khởi động (thường ở `db.Database.MigrateAsync()` trong bước init database), trong khi `dotnet run` với `ASPNETCORE_ENVIRONMENT=Development` lại chạy ngon. Dễ chẩn đoán nhầm thành "sai connection string" hoặc "sai port".
- **Nguyên nhân:** không có `Properties/launchSettings.json` thì không ai đặt `ASPNETCORE_ENVIRONMENT`, mà mặc định của ASP.NET Core là **Production** ⇒ nạp `appsettings.Production.json`. VS vẫn chạy được project bình thường nên không có cảnh báo nào cả.
- **Vì sao nguy hiểm hơn một lỗi login:** nếu tài khoản production đăng nhập ĐƯỢC (vd chuỗi kết nối để `Server=.` và user đó có thật trên máy dev), `MigrateAsync` sẽ apply migration lên đúng database production ngay khi bấm F5. Lỗi login ở trên thực ra là cái may.
- **Cách sửa:** tạo `Properties/launchSettings.json` cho MỌI project khởi chạy được, đặt `"ASPNETCORE_ENVIRONMENT": "Development"` và `applicationUrl` khớp `"Urls"` trong appsettings. Kiểm chứng bằng dòng log `Hosting environment: Development` lúc khởi động, đừng tin cảm giác.
- **Bẫy kèm theo — generic host dùng biến KHÁC:** project dựng bằng `Host.CreateApplicationBuilder` (worker/console, không phải `WebApplication`) đọc `DOTNET_ENVIRONMENT`, **không** đọc `ASPNETCORE_ENVIRONMENT`; `applicationUrl` trong profile cũng vô nghĩa với nó. Đặt nhầm biến thì profile trông đúng mà chẳng có tác dụng gì.
- **Bẫy kèm theo — cờ dev tự chế:** có host quyết định chế độ dev bằng tham số dòng lệnh (`args.Contains("--dev")`) chứ không theo biến môi trường; khi đó phải thêm `"commandLineArgs": "--dev"` vào profile.
- `launchSettings.json` là JSON thuần: **không viết comment `//`** trong đó (khác `appsettings.json` đọc qua `Microsoft.Extensions.Configuration.Json` vốn bỏ qua comment).

## EF Core: thêm entity con vào navigation của parent đã tracked ⇒ state **Modified** chứ không phải Added (khi khoá sinh sẵn ở client)

- **Triệu chứng:** `SaveChanges` ném `DbUpdateConcurrencyException: The database operation was expected to affect 1 row(s), but actually affected 0 row(s)`. Nghe như xung đột đồng thời (và tài liệu Microsoft cũng dẫn về trang optimistic concurrency) nên rất dễ đi lạc sang hướng rowversion/retry. Thực ra là EF phát `UPDATE` vào một hàng **chưa hề tồn tại**.
- **Nguyên nhân:** entity con khai khoá kiểu `public Guid Id { get; set; } = Guid.NewGuid();`. Khi `DetectChanges` gặp entity mới trong navigation collection của một parent đang tracked, EF quyết định state theo khoá: khoá đã có giá trị khác default ⇒ coi là hàng cũ ⇒ `Modified`. Không có exception nào lúc Add, chỉ vỡ ở `SaveChanges`.
- **Vì sao dễ lọt:** cùng entity đó lúc TẠO MỚI parent thì chạy tốt, vì `DbSet.Add(parent)` đánh dấu cả graph là `Added`. Bug chỉ hiện ở luồng SỬA, tức luồng viết sau và thường ít test hơn.
- **Cách kiểm trong 10 giây** (đừng đoán state, in nó ra):
  ```csharp
  db.ChangeTracker.DetectChanges();
  foreach (var e in db.ChangeTracker.Entries<ChildDbo>())
      Console.WriteLine($"{e.Entity.SomeField} => {e.State}");   // mong doi Added
  ```
- **Cách sửa:** nói rõ state ra, đừng để EF suy: thêm vào navigation **và** `db.Add(child)` (đối xứng với `db.Remove(child)` khi xoá). `DbContext.Add` luôn đặt `Added` bất kể khoá đã có giá trị hay chưa.
- **`DbSet.Update(parent)` KHÔNG cứu được** — nó dùng đúng luật "khoá đã set ⇒ Modified" cho cả graph, nên hàng con mới vẫn thành `UPDATE`.
- **Cách chẩn đoán rẻ nhất khi lỗi chỉ xảy ra trên server:** dựng một console app tạm tham chiếu thẳng project Repositories/DbContext, seed một hàng giả bằng SQL rồi gọi đúng method đang nghi. Tái hiện được thì có stack trace thật ngay, khỏi phải xin log production. Nhớ `<ManagePackageVersionsCentrally>false</ManagePackageVersionsCentrally>` trong csproj tạm nếu repo dùng CPM, và xoá hàng giả sau khi xong.

## Xoá thư mục clone sau khi `dotnet build`: `Device or resource busy` dù thư mục đã rỗng

- **Triệu chứng:** build/test xong trong một bản clone tạm (vd `d:\tmp\<repo>`), `rm -rf` xoá được hết nội dung nhưng thư mục GỐC không xoá nổi: `rm: cannot remove '<dir>': Device or resource busy`. `ls -a` cho thấy chỉ còn `.` và `..`.
- **Nguyên nhân:** các build server của .NET vẫn sống sau khi lệnh build trả về và giữ thư mục làm việc — VBCSCompiler (Roslyn) và MSBuild server. `-nodeReuse:false` trong `Directory.Build.rsp` chỉ tắt node reuse của MSBuild worker, KHÔNG tắt hai server này.
- **Cách sửa:** `dotnet build-server shutdown` rồi xoá lại (`rmdir` là đủ nếu nội dung đã bay). Không cần kill `dotnet.exe` theo tên — làm vậy có thể giết luôn tiến trình khác của user.
- **Phòng trước:** trong quy trình "clone ra chỗ khác → build → xoá", gọi `dotnet build-server shutdown` NGAY trước bước xoá, coi như một bước cố định.

## Header HTTP chỉ chở được ASCII in được — `TryAddWithoutValidation` KHÔNG kiểm

- **Triệu chứng:** lời gọi HttpClient hỏng sau ~2–5ms với
  `System.Net.Http.HttpRequestException: Request headers must contain only ASCII characters`
  tại `HttpConnection.WriteHeaderCollection`. Vì thời gian gần 0ms nên **request chưa hề rời máy**:
  phía server không có một dòng log nào, và mọi thông báo lỗi kiểu "xem log phía server" đều dẫn sai
  đường. Dễ chẩn nhầm thành lỗi mạng/lỗi provider.
- **Nguyên nhân:** đặt chuỗi phi ASCII (tên tiếng Việt, tên người dùng, tên tệp...) vào header.
  `DefaultRequestHeaders.TryAddWithoutValidation` — như đúng tên gọi — **nhận tuốt, không kiểm gì**;
  lỗi chỉ nổ ở tầng ghi socket. `Add()` có kiểm nhưng cũng chỉ kiểm định dạng, không cứu được ở đây.
- **Cách sửa:** mã hoá giá trị trước khi đặt vào header. Dạng RFC 2047 `=?UTF-8?B?<base64>?=` là gọn
  nhất: nhìn log biết ngay là chuỗi đã mã hoá, và bên nhận chỉ giải mã khi thấy dấu hiệu đó nên bên
  gửi bản cũ (chưa mã hoá) vẫn chạy được với bên nhận bản mới.
- **Bẫy kèm:** header đặt vào `DefaultRequestHeaders` **dính lại trên HttpClient**, nên một giá trị
  hỏng làm chết mọi lời gọi sau trên cùng client, không riêng lời gọi đầu tiên.
- **Viết test cho đúng:** `TryAddWithoutValidation` rồi assert là **vô dụng** — nó luôn thành công.
  Phải dựng `HttpRequestMessage` rồi kiểm từng ký tự nằm trong `' '..'~'` (hoặc gửi thật qua một
  handler giả) mới bắt được.

## Newtonsoft: `token["x"]?.Value<string>()` không biên dịch được

`JToken` cũng cài `IEnumerable<JToken>`, nên khi gọi `.Value<string>()` trên một `JToken` lấy bằng
chỉ mục, trình biên dịch chọn nhầm extension `JToken.Value<T>(this IEnumerable<JToken>, object key)`
và báo:

```
error CS7036: There is no argument given that corresponds to the required parameter 'key' of 'JToken.Value<T>(object)'
```

Cách viết đúng: ép kiểu tường minh `(string?)token["x"]`, hoặc `token["x"]?.ToString()`.
Chỉ dùng `.Value<T>()` khi biến đã có kiểu tĩnh là `JValue`/`JProperty`, không phải `JToken`.

## Thiếu thư mục `wwwroot` làm hỏng publish ở bước sinh script EF migration

- **Triệu chứng:** publish từ Visual Studio dừng với đúng một dòng
  `Error | Entity Framework SQL Script generation failed | <Project> | | 0` — không tên file, không
  số dòng, không nói migration nào. Nhìn như migration hỏng, nhưng migration hoàn toàn bình thường.
- **Tái hiện ngoài VS** (VS gọi EF với project = startup-project = chính project API, không phải
  project chứa DbContext):
  `dotnet ef migrations script --idempotent --context <Ns.MyDbContext> --project <ApiProject> --startup-project <ApiProject>`
  Ra hai dòng thật sự có ích:
  `An error occurred while accessing the Microsoft.Extensions.Hosting services. Continuing without the application service provider. Error: <đường dẫn>\wwwroot\`
  rồi `No DbContext named '<Ns.MyDbContext>' was found.`
  Dòng thứ hai chỉ là hệ quả: mất service provider nên EF quay sang quét assembly của ApiProject,
  nơi không có DbContext.
- **Nguyên nhân:** `WebApplication.CreateBuilder(...)` dựng `WebRootFileProvider` **ngay lúc khởi
  tạo**, và một `PhysicalFileProvider` trỏ vào thư mục không tồn tại ném
  `DirectoryNotFoundException` mà `Message` **chỉ là đường dẫn trần** — nên EF in ra mỗi cái path.
  `wwwroot` thường không được commit (không file nào của nó được theo dõi), nên máy vừa clone hoặc
  vừa `git clean -fdx` là không có.
- **Cách sửa: commit một file giữ chỗ** (`wwwroot/.gitkeep`, hoặc đặt sâu hơn để giữ cả cây).
- **ĐỪNG sửa bằng `Directory.CreateDirectory` trong `Program.cs`** — ĐÃ THỬ VÀ THẤT BẠI, kể cả khi
  đặt ngay trước `builder.Build()`. Lỗi phát sinh từ `CreateBuilder` ở dòng đầu, trước mọi dòng lệnh
  của mình, nên không chỗ nào trong `Program.cs` là đủ sớm.
- **Vì sao chạy app bình thường lại không lộ:** app thật chạy đủ nên các nhánh tạo thư mục phía sau
  kịp có tác dụng cho những lần sau; còn `dotnet ef` chỉ DỰNG host chứ không chạy, nên ăn trọn
  exception ngay lần đầu.
- **Bẫy đi kèm, khác lỗi trên nhưng cùng chỗ:** `AppContext.BaseDirectory` (thư mục chứa dll) và
  `IWebHostEnvironment.ContentRootPath` (thư mục project khi chạy dev) **KHÁC NHAU lúc F5** và chỉ
  trùng nhau sau khi publish. Controller đọc/ghi file theo `BaseDirectory` trong khi `UseStaticFiles`
  phục vụ theo `ContentRootPath` là: upload thành công, mà tải về 404 — chỉ ở môi trường dev.

## Razor: tên biến trùng directive `@section`

Đặt tên biến vòng lặp trong `.cshtml` là `section` (vd `@foreach (var section in ...)`) làm trình
biên dịch Razor đọc `@section.Key` thành directive `@section`, ném `RZ2005: The 'section' directive
must appear at the start of the line` + `RZ1011` — thông báo lỗi không hề nhắc tới tên biến, nên
rất dễ đi tìm sai chỗ. Đổi tên biến là xong. Cùng bẫy với mọi từ khoá directive khác khi đứng ngay
sau `@`: `section`, `page`, `model`, `using`, `functions`, `inject`, `code`.

## "App WPF treo" — dump stack bằng `dotnet-stack` trước khi đoán

Người dùng báo app đứng im giữa một lượt chạy dài (gọi AI, Selenium…) thì đừng đọc code đoán chỗ
treo. Cài một lần `dotnet tool install -g dotnet-stack`, rồi:

```powershell
$env:PATH += ";$env:USERPROFILE\.dotnet\tools"
dotnet-stack report --process-id <pid>
```

Nó in stack **managed** của mọi luồng, đọc là thấy ngay. Ca thật (2026-08-29, VeoFlow): stack luồng
UI là `DialogService.Confirm ← ScriptInputViewModel.AskReplace ← ApplyPlan` ⇒ app **không** treo, nó
đang chờ người dùng trả lời một hộp thoại mà người dùng không nhìn thấy. Đoán mò từ code sẽ đi tìm
chỗ chờ trong tầng Selenium — sai hoàn toàn.

Mẹo kèm theo: liệt kê cửa sổ của tiến trình bằng UI Automation để biết có hộp thoại nào đang mở:

```powershell
Add-Type -AssemblyName UIAutomationClient, UIAutomationTypes
$root = [System.Windows.Automation.AutomationElement]::RootElement
$w = [System.Windows.Automation.TreeWalker]::ControlViewWalker
$c = $w.GetFirstChild($root)
while ($null -ne $c) { if ($c.Current.ProcessId -eq $pid) { $c.Current.Name }; $c = $w.GetNextSibling($c) }
```

## MessageBox có owner thì KHÔNG bao giờ hiện trên thanh tác vụ

`MessageBox.Show(owner, …)` tạo cửa sổ **owned**, mà Windows không cấp nút taskbar riêng cho cửa sổ
owned. Cộng thêm luật "chỉ app đang ở tiền cảnh mới được tự đưa cửa sổ lên trước": app chạy nền bật
hộp thoại lên thì nó nằm chìm dưới cửa sổ người dùng đang dùng, **không có mục nào trên taskbar để
bấm vào** — nhìn hệt như app treo.

Ba đường xử lý, không đường nào miễn phí:

1. **Bỏ owner** (`MessageBox.Show(message, …)`) — có nút taskbar, nhưng mất tính modal: người dùng
   bấm được vào cửa sổ chính và hộp thoại lại chui xuống dưới nó.
2. **`FlashWindowEx`** trên cửa sổ chủ (cờ `FLASHW_TRAY | FLASHW_TIMERNOFG`) — nháy nút taskbar cho
   tới khi được đưa lên trước. Đây là cách Windows chừa lại đúng cho tình huống này.
3. **Cửa sổ tự vẽ** (`Window` + `Owner` + `ShowInTaskbar = true`) — vừa modal vừa có mục taskbar,
   đổi lại phải tự dựng nút và tự map kết quả.

Chỗ báo lỗi cuối cùng (khởi động hỏng, `DispatcherUnhandledException`) thì cứ giữ `MessageBox` không
owner: lúc ấy DI hoặc theme có thể chính là thứ đang hỏng, mà cửa sổ tự vẽ lại cần cả hai.

## WPF: `DataGridCheckBoxColumn` bind vào property chỉ có `get` ⇒ crash ngay `Window.Show()`
- Triệu chứng: app WPF chết ngay lúc khởi động, stack dừng ở `App.OnStartup` → `Window.Show()`, exception thật là `InvalidOperationException: A TwoWay or OneWayToSource binding cannot work on the read-only property 'X' of type 'Y'`, ném từ `PropertyPathWorker.CheckReadOnly` trong `DataBindEngine.OnLayoutUpdated`.
- Nguyên nhân: `CheckBox.IsChecked` đăng ký với `BindsTwoWayByDefault = true`. `DataGridCheckBoxColumn` gắn binding vào chính DP đó, nên `Mode` mặc định resolve thành **TwoWay** — kể cả khi DataGrid đã `IsReadOnly="True"` (cờ đó chỉ chặn edit của người dùng, không đổi Mode của binding). Source là property `{ get; }` ⇒ WPF ném lỗi lúc attach binding.
- Vì sao `DataGridTextColumn` cùng grid lại KHÔNG lỗi: display element của nó là `TextBlock`, mà `TextBlock.Text` mặc định OneWay ⇒ property chỉ có `get` vẫn chạy tốt. Chỉ cột check-box (và các DP TwoWay-by-default khác: `Selector.SelectedItem`, `TextBox.Text`, `ToggleButton.IsChecked`) mới dính.
- Cách sửa: ghi `Mode=OneWay` tường minh — `Binding="{Binding IsAttached, Mode=OneWay}"`. Đừng "sửa" bằng cách thêm `set;` vô nghĩa cho property.
- Bẫy chẩn đoán: lỗi này KHÔNG hiện lúc build, cũng không hiện trong designer; nó chỉ nổ ở lần layout đầu tiên. Khi debugger dừng ở `Window.Show()` mà không rõ lý do, đọc `$exception.ToString()` chứ đừng đọc mỗi `LastBreakReason`.

## WPF: `DataGridComboBoxColumn.ItemsSource` bind bằng `RelativeSource` ⇒ dropdown rỗng, không báo lỗi

**Triệu chứng**: mọi ComboBox trong `DataGrid` xổ ra danh sách trống. Build sạch, không exception, không dòng log nào.

**Nguyên nhân**: `DataGridColumn` KHÔNG phải con visual, cũng KHÔNG phải con logical của `DataGrid` — nó chỉ nằm trong `Columns` collection. Binding đặt trên thuộc tính của chính cột vì thế không có tổ tiên để đi ngược lên:

```xml
<!-- SAI: không bao giờ phân giải, ItemsSource ở lại null -->
<DataGridComboBoxColumn ItemsSource="{Binding DataContext.Kinds,
                                      RelativeSource={RelativeSource AncestorType=UserControl}}"/>
```

Lưu ý phân biệt: `SelectedItemBinding` / `SelectedValueBinding` / `Binding` của cột thì CHẠY BÌNH THƯỜNG — chúng được áp vào cell (đang ở trong tree), chỉ binding trên bản thân cột mới hỏng. Nên lỗi rất dễ bị bỏ qua: cột vẫn lưu được giá trị, chỉ là không có gì để chọn.

**Cách sửa — `BindingProxy` (Freezable)**: một `Freezable` đặt trong `Resources` của phần tử sẽ được WPF cấp inheritance context của phần tử đó, gồm cả `DataContext`.

```csharp
public sealed class BindingProxy : Freezable
{
    public static readonly DependencyProperty DataProperty =
        DependencyProperty.Register(nameof(Data), typeof(object), typeof(BindingProxy));
    public object? Data { get => GetValue(DataProperty); set => SetValue(DataProperty, value); }
    protected override Freezable CreateInstanceCore() => new BindingProxy();
}
```

```xml
<UserControl.Resources>
    <b:BindingProxy x:Key="Vm" Data="{Binding}"/>
</UserControl.Resources>
...
<DataGridComboBoxColumn ItemsSource="{Binding Data.Kinds, Source={StaticResource Vm}}"/>
```

**Cách test tự động** (lỗi này chỉ lộ khi chạy thật, nên đáng viết guard): dựng view trên STA thread, gán `DataContext` là một stub có đủ tên thuộc tính, `window.Show()` + `UpdateLayout()`, rồi duyệt visual tree tìm `DataGrid` và khẳng định mọi `DataGridComboBoxColumn.ItemsSource != null`.

Ba cái bẫy khi viết guard đó:
1. **`Application.ShutdownMode` mặc định là `OnLastWindowClose`** — đóng window đầu tiên là dispatcher tắt, mọi view sau đó `IsLoaded == false`, `VisualTreeHelper.GetChildrenCount` trả 0, test "pass" vì không soi cái gì. Đặt `ShutdownMode = OnExplicitShutdown`.
2. **Phải đếm số cột đã soi và assert con số đó**, nếu không cây tree rỗng sẽ làm test xanh giả.
3. **Một `Application` cho mỗi process**: xUnit chạy các test class song song, hai class cùng `new Application()` sẽ ném `Cannot create more than one System.Windows.Application instance`. Cho chúng chung một `[CollectionDefinition]` để chạy tuần tự.

**Bài học về cách chẩn đoán**: đừng đoán từ đọc XAML. Tôi đã nghi nhầm `<Binding/>` rỗng trong `MultiBinding` (thực ra chạy đúng) và suýt "sửa" nhầm chỗ. Cách nhanh và chắc là dựng luôn control thật trong một xUnit STA test rồi in ra trạng thái: `BindingOperations.GetMultiBindingExpression(...).Status`, `ItemsSource`, `ItemContainerGenerator.ContainerFromIndex(i)`. Có số liệu rồi mới sửa — và luôn kiểm lại rằng bản CHƯA sửa thật sự làm test đỏ, nếu không là đang sửa nhầm chỗ.

## WPF: `DataGridColumn.Header` bind `{DynamicResource}` ⇒ đóng băng ngôn ngữ lúc lưới được dựng

Cùng gốc với lỗi `ItemsSource` ở mục trên: cột không nằm trong tree nên **không nhận được tín hiệu invalidate** khi ta hoán đổi `ResourceDictionary` chuỗi trong `Application.Resources`. `Header="{DynamicResource Str.App.Type}"` phân giải MỘT LẦN lúc lưới được realize rồi giữ nguyên ngôn ngữ tại thời điểm đó suốt phiên.

Triệu chứng rất dễ đọc nhầm: đổi ngôn ngữ thì mọi nhãn khác đổi theo, riêng tiêu đề cột đứng im ⇒ nhìn như "chỗ này quên dịch", trong khi chuỗi dịch có đủ. Nếu app khởi động bằng ngôn ngữ A rồi user chuyển sang B thì tiêu đề cột kẹt ở A còn phần còn lại là B — nhìn vào ảnh chụp màn hình sẽ tưởng ngược lại là phần còn lại mới hỏng.

**Cách sửa** — bind vào một object tĩnh (không cần tree) thay đổi theo ngôn ngữ, tra chuỗi trong converter:

```csharp
public sealed class LocalizedBinding : Binding
{
    public LocalizedBinding(string key) : base(nameof(LocalizationScope.Version))
    {
        Source = LocalizationScope.Instance;   // static ⇒ không cần visual tree
        Mode = BindingMode.OneWay;
        Converter = KeyLookup;                 // trả LocalizationManager.Get((string)parameter)
        ConverterParameter = key;
    }
}
```
```xml
Header="{b:LocalizedBinding Str.App.Type}"
```

`Binding` vốn là `MarkupExtension` nên kế thừa nó cho cú pháp gọn; constructor nhận string dùng được ở dạng positional trong XAML.

**Bẫy khi viết test cho nó**: cách so tự nhiên là "chụp tiêu đề ở EN, đổi sang VI, khẳng định mọi cặp đều khác nhau" — nhưng có những từ dịch y hệt nhau ở hai ngôn ngữ (`Argument`), làm test đỏ giả. Đừng hard-code danh sách ngoại lệ: duyệt cả hai `ResourceDictionary`, dựng tập các chuỗi mà bản dịch trùng chính nó, rồi loại tập đó ra. Tự bảo trì khi thêm chuỗi mới.

## WPF: template TextBox/PasswordBox đặt `Margin="{TemplateBinding Padding}"` lên `PART_ContentHost` là ÁP PADDING HAI LẦN

- Triệu chứng: placeholder (watermark TextBlock nằm chồng trong template) và chữ người dùng gõ KHÔNG
  thẳng hàng; con trỏ nhấp nháy lệch hẳn so với chữ mờ nó sắp thay thế. Không lỗi build, không binding
  error, nhìn qua tưởng lỗi font.
- Nguyên nhân: `TextBox`/`PasswordBox` TỰ thụt `TextBoxView` của nó theo `Padding` rồi. Template mà
  còn đặt thêm `Margin="{TemplateBinding Padding}"` lên `PART_ContentHost` thì chữ bị thụt HAI lần,
  còn watermark do template tự bố trí nên chỉ thụt MỘT lần ⇒ lệch đúng bằng `Padding.Left`.
  Sau khi bỏ margin đó vẫn còn lệch **2px** nữa: `TextBoxView` chừa sẵn một rãnh 2px bên trái cho
  con trỏ. Muốn chữ trùng watermark thì đặt `Margin="-2,0,0,0"`.
- Cách đo cho chắc, đừng ước lượng bằng mắt (đã mất thời gian vì tin vào suy đoán):
  - Toạ độ chữ: `textBox.GetRectFromCharacterIndex(0).X` (phải có sẵn ký tự; hộp rỗng vẫn trả về
    được nhưng nên set `Text="x"` cho chắc). Toạ độ watermark:
    `textBlock.TransformToAncestor(textBox).Transform(new Point(0,0)).X`.
  - Chốt hạ bằng pixel: `RenderTargetBitmap.Render(control)` + `CopyPixels`, quét cột đầu tiên có
    pixel khác màu nền. Đây là thứ duy nhất không cãi được.
  - `PasswordBox` không có `GetRectFromCharacterIndex` ⇒ so bằng vị trí `PART_ContentHost`
    (`control.Template.FindName("PART_ContentHost", control)`).
- Sửa xong PHẢI để lại test giữ số đó: con số `-2` là fudge factor, không có test thì sau này không
  ai dám đụng vào nữa.
- Nhớ soát HẾT các template cùng họ trong theme: `TextBox`, `PasswordBox`, `DatePickerTextBox`,
  `ComboBox` (phần editable) đều có `PART_ContentHost` và thường được chép qua lại nên dính cùng lỗi.

## WPF: đặt `IsReadOnly=True` trong style implicit của `DataGrid` (theme) = khoá lén toàn bộ ô text/checkbox

- Triệu chứng: user báo "grid không sửa được" nhưng chỉ đúng một nửa — click vào ô text/checkbox
  không có gì xảy ra (không vào edit mode, không hiện caret), trong khi các cột
  `DataGridComboBoxColumn` VẪN chọn được bình thường. Nhìn như grid hỏng lỗ chỗ chứ không như một
  bảng chỉ-đọc, nên rất dễ đi tìm nhầm chỗ (binding, converter, UpdateSourceTrigger...).
- Nguyên nhân: `DataGridComboBoxColumn.GenerateElement` sinh ra một `ComboBox` THẬT làm phần tử
  hiển thị (không phải TextBlock, không tắt hit-test), và nó ghi thẳng vào source qua
  `SelectedItemBinding` mà không cần grid vào edit mode. Ngược lại `DataGridTextColumn` hiển thị
  `TextBlock` và `DataGridCheckBoxColumn` sinh `CheckBox` với `IsHitTestVisible=false` —
  cả hai phụ thuộc hoàn toàn vào edit mode, mà `IsReadOnly=True` chặn đúng chỗ đó.
- Bài học chung: **theme chỉ nói control TRÔNG thế nào, không nói người dùng ĐƯỢC LÀM GÌ.**
  `IsReadOnly`, `CanUserAddRows`, `IsEnabled`, `SelectionMode`... đặt tại nơi dùng, không đặt trong
  style implicit dùng chung cho cả app. Grid nào thật sự chỉ-đọc thì tự khai báo tại chỗ.
- Bẫy khi viết test: `DataGridColumn.IsReadOnly` có coerce từ `DataGrid.IsReadOnly` (grid read-only
  ⇒ MỌI cột trả về true) và `DataGridBoundColumn` còn coerce true khi binding là OneWay. Nên khi
  duyệt cột để tìm cột bị khoá, phải bỏ qua các grid vốn đã read-only, nếu không test sẽ báo lỗi giả
  cho từng cột của bảng chỉ-đọc.
- Muốn một HÀNG hoàn toàn không sửa được (vd bản ghi built-in), đừng khoá từng cột: đặt
  `DataGrid.RowStyle` + `DataTrigger` → `IsEnabled=False` (kèm `Opacity` cho thấy rõ). Đó là cách
  duy nhất chặn luôn được `ComboBox` hiển thị nói trên.
- `BasedOn="{StaticResource {x:Type X}}"` cho style **implicit** đặt trong chính
  `UserControl.Resources` là tự tham chiếu (cyclic) — đặt inline trong `DataGrid.RowStyle` hoặc đổi
  sang `x:Key` để tránh.
- Trigger so enum trong XAML: dùng `Value="{x:Static ns:Enum.Member}"` thay vì `Value="Ten"` để lỗi
  sai tên bị bắt lúc parse chứ không im lặng thành "không hàng nào khớp".

**Hệ quả kèm theo (đã mắc)**: khoá hàng built-in bằng `IsEnabled=False` + `Opacity` cho cả
`DataGridRow` nghe hợp lý, nhưng nếu những hàng đó là TẤT CẢ hoặc gần hết nội dung lúc mới cài
(cấu hình mặc định của app chỉ có Direct + Block) thì cả lưới trông như bị tắt và user báo
"grid bị disable toàn bộ". Liều đúng: hàng vẫn sống, chỉ (1) huỷ `BeginningEdit` trong code-behind
cho đúng hàng đó, (2) `DataGrid.CellStyle` + `DataTrigger` đổi `Foreground` sang màu phụ để báo
"không phải của bạn", (3) style riêng cho `ComboBox` của cột combo vì nó không đi qua edit mode.
Ghi nhớ: `DataGridCell` có setter `Foreground` riêng trong theme nên đặt `Foreground` trên
`DataGridRow` KHÔNG tới được chữ — phải đặt ở `CellStyle`.

## Test project xUnit v3 (Microsoft.Testing.Platform): `dotnet test` HỎNG, phải chạy assembly trực tiếp

- Triệu chứng: `dotnet test <Tests>.csproj` báo `Testhost process ... exited with error: An assembly specified in the application dependencies manifest (testhost.deps.json) was not found: package: 'Newtonsoft.Json', version: '13.0.3'` — trong khi gói ĐÓ CÓ ĐỦ trong `~/.nuget/packages` (kiểm `ls` thấy cả `lib/net6.0/Newtonsoft.Json.dll`). Xoá `bin`/`obj` và `dotnet restore` đều không cứu.
- Nguyên nhân: project dùng **xUnit v3** (`<PackageReference Include="xunit.v3">`) + `OutputType=Exe`, chạy bằng **Microsoft.Testing.Platform**, KHÔNG dùng `Microsoft.NET.Test.Sdk`/VSTest. `dotnet test` vẫn cố khởi động testhost của VSTest, và testhost đó có `deps.json` riêng đòi gói mà project không hề tham chiếu.
- Cách chạy đúng: `dotnet build <Tests>.csproj` rồi `dotnet exec <Tests>/bin/Debug/<tfm>/<Tests>.dll` (hoặc chạy thẳng .exe). Lọc theo trait: `-trait- "Category=Integration"`.
- Nhận biết trước khi chạy: mở `tests/Directory.Build.props` (hoặc .csproj) xem có `xunit.v3` + `OutputType=Exe` không — repo tử tế thường ghi luôn lý do trong comment ở đó. Đừng kết luận "môi trường hỏng, thiếu gói".
- Kiểm chứng nhanh rằng lỗi KHÔNG do thay đổi của mình: chạy `dotnet test` trên một test project khác, không liên quan, trong cùng repo — cùng lỗi ⇒ là cách chạy sai, không phải code sai.

## `Environment.TickCount64` không có trên netstandard2.0

- Chỉ có từ .NET Core 3.0/.NET Standard 2.1. Project multi-target `netstandard2.0;net8.0` sẽ chỉ lỗi ở nhánh netstandard2.0: `error CS0117: 'Environment' does not contain a definition for 'TickCount64'`.
- Thay bằng `System.Diagnostics.Stopwatch` (`Stopwatch.StartNew()` + `ElapsedMilliseconds`) — có ở mọi TFM, và đúng ngữ nghĩa hơn cho **timeout**: đồng hồ đơn điệu, không nhảy khi giờ hệ thống bị chỉnh. `Environment.TickCount` (32-bit) thì tràn sau ~24,9 ngày, đừng dùng thay thế.

## WPF: `WindowStartupLocation="CenterScreen"` không ăn khi app khai báo PerMonitorV2

- Triệu chứng: cửa sổ mở ra ở góc trên trái hoặc vị trí mặc định ngẫu nhiên của Windows, dù XAML đã có `WindowStartupLocation="CenterScreen"`. Hay gặp khi app.manifest có `<dpiAwareness>PerMonitorV2</dpiAwareness>` và cửa sổ dùng `WindowChrome` (custom title bar).
- Nguyên nhân: WPF tính vị trí CenterScreen **trước khi hwnd tồn tại**, nên phải quy đổi Width/Height (DIP) sang pixel bằng **DPI hệ thống** chứ không phải DPI thật của màn hình cửa sổ sẽ nằm lên. Lệch DPI ⇒ phép tính trượt ⇒ rơi về vị trí mặc định.
- Cách sửa chắc chắn: đặt `WindowStartupLocation="Manual"`, rồi override `OnSourceInitialized` (lúc này hwnd đã có, kích thước pixel thật đã biết) và đặt vị trí **bằng pixel thuần** qua Win32 — không đụng tới DIP thì không phải lo DPI:
  ```csharp
  IntPtr hwnd = new WindowInteropHelper(this).Handle;
  GetWindowRect(hwnd, out RECT bounds);
  IntPtr mon = MonitorFromWindow(hwnd, MONITOR_DEFAULTTOPRIMARY);
  var info = MONITORINFO.Create();          // phải gán cbSize trước, không GetMonitorInfo trả false
  GetMonitorInfoW(mon, ref info);
  var work = info.rcWork;                    // rcWork, KHÔNG rcMonitor — rcMonitor đẩy cửa sổ xuống dưới taskbar
  SetWindowPos(hwnd, IntPtr.Zero,
      work.Left + (work.Width - bounds.Width) / 2,
      work.Top + (work.Height - bounds.Height) / 2,
      0, 0, SWP_NOSIZE | SWP_NOZORDER | SWP_NOACTIVATE);
  ```
- Dùng `MonitorFromWindow` (màn hình cửa sổ đang nằm) thay vì ép về màn hình chính: di chuyển trong cùng một màn hình thì không vượt biên DPI, nên không sinh `WM_DPICHANGED` — mà WM_DPICHANGED sẽ resize cửa sổ và phá luôn phép căn giữa vừa làm.
- Nhớ `Math.Max` kẹp về góc trái trên: cửa sổ cao hơn work area mà căn giữa thì thanh tiêu đề chui lên trên mép màn hình, chuột không với tới để kéo.

## NU1105 sau khi thêm project mới: thủ phạm là `obj/` của project TIÊU THỤ, không phải file csproj mới

- Triệu chứng: thêm một project mới vào solution (`.slnx`/`.sln`) và tham chiếu nó từ project A, rồi `dotnet build <solution>` báo:
  `error NU1105: Unable to find project information for '...\ProjectMoi.csproj'` — nhưng lỗi lại gắn tên **project A** (và các project phụ thuộc A), không phải project mới.
- Đánh lạc hướng: `dotnet restore <project mới>` và `dotnet restore <solution>` đều in `All projects are up-to-date for restore` (kể cả `--force` vẫn có thể không chạm tới A), `dotnet sln list` liệt kê đủ project mới ⇒ dễ kết luận nhầm là csproj mới hỏng XML, sai BOM, hay thiếu `<Import>` targets. Kiểm tra những thứ đó chỉ tốn thời gian.
- Nguyên nhân: đồ thị phụ thuộc NuGet nằm trong `obj/project.nuget.cache` + `obj/*.nuget.dgspec.json` + `obj/project.assets.json` của **từng project tiêu thụ**. Bản cũ được sinh ra khi project mới chưa tồn tại nên không có mục cho nó; restore coi các file đó còn hợp lệ nên không sinh lại.
- Cách sửa (nhanh và chắc): xoá `obj/` của project tham chiếu trực tiếp tới project mới **và** của mọi project phụ thuộc chúng — chính là các project bị nêu tên trong thông báo NU1105 — rồi build lại:
  ```bash
  rm -rf path/to/A/obj path/to/B/obj path/to/ProjectGop/obj
  dotnet build Solution.slnx
  ```
- **Đọc thông báo lỗi theo hướng ngược**: NU1105 nêu tên project *đang thiếu thông tin về* project mới. Đó chính là danh sách `obj/` cần xoá — không cần đoán.

- `App đang chạy khoá bin thì build vào thư mục tạm, ĐỪNG giết tiến trình của user`: Đang debug/F5 mà build bằng CLI sẽ dính `MSB3021/MSB3027 Unable to copy file ... The file is locked by: "<App> (PID)"`. Thông báo có nêu đích danh tiến trình đang giữ file — đọc nó trước, thường là app của user hoặc `Microsoft Visual Studio`.
    + How: `dotnet build <proj>.csproj -p:BaseOutputPath=D:/tmp/x/` (hoặc `dotnet test ... -p:BaseOutputPath=...`, test runner tự tìm dll ở output mới). Toàn bộ output đi chỗ khác nên không đụng `bin/` đang bị khoá; xong thì `rm -rf` thư mục tạm.
    + KHÔNG kèm `-p:BaseIntermediateOutputPath=...`: cờ trên dòng lệnh áp cho MỌI project trong cây build, nên tất cả dùng chung một `obj/` và đâm nhau bằng `error CS0579: Duplicate 'TargetFrameworkAttribute'`. `obj/` không bị tiến trình đang chạy khoá (chỉ `bin/` bị), nên cứ để nguyên. Dùng chung một `bin/` tạm thì vô hại.
    + Nhắc: project test thường tham chiếu bắc cầu tới project host, nên chạy test cũng phải build host — app chạy là test cũng tịt, không riêng gì build.
    + Cách gọn hơn trên .NET 8+: `dotnet test <proj> --artifacts-path <dir>` — đưa CẢ bin lẫn obj của từng project vào cây riêng (mỗi project một thư mục con, nên không đâm nhau như `BaseIntermediateOutputPath`). BẪY: đặt `<dir>` NGOÀI repo thì mọi test tự dò gốc repo bằng cách đi ngược từ `AppContext.BaseDirectory` tìm `*.sln`/`*.slnx` sẽ hỏng ("Không tìm thấy gốc repo") — trông như test đỏ vì code vừa sửa. Đặt `<dir>` BÊN TRONG repo, dưới một thư mục đã gitignore (vd `<repo>/bin/claude-artifacts`, vì `**/bin/` luôn bị ignore), xong `rm -rf`. Đã dính 2026-09-11 ở k-rag-platform (`FieldLimitKeyTests`).

## Regex: timeout nằm trong khoá cache, và ngân sách bằng đồng hồ tường là sai

Gặp 2026-09-08, mất hai đợt mới truy ra vì nó chỉ hỏng khi máy bận.

**1. `Regex.IsMatch(input, pattern, options, matchTimeout)` static — `matchTimeout` là một phần của khoá cache.**
.NET cache biểu thức đã dựng theo `(pattern, culture, options, matchTimeout)`. Truyền timeout **đổi mỗi lần gọi** (kiểu "phần hạn còn lại") ⇒ trượt cache 100%, dựng lại pattern từ source mỗi lần. Trên đường nóng (mỗi tiến trình, mỗi lần quét) thì đây là chi phí lớn mà không ai thấy.
→ Luôn truyền timeout **hằng số**. Muốn giới hạn tổng thì đếm riêng, đừng nhét vào timeout.

**2. Đừng lấy `Environment.TickCount64 + N` làm ngân sách cho công việc CPU.**
Khoảng đó tính cả lúc luồng **không được lịch chạy**. Đo thật: 32 luồng quay ở `ThreadPriority.Highest` trên máy 32 core làm **47/100 ms** bốc hơi giữa lúc đóng dấu hạn và lúc bắt đầu regex — chưa chạy ký tự nào. Máy rảnh vẫn thấy 15 ms (đúng bằng độ phân giải `TickCount64`).
Hậu quả kiểu điển hình: hết hạn → nuốt lỗi → trả "không kết luận được" → **tính năng âm thầm không chạy, không log, không exception**.
→ Ngân sách phải đo thời gian **thực sự nằm trong công việc đó**: `long t = Stopwatch.GetTimestamp(); ... finally { spent += Stopwatch.GetElapsedTime(t); }` (`GetElapsedTime` có từ .NET 7).

**3. Dấu hiệu nhận ra loại bug này trong test:** test rớt **chỉ ở lượt chạy ngay sau build**, chạy riêng hoặc `--no-build` thì xanh mãi. Đừng vội gọi là "flaky test" — nhiều khi là bug thật của sản phẩm, chỉ lộ ra khi máy tải nặng. Cách dựng lại: viết probe tạo N luồng quay CPU rồi đo, **N ≤ số core** và có timeout ngắn, đừng để 128–256 luồng (nó tự bóp cổ, chạy quá 10 phút và ăn hết CPU của user).

### Bổ sung: cache regex tự viết — hai con số và một cái bẫy

Đo end-to-end (2026-09-08), một lượt quét 301 tiến trình qua `ProcessRuleMatcher`:

| Số pattern | `Regex.IsMatch` tĩnh | Giữ instance, thông dịch | Giữ instance, `Compiled` |
|---|---|---|---|
| 8 | 2,04 ms | 1,94 ms | 1,04 ms |
| 20 | 24,46 ms | 5,32 ms | 2,61 ms |
| 40 | 37,58 ms | 8,89 ms | 5,24 ms |

- Vực thẳm giữa 8 và 20 là `Regex.CacheSize` = **15**. Vượt là mọi lần gọi trượt cache và **parse lại pattern từ source**. Nhớ đếm số pattern *thực tế* (một luật có thể sinh 2 pattern), đừng đếm số luật.
- **Giữ instance là thứ xoá được vực thẳm** (24,46 → 5,32). `Compiled` giảm tiếp ~một nửa ở mọi cỡ, giá ~2 ms/pattern dựng một lần; và **dưới** mốc 15 thì nó là thứ duy nhất có tác dụng (8 pattern: giữ instance mà thông dịch KHÔNG nhanh hơn cache sẵn của framework). Muốn nhanh nhất thì làm cả hai.
- Nâng `Regex.CacheSize` là thiết lập **toàn tiến trình** — thư viện không nên tự ý đặt; và vẫn còn chi phí băm khoá `(pattern, options, culture, timeout)`.

**Bẫy suýt làm hỏng cả việc**: bản cache đầu tiên gọi `dict.Count >= Capacity` ở **mỗi lần tra**. `ConcurrentDictionary.Count` **khoá toàn bộ bucket** ⇒ bản "tối ưu" **chậm hơn** bản cũ (3,07 vs 2,59 ms). Đường trúng phải là `TryGetValue` trần; chỉ kiểm `Count` khi trượt. Bài học rộng hơn: **microbenchmark nói thắng, đo end-to-end nói thua** — luôn đo lại trên đường chạy thật bằng cách `git stash` bản sửa rồi chạy cùng một probe.

**Bẫy đo đạc, đắt hơn cả bẫy code**: lượt chạy đầu của một probe .NET cao gấp ~2× các lượt sau (4,48 vs 1,95 ms cho cùng một thứ) do JIT + cache đĩa nguội. Đo một lượt mỗi cấu hình rồi lập bảng là **sai**, và tôi đã ghi bảng sai đó vào code trước khi phát hiện. Luật: mỗi ô là **best-of-3 sau khi làm nóng**, cùng một probe, cùng một phiên. Và khi lấy mốc nền bằng `git stash`, nhớ stash chỉ đưa về **HEAD** — muốn về một commit cũ hơn phải `git show <sha>:<path> > <path>` rồi build.

- `Comment XML/MSBuild không được chứa hai dấu gạch ngang liền nhau`: Viết `--enable-gpl`, `--no-build`, hay bất kỳ cờ dòng lệnh nào vào trong `<!-- ... -->` của `.csproj`/`Directory.Packages.props`/`.props`/`.targets` là hỏng cả file: `error MSB4025: The project file could not be loaded. An XML comment cannot contain '--', and '-' cannot be the last character`.
    + Why: luật của chính XML, không phải của MSBuild. Nguy ở chỗ triệu chứng KHÔNG chỉ vào comment: `Directory.Packages.props` không nạp được nghĩa là Central Package Management coi như tắt, nên mọi project đồng loạt ăn `NU1015: The following PackageReference item(s) do not have a version specified` cho TOÀN BỘ gói, kèm cả `NU1701` vì NuGet đi giải phiên bản bừa. Nhìn lỗi thì tưởng vừa làm hỏng CPM hoặc thiếu `PackageVersion`, trong khi thủ phạm là một dòng chú thích.
    + How: viết `cờ enable-gpl` thay vì `--enable-gpl`, hoặc dùng dấu gạch dài `—`. Nghi file props/targets hỏng XML thì kiểm thẳng: `dotnet msbuild <file> -t:Help -v:q -nologo` in ra đúng dòng và cột.

## `SafeHandle` che lời gọi ĐANG bay, không che lời gọi KẾ TIẾP

Triệu chứng: `System.ObjectDisposedException: Cannot access a disposed object` ném từ `SafeHandle.DangerousAddRef`, trên luồng thread pool, ở giữa một vòng lặp đọc.

Lập luận sai rất dễ viết ra (và tôi đã gặp nó viết sẵn thành comment trong code):

> "chờ 1 giây rồi cứ `Dispose()` vẫn an toàn, vì mọi lời gọi native đều giữ tham chiếu trên handle"

Vế đầu **đúng**: marshaller `DangerousAddRef` trước khi gọi, nên `ReleaseHandle` bị hoãn tới khi lời gọi đang chạy trả về. Vế sau **thiếu**: vòng lặp quay lại gọi lần nữa, và `DangerousAddRef` trên handle đã dispose thì **ném ngay**. Đóng handle dưới chân một vòng lặp đọc luôn sai, kể cả khi có `SafeHandle`.

- **Bất biến cần giữ**: *chỉ luồng đang đọc handle mới được đóng nó*. Vòng lặp bọc `try { ... } finally { handle.Dispose(); }`; bên ngoài chỉ được `Shutdown()`/`Cancel()` rồi `Wait(timeout)`. `Wait` hết giờ thì handle đóng muộn hơn một chút — rẻ hơn nhiều so với một cú ném.
- **Ngoại lệ duy nhất**: chưa `Start()` thì không có vòng lặp để giao, `Dispose` phải tự đóng. Cờ `_started` phải set **trước** `Task.Run`, vì `_task = Task.Run(...)` gán *sau* khi task đã chạy — khe hở đó đủ để `Dispose` kết luận nhầm là không ai sở hữu.
- **Vì sao lỗi này sống lâu**: exception rơi trên task không ai `Wait`, thành *unobserved* — .NET Core không giết tiến trình, không log gì. Chỉ thấy khi bật debugger. Nơi nguy nhất là chỗ khởi tạo thua đua rồi vứt tài nguyên đi (`if (!dict.TryAdd(...)) { resource.Dispose(); return; }`) trong khi worker cho tài nguyên đó đã `Task.Run` mấy dòng trước.

## Test cho lỗi tranh chấp phải chờ đúng HÀNH VI SAI, không phải điều kiện dẫn tới nó

Viết test cho lỗi trên, hai test đầu **xanh cả với code hỏng**. Lý do: chúng đợi `handle` bị đóng rồi mới assert — mà "đóng" chính là việc code hỏng làm **sớm** (mốc 1 giây), còn lời gọi vi phạm mãi giây thứ 2 mới xảy ra. Test kết luận xong trước khi lỗi kịp xảy ra.

→ Mốc chờ phải là **chính lời gọi sai** (đếm nó trong đối tượng giả rồi `SpinWait.SpinUntil(() => calls > 0, timeout)`), không phải trạng thái dẫn tới nó.
→ Đối tượng giả phải **đếm** vi phạm chứ không chỉ ném: exception trên luồng worker không ai chờ sẽ biến mất, đúng như cách bug thật lọt lưới.
→ Và luôn kiểm ngược bằng cách `git show <sha>:<path> > <path>` để dựng lại code cũ rồi chạy — "test xanh" không chứng minh được gì nếu chưa thấy nó đỏ.

## `dotnet build <sln>` và `dotnet test <csproj>` có thể chạy HAI thư mục output khác nhau

Project nào có `<RuntimeIdentifier>` hoặc `<Platform>x64</Platform>` (vd project WPF cần native
win-x64) thì solution build ra `bin/x64/Debug/<tfm>/`, còn `dotnet test <csproj>` không truyền
platform nên nạp `bin/Debug/<tfm>/`. Hậu quả: sửa code → `dotnet build <App>.sln` (0 lỗi) →
`dotnet test <csproj> --no-build` → **chạy DLL cũ**. Triệu chứng đúng chỗ dễ bỏ qua: thêm 6 test mới
mà tổng số test KHÔNG đổi, và `--filter` tên class mới báo "No test matches".

→ Trước khi `dotnet test --no-build`, build **đúng project test** (`dotnet build <test>.csproj`), hoặc
bỏ `--no-build`. Nghi ngờ thì `ls -la <bin>/…/<Tests>.dll` ở CẢ hai đường dẫn và so mốc thời gian.

## Build .NET trên Windows hỏng vì app đang chạy giữ file DLL

`error MSB3027/MSB3021: Could not copy … The file is locked by: "<App>.exe (60696)"` — app vừa
chạy thử vẫn đang giữ output. Thông báo có **kèm tên và pid** tiến trình đang khoá, đọc là biết ngay.

→ Đừng tự kill: đó là app user đang dùng (ở đây còn là app chạy admin đang giữ driver mạng). Báo user
tắt rồi build lại; trong lúc chờ thì làm tiếp phần không cần build (viết test, sửa doc).

## `new HttpClient()` cắt request ở 100 giây, và cú cắt đó ngụy trang thành "người dùng huỷ"

`HttpClient.Timeout` mặc định **100 giây**, áp cho cả những lời gọi vốn dĩ phải lâu (bóc lời một mảnh
audio 10 phút, sinh ảnh, model reasoning). Tệ hơn con số đó là **kiểu exception**: hết `Timeout` thì
`HttpClient` ném `TaskCanceledException` — lớp con của `OperationCanceledException`, đúng thứ mà mọi
tầng UI đang bắt để hiện "đã huỷ". Kết quả: chức năng hỏng mà giao diện chỉ ghi "Đã huỷ", không lỗi,
không log, không stack trace ở đâu cả.

Bẫy đi kèm: code thường đã có timeout riêng qua `CancellationTokenSource` (vd 10 phút) và bắt bằng
`catch (OperationCanceledException) when (!ct.IsCancellationRequested && timeoutCts.IsCancellationRequested)`.
Bộ lọc đó **không bắt được** cú cắt của `HttpClient`: mốc 100 giây đến trước, `timeoutCts` chưa kích
hoạt, exception lọt thẳng ra ngoài.

→ Client của tầng gọi API luôn dựng qua MỘT factory đặt `Timeout = Timeout.InfiniteTimeSpan`, để thời
gian chờ chỉ do `CancellationTokenSource` quản (nó huỷ được, ghép được với token của người dùng, và
báo được số giây thật trong thông điệp).
→ Bộ lọc catch chỉ nên hỏi **"có phải người dùng huỷ không"** (`when (cts.IsCancellationRequested)` —
xét đúng CTS của lượt chạy đó), mọi lần huỷ còn lại phải rơi xuống nhánh báo lỗi. Đừng viết bộ lọc
theo hướng "liệt kê các nguồn timeout đã biết" — nguồn thứ ba luôn xuất hiện.
→ Test khoá được: handler giả trễ 30 giây + `HttpClient { Timeout = 200ms }` ⇒ khẳng định nhận được
exception nghiệp vụ chứ không phải `OperationCanceledException`.

## `FrameworkReference` mới làm nảy NU1510, nhưng gỡ ghim CVE theo lời khuyên đó thì CVE quay lại

Thêm `<FrameworkReference Include="Microsoft.AspNetCore.App" />` vào một thư viện thường có thể làm
NuGet cảnh báo NU1510 cho một `PackageReference` đang có sẵn: *"will not be pruned. Consider removing
this package"*. Lời khuyên đó **không phải lúc nào cũng đúng**: nếu package đó đang được ghim để đè
một phiên bản dính CVE mà một package khác kéo vào (NU1903), gỡ nó ra là cảnh báo CVE hiện lại ở mọi
project tiêu thụ — framework chỉ phủ được assembly của chính nó, không phủ nhánh phụ thuộc của
package bên thứ ba.

**Cách kiểm trước khi gỡ**: `dotnet restore <consumer.csproj> --force` rồi grep `NU1903`. Build
thường KHÔNG hiện lại cảnh báo restore đã cache, nên `dotnet build` sạch không chứng minh được gì.

Giữ ghim và tắt đúng một cảnh báo tại chỗ, kèm lý do:

```xml
<!-- NoWarn vì từ khi có FrameworkReference, NuGet coi ghim này là thừa (NU1510) — nó nhìn nhầm. -->
<PackageReference Include="System.Security.Cryptography.Xml" NoWarn="NU1510" />
```

## System.Text.Json: `[JsonIgnore]` trên property abstract/virtual ở lớp gốc KHÔNG chặn được override

- Triệu chứng (2026-09-11, .NET 8): `LeafCondition` khai `[JsonIgnore] public abstract ConditionSubject Subject { get; }`, lớp con `override` không gắn attribute ⇒ file config bị ghi thêm `"Subject": { ... }` (cả một object) trong mọi phần tử. Không lỗi build, không lỗi runtime — chỉ lộ ra nhờ test đọc lại chuỗi JSON.
- Lý do: STJ lấy property ở kiểu dẫn xuất (khai báo override) và đọc attribute trên chính khai báo đó; attribute đặt ở khai báo abstract của lớp gốc không tới được.
- Cách sửa bền: property public để **không virtual** ở lớp gốc (gắn `[JsonIgnore]` ở đó), phần thay đổi theo lớp con để sau member **không public** (`private protected abstract object BoxedMatcher { get; }`, hoặc nhận qua constructor). STJ không bao giờ đọc member non-public nên không phụ thuộc chuyện attribute kế thừa hay không. Cách tạm: gắn `[JsonIgnore]` lên từng override — dễ quên ở lớp con mới.
- Nhớ thêm: STJ mặc định **ghi cả property chỉ có getter** (`IgnoreReadOnlyProperties = false`), nên property tính toán thêm vào model serialize phải `[JsonIgnore]` — và nếu property tên trùng discriminator (`"kind"`) thì STJ **ném** lúc serialize.
- Cách phát hiện: test round-trip phải `Assert.DoesNotContain("\"tênProperty\"", json, OrdinalIgnoreCase)` trên chuỗi đã ghi, không chỉ so object đọc lại (đọc lại vẫn đúng vì property không có setter).

## Raw interpolated string `$"""` KHÔNG cho escape `{{` — muốn ngoặc nhọn trần thì dùng `$$"""`
- Triệu chứng (2026-09-11, script SQL chèn JSON): `var sql = $""" ... '[{{"call_id":"c1"}}]' ... {expr} """;` báo `CS1733: Expected expression` ở đúng dòng có `{{`. Quen tay từ chuỗi `$"..."` thường, nơi `{{` là escape.
- Luật: với raw string, số dấu `$` = số ngoặc mở một lỗ nội suy. `$"""` ⇒ `{` luôn là nội suy, không có cách viết ngoặc trần. `$$"""` ⇒ `{` / `}` là chữ, nội suy viết `{{expr}}`.
- Cách làm: chuỗi chứa JSON/CSS/ngoặc nhọn thì mở bằng `$$"""` ngay từ đầu.

## SSH.NET `ForwardedPortLocal`: mỗi kết nối chiếm chết MỘT luồng thread pool suốt đời nó
- Triệu chứng (2026-09-11, SSH.NET 2025.1.0/2026.0.0): mở vài chục tunnel direct-tcpip cùng lúc qua một `SshClient` ⇒ `TcpClient.ConnectAsync` tới chính cổng loopback của forwarder **timeout**, cả tiến trình khựng. 40 tunnel với `MinThreads` worker = 32 là hỏng; nâng min lên thì xong trong 200 ms.
- Nguyên nhân: `ForwardedPortLocal.AcceptCompleted` gọi `ProcessAccept` NGAY trên luồng pool hoàn tất accept, và trong đó `channel.Bind()` là vòng `Socket.Receive` **đồng bộ** chạy tới khi tunnel đóng. Pool chỉ thêm ~1 luồng/0,5 s ⇒ starvation. `ChannelDirectTcpip` (thứ cho mở channel không cần cổng loopback) là **internal** — không né được bằng API công khai.
- Cách xử lý: nâng `ThreadPool.SetMinThreads(worker + 1, io)` cho mỗi tunnel đang sống, trả lại khi tunnel đóng, không hạ dưới giá trị lúc tunnel đầu tiên tới (tôn trọng giá trị code khác đặt). Chỉ cần worker, không cần IOCP.
- Kèm theo: `RequestReceived` của forwarded port bắn TRƯỚC khi mở channel; ném exception trong handler ⇒ SSH.NET `CloseClientSocket` ⇒ dùng được để chỉ cho đúng socket của mình (bind socket trước để biết cổng nguồn) và chặn tiến trình khác nối ké cổng loopback.
- Kiểm thật không cần admin/Docker/WSL: `C:\Windows\System32\OpenSSH\sshd.exe -D -e -f <cfg>` chạy được ở quyền user (ListenAddress 127.0.0.1, Port lạ, HostKey/AuthorizedKeysFile đường dẫn tuyệt đối, `StrictModes no`; chỉ pubkey; lỗi "Couldn't create pid file" vô hại). Bẫy: sshd Windows phân giải `localhost` ra `::1` và KHÔNG lùi về IPv4; getaddrinfo Windows nhận `"[::1]"` nên không bắt được lỗi quên bỏ ngoặc IPv6 (glibc thì từ chối). Tắt bằng PID sau khi kiểm command line, không kill theo tên.

## `IHttpClientFactory` POOL primary handler ⇒ đặt `CookieContainer` vào đó là rò/tráo phiên giữa các người dùng
- Triệu chứng (k-rag-platform, 2026-09-14, .NET 10): web host gọi API nội bộ bằng typed client. Hai người đăng nhập ở **hai profile Chrome khác nhau** thì **tráo hẳn danh tính cho nhau**, ai đăng nhập sau thì thắng; F5 không cứu được. Rất dễ đổ oan cho phân quyền vì biểu hiện đầu tiên thường chỉ là một lượt `403`.
- Nguyên nhân: `.ConfigurePrimaryHttpMessageHandler(() => new SocketsHttpHandler { UseCookies = true, CookieContainer = new CookieContainer() })`. Lambda đó **không chạy mỗi request** — `IHttpClientFactory` gộp và tái dùng primary handler theo tên client (mặc định 2 phút), nên `CookieContainer` là **dùng chung toàn tiến trình**. `Set-Cookie` của người đăng nhập sau ghi đè cookie của người trước (cùng tên, cùng domain/path).
- Phân biệt cái KHÔNG rò: typed client thường là `Transient` và `CreateClient()` trả **`HttpClient` mới mỗi lần**, nên `DefaultRequestHeaders` (bearer token gắn qua `SetBearerToken`) là **riêng từng instance, an toàn**. Chỉ **handler** bị pool ⇒ **chỉ state nằm trong handler** (cookie, `CookieContainer`, `ClientCertificates`, `Credentials`) mới rò. Đây là lý do lỗi sống lâu: bearer đúng nên mọi thứ trông hợp lý.
- Thứ biến rò thành **tráo vĩnh viễn**: server ưu tiên cookie hơn header. `JwtBearerEvents.OnMessageReceived` thấy cookie `access_token` là ghi đè `context.Token`; endpoint `refresh` đọc cookie `refresh_token` trước thân request. Lượt refresh của A gửi thân = token A nhưng cookie = token B ⇒ server làm mới phiên **B** rồi trả token B, host ghi token B vào vé đăng nhập của A. Từ đó A **thật sự là** B, không phải hiển thị sai.
- Cách sửa: **đừng** dùng cookie ở tầng handler cho client dùng chung — truyền token qua **thân request**. Lưu ý `AddHttpMessageHandler` (DelegatingHandler) **cũng bị pool cùng chuỗi**, nên nhét `CookieHandler` vào đó KHÔNG tách được theo người dùng. Muốn cookie riêng phiên thì tự gắn header `Cookie` cho từng request, hoặc tự dựng `HttpClient` ngoài factory.
- Bẫy phụ, nhớ khi rà: `ConfigureHttpClientDefaults(... UseCookies = false ...)` **chỉ là mặc định** — một dòng `ConfigurePrimaryHttpMessageHandler` ở client cụ thể ghi đè nó, **im lặng, không cảnh báo**. Đừng coi default đó là rào chắn; phải grep `UseCookies|CookieContainer|ConfigurePrimaryHttpMessageHandler` trên toàn repo.
- Kiểm chứng: 2 tài khoản khác vai, 2 profile trình duyệt, **ép qua mốc refresh** (access token ngắn hạn) rồi F5 cả hai. Chỉ đăng nhập rồi xem ngay thì có thể chưa lộ.
- Dọn hậu quả: sau khi sửa phải **thu hồi toàn bộ phiên**, vì các vé đăng nhập đã bị ghi token của người khác vẫn còn hạn (7–30 ngày).

## `HttpClientHandler` mặc định `UseCookies = true` ⇒ handler `static` là hộp cookie chung cho cả tiến trình
- Phát hiện khi soi thư viện HTTP nội bộ (2026-09-14): `NetSingleton.HttpClientHandler` là **một instance static** `WrapperHttpClientHandler : HttpClientHandler`, lớp con **không** đặt `UseCookies = false`. `BaseApi()` và `BaseApi(string apiKey)` đều `new HttpClient(NetSingleton.HttpClientHandler, false)` ⇒ mọi class kế thừa qua 2 ctor đó **dùng chung một `CookieContainer`** bất kể bao nhiêu instance, bao nhiêu người dùng. Ctor nhận `HttpClient` từ ngoài thì không chạm vào singleton nên an toàn.
- Luật chung: `HttpClientHandler.UseCookies` và `AllowAutoRedirect` **mặc định `true`**. Bất kỳ handler nào có tuổi thọ dài hơn một người dùng (static, singleton DI, pool của `IHttpClientFactory`) đều phải đặt `UseCookies = false` một cách tường minh.
- Đi kèm: `WrapperHttpClientHandler.Dispose` bị vô hiệu hoá (thân rỗng) để bảo vệ singleton ⇒ truyền nó vào ctor với `disposeHandler: true` là câu lệnh **không có tác dụng**, không báo lỗi.
- Bài học rà code: chú thích kiểu "handler này an toàn vì X lo cookie" phải **grep xác minh X có tồn tại**. Ở ca này chú thích trỏ tới `BaseSiteImplement` — grep toàn bộ thư viện thấy **type đó không tồn tại**, chỉ có đúng dòng comment. Chú thích sai đã khiến người sau tin là bật lại cookie thì vô hại.

## Bump package ở MỘT project của repo nhiều project ⇒ NU1605 "downgrade" ở mọi project phụ thuộc
- Triệu chứng (thư viện HTTP nội bộ, 2026-09-14): VS NuGet UI "Update" một package trong `<Lib>.csproj` (nó ghi thành `<PackageReference Update="..." Version="..."/>` ở một ItemGroup mới cuối file). Project đó build xanh, nhưng `dotnet build <sln>` **đỏ 8 project** với `error NU1605: Warning As Error: Detected package downgrade: X from 1.0.0.3 to 1.0.0.2`.
- Nguyên nhân: version dùng chung khai ở file `.targets`/`.props` được từng csproj import (vd `ProjectBuildProperties.targets`). Nâng riêng project nền thì project phụ thuộc vừa tham chiếu **trực tiếp** bản cũ (qua targets) vừa nhận bản mới **truyền tiếp** qua project nền ⇒ NuGet coi là downgrade. Repo bật `TreatWarningsAsErrors` thì thành lỗi.
- Bẫy chẩn đoán: **build project lẻ KHÔNG lộ ra**, chỉ build cả solution mới thấy. Nên sau khi sửa csproj luôn build sln, đừng dừng ở project vừa sửa.
- Sửa nhanh: nâng ở chính file `.targets`/`.props` dùng chung, rồi bỏ dòng `PackageReference Update` trong csproj lẻ.
- Sửa gốc: chuyển sang **Central Package Management** — `Directory.Packages.props` ở gốc repo với `<ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>` và một `<PackageVersion Include="..." Version="..."/>` cho mỗi package; csproj chỉ còn `<PackageReference Include="..."/>` **không kèm Version** (kèm là `NU1008`).
- Hai bẫy của CPM khi migrate:
  + **Chỉ được MỘT `PackageVersion` cho mỗi package id.** Project đa target framework cần bản khác nhau theo TFM (vd `Microsoft.Net.Http.Headers` 2.x cho netstandard2.0, 8.x cho net8.0) thì để bản chính ở `Directory.Packages.props`, còn nhánh kia dùng `<PackageReference Include="..." VersionOverride="2.3.13"/>` trong csproj. Đừng đặt `PackageVersion` có `Condition="$(TargetFramework)..."` — file này import rất sớm, ở outer build `TargetFramework` còn rỗng.
  + Cùng một package đang ở **hai version tại hai project** thì buộc phải hợp nhất (hoặc `VersionOverride`). Đây là thay đổi version thật cho một project mình không được nhờ sửa ⇒ phải build lại và **đối chiếu số cảnh báo theo từng mã** trước/sau (`dotnet build -t:Rebuild | grep -oE "warning [A-Z]+[0-9]+" | sort | uniq -c`), đừng chỉ xem "0 Error(s)".
- Lưu ý đo lường: so cảnh báo phải dùng `-t:Rebuild` cả hai lần. Build tăng dần (incremental) chỉ báo cảnh báo của project được biên dịch lại nên số liệu lệch hẳn (ca này 1333 vs 2012) và tưởng là hồi quy.
- `PackageReference Update` (khác `Include`) là dấu hiệu VS tự ghi để ghim version cho package đến từ file props/targets import sẵn. Thấy nó xuất hiện trong diff thì hiểu ngay: có người "Update" package bằng UI, và version đang bị ghim lệch ở một project.

## SkiaSharp: `SKBitmap.Decode(byte[])` NÉM chứ không trả null khi dữ liệu không phải ảnh

- Gặp 2026-09-19 (SkiaSharp 3.119): `SKBitmap.Decode(new byte[]{1,2,3,4})` ném `ArgumentNullException (Parameter 'codec')` — bên trong nó `SKCodec.Create` ra null rồi truyền tiếp. Code viết `SKBitmap.Decode(bytes) ?? throw new MyException(...)` tưởng đã bắt ca "không phải ảnh" nhưng nhánh `??` KHÔNG BAO GIỜ chạy; lỗi lọt ra là `ArgumentNullException` trần, tầng trên dịch thành 500.
- **How to apply:** tự dựng codec trước: `using var stream = new SKMemoryStream(bytes); using var codec = SKCodec.Create(stream) ?? throw ...; using var bitmap = SKBitmap.Decode(codec) ?? throw ...;`. Viết kèm một test đưa mảng rác vào — chỉ test mới lộ ra, đọc code thì thấy `??` hợp lý.
- Kèm theo: muốn tính toán từng điểm ảnh có alpha thì đọc ra `SKColorType.Rgba8888` + `SKAlphaType.Unpremul` qua `bitmap.PeekPixels().ReadPixels(info, ptr, rowBytes, 0, 0)` rồi tự làm trên mảng byte; Skia chỉ VẼ lên bề mặt premultiplied, làm trên đó thì màu của điểm gần trong suốt bị làm tròn mất (mép chủ thể ám tối sau khi tách nền). Ghi ngược ra ảnh bằng `SKImage.FromPixelCopy(info, ptr, rowBytes).Encode(...)`.

## Newtonsoft: `[JsonConverter]` trên lớp CƠ SỞ của cây type → converter tự gọi lại chính nó, TRÀN STACK

- Gặp 2026-09-20 (Vsid): `abstract record ControlMessage` gắn `[JsonConverter(typeof(ControlMessageConverter))]`,
  converter đọc trường `type` rồi `json.ToObject(typeof(AgentHelloMessage), serializer)`. `JsonConverterAttribute`
  là attribute **kế thừa**, nên contract của record dẫn xuất cũng mang converter đó ⇒ `ToObject` gọi lại
  `ReadJson` vô hạn. `CanConvert` KHÔNG cứu được: attribute áp converter trực tiếp, không hỏi `CanConvert`.
- Triệu chứng dễ đọc sai: `dotnet test` in `Passed! - Failed: 0, Passed: 41` **rồi** `Test Run Aborted` —
  StackOverflowException giết luôn test host nên không có test nào bị đánh đỏ, nhìn như lỗi môi trường.
- **How to apply**: đừng gắn attribute lên base type. Đăng ký converter trong `JsonSerializerSettings`
  (`settings.Converters.Add(...)`) và để `CanConvert` chỉ nhận đúng base type — lúc đọc ra kiểu cụ thể,
  `CanConvert` trả false nên vòng lặp dừng. Gói settings đó thành một `static class` riêng cho cây type ấy
  để mọi bên dùng chung một cách đọc.

## Newtonsoft `CamelCasePropertyNamesContractResolver` đổi CẢ KHOÁ DICTIONARY, không chỉ tên property

- Gặp 2026-09-20 (Vsid): settings JSON chung dùng `new CamelCasePropertyNamesContractResolver()`. Ghi
  `Dictionary<string,string>` xuống rồi đọc lại thì khoá `"App.dll"` thành `"app.dll"`, `"ASPNETCORE_ENVIRONMENT"`
  thành `"aspnetcoreENVIRONMENT"` — biến môi trường truyền cho app, đường dẫn file trong cache hash và
  `details` của lỗi đều là **dữ liệu**, đổi chúng là hỏng dữ liệu chứ không phải "đổi quy ước đặt tên".
- Triệu chứng đánh lừa: ghi rồi đọc trong cùng tiến trình vẫn "chạy" ở nhiều chỗ (cả hai chiều cùng sai một
  kiểu) — chỉ vỡ khi so khoá với dữ liệu đến từ nơi khác (file trên đĩa, request của bên kia).
- **How to apply**: luôn tắt xử lý khoá dictionary ở resolver dùng chung:
  ```csharp
  settings.ContractResolver = new CamelCasePropertyNamesContractResolver
  {
      NamingStrategy = new CamelCaseNamingStrategy { ProcessDictionaryKeys = false, OverrideSpecifiedNames = true },
  };
  ```
  Kèm một test ghi/đọc `Dictionary<string,string>` với khoá UPPER_SNAKE và khoá có dấu chấm — đó là thứ
  duy nhất bắt được lỗi này sớm. (System.Text.Json có bẫy tương tự với `DictionaryKeyPolicy`.)

## WPF: thoát ứng dụng từ bên trong `App.OnStartup` (vd chặn instance thứ 2)

Cả hai cách quen thuộc đều KHÔNG kết thúc tiến trình khi gọi trong `OnStartup` (đã kiểm chứng
trên .NET Framework 4.6.2 / WPF):

- `Application.Shutdown()` — process vẫn sống, không còn cửa sổ nào. Thành **zombie nền**.
- `Environment.Exit(0)` — **treo**. Nó chạy handler `AppDomain.ProcessExit`; WPF đăng ký handler
  tắt Dispatcher, mà Dispatcher chính là luồng đang đứng trong `OnStartup` → tự khoá nhau.

Cách chạy được: `System.Diagnostics.Process.GetCurrentProcess().Kill()` (TerminateProcess).
Chấp nhận được vì ở thời điểm đó app chưa mở tài nguyên gì. Muốn sạch hơn thì viết `Main` riêng
(`<StartupObject>`, đổi `App.xaml` sang build action `Page`) và kiểm tra TRƯỚC khi `new App()`.

Bẫy đi kèm — **exception trong `OnStartup` bị nuốt**: nếu `App.xaml` có
`DispatcherUnhandledException` đặt `e.Handled = true` thì mọi exception ném từ `OnStartup` biến
mất không dấu vết, app chạy tiếp mà không dựng nổi cửa sổ (lại là zombie). Triệu chứng giống hệt
hai lỗi trên nên rất dễ chẩn đoán nhầm. Lần ra bằng cách ghi marker ra file temp sau từng dòng.
Ví dụ dính bẫy: gán `StartupUri = null` trong `OnStartup` để chặn tạo MainWindow.

- `Chuỗi rỗng trong cột DB nullable là một giá trị thứ ba không ai khai báo`: Cột chuỗi cho phép NULL mà code lại ghi `""` vào thì bảng có HAI cách diễn đạt "không có". Mọi nơi đọc phải nhớ kiểm cả hai, và chỉ cần một chỗ quên là sinh ra hành vi lệch **mà không có gì đỏ lên** — khác hẳn `NullReferenceException`, cái này chạy êm và trả về số sai.
    + Why: hai câu truy vấn viết cách nhau vài tháng gần như chắc chắn lọc khác nhau. Gặp thật 22/9/2026 ở bảng `BillingLogs` (sổ tiền của tenant node): `GetCostByUserAsync` — đường trang hạn mức đọc để CHẶN người dùng — lọc `UserId != null`, còn `GetTotalsAsync` chiều người dùng lọc `UserId != null && UserId != ""`. Một hàng mang `""` vì thế **bị cộng vào số đem đi chặn nhưng không hiện ra ở bảng nào**: người dùng ăn 403 ở một con số không tồn tại trên màn hình. Bảng lại có trigger cấm UPDATE nên hàng ấy sai vĩnh viễn.
    + Nguồn của `""` gần như luôn là `= ""` khai làm giá trị mặc định cho qua bộ phân tích nullable, chứ không ai cố ý ghi. Trong ca trên là `public string UserId { get; init; } = "";` trên một lớp tham số job. Nó còn giết luôn một nhánh code: `parameters.UserId ?? meeting.UserId` **không bao giờ chạy nhánh phải**, vì giá trị là `""` chứ không phải `null` — chú thích ngay trên đó mô tả đúng ý định, code thì làm ngược, và không test nào thấy.
    + How: (1) mỗi cột chuỗi chọn MỘT lập trường — `NOT NULL` và luôn có ≥1 ký tự thật, hoặc nullable và dùng `NULL` để nói "không có"; (2) trong class dùng `string?` hoặc `required string`, **đừng dùng `= ""`/`= string.Empty`** để làm im cảnh báo nullable (`required` bắt trình biên dịch chỉ ra mọi nơi dựng còn thiếu — đó là tính năng, không phải phiền toái); (3) phép quy `""` → `null` đặt ở **ĐIỂM THẮT** của đường ghi (writer/repository), KHÔNG chép tay ở từng nơi gọi — ca trên có 3 bản `NullIfBlank` giống hệt nhau ở 3 nơi gọi và nơi thứ tư quên mất, đúng kiểu lỗi mà việc tập trung sinh ra để chặn.
    + Cách quét database đang có (PostgreSQL) — duyệt mọi cột text rồi đếm, thay vì đoán:
      ```sql
      DO $scan$ DECLARE r record; c bigint; BEGIN
        FOR r IN SELECT table_name, column_name FROM information_schema.columns
                 WHERE table_schema='public' AND data_type IN ('character varying','text','character')
        LOOP
          EXECUTE format('SELECT count(*) FROM public.%I WHERE %I = %L', r.table_name, r.column_name, '') INTO c;
          IF c > 0 THEN RAISE NOTICE '% . % = %', r.table_name, r.column_name, c; END IF;
        END LOOP;
      END $scan$;
      ```
      Kết quả thường rất ít cột — việc dọn nhỏ hơn nhiều so với cảm giác ban đầu, nên đừng ngại quét.
    + Ngoại lệ hợp lệ, phải nói rõ: cột `NOT NULL` mà `""` MANG NGHĨA. Ví dụ `chat_messages.content` — một bong bóng chỉ gọi công cụ thì thật sự không nói câu nào; đổi sang NULL chỉ bắt mọi nơi hiển thị thêm một nhánh mà không đổi được ý nghĩa gì.

- `_ => null` trong switch expression trả struct "enum mở rộng" KHÔNG trả null — nó gọi `op_Implicit(string)` với null rồi ném: Các SDK hiện đại (OpenAI, Azure, System.ClientModel…) khai enum dạng `readonly struct` có `implicit operator T(string)` và ctor `AssertNotNull`. Khi mọi nhánh khác của switch trả `T`, trình biên dịch lấy **natural type = `T`** (chứ không lấy kiểu trả `T?` của hàm làm đích), rồi thấy `null` literal chuyển được sang `T` qua phép chuyển ngầm từ `string` ⇒ sinh ra `op_Implicit((string)null)` → `ArgumentNullException: Value cannot be null. (Parameter 'value')` **ngay tại dòng `_ => null`**. Không có cảnh báo nào lúc build.
    + Why: đọc code thì dòng đó hiển nhiên là "không có giá trị", mà stack trace lại chỉ vào một `op_Implicit` không ai viết ⇒ rất dễ đổ cho nhà cung cấp. Gặp thật 22/9/2026: `OpenAiImageGenerationProvider.ToBackground` trả `GeneratedImageBackground?`, ba nhánh đầu trả `GeneratedImageBackground.Transparent/Opaque/Auto`, nhánh `_ => null` nổ với MỌI lời gọi không truyền nền — tức toàn bộ đường vẽ ảnh mới; lỗi ném **trước khi có một byte nào đi tới OpenAI**.
    + How: ép kiểu ngay tại nhánh — `_ => (T?)null` — hoặc ép ở nhánh ĐẦU (`ImageBackgrounds.Transparent => (T?)GeneratedImageBackground.Transparent`), hoặc viết `if/else` với `return null;` (return lấy kiểu trả của hàm làm đích nên an toàn). Cùng lý do, `return cond ? new T(...) : null;` chỉ an toàn khi `T` KHÔNG có phép chuyển ngầm từ `string`.
    + Cách kiểm một kiểu có dính bẫy không, không cần đọc mã nguồn SDK:
      ```powershell
      $a=[Reflection.Assembly]::LoadFrom("$env:USERPROFILE\.nuget\packages\<pkg>\<ver>\lib\netstandard2.0\<Asm>.dll")
      $a.GetType("<Ns>.<Type>").GetMethods() | ? { $_.Name -eq 'op_Implicit' }
      ```
      Có dòng `T op_Implicit(System.String)` là dính. (Bản `netstandard2.0` nạp được bằng Windows PowerShell 5.1; bản net8/net10 thì không.)

## Tách/sinh lại controller bằng script: hàm dựng mới phải giữ KIỂU THAM SỐ gốc, không suy từ kiểu field

- Script tách controller dựng hàm dựng mới từ danh sách field (`private readonly T _x;` → tham số `T x`). Field `AgentLoopOptions _opts` thật ra được gán từ tham số `IOptions<AgentLoopOptions>` (`_opts = opts.Value;`), nên bản sinh ra đòi DI một `AgentLoopOptions` trần mà container không có (`Configure<T>` chỉ đăng ký `IOptions<T>`) ⇒ controller không dựng được, 500 mọi request.
  + Why: build sạch, mọi unit test vẫn qua (không test nào dựng controller qua DI) — chỉ lộ lúc chạy thật. Gặp 23/9/2026 khi tách `NodeStatusController`; review bắt được, không phải build.
  + How: lấy kiểu tham số từ CHÍNH chữ ký hàm dựng gốc, không từ kiểu field; sau khi tách chạy phép kiểm tĩnh "mọi kiểu tham số hàm dựng mới đều có trong hàm dựng gốc (trừ `ILogger<chính nó>`)", và grep phép gán trong hàm dựng gốc không theo dạng `_x = x;` (`.Value`, `.CreateClient()`, `?? throw`…) để xử lý tay.

## xUnit: test gửi việc nền rồi kết thúc ngay → Dispose xoá tệp đang bị việc nền giữ (chập chờn)
- Triệu chứng: `IOException: ... being used by another process` ném từ `Dispose()` của lớp test, chỉ vài lần trên chục lần chạy.
- Nguyên nhân: test gửi việc vào hàng đợi/worker nền (việc đó mở tệp tạm) rồi khẳng định xong là thoát; dispose hàng đợi thường chỉ đợi vòng điều phối chứ không đợi việc đang chạy, nên `File.Delete` của test đụng tệp còn mở.
- Cách làm: test nào đẩy việc thật vào hàng đợi thì ĐỢI việc đó tới trạng thái kết thúc (có trần thời gian) trước khi thoát; dọn tệp trong `Dispose` bằng xoá chịu lỗi (`catch IOException or UnauthorizedAccessException`). Kiểm chập chờn bằng cách chạy lặp `dotnet test --no-build` 15–20 lần, đừng tin một lần xanh.
- Dispose đồng bộ mà bộ chứa có singleton chỉ cài `IAsyncDisposable`: đợi qua `Task.Run(async () => await sp.DisposeAsync()).GetAwaiter().GetResult()` để continuation không bị post về SynchronizationContext của xUnit (luồng đang chặn chính là worker của nó).

## EF Core: `ExecuteUpdate` xong muốn đồng bộ thực thể đang theo dõi mà không đánh dấu sửa
- `ExecuteUpdate`/`ExecuteDelete` bỏ qua change tracker: thực thể tracked vẫn mang giá trị cũ. Nếu sau đó thực thể bị ghi cả hàng (`Update()` khi Detached) thì giá trị cũ đè lại DB.
- Bẫy: gán `entry.Property(x => x.P).CurrentValue = v` LUÔN đánh dấu cột Modified (EF so với giá trị hiện tại, không so với gốc) — kể cả khi đã gán `OriginalValue = v` trước. Cách đúng: `OriginalValue = v; CurrentValue = v; IsModified = false;` — không còn cột nào sửa thì EF tự trả thực thể về `Unchanged`, còn cột khác đang sửa thì giữ nguyên.
- Lấy `DbSet.Local` (vd `Local.FindEntry(key)`) là EF đã chạy `DetectChanges` khi `AutoDetectChangesEnabled`, nên `IsModified` sau đó phản ánh cả phép gán chưa lưu của người gọi. Kiểm thêm `entry.State is Unchanged or Modified` để không đụng entry Deleted/Added.

## `Progress<T>` ở server (ASP.NET Core / BackgroundService) chạy callback SONG SONG trên thread pool
- `Progress<T>` chụp `SynchronizationContext` lúc `new`; server không có context nên mỗi `Report` được post thẳng vào thread pool và trả về ngay. Các callback chạy song song với nhau VÀ với code đang gọi `Report`.
- Bẫy thật (một bộ sinh tài liệu): callback gọi `repo.UpdateAsync(entity).Wait(100)` trên cùng `DbContext` scoped mà luồng chính cũng đang dùng → `InvalidOperationException: A second operation was started on this context`. Ném ra từ `.Wait` trên luồng thread pool = ngoại lệ không ai bắt ⇒ **sập cả tiến trình**. `.Wait(timeout)` còn tệ hơn: hết giờ thì bỏ đi trong khi lệnh lưu vẫn chạy.
- Luật: callback `Progress<T>` ở server chỉ gán biến/đẩy vào channel; muốn ghi DB theo tiến độ thì truyền `Func<int, CancellationToken, Task>` và `await` ngay trong vòng lặp (có hạn tần suất).

## Singleton tiêm thẳng service SCOPED (repository/DbContext) = một DbContext chung cả tiến trình
- `CreateApplicationBuilder`/`WebApplication` chỉ bật `ValidateScopes` ở môi trường Development. Ở Production singleton (kể cả `BackgroundService`) nhận repository scoped qua hàm dựng thì được resolve từ root scope ⇒ mọi singleton dùng chung MỘT `DbContext` không bao giờ dispose, và chúng chạy song song (vòng heartbeat, vòng health check, request `Task.Run`) ⇒ lỗi "second operation" ngẫu nhiên + dữ liệu tracked cũ.
- Luật: singleton nhận `IServiceScopeFactory` (hoặc một lớp đọc riêng tự mở scope mỗi lần); chuỗi đọc→sửa→lưu phải nằm trong CÙNG một scope để EF chỉ ghi cột đổi.

## DelegatingHandler đổi `RequestUri` thì phải xử lý cả header `Host`
- Một số thư viện client (vd thư viện HTTP nội bộ với `RequestBuilder`) tự gán `request.Headers.Host = uri.Host` trước khi gửi. Handler phía sau đổi `RequestUri` sang host khác mà để nguyên `Host` ⇒ `SocketsHttpHandler` lấy **SNI và tên kiểm chứng chỉ từ header `Host`** ⇒ HTTPS hỏng (sai tên cert), HTTP thì gửi sai Host (IIS/proxy định tuyến theo host trả 404/400).
- Sửa: sau khi đổi URI thì `request.Headers.Host = null` (để suy ra từ URI, kèm cổng) nếu Host đang là host cũ. Kiểm bằng handler giả làm primary handler (đọc `RequestUri` + `Headers.Host`) VÀ một lời gọi HTTPS thật.

## Resolve transient `IDisposable` (typed HttpClient, v.v.) từ ROOT provider = rò bộ nhớ tới lúc tắt tiến trình
- MS DI giữ tham chiếu mọi transient `IDisposable` nó tạo ra để dispose cùng scope sở hữu. Resolve từ root (`_serviceProvider.GetRequiredService<T>()` trong singleton/BackgroundService) ⇒ scope sở hữu là root ⇒ chỉ dispose khi tắt app. Vòng lặp heartbeat 30s là 2880 bản/ngày không bao giờ giải phóng.
- Typed client kiểu `BaseApi : IDisposable` rơi đúng bẫy này. Sửa: `using var scope = _serviceProvider.CreateScope(); var api = scope.ServiceProvider.GetRequiredService<T>();`, scope phải sống hết phần dùng (kể cả khi đọc stream response).

## EF Core: SaveChanges hỏng để lại hàng chờ ghi trong tracker, và Detach KHÔNG dọn sạch đồ thị
- `SaveChanges` ném (vi phạm unique, lỗi mạng, validation tự viết) thì mọi hàng Added/Modified/Deleted vẫn nằm trong ChangeTracker. DbContext scoped dùng chung cả request ⇒ lệnh lưu kế tiếp (nhánh bù trừ) ghi lại hàng hỏng, ném lại lỗi cũ, che lỗi gốc.
- Gỡ tập trung trong override `SaveChanges`: catch → đặt các entry đó `Detached` → `throw;`. **Detached, KHÔNG Unchanged**: khối chạy lại trong execution strategy đọc lại hàng sẽ nhận đúng instance đang theo dõi; Unchanged biến giá trị chưa ghi thành "giá trị gốc", gán lại cùng giá trị không còn là thay đổi ⇒ lượt sau lưu mà không ghi gì, không báo lỗi.
- Đo thật trên EF 10 (2 bẫy khi Detach):
  - Hàng **Added** bị Detach thì EF tự lấy khỏi collection của cha đang theo dõi; hàng **Modified** thì KHÔNG ⇒ instance chết nằm lại trong collection, code chạy lại khớp vào nó và sửa trên nó (không bao giờ được ghi), còn query kèm Include nạp thêm bản mới cùng khoá. Phải tự gỡ: `collection.Metadata.GetCollectionAccessor()!.Remove(parent, item)` cho mọi entry còn theo dõi.
  - Detach một hàng cha thì EF **gỡ lan** xuống mọi hàng con có quan hệ cascade, kể cả con Unchanged, và lan tiếp theo chuỗi. Tập hàng cần dọn khỏi collection phải đọc lại `State == Detached` SAU khi gỡ, đừng dùng danh sách chụp trước.
- Tắt `ChangeTracker.AutoDetectChangesEnabled` trong lúc dọn: `Entries()` chạy DetectChanges, mà nếu chính DetectChanges là thứ vừa ném thì catch ném lần nữa và thay mất lỗi gốc.
- Test không cần DB: `UseNpgsql("Host=127.0.0.1;Port=1;Timeout=1")` — lượt lưu hỏng thật ở tầng mạng, rồi assert State/collection.
