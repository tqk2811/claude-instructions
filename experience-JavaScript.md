# Kinh nghiệm JavaScript / trình duyệt

## `BarcodeDetector` KHÔNG có trên Chrome/Edge bản Windows

Shape Detection API (`window.BarcodeDetector`) dựa vào API quét mã của **hệ điều hành**, nên chỉ
có trên **Android, ChromeOS và macOS**. Chrome/Edge trên **Windows** và Linux, Firefox và Safari
đều không có — dù cùng một phiên bản Chrome, cùng HTTPS, quyền camera đã cấp đủ.

Triệu chứng đặc trưng: điện thoại Android quét QR bình thường, máy tính Windows bấm "bật camera"
là báo lỗi ngay **mà đèn camera không hề sáng** — vì code kiểm `typeof BarcodeDetector` rồi thoát
trước cả khi gọi `getUserMedia`. Rất dễ đổ oan cho quyền camera của trình duyệt.

Xác nhận trong 5 giây: mở Console gõ `typeof BarcodeDetector` → `"undefined"`.

Cách vá: nạp một bộ giải QR bằng JavaScript thuần (jsQR — Apache-2.0, ~250KB thô / ~57KB gzip)
làm nhánh dự phòng. Lưu ý khi dùng:

- Thu nhỏ khung hình trước khi giải. Đo thật trên V8: **1280×720 mất ~166ms/khung**, **640×480
  còn ~60ms/khung**. Giải bằng CPU ngay trên luồng giao diện, nên khung to là vừa treo giao diện
  vừa chậm hơn cả nhịp quét.
- Đặt `inversionAttempts: 'dontInvert'` nếu mã QR của mình luôn tối trên nền sáng — mặc định
  `attemptBoth` quét mỗi khung hai lượt, gấp đôi thời gian.
- Ngưng giải mã trong lúc có animation chạy (canvas, vòng quay...), nhưng **giữ camera bật**:
  tắt camera rồi bật lại bắt người dùng chờ trình duyệt mở lại thiết bị.

## Phân loại lỗi `getUserMedia` theo `err.name`

`catch` rồi báo chung một câu "không mở được camera" là người dùng đi cấp lại quyền trong khi lý
do thật hoàn toàn khác. Tách tối thiểu bốn nhánh:

- `NotAllowedError` / `SecurityError` → người dùng hoặc chính sách chặn quyền.
- `NotFoundError` / `DevicesNotFoundError` / `OverconstrainedError` → không có camera nào khớp.
- `NotReadableError` / `TrackStartError` → **camera đang bị ứng dụng khác chiếm** (Zoom, Teams,
  OBS...). Rất hay gặp trên máy tính, và không có cách nào đoán ra từ giao diện.
- Còn lại → lỗi chung.

Ngoài ra `navigator.mediaDevices` chỉ tồn tại trong **secure context** (HTTPS hoặc `localhost`);
mở bằng `http://<ip-nội-bộ>` là thuộc tính đó `undefined`, không có cách nào lách.

## Test thư viện trình duyệt bằng Node mà không cần trình duyệt

File UMD kiểm `typeof module === 'object'` trước, nên `require()` trong Node đi nhánh CommonJS và
**không** gắn biến toàn cục — muốn kiểm chứng đúng hành vi trên trình duyệt thì chạy trong `vm`
với sandbox tự dựng `window`/`self`:

```js
const sb = {}; sb.window = sb; sb.self = sb;
vm.createContext(sb); vm.runInContext(fs.readFileSync(file, 'utf8'), sb);
console.log(typeof sb.jsQR);   // 'function'
```

Muốn kiểm thật (không chỉ "nạp được"), dựng dữ liệu ảnh RGBA thô bằng chính bộ sinh của hệ thống
rồi đưa thẳng vào bộ giải — khỏi cần thư viện đọc PNG.

## Kiểm giao diện canvas/JS bằng headless Chrome, không cần dựng server

Trang thử đặt trong thư mục tạm, nạp **thẳng file js/css thật** qua `file:///D:/...` (đường dẫn
tuyệt đối trong `src`/`href`), stub các thứ trang cần mà máy không có (camera, WebSocket), rồi:

```bash
chrome.exe --headless=new --disable-gpu --allow-file-access-from-files --hide-scrollbars \
  --virtual-time-budget=4000 --window-size=1280,800 --screenshot=out.png "file:///.../thu.html"
```

Sửa code thật → chụp lại → nhìn ảnh. Nhanh hơn hẳn dựng cả server chỉ để xem một chỗ vẽ.

Ba cái bẫy đã vấp:

- **Thiếu framework CSS mà trang thật có** (Bootstrap...): những class như `d-none` mất tác dụng,
  nên mọi khối lẽ ra bị ẩn (popup, overlay) hiện hết và phủ mờ đúng thứ cần xem. Ảnh chụp ra tối
  om nhưng KHÔNG phải lỗi màu — tự thêm `.d-none{display:none!important}` vào trang thử.
