## Chung

- Luôn trả lời bằng tiếng Việt.
- Khi không chắc chắn về yêu cầu, hỏi lại thay vì đoán.
- Thay đổi lớn luôn lên kế hoạch và xem xét các vấn đề trước khi code.
- Tên biến/hàm/class trong code luôn dùng tiếng Anh.
- **Python trên Windows chạy bằng `py`** (py launcher), vd `py - "$f" <<'EOF'`, `py -I script.py`. `python`/`python3` không có trong PATH; lỡ gặp `python: command not found` thì chạy lại bằng `py`, không bỏ sang cách khác. Chi tiết ở `Experiences/Python.md`.
  - **Why**: thông tin này từng chỉ nằm trong file experience; lượt sửa nhỏ quên đọc nên gọi `python`, lỗi, rồi bỏ Python dù máy có sẵn `py`.

## File rule cần đọc trước

Gặp đúng tình huống thì đọc file tương ứng TRƯỚC khi làm. Mỗi phiên đọc một lần; sau compact thì đọc lại.

| Khi | Đọc |
|---|---|
| Thao tác git (commit/push/rebase/reset/merge/cherry-pick…) | `~/.claude/git.md` |
| Lần đầu trong phiên gọi `Agent`/`Workflow`/fork, chạy lệnh nền dài, giao việc cho CLI agent bên ngoài | `~/.claude/agents.md` |
| Sắp giao agent review một đợt sửa | `~/.claude/review.md` |
| Ghi mục backlog, tạo hoặc chia file backlog, sửa một lỗi có thể đang nằm trong backlog | `~/.claude/backlog.md` |
| Code C# (.cs/.csproj/.sln) | `~/.claude/csharp.md` |
| Entity Framework Core (migration, `migrationBuilder.Sql`, raw SQL, `DbContext`, `dotnet ef migrations script`) | `~/.claude/database-EF.md` |
| Giao diện người dùng ở mọi nền tảng (WPF/WinForms/web/mobile: grid, list, table, cuộn, phím tắt…) | `~/.claude/ui-ux.md` |
| Project ASP.NET (.cshtml/Razor/wwwroot/Startup/Program.cs ASP) | `~/.claude/asp.md` |
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
    - **Why**: từ dịch gượng làm người đọc phải đoán nghĩa, lại dễ lẫn với từ thường ("hồ" lẫn với "hồ sơ").
- `Quy tắc sơ đồ`: sơ đồ (kiến trúc, luồng chạy, sequence, trạng thái, ERD…) trong file Markdown LUÔN vẽ bằng Mermaid; KHÔNG dùng ASCII art trong file `.md` (dễ vỡ khi đổi font hoặc sửa một nhánh). Trong câu trả lời chat thì KHÔNG vẽ Mermaid (khung chat không render). Thay vào đó:
  - Được vẽ một sơ đồ ASCII **tổng quan ngắn** trong khối code, rộng tối đa ~80 cột, nhãn chỉ dùng chữ không dấu hoặc tên code tiếng Anh (chữ có dấu và ký tự vẽ khung dễ lệch cột), chỉ vẽ trục chính (ai gọi ai, cái gì chạy nền). Luồng nhiều nhánh rẽ thì đừng cố nhồi vào ASCII.
  - Sau sơ đồ LUÔN có danh sách bước đánh số (thụt lề cho nhánh rẽ) hoặc bảng, kèm link code từng bước, vì link trong khối code không bấm được. Luồng đơn giản thì bỏ sơ đồ, chỉ cần danh sách.
  - User muốn xem hình thật thì đề xuất ghi vào file `.md`.
  - Vẽ/sửa khối Mermaid trong file: đọc `~/.claude/Experiences/Mermaid.md` TRƯỚC dòng đầu tiên, và chỉ báo xong khi đã render ra ảnh và tự xem lại theo mục "Quy tắc bắt buộc" của file đó.
    - **Why**: không đọc file experience có sẵn thì vẽ ra ERD có cạnh đi chung một đoạn, user phải tự phát hiện rồi nhắc lại luật đã ghi.

## Agent & việc nặng

Chi tiết và lý do ở `~/.claude/agents.md` (đọc theo bảng trên). Tóm tắt bắt buộc:

