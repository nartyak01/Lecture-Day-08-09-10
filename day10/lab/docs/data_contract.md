# Data contract — Lab Day 10

> Bắt đầu từ `contracts/data_contract.yaml` — mở rộng và đồng bộ file này.

---

## 1. Nguồn dữ liệu (source map)

| Nguồn | Phương thức ingest | Failure mode chính | Metric / alert |
|-------|-------------------|-------------------|----------------|
| Policy DB export | CSV batch nightly (`data/raw/policy_export_dirty.csv`) | duplicate, stale refund 14 ngày | `quarantine_records`, expectation `refund_no_stale_14d_window` |
| HR policy sync | CSV/API | conflict 10 vs 12 ngày phép | expectation `hr_leave_no_stale_10d_annual` (halt) |
| IT Helpdesk FAQ export | CSV cùng batch | ngày `DD/MM/YYYY`, chunk quá ngắn | quarantine `invalid_effective_date_format` / `chunk_too_short` |

---

## 2. Schema cleaned

| Cột | Kiểu | Bắt buộc | Ghi chú |
|-----|------|----------|---------|
| chunk_id | string | Có | hash ổn định từ doc_id + chunk_text + seq |
| doc_id | string | Có | phải thuộc allowlist trong cleaning_rules.py |
| chunk_text | string | Có | min 20 ký tự sau strip; refund stale 14→7 ngày được fix trước publish |
| effective_date | date | Có | ISO YYYY-MM-DD sau normalize |
| exported_at | datetime | Có |  timestamp export từ nguồn, bắt buộc không rỗng |

---

## 3. Quy tắc quarantine vs drop

Record fail rule → `artifacts/quarantine/quarantine_<run-id>.csv`, **không** embed vào Chroma. Lý do quarantine trên bộ mẫu Sprint 1 (`run_id=sprint1`): `duplicate_chunk_text`, `missing_effective_date`, `stale_hr_policy_effective_date`, `unknown_doc_id`. Review thủ công trước khi sửa nguồn và re-ingest. Không auto-merge vào cleaned.

---

## 4. Phiên bản & canonical

Refund: `data/docs/policy_refund_v4.txt` (7 ngày làm việc). HR leave: `data/docs/hr_leave_policy.txt` (12 ngày, effective ≥ 2026-01-01 theo `policy_versioning.hr_leave_min_effective_date` trong YAML). Export CSV là snapshot — cleaned phải khớp canonical sau rule fix.

---

## 5. Freshness & alert (đồng bộ `contracts/data_contract.yaml`)

| Trường | Giá trị | Ghi chú |
|--------|---------|---------|
| `owner_team` | Khoa | Chịu trách nhiệm contract + quarantine review |
| `freshness.measured_at` | publish | Đo `latest_exported_at` trên manifest sau embed |
| `freshness.sla_hours` | 24 (YAML) / 2000 (`.env` lab) | Production: 24h; lab demo PASS: `FRESHNESS_SLA_HOURS=2000` |
| `freshness.alert_channel` | 26ai.khoatnd@vinuni.edu.vn | Alert khi `freshness_check=FAIL` |

**Quy trình khi FAIL:** xem `docs/runbook.md` — kiểm tra manifest → re-ingest hoặc cập nhật snapshot nguồn → rerun `python etl_pipeline.py run`.
