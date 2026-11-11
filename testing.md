# Test hồi quy khi sửa lỗi

Đọc trước khi sửa lỗi trong workspace có bộ test. Phần tóm tắt bắt buộc nằm ở `Quy tắc test hồi quy` trong `~/.claude/CLAUDE.md`.

- Workspace đã có bộ test thì **mọi lần sửa lỗi đều kèm test chặn lỗi cũ**, commit chung với bản sửa, không cần user nhắc.
- Test phải đi đúng đường gây lỗi và khẳng định hành vi đúng. Có sẵn bài `Skip` mô tả đúng lỗi đó thì bỏ `Skip` và đổi tên cho khớp, thay vì viết bài trùng.
- **Chứng minh test bắt được lỗi**: tạm gỡ bản sửa (hoặc chạy trên code cũ) để thấy test ĐỎ, rồi khôi phục và thấy XANH. Test chưa từng đỏ trên code lỗi thì chưa chứng minh được gì.
- Phủ thêm các ca biên sát bản sửa khi chúng rẻ để viết (vd không đụng dữ liệu của người khác, ca tự thao tác lên chính mình).
- Không test được (lỗi chỉ lộ ở hạ tầng thật, UI thuần, chưa có chỗ test phù hợp) thì NÓI RÕ lý do; đáng làm về sau thì ghi backlog.
- Workspace chưa có test thì không tự dựng project test mới chỉ vì một bản sửa; hỏi user trước.
- **Why**: lỗi đã sửa mà không có test giữ lại thì lần refactor sau dễ làm nó quay lại mà không ai hay.
