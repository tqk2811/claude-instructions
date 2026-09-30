# Kinh nghiệm Markdown

## Link tới chỗ khác trong CÙNG một file `.md`: dùng id tiêu đề, KHÔNG dùng `#Lxx`

- **Triệu chứng:** trong Preview của VSCode, bấm `[xem mục 6](#L461)` không nhảy đi đâu cả, cũng không báo lỗi.
- **Vì sao:** Preview để trình duyệt tự tìm phần tử có `id` trùng với phần sau dấu `#`. Không có phần tử nào mang
  id `L461`, nên bấm không có tác dụng. `#Lxx` chỉ chạy trong hai trường hợp:
  - link sang một file KHÁC: VSCode mở file đó qua extension và nhảy tới dòng;
  - link trong khung chat Claude Code (vì thế `Quy tắc thuật ngữ` và `Quy tắc link code` vẫn dùng `#Lxx` khi trả lời chat).
- **Cách viết id tiêu đề** (theo github-slugger):
  - lấy chữ của tiêu đề, bỏ dấu `` ` ``;
  - viết thường, bỏ dấu câu, **giữ nguyên chữ có dấu tiếng Việt**;
  - mỗi khoảng trắng thành một dấu `-`;
  - hai tiêu đề ra cùng id thì cái sau thêm `-1`, `-2`…
  - Ví dụ: `## 6.1 Quy ước chung` → `#61-quy-ước-chung`;
    `## Sự đồng ý của chủ thể dữ liệu (consent)` → `#sự-đồng-ý-của-chủ-thể-dữ-liệu-consent`;
    `### admin_users — Tài khoản quản trị nền tảng` → `#admin_users--tài-khoản-quản-trị-nền-tảng`
    (dấu `—` bị bỏ, hai khoảng trắng hai bên thành hai dấu `-`).
- **Bẫy:** đổi chữ một tiêu đề là mọi link trỏ tới id cũ chết im lặng. Đổi tiêu đề xong thì
  `grep -n '](#' <file>` và so với các tiêu đề hiện có.
- **Kiểm nhanh sau khi viết:** liệt kê mọi link trong file bằng `grep -o '](#[^)]*)' <file> | sort -u`, rồi đối chiếu
  từng id với tiêu đề thật.
