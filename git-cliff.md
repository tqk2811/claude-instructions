## git-cliff — changelog + GitHub release notes (đi với GitVersion)

git-cliff lo **changelog/timeline**; **version** do GitVersion lo (`~/.claude/gitversion.md`).
Repo mẫu (mẫu đầy đủ theo model CI, mẫu đơn giản tag-trigger, mẫu đa nhánh): xem `~/.claude/local.md`.

**Quy tắc commit (bắt buộc khi repo dùng git-cliff):**
- Mọi commit theo **Conventional Commits** (`feat`, `fix`, `docs`, `ci`, `chore`, `perf`, `refactor`, `test`, `style`, `build`, `sample`, `revert`...) — git-cliff phân loại/đánh số theo `type`; commit không-conventional bị `filter_unconventional` bỏ → lệch số & nhóm changelog (xem caveat đánh số bên dưới).
- **Sinh lại CHANGELOG.md trước khi commit** rồi mới `git add` + commit (flow chi tiết ở Track 1 / `Release.ps1`).
- Commit message viết bằng **tiếng Anh** (quy tắc chung ở `~/.claude/git.md`).

**Hai track TÁCH BIỆT — đừng trộn:**
- **CHANGELOG.md (local):** full timeline mọi version. Config `cliff.toml`, sinh bằng `Changelog.ps1`,
  cập nhật bằng `Release.ps1`. KHÔNG đụng CI.
- **GitHub /releases (CI):** 1 Release cho mỗi dòng minor, notes đã lọc + đánh số. Config riêng
  `cliff-release.toml`, workflow `.github/workflows/release.yml`.

### Cài đặt
- Local: `winget install git-cliff` / `cargo install git-cliff` / fallback `npx -y git-cliff@latest`.
- CI windows runner: tải binary `git-cliff-<ver>-x86_64-pc-windows-msvc.zip` từ GitHub releases của
  orhun/git-cliff → giải nén → `Add-Content $env:GITHUB_PATH <dir>`. (`orhun/git-cliff-action` chạy
  container Linux, KHÔNG hợp job windows.)

### git-cliff ↔ GitVersion: đánh số `M.N.<commits>`
- GitVersion (master `increment: None`) → version = `Major.Minor.CommitsSinceVersionSource` =
  `M.N.<số commit kể từ tag M.N.0>`. Muốn bump → bump MINOR của tag.
- **git-cliff KHÔNG biết version thật.** Khi range không chứa tag, nó TỰ "bump" đoán version (vd
  `1.1.0..HEAD` → đoán `1.2.0`) và gán vào `previous.version`/`version`. ĐỪNG tin các field này khi
  cần số đúng.
- **Trick `__MN__` + `loop.index` (CHỈ cho lịch sử TUYẾN TÍNH):** template emit
  `__MN__.{{ loop.index }}`, CI thay `__MN__` → `M.N`. Caveat: `loop.index` đếm commit
  **CONVENTIONAL** trong range — chỉ khớp GitVersion (đếm MỌI commit) khi mọi commit đều
  conventional VÀ không có merge. Repo đa nhánh merge qua lại → số lệch xa (ca thật: notes tới
  `.27` trong khi nupkg `4.0.94`) và merge thêm làm SHIFT số cũ → đừng dùng.
- **Trick `__SHA_<sha>__` + gom theo build `[release]` (đa nhánh — mặc định, repo mẫu ở `local.md`):**
  git-cliff CHỈ format từng commit, template emit `__SHA_{{ commit.id }}__ <nội dung>` (không đánh
  số, không cần thứ tự output). CI (PowerShell) parse thành map `sha→dòng` rồi tự dựng cấu trúc:
  điểm build R = các commit **first-parent** có marker `[release]` trong message (`git log -1
  --format=%B` -match — trùng điều kiện trigger CI) **+ HEAD của run hiện tại** (build đang phát
  hành; không trùng nếu HEAD chính là `[release]`). Mỗi R → header `- M.N.<zzz>` với
  `zzz = git rev-list --count tag..R` (= đúng version nupkg vì GitVersion đếm MỌI commit); mục con
  `  + <nội dung>` = `git rev-list --reverse R --not R_trước` — mọi commit mới trong `(R_trước..R]`,
  TỰ GỒM cả commit merge từ branch khác (vd fix trên 2.4 merge vào 4.0 nằm dưới header build 4.0
  đầu tiên chứa nó). Header rỗng (build chỉ có type bị lọc) thì BỎ. Bất biến: nupkg `M.N.K` chứa
  đúng các mục có header ≤ K; số ổn định vĩnh viễn, không shift khi merge thêm; cùng 1 commit hiện
  trên mỗi dòng release theo số riêng (2.4 → dưới `- 2.4.158`, 4.0 → dưới `- 4.0.97`). Không cần
  sort (fp walk tăng dần sẵn); `skip` trong commit_parsers không phá gì (tra cứu theo sha).
  Caveat: build dispatch không có `[release]` chỉ hiện là header ở đúng run đó (HEAD); lần regen
  sau nhóm của nó dồn vào build `[release]` kế tiếp.

