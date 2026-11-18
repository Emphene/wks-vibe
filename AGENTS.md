# AGENTS.md — wks-vibe

Instructions for AI agents working in this repository.

> **Generated file.** This document is rendered from
> `.agents/skills/templates/AGENTS.md.tmpl` by `sync-docs` (which runs
> automatically on every `skills-sync pull`). Do not edit it directly —
> edit the template in Central Skills and publish it instead.

## Repository layout

- `.agents/skills/` — a **git subtree** mirrored from Central Skills
  (`Emphene/central-skills`, branch `skills-split`). Contains shared
  QuantLib/marimo skills (each folder has a `SKILL.md`), the sync engine
  (`sync_engine.py`), and the `templates/` folder this file was rendered from.
  The directory name is not arbitrary: it is **this agent CLI's native
  skill-discovery directory** (`.agents/skills/` for OpenCode, Oh My Pi,
  MiMo Code, Pi, Vibe and Codex, `.claude/skills/` for Claude Code), so the
  synced skills are picked up and loaded by the agent working in this
  workspace without any manual wiring.
  The engine auto-detects the prefix from its own location (override:
  `AGENT_SKILLS_DIR` env var or `--prefix` flag), so the same engine file
  works unchanged in every agent repo.
- `main.py` — orchestrator demo of the pull → work → push cycle.
- `.venv/` — Python environment; kept in sync by `sync-env` (see below).
- `requirements.txt` — **rendered** transport of Central Skills'
  dependency set; the hand-maintained source is Central Skills'
  `pyproject.toml` + `uv.lock` (`.agents/skills/templates/requirements.txt.tmpl`
  is the generated intermediate). Do not hand-edit any of them.
- `AGENTS.md`, `README.md` (repo root) — **rendered** from
  `.agents/skills/templates/` via `sync-docs`; do not hand-edit.
