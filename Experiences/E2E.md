# Bài học khi chạy test E2E (host web thật + trình duyệt headless) song song

Áp dụng cho bộ E2E kiểu: fixture dựng host ASP.NET (Kestrel) + API giả trong bộ nhớ + trình duyệt
headless (Playwright/Selenium), chạy bằng xUnit. Bộ ~150 bài chạy tuần tự mất 5–15 phút.

## Cách chuẩn: song song bằng NHIỀU PROCESS, trong mỗi process vẫn tuần tự

- Trong một process giữ nguyên: mọi lớp chung MỘT `ICollectionFixture` (một host + một trình duyệt),
  `xunit.runner.json` tắt song song (`parallelizeTestCollections: false`, `parallelizeAssembly: false`).
  Bật song song trong process thì các bài xoá/ghi đè trạng thái giả của nhau, và static của cả process
  (culture, `IHttpContextAccessor`/`AsyncLocal`, biến môi trường) bị dùng chung ⇒ đỏ chập chờn.
  KHÔNG "sửa" bằng cách chuyển sang `IClassFixture` + bật parallel trong process.
- Song song đến từ N process `dotnet test --no-build --filter ...` chạy cùng lúc, mỗi process tự dựng
  fixture riêng. Viết thành script ở gốc repo (vd `run-e2e-parallel.ps1`):
  1. build một lần;
  2. `dotnet test --no-build --list-tests`, nhóm tên bài theo lớp;
  3. chia lớp vào N shard bằng tham lam (lớp nặng nhất vào shard nhẹ nhất); trọng số = thời lượng từng
     lớp đọc từ trx lượt trước (`-DurationTrx`), không có thì theo số bài; kiểm tổng bài các shard bằng
     tổng bài liệt kê;
  4. mỗi shard: `--filter "FullyQualifiedName~<Lớp>.|FullyQualifiedName~<Lớp2>."` (dấu `.` cuối để
     `FooTests.` không khớp nhầm `FooTestsX`), `--logger "console;verbosity=normal"`, trx riêng
     (`--logger "trx;LogFileName=<shard>.trx"`, `--results-directory`), stdout/stderr ra file riêng;
  5. chờ hết, đọc trx từng shard: in số đỗ/đỏ + tên bài đỏ; shard không có trx = process chết ⇒ báo đỏ;
  6. tuỳ chọn `-RerunFailed`: chạy lại tuần tự các bài đỏ trong một process; mã thoát VẪN là 1 nếu lượt
     đầu có đỏ, để không che bài chập chờn.
- PowerShell: `Start-Process ... -PassThru` rồi đọc `$p.Handle` ngay, nếu không `ExitCode` rỗng sau khi
  process thoát.
- Số shard mặc định 4: mỗi shard là một host + một trình duyệt, nhiều hơn thì tranh CPU và nổ timeout
  điều hướng. Nhiều bộ E2E (web người dùng, web quản trị) thì mỗi bộ chia shard riêng, chạy cùng lúc được.

## Điều kiện để các process không đụng nhau

- Host nghe cổng động (`http://127.0.0.1:0`, đọc lại cổng thật sau khi start), không cổng cố định.
- Trình duyệt profile riêng (Playwright tự tạo profile tạm mỗi lần launch; Selenium thì `--user-data-dir`
  tạm riêng; xem `chrome.md` trước khi đụng tiến trình chrome).
- Thư mục tạm, file artifact (ảnh chụp lỗi, trace) đặt theo process/shard, không đường dẫn cố định chung.
  Code app hay dùng `Path.GetTempPath()` + tên thư mục cố định (vd thư mục nhập file) ⇒ các process giẫm
  nhau. Cách rẻ: fixture đặt `TMP`/`TEMP` của process sang thư mục riêng `<app>-e2e-<pid>` trước khi
  dựng host, xoá khi thoát.
- Bài Theory làm số kết quả trong trx lớn hơn số dòng `--list-tests`: chỉ coi là thiếu (process chết)
  khi trx ÍT hơn số bài liệt kê.
- Có DB thật: DB tạm tên `<app>_test_<Guid:N>` dựng bằng chuỗi migration thật; dọn thì chỉ clear pool của
  DB đó (KHÔNG `ClearAllPools`) rồi drop; migrate hỏng giữa chừng cũng drop. DB test tên cố định dùng
  chung là nguồn đỏ chập chờn số một khi nhiều process/nhiều người chạy cùng lúc.

## Đo và chống chập chờn

- Đo trước/sau trên cùng máy; ghi rõ nếu máy đang có việc khác chạy.
- Chạy bản nhiều process ít nhất 2–3 lượt. Bài đỏ chỉ khi chạy nhiều process là lỗi cô lập (tài nguyên
  cố định dùng chung): tìm và tách, đừng tăng timeout bừa, đừng nới điều kiện, đừng `Skip`. Chưa tái
  hiện được thì giữ artifact của lượt hỏng và ghi Backlog.
- Bài timeout điều hướng chỉ lúc máy tải nặng thì giảm số shard trước khi đụng code.
- Ghi cách chạy (lệnh mẫu, tham số) vào tài liệu test của repo.
