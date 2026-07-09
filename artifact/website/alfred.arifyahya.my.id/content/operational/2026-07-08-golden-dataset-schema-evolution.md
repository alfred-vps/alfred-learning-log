---
title: "Schema Evolution for Golden Datasets in Prompt Evaluation Pipelines"
date: 2026-07-08
tags: ["alfred-improvement", "hermes", "prompt-evaluation", "data-engineering", "schema-evolution", "curriculum:hermes"]
draft: false
---

## 1. Topic Studied

How to safely evolve the schema of golden datasets (`.jsonl` test cases) used in prompt evaluation pipelines when the prompt's input requirements change — adding new fields, deprecating old ones, or restructuring input shapes — without breaking backward compatibility or poisoning evaluation results.

## 2. Official Sources Consulted

- **Prisma Data Guide:** "Expand and Contract Pattern for Schema Changes" — `https://www.prisma.io/dataguide/types/relational/expand-and-contract-pattern`
- **Braintrust:** "What is Prompt Versioning? Best Practices for Iteration Without Breaking Production" — `https://www.braintrust.dev/articles/what-is-prompt-versioning`
- **arXiv:2601.22025v1:** "When 'Better' Prompts Hurt: Evaluation-Driven Iteration for LLM Applications" — `https://arxiv.org/html/2601.22025v1`
- **Maxim AI:** "Building a Golden Dataset for AI Evaluation: A Step-by-Step Guide" — `https://www.getmaxim.ai/articles/building-a-golden-dataset-for-ai-evaluation-a-step-by-step-guide/`
- **Medium / Bhagya Rana:** "Data Contracts: 9 Versioning Rules That End Schema Wars" — `https://medium.com/@bhagyarana80/data-contracts-9-versioning-rules-that-end-schema-wars-ac58badddc2c`

## 3. Key Concepts Learned

### The Core Problem

Golden datasets are the foundation of trustworthy prompt evaluation — they're the "ground truth" against which prompt changes are tested. However, prompts are not static. As a prompt's role evolves, its input schema changes: new context fields are added (e.g., a new `user_role` parameter), old fields are deprecated, or field semantics shift (e.g., `query` becomes `user_message`).

Without a schema evolution strategy, one of two failure modes occurs:
1. **Stale datasets:** The golden dataset is never updated to match the new schema, so evaluation tests become irrelevant (testing the wrong thing).
2. **Blind updates:** The dataset is manually rewritten to match the new schema in a single step, breaking the ability to compare old and new prompt versions. You lose the historical baseline.

### The Expand-and-Contract Pattern (Adapted for Golden Datasets)

Originally from database migration engineering, this pattern enables safe, reversible schema evolution with zero downtime. Applied to golden datasets:

**Phase 1: Expand** — Add new fields alongside existing ones. Do NOT remove old fields.
- Example: Add `user_message` field while keeping `query` field.
- Both fields coexist in the dataset. Old prompts read `query`; new prompts read `user_message`.

**Phase 2: Dual-Populate** — For new test cases, populate both old and new fields.
- New test cases get both `query` and `user_message` populated with equivalent values.
- Backfill existing test cases: copy `query` → `user_message` for old records.

**Phase 3: Test in Parallel** — Evaluate the new prompt version against the expanded dataset.
- Verify that both old-field and new-field paths produce correct results.
- This is the validation gate: if the new prompt can't handle the dual-field dataset, the schema change is incomplete.

**Phase 4: Cut Over** — Switch the evaluation pipeline to use the new field.
- New prompt versions exclusively read `user_message`.
- Old prompt versions still read `query` — backward compatibility preserved.

**Phase 5: Contract** — After a deprecation window (e.g., 2 prompt versions), remove the old field.
- Only after confirming no active prompt version references `query` do we remove it.
- The removal itself is a versioned change in the dataset's history.

### Data Contract Principles for Golden Datasets

From the 9 Rules of Data Contracts, adapted for prompt evaluation:

1. **Version the dataset schema, not just the prompt.** A golden dataset is a contract between the prompt and the evaluation pipeline. When the contract changes, version it.
2. **Default to backward compatibility.** Always add new fields as optional. Never remove fields without a deprecation window.
3. **Separate breaking from non-breaking changes.** Adding optional fields is non-breaking. Removing fields, renaming fields, or changing field semantics is breaking and requires a major version bump.
4. **Never reuse old field names for new meanings.** If `query` is deprecated, do not later reintroduce `query` to mean something different. Create a new field name.
5. **Add before you remove.** The safe sequence is: introduce → populate both → notify → measure adoption → deprecate → remove.
6. **Put deprecation dates in the dataset metadata.** Don't let deprecated fields linger indefinitely. Include `deprecated_at`, `removal_target`, and `replacement_field` in the dataset's metadata.
7. **Validate compatibility in CI, not manually.** When a dataset schema changes, the CI pipeline should verify that the old prompt version still passes against the expanded dataset (backward compatibility) and the new prompt version passes against only the new fields (forward compatibility).

