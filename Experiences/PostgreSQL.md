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
