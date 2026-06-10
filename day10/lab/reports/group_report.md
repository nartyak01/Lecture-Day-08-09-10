# Báo Cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** Khoa  
**Thành viên:** *(làm cá nhân — một người đảm nhiệm các vai trò)*
| Tên | Vai trò (Day 10) | Email |
|-----|------------------|-------|
| Khoa | Ingestion / Raw Owner | 26ai.khoatnd@vinuni.edu.vn |
| Khoa | Cleaning & Quality Owner | 26ai.khoatnd@vinuni.edu.vn |
| Khoa | Embed & Idempotency Owner | 26ai.khoatnd@vinuni.edu.vn |
| Khoa | Monitoring / Docs Owner | 26ai.khoatnd@vinuni.edu.vn |

**Ngày nộp:** 2026-06-10  
**Repo:** `day10/lab`  
**Độ dài khuyến nghị:** 600–1000 từ

---

> **Nộp tại:** `reports/group_report.md`  
> **Sprint 1–4:** `sprint1`, `sprint2-clean`, `inject-bad`, `after-fix`  
> Artifact: `artifacts/eval/grading_run.jsonl`, `after_inject_bad.csv`, `after_clean.csv`, `docs/quality_report.md`, `docs/runbook.md`

---

## 1. Pipeline tổng quan (150–200 từ)

**Tóm tắt luồng:**

Pipeline đọc export raw `data/raw/policy_export_dirty.csv` (10 dòng) mô phỏng batch nightly từ Policy DB + HR + IT Helpdesk. Luồng: **ingest** (`load_raw_csv`) → **clean** (`transform/cleaning_rules.py`: allowlist `doc_id`, chuẩn hóa ngày ISO, quarantine HR cũ, fix refund 14→7 ngày, dedupe) → **validate** (`quality/expectations.py`, halt nếu fail) → **embed** Chroma collection `day10_kb` (upsert `chunk_id`, prune id thừa) → ghi **manifest** + **freshness_check**.

Sprint 1 (`run_id=sprint1`): thiết lập ingest + log số liệu — `raw_records=10`, `cleaned_records=6`, `quarantine_records=4`. Sprint 2 (`run_id=sprint2-clean`): thêm 3 cleaning rule và 3 expectation mới; pipeline exit 0; eval retrieval pass 4/4 câu golden. Idempotency: rerun `sprint2-rerun1` và `sprint2-rerun2` đều `embed_upsert count=6`, không phình collection.

`run_id` lấy từ log `artifacts/logs/run_<run-id>.log` dòng đầu hoặc `artifacts/manifests/manifest_<run-id>.json`.

**Lệnh chạy một dòng:**

```powershell
python etl_pipeline.py run --run-id after-fix; python eval_retrieval.py --out artifacts/eval/after_clean.csv; python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_after-fix.json; python grading_run.py --out artifacts/eval/grading_run.jsonl
```

---

## 2. Cleaning & expectation (150–200 từ)

Baseline: allowlist `doc_id`, parse `effective_date`, quarantine HR `effective_date < 2026-01-01`, fix refund stale, dedupe `chunk_text`. Nhóm thêm **3 rule** và **3 expectation** mới.

**Rule mới:** `chunk_too_short` (<20 ký tự), `missing_exported_at`, `future_effective_date` (>2026-12-31).

**Expectation mới:** `exported_at_not_empty` (halt), `no_duplicate_chunk_id` (halt), `quarantine_ratio_max_50pct` (warn, ngưỡng 50%).

**Expectation halt (dừng pipeline):** E1 `min_one_row`, E2 `no_empty_doc_id`, E3 `refund_no_stale_14d_window`, E5 `effective_date_iso_yyyy_mm_dd`, E6 `hr_leave_no_stale_10d_annual`, E7 `exported_at_not_empty`, E8 `no_duplicate_chunk_id`. **Warn:** E4 `chunk_min_length_8`, E9 `quarantine_ratio_max_50pct`.

### 2a. Bảng metric_impact (bắt buộc — chống trivial)

| Rule / Expectation mới (tên ngắn) | Trước (số liệu) | Sau / khi inject (số liệu) | Chứng cứ (log / CSV / commit) |
|-----------------------------------|------------------|-----------------------------|-------------------------------|
| `chunk_too_short` | 0 quarantine (rule chưa có) | +1 nếu thêm dòng `chunk_text` <20 ký tự vào raw | `quarantine_*.csv` reason=`chunk_too_short` |
| `missing_exported_at` | 0 quarantine | +1 nếu `exported_at` rỗng | `quarantine_sprint2-clean.csv` (baseline dòng 5 đã bị `missing_effective_date` trước) |
| `future_effective_date` | 0 quarantine | +1 nếu `effective_date` > 2026-12-31 | `transform/cleaning_rules.py` |
| `exported_at_not_empty` (halt) | N/A | pass: `empty=0` trên sprint2-clean | `artifacts/logs/run_sprint2-clean.log` |
| `no_duplicate_chunk_id` (halt) | N/A | pass: `duplicates=0` | `run_sprint2-clean.log` |
| `quarantine_ratio_max_50pct` (warn) | ratio=0.40 (4/10) | >0.5 nếu inject thêm dòng lỗi vào CSV | log `expectation[quarantine_ratio_max_50pct] OK (warn)` |

**Rule chính (baseline + mở rộng):**

- Quarantine: `unknown_doc_id`, `missing_effective_date`, `stale_hr_policy_effective_date`, `duplicate_chunk_text`, `missing_chunk_text`
- Fix: refund `14 ngày làm việc` → `7 ngày làm việc` + marker `[cleaned: stale_refund_window]`
- Mở rộng: `chunk_too_short`, `missing_exported_at`, `future_effective_date`

