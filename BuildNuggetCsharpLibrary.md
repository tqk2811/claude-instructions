# NuGet packaging notes

Áp dụng cho các thư viện riêng của user (danh sách ở `~/.claude/local.md`; đóng gói bằng `dotnet pack` + `.nuspec`, version bằng GitVersion). Cấu hình chung đặt ở `ProjectBuildProperties.targets` (root repo), mỗi packable csproj `<Import Project="..\ProjectBuildProperties.targets" />`.

## 1. Version scheme (tag `vMAJOR.MINOR.0`)

- `AssemblyVersion = Major.Minor.0.0` (cố định theo minor → không vỡ binding redirect)
- `FileVersion     = Major.Minor.<commits kể từ tag>.0`
- `PackageVersion  = Major.Minor.<commits kể từ tag>` (= NuGet version)

Set bằng target chạy **sau** GitVersion:
```xml
<Target Name="SetVersionFromGitVersion" AfterTargets="GetVersion"
        Condition="'$(EnableGitVersion)' == 'true' and '$(DisableGitVersionTask)' != 'true'">
  <PropertyGroup>
    <Version>$(GitVersion_Major).$(GitVersion_Minor).$(GitVersion_CommitsSinceVersionSource)</Version>
    <PackageVersion>$(Version)</PackageVersion>
    <AssemblyVersion>$(GitVersion_Major).$(GitVersion_Minor).0.0</AssemblyVersion>
    <FileVersion>$(GitVersion_Major).$(GitVersion_Minor).$(GitVersion_CommitsSinceVersionSource).0</FileVersion>
    <InformationalVersion>$(Version)</InformationalVersion>
    <NuspecProperties>id=$(MSBuildProjectName);version=$(Version)</NuspecProperties>
  </PropertyGroup>
</Target>
```
- Cấu hình `GitVersion.yml` (ManualDeployment + `increment: None`): xem `~/.claude/gitversion.md`. Patch của output = số commit kể từ tag, **không** lấy patch của tag → muốn bump version phải bump MINOR của tag.
- **Tag tại đúng commit cần phát hành** (commits=0 → version `M.m.0`). Nếu tạo commit mới sau tag thì version +1 → khi sửa file đã commit-versioning, dùng `git commit --amend` + `git tag -f` (commit chưa push) để giữ version, KHÔNG tạo commit mới.
- **Tránh downgrade**: trước khi đặt tag, check version đã publish trên nuget.org (`https://api.nuget.org/v3-flatcontainer/<id-lower>/index.json`). Bản `1.0.0-build<date>` là prerelease → stable `1.0.0` lớn hơn (OK). Nhưng nếu đã có stable cao hơn (vd `1.0.2`) thì tag `v1.0.0` sẽ ra package thấp hơn → phải bump minor (vd `v1.1.0`).

## 2. `dotnet pack` + nuspec

- Bật dùng nuspec: `<NuspecFile>$(MSBuildProjectName).nuspec</NuspecFile>`; truyền token qua `NuspecProperties` (`$id$`, `$version$`).
- **`<releaseNotes>` → tab "Release Notes" trên nuget.org** (chỉ field này + `<readme>` được surface; file `.md` pack vào package KHÔNG hiển thị ở UI). Để URL trần trỏ `CHANGELOG.md`/`/releases`. Sinh changelog + CI publish lên GitHub /releases: xem `~/.claude/git-cliff.md`.
- Glob trong `<files>` tính tương đối từ **thư mục project** (giống `nuget pack` cũ).
- **Folder convention NuGet/VS chỉ hiểu**: `lib/<tfm>/` (reference dll + IntelliSense xml), `ref/`, `contentFiles/`, `build/`, `buildTransitive/`, `analyzers/`, `tools/`, `runtimes/`, `native/`.
- **`src/` KHÔNG phải convention** → để `.cs` vào `src/` thì Visual Studio **không** dùng (không compile, không reference, không hiện trong solution). Muốn xem source khi debug thì dùng SourceLink / nhúng source vào PDB, **không** dùng `src/`. → Đã bỏ dòng `<file src="**\*.cs" target="src\"/>` khỏi nuspec.

## 3. XML doc (`GenerateDocumentationFile`)

