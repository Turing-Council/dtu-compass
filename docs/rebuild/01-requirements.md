# DTU Course Analyzer v2: requirements (FR and NFR)

Status: draft for review · Date: 2026-11-21 · Owner: Turing Council (with DTU approval)

This document lists what the rebuild must do. It comes from the extraction reports in [extraction/](extraction/):
- [extract_pipeline.md](extraction/extract_pipeline.md): scrapers, data model, formulas
- [extract_backend.md](extraction/extract_backend.md): routes, filtering, recommender
- [extract_frontend.md](extraction/extract_frontend.md): pages, UI, UX pain points

Those reports have file:line citations into the current code.

**Legend**
- **Origin:**
  - `E`: existing behaviour to keep.
  - `F`: existing behaviour that is broken today and must work correctly.
  - `N`: new.
- **Priority:**
  - `P0`: launch. **Launch = end of milestone M2** (02-architecture §13): v1 parity plus the core planner.
  - `P1`: first release after launch.
  - `P2`: later.

---

## 1. Context in one paragraph

Today the site is a Flask app on PythonAnywhere. It serves about 1,750 DTU courses from pickles and JSON files that a manual scraping run builds (around 100k sequential requests, several hours, 1–2 times a year).

Students can do four things:
- filter courses (OR within a category, AND across categories)
- sort them on one metric
- open a course page with grades, evaluations and teachers
- use an embedding-based recommender

There are no accounts and nothing is saved. There is no plan builder, no schedule-conflict check and no sharing. About a dozen bugs silently change results; examples:
- four study-line filters don't match anything
- the recommender's language filter doesn't work
- the July and August filters are ignored
- searches are case-sensitive

Browse, home and recommend requests re-parse JSON files on disk every time, which limits how many users the site can serve at once.

## 2. Users and their main jobs

| Persona | Main job |
|---|---|
| **New student** (arrives from a kurser.dtu.dk course page or Google) | "Is this course good for me, does it fit my schedule, and what else should I take this semester?" |
| **Current student** | Plan the next semesters, find electives, check prerequisites and schedule conflicts, compare courses |
| **Student using an AI agent** (Claude, ChatGPT, Cursor, a local LLM) | Ask questions about courses and edit their plan in natural language through MCP or tool calls |
| **Student sharing or using a template** | Publish "my BSc Software Technology path" or start from someone else's |
| **Maintainer / Turing Council admin** | Refresh data, check data quality, moderate shared content, watch costs |

---

## 3. Functional requirements

### 3.1 Course catalogue and data (CAT)

| ID | Requirement | Origin | Pri |
|---|---|---|---|
| CAT-01 | Store every course offered per academic year (2020/21 onward) with: code, English and Danish title, ECTS, language, level (BSc/MSc/BEng/PhD, with mappings: BScMSc→MSc, Deltidsdiplom→BEng, Part-time master→MSc), campus, semester period(s), DTU schedule slots, course duration, exam form, aids, examiner, grading scale, assignments, exam duration, exam date, scope and form, department, home page, "not applicable together with" (point lock), previous course, sign-up info, and the free-text description, objectives, content, remarks and highlighted message | E | P0 |
| CAT-02 | Keep course codes as case-insensitive strings. 195 codes contain letters (e.g. `KU181`), so they are not 5-digit numbers | F | P0 |
| CAT-03 | Store raw grade counts per course per exam term (12, 10, 7, 4, 02, 00, -3, passed, failed, absent). Sick, no-show and not approved count as absent, including the Danish "Ikke godkendt", which today is never matched | E/F | P0 |
| CAT-04 | Derive the average grade as Σ(weight·count)/numeric count, plus pass %, fail % and absent %. The result is "pass/fail" when the numeric count × 10 ≤ passed + failed. All derived values are computed in one shared module, and its formulas are tested | E | P0 |
| CAT-05 | Derive "students per term (graded)" (v1 calls this "sign-ups", but it comes from grade counts, not registrations) and "number of semesters run". Exclude re-exam-only terms with the existing heuristic, or a documented replacement for it | E | P0 |
| CAT-06 | Store evaluation answer counts (1–5) per course per term for learning, motivation, feedback and workload. Derive the average, the share of 4–5 answers, votes and tier (0–9, half-star bands). Rating = the pooled learning, motivation and feedback answers. Workload is reported separately (1 = much less time than expected, 5 = much more). The "overworked %" uses workload votes as its denominator (today it wrongly uses learning votes) | E/F | P0 |
| CAT-07 | Store teachers (main and co-responsible) with name, photo URL and profile URL, linked to the specific course offering. Historical years must not show today's teachers: past teachers are parsed from the year-specific course page's "Responsible" and "Course co-responsible" fields (the `/course/{code}/info` page is current-only and is fetched once per course, not per year). Teachers without a photo show initials | E/F | P0 |
| CAT-08 | Normalise study lines to a canonical list (currently 74 names plus "none"), handling renames between years. The mapping rules are data, not code, and an unknown name stops the publish step. The same table holds the manual orbit person-slug overrides | E | P0 |
| CAT-09 | Parse course codes mentioned in prerequisite text into structured edges (recommended or mandatory), including codes with letters. Alternatives (`01001/01003`, "or") become one OR-group: any course in the group satisfies it. Clauses that can't be parsed confidently become informational hints, never warnings. Derive "subsequent courses" by reversing those edges. Mandatory prerequisites must also be fed into the embedding text (lost today) | E/F | P0 |
| CAT-10 | Derive department from the course-code prefix as today (19 departments plus "partner university"), and also keep the department name scraped from the course page | E | P0 |
| CAT-11 | Use one explicit "missing" value (`null`) everywhere. The strings "None", "No data", "NO_DATA" and "–" must never be stored. "Grades hidden (≤3 students)" (the grade page has only 2 tables) is a separate state from "no data" | F | P0 |
| CAT-12 | Each dataset release records its source timestamps and the date each kind of data was last refreshed (info, grades, evaluations). The UI and the API both show "data as of" | N | P0 |
| CAT-13 | Keep historical (discontinued) courses searchable, with their latest known data and a "discontinued" flag | E | P0 |
| CAT-14 | Bulk export of the dataset (CSV and JSON) per release under an open licence, replacing today's GitHub CSV | E | P0 |
| CAT-15 | Ingest the prerequisite graph ourselves (CAT-09) instead of fetching third-party JSON from GitHub when the page loads | F | P0 |
| CAT-17 | Predecessor back-fill: a course's stats include terms inherited from its predecessors (`previous_codes[]`; DTU's field can list several, split on `and`, `/`, `,`) wherever it has no data of its own for that term (v1 does this in csv_creator.py:99-117). If two predecessors have data for the same term, the first listed wins. Chains are followed with a cycle guard. Inherited terms are labelled "as 0xxxx" on the course page | E | P0 |
| CAT-16 | Programme rules: which courses are mandatory, polytechnical foundation, programme-specific or electives per programme and start year, from DTU study regulations (sdb.dtu.dk). Until this exists, starter and curated templates (SHR-05a, SHR-05b) stand in for it | N | P2 |

