# DTU Compass — TODO

Quick-start checklist for building v2, ordered by milestone ([`docs/rebuild/02-architecture.md §13`](docs/rebuild/02-architecture.md)). Each item is sized so it can be handed directly to a coding agent.

## Day-1 Parallel Blockers (non-code)

- [ ] Request DTU IT app registration for sign-in (Entra ID / WAYF) — lead time can be weeks ([`02-architecture.md §13`](docs/rebuild/02-architecture.md)).
- [ ] Run v1 `webhost_access_logs_analyzer.py` on the last registration week to get real traffic mix, peak RPS, and bot share ([`02-architecture.md §9`](docs/rebuild/02-architecture.md)).
- [ ] Name the GDPR data controller (`PRIV-01`) and confirm ePrivacy Art. 5(3) handling for the `localStorage` survey dismissal flag (`FDB-03`) before M3.

---

## M0 — Foundations (Start Here)

Goal: repo scaffold, CI, pure domain logic with parity tests, pipeline, Postgres schema, snapshot loader, and the 3-day auth spike.

- [ ] **M0.1 Repo & local stack scaffold (`OPS-01`, `OPS-04`)**
  - Create `/backend/dtuca/` (Python 3.14, FastAPI, Pydantic v2, SQLAlchemy 2, Alembic, pytest, mypy/pyright, ruff), `/web/` (Next.js App Router, TypeScript strict, Tailwind + shadcn/ui), and `/infra/docker-compose.yml` (Postgres 18 + api + web).
  - Add GitHub Actions CI workflow: lint, type check, unit tests.
- [ ] **M0.2 Postgres schema & initial migration (`02-architecture.md §3`)**
  - Model catalogue tables (`dataset_release`, `course`, `course_term_stats`, `prereq_edge`, `prereq_hint`, `course_person`, `person`, `study_line`, `course_study_line`, `course_embedding`, `vocabulary_map`) and user/feedback tables (`app_user`, `web_session`, `plan`, `plan_item`, `plan_version`, `pending_approval`, `plan_star`, `plan_report`, `personal_token`, `assistant_conversation`, `assistant_message`, `llm_usage_daily`, `feedback`).
- [ ] **M0.3 Pure `domain/` core (`OPS-02`, `DQ-02`, `DQ-03`)**
  - Implement I/O-free modules under `backend/dtuca/domain/`: metrics, terms/periods/spans (`PLAN-12`), slots, clash detection, prerequisite OR-group checks, and plan validation.
  - Add unit tests (≥ 90% coverage) + regression tests for §5 bugs #9–15 and #19 ([`01-requirements.md §5`](docs/rebuild/01-requirements.md)).
- [ ] **M0.4 Data pipeline (`PIPE-*`, `CAT-17`)**
  - Build `backend/dtuca/pipeline/`: async `httpx` fetchers + raw HTML store, fixture-tested `selectolax` parsers (info, grades, evals, people), normaliser (with `previous_codes` predecessor back-fill and cycle guard), validation gates, and atomic publish.
  - Run parity script against the 50 reference courses (`DQ-02`).
- [ ] **M0.5 In-memory `Snapshot` loader & embeddings (`PERF-05`, `COST-03`)**
  - Load published release into immutable `Snapshot` with local msgpack+npz fallback (`SCAL-03`) and 30 s release polling.
  - Benchmark `multilingual-e5-small` int8 ONNX (`intra_op_num_threads=1`) vs `all-MiniLM-L6-v2` on the 50-query relevance set; measure RSS and p95 latency.
- [ ] **M0.6 Auth spike (time-boxed 3 days, `02-architecture.md §1, §13`)**
  - Test Keycloak (first) / Zitadel against a stub `/mcp` with Claude and ChatGPT (CIMD/DCR, PKCE, `aud` bound to `/mcp`, SSRF check) using a mock OIDC IdP.
  - Record decision as an ADR in `docs/decisions/` (chosen broker + version/flags, or no-broker fallback). Do not build auth features until M3.

---

## M1 — Catalogue & Public Beta

- [ ] Tool registry (`backend/dtuca/tools/registry.py`) generating REST, OpenAI tools (`/v1/tools`), and stateless `/mcp/public` (`AGT-05`).
- [ ] Public tools: `search_courses`, `get_courses`, `recommend_courses`, `validate_plan`, `parse_study_planner_text`, `get_dataset_info`.
- [ ] CI checks: tool schema strict-mode check + capability-matrix parity test (`AGT-13`).
- [ ] Web UI: `/`, `/search`, `/course/{code}` (SSR SVG charts, "Similar", history switcher), `/discover`, i18n (en/da), light/dark theme, SEO & v1 redirects (`02-architecture.md §12`).
- [ ] Run k6 load test at 1,000 req/s with one node drained (`PERF-01`, `PERF-02`).
- [ ] Run 5-student campus hallway usability tests on the beta (`03-ux-and-product.md §5.6, §8`).

---

## M2 — Planner (Launch, P0)

- [ ] Local plans in `localStorage` (up to 5): timeline, week grid, spans, live `validate_plan` checks, version undo, link/JSON export & import preview, Safari eviction warning.
- [ ] Public tools: `make_plan_link`, `get_shared_plan` (with `start_term` shift), `list_templates`, `report_data_issue`.
- [ ] Programme starter templates (`SHR-05a`) and "Fill this term" (`PLAN-07`).
- [ ] Inline **"Report a data issue"** on `/course/{code}` + `feedback` table + private Discord webhook (`FDB-01`, `FDB-04`; global feedback popover optional).
- [ ] Accessibility audit of J1 (`UX-01`).

---

## M3 — Accounts, Assistant & Analytics (P1)

- [ ] Auth broker (or no-broker fallback), DTU sign-in, synced plans, `/mcp` with OAuth, personal access tokens, and sign-in tools (`get_me`, `update_me`, `get_plan`, `create_plan`, `edit_plan`, `restore_plan_version`, `set_sharing`, `delete_plan`).
- [ ] Owner approvals in web UI (`AGT-08`), `export_plan` (iCal, `PLAN-08`), `/compare` (`PLAN-11`).
- [ ] Built-in assistant with streaming SSE, tool loop (≤ 8 calls/turn), quotas (`COST-02`), and prompt-injection test (`SEC-06`).
- [ ] Self-hosted cookieless Umami on Postgres + typed `web/lib/events.ts` (`ADM-03`, [`ADR-001`](docs/decisions/001-usage-analytics-and-interaction-tracking.md)).
- [ ] Global feedback popover (if not in M2), assistant & `/discover` thumbs up/down, in-session micro-survey (`FDB-02`, `FDB-03`), `sentry_event_id` linkage, and server-side assistant undo-rate tracking.

---

## M4 — Community

- [ ] Public `/p/{slug}` & unlisted `/s/{slug}` sharing with `SHR-09` privacy rules, gallery, forks, stars (`star_plan`).
- [ ] 10 curated templates (`SHR-05b`), k-anonymous signals (`PRIV-04`), and moderation runbook (`SHR-07`, `ADM-02`).
