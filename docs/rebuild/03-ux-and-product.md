# DTU Course Analyzer v2: UX, UI and product plan

Status: draft for review · Date: 2026-11-21 · Requirements: [01-requirements.md](01-requirements.md) · Architecture: [02-architecture.md](02-architecture.md)

## 1. Problems we are fixing

The current site is built for browsing. A student can look courses up but can't make a decision with them or act on it. These problems come from [extraction/extract_frontend.md §5](extraction/extract_frontend.md):

- **Landing.** A student who arrives on `/course/<id>` gets no orientation and no verdict at a glance. The metric legend is only an image in the FAQ, and the prerequisite codes aren't clickable.
- **Planning.** There is nothing for it: no plan, no clash check, no ECTS total, no saving, no sharing. The recommender basket is lost on reload.
- **Finding courses.** Filters are split between Home and the recommender modal, and the two sets don't match. "Adjust filters" resets everything. Zero results is a dead end. Enter on a course name gives an error.
- **Visual debt.**
  - a mix of Bootstrap alpha and BS4 class names
  - emoji used as labels
  - personal branding
  - one column below 1,400 px
  - no dark mode
  - layouts that break on mobile
- **Accessibility.**
  - Clickable images and toggles can't be focused.
  - Labels are images.
  - Alt text is missing.
  - Contrast is low.

## 2. Product principles

1. **Decide, then act.** Every course view answers "is this good for me, and does it fit?" and offers "Add to plan" right there.
2. **The plan is the centre.** Search, the recommender, the assistant and templates all read from and write to the current plan.
3. **Agent-equal.** Anything you can click, an agent can do through MCP, including building a whole plan in one step, and anything an agent does shows up in the UI and can be undone in one step. The few web-only actions (approvals, account, tokens, reports) are listed with reasons and checked in CI (AGT-13).
4. **Honest numbers.** Every number shows its sample size, how fresh it is, and how it is calculated.
5. **No account needed to get value.** Login only adds syncing, sharing and the built-in assistant.

## 3. Information architecture

```
/                       Home: search-first, "Start your plan" CTA, programme templates, "Connect your AI"
/search?…               Unified search + facets (replaces Home filter form + /browse + recommender filters)
/course/{code}          Course page (landing page from kurser.dtu.dk and Google)
/compare?c=…            Side-by-side 2–4 courses
/plan                   My plans (local or synced) → /plan/{id} planner
/plan/import#v1.…       Import a plan from a link (preview first; adds, never overwrites)
/p/{slug}               Public shared plan (read-only, fork; CDN-cached)
/s/{slug}               Unlisted shared plan (read-only, fork; never cached, noindex)
/templates              Gallery: curated + community plans, filter by programme/start year
/discover               Recommender ("Find courses like…"), now feeding into the plan
/connect                Connect your AI (MCP setup, tool list, tokens)
/about /methodology /faq /privacy /data
```

- **Global header:** search with a ⌘K command palette; a "My plan" pill showing the active plan's name, the current term's ECTS and a warning count; the assistant toggle (signed-in users, from M3); language and theme switches; sign in. Everything user-specific in the header loads in the browser, because the page itself is a cached anonymous shell.
- **Global footer:** the dataset's data-as-of date and a link to the methodology page.

## 4. Key journeys

### J1. A new student arrives from kurser.dtu.dk (the main journey)

With DTU's approval, ask DTU to add a "See grades, reviews & plan → DTU Course Analyzer" link on each kurser.dtu.dk course page, pointing to `/course/{code}?ref=kurser`.

1. **Course page, first screen** (mobile and desktop):
   - Title, code, ECTS, level, language, campus.
   - **At-a-glance strip** of 4 tiles:
     - Rating 4.1/5, top 20% in the department, 812 responses
     - Workload "Higher than expected" with its distribution sparkline
     - Pass rate 91%
     - Average grade 7.4
   - Each tile has an ⓘ popover showing the formula and data source.
   - **Schedule chip** ("Autumn · Mon 8–12 · E1A") with a mini week grid.
   - **Exam chip** ("Written, 4h, all aids").
   - **Primary button** "Add to plan" opens a term picker, with a sensible default: the next term in which the course is offered. From M3: secondary buttons "Compare" and, for signed-in users only, "Ask about this course" (anonymous visitors never hit a sign-in wall on the landing page).