### 3.2 Data pipeline (PIPE)

| ID | Requirement | Origin | Pri |
|---|---|---|---|
| PIPE-01 | Fetch from the same sources as today: kurser.dtu.dk (archive and course pages, including the cookie re-fetch), karakterer.dtu.dk (histograms), evaluering.dtu.dk (search plus result pages) and orbit.dtu.dk (teacher profiles). Prefer official DTU exports or APIs wherever DTU can provide them (approval is in place) | E | P0 |
| PIPE-02 | Store every raw response (HTML) by URL and fetch date, so parsers can be re-run without fetching again | N | P0 |
| PIPE-03 | Fetch politely and in parallel: limited concurrency (default 4 per host), retries with backoff, and a timeout on every request. Fix the requests that have no timeout today | F | P0 |
| PIPE-04 | Make runs incremental and able to resume: checkpoint after each item, and skip unchanged pages (ETag, hash or age). A crash must not lose the whole run | N | P0 |
| PIPE-05 | Parsers are pure functions from HTML to a typed record, tested against saved fixture pages. A parse failure marks that one record as failed; it does not crash the run. Named parser rules kept from v1: (a) an evaluation term is dropped if its answer counts don't sum to the stated number of responses (±2 tolerated) or if the answer labels don't match; (b) grades hidden when there are ≤3 students; (c) orbit photo matching by name variants plus an email check plus manual overrides | N/E | P0 |
| PIPE-06 | Validate each candidate release before it is published: schema checks, row counts vs the previous release (e.g. no more than 5% of courses lost), new unknown vocabulary values, and a sample diff report. If validation fails, the release is not published | N | P0 |
| PIPE-07 | Publish atomically. The site switches to a new release without restarting, and an admin can roll back to the previous release | N | P0 |
| PIPE-08 | Run on a schedule: info every week during registration periods and monthly otherwise; grades and evaluations after each exam or evaluation release. Admins can also trigger a run on demand | N | P1 |
| PIPE-09 | Compute course embeddings after each publish, only for courses whose text changed. Record the model name and version with each vector | E | P0 |
| PIPE-10 | Teacher photos are cached by our system, not hotlinked from orbit, and fall back to initials when there is no photo | F | P1 |

### 3.3 Search and discovery (SRCH)

