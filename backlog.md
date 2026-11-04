# Backlog của project

Đọc trước khi ghi mục backlog mới, tạo file backlog, hoặc chia lại file. Phần tóm tắt bắt buộc nằm ở `Quy tắc Backlog` trong `~/.claude/CLAUDE.md`.

## Ghi gì

- **Việc còn treo và việc nên làm** của một project: việc làm dở, lỗi có sẵn phát hiện ngoài phạm vi đang sửa, đề xuất cải tiến chưa được duyệt, finding review bị hoãn.
- Mỗi việc một mục, ghi: vấn đề, vị trí code (path + dòng), vì sao nên làm, ngày ghi.
- Memory local vẫn giữ tiến độ phiên; backlog là bản dùng chung cho cả nhóm.

## Dạng lưu

- Backlog nằm trong thư mục tài liệu trong git của project (`docs/` hoặc `Docs/`, theo thư mục project đang dùng; chưa có thì tạo).
- Project nhỏ dùng một file `Backlog-vi.md`.
- Khi file vượt ~800 dòng hoặc project có nhiều thành phần tách biệt thì chuyển sang thư mục `Backlog/`: `README-vi.md` là mục lục (bảng file → phạm vi, cách chọn file), mỗi khu vực một file `<Khu>-vi.md`. Cách chia khu vực của từng project ghi trong `CLAUDE.md` của project đó.
- **Ghi mục mới**: đặt vào file của khu vực chứa chỗ sửa CHÍNH; việc thật sự chạm nhiều khu vực ngang nhau thì vào file chung (vd `Shared-vi.md`). Không chép một mục sang nhiều file; cần thì file kia ghi một dòng link trỏ sang. Không chắc thuộc đâu thì tra bảng phạm vi ở `README-vi.md`. KHÔNG chia file theo mức nặng nhẹ (mức đổi theo thời gian thì phải dời mục, đứt lịch sử).
- **Why**: một file backlog chung phình tới hàng nghìn dòng (vài trăm mục) thì đọc tốn ngữ cảnh, khó tìm mục trùng, và hay xung đột merge khi nhiều nhánh cùng thêm mục.

## Nhãn mức

- Tiêu đề mỗi mục bắt đầu bằng:
  - `[Cao]`: sai dữ liệu/tiền, lỗ hổng quyền/bảo mật, sự cố production.
  - `[Vừa]`: lỗi chức năng có đường né, đua hiếm.
  - `[Thấp]`: giao diện, dọn code, thiếu test, đề xuất.
- Ví dụ: `### [Cao] Khoá chống trùng không nhả khi lượt hỏng`.
- Trong mỗi file xếp mục `[Cao]` lên đầu. Xem việc nặng toàn project: grep `^### \[Cao\]`.

## Xử lý xong

- Xử lý xong một mục thì xoá mục đó (cả tiêu đề `###` lẫn nội dung) trong cùng commit với bản sửa. Không đánh dấu "đã xong", không gạch ngang, không dời sang mục lưu trữ.
- Mới xử lý một phần thì sửa lại mục cho khớp phần còn treo thay vì xoá.
- Trước khi commit, rà xem đợt sửa có đóng mục nào trong file backlog không.
