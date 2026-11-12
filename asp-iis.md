# Host ASP.NET Core trên IIS: cấu hình app pool và site

Đọc khi deploy ASP.NET Core lên IIS, viết hoặc sửa script cấu hình IIS (`WebAdministration`, `appcmd`), hoặc khi host trên
IIS bị ngủ, khởi động nguội, chết hẳn sau khi máy chủ khởi động lại, hay chạy chồng hai process.

## Thông số nên đặt

Áp cho app pool của host **cần luôn chạy** (API có job nền, host mà web khác gọi tới). Site web thuần có thể để mặc định.

| Thông số | Mặc định IIS | Nên đặt | Vì sao |
|---|---|---|---|
| `startMode` (app pool) | `OnDemand` | `AlwaysRunning` | Process lên ngay khi app pool khởi động, không đợi request đầu. |
| `processModel.idleTimeout` | 20 phút | `00:00:00` | Không cho ngủ; request đầu sau lúc ngủ phải chờ khởi động nguội, client gọi sang dễ hết hạn chờ. |
| `applicationDefaults.preloadEnabled` (site) | `false` | `true` | Nạp ứng dụng ngay khi app pool khởi động hoặc recycle. Cần tính năng `Web-AppInit` (Application Initialization). |
| `recycling.periodicRestart.time` | 29 giờ | `00:00:00` | Tắt recycle theo chu kỳ (trôi giờ, rơi vào giờ cao điểm). |
| `recycling.periodicRestart.schedule` | trống | một mốc giờ vắng, vd `03:00` | Recycle hằng ngày vào giờ cố định. |
| `processModel.maxProcesses` | 1 | `1` (ghi tường minh) | Không web garden: job nền, cache trong bộ nhớ chỉ có một bản. |
| `recycling.disallowOverlappingRotation` | `false` | `true` | Recycle/deploy thì process cũ tắt hẳn rồi process mới mới lên, job nền không chạy song song hai bản. Đổi lại site ngừng vài giây. |
| `failure.rapidFailProtection` | `true` | `true` | Giữ bật, nhưng nới ngưỡng ở hai dòng dưới. |
| `failure.rapidFailProtectionMaxCrashes` | 5 | `100` | Mặc định: 5 lần sập trong khung đếm là IIS **dừng hẳn app pool**, chỉ deploy lại hoặc Start tay mới chạy. |
| `failure.rapidFailProtectionInterval` | 5 phút | `00:01:00` | Khung đếm ngắn: chỉ sập dồn dập thật mới bị dừng; sập lặp lại lúc chờ phụ thuộc (DB) vẫn được IIS khởi động lại. |
| `processModel.startupTimeLimit` | 90 giây | `00:05:00` | Khởi động chậm (migration, nạp dữ liệu, chờ DB) không bị IIS giết giữa chừng. |

- **Sự cố (2026-11-12)**: máy chủ khởi động lại lúc đêm, SQL Server lên chậm (lỗi `requires encryption but this machine
  does not support it`, rồi `Cannot open database ... Login failed`, mã 4060). Code khởi động truy vấn DB nên ném lỗi,
  process chết, và app pool bị dừng hẳn suốt nhiều giờ, tới lúc deploy lại mới chạy. Bài học: nới Rapid-Fail Protection
  như bảng trên, VÀ trong code chờ DB sẵn sàng (thử `Database.CanConnectAsync` lặp lại có giới hạn thời gian) trước khi
  chạy việc khởi động chạm DB. IIS không có delay tăng dần giữa các lần khởi động lại.
- Máy chủ: dịch vụ SQL Server để `Automatic`; nếu cùng máy, có thể cho `W3SVC` phụ thuộc dịch vụ SQL.

## Mẫu script (PowerShell, quyền Administrator, chạy trên máy chủ)

```powershell
Import-Module WebAdministration
$site = 'MyApp.Api'
$sitePath = "IIS:\Sites\$site"
$pool = Get-ItemProperty $sitePath -Name applicationPool
$poolPath = "IIS:\AppPools\$pool"

if (Get-Command Get-WindowsFeature -ErrorAction SilentlyContinue) {
    if (-not (Get-WindowsFeature Web-AppInit).Installed) { Install-WindowsFeature Web-AppInit | Out-Null }
}

Set-ItemProperty $poolPath -Name startMode -Value 'AlwaysRunning'
Set-ItemProperty $poolPath -Name processModel.idleTimeout -Value ([TimeSpan]::Zero)
Set-ItemProperty $poolPath -Name recycling.periodicRestart.time -Value ([TimeSpan]::Zero)
Clear-ItemProperty $poolPath -Name recycling.periodicRestart.schedule
New-ItemProperty $poolPath -Name recycling.periodicRestart.schedule -Value @{ value = [TimeSpan]'03:00:00' } | Out-Null
Set-ItemProperty $poolPath -Name processModel.maxProcesses -Value 1
Set-ItemProperty $poolPath -Name recycling.disallowOverlappingRotation -Value $true
Set-ItemProperty $poolPath -Name failure.rapidFailProtection -Value $true
Set-ItemProperty $poolPath -Name failure.rapidFailProtectionMaxCrashes -Value 100
Set-ItemProperty $poolPath -Name failure.rapidFailProtectionInterval -Value ([TimeSpan]'00:01:00')
Set-ItemProperty $poolPath -Name processModel.startupTimeLimit -Value ([TimeSpan]'00:05:00')
Set-ItemProperty $sitePath -Name applicationDefaults.preloadEnabled -Value $true
```

- Script nên có `[CmdletBinding(SupportsShouldProcess)]` để chạy thử bằng `-WhatIf`, kiểm quyền Administrator, và in lại
  giá trị thực tế sau khi đặt.
- Trả một site về mặc định thì dùng `Clear-ItemProperty` (bỏ giá trị ghi đè, kế thừa `applicationPoolDefaults`), không
  ghi đè bằng số mặc định. App pool dùng chung với site cần luôn chạy thì đừng reset.
- Đổi xong thì recycle app pool để áp dụng ngay.