### Track 1 — CHANGELOG.md (local): `cliff.toml` + `Changelog.ps1` + `Release.ps1`
- **`tag_pattern` phải khớp tag-prefix GitVersion.** Tag có `v` → `"v[0-9]*"`. Tag không `v` (`1.1.0`,
  prefix `[vV]?`) → literal-string TOML (nháy đơn, khỏi escape): `tag_pattern = '[vV]?[0-9]+\.[0-9]+\.[0-9]+'`.
  Sai pattern → group changelog vỡ.
- **`-Since` floor** trong Changelog.ps1: nhóm commit cũ nhất luôn để lại 1 mục RỖNG nếu lịch sử cũ
  không conventional → đặt floor ở tag mà nhóm cũ nhất CÓ conventional commit (vd `'1.0.1'`). `''` = full.
- **Format timeline đọc dưới-lên** (Wpf.Interop.DirectX): commit đánh số `M.N.<n>`, mới-nhất-trên
  (`sort_commits="newest"`), header tag `## [M.N.0]` ở ĐÁY mỗi nhóm (đặt `{% for %}` trước, header sau).
  Track local có thể suy số từ `previous.version` (luôn dùng range `floor..HEAD` nên số ổn).
- **Cập nhật CHANGELOG.md = `Release.ps1`:** regenerate → `git add CHANGELOG.md` → (với `-Message`) in
  tóm tắt → **hỏi `Press Enter to proceed (Ctrl+C to abort)`** → `git commit` (tự chèn `[release]` nếu
  thiếu) → `git push` (nếu `-Push`). Lý do gắn vào commit release: đó là điểm đồng bộ tự nhiên, không
  rải nhiễu CHANGELOG.md khắp lịch sử. Mọi cách "sinh rồi commit" **trễ đúng 1 commit** (file không liệt
  kê chính commit cập nhật nó) — bản chất, chấp nhận.

### Track 2 — GitHub /releases (CI): `release.yml` + `cliff-release.toml` + `BuildCI.ps1`
Mô hình: **1 Release / dòng minor**, tên theo tag `M.N.0`; mỗi build CI → `M.N.<N>.nupkg` upload làm
**asset TÍCH LUỸ** + notes **tăng dần, đã lọc**.
- **Trigger opt-in bằng marker:** `on: push: branches:[master]` + `workflow_dispatch`; job
  `if: github.event_name=='workflow_dispatch' || contains(github.event.head_commit.message,'[release]')`.
  → master mặc định SKIP, chỉ build khi commit head chứa `[release]`. (GitHub có sẵn `[skip ci]` nhưng là
  opt-OUT; muốn opt-IN phải dùng `if: contains`.) **Gotcha:** bất kỳ message chứa chuỗi `[release]` đều
  trigger — kể cả commit *mô tả về nó* (tránh để literal `[release]` trong message không-release).
- **Tiền điều kiện:** tag `M.N.0` phải tồn tại **TRÊN ORIGIN** trước (`git push origin M.N.0`) — GitVersion
  lấy làm mốc + tên Release. `gh release create M.N.0 --verify-tag` fail nếu tag chưa push.
- **`cliff-release.toml`** khác cliff.toml: `header=""`/`footer=""`, `sort_commits="oldest"`,
  body emit placeholder (repo tuyến tính: `__MN__`+`loop.index`, ĐỪNG `skip` vì phá loop.index;
  repo đa nhánh: `__SHA_<sha>__` — xem trick ở trên) + lọc loại hiển thị ở body:
  `{% if commit.group in ["feat","fix","perf","revert"] %}...{% endif %}`.
