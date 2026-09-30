# Kinh nghiệm perl (dùng làm dao mổ sửa file trên Windows/git bash)

- **Script perl có ký tự tiếng Việt trong chuỗi thì BẮT BUỘC `use utf8;`.** Không có nó, perl đọc mã nguồn theo latin-1 nên mỗi byte UTF-8 thành một ký tự riêng; ghi ra bằng `>:encoding(UTF-8)` là **mã hoá hai lần** → file đích đầy `Ä`, `áº`, `Æ°` (mojibake). Nguy ở chỗ script vẫn chạy, vẫn báo "xong", chỉ phần chèn thêm hỏng còn phần cũ của file thì nguyên vẹn — rất dễ commit nhầm.
  + Bộ ba luôn đi cùng nhau: `use utf8;` (giải mã chính mã nguồn) + `open ... '<:encoding(UTF-8)'` / `'>:encoding(UTF-8)'` (giải mã/mã hoá file) + `binmode(STDOUT, ':encoding(UTF-8)')` (in ra terminal).
  + Kiểm ngay sau khi chạy: `grep -n "Ã\|áº\|Ä " <file>` — có dòng nào là hỏng, `git checkout -- <file>` rồi làm lại. Đừng tin thông báo "xong" của script.
- Thay khối nhiều dòng thì dùng `local $/;` đọc cả file, heredoc `<<'END'` giữ nguyên văn bản mẫu, và ghép `\Q...\E` để khỏi phải escape regex: `my $n = ($t =~ s/\Q$old\E/$new/); die unless $n == 1;`. **Luôn kiểm số lần khớp bằng 1** — khớp 0 lần (mẫu lệch một khoảng trắng) hay 2 lần đều là hỏng âm thầm.
- Đừng sinh script mới bằng cách `sed` đổi tên biến trong script cũ để "đảo chiều" phép thay: sed dễ đụng cả dòng khai báo `my $x =` và chỉ báo lỗi ở dòng khác hẳn. Viết thẳng script thứ hai nhanh và chắc hơn.

## `s|PAT|REPL|` mà PAT chứa `||` → perl chèn REPL vào ĐẦU file, không báo lỗi (2026-09-09)

Thay một khối C# có `if (a || b)` bằng `perl -0pi -e 's|...\|\| ...|...|' file` và kết quả là chuỗi
thay thế bị dán lên **đầu file**, còn khối gốc vẫn nguyên. Exit code 0, không cảnh báo gì.

**Vì sao**: với dấu phân cách không phải cặp ngoặc (ở đây là `|`), perl **bóc dấu `\` trước dấu phân
cách** ngay ở tầng phân tích chuỗi, TRƯỚC khi regex engine nhìn thấy pattern. Nên `\|\|` không còn là
"hai ký tự pipe" mà thành hai toán tử **alternation** rỗng:

```
pattern viết:  if \(x \|\| y\)\n    go;
regex nhận:    if \(x || y\)\n    go;
tức là:        (if \(x )  HOẶC  ()  HOẶC  ( y\)\n    go;)
```

Nhánh **rỗng** khớp ngay tại offset 0 ⇒ `s///` (không `/g`) thay chuỗi rỗng ở đầu file = chèn thêm.

Kiểm chứng:

```bash
printf 'a\nif (x || y)\n    go;\nb\n' > t.txt
perl -0pi -e 's|if \(x \|\| y\)\n    go;|CHANGED|' t.txt   # → "CHANGEDa\nif (x || y)\n    go;\nb"
perl -0pi -e 's{if \(x \|\| y\)\n    go;}{CHANGED}' t.txt  # → đúng
```

