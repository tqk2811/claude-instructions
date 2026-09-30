# Android SDK / NDK

- **Đường dẫn thật trên máy** (thư mục gốc Android, SDK, NDK, `adb`, JDK): xem `~/.claude/local.md`.
  Dưới đây gọi thư mục gốc là `<ANDROID_ROOT>`.
  - **SDK**: `<ANDROID_ROOT>\sdk` — dùng làm `ANDROID_HOME` (ưu tiên) và `ANDROID_SDK_ROOT`.
  - **NDK**: `<ANDROID_ROOT>\ndk\<ver>` (khi cần build native).
- Cài kiểu "giải nén thư mục" (KHÔNG dùng Android Studio installer): SDK/NDK chỉ là file, không đụng registry.
- `adb` có thể là bản standalone nằm ngoài SDK (vị trí ở `local.md`).

## Layout chuẩn dưới `<ANDROID_ROOT>`
```
<ANDROID_ROOT>\
  sdk\
    platforms\android-<API>\android.jar    # bootclasspath khi javac (class android.*)
    build-tools\<ver>\d8.bat                # .class -> classes.dex (app_process chạy dex)
    cmdline-tools\latest\bin\sdkmanager.bat # (tùy chọn)
    platform-tools\                         # (tùy chọn; adb có thể dùng bản riêng)
  ndk\<ver>\                                # NDK (khi build native)
```

## Biến môi trường
- Đặt `ANDROID_HOME=<ANDROID_ROOT>\sdk` và `ANDROID_SDK_ROOT=<ANDROID_ROOT>\sdk`.
- JDK thật (thư mục cài có `javac` + `jar.exe`, xem `local.md`). Lưu ý `jar` KHÔNG có trong
  Oracle javapath shim (`C:\Program Files\Common Files\Oracle\Java\javapath`) — khi build cần `jar`/`d8`
  thì thêm `<JDK>\bin` vào PATH cho phiên đó.

## Tải trực tiếp (download + giải nén, không sdkmanager)
- **build-tools** (có `d8`): URL ổn định `https://dl.google.com/android/repository/build-tools_r34-windows.zip`
  → giải nén ra thư mục `android-<codename>` → đổi tên/di chuyển thành `<ANDROID_ROOT>\sdk\build-tools\34.0.0\`.
- **platform** (`android.jar`): gói `platform-<API>_r<rev>.zip` trên `dl.google.com/android/repository/`
  — số `<rev>` đổi theo thời gian, HEAD-check trước khi tải; giải nén (thư mục `android-<codename>`) →
  đặt thành `<ANDROID_ROOT>\sdk\platforms\android-<API>\`. API biên dịch không cần trùng API thiết bị.

## Build server `app_process` (tham chiếu; project mẫu ghi ở `local.md`)
- `server/build.bat` (Windows) cần: `ANDROID_HOME` + `platforms\android-<API>\android.jar`
  + `build-tools\<ver>\d8.bat` + `jar` trong PATH. API mặc định `34`, đổi bằng biến `API`.
- Chuỗi build: `javac` (`-classpath` android.jar, KHÔNG dùng `-bootclasspath`) → `d8` (.class → classes.dex)
  → `jar` (đóng gói) → `server\build\<name>-server.jar`.
- Lưu ý JDK mới (vd JDK 25): phải để android.jar ở `-classpath`, không `-bootclasspath`, nếu không lambda
  desugar lỗi `Unable to find method metafactory` (stub `LambdaMetafactory` trong android.jar thiếu method).
