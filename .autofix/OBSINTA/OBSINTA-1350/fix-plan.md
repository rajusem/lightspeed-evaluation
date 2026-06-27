## Fix Plan for OBSINTA-1350

### Version
Plan v1 | Approved (audit skipped — simple fix, all confidence HIGH, 2 files, <20 lines)

### Root Cause
`validate_evaluation_data()` is called **twice** on the same data:

1. **First call** — In `DataValidator.load_evaluation_data()` (validator.py:142), called from the runner (`runner/evaluation.py:58`). This validates the data as part of loading it from YAML.

2. **Second call** — In `EvaluationPipeline.run_evaluation()` (pipeline.py:141), which calls `self.validate_data()` → `self.data_validator.validate_evaluation_data()`. This re-validates the exact same already-validated data before processing.

Since `run_evaluation()` in the runner (`evaluation.py:58`) always calls `load_evaluation_data()` (which validates internally) before passing data to `pipeline.run_evaluation()`, the pipeline's validation at line 141 is redundant. Both create separate `DataValidator` instances with the same config, and both call `validate_evaluation_data()` which runs `_validate_metrics_availability()` and `_validate_metric_requirements()`.

### Approach
Remove the duplicate validation from `EvaluationPipeline.run_evaluation()`. The pipeline should trust that incoming data has already been validated by the caller (the runner). This is the simplest fix:

1. Remove the `validate_data()` call from `EvaluationPipeline.run_evaluation()` (pipeline.py lines 139-142)
2. Remove the `validate_data()` method from `EvaluationPipeline` (pipeline.py lines 117-119) since it becomes unused
3. Remove the `DataValidator` import and instantiation from the pipeline since it's no longer needed there (pipeline.py lines 20, 59-68)
4. Update tests to reflect the removed validation from the pipeline

### Alternatives Considered
| # | Approach | Pros | Cons | Why Not |
|---|----------|------|------|---------|
| 1 | Keep validation in pipeline, remove from load | Centralizes validation in one place | Would allow invalid data to be loaded without error; other callers of `load_evaluation_data` would lose validation | Validation during load is the safer default — catch errors early |
| 2 | Add a `skip_validation` flag to pipeline's `run_evaluation` | Backward compatible, allows both paths | Adds complexity for a simple problem; flag defaults are confusing | Over-engineering for what should be a simple removal |
| 3 | Cache validation result on data objects | Avoids re-running but keeps both calls | Requires modifying data models; still unnecessary complexity | Over-engineering |

### Files to Change
| File | Change | Reason |
|------|--------|--------|
| `src/lightspeed_evaluation/pipeline/evaluation/pipeline.py` | Remove `validate_data()` method, remove validation call from `run_evaluation()`, remove `DataValidator` import and `self.data_validator` initialization | Eliminate the duplicate validation |
| `tests/unit/pipeline/evaluation/test_pipeline.py` | Remove `test_validate_data` test, remove `test_run_evaluation_validation_failure` (validation no longer happens in pipeline), remove `DataValidator` mock setup from remaining tests | Tests must reflect the removed validation |

### Detailed Changes

#### `src/lightspeed_evaluation/pipeline/evaluation/pipeline.py`

1. **Remove import** (line 20): Remove `DataValidator` from the `from lightspeed_evaluation.core.system import ConfigLoader, DataValidator` import. Change to:
   ```python
   from lightspeed_evaluation.core.system import ConfigLoader
   ```

2. **Remove `self.data_validator` initialization** (lines 59-68 in `_initialize_components`): Remove the entire block:
   ```python
   # Data validator
   config = self.config_loader.system_config
   if config is None:
       raise ValueError(
           "SystemConfig must be loaded before initializing components"
       )
   self.data_validator = DataValidator(
       api_enabled=config.api.enabled,
       fail_on_invalid_data=config.core.fail_on_invalid_data,
   )
   ```
   Replace with just the config check (still needed for `MetricManager`):
   ```python
   config = self.config_loader.system_config
   if config is None:
       raise ValueError(
           "SystemConfig must be loaded before initializing components"
       )
   ```

