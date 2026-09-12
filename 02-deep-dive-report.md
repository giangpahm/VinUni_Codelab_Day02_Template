# Phase 3 — DEEP-DIVE: Xanh SM Intelligent Dispatcher

## 3.1. Current-State Workflow

### Quy trình xử lý sự cố hết pin thực địa hiện tại

Khi tài xế Xanh SM báo sự cố hết pin, điều phối viên phải thực hiện **05 bước thủ công**, với tổng thời gian xử lý trung bình khoảng **15 phút/lượt**.

```text
┌──────────────────┐
│ BƯỚC 1           │
│ Nhận cuộc gọi    │
│ sự cố hết pin    │
├──────────────────┤
│ 👤 Dispatcher    │
│ ⏱ 2 phút         │
│                  │
│ INPUT:           │
│ • Điện thoại     │
│                  │
│ OUTPUT:          │
│ • Log sự cố      │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ BƯỚC 2           │
│ Tra cứu vị trí   │
│ GPS của xe       │
├──────────────────┤
│ 👤 Dispatcher    │
│ ⏱ 2 phút         │
│                  │
│ INPUT:           │
│ • Biển số xe     │
│                  │
│ OUTPUT:          │
│ • Tọa độ GPS     │
└────────┬─────────┘
         │
         ▼
┌────────────────────────┐
│ BƯỚC 3 🔴 BOTTLENECK   │
│ Tra cứu trạm sạc       │
│ VinFast còn trụ trống  │
├────────────────────────┤
│ 👤 Dispatcher          │
│ ⏱ 5 phút               │
│                        │
│ INPUT:                 │
│ • Vị trí GPS           │
│ • Loại xe               │
│                        │
│ OUTPUT:                │
│ • Địa chỉ trạm sạc     │
│ • Trụ sạc phù hợp      │
└───────────┬────────────┘
            │
            ▼
┌────────────────────────┐
│ BƯỚC 4 🔴 BOTTLENECK   │
│ Soạn tin nhắn hướng    │
│ dẫn và gửi tài xế      │
├────────────────────────┤
│ 👤 Dispatcher          │
│ ⏱ 5 phút               │
│                        │
│ INPUT:                 │
│ • Raw data trạm sạc    │
│                        │
│ OUTPUT:                │
│ • SMS / App message    │
└───────────┬────────────┘
            │
            ▼
┌────────────────────────┐
│ BƯỚC 5                  │
│ Gọi xe cứu hộ nếu cần  │
├────────────────────────┤
│ 👤 Dispatcher          │
│ ⏱ 1 phút               │
│                        │
│ Trigger: Pin < 5%      │
└────────────────────────┘
```

### Key Observation

> 🔴 **Hai bottleneck chính là Bước 3 và Bước 4**, chiếm khoảng **10/15 phút** của toàn bộ quy trình.

**Tổng thời gian xử lý thủ công: ~15 phút/lượt.**

---

# 3.2. Problem Statement — 6-Field Framework

