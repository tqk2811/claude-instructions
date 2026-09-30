# Kinh nghiệm git

- Lỗi `'X' is already used by worktree at '<path>'` khi switch/checkout branch: branch đang bị checkout ở một worktree khác (thường là worktree tạm do phiên Claude khác tạo trong scratchpad `%LOCALAPPDATA%\Temp\claude\...`). Xử lý: `git worktree list` để tìm; **trước khi gỡ phải kiểm tra** worktree đó `status --short` sạch và commit chưa push vẫn nằm trên branch (gỡ worktree không mất commit, chỉ mất thay đổi chưa commit); rồi `git worktree remove <path>`.
- **Backtick trong `git commit -m "..."` chạy qua Bash tool bị nuốt mất.** Bash tool chạy bằng bash nên dấu backtick trong chuỗi nháy kép là command substitution: viết đoạn code kiểu `` `return Page()` `` trong `-m "..."` thì bash cố chạy `return Page()`, in `syntax error near unexpected token '('`, rồi **thay cả cụm bằng chuỗi rỗng** — commit vẫn tạo bình thường, chỉ là câu văn cụt mất một đoạn và không có gì báo động. Cùng lý do với `$` và `\`. Xử lý: mặc định dùng **nháy đơn** `-m '...'` (cần nháy đơn bên trong thì nối `'...it'"'"'s...'`), hoặc bỏ hẳn backtick khỏi message. Commit xong đọc lại bằng `git log -1 --format=%B` để kiểm chứng — phát hiện muộn thì phải amend/rebase, thứ không được tự ý làm.
- **Commit message nhiều dòng qua PowerShell tool: dùng `git commit -F <file>`, đừng dùng here-string `@'...'@`.** Here-string tự nó đúng cú pháp, nhưng PowerShell dựng lại dòng lệnh cho native exe theo quy tắc quoting riêng: mọi dấu `"` NẰM TRONG nội dung message bị hiểu là ranh giới đối số, nên git nhận một mớ đối số rời và báo `error: pathspec '<một mẩu message>' did not match any file(s) known to git` — commit KHÔNG được tạo (may là lỗi ồn ào chứ không âm thầm). Xử lý: ghi message ra file UTF-8 bằng tool Write rồi `git commit -F <file>`; cách này cũng an toàn cho tiếng Việt và cho mọi ký tự đặc biệt khác. Nếu vẫn muốn `-m` thì message tuyệt đối không được chứa `"`.
- `git worktree remove` trên Windows có thể fail `error: failed to delete '...': Filename too long` khi path worktree quá dài (thường gặp với scratchpad temp dài). Xử lý: xóa thư mục thủ công bằng `rm -rf <path>` (Bash tool/git bash xử lý được long path) rồi `git worktree prune` để dọn metadata.

## `git clone <đường dẫn local>` đặt origin trỏ về REPO LOCAL, không phải GitHub

Clone một repo đang nằm trên đĩa để làm việc song song (tránh đụng thư mục đang có người/agent khác
sửa) thì `origin` của bản clone là **đường dẫn thư mục đó**, không phải remote GitHub của nó.
`git push -u origin <branch>` vì thế chạy trót lọt, in ra `* [new branch]` — nhưng branch được tạo
**bên trong repo local**, GitHub không có gì cả.

Xử lý, đúng thứ tự:
1. Ngay sau khi clone, chạy `git remote set-url origin <url GitHub>` **TRONG THƯ MỤC CLONE**. Chạy
   nhầm ở repo gốc thì nó chỉ set lại chính URL cũ của repo gốc, im lặng thành công, không có gì báo
   là bạn vừa sửa nhầm chỗ.
2. Kiểm bằng `git remote -v` và phải thấy `https://...`, không phải `d:/...`.
3. Nếu đã lỡ push nhầm: vào repo gốc `git branch -D <branch>` để xoá ref rác, rồi push lại từ clone.

