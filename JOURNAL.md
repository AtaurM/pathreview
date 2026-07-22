## Week 7 — Issue selection

**Issue link:** https://github.com/ascherj/pathreview/issues/153

**Issue title:** Faithfulness checker crashes when a context chunk has text: None

**Tier:** [x] Tier 1  [ ] Tier 2  [ ] Tier 3

**Problem summary:**
The faithfulness evaluator in the RAG module crashes on a specific but valid
input shape. In `FaithfulnessChecker.check()`, context is assembled with
`chunk.get("text", "")`, but the `""` default only kicks in when the `text`
key is absent, meaning: if a chunk explicitly carries `text: None`, `.get()` returns
`None`, and the following `" ".join(...)` raises a `TypeError`. So instead of
scoring the feedback, the whole call blows up whenever any retrieved chunk has
a null text field (a realistic case for empty or failed extractions). A
successful fix coerces missing/None text to an empty string so those chunks are
simply skipped in the concatenation, making `check()` return a normal score.
It's verified by the existing failing test `test_none_context_chunk_text` in
`tests/unit/test_faithfulness_checker.py`.

**Branch name:** fix/153-faithfulness-none-context-text

**Setup confirmation:** [x] App runs locally at localhost:5173

**Cohort ledger:** [x] Issue added to cohort ledger

### "Is this right for me?" -- scope reasoning
Tier 1 fits my comfort level for a first contribution to a large codebase. The
defect is isolated to a single method in one file (`rag/evaluator/
faithfulness_checker.py`), the reproduction is a two-line snippet, and there's
already a named failing test pinning the expected behavior. The scope is
bounded and the definition of "done" is objective. No API, schema, or
cross-module changes are involved, which is why I chose it over a broader
Tier 2/3 issue while I'm still learning the repo layout.
