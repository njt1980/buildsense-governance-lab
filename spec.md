# Specification: Meta-Governance Cycle 1 — Secrets/Debug Lint, AGENTS.md↔Phase-Gate Coupling Check, Push-Freshness Guard

## 1. Goal Description

This cycle is not an application feature — it is tooling that closes three specific, evidence-backed gaps in how this repository verifies its own process is being followed, found during a governance review of the audit-remediation effort (Cycles 1-5) and its supporting scripts:

1. **No automated check for hardcoded secrets or stray debug statements.** AGENTS.md's Definition of Done lists "No hardcoded secrets, API keys, or unhandled debug logs (`print()`, `console.log()`)" as a completion criterion, but nothing mechanically checks it — it depends entirely on whoever is reviewing a commit remembering to look.
2. **AGENTS.md's own coupling note is unenforced.** `scripts/check_phase_gate.py`'s docstring states: "the commit-message prefixes this script greps for... are defined in AGENTS.md's MANDATORY_WORKFLOW. If those strings change there, update the patterns below in the same commit." Nothing verifies that pairing — the two literal commit-message strings in AGENTS.md and the `SPEC_GREP`/`DESIGN_GREP` constants in `check_phase_gate.py` could drift apart silently.
3. **No detection for a stalled remote.** This session found 76 commits — spanning all five audit-remediation cycles, including the fix to `reliability-phase1.yml` itself — sitting unpushed to `origin/master` for 5 days. During that window, every CI workflow in `.github/workflows/` ran zero times against this work, because none of them trigger on anything but push/PR. Nothing in the repo would have surfaced this on its own; it was found by manual inspection.

Explicitly out of scope for this cycle: branch protection settings on `master` (a GitHub repository setting, not source code — handled separately, outside this spec/design/code ritual, pending a decision on PR-based vs. direct-to-master workflow). Also out of scope: the deterministic proxy checks for LLM-judged prompt properties (Fourth-Wall leakage, zero-jargon) and the production-feedback/eval-fixture pipeline — both depend on this cycle's script-authoring pattern but are larger, separate efforts (Phases 3 and 5 of the governance improvement plan).

---

## 2. Functional Requirements

### 2.1 `scripts/check_secrets_and_debug_statements.py` (new)
- Pure-stdlib script, following `check_phase_gate.py`'s existing style (git-plumbing only, `--staged-files` override flag for CI use, independent of the `apps/api` virtualenv).
- Scope: only files matching `^apps/api/app/.*\.py$` or `^apps/web/src/.*\.(ts|tsx)$` (application source only — excludes `scripts/`, `apps/api/tests/`, `apps/api/evals/`, and `.env.example`, where `print()`/placeholder key names are legitimate).
- Debug-statement check: flags any matched file containing `print(` (Python) or `console.log(` (TypeScript/TSX).
- Secret check: flags any matched file containing (a) an AWS-style access key pattern (`AKIA[0-9A-Z]{16}`), (b) a private-key header (`-----BEGIN (RSA |EC |OPENSSH )?PRIVATE KEY-----`), or (c) an assignment of the form `(api[_-]?key|secret|token|password)\s*[:=]\s*["'][A-Za-z0-9+/=_-]{16,}["']` (case-insensitive) — i.e. a plausibly-real secret value, not an empty placeholder or a variable name alone.
- On any match, print the file, line number, and matched pattern category to stderr and exit 1. Exit 0 if no matched files or no findings.