- The as-built guide for this network lives in Central Skills (it is
  never copied into agent repos):
  [`AI Skills Architecture.md`](https://github.com/Emphene/central-skills/blob/main/AI%20Skills%20Architecture.md)

## The architecture you are part of

This repo is one node of a multi-agent skills network:

```
central-skills (Central Skills)
 ├─ main .............. whole project; .agents/skills/ = mirror of the skills line
 └─ skills-split ...... skills-only publishing line (canonical, orphan branch)
        ▲            ▲
   subtree push   subtree pull (NEVER squashed)
        │            │
wks-opencode        wks-claude          wks-omp             wks-mimo            wks-pi              wks-vibe            wks-codex           (future agents: same pattern)
```

- `skills-split` carries *only* the skills tree; that is what keeps subtree
  push/pull fast-forward in both directions (a push onto Central Skills'
  `main` would be non-fast-forward and rejected, since `main` holds the whole
  project).
- Central Skills' `.agents/skills/` is a *mirror* of `skills-split`,
  refreshed by `promote`. Both lines stay ancestry-linked, so a `push` from
  here is always a fast-forward.
- Everything under `.agents/skills/` is visible to every agent in the network
  (`wks-opencode`, `wks-claude`, `wks-omp`, `wks-mimo`, `wks-pi`, `wks-vibe`,
  `wks-codex`, …). Treat it as a shared
  publish/subscribe channel, not as private scratch space.

## When to sync skills

1. **At session start** — absorb the latest skills published by other agents
   before doing any task:

   ```bash
   .venv/bin/python .agents/skills/sync_engine.py pull
   ```

   Never run `pull` with uncommitted changes; commit or stash first.
   `pull` also re-renders the root `AGENTS.md`/`README.md` from the central
   templates, so agent docs stay aligned across the whole network.

2. **Then make the venv match** — especially after Central Skills published
   a dependency upgrade (e.g. a new quantlib version):

   ```bash
   .venv/bin/python .agents/skills/sync_engine.py sync-env
   ```

   This runs `uv pip sync requirements.txt`, so the venv reflects the
   shared requirements exactly (installs, upgrades, removes strays).

3. **After discovering reusable skill knowledge** — anything another agent
   would benefit from (QuantLib gotchas, marimo patterns, validation rules,
   formatting constraints): add or edit a file under the relevant skill folder
   (e.g. `.agents/skills/marimo-notebook/references/`), then publish it:

   ```bash
   .venv/bin/python .agents/skills/sync_engine.py push -m "wks-vibe-sync: <what changed and why>"
   ```

4. **Periodically (ask the user)** — skill edits only reach Central Skills'
   `main` branch after a promote, run from Central Skills checkout:

   ```bash
   python .agents/skills/sync_engine.py promote   # in central-skills
   ```

   A skill edit made *directly* in Central Skills' `main` must be
   published with `central-push` instead of committing and pushing blindly.

## Engine command reference (`.agents/skills/sync_engine.py`)

| Command | Runs in | What it does |
|---|---|---|
| `pull` | agent repo | Non-squashed `subtree pull` of `skills-split` into `.agents/skills/`, auto-renders root docs, pushes the agent's `main` |
| `push -m "..."` | agent repo | Commits `.agents/skills/`, pushes the agent's `main`, then subtree-pushes to `skills-split` on Central Skills |
| `sync-docs [--push]` | agent repo | Renders root `AGENTS.md`/`README.md` from `.agents/skills/templates/*.tmpl` (also runs automatically on `pull`) |
| `sync-env` | any repo | Central Skills: `uv sync` (from `uv.lock`); agents: `uv venv` (if needed) + `uv pip sync requirements.txt` |
| `export-requirements` | Central Skills | Regenerates `templates/requirements.txt.tmpl` from `uv.lock` (`uv export`), ready for `central-push` |
| `promote` | Central Skills | Fetches `skills-split`, records it as a merge parent (`-s ours --allow-unrelated-histories`), overlays `.agents/skills/`, amends the merge commit, pushes `main` |
| `central-push -m "..."` | Central Skills | Commits edits under `.agents/skills/`, links any newer `skills-split` commits, pushes `main` + subtree |

The central URL defaults to `git@github.com:Emphene/central-skills.git`
(override with the `CENTRAL_SKILLS_REPO` env var). The agent name/id used for
doc rendering default to the `origin` repo name (override with `AGENT_NAME` /
`AGENT_ID`). Git auth is SSH through the key held by `ssh-agent` — never
hardcode a token in code, URLs, or commits.

## Sync rules

- Central Skills is the source of truth; `skills-split` is its
  skills-only publishing branch. Never `git push --force` to it.
- Never run `git subtree pull/add ... --squash` in this repo — it severs the
  shared history and makes every subsequent subtree push fail as
  non-fast-forward. (This repo's subtree was added without `--squash`.)
- If `push` reports non-fast-forward, another agent published meanwhile: run
  `pull`, resolve conflicts, then retry the push.
- The root `AGENTS.md` and `README.md` are **generated files**: rendered
  from `.agents/skills/templates/*.tmpl`. To change them, edit the template,
  publish it with `push`, and let every agent absorb it on its next `pull` —
  never hand-edit the root copies.
- The root `requirements.txt` is a **generated file too** — a transport of
  Central Skills' `pyproject.toml` + `uv.lock` (the only hand-edited
  dependency files). Never edit it locally. Dependency changes flow one way:
  in Central Skills edit `pyproject.toml` (or `uv lock --upgrade-package
  <pkg>`) + `uv sync`, then `export-requirements`, then `central-push`;
  agents then `pull` + `sync-env`.
- Do not hand-edit files under `.agents/skills/` outside a sync cycle, and do
  not commit `.agents/skills/` changes without publishing them via `push`
  (other agents would never receive them).
- Keep each skill's `SKILL.md` and reference files short, imperative, and
  scoped to one topic each.
- Skill *code* for this project (notebooks, QuantLib helpers) still lives
  outside `.agents/skills/`; only cross-agent knowledge belongs in it.

## Onboarding a new agent (Agent D, E, …)

```bash
git clone git@github.com:Emphene/<new-agent-repo>
cd <new-agent-repo>
git subtree add --prefix=.agents/skills/ git@github.com:Emphene/central-skills.git skills-split
cp <path-to-existing-agent>/{main.py,.gitignore} .
uv venv && uv pip install GitPython                                   # one-time bootstrap interpreter
.venv/bin/python .agents/skills/sync_engine.py sync-docs --push         # renders AGENTS.md + README.md + requirements.txt
.venv/bin/python .agents/skills/sync_engine.py sync-env                 # .venv ← rendered requirements.txt
```

The `subtree add` line must never gain `--squash` (see rules above). Root
docs and `requirements.txt` need no copying — `sync-docs` renders them from
the templates. See also `wks-claude`, `wks-mimo`, `wks-pi`, `wks-vibe`
and `wks-codex`, which were onboarded this way.
