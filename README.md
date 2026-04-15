# Lab Day 10 — Data Pipeline & Data Observability

**Môn:** AI in Action (AICB-P1)
**Nhóm:** Nhóm 9 - E403
**Thành viên:**

| Tên | Vai trò | MSSV |
|-----|---------|------|
| Nguyễn Quang Tùng | Ingestion / Raw Owner | 2A202600197 |
| Đặng Văn Minh | Cleaning & Quality Owner | 2A202600027 |
| Nguyễn Thị Quỳnh Trang | Embed & Idempotency Owner | 2A202600406 |
| Đồng Văn Thịnh | Monitoring / Docs Owner | 2A202600365 |

**Repo:** <https://github.com/DangMinh21/Day10-E403-Data-Pipeline-Observability>

---

## Giới thiệu

Pipeline ETL với Data Observability cho bộ corpus CS + IT Helpdesk (tiếp nối Day 09 Multi-agent). Pipeline đảm bảo dữ liệu được làm sạch, validate, và embed vào ChromaDB theo cách **idempotent** — agent chỉ nhận context đúng version, không bị nhiễu bởi data stale.

**Vấn đề giải quyết:** CSV export raw từ hệ nguồn có duplicate, ngày sai định dạng, doc_id không hợp lệ, xung đột version HR (10 vs 12 ngày phép), chunk sai cửa sổ hoàn tiền (14 vs 7 ngày). Pipeline phát hiện, cô lập (quarantine), và sửa tự động — có log và manifest truy vết theo `run_id`.

---

## Cấu trúc thư mục

```
Day10-E403-Data-Pipeline-Observability/
├── etl_pipeline.py               # Entrypoint chính: ingest → clean → validate → embed
├── eval_retrieval.py             # Eval retrieval before/after (xuất CSV)
├── grading_run.py                # Chạy grading JSONL (3 câu gq_d10_01..03)
├── instructor_quick_check.py     # GV kiểm tra nhanh artifact
│
├── transform/
│   └── cleaning_rules.py         # 6 rule baseline + 3 rule mới (Member 2)
├── quality/
│   └── expectations.py           # 6 expectation baseline + E7, E8 mới (Member 3)
├── monitoring/
│   └── freshness_check.py        # Kiểm tra SLA freshness từ manifest
│
├── contracts/
│   └── data_contract.yaml        # Owner, SLA 124h, allowed_doc_ids, schema cleaned
│
├── data/
│   ├── docs/                     # 5 tài liệu policy (policy_refund_v4, sla_p1_2026, ...)
│   ├── raw/
│   │   ├── policy_export_dirty.csv   # Raw export mẫu (có lỗi cố ý)
│   │   └── inject_dirty.csv          # File inject Sprint 3 (chunk 14 ngày sai)
│   ├── test_questions.json           # 4 câu golden retrieval
│   └── grading_questions.json        # 3 câu chấm điểm
│
├── artifacts/
│   ├── logs/                     # run_*.log — có run_id, raw/cleaned/quarantine_records
│   ├── manifests/                # manifest_*.json — metadata + freshness_check
│   ├── quarantine/               # quarantine_*.csv — record bị loại + lý do
│   ├── cleaned/                  # cleaned_*.csv — data sau cleaning
│   └── eval/
│       ├── before_fix.csv        # Eval sau inject (retrieval tệ — hits_forbidden=yes)
│       ├── after_fix.csv         # Eval sau pipeline chuẩn (retrieval tốt)
│       ├── before_after_eval.csv # Eval chuẩn
│       └── grading_run.jsonl     # Kết quả 3 câu grading
│
├── docs/
│   ├── pipeline_architecture.md  # Sơ đồ luồng + ranh giới ingest/clean/embed
│   ├── data_contract.md          # Source map, schema, quarantine rules
│   ├── runbook.md                # Symptom → Detection → Diagnosis → Mitigation → Prevention
│   └── quality_report_template.md  # Quality report với số liệu run 2026-04-15T16-20Z
│
├── reports/
│   ├── group_report.md           # Báo cáo nhóm đầy đủ
│   └── individual/               # Báo cáo cá nhân 4 thành viên
│
├── .env                          # FRESHNESS_SLA_HOURS=124, CHROMA_COLLECTION=day10_kb
├── requirements.txt
└── SCORING.md
```

---

