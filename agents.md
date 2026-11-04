# Giao việc cho agent và chia nhỏ việc nặng

Đọc trước lần đầu trong phiên gọi `Agent`/`Workflow`/fork, chạy lệnh nền dài, hoặc giao việc cho CLI agent bên ngoài. Phần tóm tắt bắt buộc nằm ở mục "Agent & việc nặng" trong `~/.claude/CLAUDE.md`.

## Quy tắc chọn model cho agent

Khi giao việc cho subagent (tool `Agent`, tham số `model`), chọn theo độ phức tạp của việc, KHÔNG để trống rồi kế thừa model phiên:

- **Đơn giản** (tìm file/grep, đọc & tóm tắt, sửa cơ học, đổi tên, chạy lệnh rồi báo kết quả, viết/sửa code đơn giản đã rõ phải làm gì) → `sonnet`.
- **Sửa code phức tạp hơn một chút** (sửa logic vài chỗ, phải hiểu luồng mới biết sửa gì) → tách hai bước: `opus` (phiên chính hoặc agent `opus`) phân tích và chốt danh sách sửa cụ thể (path + dòng, sửa thành gì, lý do), rồi giao `sonnet` làm đúng theo danh sách đó; xong thì phiên chính tự kiểm lại (đọc diff, build/test). Chỉ để `opus` tự sửa luôn khi các chỗ sửa đan xen tới mức không viết thành danh sách rõ ràng được.
- **Phân tích thuần** (phân tích nhiều file, lần luồng chạy, dựng kế hoạch) → `opus`. Riêng agent review thì tính theo độ khó của thứ BỊ review, xem `~/.claude/review.md`.
- **Cực kỳ phức tạp** (thiết kế kiến trúc lớn, debug nhiều tầng, suy luận dài nhiều ràng buộc) → mặc định `opus`. Thấy việc thật sự cần `fable` thì ĐỀ XUẤT và xin phép trước, theo ý ngay dưới đây.
- **Dùng `fable` cho subagent LUÔN phải xin phép user**:
  - Áp cho MỌI subagent: tool `Agent`, `agent()` trong Workflow, agent review, fork.
  - Áp **kể cả khi phiên chính đang chạy `fable`**. Phiên chính dùng `fable` là lựa chọn của user cho cuộc trò chuyện, KHÔNG phải giấy phép cho các agent con.
  - Cách làm: hỏi TRƯỚC khi spawn, nói rõ agent nào, làm việc gì, và vì sao cần `fable` thay vì `opus`; chờ trả lời rồi mới chạy. Chưa hỏi được, hoặc user chưa trả lời, thì chạy `opus`.
  - Luôn truyền `model` tường minh. Riêng subagent kiểu fork luôn kế thừa model của phiên (tham số `model` bị bỏ qua), nên phiên đang chạy `fable` thì dùng fork cũng tính là dùng `fable`: hoặc xin phép, hoặc đổi sang agent thường có `model` tường minh.
  - Sự đồng ý chỉ có giá trị cho đúng agent/đợt việc đã hỏi, KHÔNG thành quyền dùng lâu dài, trừ khi user tự nói phạm vi rộng hơn (vd "cả phiên này cứ dùng fable").
  - **Why**: `fable` đắt và chậm hơn hẳn các model khác; user muốn tự quyết từng lần dùng.
- **BÁO CÁO**: mỗi lần giao việc phải nói rõ trong câu trả lời agent nào chạy model nào và vì sao chọn mức đó; giao nhiều agent thì liệt kê từng agent. Agent xong thì ghi thêm `subagent_tokens` lấy từ khối usage trong kết quả tool `Agent` (chỉ thấy khi agent đã xong), vd `Explore → sonnet (tìm file), 71k token`.
- **Why**: model mạnh cho việc vặt chỉ tốn token và chậm hơn, còn model yếu cho việc khó thì trả kết quả sai mà vẫn nghe hợp lý. User cần thấy mức model để biết có nên tin kết quả không, và thấy số token để theo dõi chi phí.

## Quy tắc chia nhỏ việc nặng

