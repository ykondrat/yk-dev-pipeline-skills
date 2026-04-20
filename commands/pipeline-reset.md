---
description: Reset ai-tools pipeline state — archive current pipeline-state.json and artifacts so a fresh run can start.
argument-hint: "[confirm]"
allowed-tools: Read, Write, Bash, Glob
---

You are **resetting** the ai-tools pipeline run in the working directory.

## Confirmation

$ARGUMENTS

## What to do

1. **Read `pipeline-state.json`** if it exists. If missing, tell the user there is
   nothing to reset and exit.

2. **Always confirm before destructive action.** Unless `$ARGUMENTS` contains the
   literal string `confirm`, use AskUserQuestion to ask:

   > "This will archive `pipeline-state.json`, `spec.md`, `plan.md`, `review.md`,
   > `fix-plan.md`, `fix-test-plan.md`, and `test-report.md` to a timestamped
   > archive folder. Source code and tests under `src/` / `tests/` are left
   > untouched. Proceed?"

   Offer: `Archive and reset` / `Delete without archiving` / `Cancel`.

3. **On confirmation** — archive (do NOT delete by default):

   ```bash
   STAMP=$(date -u +%Y%m%dT%H%M%SZ)
   DIR=".pipeline-archive/${STAMP}"
   mkdir -p "$DIR"
   for f in pipeline-state.json spec.md plan.md design.md \
            review.md review-security.md review-quality.md review-compliance.md \
            fix-plan.md fix-test-plan.md test-report.md; do
     [ -f "$f" ] && mv "$f" "$DIR/"
   done
   ```

   Then report which files were moved and the archive path.

4. **Never touch** anything under `src/`, `tests/`, `.git/`, `node_modules/`, or
   any file not in the allowlist above. Never run `git reset --hard`, `git clean
   -fd`, or `rm -rf`. If the user asks for a harder reset, tell them to do it
   manually outside this command.

5. **After reset**, suggest `/pipeline-start` to begin a new run.

Resume-safety: this command is idempotent — running it twice in a row produces two
empty archive directories (safe, noisy).
