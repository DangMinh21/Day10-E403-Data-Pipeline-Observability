# Runbook — Lab Day 10 (incident tối giản)

---

## Symptom

> User / agent thấy gì? (VD: trả lời “14 ngày” thay vì 7 ngày)
Agent trả lời sai thông tin chính sách (ví dụ: "hoàn tiền trong 14 ngày" thay vì 7 ngày), hoặc không trả lời được do dữ liệu thiếu. User thấy "không tìm thấy thông tin" hoặc thông tin lỗi thời.
---

## Detection

> Metric nào báo? (freshness, expectation fail, eval `hits_forbidden`)

- Freshness FAIL: `latest_exported_at` > 124 giờ so với `run_timestamp` trong manifest.
- Expectation fail: Halt trên E3 (refund_no_stale_14d_window) hoặc E7 (chunk_length).
- Eval: `hits_forbidden=yes` trong `artifacts/eval/*.csv`, hoặc `top1_doc_expected=no`.

---

## Diagnosis

| Bước | Việc làm | Kết quả mong đợi |
|------|----------|------------------|
| 1 | Kiểm tra `artifacts/manifests/*.json` | Xem `freshness_status`, `quarantine_records` > 0 báo lỗi data quality. |
| 2 | Mở `artifacts/quarantine/*.csv` | Xem lý do quarantine (ví dụ: "invalid_date", "empty_doc_id"). |
| 3 | Chạy `python eval_retrieval.py` | So sánh `before_after_eval.csv` vs `after_inject_bad.csv` để xác định degradation. |

---

## Mitigation

> Rerun pipeline, rollback embed, tạm banner “data stale”, …

- **Rerun Pipeline:** Chạy `python etl_pipeline.py run --run-id fix_<incident>` để reprocess data sạch.
- **Manual Clear:** Xóa `chroma_db/` và rerun để reset embedding nếu upsert fail.
- **Maintenance Mode:** Cấu hình agent trả lời fallback: "Dữ liệu đang cập nhật, vui lòng thử lại sau."

---

## Prevention

> Thêm expectation, alert, owner — nối sang Day 11 nếu có guardrail.

- **Tighten Expectations:** Thêm E9 kiểm tra version conflict (HR 2025 vs 2026).
- **Auto-Alert:** Bot Slack alert khi quarantine > 10 records.
- **Guardrail (Day 11):** Output filter chặn response nếu confidence < 0.8 từ retrieval.
