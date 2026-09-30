# Experience — Python

## `ctypes.Structure` lệch layout với struct C = hỏng ÂM THẦM — luôn assert `sizeof`

`ctypes` **không** kiểm tra được struct Python có khớp struct C trong DLL hay không: nó chỉ tin
khai báo `_fields_`. Thừa/thiếu/sai thứ tự một field ⇒ mọi field phía sau lệch offset ⇒ native đọc
giá trị rác. Không exception, không warning, không crash ngay — chương trình chạy tiếp với hành vi
sai lung tung, rất khó lần ra.

Ca thật: DLL được rebuild từ commit đã **xoá 2 field `UINT32`** khỏi struct, phía Python vẫn giữ
⇒ field bool cuối lệch **8 byte**. Kiểu lỗi này chỉ lộ ra khi có người ngồi so từng field với header C.

**Luật:** mỗi `ctypes.Structure` map sang struct C phải kèm assert kích thước ngay tại module,
chạy lúc import — rẻ, và bắt lỗi ngay lần chạy đầu thay vì lúc đang debug hành vi lạ:
```python
assert ctypes.sizeof(NativeConfig) == 20, \
    f"layout mismatch: {ctypes.sizeof(NativeConfig)} != 20"
```
Muốn chắc hơn thì assert cả offset (`NativeConfig.Foo.offset == 8`). Nhớ tính cả
**padding alignment** của C (6 `bool` liên tiếp rồi tới `INT32` ⇒ chèn 2 byte đệm; `ctypes` áp dụng
cùng quy tắc nên chỉ cần khai báo đúng thứ tự và kiểu). Cảnh giác riêng với Windows `BOOL` =
**4 byte** (`c_int32`), khác `bool` C = 1 byte (`c_ubyte`).

**Khi nào phải rà lại:** bất cứ lúc nào DLL native được build lại từ source mới — kể cả khi danh
sách hàm export không đổi. Export giống hệt nhau KHÔNG bảo đảm struct truyền vào còn giống.

## Windows nhiều bản Python / venv: ModuleNotFoundError & exit 127 "command not found"

- Trong Bash tool trên Windows thường KHÔNG có `python`/`python3`/`pip` trên PATH → `python xxx.py` trả **exit 127**. Chỉ có `py` launcher; `py -0p` liệt kê các bản đã cài.
- Bản global (`py -3.x`) hay THIẾU lib nặng (torch/cv2/ultralytics...) → chạy được import nhẹ nhưng fail ở import top-level của module dự án.
- **Interpreter đúng gần như luôn nằm trong venv của project:** `<project>/.../.venv/Scripts/python.exe`, KHÔNG phải `py`/global. Dò bằng cách tìm `pyvenv.cfg`, rồi TEST import chain thật (`import torch,cv2,...`) trước khi launch job dài — tránh chạy nền xong mới phát hiện thiếu lib.
- Lưu ý stdout của Python bị **block-buffered** khi không phải tty (redirect ra file/log): `print()` chưa hiện ngay, còn warning/traceback (stderr) hiện liền → đừng tưởng treo. Cần thấy print sớm thì chạy `python -u` hoặc `PYTHONUNBUFFERED=1`.

## Vòng lặp nhịp CỐ ĐỊNH trên Windows (control loop, ghi hình, gửi lệnh theo tick)

- `time.sleep()` của CPython trên Windows có granularity **~15.6ms** (mặc định hệ thống). Xin ngủ 3ms → ngủ THẬT ~15.6ms ⇒ phá nát mọi cadence < 60Hz. **Đo được: 15.59ms → 3.03ms sau `ctypes.windll.winmm.timeBeginPeriod(1)`**. Bắt buộc gọi 1 lần lúc khởi động (ảnh hưởng CẢ process). Kèm **spin 2ms cuối** bằng `perf_counter` cho chính xác ~0.1ms.
- Dùng **deadline CỘNG DỒN TUYỆT ĐỐI** (`deadline += period`), KHÔNG dùng `sleep(period - elapsed)` — cách sau cộng dồn sai số mỗi vòng ⇒ trôi nhịp. Đo: cách tuyệt đối cho trôi **+0.0ms/60 vòng**.
- **Quá hạn > 1 chu kỳ** (GPU khựng, I/O treo) ⇒ **neo lại `deadline = now`**, ĐỪNG chạy bù các chu kỳ đã lỡ (bù = một loạt tick 0ms liên tiếp = ra lệnh loạn tốc, tệ hơn là trễ).
- Trễ **nhỏ hơn 1 chu kỳ** thì bù nhưng phải **CHẶN mức bù mỗi bước** (vd ≤10ms): nếu không, 1 bước trễ 26ms làm bước kế chỉ còn `period−26` ⇒ vi phạm dung sai. Chặn bù mà vẫn giữ mốc lý tưởng ⇒ đuổi kịp sau vài bước, KHÔNG trôi.
- Hằng số tính bằng **SỐ FRAME/STEP** (timeout, cooldown, debounce, `time_limit`) là **bom hẹn giờ** khi đổi tần số vòng lặp: chúng vô hại ở nhịp cũ rồi bùng ở nhịp mới. Luôn khai báo theo **GIÂY** rồi quy ra step qua cadence. (Ca thật: `time_limit=1000` step = 20 phút ở 0.8Hz nhưng = 83 GIÂY ở 12Hz ⇒ cắt vụn mọi episode, cờ kết-thúc không bao giờ bắn.)

