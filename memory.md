# Memory local của project

Đọc trước lần ghi memory đầu tiên trong phiên (sau compact thì đọc lại). Phần tóm tắt bắt buộc nằm ở mục "Memory, backlog, experience" trong `~/.claude/CLAUDE.md`.

- Luôn chủ động lưu vào memory local của project (`~/.claude/projects/<project>/memory/`) NGAY khi có tiến triển/quyết định/việc dở, KHÔNG đợi cuối phiên (phòng compact mất ngữ cảnh).
- Tách hai loại tệp:
  - **Kiến thức** (quyết định + lý do, bẫy, luật user dặn, thông tin tra cứu): mỗi chủ đề một file `<chủ-đề>.md`, ưu tiên sửa file sẵn có. KHÔNG ghi hash commit, tên nhánh, "đã merge/chưa push", nhật ký từng bước.
  - **Tiến độ** (đang làm tới đâu, nhánh/commit, việc còn treo, bước kế tiếp): file riêng `progress-<chủ-đề>.md` (`type: project`), file kiến thức trỏ sang nó.
- **Việc xong thì xoá tiến độ và lịch sử** (BẮT BUỘC, ngay trong lượt việc đó xong, hoặc khi user nói thôi không làm): chuyển bài học còn giá trị sang file kiến thức, rồi XOÁ file `progress-*` và dòng của nó trong `MEMORY.md`, đồng thời xoá mọi dòng lịch sử của việc đó còn sót ở file khác. Không giữ "đã xong", không gạch ngang, không dời sang mục lưu trữ; lịch sử đã có trong git.
- **Ghi vào đúng repo đã sửa**: phiên mở ở repo A mà sửa sang repo B (thư viện và app dùng nó, worktree/clone tạm của repo khác) thì memory ghi vào memory của project B (đường dẫn theo repo gốc của B, không theo worktree), backlog ghi vào `docs/` của B. Memory của A chỉ giữ phần thuộc A, cần thì một dòng trỏ sang B.
- `MEMORY.md` chỉ là mục lục: mỗi file một dòng ngắn (chủ đề + hook), không ghi trạng thái nhánh/commit.
- **Why**: tiến độ lẫn vào file kiến thức làm memory phình (có project lên ~250 file, 1,1 MB) toàn lịch sử việc đã xong; tiến độ repo khác dồn vào repo mở phiên thì mở phiên ở repo kia không thấy gì.
