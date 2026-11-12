## Chung

- Luôn trả lời bằng tiếng Việt.
- Khi không chắc chắn về yêu cầu, hỏi lại thay vì đoán.
- Thay đổi lớn luôn lên kế hoạch và xem xét các vấn đề trước khi code.
- Tên biến/hàm/class trong code luôn dùng tiếng Anh.
- `Quy tắc gom phần giống nhau`: thấy nhiều bảng/class có cùng một nhóm cột hoặc chức năng giống nhau (vd nhiều bảng đều có cột nội dung soạn thảo HTML + Markdown + bảng tệp đi kèm) thì HỎI user có tách nhóm đó ra thành bảng/thành phần dùng chung không, không tự quyết. User không đồng ý tách thì ghi comment ngắn tại chỗ trong code (vd ở entity/configuration) nói rằng đã cân nhắc gom chung và user chọn giữ riêng, kèm lý do nếu có.
  - **Why**: phần giống nhau nằm rải rác nhiều nơi thì khó quản lý; comment giữ lại quyết định để lần sau không đề xuất lại.
- **Python trên Windows chạy bằng `py`** (py launcher), vd `py - "$f" <<'EOF'`, `py -I script.py`. `python`/`python3` không có trong PATH; lỡ gặp `python: command not found` thì chạy lại bằng `py`, không bỏ sang cách khác. Chi tiết ở `Experiences/Python.md`.

## File rule cần đọc trước

Gặp đúng tình huống thì đọc file tương ứng TRƯỚC khi làm. Mỗi phiên đọc một lần; sau compact thì đọc lại.

| Khi | Đọc |
|---|---|
| Thao tác git (commit/push/rebase/reset/merge/cherry-pick/clone tạm…) | `~/.claude/git.md` |
| Lần đầu trong phiên gọi `Agent`/`Workflow`/fork, chạy lệnh nền dài, giao việc cho CLI agent bên ngoài | `~/.claude/agents.md` |
| Sắp giao agent review một đợt sửa, hoặc phân vân đợt sửa có cần review không | `~/.claude/review.md` |
| Lần ghi memory đầu tiên trong phiên | `~/.claude/memory.md` |
| Sửa lỗi trong workspace có bộ test | `~/.claude/testing.md` |
| Ghi mục backlog, tạo hoặc chia file backlog, sửa một lỗi có thể đang nằm trong backlog | `~/.claude/backlog.md` |
| Tạo/sửa/commit file trong repo `~/.claude` | `~/.claude/claude-repo.md` |
| Code C# (.cs/.csproj/.sln) | `~/.claude/csharp.md` |
| Entity Framework Core (migration, `migrationBuilder.Sql`, raw SQL, `DbContext`, `dotnet ef migrations script`) | `~/.claude/database-EF.md` |
| Giao diện người dùng ở mọi nền tảng (WPF/WinForms/web/mobile: grid, list, table, cuộn, phím tắt…) | `~/.claude/ui-ux.md` |
| Project ASP.NET (.cshtml/Razor/wwwroot/Startup/Program.cs ASP) | `~/.claude/asp.md` |
| Deploy ASP.NET lên IIS, script cấu hình IIS (app pool, site, `WebAdministration`) | `~/.claude/asp-iis.md` |
| ASP.NET: dựng project mới; thêm project/tầng/bảng/entity/repository/service/controller/trang Razor/client typed/tài nguyên i18n; refactor lớn hoặc tạo mới trong project cũ | `~/.claude/asp-architecture.md` (xem `Quy tắc kiến trúc ASP`) |
| Android SDK/NDK (`app_process` bằng javac + d8, `ANDROID_HOME`, `android.jar`, `build-tools/<ver>/d8`, NDK) | `~/.claude/android.md` |
| Chrome (headless, `--screenshot`, `--dump-dom`, CDP/`--remote-debugging-port`), và **mọi lúc định giết `chrome.exe`** | `~/.claude/chrome.md` |
| ffmpeg/ffprobe trong code (dựng lệnh, filter graph, probe, transition, render) | `~/.claude/ffmpeg.md` |
| GitVersion (GitVersion.yml/.json, `$(GitVersion_*)` trong .csproj/.targets) | `~/.claude/gitversion.md` |
| changelog/git-cliff (cliff.toml, `Changelog.ps1`, CHANGELOG.md, `changelog.yml`, release notes) | `~/.claude/git-cliff.md` |
| CI phát hành ứng dụng (`release.yml`, tag phát hành, GitHub Releases, version lấy từ tag) | `~/.claude/ci-release.md` |
| Đóng gói/publish NuGet thư viện C# (nuspec, `dotnet pack`, version package, `GenerateDocumentationFile`, `EmbedAllSources`, `DebugType`, pack pdb, `ProjectBuildProperties.targets`) | `~/.claude/BuildNuggetCsharpLibrary.md` |