## Cache kết quả xử lý theo IDENTITY object (không phải hash) để khử tính trùng

Khi 2 hàm (vd `is_playable()` gác cổng rồi `build_obs()` dựng dữ liệu) cùng chạy pipeline nặng trên **cùng một** object (numpy array/ảnh), đừng hash để cache (hash mảng lớn = tốn ngang tính lại). Lưu `self._cache_key = frame` rồi so bằng **`frame is self._cache_key`**: O(1), không bao giờ trùng nhầm frame khác. Điều kiện: hàm được cache phải **THUẦN** — tách mọi thứ CÓ STATE (bộ lọc EMA, latch, debounce) ra khỏi nó, để riêng ở hàm gọi sau. Nếu cache có nhiều "mức đầy đủ" (vd bản fail-fast return sớm vs bản chạy hết), phải lưu kèm cờ `complete` — bản dở KHÔNG được phục vụ request cần đầy đủ. (Ca thật: khử 1 lần YOLO 22ms/chu kỳ, 59ms → 32ms.)

## OpenCV template-match: ngưỡng TUYỆT ĐỐI tốt hơn "khoảng cách giữa 2 ứng viên" (gap)

Khi chọn 1 trong 2 vị trí bằng `matchTemplate` (vd chữ A ở trái hay phải), đừng dùng luật *"chênh lệch corr ≥ gap thì mới tin"*. `gap` là proxy TỆ cho độ tin cậy, sai **cả 2 chiều**:
- **Quá chặt:** khi ứng viên nhiễu tình cờ GIỐNG template (vd 'TOR' vs 'TQR' — trùng chữ đầu, O≈Q) corr nhiễu vọt lên 0.88 → gap co lại → trả None dù đáp án đúng rõ mười mươi (0.98).
- **Quá lỏng:** trên ảnh KHÔNG hề chứa đối tượng (menu/ad), cả 2 bên chỉ là nhiễu (0.44 vs 0.30) — gap vẫn đủ lớn → **đoán bừa ra đáp án** → lỗi âm thầm, nguy hiểm hơn None.

**Luật đúng:** một template khớp CHÍNH NÓ luôn cho corr rất cao và ổn định (đo thực tế: 0.93–0.99), còn thứ-không-phải-nó thì trần thấp hơn hẳn (≤0.88) → đặt **sàn tuyệt đối** giữa 2 phân bố rồi mới `argmax`:
```python
if max(c1, c2) < MIN_CORR:   # đối tượng KHÔNG hiện → không đoán
    return None
return "1" if c1 >= c2 else "2"
```
Đơn giản hơn mà đúng hơn (đo được: 36/36 vs 28/36 của luật gap, và argmax-trần-không-sàn sai 13/36). **Luôn đo phân bố corr của lớp-đúng vs lớp-sai trên tập ảnh thật rồi mới chọn ngưỡng** — đừng chỉnh mò.

Hệ quả: **đừng vội đổ lỗi cho ROI/tiền xử lý.** Ở ca trên tôi từng nới ROI + thử tách nền bằng mask chữ trắng (HSV) — mask chỉ nhích gap 0.1029→0.1142, vô ích, vì lỗi nằm ở **LUẬT QUYẾT ĐỊNH** chứ không phải chất lượng ảnh. Test giả thuyết bằng số TRƯỚC khi sửa code.

## `shutil.move` cho THƯ MỤC: hai chế độ hỏng âm thầm — dùng `os.rename` trần

Di chuyển hàng loạt thư mục (gom data, dọn kho) đừng dùng `shutil.move(src, dst)`. Nó có hai hành vi
"tự chữa" mà trong việc gom data đều là SAI:

- **Đích đã tồn tại và là thư mục ⇒ nó NHÉT LỒNG**, thành `dst/<basename(src)>` tức `.../ep_x/ep_x`,
  KHÔNG báo lỗi. Vòng lặp gom 700 thư mục sẽ tạo ra một mớ lồng nhau mà không dòng log nào cảnh báo.
- **`os.rename` lỗi ⇒ nó âm thầm rơi về `copytree` + `rmtree`.** Cùng ổ đĩa thì rename là thao tác
  metadata (0,00 s); copytree là copy THẬT từng byte. Ca đo được 2026-07-28: 56 GB / 693 thư mục, đáng
  lẽ vài giây, thực tế ~3,6 MB/s ⇒ **4+ giờ**, và `WriteTransferCount` của tiến trình cho thấy nó ghi
  đúng bằng lượng đã đọc.