Dấu hiệu nhận ra ngay ở output: dòng cuối của `git push` ghi `To d:/...` thay vì
`To https://github.com/...`, và **không có** khối `remote: Create a pull request ...`.

## `git rm` (và mọi thứ đã stage từ trước) bị commit ké khi chia commit theo từng phần

- **Triệu chứng:** chia việc thành nhiều commit bằng `git add <đúng danh sách file>` rồi
  `git commit`, nhưng commit ra lại chứa thêm file của việc khác — thường là các file `D` (deleted).
  Commit đó lẫn hai việc và hay **không build được**.
- **Nguyên nhân:** `git rm` **stage ngay** lệnh xoá vào index. `git commit` không kèm `-a` cũng
  không kèm đường dẫn thì commit **TOÀN BỘ INDEX**, chứ không phải "những gì vừa `git add`".
  Mọi thứ stage từ trước đó — `git rm`, `git mv`, một lần `git add` cũ — đều đi ké.
- **Cách tránh:** trước mỗi commit từng phần, `git status --short` và nhìn cột thứ nhất; ký tự nào
  khác khoảng trắng là đã nằm trong index. Hoặc commit theo đường dẫn: `git commit -o <paths> -m ...`
  (chỉ commit đúng path đó, kệ phần index còn lại).
- **Hệ quả nếu lỡ:** commit chưa push thì `git reset --soft HEAD~1` rồi chia lại; đã push thì
  KHÔNG sửa, commit chồng lên và chấp nhận lịch sử có một commit không build được.
- **Mẹo dọn:** muốn xoá file nhưng chưa muốn stage thì dùng `rm` thường, để `git add -A` gom lúc
  đúng commit.

## Đổi branch trong IDE làm "biến mất" toàn bộ công việc chưa commit — tìm ở `git stash list`

- **Triệu chứng:** đang sửa dở một khối lượng lớn (chưa commit), quay lại thì working tree
  **sạch trơn** (`git status` 0 dòng), file mới tạo cũng mất, chỉ còn `bin/`+`obj/` nên thư mục
  project vẫn đứng đó mà rỗng. Trông y như bị `git reset --hard` mất trắng.
- **Nguyên nhân:** Visual Studio / VS Code khi checkout sang branch khác mà working tree bẩn thì
  **tự động stash** rồi checkout, và khi quay về branch cũ nó **KHÔNG tự pop**. Stash đó có message
  rất dễ nhận: `On <branch>: Checking out <branch khác> Stashing uncommitted changes before
  performing a checkout of '<branch>'. Pop or apply these changes to restore them.`
- **Chẩn đoán, theo thứ tự, TOÀN BỘ chỉ đọc — đừng đụng gì vào working tree trước khi xong:**
  1. `git stash list` ← khả năng cao nhất, xử lý xong ở đây là hết chuyện.
  2. `git reflog -10` — nhìn có dòng `checkout: moving from X to Y` không; có thì càng chắc là stash.
  3. `git fsck --lost-found` — chỉ khi hai bước trên trắng tay.
- **Xác minh trước khi khôi phục:** `git stash show --stat stash@{0}` (đếm file) và
  `git stash show --name-status stash@{0}` (xem đúng việc mình làm không).
- **Khôi phục: dùng `git stash apply`, KHÔNG dùng `git stash pop`.** `apply` giữ stash lại làm bản
  sao lưu; `pop` xoá stash ngay, mà nếu apply ra xung đột nửa vời thì mất luôn đường lùi.
- **`apply` có lấy lại cả file mới (untracked) không:** CÓ, nếu stash được tạo kèm untracked —
  IDE luôn tạo kiểu đó. Không cần thêm cờ `-u`. Kiểm bằng cách so
  `git stash show --stat` với `git stash show --include-untracked --stat`: chênh lệch chính là số
  file untracked nằm trong stash.
