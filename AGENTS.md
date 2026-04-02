# Repository Guidelines

## Project Structure & Module Organization

This repository is a lightweight skill package centered on [SKILL.md](/Volumes/Work/code/protocol-api-simulation-skill/SKILL.md). That file defines the skill metadata, scope, safety boundaries, workflow steps, and expected outputs for protocol-level browser/API simulation work. There is currently no `src/`, `tests/`, or asset directory; keep repository changes focused on Markdown documentation unless the structure is intentionally expanded.

## Build, Test, and Development Commands

There is no build system or automated test runner in the current repository. Use simple content checks while editing:

- `sed -n '1,120p' SKILL.md` — inspect the current skill definition.
- `wc -w AGENTS.md SKILL.md` — confirm document size stays concise.
- `rg -n '^## ' SKILL.md AGENTS.md` — verify heading structure.

If you add tooling later, document the exact command here instead of relying on implicit conventions.

## Coding Style & Naming Conventions

Write documentation in clear Markdown with short sections and direct instructions. Use `#`, `##`, and flat bullet lists; avoid decorative Unicode symbols and unnecessary prose. Keep filenames uppercase for top-level contributor docs (`AGENTS.md`, `SKILL.md`). When adding new repository files, prefer descriptive names such as `examples/redirect-trace.md` or `docs/state-machine-notes.md`.

## Testing Guidelines

Validation is documentation-focused. Before submitting changes:

- Check headings and list structure render cleanly.
- Confirm examples match the repository purpose: authorized protocol testing, session consistency, PKCE, redirect tracing, and risk-control boundaries.
- Re-read for contradictions between contributor guidance and [SKILL.md](/Volumes/Work/code/protocol-api-simulation-skill/SKILL.md).

If executable examples or scripts are introduced later, add a matching `tests/` directory and document how to run those checks here.

## Commit & Pull Request Guidelines

Local Git history is not available in this workspace, so no repository-specific commit pattern can be inferred from history. Use short imperative commit messages such as `docs: tighten redirect tracing guidance` or `docs: add contributor rules`. Pull requests should describe the purpose, affected files, and any behavior or policy changes. Include rendered Markdown screenshots only when formatting changes are significant.

## Security & Scope Notes

Do not add guidance that enables bypassing CAPTCHA, anti-abuse controls, MFA, device binding, or unauthorized token capture. All repository content must stay aligned with the explicit authorization and safety limits defined in [SKILL.md](/Volumes/Work/code/protocol-api-simulation-skill/SKILL.md).
