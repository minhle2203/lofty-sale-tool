# [Technical] Apply Discount - Update Order (Single Instance)

> Tài liệu kỹ thuật dành cho developer. Mô tả chi tiết logic, database, function flow, và hướng dẫn khi cần rework/renew tính năng discount.

---

## 1. Tổng quan kiến trúc

### Discount là gì?

Discount là cơ chế giảm giá áp dụng lên các loại phí của khách hàng khi admin thực hiện **Update Order**. Discount được lưu dưới dạng JSON trong DB và được đọc lại mỗi kỳ billing để tính giá.

### Modules liên quan

| Module | Vai trò |
|--------|---------|
| `homethy-sale-admin-web` | Controller nhận request từ admin UI |
| `homethy-payment-business-service` | Core business logic: tính toán, lưu trữ, áp dụng discount |
| `homethy-payment-single-task` | Billing task: đọc discount và tính giá cho từng kỳ thanh toán |
| `homethy-billing-task` | Xử lý billing hàng tháng |

### Flow tổng quát

```
Admin UI → ContractGenerateController.updateOrder()
              ↓
         ContractOrderRequest (chứa discount JSON params)
              ↓
         SimpleContractGenerator.invoke()
              ↓
         Detail Generators (set discountData vào PaymentInfoDetail)
              ↓
         ContractOrderUtils.updatePaymentDetailsPriceWithDiscount()
              ↓
         Lưu DB: payment_info_detail.discount_data
              ↓
         Payment thành công → PaySuccessRunnable
              ↓
         updateDiscountForUpdate() → tính startDate/endDate → update DB
              ↓
         Billing hàng tháng đọc lại discount_data → tính giá
```

---

## 2. Database Schema

### Bảng `payment_info_detail`

| Column | Type | Mô tả |
|--------|------|--------|
| `discount_data` | varchar(255) | JSON chứa thông tin discount (DiscountInfo) |
| `discount_percent` | decimal(10,2) | Backup field cho % discount |
| `listing_price` | decimal(10,2) | Giá gốc trước khi discount |
| `total_fee` | decimal(10,2) | Giá sau discount (giá thực tính) |
| `product_id` | int | Loại sản phẩm (CRM, IDX, ...) |
| `extra_one` | varchar | Chứa seat count (dùng cho CRM discount) |

**Migration SQL** (`sql_change/2023.sql`):
```sql
ALTER TABLE payment_info_detail ADD COLUMN `discount_data` varchar(255);
```

### Bảng `contract_order`

| Column | Type | Mô tả |
|--------|------|--------|
| `platform_fee_percentage_off` | int(2) | % giảm platform fee (áp dụng toàn order) |
| `approved_by` | varchar(64) | Người phê duyệt discount |
| `first_month_billing_setting` | int | Cấu hình billing tháng đầu (ảnh hưởng ngày bắt đầu discount) |

**Migration SQL** (`sql_change/2023.sql`):
```sql
ALTER TABLE contract_order ADD COLUMN `platform_fee_percentage_off` int(2) NOT NULL DEFAULT 0;
ALTER TABLE contract_order ADD COLUMN `approved_by` varchar(64) NOT NULL DEFAULT '';
```

---

## 3. Data Model

### DiscountInfo (lưu trong `discount_data`)

**File:** `homethy-payment-business-service/.../service/contract/model/DiscountInfo.java`

```java
public class DiscountInfo {
    private int discountType;          // 0 = fixed amount, 1 = percentage
    private BigDecimal discountAmount; // Số tiền giảm (khi discountType = 0)
    private String percentageOff;      // % giảm, VD: "10" = 10% (khi discountType = 1)
    private int duration;              // 1-12: số tháng, 99: vĩnh viễn
    private boolean active;            // Discount còn hiệu lực hay không
    private String startDate;          // Ngày bắt đầu, format "yyyyMMdd"
    private String endDate;            // Ngày kết thúc, format "yyyyMMdd"
}
```

