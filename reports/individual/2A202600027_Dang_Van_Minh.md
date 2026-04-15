# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Đặng Văn Minh  
**Mã số học viên:** 2A202600027  
**Vai trò:** Thiết kế quy tắc làm sạch  
**Ngày nộp:** 15/04/2026  
**Độ dài yêu cầu:** 400–650 từ

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

Tôi chịu trách nhiệm thiết kế và triển khai các quy tắc làm sạch dữ liệu trong pipeline. Cụ thể, tôi đã thêm ba quy tắc mới vào file `transform/cleaning_rules.py`, bao gồm: loại bỏ BOM/encoding không hợp lệ, kiểm tra ngày hợp lệ (trong khoảng 2024–2027), và loại bỏ các chunk chứa URL hoặc thông tin nhạy cảm. Các quy tắc này nhằm đảm bảo dữ liệu đầu vào sạch và tuân thủ các yêu cầu SLA đã định nghĩa trong `data_contract.yaml`. Tôi cũng kiểm tra hiệu quả của các quy tắc bằng cách chạy pipeline và ghi lại số liệu trước/sau.

---

## 2. Quy tắc làm sạch mới (120–150 từ)

### Quy tắc 1: Loại bỏ BOM/encoding không hợp lệ
- **Mục đích:** Đảm bảo dữ liệu không chứa ký tự BOM hoặc encoding gây lỗi.
- **Kết quả:** 2 bản ghi bị loại bỏ do chứa BOM.

### Quy tắc 2: Kiểm tra ngày hợp lệ
- **Mục đích:** Loại bỏ các bản ghi có ngày không nằm trong khoảng 2024–2027.
- **Kết quả:** 3 bản ghi bị loại bỏ do ngày không hợp lệ.

### Quy tắc 3: Loại bỏ chunk chứa URL/thông tin nhạy cảm
- **Mục đích:** Đảm bảo dữ liệu không chứa thông tin nhạy cảm như URL hoặc API key.
- **Kết quả:** 1 bản ghi bị loại bỏ do chứa URL.

---

## 3. Bằng chứng trước/sau (80–120 từ)

**Run ID: rule_test**
```
raw_records=15
cleaned_records=9
quarantine_records=6
expectation[valid_date_range] OK (halt) :: invalid_date_count=3
expectation[no_sensitive_info] OK (halt) :: sensitive_info_count=1
expectation[no_bom_encoding] OK (halt) :: bom_count=2
```

**Nhận xét:** Sau khi áp dụng các quy tắc làm sạch, số lượng bản ghi sạch giảm từ 15 xuống còn 9. Các bản ghi không hợp lệ đã được quarantine, đảm bảo dữ liệu đầu ra đáp ứng yêu cầu chất lượng.

---

## 4. Đóng góp vào nhóm (50–80 từ)

Tôi đã phối hợp với các thành viên khác để đảm bảo các quy tắc làm sạch tích hợp tốt với kỳ vọng (expectations) và pipeline tổng thể. Tôi cũng cập nhật bảng `metric_impact` trong báo cáo nhóm để phản ánh hiệu quả của các quy tắc mới. Ngoài ra, tôi đã tạo pull request và nhận phản hồi từ nhóm để cải thiện code.

---

## 5. Hạn chế & việc chưa làm (50–80 từ)

Một số hạn chế bao gồm việc các quy tắc làm sạch hiện tại chưa xử lý được tất cả các trường hợp dữ liệu phức tạp, ví dụ như các bản ghi có lỗi định dạng phức tạp. Trong tương lai, tôi dự định mở rộng các quy tắc để xử lý tốt hơn các trường hợp này và bổ sung thêm kiểm tra tự động để giảm thiểu lỗi.