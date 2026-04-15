# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Đặng Văn Minh  
**Mã số học viên:** 2A202600027  
**Vai trò:** Cleaning & Quality Owner  
**Ngày nộp:** 15/04/2026  
**Độ dài yêu cầu:** 400–650 từ

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

Tôi phụ trách thiết kế và triển khai các quy tắc làm sạch dữ liệu trong file `transform/cleaning_rules.py`. So với baseline nhận được (6 bước: allowlist doc_id, chuẩn hoá ngày, quarantine HR cũ, quarantine text rỗng, dedupe text, fix refund), tôi đã thêm 3 rule mới: (1) `_contains_sensitive_info` — phát hiện và quarantine chunk chứa URL hoặc thông tin nhạy cảm; (2) `_validate_exported_at` — kiểm tra trường `exported_at` đúng định dạng ISO và nằm trong khoảng 2024–2027; (3) `_deduplicate_doc_id` — loại bỏ record trùng `(doc_id, ngày)` để tránh nhiễu trong collection.

**File phụ trách:** `transform/cleaning_rules.py` — hàm `clean_rows`, `_contains_sensitive_info`, `_validate_exported_at`, `_deduplicate_doc_id`.

**Kết nối với thành viên khác:** Rule của tôi là input cho Expectations (Nguyễn Thị Quỳnh Trang — E7, E8) và ảnh hưởng trực tiếp đến `quarantine_records` trong manifest mà Đồng Văn Thịnh dùng để viết runbook.

---

## 2. Một quyết định kỹ thuật (100–150 từ)

Khi thiết kế rule `_validate_exported_at`, tôi phải quyết định đặt severity là **warn** hay **halt**.

Lập luận ban đầu là dùng `warn`: trường `exported_at` không ảnh hưởng trực tiếp đến nội dung chunk, nên pipeline có thể chạy tiếp. Tuy nhiên, tôi nhận ra `exported_at` là trường duy nhất dùng để đo **freshness** trong manifest — nếu giá trị sai hoặc nằm ngoài khoảng hợp lệ (ví dụ `exported_at = 2010-01-01`), `freshness_check` sẽ luôn báo FAIL sai, làm mất tín hiệu monitoring.

Tôi quyết định đặt `_validate_exported_at` là **halt**: nếu `exported_at` không hợp lệ, record bị quarantine và không được embed. Điều này đảm bảo mọi vector trong Chroma đều có metadata truy vết được, phù hợp với tinh thần observability. Quyết định này cũng đồng bộ với Expectation E8 (`metadata_completeness`) do thành viên Quỳnh Trang bổ sung sau đó.

---

## 3. Một lỗi đã xử lý (100–150 từ)

**Triệu chứng:** Trong lần chạy đầu tiên với `run_id=rule_test`, pipeline báo `PIPELINE_HALT` với log:

```text
cleaned_records=0
quarantine_records=10
expectation[min_one_row] FAIL (halt) :: cleaned_rows=0
PIPELINE_HALT: expectation suite failed (halt).
```

Toàn bộ 10 record bị quarantine, không có gì qua được cleaning — ngược với kỳ vọng.

**Phát hiện:** Tôi mở `artifacts/quarantine/quarantine_rule_test.csv` và thấy tất cả record có `reason=invalid_exported_at_format`. Kiểm tra lại hàm `_validate_exported_at` trong `cleaning_rules.py`, tôi phát hiện regex pattern bị escape sai: dùng `r"^\\d{4}-..."` thay vì `r"^\d{4}-..."`, khiến mọi chuỗi ngày hợp lệ đều bị reject.

**Fix:** Sửa regex pattern. Chạy lại `run_id=rule_test`:

```text
cleaned_records=4
quarantine_records=4
expectation[min_one_row] OK (halt) :: cleaned_rows=4
PIPELINE_OK
```

Pipeline hoạt động đúng — 4 record hợp lệ vào cleaned, 4 record lỗi vào quarantine.

---

## 4. Bằng chứng trước / sau (80–120 từ)

Sau khi fix cleaning rules, tôi tạo file inject `data/raw/inject_dirty.csv` (chứa chunk "14 ngày làm việc" sai policy) và chạy eval so sánh.

**Trước khi fix** (`artifacts/eval/before_fix.csv` — run `before-inject`, embed dữ liệu bẩn):

```text
q_refund_window | top1_doc_id=it_helpdesk_faq | contains_expected=no | hits_forbidden=yes
```

Agent trả lời sai — top-1 trả về doc IT helpdesk thay vì `policy_refund_v4`, context chứa từ khoá cấm.

**Sau khi fix** (`artifacts/eval/after_fix.csv` — pipeline chuẩn, cleaning đúng):

```text
q_refund_window | top1_doc_id=policy_refund_v4 | contains_expected=yes | hits_forbidden=no
```

Retrieval trả về đúng document và đúng nội dung "7 ngày làm việc", khớp với `grading_run.jsonl` (`gq_d10_01: contains_expected=true, hits_forbidden=false`).

---

## 5. Cải tiến tiếp theo (40–80 từ)

Nếu có thêm 2 giờ, tôi sẽ đọc `hr_leave_min_effective_date` trực tiếp từ `contracts/data_contract.yaml` thay vì hard-code `"2026-01-01"` trong `cleaning_rules.py`. Hiện tại nếu policy HR cập nhật cutoff date, phải sửa code và redeploy. Đọc từ contract cho phép thay đổi qua config, đúng tinh thần Distinction (d) trong SCORING và đảm bảo cleaning rules luôn đồng bộ với contract.
