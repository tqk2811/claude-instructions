# Làm việc với Chrome (headless, chụp màn hình, đo bằng CDP)

Máy này là **máy người dùng đang dùng thật**, Chrome cá nhân của họ gần như luôn đang mở với một
đống tab công việc. Mọi luật dưới đây tồn tại vì lý do đó.

---

## 1. TRƯỚC KHI GIẾT BẤT KỲ TIẾN TRÌNH CHROME NÀO: KIỂM ĐƯỜNG DẪN PROFILE

**Luật bắt buộc, không có ngoại lệ:** không bao giờ giết Chrome theo **tên ảnh**. Phải liệt kê ra,
đọc `--user-data-dir` của từng tiến trình, và chỉ giết những tiến trình mang **đúng thư mục profile
mình đã tạo trong scratchpad**.

Cấm tuyệt đối:

```bash
taskkill //F //IM chrome.exe //T          # ❌ giết luôn Chrome cá nhân của người dùng
```

`/IM` nhắm theo tên tiến trình, mà Chrome cá nhân và Chrome headless đều là `chrome.exe`. Không có
cảnh báo, không có đường hoàn tác — người dùng mất sạch tab đang làm dở và phải tự bấm "Khôi phục".

### Bước 1 — LIỆT KÊ và ĐỌC trước (bắt buộc, kể cả khi "chắc chắn")

```powershell
Get-CimInstance Win32_Process -Filter "Name='chrome.exe'" |
  Select-Object ProcessId, @{n='Profile';e={
      if ($_.CommandLine -match '--user-data-dir=("[^"]+"|\S+)') { $Matches[1] } else { '<PROFILE CÁ NHÂN>' } }} |
  Format-Table -AutoSize
```

Tiến trình **không** có `--user-data-dir` chính là Chrome cá nhân của người dùng. Thấy dòng nào như
vậy mà mình vẫn định giết theo tên ảnh thì dừng lại.

### Bước 2 — chỉ giết cái mang đúng profile của mình

```powershell
$mine = 'C:\...\scratchpad\cprof'          # đúng thư mục MÌNH đã truyền cho --user-data-dir
Get-CimInstance Win32_Process -Filter "Name='chrome.exe'" |
  Where-Object { $_.CommandLine -like "*$mine*" } |
  ForEach-Object { Stop-Process -Id $_.ProcessId -Force }
```

Chuỗi lọc phải là **đường dẫn profile**, không phải `chrome`, không phải `headless` — người dùng có
thể cũng đang mở một cửa sổ headless của việc khác.

### Bước 3 — tốt nhất là đừng giết gì cả

`--dump-dom` và `--screenshot` **tự thoát** khi xong. Còn tiến trình sống sót thường nghĩa là lệnh
bị treo, mà cách gỡ đúng nằm ở mục 2 chứ không phải giết.

---

## 2. Headless bị treo tới hết timeout — KHÔNG phải lý do để giết tiến trình

Hai nguyên nhân thật, cả hai đều gỡ được mà không đụng tới tiến trình nào:

1. **`--user-data-dir` đang bị khoá** bởi một tiến trình Chrome khác.
   → Đổi sang **thư mục profile MỚI** (`cprof2`, `cprof3`, …) rồi chạy lại. Xong ngay.
   Phản xạ "giết hết cho sạch khoá" vừa thừa vừa phá — đã vấp thật 2 lần trong một ngày.
2. **`--screenshot` không ghi được file**: đường dẫn tương đối trên Windows hay rơi vào thư mục
   không có quyền, báo `Access is denied` rồi treo.
   → Luôn truyền **đường tuyệt đối kiểu Windows**: `--screenshot=C:\...\out.png`.

---

## 3. Quy ước khởi động

```bash
"/c/Program Files/Google/Chrome/Application/chrome.exe" \
  --headless --disable-gpu --no-sandbox \
  --user-data-dir="C:\...\scratchpad\cprof" \    # BẮT BUỘC: hồ sơ riêng, và là dấu nhận dạng để dọn
  --virtual-time-budget=6000 \
  --window-size=1942,1234 \
  --screenshot="C:\...\scratchpad\out.png" \      # tuyệt đối, kiểu Windows
  "file:///C:/.../trang.html"
```

- `--user-data-dir` riêng trong scratchpad là **bắt buộc**, kể cả cho một lệnh chạy một lần: nó vừa
  tách khỏi hồ sơ người dùng, vừa tạo sẵn dấu nhận dạng để lọc lúc dọn.
- `--allow-file-access-from-files` khi trang `file://` cần nạp file khác (CSS, font, ảnh).
- `--virtual-time-budget` **giết `requestAnimationFrame`** — đừng dùng khi đang đo hoạt ảnh/nhịp
  thời gian thật; lúc đó phải đo qua CDP với thời gian thật.

### `--window-size` KHÔNG bằng viewport (Windows)

Viewport thật nhỏ hơn cửa sổ đúng **22 × 154 px** (bề rộng × chiều cao), cố định ở mọi cỡ. Muốn
viewport đúng `1920×1080` thì truyền `--window-size=1942,1234`. **Luôn in `innerWidth`/`innerHeight`
ra để đối chiếu** — sai chỗ này là mọi số đo theo `vh`/`dvh` sai theo mà trang trông vẫn bình thường
(đã vấp: đo `/Board` ra 2 dòng chữ trong khi thật ra là 3).

