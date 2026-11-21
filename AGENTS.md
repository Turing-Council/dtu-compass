# Rules for AI coding agents

These rules apply to any agent working in this repository (Claude Code, Codex, Cursor, Copilot, …). Humans follow [`CONTRIBUTING.md`](CONTRIBUTING.md); everything there applies to agents too.

## Read before acting

- The spec is [`docs/rebuild/01-requirements.md`](docs/rebuild/01-requirements.md) (requirement IDs) and [`docs/rebuild/02-architecture.md`](docs/rebuild/02-architecture.md). Read the sections for the area you touch before changing it.
- Check [`docs/decisions/`](docs/decisions/) for ADRs that constrain the area.
- If the task conflicts with the spec, or the spec is ambiguous, stop and ask. Don't guess. If the human decides, draft an ADR.

## Architecture rules (do not break)

- `backend/dtuca/domain/` is pure: no I/O, no network, no DB, no clock or env reads. Everything depends on it, never the other way round.
- Every capability is a tool defined once in the tool registry (`AGT-05`). REST, MCP and the assistant are generated from it. No UI-only or REST-only features.
- User-generated and scraped text goes in fields named `*_untrusted` (`SEC-06`).
- Use `null` for missing values; never store strings like `"None"`, `"No data"` or `"–"` (`CAT-11`).
- Never hand-edit generated files (OpenAPI client, generated REST/MCP code). Regenerate them with the `just` recipe.

## Hard limits

- Never push to `main`, force-push a shared branch, or merge a PR unless the human asks.
- Never commit secrets, tokens, `.env` files or real student data. Use fixtures.
- Don't modify these unless the human explicitly asks: `infra/` (including Cloudflare/OpenTofu), auth code, DB migrations that drop or rename anything, `.github/workflows/` permissions or triggers.
- Never add a checkout of PR code to a `pull_request_target` workflow.
- Don't add a dependency without saying why in the PR; a new service or major dependency needs an ADR.

## How to work

- Keep diffs minimal and on-task: no drive-by refactors, renames or reformatting.
- Add or update tests with every behaviour change. `domain/` keeps ≥ 90% coverage (`OPS-02`). Bug fixes start with a failing regression test.
- Run lint, type checks and tests before claiming the work is done (once M0 lands: `just lint test`). Report failures honestly.
- Use Conventional Commit messages and PR titles (`feat(api): …`, `fix(pipeline): …`). Fill in the PR template, including the AI-assistance line.
- Reference requirement IDs in commits and PRs.
