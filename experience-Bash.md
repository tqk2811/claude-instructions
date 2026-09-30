# Bash / shell scripting experience

- **`pgrep -x <tên>` KHÔNG BAO GIỜ khớp nếu tên chương trình dài quá 15 ký tự** — Linux lưu `comm` (tên tiến trình trong kernel) tối đa 15 byte, nên `vmware-toolbox-cmd` (18 ký tự) nằm trong `/proc/<pid>/comm` là `vmware-toolbox-`. `pgrep -x` so khớp CHÍNH XÁC với `comm` ⇒ trả về rỗng, exit code 1, dù tiến trình đang chạy rành rành.
  - **Triệu chứng đánh lừa:** vòng lặp `while pgrep -x foo > /dev/null; do ...; done` dùng để chờ một tác vụ dài **thoát ngay lập tức ở vòng đầu**, in ra "đã xong" trong khi việc mới bắt đầu. Không có lỗi nào, chỉ là kết luận sai — y hệt trường hợp tác vụ thật sự đã kết thúc.
  - **Fix:** khớp theo command line đầy đủ: `pgrep -f "[v]mware-toolbox-cmd disk shrink"`. Dùng mẹo ngoặc vuông `[v]` để chính lệnh `pgrep` (và lệnh ssh bao ngoài, vì chuỗi lệnh remote cũng nằm trong command line) không tự khớp chính nó.
  - **Kiểm nhanh trước khi tin `pgrep -x`:** `cat /proc/<pid>/comm` xem tên có bị cắt không, hoặc đơn giản là dùng `-f` ngay từ đầu cho mọi tên dài hơn 15 ký tự.

- **Nested heredocs through SSH mangle literal backslash-escape sequences (e.g. `\n` inside a string meant to stay literal, like a `printf` format string) — do NOT try to author multi-layer escaped content (local bash heredoc → ssh → remote bash → python/perl heredoc) by typing extra backslashes and hoping they survive each layer.** Each layer (local shell parsing the command text, SSH forwarding stdin, remote shell heredoc, then a script interpreter parsing its own string literals) can reinterpret backslash sequences differently, and `\\n` vs `\n` vs an actual newline byte get silently swapped — even a quoted heredoc delimiter (`<<'EOF'`) only protects against the immediate shell layer, not the ones further down the chain.
  - **Fix that reliably works:** write the exact target content to a **local file** with the Write tool (byte-for-byte, no shell/string re-parsing involved), `scp` it to the remote host, then splice it into the target file with a small script (e.g. Python) that reads the block as **raw file bytes/lines** (`readlines()`), not as a string literal embedded in the script source. This sidesteps every layer of escape reinterpretation because the content never passes through a shell or language string-literal parser after the initial Write.
  - Symptom that flags this is happening: after an edit, literal `\n` (meant to stay as two characters, e.g. inside a `printf '...\n...'` format string) shows up as a real line break when you `cat`/`sed` the file back, or a sed one-liner with escaped quotes silently produces broken/extra lines.

- **Overriding the `/usr/bin/nproc` binary does NOT limit ninja/cmake auto-detected parallelism.** A common RAM-safety trick on high-core build hosts is to replace `/usr/bin/nproc` with a wrapper that caps its output, so `make -j$(nproc)` uses fewer jobs. But **ninja reads the CPU count directly via libc `sysconf(_SC_NPROCESSORS_ONLN)`, not by shelling out to the `nproc` command** — so a ninja build (or `cmake --build` with the Ninja generator, when no explicit `-j` is passed) spawns `cores+2` jobs regardless of the nproc wrapper. Symptom: after capping nproc, `ps -e | grep -c cc1plus` still shows ~`cores+2` concurrent compilers during a ninja/cmake build, and the host OOMs.
  - **Fix:** control the build's OWN parallelism knob, not nproc. For CMake-driven ninja, pass `-j N` (e.g. `cmake --build . -jN`) or set `CMAKE_BUILD_PARALLEL_LEVEL=N`; for a project's build script, find the var it feeds to `-j` (many use `${CORES:+-j${CORES}}` or similar — set that env var). As a last-resort host-level cap that DOES affect `sysconf`, limit the container/cgroup's visible CPUs with `--cpuset-cpus` (uncertain propagation) or give the container a memory limit so OOM stays contained instead of killing host `systemd`/`dbus`/`tmux`.
  - Corollary for Docker layer caches: editing an early/base Dockerfile layer invalidates every downstream layer FROM it, forcing expensive rebuilds (e.g. a full LLVM/clang toolchain) that were previously cached — a "small" edit to a base image can trigger the heaviest build in the whole pipeline. Weigh that before editing a base layer just to tweak one setting.

- **Vòng lặp bash chạy nền KHÔNG chết theo lệnh dừng tác vụ của harness — nó thành mồ côi và tiếp tục chạy.** Mỗi lần "dừng rồi khởi động lại" một script giám sát là thêm MỘT bản sao sống song song. Sau 5 lần vá-rồi-chạy-lại sẽ có 5 con cùng thao tác lên cùng tài nguyên (2026-07-20: 14 tiến trình supervisor cùng force-stop một app, cùng dựng server, cùng chiếm một cổng).
  - **Vì sao cực khó chẩn đoán:** triệu chứng trông y hệt lỗi hạ tầng — tiến trình bị giết liên tục (`signal 9`), server "hồi sinh" ngay sau khi kill, cổng luôn bị chiếm, client luôn timeout. Mọi giả thuyết rút ra trên nền nhiễu đó đều sai, và mỗi lần vá script lại đẻ thêm một con mồ côi ⇒ vòng xoáy.
  - **LUẬT: trước khi chẩn đoán bất cứ điều gì, ĐẾM số bản sao của chính mình đang chạy.** `Get-CimInstance Win32_Process -Filter "Name like '%bash%'" | Where-Object { $_.CommandLine -like '*<tên-script>*' }` — lọc theo CommandLine, vì tên tiến trình chỉ là `bash.exe`.
  - **Dọn:** giết theo PID bằng PowerShell tool (`Stop-Process -Id`). Gọi `powershell -Command` xuyên qua Bash tool rất hay hỏng vì lồng nhiều lớp nháy — dùng thẳng PowerShell tool.
  - **Phòng:** script dài hạn nên tự kiểm "đã có bản khác chạy chưa" (lock file / đếm process) rồi thoát sớm, thay vì tin rằng bản trước đã chết.

- **Trên Windows/Git Bash, `pid=$(ham_launch_nen)` TREO VĨNH VIỄN nếu tiến trình nền là chương trình Windows NATIVE (python.exe, node.exe, java.exe...).** Command substitution `$(...)` đọc pipe tới khi EOF; EOF chỉ tới khi MỌI tiến trình giữ đầu ghi của pipe đóng nó. Tiến trình native kế thừa handle pipe của msys và không đóng ⇒ `$(...)` chờ mãi, dù công việc bên trong đã chạy ngon lành.
  - **Triệu chứng đánh lừa hoàn toàn:** tiến trình nền CÓ chạy thật (thấy trong Task Manager, log của nó vẫn ghi), nhưng script cha đứng im — không log thêm dòng nào, không sang vòng lặp kế tiếp. Nhìn vào thì tưởng script "chạy chậm" hoặc kẹt ở tiến trình nền, thực ra nó kẹt ở chính lệnh gán biến. Với script quản nhiều máy: máy đầu tiên chạy bình thường, các máy sau KHÔNG BAO GIỜ được khởi động (2026-07-20, AutoDLS supervisor 2 máy).
  - **Fix:** đừng trả PID qua stdout. Cho hàm gán vào **biến toàn cục** rồi đọc lại — hàm chạy trong shell hiện tại nên `$!` vẫn đúng:
    ```bash
    LAST_PID=""
    launch_one() { ( cmd >> log 2>&1 ) & LAST_PID=$!; }
    launch_one; pids[$i]=$LAST_PID      # KHÔNG dùng pids[$i]=$(launch_one)
    ```
    Cách khác: ghi PID ra file trong hàm (`echo $! > /tmp/x.pid`) rồi `cat` lại.
  - **Luật chung:** trong Git Bash, KHÔNG bọc `$(...)` quanh bất cứ thứ gì spawn tiến trình nền native. Redirect stdout của tiến trình nền vào file KHÔNG cứu được, vì handle đã kế thừa từ trước khi redirect có hiệu lực.