Phiên chính giữ vai **điều phối** (kế hoạch, quyết định, kết luận, kiểm lại diff); phần việc làm phình ngữ cảnh thì giao subagent, chỉ nhận về kết luận ngắn.

**Why**: mỗi request (tức mỗi lần gọi tool) gửi lại toàn bộ lịch sử phiên. Có cache vẫn tính phí đọc cache theo độ dài lịch sử: phiên 800k tốn gấp 4 lần phiên 200k cho mỗi lượt gọi tool, một prompt có 20 lượt gọi tool là nhân 20. Lỡ cache hết hạn thì phải ghi lại cả lịch sử (phiên 800k tốn vài USD cho một lần). Ngữ cảnh quá dài còn làm sót chi tiết cũ. Các task nặng từng chạy một mạch trong phiên chính đã phình tới 600–900k token.

### Chia việc

- **Việc nặng thì chia trước khi làm**: việc trải nhiều file/tầng, nhiều bước, hoặc sẽ phải đọc nhiều thì lúc lên kế hoạch tách thành các phần có đầu vào và đầu ra rõ ràng; mỗi phần giao một agent, chạy song song khi các phần không phụ thuộc nhau.
- **Nên giao agent**: đọc/grep/khảo sát nhiều file; sửa cơ học lặp lại trên nhiều file; chạy build/test hoặc lệnh có output dài rồi báo lỗi chính; review theo `~/.claude/review.md`.
- **Không chia quá vụn**: việc nhỏ, đọc 1–2 file đã biết chỗ, sửa gọn một chỗ thì tự làm. Mỗi agent tốn chi phí khởi động (system prompt + brief) và phải tự dựng lại ngữ cảnh, nên chia chỉ đáng khi phần việc tự nó sẽ đọc nhiều hơn thế.
- **Subagent KHÔNG được tạo thêm subagent** (tool `Agent`, `Workflow`, fork): chỉ phiên chính điều phối và giao việc. Việc giao cho agent mà quá lớn thì phiên chính tự chia thành nhiều agent ngang hàng từ đầu, hoặc agent làm phần của mình rồi báo về phần còn lại để phiên chính giao tiếp.
  - **Why**: agent con lồng nhau nằm ngoài tầm mắt phiên chính: không thấy brief, không chọn được model, không kiểm được kết quả trung gian, token phình (một agent từng tự đẻ 5 agent con tốn ~860k token), và để lại file rác trong repo mà không ai biết nguồn.
- **Đừng nối dài một agent đã nặng**: `SendMessage` tiếp vào agent cũ thì nó giữ nguyên toàn bộ lịch sử, và không có cách nào nén ngữ cảnh của subagent (`/compact` chỉ user gõ được, model không gọi được). Agent trả về `subagent_tokens` lớn (khoảng > 150k) thì phần việc kế tiếp giao cho agent MỚI, kèm brief tóm tắt những gì agent cũ đã tìm ra; chỉ `SendMessage` agent cũ cho câu hỏi ngắn bám sát việc nó vừa làm.
- **Mốc chuyển phần việc**: xong một phần lớn mà phiên đã dài (đã đọc nhiều file lớn, nhiều vòng build/test) thì lưu tiến độ vào memory rồi đề xuất user `/compact`, hoặc mở phiên mới cho phần kế tiếp.

### Brief giao agent

Agent không thấy lịch sử phiên, nên brief phải có:

- Mục tiêu, path tuyệt đối + dòng, quyết định đã chốt, file rule cần đọc.
- Luật cấm: không tạo subagent con (`Agent`, `Workflow`, fork), không tạo cron.
- **Trả về gọn**: không dán nguyên file/log, chỉ trả kết luận, path + dòng, lỗi chính, việc còn treo.
- Agent chạy lệnh dài thì dặn luật **lọc nhưng không mất log** (mục dưới).
- Agent sửa code thì dặn `Quy tắc line ending` trong `~/.claude/CLAUDE.md`.

### Lọc output nhưng không mất log

