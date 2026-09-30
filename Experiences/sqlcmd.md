# Bài học khi chạy script SQL bằng `sqlcmd`

## `sqlcmd -i file.sql` mặc định đọc file theo codepage ANSI, không phải UTF-8

- **Triệu chứng**: chữ có dấu ghi vào cột `nvarchar` biến thành mojibake kiểu
  `Xác nhận` → `XÃ¡c nháº­n` (đúng đặc trưng byte UTF-8 bị đọc theo Latin-1/CP1252).
  Nhìn trong SSMS hay trên web đều hỏng như nhau vì dữ liệu trong DB đã sai thật.
  Kiểm chứng nhanh: `SELECT UNICODE(SUBSTRING(cot,2,1)), LEN(cot)` — `á` phải ra 225;
  ra 195 (`Ã`) và `LEN` phình lên là đã dính.
- **Vì sao**: file `.sql` do editor/công cụ sinh ra thường là **UTF-8 KHÔNG BOM**. `sqlcmd`
  không tự đoán UTF-8; không có BOM thì nó decode theo codepage ANSI của máy. Tiền tố `N'...'`
  KHÔNG cứu được — `N` chỉ nói literal là Unicode, còn ký tự đã bị decode sai từ trước khi
  SQL Server nhìn thấy câu lệnh.
- **Cách xử lý** (chọn một):
  - `sqlcmd -f 65001 -i file.sql` — ép codepage đầu vào là UTF-8. Ngắn nhất, đáng làm mặc định.
  - Ghi file `.sql` bằng UTF-16LE có BOM — `sqlcmd` nhận diện được BOM này từ lâu.
  - Tránh hẳn ký tự ngoài ASCII trong script: ghép chuỗi bằng `NCHAR(0xXXXX)`.
- **Luôn kiểm chứng lại sau khi chạy**, đừng tin script chạy không báo lỗi là dữ liệu đúng:
  câu `UPDATE` với chuỗi mojibake vẫn thành công 100%, không có cảnh báo nào.
- **Bẫy phụ**: lỗi này thường lọt qua vì script *seed/cleanup dùng khi test* mới có chữ có dấu,
  còn migration EF chạy bằng `dotnet ef database update` thì đi qua ADO.NET nên hoàn toàn đúng.
  Kết quả là chỉ vài cột do script vá tay bị hỏng, các cột seed khác trong cùng bảng vẫn sạch —
  rất dễ đổ oan cho code ứng dụng. **So chính xác cột nào hỏng, cột nào không** trước khi đi
  sửa code: nếu dữ liệu seed cùng migration vẫn đúng thì thủ phạm chắc chắn là đường ghi tay.

## Đường dẫn dùng dấu `/` truyền cho `-i` bị đọc thành SWITCH, báo lỗi về `-U/-P`

- **Triệu chứng**: `sqlcmd -S . -E -d Db -i "C:/Users/me/x.sql"` trả về đúng một dòng
  `Sqlcmd: The -E and the -U/-P options are mutually exclusive.` — một câu nói về **cách đăng
  nhập**, trong khi dòng lệnh không hề có `-U` lẫn `-P`. Rất dễ đi sửa nhầm sang chuỗi kết nối,
  quyền Windows-auth hay biến môi trường, mà chẳng cái nào liên quan.
- **Vì sao**: `sqlcmd` (bản ODBC cổ điển, đã đo trên `16.0.1000.6`) nhận switch theo **cả hai
  kiểu** `-E` lẫn `/E`. Nên mọi dấu `/` trong **giá trị** của một tham số đều có thể bị đọc lại
  thành switch mới: `C:/Users/...` chứa `/U` ⇒ nó tưởng mình vừa khai `-U`, và `/tmp/...` cũng
  dính y hệt. Đây thuần tuý là chuyện **dấu phân cách đường dẫn**, không dính gì tới `-f`, tới
  codepage hay tới quyền.
- **Cách xử lý**: truyền đường dẫn kiểu Windows bằng **dấu `\`** (`"C:\Users\me\x.sql"`), hoặc
  `cd` tới thư mục rồi dùng **tên file trần**. Cả hai đều chạy ngay, kể cả kèm `-E` và `-f 65001`.
- **Cách tự kiểm khi lạc hướng**: đổi `-i file` thành `-Q "SELECT 1"` trên đúng dòng lệnh đó. Chạy
  được nghĩa là phần đăng nhập vốn không sai — thủ phạm nằm trong **chuỗi đường dẫn**, chứ câu báo
  lỗi thì đang chỉ sang hướng khác.
