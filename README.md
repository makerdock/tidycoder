# tidy

A Claude Code skill that plays janitor — purges regenerable build artifacts (`node_modules`, `.next`, `.turbo`, `dist`, `build`, `out`, `.expo`, framework caches, Python caches, Rust `target`) from projects you haven't touched recently.

Never touches source code, `.env*`, lockfiles, configs, or `.git`.

## Install

```bash
/plugin install makerdock/tidy
```

Or clone and symlink for local development:

```bash
git clone https://github.com/makerdock/tidy ~/Documents/code/tidy
ln -s ~/Documents/code/tidy/skills/tidy ~/.claude/skills/tidy
```

## Usage

Run `/tidy` inside a parent folder that contains one or more project subfolders.

| Command | Behavior |
| --- | --- |
| `/tidy` | Scan every subfolder of cwd, purge projects stale for ≥7 days |
| `/tidy piggy` | Target a single repo (skips staleness check — explicit path = intent) |
| `/tidy 14d` | Custom staleness threshold |
| `/tidy --all` | Ignore staleness, purge everything in scope |
| `/tidy --dry-run` | Preview the plan, delete nothing |

Before any deletion the skill prints a plan table (age, size, targets per project), shows which projects are being kept, and asks for confirmation. Then it records disk free before/after and reports how much was reclaimed.

## What it deletes

**Dependency dirs:** `node_modules`

**Build/cache dirs (all regenerable):**
- Frontend: `.next`, `.turbo`, `.expo`, `.nuxt`, `.svelte-kit`, `.parcel-cache`, `.vite`, `.cache`
- Generic: `dist`, `build`, `out`, `coverage`
- Solidity: `artifacts`, `cache` (only next to `foundry.toml` / `hardhat.config.*`)
- Python: `__pycache__`, `.pytest_cache`, `.mypy_cache`, `.ruff_cache`
- Rust: `target` (only next to `Cargo.toml`)

Up to 4 levels deep from each project root (covers monorepo packages).

## What it never touches

Source dirs (`src`, `app`, `lib`, `pages`, `components`, `contracts/src`, test dirs), `.env*`, `.git`, `.github`, `.vscode`, `docs`, `public`, `assets`, `package.json`, all lockfiles, and any `*.config.*`.

## Staleness logic

A project is "stale" if the latest mtime of files *outside* regenerable dirs is older than the threshold. Dependency install timestamps don't reset the clock — only real source edits do.

## License

MIT
