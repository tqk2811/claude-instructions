## git-cliff — changelog + GitHub release notes (đi với GitVersion)

git-cliff lo **changelog/timeline**; **version** do GitVersion lo (`~/.claude/gitversion.md`).
Repo mẫu: xem `~/.claude/local.md`.

**Quy tắc commit (bắt buộc khi repo dùng git-cliff):**
- Mọi commit theo **Conventional Commits** (`feat`, `fix`, `docs`, `ci`, `chore`, `perf`, `refactor`, `test`, `style`, `build`, `sample`, `revert`...) — git-cliff phân loại theo `type`; commit không-conventional bị `filter_unconventional` bỏ → lệch nhóm changelog.
- **Sinh lại CHANGELOG.md trước khi commit** rồi mới `git add` + commit (flow ở Track 1 / `Release.ps1`).
- Commit message viết bằng **tiếng Anh** (quy tắc chung ở `~/.claude/git.md`).

**Hai track TÁCH BIỆT — đừng trộn:**
- **CHANGELOG.md (local):** full timeline mọi version. Config `cliff.toml`, sinh bằng `Changelog.ps1`,
  cập nhật bằng `Release.ps1`. KHÔNG đụng CI.
- **GitHub /releases (CI):** 1 Release cho mỗi tag `vM.N.P`, notes đã lọc. Config riêng
  `cliff-release.toml`, workflow `.github/workflows/release.yml`.

### Cài đặt
- Local: `winget install git-cliff` / `cargo install git-cliff` / fallback `npx -y git-cliff@latest`.
- CI windows runner: tải binary `git-cliff-<ver>-x86_64-pc-windows-msvc.zip` từ GitHub releases của
  orhun/git-cliff → giải nén → `Add-Content $env:GITHUB_PATH <dir>`. (`orhun/git-cliff-action` chạy
  container Linux, KHÔNG hợp job windows.)

### git-cliff ↔ version
- Version = đúng tag `vM.N.P` (mô hình tag, `~/.claude/ci-release.md`); mỗi tag = một Release.
- **git-cliff KHÔNG biết version thật.** Khi range không chứa tag, nó TỰ "bump" đoán version (vd
  `1.1.0..HEAD` → đoán `1.2.0`) và gán vào `previous.version`/`version`. ĐỪNG tin các field này khi
  cần số đúng.

### Track 1 — CHANGELOG.md (local): `cliff.toml` + `Changelog.ps1` + `Release.ps1`
- **`tag_pattern` phải khớp tag-prefix GitVersion.** Tag có `v` → `"v[0-9]*"`. Tag không `v` (`1.1.0`,
  prefix `[vV]?`) → literal-string TOML (nháy đơn, khỏi escape): `tag_pattern = '[vV]?[0-9]+\.[0-9]+\.[0-9]+'`.
  Sai pattern → group changelog vỡ.
- **`-Since` floor** trong Changelog.ps1: nhóm commit cũ nhất luôn để lại 1 mục RỖNG nếu lịch sử cũ
  không conventional → đặt floor ở tag mà nhóm cũ nhất CÓ conventional commit (vd `'1.0.1'`). `''` = full.
- **Format timeline** (mẫu ở `local.md`): mới-nhất-trên (`sort_commits="newest"`), mỗi nhóm một
  header tag `## [M.N.P]`.
- **Cập nhật CHANGELOG.md = `Release.ps1`:** regenerate → `git add CHANGELOG.md` → (với `-Message`) in
  tóm tắt → **hỏi `Press Enter to proceed (Ctrl+C to abort)`** → `git commit` → `git push` (nếu `-Push`).
  Gắn tag là bước riêng của user sau đó. Mọi cách "sinh rồi commit" **trễ đúng 1 commit** (file không
  liệt kê chính commit cập nhật nó) — bản chất, chấp nhận.