2. **The first "Add to plan" without an account:**
   - A small sheet asks "Which programme and start term?" and can be skipped.
   - If a programme is chosen, the plan starts from that programme's **starter template** (SHR-05a): the first-year mandatory and polytechnical-foundation courses are already placed, and this course is added on top.
   - The plan is created locally, and a toast offers "Open plan".
   - A persistent note says "Saved in this browser only. Safari may delete it after 7 days without a visit." It offers "Copy link" (URL-encoded plan, never including grades) and "Sign in with DTU to sync".
   - Signing in later with a local plan while synced plans already exist asks "Import as new plan" or "Discard"; nothing is merged silently.
3. **Course page sections** (sticky tabs on mobile):
   - **Overview:** description and learning objectives.
   - **Grades:** accessible bar chart, per-term trend, re-exam terms marked.
   - **Reviews:** distributions for learning, motivation, feedback and workload, plus trend.
   - **Prerequisites:** chips with ✓ / ✗ against the plan, plus the graph.
   - **People.**
   - **Similar & often taken with.**
   - **Official info:** every DTU field, and a link back to kurser.dtu.dk.
4. **Getting from one course to a whole term.** With the starter template the mandatory courses are already in place, so the student only chooses electives. In the planner, the term shows "22.5 / 30 ECTS". An empty-slot card says "Fill Autumn 2027 →", which opens `/discover` already limited to that term, free slots and met prerequisites (PLAN-07). Students add courses until the term reaches 30 ECTS, and clash or prerequisite warnings update live.
5. **Save and share:** "Sign in with DTU to sync", then "Share" (unlisted link) to send to a study buddy or counsellor.

**Target:** starting from the starter template, a new student goes from landing to a first term with no clashes in under 5 minutes (M2 exit criterion, launch).

### J2. Start from a template

On `/templates`, the student filters by programme ("BSc Software Technology, start 2027") and opens a curated template. Each card shows a timeline thumbnail, total ECTS, number of forks and a "Curated" badge.

They click **Use this template**:
- It is forked into "My plans" and shifted by whole academic years to the student's start. If the student starts in the other semester, terms are realigned and every course not offered in its new term is flagged.
- The planner opens with every term filled in, and the electives are marked as "suggested" (dashed outline) so they can be swapped.

### J3. Plan with an AI agent (agent-first)

1. On `/connect`, the student sees:
   - **Two MCP URLs**, each with a copy button: `/mcp/public` (no sign-in, courses and plan checks) and `/mcp` (sign in with DTU, edit your plans)
   - **Client tabs:** Claude, ChatGPT, Cursor and VS Code, each with a config snippet or a one-click "Add to …" deeplink where the client supports one
   - **Tool list:** each tool's description
   - **Advanced:** a personal access token
2. In their own agent: "Plan my 2nd semester: I like ML, avoid Friday mornings, max 30 ECTS."
   - The agent calls these tools: `get_plan`, `recommend_courses` (interests text, `exclude_days`, `fits_terms`, `prereqs_met`), then one `edit_plan` with 3 `add` ops. Any issues come back with `fix_ops` it can apply in one more `edit_plan`.
   - OAuth consent happens once, with scopes shown in plain words.
   - If the agent tries to publish, share grades or delete, it gets an approval link instead. The planner shows "Claude wants to make this plan public · Approve / Decline", and nothing happens until the student approves.
3. Within about 5 s the web planner shows the change bar "Claude added 02450, 02456, 42500 to Spring 2028 · Undo". The whole batch is one history entry, so one Undo reverts it.

### J3b. The brave new student: a whole plan from their own AI, no account needed

Before starting at DTU, a student asks their Claude: "Build my full BSc plan for Software Technology. I like ML and security, no Friday mornings."
1. The agent connects to `/mcp/public` (no sign-in). It reads valid study-line codes from `get_dataset_info`, gets the programme's courses with `search_courses(filter.study_lines)`, and may add courses it read itself from the study regulations, hydrated with `get_courses(codes)`.
2. It starts from the programme's starter template (`get_shared_plan(template_id, start_term)`), then ranks electives for every term in one `recommend_courses` call (`within_codes`, `fits_terms`, `prereqs_met`).
3. It calls `validate_plan(plan_json, groups)` to get clashes, missing prerequisites and ECTS per term and per category it supplied (shown as sums only; no degree verdict until programme rules exist), applies the `fix_ops`, and calls `make_plan_link`.
4. The student opens the link. The import page shows a preview ("From a link: 34 courses, 6 terms · 2 warnings"), with the plan's text shown as plain text and a "came from a link" banner, then adds it as a local plan. Nothing touches their existing plans.
5. Later, signed in, the same agent can save it with `create_plan(from_plan_link)` and keep editing with `edit_plan`.

