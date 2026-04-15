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

Baseline đã có 6 expectations (E1-E6). Member 3 (Nguyễn Thị Quỳnh Trang) thêm 2 kỳ vọng mới:

**E7: Độ dài chunk_text (10-5000 ký tự)** - HALT  
Đảm bảo chunks không quá ngắn (< 10) hoặc quá dài (> 5000), phát hiện vấn đề chunking.

**E8: Tính đầy đủ metadata (doc_id, exported_at)** - HALT  
Kiểm tra mỗi record có doc_id và exported_at không rỗng, bảo vệ traceability và freshness check.

### 2a. Bảng metric_impact (bắt buộc — chống trivial)

| Rule / Expectation mới (tên ngắn) | Trước (số liệu) | Sau / khi inject (số liệu) | Chứng cứ (log / CSV / commit) |
|-----------------------------------|------------------|-----------------------------|-------------------------------|
| E7: chunk_text_length_10_5000 | raw=10, cleaned=4 | inject errors: quarantine=5, cleaned=4 | artifacts/cleaned/*, artifacts/quarantine/* |
| E8: metadata_completeness | raw=10, cleaned=4 | inject errors: quarantine=5, cleaned=4 | artifacts/logs/run_*.log |
| **Cleaning rules baseline** | Allowlist (4 doc_ids), ISO format, HR stale, refund fix, dedupe | Loại bỏ doc_id lạ, invalid date, rỗng text, duplicates | quality/expectations.py, transform/cleaning_rules.py |

**Expectations chính (baseline + mở rộng):**

- E1: min_one_row (HALT)
- E2: no_empty_doc_id (HALT)
- E3: refund_no_stale_14d_window (HALT)
- E4: chunk_min_length_8 (WARN)
- E5: effective_date_iso_yyyy_mm_dd (HALT)
- E6: hr_leave_no_stale_10d_annual (HALT)
- **E7: chunk_text_length_10_5000 (HALT)** 
- **E8: metadata_completeness_doc_id_exported_at (HALT)**

**Ví dụ expectation behavior:**

Run member3_baseline: raw_records=10 → cleaned=4, quarantine=4. Tất cả expectations PASS vì cleaning rules loại bad data trước. Run member3_error_test: thêm errors (doc_id rỗng, chunk quá dài/ngắn, exported_at rỗng) → và đều được quarantine, E7 & E8 vẫn PASS vì data có vấn đề không qua được cleaning.

## 3. Before / after ảnh hưởng retrieval hoặc agent (200–250 từ)

> Bắt buộc: inject corruption (Sprint 3) — mô tả + dẫn `artifacts/eval/…` hoặc log.

**Kịch bản inject:**

Trong Sprint 3, nhóm đã inject các lỗi vào dữ liệu thô để kiểm tra độ robust của pipeline. Các lỗi bao gồm: thêm chunk rỗng, thay đổi doc_id không hợp lệ, inject nội dung sai (ví dụ: refund window 14 ngày thay vì 7), và thêm metadata thiếu. File inject được lưu tại `data/raw/policy_export_dirty.csv` với run_id `inject-bad`.

**Kết quả định lượng (từ CSV / bảng):**

Từ `artifacts/eval/before_after_eval.csv` và `artifacts/eval/after_inject_bad.csv`, pipeline đã duy trì chất lượng retrieval cao:

- **Trước inject:** 4/4 câu hỏi có `contains_expected=yes`, `hits_forbidden=no`, `top1_doc_expected=yes` cho 1/4 câu hỏi.
- **Sau inject:** Giữ nguyên 4/4 `contains_expected=yes`, `hits_forbidden=no`, nhờ cleaning rules và expectations loại bỏ bad data vào quarantine.

Tuy nhiên, trong run `inject-bad` (manifest_inject-bad.json), cleaned_records giảm từ 6 xuống 4, quarantine tăng lên 6, chứng minh pipeline phát hiện và cô lập lỗi hiệu quả. Điều này đảm bảo agent không nhận thông tin sai lệch như "14 ngày hoàn tiền" thay vì 7 ngày.

---

## 4. Freshness & monitoring (100–150 từ)

> SLA bạn chọn, ý nghĩa PASS/WARN/FAIL trên manifest mẫu.

Nhóm chọn SLA 124 giờ (5 ngày rưỡi) cho freshness, đo tại "publish" (sau embed). Nếu `latest_exported_at` của batch cũ hơn 124 giờ so với `run_timestamp`, freshness = FAIL.

Trong manifest `ci-smoke.json`: run_timestamp = 2026-04-14T19:53:37, latest_exported_at = 2026-04-10T08:00:00. Chênh lệch ~4.5 ngày (<124 giờ) → PASS.

Monitoring bao gồm: số quarantine_records, expectation fails, và freshness alerts gửi đến "team-data-alerts". Logs tại `artifacts/logs/` ghi lại chi tiết.

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
