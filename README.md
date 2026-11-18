# BÀI TẬP CHUYÊN SÂU: KỸ THUẬT PROMPTING NÂNG CAO (LOGIC & CHUỖI)

---

## PHẦN 1: THIẾT KẾ GIẢI PHÁP PROMPTING ỨNG DỤNG THỰC TẾ

### 1. Bối cảnh & Vấn đề thực tế
* **Bối cảnh:** Cửa hàng bán lẻ nhận lượng lớn tin nhắn đặt hàng qua mạng xã hội với nội dung tự nhiên, viết tắt, thiếu thông tin hoặc sai định dạng.
* **Vấn đề:** Cần tự động hóa quy trình đọc hiểu tin nhắn bằng **Chain-of-Thought (CoT)** và **Prompt Chaining**, đồng thời xây dựng cơ chế **xử lý đứt gãy (Error Handling)** khi dữ liệu đầu vào không đạt chuẩn trước khi đẩy đơn vào hệ thống.

### 2. Thiết kế giải pháp (Input / Output)
* **Input (Đầu vào):** Một chuỗi văn bản thô (tin nhắn của khách hàng) chứa các thông tin như tên, số điện thoại, sản phẩm, số lượng và địa chỉ.
* **Output (Đầu ra):** Dữ liệu chuẩn hóa định dạng **JSON** gồm 4 trường cố định:
  * `phan_loai`: Đặt hàng / Khiếu nại / Hỏi thông tin.
  * `trang_thai`: `HOP_LE` hoặc `DUT_GAY_DU_LIEU`.
  * `du_lieu`: Chi tiết từng trường thông tin đã được làm sạch và kiểm tra.
  * `hanh_dong_tiep_theo`: Lệnh đẩy đơn sang kho hoặc tin nhắn mẫu tự động xin bổ sung thông tin.

### 3. Thiết kế Prompt hoàn chỉnh (Tích hợp bẫy dữ liệu)

```markdown
# VAI TRÒ (ROLE)
Bạn là Chuyên viên Tự động hóa Vận hành Đơn hàng của cửa hàng bán lẻ. Nhiệm vụ của bạn là phân tích tin nhắn khách hàng, kiểm tra lỗi dữ liệu và xuất kết quả chuẩn hóa.

# QUY TRÌNH XỬ LÝ CHUỖI & SUY LUẬN (PROMPT CHAINING & CHAIN-OF-THOUGHT)
Hãy thực hiện tuần tự qua 3 bước suy luận trước khi đưa ra kết quả cuối cùng:

* **Chuỗi 1 - Phân loại ý định:** Đọc tin nhắn và xác định mục đích chính của khách (Đặt hàng, Khiếu nại, hay Hỏi đáp).
* **Chuỗi 2 - Trích xuất thực thể:** Lọc ra các trường bắt buộc gồm: [Tên khách], [Số điện thoại], [Sản phẩm], [Số lượng], [Địa chỉ].
* **Chuỗi 3 - Kiểm tra ràng buộc & Xử lý đứt gãy (Error Handling):**
  * *Trường hợp dữ liệu trống:* Nếu thiếu bất kỳ trường bắt buộc nào, gán giá trị trường đó là `"MISSING"`. Tuyệt đối không tự ý suy đoán thông tin.
  * *Trường hợp dữ liệu sai lệch:* Số điện thoại phải đủ 10 chữ số và bắt đầu bằng số 0. Số lượng phải là số nguyên dương lớn hơn 0. Nếu sai, gán trạng thái `"INVALID"`.
  * *Xử lý đứt gãy:* Nếu có bất kỳ trường nào bị `"MISSING"` hoặc `"INVALID"`, dừng quy trình lên đơn và soạn một tin nhắn lịch sự yêu cầu khách bổ sung đúng phần còn thiếu.

# ĐỊNH DẠNG ĐẦU RA (OUTPUT FORMAT)
Chỉ trả về kết quả theo cấu trúc JSON dưới đây, không giải thích thêm:
{
  "suy_luan_tung_buoc": "Tóm tắt ngắn gọn quá trình kiểm tra 3 chuỗi",
  "phan_loai": "Đặt hàng | Khiếu nại | Khác",
  "trang_thai": "HOP_LE | DUT_GAY_DU_LIEU",
  "du_lieu": {
    "ten_khach": "...",
    "so_dien_thoai": "...",
    "san_pham": "...",
    "so_luong": 0,
    "dia_chi": "..."
  },
  "hanh_dong_tiep_theo": "Đẩy đơn sang kho | Tin nhắn phản hồi khách"
}

# DỮ LIỆU ĐẦU VÀO (INPUT)
"Shop ơi cho chị 2 áo thun trắng size M nhé, giao về 123 Nguyễn Trãi, Q1. SĐT chị là 0912345"
```

