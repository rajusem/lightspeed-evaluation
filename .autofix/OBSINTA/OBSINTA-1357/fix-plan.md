## Fix Plan for OBSINTA-1357

### Version
Plan v1 | Iteration 0 (initial draft)

### Root Cause
The `EvaluationResult` Pydantic model in `src/lightspeed_evaluation/core/models/data.py` defines the field as `metrics_metadata` (plural, with 's' on line 502), but the CSV column name used everywhere else is `metric_metadata` (singular, without 's'):
- `src/lightspeed_evaluation/core/constants.py:88` — `"metric_metadata"` in `SUPPORTED_CSV_COLUMNS`
- `config/system.yaml:194` — `"metric_metadata"` in `csv_columns`
- `docs/configuration.md:236` — documents `metric_metadata`

The CSV generator in `src/lightspeed_evaluation/core/output/generator.py:194` uses `hasattr(result, column)` to check for each column name on the `EvaluationResult` object. Since the column name is `"metric_metadata"` but the model field is `"metrics_metadata"`, `hasattr` returns `False` and the generator outputs an empty string `""` for every row. This causes the `metric_metadata` column to always be empty in the CSV output.

### Approach
Rename the `metrics_metadata` field in the `EvaluationResult` model to `metric_metadata` (singular) to match the canonical name used in the CSV column constant, system configuration, and documentation. Update all references that construct `EvaluationResult` instances to use the new field name.

This is the correct direction because:
1. The singular form `metric_metadata` is used in 3 locations (constants, config, docs) — it is the canonical name
2. The plural form `metrics_metadata` is only used in the model definition and 2 constructor call sites
3. The other `*_metrics_metadata` fields (e.g., `turn_metrics_metadata`, `conversation_metrics_metadata`) are different — they are configuration fields that hold metadata for multiple metrics, whereas `metric_metadata` on `EvaluationResult` holds metadata for a single metric result

### Alternatives Considered
| # | Approach | Pros | Cons | Why Not |
|---|----------|------|------|---------|
| 1 | Rename CSV column constant to `metrics_metadata` | Fewer source changes (1 file) | Breaks existing CSV consumers, requires config/docs updates, singular form is semantically correct for a single result's metadata | Would be a breaking change for users who parse CSV by column name |
| 2 | Add alias in Pydantic model (`Field(alias="metric_metadata")`) | No rename needed | Adds complexity, Pydantic aliases affect serialization not attribute access, `hasattr` still wouldn't work with the alias | Aliases don't solve the `hasattr`/`getattr` lookup |

### Files to Change
| File | Change | Reason |
|------|--------|--------|
| `src/lightspeed_evaluation/core/models/data.py` | Rename field `metrics_metadata` → `metric_metadata` on line 502 | Fix the field name to match CSV column constant |
| `src/lightspeed_evaluation/pipeline/evaluation/evaluator.py` | Update keyword argument `metrics_metadata=` → `metric_metadata=` on lines 172 and 511 | Update constructor calls to use the renamed field |
| `tests/unit/pipeline/evaluation/conftest.py` | No change needed | The conftest doesn't reference the `metrics_metadata` field on `EvaluationResult` |

### Dependencies & Side Effects
- [ ] Public API change? — Minor: field name on `EvaluationResult` model changes. However, this model is internal to the evaluation pipeline and not part of a public-facing API.
- [ ] Config / env var change? — No
- [ ] Database migration? — No
- [ ] Downstream consumer impact? — No. The field was always empty in CSV output due to this bug, so no consumers could depend on the old (broken) behavior.
- [ ] Error handling / logging change? — No
- [ ] Performance characteristics change? — No

### Risk Assessment
| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Field rename breaks serialization | Low | Low | `EvaluationResult` has `extra="forbid"` — any misspelled field would raise a validation error immediately |
| Missed reference to old field name | Low | Low | grep for `metrics_metadata` across entire codebase to catch all references |

### Test Strategy
- Existing tests to verify: `tests/unit/pipeline/evaluation/` — existing evaluator tests construct `EvaluationResult` objects
- Existing tests: `tests/unit/core/metrics/test_manager.py` — tests for `get_metric_metadata` (unaffected, different function)
- New regression test: Add a test in `tests/unit/core/output/` that verifies the `metric_metadata` column in CSV output is populated when `EvaluationResult` has a non-None `metric_metadata` value

### Confidence
| Dimension | Score | Proof |
|-----------|-------|-------|
| Root cause certainty | HIGH | Direct evidence: `data.py:502` has `metrics_metadata`, `constants.py:88` has `"metric_metadata"`, `generator.py:194` uses `hasattr(result, column)` which fails due to the mismatch |
| Approach correctness | HIGH | Renaming to match the canonical CSV column name is the simplest fix; only 2 constructor sites need updating |
| Scope completeness | HIGH | grep confirms only 3 source files reference `metrics_metadata` on `EvaluationResult` (model def + 2 constructor calls) |

### Investigation Strategy
**Signals detected**: default
**Strategy used**: Standard investigation (grep, file reads, code path tracing)
**Key findings from strategy**:
- The `EvaluationResult` model field `metrics_metadata` (plural) was introduced with a typo — an extra 's' — that doesn't match the CSV column name `metric_metadata` (singular) used in constants, config, and documentation
- The CSV generator dynamically maps column names to model attributes via `hasattr`/`getattr`, making the name mismatch a silent failure (empty string output instead of an error)