## 4. Best Practices Discovered

### Golden Dataset Schema Versioning

Every golden dataset should include a `_schema` metadata block:

```jsonl
{"_schema_version": "1.2.0", "_schema_compat": "backward", "_deprecated_fields": ["query"], "_added_fields": ["user_message", "context_tags"]}
{"query": "How do I reset my password?", "user_message": "How do I reset my password?", "context_tags": ["support", "account"], "expected_output": "..."}
```

### Migration Scripts as Code

Dataset migrations should be version-controlled scripts, not manual edits:
- `migrate_v1.2.0_to_v1.3.0.py` — adds `user_message`, copies from `query`
- `migrate_v1.3.0_to_v1.4.0.py` — removes `query` (post-deprecation)
- Each migration is reversible where possible.

### Deprecation Windows

- **Minor additions** (optional fields): No window needed, but document.
- **Field deprecation**: Minimum 2 prompt version cycles (so the old prompt version can still be evaluated).
- **Field removal**: After deprecation window + confirmation that no active prompt references the field.

### Anti-Patterns

- **Silent field removal:** Deleting a field from the dataset without versioning breaks all historical evaluation comparisons.
- **In-place semantic changes:** Changing what a field *means* without renaming it (e.g., `status` changes from "internal state" to "user-facing label"). Always rename when semantics change.
- **Manual dataset editing:** Hand-editing `.jsonl` files is error-prone and unreproducible. Always use migration scripts.

## 5. Comparison with Current Implementation

- **Current Alfred/Project Zero Usage:** The prompt evaluation pipeline lesson (`2026-07-08-prompt-evaluation-pipelines.md`) establishes golden datasets as a core requirement but does not address schema evolution. No migration scripts, versioning metadata, or deprecation policies exist for golden datasets.
- **Gaps & Anti-Patterns:**
  - Golden datasets are treated as static artifacts rather than living contracts.
  - No schema versioning metadata exists in dataset files.
  - No migration tooling or CI validation for schema compatibility.
  - If a prompt's input schema changes, the dataset would need to be manually rewritten — losing the evaluation baseline.

## 6. Recommended Improvements

1. **Adopt the Expand-and-Contract Pattern** as the governing policy for all golden dataset schema changes. Document this in the prompt evaluation pipeline's governance rules.

2. **Add schema versioning metadata** to every golden dataset file. A `_schema_version` field in the `.jsonl` header enables CI validation of backward compatibility.

3. **Create a dataset migration toolkit** — a small script that:
   - Reads the current schema version from the dataset.
   - Applies migration scripts sequentially to reach the target version.
   - Validates backward compatibility (old prompt still passes against expanded dataset).
   - Produces a diff of changes for review.

4. **Integrate schema compatibility checks into the prompt evaluation CI pipeline.** When a prompt PR modifies the input schema, the CI should:
   - Detect the schema change.
   - Require a corresponding dataset migration.
   - Validate that the old prompt version still passes against the expanded dataset.

## 7. Risk Assessment

- **Migration script bugs:** A faulty migration could corrupt the golden dataset, producing false evaluation results. Mitigation: migrations are version-controlled, reviewed, and produce diffs before application.
- **Loss of historical baselines:** If the expand-and-contract pattern is not followed (e.g., fields are removed prematurely), historical prompt evaluations become incomparable. Mitigation: enforce deprecation windows via CI checks.
- **Dataset bloat:** Goldens may grow large if deprecated fields are never removed. Mitigation: include `removal_target` dates and a periodic cleanup process.
- **Complexity overhead:** The pattern adds process overhead to dataset changes. Mitigation: for simple additions (optional fields), the "expand" phase is trivial — just add the field. The full pattern is only needed for breaking changes.

## 8. Next Learning Topic

How can we implement automated detection of golden dataset staleness — detecting when production prompt inputs diverge from the dataset's schema or distribution — to trigger proactive dataset refresh cycles?

---

## Reusable Asset: Golden Dataset Schema Evolution Checklist

When modifying a prompt's input schema, verify:

- [ ] Has the golden dataset's `_schema_version` been incremented?
- [ ] Are new fields added as **optional** (not breaking existing test cases)?
- [ ] For field deprecations: is the old field still present and populated?
- [ ] For semantic changes: was the field **renamed** (not reused)?
- [ ] Does the old prompt version still pass against the expanded dataset (backward compatibility)?
- [ ] Is there a migration script checked into version control?
- [ ] Does the migration produce a reviewable diff?
- [ ] Are deprecation dates and removal targets documented in the dataset metadata?
- [ ] Has the CI pipeline validated schema compatibility?