- Lệnh tự chạy trong phiên chính cũng lọc output (`head`/`tail`/chỉ lấy dòng lỗi) thay vì in hết.
- Lệnh dài (test, build, script chạy lâu) thì ghi log ĐẦY ĐỦ ra file trong scratchpad (vd `tee`, `dotnet test --logger "console;verbosity=normal"` hoặc `trx`) rồi mới lọc; KHÔNG chạy chế độ ít log (`-v q`) rồi chỉ lấy dòng tổng kết. Cần xem chi tiết (bài nào đỏ, lỗi gì) thì đọc lại file log thay vì chạy lại.
  - **Why**: (2026-10-09) bộ E2E ~14 phút chạy với `-v q` chỉ báo "10 đỏ" mà không có tên bài, phải chạy lại cả bộ mới biết bài nào.

### Chạy nền, không khoá phiên chính

- Agent và lệnh dài (build, test, cài gói, script chạy quá ~30 giây) chạy nền (`run_in_background: true` của `Agent`/`Bash`/`PowerShell`), để user vẫn nhắn chen được trong lúc chờ.
- Giao xong thì báo ngắn đã giao gì (agent nào, model gì, lệnh gì), rồi làm tiếp phần KHÔNG phụ thuộc kết quả; hết việc thì kết thúc lượt và chờ thông báo hoàn tất. Không poll, không `sleep` chờ, không đoán trước kết quả.
- Chỉ chạy tiền cảnh (`run_in_background: false`) khi bước ngay sau cần kết quả VÀ việc ngắn (lệnh vài giây, agent tra một chỗ).
- User nhắn chen khi việc nền đang chạy: xử lý tin đó trước; nếu tin mới làm việc nền mất ý nghĩa thì dừng nó (`TaskStop`) thay vì để chạy tiếp tốn token.
- Việc nền đụng cùng file với việc đang làm (build trong lúc sửa code, hai agent sửa cùng file) thì không chạy song song: chờ xong hoặc tách phạm vi.
- **Why**: agent/lệnh chạy tiền cảnh khoá phiên chính tới khi xong; user không chen được yêu cầu mới hay sửa hướng giữa chừng.

### Cron (chỉ phiên chính đặt)

- **Giữ cache khi chạy lâu**: việc dài (agent/lệnh chạy nền lâu, kế hoạch nhiều đợt chạy liên tục) thì kiểm `CronList` xem đã có cron job mỗi 30 phút giữ prompt cache sống chưa; chưa có thì tạo (`CronCreate`, `recurring: true`, prompt ngắn kiểu "kiểm agent đang chạy đã báo chưa; chưa thì chỉ trả lời một dòng 'vẫn chạy'").
  - Sau khi ngữ cảnh bị compact, việc ĐẦU TIÊN là kiểm lại cron này (job chỉ sống trong phiên, tự hết hạn sau 7 ngày).
  - Việc dài đã xong thì xoá cron (`CronDelete`).
  - KHÔNG ghi trạng thái cron (job ID, "đã đặt cron") vào memory hay file nào: luôn hỏi `CronList` mới biết thật.
  - **Why**: prompt cache hết hạn khi phiên đứng yên chờ agent nền lâu; lượt kế tiếp phải ghi lại cả lịch sử, phiên dài tốn vài USD mỗi lần.
- **Canh CLI agent chạy nền**: khi giao việc cho một CLI agent bên ngoài (Gemini CLI, Codex/OpenAI CLI, Deep Code…) chạy nền, NGAY sau khi dispatch đặt thêm cron job/loop định kỳ (vd 10 phút, `CronCreate` hoặc `Monitor`) kiểm có file nào được sửa không (`git status`/mtime worktree, log của CLI). Hai lượt liên tiếp không đổi thì coi là kẹt: báo user, dừng tiến trình hoặc chuyển việc sang agent khác, không ngồi chờ hết timeout. Task xong thì xoá cron.
  - **Why**: CLI agent có thể đứng im (vd chờ hộp hỏi quyền không ai trả lời) tới hết timeout mà không sửa gì, log trống; user phải tự phát hiện.
