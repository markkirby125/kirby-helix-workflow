# Bootstrapping Kirby Helix — Dispatcher

> Complete operational knowledge for bootstrapping the Helix agentic workflow. The root `SKILL.md` is a hollow dispatcher; execute from this file only.

## When to Use

Use this skill when the user asks to implement a strict AI coding workflow, set up deterministic gates, bootstrap a "Helix" workflow, or configure dual-tier agent memory for a new or existing project.

## Limitations (When NOT to Use)

- Do **NOT** use this skill if the user is explicitly requesting to bypass testing or safety gates.
- Do **NOT** use this skill on read-only systems or environments where you lack permission to write to `~/.agents/`.
- Do **NOT** use this skill to modify configuration files for alternative agent frameworks (e.g., AutoGPT, BabyAGI); it is specifically designed for environments that parse `AGENTS.md`.

## The Architecture (Context for the Agent)

This workflow is based on Shopify's internal "Helix" pipeline.

1. **Checkpoints**: Ordered task breakdown.
2. **Context Isolation**: Fresh contexts per subagent.
3. **The 4 Gates**: Tests, UI, Adversarial Review, Human-in-the-Loop.
4. **The Ralph Loop & Memory**: OS-level exit code traps and dual-tier `learnings.md` files to prevent repeated mistakes.

---

## Execution Steps

When invoked, you MUST execute the following installation phases sequentially. **Do not skip the pre-flight checks.**

### Phase 1: Pre-Flight & State Detection

1. Check if `~/.agents/AGENTS.md` exists.
2. **Scan**: If the file exists, search it for existing `<gates>` or `<rules>` tags, and record the scan result as one of: `fresh` (file absent), `tagless` (file present, no tags), or `tagged` (tags found).
3. **User Choice**: If the scan result is `tagged`, PAUSE and ask the user how to proceed. Present these exact options:
   - *1. Modular Install:* Pick which pillars to add (Gates only / Memory only).
   - *2. Selective Overwrite:* Review diffs and selectively replace blocks.
   - *3. Full Overwrite:* Wipe ONLY the existing `<gates>` and `<rules>` blocks and replace them, leaving the rest of `AGENTS.md` completely untouched.
   - *4. Abort:* Cancel installation.
   If the scan result is `tagless`, PAUSE and ask: "Append the Helix `<gates>` and `<rules>` blocks to your existing `AGENTS.md`, or abort?" — carry the answer into Phase 3. If the scan result is `fresh`, proceed: Phase 3 creates a new `~/.agents/AGENTS.md` containing both blocks.
