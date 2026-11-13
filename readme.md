# Giải Pháp Prompting Tự Động Hóa Chuẩn Hóa Dữ Liệu (CREATE + Few-shot + JSON)

Tài liệu thiết kế giải pháp Prompting ứng dụng **Cấu trúc Prompt Tiêu chuẩn (CREATE)**, kỹ thuật **Few-shot**, **Ép định dạng JSON** và **Điều kiện xử lý lỗi** cho quy trình tiếp nhận thông tin tự động tại cửa hàng **TechShop**.

## Mục tiêu & Bối cảnh

- **Mục tiêu:** Rèn luyện tư duy thiết kế Input/Output (I/O) và lập luận xử lý ngoại lệ trước khi triển khai code tự động hóa.
- **Bối cảnh thực tế:** Bộ phận CSKH/Bảo hành của TechShop hằng ngày nhận hàng trăm tin nhắn thô từ khách hàng qua Zalo/Messenger. Tin nhắn thường thiếu cấu trúc, viết tắt, bỏ sót mã đơn hàng hoặc nhập sai định dạng số điện thoại/ngày tháng.
- **Giải pháp:** Xây dựng Prompt chuẩn hóa bóc tách văn bản tự nhiên thành **JSON** hợp lệ để đẩy thẳng vào CRM/Google Sheets, đồng thời tự động gắn cờ cảnh báo khi dữ liệu bị trống hoặc sai lệch.

---

## Bước 1: Thiết kế giải pháp (Input / Output)

### 1. Input (Đầu vào)
Chuỗi văn bản thô (tin nhắn khách hàng) chứa các thông tin rời rạc như: tên khách hàng, số điện thoại, mã đơn hàng, tên sản phẩm, nội dung khiếu nại/bảo hành và ngày mua hàng (có thể bị thiếu hoặc sai định dạng).

### 2. Output (Đầu ra mong muốn)
01 mảng **JSON** chuẩn RFC 8259 (không kèm văn bản giải thích thừa) gồm các trường cố định:

| Trường dữ liệu | Kiểu dữ liệu | Mô tả & Quy tắc chuẩn hóa |
| :--- | :--- | :--- |
| `khach_hang` | `string \| null` | Tên hoặc danh xưng khách hàng trích xuất từ tin nhắn. |
| `so_dien_thoai` | `string \| null` | Số điện thoại hợp lệ (đủ 10 chữ số Việt Nam); gán `null` nếu thiếu hoặc sai. |
| `ma_don_hang` | `string \| null` | Mã đơn hàng chuẩn `TS-...`; gán `null` nếu không có. |
| `san_pham` | `string \| null` | Tên linh kiện/thiết bị khách hàng đề cập. |
| `nhom_van_de` | `string` | Phân loại cố định: `BAO_HANH`, `GIAO_HANG`, `DOI_TRA`, hoặc `KHAC`. |
| `trang_thai_du_lieu` | `string` | Trạng thái kiểm định: `HOP_LE`, `CAN_BO_SUNG`, hoặc `LOI_DU_LIEU`. |
| `missing_fields` | `array<string>` | Danh sách tên các trường bắt buộc bị bỏ trống. |
| `error_flags` | `array<string>` | Danh sách mô tả chi tiết các lỗi sai lệch định dạng hoặc logic. |

---

## Bước 2: Prompt hoàn chỉnh và Kết quả chạy thử

### 1. Prompt hoàn chỉnh (CREATE + Few-shot + JSON + Error Handling)