| ID | Requirement | Origin | Pri |
|---|---|---|---|
| SRCH-01 | One search box that takes a course code (case-insensitive, prefix), a title (fuzzy, Danish and English) or a natural-language description (semantic). Pressing Enter on a name works (today it gives an error) | E/F | P0 |
| SRCH-02 | Filter facets: language, level, semester period (all 6, including July and August), schedule block (day and time and the exact slot code), exam type, grading scale, aids, assignments, examiner, campus, duration, study line, department. Every option the backend supports appears in the UI, and none is silently ignored. Exclusion filters `exclude_slots` / `exclude_days` ("avoid Friday mornings"): a course with any slot in the excluded set is left out | E/F/N | P0 |
| SRCH-03 | New range filters: ECTS, average grade, pass rate, rating, workload, minimum number of evaluation votes | N | P0 |
| SRCH-04 | "Fits my plan" filter: courses with no schedule clash and offered in the chosen plan term(s); a separate "prerequisites met" filter. Shared with PLAN-07 and `recommend_courses` | N | P0 |
| SRCH-05 | Facet counts next to each option, and a way out of zero results ("remove X to get N results") | N | P0 |
| SRCH-06 | Sort by every current key (code, name, rating, workload, grade, ECTS, students per term, schedule, fail rate, votes, learning, motivation, feedback) in either direction. The active sort is shown. Fix the "sort by code" and "sort by responsible" bugs. Sort presets with a published formula, available to the UI and to `search_courses` alike, e.g. `best_value` = rating z-score − workload z-score, among courses with ≥10 votes | E/F/N | P0 |
| SRCH-07 | The URL holds the whole search state, readably (`?lang=en&level=bsc&slot=E2A`). Refining keeps the current filters; today "Adjust filters" resets them | F | P0 |
| SRCH-08 | Server-side pagination or cursors. A results page is never more than 50 courses (today one page can be about 10 MB of HTML) | F | P0 |
| SRCH-09 | Summary stats for the result set (average rating, workload, grade, fail rate). Courses with 0% fail count in the average (today they are excluded, which biases it upward) | E/F | P0 |
| SRCH-10 | Random course ("surprise me"); `/random/` and `/course/xxxxx` keep working | E | P0 |
| SRCH-11 | Command palette (⌘K or `/`): jump to a course, plan or action | N | P1 |

### 3.4 Course page (CRS)

| ID | Requirement | Origin | Pri |
|---|---|---|---|
| CRS-01 | Stable URL `/course/{code}` (the old URL keeps working), plus `/course/{code}/{year}` for history. An unknown code returns HTTP 404 with suggestions | E/F | P0 |
| CRS-02 | An "At a glance" header in plain language: rating, workload, pass rate and grade, each with sample size and a percentile within the department or level. It also shows the schedule slot as a mini week grid, exam form, ECTS and language | N | P0 |
| CRS-03 | Primary action "Add to plan" (choosing the term) at launch (P0). "Compare" arrives with PLAN-11 and "Ask the assistant" with AGT-06 (both P1); the assistant button is shown only to signed-in users | N | P0/P1 |
| CRS-04 | Grade distribution chart with counts and percentages, and a history per term (grade, fail %, students). Show every term, marking re-exam-only terms rather than hiding them | E/F | P0 |
| CRS-05 | Evaluation breakdown for learning, motivation, feedback and workload: the distribution of answers (not only the mean), a per-term trend, and a warning when there are fewer than 10 votes | E | P0 |
| CRS-06 | Teachers (main and co-responsible) with photo and their other courses as links | E | P0 |
| CRS-07 | Prerequisites and subsequent courses as clickable chips and a graph. Each chip shows "✓ in your plan or passed" when a plan is active; this state is added in the browser, never in the cached page | E/N | P0 |
| CRS-08 | Every official text field (description, objectives, content, remarks, exam, aids, duration, location, study lines, home page, highlighted message), shown as sanitised HTML. Hide empty fields and never show "None" | E/F | P0 |
| CRS-09 | "Similar courses" from embeddings (P0). "Often planned together" from plans, k-anonymous, see PRIV-04 (P1) | N | P0/P1 |
| CRS-10 | Link back to the official kurser.dtu.dk page, the grade page and the evaluation page | E/N | P0 |
| CRS-11 | Explain each metric on the page (tooltip or popover with the formula and data source), so the legend is no longer only an image in the FAQ | N | P0 |
| CRS-12 | Per-course SEO: unique `<title>` and description, `Course` structured data, Open Graph image | F/N | P0 |

### 3.5 Recommender (REC)

| ID | Requirement | Origin | Pri |
|---|---|---|---|
| REC-01 | Recommendations from a "basket" of courses (current or historical) plus free-text interests. The score is cosine similarity on normalised embeddings, with 50/50 weighting between courses and text (configurable) | E | P0 |
| REC-02 | Leave out the basket courses and near-duplicates (similarity above a threshold recalibrated for the chosen embedding model; v1 uses 0.99). Also leave out point-lock courses (new) | E/N | P0 |
| REC-03 | Use the same filter set as search (SRCH-02/03). Remove the recommender's separate, divergent filter set and its language and July/August bugs | F | P0 |
| REC-04 | A per-input breakdown: how much each basket item contributed to each result | E | P0 |
| REC-05 | "Prerequisites met: N courses unlocked" based on the basket or plan | E | P0 |
| REC-06 | Import the basket by pasting text from studieplan.dtu.dk (extract codes, including codes with letters), or from a study line | E/F | P0 |
| REC-07 | The basket is the user's plan or "passed courses" set, so it is never lost on reload | N | P0 |
| REC-08 | Results are paginated (no 9001-item responses). Input limits: at most 20 basket items; free text is truncated to 256 tokens for recommendations and 128 tokens for search (bounded inference cost) | F | P0 |