**Fix:** gọi thẳng `os.rename(src, dst)`, lỗi thì GHI NHẬN rồi bỏ qua — không bao giờ copy thay thế:
```python
try:
    os.rename(src, dst)
except OSError as e:                       # đích đã có / handle còn kẹt / khác volume
    problems.append(f"{src.name}: rename lỗi {type(e).__name__} {e}")
    continue
```
**Cách phát hiện đang bị copy nhầm:** đọc bộ đếm I/O của chính tiến trình —
`Get-CimInstance Win32_Process ... | select ReadTransferCount, WriteTransferCount`. Rename thuần thì
hai số này ~0. Thấy chúng tăng đều bằng nhau = đang copytree.

Kèm theo: `WinError 5 Access is denied` khi rename thư mục trên Windows thường chỉ là **handle còn kẹt
tạm thời** (antivirus/indexer/tiến trình vừa bị kill) — chạy lại là qua, đừng vội chuyển sang copy.

## Log gộp nhiều điều kiện = chẩn nhầm về sau

Guard kiểu `if cond_A(x) is None or cond_B(x) is None: print("FAIL A")` sẽ đổ tội cho A ngay cả khi B mới là thủ phạm. Hậu quả thật: log ghi "FAIL score OCR" suốt nhiều phiên → tài liệu/ghi chú chép lại "lỗi OCR" → điều tra sai hướng nhiều lần, trong khi OCR luôn chạy tốt. **Mỗi điều kiện guard một dòng log riêng, nói rõ cái nào hỏng và cái nào vẫn OK.** Khi debug guard, đừng tin nhãn log — đo trực tiếp từng điều kiện con.

## `print` trong worker con của multiprocessing KHÔNG thừa hưởng `reconfigure(encoding=...)`

Mẫu quen thuộc để in tiếng Việt trên Windows console (cp1252) đặt ở CUỐI file:
```python
if __name__ == "__main__":
    for _s in (sys.stdout, sys.stderr):
        _s.reconfigure(encoding="utf-8")
    main()
```
Với `mp.get_context("spawn")` (mặc định trên Windows), tiến trình CON **import lại module nhưng KHÔNG
chạy khối `if __name__ == "__main__"`** ⇒ stdout của nó vẫn là cp1252. Mọi `print` nằm trong hàm worker
mà chứa ký tự non-ASCII (`→`, `⚠️`, chữ có dấu) sẽ ném `UnicodeEncodeError` và **giết worker giữa
chừng**, trong khi cùng chuỗi đó in từ tiến trình cha lại chạy tốt.

Bẫy ở chỗ nó **ẩn theo nhánh code**: nếu dòng print chỉ chạy ở nhánh hiếm (cảnh báo, lỗi lạ) thì test
thường không chạm tới, và khi nó nổ thì nổ đúng lúc đang xử lý ca bất thường.

**Luật:** mọi `print` chạy trong worker con phải **ASCII thuần**. Muốn giữ tiếng Việt thì đừng in từ
worker — trả chuỗi/status về cho tiến trình cha in (cha đã reconfigure). Cách nhớ: hàm nào được đưa
cho `Pool.imap`/`Process(target=...)` thì mọi print bên trong nó coi như đang ở môi trường cp1252.

## Redirect stdout ra FILE trên Windows → cp1252, không phải utf-8 (khác lỗi console)

Script chạy tay ngon lành, nhưng `python foo.py > out.log 2>&1` lại chết ngay dòng `print` đầu tiên có
dấu tiếng Việt:

```
UnicodeEncodeError: 'charmap' codec can't encode character 'ỏ' ... Lib/encodings/cp1252.py
```

Khi stdout là **file** (hoặc pipe), Python 3.11 lấy encoding từ **locale ANSI của hệ** (cp1252 ở máy VN),
KHÔNG phải utf-8 — trong khi in ra console thì terminal thường đã là utf-8 nên không lộ. Kết quả:
lỗi CHỈ xuất hiện khi ta redirect để chạy nền, tức đúng lúc chạy job dài rồi bỏ đi.

**Luật:** chạy script của người khác (không chắc nó có `sys.stdout.reconfigure`) mà có redirect thì
LUÔN đặt biến môi trường, đừng sửa script:
```bash
PYTHONIOENCODING=utf-8 python foo.py > out.log 2>&1     # Git Bash
$env:PYTHONIOENCODING="utf-8"; python foo.py             # PowerShell
```
Liên quan nhưng KHÁC nhau: mục "print trong worker con của multiprocessing" ở trên là do tiến trình con
không chạy khối `if __name__ == "__main__"`. `PYTHONIOENCODING` vá được CẢ HAI vì nó đi theo môi trường,
di truyền sang tiến trình con.
