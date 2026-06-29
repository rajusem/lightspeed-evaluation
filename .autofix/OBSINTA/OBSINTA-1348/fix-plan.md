# Fix Plan for OBSINTA-1348 / LEADS-205

### Version
Plan v1 | Iteration 0 (initial draft)

### Root Cause
Data validation runs **twice** unnecessarily in the evaluation pipeline:

1. **First validation** (in `runner/evaluation.py`, line 58): `DataValidator.load_evaluation_data()` — loads YAML, deserializes into Pydantic `EvaluationData` objects (triggering Pydantic field validators for deduplication), then calls `DataValidator.validate_evaluation_data()` to check metric availability and requirements.

2. **Second validation** (in `pipeline/evaluation/pipeline.py`, line 141): `EvaluationPipeline.run_evaluation()` calls `self.validate_data(evaluation_data)`, which delegates to the pipeline's own `DataValidator.validate_evaluation_data()` — repeating the same metric availability and metric requirements checks just performed in step 1.

The Pydantic field validators (in `src/lightspeed_evaluation/core/models/data.py`) already deduplicate metrics and validate format. The `DataValidator` then validates metric availability and requirements **twice** — once in `load_evaluation_data()` and once in `run_evaluation()`. This is unnecessary overhead since the data is already validated and ready to use by the time `run_evaluation()` receives it.

### Approach
Remove the redundant `validate_data()` call in `EvaluationPipeline.run_evaluation()`. The data is already validated during `DataValidator.load_evaluation_data()` in the runner — the pipeline receives pre-validated `EvaluationData` objects.

**File**: `src/lightspeed_evaluation/pipeline/evaluation/pipeline.py`
- Remove lines 139-142 (`# Step 1: Validate data` and `validate_data()` call + check)
- Update the comment ordering for remaining steps

### Alternatives Considered
| # | Approach | Pros | Cons | Why Not |
|---|----------|------|------|---------|
| 1 | Merge validation into `Pipeline.__init__` | Keeps validation in pipeline | Changes API contract, more complex | Unnecessary — data already validated at entry |
| 2 | Add `validate_on_run` flag to pipeline | Configurable, flexible | Extra config surface area | Overkill for a single redundant call |
| 3 | Keep current behavior with logging | No change needed | Leaves unnecessary overhead | Defeats the purpose of the ticket |
| 4 | Remove validation entirely | Simplest | Removes the defensive check | We need at least the early-exit validation |

### Files to Change
| File | Change | Reason |
|------|--------|--------|
| `src/lightspeed_evaluation/pipeline/evaluation/pipeline.py` | Remove `validate_data()` call from `run_evaluation()` | Eliminates duplicate validation |

### Dependencies & Side Effects
- [ ] Public API change? — No (internal behavior change only)
- [ ] Config / env var change? — No
- [ ] Database migration? — No
- [ ] Downstream consumer impact? — No (data is validated before reaching pipeline)
- [ ] Error handling / logging change? — Yes, removes a logging line ("Validating data")
- [ ] Performance characteristics change? — Slight improvement (one fewer validation pass over evaluation data)

### Risk Assessment
| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Pipeline receives unvalidated data if used directly | Low | Medium | Data models still have Pydantic validators; existing tests cover validation path |
| Test breakage | Low | Low | Test mocks validate_evaluation_data; removing the call from run_evaluation is straightforward |

### Test Strategy
- Existing tests: `tests/unit/pipeline/evaluation/test_pipeline.py` has mocks for `validate_data()`. After the change, the mock is not needed for `run_evaluation()` flow. Ensure tests still pass.
- New regression test: Add a test to `test_pipeline.py` verifying `run_evaluation()` does not call `data_validator.validate_evaluation_data()`.

### Confidence
| Dimension | Score | Proof |
|-----------|-------|-------|
| Root cause certainty | HIGH | Code path traced: runner calls `load_evaluation_data()` → `validate_evaluation_data()`, then pipeline calls `validate_data()` → `validate_evaluation_data()` again. Both methods do identical metric availability + requirements checks (validator.py lines 154-177). |
| Approach correctness | HIGH | Single-line removal; data already validated by Pydantic + DataValidator at load time. |
| Scope completeness | HIGH | Only one redundant call identified; no other duplicate validations in the flow. |

### Investigation Strategy
**Signals detected**: default (code improvement/cleanup — redundant validation)
**Strategy used**: Standard investigation
**Key findings from strategy**:
- `DataValidator.validate_evaluation_data()` is the core method being called twice
- Pydantic field validators handle format/dedup; `DataValidator` handles availability + requirements
- The call in `load_evaluation_data()` happens during the runner's initialization (step 2)
- The call in `run_evaluation()` happens right before processing (step 1)
- Removing the pipeline-level call is safe because data validation has already occurred
