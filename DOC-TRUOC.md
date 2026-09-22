# GITAmathV20nexus — đọc trước khi dùng

Gói này là **toàn bộ mã nguồn** của hệ thống. Không có `node_modules`, không có
tệp `.exe` đã dựng, không có dữ liệu chạy thật — ba thứ ấy đều dựng lại được từ
đây, và để chúng vào thì gói nặng gấp trăm lần mà không thêm thông tin nào.

## Chạy thử ngay trong ba mươi giây

Cần **Node.js 22 trở lên** (https://nodejs.org). Không cần cài gì thêm —
hệ thống này **không phụ thuộc một gói ngoài nào**.

```bash
cd nexus
node bin/gita.js              # xem danh sách lệnh
npm start                     # bật máy chủ tại http://localhost:9999
```

Mở trình duyệt vào `http://localhost:9999/ui/cong.html`.

## Tự kiểm tra hệ thống

```bash
npm test          # 876 mục kiểm
npm run verify    # 147 mục kiểm định, soi cả những chỗ mục kiểm thường không soi
npm run smoke     # 343 lượt gọi API trên máy chủ đang chạy thật
node bin/gita.js to-thanh-tra --ca-ngay    # tổ thanh tra soi mười lượt một ngày
```

Cả bốn phải xanh. Một mục đỏ nghĩa là có thứ hỏng thật, không phải nhiễu.

## Dựng bản cài Windows 64-bit

Trên chính máy Windows, cần Node.js 22:

```cmd
npm --prefix desktop install
npm run dist:win
```

Xong, hai tệp nằm ở `desktop\dist\`:

- `GITAmath-Nexus-V20-20.0.0-x64.exe` — trình cài đặt (72 MB)
- `GITAmath-Nexus-V20-20.0.0-portable.exe` — chạy thẳng, không cài (72 MB)

Bộ dựng tự soi máy trước khi chạy và tự kiểm tệp sau khi dựng. Hướng dẫn cài
đặt đầy đủ ở `desktop/CAI-DAT-WINDOWS.md`.

Dựng từ máy Linux hoặc macOS thì cần thêm `wine` và `wine32` — bộ dựng sẽ nói
rõ thiếu gì kèm lệnh cài.

## Ba tài liệu chính

| Tệp | Nội dung |
|---|---|
| `README.md` | Toàn bộ hệ thống: kiến trúc, từng mảng, API, cách mở rộng |
| `RUNBOOK.md` | Sổ tay vận hành: chạy hằng ngày, sự cố, khôi phục, ngưỡng cảnh báo |
| `desktop/CAI-DAT-WINDOWS.md` | Cài trên Windows, và cách xử lý khi trục trặc |

## Cấu hình

Chép `.env.example` thành `.env` rồi sửa. Không có tệp `.env` thì hệ thống vẫn
chạy đủ tính năng bằng bộ giả lập mô hình ngôn ngữ **tất định** — cùng một câu
hỏi luôn cho cùng một câu trả lời, nên mọi mục kiểm lặp lại được.

Muốn dùng mô hình ngôn ngữ thật thì điền một trong các khoá `GROQ_API_KEY`,
`GOOGLE_API_KEY`, `OPENROUTER_API_KEY`, `CEREBRAS_API_KEY`. Hệ thống tự nhận
và tự chuyển sang dùng.

## Dữ liệu nằm ở đâu

Chạy bằng dòng lệnh thì dữ liệu vào `nexus/data/`. Chạy bằng ứng dụng máy tính
thì vào thư mục hồ sơ người dùng của Windows — **không nằm trong thư mục cài
đặt**, nên gỡ ra cài lại không mất gì.

## Một điều nên biết trước

Hệ thống này **từ chối bịa số**. Sổ cái dòng tiền không ghi khoản chưa nối
nguồn; công cụ CRM nói "chưa có" khi kho rỗng thay vì đắp một con số cho đỡ
trống; bảng điểm chỉ chấm trên chỉ số đo được và in ra số chỉ số chưa đo được.

Nếu thấy một chỗ trả về "chưa có" hay "chưa nối nguồn", đó là hệ thống đang
nói thật chứ không phải đang hỏng. Cách sửa là nạp dữ liệu thật vào, không
phải sửa mã để nó trả ra một con số.
