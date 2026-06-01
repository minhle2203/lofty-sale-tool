# Apply Discount - Update Order (Single Instance)

## Tổng quan

Khi admin cập nhật đơn hàng (update order) cho khách hàng loại **Single Instance** (cá nhân, 1 tài khoản), hệ thống cho phép áp dụng **giảm giá** lên các loại phí khác nhau. Discount có thể là **giảm một số tiền cố định** (VD: giảm $100) hoặc **giảm theo %** (VD: giảm 10%).

---

## Các loại phí được áp dụng discount

| Loại phí | Ý nghĩa |
|---|---|
| **Package fee** | Phí gói dịch vụ IDX (website bất động sản) |
| **Seat fee** | Phí chỗ ngồi CRM (phần mềm quản lý khách hàng) |
| **Activation fee** | Phí kích hoạt / phí thiết lập ban đầu |
| **AI Bundle fee** | Phí gói AI đi kèm |
| **Platform fee** | Phí nền tảng (chỉ giảm theo %) |

---

## Cách tính giảm giá

Mỗi loại phí có thể được giảm theo **1 trong 2 cách**:

### Cách 1 — Giảm số tiền cố định

> Ví dụ: Phí gốc $500, giảm $100 → Còn **$400**

### Cách 2 — Giảm theo phần trăm

> Ví dụ: Phí gốc $500, giảm 20% → Còn **$400**

### Trường hợp đặc biệt với Seat fee (CRM)

> Nếu giảm cố định $10/seat và khách có 5 seats → Tổng giảm = $10 × 5 = **$50**

### Quy tắc an toàn

- Giá sau giảm **không bao giờ âm** — nếu discount lớn hơn giá gốc, hệ thống tự đặt về **$0**
- Nếu hệ thống gặp lỗi khi đọc thông tin discount → **không giảm giá**, giữ nguyên giá gốc

---

## Thời hạn áp dụng discount

| Thiết lập | Ý nghĩa |
|---|---|
| **1–12 tháng** | Discount chỉ có hiệu lực trong số tháng đã chọn |
| **Vĩnh viễn (99)** | Discount áp dụng mãi, không hết hạn |
| **Theo ngày kết thúc** | Discount có hiệu lực đến một ngày cụ thể |

---

## Quy trình xử lý từ đầu đến cuối

1. **Admin nhập thông tin** — chọn loại discount, nhập số tiền/%, chọn thời hạn, ghi tên người duyệt
2. **Hệ thống nhận yêu cầu** — đọc thông tin discount cho từng loại phí
3. **Tính toán từng phí** — mỗi loại phí (package, seat, activation, AI bundle) được tính riêng, áp discount tương ứng
4. **Kiểm tra hợp lệ** — đảm bảo giá không âm, xử lý lỗi nếu có
5. **Lưu kết quả** — thông tin discount được lưu kèm đơn hàng để các kỳ thanh toán sau tiếp tục áp dụng đúng thời hạn

---

## Thông tin cần có khi apply discount

- **Loại discount**: cố định hay theo %
- **Giá trị**: số tiền hoặc % giảm
- **Thời hạn**: bao nhiêu tháng, hoặc đến ngày nào
- **Người duyệt** (approvedBy): ai là người phê duyệt discount này
