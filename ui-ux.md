# Quy ước UI/UX chung

Áp cho mọi loại giao diện: desktop (WPF, WinForms, Avalonia…), web, mobile có chuột/touchpad.

## Cuộn ngang: Shift + lăn chuột

- Mọi vùng có nội dung rộng hơn khung nhìn (DataGrid, ListView, TreeView, TreeListView, table, log viewer, vùng code…) phải cuộn
  ngang được bằng **Shift + lăn chuột**. Đây là cử chỉ chuẩn trên Windows, Linux và macOS mà người dùng đã quen.
- Không dùng Alt hay Ctrl cho việc này: Ctrl + lăn thường để zoom, còn Alt + lăn mỗi app hiểu một kiểu (vd cuộn nhanh).
- Chỉ nhận cử chỉ khi vùng đó thật sự cuộn ngang được. Vùng không có gì để cuộn ngang thì để sự kiện lan ra ngoài,
  không nuốt mất nó.
- Theo nền tảng:
  - **Trình duyệt**: tự có sẵn với phần tử `overflow-x: auto|scroll`; đừng tự chặn `wheel` làm hỏng nó.
  - **WPF**: `ScrollViewer` KHÔNG có sẵn. Đăng ký một lần cho cả app bằng
    `EventManager.RegisterClassHandler(typeof(DataGrid|ListBox|TreeView), UIElement.PreviewMouseWheelEvent, ...)`.
    Trong handler: kiểm `Keyboard.Modifiers == ModifierKeys.Shift`, tìm `ScrollViewer` trong visual tree,
    `ScrollableWidth > 0` thì `ScrollToHorizontalOffset(...)` rồi `e.Handled = true`.
    Đăng ký trên lớp control thay vì gắn từng control trong XAML, để grid thêm sau cũng tự có.
  - **WinForms / UI tự vẽ**: tự xử lý `MouseWheel` + `Control.ModifierKeys == Keys.Shift`.
- Bánh nghiêng của chuột (tilt wheel) và vuốt ngang trên touchpad gửi `WM_MOUSEHWHEEL`. Nếu framework không tự xử lý
  thì nên hỗ trợ thêm khi rẻ.