| Field                       | Nội dung                                                                                                                                                                                                                                                                                                                                                     |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **1. Actor / Operator**     | Điều phối viên (Dispatcher) thuộc Trung tâm Điều vận Xanh SM.                                                                                                                                                                                                                                                                                                |
| **2. Current Workflow**     | Khi tài xế báo hết pin, Dispatcher phải: (1) tra cứu vị trí xe trên bản đồ nội bộ; (2) mở Dashboard trạm sạc VinFast để tìm trụ sạc còn trống; (3) kiểm tra tính phù hợp với dòng xe; (4) soạn tin nhắn hướng dẫn và gửi cho tài xế; (5) gọi cứu hộ nếu pin dưới 5%. Toàn bộ quy trình gồm **05 bước thủ công**, mất khoảng **15 phút/lượt**.                |
| **3. Bottleneck**           | **Bước 3 & 4**, chiếm khoảng **10 phút**: tra cứu thủ công trụ sạc phù hợp với từng dòng xe (VF5 / VFe34 / VF8) và soạn tin nhắn hướng dẫn đường đi bằng tiếng Việt thân thiện, dễ hiểu.                                                                                                                                                                     |
| **4. Business Impact**      | Trung bình khoảng **80 sự cố pin/ngày tại Hà Nội**, tương đương khoảng **20 giờ công/ngày** của đội điều vận. Thời gian xử lý kéo dài làm tăng thời gian chờ của tài xế, giảm khả năng tiếp tục nhận chuyến và tạo áp lực trong giờ cao điểm.                                                                                                                |
| **5. Success Metrics**      | **Efficiency:** Giảm thời gian xử lý từ **15 phút → <3 phút/lượt**. <br><br> **Quality:** Tỷ lệ đề xuất đúng địa điểm và đúng loại trụ sạc đạt **≥98%**.                                                                                                                                                                                                     |
| **6. Operational Boundary** | AI được phép: (1) truy xuất API định vị xe; (2) truy xuất dữ liệu trạm sạc VinFast theo thời gian thực; (3) xác định trạm phù hợp; (4) tự động tạo **message draft**. <br><br> **Không được phép:** AI tự động gửi tin mà chưa có Dispatcher phê duyệt; không đề xuất trạm không tương thích với xe; không đề xuất tuyến/trạm cách xe **>5 km khi pin <5%**. |

---

# 3.3. Future-State Flow & AI Fit

## AI Fit Decision

### Lựa chọn: **LLM Feature + Rule-based Logic**

Không cần triển khai **Autonomous Agent** ở giai đoạn đầu vì:

* Quy trình có cấu trúc tương đối cố định.
* Dữ liệu đầu vào và đầu ra có thể xác định rõ.
* Quyết định liên quan đến trạm sạc có **rủi ro vận hành cao**.
* Cần duy trì **Human-in-the-loop (HITL)**.
* AI phù hợp nhất với vai trò **tổng hợp dữ liệu + đề xuất + soạn thảo**, thay vì tự động ra quyết định cuối cùng.

---

## Future-State Workflow

```text
┌──────────────────┐
│ BƯỚC 1           │
│ Nhận cuộc gọi    │
│ sự cố hết pin    │
│                  │
│ 👤 Dispatcher    │
└────────┬─────────┘
         │
         ▼
┌──────────────────────────┐
│ BƯỚC 2 🔵 AUTO-PULL      │
│ Tự động lấy dữ liệu      │
│ vị trí + xe + trạm sạc   │
├──────────────────────────┤
│ 🤖 System / API          │
│                          │
│ • GPS xe                 │
│ • Mức pin                │
│ • Loại xe                │
│ • Trạm sạc khả dụng      │
│ • Khoảng cách            │
└───────────┬──────────────┘
            │
            ▼
┌──────────────────────────┐
│ BƯỚC 3 🔵 AI DRAFT       │
│ Tạo phương án xử lý      │
│ và tin nhắn hướng dẫn    │
├──────────────────────────┤
│ 🤖 LLM + Business Rules  │
│                          │
│ OUTPUT:                 │
│ • Trạm sạc đề xuất       │
│ • Khoảng cách             │
│ • Tuyến đường             │
│ • Message draft           │
│                          │
│ ⚠️ [DRAFT_ONLY]          │
└───────────┬──────────────┘
            │
            ▼
┌──────────────────────────┐
│ BƯỚC 4 🟢 HUMAN REVIEW   │
│ Dispatcher kiểm tra      │
│ và click "Duyệt & Gửi"   │
├──────────────────────────┤
│ 👤 Dispatcher            │
│                          │
│ ✓ Kiểm tra trạm          │
│ ✓ Kiểm tra loại xe       │
│ ✓ Kiểm tra khoảng cách   │
│ ✓ Duyệt message          │
└───────────┬──────────────┘
            │
            ▼
       ┌───────────┐
       │ GỬI TÀI XẾ│
       └───────────┘
```

### Fallback Mechanism