- **Chờ một sự kiện bằng cách `grep` log GHI NỐI = khớp ngay lập tức vào LỊCH SỬ, không phải vào lần chạy hiện tại.** `until grep -q "train 15000 step" run.log; do sleep 30; done` trông rất hợp lý, nhưng nếu `run.log` là file append qua nhiều lần chạy thì chuỗi đó đã nằm sẵn từ hôm trước ⇒ vòng lặp thoát ở vòng đầu tiên và hành động kích hoạt SỚM (2026-07-31: cắm cờ dừng lúc đang record thay vì lúc train, suýt cắt cụt chu kỳ chỉ còn 2/12 trận).
  - **Triệu chứng đánh lừa:** không có lỗi nào cả — lệnh chạy "thành công", chỉ là thành công quá sớm. Thêm điều kiện `&&` với một mốc của lần chạy hiện tại (vd `grep -c 'CHU KỲ 68'`) VẪN sai, vì cả hai vế đều đúng ngay lập tức khi mốc kia vừa được ghi.
  - **Fix — chờ thứ chỉ tồn tại ở lần chạy này:** ưu tiên **một file mới sinh ra theo lần chạy** (`until [ -f train_c68.log ]`), hoặc chốt số dòng trước rồi chỉ đọc phần MỚI: `n0=$(wc -l < run.log); until tail -n +$((n0+1)) run.log | grep -q "..."; do sleep 30; done`.
  - **Luật chung:** trước khi poll một log, hỏi "chuỗi này đã có trong file TỪ TRƯỚC chưa?" — `grep -c` toàn file để kiểm, đừng tin `tail`.

- **Đừng mở kết nối "thăm dò" tới server chỉ phục vụ MỘT client.** Một probe tự viết gửi sai khung giao thức sẽ làm server kẹt VĨNH VIỄN: nó vẫn `listening`, vẫn nhận `client connected`, nhưng không trả lời ai nữa — kể cả client thật. Thành ra công cụ chẩn đoán tự tạo ra đúng sự cố mà nó định phát hiện, rồi mọi phép thử sau đó đều cho kết quả sai.
  - **Thay bằng:** suy ra sức khoẻ từ TRIỆU CHỨNG của client thật (vd "client chết trong <90s ⇒ dựng lại server"), hoặc dùng chính thư viện client của project, KHÔNG tự dựng lại giao thức.

- **Trên Windows/Git Bash (msys), mọi ARGUMENT bắt đầu bằng `/` bị tự động dịch thành đường dẫn Windows trước khi chương trình nhận được.** Truyền `--event-dev /dev/input/event5` cho một script Python qua Bash tool thì Python nhận `C:/Program Files/Git/dev/input/event5`. Không có cảnh báo nào; chương trình chạy bình thường trên đường dẫn vô nghĩa.
  - **Đặc biệt nguy hiểm với đường dẫn của máy KHÁC** (Android qua adb, container, máy remote qua ssh): `/dev/...`, `/data/...`, `/sdcard/...`, `/proc/...` đều dính. Chúng hợp lệ ở đầu bên kia nhưng msys tưởng là đường dẫn local.
  - **Vì sao khó phát hiện:** với `adb exec-out cat <path-sai>`, adb in thông báo lỗi ra **stdout** (exec-out gộp stderr của thiết bị vào stdout). Nếu code đang đọc luồng NHỊ PHÂN, đoạn text đó chen vào giữa; độ dài text hiếm khi chia hết cho kích thước record ⇒ **toàn bộ luồng lệch vĩnh viễn**, mọi record sau đó decode ra rác. Bộ đếm "số event nhận được" vẫn tăng đều, log vẫn đẹp, chỉ có KẾT QUẢ là rỗng/vô nghĩa. Mất 2 lần thu dữ liệu vì đúng chỗ này (2026-07-20, AutoDLS/AutoGenius).
  - **Fix:** đặt `MSYS_NO_PATHCONV=1` cho lệnh đó (`MSYS_NO_PATHCONV=1 python x.py --dev /dev/foo`), hoặc nhân đôi gạch chéo `//dev/input/event5`, hoặc đơn giản nhất là **đừng truyền cờ** nếu giá trị mặc định trong code đã đúng — hằng số nằm trong Python/C# không bị msys đụng tới.
  - **Không chỉ argument BẮT ĐẦU bằng `/` — phần sau dấu `=` bên trong một argument đã bọc nháy cũng bị dịch.** `curl --data-urlencode "ReturnUrl=/Guest/Profile"` gửi đi `ReturnUrl=C:/Program Files/Git/Guest/Profile`. Nháy kép KHÔNG cứu được, vì msys dịch sau khi shell đã bỏ nháy. Dính với mọi form field, query param, JSON inline... có giá trị là đường dẫn URL tuyệt đối (2026-07-28: test `returnUrl` cứ trả về trang mặc định, tưởng model binding hỏng, thật ra server nhận đúng cái nó được gửi và từ chối đúng vì đó không phải URL nội bộ).
  - **Mặt NGƯỢC LẠI của cùng cái bẫy, và nó im lặng hơn hẳn: `=@/đường/dẫn` thì msys KHÔNG dịch.** Dấu `@` đứng chen vào nên chuỗi không còn "bắt đầu bằng `/`" theo luật của msys. Mà `@` chính là cú pháp gửi FILE của curl: `curl -F "Logo=@/tmp/x.png"` đưa nguyên chuỗi `/tmp/x.png` cho `curl.exe` bản Windows, thứ không hiểu đường dẫn kiểu Unix ⇒ `curl: (26) Failed to open/read local data from file/application`. Dính cả `--data-binary @file` và `-T file` (2026-08-13: thử upload logo nhà tài trợ).
    - **Vì sao mất thời gian:** với `-s` thì curl **không in gì cả**, exit code khác 0 nhưng script dùng `$(...)` thì không ai nhìn — nhìn ra y hệt "server trả về rỗng". Tôi đã đi đọc log server và tin là handler ném exception, trong khi request chưa bao giờ rời khỏi máy.
    - **Fix:** đổi sang đường Windows bằng `cygpath -w` (`-F "Logo=@$(cygpath -w /tmp/x.png)"`), và **bỏ `-s` ở lượt chạy đầu tiên** của mọi lệnh curl có upload file.
  - **Cách phát hiện:** cho chương trình **in ra đường dẫn nó thực sự dùng** ngay lúc khởi động. Một dòng `print(f"device={a.device} event={a.event_dev}")` là đủ để lộ ngay; nếu không có nó thì phải đi decode byte thô mới tìm ra. Với web: xem giá trị server ECHO lại (vd render lại field trong form) trước khi kết luận code sai.
  - Bài học chung: khi đọc luồng nhị phân có cấu trúc, **luôn validate từng record** (vd type/magic phải thuộc tập hợp cho phép) và đếm số lần lệch, thay vì tin rằng luồng luôn thẳng hàng. Im lặng khi lệch = hỏng dữ liệu mà không ai biết.

- **Trên Windows/Git Bash, ĐỐI SỐ chứa ký tự ngoài ASCII bị chuyển sang codepage ANSI trước khi tới chương trình Windows NATIVE (`curl.exe` của System32, `sqlcmd`, mọi .exe không phải msys).** `curl -d '{"Name":"Phòng ban"}'` gửi đi byte `F2` (CP1258/1252 của `ò`) thay vì `C3 B2` của UTF-8. Shell KHÔNG sai — `printf 'ò' | od -tx1` vẫn ra `c3 b2`; chỗ hỏng nằm ở lớp msys dựng dòng lệnh Win32.
  - **Cực khó lần vì hỏng KHÔNG ĐỀU:** ký tự nào CÓ trong codepage ANSI thì bị đổi, ký tự nào không có thì giữ nguyên UTF-8. Body ra nửa UTF-8 nửa ANSI. Server ASP.NET ném `DecoderFallbackException: Unable to translate bytes [F2] at index 111` và trả 500 kèm thông báo trung tính, nên nhìn từ ngoài tưởng lỗi nghiệp vụ. Cùng một script, chuỗi `Nguyễn` (có `ễ` không nằm trong codepage) đi lọt mà `Phòng` lại hỏng (2026-07-29, Eventify).
  - **Fix:** đưa nội dung qua **file**, đừng qua argv — `printf '%s' "$json" > body.json && curl --data-binary @body.json`. File do shell ghi nên giữ nguyên byte UTF-8. Cùng cách chữa với bẫy nested-heredoc ở đầu file này.
  - **Cách phát hiện:** `od -tx1` cái mình ĐỊNH gửi rồi so với byte server thật sự nhận (log exception thường in ra chỉ số byte hỏng). Đừng chỉ nhìn output của `echo`.
  - **`--data-urlencode` cũng dính, và đây là biến thể NGUY HIỂM NHẤT vì nó KHÔNG lỗi — nó ghi rác vào database.** POST form có tiếng Việt bằng `curl --data-urlencode "input.Name=bố trí tuyến"` thì server nhận `bố tr%ED tuy?n`: byte ANSI `ED` không phải UTF-8 hợp lệ nên `%`-encoding của curl để lại chuỗi lai, ASP.NET bind được (không ném gì), lưu xuống DB, trả 302 thành công. Nhìn từ ngoài là "test chạy ngon". Chỉ phát hiện khi đọc lại dữ liệu bằng công cụ UTF-8 thật (2026-08-15: smoke test cache làm hỏng một dòng seed).
    - **Cách chữa `--data-binary @file` ở trên KHÔNG áp dụng được** cho form nhiều field cần url-encode từng cái. Dùng script Python `urllib` (`urlencode(fields, encoding='utf-8')`) hoặc PowerShell thay vì curl trong bash.
    - **Luật rút ra:** smoke test mà form có ký tự ngoài ASCII thì ĐỪNG dùng curl trong Git Bash. Và sau mọi smoke test có ghi dữ liệu, đọc lại bản ghi vừa ghi bằng công cụ UTF-8 để xác nhận trước khi coi là xong — terminal in `?` thay dấu nên nhìn bằng mắt không phân biệt được "codepage hiển thị" với "dữ liệu đã hỏng".

