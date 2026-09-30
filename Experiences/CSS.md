# Kinh nghiệm CSS

## Viết `*` + `/` vào giữa một chú thích CSS là giết luôn khối luật ngay dưới nó

**Đã vấp thật BA lần trong cùng một file** (`award-screen.css` của một dự án), lần thứ hai là do
chính câu cảnh báo viết cho lần thứ nhất — nó nhắc "đừng viết dấu đóng chú thích" bằng cách viết ra
đúng dấu đó, có bọc backtick.

Lần thứ ba (2026-08-15) là một dạng khác và không dính gì tới việc *nói về* dấu ấy: **chèn thêm một
đoạn chú thích vào ngay sau dấu đóng của khối chú thích đang có**, kiểu "thêm mấy dòng giải thích
phía dưới". Đoạn mới không có dấu mở của riêng nó, nên nó thành prelude của selector kế tiếp và cả
khối luật ngay dưới bị bỏ. Rút ra: mỗi lần chèn chữ vào một khối chú thích sẵn có thì **chèn vào
BÊN TRONG** (trước dấu đóng), và chạy dòng `rg` bên dưới ngay sau **mỗi** lần sửa file CSS chứ không
phải để tới cuối buổi — nó chạy mất một giây, còn triệu chứng thì đọc thành "bố cục sai".

CSS **không có escape trong chú thích và không biết backtick là gì**. Trình phân tích thấy dấu đóng
đầu tiên là đóng ngay tại đó, phần chữ còn lại trở thành **prelude của selector kế tiếp** (gộp với
selector thật thành một chuỗi vô nghĩa), và trình duyệt bỏ **CẢ khối luật** đó. Không lỗi, không
cảnh báo, không có gì trong console. Linter/bundler cũng không kêu, vì về mặt cú pháp đây là một
selector không khớp gì cả — hoàn toàn hợp lệ.

Cùng cái bẫy: chú thích **lồng nhau** không tồn tại trong CSS. Bọc một đoạn đã có chú thích vào một
chú thích lớn hơn là dấu đóng bên trong cắt cụt cả khối bao ngoài.

**Cách viết đúng khi cần nói về dấu đó trong chú thích**: diễn đạt bằng lời ("dấu đóng chú thích",
"sao rồi gạch chéo"), hoặc tách chữ ra (`* /`). Đừng tin vào backtick, dấu nháy hay `\`.

### Vì sao khó lần ra

Triệu chứng hiện ra ở **luật bị nuốt**, cách chỗ sai vài chục dòng, và thường đọc thành "bố cục
sai" chứ không thành "CSS lỗi". Lần vấp thật gần nhất: khối bị nuốt mang
`container-type: inline-size`, mà **mất container thì mọi đơn vị `cqw`/`cqh` bên trong rơi về kích
thước VIEWPORT** — một `max-height: 100cqw` (ý là "cao không quá bề ngang thẻ") lặng lẽ thành
`max-height: 1920px`, tức là không còn trần nào. Đi tìm ở luật `max-height`, ở flexbox, ở markup —
đúng những chỗ không sai.

### Cách bắt nó trong một phút, không cần đoán

Đừng đọc CSS bằng mắt. Dựng một trang `file://` nhỏ **nạp đúng file CSS thật** cùng đoạn markup thật,
rồi đo bằng headless Chrome:

```js
// in ra kích thước THẬT của phần tử nghi ngờ + một cờ so sánh với kỳ vọng
var r = el.getBoundingClientRect();
```

Số đo lệch hẳn kỳ vọng (ảnh cao 458px trong khi trần phải là 344px) là bằng chứng luật **không hề
được áp**, khác hẳn với "luật áp nhưng giá trị sai". Từ đó tìm ngược lên trên là ra ngay chú thích
hỏng. Kiểm nhanh trong trình duyệt: `getComputedStyle(el).paddingTop` — khối bị nuốt thì **mọi**
thuộc tính trong nó về giá trị mặc định cùng lúc, đó là dấu hiệu nhận dạng.

Grep cũng bắt được phần lớn ca này:

```
rg '\*/.+\S' -g '*.css'     # dấu đóng chú thích mà PHÍA SAU còn chữ
```

## Đơn vị container query (`cqw`, `cqi`, `cqh`) không có container thì rơi về VIEWPORT