- **Sinh notes (CI):** `git-cliff --config cliff-release.toml "<M.N.0>..HEAD"` rồi xử lý placeholder
  (`__MN__` → `$M.$N`, hoặc dựng cấu trúc từ `__SHA_<sha>__` theo trick first-parent).
  **Dùng RANGE, KHÔNG `--latest`/`--unreleased`** (xem Bẫy). Range `tag..HEAD` TỰ GỒM cả commit
  của branch khác đã merge vào — đúng ý cho notes đa nhánh.
- **gh release:** `gh release view <tag> *>$null; if($LASTEXITCODE -eq 0){ gh release edit <tag> --notes-file }`
  `else{ gh release create <tag> --title <tag> --notes-file --verify-tag }`; rồi
  `gh release upload <tag> <nupkg> --clobber`.
- **nuget push có điều kiện:** `env: NUGET_KEY: ${{ secrets.nugetKey }}` rồi
  `if(-not [IO.Path]::... IsNullOrWhiteSpace($env:NUGET_KEY)){ dotnet nuget push ... --skip-duplicate }`.
  Không có secret → skip (an toàn, không lỡ publish thật).
- **`BuildCI.ps1`** đặt ở `.github/workflows/` (GitHub bỏ qua file non-yml ở đó): build native x64/Win32 +
  pack, KHÔNG push. Resolve src: `$root = (Resolve-Path (Join-Path $PSScriptRoot '..\..\src')).Path`.
  release.yml gọi `./.github/workflows/BuildCI.ps1`; nuget push do release.yml lo riêng. (KHÔNG truyền
  nugetKey vào BuildCI → khối push trong script tự skip, không `pause`.)

### push nuget với "Release Notes"
- nuget.org **chỉ surface `<readme>` và `<releaseNotes>`**. File `.md` pack vào `.nupkg` thì nằm trong
  package nhưng KHÔNG hiện ở UI → vô nghĩa cho mục đích "người dùng xem".
- **`<releaseNotes>` tạo TAB "Release Notes"** trên trang package (sau tab Versions). Nội dung hiển thị
  dạng **text + auto-link URL**, KHÔNG render markdown tin cậy (field cũ). MS chỉ để 1 URL → ra link bấm được.
- Nuspec token-based: `<releaseNotes>https://github.com/<owner>/<repo>/blob/master/CHANGELOG.md</releaseNotes>`
  (URL trần) → tab Release Notes trỏ changelog. Muốn per-version thì trỏ `/releases`.

### Bẫy đã gặp
- **`--latest`/`--unreleased` quét MỌI nhánh, bỏ topology HEAD.** Repo nhiều nhánh (vd có `dev`):
  `--latest` kéo commit nhánh khác vào theo NGÀY và bỏ qua range. → **luôn dùng range `<tag>..HEAD`**
  cho notes (đã verify: range tôn trọng topology HEAD). Range không chứa tag → git-cliff tự bump version
  → đừng dùng version đó, dùng `__MN__`.
- **PS 5.1 + redirect stderr native exe:** ĐỪNG `2>$null` lên git-cliff trong script/tool có
  `$ErrorActionPreference='Stop'` — PS bọc stderr (kể cả WARN vô hại) thành lỗi → script "fail" dù exit 0.
  Chạy thẳng, không redirect.
- **Encoding hiển thị:** git-cliff xuất UTF-8 đúng; console/`Get-Content` PS 5.1 đọc ANSI → mojibake
  `â€"` cho `—`. File vẫn đúng — kiểm bằng tool Read / `[IO.File]::ReadAllText($p,[Text.Encoding]::UTF8)`.
- **WARN `N commit(s) skipped`** = commit không-conventional bị bỏ, bình thường.
- Comment trong file config (.toml/.ps1/.yml) viết **ASCII không dấu** để né lỗi encoding (PS 5.1 ANSI /
  file BOM). Guide `.md` này thì dấu tiếng Việt OK.
- Workflow `gh run watch <id> --exit-status` để chờ + biết pass/fail; push run không có marker hiện
  `completed/skipped` (đúng).