About 5–6 calls in total.

### J4. Built-in assistant

The assistant is a docked side panel on desktop and a bottom sheet on mobile. It knows the current context: the course you are on, or the plan you have open.

- **Suggested prompts change with the context.** On a course page: "Is this course hard?", "What should I take with it?". In the planner: "Fix my clashes", "Fill this term with ML electives".
- **Tool calls appear as compact, expandable steps** ("Searched courses · 24 results"), so users can check its work.
- **Answers cite courses as chips** that link to the course page and have "Add" buttons.
- **Writes appear as diff cards** ("+ 02450 → Spring 2028"). These are applied immediately and can be undone by restoring the previous version. Publishing, sharing grades and deleting show an Approve button and are never applied without it.
- **Quota.** A quiet "N messages left today" appears when the user gets close to the limit. If the LLM is down: "Assistant unavailable, everything else works."

### J5. Share and get feedback

- **Share dialog.** Three visibility options: Private, Link-only or Public in gallery. Link-only uses a random, unguessable link that is not indexed and can be reset ("Reset link"). Next to them: a "Show my grades" toggle (off by default and visibly separate), an editable title, a description (500 characters), and tags (specialisation, exchange, part-time).
- **Shared page** (`/p/{slug}` if public, `/s/{slug}` if link-only) shows:
  - the read-only timeline and week grids
  - the author (display name or "Anonymous DTU student")
  - "Use as template", "Star" and "Open in my AI" (copies an MCP prompt that references `plan://…`)
  - an OG image of the timeline, so links preview nicely in chat apps

### J6. Compare

Students pick up to 4 courses from search results, course pages or the plan; a "Compare (3)" tray then appears at the bottom.

The comparison is a table with one row per field: metrics with distributions, schedule (with clashes against the plan highlighted), exam, prerequisites met, ECTS and language. Each column has an "Add to plan" button.

## 5. Screen notes

### 5.1 Search `/search`
- **Layout:** facets in a left rail on desktop and a bottom-sheet filter on mobile; each facet shows its count.
- **Range sliders:** ECTS, rating, workload, pass rate, minimum votes.
- **Chips row:** the active filters as removable chips, plus a "Clear all" button.
- **Zero results:** instead of a dead end, show "Remove *Friday* (→ 14 results)" suggestions.
- **Result card** (2–4 columns from 768 px):
  - code and title
  - tiles: rating, workload, pass rate, ECTS
  - schedule chip and exam chip
  - buttons: "+ Plan", "Compare"
  - a clash badge if the course conflicts with the active plan term
- **Sort control:** shows the current sort and its direction. Presets such as "Best value" (rating z-score − workload z-score, ≥10 votes) have a published formula and are available to agents through `search_courses` too (SRCH-06).

### 5.2 Planner `/plan/{id}`
- **Desktop.** A horizontal timeline of term columns. Each column header shows the term name, an ECTS bar (green at 25–35) and its warnings. The courses below are draggable cards with a status pill. A toggle switches to a week-grid view of the selected term: DTU slots E1A–E5B as a Mon–Fri × morning/afternoon grid, an evening row, and 3-week periods as separate lanes. Clashes are shown with a red hatched overlap and a text label, never colour alone.
- **Mobile.** Terms are stacked accordions; the week grid becomes a per-day list. Courses move between terms through a "Move to…" menu instead of dragging.
- **Non-drag alternative everywhere.** Every course card has a "Move to…" menu on all viewports, for pointer and keyboard users (WCAG 2.5.7). Dragging is an enhancement.
- **Spans.** 16- and 26-week courses, and courses that span e.g. Spring + June, are drawn across every period they occupy, with their ECTS split. A course with an unknown schedule shows "Schedule not published" and is never blocked.
- **Validation panel.** It groups the issues as Clashes, Prerequisites, Not offered, Point-lock overlap and ECTS load. Each issue has a fix action, such as "Move 02105 to Autumn" or "Find alternative".
- **History drawer.** Lists the plan's versions with who made each change (you, Assistant, Claude via MCP) and the diff. Any version can be restored.

### 5.3 Course page `/course/{code}`
Covered in J1.
- **History:** a year switcher (`/course/{code}/{year}`) for historical offerings.
- **Discontinued courses:** a banner pointing to the successor (any course that lists it in `previous_codes`).

