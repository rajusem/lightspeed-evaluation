## Fix Plan for OBSINTA-1351

### Version
Plan v2 | Iteration 1 (post-audit, approved)

### Ticket
[OBSINTA-1351] [EVAL] LEADS-205 x Claude Sonnet 4.6 - Duplicate data validation

---

## Root Cause
`validate_evaluation_data()` is called twice for every evaluation run:

1. **First call (loader)**: Inside `DataValidator.load_evaluation_data()` in
   `src/lightspeed_evaluation/core/system/validator.py` (line 142). This validates data
   immediately after loading from YAML.

2. **Second call (pipeline — redundant)**: Inside `EvaluationPipeline.run_evaluation()`
   in `src/lightspeed_evaluation/pipeline/evaluation/pipeline.py` (lines 139-142). This
   re-validates data that was already validated during load.

Since the runner (`src/lightspeed_evaluation/runner/evaluation.py`) always calls
`load_evaluation_data()` first (which validates), then passes the already-validated list
to `pipeline.run_evaluation()`, the pipeline's call is redundant overhead — every metric
requirement and field check is performed twice for every evaluation.

---

## Approach
Remove the redundant validation block from `EvaluationPipeline.run_evaluation()`.

**Exact lines to remove** from `src/lightspeed_evaluation/pipeline/evaluation/pipeline.py`:

```python
        # Step 1: Validate data                                          <- line 139, REMOVE
        logger.info("Validating data")                                    <- line 140, REMOVE
        if not self.validate_data(evaluation_data):                       <- line 141, REMOVE
            raise ValueError("Data validation failed. Cannot proceed with evaluation.")  <- line 142, REMOVE
```

**Do NOT remove**: line 137 (`results: list[EvaluationResult] = []`), line 138 (blank),
or any other code.

**Docstring update** for `run_evaluation()` — add a `Note:` section:
```
Note:
    Data is expected to be pre-validated before calling this method.
    Use DataValidator.load_evaluation_data() which validates during load,
    or call pipeline.validate_data() explicitly before run_evaluation().
```

The public `validate_data()` method on `EvaluationPipeline` is kept intact — it remains
available for callers who need to validate data explicitly outside the standard flow.

---

## Planned Files to Change

| File | Change | Reason |
|------|--------|--------|
| `src/lightspeed_evaluation/pipeline/evaluation/pipeline.py` | Remove lines 139-142 (validation block). Update `run_evaluation()` docstring to document pre-validation contract. | Eliminates duplicate validation |
| `tests/unit/pipeline/evaluation/test_pipeline.py` | Delete `test_run_evaluation_validation_failure`. Add `test_run_evaluation_does_not_double_validate`. | Keeps test suite aligned with new behavior |

---

## Test Specification

### Delete test
Remove `test_run_evaluation_validation_failure` from `tests/unit/pipeline/evaluation/test_pipeline.py` — this test relies on the removed code path.

### Add test (exact specification)
```python
def test_run_evaluation_does_not_double_validate(
    self, mock_config_loader, sample_evaluation_data, mocker
):
    """Test that validation only happens once (in loader, not pipeline)."""
    mock_validator_class = mocker.patch(
        "lightspeed_evaluation.pipeline.evaluation.pipeline.DataValidator"
    )
    mock_validator_instance = mock_validator_class.return_value
    mock_validator_instance.validate_evaluation_data.return_value = True

    mocker.patch("lightspeed_evaluation.pipeline.evaluation.pipeline.MetricManager")
    mocker.patch(
        "lightspeed_evaluation.pipeline.evaluation.pipeline.APIDataAmender"
    )
    mocker.patch(
        "lightspeed_evaluation.pipeline.evaluation.pipeline.EvaluationErrorHandler"
    )
    mocker.patch(
        "lightspeed_evaluation.pipeline.evaluation.pipeline.ScriptExecutionManager"
    )
    mocker.patch(
        "lightspeed_evaluation.pipeline.evaluation.pipeline.MetricsEvaluator"
    )
    mock_processor = mocker.Mock()
    mock_processor.process_conversation.return_value = []
    mocker.patch(
        "lightspeed_evaluation.pipeline.evaluation.pipeline.ConversationProcessor",
        return_value=mock_processor,
    )

    pipeline = EvaluationPipeline(mock_config_loader)
    pipeline.run_evaluation(sample_evaluation_data)

    # Validation should NOT be called inside run_evaluation
    mock_validator_instance.validate_evaluation_data.assert_not_called()
```

---

## Dependencies & Side Effects
- **Public API change**: `run_evaluation()` no longer validates internally. Standard callers
  using `load_evaluation_data()` are unaffected (pre-validated by loader). Direct API
  callers must use `validate_data()` explicitly. Docstring documents this contract.
- **Config / env var change**: None
- **Database migration**: None
- **Exception type**: `DataValidationError` inherits from `EvaluationError(Exception)`,
  NOT `ValueError`. The runner's `except (FileNotFoundError, ValueError, RuntimeError)`
  clause already doesn't catch it — this is a pre-existing gap, NOT introduced by this fix.
- **Log observability**: The "Validating data" log message is removed from the pipeline.
  Consider adding a `logger.debug()` in the runner at the load step to maintain traceability.
- **Performance**: Positive — eliminates one full traversal of all evaluation data and metric
  requirements checking per run.

## Risk Assessment
| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Developer removes wrong lines (e.g., line 137 results init) | Low | High | Exact verbatim code block specified above |
| Direct API users bypass loader with invalid data | Low | Medium | Docstring update; `validate_data()` remains public |
| Test suite misses regression | Low | Low | Behavioral `assert_not_called()` test, not structural spy |

---

## Alternatives Considered
| # | Approach | Pros | Cons | Why Not |
|---|----------|------|------|---------|
| 1 | Remove `validate_data()` call + docstring (selected) | Simple, minimal, eliminates duplication | Callers bypassing loader lose implicit guard | Selected — docstring makes contract explicit |
| 2 | Add `pre_validated` flag to `run_evaluation()` | Allows callers to skip validation explicitly | More complex API change | More complex than necessary |
| 3 | Remove `validate_evaluation_data()` from `load_evaluation_data()` | Single validation point | Breaks loader's self-contained contract | Loader should validate what it loads |
| 4 | Keep both, add skip flag to DataValidator | Flexible | Over-engineered | Unnecessary complexity |

---

## Confidence
| Dimension | Score | Proof |
|-----------|-------|-------|
| Root cause certainty | HIGH | Both validation calls confirmed by line-number verified code reading: `validator.py:142`, `pipeline.py:141` |
| Approach correctness | HIGH | Removing lines 139-142 (exact block verified) is straightforward. Loader pre-validates; pipeline re-validation is provably redundant. |
| Scope completeness | HIGH | 2 files only. Exception hierarchy verified: `DataValidationError(EvaluationError(Exception))` — pre-existing gap in runner's catch clause, not introduced by fix. |

---

## Audit Trail
**Iteration 1** (2026-06-28):
- **Architecture** (REVISE → addressed): ARCH-001 public API contract, ARCH-002 wrong line numbers, ARCH-003 dual DataValidator config, ARCH-004 incomplete test, ARCH-005 exception type
- **PE** (APPROVE): PE-001 log observability (noted in side effects), PE-002 docstring (added to plan)
- **Language Expert** (REVISE → addressed): LANG-001 test underspecified (full spec added), LANG-002 brittle spy test (replaced with `assert_not_called()`), LANG-003 docstring missing (added to plan)

All MAJOR findings addressed. No CRITICAL findings. Plan **APPROVED**.