- **Trên Windows/Git Bash (msys), `sed -i` biến CRLF thành LF, và `grep -q $'\r'` / `sed -n 'Np' | cat -A` KHÔNG phát hiện được `\r`** — các tool msys mở file ở text mode nên nuốt CR trước khi bạn kiểm tra. Hậu quả thực tế: một loạt `sed -i 's/A/B/'` trên file C#/csproj (vốn CRLF) làm toàn bộ file bị đổi line ending, `git diff` phình từ "1 dòng" lên "90 dòng đổi", che mất thay đổi thật.
  - **Càng tệ hơn khi cố tự sửa:** `sed -i 's/$/\r/'` để khôi phục CRLF chạy trên file *đã* CRLF sẽ sinh `\r\r\n`; mà chính `grep` cũng không thấy `\r\r` nên vòng lặp "kiểm tra rồi sửa" bằng msys tool cho kết quả sai ở cả hai chiều — dễ tưởng đã sửa xong trong khi chưa.
  - **Fix:** đừng dùng `sed -i` cho việc sửa hàng loạt trên repo Windows. Dùng tool Edit, hoặc PowerShell đọc/ghi **byte** (`[IO.File]::ReadAllBytes` / `WriteAllBytes`, so sánh cặp byte `13,10`) — đó là cách duy nhất phát hiện và chuyển đổi line ending đáng tin trên nền này.
  - **Cách phát hiện sớm:** sau khi sửa hàng loạt, luôn chạy `git diff --numstat`. Nếu số dòng đổi ≈ tổng số dòng file thì gần như chắc chắn là line ending chứ không phải nội dung; đối chiếu thêm bằng `git diff -w --stat` để thấy thay đổi thật.
  - Lưu ý phụ: **line ending không đồng nhất trong cùng repo** (project này: cây `Khoai89` dùng CRLF, `Khoai89_1`/`Khoai89_Demo` dùng LF), nên tuyệt đối không convert hàng loạt theo một chuẩn — phải khôi phục theo từng file, đối chiếu blob trong `git show HEAD:<path>`.

- **`cd X && nohup bash script.sh & ` trong một tool-call: PID nhận được ($! hoặc `ps` cha=1 đầu tiên) có thể là SUBSHELL WRAPPER, không phải bash đang chạy script — kill nó thì script VẪN SỐNG.** Compound command đưa cả `cd && nohup bash` vào một subshell nền; bash-chạy-script là CON của wrapper đó. Kill wrapper → script bash thành mồ côi (PPID=1) và tiếp tục chạy như không có gì (2026-07-23, AutoDLS: tưởng đã chặn supervisor trước bước train, 12 phút sau nó vẫn build+cp làm bẩn pool, phải dọn ngược).
  - **LUẬT: sau khi kill, phải XÁC NHẬN theo CÔNG VIỆC chứ không theo PID** — grep process theo CommandLine chứa tên script (`Get-CimInstance Win32_Process ... CommandLine -like '*script.sh*'`), hoặc xem log của script có thêm dòng mới không. `ps -p <pid>` trả "không còn" CHƯA chứng minh script chết.
  - **Nhận diện đúng bash-script trong cây MSYS:** `ps` full rồi lần theo PPID; wrapper là con trực tiếp của PID 1 với STIME đúng lúc launch, script bash là CON của wrapper (cùng STIME). Kill CẢ CÂY: kill wrapper + con của nó, hoặc kill theo CommandLine.

- **Kill theo CommandLine KHÔNG bắt được tiến trình con của `multiprocessing` (Python) — chúng không mang tên script trong dòng lệnh.** Worker spawn ra có CommandLine kiểu `python.exe -c "from multiprocessing.spawn import spawn_main..."`, nên bộ lọc `CommandLine -match 'ten_script|ten_module'` bỏ sót sạch. Kill cha xong, lệnh kiểm ngay sau đó báo "CLEAN" nhưng worker **vẫn chạy tiếp và vẫn GHI FILE** cho tới khi làm xong việc đang dở (2026-07-27, AutoDLS: kill lúc 01:32, file mới vẫn xuất hiện lúc 01:38:48 — may là ghi đúng, không hỏng).
  - **Vì sao nguy hiểm:** nếu ngay sau khi kill mà bạn `rm -rf` thư mục ra hoặc chạy lại tác vụ, worker sống sót sẽ ghi đè/chen vào giữa ⇒ file cụt hoặc trộn hai lần chạy.
  - **Kiểm đúng:** lọc process theo **tên chương trình + thư mục venv** (vd mọi `python.exe` có đường dẫn venv của project) chứ không chỉ theo tên script; và **xác nhận bằng mtime của thư mục ra** — đợi tới khi không còn file mới trong ~1 phút mới coi là đã dừng.

- **SỬA một file `.sh` ĐANG CHẠY là làm hỏng nó giữa chừng: bash đọc script theo OFFSET BYTE, không nạp trọn file.** Sau khi chạy xong khối lệnh hiện tại, nó `seek` về đúng vị trí byte cũ rồi đọc tiếp. Chèn/xoá dòng ở PHÍA TRÊN vị trí đó ⇒ lần đọc kế tiếp rơi vào GIỮA một dòng ⇒ chạy nhầm lệnh hoặc syntax error, mà file trên đĩa thì hoàn toàn hợp lệ nên `bash -n` không phát hiện được gì.
  - Rình rập nhất với script **giám sát/poll dài hơi** (`for _ in $(seq 1 480); do ... sleep 30; done`): nó đứng im hàng giờ ở giữa file, vừa đúng lúc bạn muốn vá.
  - **Fix:** `kill` tiến trình rồi `nohup bash script.sh &` lại bản mới. Script poll thường idempotent (đọc lại trạng thái từ log/đĩa) nên khởi động lại không mất gì — cứ khởi động lại, đừng sửa nóng.
  - **Khác hẳn Python/Node:** chúng nạp trọn file lúc khởi động, sửa file giữa chừng KHÔNG ảnh hưởng tiến trình đang chạy. Đừng suy luận từ Python sang bash.

- **File danh sách do Python trên Windows ghi ra là CRLF; `while read -r d` giữ lại `\r` ⇒ mọi phép kiểm đường dẫn đều SAI mà không báo lỗi.** `[ -d "$d" ]` trả false cho *mọi* dòng trừ dòng cuối (dòng cuối không có newline nên không dính `\r`) — triệu chứng rất đặc trưng: **18/19 mục bị "không tìm thấy", đúng 1 mục cuối chạy được**.
  - **Fix:** cắt CR ngay khi đọc — `d="${d%$'\r'}"` (và `[ -z "$d" ] && continue` cho dòng trống). Hoặc ghi danh sách bằng `open(..., newline='\n')` phía Python.
  - Cùng họ với bẫy `sed -i`/CRLF ở trên: msys nuốt CR khi HIỂN THỊ nên `echo`/`cat` nhìn vẫn bình thường, chỉ so sánh chuỗi mới lộ.

- **`ps -W` trong Git Bash CHỈ in TÊN process, KHÔNG in dòng lệnh ⇒ `ps -W | grep <tên_script>` KHÔNG BAO GIỜ khớp.** Mọi tiến trình Windows hiện lên đúng một chữ `python.exe` / `node.exe`, nên cổng bảo vệ kiểu `if ps -W | grep -q 'record_r2d'; then exit 1; fi` **luôn cho qua** — nó không "kiểm rồi thấy sạch", nó *mù*. Nguy ở chỗ vế đúng (không có tiến trình) và vế sai (có mà không thấy) cho ra CÙNG một kết quả, nên test thủ công lúc máy rảnh vẫn PASS (2026-07-29, AutoDLS: cổng "cấm rebuild pool khi worker record đang chạy" trong `chain_v17.sh` đã tắt âm thầm suốt nhiều lần chạy).
  - **Fix:** hỏi WMI và lọc theo CommandLine:
    `n=$(powershell -NoProfile -Command "@(Get-CimInstance Win32_Process -Filter \"Name='python.exe'\" | Where-Object { \$_.CommandLine -like '*script.py*' }).Count" | tr -dc '0-9')`
  - **Cách tự bắt lỗi này:** với mọi lệnh kiểm dạng "phải KHÔNG có gì", chạy thử lúc thứ đó ĐANG CÓ và xác nhận nó thật sự khớp. Điều kiện bảo vệ chưa từng thấy trạng thái dương là điều kiện chưa được kiểm chứng.
  - Cùng họ với bẫy `ps aux | grep` không thấy tiến trình Windows: trong MSYS, mọi câu hỏi về tiến trình Windows đều phải đi qua `Get-CimInstance Win32_Process`.

