## Fix Plan for OBSINTA-1360

### Version
Plan v1 — APPROVED, audit skipped

Simple fix — all confidence HIGH, ≤2 files, <20 lines.

### Root Cause
Field name mismatch in the data model — `EvaluationResult` uses `metrics_metadata` (plural) but CSV expects `metric_metadata` (singular). The evaluator correctly extracts metadata but since the field names don't match, the CSV column is always empty.

### Approach
Rename the field in `EvaluationResult` from `metrics_metadata` to `metric_metadata` (line 502 of data.py) to match the CSV column definition in `constants.py`.

### Planned Files
- `src/lightspeed_evaluation/core/models/data.py` — Rename field `metrics_metadata` → `metric_metadata` at line 502

### Audit Trail
- Not required: Simple fix with HIGH confidence on all dimensions