4. **Backup**: If `~/.agents/AGENTS.md` exists, before making ANY modifications copy it to a unique, portable backup path — use `BACKUP_PATH="$HOME/.agents/AGENTS.md.bak.$(date +%Y%m%d-%H%M%S).$$"` (the PID suffix keeps names unique on BSD/macOS, where `date +%N` is not supported). Do NOT set `BACKUP_PATH` when the file does not exist yet — record instead that this is a fresh install. Add a cleanup step to retain only the 5 most recent backup files, run that cleanup ONLY after the install has passed validation on the success branch (never prune the current session's `$BACKUP_PATH`), and treat it as a no-op on a fresh install.

### Phase 2: Environment Checks

1. **Git Check**: Run `git rev-parse --is-inside-work-tree`. If it fails, ask the user if they want to initialize a git repo or skip `.gitignore` modification.
2. **Monorepo Check**: Scan the root directory. If multiple package roots are detected (e.g., multiple `package.json` files), ask the user if the local `learnings.md` should live at the absolute root or in specific packages.

### Phase 3: Config Injection & Validation

Write the requested changes to `AGENTS.md` based on the Phase 1 scan result and the user's choice. Write semantics per path:

- *Full Overwrite:* replace the existing `<gates>` and `<rules>` blocks wholesale with the payload below.
- *Modular Install:* write ONLY the selected pillar — `<gates>` for Gates, `<rules>` for Memory — replacing that block if present, inserting it if absent, and leaving the other block untouched.
- *Selective Overwrite:* show the user a per-block diff (current vs payload) and replace only the blocks they approve.
- *Tagless append:* insert both blocks without disturbing any existing content.
- *Fresh:* create `~/.agents/AGENTS.md` containing exactly both blocks.

In every path, ensure the following logic is embedded (use XML formatting):

```xml
<gates label="HARD STOPS | Priority: gates > rules | Missing=STOP">
GATE-1 ambiguity:
  trigger: 2+ conflicting interpretations for destructive or high-impact action
  action: ask exactly 1 targeted question | ban multi-question surveys

GATE-2 safety:
  trigger: secret-handling | irreversible-deletion | credential-exposure
  action: abort & request explicit confirmation

GATE-3 validation:
  trigger: code-modifications-completed
  action:
    1. Verify trust: If in an untrusted repo, ask for human permission before running tests.
    2. Execute tests: Run native test/lint commands explicitly (e.g., `npm run test`).
    3. Evaluate Exit Codes:
       - If Exit 127 (Command not found) OR stderr contains "no test specified": Skip gate. Ask user to install test runner.
       - If Exit != 0 (Tests failed): TRAP. Read stderr, fix code, retry.
    4. Escape Hatch: Max 3 retries. If tests still fail, abort and request human intervention. Do NOT use `git commit` as a trap door.
</gates>

<rules label="EXECUTION & TOKEN ECONOMY">
INITIALIZATION & MEMORY:
  trigger: session-start
  action:
    1. Read `~/.agents/learnings.md` (Global) and local `learnings.md`.
    2. If missing, create them.

QA & CONTINUOUS LEARNING:
  trigger: receiving-human-feedback | test-rejection
  action:
    1. Extract root cause of failure.
    2. Safety Scan: Run a regex scan on the extracted rule for API keys, tokens, or proprietary paths. Refuse to write if secrets are detected.
    3. Context Cap: If the target `learnings.md` exceeds 200 lines, automatically archive older entries into `learnings.archive.md` to prevent token bloat and enable headless execution.
    4. Deduplicate: Grep for near-duplicates in the file. If similar, update existing; else, append. Use file locks or atomic sequential writes to prevent race conditions.
    5. Categorize: General lessons go to Global; Project quirks go to Local.
    6. Attempt code fix ONLY after memory is updated.
</rules>
```

**Post-Write Validation (Human-Gated Rollback):**

Parse `~/.agents/AGENTS.md` to ensure XML tags are perfectly balanced, and compare the result against the pre-install state (the `$BACKUP_PATH` copy if one exists, otherwise "file absent").

- If the FIRST parse is balanced → skip the gate: report success and continue directly to Phase 4.

Otherwise:

1. **STOP on failure** — do NOT restore or delete anything automatically. Overwriting or removing `~/.agents/AGENTS.md` is a user-config mutation and requires a confirmation gate.
2. Report the specific imbalance found (tag, line, offending snippet), and state whether the imbalance is new (caused by this install) or pre-existing in the backup. A diff against `$BACKUP_PATH` is the arbiter; if no backup exists, any imbalance is new.
3. Offer the user both remedies:
   - *Full rollback:* `cp "$BACKUP_PATH" ~/.agents/AGENTS.md` (only when a backup exists).
   - *Surgical revert:* remove only the `<gates>` / `<rules>` blocks this install wrote, preserving the rest of the file (preferred for Modular or Selective installs).
   - *Fresh install:* there is no backup to restore — the remedy is `rm ~/.agents/AGENTS.md` (removing the file this install created), never `cp`.
4. Apply the chosen remedy ONLY after the user confirms, then re-run the validation parse:
   - *Surgical revert* re-parses balanced → the surviving install is intact; report success and continue to Phase 4.
   - *Full rollback* or *fresh-install `rm`* restored the pre-install state (a restored-absent file counts as balanced) → the installation is ABORTED: report the final state, do NOT continue to Phase 4, and offer to re-attempt from Phase 1 if the user wants.
   - If the re-parse is STILL unbalanced → declare the installation FAILED as a terminal state: make no further writes, leave the user's file untouched, and hand the raw diff to the user for manual repair.
5. If the user declines all remedies, abort the install: make no further writes, and report which files (if any) were modified or backed up.

### Phase 4: Scaffolding Memory & Gitignore

1. **Global Memory:** `mkdir -p ~/.agents` and create `~/.agents/learnings.md` (if missing) with a basic markdown header.
2. **Local Memory:** Create `learnings.md` in the target project root (if missing).
3. **Idempotent Gitignore:** If inside a git repo, use `git check-ignore **/learnings.md`. If it is NOT ignored, cleanly append the recursive wildcard `**/learnings.md` to `.gitignore` to prevent any memory files or sub-package files from leaking to source control.

### Phase 5: Handoff & Observability

Output a final installation summary to the user:

- List exactly which files were Created, Modified, Skipped, or Backed Up.
- Provide the exact command to rollback the installation using the specific `$BACKUP_PATH` (only when a backup exists; for fresh installs, state that removal of the created file is the undo path).
- Remind them to use their workflow orchestrator (e.g., `/pickle-pipeline`) to begin coding.

---

## Examples

### Example 1: Handling Existing Configuration

```text
User: Please bootstrap the helix workflow for me.

Agent: I found an existing `~/.agents/AGENTS.md` file that already contains `<gates>` tags. How would you like to proceed?
1. Modular Install (Pick which pillars to add)
2. Selective Overwrite (Review diffs and selectively replace blocks)
3. Full Overwrite (Wipe ONLY the existing <gates> and <rules> blocks and replace them)
4. Abort

User: 3

Agent: Understood. Creating backup at `~/.agents/AGENTS.md.bak.20260928-214530.4021`...
```

### Example 2: Monorepo Detection

```text
Agent: I detect multiple `package.json` files in this repository (e.g., `frontend/package.json` and `backend/package.json`).
Should I create the local `learnings.md` file at the absolute root of the repository, or in each individual package directory?

User: Just the absolute root.

Agent: Acknowledged. Proceeding with scaffolding `learnings.md` at the repository root and appending `**/learnings.md` to `.gitignore`.
```

### Example 3: Corrupted Merge Rollback (Human Gate)

```text
Agent: Post-write validation failed — the `<gates>` block in `~/.agents/AGENTS.md` has an unbalanced closing tag, and the diff against the backup confirms this install introduced it.

Proposed rollback: `cp "$BACKUP_PATH" ~/.agents/AGENTS.md`

Shall I restore the backup?

User: Yes.

Agent: Backup restored and re-validated. `~/.agents/AGENTS.md` is back to its pre-install state with balanced tags. No changes were applied. Installation aborted.
```
