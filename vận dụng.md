


# Giải pháp Prompting: Trích xuất đơn hàng từ tin nhắn khách (MartX)

**Bài toán:** Khách nhắn đơn qua Zalo/Facebook bằng văn bản tự do. Nhân viên phải đọc và gõ lại thủ công, dễ sót và sai. Giải pháp là dùng AI chuyển tin nhắn thành JSON để nhập thẳng vào hệ thống.

## Bước 1. Thiết kế giải pháp (I/O)

**Input**
| Thông tin | Mô tả |
|---|---|
| `tin_nhan` | Tin nhắn đặt hàng của khách (văn bản tự do, có thể viết tắt, thiếu dấu) |

**Output mong muốn (JSON, không kèm lời giải thích)**
| Trường | Kiểu | Quy tắc |
|---|---|---|
| `ten_khach` | string / null | Tên khách |
| `sdt` | string / null | Đúng 10 số, bắt đầu bằng 0, sai thì `null` |
| `dia_chi` | string / null | Không có thì `null` |
| `san_pham` | mảng `{ten, so_luong, don_vi}` | Số lượng không rõ thì `null` |
| `thoi_gian_giao` | string / null | Nếu khách có nói |
| `trang_thai` | `"hop_le"` / `"thieu_thong_tin"` | `thieu_thong_tin` nếu thiếu sđt, địa chỉ hoặc sản phẩm |
| `truong_thieu` | mảng string | Liệt kê trường bị trống/sai |
| `canh_bao` | mảng string | Giải thích dữ liệu sai lệch |

**Bẫy cần xử lý:** thiếu SĐT/địa chỉ, SĐT sai độ dài, số lượng không rõ, tin nhắn không phải đơn hàng.

## Bước 2. Prompt hoàn chỉnh (CREATE + Few-shot + ép JSON)

```text
# VAI TRÒ (Character)
Bạn là nhân viên nhập liệu của siêu thị MartX, làm việc cẩn thận và không bao giờ đoán thông tin.

# NHIỆM VỤ (Request)
Đọc tin nhắn đặt hàng của khách và trích xuất thành JSON theo đúng cấu trúc bên dưới.

# QUY TẮC XỬ LÝ LỖI (Adjustments)
1. Chỉ lấy thông tin có trong tin nhắn. TUYỆT ĐỐI không tự bịa hay suy đoán.
2. Nếu một trường không có thông tin, đặt giá trị là null.
3. Nếu sdt không đủ 10 chữ số hoặc không bắt đầu bằng 0, đặt sdt là null và ghi lý do vào canh_bao.
4. Nếu thiếu sdt, dia_chi hoặc san_pham thì trang_thai = "thieu_thong_tin" và liệt kê tên trường thiếu vào truong_thieu. Ngược lại trang_thai = "hop_le" và truong_thieu = [].
5. Nếu số lượng không rõ ràng, so_luong = null và ghi vào canh_bao.
6. Nếu tin nhắn không phải đơn hàng, trả về: {"loi": "Tin nhắn không phải đơn đặt hàng"}

# ĐỊNH DẠNG ĐẦU RA (Type of output)
Chỉ trả về một JSON hợp lệ, không thêm lời giải thích, không dùng markdown.
Cấu trúc:
{"ten_khach":string|null,"sdt":string|null,"dia_chi":string|null,
 "san_pham":[{"ten":string,"so_luong":number|null,"don_vi":string|null}],
 "thoi_gian_giao":string|null,"trang_thai":string,
 "truong_thieu":[string],"canh_bao":[string]}

# VÍ DỤ (Examples - Few-shot)
Tin nhắn: "Anh Nam 0912345678, giao 3kg thịt heo và 1 chục trứng tới 12 Lê Lợi Q1 trước 5h chiều"
Kết quả: {"ten_khach":"Nam","sdt":"0912345678","dia_chi":"12 Lê Lợi, Q1","san_pham":[{"ten":"thịt heo","so_luong":3,"don_vi":"kg"},{"ten":"trứng","so_luong":10,"don_vi":"quả"}],"thoi_gian_giao":"trước 5h chiều","trang_thai":"hop_le","truong_thieu":[],"canh_bao":[]}

Tin nhắn: "Cho mình ít rau với cá nhé"
Kết quả: {"ten_khach":null,"sdt":null,"dia_chi":null,"san_pham":[{"ten":"rau","so_luong":null,"don_vi":null},{"ten":"cá","so_luong":null,"don_vi":null}],"thoi_gian_giao":null,"trang_thai":"thieu_thong_tin","truong_thieu":["ten_khach","sdt","dia_chi"],"canh_bao":["Số lượng rau không rõ","Số lượng cá không rõ"]}

# DỮ LIỆU CẦN XỬ LÝ (Extras)
Tin nhắn: "chị Lan đây, lấy giúp chị 2 hộp sữa tươi với 1kg cam, giao chiều nay nha, sdt 09123456"
```

## Kết quả AI phản hồi (chạy thử)

```json
{
  "ten_khach": "Lan",
  "sdt": null,
  "dia_chi": null,
  "san_pham": [
    {"ten": "sữa tươi", "so_luong": 2, "don_vi": "hộp"},
    {"ten": "cam", "so_luong": 1, "don_vi": "kg"}
  ],
  "thoi_gian_giao": "chiều nay",
  "trang_thai": "thieu_thong_tin",
  "truong_thieu": ["sdt", "dia_chi"],
  "canh_bao": ["SĐT '09123456' không hợp lệ (cần 10 chữ số)"]
}
```


