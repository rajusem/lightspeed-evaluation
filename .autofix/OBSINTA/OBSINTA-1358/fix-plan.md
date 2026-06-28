# Fix Plan for OBSINTA-1358

**Ticket**: [OBSINTA-1358](https://stage-redhat.atlassian.net/browse/OBSINTA-1358)
**Original Issue**: LEADS-230 — Missing metric_metadata in CSV
**Plan Version**: v1 (audit skipped — all confidence HIGH, simple fix ≤3 files, ≤4 lines)

---

## Root Cause

The `metric_metadata` column name in `src/lightspeed_evaluation/core/constants.py` (line 88)
and `config/system.yaml` (line 194) does **not** match the actual `EvaluationResult` model
field name `metrics_metadata` (plural, with an `s`) defined in
`src/lightspeed_evaluation/core/models/data.py` (line 502).

The CSV generator in `src/lightspeed_evaluation/core/output/generator.py` (lines 193–202) uses
`hasattr(result, column)` to check if an `EvaluationResult` has a given column. When the column
name is `"metric_metadata"`, `hasattr` returns `False` (because the actual field is
`"metrics_metadata"`), so an empty string is written to the CSV — causing the `metric_metadata`
column to always be empty even when metadata is available.

The evaluator correctly computes the metadata value:
```python
# evaluator.py:172 and evaluator.py:511
metrics_metadata=self._extract_metadata_for_csv(request)
```
...but it's stored under `metrics_metadata` on the result object, which cannot be accessed
by the wrong column name `"metric_metadata"`.

---

## Approach

Fix the column name in both `constants.py` and `config/system.yaml` to use `"metrics_metadata"`
(plural, matching the actual model field name). This is the minimal fix: 2 files, 2 single-line
corrections, no model or evaluator changes needed.

---

## Files to Change

| File | Line | Change |
|------|------|--------|
| `src/lightspeed_evaluation/core/constants.py` | 88 | `"metric_metadata"` → `"metrics_metadata"` |
| `config/system.yaml` | 194 | `- "metric_metadata"` → `- "metrics_metadata"` |
| `tests/unit/core/output/test_generator.py` | New test | Add `test_metrics_metadata_column_populated` regression test |

### Exact diff for `constants.py`
```diff
-    "metric_metadata",
+    "metrics_metadata",
```

### Exact diff for `config/system.yaml`
```diff
-    - "metric_metadata"
+    - "metrics_metadata"
```

### New regression test (in `tests/unit/core/output/test_generator.py`)
```python
def test_metrics_metadata_column_populated(
    self, tmp_path: Path, mocker: MockerFixture
) -> None:
    """Regression test: metrics_metadata column must not be empty when data is present."""
    import csv as csv_module

    results = [
        EvaluationResult(
            conversation_group_id="conv1",
            turn_id="turn1",
            metric_identifier="ragas:faithfulness",
            result="PASS",
            score=0.9,
            threshold=0.7,
            reason="Good",
            metrics_metadata='{"model": "gpt-4", "temperature": 0.7}',
        )
    ]

    mocker.patch("builtins.print")

    system_config = mocker.Mock()
    system_config.output.csv_columns = [
        "conversation_group_id",
        "metrics_metadata",
    ]
    handler = OutputHandler(output_dir=str(tmp_path), system_config=system_config)
    csv_file = handler._generate_csv_report(results, "test_metrics_metadata")

    with open(csv_file, encoding="utf-8") as f:
        reader = csv_module.DictReader(f)
        rows = list(reader)

    assert len(rows) == 1
    # This assertion would have FAILED before the fix (empty string instead of JSON)
    assert rows[0]["metrics_metadata"] == '{"model": "gpt-4", "temperature": 0.7}'
```

---

## Side Effects & Considerations

- **Custom YAML overrides**: Any user with a custom `system.yaml` containing `- "metric_metadata"`
  in `csv_columns` will need to rename it to `- "metrics_metadata"`. Since the column was always
  empty before this fix, real downstream impact is minimal.
- **No API changes**: The `EvaluationResult` model is unchanged.
- **No evaluator changes**: The evaluator already correctly assigns `metrics_metadata`.

---

## Confidence

| Dimension | Score | Evidence |
|-----------|-------|----------|
| Root cause certainty | HIGH | Direct code trace: `constants.py:88` → `generator.py:194` → `hasattr` fails → empty column |
| Approach correctness | HIGH | All other columns (e.g., `conversation_group_id`) match model field names exactly |
| Scope completeness | HIGH | `grep -rn "metric_metadata"` confirms only 2 non-test occurrences: `constants.py:88` and `system.yaml:194` |

---

## Audit Trail

- **Audit**: Skipped (AUDIT_SKIP_SIMPLE=true, all dimensions HIGH, ≤3 files, ≤4 lines changed, regression signal with clear root cause)