- **Bẫy:** `bin/`/`obj/` không bị stash nên thư mục project mới vẫn tồn tại sau khi mất file nguồn.
  Đừng lấy "thư mục còn đó" làm bằng chứng là code còn — kiểm bằng
  `find <dir> -name "*.cs" -not -path "*/obj/*" -not -path "*/bin/*"`.
- **Phòng ngừa:** làm việc lớn thì commit sớm lên branch riêng, đừng để hàng trăm file treo ở
  working tree qua một lần đổi branch.

## Tự `git config user.email` bằng email trong context phiên → sai tác giả commit

**Triệu chứng**: commit lên GitHub hiện tác giả là một email khác với tài khoản GitHub của user, không link được
vào profile, dù `user.name` nhìn thì đúng.

**Nguyên nhân**: khởi tạo repo mới rồi tự chạy `git config user.email "<email lấy từ context phiên>"`. Máy user
vốn đã có `~/.gitconfig` với identity ĐÚNG; lệnh đó tạo override ở config **local** đè lên global đúng đó. Email
trong context phiên là để nhận diện user, KHÔNG phải identity commit — hai thứ này có thể khác nhau.

**Cách xử lý**: `git init` rồi để yên — repo tự kế thừa global. Muốn chắc thì chỉ ĐỌC:

```bash
git config --show-origin user.email     # xem giá trị và nó đến từ file nào
git config --global --list | grep user. # xem global có sẵn chưa
```

Chỉ khi global trống mới hỏi user; có thể tham chiếu repo khác của user trên cùng máy để đề xuất:
`git log -1 --format='%an <%ae>'`. Gỡ override lỡ tạo: `git config --unset user.email` (và `user.name`).

**Sửa hậu quả**: commit đã tạo sai tác giả thì phải viết lại history mới đổi được
(`git commit --amend --reset-author` cho commit cuối, `git rebase -r --exec 'git commit --amend --no-edit --reset-author'`
cho nhiều commit) — mà viết lại history đã push thì bị cấm. Nên phát hiện TRƯỚC commit đầu tiên là rẻ nhất.

## `git apply --cached --unidiff-zero` với patch BỎ BỚT hunk → đặt code sai chỗ, commit không build được

- Bối cảnh: muốn chia một file sửa nhiều việc thành nhiều commit, nên `git diff -U0 <file> > all.patch`, lọc ra vài hunk rồi `git apply --cached --unidiff-zero`.
- **Bẫy**: với `-U0` patch KHÔNG có dòng ngữ cảnh, nên git không dò được vị trí; nó tin số dòng trong header `@@ -a,b +c,d @@`. Số `+c` được tính khi CÓ ĐỦ mọi hunk. Bỏ một hunk ở giữa (vd hunk thêm 19 dòng) là mọi hunk sau đó lệch đúng bấy nhiêu dòng ⇒ code bị chèn vào GIỮA một hàm/khối comment khác. Git **không báo lỗi**, apply "thành công".
- Đã vấp thật: khối `try { ConfigurationApplied?.Invoke(); }` bị chèn vào giữa doc-comment của một method khác ⇒ commit đó `error CS1519: Invalid token 'try' in a member declaration`. HEAD cuối cùng vẫn đúng (vì các commit sau "sửa" lại), nên `dotnet build` ở working tree KHÔNG phát hiện được — chỉ lộ ra khi checkout đúng commit đó.
- **Cách làm đúng khi cần tách một file thành nhiều commit**: dựng NỘI DUNG file cho từng trạng thái trung gian (lấy bản ở commit trước rồi chèn thêm bằng `awk`/heredoc theo NEO CHUỖI, không theo số dòng), ghi đè file thật, **build**, rồi mới `git add` + commit. Bản cuối luôn giữ một bản sao (`cp file $SCRATCH/final.cs`) để khôi phục.
- **Luôn kiểm chứng lịch sử vừa tạo**: `git worktree add --detach <dir> <sha>` rồi build từng commit. Đường dẫn worktree phải NGẮN (scratchpad của Claude quá dài ⇒ `git worktree remove` báo `Filename too long`); dùng `D:\tmp\<tên>`.
- Worktree + submodule: `git checkout <sha>` trong worktree **abort** với `fatal: failed to unpack tree object <sha submodule>` vì object của submodule chưa có trong store của worktree. Nạp trước rồi checkout lại:
  ```
  git -C libs/<Sub> fetch <đường-dẫn-submodule-ở-repo-chính> master
  git checkout --detach <sha>; git submodule update --recursive
  ```
  Bẫy kèm: khi checkout abort, lệnh build ngay sau đó vẫn chạy trên commit CŨ và báo "Build succeeded" ⇒ tưởng đã kiểm chứng. Luôn `git log --oneline -1` xác nhận đang ở đúng commit trước khi build.