**Ví dụ JSON lưu trong DB:**
```json
{
  "discountType": 0,
  "discountAmount": 100.00,
  "percentageOff": null,
  "duration": 12,
  "active": true,
  "startDate": "20250501",
  "endDate": "20260430"
}
```

### ContractOrderRequest (request object)

**File:** `homethy-payment-business-service/.../service/contract/request/ContractOrderRequest.java` (lines 106-125)

```java
private int firstMonthBillingSetting;      // Cấu hình tháng đầu
private int platformFeePercentageOff;      // % giảm platform fee
private String packageFeeDiscountJson;     // Discount cho phí package (IDX)
private String seatFeeDiscountJson;        // Discount cho phí seat (CRM)
private String activationFeeDiscountJson;  // Discount cho phí activation/setup
private String approvedBy;                 // Người duyệt
private int aiBundle;                      // Flag AI Bundle
private String aiBundleDiscountJson;       // Discount cho AI Bundle
```

### ClientType (xác định Single Instance)

**File:** `homethy-payment-business-service/.../model/client/UserInfoAdmin.java`

```java
AGENT_SOLO(1, "Single Instance")    // ← Đây là Single Instance
MULTI_OFFICE(2, "Multi-Team")
TEAM_AGENT(4, "Team Agent")
TEAM_INSTANCE(8, "Team Instance")
```

---

## 4. Endpoint & Controller

### POST `/order/update`

**File:** `homethy-sale-admin-web/.../controller/payment/ContractGenerateController.java` (lines 838-1090)

**Request params liên quan đến discount:**

```java
@RequestParam(value = "platformFeePercentageOff", defaultValue = "0") int platformFeePercentageOff,
@RequestParam(value = "packageFeeDiscountJson", defaultValue = "") String packageFeeDiscountJson,
@RequestParam(value = "seatFeeDiscountJson", defaultValue = "") String seatFeeDiscountJson,
@RequestParam(value = "activationFeeDiscountJson", defaultValue = "") String activationFeeDiscountJson,
@RequestParam(value = "approvedBy", defaultValue = "") String approvedBy,
@RequestParam(value = "aiBundleDiscountJson", defaultValue = "") String aiBundleDiscountJson,
```

**Business rule quan trọng (lines 931-934):**
```java
// Platform fee discount CHỈ áp dụng cho prepaid order
if (seatPrePaid == 0) {
    platformFeePercentageOff = 0;
}
```

**Build request (lines 1057-1072):**
```java
ContractOrderRequest request = ContractOrderRequest.builder()
    .firstMonthBillingSetting(firstMonthBillingSetting)
    .platformFeePercentageOff(platformFeePercentageOff)
    .packageFeeDiscountJson(packageFeeDiscountJson)
    .seatFeeDiscountJson(seatFeeDiscountJson)
    .approvedBy(approvedBy)
    .aiBundleDiscountJson(aiBundleDiscountJson)
    .build();
```

---

## 5. Detail Generators - Gắn discount vào từng loại phí

### SimpleContractGenerator (Orchestrator)

**File:** `homethy-payment-business-service/.../service/contract/generate/SimpleContractGenerator.java` (lines 49-71)

Dựa vào client type để chọn factory tạo generators:

```java
if (userInfoAdmin.isMember() || userInfoAdmin.isTeamAgent()) {
    factory = new MemberUpdateDetailGeneratorFactory();
} else {
    // Single Instance đi vào đây
    factory = new UpdateDetailGeneratorFactory(isChimePrePaid);
}
```

### Các Detail Generators

| Generator | File | Loại phí | Lines |
|-----------|------|----------|-------|
| `MonthlyCrmFeeDetailGenerator` | `.../generate/update/crm/MonthlyCrmFeeDetailGenerator.java` | Seat fee (CRM) | 119-122 |
| `MonthlyIdxDetailGenerator` | `.../generate/update/idx/MonthlyIdxDetailGenerator.java` | Package fee (IDX) | 65-68 |
| `SetupFeeDetailGenerator` | `.../generate/update/setup/SetupFeeDetailGenerator.java` | Activation fee | 34-37, 81-84 |
| `AiBundleAddonHelper` | `.../service/contract/helper/AiBundleAddonHelper.java` | AI Bundle fee | 76-150, 315-325 |