### 5.4 Home `/`
- **Hero:** a large search box with the line "Search 1,750 DTU courses by name, code or what you want to learn".
- **Two entry cards:** "Start your plan" (choose programme and start term) and "Browse templates".
- **Strip:** "Popular this term", showing k-anonymous data only once it exists; curated before that.
- **Small banner:** "Use it from your AI assistant →" (links to `/connect`).
- No personal branding. Credits move to `/about`.

### 5.5 States

| Situation | What the user sees |
|---|---|
| Course with no grades or no evaluations (new course) | Tiles show "No data yet (new course)", never 0 |
| Grades hidden by DTU (≤3 students) | "Hidden by DTU (fewer than 4 students)" |
| Statistics inherited from a predecessor course | Tiles labelled "as 0xxxx" with a link |
| Schedule or period unknown | "Schedule not published"; the plan check warns but doesn't block |
| First visit to `/plan` | Empty state with two buttons: "Start from my programme" and "Start blank" |
| API unreachable | Planner keeps working from localStorage and shows "Checks paused, server unreachable"; search shows a retry |
| Assistant tool call failed | The failed step is shown inline with its error and a "Retry" button |
| LLM provider down | "Assistant unavailable, everything else works" |
| Empty or new gallery | Curated and starter templates only, with "Share yours" |
| Zero search results | Suggestions to remove a filter, with the count each would give |

## 6. Design system

- **Components.**
  - Built from shadcn/ui (Radix primitives, so accessible by default) with Tailwind, styled through tokens.
  - One icon set (Lucide).
  - No emoji as labels.
- **Colour.**
  - Neutral surfaces with DTU red (`#990000`) used only as the brand accent.
  - The metric scales use a colour-blind-safe palette (viridis-like) with numbers always visible.
  - Light and dark themes come from the same tokens and follow the system setting.
- **Typography.** A system font stack with tabular numbers for metrics; base size 16 px.
- **Charts.**
  - Course-page charts (grade distribution, evaluation bars, trends) are server-rendered SVG with no charting library (PERF-03). A small library is allowed only on interactive planner views, loaded lazily.
  - Every chart has a data table fallback and an `aria-describedby` summary.
- **Motion.** Subtle and short: drag lifts, list reordering, toasts. All of it respects `prefers-reduced-motion`.
- **Copy.**
  - Plain English and Danish, using the words students use ("workload", "pass rate").
  - Every metric has a one-line "what this means" explanation.

## 7. Sharing and template model (product rules)

| Rule | Why |
|---|---|
| Private by default; sharing is an explicit action | Plans can reveal grades or failures |
| Grades are never shared unless "Show my grades" is turned on, separately for each share | Data minimisation (PRIV-02) |
| A fork records its source (`forked_from`) and the original shows the fork count | Credit to the author and a quality signal for the gallery |
| "Curated" badge only for admin or ambassador templates, which are reviewed each academic year | Trust; prevents outdated paths |
| Gallery ranking = stars + forks with a time decay, and curated templates are pinned per programme | Keeps fresh and useful plans visible |
| Aggregates ("N students planned this", "often taken together") only for groups of 10 or more distinct plan owners, one plan per owner, from shared plans or private plans whose owner opted in | k-anonymity (PRIV-04); counting plans would let one user fake a group |
| A plan is hidden after 3 reports from accounts older than 7 days, until an admin reviews it; curated plans are never auto-hidden; display names can't use reserved words or staff names | Moderation that scales with few admins without letting 3 new accounts take any plan down |
| Templates "shift" to the forker's start term and flag courses no longer offered | Templates stay usable across years |

## 8. Success metrics

| Metric | Target 3 months after launch |
|---|---|
| Course page → "Add to plan" conversion | ≥ 15% of visits that arrive from kurser.dtu.dk |
| Students with a synced plan | ≥ 2,000 |
| Time to a first term with no clashes (usability test) | < 5 min median |
| Plans shared (unlisted + public) | ≥ 10% of synced plans |
| MCP-connected users (distinct tokens used weekly) | ≥ 300 |
| Assistant: share of answers where the user later added a course mentioned in that answer | Track; target set after 1 month of data |
| Accessibility | 0 serious axe violations on key routes; screen-reader walkthrough of J1 passes |
| p75 LCP on course page (mobile) | < 2.0 s |