## `.git` phình lên hàng GB vì blob mồ côi — `git gc --auto` KHÔNG bao giờ tự dọn

- **Triệu chứng**: `.git` nặng vài GB trong khi repo chỉ có code/script (lịch sử thật vài trăm KB).
  `du -sh .git/*` cho thấy gần như toàn bộ nằm ở `.git/objects`, và `objects/pack` thì bé tí.
- **Nguyên nhân**: `git add` (thường là `git add .` hoặc `-f`) lỡ nuốt file build lớn — `.dll`,
  `.zip`, `.nupkg`, artifacts — khi `.gitignore` chưa che hết. `git add` **ghi blob vào
  `.git/objects` ngay lập tức**, kể cả khi sau đó bạn bỏ file khỏi index và KHÔNG BAO GIỜ commit.
  Blob thành mồ côi (unreachable) nhưng vẫn chiếm đĩa vĩnh viễn.
- **Tại sao git không tự dọn**: `git gc --auto` (chạy ké sau `commit`/`merge`...) chỉ kích hoạt khi
  **số lượng** loose object vượt `gc.auto` (mặc định 6700). Ngưỡng tính theo SỐ OBJECT, không theo
  DUNG LƯỢNG — 2000 blob × 50 MB = 4 GB vẫn dưới ngưỡng nên nằm im mãi mãi.
- **Chẩn đoán** (toàn bộ chỉ đọc):
  ```bash
  git count-objects -vH                 # so 'size' (loose) voi 'size-pack'
  git rev-list --objects --all | wc -l  # so object THAT SU reachable
  find .git/objects -type f -not -path "*/pack/*" -size +10M -printf "%TY-%Tm-%Td\n" | sort | uniq -c
  ```
  Dòng cuối cho biết rác sinh vào NGÀY nào — thường dồn vào 1-2 ngày, đúng những lần `git add` nhầm.