### 2.2 AGENTS.md ↔ `check_phase_gate.py` coupling check
- Add a new check function inside `scripts/check_phase_gate.py` (not a separate script — this is specifically about that script's own documented coupling): `check_agents_md_coupling()`.
- Reads `AGENTS.md`, locates the two literal commit command strings in the Phase 1 and Phase 2 sections (`git commit --no-verify -m "docs: finalize specification"` and `git commit --no-verify -m "docs: finalize system design"`), and asserts their message text matches `SPEC_GREP`/`DESIGN_GREP` (with the `^` anchor stripped) exactly.
- Runs unconditionally in `main()`'s check list (not gated on staged files — a single-file text comparison, negligible cost), so drift is caught on every commit, not just ones touching phase-gate-relevant files.
- On mismatch, print both the AGENTS.md text and the script's constant so the discrepancy is immediately visible, and exit 1.

### 2.3 `.github/workflows/push-freshness.yml` (new)
- Triggers: `schedule` (daily cron) and `workflow_dispatch` (manual run).
- Checks out the repository (`fetch-depth: 0` not required — only the current default-branch HEAD commit's date is needed).
- Computes days elapsed since that commit's authored date; fails the job with a clear message if elapsed days exceed a threshold (default 3, set as a workflow-level env var so it's a one-line change to tune).
- No external notification integration in this cycle (e.g. Slack) — relies on the workflow run showing red in the Actions tab, plus GitHub's default behavior of emailing repository owners on scheduled-workflow failure, which requires no additional configuration.

### 2.4 Wiring
- `.githooks/pre-commit`: add a call to `scripts/check_secrets_and_debug_statements.py`, alongside the existing `check_phase_gate.py`/`sync_agent_rules.py --check` calls, before the pytest step.
- `.github/workflows/reliability-phase1.yml`: add a step running `check_secrets_and_debug_statements.py --staged-files $CHANGED`, using the same `$CHANGED` file list already computed for the existing `check_phase_gate.py` step.

---

## 3. Non-Functional Requirements

- All new scripts are pure stdlib (no new pip/npm dependency), consistent with `check_phase_gate.py`'s existing rationale (must run in the fast pre-commit path without a virtualenv).
- No new CI step may use `|| true` or otherwise swallow a non-zero exit code (per the defect this exact pattern caused earlier in `reliability-phase1.yml`).
- No application code (`apps/api/app/`, `apps/web/src/`) is modified in this cycle — tooling and CI/workflow files only.
- `check_secrets_and_debug_statements.py` must produce zero findings against the current, already-clean state of `apps/api/app/` and `apps/web/src/` (i.e., it must not be introduced already-failing).

---

## 4. Acceptance Criteria

1. `scripts/check_secrets_and_debug_statements.py` exists, exits 0 against the current codebase, and exits 1 against a deliberately-introduced test fixture containing a fake AWS-style key and a stray `print()`.
2. `check_phase_gate.py` contains `check_agents_md_coupling()`, wired into `main()`; it exits 0 against the current AGENTS.md/script pairing.
3. `.github/workflows/push-freshness.yml` exists, is valid workflow YAML, and its threshold logic is unit-testable independent of an actual 3-day wait (e.g. via a small pure-Python date-diff function covered by a local script test, not requiring a real stale commit to verify).
4. `.githooks/pre-commit` and `reliability-phase1.yml` both invoke the new secrets/debug check.
5. Existing `pytest apps/api/tests/ -v` and `mypy app/` are unaffected (no application code touched).
6. `python scripts/check_phase_gate.py` and `python scripts/sync_agent_rules.py --check`, run locally, both still exit 0.

---

## 5. Verification Plan

- `python scripts/check_secrets_and_debug_statements.py` (repo root) — exits 0 against current tree.
- Manual fixture test: create a scratch file with a fake `AKIA...` key and a `print(`, run the script with `--staged-files` pointing at it, confirm exit 1 and correct findings reported; discard the scratch file.
- `python scripts/check_phase_gate.py` (repo root) — exits 0, confirms the new coupling check passes.
- Manual: temporarily edit one character of the commit-message string in a local scratch copy of AGENTS.md, confirm `check_agents_md_coupling()` fails with a clear diff, then revert.
- `pytest apps/api/tests/ -v`, `mypy app/` (from `apps/api`) — confirm no regression.
- `python scripts/sync_agent_rules.py --check` (repo root).

---

## Lab Experiment: Feature B (Bob)

This section is added by simulated developer "Bob" as part of the
BuildSense governance-lab multi-developer experiment. It represents a
fictitious Phase 1 spec for a fictitious "Feature B", started concurrently
with a competing "Feature A" spec from simulated developer "Alice" on a
separate branch, to observe what happens when two ritual participants edit
spec.md at the same time.
