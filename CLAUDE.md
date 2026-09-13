# CLAUDE.md

Guidance for agents working in this repository. Detailed architecture notes and
the reasoning behind every rule below live in `docs/agent-notes.md`; read the
section for a module before changing it.

## Project Overview

**Masking Frame**: the darkroom device that holds paper under an enlarger. Keep the interface vocabulary in that world: frames, formats, sources, the contact sheet.

Python tool that turns a panoramic JPG into Instagram-ready images at `4:5`/Portrait, `1:1`/Square or `1.91:1`/Landscape: one bordered frame of the whole panorama plus at least two zoomed detail frames. It also composes two to six images of any orientation into one composite, arranged automatically. CLI plus a two-tab PySide6 GUI (Split, Compose).

## Commands

Setup (installs Python and uv via mise, then syncs dependencies):

```bash
mise install
mise run setup
```

`mise run setup` also installs the pre-commit hooks (`pre-commit install --hook-type pre-commit --hook-type pre-push`), so a fresh clone gets `ruff`/`ruff-format` on commit and `mypy`/`pytest` on push per `.pre-commit-config.yaml`.

Run:

```bash
mise run gui                    # GUI
mise run split -- <input.jpg> [output_prefix] --ratio 4:5   # single file
mise run split -- <input_dir> [output_dir] --ratio 1:1      # batch, defaults output_dir to ./output
mise run split -- compose <a.jpg> <b.jpg> [...] -o <prefix>   # two to six panels
```

Verify:

```bash
mise run check   # ruff lint, mypy --strict, pytest — run this before committing
```

`mise run check` is the single command that must pass. It runs `lint`, `fmtcheck`, `typecheck`, and `test` (see `mise.toml` for the individual tasks).

Every hook runs the project's own tool through `scripts/run-tool`. Never add a second, separately pinned ruff: they drift and fight over formatting. `docs` is excluded from ruff on purpose.

## Architecture

Src-layout package at `src/maskingframe/`. Dependencies are one-way: `geometry` and `layout` are leaves; `compose` uses both; `pipeline` uses all three; `cli` and `gui/` use **only** `pipeline`.

- `geometry.py`: pure PIL transforms, no filesystem. Owns `AspectRatio`, `RATIOS`, the position model, and the border model (`FrameStyle`, `parse_colour`, the single colour parser).
- `layout.py`: pure arithmetic, no PIL or I/O. Generates composite arrangements (one level of grouping, on purpose), scores them on fill, `rank()`/`solve()` share one sort key, `short_name()`/`parse_name()` own the two spellings.
- `compose.py`: PIL in, PIL out. Border colour, then gutters, then panels, in that order.
- `pipeline.py`: the only module that touches the filesystem. Owns output filenames, batch processing and `compose_images()`.
- `cli.py`: argparse entry points. Splitting stays top-level; `compose` is dispatched on the literal first word.
- `gui/`: PySide6 front end. `settings.py` is the only `QSettings` user, `work.py` runs slow work off the GUI thread, `theme.py` is the design system.

Rules that are easy to break:

- **Don't remove `pipeline`'s re-exports** from `geometry` and `layout`. They are what keep `cli` and `gui/` off the lower modules.
- **Never `sorted()` `RATIOS`.** Insertion order (Portrait, Square, Landscape) is the presentation order everywhere.
- **`style` is always the last parameter, with a default**, on the six `pipeline` entry points, and never module state. New options go in front of it.
- **Compose never crops and never permutes input order.** An unknown or impossible arrangement name is a `ValueError`, never a silent fallback.
- **`process_folder()` reports per-file failures in `BatchResult`**; it doesn't raise.
- **`cli` never exits on import.** The PySide6 guard lives in `gui_main()`.
- **GUI concurrency: a job returns plain data; the callback runs on the GUI thread.** Use `work.submit(job, on_done, on_failed, owner=self)` whenever the callback touches a widget. Never touch a widget inside a job.
- **Stored settings are untrusted input.** `load_style()` validates through `FrameStyle` and falls back to `DEFAULT_STYLE` whole.
- **Theme:** keep the palette's range and every text pairing at WCAG AA. `CHINAGRAPH` marks up (primary action, selection, numbering, progress, errors) and never chrome.
- **Tabs expose only `subject`, `detail` and `band_changed`** to the shell, and never know about each other.

## Beads Issue Tracker

This project uses **bd (beads)** for issue tracking. Run `bd prime` to see full workflow context and commands.

### Quick Reference

```bash
bd ready              # Find available work
bd show <id>          # View issue details
bd update <id> --claim  # Claim work
bd close <id>         # Complete work
```

### Rules

- Use `bd` for ALL task tracking — do NOT use TodoWrite, TaskCreate, or markdown TODO lists
- Run `bd prime` for detailed command reference and session close protocol
- Use `bd remember` for persistent knowledge — do NOT use MEMORY.md files

**Architecture in one line:** issues live in a local Dolt DB; sync uses `refs/dolt/data` on your git remote; `.beads/issues.jsonl` is a passive export. See https://github.com/gastownhall/beads/blob/main/docs/SYNC_CONCEPTS.md for details and anti-patterns.

## Agent Context Profiles

The managed Beads block is task-tracking guidance, not permission to override repository, user, or orchestrator instructions.

- **Conservative (default)**: Use `bd` for task tracking. Do not run git commits, git pushes, or Dolt remote sync unless explicitly asked. At handoff, report changed files, validation, and suggested next commands.
- **Minimal**: Keep tool instruction files as pointers to `bd prime`; use the same conservative git policy unless active instructions say otherwise.
- **Team-maintainer**: Only when the repository explicitly opts in, agents may close beads, run quality gates, commit, and push as part of session close. A current "do not commit" or "do not push" instruction still wins.

## Session Completion

This protocol applies when ending a Beads implementation workflow. It is subordinate to explicit user, repository, and orchestrator instructions.

1. **File issues for remaining work** - Create beads for anything that needs follow-up
2. **Run quality gates** (if code changed) - Tests, linters, builds
3. **Update issue status** - Close finished work, update in-progress items
4. **Handle git/sync by active profile**:
   ```bash
   # Conservative/minimal/default: report status and proposed commands; wait for approval.
   git status

   # Team-maintainer opt-in only, unless current instructions forbid it:
   git pull --rebase
   git push
   git status
   ```
5. **Hand off** - Summarize changes, validation, issue status, and any blocked sync/commit/push step

**Critical rules:**
- Explicit user or orchestrator instructions override this Beads block.
- Do not commit or push without clear authority from the active profile or the current user request.
- If a required sync or push is blocked, stop and report the exact command and error.
<!-- END BEADS INTEGRATION -->