**Pattern chung của tất cả generators:**
```java
String discountJson = request.getXxxDiscountJson();
if (discountJson != null && !"".equals(discountJson)) {
    detail.setDiscountData(discountJson);  // Gắn discount JSON vào PaymentInfoDetail
}
```

---

## 6. Core Discount Calculation

### File chính: `ContractOrderUtils.java`

**Path:** `homethy-payment-business-service/.../utils/ContractOrderUtils.java`

### 6.1 `getDiscountPrice()` (lines 1034-1068) - Tính giá 1 tháng

```java
public static BigDecimal getDiscountPrice(BigDecimal totalPrice,
                                          String discountInfoJson,
                                          PaymentInfoDetail paymentInfoDetail) {
    BigDecimal realPrice = totalPrice;

    if (StringUtils.isNotBlank(discountInfoJson)) {
        DiscountInfo discountInfo = PaymentJacksonUtil.fromJson2Obj(discountInfoJson, DiscountInfo.class);

        if (discountInfo.getDiscountType() > 0) {
            // PERCENTAGE: realPrice = totalPrice * (1 - percentage/100)
            BigDecimal divide = new BigDecimal(discountInfo.getPercentageOff()).divide(BIG_DECIMAL_100);
            realPrice = totalPrice.multiply(BigDecimal.ONE.subtract(divide))
                        .setScale(2, RoundingMode.FLOOR);
        } else {
            // FIXED AMOUNT
            BigDecimal subPrice = discountInfo.getDiscountAmount();

            // Đặc biệt: CRM seat fee → nhân discount với số seat thực tế
            if (paymentInfoDetail.getProductId() == ProductConstant.CHIME_CRM) {
                int seatNum = getRealDiscountSeat(paymentInfoDetail, paymentInfoDetail.getExtraOne());
                subPrice = subPrice.multiply(BigDecimal.valueOf(seatNum));
            }

            realPrice = totalPrice.subtract(subPrice);
            if (realPrice.compareTo(BigDecimal.ZERO) < 0) {
                realPrice = BigDecimal.ZERO;  // Không cho giá âm
            }
        }
    }
    return realPrice.setScale(2, RoundingMode.FLOOR);
}
```

### 6.2 `getRealDiscountSeat()` (lines 1257-1306) - Tính số seat thực tế

Mỗi CRM package có số seat miễn phí đi kèm. Method này trừ đi số seat free:

| Package | Seat miễn phí |
|---------|---------------|
| CRM_ONLY | 1 |
| LOFTY_AGENT_NEW | 2 |
| LOFTY_ESSENTIALS / LOFTY_TEAM | 5 |
| LOFTY_ELITE / LOFTY_BROKER_15 | 15 |
| LOFTY_BROKER_50 | 50 |
| LOFTY_ENTERPRISE | 100 |
| REAL_TEAM_5 | 5 |
| REAL_TEAM_15 | 15 |
| REAL_TEAM_50 | 50 |
| AGENT | 3 |

**Ví dụ:** Khách mua LOFTY_TEAM (5 free seats) + 8 seats tổng → `getRealDiscountSeat()` = 3 seats chịu discount.

### 6.3 `getTotalDiscountPrice()` (lines 1073-1131) - Tính tổng cho prepaid order

```java
public static BigDecimal getTotalDiscountPrice(int prePayMonth,
                                               PaymentInfoDetail paymentInfoDetail,
                                               Date chargeDate, Date nextChargeDate) {
    // Logic:
    // 1. Nếu có endDate:
    //    - endDate < chargeDate → discount hết hạn → dùng giá gốc
    //    - nextChargeDate <= endDate → tất cả tháng đều được discount
    //    - Còn lại: tính số tháng discount = intervalMonth(chargeDate, endDate) + 1
    //
    // 2. Nếu không có endDate (dùng duration):
    //    - discountNum = min(duration, prePayMonth)
    //
    // 3. Tính tổng:
    //    - oldPriceCount = prePayMonth - discountNum
    //    - totalPrice = (giá gốc * oldPriceCount) + (giá discount * discountNum)
}
```