- File `.xml` đặt cạnh dll cùng tên trong `lib/<tfm>/` → **VS dùng cho IntelliSense** (tooltip/Quick Info mô tả class/method/param). Khác hẳn `src/`.
- Chỉ sinh khi `<GenerateDocumentationFile>true</GenerateDocumentationFile>`; không bật → không có `.xml` → dòng pack `.xml` trong nuspec không match (im lặng) → package thiếu doc IntelliSense.
- Bật ở targets chung cho mọi packable project. Kèm `<NoWarn>$(NoWarn);CS1591</NoWarn>` để tắt cảnh báo CS1591 (public member thiếu `///` comment) — vẫn xuất `.xml`, chỉ bớt spam warning.

## 4. Source & PDB

- `<EmbedAllSources>true</EmbedAllSources>` → nhúng toàn bộ text `.cs` vào **PDB** (nén).
- `DebugType`:
  - `embedded` → PDB (kèm source) nhúng **vào trong dll** → 1 file duy nhất, không có `.pdb` riêng, không cần `src/`. Nhược: dll to hơn; **ai có dll là trích ngược được source** → chỉ dùng cho lib công khai.
  - `portable` (mặc định) → có file `.pdb` riêng (chứa source nếu bật EmbedAllSources). Muốn consumer step-into offline thì **pack `.pdb` vào `lib/<tfm>/`** qua nuspec: `<file src="bin\Release\**\$id$.pdb" target="lib\" />`. dll sạch (không chứa source).
- **Tránh lộ local path trong PDB**: `<Deterministic>true</Deterministic>` + `<ContinuousIntegrationBuild>true</ContinuousIntegrationBuild>` (+ `<PathMap>$(MSBuildProjectDirectory)=$(MSBuildProjectName)</PathMap>`) → đường dẫn source thành dạng `ProjectName\File.cs`, không phải `D:\...`.
- SourceLink: SDK tự bật khi CI build + remote GitHub → PDB nhúng URL `raw.githubusercontent.com/...` (chỉ hoạt động sau khi **đã push** commit/tag). PDB nhúng source là fallback offline, không phụ thuộc push.
- **Hiệu suất consumer**: nhúng source KHÔNG ảnh hưởng build/runtime của project dùng thư viện — consumer chỉ reference dll đã build sẵn, không recompile source nhúng (source chỉ được debugger đọc khi step-into). Chỉ làm dll/pdb lớn hơn chút.

## 5. Verify nhanh

- Liệt kê nupkg: mở `.nupkg` như zip (`System.IO.Compression.ZipFile`), check `lib/<tfm>/{dll,pdb,xml}`, không còn `src/`.
- Đọc PDB (cần `System.Reflection.Metadata`, dùng app net8 nếu PS 5.1 thiếu): PE debug directory có entry `EmbeddedPortablePdb` (embedded) hay `CodeView` (pdb riêng); document có embedded source = `CustomDebugInformation` kind GUID `0E8A571B-6926-466E-B4AD-8AB04611F5FE`.

## 5b. Push nuget.org — quota & bẫy wildcard (bài học 2026-08-06, khi push hàng loạt)

- **nuget.org có QUOTA push theo cửa sổ thời gian.** Push liên tục nhiều gói (quan sát: **~448 gói trong một đợt**) thì bắt đầu bị chặn: server trả **`403 (Quota Exceeded)`** kèm header **`retry after: <giây>`** (quan sát ~1327s ≈ 22 phút). KHÔNG phải lỗi gói — chỉ cần **chờ hết `retry-after` rồi push lại** phần còn lại (`-SkipDuplicate` để bỏ qua gói đã lên). Với mẻ lớn nên **giãn nhịp** (Start-Sleep 1-2s giữa mỗi push) để đỡ chạm quota.
- **`nuget push ".\Packages\*.nupkg"` (wildcard) DỪNG NGAY ở gói lỗi ĐẦU TIÊN** — các gói sau KHÔNG được thử. Một timeout/403 lẻ là mất phần đuôi. Muốn robust: **lặp từng file** trong PowerShell, `continue-on-error`, kiểm `$LASTEXITCODE` mỗi gói, gom danh sách fail rồi retry — đừng dựa vào wildcard cho mẻ lớn.
- **`-SkipDuplicate`** chỉ nuốt **409 Conflict** (gói+version đã tồn tại → coi như OK, exit 0). **403 Quota / timeout vẫn là exit 1** → phải tự đếm fail và retry.
- **Timeout upload**: gói Native lớn (~50-100MB) có gói mất 90s+; đặt `-Timeout <giây>` (mặc định 300s). "A task was canceled / Pushing took too long" = client bỏ cuộc chờ — gói có thể đã lên server hay chưa, nhưng `-SkipDuplicate` khi push lại xử lý được cả 2.
- **Bẫy PS + native exe**: `& nuget.exe ... 2>&1 | Out-Null` trong PS 5.1 làm `$?`=false dù exit 0 — LUÔN dùng **`$LASTEXITCODE`** (mã thật của exe) để phán đoán, đừng dùng `$?`.
- Chạy mẻ push dài bằng **background task** (không block), ghi log per-gói để đếm `OK/FAIL` và biết chính xác gói nào còn thiếu.