### 4. Kết quả chạy thử từ AI (Tình huống bẫy dữ liệu)

```json
{
  "suy_luan_tung_buoc": "Chuỗi 1: Khách muốn đặt hàng. Chuỗi 2: Trích xuất được 2 áo thun trắng size M, địa chỉ 123 Nguyễn Trãi Q1, SĐT 0912345, thiếu tên. Chuỗi 3: Phát hiện thiếu tên khách (MISSING) và SĐT không đủ 10 số (INVALID) -> Kích hoạt luồng xử lý đứt gãy.",
  "phan_loai": "Đặt hàng",
  "trang_thai": "DUT_GAY_DU_LIEU",
  "du_lieu": {
    "ten_khach": "MISSING",
    "so_dien_thoai": "INVALID (0912345)",
    "san_pham": "Áo thun trắng size M",
    "so_luong": 2,
    "dia_chi": "123 Nguyễn Trãi, Q1"
  },
  "hanh_dong_tiep_theo": "Dạ shop đã nhận thông tin đặt 2 áo thun trắng size M ạ. Chị vui lòng bổ sung giúp shop Tên người nhận và kiểm tra lại Số điện thoại (hiện đang thiếu số) để shop lên đơn ngay nhé!"
}
```

---

## PHẦN 2: PHÂN TÍCH SO SÁNH TRADE-OFF CÁC GIẢI PHÁP RA LỆNH

### 1. Đề xuất 2 Prompt mẫu

#### Giải pháp 1: Prompt Đơn giản, Ngắn gọn (Zero-shot / Ít ràng buộc)
Tập trung vào sự ngắn gọn, giao tiếp tự nhiên, không ép kiểu dữ liệu đầu ra.

```markdown
Bạn là nhân viên xử lý đơn hàng. Hãy đọc tin nhắn của khách hàng dưới đây, suy nghĩ từng bước để lọc ra Tên, SĐT, Sản phẩm, Số lượng và Địa chỉ. Nếu thiếu hoặc sai thông tin nào thì báo lại để hỏi khách, nếu đủ thì xác nhận lên đơn.

Tin nhắn khách: "Cho anh 1 áo khoác gió đen size L qua 45 Lê Lợi nha, số anh 0909123"
```

#### Giải pháp 2: Prompt Chi tiết, Nhiều ràng buộc (Structured CoT & Error Handling)
Chia nhỏ quy trình thành các chuỗi rõ ràng, đặt bẫy kiểm tra định dạng dữ liệu và bắt buộc xuất JSON.

