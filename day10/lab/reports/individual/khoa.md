# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Khoa  
**Vai trò:** Cleaning & Quality Owner (+ Embed support)  
**Ngày nộp:** 2026-06-10  
**Độ dài yêu cầu:** **400–650 từ**

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

**File / module:**

- `transform/cleaning_rules.py` — thêm 3 rule: `chunk_too_short`, `missing_exported_at`, `future_effective_date`
- `quality/expectations.py` — thêm E7 `exported_at_not_empty`, E8 `no_duplicate_chunk_id`, E9 `quarantine_ratio_max_50pct`; mở rộng signature `run_expectations(..., raw_count=, quarantine_count=)`
- `etl_pipeline.py` — truyền `raw_count` / `quarantine_count` vào expectation suite

**Kết nối với thành viên khác:**

Ingestion Owner chạy `etl_pipeline.py run` và lưu manifest; tôi đảm bảo clean + validate trước embed. Embed Owner kiểm tra `embed_upsert count=6` sau mỗi run. Monitoring Owner điền runbook và freshness dựa trên manifest tôi giúp validate.

**Bằng chứng:**

Commit trên `cleaning_rules.py`, `expectations.py`, `etl_pipeline.py`; log `run_id=sprint2-clean`, `inject-bad`, `after-fix`.

---

## 2. Một quyết định kỹ thuật (100–150 từ)

Tôi chọn **`quarantine_ratio_max_50pct` severity `warn`** thay vì `halt`.

Bối cảnh: tỷ lệ quarantine cao cho thấy nguồn xấu, nhưng không phải lúc nào cũng nên dừng publish — ví dụ batch có nhiều duplicate lịch sử vẫn còn đủ chunk hợp lệ cho retrieval. `halt` phù hợp cho lỗi semantic (refund 14 ngày, HR 10 ngày) vì ảnh hưởng trực tiếp câu trả lời agent.

Trade-off: `warn` có thể bỏ sót nguồn quá bẩn nếu không có alert. Nhóm bù bằng log `expectation[quarantine_ratio_max_50pct]` và bảng `metric_impact` trong group report. Trên mẫu lab: ratio=0.40 → OK; inject thêm dòng lỗi có thể đẩy >0.5 → FAIL warn nhưng pipeline vẫn chạy để demo.

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

**Triệu chứng:** `python etl_pipeline.py run --run-id sprint2-clean` crash với `TypeError: run_expectations() takes 1 positional argument but 3 were given`.

**Phát hiện:** traceback tại `etl_pipeline.py` dòng gọi `run_expectations(cleaned, raw_count, len(quarantine))` — E9 cần `quarantine_count` nhưng hàm chưa nhận tham số keyword-only.

**Fix:** (1) Cập nhật signature `run_expectations(..., *, raw_count=0, quarantine_count=0)`. (2) Gọi bằng keyword: `raw_count=raw_count, quarantine_count=len(quarantine)`. (3) Di chuyển E7–E9 lên trước `return` — trước đó nằm sau `return` nên là dead code.

**Evidence:** Pipeline `sprint2-clean` exit 0; log có đủ 9 expectation; E9 `quarantine=4 raw=10 ratio=0.40`.

---

## 4. Bằng chứng trước / sau (80–120 từ)

**run_id:** `inject-bad` (trước) → `after-fix` (sau)

**Trước** (`artifacts/eval/after_inject_bad.csv`, `q_refund_window`):
`contains_expected=yes, hits_forbidden=yes` — top-k vẫn còn chunk “14 ngày làm việc” dù top-1 hiển thị 7 ngày.

**Sau** (`artifacts/eval/after_clean.csv`):
`contains_expected=yes, hits_forbidden=no`

**Grading** (`artifacts/eval/grading_run.jsonl` sau `after-fix`):
`gq_d10_01`: `contains_expected=true`, `hits_forbidden=false` — MERIT_CHECK OK.

---

## 5. Cải tiến tiếp theo (40–80 từ)

Nếu có thêm 2 giờ, tôi sẽ đọc `hr_leave_min_effective_date` từ `contracts/data_contract.yaml` / env thay vì hard-code `2026-01-01` trong `cleaning_rules.py`, rồi chứng minh inject đổi quyết định quarantine khi đổi cutoff — hướng Merit/Distinction rubric.
