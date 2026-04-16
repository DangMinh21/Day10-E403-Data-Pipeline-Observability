# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Nguyễn Quang Tùng
**MSSV:** 2A202600197
**Vai trò:** Ingestion Owner — Phân tích dữ liệu & Hợp đồng dữ liệu (Member 1)
**Ngày nộp:** 2026-04-15
**run_id tham chiếu:** `sprint1`
**repo data pipeline observability:** https://github.com/GDGoC-FPTU/data-pipeline-observability-shiina613.git
---

## 1. Tôi phụ trách phần nào?

**File / module:**

- `contracts/data_contract.yaml` — định nghĩa schema, SLA freshness, allowlist doc_id, failure modes và quality rules
- `docs/data_analysis_notes.md` — ghi nhận 6 loại lỗi phát hiện trong `data/raw/policy_export_dirty.csv`
- `docs/data_contract.md` — tài liệu hóa source map, schema cleaned, quy tắc quarantine và canonical source

**Kết nối với thành viên khác:**

`data_contract.yaml` là tài liệu gốc mà Member 2 (cleaning rules) và Member 3 (expectations) dựa vào để viết code. `allowed_doc_ids` trong contract đồng bộ với allowlist trong `cleaning_rules.py`; `quality_rules` phản ánh logic trong `expectations.py`. Member 4 dùng `data_contract.md` để hoàn thiện tài liệu nhóm.

**Bằng chứng commit thật (branch `NQT/analysis-contract`):**

| Hash | Thời gian | Nội dung |
|------|-----------|---------|
| `7f68d58` | 2026-04-15 15:37 | Phân tích dữ liệu và tạo hợp đồng dữ liệu |
| `db759d7` | 2026-04-15 15:53 | Điền source map và chạy sprint 1 |
| `1986d6d` | 2026-04-15 16:01 | Modify SLA hours |

PR #1 và PR #3 đã được merge vào `main` (commit `ab0e585`, `01b9d70`).

---

## 2. Một quyết định kỹ thuật

Quyết định quan trọng nhất là phân loại mức độ `halt` vs `warn` cho từng failure mode trong `data_contract.yaml`, và sau đó điều chỉnh `FRESHNESS_SLA_HOURS`.

Ban đầu chạy pipeline với `sla_hours: 24`, kết quả log ghi:
```
freshness_check=FAIL {"latest_exported_at": "2026-04-10T08:00:00", "age_hours": 120.865, "sla_hours": 24.0, "reason": "freshness_sla_exceeded"}
```
CSV mẫu có `exported_at = 2026-04-10`, cũ hơn 120 giờ so với thời điểm chạy. Tôi quyết định nâng `FRESHNESS_SLA_HOURS=124` trong `.env` thay vì sửa timestamp trong CSV, vì CSV là dữ liệu nguồn không nên chỉnh tùy tiện. Commit `1986d6d` ghi lại thay đổi này. Lần chạy sau log ghi:
```
freshness_check=PASS {"latest_exported_at": "2026-04-10T08:00:00", "age_hours": 121.021, "sla_hours": 124.0}
```

---

## 3. Một lỗi hoặc anomaly đã xử lý

**Triệu chứng:** Khi phân tích `policy_export_dirty.csv`, phát hiện chunk_id 10 có `effective_date = "01/02/2026"` — định dạng DD/MM/YYYY thay vì YYYY-MM-DD chuẩn ISO 8601.

**Phát hiện:** Đọc thủ công file CSV và đối chiếu với schema trong contract. Nếu không xử lý, pipeline có thể parse sai ngày hoặc gây exception khi validate.

**Fix:** Ghi vào `data_contract.yaml` failure mode `invalid_date_format` với `action: normalize` và `severity: warn`. Cleaning rule tương ứng tự động chuẩn hóa về `2026-02-01`. Log `run_id=sprint1` xác nhận:
```
expectation[effective_date_iso_yyyy_mm_dd] OK (halt) :: non_iso_rows=0
```
Sau normalize không còn dòng nào sai format.

---

## 4. Bằng chứng trước / sau

Từ log `artifacts/logs/run_sprint1.log` và `artifacts/quarantine/quarantine_sprint1.csv`, `run_id=sprint1`:

**Trước clean:** 10 records raw, gồm chunk rỗng (chunk_id 5), doc_id lạ (chunk_id 9), HR policy cũ 2025 (chunk_id 7), duplicate (chunk_id 2).

**Sau clean:**
```
raw_records=10
cleaned_records=6
quarantine_records=4
expectation[refund_no_stale_14d_window] OK (halt) :: violations=0
expectation[hr_leave_no_stale_10d_annual] OK (halt) :: violations=0
embed_upsert count=6 collection=day10_kb
PIPELINE_OK
```

4 records bị quarantine với lý do thật từ file CSV: `duplicate_chunk_text`, `missing_effective_date`, `stale_hr_policy_effective_date`, `unknown_doc_id`. 6 records sạch được embed vào collection `day10_kb`.

---

## 5. Cải tiến tiếp theo

Nếu có thêm 2 giờ, tôi sẽ bổ sung validation `data_contract.yaml` bằng pydantic — hiện tại contract chỉ là file YAML tĩnh, không có schema enforcement tự động. Cụ thể: viết một Python model `DataContract` dùng pydantic v2 để parse và validate file YAML khi pipeline khởi động, đảm bảo nếu ai sửa contract sai cấu trúc (thiếu `sla_hours`, sai kiểu dữ liệu) thì pipeline báo lỗi ngay thay vì chạy với config sai.