- **`grep -c` LUÔN in số VÀ trả exit 1 khi đếm được 0 ⇒ `n=$(grep -c pat f || echo 0)` cho ra HAI số `"0\n0"`, làm vỡ mọi phép so sánh số học sau đó.** Thói quen `|| echo 0` là đúng cho `grep -o`/`grep` thường (không in gì khi trượt) nhưng SAI cho `-c`. Triệu chứng: `[: 0\n0: integer expression expected` rồi `syntax error in expression (error token is "0")` — và nếu đang chạy trong vòng lặp giám sát thì script CHẾT NGAY, tức là cái monitor dựng lên để canh sự cố lại tự tắt đúng lúc cần nhất (2026-07-29, AutoDLS).
  - **Fix:** bỏ hẳn nhánh `||`, chỉ cần chống rỗng: `n=$(grep -c "$pat" "$f" 2>/dev/null); n=${n:-0}`. Exit code khác 0 không sao vì không dùng `set -e` cho dòng này.
  - **Luật chung:** trước khi bọc `|| <mặc-định>` quanh một lệnh, kiểm xem lệnh đó có in gì ra stdout khi "thất bại" không. Với các lệnh mà exit≠0 nghĩa là "không tìm thấy" chứ không phải "lỗi" (`grep -c`, `grep -q`, `diff`, `cmp`, `test`), `||` sẽ CỘNG THÊM chứ không THAY THẾ.

- **`perl -i -pe 's/\x01/\n/g'` chèn KÝ TỰ XUỐNG DÒNG THẬT chứ không chèn hai ký tự `\` + `n`.** Vế thay thế của `s///` được nội suy như chuỗi nháy kép, nên qua một lớp bóc nữa: muốn ra literal `\n` trong file phải viết `'s/\x01/\\n/g'` (bốn dấu gạch chéo). Triệu chứng cực xấu khi file là mã nguồn: một chuỗi `'...'` của JS/C# bị cắt đôi bằng newline thật ⇒ **file vỡ cú pháp**, mà `git diff` thì hiện ra một dòng trông rất bình thường. Tôi đã tự tay làm hỏng `board.js` đúng kiểu này (2026-08-07).
  - **Fix an toàn hơn hẳn `perl -pe` cho việc thay chuỗi trong mã nguồn:** viết một script node/python 3 dòng đọc file, `split(chuỗi_cũ).join(chuỗi_mới)`, ghi lại. Không có lớp nội suy nào, không phải đếm dấu gạch chéo.
  - **Luật chung:** mọi lần sửa file bằng regex trên dòng lệnh phải kiểm lại NGAY bằng công cụ hiểu cú pháp của chính ngôn ngữ đó (`node --check`, `dotnet build`, `python -m py_compile`), đừng dừng ở việc "diff nhìn đúng".

- **Quét ký tự điều khiển sau khi sửa bất kỳ file text nào.** Nội dung do mình sinh ra hoàn toàn có thể mang `\x01`, `\x02`, `\x00`... thật thay vì escape hai ký tự, và chúng **vô hình trong mọi công cụ đọc thường**. Hậu quả: git coi file là **binary** nên diff không đọc được nữa, còn phép so sánh chuỗi thì sai một cách không giải thích nổi. Lệnh quét:
  `perl -ne 'while (/([\x00-\x08\x0b\x0c\x0e-\x1f])/g) { printf("%s dòng %d: 0x%02x\n", $ARGV, $., ord($1)); }' <file>`
  Chạy nó cho toàn bộ file đã sửa (`git status --short | awk '{print $NF}'`) trước khi coi là xong.

- **Backtick trong `node -e "..."` bị bash NUỐT SẠCH trước khi node nhìn thấy — và script vẫn báo "OK".** Trong nháy kép, bash coi `` ` `` là command substitution: nó chạy phần bên trong như lệnh, in "command not found" ra stderr, rồi thay bằng **chuỗi rỗng**. Hậu quả khi đang sửa tài liệu Markdown: mọi ````code```` trong nội dung mới biến mất, file ghi ra thiếu hàng chục đoạn mã mà node vẫn in dòng "OK, đã thay N khối" — tức là **báo thành công cho một kết quả hỏng** (2026-08-08: viết lại `docs/Glossary-vi.md`, bốn mục mới mất sạch tên bảng và tên cột).
  - Nhìn ra được nhờ stderr: một loạt dòng `PrizeDraws: command not found` xen giữa output. **Đừng bỏ qua stderr chỉ vì exit code là 0.**
  - **Fix:** đừng nhét nội dung nhiều dòng vào `node -e`. Ghi script ra file bằng công cụ ghi file rồi `node file.js`. Trong file thì backtick, `$`, dấu nháy đều là ký tự thường.
  - Cùng họ: heredoc `<<'EOF'` (nháy đơn quanh EOF) cũng chặn được, nhưng file script vẫn hơn vì sửa lại được và đọc được diff.

- **File `.md`/`.cs` trong repo Windows thường là CRLF; template literal trong script Node là LF ⇒ `s.includes(chuỗi_nhiều_dòng)` trượt 100% mà KHÔNG có lỗi nào.** Script chỉ lặng lẽ báo "0 khối được thay". Rất dễ đổ oan cho việc "chuỗi gốc chắc đã bị sửa rồi" và đi tìm nhầm chỗ (2026-08-08, Eventify).
  - **Fix:** chuẩn hoá cả hai vế trước khi so — `const crlf = x => x.replace(/\r\n/g,'\n').replace(/\n/g,'\r\n');` rồi `s.includes(crlf(from))`.
  - **Và ghi lại đúng kiểu xuống dòng cũ.** `split(/\r?\n/).join('\n')` là cách âm thầm đổi cả file sang LF ⇒ git hiện diff **toàn bộ file** thay vì vài dòng thật sự sửa. Kiểm nhanh: `(s.match(/\r\n/g)||[]).length` so với `(s.match(/(?<!\r)\n/g)||[]).length`.
  - **Luật chung:** mọi script sửa file hàng loạt phải in ra số khối đã thay VÀ số khối trượt, rồi tự mình đọc lại một đoạn kết quả. "Không có lỗi" không đồng nghĩa "đã sửa".

- **`taskkill //F //IM chrome.exe` giết CẢ trình duyệt cá nhân của người dùng, không riêng bản headless mình mở — mất sạch tab họ đang làm việc dở.** Tôi đã làm đúng việc này (2026-08-09: dọn dẹp sau khi đo `/Board` bằng CDP). `/IM <tên ảnh>` nhắm theo **tên tiến trình**, mà Chrome người dùng và Chrome headless là cùng một `chrome.exe`. Không có gì cảnh báo, và không có đường hoàn tác — người dùng phải tự bấm "Khôi phục" lúc mở lại.
  - 🔴 **ĐÃ TÁI PHẠM CÙNG NGÀY, ngay sau khi viết chính mục này** (2026-08-09, đo font `/Board`). Lần hai không phải lúc dọn dẹp mà **giữa chừng công việc**, nên đọc lại mục này lúc "dọn cuối phiên" là quá muộn. Nhớ nó như một luật lúc GÕ LỆNH, không phải một mục để tra lúc kết thúc.
  - **Tình huống dụ mình gõ lệnh đó lần hai: một lệnh `chrome --headless ... --screenshot` bị treo tới hết timeout.** Phản xạ "chắc còn tiến trình cũ giữ khoá, giết hết cho sạch" là SAI ở cả hai vế:
    - Nguyên nhân thật gần như luôn là **`--user-data-dir` đang bị một tiến trình Chrome khác khoá** (kể cả Chrome cá nhân, nếu lỡ trỏ vào profile thật).
    - **Cách gỡ đúng là đổi sang một `--user-data-dir` MỚI** (`...\cprof2`, `cprof3`, …) rồi chạy lại — không đụng tới tiến trình nào. Lần này làm vậy là chạy được ngay, tức là cú `taskkill` vừa thừa vừa phá.
    - Kèm theo, headless treo còn hay do **thiếu đường ghi file**: `--screenshot=tên-tương-đối.png` trên Windows ghi vào thư mục không có quyền ⇒ `Access is denied`. Luôn truyền **đường tuyệt đối kiểu Windows** cho `--screenshot`.
  - **Fix, theo thứ tự ưu tiên:**
    1. Đóng qua chính giao thức đang dùng: với Chrome mở kèm `--remote-debugging-port`, gọi `curl http://127.0.0.1:9222/json/close/<targetId>` cho từng tab rồi `Browser.close` qua CDP. Sạch nhất vì nó chỉ đụng đúng phiên mình tạo.
    2. Lưu PID lúc khởi động rồi giết đúng PID đó: `chrome.exe ... & echo $! > pid.txt` không dùng được cho tiến trình Windows tách rời, nên lấy qua `Get-CimInstance Win32_Process` **lọc theo `CommandLine`** — bản headless của mình luôn có `--user-data-dir=<thư mục scratchpad riêng>`, đó là dấu nhận dạng không trùng với ai:
       `powershell -NoProfile -Command "Get-CimInstance Win32_Process -Filter \"Name='chrome.exe'\" | Where-Object { \$_.CommandLine -like '*<scratchpad>*' } | ForEach-Object { Stop-Process -Id \$_.ProcessId -Force }"`
    3. **Luôn** mở trình duyệt tự động hoá với `--user-data-dir` riêng trong scratchpad — vừa tách hồ sơ, vừa tạo sẵn cái mốc để lọc lúc dọn.
  - **Luật chung: đừng bao giờ giết tiến trình theo TÊN ẢNH trên máy của người dùng.** Nhắm theo PID mình đã lưu, hoặc theo dòng lệnh chứa dấu nhận dạng của riêng mình. Luật này áp cho mọi tên ảnh dùng chung: `chrome.exe`, `node.exe`, `dotnet.exe`, `python.exe`. Riêng `taskkill //FI "IMAGENAME eq <app>.exe"` cho tiến trình do chính mình `dotnet run` thì vẫn hẹp — tên ảnh `<App>.Api.exe` là của riêng dự án — nhưng cũng nên kiểm trước bằng `tasklist` xem có đúng một tiến trình không.

