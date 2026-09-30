## GitVersion config notes (v6.x)

- Tag bắt buộc full SemVer `X.Y.Z` (vd `v1.0.0`). Tag thiếu (`v1.0`) bị **âm thầm bỏ qua**: GitVersion fallback về `0.0.1` và `CommitsSinceVersionSource` = tổng số commit từ gốc repo. Không có warning.
- `mode: ContinuousDeployment` + `increment: Patch` chỉ +1 cho patch (không phải = số commit). Vd tag `v1.0.0` + 26 commit → `MajorMinorPatch = 1.0.1`, không phải `1.0.26`.

### Pattern MAJOR.MINOR.<commits-since-tag>

`GitVersion.yml`:
```yaml
mode: ManualDeployment
tag-prefix: 'v'
branches:
  master:
    regex: ^master$
    increment: None
  main:
    regex: ^main$
    increment: None
```

MSBuild (`.targets` / `.csproj`):
```xml
<Version>$(GitVersion_Major).$(GitVersion_Minor).$(GitVersion_CommitsSinceVersionSource)</Version>
```

Lấy Major/Minor trực tiếp từ tag (`increment: None` giữ nguyên), patch = số commit kể từ tag → ra 3-segment `1.0.N`.

### Debug GitVersion

`dotnet tool install -g GitVersion.Tool` rồi chạy `dotnet-gitversion` ở root repo. Check `VersionSourceSemVer` và `VersionSourceSha`: nếu rỗng/`0.0.0` thì tag không được nhận.

### Nhánh phụ build đỏ: "An orphaned branch ... has been detected" → "No base versions determined"

Triệu chứng (một repo thư viện, 2026-09-14, GitVersion.MsBuild 6.8.2): `master` build 0 lỗi, nhưng
**mọi nhánh khác** đỏ `error MSB3073: ... gitversion.dll ... exited with code 1`. Chạy trực tiếp
`gitversion.dll <đường dẫn project>` mới thấy lý do thật:

```
An orphaned branch 'feature/xxx' has been detected and will be skipped=True.
GitVersion.GitVersionException: No base versions determined on the current branch.
```

GitVersion cần một **base version**, lấy từ một trong hai đường — và ở ca này CẢ HAI đều tắt:

1. **Tag** — repo chỉ có tag `v1.1`. Không phải full SemVer `X.Y.Z` nên bị **âm thầm bỏ qua**
   (xem mục đầu file này). Không có tag hợp lệ ⇒ không có base version từ tag.
2. **Nhánh nguồn** — GitVersion tìm nhánh mà nhánh hiện tại **phân kỳ khỏi**. Nếu nhánh phụ chỉ
   **đi thẳng phía trước** `master` trên cùng một đường (master là tổ tiên trực tiếp, master không
   có commit riêng nào sau điểm chẻ) thì không có điểm phân kỳ ⇒ GitVersion không nhận ra master là
   nhánh nguồn ⇒ coi nhánh là **mồ côi**.

Ở `master` thì không cần nhánh nguồn (master có config riêng trong `branches:`) nên vẫn chạy — đó là
lý do lỗi chỉ lộ khi sang nhánh phụ.

**Cách sửa gốc: tag lại cho đúng full SemVer** (`v1.1` → `v1.1.0`). Có tag hợp lệ là đường (1) hoạt
động và nhánh phụ build được, không cần chạm `branches:`.

**Những cách KHÔNG sửa được** (đã thử, vẫn "orphaned"):
- Bỏ dòng `workflow: GitFlow/v1`.
- Đổi tên nhánh cho khớp GitFlow (`build/x` → `feature/x`).
- Thêm mục `feature:` với `regex: ^.+$` và `source-branches: [master, main]` vào `branches:`.

**Bài học quy trình:** repo dùng GitVersion mà chỉ có tag sai định dạng thì **không build được trên
nhánh phụ**. Trước khi tự tạo nhánh trong một repo lạ có `GitVersion.yml`, kiểm `git tag` xem tag có
đủ 3 segment không — nếu không thì hoặc sửa tag trước, hoặc làm thẳng trên nhánh chính. Đừng kết
luận "thay đổi của mình làm vỡ build": đối chiếu bằng cách build lại trên nhánh chính.

### Nhánh phụ "orphaned" NGAY CẢ KHI tag đã đúng full SemVer

Ca thứ hai (repo thư viện B, 2026-09-20, GitVersion.MsBuild 6.7.0): repo có tag **`v1.0.0` đúng
định dạng** và `git merge-base --is-ancestor v1.0.0 HEAD` xác nhận tag LÀ tổ tiên của HEAD, vậy mà
nhánh phụ vẫn đỏ y hệt:

```
Finding branches source of 'fix/log-file-write-off-ui-thread'
An orphaned branch 'fix/...' has been detected and will be skipped=True.
GitVersion.GitVersionException: No base versions determined on the current branch.
```

⇒ **Sửa tag KHÔNG phải lúc nào cũng đủ.** Khi GitVersion kết luận nhánh là orphaned, nó bỏ qua cả
đường base-version từ tag chứ không chỉ đường nhánh-nguồn. Điều kiện gây ra: nhánh phụ tạo từ
`master` và **master không có commit riêng nào sau điểm chẻ** ⇒ không có điểm phân kỳ ⇒ orphaned.

**Cách đi tiếp khi chỉ cần BUILD để verify code trên nhánh phụ** (không pack):

```
dotnet build <sln> -c Debug -p:DisableGitVersionTask=true
```

`DisableGitVersionTask` là property của GitVersion.MsBuild, tắt hẳn task nên không cần chạm
`GitVersion.yml` hay đặt tên nhánh theo GitFlow. Khi pack thật thì đã merge về `master` (master có
config riêng trong `branches:` nên không cần nhánh nguồn) ⇒ chạy bình thường.

Lưu ý: cùng một máy, cùng kiểu nhánh `fix/*`, repo thư viện A build nhánh phụ KHÔNG lỗi còn
repo thư viện B thì lỗi — khác nhau ở chỗ master của repo kia có commit riêng sau điểm chẻ. Đừng
suy từ "repo A chạy được" ra "repo B phải chạy được".