**Ví dụ expectation fail (demo có chủ đích — Sprint 3):**

Chạy `python etl_pipeline.py run --no-refund-fix --skip-validate` → E3 `refund_no_stale_14d_window` FAIL (halt) nhưng pipeline vẫn embed nhờ `--skip-validate`. Pipeline chuẩn (không flag) → E3 OK, eval `q_refund_window` `hits_forbidden=no` (`artifacts/eval/after_clean.csv`).

**Eval Sprint 2 (`artifacts/eval/after_clean.csv`):**

| question_id | contains_expected | hits_forbidden | top1_doc_expected |
|-------------|-------------------|----------------|-------------------|
| q_refund_window | yes | no | — |
| q_leave_version | yes | no | yes |
| q_p1_sla | yes | no | — |
| q_lockout | yes | no | — |

---

## 3. Before / after ảnh hưởng retrieval hoặc agent (200–250 từ)

**Kịch bản inject**

Nhóm mô phỏng incident “export refund stale vẫn được publish vào vector store” bằng:

```powershell
python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate
python eval_retrieval.py --out artifacts/eval/after_inject_bad.csv
```

`--no-refund-fix` giữ chunk chứa “14 ngày làm việc” trong cleaned; `--skip-validate` cho phép embed dù expectation `refund_no_stale_14d_window` halt fail — tương đương bypass quality gate. Manifest `manifest_inject-bad.json` ghi `no_refund_fix=true`, `skipped_validate=true`.

Sau đó chạy pipeline chuẩn `after-fix` (không flag) và eval lại → `artifacts/eval/after_clean.csv`.

**Kết quả định lượng**

| scenario | run_id | file eval | q_refund_window | q_leave_version |
|----------|--------|-----------|-----------------|-----------------|
| Trước fix | inject-bad | `artifacts/eval/after_inject_bad.csv` | contains=yes, **hits_forbidden=yes** | top1_doc_expected=yes |
| Sau fix | after-fix | `artifacts/eval/after_clean.csv` | contains=yes, **hits_forbidden=no** | top1_doc_expected=yes |

Điểm quan trọng: trước fix, `top1_preview` vẫn hiển thị “7 ngày làm việc” nhưng `hits_forbidden=yes` vì `eval_retrieval.py` quét **toàn bộ top-k=3** — chunk stale 14 ngày vẫn nằm trong context retrieval. Agent chỉ đọc top-1 có thể đúng, nhưng grounded prompt với full top-k vẫn rủi ro. Đây là lý do Day 10 đo data layer (và forbidden trên cả top-k) trước khi debug model/prompt.

Sau `after-fix`, rule fix 14→7 và expectation pass → embed sạch, `hits_forbidden` về `no`. Chi tiết: `docs/quality_report.md`.

---

## 4. Freshness & monitoring (100–150 từ)

Sau `run_id=after-fix`, chạy:

```powershell
python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_after-fix.json
```

Kết quả: **PASS** — `latest_exported_at=2026-04-10T08:00:00`, `age_hours≈1465`, `sla_hours=2000` (`.env` `FRESHNESS_SLA_HOURS=2000` cho lab demo). Với SLA production **24h** (trong `contracts/data_contract.yaml`), cùng snapshot sẽ **FAIL** vì export cũ hơn 24 giờ — điều này hợp lý và đã ghi trong `docs/runbook.md`.

SLA đo tại **publish** (`measured_at: publish`): timestamp lấy từ `latest_exported_at` trên manifest, không phải thời điểm máy chạy pipeline. Alert channel: `26ai.khoatnd@vinuni.edu.vn`. Chi tiết bảng PASS/WARN/FAIL và lab vs production SLA: `docs/runbook.md` mục **Freshness**.

---

## 5. Liên hệ Day 09 (50–100 từ)

Pipeline Day 10 là **tầng data** dưới multi-agent Day 09. Cùng narrative CS + IT Helpdesk và corpus gốc `data/docs/`, nhưng ingest qua export CSV (`policy_export_dirty.csv`) thay vì đọc file trực tiếp — mô phỏng batch từ DB/API.

Embed vào collection **`day10_kb`** (tách khỏi index Day 09 để thử pipeline an toàn). Day 09 `retrieval_worker` có thể trỏ `.env`: `CHROMA_COLLECTION=day10_kb`, cùng `EMBEDDING_MODEL=all-MiniLM-L6-v2`. Manifest ghi `run_id` trên metadata chunk — trace được publish boundary khi debug agent.

Luồng tích hợp: Day 10 publish sạch → Day 09 retrieve → `policy_worker` / `synthesis_worker` — garbage in đã được chặn ở expectation halt trước embed.

---

## 6. Rủi ro còn lại & peer review

**Rủi ro còn lại:**
- Freshness PASS trên lab nhờ SLA 2000h — production cần 24h + re-ingest thật
- Chưa có CDC/streaming; chỉ batch CSV
- Bypass `--skip-validate` chỉ dùng demo — phải chặn trên CI production

**Đã hoàn thành:** Sprint 1–4 — pipeline, inject before/after, `quality_report.md`, `data_contract.md`, `pipeline_architecture.md` (Mermaid), `runbook.md` (gồm Freshness PASS/WARN/FAIL), `reports/individual/khoa.md`, `grading_run.jsonl` (3/3 MERIT_CHECK OK). Log runtime: `*.log` gitignore — số liệu chính thức trên `artifacts/manifests/manifest_*.json`.

**Peer review (3 câu):**
1. *Ngưỡng `chunk_too_short` 20 ký tự có quá thấp không? Có nên đọc từ contract thay vì hard-code?*
2. *`quarantine_ratio_max_50pct` nên warn hay halt khi vượt ngưỡng?*
3. *Có nên chặn embed khi `freshness_check=FAIL` dù expectation pass?*