### 6.4 `updatePaymentDetailsPriceWithDiscount()` (lines 1134-1255) - Apply all discounts

Method chính được gọi khi generate order. Áp dụng:
1. Discount riêng từng loại phí (từ `discountData`)
2. Platform fee discount (từ `platformFeePercentageOff`)

```java
// Platform fee discount được áp SAU discount riêng, CHỈ cho CRM/IDX
if (platformFeePercentageOff > 0
    && PaymentConstant.ALL_PLATFORM_TYPES.contains(detail.getProductId())) {
    BigDecimal platformFeeDivide = BigDecimal.valueOf(platformFeePercentageOff)
        .divide(BIG_DECIMAL_100, 2, RoundingMode.FLOOR);
    BigDecimal discountPrice = detail.getTotalFee()
        .multiply(BigDecimal.ONE.subtract(platformFeeDivide));
    detail.setTotalFee(discountPrice.setScale(2, RoundingMode.FLOOR));
}
```

### 6.5 `getDiscountPriceForPay()` (lines 1308-1383) - Cho billing hàng tháng

Tính discount có tính đến proration (chia theo ngày):
- **Monthly:** Apply trực tiếp percentage hoặc fixed amount
- **Prepaid:** Check nếu `nextChargeDate <= discountEndDate` thì apply full, ngược lại prorate theo số ngày còn lại

### 6.6 AI Bundle Discount (logic riêng)

**File:** `AiBundleAddonHelper.java` (lines 315-325)

```java
private BigDecimal applyDiscount(BigDecimal price, DiscountInfo discount) {
    if (discount.getDiscountType() == 0 && discount.getDiscountAmount() != null) {
        return price.subtract(discount.getDiscountAmount());  // Fixed
    }
    if (discount.getDiscountType() == 1 && StringUtils.isNotBlank(discount.getPercentageOff())) {
        BigDecimal pct = new BigDecimal(discount.getPercentageOff().replace("%", ""))
            .divide(BigDecimal.valueOf(100), 10, RoundingMode.HALF_UP);
        return price.multiply(BigDecimal.ONE.subtract(pct));  // Percentage
    }
    return price;
}
```

> **Lưu ý:** AI Bundle dùng `RoundingMode.HALF_UP` (làm tròn lên), trong khi các loại khác dùng `RoundingMode.FLOOR` (làm tròn xuống). Đây là điểm cần chú ý khi rework.

---

## 7. Lifecycle của Discount sau khi lưu

### 7.1 Khi payment thành công → Set startDate/endDate

**File:** `homethy-payment-business-service/.../runnable/PaySuccessRunnable.java`

#### Cho Update Order (lines 1346-1402):

```java
private void updateDiscountForUpdate(...) {
    // So sánh old duration vs new duration
    // Nếu duration thay đổi hoặc chưa có endDate → tính lại endDate

    // CASE 1: Discount đã tồn tại và active
    if (oldDiscountInfo != null && oldDiscountInfo.isActive()) {
        if (oldDuration != duration || StringUtils.isBlank(endDate)) {
            // duration = 99 → endDate = "29991231" (vĩnh viễn)
            // duration khác → endDate = now + duration months (lấy ngày cuối tháng)
        }
    }

    // CASE 2: Discount mới
    else {
        startDate = now;
        endDate = now + duration months;
    }
}
```

#### Cho New Order (lines 1310-1343):

```java
private void updateDiscountForSetup(...) {
    startDate = chargeInfo.getPlatformFirstChargeDate();

    // Nếu firstMonthBillingSetting > 0: đẩy startDate thêm 1 tháng
    if (contractOrder.getFirstMonthBillingSetting() > 0) {
        startDate = startDate + 1 month;
    }

    endDate = startDate + duration months (ngày cuối tháng);
    // duration = 99 → endDate = "29991231"
}
```

