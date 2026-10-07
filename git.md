- Không tự ý push/rebase/reset/merge khi chưa được yêu cầu rõ ràng.
- **MỌI commit, trên nhánh nào cũng vậy, đều phải có lượt agent review trước** theo `Quy tắc review & phản biện` trong `~/.claude/CLAUDE.md`.
- Commit: **KHÔNG tự ý commit vào `main`/`master`** — nhánh chính phải có yêu cầu rõ ràng cho từng lần. Trên **nhánh phụ** (mọi nhánh khác) thì được tự commit, không phải xin: làm xong một việc là commit ngay việc đó.
    + Xem xét chức năng mà chia commit, hạn chế commit toàn bộ một lần. Đang đứng ở `main`/`master` mà sắp có nhiều việc thì tạo nhánh mới rồi commit dần trên đó (TRỪ repo `~/.claude` — xem NGOẠI LỆ bên dưới) — xem `Quy tắc branch & commit theo từng bước` trong `~/.claude/CLAUDE.md`. Tự tạo nhánh được, nhưng phải nói tên nhánh cho user biết.
    + **Lệnh commit vào `main`/`master`, và MỌI lệnh push, chỉ có giá trị cho ĐÚNG lần yêu cầu đó**, KHÔNG phải quyền đứng lâu dài. Làm xong lần đó thì thôi; lần kế tiếp phải được yêu cầu lại. Kể cả khi user nói "cứ làm hết", "tự làm đi"... thì đó là cho phép LÀM (sửa code) và commit trên nhánh phụ, KHÔNG mặc nhiên gồm commit vào nhánh chính hay push.
    + Đưa việc đã xong trên nhánh phụ về nhánh chính (merge/rebase/push) là việc RIÊNG, vẫn phải xin.
    + **Repo `~/.claude` (bộ hướng dẫn global) KHÔNG chia nhánh phụ** — mọi thay đổi commit thẳng vào `master` của repo đó, đừng tạo branch rồi xin merge. Các luật khác giữ nguyên: mỗi lần commit vào `master` và mỗi lần push vẫn phải có yêu cầu rõ ràng của user.
- KHÔNG tự ý sửa lại các commit đã tạo (amend/reset/rebase/squash/cherry-pick sửa nội dung...) trừ khi user yêu cầu rõ ràng. Cần thay đổi thì tạo commit MỚI chồng lên.
    + Tuyệt đối KHÔNG amend/reset/force commit đã được push — gây lệch lịch sử local/remote dẫn tới conflict khi push.
    + **"Đã push hay chưa" phải KIỂM, đừng suy từ việc mình không gõ `git push`.** Kiểm bằng `git ls-remote origin <branch>` (hỏi thẳng remote) chứ không phải `git log origin/master..master` — ref `origin/*` là bản chụp local, có thể cũ hoặc đã được thứ khác (IDE, tool, hook) cập nhật. Submodule cũng phải kiểm riêng từng cái.