- **`sed -i` trong git-bash (MSYS) NUỐT ký tự CR — thay chuỗi hàng loạt trên repo Windows là hỏng line-ending.** File CRLF (hoặc file TRỘN LF/CRLF, rất hay gặp trong project Visual Studio) sau khi `sed -i` bị ghi lại toàn LF: nội dung vẫn đúng, build vẫn pass, nhưng `git diff` phình từ 2 dòng lên hàng nghìn dòng giả và blame/PR review thành vô dụng. Xử lý: dùng `perl -pe 's/.../.../g'` (perl coi `\r` là phần nội dung dòng nên giữ nguyên từng dòng một, kể cả file trộn). Lỡ hỏng rồi thì dựng lại file từ blob gốc: `git show <ref>:<path> | perl -pe '<phép thay>' > <file>` — KHÔNG "sửa" bằng cách chuyển đồng loạt sang CRLF, vì file trộn sẽ sai ở đúng những dòng vốn là LF.
- **Đừng kiểm CRLF bằng `grep $'\r'` trong git-bash — luôn báo 0 dù file đầy CR.** grep của MSYS xử lý CR như phần kết dòng nên khớp không ra, khiến ta kết luận nhầm là "file không có CRLF". Đếm bằng byte thay vì bằng dòng: `tr -dc '\r' < file | wc -c`, và với blob trong git thì `git show <ref>:<path> | tr -dc '\r' | wc -c`. So sánh số này trước/sau khi sửa là cách duy nhất chắc chắn. (`file <path>` cũng nói "with CRLF line terminators" nhưng không phát hiện được file TRỘN.)
- **`dos2unix` XOÁ LUÔN BOM UTF-8 — "sửa" line-ending bằng nó là đổi thêm một thứ mình không định đổi.** Đi kèm bẫy `sed -i` ở trên: sau khi `sed` nuốt CR, phản xạ chạy `dos2unix`/`unix2dos` để nắn lại sẽ làm mất `EF BB BF` ở đầu mọi file vốn có BOM (EF Core sinh `*.Designer.cs` và `*ModelSnapshot.cs` LUÔN có BOM). Diff hiện ra dạng `-﻿// <auto-generated />` / `+// <auto-generated />` — nhìn như dòng trùng nhau nên rất dễ ngó lơ. Giữ BOM thì dùng `unix2dos -b` / `dos2unix -b`, nhưng đúng nhất vẫn là đừng đụng tới: sửa nội dung bằng `perl -pe` hoặc PowerShell đọc/ghi qua encoding `ISO-8859-1` (round-trip byte-for-byte, giữ nguyên cả BOM lẫn kiểu xuống dòng trộn).
- **`git diff --ignore-cr-at-eol` BỎ HẲN file khỏi danh sách khi khác biệt chỉ nằm ở CR — nên `join` hai bản numstat sẽ im lặng cho qua đúng những file hỏng.** Tôi tự kiểm bằng `join <(git diff --numstat) <(git diff --ignore-cr-at-eol --numstat)` rồi kết luận "sạch", trong khi `join` chỉ in dòng có khoá ở CẢ HAI vế — file thuần nhiễu EOL không có ở vế thứ hai nên bị loại khỏi kết quả, và nhiễu lọt thẳng vào commit (2026-08-19, k-rag-platform). Cách kiểm đúng: duyệt từng file `for f in $(git diff --name-only <ref>); do [ -z "$(git diff --ignore-cr-at-eol <ref> -- "$f")" ] && echo "EOL-only: $f"; done` — rỗng mới là sạch.
- **Kiểm BOM/EOL PHẢI làm trước khi commit, không phải sau.** Sau khi commit rồi thì hoặc phải xin phép sửa lịch sử, hoặc phải đẻ thêm commit "fix" chỉ để dọn nhiễu do chính mình tạo ra.
- **`curl -F "field=/duong/dan"` trong git-bash bị MSYS ĐỔI giá trị thành đường dẫn Windows — test API trả kết quả sai mà trông rất thuyết phục.** Mọi tham số bắt đầu bằng `/` đều bị MSYS coi là đường dẫn POSIX và dịch sang đường Windows trước khi tới curl: `-F "ReturnUrl=/plans"` gửi đi thật sự là `ReturnUrl=C:/Program Files/Git/plans`, `-F "ReturnUrl=/"` thành `C:/Program Files/Git/`. Triệu chứng (2026-08-20, k-rag-platform): server trả `redirect: "/dashboard"` thay vì `/plans`, tôi đã tưởng model binder của Razor Pages hỏng và suýt đi sửa code đang đúng — thực tế `Url.IsLocalUrl("C:/Program Files/Git/plans")` từ chối là hoàn toàn chính xác.
  - Nguy hiểm ở chỗ **giá trị vẫn hợp lệ về kiểu** nên không có lỗi nào nổi lên; chỉ có kết quả nghiệp vụ sai.
  - Không chỉ `-F`: dính cả `-d`, `--data`, `-H`, và đối số của mọi `.exe` không phải MSYS (docker, kubectl, sqlcmd…).
  - **TÁI PHÁT 2026-08-24** (cũng k-rag-platform, cũng Razor Pages): lần này là `--data-urlencode "returnUrl=/dashboard"` — dính y hệt, và lại mất một vòng chẩn đoán vì tôi KHÔNG đọc file này trước khi test. Dấu hiệu nhận ra ngay: giá trị có `?` hoặc `//` ở đầu thì CHẠY ĐÚNG, giá trị `/mot-tu` thuần thì hỏng — đó là chữ ký của path conversion chứ không phải của model binder.
  - Cách xác minh dứt điểm trong 1 phút: log giá trị server NHẬN ĐƯỢC (`Request.Form["x"]`). Thấy `C:/Program Files/Git/...` là xong, khỏi đoán tiếp.
  - **Fix:** `export MSYS_NO_PATHCONV=1` cho cả script, hoặc `MSYS_NO_PATHCONV=1 curl ...` cho từng lệnh. Cách né khác: viết giá trị dạng URL-encode (`%2Fplans`) khi endpoint chấp nhận, nhưng với form field thì server nhận đúng chuỗi `%2Fplans` chứ không tự giải mã — chỉ `MSYS_NO_PATHCONV` mới đúng.
  - **Luật chung:** trước khi kết luận "server bind sai/parse sai", hãy in ra thứ server THỰC SỰ nhận được (echo lại field trong response, hoặc log request) — đừng suy từ thứ mình nghĩ là đã gửi.