```text
                 ┌──────────────────────┐
                 │ AI tạo Draft         │
                 └──────────┬───────────┘
                            │
                    ┌───────▼───────┐
                    │ Draft hợp lệ? │
                    └───────┬───────┘
                       Có    │    Không
                       │     │
                       ▼     ▼
                ┌─────────┐ ┌─────────────────┐
                │ HITL    │ │ FALLBACK        │
                │ Review  │ │ Dispatcher      │
                │ & Send  │ │ xử lý thủ công  │
                └─────────┘ └─────────────────┘
```

**Nguyên tắc:** Nếu AI không đủ dữ liệu, confidence thấp hoặc vi phạm business rule → **không đưa ra đề xuất tự động**, chuyển ngay về quy trình thủ công.

---

# Phase 5 — EVALUATE & DECISION

## AI Readiness Checklist

| Dimension               | Assessment                                                                                                   | Status       |
| ----------------------- | ------------------------------------------------------------------------------------------------------------ | ------------ |
| **Data Readiness**      | GPS và telemetry trạm sạc VinFast đã có khả năng kết nối qua API theo thời gian thực.                        | ✅ READY      |
| **Risk Management**     | Kiểm soát bằng `[DRAFT_ONLY]` và Human-in-the-loop. Dispatcher bắt buộc kiểm tra và phê duyệt trước khi gửi. | ✅ CONTROLLED |
| **Organizational Need** | Trung tâm Điều vận có nhu cầu giảm tải thao tác thủ công, đặc biệt trong giờ cao điểm.                       | ✅ READY      |
| **Fallback**            | Khi AI không tạo được kết quả hợp lệ, Dispatcher quay lại quy trình thủ công hiện tại.                       | ✅ READY      |
| **Measurability**       | Có thể đo trực tiếp thời gian xử lý và độ chính xác của đề xuất trạm sạc.                                    | ✅ READY      |

---

# Final Decision

## 🟢 GO — Bắt đầu xây dựng Prototype

### Justification

Use case này đáp ứng tốt các tiêu chí để triển khai Prototype:

1. **Phạm vi bài toán rõ ràng**
   Quy trình có input, output và business rule tương đối cụ thể.

2. **ROI có thể đo lường trực tiếp**
   Mục tiêu giảm thời gian xử lý từ **15 phút xuống dưới 3 phút/lượt**, tương đương giảm đáng kể thời gian thao tác thủ công của Dispatcher.

3. **AI Fit phù hợp**
   LLM được sử dụng cho các tác vụ có giá trị cao như **tổng hợp dữ liệu, lựa chọn phương án theo rule và tạo message draft**, thay vì trao quyền tự động điều phối.

4. **Rủi ro được kiểm soát**
   Cơ chế `[DRAFT_ONLY]`, Human-in-the-loop và fallback giúp đảm bảo AI không tự ý gửi thông tin hoặc đưa ra quyết định vận hành cuối cùng.

5. **Có khả năng mở rộng**
   Sau khi Prototype đạt KPI, giải pháp có thể mở rộng sang các tình huống khác như sự cố xe, hỗ trợ tài xế, điều phối cứu hộ và tối ưu phân bổ nguồn lực.

### Prototype KPI

| KPI                       | Current State |                 Target |
| ------------------------- | ------------: | ---------------------: |
| Thời gian xử lý/lượt      |      ~15 phút |           **< 3 phút** |
| Độ chính xác đề xuất trạm |        Manual |              **≥ 98%** |
| AI tự động gửi tin        |         Không | **0% — bắt buộc HITL** |
| Fallback khi AI lỗi       |        Manual |               **100%** |
| Human Approval            | Không áp dụng | **100% trước khi gửi** |

**Recommended Prototype Architecture:**

```text
GPS API ───────────┐
                   │
Charging Station API ──→ Business Rules ──→ LLM ──→ [DRAFT_ONLY]
                   │                              │
Vehicle / Battery ─┘                              ▼
                                           Dispatcher Review
                                                  │
                                           ┌──────┴──────┐
                                           │             │
                                        APPROVE       REJECT
                                           │             │
                                           ▼             ▼
                                      Send Driver    Manual Flow
```
