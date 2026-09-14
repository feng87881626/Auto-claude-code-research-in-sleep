# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

ARIS (Auto-claude-code-research-in-sleep) is a **research harness, not a traditional application**: ~82 composable Markdown skills (`skills/<name>/SKILL.md`) that orchestrate the ML research lifecycle through cross-model adversarial collaboration. The "code" is mostly SKILL.md prose plus Python/shell helpers in `tools/`. There is no build step.

**Hierarchy of truth**: `AGENT_GUIDE.md` is a routing index only. If it conflicts with a `skills/<name>/SKILL.md`, the SKILL.md wins. System-wide contracts live in `skills/shared-references/*.md`.

## Commands

```bash
# Full test suite (what CI runs on both Linux and macOS)
python -m pytest tests/ -q

# Single test file / single test
python -m pytest tests/test_codex_skill_mirror.py -q
python -m pytest tests/test_research_wiki_ideas.py -q -k "test_name"

# Test deps (no requirements.txt; CI installs exactly this)
python -m pip install pytest httpx

# Advisory lint: flags hardcoded `python3 tools/foo.py` in SKILL.md (never fails CI)
bash tools/lint_skills_helpers.sh

# Skill inventory drift check (CI-enforced): skills/ must match docs/SKILLS_CATALOG.md,
# README.md, README_CN.md, AGENT_GUIDE.md
python tools/check_skills_inventory.py
```

macOS matters: CI runs a macOS leg specifically because stock `/bin/bash` is 3.2 — `set -u` + empty-array `"${ARR[@]}"` expansion aborts there but not on Linux bash ≥ 4.4. Write POSIX-safe shell (`set -e` + `set -u` compatible); see example blocks in `skills/shared-references/integration-contract.md`.

## Architecture

### Skill layout

- `skills/<name>/SKILL.md` — mainline skills (Claude Code / Cursor / Trae / Antigravity / Copilot CLI)
- `skills/skills-codex/` — **mirror** of the mainline skills for Codex CLI (uses `spawn_agent` instead of `mcp__codex__codex`). Parity is tested by `tests/test_codex_skill_mirror.py` — when you edit a mainline SKILL.md, check whether the mirror needs the same change.
- `skills/skills-codex-claude-review/` and `skills/skills-codex-gemini-review/` — overlays on the Codex mirror that swap the reviewer
- Skills communicate through **plain-text artifact contracts** (e.g. `IDEA_REPORT.md` → `/experiment-bridge`, `NARRATIVE_REPORT.md` → `/paper-writing`, `REVIEW_STATE.json` for loop resume). The full artifact table is in `AGENT_GUIDE.md` § Artifact Contracts. When adding a skill or changing what one produces, keep the producer/consumer chain consistent.

### Helper resolution (when writing or editing SKILL.md)

Never hardcode `python3 tools/foo.py`. Resolve helpers through the chain in `skills/shared-references/integration-contract.md` §2:

```
Layer 0: ${CLAUDE_SKILL_DIR}/scripts/<helper>   # single-owner helpers live here
Layer 1: .aris/tools/<helper>                   # project-local symlink
Layer 2: tools/<helper>                         # repo-local
Layer 3: $ARIS_REPO/tools/<helper>              # global fallback
```

Single-owner helpers (used by exactly one skill) belong at `skills/<owner>/scripts/<helper>`; multi-owner helpers stay in `tools/`. Pick a failure policy (A gate / B side-effect / C forensic / D cascade / D2 aggregate / E diagnostic) from the integration contract.

### Cross-model review rules (system invariants)

These are the core design of ARIS; violating them silently breaks the product:

- **Executor and reviewer must be different model families.** Same-family review is recorded as `review_independence: same-family` / `acceptance_status: provisional`, never as cross-model acceptance.
- **Fresh threads for audits**: every reviewer call uses `mcp__codex__codex` (a fresh thread), never `codex-reply` — narrative accumulation inflates scores. (Exception: `/auto-review-loop`'s in-loop review memory uses one `threadId` deliberately.)
- **Reviewer independence**: pass file paths only, never summaries or interpretations; the executor must not judge its own integrity.
- **Provenance-as-authorization** (`skills/shared-references/skill-governance.md`, `tools/provenance.py`): auto-curation may only modify artifacts with a valid `.provenance.json` sidecar; `stamp()` structurally refuses same-family author/reviewer pairs. A deterministic verifier (e.g. `deterministic:pytest`) is a valid reviewer.
- Audit verdicts use the 6-state schema: `PASS | WARN | FAIL | BLOCKED | ERROR | NOT_APPLICABLE` (`skills/shared-references/assurance-contract.md`).
- Reviewer traces go to `.aris/traces/<skill>/<date>_run<NN>/`.

### Other directories

- `tools/` — Python helpers (`arxiv_fetch.py`, `research_wiki.py`, `provenance.py`, `verify_paper_audits.sh`, …) plus install/update scripts (`install_aris.sh`, `smart_update.sh`)
- `mcp-servers/` — MCP servers for reviewer routing (codex-image2, claude-review, gemini-review, llm-chat, minimax-chat, manual-review, feishu-bridge). Config keys in `.env.example`; tests skip themselves when the needed env/API key is absent.
- `tests/` — pytest suite; many tests are contract tests over SKILL.md content and shell scripts (they parse the Markdown, not run the skills)
- `templates/` — artifact templates referenced by skills
- `aris-monitor/` — standalone macOS status widget (`cd aris-monitor && ./run.sh`)

## Conventions

- Adding or renaming a skill requires updating the inventory: `docs/SKILLS_CATALOG.md`, `README.md`/`README_CN.md`, and `AGENT_GUIDE.md` — CI (`check-skills-inventory.yml`) fails on drift.
- Do not wrap internally-looping skills (`/auto-review-loop` etc.) in `/loop`, `/schedule`, or `CronCreate`; see `skills/shared-references/external-cadence.md`.
- SKILL.md frontmatter: `name`, `description`, `argument-hint`, `allowed-tools`. See `CONTRIBUTING.md` for the template.
- Effort (`lite | balanced | max | beast`) and assurance (`draft | polished | conference-ready | submission`) are independent control axes; parameters pass through workflow chains automatically.
- Documentation is bilingual: significant docs have `_CN` twins (README/CONTRIBUTING/SETUP_GUIDE, many `docs/` guides). Keep both in sync when editing one.
- **Respond to the user in Chinese (Simplified).** All user-facing prose — replies, summaries, clarifying questions — is written in Chinese. Code, committed artifacts, and paper-writing outputs still follow their own conventions (paper-writing emits English LaTeX for venue submission).
