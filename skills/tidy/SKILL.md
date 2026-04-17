---
name: tidy
version: 1.0.0
description: |
  Janitor mode. Purge regenerable artifacts (node_modules, .next, .turbo, dist, build, out, .expo, caches) from stale projects. Target one repo or scan all subfolders. Never touches source, .env*, configs, or .git.
allowed-tools:
  - Bash
  - Read
  - AskUserQuestion
---

# Tidy: Purge Regenerable Artifacts

You are running `/tidy`. You are the janitor — reclaim disk by deleting regenerable build/dep artifacts from projects the user hasn't touched recently. **Never delete source code or user work.**

## Argument parsing

Parse the user's `/tidy` argument — it may be empty, a path, a number of days, or a mix:

- **No args** → scan every direct subfolder of the current working directory (scope = "all").
- **A path** (e.g. `./myrepo`, `piggy`, `/abs/path`) → scope = that single project.
- **A days value** (e.g. `14`, `14d`, `--days 30`) → override the staleness threshold (default: **7 days**).
- **`--all`** or **`all`** → ignore staleness, purge everything matching in scope.
- **`--dry-run`** or **`dry`** → plan only, no deletion, no confirmation prompt.

Examples:
- `/tidy` → all subfolders, ≥7 days stale
- `/tidy piggy` → just `piggy/`, regardless of staleness (single repo implies user intent)
- `/tidy 14d` → all subfolders, ≥14 days stale
- `/tidy ./othr-v1 --all` → purge `othr-v1` caches without staleness check
- `/tidy --dry-run` → show what would be purged, delete nothing

When the user passes a single repo path, default to purging that repo without the staleness check — the explicit path is the user's intent signal. Still offer dry-run semantics.

## What to delete

**Dependency directories:**
- `node_modules`

**Build/cache directories** (all regenerable):
- `.next`, `.turbo`, `.expo`, `.nuxt`, `.svelte-kit`, `.parcel-cache`, `.vite`, `.cache`
- `dist`, `build`, `out`, `coverage`
- `artifacts`, `cache` (hardhat/foundry — only when found alongside `foundry.toml` or `hardhat.config.*`)
- `__pycache__`, `.pytest_cache`, `.mypy_cache`, `.ruff_cache` (Python)
- `target` (Rust — only when `Cargo.toml` is present in the same dir)

**Search depth:** up to 4 levels deep from each project root. This catches monorepo packages (`apps/*/node_modules`, `packages/*/dist`) without descending into nested dep trees.

## What to NEVER touch

- Any source directory: `src`, `app`, `lib`, `components`, `pages`, `contracts/src`, `test`, `tests`, `spec`, etc.
- `.env`, `.env.*`, any dotfile that isn't a known cache pattern above
- `.git`, `.github`, `.vscode`, `.idea`, `.claude`, `.opencode` config (but `.opencode/node_modules` inside them IS fair game)
- `docs`, `public`, `assets` (user-authored content)
- `package.json`, `package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`, `bun.lockb`, `Cargo.lock`
- Config files: `*.config.*`, `tsconfig*.json`, `.eslintrc*`, etc.
- Any `out/` that does NOT sit next to `next.config.*` or `foundry.toml` — when ambiguous, skip it and note in the report.

## Staleness check

A project is "stale" if its latest source-file mtime is ≥ threshold days ago. Compute it by finding the most recently modified file in the project, excluding regenerable dirs (so `npm install` timestamps don't reset the clock).

```bash
find "$proj" -type f \
  \( -path "*/node_modules" -o -path "*/.next" -o -path "*/.git" \
     -o -path "*/dist" -o -path "*/build" -o -path "*/.turbo" \
     -o -path "*/.expo" -o -path "*/.cache" -o -path "*/out" \
     -o -path "*/coverage" -o -path "*/__pycache__" \) -prune \
  -o -type f -print 2>/dev/null \
  | xargs stat -f "%m %N" 2>/dev/null | sort -rn | head -1
```

Age in days: `(( ($(date +%s) - <mtime>) / 86400 ))`.

If a project has no files outside the pruned dirs (rare), fall back to `stat -f %m` on the project directory itself.

## Execution flow

1. **Start Amphetamine** (per user's global rule):
   ```bash
   osascript -e 'tell application "Amphetamine" to start new session'
   ```

2. **Discover projects in scope.**
   - Single path → `projects=("$path")`.
   - All subfolders → list direct children of cwd that contain any of: `package.json`, `Cargo.toml`, `foundry.toml`, `hardhat.config.*`, `pyproject.toml`, or already have a `node_modules`.

3. **For each project: compute staleness** (skip if `--all` or single-path scope).

4. **For each stale (or in-scope) project: enumerate deletion candidates** (node_modules first, then caches). Compute `du -sk` per candidate.

5. **Print a plan table**:
   ```
   Age   Size     Project                  Targets
   45d   1.2 GB   ./corners-fe             node_modules, .next, .turbo
   ...
   ─────────────────────────────────────────────────
   TOTAL: X GB across N dirs in M projects
   ```
   Also list projects skipped as fresh (age < threshold) so the user sees what was preserved.

6. **Confirmation:**
   - If `--dry-run` → stop here, report only.
   - Otherwise → ask the user to confirm with a single yes/no (`AskUserQuestion` or inline prompt). Only proceed on explicit yes.

7. **Record disk before**: `df -h / | tail -1 | awk '{print $4}'`.

8. **Delete** each target with `rm -rf`. Log every removed path. Keep going on individual failures but report them at the end.

9. **Record disk after** and print a summary:
   ```
   Freed ~X GB (before: Y free → after: Z free)
   Removed N node_modules and M build/cache dirs across K projects
   Kept: <list of fresh projects>
   ```

10. **End Amphetamine**:
    ```bash
    osascript -e 'tell application "Amphetamine" to end session'
    ```

## Safety rules

- **Never delete a path whose basename doesn't match the allow-list above.** No heuristic expansion.
- **Never descend into `.git`.** Prune it from all finds.
- **Never touch a project if it has uncommitted work outside regenerable dirs** — actually, don't check git state. Regenerable artifacts are safe to delete regardless. Just don't widen the allow-list.
- **Never use `find -delete`** or `rm -rf $VAR/*` with an unvalidated `$VAR`. Always iterate explicit paths from a `find ... -print` loop.
- When a candidate path looks ambiguous (e.g. a `build/` that might contain a user's webpack config written by hand), skip it and note in the report. Err on the side of preserving.
- If the scope is a single explicit path and that path doesn't exist, abort with a clear error.

## Output style

- Be terse. One line per removed dir is fine; don't repeat the plan after execution.
- Show the final summary prominently — that's what the user cares about.
- If nothing is stale, say so in one sentence and exit. Don't delete anything.
