# Kiến trúc pipeline — Lab Day 10

**Nhóm:** Nhóm 9 - E403
**Cập nhật:** 

---

## 1. Sơ đồ luồng (bắt buộc có 1 diagram: Mermaid / ASCII)

```
raw export (CSV/API/…)  →  clean  →  validate (expectations)  →  embed (Chroma)  →  serving (Day 08/09)
```

> Vẽ thêm: điểm đo **freshness**, chỗ ghi **run_id**, và file **quarantine**.

```
graph TD
    Raw[Raw Data Export] --> Clean[Clean Stage]
    
    subgraph Member_2 [Cleaning]
        Clean --> |Normalize/Fix| Cleaned[Cleaned Data]
    end

    subgraph Quality_Control [Quality & Contract]
        Contract([data_contract.yaml]) -.-> |Rules| Val
        Expect([expectations.py]) -.-> |Assertions| Val
        Cleaned --> Val{Validate & Expect}
    end
    
    Val -- "Fail (Halt)" --> Quar[Quarantine File]
    Val -- "Pass" --> Embed[Embedding / Chroma]
    
    Inject((Error Injection)) -.-> |Test Stress| Raw
    Quar -.-> |Analysis| Inject

    Embed --> Serving[Serving / RAG]

    subgraph Obs [Observability]
        ID[run_id]
        Logs[Expectations Matrix]
        Fresh[Metric: Data Freshness]
    end

    ID -.-> Val
    Val -.-> Logs
    
    Cleaned -.-> |Check Timestamp| Fresh
    Fresh -.-> |Alert if Stale| Logs
```

---

## 2. Ranh giới trách nhiệm

| Thành phần | Input | Output | Owner nhóm |
|------------|-------|--------|--------------|
| Ingest | File JSON/CSV thô từ hệ thống (policy_export_dirty.csv) | List of Dictionaries (Dữ liệu thô trong bộ nhớ) | 1 |
| Transform | List of Dictionaries (Thô) | (Thô)	Cleaned DataFrame (Đã sửa lỗi font, ngày tháng, xóa URL) | 2 |
| Quality | Cleaned DataFrame + `data_contract.yaml` | Validated DataFrame & Quarantine File (Lưu bản ghi lỗi) | 1 & 3 |
| Embed | Validated DataFrame | Vector Database (ChromaDB) / Final CSV | All |
| Monitor | Pipeline Logs, `run_id`, Metadata | Expectations Matrix & Freshness Metrics | 3 |

---

## 3. Idempotency & rerun

> Mô tả: upsert theo `chunk_id` hay strategy khác? Rerun 2 lần có duplicate vector không?

---

## 4. Liên hệ Day 09

> Pipeline này cung cấp / làm mới corpus cho retrieval trong `day09/lab` như thế nào? (cùng `data/docs/` hay export riêng?)

---

## 5. Rủi ro đã biết

- …