Không phải lỗi, là quy định của spec (rơi về "small viewport"). Hệ quả thực tế: một trần khai bằng
`100cqw` mất container sẽ **nới ra gấp nhiều lần** thay vì mất tác dụng — im lặng và trông rất giống
"trần này bị ai đó gỡ". Ba nguyên nhân hay gặp: khối khai `container-type` bị nuốt (mục trên), khai
nhầm cấp (đặt trên chính phần tử đang dùng đơn vị — nó tra ancestor, không tra chính mình), hoặc
`container-type` bị một luật khác đè.

Khi trần chiều cao là thứ giữ cho bố cục không vỡ, luôn giữ thêm một dòng dự phòng **ngay trước**:

```css
max-height: clamp(3.5rem, 22vh, 16rem);   /* trình duyệt không hiểu cqw sẽ dừng ở dòng này */
max-height: 100cqw;
```

## Flexbox: muốn một khối co lại được thì `min-height: 0` là bắt buộc

Flex item mặc định có kích thước tối thiểu **bằng nội dung** (`min-height: auto` ở cột dọc,
`min-width: auto` ở hàng ngang), nên `flex-shrink` không có tác dụng gì cho tới khi hạ nó xuống `0`.
Không có gì báo — khối chỉ đơn giản là tràn ra ngoài khung cha.

Kèm theo: phép co **chỉ chạy khi khung cha có chiều cao xác định**. Trong lưới `flex-wrap`, chiều cao
mỗi hàng do chính nội dung quyết định, nên không có gì để co — phải kẹp bằng `max-height` (phần trăm
tính theo chiều cao của flex container, và nó xác định nếu container được `stretch`) trước đã.

Muốn một khối bị **cắt** chứ không bị **bóp méo** thì khung cắt giữ `display: block` +
`overflow: hidden`. Đổi khung đó sang `display: flex` là đứa con (ảnh chẳng hạn) thành flex item và
bị nén theo chiều dọc — ra một tấm ảnh sai tỉ lệ, trông giống lỗi file ảnh chứ không giống bố cục.

---

## Đo bố cục bảng bằng headless Chrome: ba cái bẫy im lặng

Bối cảnh: chia lại bề rộng cột của một bảng chiếu lên máy chiếu, mỗi cột một trần cỡ chữ.

1. **Trần theo `vw` không nhìn thấy cột hẹp đi.** `vw` đo *cửa sổ*, không đo *cột*. Đổi tỉ lệ cột
   (thêm một cột chẳng hạn) là chữ trong cột hẹp nhất âm thầm bị cắt trên đúng cái màn hình vừa
   chạy tốt hôm qua. Trần của một chuỗi nằm trong cột phải khai bằng `cqw` + `container-type:
   inline-size` trên khung của cột.

2. **`scrollWidth` bị kẹp ở `clientWidth`** khi nội dung còn vừa, nên nó chỉ trả lời "có tràn
   không" — không nói còn dư bao nhiêu. `scrollWidth === clientWidth` đọc ra như "vừa khít" trong
   khi thật ra có thể còn dư nửa cột. Muốn biết slack thật:

   ```js
   var r = document.createRange(); r.selectNodeContents(cell);
   var textW = r.getBoundingClientRect().width;
   var st = getComputedStyle(cell);
   var slack = cell.clientWidth - parseFloat(st.paddingLeft) - parseFloat(st.paddingRight) - textW;
   ```

3. **Đo thiếu một phần tử = bộ số sạch trơn cho một bố cục đang hỏng.** Đo hai trong ba cột, cả sáu
   ca đều `overflow=0`, mà ảnh chụp thì cột thứ ba hiện ra `2…` và tiêu đề `S…`. Đo cái mình vừa
   sửa thì dễ nhớ; cái bị **thu hẹp lại để nhường chỗ** mới là cái hỏng. Sau mỗi lượt tinh chỉnh
   phải **chụp một ảnh và tự đọc lại ảnh đó** — số đo không thay được mắt.

Kèm bẫy webfont (đã ghi ở `chrome.md` nhưng vấp lại): `document.fonts.check()` trả `false` nếu chưa
`await document.fonts.load('<weight> <size>px "<Font>"', chuỗi_mẫu)`. Lượt đo đầu chạy trên font hệ
thống, ra bộ số lệch hẳn mà trông vẫn hợp lý — luôn in cờ "đã nạp font chưa" ra cùng bảng số.