- `Quy tắc kiến trúc ASP` (BẮT BUỘC): tuân theo `asp-architecture.md` (tách `Database.Entities` / `Database.<Provider>` / `Repositories.Abstractions` / `Repositories`; tên hàm/class/file/bảng/cột dạng lạc đà; i18n qua project `<P>.Resources`). Việc định làm xung đột với kiến trúc đang có của project cũ thì HỎI user trước, không tự quyết, không trộn hai khuôn.
- Chrome: **kiểm `--user-data-dir` của từng tiến trình trước khi kill, không bao giờ kill theo tên ảnh**. Chrome cá nhân của user và Chrome headless là cùng một `chrome.exe`.

## Trình bày câu trả lời

- `Quy tắc link code`: file trong workspace ghi dạng markdown link `[filename.ext:line](relative/path#Lline)` (range dùng `#Lstart-Lend`). File ngoài workspace KHÔNG bọc markdown link (VSCode không click được path tuyệt đối), chỉ in path trần kèm dòng, vd `C:\Users\...\file.md:5`.
- Sau mỗi lần edit/write code, kết thúc câu trả lời bằng danh sách các vị trí đã sửa hoặc thêm theo `Quy tắc link code`, kèm tóm tắt nội dung để user theo dõi. Sửa nhiều block trong cùng 1 file thì liệt kê từng block riêng.
- Khi được yêu cầu mô tả luồng chạy, luôn dẫn kèm link từng khối code theo `Quy tắc link code`.
- `Quy tắc thuật ngữ`: dùng thuật ngữ chuyên ngành trong câu trả lời thì dẫn markdown link tới mục giải thích trong `docs/Glossary-vi.md` của project. Link PHẢI dùng anchor SỐ DÒNG `#Lxx` (vd `[FSM](docs/Glossary-vi.md#L75)`), KHÔNG dùng anchor heading (vd `#fsm`) vì khung chat VSCode chỉ nhảy được anchor số dòng. File hoặc mục chưa có thì tạo/bổ sung (grep lấy số dòng thật) rồi mới dẫn link.
  - **Không dịch gượng thuật ngữ IT**: thuật ngữ dân IT Việt Nam vẫn quen gọi bằng tiếng Anh thì giữ nguyên (`pool`, `cache`, `queue`, `thread`, `lock`, `deploy`, `commit`). Chỉ dùng từ tiếng Việt khi nó thật sự thông dụng trong ngành (vd `pool` không dịch "hồ"; muốn dùng tiếng Việt thì "bể chứa").
- `Quy tắc sơ đồ`: sơ đồ trong file Markdown LUÔN vẽ bằng Mermaid, KHÔNG dùng ASCII art trong file `.md`. Trong câu trả lời chat thì KHÔNG vẽ Mermaid (khung chat không render):
  - Được vẽ một sơ đồ ASCII **tổng quan ngắn** trong khối code, rộng tối đa ~80 cột, nhãn chỉ dùng chữ không dấu hoặc tên code tiếng Anh, chỉ vẽ trục chính. Luồng nhiều nhánh rẽ thì đừng cố nhồi vào ASCII.
  - Sau sơ đồ LUÔN có danh sách bước đánh số (thụt lề cho nhánh rẽ) hoặc bảng, kèm link code từng bước. Luồng đơn giản thì bỏ sơ đồ, chỉ cần danh sách. User muốn xem hình thật thì đề xuất ghi vào file `.md`.
  - Vẽ/sửa khối Mermaid trong file: đọc `~/.claude/Experiences/Mermaid.md` TRƯỚC dòng đầu tiên, và chỉ báo xong khi đã render ra ảnh và tự xem lại theo mục "Quy tắc bắt buộc" của file đó.

## Agent & việc nặng (chi tiết ở `agents.md`)

- Luôn truyền `model` tường minh cho subagent: việc đơn giản → `sonnet`; phân tích, việc khó → `opus`. **`fable` cho BẤT KỲ subagent nào (kể cả review, kể cả fork khi phiên đang chạy `fable`) phải xin phép user trước từng lần.** Mỗi lần giao việc báo agent nào → model nào, vì sao, và `subagent_tokens` khi xong.
- Phiên chính điều phối; việc làm phình ngữ cảnh giao agent. Subagent KHÔNG tạo subagent con và không tạo cron (ghi rõ trong brief). Agent và lệnh dài chạy nền, không poll; log đầy đủ ra file trong scratchpad rồi mới lọc.
- **Sau compact, việc ĐẦU TIÊN là kiểm lại cron giữ cache** (`CronList`), xem `agents.md`.

## Git & commit (chi tiết ở `git.md`, `review.md`, `testing.md`)

