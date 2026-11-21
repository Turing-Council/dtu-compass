# Contributing to DTU Compass

Thanks for helping. This is a student project run by Turing Council, so the process is deliberately light: a few rules, enforced by GitHub where possible.

## Before you start

- **The spec is in [`docs/rebuild/`](docs/rebuild/README.md).** Requirements have IDs (`PLAN-12`, `AGT-05`, …) in [`01-requirements.md`](docs/rebuild/01-requirements.md); the architecture is in [`02-architecture.md`](docs/rebuild/02-architecture.md).
- **Pick work** from [`TODO.md`](TODO.md), the [milestones](https://github.com/Turing-Council/dtu-compass/milestones) or issues labelled `good first issue`. Comment on the issue so nobody else picks it up.
- Using an AI coding agent? It must follow [`AGENTS.md`](AGENTS.md), and you are responsible for what it produces.

## Workflow

1. **Issue first** for anything non-trivial (use the Feature, Bug or Spike form). Typos and small doc fixes can go straight to a PR.
2. **Branch** from `main` as `type/short-slug`, e.g. `feat/search-facets`, `fix/overworked-denominator`.
3. **Open a PR** early (draft is fine) and fill in the template.
4. **Squash merge.** The PR title becomes the commit on `main`, so it must follow [Conventional Commits](https://www.conventionalcommits.org/): `type(scope): summary`, e.g. `feat(api): add course search facets`. CI checks it. Allowed types and scopes are listed in [`.github/workflows/pr-title.yml`](.github/workflows/pr-title.yml).
5. **Merging your own PR is allowed** once checks are green and review threads are resolved. Ask for a review when the change is risky (auth, migrations, infra, anything touching user data) or when you're unsure.

Scopes are finer than issue areas: the `backend` area maps to the `api`, `domain` and `catalogue` scopes.

## Definition of done

- Lint, type checks and tests pass locally and in CI (once M0 lands: `just lint test`).
- Requirement or regression IDs are referenced in the PR.
- Docs are updated when behaviour changes; an ADR is added when a decision is made (see below).
- DB migrations are expand/contract, compatible with the previous release (`OPS-05`).
- Generated code (OpenAPI client, etc.) is regenerated, never hand-edited.

Keep PRs small: aim for under ~400 changed lines, excluding generated files and fixtures. Split larger work.

## When to write an ADR

Copy [`docs/decisions/000-template.md`](docs/decisions/000-template.md) when a choice is costly to reverse, adds a dependency or external service, or departs from `02-architecture.md`. Add it to the table in [`docs/decisions/README.md`](docs/decisions/README.md).

## Local setup

The code scaffold doesn't exist yet (M0, issue #4). Once it lands, `just up` starts the full stack and `just --list` shows every task.

## Reporting security issues

Don't open a public issue. See [`SECURITY.md`](SECURITY.md).

## Conduct

Be decent. See [`CODE_OF_CONDUCT.md`](CODE_OF_CONDUCT.md).
