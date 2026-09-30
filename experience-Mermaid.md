# Kinh nghiệm Mermaid

## Quy tắc bắt buộc: sơ đồ trong file phải tránh chồng chéo hết mức có thể

Được trỏ tới từ `Quy tắc sơ đồ` trong `~/.claude/CLAUDE.md`. Áp cho MỌI khối ```` ```mermaid ```` mình viết
hoặc sửa trong file `.md`.

1. **Không có đường cắt nhau, không có cạnh đi chung một đoạn, không có cạnh chạy xuyên qua ô, không có
   nhãn đè lên cạnh hoặc lên ô khác.** Không khử hết được (vd cấu trúc K2,2 ở mục dưới) thì giảm xuống ít
   nhất có thể, rồi nói rõ chỗ còn lại cho user.
2. **Render ra ảnh và TỰ XEM trước khi báo xong** (cách render ở mục "Render thử ra PNG" bên dưới). Mã
   Mermaid hợp lệ KHÔNG có nghĩa là bố cục sạch — dagre tự xếp chỗ và hay dồn cạnh vào cùng một đường.
3. **Một bảng/nút toả quá ~5 cạnh, hoặc sơ đồ quá ~10 bảng → tách thành nhiều sơ đồ theo cụm nghĩa**
   (vd "sổ + khách + phụ trợ" và "trường động + giá trị"). Bảng làm cầu nối được phép xuất hiện ở cả hai
   sơ đồ, kèm một câu nói rõ đó là cùng một bảng. Tách cụm thường sạch hơn hẳn mọi cách sắp lại thứ tự.
   - Một quan hệ phụ (vd `Orders → OrderItems`) nên nằm ở sơ đồ nào cũng có đầu mút của nó, thì dời
     sang sơ đồ đang thưa để giảm số cạnh toả ra từ nút đông nhất.
4. Còn đè thì chữa theo thứ tự ở mục kế tiếp: sắp lại thứ tự khai báo → viết ngược cạnh để đổi tầng →
   `direction LR` → (`layout: elk` chỉ khi chắc chỗ xem hỗ trợ).
5. Ghi lại vào file nguồn xong thì tách các khối ra lần nữa và `diff` với bản đã render.

## Sơ đồ bị đè nhãn / đường cắt nhau: chỉnh bằng THỨ TỰ khai báo, không có toạ độ

- `erDiagram`/`flowchart` xếp chỗ bằng dagre: tầng (rank) theo chiều cạnh (bên trái trên, bên phải
  dưới), thứ tự trái→phải trong một tầng theo **thứ tự bảng xuất hiện lần đầu** trong khai báo.
  Không có cách đặt toạ độ tay.
- Cách chữa, theo thứ tự nên thử:
  1. Sắp lại dòng quan hệ: khai nhánh con của cùng một cha liền nhau, bảng "ngoài lề" (vd `Files`) khai sau cùng.
  2. Hai bảng A, B cùng tầng cùng nối tới hai bảng con C, D ở tầng dưới (K2,2) thì **kiểu gì cũng có
     một chỗ cắt**. Muốn hết phải đẩy B xuống tầng khác bằng cách viết ngược cạnh: `C }o--|| B`
     (cùng nghĩa với `B ||--o{ C`, ký hiệu vẫn nằm sát đúng bảng). Nhớ kiểm tài liệu có quy ước
     "bên trái là cha" không — viết ngược sẽ phá quy ước đó.
  3. `direction LR` — hết đè nhưng sơ đồ bẹt và chữ nhỏ.
  4. `config: layout: elk` — đẹp nhất nhưng GitHub/nhiều preview không render.
- Hai cạnh song song giữa cùng cặp bảng có nhãn dài hay đè nhau; đổi thứ tự khai các bảng con của
  bảng cha đó (không đảo chiều) thường tách được.

## Render thử ra PNG để so, không cài Chromium riêng

```bash
npm i @mermaid-js/mermaid-cli     # trong scratchpad; postinstall puppeteer bị chặn → không có Chromium
```

`pp.json` trỏ vào Chrome có sẵn + profile riêng trong scratchpad (luật `~/.claude/chrome.md`):

```json
{"executablePath":"C:\\Program Files\\Google\\Chrome\\Application\\chrome.exe","headless":"new","userDataDir":"<scratchpad>\\cprof1"}
```

`node_modules/.bin/mmdc -p pp.json -i d.mmd -o d.png -s 1.5 -b white` — tự đóng Chrome khi xong.
mermaid-cli 12 KHÔNG còn `-w`/`--width` (báo `unknown option '-w'`); sơ đồ rộng thì dùng `--size 5000`
(cạnh dài nhất của ảnh), không thì ảnh bị ép về ~1200px và nhãn không đọc được. Mermaid 12 vẽ cạnh
`erDiagram` kiểu gấp khúc vuông góc, nên "cạnh đi chung một đoạn" dễ thấy hơn hẳn bản cũ.

## Sơ đồ toàn cảnh 50+ bảng: vẽ ở mức CỤM, đừng cố gỡ

erDiagram 54 bảng / 62 cạnh ra hàng chục chỗ cắt, sắp lại thứ tự không cứu được. Thay bằng `flowchart`
mỗi nút một cụm (mục của tài liệu), mũi tên từ cụm giữ khoá ngoại tới cụm bị trỏ, nhãn là tên cột; bảng
"hub" bị trỏ từ gần mọi cụm (vd `Files`) tách thành một sơ đồ hình sao riêng. Trước khi bỏ sơ đồ bảng
phải kiểm bằng script rằng mọi khoá ngoại vẫn có mặt ở ít nhất một sơ đồ theo mục.
Viết `pp.json` bằng tool Write, đừng dựng bằng heredoc + sed (escape `\\` trong bash rất dễ hỏng).
Tách các khối ```` ```mermaid ```` ra file bằng awk rồi render hàng loạt; sau khi ghi lại vào file
nguồn thì tách lại lần nữa và `diff` với bản đã render để chắc thứ nằm trong file đúng là thứ đã xem.

## Cụm nhỏ 2 tầng đặt giữa một "cha đông con" → cạnh chạy XUYÊN qua ô

Một cụm độc lập (vd `system_api_keys` + 2 con) nằm ở tầng 0–1, trong khi bảng cha lớn bên cạnh
toả cạnh xuống tầng 1–2: dagre hay nhét cụm đó vào giữa cái quạt cạnh, và các cạnh dài vẽ đè ngang
qua ô của nó (không phải "cắt nhau" nên dễ bỏ sót khi nhìn ảnh thu nhỏ). Chữa bằng cách dời khối khai
báo của cụm tới GIỮA hai bảng cha lớn (khoảng trống phía trên các con), rồi phóng to đúng vùng đó để
kiểm: `ffmpeg -loglevel error -y -i in.png -vf "crop=W:H:X:Y" out.png` (ffmpeg có sẵn trên máy).
Sơ đồ toàn cảnh rộng thì render `-w 2400` mới đọc được nhãn.
