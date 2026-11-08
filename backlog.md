# Backlog của project

Đọc trước khi ghi mục backlog mới, tạo file backlog, hoặc chia lại file. Phần tóm tắt bắt buộc nằm ở `Quy tắc Backlog` trong `~/.claude/CLAUDE.md`.

## Ghi gì

- **Việc còn treo và việc nên làm** của một project: việc làm dở, lỗi có sẵn phát hiện ngoài phạm vi đang sửa, đề xuất cải tiến chưa được duyệt, finding review bị hoãn.
- Mỗi việc một mục, ghi: vấn đề, vị trí code (path + dòng), vì sao nên làm, ngày ghi.
- Memory local vẫn giữ tiến độ phiên; backlog là bản dùng chung cho cả nhóm.

## Dạng lưu

- Backlog nằm trong thư mục tài liệu trong git của project (`docs/`, `Docs/` hoặc `.docs/`, theo thư mục project đang dùng; chưa có thì tạo `docs/`).
- **Mỗi repo một backlog riêng, mục nằm ở repo chứa chỗ sửa CHÍNH**: submodule, thư viện tách repo, repo được vendor vào đều là project riêng. Lỗi/việc nằm trong code của submodule thì ghi vào backlog của chính submodule đó (đường dẫn trong mục tính từ gốc submodule), KHÔNG ghi vào backlog của repo cha chỉ vì phiên đang mở ở repo cha. Repo cha chỉ giữ việc mà bản sửa nằm ở code của repo cha (kể cả "cách app dùng thư viện").
  - Việc cần sửa ở cả hai phía: tách thành mục riêng ở mỗi repo cho phần của repo đó; phần dính nhau ngang bằng thì để ở repo cha, repo kia ghi một dòng link trỏ sang.
  - Commit mục backlog của submodule là commit trong repo submodule (theo luật branch/commit của repo đó), rồi repo cha cập nhật con trỏ submodule.
  - **Why**: (2026-11-08) backlog repo app gom cả lỗi của 3 submodule; mở phiên ở repo thư viện không thấy việc của nó, sửa xong ở thư viện lại quên xoá mục bên app.
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