## 6. Thư viện có native C++ DLL (vd AudioCapture, AudioPlayer.*, WindowCapture, Wpf.Interop.DirectX)

Cấu trúc: 1 managed csproj (multi-TFM) + 1 `*.Native.vcxproj` (DynamicLibrary, build x64 + **Win32** cho 32-bit, OutDir `$(SolutionDir)x64|x86\Release\`) + `Build.ps1` riêng (KHÔNG dùng CsharpNugetPush). nuspec pack managed → `lib/<tfm>/`, native dll → `runtimes/win-x64|x86/native/`.

### Đánh version cho native dll (resource `.rc`)
Native dll không có version từ MSBuild — phải qua **resource `VS_VERSION_INFO`**:
- `version.rc`: `#include "version.generated.h"` + khối `VS_VERSION_INFO` (`FILEVERSION`/`PRODUCTVERSION` từ macro) + `StringFileInfo` (FileVersion, ProductVersion, CompanyName, OriginalFilename...). **`.rc` và `.h` phải có newline cuối** nếu không `rc.exe` báo `RC1004 unexpected end of file`.
- `version.generated.h`: định nghĩa `VER_FILE_MAJOR/MINOR/BUILD`, `VER_FILEVERSION_STR`, `VER_PRODUCTVERSION_STR`. FileVersion = `Major.Minor.<commits>.0`, ProductVersion = `Major.Minor.<commits>` (khớp managed/package).
- vcxproj: thêm `<ResourceCompile Include="version.rc" />` + `<ClInclude Include="version.generated.h" />`.

### version.generated.h tự sinh + git sạch
- **Git-ignore** `version.generated.h` (`git rm --cached` + thêm `.gitignore`) → working tree không bao giờ bẩn.
- vcxproj có **target tự tạo fallback `0.0.0` khi thiếu** (clean checkout / build IDE vẫn compile):
  ```xml
  <Target Name="EnsureVersionHeader" BeforeTargets="ResourceCompile">
    <WriteLinesToFile Condition="!Exists('$(MSBuildProjectDirectory)\version.generated.h')"
        File="$(MSBuildProjectDirectory)\version.generated.h" Overwrite="true"
        Lines="#pragma once;#define VER_FILE_MAJOR    0;...;#define VER_PRODUCTVERSION_STR &quot;0.0.0&quot;" />
  </Target>
  ```
- `Build.ps1` ghi version **thật** vào header trước khi build (chỉ ghi khi file thiếu → target không đè giá trị thật của script).

### Build.ps1 robust mọi máy (vswhere + MSBuild, KHÔNG devenv)
- Tìm MSBuild qua **vswhere** (path cố định `${ProgramFiles(x86)}\Microsoft Visual Studio\Installer\vswhere.exe`): `& $vswhere -latest -requires Microsoft.Component.MSBuild -find 'MSBuild\**\Bin\MSBuild.exe'`.
- Lấy version: `dotnet-gitversion /output json | ConvertFrom-Json` (auto `dotnet tool install -g GitVersion.Tool` nếu thiếu) → Major/Minor/CommitsSinceVersionSource.
- Ghi `version.generated.h` (nhớ **newline cuối**).
- Build native: `& $msbuild Native.vcxproj /t:Rebuild /p:Configuration=Release /p:Platform=x64 /p:SolutionDir="$repo\"` và `Platform=Win32` (build vcxproj trực tiếp cần `/p:SolutionDir` để OutDir đúng).
- `dotnet pack` managed (version từ GitVersion target).

### Verify native
`[Diagnostics.FileVersionInfo]::GetVersionInfo("x64\Release\*.Native.dll").FileVersion` = `Major.Minor.<commits>.0`. nupkg có `runtimes/win-x64|x86/native/*.Native.dll`.