- `Quy tắc chọn model cho agent`: luôn truyền `model` tường minh. Việc đơn giản (tìm file, đọc & tóm tắt, sửa cơ học, chạy lệnh) → `sonnet`; phân tích, việc khó → `opus`; sửa logic vài chỗ → `opus` chốt danh sách sửa rồi `sonnet` làm theo.
  - **`fable` cho BẤT KỲ subagent nào phải xin phép user trước từng lần**, kể cả agent review, kể cả fork khi phiên đang chạy `fable`.
  - Mỗi lần giao việc, báo trong câu trả lời: agent nào → model nào, vì sao, và `subagent_tokens` khi agent xong.
- `Quy tắc chia nhỏ việc nặng`: phiên chính điều phối; việc làm phình ngữ cảnh (đọc nhiều file, sửa cơ học hàng loạt, build/test output dài) giao agent và chỉ nhận kết luận ngắn. Việc nhỏ thì tự làm.
  - Subagent KHÔNG tạo subagent con (`Agent`, `Workflow`, fork) và không tạo cron; brief phải ghi rõ.
  - Agent và lệnh dài (quá ~30 giây) chạy nền; không poll, không `sleep` chờ.
  - Lệnh dài ghi log đầy đủ ra file trong scratchpad rồi mới lọc; không chạy chế độ ít log.
  - **Cron giữ cache** (chỉ phiên chính): việc nền chạy lâu thì kiểm `CronList`; chưa có cron 30 phút giữ prompt cache thì tạo, xong việc thì xoá. **Sau compact, việc ĐẦU TIÊN là kiểm lại cron này.** Giao CLI agent bên ngoài chạy nền thì đặt thêm cron canh kẹt. Chi tiết ở `agents.md`.

## Git & commit

- `Quy tắc branch & commit theo từng bước`: yêu cầu sẽ đẻ ra từ 2 thay đổi/chức năng trở lên (kể cả việc user thêm giữa chừng) thì NGAY TỪ ĐẦU tạo branch mới và chạy theo nhịp `làm xong 1 việc → build/test → (review nếu phức tạp) → commit → việc tiếp`. KHÔNG dồn hết rồi mới chia commit ở cuối.
  - Được tự commit và tự push nhánh phụ đó, chỉ báo tên branch. Commit/push vào `main`/`master`, merge/rebase về nhánh chính vẫn phải có yêu cầu rõ ràng (chi tiết ở `~/.claude/git.md`).
  - **NGOẠI LỆ, repo `~/.claude`**: KHÔNG chia nhánh phụ; mọi thay đổi commit thẳng vào `master`, mỗi lần commit và push đều cần user yêu cầu.
- `Quy tắc review & phản biện`: chỉ đợt sửa PHỨC TẠP mới cần **một** lượt subagent review độc lập trước `git commit` của đợt đó (không dùng git thì trước khi báo xong). Phức tạp là khi đụng logic nghiệp vụ/luồng chạy, kiến trúc, hợp đồng API, schema/migration, dữ liệu không hoàn tác được, quyền/xác thực/tính tiền, đồng thời, trải nhiều file/tầng, hoặc mình chưa chắc. Cần review thì đọc `~/.claude/review.md` trước khi giao agent. Agent review chạy READ-ONLY; tự kiểm lại finding trước khi sửa theo; tối đa 2 vòng review cho một đợt. Bỏ qua review thì nói rõ trong câu trả lời và vì sao.
- `Quy tắc line ending`: **dùng CRLF**.
  - File mới ghi CRLF. File đã có thì giữ kiểu sẵn có: dòng không đổi giữ nguyên, dòng thêm/sửa theo kiểu đa số của file (đối chiếu `git show HEAD:<path>`), KHÔNG ép cả file sang kiểu khác.
  - **Ngoại lệ chỉ dùng LF**, kể cả file mới: `*.sh`, `*.bash`, file có shebang `#!`, `Dockerfile`, `docker-entrypoint`, file `systemd`/`cron`, `Makefile` (CRLF làm script lỗi `/bin/bash^M: bad interpreter`). Repo có `.gitattributes` quy định khác thì theo `.gitattributes`.
  - `sed -i` / `perl -pi` / script node/python chạy qua Git Bash dễ ghi ra LF. Sửa hàng loạt xong thì so `git diff --stat` với `git diff --stat --ignore-cr-at-eol`; lệch là đã đổi line ending ngoài ý muốn. Giao agent sửa code thì dặn luật này trong brief.
  - Trước MỌI commit có bước kiểm riêng, xem `~/.claude/git.md`. Bẫy msys nuốt `\r` khi hiển thị: `~/.claude/Experiences/Bash.md`.