- **`--virtual-time-budget` đóng băng `performance.now()`**: mọi phép đo thời gian ra đúng `0.00`.
  Muốn đo hiệu năng thì bỏ cờ đó, chạy đo trong handler `load` rồi in ra `document.title` và đọc
  bằng `--dump-dom`. Giữ cờ đó cho ảnh chụp (nó chờ ảnh/timer xong mới chụp).
- Đừng chỉ nhìn mắt để kết luận hình có tràn khỏi vùng cho phép hay không — đọc pixel:
  `getImageData` rồi tìm điểm đục xa tâm nhất, in ra `document.title`, đọc bằng `--dump-dom`.
  Tôi đã "thấy rõ ràng" ảnh tràn khỏi vòng tròn, đo ra thì không, phần thò ra là bóng đổ vẽ sẵn
  trong chính file PNG.

`--dump-dom` in DOM sau khi trang tải xong, nên mọi kết quả đo cứ nhét vào `document.title` rồi
`grep -o "<title>[^<]*</title>"` là lấy được.

## Đo chuyển động chạy bằng `requestAnimationFrame`: phải cho `--dump-dom` chờ bằng tài nguyên chậm

`--virtual-time-budget` **không đo được** thứ gì chạy bằng rAF: nó tua thời gian nên hai khung hình
liên tiếp cách nhau cả trăm ms, mà chuyển động viết đúng cách nào cũng có nhánh bỏ qua khung hình
có `elapsed` quá lớn (tab vừa hiện lại mang theo cả quãng thời gian bị ẩn). Kết quả đo ra là vị trí
**đứng im**, trông y hệt chuyển động hỏng — mất công đi tìm lỗi trong code không có lỗi.

Cách chạy được đồng hồ **thật** mà vẫn dùng `--dump-dom`: dựng một server node nhỏ phục vụ trang
thử + thư mục static thật, thêm một route trả lời chậm (`setTimeout` rồi mới `res.end`) và nạp nó
bằng `<script src="/slow.js">` ở cuối trang. `load` chưa bắn nên `--dump-dom` chờ đúng ngần ấy giây,
trong lúc đó rAF chạy theo thời gian thật. Probe trong trang lấy mẫu bằng `setTimeout` ở nhiều mốc
rồi ghi cả mảng vào `document.title`, đọc ra là thấy tốc độ có đều không, có nhảy ở đâu không.

Server đó còn tiện để giả luôn endpoint mà trang tự gọi theo chu kỳ (trả JSON đổi nội dung mỗi lượt),
nhờ vậy kiểm được cả tương tác giữa nhịp đọc lại và chuyển động — loại lỗi chỉ lộ ra khi hai thứ đó
gặp nhau, mà trang thử tĩnh thì không bao giờ dựng được.

**Đừng dùng ảnh `--screenshot` để phán đoán trạng thái tĩnh của một chuyển động có nhịp.** Chrome
chụp sau `load` cộng thêm 1–2 giây overhead **không đoán trước được**, nên với một chuyển động kiểu
"đẩy rồi nghỉ" thì ảnh nào cũng rơi vào đúng lúc đang đẩy, và cái lệch vài chục pixel của khung hình
giữa chừng nhìn ra y hệt một lỗi canh lề. Tôi đã đi tìm lỗi không có thật mất khá lâu vì tin vào ảnh.
Đo bằng số ở nhiều mốc thời gian mới nói được trạng thái nghỉ có đúng hay không.

**Cách gọn hơn hẳn khi trang chạy trên server thật: lái bằng CDP, thời gian thật, không cần
`--virtual-time-budget` lẫn route chậm.** Mở Chrome `--headless=new --remote-debugging-port=9333
--user-data-dir=<riêng>`, rồi trong node: `Page.navigate` → `setTimeout` chờ đúng số giây thật →
`Runtime.evaluate` đọc trạng thái + `Page.captureScreenshot`. Vòng lặp lấy mẫu mỗi 150ms đo được cả
**khoảng cách giữa hai lượt đổi** lẫn **thời lượng một lớp CSS tồn tại** (ghi mốc lúc lớp xuất hiện
và lúc biến mất) — thứ mà ảnh chụp không bao giờ nói ra được. Node 22+ có `WebSocket` global nên
không phải cài `ws`.

Kèm theo, khi chụp đúng lúc một hiệu ứng vừa bắt đầu thì **phải chờ thêm** cho transition chạy xong
mới chụp: bắt được `classList.contains('is-fresh')` không có nghĩa là màu đã lên: `transition:
background .3s` vẫn còn đang nội suy, ảnh ra vẫn là màu cũ và dễ kết luận nhầm "class có mà CSS
không ăn".

