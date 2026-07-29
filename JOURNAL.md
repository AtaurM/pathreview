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

## Reproduction (Week 8) — issue #153 confirmed locally

Confirmed the crash reproduces reliably in my local environment on branch
`fix/153-faithfulness-none-context-text`. It is deterministic: it fails on
every run, with no setup beyond the standard local install.

### Steps to reproduce

Run the existing test that already pins this behavior (from the repo root):

```
python -m pytest tests/unit/test_faithfulness_checker.py -k none_context_chunk_text
```

Observed output:

```
>       context_text = " ".join([
            chunk.get("text", "") for chunk in context_chunks
        ])
E       TypeError: sequence item 0: expected str instance, NoneType found
rag\evaluator\faithfulness_checker.py:34: TypeError
```

The same crash reproduces in two lines without pytest:

```python
from rag.evaluator.faithfulness_checker import FaithfulnessChecker
FaithfulnessChecker().check("Has Python skills", [{"text": None}])
```

### Where the issue lives

`FaithfulnessChecker.check()` in `rag/evaluator/faithfulness_checker.py`,
lines 34-36. Context is assembled with `chunk.get("text", "")`, but `dict.get`
only falls back to the `""` default when the **key is absent**. A chunk that
carries the key with an explicit `None` value returns `None`, and the
enclosing `" ".join(...)` rejects it with a `TypeError`.

That the sibling test `test_missing_text_key_in_chunk` (chunk `{"content": ...}`
with no `text` key at all) **passes** confirms the default works for missing
keys and isolates an explicit `None` value as the sole trigger.

### Test baseline before any fix

`python -m pytest tests/unit/test_faithfulness_checker.py` → **4 failed, 18 passed**.

Only `test_none_context_chunk_text` is caused by this issue. The other three
failures (`test_partial_support_returns_middle_score`,
`test_multiple_context_chunks`, `test_multiple_claims_varying_support`) are
pre-existing and unrelated: they are scoring-threshold assertions, where
single-claim feedback scores exactly 0.0 or 1.0 and never lands in the
expected middle range. They are out of scope for #153. A correct fix should
therefore move the file to **3 failed, 19 passed**, not to all-green.

### Notes gathered while reproducing

1. **A sibling module crashes first on the same input.** Through the real
   entry point `EvalSuite.run()` (`rag/evaluator/eval_suite.py:28`),
   `RelevanceScorer.score()` is called on line 40, *before* the faithfulness
   check on line 43. `relevance_scorer.py:32` uses the identical
   `chunk.get("text", "")` pattern and dies earlier with
   `AttributeError: 'NoneType' object has no attribute 'lower'`. So fixing
   only the faithfulness checker satisfies issue #153 and its test, but the
   end-to-end evaluator still breaks on a `text: None` chunk. Recording this
   as a follow-up rather than expanding scope.

2. **`text: None` is reachable, not hypothetical.** `rag/retriever/hybrid.py:126`
   copies ChromaDB `documents` values straight into the `text` field with no
   coercion, so a stored-but-empty document propagates a null into exactly the
   chunk dicts the evaluator consumes.

## Week 8 — Reproduction & solution planning

**Reproduction commit link:** https://github.com/AtaurM/pathreview/commit/c7aaae6f8cf885a904a061113902fcbfd940dfbb

**Reproduction summary:**
I reproduced the crash locally by running the repo's existing test
`python -m pytest tests/unit/test_faithfulness_checker.py -k none_context_chunk_text`,
which fails deterministically with
`TypeError: sequence item 0: expected str instance, NoneType found` at
`rag/evaluator/faithfulness_checker.py:34` — the `" ".join(...)` over
`chunk.get("text", "")`, whose `""` default never fires for a key that is
present with a `None` value. I confirmed the same crash in a two-line snippet
without pytest, and confirmed that the sibling test
`test_missing_text_key_in_chunk` (a chunk with no `text` key at all) passes,
which isolates an explicit `None` as the sole trigger rather than a general
problem with the default.

**PLAN.md link:** https://github.com/AtaurM/pathreview/blob/fix/153-faithfulness-none-context-text/PLAN.md

**Blockers or open questions:**
1. **Which layer should hold the guard?** Fixing `FaithfulnessChecker` satisfies
   the issue as written, but the null actually enters at
   `rag/retriever/hybrid.py:126`, and a guard there would fix every consumer at
   once. I plan to ask on the issue thread before Week 9.
2. **A sibling module crashes first on the same input.** Through the real entry
   point `EvalSuite.run()`, `RelevanceScorer.score()`
   (`rag/evaluator/relevance_scorer.py:32`) raises
   `AttributeError: 'NoneType' object has no attribute 'lower'` *before* the
   faithfulness check ever runs. My fix will make issue #153's test pass while
   the end-to-end evaluator stays broken on that input. I intend to file this
   separately rather than widen a Tier 1 PR, but I want to confirm that is the
   preferred etiquette here.
3. **Non-string `text` values.** I have not yet traced whether the retriever can
   emit something like `42` in the `text` field, which decides between
   `chunk.get("text") or ""` and a `str()` coercion. This is sub-task 2 in
   PLAN.md.