- `Quy tắc clone tạm`: cần làm song song trên bản clone riêng (vd agent khác đang dùng workspace) thì clone vào thư mục clone tạm ghi ở `~/.claude/local.md`, dạng `<gốc clone>\<tên repo>\<tên branch>`, KHÔNG clone vào scratchpad; luôn bật `-c core.longpaths=true`. Làm xong (commit + push branch mới) thì xoá thư mục clone.
  - **Why**: đường dẫn scratchpad rất dài nên checkout repo có file nằm sâu lỗi "Filename too long" trên Windows.
- `Quy tắc test hồi quy`: workspace đã có bộ test thì **mọi lần sửa lỗi đều kèm test chặn lỗi cũ**, commit chung với bản sửa, không cần user nhắc.
  - Test phải đi đúng đường gây lỗi và khẳng định hành vi đúng. Có sẵn bài `Skip` mô tả đúng lỗi đó thì bỏ `Skip` và đổi tên cho khớp, thay vì viết bài trùng.
  - **Chứng minh test bắt được lỗi**: tạm gỡ bản sửa (hoặc chạy trên code cũ) để thấy test ĐỎ, rồi khôi phục và thấy XANH. Test chưa từng đỏ trên code lỗi thì chưa chứng minh được gì.
  - Phủ thêm các ca biên sát bản sửa khi chúng rẻ để viết (vd không đụng dữ liệu của người khác, ca tự thao tác lên chính mình).
  - Không test được (lỗi chỉ lộ ở hạ tầng thật, UI thuần, chưa có chỗ test phù hợp) thì NÓI RÕ lý do; đáng làm về sau thì ghi backlog.
  - Workspace chưa có test thì không tự dựng project test mới chỉ vì một bản sửa; hỏi user trước.
  - **Why**: lỗi đã sửa mà không có test giữ lại thì lần refactor sau dễ làm nó quay lại mà không ai hay.

## Memory, backlog, experience

- Luôn chủ động lưu vào memory local của project (`~/.claude/projects/<project>/memory/`) NGAY khi có tiến triển/quyết định/việc dở, KHÔNG đợi cuối phiên (phòng compact mất ngữ cảnh). Tách hai loại tệp:
  - **Kiến thức** (quyết định + lý do, bẫy, luật user dặn, thông tin tra cứu): mỗi chủ đề một file `<chủ-đề>.md`, ưu tiên sửa file sẵn có. KHÔNG ghi hash commit, tên nhánh, "đã merge/chưa push", nhật ký từng bước.
  - **Tiến độ** (đang làm tới đâu, nhánh/commit, việc còn treo, bước kế tiếp): file riêng `progress-<chủ-đề>.md` (`type: project`), file kiến thức trỏ sang nó.
  - **Việc xong thì xoá tiến độ và lịch sử** (BẮT BUỘC, ngay trong lượt việc đó xong, hoặc khi user nói thôi không làm): chuyển bài học còn giá trị sang file kiến thức, rồi XOÁ file `progress-*` và dòng của nó trong `MEMORY.md`, đồng thời xoá mọi dòng lịch sử của việc đó còn sót ở file khác. Không giữ "đã xong", không gạch ngang, không dời sang mục lưu trữ; lịch sử đã có trong git.
  - **Ghi vào đúng repo đã sửa**: phiên mở ở repo A mà sửa sang repo B (thư viện và app dùng nó, worktree/clone tạm của repo khác) thì memory ghi vào memory của project B (đường dẫn theo repo gốc của B, không theo worktree), backlog ghi vào `docs/` của B. Memory của A chỉ giữ phần thuộc A, cần thì một dòng trỏ sang B.
  - `MEMORY.md` chỉ là mục lục: mỗi file một dòng ngắn (chủ đề + hook), không ghi trạng thái nhánh/commit.
  - **Why**: tiến độ lẫn vào file kiến thức làm memory phình (có project lên ~250 file, 1,1 MB) toàn lịch sử việc đã xong; tiến độ repo khác dồn vào repo mở phiên thì mở phiên ở repo kia không thấy gì.
