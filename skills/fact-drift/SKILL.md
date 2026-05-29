---
name: fact-drift
description: A Claude Code skill that checks whether facts you cite from pages you don't control (prices, versions, limits, rules, deadlines) are still current. Reads a JSON config of targets, fetches each source URL via parallel subagents using WebFetch, extracts the configured value, diffs against a stored baseline, writes a snapshot report, and on your approval updates every file that still cites an outdated value. Invoked as `/fact-drift`, `/fact-drift --update-baseline`, or `/fact-drift --apply`. Runs on the Claude Code subscription, no API key. Use when checking whether cited numbers, thresholds, quotas, prices, or policy dates are still current, and fixing them across your files when they are not.
---

# fact-drift

Verifies that facts cited in your content still match what the source page says. Catches silent upstream changes that would otherwise rot your docs.

This skill is the cheap path: it uses parallel subagents calling WebFetch on the Claude Code subscription. It does not spawn a browser runner or any API-billed tool. Reserve a real browser for targets that require JavaScript rendering or login.

## Inputs

- **Config**: `rules.config.json` in your project root. A list of targets, each with `id`, `name`, `url`, `extract` (a natural-language extraction instruction), `doc_slug`, and `notes`. See `examples/rules.config.example.json`.
- **Baselines**: `fact-drift-baseline/{target_id}.json`. The last captured value per target. Created by the first `--update-baseline` run.

## Outputs

- **Report**: `fact-drift-snapshots/snapshot-YYYY-MM-DD.md`. A status table plus drift detail.
- **Updated baselines**: only when `--update-baseline` is passed, or after an approved apply.
- **Edited content files**: only after you approve an apply step. The skill never writes to your content without confirmation.

## Modes

### `/fact-drift` (default)
Diff mode. For each target: fetch, extract, compare to baseline, classify as OK / DRIFT / NEW / ERROR. Write the snapshot report. If anything drifted, offer to apply the fixes (see the Apply step).

### `/fact-drift --apply`
Diff, then fix. For each DRIFT, find every file in the project that still contains the old value, show the planned edits, and after you confirm, update them all and refresh the baseline. Never edits without showing the plan first.

### `/fact-drift --update-baseline`
Capture mode. Fetch every target and write the current value as the new baseline. No diff. Use on the first run, and after you confirm a drift is the new normal.

### `/fact-drift --target <id>`
Single-target mode. Same as default, restricted to one id.

## Execution flow

1. Resolve today's date (`date +%Y-%m-%d`).
2. Read `rules.config.json`. Filter to `--target <id>` if provided.
3. Confirm `fact-drift-baseline/` exists; create it if not.
4. Spawn one subagent per target in parallel (single message, multiple Agent calls, `subagent_type: general-purpose`). Each subagent receives the target's `url` and `extract` instruction, plus the rules below.
5. Aggregate subagent results.
6. For each result:
   - If `--update-baseline`: write `fact-drift-baseline/{id}.json` as `{ value, source_url, captured_at }`.
   - Else: read the existing baseline and classify:
     - No baseline file -> `NEW`
     - Baseline value matches -> `OK`
     - Baseline value differs -> `DRIFT`
     - Subagent returned `NOT_FOUND` or errored -> `ERROR`
7. Write the snapshot report:

```markdown
# fact-drift snapshot, {today}

| Target | Status | Current | Baseline | Source |
| ------ | ------ | ------- | -------- | ------ |
| {id}   | OK     | 60      | 60       | [source]({url}) |
| {id}   | DRIFT  | 80      | 60       | [source]({url}) |

## Drift detail

### {id}
- Notes: {target.notes}
- Doc affected: {target.doc_slug}
- Diff: was 60, now 80
- Recommended action: run `--apply` to update every file that cites this value.
```

8. **Apply** (on `--apply`, or when the user confirms after a default run). For each DRIFT:
   - Search the project for files containing the old value. Start near the target's `doc_slug`, then widen if needed. Use grep.
   - Show the user every match with file and line context, and the proposed change (`old -> new`).
   - Ask for confirmation. Let the user deselect any match: a raw value like `60` can appear in unrelated places, and only the user knows which should change.
   - On confirmation, edit each approved match, then refresh that target's baseline to the new value.
   - Report which files changed. Do not git commit.
9. Print a one-screen summary: X of Y targets OK, any DRIFT items with the doc each affects, what was edited if anything, and the path to the full snapshot.

## Subagent prompt template

Use this as the body of each parallel `Agent` call. Substitute `{url}` and `{extract}` from the target's config.

> You are checking a single source page. Use WebFetch to retrieve {url}. Read the returned content carefully and answer this question, returning ONLY the value with a one-line provenance note:
>
> {extract}
>
> If the value is found, return:
> `VALUE: <value>`
> `PROVENANCE: <one line describing where on the page you found it>`
>
> If you cannot find it after reading the page in full, return:
> `VALUE: NOT_FOUND`
> `PROVENANCE: <one line on what you did find, or why the page lacked it>`
>
> Do not call any other tool. Do not search the web. Do not summarize the rest of the page. Under 80 words total.

## What this skill does not do

- It does not edit anything without showing you the plan and getting a yes. You can deselect any individual change.
- It does not handle login-walled or JS-rendered pages. If WebFetch returns boilerplate ("Enable JavaScript to view this page"), report ERROR with that message.
- It does not commit anything. It writes the report and baseline files only.

## When the cheap path stops being enough

If a target consistently returns `ERROR: requires-JS`, the page genuinely needs a real browser. Options:
1. Find a static or printable version of the same data.
2. Build a one-off browser script for just that target, accepting the API cost.
3. Drop the target if the data is not worth the cost.

Document the decision in the target's `notes` field.