**Luật**: pattern có `|` (rất hay gặp với code C#/JS: `||`, `|=`, bitwise) thì **KHÔNG dùng `|` làm dấu
phân cách**. Dùng `s{...}{...}` (cặp ngoặc — perl giữ nguyên `\|` bên trong), hoặc `s#...#...#`.
Cùng bẫy với mọi dấu phân cách: `s/…/…/` mà pattern có `/`, `s,…,…,` mà pattern có `,`.

**Dấu hiệu nhận biết khi đã lỡ**: `git diff --stat` ra số dòng nhỏ bất thường nhưng `head -3 file` thấy
nội dung lạ dán vào dòng 1, và dòng đầu cũ bị nối đuôi (`    continue;using System;`).

**Cứu**: `git checkout -- <path>` nếu file chưa có sửa đổi khác; nếu có rồi thì `sed -i '1,Nd'` phần
rác rồi vá lại dòng bị dính. Lưu ý `sed -e '1,4d' -e '1s/…/…/'` **không** chạy được vế sau — địa chỉ
dòng của `sed` tính trên dòng ĐẦU VÀO, dòng 1 đã bị xoá nên lệnh `1s` không bao giờ khớp; phải chạy
hai lượt `sed` riêng.

## Chèn code có dấu `\` bằng phần THAY THẾ của `s///` thì dấu `\` bị nuốt (2026-09-18)

`perl -0pi -e 's/(anchor)/const RE = \/\.(png|jpe?g)\$\/i;\n$1/' chat.js` ghi ra `const RE = /.(png|jpe?g)$/i;`
— mất dấu `\` trước dấu chấm, regex JS đổi nghĩa ("ký tự bất kỳ" thay vì "dấu chấm") mà vẫn chạy,
`node --check` vẫn qua. Phần thay thế của `s///` là chuỗi nội suy: nó đi qua tầng bóc escape của dấu
phân cách, rồi tầng escape kiểu nháy kép, cộng thêm một tầng của shell — đếm cho đúng số `\` gần như
không thể.

**Luật**: đoạn code chèn vào mà có `\`, `$` hay `@` thì KHÔNG viết nó trong phần thay thế. Ghi đoạn đó ra
tệp bằng heredoc `<<'EOF'` (nháy đơn — shell không đụng gì), rồi đọc vào biến:
`perl -0pi -e 'BEGIN{local $/; open F,"/tmp/snip.txt"; $r=<F>; close F} s/ANCHOR\n/$r/' file` —
`$r` chỉ nội suy MỘT lần, nội dung biến giữ nguyên văn. Hoặc dùng Edit tool. Chèn xong luôn `grep` lại
đúng dòng vừa chèn để so từng ký tự.

## Dấu phân cách `s|…|…|` nuốt dấu `|` trong mẫu — kể cả `\|`, kể cả khi mẫu cần "hoặc"

- Gặp 2026-09-19: `perl -0pi -e 's|...<c>text</c> \| <c>textarea</c>...(A\|B)...|...$1...|' f1 f2` — trong mẫu vừa có `|` chữ (tài liệu `a | b`) vừa muốn nhóm lựa chọn `(A|B)`. Perl coi `|` là dấu phân cách nên cắt mẫu giữa chừng; thay thế chạy với mẫu lệch, GHI HỎNG CẢ HAI FILE (chèn nửa khối vào giữa dòng) mà lệnh vẫn thoát 0.
- **How to apply:** khối C#/doc chứa `|` thì chọn dấu phân cách không xuất hiện trong mẫu (`s{…}{…}` hoặc `s#…#…#`), và với khối nhiều dòng thì dùng công cụ Edit thay vì perl. Sau MỌI lệnh perl sửa nhiều file, `git diff --stat` + đọc lại vùng vừa sửa trước khi chạy tiếp; hỏng thì `git checkout -- <file>` (file chưa có thay đổi riêng nào khác).

## Trong phần thay thế `s{...}{...}e`, regex phụ bên trong ghi đè `$&` (2026-09-26)

Dùng `$&` (hoặc `$1`…) trong khối `/e` SAU KHI khối đó chạy một phép so khớp khác (`$x =~ /.../`,
`$path =~ s{...}{}`) thì `$&` đã là kết quả của phép so khớp phụ, không còn là đoạn đang thay. Ca thật:
script sửa anchor trả `$&` cho nhánh "không tìm thấy" → hàng trăm link trong tài liệu biến thành chuỗi
rác (`Docs/../`), exit 0.

**Luật**: dòng ĐẦU của khối `/e` chép hết thứ cần vào biến riêng (`my ($all, $a, $b) = ($&, $1, $2);`),
sau đó chỉ dùng biến đó. Chạy xong luôn xem `git diff --stat` — số dòng đổi lớn bất thường là dấu hiệu.
