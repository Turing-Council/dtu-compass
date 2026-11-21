# ADR-001: Usage analytics and interaction tracking

- **Status:** Proposed
- **Date:** 2026-11-21
- **Related Issue / Requirement:** `ADM-03`, `PRIV-01`, `PRIV-02`, `PRIV-03`, `PERF-03`, `OPS-03`, UX §8 success metrics

## Context

We want to know how students use DTU Compass: which controls they use, where they drop off, and whether we hit the UX §8 success metrics. A common way to do this is coordinate heatmaps and session recording (Hotjar, PostHog, Sentry Replay, rrweb).

These constraints from the plan apply:

- **PRIV-03:** no third-party tracking, strictly necessary cookies only, no consent banner. Session recording and mouse tracking store and read data on the device that is not strictly necessary, so under the ePrivacy rules they need consent.
- **PERF-03:** course-page JS < 150 KB gzip, LCP < 2.0 s at p75 on mobile. Recording scripts are tens of KB and run all the time.
- **Architecture §0 / §11:** boring parts, no new infrastructure without a measured need. Plausible was rejected because it needs ClickHouse; PostHog self-hosted needs ClickHouse and Kafka.
- **Architecture §0:** agents (MCP, assistant) are the main interface. Browser-side tracking does not see them.
- **ADM-03 / §10:** self-hosted, cookieless Umami on our Postgres is already the analytics choice.
- **Tool calls:** tool arguments are never logged (05-agent-parity-review #3).
- Layouts vary by breakpoint, language (en/da) and theme, so pixel coordinates are not comparable between visits.

## Decision

We measure usage in three layers and do **not** run always-on heatmaps or session recording.

### 1. Browser: Umami named events

- Umami tracks page views and a small, fixed set of named events. The script is served from our own domain and sits in the cached anonymous page shell.
- Click counts are **per element, not per coordinate**: key controls carry `data-umami-event="<name>"` (or call `umami.track(name, props)`).
- All event names live in one typed module in `web` (for example `web/lib/events.ts`). Every event must map to a UX §8 metric or a named product question. Start with about 10–15 events, for example `add_to_plan`, `compare_open`, `plan_export_link`, `palette_search`, `template_fork`.
- **Allowed properties:** enums and small numbers only (`source`, `term_offset`, `zero_results`, `query_length`). **Never:** plan contents, grades, course lists from a plan, free-text search queries, user ids or tokens.
- Funnels (for example "arrived from kurser.dtu.dk → `add_to_plan`") use Umami's referrer filter and funnel report.

### 2. Server: domain tables and the tool registry

- Metrics that follow from stored data are SQL queries on existing tables, shown on the admin page (ADM-02). No events are sent for them:
  - synced plans, plans shared (unlisted + public)
  - weekly MCP users (distinct tokens used)
  - anonymous plan loss (review #13)
- Tool usage across web, MCP and assistant comes from the tool registry's existing structured log fields (`actor_kind`, tool name, hashed user id), never from arguments.
- "User later added a course the assistant mentioned" is computed on the server from conversations and plan versions, not from click events.

### 3. Heatmaps and recordings: time-boxed studies only

- Visual heatmaps and session recordings are used only in moderated usability tests (UX §8, "time to a first term with no clashes") or in short opt-in studies with explicit consent, on staging or for consenting participants.
- They are never loaded for ordinary visitors in production.

### Operational rules

- Umami uses its own database (at least a separate database on the Postgres cluster), so its writes do not compete with plan writes. Umami is included in the M1 k6 load test.
- Umami is listed in the PRIV-01 processing record and on `/privacy`. Cookieless analytics still processes personal data (hashed IP + user agent).
- The bot/human page-view report comes from CDN logs (ADM-03), not from Umami.

## Alternatives Considered

- **Hotjar / Microsoft Clarity / hosted PostHog:** full heatmaps and recordings with no work. Third-party tracking breaks PRIV-03 and needs a consent banner and DPAs; the script weight threatens PERF-03.
- **PostHog self-hosted:** first-party, rich product analytics, heatmaps and recordings. Needs ClickHouse, Kafka and more; too much to run for a small team and the COST-01 budget. Recording still needs consent.
- **Sentry Session Replay:** Sentry is already a processor (PRIV-01) and masks text by default. It still needs consent for non-essential storage and adds noticeable JS to every page.
- **A homemade click/coordinate collector:** no new vendor. Coordinates don't compare across breakpoints, languages and themes, and we'd own a write-heavy ingest path for little gain over named events.
- **No analytics:** simplest, but we couldn't check the UX §8 success metrics and ADM-03 would not be met.

## Consequences

- No consent banner; PRIV-03 holds.
- Course-page JS budget is barely affected (one small script).
- Agent and assistant usage is measured the same way as web usage, through the tool registry.
- We get per-element click counts and funnels, not visual heatmaps. Questions like "where on the page do people look" need a usability study.
- New events need a code change to the typed event list and a link to a metric, which keeps the set small.
- Umami is one more service with a database to back up, upgrade and load-test.

## Revisit Trigger

- A product question comes up that named events, funnels and usability tests can't answer, and it justifies adding a consent flow.
- Umami's write load shows up in the M1 load test or in production Postgres metrics, so it needs its own database server.
- PRIV-03 changes (for example a consent banner is added for another reason).
