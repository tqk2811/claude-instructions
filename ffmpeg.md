# Làm việc với ffmpeg trong code C#

Tên thật của các thư viện riêng nhắc ở đây (thư viện dựng filter graph, thư viện transition) ghi ở `~/.claude/local.md`, mục "ffmpeg".

## Luật bắt buộc

- **Không tự viết chuỗi lệnh / chuỗi filter ffmpeg** (`-filter_complex "..."`, ghép `string` các filter `scale=`, `overlay=`, `xfade=`…).
  Dựng filter graph bằng **thư viện dựng filter graph riêng của user** (object graph: input → filter → output, có kiểm kiểu).
  - Filter nào thư viện chưa có thì báo user (bổ sung vào thư viện là việc của thư viện), KHÔNG tự chèn chuỗi thô vào app.
- **Probe, chụp frame, chạy tiến trình ffmpeg/ffprobe đơn giản**: dùng gói NuGet public `FFMpegCore`
  (`FFProbe.AnalyseAsync`, `FFMpeg.Snapshot`, `FFMpegArguments`…). Không tự viết lớp gọi `ffprobe` rồi parse JSON/stdout.
- **Hiệu ứng chuyển giữa hai clip/ảnh**: ưu tiên **thư viện transition riêng của user** (dựng trên thư viện filter graph ở trên),
  trước khi tự viết biểu thức `xfade`/`blend`/`geq`.
- Chỉ lệch luật khi user đồng ý cho từng trường hợp (vd thư viện thiếu API mà việc đang gấp).

## Why

(2026-10-15) Một app dựng video đã tự viết `MediaPreparer` và cả tầng render bằng chuỗi filter: khó đọc, khó test (test so từng
chuỗi filter), trùng việc với thư viện sẵn có của user, và không tận dụng được bộ transition user đã viết. User quyết bỏ hết để
chuyển sang các thư viện trên.
