---
description: "Diagnose decision conflicts, duplicates, and superseded entries across all design documents. Helps maintain decision hygiene as the project evolves. Usage: /plan-cascade:memory-doctor"
---

# Memory Doctor — 决策健康诊断

You are running a full diagnosis on all Architecture Decision Records (ADRs) across the project's design documents.

## Step 1: Collect All Decisions

**CRITICAL**: Use Bash to run the memory doctor script in collect mode (no LLM required):

```bash
uv run python "${CLAUDE_PLUGIN_ROOT}/skills/hybrid-ralph/scripts/memory-doctor.py" \
  --mode collect \
  --project-root "$(pwd)"
```

This script collects all decisions from every `design_doc.json` in the project (root, worktrees, feature directories) and outputs them as structured JSON to stdout.

Exit code handling:
- **Exit 0**: Decisions collected successfully — parse the JSON output and proceed to Step 2
- **Exit 2** (or script crash/traceback): Infrastructure error (e.g., no design_doc.json files found). Display the error message and stop here.

Parse the JSON output. It has this structure:
```json
{
  "mode": "full",
  "total_decisions": <int>,
  "source_count": <int>,
  "existing_decisions": [...],
  "new_decisions": [],
  "sources": {"<filepath>": [<decision>, ...]}
}
```

If `total_decisions` < 2, display "Not enough decisions to compare (found N)" and stop here.

## Step 2: Analyze Decisions for Issues

**YOU perform the semantic analysis.** Read through all decisions from the JSON output and compare them to identify issues.

For each pair of decisions, check for these issue types:

### CONFLICT
Two decisions **contradict** each other on the **same topic**. They cannot both be true or followed simultaneously.
- Example: "Use REST for all APIs" vs "Use GraphQL for all APIs"
- NOT a conflict: Decisions about different topics (e.g., "Use REST" and "Use PostgreSQL")

### SUPERSEDED
A **newer** decision covers the same scope as an **older** one, effectively replacing it.
- Example: "Use MySQL" (older) → "Migrate to PostgreSQL" (newer)
- The newer decision must explicitly or implicitly replace the older one

### DUPLICATE
Two decisions say **essentially the same thing** in different words.
- Example: "Use JWT for authentication" vs "Authentication via JWT tokens"
- Must be semantically identical, not just related

**IMPORTANT**: Only report **genuine issues**. Decisions about different topics or compatible decisions should NOT be reported. Be conservative — when in doubt, do not flag it.

Format your findings as a JSON array:
```json
[
  {
    "type": "conflict|superseded|duplicate",
    "decision_a_id": "<id of older/first decision>",
    "decision_b_id": "<id of newer/second decision>",
    "decision_a": {"id": "...", "_source": "..."},
    "decision_b": {"id": "...", "_source": "..."},
    "source_a": "<filepath of decision_a>",
    "source_b": "<filepath of decision_b>",
    "explanation": "<brief explanation of the issue>",
    "suggestion": "<recommended action>"
  }
]
```

## Step 3: Display Results

If no issues found, display:
```
┌──────────────────────────────────────────────────────────────────┐
│  MEMORY DOCTOR — 决策诊断报告                                      │
├──────────────────────────────────────────────────────────────────┤
│  Scanned: N decisions from M sources | Found: 0 issues          │
└──────────────────────────────────────────────────────────────────┘

  ✅ No issues found. All decisions are healthy.
```
Stop here.

If issues found, display the report grouped by type:
- 🔴 **CONFLICT** (N): Contradictory decisions on the same concern
- 🟠 **SUPERSEDED** (N): A newer decision covers the scope of an older one
- 🟡 **DUPLICATE** (N): Semantically identical decisions with different wording

For each finding, show:
```
  ADR-XXX vs ADR-YYY:
    旧: "<title>" (<source_file>)
    新: "<title>" (<source_file>)
    说明: <explanation>
    建议: <suggestion>
```

## Step 4: Batched Interactive Resolution

**CRITICAL**: Group all diagnosis findings into batches of up to 4 and use one `AskUserQuestion` per batch. This avoids asking the user N separate questions.

**Option mapping by issue type:**
- **CONFLICT** findings: options = `["Deprecate (recommended)", "Skip"]`
- **SUPERSEDED** findings: options = `["Deprecate (recommended)", "Skip"]`
- **DUPLICATE** findings: options = `["Merge (recommended)", "Skip"]`

```
# Group findings into batches of up to 4
BATCH_SIZE = 4
batches = [findings[i:i+BATCH_SIZE] for i in range(0, len(findings), BATCH_SIZE)]

all_responses = []
For batch_num, batch in enumerate(batches):
    questions = []
    For idx, finding in enumerate(batch):
        global_idx = batch_num * BATCH_SIZE + idx + 1

        # Build options based on finding type
        If finding.type in ["conflict", "superseded"]:
            options = ["Deprecate (recommended)", "Skip"]
        Elif finding.type == "duplicate":
            options = ["Merge (recommended)", "Skip"]
        Else:
            options = ["Deprecate", "Merge", "Skip"]

        questions.append({
            "header": f"Issue #{global_idx}",
            "question": (
                f"[{finding.type.upper()}] {finding.decision_a.id} vs {finding.decision_b.id}\n"
                f"  旧: \"{finding.decision_a.title}\" ({finding.source_a})\n"
                f"  新: \"{finding.decision_b.title}\" ({finding.source_b})\n"
                f"  说明: {finding.explanation}\n"
                f"  建议: {finding.suggestion}"
            ),
            "options": options
        })

    AskUserQuestion(questions=questions)
    # Collect responses for this batch
    all_responses.extend(batch responses)

    If more batches remain:
        echo "--- Batch {batch_num+1}/{len(batches)} complete. Continuing... ---"
```

After all batches are answered, map responses to actions for Step 5:
- **"Deprecate (recommended)"** or **"Deprecate"** → action = `"deprecate"`
- **"Merge (recommended)"** or **"Merge"** → action = `"merge"`
- **"Skip"** → omit from the actions array entirely

## Step 5: Apply Changes

**CRITICAL**: Construct a JSON array of the user's choices and invoke the script to apply them.

Save the user's choices to a temporary file `_doctor_actions.json`:
```json
[
  {
    "action": "deprecate",
    "diagnosis": {
      "type": "conflict",
      "decision_a": {"id": "ADR-F003", "_source": "path/to/design_doc.json"},
      "decision_b": {"id": "ADR-F012", "_source": "other/design_doc.json"},
      "source_a": "path/to/design_doc.json",
      "source_b": "other/design_doc.json"
    }
  }
]
```

Then run:
```bash
uv run python "${CLAUDE_PLUGIN_ROOT}/skills/hybrid-ralph/scripts/memory-doctor.py" \
  --apply _doctor_actions.json \
  --project-root "$(pwd)"
```

Clean up the temporary file after execution:
```bash
rm -f _doctor_actions.json
```

Note: Entries where the user chose **Skip** should be omitted from the actions array entirely.

## Step 6: Summary

Display a summary of all actions taken:

```
Memory Doctor — 处理结果
━━━━━━━━━━━━━━━━━━━━━━━━
✓ Deprecated: N decisions
✓ Merged: N decision pairs
○ Skipped: N findings
```

If any design_doc.json files were modified, list them so the user knows which files changed.

## Step 7: Next Steps

After the diagnosis is complete:
- Review changed files with `git diff` to verify the modifications
- Commit the changes if satisfied
- Note: Decision conflict checks also run automatically during `/plan-cascade:hybrid-auto` and `/plan-cascade:mega-plan` when new design documents are generated