- **`perl -pe 's|A \|\| B|X|'` KHÔNG khớp gì nhưng vẫn CHÈN `X` vào đầu file — vì `|` vừa là dấu phân cách vừa là toán tử "hoặc".** Khi dùng `|` làm delimiter cho `s|||`, perl bóc bỏ dấu `\` trước delimiter rồi mới đưa mẫu cho regex engine, nên `\|\|` trong nguồn thành `||` trong regex = alternation với NHÁNH RỖNG ⇒ mẫu khớp chuỗi rỗng tại vị trí 0 và replacement bị chèn vào đầu file. Triệu chứng (2026-08-21, k-rag-platform): file `.cs` đột nhiên bắt đầu bằng một mẩu code lạ dính liền `using ...`, compiler báo `CS1529: A using clause must precede all other elements`. Hai lần liên tiếp tôi tưởng là lỗi khác nhau.
  - Dính với MỌI mẫu chứa `||` (điều kiện C#/JS: `a == null || b == null`), và tương tự nếu chọn delimiter `/` cho mẫu chứa `//`.
  - **Fix:** đổi delimiter sang cặp ngoặc khi mẫu có `||`: `perl -pe 's{a \|\| b}{x}'`. Nhưng nhớ perl CÂN BẰNG ngoặc với `s{}{}` — replacement chứa `{`/`}` lệch nhau (khối code C#) lại sinh lỗi `Unknown regexp modifier`. Mẫu nhiều dòng có cả `||` lẫn `{}` thì **đừng dùng perl** — sửa bằng tool Edit / thay thế chuỗi chính xác.
  - **Luật chung:** sau mỗi lần `perl -0pi -e` thay khối nhiều dòng, kiểm ngay `head -3 <file>` chứ đừng chỉ tin exit code — thay thế hỏng kiểu này im lặng hoàn toàn.

- **Heredoc dài chứa mã JS/JSON qua Bash tool trên Windows: đã hai lần cho ra kết quả sai, dùng tool `Write` thay thế.** Cùng một lệnh `cat > file.js <<'EOF' … EOF` (delimiter có nháy đơn, nội dung có `'[data-x="y"]'`, `${}`, nhiều nháy lồng nhau) lúc thì làm chính bash báo `unexpected EOF while looking for matching '` — tức nó KHÔNG coi phần thân là literal như quoted heredoc phải thế — lúc thì tạo ra file mà `node` từ chối parse. Không tốn thời gian đi tìm ký tự thủ phạm: viết file bằng tool `Write` (không qua shell), rồi mới `node --check` / chạy. Với việc chèn một khối vào giữa file có sẵn thì `awk` đọc khối từ file đã Write ra (`while ((getline line < f) > 0) print line`) là cách ghép an toàn.

- **`rm -rf <thư mục>` trong git bash báo `Device or resource busy` mà không tiến trình nào giữ file thật.** Sau khi build .NET trong thư mục đó, `rm -rf` thất bại ngay cả khi đã `cd` ra ngoài và `Get-CimInstance Win32_Process` không thấy tiến trình nào có đường dẫn đó trong command line. `Remove-Item -LiteralPath <path> -Recurse -Force` của PowerShell xoá sạch ngay lần đầu. Đừng đi giết `dotnet.exe`/MSBuild node để "giải phóng" — phiên khác trên cùng máy có thể đang build.
    + **Biến thể 2026-09-11**: `rm -rf` xoá hết NỘI DUNG nhưng còn lại đúng thư mục rỗng, và cả `Remove-Item` lẫn `rmdir` đều báo "being used by another process". Nguyên nhân: *primary working directory* của phiên Claude Code đang là chính thư mục đó (nó tự nhảy theo lệnh `cd` gần nhất). `cd` ra ngoài trong một lệnh chưa đủ — phải để một lệnh chạy xong ở thư mục cha (vd `Set-Location D:\tmp` trong PowerShell) cho tới khi harness báo đã đổi primary working directory, rồi `rmdir` mới qua. Cách tránh từ đầu: làm việc trong clone tạm bằng đường dẫn tuyệt đối / `git -C`, đừng `cd` vào nó.

- **`perl -i` mà KHÔNG có tên file trên dòng lệnh (đọc STDIN) sẽ nuốt output và ghi in-place NHẦM sang file mở bằng `<>` bên trong script — mất trắng file đích.** Dạng gây hoạ (2026-08-24, k-rag-platform): `perl -0777 -i -e 'my $block = do { local(@ARGV,$/) = ("/tmp/block.txt"); <> }; my $s = <STDIN>; ...; print $s;' < src.cs > out.cs && mv out.cs src.cs`. Perl chỉ cảnh báo `-i used with no filenames on the command line, reading from STDIN.` rồi **exit 0**: `-i` đã chuyển hướng STDOUT vào cơ chế in-place nên `out.cs` ra RỖNG, `mv` chép cái rỗng đè lên `src.cs` (mất 3332 dòng), đồng thời `/tmp/block.txt` bị ghi đè bằng nội dung vừa đọc từ STDIN — lần chèn sau lấy nhầm block khổng lồ và nhân đôi cả file.
  - **Luật:** `-i` CHỈ đi cùng danh sách tên file (`perl -0777 -i -pe '...' file.cs`). Cần đọc/ghi thủ công thì bỏ hẳn `-i`, nhận đường dẫn qua `@ARGV` rồi tự `open`/`print`/`close` — và mở file phụ bằng `open(my $fh, "<:raw", $path)`, đừng dùng `<>`/`@ARGV` cho file phụ khi `-i` đang bật.
  - **Chốt chặn rẻ:** sau mỗi phép chèn khối, kiểm `wc -l` so với trước — file rỗng hoặc phình gấp đôi là lộ ra ngay; và với repo git, `git checkout -- <file>` cứu được nhưng chỉ về mức đã commit (mọi sửa chưa commit trong file đó mất theo).
  - **An toàn hơn:** chèn/thay khối nhiều dòng trong mã nguồn thì viết script `node` đọc file, `indexOf` + `slice` (khẳng định mẫu xuất hiện đúng 1 lần rồi mới ghi), hoặc dùng thẳng tool Edit.

- **`curl -d '{"content":"xin chào"}'` từ Bash tool trên Windows gửi thân JSON KHÔNG phải UTF-8 → server .NET trả 500 và trông y hệt một bug production.** Triệu chứng (2026-08-24, k-rag-platform): mọi lời gọi POST có tiếng Việt trong thân request đều trả `500 INTERNAL_ERROR`, gọi bằng payload thuần ASCII thì 200. Đọc log server mới thấy `System.Text.DecoderFallbackException: Unable to translate bytes [E0] at index 18` ném từ `NewtonsoftJsonInputFormatter` — tức byte `0xE0` (chữ "à" theo Latin-1) lọt vào chỗ ASP.NET đang giải mã UTF-8. Tôi đã kết luận nhầm "chat production hỏng hoàn toàn" và báo cho user trước khi có log.
  - **Fix:** ghi thân request ra file bằng tool `Write` (UTF-8 thật) rồi `curl --data-binary "@file.json"`, kèm `-H "Content-Type: application/json; charset=utf-8"`. Đừng nhúng chữ có dấu thẳng vào `-d '...'`.
  - **Luật chung:** khi chỉ MÌNH bạn tái hiện được lỗi 5xx còn giao diện của user thì không, hãy nghi công cụ gọi API trước khi nghi máy chủ — và đừng công bố kết luận "production đang chết" khi chưa đọc được stack trace.

## `sed -i 'Ns|...|...|'` theo SỐ DÒNG cực dễ ghi đè nhầm sau khi vừa xoá dòng
- Kịch bản dính hai lần trong một phiên: xoá `using X;` bằng `sed -i '/^using X;$/d'` rồi mới chạy
  `sed -i '90s|.*|dòng mới|'` với số dòng lấy từ lần `grep -n` TRƯỚC khi xoá. Mọi số dòng đã dịch lên 1,
  nên lệnh ghi đè trúng dòng kế bên — có lần nuốt mất dòng `);` đóng ngoặc của một constructor, có lần
  nuốt mất dòng trống. Lỗi biên dịch sau đó chỉ ra chỗ khác, rất mất công lần.
- Cách tránh: **đừng bao giờ địa chỉ hoá bằng số dòng trong một chuỗi lệnh có xoá/thêm dòng.**
  - thay thế một dòng → `sed -i 's|chuỗi cũ duy nhất|chuỗi mới|'`
  - thay thế khối nhiều dòng → `perl -0pi -e 's/.../.../s'`
  - hoặc dùng công cụ Edit của harness (khớp chuỗi chính xác, báo lỗi nếu không duy nhất)
- Nếu buộc phải dùng số dòng: chạy lại `grep -n` NGAY TRƯỚC lệnh đó, đừng dùng số lấy từ đầu phiên.

## Heredoc `<<'EOF'` vỡ với nội dung dài
- Viết file nguồn dài (vài trăm dòng C#/JS) bằng `cat > f <<'EOF'` có lúc trả
  `unexpected EOF while looking for matching`, dù nội dung không hề chứa dấu nháy lệch hay chuỗi `EOF`.
- Dùng công cụ Write cho file dài; giữ heredoc cho đoạn ngắn (dưới ~50 dòng).

## `perl -i` + gán `@ARGV` trong script = XOÁ TRẮNG file, không báo lỗi

Muốn nhét nội dung một file khác vào phần thay thế của `perl -0pi -e`, đừng slurp bằng cách gán `@ARGV`:

```bash
# SAI — file đích thành 0 byte, exit code vẫn 0, không một dòng cảnh báo
perl -0pi -e 'my $new = do { local(@ARGV,$/) = ("/tmp/new.txt"); <> }; s{...}{$new}s;' target.cshtml
```

`-i` (ghi ngược tại chỗ) chạy bằng CHÍNH `@ARGV`: perl đọc tên file cần sửa từ đó, mở file tạm rồi
đổi tên đè lên. Gán `local(@ARGV, ...)` bên trong script cướp mất mảng đó giữa chừng, nên perl kết thúc
vòng lặp mà không còn file nào để ghi — kết quả là file đích bị cắt còn rỗng. Nguy hiểm ở chỗ **không có
lỗi nào**: exit code 0, stderr trống, và lệnh `sed -n` kiểm tra ngay sau đó chỉ im lặng vì file đã rỗng.

```bash
# ĐÚNG — truyền qua biến môi trường, @ARGV để yên cho perl
NEW=$(cat /tmp/new.txt) NEW="$NEW" perl -0pi -e 'my $new = $ENV{NEW} . "\n"; s{...}{$new}s;' target.cshtml
```

Kèm theo hai thói quen: chạy `perl -i` trên file ĐÃ commit (còn `git checkout -- <file>` mà cứu), và sau
mỗi lần sửa hàng loạt thì `wc -l` file vừa đụng — 0 dòng là dấu hiệu duy nhất của kiểu hỏng này.

## `sed` nhiều lệnh thay thế: tên ngắn ăn mất tên dài hơn chứa nó

Đổi tên hàng loạt bằng một file `.sed` nhiều dòng thì THỨ TỰ quyết định kết quả, vì `sed` áp lần lượt
từng lệnh lên cùng một dòng. Nếu có `Foo.Bar(` và `ProxyFoo.Bar(`, lệnh thay `Foo.Bar(` chạy trước sẽ
cắn vào phần đuôi của `ProxyFoo.Bar(` và sinh ra rác kiểu `Proxynew Bar(...)` — vẫn "thành công",
không lỗi, chỉ vỡ lúc build.

```bash
# SAI: dòng RedirectorRunner khớp luôn phần đuôi của ProxyRedirectorRunner
s|RedirectorRunner\.RunAsync(|new RedirectorRunner(x).RunAsync(|g
s|ProxyRedirectorRunner\.RunAsync(|new ProxyRedirectorRunner(x).RunAsync(|g
```

Cách tránh:
- Đặt **tên dài/cụ thể hơn TRƯỚC** tên ngắn.
- Hoặc neo đầu chuỗi: `s|\bRedirectorRunner\.|...|` — nhưng `\b` KHÔNG chặn được ở đây vì `ProxyRedirectorRunner`
  không có ranh giới từ trước `Redirector`; phải neo bằng ký tự thật đứng trước (khoảng trắng, `await `, `(`).
- Sau khi chạy, `grep` lại chính chuỗi mới sinh để chắc không có mảnh ghép lạ trước khi build.

## `cat > f <<'EOF'` với thân RỖNG, kèm `|| (...)` → treo tới hết timeout

```bash
# SAI — treo 2 phút rồi bị giết
cat > path/file <<'EOF' 2>/dev/null || (mkdir -p path && cat > path/file)
EOF
```

Heredoc gắn với lệnh `cat` ĐẦU TIÊN. Khi nó thất bại (thư mục chưa có), nhánh `||` chạy một `cat` khác
**không có heredoc nào** nên nó đọc stdin và ngồi đợi mãi. Tách hẳn ra: `mkdir -p` trước, rồi mới ghi file.

## Heredoc của Bash tool NUỐT một dấu gạch chéo ngược: `\\` ghi ra thành `\`

Ghi file bằng `cat > f <<'EOF'` (delimiter ĐÃ trích dẫn, đúng ra là literal 100%) mà thân có `\\`
thì file nhận được chỉ còn `\`. Vấp thật: chép nguyên một file C# có `if (value[1] is '/' or '\\')`
→ file ghi ra thành `'\'` ⇒ `error CS1010: Newline in constant`. Một `\` đơn (vd `\n` trong regex
Python) thì KHÔNG sao, chỉ cặp `\\` mới bị rút gọn.

Cách tránh khi nội dung có escape sequence:
- Đừng viết `\\` thẳng trong heredoc. Sinh nó bằng code: `b = chr(92)` rồi ghép chuỗi (Python), hoặc
  dùng Write tool.
- Với sửa tại chỗ, viết script `.py` ra scratchpad rồi chạy `py script.py` — vẫn là heredoc nên vẫn phải
  tránh `\\` trong thân script.
- Luôn `grep` lại đúng dòng có escape sau khi ghi, trước khi build.
- Biến thể nguy hiểm hơn: ghép đường dẫn Windows kiểu `"$DIR\\$name.png"` trong script ghi bằng heredoc
  → file nhận `"$DIR\$name.png"`, mà `\$` là thoát dấu đô-la ⇒ biến KHÔNG được mở, Chrome ghi ảnh ra
  đúng cái tên `...scratchpad$name.png`. Với đường dẫn Windows trong bash, dùng luôn gạch chéo xuôi
  (`C:/Users/.../$name.png`) — Chrome/.NET/đa số tool Windows đều nhận.

## `perl -pi -e` với ký tự non-ASCII trong chuỗi thay thế → hỏng encoding cả file (mojibake)

- Triệu chứng: sau khi chạy `perl -0pi -e 's/.../...\x{2014}.../s' file.cs`, perl in cảnh báo `Wide character in print`, và **mọi ký tự UTF-8 có sẵn trong file bị hỏng**: `§` → `Â§`, `—` → `â€"`. Code vẫn build (chỉ là comment) nhưng diff phình ra và tiếng Việt/ký tự đặc biệt thành rác.
- Nguyên nhân: chỉ cần MỘT ký tự non-ASCII (kể cả viết dạng `\x{2014}`) là chuỗi thành "wide"; perl không có layer `:encoding(UTF-8)` nên đọc file như latin1 rồi ghi lại thành UTF-8 ⇒ mã hoá hai lần.
- Cách tránh:
  - Thay thế chỉ dùng ASCII, hoặc
  - `perl -CSD -Mopen=':std,:encoding(UTF-8)' -0pi -e ...`, hoặc
  - **An toàn nhất**: không nhúng văn bản vào lệnh. Trích đoạn cần chèn ra file bằng `sed -n 'a,bp' nguồn > snip.txt` (thuần byte) rồi chèn bằng `awk '{print} /neo/ {while ((getline l < "snip.txt") > 0) print l}'`.
- Kiểm tra sau khi sửa file có ký tự non-ASCII: `grep -c 'Â\|â€' file` phải ra 0.
- Tái phạm 2026-09-07 (repo k-rag-platform, sửa `source-preview.js` có comment tiếng Việt): viết `\x{1eaf}`… trong chuỗi thay thế vẫn dính đúng cái bẫy này — **`\x{}` KHÔNG cứu được**, nó vẫn làm chuỗi thành wide. Chốt lại luật thực dụng: **file có tiếng Việt thì dùng thẳng tool `Edit`, đừng perl/sed**; auto mode ưu tiên Bash nhưng đây là chỗ Bash "genuinely cannot do the job".
- Cứu khi đã lỡ: file chưa sửa gì khác thì `git checkout -- <path>` rồi làm lại bằng `Edit`. Nhớ kiểm `git status` trước để không mất các sửa đổi khác trong cùng file.
- Tái phạm 2026-09-09 (repo k-rag-platform, chèn khối `catch` vào `ExceptionWrapperMiddleware.cs`): lần này KHÔNG nhúng chữ vào `-e` mà để khối cần chèn trong file tạm rồi `BEGIN { open(my $fh, "<:encoding(UTF-8)", ...) }` — vẫn dính, vì layer `:encoding(UTF-8)` decode khối thành wide char, perl bật ngầm utf8 cho STDOUT, thế là phần CÒN LẠI của file (đọc dạng byte) bị encode lần hai. Dấu hiệu nhận ra nhanh: `git diff --stat` chỉ ra vài dòng — đúng bằng số dòng có tiếng Việt trong file — chứ không phải cả file.
  - Sửa: bỏ layer, đọc thuần byte `open(my $fh, "<", $file); binmode($fh);` thì cả khối chèn lẫn nội dung cũ đều đi qua nguyên xi.
  - `python3` KHÔNG có trên máy này (Git Bash) — đừng lấy python làm đường lui.

## `perl -0pi -e` với chuỗi lạ: đừng dùng, hỏng file cả loạt

Pattern chứa em-dash (`—`), dấu `|` (kể cả khi `|` là delimiter và đã escape `\|`), hoặc chuỗi nội suy C# (`$"{verb}:{pid}"`) làm `s///` trượt theo cách **không báo lỗi mà vẫn ghi file**. Ba kiểu hỏng đã gặp thật:
- chèn khối thay thế vào **đầu file** thay vì đúng chỗ (file `.cs` mất `using`);
- prepend chuỗi thay thế vào **mọi dòng** của file markdown 800 dòng;
- `$"` và `{...}` trong replacement bị perl nội suy mất, còn lại `PidCalls.Add(:{pid}");`.

**Cách làm đúng**: dùng Edit tool (khớp chuỗi nguyên văn, báo lỗi nếu không khớp) cho mọi sửa có ký tự ngoài ASCII; thay/chèn cả khối trong file lớn thì `head -N file > new && cat block >> new && tail -n +M file >> new && mv new file` — đếm dòng bằng `grep -n` trước, và làm **từ dưới lên** nếu có nhiều khối.

**Luôn kiểm sau khi ghi**: `head -3` + `grep -n` mốc quen thuộc. File đã commit thì `git checkout -- <path>` cứu được; file chưa commit thì mất.

## Heredoc chứa nhiều `'''` của Python → `unexpected EOF while looking for matching '`

Bash tool bọc lệnh trong `bash -c '...'`, nên **mọi dấu nháy đơn trong thân heredoc vẫn được đếm**,
kể cả khi delimiter đã trích dẫn (`<<'PY'`). Script Python dùng chuỗi `'''...'''` cho các đoạn code
nhiều dòng thì rất dễ lẻ số nháy → bash báo lỗi ở dòng CUỐI (`line 161: unexpected EOF`), làm tưởng
nhầm là heredoc thiếu delimiter.

**Cách làm chạy được**: viết file script bằng tool `Write` vào scratchpad rồi gọi
`py "<đường dẫn>"`. Không mất thời gian đoán chỗ lẻ nháy, và sửa lại script cũng dễ hơn.

Hai bẫy đi kèm khi chạy Python từ Git Bash trên Windows:

- `python3` KHÔNG có trong PATH (`command not found`) dù Python đã cài — phải gọi `py`. Kiểm bằng
  `python3 --version || py --version` thì output không nói cho biết lệnh nào đã chạy; thử riêng.
- `print` chuỗi tiếng Việt chết với `UnicodeEncodeError: 'charmap' codec` vì console mặc định cp1252
  — **và nó chết ở CUỐI script, sau khi mọi thay đổi file đã ghi xong**, nên đọc exit code 1 rồi
  chạy lại là sửa file hai lần. Đặt `export PYTHONIOENCODING=utf-8` trước khi gọi `py`.

## `grep -c $'\r'` trong msys đếm TỔNG SỐ DÒNG, không đếm CR → tưởng nhầm file là CRLF

- Triệu chứng (2026-09-11): `grep -c $'\r' docs/Glossary-vi.md` trả về đúng bằng `wc -l` cho một file **LF thuần** (HEAD lưu LF, `core.autocrlf=false`). Tin vào đó rồi chạy `perl -pi -e 's/\r?\n$/\r\n/'` → file bị CRLF-hoá toàn bộ, `git diff` phình thành cả file (585+/565−).
- Nguyên nhân: msys nuốt `\r` khi truyền đối số nên pattern thành chuỗi rỗng → khớp mọi dòng. Cùng gốc với bẫy "grep/cat -A không thấy `\r`" đã ghi ở trên.
- Cách đo đúng: `tr -cd '\r' < file | wc -c` (số byte CR) so với `wc -l`; đối chiếu HEAD bằng `git show HEAD:path | tr -cd '\r' | wc -c`. Chỉ khi HEAD có CR mới được chuẩn hoá về CRLF; ngược lại cứu bằng `perl -pi -e 's/\r\n$/\n/'`.
- Bẫy đi kèm: `$'\r'` lồng trong `"$( ... )"` làm bash báo `unexpected EOF while looking for matching ')'` — tách ra lệnh riêng.

## awk `getline line < "file"` với đường dẫn tương đối: resolve theo CWD lúc chạy awk, không theo chỗ đặt script

- Triệu chứng: script awk thay dòng bằng nội dung snippet (`while ((getline l < f) > 0) print l`) chạy từ thư mục repo trong khi snippet nằm ở scratchpad → `getline` trả −1, vòng lặp không in gì, nhưng cờ `replaced=1` vẫn bật ⇒ mọi dòng "được thay" biến mất, output ngắn bất thường mà không có lỗi nào.
- Cách tránh: `cd` vào thư mục snippet trước khi gọi awk, hoặc ghi đường dẫn tuyệt đối trong control file; và luôn kiểm số dòng + heading của output trước khi ghi đè file thật (`grep -nE '^#{2,4} '`).
- Cùng phiên: một lệnh Bash gộp ~30 KB heredoc bị `ENAMETOOLONG: name too long, uv_spawn` (không chạy gì cả) — chia thành nhiều lệnh dưới ~10 KB, hoặc dùng tool Write cho snippet dài.

## Heredoc qua Bash tool ăn mất một lớp backslash — code C#/JS có `"\\"` ghi ra file thành `"\"`

- Gặp 2026-09-20 (VisualStudioInternetDebug, M0): viết file `.cs` bằng `cat > f <<'EOF'` với delimiter
  ĐÃ nháy đơn (lẽ ra heredoc quoted không xử lý escape gì cả), nhưng nội dung `path.Replace('\', '/')`
  ghi ra đĩa thành `path.Replace('\', '/')` ⇒ `error CS1012: Too many characters in character literal`.
  Cùng lý do, một lệnh gộp nhiều heredoc có `'\'` chết ngay ở bước parse: `unexpected EOF while looking
  for matching '''` — **không file nào được ghi**, dễ tưởng là heredoc thiếu delimiter.
- Nguyên nhân: lớp truyền lệnh vào Bash tool tự bóc một tầng escape trước khi bash thấy nội dung, nên
  delimiter được nháy cũng không cứu được.
- **How to apply**: nội dung có backslash (đường dẫn Windows, regex, chuỗi C# `"\\"`, `\n` trong code) thì
  đừng viết bằng heredoc — dùng tool Write/Edit. Nếu đã viết rồi thì `grep -n '\\' file` (hoặc mở đúng
  dòng) để soát; sửa từng chỗ bằng Edit. Ký tự escape khác (`\"`, `\u0001`) đi qua bình thường.

- Cùng nguyên nhân đó, `perl -e '...'` / `sed`/`awk` inline cũng dính: chuỗi perl viết hai backslash liền
  (`"...Resources\\appicon.ico..."`) lẽ ra ra một dấu `\`, nhưng qua Bash tool thì backslash biến mất hẳn (chuỗi ngắn đi 1 ký tự,
  `index()` trả -1, thay thế im lặng không khớp). Triệu chứng: script báo "not replaced" dù `grep` thấy
  đúng dòng đó trong file.
- **How to apply**: trong perl inline, viết backslash bằng `chr(92)` (vd
  `"<ApplicationIcon>Resources" . chr(92) . "appicon.ico</ApplicationIcon>"`) thay vì hai backslash liền. Hoặc tránh hẳn
  backslash bằng cách neo vào đoạn không có nó (dùng `(?=\Qmốc\E)` để chèn trước một dòng mốc). Luôn cho
  script `die` khi số lần thay thế != 1 — nếu không thì lỗi này trôi qua mà file không đổi.
- **Gặp lại 2026-09-23 (k-rag-platform)**: cùng lỗi với `perl -pi -e '...'` có `[/\]` trong character class → `\]` thoát dấu đóng ngoặc, class nuốt cả phần regex phía sau, khớp nhầm hàng loạt (`Docs/` trần cũng bị đổi); replacement `..\..` ra `....`. Và `cmd //c "Docs\x\${v}"` cũng hỏng. Cách làm đúng: viết perl ra FILE script (`perl -pi script.pl files`) và chạy `.cmd` qua tool PowerShell. Luôn soát `git diff` ngay sau lệnh thay hàng loạt.

## Script trong scratchpad sống qua compact — trùng tên + lệnh ghi hỏng là chạy nhầm script CŨ
- 2026-09-26: một lệnh dạng `cat > x 2>/dev/null; cat > scratch/fix2.js <<'EOF' ... EOF; node scratch/fix2.js` bị treo ở lệnh `cat > x` đầu tiên (không có heredoc nên nó đọc stdin), harness đẩy nó ra nền rồi tôi dừng. Lần chạy lại chỉ gọi `node scratch/fix2.js` — nhưng file đó chưa hề được ghi lại, nên node chạy một `fix2.js` còn sót từ đợt làm việc trước, và nó chèn khối trùng vào hai file không liên quan. Không có lỗi nào, chỉ in vài dòng MISS/OK lạ.
- Luật: đặt tên script scratchpad theo việc + đợt (`p14-email-fix.js`), không dùng `fix1/fix2`; ghi script bằng tool Write rồi mới chạy; và sau mỗi script sửa hàng loạt chạy `git status --short` xem có file nào ngoài dự kiến bị đụng.
- Không bao giờ viết `cat > file` mà không có heredoc/pipe đi kèm — nó chờ stdin tới hết timeout.
