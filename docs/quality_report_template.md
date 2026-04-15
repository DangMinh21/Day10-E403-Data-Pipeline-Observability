# Quality report — Lab Day 10 (nhóm)

**run_id:** ci-smoke  
**Ngày:** 15/04/2026

---

## 1. Tóm tắt số liệu

| Chỉ số | Trước | Sau | Ghi chú |
|--------|-------|-----|---------|
| raw_records | 10 | 10 | Từ policy_export_dirty.csv |
| cleaned_records | - | 6 | Sau cleaning rules |
| quarantine_records | - | 4 | Bad records isolated |
| Expectation halt? | No | No | All expectations PASS |

---

## 2. Before / after retrieval (bắt buộc)

> Đính kèm hoặc dẫn link tới `artifacts/eval/before_after_eval.csv` (hoặc 2 file before/after).

**Câu hỏi then chốt:** refund window (`q_refund_window`)  
**Trước:** contains_expected=yes, hits_forbidden=no, top1_doc_id=policy_refund_v4  
**Sau:** contains_expected=yes, hits_forbidden=no, top1_doc_id=policy_refund_v4 (không thay đổi)

**Merit (khuyến nghị):** versioning HR — `q_leave_version` (`contains_expected`, `hits_forbidden`, cột `top1_doc_expected`)

**Trước:** contains_expected=yes, hits_forbidden=no, top1_doc_expected=yes  
**Sau:** contains_expected=yes, hits_forbidden=no, top1_doc_expected=yes (stable)

---

## 3. Freshness & monitor

> Kết quả `freshness_check` (PASS/WARN/FAIL) và giải thích SLA bạn chọn.

Freshness: PASS. SLA 124 giờ, latest_exported_at 2026-04-10T08:00:00, run_timestamp 2026-04-14T19:53:37 → chênh lệch ~4.5 ngày < 124 giờ.

---

## 4. Corruption inject (Sprint 3)

> Mô tả cố ý làm hỏng dữ liệu kiểu gì (duplicate / stale / sai format) và cách phát hiện.

Inject lỗi: thêm chunk rỗng, doc_id invalid, refund "14 ngày" stale, metadata thiếu. Phát hiện qua cleaning rules (quarantine) và expectations (halt nếu fail).

---

## 5. Hạn chế & việc chưa làm

- Chưa auto-alert khi quarantine > threshold.
- Chunking chưa tối ưu cho retrieval speed.
- Thiếu integration với Day 11 guardrails.