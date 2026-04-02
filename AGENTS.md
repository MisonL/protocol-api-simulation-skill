# Repository Guidelines

## Project Structure & Module Organization

This repository is a lightweight skill package centered on `SKILL.md`. That file defines the skill metadata, scope, safety boundaries, workflow steps, and expected outputs for protocol-level browser/API simulation work. Reference material lives in `references/`, which currently contains product-specific lessons in `references/codex-console-lessons.md` and generic lab/observability patterns in `references/security-lab-lessons.md`. There is currently no `src/`, `tests/`, or asset directory; keep repository changes focused on Markdown documentation unless the structure is intentionally expanded.

## Build, Test, and Development Commands

There is no local build system in this repository. Use lightweight documentation checks while editing:

- `sed -n '1,120p' SKILL.md` — inspect the current skill definition.
- `sed -n '1,120p' references/security-lab-lessons.md` — review the generic lab-pattern reference in isolation.
- `wc -w AGENTS.md SKILL.md` — confirm document size stays concise.
- `rg -n '^## ' SKILL.md AGENTS.md` — verify heading structure.
- `rg -n '/Users/|/Volumes/' README.md SKILL.md references/*.md AGENTS.md` — catch machine-local paths before committing.
- `git diff --check` — catch whitespace and patch-format issues in Markdown edits.
- `uv run --with pyyaml python <path-to-skill-creator>/scripts/quick_validate.py .` — validate skill frontmatter and naming rules.

If you add tooling later, document the exact command here instead of relying on implicit conventions.

## Coding Style & Naming Conventions

Write documentation in clear Markdown with short sections and direct instructions. Use `#`, `##`, and flat bullet lists; avoid decorative Unicode symbols and unnecessary prose. Keep filenames uppercase for top-level contributor docs (`AGENTS.md`, `SKILL.md`). When adding new repository files, prefer descriptive names such as `examples/redirect-trace.md` or `docs/state-machine-notes.md`.

## Testing Guidelines

Validation is documentation-focused. Before submitting changes:

- Check headings and list structure render cleanly.
- Confirm `README.md`, `AGENTS.md`, and `SKILL.md` describe the same current repository shape.
- Confirm examples match the repository purpose: authorized protocol testing, session consistency, PKCE, redirect tracing, and risk-control boundaries.
- Re-read for contradictions between contributor guidance and `SKILL.md`.
- Re-read `references/` links and filenames after renames or structural edits.
- Run the `quick_validate.py` command above after substantial skill edits.

If executable examples or scripts are introduced later, add a matching `tests/` directory and document how to run those checks here.

## Commit & Pull Request Guidelines

Current history uses short, single-purpose commit subjects such as `docs: add protocol-api-simulation skill`, `docs: add security lab reference patterns`, and `docs: refine protocol skill workflow`. Keep commits scoped to one documentation change and use a brief imperative summary. Pull requests should describe the guidance change, list the validation commands you ran, and call out whether the change affects the main `SKILL.md`, repository docs, or only reference material.

## Security & Scope Notes

Do not add guidance that enables bypassing CAPTCHA, anti-abuse controls, MFA, device binding, or unauthorized token capture. All repository content must stay aligned with the explicit authorization and safety limits defined in `SKILL.md`.