**Thiết kế bộ chặn khung hình dài: KẸP, đừng BỎ.** Nhánh `if (elapsed > 200) return;` (bỏ hẳn khung
hình) là cách thường thấy để tab vừa hiện lại không nhảy vọt mấy vòng, nhưng nó có mặt trái nặng:
máy nào vẽ chậm hơn ngưỡng đó — máy chiếu cũ, máy đang mở nhiều tab — là **khung hình nào cũng bị
bỏ**, đồng hồ đứng yên và trang không bao giờ chuyển động nữa. Triệu chứng đọc ra thành "trang bị
treo" trong khi nó vẫn chạy và vẫn đang gọi API đều đặn. `if (elapsed > 200) elapsed = 200;` chặn
được cú nhảy sau khi tab ẩn y hệt, mà vẫn sống trên máy chậm.

Muốn đọc màu thật của một ảnh chụp (thay vì đoán bằng mắt): nạp PNG vào `<canvas>` trong một trang
khác, quét một cột pixel rồi in các dải màu liên tiếp ra `document.title`. Đủ để trả lời "vệt trắng
kia cao bao nhiêu pixel và nằm ở đâu" mà không cần thư viện ảnh nào.

## Ký tự phân tách vô hình trong chuỗi JS — đã vấp hai lần trên cùng một file

Dấu phân tách của một "chuỗi chữ ký" (nối vài trường lại để so xem có đổi không) rất hay bị viết
thành **ký tự điều khiển thật** (`\x00`, `\x01`) thay vì hai ký tự `\` + `n`. Ba hậu quả, cái nào
cũng khó lần:

- Git coi file là **binary** ⇒ mọi diff của nó thành `Binary files differ`, không review được nữa.
- Hai chỗ sinh chuỗi dùng dấu phân tách khác nhau (một chỗ ký tự thật, một chỗ dấu cách) thì hai
  chuỗi **không bao giờ bằng nhau** ⇒ mọi thứ đều bị coi là "vừa đổi", giao diện nhấp nháy toàn bộ.
- Dấu phân tách **rỗng** (`a + '' + b`) thì ngược lại: hai danh sách khác nhau ra cùng một chuỗi ⇒
  giao diện **im lặng không cập nhật**.

Dùng `'\n'` (hai ký tự trong mã nguồn) và **`node --check <file>` sau mỗi lần sửa** — nó bắt được cả
trường hợp chuỗi bị cắt đôi bởi một newline thật, thứ mà mắt đọc diff không thấy.

## Điều khiển Chrome bằng CDP không cần cài gì (Node ≥ 22)

Node có sẵn `WebSocket` toàn cục nên **không cần puppeteer/ws** để lái một trang thật:

```
chrome.exe --headless=new --remote-debugging-port=9333 --user-data-dir=<abs> about:blank
```
rồi trong node: `fetch('http://127.0.0.1:9333/json/list')` → lấy `webSocketDebuggerUrl` của target
`type === 'page'` → `new WebSocket(url)` → gửi `{id, method, params}` và khớp phản hồi theo `id`.

Đủ để làm test giao diện thật: `Page.navigate`, `Runtime.evaluate` (đo trạng thái theo nhiều mốc
thời gian), `DOM.setFileInputFiles` (**upload file thật** — thứ curl không thay thế được vì nó bỏ
qua toàn bộ validation phía trang), `Page.addScriptToEvaluateOnNewDocument` (gắn MutationObserver /
bắt `window.onerror` / ghi đè `window.confirm` TRƯỚC khi script của trang chạy).

**Bẫy khi lái trang bằng `querySelector`:** danh sách bộ chọn ngăn bằng dấu phẩy trả về phần tử đầu
theo **thứ tự DOM**, không phải theo thứ tự bộ chọn — `querySelector('form[action*=x], form button')`
rất dễ vớ phải form của layout (đổi ngôn ngữ, đăng xuất) nằm phía trên. Triệu chứng đọc ra y hệt
"bấm nút không có gì xảy ra", và tôi đã tưởng là lỗi ứng dụng. Bám vào một ô nhập chắc chắn là của
form mình cần rồi `.closest('form')`.

## Con của flex container bị BÓP LẠI thay vì làm trang cuộn

Layout dashboard kiểu quen thuộc: khung ngoài `height: 100vh; overflow: hidden`, cột nội dung
`display: flex; flex-direction: column; overflow-y: auto`, bên trong là một vùng nội dung
`flex: 1; display: flex; flex-direction: column`. Nội dung dài hơn màn hình thì **trang không cuộn**
và bảng dữ liệu chỉ còn vài dòng.

Hai nguyên nhân cộng lại, cả hai đều là mặc định của flexbox nên không thấy ở đâu trong CSS:

1. `flex: 1` = `flex: 1 1 0%`. `flex-basis: 0%` nghĩa là chiều cao của vùng đó bằng **chỗ trống còn
   lại của màn hình**, hoàn toàn không liên quan tới nội dung bên trong.
2. Con trực tiếp của một flex container là **flex item**, mặc định `flex-shrink: 1`. Nội dung dài
   hơn khung thì trình duyệt **bóp từng đứa con lại** cho vừa, chứ không để chúng tràn ra và đẩy
   khung ngoài cuộn.

Kết quả là mọi thứ "vừa khít" nên khung ngoài không có gì để cuộn, còn đứa con bị bóp thì nếu nó
tình cờ có `overflow` khác `visible` sẽ lặng lẽ biến thành một khung cuộn tí hon lồng bên trong.
Bootstrap `.table-responsive` dính đúng ca này: nó khai `overflow-x: auto`, mà theo đặc tả thì một
trục là `auto` sẽ kéo trục kia từ `visible` sang `auto` — nên bảng cao 901px bị nhốt trong 122px và
có thanh cuộn riêng. Trang trông hoàn toàn bình thường.

Sửa hai dòng:

```css
.content-area      { flex: 1 0 auto; }   /* cao theo nội dung, chỉ nở thêm khi thừa chỗ */
.content-area > *  { flex-shrink: 0; }   /* con không được bóp lại */
```

Chẩn đoán trong 5 giây — so `scrollHeight` với `offsetHeight` của từng đứa con; đứa nào `scrollHeight`
lớn hơn hẳn là đứa đang bị bóp:

```js
[...document.querySelector('.content-area').children]
    .map(c => [c.className, c.offsetHeight, c.scrollHeight])
