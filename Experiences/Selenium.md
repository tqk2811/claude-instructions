# Kinh nghiệm: tự động hoá trình duyệt (Selenium / CDP / Chrome)

Bài học chung, không phụ thuộc project. Đọc trước khi lái trình duyệt bằng code.

## 1. `isTrusted = true` CHƯA đủ — trang còn xem cả quãng đường trước cú bấm

Các trang có token chống lạm dụng (họ BotGuard của Google — chuỗi mờ vài KB mở đầu bằng `!`, gửi kèm
mỗi request) **gom dấu vết tương tác chuột/phím**, không chỉ xem sự kiện có "thật" hay không.

Ba mức, từ dễ bị chặn tới khó:

1. `el.dispatchEvent(new MouseEvent('click'))` — `isTrusted = false`, chặn dễ nhất.
2. CDP `Input.dispatchMouseEvent` bắn **đúng một** `mousePressed`/`mouseReleased` tại toạ độ đích —
   `isTrusted = true` nhưng con trỏ **chưa hề di chuyển**. Vẫn bị chặn.
3. CDP `Input.dispatchMouseEvent` với **cả chuỗi `mouseMoved`** dẫn tới đích rồi mới nhấn. Qua được.

Mức 3 làm thế nào: đường cong Bézier bậc 3 (2 điểm điều khiển lệch **vuông góc** với đường nối), tiến
độ theo ease-in-out chứ không tuyến tính, rung tay biên độ nhỏ và **tắt dần khi gần đích**, ~35% số cú
bấm thì vọt quá đích rồi chỉnh lại, khoảng nghỉ giữa các bước ngẫu nhiên (4–15ms, thỉnh thoảng khựng
30–70ms), và **điểm bấm lệch ngẫu nhiên quanh tâm** chứ không đúng tâm.

Triệu chứng khi thiếu: máy chủ trả `403` với thông điệp kiểu *"The caller does not have permission"*.
**Đừng tin chữ nghĩa của thông điệp đó** — nó nghe như tài khoản thiếu quyền, nhưng cùng tài khoản bấm
tay thì được.

## 2. Loại trừ một biến thì phải chắc thí nghiệm THỰC SỰ chứa biến đó

Sai lầm đã mắc: thử `Input.dispatchMouseEvent` (chuột thật qua CDP) → vẫn 403 → kết luận "chuột không
phải nguyên nhân" → đi lạc cả buổi sang giả thuyết khác (schema, throttle). Thực ra thí nghiệm đó chỉ
có **cú nhấn**, không có **quãng đường di chuyển** — tức chưa hề chạm vào biến định loại trừ.

Trước khi viết "đã loại trừ X", hỏi lại: *phép thử vừa rồi có thật sự chứa X ở dạng đầy đủ không?*

## 3. So hai payload mã hoá thì KHÔNG suy ra được biến nào khác nhau

Token chống lạm dụng đã mã hoá ⇒ đổi một bit đầu vào là cả chuỗi đổi. Thấy hai token "khác nhau ngay
từ ký tự thứ 2" là chuyện đương nhiên, **không** nói lên tín hiệu nào gây khác biệt. Chỉ thí nghiệm
đối chứng (đổi đúng một biến, giữ nguyên phần còn lại) mới trả lời được.

## 4. Luôn kẹp một lượt làm TAY ngay cạnh lượt tự động

Không có đối chứng thì không phân biệt được "code sai" với "tài khoản/phiên đang bị chặn". Đo cách
nhau vài giờ cũng vô nghĩa vì trạng thái phía máy chủ đổi.

## 5. Kiểm `elementFromPoint` trước khi nhấn

Bơm chuột thật có một điểm hơn hẳn `dispatchEvent`: lớp phủ còn sót (`cdk-overlay-backdrop` của hộp
thoại chưa đóng…) sẽ **nuốt** cú bấm. `dispatchEvent` thì không thấy gì cả và báo "thành công" trong
khi trang chẳng nhận được gì. Nên trước khi nhấn, kiểm `document.elementFromPoint(x, y)` có phải phần
tử đích (hoặc con/cha của nó) không — chấp nhận cả con lẫn cha vì Material hay phủ lớp ripple lên trên.

## 6. Vài bẫy Chrome/CDP hay gặp

- **Preflight `OPTIONS` cùng URL và kết thúc TRƯỚC** request thật, mà nó không có thân phản hồi ⇒ lọc
  request theo URL mà quên lọc `POST` thì vòng chờ vớ ngay phải nó rồi báo "không đọc được".
- **Cửa sổ bị che hoàn toàn ⇒ `document.visibilityState === "hidden"`** (Chrome theo dõi occlusion ở
  tầng OS, không phải minimize). Tắt bằng
  `--disable-features=CalculateNativeWinOcclusion --disable-backgrounding-occluded-windows`.
- **`Network.getResponseBody` phải gọi NGAY khi request kết thúc** — Chrome chỉ giữ nội dung một lúc.
- **Gán thẳng `input.value` thì framework (Angular/React) không biết** — nó nghe sự kiện chứ không
  theo dõi property. Phải gọi setter gốc trên prototype rồi tự phát `input`/`change`.
- **`element.click()` không đủ cho nhiều component Material** — chúng nghe `pointerdown`. Cần cả chuỗi
  `pointerdown → mousedown → pointerup → mouseup → click`.
