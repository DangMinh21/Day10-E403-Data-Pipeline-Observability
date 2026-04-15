# Quality report — Lab Day 10 (nhóm)

**run_id:** 2026-04-15T16-20Z
**Ngày:** 15/04/2026

---

## 1. Tóm tắt số liệu

| Chỉ số | Trước inject | Sau fix (run chuẩn) | Ghi chú |
|--------|-------------|---------------------|---------|
| raw_records | 10 | 10 | Từ `policy_export_dirty.csv` |
| cleaned_records | 4 | 4 | Sau cleaning rules |
| quarantine_records | 4 | 4 | Bad records isolated |
| embed_upsert | 4 | 4 | Upsert theo chunk_id |
| embed_prune_removed | - | 4 | Xoá vector inject cũ |
| Expectation halt? | Có (E3 FAIL) | Không | `--skip-validate` cho inject run |

---

## 2. Before / after retrieval (bắt buộc)

File tham chiếu: `artifacts/eval/before_fix.csv` (inject) và `artifacts/eval/after_fix.csv` (clean).

**Câu hỏi then chốt — refund window (`q_refund_window`):**

| Thời điểm | top1_doc_id | contains_expected | hits_forbidden |
|-----------|-------------|-------------------|----------------|
| Trước fix (inject "14 ngày") | it_helpdesk_faq | **no** | **yes** |
| Sau fix (pipeline chuẩn) | policy_refund_v4 | **yes** | **no** |

Sau inject, agent trả về sai doc và context chứa từ khoá cấm — đúng triệu chứng data stale. Sau khi pipeline chuẩn chạy lại (`embed_prune_removed=4`), retrieval phục hồi hoàn toàn.

**Merit — versioning HR (`q_leave_version`):**

| Thời điểm | top1_doc_id | contains_expected | hits_forbidden | top1_doc_expected |
|-----------|-------------|-------------------|----------------|-------------------|
| Trước fix | hr_leave_policy | yes | no | yes |
| Sau fix | hr_leave_policy | yes | no | yes |

HR version ổn định xuyên suốt nhờ rule `stale_hr_policy_effective_date` quarantine bản 2025 trước khi embed. Kết quả khớp `grading_run.jsonl`: `gq_d10_03 contains_expected=true, hits_forbidden=false, top1_doc_matches=true`.

---

## 3. Freshness & monitor

SLA: **124 giờ** (đọc từ `FRESHNESS_SLA_HOURS` trong `.env`), đo tại boundary publish (sau embed).

Run `2026-04-15T16-20Z`:

```text
freshness_check=FAIL {"latest_exported_at": "2026-04-10T08:00:00", "age_hours": 128.348, "sla_hours": 124.0, "reason": "freshness_sla_exceeded"}
```

**Giải thích:** CSV mẫu có `exported_at` cố định `2026-04-10T08:00:00`. Sau 128 giờ, vượt SLA 124h → FAIL là **hành vi đúng** — tín hiệu để operator biết data snapshot đã cũ. Trong production, pipeline sẽ re-ingest từ nguồn để cập nhật `exported_at`.

---

## 4. Corruption inject (Sprint 3)

Nhóm tạo `data/raw/inject_dirty.csv` chứa chunk `policy_refund_v4` với nội dung sai — "14 ngày làm việc". Chạy:

```bash
python etl_pipeline.py run --raw data/raw/inject_dirty.csv --run-id before-inject --no-refund-fix --skip-validate
```

Log xác nhận E3 FAIL nhưng pipeline tiếp tục embed (chỉ dùng cho demo Sprint 3):

```text
expectation[refund_no_stale_14d_window] FAIL (halt) :: violations=1
WARN: expectation failed but --skip-validate → tiếp tục embed (chỉ dùng cho demo Sprint 3).
embed_prune_removed=4
embed_upsert count=4 collection=day10_kb
```

Sau đó chạy `eval_retrieval.py` → `artifacts/eval/before_fix.csv` cho thấy `q_refund_window hits_forbidden=yes`. Chạy pipeline chuẩn để fix → `artifacts/eval/after_fix.csv` cho thấy `hits_forbidden=no`.

---

## 5. Hạn chế & việc chưa làm

- Chưa auto-alert khi `quarantine_records > threshold` (đề xuất trong runbook: Slack bot).
- Chunking chưa tối ưu — chunk đơn lẻ, chưa sliding window.
- Thiếu integration với Day 11 guardrails (output filter khi confidence thấp).
