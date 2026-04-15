# Báo Cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** Nhóm 9 - E403  
**Thành viên:**
| Tên | Vai trò (Day 10) | Email |
|-----|------------------|-------|
| Nguyễn Quang Tùng | Ingestion / Raw Owner | quangtungnguyen613@gmail.com |
| Đặng Văn Minh | Cleaning & Quality Owner | minhdv0201@gmail.com |
| Nguyễn Thị Quỳnh Trang | Embed & Idempotency Owner | quynhtrang1225@gmail.com |
| Đồng Văn Thịnh | Monitoring / Docs Owner | dvttvdthanhan@gmail.com |

**Ngày nộp:** 15/04/2026  
**Repo:** https://github.com/DangMinh21/Day10-E403-Data-Pipeline-Observability  
**Độ dài khuyến nghị:** 600–1000 từ

---

> **Nộp tại:** `reports/group_report.md`  
> **Deadline commit:** xem `SCORING.md` (code/trace sớm; report có thể muộn hơn nếu được phép).  
> Phải có **run_id**, **đường dẫn artifact**, và **bằng chứng before/after** (CSV eval hoặc screenshot).

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

> Baseline đã có nhiều rule (allowlist, ngày ISO, HR stale, refund, dedupe…). Nhóm thêm **≥3 rule mới** + **≥2 expectation mới**. Khai báo expectation nào **halt**.

### 2a. Bảng metric_impact (bắt buộc — chống trivial)

| Rule / Expectation mới (tên ngắn) | Trước (số liệu) | Sau / khi inject (số liệu) | Chứng cứ (log / CSV / commit) |
|-----------------------------------|------------------|-----------------------------|-------------------------------|
| … | … | … | … |

**Rule chính (baseline + mở rộng):**

- …

**Ví dụ 1 lần expectation fail (nếu có) và cách xử lý:**

_________________

---

## 3. Before / after ảnh hưởng retrieval hoặc agent (200–250 từ)

> Bắt buộc: inject corruption (Sprint 3) — mô tả + dẫn `artifacts/eval/…` hoặc log.

**Kịch bản inject:**

_________________

**Kết quả định lượng (từ CSV / bảng):**

_________________

---

## 4. Freshness & monitoring (100–150 từ)

> SLA bạn chọn, ý nghĩa PASS/WARN/FAIL trên manifest mẫu.

_________________

---

## 5. Liên hệ Day 09 (50–100 từ)

> Dữ liệu sau embed có phục vụ lại multi-agent Day 09 không? Nếu có, mô tả tích hợp; nếu không, giải thích vì sao tách collection.

_________________

---

## 6. Rủi ro còn lại & việc chưa làm

- …