- **Chrome đời mới khoá cookie theo app-bound encryption** (tiền tố `v20`, khoá buộc vào chữ ký của
  chính `chrome.exe`) ⇒ trình duyệt nhúng (CEF/CefSharp) **không đọc được profile Chrome sẵn có**.

## 7. Chụp màn hình bằng `--headless` để kiểm giao diện

- **Luôn truyền `--user-data-dir` và đường dẫn TUYỆT ĐỐI cho `--screenshot`.** Thiếu một trong hai
  thì Chrome trên Windows báo `Failed to write file ...: Access is denied. (0x5)` — nghe như lỗi
  quyền thư mục, nhưng thư mục vẫn ghi được bằng mọi cách khác.
- **Chrome trên Windows kẹp bề rộng cửa sổ ở khoảng 512px.** `--window-size=390,844` vẫn cho ra
  ảnh 390px, nhưng trang bên trong **dàn ở 512px rồi bị cắt** cho vừa khung ảnh. Nhìn ảnh đó ra
  đúng triệu chứng "tràn ngang trên điện thoại" trong khi trang hoàn toàn không tràn. Cả
  `--headless=new` lẫn `--headless=old` đều thế. Muốn bề rộng nhỏ thật thì phải dùng CDP
  `Emulation.setDeviceMetricsOverride`, không phải `--window-size`.
- **Đừng đọc bố cục bằng mắt qua ảnh — đo số.** Lưu HTML về thư mục tạm, chèn một `<script>` ghi
  `scrollWidth`/`clientHeight`/`getBoundingClientRect()` vào `document.title`, rồi đọc bằng
  `--dump-dom --virtual-time-budget=3000`. Rẻ hơn và trả lời đúng câu đang hỏi ("có tràn không",
  "khung cuộn có cuộn được không") thay vì để mình đoán từ vài pixel ở mép ảnh.

## 8. Sự kiện chuột bơm bằng CDP: THIẾU `force` là bị nhận ra ngay

Đo trên Google Flow ngày 24/08/2026, nhưng đây là bài học chung cho mọi trang có chốt chống bot:

- **`Input.dispatchMouseEvent` mặc định `force = 0`**, nên mọi `pointerdown` bơm bằng CDP ra
  `event.pressure === 0`, trong khi **chuột thật nhấn luôn cho `0.5`**. Một phép so bằng là lọc sạch
  bot, và có trang lọc thật (trả về "We noticed some unusual activity", không phải lỗi HTTP).
  ⇒ Luôn khai `force: 0.5` cho `mousePressed`, `0` cho `mouseMoved`/`mouseReleased`.
- Triệu chứng rất dễ đổ tội nhầm: cùng trình duyệt / tài khoản / IP, người bấm tay thì chạy, script
  bấm thì hỏng — nên dễ kết luận "tài khoản bị khoá" hoặc "trang đổi giao diện". Cách phân biệt rẻ
  nhất: nhờ người dùng bấm tay một lượt xen giữa hai lượt script.
- `event.isTrusted === true` **không cứu được gì**: sự kiện qua CDP vẫn `isTrusted` nhưng vẫn bị lọc
  vì các trường vật lý (pressure, tilt) sai.
- **`twist` phải là int32**; truyền `0.5` là CDP trả
  `Failed to deserialize params.twist - BINDINGS: int32 value expected`. `tiltX/tiltY` nhận số thực.
- **Đừng khai `movementX/movementY`** — không có trong tham số của lệnh; Chrome tự tính hiệu hai bước
  liên tiếp (bước đầu ra 0, các bước sau ra đúng khoảng dịch). `screenX/screenY` Chrome cũng tự điền
  đúng toạ độ màn hình thật.
- Bàn phím: `Input.dispatchKeyEvent` thiếu `code` thì `event.code` ra **chuỗi rỗng** — thứ không bàn
  phím thật nào tạo ra. Còn `Input.insertText` không sinh **một sự kiện phím nào**; ở ca đo trên nó
  không bị chặn, nhưng nó vẫn là dấu vết rẻ nhất mà một trang có thể bắt.

## Playwright kênh `chromium` đầy đủ trên máy có IDM: mất sự kiện Download, IDM bật hộp tải
- **Triệu chứng (26/9/2026, bộ test E2E):** đổi bộ test từ `chromium-headless-shell` (mặc định của headless) sang kênh `chromium` đầy đủ thì test chờ `page.RunAndWaitForDownloadAsync` treo hoặc hỏng. Cùng lúc Internet Download Manager hiện hộp bắt tải cho URL `http://127.0.0.1:<port>/...` của máy chủ giả.
- **Nguyên nhân (suy luận, khớp triệu chứng):** bản đầy đủ chạy bằng `chrome.exe`, nên IDM gắn được vào tiến trình và cướp lượt tải. `chrome-headless-shell.exe` không bị gắn.
- **Cách làm:** giữ headless-shell cho bộ chung. Chỉ test nào bắt buộc cần bản đầy đủ (vd `getUserMedia` với micro giả, headless-shell báo `NotSupportedError`) mới chạy trên một browser riêng, và những test đó không được tải tệp. Thấy test tải tệp hỏng mà máy có IDM thì nghi IDM trước.