### 7.2 Billing hàng tháng → Đọc discount và tính giá

**File:** `homethy-payment-business-service/.../billing/monthly/AfterPayMonthlyBilling.java` (lines 305-308)

```java
String discountDataJson = paymentInfoDetail.getDiscountData();
totalFee = ContractOrderUtils.getDiscountPrice(totalFee, discountDataJson, paymentInfoDetail);
```

**File:** `homethy-payment-single-task/.../service/OrderBillingSubDetailService.java` (lines 318-346)

```java
// Đọc discount
String discountDataJson = paymentInfoDetail.getDiscountData();
totalFee = ContractOrderUtils.getDiscountPrice(totalFee, discountDataJson, paymentInfoDetail);

// Cho prepaid: tính tổng có tính duration
totalFee = ContractOrderUtils.getTotalDiscountPrice(prePayMonth, paymentInfoDetail, chargeDate, nextChargeDate);
```

### 7.3 Discount hết hạn

**File:** `PaySuccessRunnable.java` (lines 1286-1302)

```java
// Sau mỗi lần payment thành công, check expiry
if (endDate.before(new Date())) {
    discountInfo.setActive(false);  // Đánh dấu hết hạn
    paymentInfoDetailService.updateDiscountInfo(id, JacksonUtils.toJson(discountInfo));
}
```

> **Lưu ý:** Hệ thống KHÔNG tự động gửi notification khi discount hết hạn. Expiry được phát hiện passive khi billing tiếp theo chạy.

### 7.4 Duration tracking

Hệ thống **KHÔNG decrement** duration mỗi tháng. Thay vào đó:
- Lúc tạo/update: tính `endDate = startDate + duration months`
- Mỗi kỳ billing: so sánh current date vs endDate để biết còn hiệu lực không

---

## 8. Thứ tự áp dụng Discount

```
Giá gốc (listing_price / totalFee)
    ↓
[1] Apply discount riêng (packageFee / seatFee / activationFee / aiBundle)
    ↓
[2] Apply platformFeePercentageOff (chỉ cho CRM/IDX, chỉ prepaid)
    ↓
Giá cuối cùng (totalFee sau discount)
```

**Quan trọng:** Platform fee discount áp dụng SAU discount riêng → nó giảm trên giá đã được discount rồi (stacking).

---

## 9. So sánh: New Order vs Update Order

| Khía cạnh | New Order | Update Order |
|-----------|-----------|--------------|
| Method xử lý discount dates | `updateDiscountForSetup()` | `updateDiscountForUpdate()` |
| startDate | `chargeInfo.platformFirstChargeDate` | `new Date()` (ngày hiện tại) |
| Ảnh hưởng firstMonthBillingSetting | Có (đẩy start +1 tháng) | Không |
| So sánh old vs new duration | Không | Có |
| Khi nào recalc endDate | Luôn luôn (nếu chưa có) | Chỉ khi duration thay đổi hoặc endDate trống |

---

## 10. Validation & Constraints

| Rule | Chi tiết |
|------|----------|
| Giá không âm | `if (realPrice < 0) realPrice = 0` |
| Lỗi parse JSON | Trả về giá gốc, log error |
| Platform fee discount | Chỉ áp dụng cho prepaid (`seatPrePaid > 0`) |
| Platform fee products | Chỉ CRM và IDX (`PaymentConstant.ALL_PLATFORM_TYPES`) |
| Max discount | Không có hard limit trong code |
| Approval | Chỉ lưu `approvedBy`, không validate role |

---

## 11. Hướng dẫn Rework / Renew

### Nếu cần thêm loại phí mới có discount:

1. Thêm field discount JSON mới vào `ContractOrderRequest.java`
2. Thêm `@RequestParam` tương ứng trong `ContractGenerateController.updateOrder()`
3. Tạo/sửa Detail Generator để set `discountData` cho PaymentInfoDetail mới
4. Nếu phí mới thuộc platform type → thêm vào `PaymentConstant.ALL_PLATFORM_TYPES`