```markdown
# VAI TRÒ
Bạn là Hệ thống Xử lý Đơn hàng Tự động của cửa hàng bán lẻ.

# QUY TRÌNH XỬ LÝ CHUỖI (CHAIN-OF-THOUGHT & CHAINING)
Thực hiện lần lượt 3 bước suy luận:
1. Trích xuất thực thể: Tên khách, SĐT, Sản phẩm, Số lượng, Địa chỉ.
2. Kiểm tra bẫy dữ liệu:
   - Trường nào không có trong tin nhắn -> Gán giá trị "MISSING". Không tự suy đoán.
   - SĐT phải đủ 10 số, bắt đầu bằng số 0. Số lượng phải > 0. Nếu sai -> Gán "INVALID".
3. Xử lý đứt gãy: Nếu xuất hiện "MISSING" hoặc "INVALID", dừng lên đơn và viết câu phản hồi xin bổ sung đúng trường bị lỗi.

# ĐỊNH DẠNG ĐẦU RA (BẮT BUỘC JSON)
{
  "suy_luan": "Tóm tắt quá trình kiểm tra",
  "trang_thai": "HOP_LE | DUT_GAY",
  "du_lieu": { "ten": "", "sdt": "", "san_pham": "", "so_luong": 0, "dia_chi": "" },
  "phan_hoi_khach": "..."
}

# INPUT
"Cho anh 1 áo khoác gió đen size L qua 45 Lê Lợi nha, số anh 0909123"
```

### 2. Bảng so sánh Trade-off

| Tiêu chí | Giải pháp 1 (Đơn giản) | Giải pháp 2 (Chi tiết) |
| :--- | :--- | :--- |
| **Độ dễ hiểu** | **Cao** (Viết tự nhiên như giao tiếp bình thường, dễ sửa đổi nhanh). | **Trung bình** (Cấu trúc dài, cần tư duy logic rõ ràng để thiết lập). |
| **Tốn token** | **Ít** (Tiết kiệm chi phí API và cho tốc độ phản hồi nhanh hơn). | **Nhiều hơn** (Tiêu tốn nhiều token đầu vào cho phần quy tắc và định dạng). |
| **Chặn lỗi biên** | **Trung bình** (AI dễ tự ý suy đoán thông tin thiếu hoặc bỏ sót lỗi SĐT sai độ dài). | **Cực tốt** (Bắt chính xác dữ liệu trống/sai nhờ các điều kiện rẽ nhánh bắt buộc). |
| **Tính tự động hóa** | **Thấp** (Đầu ra dạng văn bản tự do, khó tích hợp trực tiếp vào phần mềm quản lý). | **Cao** (Đầu ra chuẩn JSON giúp hệ thống tự động đẩy đơn hoặc gửi tin nhắn). |

### 3. Lựa chọn phương án tối ưu & Kết quả chạy thử

* **Phương án lựa chọn:** **Giải pháp 2 (Prompt Chi tiết, Nhiều ràng buộc)**.
* **Lý do chọn:** Trong bài toán tự động hóa vận hành thực tế của cửa hàng, **độ chính xác của dữ liệu** và **khả năng xử lý đứt gãy** quan trọng hơn việc tiết kiệm một lượng nhỏ token. Giải pháp 1 dễ khiến AI tự suy đoán thông tin thiếu (hallucination) hoặc trả về văn bản lộn xộn làm gián đoạn luồng tự động hóa. Giải pháp 2 giải quyết triệt để các bẫy dữ liệu và xuất chuẩn JSON để kết nối thẳng với phần mềm quản lý.

#### Kết quả chạy thử với Giải pháp 2:

```json
{
  "suy_luan": "Đã lọc được sản phẩm (áo khoác gió đen size L), số lượng (1), địa chỉ (45 Lê Lợi). Phát hiện thiếu Tên khách và SĐT 0909123 chỉ có 7 chữ số (sai định dạng 10 số). Kích hoạt trạng thái đứt gãy.",
  "trang_thai": "DUT_GAY",
  "du_lieu": {
    "ten": "MISSING",
    "sdt": "INVALID (0909123)",
    "san_pham": "Áo khoác gió đen size L",
    "so_luong": 1,
    "dia_chi": "45 Lê Lợi"
  },
  "phan_hoi_khach": "Dạ shop đã ghi nhận đơn 1 áo khoác gió đen size L giao về 45 Lê Lợi ạ. Anh cho shop xin thêm Tên người nhận và kiểm tra lại Số điện thoại (hiện đang thiếu số) để shop gửi hàng cho anh nhé!"
}
```
