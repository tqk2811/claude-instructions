# Sửa repo `~/.claude`

Đọc trước khi tạo, sửa hoặc commit file trong repo `~/.claude` (CLAUDE.md, file rule, `Experiences/*.md`).

## Đặt tên file Markdown

- `Quy tắc đặt tên file Markdown`: file `.md` tiếng Anh giữ nguyên tên; file `.md` KHÔNG phải tiếng Anh thì thêm hậu tố mã ngôn ngữ ISO 639-1 trước `.md`, dạng `<tên gốc>-<mã>.md` (vd `<tên gốc>-vi.md`). Luật này áp cho mọi project.
- NGOẠI LỆ: các file trong `~/.claude` giữ tên cố định, KHÔNG thêm hậu tố.

## Ẩn danh

- `Quy tắc ẩn danh ~/.claude`: các file `.md` được git track trong `~/.claude` là bộ hướng dẫn dùng chung, phải viết CHUNG CHUNG. Thông tin riêng của máy hoặc người dùng ghi vào `~/.claude/local.md` (bị `.gitignore` chặn); file track chỉ trỏ sang đó.
- **KHÔNG ghi vào file track**: tên project/repo/sản phẩm thật, tên khách hàng, tên thư viện riêng của user, đường dẫn máy cá nhân (`D:\...`, `C:\Users\<tên>\...`), username/email, tên cấu hình build lộ tên khách, tên class/namespace/bảng mang nghiệp vụ của một sản phẩm cụ thể. Ngoại lệ: dòng `Co-Authored-By` bắt buộc trong `git.md`; thư viện của user đã publish public trên NuGet (`TqkLibrary.*`, `FFmpegArgs`) thì ghi thẳng tên gói/API (đường dẫn source trên máy vẫn để ở `local.md`).
- **Thay bằng**: placeholder (`<App>`, `<repo>`, `MyApp`, `C:\src\...`), ví dụ nghiệp vụ trung tính (`Product`, `Order`), hoặc mô tả vai trò ("thư viện HTTP nội bộ"). Ghi chú sự cố thì giữ ngày và bài học, bỏ tên project: "(2026-09-11)", không phải "(ProjectX, 2026-09-11)".
- Thông tin riêng mà vẫn cần cho việc chạy (vị trí SDK, repo mẫu, tên gói polyfill…) để trong `local.md`; file track ghi "xem `~/.claude/local.md`". Gặp chỉ dẫn như vậy thì đọc `local.md` trước khi làm.
- Trước khi commit repo `~/.claude`, grep phần diff tìm tên project, đường dẫn máy và username.
- **Why**: repo `~/.claude` có thể được chia sẻ hoặc public; tên project, khách hàng và kiến trúc nội bộ từng lọt vào file rule qua ghi chú sự cố, rồi phải đi ẩn danh lại hàng loạt.

## Đường dẫn và bảng tra

- **Đường dẫn phân biệt hoa thường**: bộ `~/.claude` còn dùng trên Linux, nên mọi đường dẫn nhắc tới phải viết ĐÚNG hoa thường như tên file thật. Tạo hay đổi tên file thì grep lại các chỗ tham chiếu và so khớp với `git ls-files`.
- Tạo file experience MỚI thì thêm một dòng vào bảng tra experience trong `CLAUDE.md` trong cùng lượt sửa; tạo file rule mới thì thêm dòng vào bảng "File rule cần đọc trước".
- Repo này KHÔNG chia nhánh phụ; commit thẳng `master`, mỗi lần commit và push đều cần user yêu cầu (chi tiết ở `git.md`).