### Nếu cần sửa logic tính discount:

1. Sửa `ContractOrderUtils.getDiscountPrice()` cho monthly
2. Sửa `ContractOrderUtils.getTotalDiscountPrice()` cho prepaid
3. Sửa `ContractOrderUtils.getDiscountPriceForPay()` cho billing có proration
4. **Chú ý:** AI Bundle có logic riêng trong `AiBundleAddonHelper.applyDiscount()` → cần sửa riêng

### Nếu cần thêm discount type mới (VD: tiered discount):

1. Thêm giá trị mới cho `discountType` trong `DiscountInfo.java`
2. Thêm nhánh xử lý trong `getDiscountPrice()` và `applyDiscount()`
3. Update cả `getDiscountPriceForPay()` và `getTotalDiscountPrice()`

### Nếu cần thêm validation/limit:

1. Thêm validation trong `ContractGenerateController.updateOrder()` (trước khi build request)
2. Hoặc thêm trong Detail Generator (trước khi set discountData)

### Nếu cần thêm notification khi discount hết hạn:

1. Thêm logic vào `PaySuccessRunnable` sau block check expiry (lines 1286-1302)
2. Hoặc tạo scheduled task riêng scan `payment_info_detail` có `discount_data.active = true` và `endDate < now`

---

## 12. File Reference

| File | Path đầy đủ |
|------|-------------|
| Controller | `homethy-sale-admin-web/src/main/java/com/homethy/sale/admin/controller/payment/ContractGenerateController.java` |
| DiscountInfo | `homethy-payment-business-service/src/main/java/com/homethy/payment/business/service/contract/model/DiscountInfo.java` |
| ContractOrderRequest | `homethy-payment-business-service/src/main/java/com/homethy/payment/business/service/contract/request/ContractOrderRequest.java` |
| ContractOrderUtils | `homethy-payment-business-service/src/main/java/com/homethy/payment/business/utils/ContractOrderUtils.java` |
| SimpleContractGenerator | `homethy-payment-business-service/src/main/java/com/homethy/payment/business/service/contract/generate/SimpleContractGenerator.java` |
| MonthlyCrmFeeDetailGenerator | `homethy-payment-business-service/src/main/java/com/homethy/payment/business/service/contract/generate/update/crm/MonthlyCrmFeeDetailGenerator.java` |
| MonthlyIdxDetailGenerator | `homethy-payment-business-service/src/main/java/com/homethy/payment/business/service/contract/generate/update/idx/MonthlyIdxDetailGenerator.java` |
| SetupFeeDetailGenerator | `homethy-payment-business-service/src/main/java/com/homethy/payment/business/service/contract/generate/update/setup/SetupFeeDetailGenerator.java` |
| AiBundleAddonHelper | `homethy-payment-business-service/src/main/java/com/homethy/payment/business/service/contract/helper/AiBundleAddonHelper.java` |
| PaySuccessRunnable | `homethy-payment-business-service/src/main/java/com/homethy/payment/business/runnable/PaySuccessRunnable.java` |
| AfterPayMonthlyBilling | `homethy-payment-business-service/src/main/java/com/homethy/payment/business/service/payment/billing/monthly/AfterPayMonthlyBilling.java` |
| OrderBillingSubDetailService | `homethy-payment-single-task/src/main/java/com/homethy/payment/single/task/service/OrderBillingSubDetailService.java` |
| PaymentInfoDetail | `homethy-payment-business-service/src/main/java/com/homethy/payment/business/model/contract/PaymentInfoDetail.java` |
| PaymentInfoDetailService | `homethy-payment-business-service/src/main/java/com/homethy/payment/business/service/contract/impl/PaymentInfoDetailServiceImpl.java` |
| UserInfoAdmin | `homethy-payment-business-service/src/main/java/com/homethy/payment/business/model/client/UserInfoAdmin.java` |
| ProductConstant | `homethy-payment-business-service/src/main/java/com/homethy/payment/business/constant/product/ProductConstant.java` |
| SQL Migrations | `sql_change/2023.sql` |