- `Quy tắc Backlog`: việc còn treo và việc nên làm của project (việc dở, lỗi có sẵn ngoài phạm vi đang sửa, đề xuất chưa duyệt, finding review bị hoãn) ghi vào backlog trong `docs/` (hoặc `Docs/`) của project đó. Mỗi mục có vấn đề, path + dòng, vì sao nên làm, ngày ghi; tiêu đề mở đầu bằng `[Cao]`/`[Vừa]`/`[Thấp]`. Xử lý xong thì XOÁ mục trong cùng commit với bản sửa; trước khi commit, rà xem đợt sửa có đóng mục nào không. Dạng lưu, cách chia file và tiêu chí nhãn: `~/.claude/backlog.md`.
- `Quy tắc experience`: bài học chung nằm ở `~/.claude/Experiences/*.md`.
  - **Đọc (BẮT BUỘC)**: trước lệnh hoặc lượt sửa ĐẦU TIÊN trong phiên dùng tới một ngôn ngữ, shell, công cụ, định dạng hay thư viện, đọc file experience khớp với nó. Áp cả cho thứ không được gọi là "ngôn ngữ lập trình" (Mermaid, CSS, sqlcmd, Selenium…) và cho việc trông rất nhỏ (một lệnh `sed`, một khối Mermaid). Mỗi phiên đọc một lần; sau compact thì đọc lại. Không chắc có file khớp không thì chạy `ls ~/.claude/Experiences/*.md`. Bảng tra hiện tại:
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
  - **Ghi**: gặp và xử lý được một lỗi (dù do user hay do mình): **bài học chung** (áp cho mọi project) ghi vào `~/.claude/Experiences/<tên ngôn ngữ/công cụ>.md` theo `Quy tắc ẩn danh ~/.claude`; **vấn đề riêng của project** ghi vào memory local của project. Tạo file experience MỚI thì thêm một dòng vào bảng tra ở trên trong cùng lượt sửa.
  - **Đường dẫn phân biệt hoa thường**: bộ `~/.claude` còn dùng trên Linux, nên mọi đường dẫn nhắc tới phải viết ĐÚNG hoa thường như tên file thật. Tạo hay đổi tên file thì grep lại các chỗ tham chiếu và so khớp với `git ls-files`.
  - **Why**: luật đọc lỏng thì thứ không được coi là "ngôn ngữ" (Mermaid) và việc nhỏ bị bỏ qua, nên lỗi đã ghi lại vẫn lặp lại.

## Repo `~/.claude`

- `Quy tắc đặt tên file Markdown`: file `.md` tiếng Anh giữ nguyên tên; file `.md` KHÔNG phải tiếng Anh thì thêm hậu tố mã ngôn ngữ ISO 639-1 trước `.md`, dạng `<tên gốc>-<mã>.md` (vd `<tên gốc>-vi.md`). NGOẠI LỆ: các file trong `~/.claude` giữ tên cố định, KHÔNG thêm hậu tố.
- `Quy tắc ẩn danh ~/.claude`: các file `.md` được git track trong `~/.claude` (CLAUDE.md, file rule, `Experiences/*.md`) là bộ hướng dẫn dùng chung, phải viết CHUNG CHUNG. Thông tin riêng của máy hoặc người dùng ghi vào `~/.claude/local.md` (bị `.gitignore` chặn); file track chỉ trỏ sang đó.
  - **KHÔNG ghi vào file track**: tên project/repo/sản phẩm thật, tên khách hàng, tên thư viện riêng của user, đường dẫn máy cá nhân (`D:\...`, `C:\Users\<tên>\...`), username/email, tên cấu hình build lộ tên khách, tên class/namespace/bảng mang nghiệp vụ của một sản phẩm cụ thể. Ngoại lệ: dòng `Co-Authored-By` bắt buộc trong `git.md`; thư viện của user đã publish public trên NuGet (`TqkLibrary.*`, `FFmpegArgs`) thì ghi thẳng tên gói/API (đường dẫn source trên máy vẫn để ở `local.md`).
  - **Thay bằng**: placeholder (`<App>`, `<repo>`, `MyApp`, `C:\src\...`), ví dụ nghiệp vụ trung tính (`Product`, `Order`), hoặc mô tả vai trò ("thư viện HTTP nội bộ"). Ghi chú sự cố thì giữ ngày và bài học, bỏ tên project: "(2026-09-11)", không phải "(ProjectX, 2026-09-11)".
  - Thông tin riêng mà vẫn cần cho việc chạy (vị trí SDK, repo mẫu, tên gói polyfill…) để trong `local.md`; file track ghi "xem `~/.claude/local.md`". Gặp chỉ dẫn như vậy thì đọc `local.md` trước khi làm.
  - Trước khi commit repo `~/.claude`, grep phần diff tìm tên project, đường dẫn máy và username.
  - **Why**: repo `~/.claude` có thể được chia sẻ hoặc public; tên project, khách hàng và kiến trúc nội bộ từng lọt vào file rule qua ghi chú sự cố, rồi phải đi ẩn danh lại hàng loạt.
