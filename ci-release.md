# Phát hành ứng dụng qua CI (GitHub Actions + tag)

Áp dụng cho mọi bản phát hành lên GitHub Releases: ỨNG DỤNG (zip/installer) và THƯ VIỆN NuGet
(nupkg) đều theo cùng mô hình tag dưới đây. Phần riêng của NuGet (target version, push nuget.org):
xem mục "Thư viện NuGet" cuối file và `~/.claude/BuildNuggetCsharpLibrary.md`.

## Mô hình chuẩn: gắn tag thủ công, CI build theo tag

- **Mỗi bản phát hành = một tag `vM.N.P` = một GitHub Release cùng tên.** Không dồn zip của nhiều
  bản vào một Release chung.
- **Người phát hành tự chọn số và tự gắn tag**, CI không tự đếm hay tự tạo tag:
  ```
  git tag v1.2.3            # trên commit muốn phát hành (thường là đầu master)
  git push origin v1.2.3    # chính lệnh này kích hoạt CI
  ```
  Gắn và push tag là việc của user: agent chỉ gắn/push tag khi được yêu cầu rõ ràng cho từng lần
  (luật push ở `~/.claude/git.md`).
- **Không dùng marker trong commit message** (kiểu `[release]`) để kích hoạt build, cũng không dùng
  "tag gốc vM.N.0 + đếm commit" để ra số. Hai cách đó làm tag vừa là mốc tính số vừa là tên Release,
  số phát hành phụ thuộc lịch sử commit và khó sửa khi CI lỗi giữa chừng.

## Workflow

```yaml
on:
  push:
    tags:
      - 'v[0-9]+.[0-9]+.[0-9]+'
  workflow_dispatch:          # chạy lại một bản: chọn ref là TAG, không phải nhánh

jobs:
  release:
    if: github.ref_type == 'tag'
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0      # cần lịch sử + tag để tính version và release notes
```

- Version của bản build lấy từ tag (vd `git describe --tags --long --match "v[0-9]*.[0-9]*.[0-9]*"`
  trong MSBuild): commit mang tag ra đúng `M.N.P`; commit sau tag (build dev ở máy) ra
  `M.N.P-dev.<số commit kể từ tag>` để không trùng số bản đã phát hành.
- Trong CI **kiểm version build ra khớp tên tag** (`$env:GITHUB_REF_NAME`), lệch thì fail. SDK .NET
  có thể nối `+<sha>` vào `ProductVersion`: cắt phần sau `+` trước khi so (hoặc tắt hẳn, xem
  `~/.claude/releaseCsharp.md`).
- Release notes: git-cliff trên khoảng `<tag trước>..<tag này>`; tag trước lấy bằng
  `git describe --tags --abbrev=0 --match "v[0-9]*.[0-9]*.[0-9]*" "<tag>^"` (tag đầu tiên thì lấy
  cả lịch sử). Cách cấu hình git-cliff ở `~/.claude/git-cliff.md`.
- Test tách thành workflow riêng (`test.yml`: push master, pull_request, `workflow_call`); `release.yml` gọi nó bằng job `test: uses: ./.github/workflows/test.yml` và job build `needs: test`. Test chạy ở mỗi push nên lỗi lộ trước khi gắn tag, và danh sách bài loại trừ chỉ nằm một chỗ.
- Tạo Release: `gh release view` có rồi thì `gh release edit`, chưa có thì
  `gh release create <tag> --verify-tag`; upload `gh release upload <tag> <files> --clobber` để chạy
  lại không lỗi. Workflow cần `permissions: contents: write`.

## Bẫy

- **`git push` nhánh KHÔNG đẩy tag.** Push master xong CI chưa chạy; phải push tag riêng.
  `git push --follow-tags` chỉ đẩy tag annotated (`git tag -a`), không đẩy tag thường.
- **Push hơn 3 tag trong một lần thì GitHub không tạo sự kiện push** ⇒ workflow không chạy. Push từng tag.
- **Tag do workflow tạo bằng `GITHUB_TOKEN` không kích hoạt workflow khác.** Không dựng chuỗi
  "workflow A gắn tag → workflow B build" bằng token mặc định.
- **`workflow_dispatch` chạy từ nhánh thì không phải tag** ⇒ job nên `if: github.ref_type == 'tag'`.
- **CI build lỗi**: sửa code rồi phát hành số mới (vd `v1.2.4`) là sạch nhất. Muốn giữ số thì xoá tag
  (`git push origin :v1.2.3`, `git tag -d v1.2.3`), gắn lại rồi push lại; không xoá/gắn lại tag mà
  Release đã có người tải.
- **Lệnh native trong `pwsh` không ném lỗi theo exit code** dù `$ErrorActionPreference = 'Stop'`:
  kiểm `$LASTEXITCODE` sau mỗi `dotnet`/`gh`/`git` quan trọng.
- **Đổi từ mô hình cũ sang mô hình tag**: số các bản đã phát hành theo cách cũ vẫn tính. Tag mới phải
  lớn hơn số cao nhất đã phát hành, kể cả số chỉ nằm trong tên file zip của Release cũ.
  Release notes của tag mới đầu tiên tính từ tag `vM.N.P` cũ gần nhất, nên gồm cả các bản đã phát hành
  mà không có tag riêng. Muốn notes gọn thì gắn bù tag lên đúng commit của các bản đó (commit cũ còn
  `release.yml` kiểu cũ nên push tag bù không kích hoạt build), rồi chạy lại workflow trên tag mới.
- **Bật test ở mỗi push dễ lộ test có race** vốn hiếm khi đỏ ở máy dev (runner nhanh/chậm khác thứ tự
  luồng). Sửa test cho tất định, đừng thêm vào danh sách loại trừ.

## Thư viện NuGet

- Version = đúng tag qua GitVersion (`~/.claude/gitversion.md`, target ở
  `~/.claude/BuildNuggetCsharpLibrary.md` mục 1): commit mang tag ra `M.N.P`, sau tag ra
  `M.N.P-dev.<n>`. CI kiểm version của nupkg == tên tag bỏ `v`.
- Chỉ publish khi commit của tag nằm trên master:
  `git merge-base --is-ancestor $GITHUB_SHA origin/master` (lệch thì fail, không push).
- `dotnet nuget push --skip-duplicate` chỉ chạy khi có secret `NUGETKEY`; không có thì bỏ qua.
- **Tránh downgrade**: tag mới phải lớn hơn version cao nhất đã có trên nuget.org.
