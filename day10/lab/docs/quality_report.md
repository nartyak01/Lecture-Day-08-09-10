# Quality report — Lab Day 10 (nhóm Khoa)

**run_id:** `inject-bad` (trước) → `after-fix` (sau)  
**Ngày:** 2026-06-10

---

## 1. Tóm tắt số liệu

| Chỉ số | Trước (inject-bad) | Sau (after-fix) | Ghi chú |
|--------|-------------------|-----------------|---------|
| raw_records | 10 | 10 | Cùng file `data/raw/policy_export_dirty.csv` |
| cleaned_records | 6 | 6 | |
| quarantine_records | 4 | 4 | |
| Expectation halt? | E3 `refund_no_stale_14d_window` **FAIL** — bỏ qua nhờ `--skip-validate` | Tất cả expectation halt **OK** | Manifest: `skipped_validate=true` vs `false` |

---

## 2. Before / after retrieval (bắt buộc)

**Artifact:** `artifacts/eval/after_inject_bad.csv` (trước) · `artifacts/eval/after_clean.csv` (sau, sau `run_id=after-fix`)

### Câu hỏi then chốt: refund window (`q_refund_window`)

**Trước (inject-bad):**

```
q_refund_window,...,policy_refund_v4,...7 ngày làm việc...,yes,yes,,3
```

- `contains_expected=yes` — top-k vẫn có chunk đúng “7 ngày”.
- `hits_forbidden=yes` — trong top-k=3 vẫn còn chunk chứa **“14 ngày làm việc”** (không fix nhờ `--no-refund-fix`).
- `top1_preview` hiển thị 7 ngày nhưng context đầy đủ vẫn toxic → agent có thể hallucinate nếu đọc cả context.

**Sau (after-fix):**

```
q_refund_window,...,policy_refund_v4,...7 ngày làm việc...,yes,no,,3
```

- Rule fix 14→7 + expectation E3 pass → không còn forbidden trong top-k.

### Merit — versioning HR (`q_leave_version`)

| | contains_expected | hits_forbidden | top1_doc_expected |
|--|-------------------|----------------|-------------------|
| Trước (inject-bad) | yes | no | yes |
| Sau (after-fix) | yes | no | yes |

Inject Sprint 3 chỉ nhắm refund stale; bản HR 10 ngày đã bị quarantine từ bước clean (`stale_hr_policy_effective_date`) nên không ảnh hưởng retrieval HR.

---

## 3. Freshness & monitor

Sau `after-fix`: `freshness_check=PASS` với `FRESHNESS_SLA_HOURS=2000` (lab); với SLA production 24h → FAIL. Không liên quan trực tiếp inject refund. Bảng PASS/WARN/FAIL và hành động: `docs/runbook.md` mục **Freshness**.

---

## 4. Corruption inject (Sprint 3)

**Lệnh inject:**

```powershell
python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate
python eval_retrieval.py --out artifacts/eval/after_inject_bad.csv
```

**Cách làm hỏng dữ liệu:**

| Flag | Tác dụng |
|------|----------|
| `--no-refund-fix` | Giữ chunk refund chứa “14 ngày làm việc” trong cleaned (mô phỏng lỗi migration/sync) |
| `--skip-validate` | Embed dù expectation `refund_no_stale_14d_window` halt fail (mô phỏng bypass quality gate) |

**Phát hiện:**

1. Log pipeline: `expectation[refund_no_stale_14d_window] FAIL (halt)` + `WARN: ... --skip-validate`
2. Manifest `manifest_inject-bad.json`: `no_refund_fix=true`, `skipped_validate=true`
3. Eval: `hits_forbidden=yes` trên `q_refund_window`

**Mitigation (sau fix):**

```powershell
python etl_pipeline.py run --run-id after-fix
python eval_retrieval.py --out artifacts/eval/after_clean.csv
```

→ `hits_forbidden=no`; manifest `after-fix`: `no_refund_fix=false`, `skipped_validate=false`.

---

## 5. Hạn chế & việc chưa làm

- Chưa inject riêng kịch bản làm fail `q_leave_version` (Merit mở rộng).
- Freshness/runbook chi tiết: hoàn thiện Sprint 4.
- Chưa chạy `grading_run.jsonl`.