- Commit message nhiều dòng: KHÔNG dùng here-string PowerShell `@'...'@` trong Bash tool — Bash tool chạy bằng bash (git bash trên Windows), không hiểu cú pháp PowerShell nên ký tự `@` bị lẫn vào subject (vd subject thành `@ Bump ...`). Thay vào đó dùng nhiều cờ `-m` (mỗi `-m` = 1 đoạn): `git commit -m "subject" -m "body/trailer"`. Nếu thật sự cần multiline thì dùng đúng shell của tool (PowerShell tool mới dùng `@'...'@`, Bash tool dùng `$'...\n...'` hoặc nhiều `-m`).
- Nội dung commit message luôn viết bằng **tiếng Anh**.
- **MỌI commit đều theo Conventional Commits**, không có ngoại lệ — kể cả khi repo đó (hoặc submodule đó) đang có lịch sử viết câu trần. Dạng: `type(scope): subject`.
    + `type` dùng: `feat`, `fix`, `perf`, `refactor`, `docs`, `test`, `build`, `ci`, `chore`, `style`, `revert`.
    + `scope` là vùng bị chạm, đặt theo cách repo đó gọi tên (thư mục/project/thành phần, vd `wireproxy`, `ipstack`, `engine`). Không rõ vùng nào thì bỏ scope: `fix: ...`.
    + `subject` viết thường, thể mệnh lệnh, KHÔNG chấm cuối câu, giữ dưới ~72 ký tự.
    + Body vẫn viết tự do (nêu vấn đề → vì sao → đã làm gì), cách subject một dòng trống.
    + Breaking change: thêm `!` sau scope (`feat(api)!: ...`) và/hoặc footer `BREAKING CHANGE: ...`.
    + **Why**: lịch sử trộn hai lối viết thì không lọc/sinh changelog được, và không nhìn subject mà biết commit thuộc loại gì. Một quy tắc duy nhất cho mọi repo còn tránh phải đoán "repo này đang theo lối nào".
- **Mọi commit message PHẢI kết thúc bằng trailer đồng tác giả**, đúng chuỗi hệ thống cấp cho phiên (dạng `Co-Authored-By: Claude <model> <noreply@anthropic.com>`), đặt ở đoạn cuối cùng, cách body một dòng trống — dùng thêm một cờ `-m` cho riêng nó. Thiếu trailer thì không nhìn ra commit nào do agent viết.
- Khi project dùng **git-cliff** thì ngoài Conventional Commits còn phải sinh lại CHANGELOG.md trước khi commit — chi tiết ở `~/.claude/git-cliff.md`.
- Gắn tag git:
    + Tag dùng full SemVer `vX.Y.Z` (đọc `~/.claude/gitversion.md` để biết pattern version).
    + Tag cho một TÍNH NĂNG MỚI: gắn vào **commit ĐẦU TIÊN của tính năng** (thường là commit `feat`), KHÔNG gắn vào commit cuối (test/docs). Lý do theo pattern GitVersion `Major.Minor.<commits-since-tag>`: pack tại đúng commit đó ra `X.Y.0`, các commit test/docs phía sau thành patch tăng dần (`X.Y.1`, `X.Y.2`...).
    + Tag là local cho tới khi push riêng: `git push origin <tag>`; `git push <branch>` KHÔNG tự đẩy tag.
- **Repo mới tạo (`git init`, `gh repo create`) luôn dùng nhánh chính tên `master`**, không dùng `main`: `git init -b master`. Tạo repo GitHub từ thư mục local (`gh repo create --source . --push`) thì đẩy nhánh `master` lên để nó thành nhánh mặc định.
    + **Why**: user thống nhất tên nhánh chính là `master` cho mọi repo; tạo nhầm `main` thì phải đổi tên cả local lẫn remote và đổi nhánh mặc định trên GitHub.
- **KHÔNG BAO GIỜ tự đổi identity git của user** (`user.name` / `user.email`), dù ở `--global`, `--local` hay qua `-c user.email=...`. Chỉ đổi khi user yêu cầu RÕ RÀNG và nói rõ giá trị mới.
    + Máy user đã có sẵn identity đúng trong `~/.gitconfig`. Repo mới `git init` tự kế thừa global — KHÔNG cần và KHÔNG được set lại ở local.
    + **Email trong context phiên (`userEmail`) KHÔNG phải identity commit.** Nó chỉ để nhận diện user, đừng đem đi `git config`; ghi đè global đúng bằng nó là làm sai tác giả commit.
    + Nếu thật sự cần biết identity của repo (vd kiểm trước khi commit): chỉ ĐỌC (`git config user.name`, `git config user.email`, `git config --show-origin user.email`). Thiếu identity thì HỎI user, hoặc lấy từ repo khác của user trên cùng máy (`git log -1 --format='%an <%ae>'`) rồi xác nhận lại — không tự chọn.