### Track 2 — GitHub /releases (CI): `release.yml` + `cliff-release.toml`
Mô hình tag (`~/.claude/ci-release.md`): mỗi tag `vM.N.P` = một Release cùng tên, notes chỉ gồm
commit của bản đó.
- **`cliff-release.toml`** khác cliff.toml: `header=""`/`footer=""`, `sort_commits="oldest"`, lọc loại
  hiển thị ở body: `{% if commit.group in ["feat","fix","perf","revert"] %}...{% endif %}`.
- **Sinh notes (CI):** `git-cliff --config cliff-release.toml "<tag trước>..<tag>"`; tag trước =
  `git describe --tags --abbrev=0 --match "v[0-9]*.[0-9]*.[0-9]*" "<tag>^"` (tag đầu tiên: cả lịch sử).
  **Dùng RANGE, KHÔNG `--latest`/`--unreleased`** (xem Bẫy).
- **gh release:** `gh release view <tag> *>$null; if($LASTEXITCODE -eq 0){ gh release edit <tag> --notes-file }`
  `else{ gh release create <tag> --title <tag> --notes-file --verify-tag }`; rồi
  `gh release upload <tag> <nupkg> --clobber`.
- **nuget push có điều kiện:** `env: NUGET_KEY: ${{ secrets.NUGETKEY }}` (tên secret thống nhất: `NUGETKEY`) rồi
  `if(-not [string]::IsNullOrWhiteSpace($env:NUGET_KEY)){ dotnet nuget push ... --skip-duplicate }`.
  Không có secret → skip (an toàn, không lỡ publish thật).

### push nuget với "Release Notes"
- nuget.org **chỉ surface `<readme>` và `<releaseNotes>`**. File `.md` pack vào `.nupkg` thì nằm trong
  package nhưng KHÔNG hiện ở UI → vô nghĩa cho mục đích "người dùng xem".
- **`<releaseNotes>` tạo TAB "Release Notes"** trên trang package (sau tab Versions). Nội dung hiển thị
  dạng **text + auto-link URL**, KHÔNG render markdown tin cậy (field cũ). MS chỉ để 1 URL → ra link bấm được.
- Nuspec token-based: `<releaseNotes>https://github.com/<owner>/<repo>/blob/master/CHANGELOG.md</releaseNotes>`
  (URL trần) → tab Release Notes trỏ changelog. Muốn per-version thì trỏ `/releases`.

### Bẫy đã gặp
- **`--latest`/`--unreleased` quét MỌI nhánh, bỏ topology HEAD.** Repo nhiều nhánh (vd có `dev`):
  `--latest` kéo commit nhánh khác vào theo NGÀY và bỏ qua range. → **luôn dùng range `<tag trước>..<tag>`**
  cho notes (đã verify: range tôn trọng topology HEAD). Range không chứa tag → git-cliff tự bump version
  → đừng dùng version đó, lấy version từ tag.
- **PS 5.1 + redirect stderr native exe:** ĐỪNG `2>$null` lên git-cliff trong script/tool có
  `$ErrorActionPreference='Stop'` — PS bọc stderr (kể cả WARN vô hại) thành lỗi → script "fail" dù exit 0.
  Chạy thẳng, không redirect.
- **Encoding hiển thị:** git-cliff xuất UTF-8 đúng; console/`Get-Content` PS 5.1 đọc ANSI → mojibake
  `â€"` cho `—`. File vẫn đúng — kiểm bằng tool Read / `[IO.File]::ReadAllText($p,[Text.Encoding]::UTF8)`.
- **WARN `N commit(s) skipped`** = commit không-conventional bị bỏ, bình thường.
- Comment trong file config (.toml/.ps1/.yml) viết **ASCII không dấu** để né lỗi encoding (PS 5.1 ANSI /
  file BOM). Guide `.md` này thì dấu tiếng Việt OK.
- Workflow `gh run watch <id> --exit-status` để chờ + biết pass/fail; run bị job `if` loại hiện
  `completed/skipped` (đúng).