- **Kiểm an toàn TRƯỚC khi xoá**: `git stash list` (rỗng) và `git fsck` không có **dangling commit**
  (dangling *blob*/*tree* thì kệ, đó chính là rác cần xoá). Có dangling commit nghĩa là còn commit
  thật chưa gắn branch — phải cứu nó trước, đừng prune.
- **Dọn**:
  ```bash
  git reflog expire --expire-unreachable=now --all
  git gc --prune=now
  ```
  Đã vấp thật: 4.2 GB → 2.8 MB, `git fsck` sạch, `git show-ref` giống hệt trước (gc KHÔNG đụng
  refs/commit/tag). Nên snapshot `git show-ref | sort > before.txt` rồi diff lại sau để yên tâm.
- **Phòng ngừa**: viết `.gitignore` (`bin`, `obj`, `artifacts/`, `Packages/`, `TempOutput/`...)
  **trước** lần `git add` đầu tiên. Với repo hay đẻ artifact lớn, hạ ngưỡng để git tự dọn:
  `git config gc.auto 500`.

## Backtick trong `git commit -m` qua Bash: nuốt mất chữ, không báo lỗi

- **Triệu chứng**: commit message thiếu hẳn một từ so với ý định. Ví dụ viết
  ``moved `size` bytes`` thì message ghi ra `moved  bytes` — hai dấu cách, mất luôn từ trong
  backtick. Nếu chuỗi trong backtick trùng tên một lệnh có thật thì còn tệ hơn: nó được **chạy**
  và kết quả stdout bị chèn vào message.
- **Vì sao**: backtick là cú pháp command substitution của POSIX shell, và `-m "..."` là chuỗi
  double-quoted nên shell vẫn diễn giải backtick bên trong. Markdown code span rất hay dùng
  backtick, nên viết commit message có tên biến/hàm là dính ngay. Bash chỉ in
  `command not found` ra stderr rồi vẫn commit — commit THÀNH CÔNG với message sai.
- **Cách tránh**: dùng here-doc single-quoted cho message nhiều dòng:
  `git commit -F- <<'EOF' ... EOF` hoặc `git commit -m @'...'@` bên PowerShell. Với `-m "..."`
  thì escape `\`` hoặc bỏ hẳn backtick, đổi sang dấu nháy đơn.
- **Hậu quả nếu lỡ**: theo `~/.claude/git.md` thì KHÔNG amend để sửa. Ghi đính chính ở nơi khác
  (doc/changelog) và rút kinh nghiệm.

## Nhánh có thể bị đổi DƯỚI CHÂN mình giữa phiên — kiểm ngay trước mỗi commit

Tự tạo nhánh phụ, commit lên đó, làm tiếp vài lượt rồi commit lượt sau — và lượt sau đó rơi thẳng
vào `master`. Lý do: giữa hai lượt, **user đã tự checkout `master` rồi merge nhánh phụ vào** (thấy rõ
trong `git reflog`: `checkout: moving from feat/x to master` rồi `merge feat/x: Fast-forward`). Cây
làm việc là của user, họ thao tác song song bằng IDE/terminal riêng là chuyện bình thường.

- **Why**: nhánh hiện tại KHÔNG phải trạng thái mình sở hữu. Nhớ "tôi đã tạo nhánh ở đầu phiên" là
  suy luận từ hành động của chính mình, không phải quan sát — đúng kiểu sai lầm mà luật "đã push hay
  chưa thì phải KIỂM" đã cảnh báo, chỉ khác đối tượng.
- **How**: phải là CỔNG CHẶN, không phải dòng in ra. `git branch --show-current && git commit ...`
  KHÔNG chặn được gì — nó in tên nhánh rồi vẫn commit, và mình chỉ đọc được cái tên đó sau khi việc
  đã rồi (đã dính đúng bẫy này lần thứ hai theo kiểu y hệt). Dạng đúng:
  `test "$(git branch --show-current)" = "<nhánh mong đợi>" && git add ... && git commit ...` —
  sai nhánh thì lệnh dừng, không có commit nào được tạo.
- **Kiểm sau**: dòng `[<branch> <hash>]` mà `git commit` in ra luôn nói thật nhánh vừa nhận commit.
  Thấy `[master ...]` mà không xin phép là phải báo user ngay.
- **Lỡ rồi thì KHÔNG tự reset/amend** để "dọn": đó lại là một thao tác nữa cần xin phép. Báo thẳng,
  nêu hash, rồi để user chọn giữ lại hay chuyển sang nhánh khác.
- **Nguyên nhân thứ hai (2026-09-11, lần dính thứ ba)**: một **phiên Claude khác** mở trên cùng repo
  (cùng working tree) chạy `checkout -b feat/y` giữa hai commit của mình ⇒ ba commit tiếp theo rơi
  vào nhánh của phiên kia, đè lên commit của nó. `ListAgents` cho thấy phiên đó; `git reflog` cho thấy
  `checkout: moving from <nhánh mình> to feat/y` mà mình không gõ. Ở đây KHÔNG được checkout/reset để
  gỡ: working tree dùng chung, đổi nhánh là giật file dưới chân phiên kia — hỏi user.
- **`git commit -q` giết luôn chốt "kiểm sau"**: `-q` nuốt dòng `[<branch> <hash>]`. Đừng dùng `-q`,
  hoặc dùng cổng `test "$(git branch --show-current)" = ...` ở trên — cổng mới là thứ chặn được.
- **Phía GÂY RA sự cố trên (cùng vụ 2026-09-11, nhìn từ phiên đã chạy `checkout -b`)**: gitStatus đầu
  phiên ghi `master`, mình tin ảnh chụp đó rồi `checkout -b` — nhưng 7 phút trước phiên kia đã chuyển
  working tree sang nhánh của nó. Hậu quả kép: nhánh mới của mình **mọc từ commit chưa merge của phiên
  kia**, và working tree bị giật khỏi nhánh nó đang làm. Trước **mọi** lệnh đổi HEAD (`checkout`,
  `switch`, `checkout -b`) phải: `git branch --show-current` + `git reflog -3` so với ảnh chụp đầu phiên;
  lệch (có `checkout: moving ...` mình không gõ) ⇒ có phiên khác đang dùng cây này ⇒ **đừng đổi HEAD**,
  làm trong bản clone riêng (`D:\tmp\<tên>`, nhớ `remote set-url` như mục trên) hoặc hỏi user. Cách
  user chốt khi có agent khác chạy: clone ra chỗ khác → commit + push nhánh mới → xoá clone.
- **TÁI PHẠM lần thứ tư (2026-09-17, k-rag-platform), đúng kịch bản đã ghi ngay trên**: gitStatus đầu
  phiên ghi `master`, tôi `git checkout -b docs/tenant-database-erd` mà KHÔNG chạy `git reflog -3`
  trước. Trong ~2 phút HEAD nằm trên nhánh đó, phiên kia commit
  `refactor(tenant-services): make NodeFileStore the one file store` — commit ấy rơi vào nhánh tạm
  của tôi, `master` bị bỏ lại sau 2 commit. Chỗ khác hẳn ba lần trước: **tôi là phía gây ra, và
  người bị hại là agent chứ không phải user**, nên không có ai kêu — chỉ lộ ra vì tôi tình cờ chạy
  `git log --oneline -1 master` và thấy hash lạ.
- **Gỡ được vì may**: `master` chỉ **đi sau** nhánh tạm chứ không phân nhánh, nên
  `git merge --ff-only <nhánh tạm>` đưa master về đúng chỗ, không rewrite gì, rồi `git branch -d`.
  Nếu phiên kia đã kịp commit tiếp lên `master` thì đã thành hai nhánh thật sự và phải hỏi user.
- **How (nhắc lại cho gọn)**: biến nó thành CỔNG, đừng nhớ suông —
  `git reflog -3 | grep -q "checkout: moving" && echo "CÓ PHIÊN KHÁC — ĐỪNG ĐỔI HEAD"` chạy TRƯỚC
  mọi `checkout`/`switch`/`checkout -b`. Và câu hỏi đặt trước đó nữa: việc này có **thật sự** cần
  nhánh không? Lần này file đích nằm trong `Docs/` — sửa xong commit thẳng là xong, nhánh chỉ đẻ ra
  rủi ro chứ không mua được gì.

## Kiểm branch NGAY TRƯỚC MỖI commit, đừng tin cái đã checkout lúc đầu phiên
- Sự cố (2026-09-11, k-rag-platform): đầu phiên tôi `git checkout -b feat/x`, code gần 40 phút rồi commit 3 lần. Giữa chừng có thứ gì đó ngoài lệnh của tôi (user khẳng định không có phiên agent nào khác — nghi IDE/Visual Studio đang mở solution) đưa HEAD về `master`, và `feat/x` (lúc đó còn rỗng) mất theo. Cả 3 commit rơi thẳng vào `master` local, phạm luật "không tự commit vào master". Chỉ phát hiện nhờ `git log` cuối việc thấy một commit lạ xen giữa.
- Cách làm: gộp kiểm vào chính lệnh commit: `test "$(git branch --show-current)" = "feat/x" && git commit ...`. Branch sai thì lệnh dừng, không commit.
- Lỡ rồi thì: `git reflog` để dựng lại dòng thời gian, `git branch feat/x <sha>` giữ commit (không checkout, không đụng master), và KHÔNG tự reset master. Hỏi user trước, vì chưa biết thứ gì đã đổi HEAD và nó còn đứng trên master không.

## `git worktree remove --force` báo "Permission denied" ngay sau build/test (Windows)
- Sự cố (2026-09-11): merge trong một worktree tạm, `dotnet build` + `dotnet test` xong thì `git worktree remove --force <dir>` báo `failed to delete ... Permission denied`. Không tiến trình nào mang đường dẫn đó trong command line — handle do tiến trình build/test/trình theo dõi file giữ, nhả sau vài giây.
- Worktree lúc đó ĐÃ bị gỡ khỏi `git worktree list`, chỉ còn thư mục trên đĩa. Đợi vài giây rồi xoá thư mục bằng `Remove-Item -Recurse -Force` là xong; đừng đi giết tiến trình dotnet chung (có thể là của phiên khác).

## `git add` với pathspec sai chữ hoa/thường: im lặng không thêm gì (Windows)

Sự cố (2026-09-17, k-rag-platform): repo có thư mục tracked là `Docs/`, tôi `git add docs/features/README.md`
— lệnh chạy sạch, **exit 0, không cảnh báo**, nhưng file không vào index. Commit xong `git status`
vẫn hiện `M Docs/features/README.md`, dễ tưởng là file bị sửa lại sau đó.

- Why: `core.ignorecase=true` (mặc định trên Windows) chỉ làm git **khớp tên file đã tracked** theo
  kiểu không phân biệt hoa thường; pathspec của `git add` thì vẫn so khớp phân biệt hoa thường với
  đường dẫn trong index. Thư mục MỚI chưa tracked lại thêm được bình thường (git lấy tên như ta gõ,
  rồi chuẩn hoá về nhánh cha đã tracked) — nên cùng một lệnh có file vào, có file không.
- How: gõ pathspec đúng y như `git ls-files` in ra. Sau mỗi `git add`, đọc `git status --short`:
  còn dòng nào `M` không có dấu staged (` M` chứ không phải `M `) là chưa thêm được.
- **Biến thể ĐỌC, nguy hơn hẳn (2026-09-17, cùng repo, cùng ngày)**: `git ls-files docs/` và
  `git ls-tree -r master -- docs/` trả về **0 dòng, exit 0** trong khi `Docs/` có 108 file tracked.
  Tôi kết luận "cả thư mục `docs/` là untracked, không nằm trong git" và **báo sai cho user hai
  lượt liền**, còn khuyên họ rằng muốn có trong master thì phải `git add` cả thư mục. `ls` trên
  Windows in ra `docs/` chữ thường (tên trên đĩa) nên không có gì mâu thuẫn để mà nghi.
- **Why**: pathspec sai hoa/thường làm lệnh ĐỌC trả rỗng y như khi thật sự không có gì — không có
  cách nào phân biệt "không khớp" với "không tồn tại" qua exit code hay cảnh báo.
- **How**: đừng bao giờ suy "0 dòng ⇒ không được tracked". Trước khi kết luận, kiểm bằng lệnh
  KHÔNG có pathspec: `git ls-files | sed 's|/.*||' | sort -u` liệt kê các thư mục gốc đang tracked
  với đúng chữ hoa thường của index. Nhanh hơn nữa: `git status --short` trên file đang sửa — nó in
  đường dẫn theo index (`M Docs/...`), lộ ngay chữ hoa.
- **Hệ quả nếu lỡ add nhầm chữ thường**: file MỚI thì git lấy tên như ta gõ ⇒ repo có cả `Docs/`
  lẫn `docs/`. Trên Windows không thấy gì, lên Linux (máy deploy) thành hai thư mục khác nhau.

## Clone vào thư mục scratchpad: "Filename too long" → file bị coi là đã xoá

Đường dẫn scratchpad (`%LOCALAPPDATA%\Temp\claude\<project>\<session>\scratchpad\...`) đã dài sẵn,
clone repo có tên file dài vào đó thì checkout in `Filename too long` và BỎ QUA các file ấy. Bản clone
vẫn dùng được nhưng `git status` thấy chúng là ` D` — `git add -A`/`commit -a` sẽ commit lệnh xoá.
Xử lý: `git config core.longpaths true` (chỉ trong bản clone) rồi `git checkout -- .`, kiểm
`git status --short` sạch trước khi sửa. Clone từ repo local thì nhánh mặc định là nhánh ĐANG checkout
ở repo nguồn, `master` chỉ có ở dạng `origin/master` — tạo nhánh từ commit hash cho chắc.

## Luôn kiểm nhánh NGAY trước mỗi `git commit` — user có thể merge/đổi nhánh giữa chừng
- Sự cố 2026-09-23 (k-rag-platform): đang commit dần trên nhánh phụ thì user tự merge nhánh đó vào `master`
  rồi xoá nhánh; working tree bị đưa về `master`. Commit kế tiếp của agent rơi THẲNG vào `master` mà không
  ai hay, cho tới khi agent review báo "repo đang ở master".
- **How to apply:** gộp kiểm nhánh vào chính lệnh commit: `test "$(git branch --show-current)" = "<nhánh dự định>" && git commit ...`.
  Sai nhánh thì DỪNG, báo user; không tự reset/cherry-pick để chữa (đó là sửa lịch sử, phải hỏi).

## `git add` với path sai hoa/thường trên Windows: file đã track KHÔNG được stage, không báo lỗi

- **Triệu chứng:** `git add docs/a.md docs/b.md` chạy êm, nhưng `git status --short` vẫn hiện ` M Docs/a.md`
  (chữ M ở cột 2 = chưa stage); commit ra chỉ có file MỚI, còn các file sửa thì bị bỏ lại.
- **Nguyên nhân:** git lưu path phân biệt hoa/thường (`Docs/`). Trên ổ Windows, file mới thêm bằng
  `docs/x.md` vẫn vào index (git lấy tên thật từ đĩa), nhưng pathspec `docs/a.md` không khớp mục
  `Docs/a.md` đã có trong index nên file sửa bị lờ đi.
- **Cách tránh:** gõ đúng hoa/thường như `git ls-files` in ra, và trước mỗi commit nhìn cột 1 của
  `git status --short`. Lỡ commit thiếu thì tạo commit mới cho phần còn lại (đừng amend khi chưa được phép).

## Clone tạm vào `D:\tmp` thì agent KHÔNG tự xoá được — `D:\tmp` là thư mục làm việc phụ của phiên

- Sự cố 2026-09-26: theo cách "clone ra chỗ khác → commit + push → xoá clone", clone vào
  `D:\tmp\<repo>-rm-upload`. Push xong thì `rm -rf` bị lớp kiểm an toàn có sẵn của Claude Code chặn
  ("would remove a workspace directory"), vì `D:\tmp` được khai là additional working directory. Chặn
  cả khi đã `cd` ra chỗ khác, và cả khi user gõ "cho phép xoá" trong chat. Chỉ lần bấm duyệt ở hộp
  hỏi quyền mới mở được, và không được lách bằng PowerShell hay script.
- **How to apply:** khi user giao kiểu "clone ra chỗ khác rồi xoá", báo trước là bước xoá có thể cần
  user bấm duyệt hoặc tự chạy `Remove-Item -Recurse -Force <dir>`. Bị chặn thì đưa lệnh cho user
  ngay, đừng thử lại nhiều lần.
