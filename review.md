# Review & phản biện

Đọc trước khi giao agent review. Phần quyết định "có cần review không" nằm ở `Quy tắc review & phản biện` trong `~/.claude/CLAUDE.md`.

## Khi nào review

- Chỉ đợt sửa PHỨC TẠP mới cần review; việc nhỏ thì tự kiểm rồi commit luôn.
- Đợt phức tạp thì trước `git commit` của đợt đó, toàn bộ sửa đổi (code, script, config, tài liệu) phải qua **một** lượt subagent review độc lập. Review theo ĐỢT/theo commit: KHÔNG review từng edit lẻ, cũng không dồn cả branch lại review một lần ở cuối. Không dùng git thì mốc là "trước khi báo với user là xong".
- **Thế nào là phức tạp (CẦN review)**: đụng logic nghiệp vụ/luồng chạy; đổi kiến trúc, hợp đồng API, schema DB/migration; xoá hoặc ghi dữ liệu không hoàn tác được; quyền hạn/xác thực/tính tiền; đồng thời (async, lock, transaction); trải nhiều file hoặc nhiều tầng; hoặc chính mình thấy chưa chắc về một chỗ trong đợt.
- **KHÔNG cần review**: sửa thuần cơ học không đổi hành vi (typo, format, đổi tên, bump version, comment); sửa nhỏ gọn trong một chỗ mà mình đã tự kiểm được bằng build/test/số đo thật; chỉnh giao diện/CSS/chữ hiển thị; và mọi lúc user nói "khỏi review". Bỏ qua thì phải NÓI RÕ trong câu trả lời là đã bỏ qua và vì sao, không im lặng: user cần biết bản này chưa có ai phản biện.
- **Why**: người vừa viết code mù đúng điểm mù của mình, nên chỗ khó thì cần một cặp mắt ngữ cảnh mới phản biện trước khi nó đi vào commit; còn bắt mọi thay đổi vặt cũng qua agent thì chỉ tốn thời gian/token và làm loãng những lượt review thật sự quan trọng.

## Giao việc cho agent review

- Agent chạy ngữ cảnh mới nên phải đưa đủ: yêu cầu gốc của user, **path tuyệt đối + số dòng** các chỗ đã sửa (agent không click được markdown link; `Quy tắc link code` chỉ dùng khi báo cho user), và chỉ đích danh file rule cần đối chiếu (vd `~/.claude/csharp.md`) chứ đừng bắt nó mò cả thư mục.
- Bảo agent tự đọc hiện trạng thay vì dán code cho nó: `git status --short`, `git diff` (chưa stage) **và** `git diff --cached` (đã stage; thiếu cái này agent sẽ kết luận nhầm "không có thay đổi nào"); đợt gồm nhiều commit rồi thì thêm `git diff <base>...HEAD`. File ngoài git (vd `~/.claude/*.md`) thì đưa path + khoảng dòng để agent tự đọc.
- **Nhiệm vụ của agent**: KHÔNG được gật đầu cho xong. Phải (1) tìm lỗi logic, trường hợp biên, rò rỉ tài nguyên/lifetime, lỗi đồng thời, (2) đối chiếu quy ước & kiến trúc project, (3) nêu ít nhất một phản biện về cách tiếp cận: có cách đơn giản/đúng hơn không, giả định nào chưa kiểm chứng. Kết luận phải là: vấn đề cụ thể + vị trí + mức nghiêm trọng; nếu thật sự không có vấn đề thì liệt kê đã kiểm những gì, cấm trả lời chung chung kiểu "code ổn". Agent review chạy READ-ONLY: không tự sửa file, không spawn agent con.
- **Phạm vi review**: chỉ soi chỗ SỬA và chỗ bị ảnh hưởng TRỰC TIẾP bởi nó (hàm/khối vừa đổi, nơi gọi trực tiếp nó, dữ liệu/trạng thái nó ghi ra). Ảnh hưởng gián tiếp thì KHÔNG quét sâu: không đi rà cả tầng gọi ở xa, không audit những phần cũ mà đợt này không đụng tới, không mở rộng thành review toàn module. Thấy mùi lạ ở chỗ xa thì nêu một dòng nhắc rồi thôi, đừng đào.
  - **Why**: quét sâu vô tội vạ vừa tốn thời gian/token vừa đẩy ra một rổ finding không liên quan tới đợt đang commit, làm lụt mất finding thật của chỗ vừa sửa.
- **Đợt lớn thì chia review cho nhiều agent** (theo `~/.claude/agents.md`): diff trải nhiều file/tầng thì một lượt review gồm nhiều agent chạy song song, mỗi agent một phần không chồng nhau, theo nhóm file/tầng (vd Repositories, Services, Controllers + UI, migration + tài liệu) hoặc theo khía cạnh nếu đợt gọn trong ít file (logic nghiệp vụ, đồng thời/transaction, quyền hạn).
  - Brief mỗi agent chỉ đúng path + dòng thuộc phần của nó, kèm một câu tóm tắt cả đợt để nó biết chỗ nối với phần khác; chỗ nối giữa các phần (hợp đồng API, DTO, tên cột) giao rõ cho một agent soi.
  - Mỗi agent trả về gọn danh sách finding (vị trí, mức, lý do); phiên chính gộp, bỏ trùng rồi mới kiểm lại.
  - Đợt nhỏ vẫn một agent. Chống vòng lặp tính theo lượt, không theo số agent.
- **Chọn model**: tính theo độ khó của thứ BỊ review. Đợt cơ học/tài liệu → `sonnet`; mọi đợt còn lại (đụng logic, trải nhiều file, đổi kiến trúc hoặc luồng chạy phức tạp) → `opus`. `fable` chỉ chạy khi user yêu cầu đích danh, hoặc khi mình hỏi trước và user đồng ý (luật chung ở `~/.claude/agents.md`). Đợt thật sự khó thì cứ nói thẳng "đợt này nên để `fable` review, có dùng không?" rồi chờ trả lời.

## Sau khi có kết quả

- Báo user tóm tắt các điểm agent nêu, rồi hoặc sửa theo, hoặc nói rõ lý do bác bỏ; KHÔNG im lặng bỏ qua.
- KHÔNG tin agent vô điều kiện: tự mở đúng chỗ agent chỉ để kiểm lại trước khi sửa theo, vì agent cũng hay báo sai một cách nghe rất hợp lý. "Phải nêu ít nhất một phản biện" là để agent không gật bừa, KHÔNG phải lệnh bắt buộc sửa.
- **Chống vòng lặp**: bản sửa theo finding chỉ review lại khi nó đụng logic mới; tối đa **2 vòng** cho một đợt. Còn bất đồng thì dừng, commit, và báo user phần còn treo.