## Setup

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env             # FRESHNESS_SLA_HOURS=124 đã được cấu hình
```

> SentenceTransformers tải model `all-MiniLM-L6-v2` (~90MB) lần đầu — cần mạng.

---

## Cách chạy

### Pipeline chuẩn (một lệnh)

```bash
python etl_pipeline.py run
```

Log đầu ra mẫu:

```text
run_id=2026-04-15T16-20Z
raw_records=10
cleaned_records=4
quarantine_records=4
expectation[refund_no_stale_14d_window] OK (halt) :: violations=0
expectation[hr_leave_no_stale_10d_annual] OK (halt) :: violations=0
expectation[chunk_text_length_10_5000] OK (halt) :: out_of_range_chunks=0
expectation[metadata_completeness_doc_id_exported_at] OK (halt) :: incomplete_metadata_count=0
embed_prune_removed=4
embed_upsert count=4 collection=day10_kb
freshness_check=FAIL {...}
PIPELINE_OK
```

> `freshness_check=FAIL` là hành vi đúng — CSV mẫu có `exported_at` cố định từ 10/04, sau 124h sẽ vượt SLA. Trong production, pipeline re-ingest từ nguồn để cập nhật timestamp.

### Kiểm tra freshness

```bash
python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_2026-04-15T16-20Z.json
```

### Eval retrieval

```bash
python eval_retrieval.py --out artifacts/eval/after_fix.csv
```

### Sprint 3 — Inject dữ liệu xấu và so sánh before/after

```bash
# Bước 1: Embed dữ liệu xấu (chunk "14 ngày" sai policy)
python etl_pipeline.py run --raw data/raw/inject_dirty.csv --run-id before-inject --no-refund-fix --skip-validate

# Bước 2: Eval — phải thấy q_refund_window hits_forbidden=yes
python eval_retrieval.py --out artifacts/eval/before_fix.csv

# Bước 3: Pipeline chuẩn để fix
python etl_pipeline.py run

# Bước 4: Eval sau fix — phải thấy hits_forbidden=no
python eval_retrieval.py --out artifacts/eval/after_fix.csv
```

### Grading

```bash
python grading_run.py --out artifacts/eval/grading_run.jsonl
```

---

## Hướng dẫn chấm bài (GV)

### Kiểm tra nhanh artifact

```bash
python instructor_quick_check.py --grading artifacts/eval/grading_run.jsonl
python instructor_quick_check.py --manifest artifacts/manifests/manifest_2026-04-15T16-20Z.json
```

### Kết quả grading JSONL (`artifacts/eval/grading_run.jsonl`)

| Câu | contains_expected | hits_forbidden | top1_doc_matches | Điểm |
|-----|-------------------|----------------|------------------|------|
| gq_d10_01 | true | false | — | 4/4 |
| gq_d10_02 | true | false | — | 3/3 |
| gq_d10_03 | true | false | true | 3/3 |

**Hạng: Merit** — `gq_d10_03` đạt đủ `contains_expected=true, hits_forbidden=false, top1_doc_matches=true`.

### Bằng chứng Sprint 3 (before/after)

| File | Câu q_refund_window | Ghi chú |
|------|---------------------|---------|
| `artifacts/eval/before_fix.csv` | `contains_expected=no, hits_forbidden=yes` | Sau inject chunk "14 ngày" |
| `artifacts/eval/after_fix.csv` | `contains_expected=yes, hits_forbidden=no` | Sau pipeline chuẩn fix |

### Cleaning rules mới (`transform/cleaning_rules.py`)

| Rule | Hàm | Tác động |
|------|-----|----------|
| Loại URL/thông tin nhạy cảm | `_contains_sensitive_info` | quarantine chunk chứa `https?://` |
| Validate exported_at (2024–2027) | `_validate_exported_at` | quarantine nếu sai format hoặc ngoài range |
| Deduplicate (doc_id, ngày) | `_deduplicate_doc_id` | loại bản ghi trùng trong batch |

### Expectations mới (`quality/expectations.py`)

| ID | Tên | Severity |
|----|-----|----------|
| E7 | chunk_text_length_10_5000 | HALT |
| E8 | metadata_completeness_doc_id_exported_at | HALT |

---

## Tài liệu tham khảo

- [docs/pipeline_architecture.md](docs/pipeline_architecture.md) — Sơ đồ luồng
- [docs/data_contract.md](docs/data_contract.md) — Source map và schema
- [docs/runbook.md](docs/runbook.md) — Incident runbook
- [docs/quality_report_template.md](docs/quality_report_template.md) — Quality report Sprint 3
- [reports/group_report.md](reports/group_report.md) — Báo cáo nhóm
- [SCORING.md](SCORING.md) — Rubric chấm điểm
