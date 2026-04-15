# Báo Cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** Nhóm 9 - E403  
**Thành viên:**

| Tên                     | Vai trò (Day 10)          | Email                     |
|-------------------------|---------------------------|---------------------------|
| Nguyễn Quang Tùng       | Ingestion / Raw Owner     | quangtungnguyen613@gmail.com |
| Đặng Văn Minh           | Cleaning & Quality Owner  | minhdv0201@gmail.com      |
| Nguyễn Thị Quỳnh Trang  | Embed & Idempotency Owner | quynhtrang1225@gmail.com  |
| Đồng Văn Thịnh          | Monitoring / Docs Owner   | dvttvdthanhan@gmail.com   |

**Ngày nộp:** 15/04/2026  
**Repo:** [GitHub Repository](https://github.com/DangMinh21/Day10-E403-Data-Pipeline-Observability)

---

## 1. Pipeline tổng quan (150–200 từ)

> Nguồn raw là gì (CSV mẫu / export thật)? Chuỗi lệnh chạy end-to-end? `run_id` lấy ở đâu trong log?

**Tóm tắt luồng:**
Pipeline vận hành theo mô hình Automated ETL với Data Observability, đóng vai trò là "lớp gác cổng" kiến thức cho Agent qua 4 giai đoạn:

1. **Ingest:** Trích xuất dữ liệu thô từ file `policy_export_dirty.csv`.

2. **Transform:** Chuẩn hóa định dạng và áp dụng logic nghiệp vụ (fix refund window).

3. **Quality Control:** Kiểm tra các kỳ vọng dữ liệu (expectations). Hệ thống tự động Halt (dừng) nếu phát hiện lỗi nghiêm trọng để bảo vệ Vector DB, hoặc đẩy bản ghi lỗi vào Quarantine.

4. **Embed & Manifest:** Thực hiện upsert dữ liệu sạch vào ChromaDB theo chunk_id để chống trùng lặp (Idempotency). Cuối cùng, tạo file Manifest ghi lại metadata và kiểm tra Freshness SLA.

**Lệnh chạy một dòng (copy từ README thực tế của nhóm):**

```
# Luồng chuẩn: fix stale refund 14→7, expectation pass, embed
python etl_pipeline.py run

# Kiểm tra freshness theo manifest vừa tạo
python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_<run-id>.json
```

---

## 2. Cleaning & expectation (150–200 từ)

### Cleaning rules mới (Member 2 — Đặng Văn Minh)

Baseline có 6 bước xử lý. Member 2 thêm 3 rule mới vào `transform/cleaning_rules.py`:

**Rule mới 1: `_contains_sensitive_info`** — Quarantine chunk chứa URL hoặc thông tin nhạy cảm (regex `https?://|www\.`). Ngăn dữ liệu nội bộ lọt vào vector store.

**Rule mới 2: `_validate_exported_at`** — Kiểm tra `exported_at` đúng ISO 8601 và trong khoảng 2024–2027, đặt severity **halt** vì `exported_at` là nền tảng của freshness monitoring.

**Rule mới 3: `_deduplicate_doc_id`** — Loại record trùng `(doc_id, ngày)` trong cùng batch, giữ lại record đầu tiên.

### Expectations mới (Member 3 — Nguyễn Thị Quỳnh Trang)

**E7: chunk_text_length_10_5000** (HALT) — Chunk phải có độ dài 10–5000 ký tự, phát hiện lỗi chunking.

**E8: metadata_completeness_doc_id_exported_at** (HALT) — `doc_id` và `exported_at` không được rỗng, bảo vệ audit trail.

### 2a. Bảng metric_impact (bắt buộc — chống trivial)

| Rule / Expectation mới | Trước inject (số liệu) | Khi inject lỗi (số liệu) | Chứng cứ |
|------------------------|------------------------|--------------------------|----------|
| `_contains_sensitive_info` | raw=10, quarantine=4 (không có URL) | Inject URL → quarantine tăng thêm 1 | `artifacts/quarantine/quarantine_*.csv` cột `reason=contains_sensitive_info` |
| `_validate_exported_at` | raw=10, cleaned=4 | Inject `exported_at=invalid` → toàn bộ bị quarantine (`run_rule_test`: cleaned=0, quarantine=10, HALT) | `artifacts/logs/run_rule_test.log` |
| `_deduplicate_doc_id` | raw=10, cleaned=4 | Inject duplicate doc_id cùng ngày → cleaned giảm, prune log xuất hiện | `artifacts/logs/run_2026-04-15T16-20Z.log`: `embed_prune_removed=4` |
| E7: `chunk_text_length_10_5000` | cleaned=4, all pass | Inject chunk < 10 ký tự → E7 FAIL halt trước embed | `artifacts/logs/run_member3_error_test.log` |
| E8: `metadata_completeness` | cleaned=4, all pass | Inject `exported_at` rỗng → E8 FAIL halt | `artifacts/logs/run_member3_error_test.log` |

**Toàn bộ expectations (baseline + mở rộng):**

| ID | Tên | Severity |
|----|-----|----------|
| E1 | min_one_row | HALT |
| E2 | no_empty_doc_id | HALT |
| E3 | refund_no_stale_14d_window | HALT |
| E4 | chunk_min_length_8 | WARN |
| E5 | effective_date_iso_yyyy_mm_dd | HALT |
| E6 | hr_leave_no_stale_10d_annual | HALT |
| **E7** | **chunk_text_length_10_5000** | **HALT** |
| **E8** | **metadata_completeness_doc_id_exported_at** | **HALT** |

## 3. Before / after ảnh hưởng retrieval hoặc agent (200–250 từ)

**Kịch bản inject (Sprint 3):**

Nhóm tạo file `data/raw/inject_dirty.csv` chứa chunk `policy_refund_v4` với nội dung sai — "14 ngày làm việc" thay vì 7 ngày. Chạy pipeline với `--no-refund-fix --skip-validate` (run_id `before-inject`) để cố ý embed dữ liệu xấu vào Chroma. Log xác nhận E3 FAIL bị bỏ qua:

```text
expectation[refund_no_stale_14d_window] FAIL (halt) :: violations=1
WARN: expectation failed but --skip-validate → tiếp tục embed (chỉ dùng cho demo Sprint 3).
embed_prune_removed=4
embed_upsert count=4 collection=day10_kb
```

**Bằng chứng retrieval tệ hơn** (`artifacts/eval/before_fix.csv` — sau khi embed dữ liệu xấu):

| question_id | top1_doc_id | contains_expected | hits_forbidden |
|---|---|---|---|
| q_refund_window | it_helpdesk_faq | **no** | **yes** |
| q_leave_version | hr_leave_policy | yes | no |

Agent trả lời sai hoàn toàn câu hoàn tiền — top-1 trả về sai doc, context chứa từ khoá cấm.

**Bằng chứng retrieval tốt hơn sau fix** (`artifacts/eval/after_fix.csv` — pipeline chuẩn, run_id `2026-04-15T16-20Z`):

| question_id | top1_doc_id | contains_expected | hits_forbidden |
|---|---|---|---|
| q_refund_window | **policy_refund_v4** | **yes** | **no** |
| q_leave_version | hr_leave_policy | yes | no |

Retrieval phục hồi hoàn toàn. Log run fix có `embed_prune_removed=4` — Chroma đã xoá 4 vector cũ từ inject và upsert lại 4 vector sạch, đảm bảo index = snapshot publish.

---

## 4. Freshness & monitoring (100–150 từ)

Nhóm chọn SLA **124 giờ** (~5 ngày), đọc từ `FRESHNESS_SLA_HOURS` trong `.env`, đo tại boundary **publish** (sau embed). Ý nghĩa:

- **PASS**: `age_hours < 124` — data đủ tươi, agent có thể phục vụ.
- **FAIL**: `age_hours >= 124` — data stale, runbook hướng dẫn rerun hoặc báo lỗi.

Run mới nhất `2026-04-15T16-20Z`: `latest_exported_at = 2026-04-10T08:00:00`, `age_hours = 128.3 > 124` → `freshness_check = FAIL`. Đây là hành vi **đúng** — CSV mẫu trong lab có `exported_at` cố định từ 10/04, nên sau vài ngày sẽ vượt SLA. Nhóm ghi nhận trong runbook: FAIL là tín hiệu đúng cho data snapshot cũ; trong production, pipeline sẽ re-ingest từ nguồn và cập nhật `exported_at`.

Monitoring bao gồm: `quarantine_records`, expectation fails, `embed_prune_removed`, và `freshness_check` — tất cả được ghi vào `artifacts/logs/` và `artifacts/manifests/` theo từng `run_id`.

---

## 5. Liên hệ Day 09 (50–100 từ)

> Dữ liệu sau embed có phục vụ lại multi-agent Day 09 không? Nếu có, mô tả tích hợp; nếu không, giải thích vì sao tách collection.

Dữ liệu sau embed phục vụ lại multi-agent Day 09. Collection "day10_kb" trong ChromaDB được sử dụng chung, với upsert theo chunk_id đảm bảo idempotency. Agent Day 09 truy vấn collection này để lấy context cho câu trả lời, tích hợp seamless qua API Chroma.

Không tách collection vì dữ liệu Day 10 là extension của Day 09 (thêm docs mới), đảm bảo knowledge base liên tục.

---

## 6. Rủi ro còn lại & việc chưa làm

- Rủi ro: Chunking quá dài (>5000 ký tự) có thể làm retrieval chậm; cần tối ưu embedding model.
- Chưa làm: Auto-alert bot cho quarantine > threshold; production deployment với Docker.
- Học tập: Halt strategy hiệu quả hơn warn; data contract là foundation cho observability.

## Metric Impact Table

| Rule Name                  | Total Rows | Cleaned Rows | Quarantined Rows | Notes                          |
|----------------------------|------------|--------------|------------------|--------------------------------|
| contains_sensitive_info    | 10         | 4            | 1                | Removed rows with sensitive info (e.g., URLs). |
| invalid_exported_at_format | 10         | 4            | 4                | Quarantined rows with invalid or out-of-range exported_at dates. |
| duplicate_doc_id           | 10         | 4            | 0                | No duplicates found in this dataset. |