```

Và kiểm khung ngoài có cuộn được thật không: `el.scrollHeight > el.clientHeight`. Bằng nhau nghĩa là
không có gì để cuộn — nếu nội dung rõ ràng dài hơn màn hình thì nó đang bị bóp ở đâu đó.

**Đổi chỗ đặt khung cuộn thì phải soát lại JS**: script lưu/khôi phục vị trí cuộn sau F5 thường bám
vào đúng một phần tử (`document.querySelector('.main-content').scrollTop`). Chuyển `overflow` sang
phần tử khác là nó đọc/ghi `scrollTop` của một thứ không cuộn — luôn bằng 0, không lỗi, không log.

## Chạy LẠI một hoạt ảnh CSS: phải ép tính lại bố cục giữa lúc gỡ và gắn lớp

Gắn một lớp mang `animation` để "bắn" một hiệu ứng ngắn (loé sáng, nảy, rung) thì lần đầu chạy đúng,
**từ lần thứ hai trở đi im lặng không chạy** — vì lớp vẫn còn đó, mà trình duyệt chỉ khởi động hoạt
ảnh khi phần tử vừa *nhận thêm* một khai báo `animation`.

Gỡ rồi gắn lại trong cùng một khung hình cũng KHÔNG cứu được: trình duyệt gộp mọi thay đổi style của
một khung thành một lần tính lại kiểu, nên nó thấy "trước và sau đều có lớp" = không đổi gì.

```js
el.classList.remove('is-flash');
void el.offsetWidth;            // ép tính lại bố cục ngay tại đây — đây mới là dòng làm việc
el.classList.add('is-flash');
```

Đọc `offsetWidth`/`offsetHeight`/`getComputedStyle` đều ép được; giá phải trả là một lượt tính lại
bố cục, nên đừng đặt nó trong vòng chạy mỗi khung hình — chỉ gọi ở đúng khoảnh khắc cần bắn.

**Điều kiện bắn phải so với TRẠNG THÁI KHUNG TRƯỚC, không so với thời gian.** Hàm vẽ của vòng
`requestAnimationFrame` chạy ở mọi khung, nên `if (t >= mốc) bắn()` là bắn lại suốt cả quãng sau mốc:

```js
if (item.wasRunning === true && !running) flash(item);   // chỉ đúng khung chuyển trạng thái
item.wasRunning = running;
```

Khởi tạo cờ đó bằng `null` ("chưa biết") chứ không phải `false`: trang mở lên giữa chừng thì lượt vẽ
đầu chỉ ghi nhận trạng thái, không bắn hiệu ứng cho một sự kiện đã xảy ra trước khi trang tồn tại.

## `MutationObserver` không đọc được thay đổi DOM trong một vòng lặp ĐỒNG BỘ

Bàn thử kiểu "giữ callback `requestAnimationFrame` lại rồi tự drive vài chục nghìn mili-giây trong
một vòng `while`" thì callback của `MutationObserver` **không chạy lần nào** cho tới khi cả vòng đó
kết thúc — nó là microtask, xếp hàng ở cuối tác vụ. Số đo in ra toàn số 0 và trông y hệt "code không
chạy".

Cách đo đúng cho từng quãng: **xoá dấu trước quãng, drive, rồi đọc lại dấu** ngay sau đó — tất cả đều
đồng bộ nên không phụ thuộc lúc nào microtask chạy.

```js
for (const mark of marks) {
    els.forEach(e => e.classList.remove('is-flash'));   // xoá dấu của quãng trước
    drive(mark);
    log(mark, els.map(e => e.classList.contains('is-flash') ? 1 : 0).join(''));
}
```

## Bootstrap xử lý click uỷ quyền ở pha CAPTURE — rình click để chặn chuyển tab là vô ích

Muốn hỏi "bạn có thay đổi chưa lưu, vẫn chuyển tab chứ?" trước khi Bootstrap đổi pane, phản xạ tự
nhiên là bắt `click` ở pha capture trên `document` rồi `preventDefault()` + `stopPropagation()`.
**Không chạy.** `EventHandler.on(document, 'click.bs.tab', selector, handler)` của Bootstrap 5 gắn
listener với tham số thứ ba (`isDelegated`) bằng `true`, tức là **capture**, và nó gắn ngay lúc nạp
`bootstrap.bundle.js` — trước mọi file script của mình. Cùng pha capture, cùng target `document`,
listener nào đăng ký trước chạy trước ⇒ tới lượt mình thì pane đã đổi xong.

Triệu chứng đọc ra rất lệch hướng: cùng một hàm kiểm "có thay đổi chưa lưu" chạy đúng cho nhánh
bấm liên kết (hỏi bình thường) nhưng im lặng cho nhánh bấm tab, nên dễ đi tìm lỗi trong hàm so
sánh giá trị. Cách chẩn đoán dứt điểm trong một lượt: in ra trạng thái ngay trong listener của
mình — `pane.id + (form.offsetParent ? 'vis' : 'hid')` cho từng form — sẽ thấy pane cũ **đã** là
`hid` và pane mới **đã** `vis` ngay tại thời điểm listener chạy.

Cách đúng là dùng chính sự kiện của Bootstrap, nó sinh ra để cho phép chặn:

```js
document.addEventListener("show.bs.tab", function (event) {
    const target = event.target.dataset.bsTarget;      // pane sắp mở; event.relatedTarget là tab cũ
    if (!coThayDoiChuaLuu()) return;
    event.preventDefault();                            // Tab.show() kiểm defaultPrevented rồi dừng
    hoiNguoiDung().then(ok => { if (ok) { hoanTac(); bootstrap.Tab.getOrCreateInstance(nut).show(); } });
});
```

Lượt `show()` phát lại sau khi người dùng đồng ý sẽ bắn `show.bs.tab` lần nữa, nhưng lúc đó form đã
sạch nên nhánh trên cho đi qua — không cần cờ "bỏ qua một lần" nào cả.

Hai thứ đi kèm khi làm cảnh báo "chưa lưu":

- **Chỉ tính các form đang NHÌN THẤY khi chặn chuyển tab**, còn khi rời trang thì tính tất cả. Pane
  con của một pane đang ẩn (tab lồng tab) vẫn giữ class `active`, nên đừng dựa vào class — dùng
  `form.offsetParent === null` để biết nó có được vẽ hay không.
- **Khôi phục giá trị thì ĐỪNG bắn `change`.** Ô phụ thuộc (kiểu "chọn model xong tự điền số chiều")
  sẽ tính lại và ghi đè đúng thứ vừa khôi phục, làm form bẩn lại ngay sau khi vừa hoàn tác. Bắn một
  `CustomEvent` riêng để nơi khác chỉ vẽ lại phần hiển thị (menu gợi ý, dòng nhắc) là đủ.

## Cảnh báo "thay đổi chưa lưu" báo oan: trình duyệt tự điền SAU khi script chụp giá trị gốc

Trang vừa mở, chưa gõ gì, bấm sang tab khác là bị hỏi "bỏ thay đổi chưa lưu?". Thủ phạm là ô
`<input type="password">` trên trang: trình quản lý mật khẩu đổ mật khẩu đã lưu **của chính site
đó** vào ô, sau khi script đã chụp "giá trị gốc" ⇒ so sánh ra khác nhau. Hai hệ quả, cái thứ hai
mới đáng sợ:

- Cảnh báo giả ở mọi lượt chuyển tab / rời trang.
- Bấm Lưu là **ghi mật khẩu đăng nhập đè lên ô bí mật khác** (API token, secret…) mà không ai nhận ra.

Chốt chặn (làm cả hai, đừng chọn một):

- `autocomplete="new-password"` cho ô mật khẩu KHÔNG phải ô đăng nhập. Chrome **bỏ qua**
  `autocomplete="off"` trên ô password (cố tình, vì site hay lạm dụng), nhưng tôn trọng
  `new-password`. Thêm `autocomplete="off"` trên thẻ `<form>` cho các ô text đi kèm — trình duyệt
  hay đoán ô text ngay trên ô password là "tên đăng nhập" và điền luôn.
- Chụp lại giá trị gốc một lần nữa ở sự kiện `load`, trừ khi người dùng đã đụng vào form. Cờ "đã
  đụng" phải nghe `keydown`/`pointerdown`/`paste`, **không nghe `input`**: lượt tự điền cũng bắn
  `input` nên cờ sẽ bật và chốt chặn thành vô dụng.

```js
if (document.readyState !== "complete") {
    let touched = false;
    ["keydown", "pointerdown", "paste"].forEach(n => form.addEventListener(n, () => touched = true, true));
    window.addEventListener("load", () => { if (!touched) guard.markClean(); });
}
```

Kèm theo, một lỗi bố cục làm chuyện này lộ ra ở mọi tab: **thẻ `<form>` bọc NGOÀI `.tab-pane`** thay
vì nằm trong. Khi đó `form.closest(".tab-pane")` là `null` (form không thuộc tab nào) và
`form.offsetParent` không bao giờ `null` (chỉ nội dung bên trong bị ẩn), nên form đó bị tính là
"đang mở" trong mọi lượt chuyển tab. Đưa `<form>` vào trong pane là xong — HTML tương đương, mà
mọi phép tra "form này thuộc tab nào, có đang hiện không" trở lại đúng.

## Một exception khi xử lý sự kiện nuốt trọn phần còn lại của luồng SSE

Vòng đọc SSE thường có dạng `while(true) { read(); for (line of lines) { ...dispatch... } }`.
Nếu một nhánh dispatch ném lỗi (rất hay gặp: `el.querySelector(...).prop = x` khi
`querySelector` trả `null`), exception văng ra khỏi **cả vòng while**, luồng bị bỏ dở, và mọi
sự kiện phía sau — kể cả toàn bộ nội dung trả lời — không bao giờ được vẽ.

Triệu chứng đánh lừa: màn hình chỉ còn một khối rỗng, không báo lỗi gì. Nếu khối `finally` có
bước "nạp lại từ server" thì nó còn **xoá luôn** dòng báo lỗi vừa viết ra, và nạp về đúng lúc
server chưa ghi xong nên dữ liệu vẫn rỗng.

Quy tắc:
- Bọc `try/catch` quanh phần dispatch **từng sự kiện**; chỉ để sự kiện lỗi-của-server được phép
  ném ra ngoài để cắt luồng.
- Thông báo lỗi phải đặt ở chỗ **sống sót qua việc thay `innerHTML`** (toast/log), đừng chỉ viết
  vào chính khối sắp bị thay.
- Khi ghép "yêu cầu ↔ kết quả" theo một `id` do bên thứ ba cấp, ĐỪNG tra trên toàn tập: chỉ tra
  trong những mục **chưa hoàn tất**. Không có gì bảo đảm id là duy nhất, và tra trúng mục đã
  xong thường dẫn thẳng tới `null.something`.

## Gửi biểu thức JS vào CDP `Runtime.evaluate`: LUÔN dùng `String.raw`

Viết biểu thức trong **template literal thường** của Node là hỏng câm, mất rất nhiều lượt mới nhận ra
vì kết quả trả về trông "gần đúng":

- `` `...replace(/\s+/g,' ')...` `` → trang nhận regex nuốt đúng **chữ `s`**: `dashboard` đọc ra
  `da hboard`, `Videos` ra `Video `, `search` ra ` earch`. Rất dễ kết luận nhầm là trang lỗi
  font/ligature hoặc ô nhập bị mất ký tự.
- `` `...replace(/[ \t\n]+/g,' ')...` `` → `\n` thành **xuống dòng thật** nằm giữa regex literal ⇒
  `SyntaxError`. `Runtime.evaluate` **không ném**: nó trả `exceptionDetails` mà nếu không đọc trường
  đó thì chỉ thấy giá trị `undefined` — im lặng hoàn toàn.

Cách đúng: `await ev(String.raw\`...\`)`, và trong hàm bọc luôn kiểm `result.exceptionDetails` rồi in
ra thay vì trả `undefined`.

- `Hàm bọc đọc thuộc tính của kết quả sẽ biến thành công thành lỗi khi action quên return`: Một helper kiểu `async function run(btn, action) { try { const p = await action(); toast(p.message || "Đã xong", "success"); } catch (e) { toast(e.message, "error"); } }` sẽ báo LỖI cho một thao tác đã chạy xong, nếu `action` là `async function () { await post(...); }` — không `return` thì nó resolve về `undefined`, `p.message` ném `TypeError`, và chính khối `catch` ngay dưới biến TypeError đó thành thông báo lỗi cho người dùng.
    + Why: triệu chứng đánh lừa hoàn toàn — server nhận request, ghi database, trả 200, mà giao diện vẫn đỏ. Đi soi log server thì thấy sạch, nên rất dễ đâm đầu vào tầng API trong khi lỗi nằm ở đúng một dòng JS phía trình duyệt. Càng khó thấy khi vài chỗ gọi khác trong repo lại `return post(...)` nên chạy đúng, và khi sau lời gọi có `reload()`/điều hướng che mất toast.
    + How: **guard trong helper** (`(p && p.message)`) thay vì trông vào việc mọi bên gọi nhớ return — helper là một chỗ, bên gọi là hàng chục. Song song đó vẫn sửa bên gọi cho trả về payload để lấy được câu báo thật của máy chủ.
    + Nhận diện nhanh: thao tác "vừa báo lỗi vừa có tác dụng thật" gần như luôn là lỗi ở tầng hiển thị kết quả, không phải ở tầng gọi. Mở DevTools xem tab Network: request 200 mà UI đỏ thì thủ phạm nằm sau `await`.

- `Giả lập DOM trong Node để test code trình duyệt: navigator KHÔNG gán đè được`: Từ Node 21,
  `globalThis.navigator` đã có sẵn và là thuộc tính CHỈ ĐỌC. `global.navigator = { mediaDevices: ... }`
  trong file không `"use strict"` **im lặng không ăn** — không lỗi, không cảnh báo, chỉ là câu gán
  bốc hơi. Code đang test rẽ vào nhánh "trình duyệt không hỗ trợ" rồi thoát sớm, và mọi assert sau đó
  fail theo kiểu vô nghĩa (thành phần chưa bao giờ được dựng).
    + Why: `window`, `document`, `MediaRecorder` gán đè bình thường vì Node không định nghĩa chúng —
      nên ta tin luôn là `navigator` cũng vậy. Đúng một biến trong đống stub cư xử khác cả đám.
    + How: `Object.defineProperty(globalThis, "navigator", { value: {...}, configurable: true })`.
      Nghi ngờ thì kiểm bằng `Object.getOwnPropertyDescriptor(globalThis, "navigator")` trước khi gán,
      hoặc bật `"use strict"` cho chính file harness để câu gán hỏng NÉM ra thay vì im lặng.

- `id của requestAnimationFrame/setTimeout có thể là 0, đừng kiểm bằng chân trị`: `if (frameId)
  cancelAnimationFrame(frameId)` bỏ sót đúng trường hợp id = 0, và vòng vẽ cứ chạy tiếp sau khi đã
  "dừng". Trình duyệt thật hiếm khi cấp id 0 nên lỗi này ngủ yên rất lâu, rồi lộ ra ở môi trường
  khác (harness test, polyfill, worker) — chỗ mà id được đánh số từ 0.
    + How: giữ biến ở `null` khi rảnh và kiểm `if (id !== null)`. Cùng lý do cho `setTimeout`,
      `setInterval`, `IntersectionObserver`… mọi handle dạng số do môi trường cấp.

## Phẳng hoá DOM thành chuỗi để tìm chữ: hỏi KHỐI chứa mẩu text, không hỏi thẻ cha

Gom text node thành một chuỗi rồi `indexOf` là cách thông dụng để định vị một đoạn chữ trong trang
đã render. Chỗ phải chèn khoảng trắng là ranh giới khối (`</h2><p>` viết dính nhau thì không chèn sẽ
thành `lắp đặtthiết bị`). Phản xạ tự nhiên là hỏi `node.parentElement` có nằm trong danh sách thẻ
inline không — **và nó sai ở đúng trường hợp phổ biến nhất**: trong `<p>…<strong>X</strong>, kèm…</p>`
thì mẩu `","` có cha là `<p>`, một thẻ khối, nên bị kết luận "vừa qua ranh giới khối" và nhận thêm
một khoảng trắng. Chuỗi ra `x , kèm`, và **mọi câu có chữ in đậm/nghiêng ở giữa đều không tìm thấy**.

- Why: thẻ cha trả lời "mẩu này nằm trực tiếp trong thẻ gì", còn câu đang hỏi là "mẩu này và mẩu
  trước có cùng một khối không". Hai câu khác nhau, trùng đáp án ở mọi ví dụ đơn giản nên lỗi ngủ rất
  lâu; nó chỉ lộ ra khi có thẻ inline ở GIỮA một khối.
- How: tính khối của mẩu = tổ tiên gần nhất không phải thẻ inline, nhớ khối của mẩu trước, chỉ chèn
  khoảng trắng khi khối đổi.

```js
function blockOf(node, root) {
    let el = node.parentElement;
    while (el && el !== root && INLINE_TAGS.has(el.tagName)) el = el.parentElement;
    return el;
}
```

- Tiện thể: giữ luôn tập vị trí ranh giới khối vào kết quả. Muốn cắt "một câu" quanh chỗ khớp thì chỉ
  dò dấu `[.!?:;]` là không đủ — tiêu đề mục và từng dòng của danh sách không có dấu chấm nào, phần
  cắt ra sẽ nuốt cả tiêu đề nằm trên nó.

## `Range.surroundContents` nhiều khoảng trong CÙNG một text node: bọc từ cuối về đầu

Bọc `<mark>` quanh nhiều khoảng rời nhau (ví dụ tô cả đoạn, trong đó một câu tô đậm hơn) thì một text
node có thể phải cắt làm hai, ba mảnh. Đi xuôi từ đầu là hỏng: `surroundContents` cắt node, mảnh
**trước** chỗ cắt ở lại đúng node cũ còn mảnh sau thành node mới, nên mọi offset phía sau mà ta đã
tính từ trước đều trỏ sai chỗ.

- How: thu thập hết các khoảng trước, rồi bọc theo thứ tự **giảm dần** vị trí. Bọc mảnh cuối không
  đụng gì tới offset của mảnh đầu trong cùng node đó.
- Kèm theo: bọc theo TỪNG text node chứ đừng dựng một range dài vắt qua nhiều thẻ —
  `surroundContents` ném `InvalidStateError` khi range cắt ngang ranh giới thẻ.

## DOMPurify XOÁ nguyên một thuộc tính nếu giá trị chứa `-->`

Sự cố (2026-09-17, k-rag-platform): nhét nguồn của khối ```mermaid vào `data-chart-src="..."` rồi
`DOMPurify.sanitize()` — thuộc tính biến mất sạch khỏi DOM. Chỗ trống rỗng, **không lỗi, không cảnh
báo, console im hoàn toàn**, các thuộc tính `data-*` khác trên cùng thẻ vẫn còn nguyên. Mất khá lâu
mới nhìn ra vì khối ```chart (không có mũi tên) chạy tốt ngay từ lượt thử đầu.

- Why: từ 3.1.3 DOMPurify có chốt SAFE_FOR_XML chống mXSS — giá trị thuộc tính chứa `-->` (hoặc
  `]>`, `<!--`, `</style`, `</title`) bị coi là mưu toan đóng một chú thích HTML giả để phần sau
  thoát ra ngoài ngữ cảnh thuộc tính. Nó không lọc ký tự mà **bỏ cả thuộc tính**. Mà `-->` chính là
  mũi tên của mọi flowchart mermaid, và cũng gặp trong bất kỳ đoạn mã/chữ nào có dấu ấy.
- How: đừng ghi chữ thô do người/mô hình sinh ra vào thuộc tính rồi mới sanitize. Mã hoá trước
  (`encodeURIComponent` lúc ghi, `decodeURIComponent` lúc đọc) — vừa diệt `-->` vừa diệt luôn
  chuyện xuống dòng và dấu nháy trong thuộc tính.
- Cách nhận ra nhanh: sanitize thử một chuỗi có `-->` và một chuỗi không có, so hai kết quả. Nếu chỉ
  bản có mũi tên mất thuộc tính thì đúng chốt này, không phải lỗi allowlist.

## Ký tự U+0000 lọt vào tệp .js làm mọi công cụ coi tệp là "binary"

Viết escape của U+0000 trong nội dung truyền cho công cụ ghi tệp thì cái lọt xuống đĩa là **ký tự
NUL thật**, không phải chuỗi escape sáu ký tự. Trình duyệt và `node --check` vẫn chạy bình thường
(NUL trong string literal là hợp lệ), nhưng `file` báo "data", `grep` báo "Binary file matches" và
`git diff` không hiện nội dung — rất mất thời gian nếu tưởng tệp hỏng.

- How: sinh escape ấy bằng cách nối chuỗi trong script (dấu gạch chéo ngược + "u0000"), hoặc dùng
  `String.fromCharCode(0)` ngay trong mã. Sửa tệp đã lỡ: đọc bằng `fs.readFileSync(p,"utf8")`, tách
  theo `String.fromCharCode(0)` rồi nối lại bằng chuỗi escape, ghi lại. Kiểm bằng `file <tệp>` —
  phải ra "JavaScript source, UTF-8 text".
- Cùng lý do đó, chuỗi công cụ gửi đi cũng không chứa được escape ấy ở dạng nguyên văn: lệnh bị từ
  chối với "command contains control characters".

## `setPointerCapture` đổi đích của `pointerup` — đừng dùng `event.target` ở đó

Sau `el.setPointerCapture(id)`, mọi sự kiện của con trỏ đó (move/up/cancel) đều có `target` là `el`,
không phải phần tử nằm dưới con trỏ. Kiểu "bấm ra nền thì đóng" viết `if (!content.contains(e.target))`
ở `pointerup` sẽ đóng cả khi bấm vào chính nội dung. Ghi `event.target` lúc `pointerdown` (trước khi
capture) rồi xét biến đó.

## Lớp phủ tự dựng mở từ trong modal Bootstrap 5

Bootstrap không chồng được hai modal: focus trap của modal đang mở kéo focus về khi focus rời nó. Lớp
phủ tự dựng thì gắn VÀO phần tử `.modal` (không phải `.modal-dialog` — nó mang `transform`, biến
`position: fixed` của con thành fixed theo dialog), và ở `keydown` Escape phải `stopPropagation()`,
không thì Bootstrap nhận Esc và đóng luôn modal bên dưới.