## `hidden` KHÔNG ẩn được phần tử đã có `display` từ một selector lớp

`[hidden] { display: none }` trong stylesheet của trình duyệt chỉ là **selector thuộc tính**; một
quy tắc của mình dạng `.my-row { display: flex }` có độ ưu tiên cao hơn nên **đè luôn nó**. Kết quả:
JavaScript đặt `el.hidden = true`, DOM đọc ra `hidden=true`, `getComputedStyle` trả `flex`, và phần
tử vẫn nằm nguyên trên trang.

Triệu chứng cực dễ đổ oan cho JavaScript: một nhãn ("Link video") hiện ra với ô trống bên dưới, mà
đoạn mã đặt `hidden` thì rõ ràng đã chạy — đọc DOM thấy `hidden=true` nên tưởng CSS đúng và đi tìm
lỗi ở chỗ khác. Chỉ **nhìn ảnh chụp** mới thấy.

Chữa: mọi lớp có đặt `display` mà cũng bị bật/tắt bằng `hidden` phải kèm luôn quy tắc riêng:

```css
.my-row { display: flex; }
.my-row[hidden] { display: none; }   /* BẮT BUỘC, không phải cho chắc */
```

Áp dụng y hệt cho `display: grid`, `inline-flex`, `block`… và cho thuộc tính `[open]`, `[disabled]`
khi dùng chúng làm công tắc hiển thị. Cách kiểm chắc chắn: in `getComputedStyle(el).display` chứ
đừng in `el.hidden` — cái đầu mới là thứ quyết định người dùng có thấy hay không.

## `prefers-reduced-motion: reduce` bắt nhiều hơn tưởng — và đừng tắt hẳn chỉ báo chờ

Windows bật **Settings → Accessibility → Visual effects → Animation effects: Off** là rơi ngay vào
nhánh `@media (prefers-reduced-motion: reduce)`. Đây KHÔNG phải nhánh hiếm chỉ người dùng trợ năng
mới thấy — máy dev bình thường cũng hay tắt cho nhẹ, và khi đó mọi hiệu ứng "đang tải" biến thành
đồ trang trí đứng chết mà không có gì báo lỗi.

Kiểm CHÍNH XÁC bằng đúng API Chrome đọc, đừng suy từ registry:

```powershell
Add-Type @'
using System; using System.Runtime.InteropServices;
public class Spi { [DllImport("user32.dll", SetLastError=true)]
  public static extern bool SystemParametersInfo(uint a, uint b, ref bool p, uint f); }
'@
$v = $false; [void][Spi]::SystemParametersInfo(0x1042, 0, [ref]$v, 0); $v   # False = reduce
```

Đã vấp: đọc bit `ClientAreaAnimation` trong `HKCU:\Control Panel\Desktop\UserPreferencesMask` rồi
đoán vị trí bit → kết luận NGƯỢC ("máy không bật reduce") và đi tìm nhầm nguyên nhân cả một lượt.
`MinAnimate = 0` trong `WindowMetrics` là dấu hiệu gợi ý, không phải bằng chứng.

**Headless Chrome mặc định trả `prefers-reduced-motion: reduce`** (`matchMedia(...).matches = true`),
nên mọi phép đo animation bằng `--dump-dom`/`--screenshot` đều đo nhánh reduce chứ không phải nhánh
thường — thấy `animation-name: none` thì đừng vội kết luận CSS sai.

Cách viết đúng cho nhánh reduce: **bỏ chuyển động, giữ phản hồi**. Chỉ báo chờ chuyển sang nhịp
opacity tại chỗ (`animation-name` đổi sang keyframes chỉ có opacity, giữ nguyên duration/delay so
le), không `translate`, không đổi `width`/layout. Tắt hẳn bằng `animation: none` là người dùng mất
tín hiệu "máy đang chạy" — chặn nhầm thứ mà quy tắc này không định chặn.

Bẫy kèm theo: nếu hiệu ứng thường dựa vào `width: 0 → n` (kiểu ba chấm chạy bằng cắt bề rộng) thì
nhánh reduce PHẢI trả `width: auto`, không thì animation tắt và bề rộng đứng ở khung hình đầu = 0 —
mất luôn nội dung.