`--headless` (cổ điển) và `--headless=new` cho cùng delta đó; `=new` cần thiết khi trang dùng API
đời mới.

### Windows chặn cửa sổ hẹp hơn ~500px — muốn chụp cỡ điện thoại phải dùng iframe

`--window-size=434,1006` **không** cho viewport 412: Windows có bề ngang cửa sổ tối thiểu, thực tế
`innerWidth` ra **497**. Tệ hơn: `--screenshot` vẫn xuất ảnh đúng 434px, nên ảnh trông y như trang
bị **tràn ngang** (chữ và thẻ bị cắt ở mép phải) trong khi bố cục hoàn toàn bình thường — rất dễ đi
sửa một cái lỗi không tồn tại.

Cách đúng để xem bố cục ở 320–500px: bọc trang trong một file khác bằng iframe cố định bề ngang rồi
chụp file bọc.

```html
<!doctype html><meta charset=utf-8>
<style>html,body{margin:0}iframe{border:0;display:block;width:360px;height:2400px}</style>
<iframe src="index.html" scrolling="no"></iframe>
```

`height` lớn + `scrolling="no"` cho luôn ảnh cả trang (`--screenshot` chỉ chụp phần thấy được), khỏi
phải cuộn. Cửa sổ chụp đặt `--window-size=<w+40>,<h+40>`.

---

## 4. Đo đạc trong trang: hai cái bẫy im lặng

- **Canvas KHÔNG tự kích hoạt tải webfont.** Không phần tử DOM nào dùng font thì `ctx.font = '... "X"'`
  lặng lẽ rơi về font hệ thống, và **hai font khác nhau đo ra hai bộ số giống hệt nhau** — trông rất
  thuyết phục mà vô nghĩa. Phải `await document.fonts.load('<weight> <size>px "X"', chuỗi_mẫu)` cho
  **từng** font trước khi đo, và luôn cài một chốt chặn: hai font đo ra cùng bề rộng thì báo lỗi chứ
  đừng in bảng số ra.
- **`document.fonts.ready` không đủ** — nó chỉ đợi những font đang *được dùng*.
- **Headless báo `prefers-reduced-motion: reduce`** (cả `--headless` cổ điển lẫn `--headless=new`, không
  có cờ tắt). Trang nào có khối `@media (prefers-reduced-motion: reduce)` tắt hoạt ảnh thì ảnh chụp
  ra đúng nhánh "đã tắt" — hoạt ảnh biến mất sạch, `getComputedStyle(...).animationName` ra `none`,
  và rất dễ kết luận là CSS hoạt ảnh vừa viết bị sai. Cách kiểm hoạt ảnh: trong TRANG THỬ (không sửa
  CSS thật) ghi đè lại bằng một khối `@media (prefers-reduced-motion: reduce)` bật trở lại, rồi ghim
  từng pha bằng `animation-play-state: paused` + `animation-delay` ÂM (mỗi pha một class) và chụp một
  ảnh duy nhất thấy đủ các pha — khỏi phải đo thời gian thật. Chấm/vòng cỡ vài px thì chụp kèm
  `--force-device-scale-factor=4` mới nhìn ra.
- **`--screenshot` có thể chụp KHUNG HÌNH TRƯỚC lượt `requestAnimationFrame`** của trang. Trang nào
  đo bố cục rồi tự sửa lại trong rAF (kiểu "đo chỗ thật rồi ghi số dòng chữ xuống CSS") thì ảnh chụp
  hiện trạng thái CHƯA sửa, trong khi DOM đọc ra đã là trạng thái sửa rồi — nhìn ảnh mà kết luận là
  sai, và mất rất nhiều lượt để nhận ra vì hai nguồn đều "có vẻ đúng". Cách gỡ: (1) lấy số đo bằng
  `--dump-dom` (ghi kết quả vào một thuộc tính `data-` của `<html>`, và BẮT BUỘC kèm
  `--virtual-time-budget`, không thì dump chạy xong trước cả `setTimeout` đầu tiên); (2) kiểm bằng
  mắt trên một trang **tĩnh, không JS**, đặt cứng đúng giá trị mà script sẽ tính ra. Hai nguồn khớp
  nhau mới tin.
- Đo chiều cao mực thật (ink) thì `measureText().actualBoundingBoxAscent` **bị làm tròn về số
  nguyên px**; ở cỡ chữ nhỏ sai số đủ để đảo ngược kết luận. Đo ở cỡ lớn (200px) rồi quy về `em`,
  hoặc quét pixel bằng `getImageData`.

---

## 5. Nguyên tắc chung

Trang chụp ra ảnh rồi **tự mình đọc lại ảnh đó**, đừng tin là nó đúng vì lệnh chạy không lỗi. Rất
nhiều lỗi bố cục (thiếu class gốc nên font không áp, chữ bị cắt, ô trống) chỉ lộ ra khi nhìn.