3. **Remove `validate_data()` method** (lines 117-119): Delete entirely:
   ```python
   def validate_data(self, evaluation_data: list[EvaluationData]) -> bool:
       """Validate evaluation data using data validator."""
       return self.data_validator.validate_evaluation_data(evaluation_data)
   ```

4. **Remove validation call from `run_evaluation()`** (lines 139-142): Remove:
   ```python
   # Step 1: Validate data
   logger.info("Validating data")
   if not self.validate_data(evaluation_data):
       raise ValueError("Data validation failed. Cannot proceed with evaluation.")
   ```
   Update the remaining step comments (Step 2 → Step 1, Step 3 → Step 2).

#### `tests/unit/pipeline/evaluation/test_pipeline.py`

1. **Remove `test_validate_data` test** (lines 144-173)
2. **Remove `test_run_evaluation_validation_failure` test** (lines 224-253) — validation no longer happens in pipeline
3. **Remove `DataValidator` mock** from all remaining pipeline tests — no longer needed since the pipeline doesn't use `DataValidator`

### Dependencies & Side Effects
- [ ] Public API change? — Yes, minor: `EvaluationPipeline.validate_data()` public method is removed. However, it is only called internally by `run_evaluation()` and not by external consumers.
- [ ] Config / env var change? — No
- [ ] Database migration? — No
- [ ] Downstream consumer impact? — No. The only caller of `pipeline.run_evaluation()` is `runner/evaluation.py`, which already validates via `load_evaluation_data()`.
- [ ] Error handling / logging change? — The `ValueError("Data validation failed")` from pipeline.py:142 is removed. The equivalent `DataValidationError("Evaluation data validation failed")` from validator.py:143 already catches the same case earlier.
- [ ] Performance characteristics change? — Yes, positive: eliminates redundant validation pass over all evaluation data (metrics availability check + metric requirements check for every conversation/turn).

### Risk Assessment
| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| External caller bypasses runner and calls pipeline directly without validating | Low | Medium | The runner is the only entry point; the pipeline constructor doesn't load data |
| Tests break due to mock changes | Medium | Low | Update tests to remove DataValidator mocks from pipeline tests |

### Test Strategy
- Existing tests to verify: `tests/unit/core/system/test_validator.py` (all validation tests remain — they test DataValidator directly)
- Tests to update: `tests/unit/pipeline/evaluation/test_pipeline.py` — remove `test_validate_data`, remove `test_run_evaluation_validation_failure`, simplify DataValidator mock setup in remaining pipeline tests
- Regression: Run full test suite (`make test`) to ensure no other code path depends on pipeline-level validation

### Confidence
| Dimension | Score | Proof |
|-----------|-------|-------|
| Root cause certainty | HIGH | Clear code trace: `load_evaluation_data()` at validator.py:142 calls `validate_evaluation_data()`, then `run_evaluation()` at pipeline.py:141 calls it again on the same data |
| Approach correctness | HIGH | Removing the second call is safe because `load_evaluation_data()` already validates and raises `DataValidationError` on failure — invalid data never reaches the pipeline |
| Scope completeness | HIGH | Only 2 source files affected (pipeline.py + its test). Validation logic in validator.py is untouched. |

### Investigation Strategy
**Signals detected**: default
**Strategy used**: Standard investigation (grep, file reads, code path tracing)
**Key findings from strategy**:
  - The `DataValidator` class is instantiated in both the runner and the pipeline with identical config
  - `validate_evaluation_data()` runs `_validate_metrics_availability()` and `_validate_metric_requirements()` — both iterate over all conversations and turns
  - The runner's `load_evaluation_data()` raises `DataValidationError` if validation fails, so invalid data never reaches the pipeline

### Audit Trail
Audit skipped — simple fix: all confidence dimensions HIGH, 2 files to change, <20 lines changed, default signal classification.
