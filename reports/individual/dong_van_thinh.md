# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Đồng Văn Thịnh  
**Vai trò:** Monitoring / Docs Owner  
**Ngày nộp:** 15/04/2026
**Độ dài yêu cầu:** 400–650 từ

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

**File / module:**

- `docs/pipeline_architecture.md` — Viết sơ đồ kiến trúc Mermaid, bảng owner table, mô tả idempotency và liên hệ Day 09.
- `docs/runbook.md` — Tạo runbook incident response với 5 phần: symptom, detection, diagnosis, mitigation, prevention.
- `reports/group_report.md` — Tổng hợp báo cáo nhóm (600–1000 từ), bao gồm pipeline overview, cleaning/expectations, before/after eval, freshness, và kết luận.

**Kết nối với thành viên khác:**

Member 1 cung cấp data_contract.yaml và analysis notes. Member 2 xây dựng cleaning rules baseline. Member 3 thêm expectations. Tôi tổng hợp tất cả thành tài liệu và báo cáo, đảm bảo consistency và dẫn chứng artifacts. Ví dụ, bảng metric_impact trong group report dựa trên logs từ members 2 và 3.

**Bằng chứng (commit/log):**

- Run docs: Tạo pipeline_architecture.md với Mermaid diagram và owner table.
- Run report: Hoàn thiện group_report.md với 6 phần, dẫn `artifacts/eval/*.csv` và `manifests/*.json`.

---

## 2. Một quyết định kỹ thuật (100–150 từ)

**Quyết định: Chọn Mermaid cho sơ đồ kiến trúc thay vì ASCII text.**

Ban đầu cân nhắc ASCII vì đơn giản và không cần tool render. Nhưng quyết định Mermaid vì:

1. **Đọcability:** Mermaid tạo diagram tương tác, dễ hiểu luồng phức tạp (subgraphs cho cleaning, quality, observability). ASCII khó vẽ arrows và subgraphs.

2. **Maintenance:** Mermaid dễ edit trong Markdown, render tự động trong GitHub/VS Code. ASCII dễ lỗi format khi edit.

3. **Professional:** Mermaid chuẩn industry cho docs, phù hợp với runbook và architecture docs. Giúp stakeholders (non-tech) hiểu pipeline nhanh.

Chiến lược này kết hợp với runbook: Mermaid cho overview, table cho diagnosis steps. Nếu không có Mermaid, dùng PlantUML, nhưng Mermaid native Markdown.

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

**Lỗi: Thiếu artifacts dẫn chứng trong group report ban đầu.**

Khi viết group_report.md, tôi nhận thấy phần before/after eval chỉ có placeholder, không dẫn file thật. Điều này làm report "trivial" và mất điểm.

**Xử lý:**

1. **Diagnosis:** Kiểm tra `artifacts/eval/` — có `before_after_eval.csv` và `after_inject_bad.csv`. Đọc CSV thấy `contains_expected=yes` cho 4/4 queries, chứng minh pipeline robust.

2. **Mitigation:** Thêm nội dung cụ thể: "Từ `artifacts/eval/before_after_eval.csv`, 4/4 câu hỏi PASS. Sau inject, vẫn 4/4 PASS nhờ quarantine."

3. **Prevention:** Thiết lập checklist: Mỗi claim phải có file path và excerpt. Ví dụ, freshness từ manifest JSON.

Lỗi này dạy: Docs không chỉ viết, mà phải verify với artifacts. Nếu không, report thành "storytelling" thay vì evidence-based.

---

## 4. Học tập và đóng góp (100–150 từ)

Trong role Monitoring/Docs, tôi học được giá trị của observability trong data pipeline. Freshness check và expectations không chỉ "nice-to-have" mà critical để catch data drift sớm.

**Đóng góp chính:**

- **Documentation:** Tạo runbook chuẩn 5-phase, giúp team respond incident nhanh. Ví dụ, diagnosis table với steps cụ thể.

- **Integration:** Đảm bảo group report link tất cả members' work, từ data_contract đến eval results. Bảng metric_impact tổng hợp impact của rules mới.

- **Quality Assurance:** Review artifacts trước submit, đảm bảo run_id và paths chính xác.

**Kết quả:** Pipeline có full docs, team có playbook cho production. Bước tiếp: Implement auto-alerts và integrate với Day 11 guardrails để chặn bad responses.