```text
[C - CHARACTER]
Bạn là Hệ thống Trích xuất & Chuẩn hóa Dữ liệu Tự động (Data Parser Engine) của chuỗi bán lẻ công nghệ TechShop. Bạn làm việc chính xác tuyệt đối, tuân thủ nghiêm ngặt schema JSON và không bao giờ suy diễn dữ liệu không có trong văn bản gốc.

[R - REQUEST]
Hãy đọc danh sách tin nhắn thô của khách hàng trong phần [INPUT], bóc tách thông tin và chuyển đổi thành một mảng JSON chuẩn để hệ thống CRM tiếp nhận tự động.

[A - ADJUSTMENT & QUY TẮC XỬ LÝ LỖI]
1. Ép định dạng: Chỉ trả về duy nhất 01 khối mã JSON hợp lệ. Không thêm lời chào, không giải thích trước hoặc sau JSON.
2. Quy tắc chống bịa đặt: Thông tin nào không xuất hiện trong tin nhắn gốc thì bắt buộc gán giá trị `null`.
3. Câu lệnh điều kiện xử lý dữ liệu trống & sai lệch:
   - NẾU thiếu bất kỳ trường bắt buộc nào (`khach_hang`, `so_dien_thoai`, `ma_don_hang`, `san_pham`): Gán `null` cho trường đó, thêm tên trường vào mảng `missing_fields`, và đặt `trang_thai_du_lieu = "CAN_BO_SUNG"`.
   - NẾU phát hiện dữ liệu sai lệch (VD: `so_dien_thoai` khác 10 chữ số hoặc chứa chữ cái; `ma_don_hang` sai định dạng; ngày mua vượt quá ngày hiện tại): Giữ nguyên hoặc gán `null` tùy mức độ, mô tả cụ thể lỗi vào mảng `error_flags`, và đặt `trang_thai_du_lieu = "LOI_DU_LIEU"`. (Ưu tiên trạng thái `"LOI_DU_LIEU"` nếu vừa thiếu vừa sai).
   - NẾU đầy đủ và hợp lệ 100%: Để `missing_fields: []`, `error_flags: []`, và `trang_thai_du_lieu = "HOP_LE"`.

[E - EXAMPLES (FEW-SHOT)]
Input mẫu 1: "Em ơi anh Minh sđt 0901234567, đơn TS-1024 mua con RTX 4060 hôm qua nay bật không lên hình, cho anh bảo hành gấp."
Output mẫu 1:
[
  {
    "khach_hang": "Anh Minh",
    "so_dien_thoai": "0901234567",
    "ma_don_hang": "TS-1024",
    "san_pham": "RTX 4060",
    "nhom_van_de": "BAO_HANH",
    "trang_thai_du_lieu": "HOP_LE",
    "missing_fields": [],
    "error_flags": []
  }
]

Input mẫu 2: "Shop ơi chị Lan nè, cái nguồn 750W bị kêu to quá, gọi lại cho chị số 0988123 nha, chị làm mất mã đơn rồi."
Output mẫu 2:
[
  {
    "khach_hang": "Chị Lan",
    "so_dien_thoai": null,
    "ma_don_hang": null,
    "san_pham": "Nguồn 750W",
    "nhom_van_de": "BAO_HANH",
    "trang_thai_du_lieu": "LOI_DU_LIEU",
    "missing_fields": ["ma_don_hang"],
    "error_flags": ["so_dien_thoai khong du 10 chu so: '0988123'"]
  }
]

[T - TYPE OF OUTPUT]
01 mảng JSON chuẩn RFC 8259.

[E - EXTRAS / INPUT CẦN XỬ LÝ]
Ngày hiện tại: 13/11/2026.
Danh sách tin nhắn thô:
1. "Alo TechShop, mình là Tuấn (0912345678), đơn hàng TS-8891 đặt thanh RAM DDR5 32GB bị giao trễ 4 ngày rồi, kiểm tra giúp mình."
2. "Em ơi kiểm tra giúp anh Hùng đơn mua màn hình LG 27 inch bị hở sáng, sđt của anh là 093456789 (nhắn gấp)."
3. "Chào shop, mình tên Vy, sđt 0977654321, mã đơn TS-9012 mua ngày 25/12/2026 bàn phím cơ bị kẹt phím, shop đổi giúp mình."
```

### 2. Kết quả phản hồi của AI (Chạy thử)

```json
[
  {
    "khach_hang": "Tuấn",
    "so_dien_thoai": "0912345678",
    "ma_don_hang": "TS-8891",
    "san_pham": "RAM DDR5 32GB",
    "nhom_van_de": "GIAO_HANG",
    "trang_thai_du_lieu": "HOP_LE",
    "missing_fields": [],
    "error_flags": []
  },
  {
    "khach_hang": "Anh Hùng",
    "so_dien_thoai": null,
    "ma_don_hang": null,
    "san_pham": "Màn hình LG 27 inch",
    "nhom_van_de": "BAO_HANH",
    "trang_thai_du_lieu": "LOI_DU_LIEU",
    "missing_fields": [
      "ma_don_hang"
    ],
    "error_flags": [
      "so_dien_thoai khong du 10 chu so: '093456789'"
    ]
  },
  {
    "khach_hang": "Vy",
    "so_dien_thoai": "0977654321",
    "ma_don_hang": "TS-9012",
    "san_pham": "Bàn phím cơ",
    "nhom_van_de": "DOI_TRA",
    "trang_thai_du_lieu": "LOI_DU_LIEU",
    "missing_fields": [],
    "error_flags": [
      "ngay_mua vuot qua ngay hien tai (13/11/2026): '25/12/2026'"
    ]
  }
]
```

### 3. Đánh giá kiểm thử bẫy dữ liệu

- **Ca 1 (Tuấn):** Dữ liệu đầy đủ và đúng định dạng nên trả về `"HOP_LE"`.
- **Ca 2 (Anh Hùng):** Bị thiếu `ma_don_hang` và `so_dien_thoai` chỉ có 9 chữ số (`093456789`). AI đã không tự ý suy diễn mà gán `null`, đồng thời ghi nhận chính xác vào `missing_fields` và `error_flags`.
- **Ca 3 (Vy):** Có ngày mua `25/12/2026` vượt quá ngày hiện tại (`13/11/2026`), được chặn lại ở trạng thái `"LOI_DU_LIEU"` để nhân viên đối soát trước khi lưu vào cơ sở dữ liệu.