### 3.6 Plan builder (PLAN), the core new capability

| ID | Requirement | Origin | Pri |
|---|---|---|---|
| PLAN-01 | Create a plan with a programme (optional), start term and title. It works anonymously (stored locally) and syncs to the account on login. Anonymous plans can be exported as a URL-encoded link and as JSON. After the first plan is created, the UI warns: "Saved in this browser only. Safari may delete it after 7 days without a visit; sign in or copy the link". Exported links and JSON never include grades. When a user who already has synced plans signs in with a local plan, they choose "Import as new plan" or "Discard"; plans are never merged silently | N | P0 |
| PLAN-02 | Multi-term timeline: Autumn → January → Spring → June (→ July/August), adding terms as needed. Drag courses between terms | N | P0 |
| PLAN-03 | Each course in a plan has a status: planned / enrolled / passed / failed, and optionally the grade (kept private) | N | P0 |
| PLAN-04 | Week grid per term showing the DTU slots (E1A–E5B, E7, F1A–F5B, F7), with clashes highlighted. 3-week courses go in their own lane | N | P0 |
| PLAN-05 | Checks that run live: schedule clash, exam clash (two courses in the same term with the same parsed exam slot code, e.g. `F2B`; "Decide with teacher" and "last day of the 3-week period" are not checked), mandatory prerequisite missing or recommended prerequisite missing (placed in a later term than the course), point-lock overlap ("not applicable together with"), course not offered in that term, and ECTS load per term (the plan's `ects_target`, default 30; warn more than 5 below or above it) | N | P0 |
| PLAN-06 | ECTS totals per term and overall (P0). Once CAT-16 exists, totals per programme category and progress towards the degree (P2) | N | P0/P2 |
| PLAN-07 | "Fill this term" suggestions: recommender results (REC) limited to free slots and met prerequisites | N | P0 |
| PLAN-08 | Import passed courses from studieplan.dtu.dk paste (REC-06). Export to PDF or image, CSV, Markdown (useful for AI tools, replacing the "CV helper"), and iCal for weekly slots when term dates are known | N/E | P1 |
| PLAN-09 | Multiple plans per user (e.g. "Plan A: AI track", "Plan B"), with duplicate and compare (`create_plan(from_plan_id)`, `get_plan(diff_against_plan_id)`). At most 20 plans per user and 100 items per plan | N | P1 |
| PLAN-10 | Undo and redo as restoring earlier plan versions ("restore version N"), including edits made by the assistant or an agent. The same model works for local plans in the browser | N | P0 |
| PLAN-11 | Compare 2–4 courses side by side (metrics, schedule, exam, prerequisites) | N | P1 |
| PLAN-12 | Multi-period courses and unknown schedules. A plan item has a start term and a span (1 or more consecutive periods) taken from the course duration (13/16/26 weeks, 3 weeks). ECTS are split across the spanned periods by duration. The normaliser stores `offered_periods[]` (alternatives, e.g. "Spring + Autumn") separately from `span` (e.g. "Spring + June" spanning both). If the period or slot is unknown, show "schedule unknown" and never block. Two 3-week courses in the same 3-week period always clash | N | P0 |

### 3.7 Sharing, templates and community (SHR)

| ID | Requirement | Origin | Pri |
|---|---|---|---|
| SHR-01 | Plan visibility: private (default), unlisted (anyone with the link) or public (listed in the gallery). Grades and pass/fail status are never shared unless the owner explicitly turns that on | N | P1 |
| SHR-02 | A shared plan has a read-only page with an OG preview image (timeline thumbnail) and buttons for "Use as template" (fork) and "Open in my assistant" | N | P1 |
| SHR-03 | Forking copies the plan into the user's account. The fork remembers where it came from, and the original shows how many times it was forked. Using a plan as a template shifts it by whole academic years only; if the forker starts in the other semester (Spring vs Autumn), terms are realigned with a warning on every course not offered in its new term | N | P1 |
| SHR-04 | Gallery of public plans filterable by programme, start year, specialisation and tags. Sort by forks, stars or recency. Starring a plan | N | P1 |
| SHR-05a | Programme starter templates: admin-authored first-year plans per BSc programme (mandatory and polytechnical-foundation courses), read-only, no gallery. Used by "Start your plan". Template items can be marked `suggested` (an elective the student is expected to swap), shown with a dashed outline | N | P0 |
| SHR-05b | Curated templates for the full degree, maintained by admins or student ambassadors and marked "Curated" in the gallery | N | P1 |
| SHR-06 | Anonymous, aggregated signals only above a k-threshold: "N students planned this", "Often planned together", "Popular electives in your programme" | N | P1 |
| SHR-07 | Report a plan; admins can hide it. A plan is hidden automatically only after 3 reports from accounts older than 7 days, and never if it is curated. Reports are rate-limited per user. Display names can't contain reserved words ("DTU", "official", "admin") or names of teaching staff (from `person`). Public plan titles and descriptions are length-limited and filtered for profanity in Danish and English | N | P1 |
| SHR-08 | Share URLs: public plans at `/p/{slug}`, unlisted plans at `/s/{slug}`. The prefixes are separate so the CDN rule can cache `/p/*` without ever caching an unlisted plan | N | P1 |
| SHR-09 | Unlisted-plan privacy: slugs are random (≥96 bits, base62) and can be reset. Unlisted and private pages send `X-Robots-Tag: noindex`, are left out of the sitemap and are served `Cache-Control: private, no-store`; only public plans may be cached by the CDN. Any edit or visibility change purges that plan's pages and OG image by its cache tag (`plan-{id}`) | N | P1 |

### 3.8 Accounts and identity (ACC)

| ID | Requirement | Origin | Pri |
|---|---|---|---|
| ACC-01 | Browsing, search, course pages, recommender and up to 5 local plans work without logging in. Opening a plan link adds a plan after a preview; it never overwrites one | E/N | P0 |
| ACC-02 | Sign in with DTU (OIDC with DTU's identity provider). The only data kept is a stable subject id, a display name and an optional programme. The student number is not stored | N | P1 |
| ACC-03 | Settings: display name (or "anonymous") for shared plans, programme, start term, language (en/da), theme | N | P1 |
| ACC-04 | GDPR self-service: export all my data (JSON) and delete my account. On deletion, public plans are either deleted or anonymised, as the user chooses | N | P1 |
| ACC-05 | Create, list and revoke connected-agent grants and personal access tokens (AGT-07) | N | P1 |

### 3.9 Agentic interface (AGT): MCP first, AI built in

| ID | Requirement | Origin | Pri |
|---|---|---|---|
| AGT-01 | A remote MCP server is the main machine interface. Target the latest MCP revision (2026-07-28 at the time of review: stateless, no sessions) and stay compatible with 2025-11-25 clients; pin an SDK version that supports it. Two endpoints: `/mcp/public` (no auth, public tools only) and `/mcp` (OAuth, all tools). Every read capability is available at P0; plan writes at P1 (AGT-03) | N | P0 |
| AGT-02 | Public tools (no login; on `/mcp/public` and `/mcp`), available at launch unless noted: `search_courses` (with a shared `CourseFilter` including `within_codes[]`, `teacher_id`, `fits_terms[]` and `prereqs_met`, plus facet counts and summary stats), `get_courses(codes[≤100], academic_year?, detail)`, `recommend_courses` (basket, interests, filter, multi-term fit), `validate_plan(plan_id\|plan_json\|plan_link, groups?)`, `make_plan_link(plan_json)`, `parse_study_planner_text`, `get_shared_plan(slug\|template_id, start_term?)`, `list_templates`, `get_dataset_info` (incl. valid filter values), and `export_plan` (iCal, M3). An agent can build a full plan and hand it over with no account (02-architecture §5.4) | N | P0 |
| AGT-03 | Sign-in tools (`/mcp` only): `get_me`, `update_me`, `get_plan` (history, diff against a version or another plan), `create_plan` (blank, from template, duplicate, fork, from plan JSON or link), `edit_plan` (atomic batch of ≤100 ops: `add`, `remove`, `update`, `set_meta`, `replace_items`; one undo step; only structural errors fail a batch; plan findings are issues with `fix_ops`), `restore_plan_version`, `star_plan`, `set_sharing`, `delete_plan`. Copy rules: a new plan is always private with grades unshared; forks drop grades, descriptions and statuses. Each tool is annotated (`readOnlyHint`, `destructiveHint`, `idempotentHint`) | N | P1 |
| AGT-04 | MCP resources `course://{code}`, `plan://{id}` (owner only) and `dataset://current`. Workflow guidance (e.g. "build my whole plan") lives in tool descriptions; MCP prompts are optional extras | N | P1 |
| AGT-05 | **Single tool registry**: each tool is defined once (name, description, JSON Schema in and out, handler). The same definition is exposed as (a) MCP tools, (b) OpenAI-compatible function or tool definitions at `GET /v1/tools`, and (c) REST endpoints in the OpenAPI spec | N | P0 |
| AGT-06 | Built-in assistant: a chat panel that calls any **OpenAI-compatible** Chat Completions endpoint (configurable base URL, model and key, e.g. a DTU-hosted model, Azure OpenAI EU, vLLM or Ollama), streams responses, and runs the same registry tools for the signed-in user | N | P1 |
| AGT-07 | MCP authorisation follows the MCP OAuth 2.1 profile. The auth broker is the authorisation server and federates sign-in to DTU (ACC-02); the api is the resource server and serves `/.well-known/oauth-protected-resource`. Client registration: Client ID Metadata Documents (preferred) and dynamic client registration (kept for clients that need it). PKCE. Scopes `plans:read`, `plans:write`. The api rejects tokens whose `aud` isn't the canonical `/mcp` URL (RFC 8707), so web-app tokens don't work at `/mcp` and the reverse. Personal access tokens are a fallback for clients without OAuth. If no broker passes the M0 spike, the deployment runs without one: web sign-in uses OIDC directly with DTU Entra ID, and `/mcp` accepts personal access tokens only | N | P1 |
| AGT-08 | Writes from the assistant or an agent show up in the web UI as a visible change ("Assistant added 02105 to Autumn 2027") that can be undone (PLAN-10). **Owner-approval actions** — `delete_plan`, `set_sharing` to unlisted or public, and `set_sharing(share_grades=true)` — are never applied by a tool call alone. The call returns `{status: "pending_approval", approve_url}` and the change happens only when the signed-in owner approves it in the web app (or, where the MCP client supports it, through an elicitation round-trip with a server-signed request state). A token held by an agent never counts as approval | N | P1 |
| AGT-09 | Tool outputs are compact and designed for agents: stable ids (including `item_id` per plan item), units in field names (`ects`, `pass_rate_pct`), a `data_as_of` field, pagination cursors, short `next_steps` hints, errors that say how to fix the call, and validation issues with machine-readable `fix_ops`. Writes return only the diff and new or resolved issues. Inputs are flat (no nested unions) and pass a CI check against OpenAI strict mode and one small open model | N | P0 |
| AGT-10 | The assistant has course context: on a course page it knows the course, in the planner it knows the plan. It answers from tool results and cites courses by code with links. It never invents grades or statistics | N | P1 |
| AGT-11 | Users can bring their own OpenAI-compatible endpoint or key (stored encrypted, optional). Otherwise the platform's default model is used, within quota (COST-02) | N | P2 |
| AGT-12 | "Connect your AI" page: step-by-step guides for Claude, ChatGPT, Cursor and VS Code, a one-click config snippet, and a list of the available tools | N | P1 |
| AGT-13 | **Capability parity, tested.** Every UI capability maps to a registry tool or is on an explicit web-only list with a reason (02-architecture §5.3). A CI test fails if a mapped tool doesn't exist or the web client calls an endpoint that is neither registry-generated nor on the allow-list | N | P0 |

### 3.10 Content and admin (ADM)

| ID | Requirement | Origin | Pri |
|---|---|---|---|
| ADM-01 | Pages for About, FAQ (rewritten), methodology ("how metrics are computed", generated from the shared formula module), data freshness and privacy | E/N | P0 |
| ADM-02 | Admin console: dataset releases (validation report, publish, roll back), pipeline runs and logs, reported plans, feedback triage, curated templates, LLM usage and cost | N | P1 |
| ADM-03 | Privacy-friendly usage analytics (cookieless, self-hosted Umami on the existing Postgres) replacing Google Analytics without consent, plus the bot/human page-view report that today's log analyser produces | E/F | P1 |
| ADM-04 | Remove leftover routes and pages: `/test`, `/wip`, `/recommender_v1`, `/recommender_v2`, dead templates | F | P0 |

### 3.11 Feedback (FDB)

| ID | Requirement | Origin | Pri |
|---|---|---|---|
| FDB-01 | **Inline data-issue reporting.** Course pages (`/course/{code}`) include a no-login "Report a data issue" control (kind: `schedule`, `prereqs`, `grades`, `other` + optional note), also exposed as `report_data_issue(course_code, kind, note?)` on `/mcp/public` and `/mcp` (AGT-13) | N | P0 |
| FDB-02 | **Global feedback and assistant ratings.** A no-login feedback popover in the footer and `⌘K` palette (bug, idea or data issue, free text, optional contact) and thumbs up/down with optional reason chips on assistant and `/discover` results. Listed on the `AGT-13` web-only allow-list | N | P1 |
| FDB-03 | **In-session micro-survey.** Triggered only by in-session value events (e.g. adding a 4th course or sharing/exporting a plan), never on landing and never via a multi-day visit counter. Stores only a `dismissed` preference flag in `localStorage` (ePrivacy Art. 5(3) strict necessity confirmed with the `PRIV-01` data controller) | N | P1 |
| FDB-04 | **Feedback privacy, routing and retention.** Anonymous by default; auto-captures scrubbed context (stripping `#v1…` plan hashes and redacting `/s/{slug}` link-only share URLs) and `sentry_event_id` when following an error; pushes notifications only to a private Turing Council Discord channel (never auto-creating public GitHub issues); purges or redacts the entire row after 90 days or upon resolution | N | P0 |

---

## 4. Non-functional requirements

### 4.1 Performance and capacity (PERF)

| ID | Requirement |
|---|---|
| PERF-01 | **Design load:** 3,000 concurrent active users and 300 requests/s sustained, with 3× burst headroom (about 1,000 requests/s), during course-registration weeks. Reasoning: about 13k DTU students, 20–25% active in registration week, about 1 request every 10 s per active user. |
| PERF-02 | API latency at design load: p95 < 150 ms for catalogue reads (search, course, recommender without free text); p95 < 400 ms for recommender and search with free text (needs an embedding), at a sustained 60 semantic queries/s; p95 < 250 ms for plan writes. |
| PERF-03 | Web: course pages are server-rendered and cached (CDN or edge) with Largest Contentful Paint < 2.0 s at p75 on mid-range mobile over 4G, and JS < 150 KB gzip for the course-page route. Course-page charts are server-rendered SVG with no charting library. Cached pages are an anonymous shell; user and plan state loads in the browser. |
| PERF-04 | Assistant: first token < 2 s at p95 (provider-dependent). At most 100 concurrent LLM streams per deployment; over that, queue or return a friendly 429. |
| PERF-05 | Catalogue reads never parse files during a request. The current release is held in memory and swapped atomically (PIPE-07). |
| PERF-06 | A full pipeline refresh takes < 2 h; an incremental run takes < 20 min. This relies on: one primed session cookie per host (no double GET per page), the `/info` page fetched once per course, and past academic years treated as frozen unless a refetch is forced. |

### 4.2 Scalability and availability (SCAL)

| ID | Requirement |
|---|---|
| SCAL-01 | API and web processes are stateless and run as N replicas behind a load balancer. No sticky sessions. |
| SCAL-02 | Availability. Browsing (search, course pages, recommender, local plans): 99.9% during registration weeks, carried by the CDN and each api node's local snapshot. Sign-in and saving: 99.5% monthly. Planned maintenance is announced and never scheduled in registration weeks. |
| SCAL-03 | Graceful degradation: if the LLM provider is down, everything except the assistant keeps working. If the database is down, catalogue reads keep working from memory and plan writes fail clearly (local plans are kept). |
| SCAL-04 | Backups: daily database dump with 30-day retention, and point-in-time recovery (RPO ≤ 15 min, RTO ≤ 2 h). Restores are tested every quarter. |

### 4.3 Security (SEC)

| ID | Requirement |
|---|---|
| SEC-01 | Scraped HTML is sanitised with an allow-list at ingest. Nothing is rendered with `|safe` or `dangerouslySetInnerHTML` without sanitising first. A strict Content Security Policy is in place. |
| SEC-02 | All input is validated against schemas. Search terms are never used as regular expressions (today they are, which allows CPU-exhausting patterns). Request bodies are limited in size. |
| SEC-03 | Rate limits. Cached anonymous catalogue reads have no per-IP limit (the CDN absorbs them; thousands of students share a few DTU NAT/eduroam IPs). Uncached expensive calls (semantic search, recommend, MCP `tools/call` on `/mcp/public`) are limited per IP at about 10/s with a high burst, and DTU egress ranges are allow-listed. Anonymous feedback (`report_data_issue` and `POST /api/feedback`) uses the same DTU egress allow-list with a loose per-IP limit for other IPs, plus a honeypot field and a 4 KB body cap. Writes (5 calls/s; a batch of up to 100 ops counts as one call) and the assistant (COST-02) are limited per user or token. Limits are cost-weighted per tool, and calls carrying code lists or plans are never treated as cached reads. The real client IP is read only from the CDN's header, trusting only the CDN's IP ranges. |
| SEC-04 | Secrets come from the environment or a secret manager; there are no fallback default keys. Debug is never on in production. Dependencies pinned and scanned (Dependabot or Renovate). |
| SEC-05 | OAuth 2.1 with PKCE. Tokens are short-lived with refresh, and scoped. Personal access tokens are stored hashed and can be revoked. Every write records who made it (user, agent client, or assistant). |
| SEC-06 | Prompt injection. The real control is authorisation: a caller can write only to its own plans, and owner-approval actions (AGT-08) need the human in the web app. On top of that, every tool output puts user-generated text (shared plan titles and descriptions) and scraped DTU free text in fields named `*_untrusted`, and tool descriptions say to treat them as data; this applies to third-party agents as well as the built-in assistant. |
| SEC-07 | OWASP ASVS level 2 for the web app. Security headers, CSRF protection on cookie-authenticated writes. |

### 4.4 Privacy and compliance (PRIV)

| ID | Requirement |
|---|---|
| PRIV-01 | GDPR: one named legal entity is the data controller (decided before M3), with a documented processing record and data processing agreements with the host, database, Sentry and LLM provider. EU data residency for the database and the LLM provider (or a DTU-hosted model). |
| PRIV-02 | Data minimisation: no student number, no email unless needed for a notification the user opted into, and grades in a plan stay private. |
| PRIV-03 | No third-party tracking. Cookies are strictly necessary only, so no consent banner is needed. |
| PRIV-04 | Aggregate signals (SHR-06, CRS-09) are shown only for groups of 10 or more distinct plan owners, counting at most one plan per owner, and only from public or unlisted plans, or from private plans whose owner opted in (`aggregate_opt_in`, off by default). |
| PRIV-05 | Assistant conversations are kept for at most 30 days by default. Users can delete them, and they are not used for training. This is configured with the provider. |
| PRIV-06 | Teacher data is limited to what DTU publishes (name, role, public profile photo). |

### 4.5 Usability, accessibility and internationalisation (UX)

| ID | Requirement |
|---|---|
| UX-01 | WCAG 2.2 AA: every control reachable and usable by keyboard, visible focus, labelled inputs, alt text, 4.5:1 contrast. Colour is never the only signal (grade bars and clashes also use a pattern or text). |
| UX-02 | Mobile-first and responsive from 320 px; no horizontal scroll; the planner works on a phone (list view instead of the week grid). |
| UX-03 | Light and dark themes (follow the system setting, with a manual switch). |
| UX-04 | English and Danish UI. Course content is shown in the language DTU provides. |
| UX-05 | Every metric shows its sample size and data freshness. Small samples are visibly marked. |
| UX-06 | No personal branding or emoji used as labels. The visual language is consistent and calm (design tokens, one component library). |

### 4.6 Maintainability and operability (OPS)

| ID | Requirement |
|---|---|
| OPS-01 | One repository with typed code (Python type hints with mypy or pyright; TypeScript strict). CI runs lint, type checks, unit tests, parser fixture tests and an end-to-end smoke test on every PR. |
| OPS-02 | Domain logic (metrics, clash detection, prerequisite checks) lives in one module with ≥ 90% test coverage. The UI, REST, MCP and the assistant all call it. |
| OPS-03 | Observability: structured JSON logs, request ids, error tracking (Sentry), metrics (RED per endpoint, LLM tokens and cost, pipeline stats), uptime checks. Alerts go to a Turing Council channel. |
| OPS-04 | The full stack runs locally with one command (`docker compose up`), with a seeded snapshot of the dataset. |
| OPS-05 | Deploys use CI/CD to staging, then production, with zero-downtime rolling deploys and database migrations that are backwards-compatible for one release. |
| OPS-06 | Contributor docs: architecture overview, how to add a tool, how to add a data source, runbook. |

### 4.7 Cost (COST)

| ID | Requirement |
|---|---|
| COST-01 | Infrastructure for non-LLM traffic ≤ about €200/month at design load including VAT, Sentry Team, staging, load balancer and CDN, or DTU-hosted resources where available. Start with 2× 4 vCPU nodes and resize from the M1 load test. |
| COST-02 | LLM spend is capped: per-user daily token quota (default 50k tokens/day), a global monthly budget with alerts at 50/80/100%, and a cheaper model for simple turns where the provider supports routing. |
| COST-03 | Embedding inference runs locally (small ONNX model, CPU). Using it costs no API fees. |

### 4.8 Data quality (DQ)

| ID | Requirement |
|---|---|
| DQ-01 | Each release passes the validation gates in PIPE-06. A human looks at the diff report before the first publish of each semester. |
| DQ-02 | The derived metrics match the current site for a reference set of 50 courses, or the difference is explained (because a bug was fixed). |
| DQ-03 | There are regression tests for every bug listed in §5. |

---

## 5. Bugs in the current system that must not come back (DQ-03)

Each line is a test case:
1. Study-line filter keys `BENG_CE`, `BENG_ITE2`, `BSC_BI`, `BSC_MT1` match nothing.
2. Recommender language filter sends `Danish`/`English` but the data keys are `dk`/`eng`.
3. Recommender ignores the July and August periods.
4. Course codes with letters: the navbar search doesn't upper-case its input and requires exactly 5 characters; `/course/<id>` is case-sensitive; pasted-text extraction uses `\d{5}`, so codes like `KU181` are missed. (The autocomplete itself is already case-insensitive.)
5. Search terms are passed to pandas as regular expressions.
6. `sort_by=course_numbers` returns 0 results; "sort by responsible" sorts by photo URL.
7. The course page Objectives row checks a field that doesn't exist, so it always renders and can print "None".
8. `gradegraph.js` is missing, so every course page logs a 404 and the bars are sized wrong.
9. "Overworked %" divides by learning votes instead of workload votes.
10. Average stats leave out courses with 0% fail.
11. ECTS values like `7,5` are read as `75`, which the code then divides by 10 to hide the error. Parse decimals correctly instead.
12. Spring evening slot: `Spring E7` vs `Spring F7` mismatch means F7 can never match.
13. Grade label "Ikke Godkendt" never matches because of capitalisation.
14. Mandatory prerequisites are left out of the embedding text, and the text clean-up strips digits and æ/ø/å.
15. Historical years show today's teachers.
16. Unknown courses return HTTP 200, and JSON errors return 200.
17. `offschedule` is used in two categories and the query parser can't tell them apart.
18. Grade-bar colours: Pass and 7 share a colour, and the percentage labels are transparent.
19. Courses with no grade data are shown as "0% fail" and sort as 0. They must show "No data".

## 6. Out of scope (on purpose)

- **Real-time collaborative editing of one plan** (Google-Docs style). Forking and sharing cover the need.
- **Native mobile apps.** The responsive web app covers it.
- **Registering for courses on DTU's behalf.** We link to DTU's own registration instead.
- **Other universities.**
- **Chat between users, comments on plans, follower graphs.** Revisit only if the gallery gets traction.
- **A recommendation model beyond embeddings plus aggregate co-occurrence.** No collaborative-filtering training pipeline.
