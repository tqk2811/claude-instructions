# Kinh nghiệm PostgreSQL / Npgsql

## Xoá dòng cha bị khoá ngoại chặn: RESTRICT ném `23001`, NO ACTION mới ném `23503`

- Khoá ngoại khai `ON DELETE RESTRICT` (EF Core: `DeleteBehavior.Restrict`) mà xoá dòng cha còn con thì PostgreSQL
  ném `23001 restrict_violation` (`PostgresErrorCodes.RestrictViolation`), KHÔNG phải `23503 foreign_key_violation`.
  `23503` là của `NO ACTION` (mặc định khi không khai) và của phía INSERT/UPDATE dòng con trỏ vào dòng cha không còn.
- **Why:** gặp thật 28/9/2026 (bộ dọn tệp mồ côi): code chỉ bắt `ForeignKeyViolation`, test đơn vị xanh vì không
  chạm database thật; chỉ lộ khi đo đua bằng hai kết nối trên PostgreSQL local — lệnh xoá ném xuyên ra ngoài.
- **How to apply:** bắt lỗi "bị khoá ngoại chặn" ở lệnh DELETE thì so cả hai mã:
  `SqlState is PostgresErrorCodes.RestrictViolation or PostgresErrorCodes.ForeignKeyViolation`. Phía INSERT dòng con thì
  vẫn là `23503`. `ExecuteDeleteAsync` ném `PostgresException` trần; `SaveChanges` bọc trong `DbUpdateException`
  (xem `InnerException`).
- Kiểm hành vi đồng thời/khoá ngoại thì đo trên PostgreSQL thật (hai `NpgsqlConnection`, một bên mở transaction giữ khoá),
  đừng tin suy luận hay DB giả.

## `SELECT … FOR UPDATE` ở READ COMMITTED không thấy dòng MỚI được commit trong lúc chờ khoá (2026-10-13)

Câu khoá chụp snapshot lúc BẮT ĐẦU. Phải chờ khoá xong thì PostgreSQL chỉ đọc lại phiên bản mới nhất của chính các dòng đã
khớp (EvalPlanQual); dòng vừa INSERT/UPDATE cho khớp điều kiện trong lúc chờ KHÔNG được thêm vào kết quả. Ví dụ khoá
"việc X + các việc con của X" để kiểm "X chưa có con": lượt kia thêm con cho X và commit trong lúc mình chờ → câu khoá
không trả con đó, kiểm cho qua sai.

**Luật**: sau câu khoá, điều kiện nào phụ thuộc vào TẬP dòng (có con không, đếm, tồn tại) thì hỏi lại bằng một câu MỚI
(snapshot mới) — an toàn khi mọi lượt làm thay đổi tập đó đều phải khoá cùng dòng "cha" mình đang giữ. Hoặc dùng SERIALIZABLE
kèm retry. Test race nên ÉP thứ tự (giữ khoá, cho lượt kia commit trong lúc lượt cần kiểm đang chờ) chứ không để ngẫu nhiên.

## Test: transaction tay + `SaveChanges` trên context EF bật retry strategy → ném lỗi, test treo (2026-10-13)

Context cấu hình `EnableRetryOnFailure` không cho `BeginTransaction` tay rồi `SaveChanges` (InvalidOperationException).
Trong test chen khoá, lỗi này làm một lượt khác chờ khoá mãi → `await` không bao giờ xong, cả lần chạy treo không báo gì.
Phía "giữ khoá" trong test dùng `NpgsqlConnection` trần + SQL thô; luôn `WaitAsync(timeout)` các task đang chờ và chạy
`dotnet test --blame-hang-timeout`.
