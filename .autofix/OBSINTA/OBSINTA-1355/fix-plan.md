## Fix Plan for OBSINTA-1355

### Version
Plan v1 | Iteration 0 (initial draft)

### Root Cause
The `threshold` column is missing from the `SUPPORTED_CSV_COLUMNS` constant in `src/lightspeed_evaluation/core/constants.py`. While the `EvaluationResult` model correctly defines a `threshold: Optional[float]` field and the evaluator populates it on every result, the CSV generator's fallback column list never includes `"threshold"`. As a result, the CSV generator never queries the threshold field from results, producing empty `threshold` values in the output.

Evidence:
- `src/lightspeed_evaluation/core/constants.py:78-104` — `SUPPORTED_CSV_COLUMNS` has 26 columns but does NOT include `"threshold"`. It includes: `conversation_group_id`, `tag`, `turn_id`, `metric_identifier`, `result`, `score`, (missing: threshold), `reason`, `query`, `response`, `execution_time`, `api_input_tokens`, `api_output_tokens`, `judge_llm_input_tokens`, `judge_llm_output_tokens`, streaming metrics, `tool_calls`, `contexts`, `expected_response`, `expected_intent`, `expected_keywords`, `expected_tool_calls`.
- `src/lightspeed_evaluation/core/models/data.py:414` — `EvaluationResult.threshold` exists as `Optional[float]` with `ge=0.0, le=1.0`.
- `src/lightspeed_evaluation/pipeline/evaluation/evaluator.py:152-155` — threshold is correctly set via `self.metric_manager.get_effective_threshold()`.
- `src/lightspeed_evaluation/core/output/generator.py:190-199` — CSV writer iterates columns from the config and calls `getattr(result, column)`. If the column isn't in the list, it's never written.

### Approach
Add `"threshold"` to the `SUPPORTED_CSV_COLUMNS` list in `src/lightspeed_evaluation/core/constants.py`. This is the simplest fix — a single line addition — that resolves the mismatch between the model field and the CSV column configuration.

The `threshold` field is already correctly defined and populated. The fix only needs to ensure the CSV generator knows about it.

No changes needed to:
- `EvaluationResult` model — threshold field already exists
- `Evaluator` — already populates threshold on every result
- `OutputHandler._generate_csv_report` — already uses `getattr` properly (works for any column in the list)
- `system.yaml` config — `csv_columns` already includes `"threshold"`, so custom configs work; this fix addresses the default/fallback path

### Alternatives Considered
| # | Approach | Pros | Cons | Why Not |
|---|----------|------|------|---------|
| 1 | Add `"threshold"` to `SUPPORTED_CSV_COLUMNS` | Minimal change (1 line), fixes root cause, aligns with all 5 data flows | None | **Chosen approach** |
| 2 | Remove `threshold` from model/generator | Hides the bug, loses data | Would break existing configs that expect threshold | Reject — destroys functionality |

### Files to Change
| File | Change | Reason |
|------|--------|--------|
| `src/lightspeed_evaluation/core/constants.py` | Add `"threshold"` to `SUPPORTED_CSV_COLUMNS` list (after `"score"` on line 85) | Missing column breaks CSV output |

### Dependencies & Side Effects
- [ ] Public API change? Yes — new column in default CSV output, but backward compatible (adds a column, doesn't remove or reorder existing ones)
- [ ] Config / env var change? No
- [ ] Database migration? No
- [ ] Downstream consumer impact? CSV consumers will now get a populated `threshold` column instead of empty cells
- [ ] Error handling / logging change? No
- [ ] Performance characteristics change? Negligible — one additional column in CSV output

### Risk Assessment
| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Breaking existing consumers who expect N columns | Medium | Medium | Adding a column doesn't break field-access by name in pandas; positions shifted only if using index-based column access (rare) |
| Duplicate threshold in CSV if config also sets it | Low | Low | The system merges default + config columns; if threshold appears twice, csv.writer will write empty duplicate |

### Test Strategy
- Existing tests to verify: `tests/unit/core/output/test_generator.py` — test CSV output includes threshold column
- New regression test: Add assertion that all `EvaluationResult` fields referenced in `SUPPORTED_CSV_COLUMNS` exist on the model, or directly assert `threshold` appears in default CSV headers

### Confidence
| Dimension | Score | Proof |
|-----------|-------|-------|
| Root cause certainty | HIGH | Direct trace: SUPPORTED_CSV_COLUMNS list → generator column iteration → getattr fails silently → empty cell |
| Approach correctness | HIGH | Single-line addition to constants, minimal change aligns constants with model definition |
| Scope completeness | HIGH | Only one missing field; all other columns in SUPPORTED_CSV_COLUMNS match EvaluationResult fields |

### Investigation Strategy
**Signals detected**: regression (default)
**Strategy used**: default (standard grep/code tracing)
**Key findings from strategy**:
- Verified all code paths: model defines threshold, evaluator sets it, generator reads it — only the constant list is missing it
- Confirmed no other missing fields by cross-referencing SUPPORTED_CSV_COLUMNS against EvaluationResult.model_fields
