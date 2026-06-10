# Runbook — Lab Day 10 (incident tối giản)

**Owner:** Khoa · **Alert:** 26ai.khoatnd@vinuni.edu.vn

---

## Symptom

Agent hoặc user thấy câu trả lời / context retrieval chứa **“14 ngày làm việc”** thay vì **7 ngày** cho policy refund — hoặc agent hallucinate số ngày không khớp canonical `policy_refund_v4`.

---

## Detection

- **Eval:** `python eval_retrieval.py` → `q_refund_window` có `hits_forbidden=yes` (quét toàn top-k, không chỉ top-1).
- **Pipeline log / manifest:** `expectation[refund_no_stale_14d_window] FAIL (halt)`; `manifest_*.json` có `no_refund_fix=true` hoặc `skipped_validate=true`.
- **Freshness:** `python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_<run-id>.json` → `FAIL` nếu snapshot export cũ hơn SLA (xem mục Freshness bên dưới).

---

## Diagnosis

| Bước | Việc làm | Kết quả mong đợi |
|------|----------|------------------|
| 1 | Mở `artifacts/manifests/manifest_<run-id>.json` | `skipped_validate=false`, `no_refund_fix=false` trên run production |
| 2 | Mở `artifacts/quarantine/quarantine_<run-id>.csv` | HR cũ (`stale_hr_policy`), doc lạ (`unknown_doc_id`) đã tách |
| 3 | Mở `artifacts/cleaned/cleaned_<run-id>.csv` | Không còn chuỗi `14 ngày làm việc` trên `policy_refund_v4` |
| 4 | `python eval_retrieval.py --out artifacts/eval/after_clean.csv` | `q_refund_window`: `hits_forbidden=no` |

---

## Mitigation

1. **Không** dùng `--skip-validate` hoặc `--no-refund-fix` ngoài demo có ghi nhận.
2. Chạy pipeline chuẩn:
   ```powershell
   python etl_pipeline.py run --run-id after-fix
   python eval_retrieval.py --out artifacts/eval/after_clean.csv
   ```
3. Verify: `hits_forbidden=no` trên `q_refund_window`; `grading_run.jsonl` nếu nộp bài chấm.
4. Rollback: rerun từ raw đã sửa nguồn hoặc cleaned CSV đã pass expectation.

---

## Prevention

- Expectation `refund_no_stale_14d_window` severity **halt** — block publish khi còn chunk 14 ngày.
- Expectation `hr_leave_no_stale_10d_annual` halt — chặn HR version conflict.
- Không embed khi expectation halt (CI/CD gate).
- Review `artifacts/quarantine/` hàng ngày — owner team Khoa.
- Alert email khi `freshness_check=FAIL` (production SLA 24h).

---

## Freshness: PASS / WARN / FAIL

Lệnh:

```powershell
python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_after-fix.json
```

| Status | Điều kiện | Ý nghĩa | Hành động |
|--------|-----------|---------|-----------|
| **PASS** | `age_hours ≤ sla_hours` | Snapshot export còn trong SLA | Tiếp tục serving |
| **WARN** | Không parse được timestamp trên manifest | Thiếu `latest_exported_at` | Kiểm tra pipeline ghi manifest |
| **FAIL** | `age_hours > sla_hours` | Data snapshot quá cũ | Re-ingest nguồn; alert owner |

**Lab vs production:**

- **Production** (`contracts/data_contract.yaml`): `sla_hours: 24`, đo tại **publish** (`measured_at: publish`).
- **Lab demo:** `.env` `FRESHNESS_SLA_HOURS=2000` → PASS với `exported_at=2026-04-10` (giải thích nhất quán trong `data_contract.md` mục 5).
- Với SLA 24h, cùng snapshot → **FAIL** (hợp lý) — không phải lỗi code pipeline.

**Ví dụ thực tế (`after-fix`):**

```
PASS {"latest_exported_at": "2026-04-10T08:00:00", "age_hours": ~1465, "sla_hours": 2000.0}
```