- `Quy tắc branch & commit theo từng bước`: việc sẽ đẻ ra từ 2 thay đổi trở lên thì NGAY TỪ ĐẦU tạo branch mới, làm xong việc nào build/test/commit việc đó. Được tự commit và push nhánh phụ; commit/push/merge vào `main`/`master` phải có yêu cầu rõ ràng. Repo `~/.claude` thì commit thẳng `master` nhưng mỗi lần commit/push đều cần user yêu cầu.
- `Quy tắc review & phản biện`: chỉ đợt sửa PHỨC TẠP mới cần **một** lượt subagent review (READ-ONLY) trước commit; tiêu chí và cách làm ở `review.md`. Bỏ qua review thì nói rõ và vì sao.
- `Quy tắc test hồi quy`: workspace đã có bộ test thì mọi lần sửa lỗi kèm test chặn lỗi cũ, phải chứng minh test từng đỏ trên code lỗi; chi tiết ở `testing.md`.
- `Quy tắc line ending`: **dùng CRLF**.
  - File mới ghi CRLF. File đã có thì giữ kiểu sẵn có: dòng không đổi giữ nguyên, dòng thêm/sửa theo kiểu đa số của file (đối chiếu `git show HEAD:<path>`), KHÔNG ép cả file sang kiểu khác.
  - **Ngoại lệ chỉ dùng LF**, kể cả file mới: `*.sh`, `*.bash`, file có shebang `#!`, `Dockerfile`, `docker-entrypoint`, file `systemd`/`cron`, `Makefile`. Repo có `.gitattributes` quy định khác thì theo `.gitattributes`.
  - `sed -i` / `perl -pi` / script node/python chạy qua Git Bash dễ ghi ra LF. Sửa hàng loạt xong thì so `git diff --stat` với `git diff --stat --ignore-cr-at-eol`. Giao agent sửa code thì dặn luật này trong brief. Trước MỌI commit có bước kiểm riêng, xem `git.md`.

## Memory, backlog, experience

- Luôn chủ động lưu vào memory local của project NGAY khi có tiến triển/quyết định/việc dở, KHÔNG đợi cuối phiên. Tách file kiến thức `<chủ-đề>.md` và file tiến độ `progress-<chủ-đề>.md`; **việc xong thì xoá file tiến độ và dòng của nó trong `MEMORY.md` ngay**; ghi vào memory của đúng repo đã sửa. Chi tiết ở `memory.md`.
- `Quy tắc Backlog`: việc còn treo/nên làm của project ghi vào backlog trong `docs/` của chính repo đó (mỗi repo, kể cả submodule, một backlog riêng), tiêu đề mở đầu `[Cao]`/`[Vừa]`/`[Thấp]`; xử lý xong thì xoá mục trong cùng commit. Chi tiết ở `backlog.md`.
- `Quy tắc experience`: bài học chung nằm ở `~/.claude/Experiences/*.md`.
  - **Đọc (BẮT BUỘC)**: trước lệnh hoặc lượt sửa ĐẦU TIÊN trong phiên dùng tới một ngôn ngữ, shell, công cụ, định dạng hay thư viện, đọc file experience khớp với nó, kể cả thứ không được gọi là "ngôn ngữ lập trình" (Mermaid, CSS, sqlcmd…) và việc trông rất nhỏ. Mỗi phiên đọc một lần; sau compact thì đọc lại. Không chắc thì `ls ~/.claude/Experiences/*.md`. Bảng tra:
    - `Experiences/Bash.md`: dùng Bash tool, viết script sh, `sed`, heredoc.
    - `Experiences/perl.md`: sửa file bằng `perl -pi` / `perl -0pi`.
    - `Experiences/PowerShell.md`: dùng PowerShell tool, viết `.ps1`.
    - `Experiences/git.md`: mọi thao tác git làm thay đổi trạng thái (commit, branch, stash, clone, rm…).
    - `Experiences/Csharp.md`: `.cs`/`.csproj`, EF Core, WPF.
    - `Experiences/CSS.md`: `.css`, khối `<style>`, CSS trong Razor.
    - `Experiences/JavaScript.md`: `.js`, `<script>`, API trình duyệt.
    - `Experiences/Markdown.md`: viết hoặc sửa file `.md` (link, anchor, tiêu đề).
    - `Experiences/Mermaid.md`: khối ```` ```mermaid ```` trong file `.md`.
    - `Experiences/Python.md`: viết hoặc chạy Python.
    - `Experiences/Selenium.md`: tự động hoá trình duyệt, CDP.
    - `Experiences/E2E.md`: test E2E (host web + trình duyệt headless), chạy song song nhiều process (chia shard theo lớp).
    - `Experiences/sqlcmd.md`: chạy SQL bằng `sqlcmd`.
    - `Experiences/PostgreSQL.md`: PostgreSQL/Npgsql (mã lỗi, khoá, transaction, SQL thô).
  - **Ghi**: gặp và xử lý được một lỗi: **bài học chung** ghi vào `~/.claude/Experiences/<tên ngôn ngữ/công cụ>.md` (viết ẩn danh, xem `claude-repo.md`); **vấn đề riêng của project** ghi vào memory local của project.
- File `.md` không phải tiếng Anh đặt tên `<tên gốc>-<mã ISO 639-1>.md` (vd `-vi.md`), trừ file trong `~/.claude`.
