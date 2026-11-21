# Review of the DTU Course Analyzer v2 rebuild plan

Scope: docs/rebuild/01-requirements.md (REQ), 02-architecture.md (ARCH), 03-ux-and-product.md (UX), the extraction reports, and spot checks of the v1 code. Licensing and DTU approval were not reviewed.

**v1 claims I checked against the code or data (all confirmed unless noted):**
- §5 bug 1: the four study-line keys. I diffed the `home.html` checkbox names against `json_filter_dct.json`. The extra keys BENG_NAOE, MSC_SES, MSC_SET and MSC_SFA are also missing from the UI.
- Bug 2: `recommender.html:1739`, `name="Danish"`.
- Bug 3: `recommender.html:878`, `addFilter('semester_period',…,4)`.
- Bug 4: `search.py:7-20` (no upper-casing) and `recommender.py:152` (`\d{5}`). The bug is only partly as stated; see finding 37.
- Bug 5: `recommender.py:105-107`, `str.contains` without `regex=False`.
- Bug 6: `course_data.json` has no `course_numbers` key, and `course_responsible` holds photo URLs.
- Bug 7: `course.html:642`.
- Bug 8: `course.html:1216`. `assets/dist` is absent.
- Bug 9: `course.html:203-207`.
- Bug 10: `context_dicts.py:194`.
- Bug 12: `format_info.py:97` against `dtu_consts.py:107`. F7 has 0 courses in the filter JSON.
- Bug 13: `scrape_grades.py:70` and `:119`.
- Bug 14: `recommend_utils.py:252`, which passes `InfoConsts.mandatory_prerequisites` without `.key_df`.
- Bug 16: `course_database.py:72`.
- Bug 17: `info_consts.py:262,295`.
- Bug 18: `course.html:81,107`.
- "195 codes contain letters": 195 of 1,746 in `course_df.csv`.

MCP claims were checked against the published spec pages for 2025-11-25 (authorization) and 2026-07-28 (changelog).

Findings are ordered from most to least severe.

---

1. **Blocker · ARCH §6 "Agents outside the app (MCP)", REQ AGT-08 / SEC-06: MCP confirmation has no human in the loop.**
   - **Problem:** The confirmation token goes back to the agent, and the agent can call the tool again with it. Client-side tool approval happens before the first call, not before the retry. A prompt injection in a shared plan description (returned by `get_shared_plan`) can therefore make a private plan public, and a plan with `share_grades` turned on publishes grades.
   - **Evidence:** ARCH §6: "The token is returned to the agent, and the human approves the action in their MCP client (the client confirms tool use)". The tools `set_plan_visibility` and `fork_plan` are both MCP-exposed (ARCH §5).
   - **Fix:**
     - Over MCP, a call to `set_plan_visibility(public|unlisted)`, to any change of `share_grades`, or to a future `delete_plan` returns `{status:"pending_approval", approve_url:"https://…/plan/{id}/approve/{id}"}`. The change is applied only after the signed-in owner clicks Approve in the web app.
     - Where the client advertises elicitation, use the 2026-07-28 Multi Round-Trip Requests `input_required` elicitation instead, with an HMAC-signed `requestState`, because no session exists.
     - Replace the ARCH §6 sentence with this rule, and add to AGT-08: "An agent-held token never counts as confirmation."

2. **Blocker · ARCH §3.1 data model: the catalogue has no academic-year dimension.**
   - **Problem:** CAT-01 stores courses "per academic year (2020/21 onward)", CRS-01 and UX §5.3 need `/course/{code}/{year}`, and CAT-07 needs teachers per offering. The schema keys courses only by `(release_id, code)`, so one release can hold only one year. This is an M0 schema decision.
   - **Evidence:**
     - ARCH:73-77: `course(release_id, code, …, PK(release_id, code))`
     - ARCH:80: `course_person(release_id, code, person_id, …)`
   - **Fix:**
     - Add `academic_year` to `course` and `course_person`, with `PK(release_id, academic_year, code)`.
     - Make `prereq_edge` and `course_study_line` year-scoped.
     - Define the "current" offering as the latest year per code; the Snapshot dict is code → latest offering, plus a side index for history.

3. **Major · REQ PLAN-02/04/05, ARCH §3.2 `plan_item(term)`: the plan model can't represent a large share of real courses.**
   - **Problem:**
     - A `plan_item` sits in exactly one term. The data has 53 courses that run 16 weeks and 17 that run 26 weeks.
     - Combined periods are common: Spring+Autumn (82 courses), Spring+June (29), Autumn+January (21).
     - 195 courses (11%) have SEMESTER_PERIOD "Unspecified", and 335 have "xweek" duration.
     - With no rules for these cases, PLAN-05's "not offered in that term", ECTS-per-term and clash checks will give wrong results.
     - A further ambiguity: "Spring + Autumn" means "offered in either", but "Spring + June" usually means "spans both".
   - **Evidence:** Counts from `course_df.csv` (SEMESTER_PERIOD) and from `json_filter_dct.json` (`course_duration`): `{'13week':1092,'3week':249,'16week':53,'26week':17,'xweek':335}`.
   - **Fix:** Add a PLAN-12 requirement:
     - "A plan item has a start term and a span (1 or more consecutive periods) taken from COURSE_DURATION. ECTS are split across the spanned terms by duration."
     - "If the period or slot is unknown, show a 'schedule unknown' warning and never block."
     - "Two 3-week courses in the same 3-week period always clash."
     - In the normaliser, decide per course whether the listed periods are alternatives or a span, and store `offered_periods[]` separately from `span`.

4. **Major · UX J1 step 4, ARCH §13 M2 exit criterion, REQ SHR-05 (P1) / CAT-16 (P2): the main journey can't succeed by M2.**
   - **Problem:** A new BSc student's first semester is mostly fixed mandatory or polytechnical-foundation courses. Without programme rules or curated templates, "build first semester" means guessing electives. M2's exit test depends on things that only arrive in M4 (curated templates) or "Later" (CAT-16). J1 step 4 also depends on PLAN-07, which is P1.
   - **Evidence:**
     - ARCH:322 (M2 exit): "A new student completes 'build first semester' in under 5 min"
     - ARCH:324: curated templates are in M4
     - UX:79: "opens `/discover` … (PLAN-07)", while REQ:148 lists PLAN-07 as P1
   - **Fix:**
     - Split SHR-05 into SHR-05a "admin-authored programme starter templates (first year per BSc programme), read-only, no gallery", at P0 in M2, and SHR-05b for community curation in M4.
     - Make PLAN-07 P0.
     - Change the M2 exit criterion to: "starting from the programme starter template, a new student adjusts electives and gets a term with no clashes in under 5 min".

5. **Major · ARCH §5.1 / §7, REQ AGT-01/AGT-07: the MCP spec target is out of date.**
   - **Problem:** The current revision, 2026-07-28, makes MCP stateless at the protocol level. It removes sessions and `initialize`, adds `server/discover`, and replaces server-initiated elicitation with Multi Round-Trip Requests. It also deprecates Dynamic Client Registration in favour of Client ID Metadata Documents (CIMD).
     - The plan talks about the SDK's "stateless Streamable HTTP mode" and treats DCR as the main registration path.
     - It never mentions RFC 8707 audience binding. The 2025-11-25 spec says "MCP servers MUST validate that access tokens were issued specifically for them".
     - AGT-07 lists "dynamic client registration and PKCE" but not CIMD.
   - **Fix:**
     - AGT-01: "Implements MCP 2026-07-28 and stays compatible with 2025-11-25 clients."
     - AGT-07: "CIMD (preferred) and DCR (kept for clients that need it, e.g. ChatGPT, verify at M3 start). The api checks that `aud` equals the canonical MCP URL. Tokens issued for the web app are rejected at /mcp and the reverse."
     - Pin an MCP Python SDK version that supports the chosen revision.

6. **Major · ARCH §1 "Auth broker: Zitadel or Keycloak", ARCH §14: the broker choice is unverified and undecided.**
   - **Problem:**
     - Keycloak's own MCP guide (26.7) lists RFC 8707 resource indicators and CIMD as experimental features behind flags.
     - Sources disagree on whether Zitadel supports DCR. Its docs say yes, disabled by default; a July 2026 compatibility table says no.
     - The architecture says the broker provides "PKCE, dynamic registration and client metadata documents" as if this were settled, and picks no product.
   - **Fix:**
     - Add to M0: "Auth spike (3 days). Pick one broker. Prove that Claude and ChatGPT complete OAuth against a stub /mcp, with the token's `aud` bound to /mcp, while federating to a DTU Entra test tenant."
     - Record the decision and the exact version and feature flags (for Keycloak, `--features=cimd,resource-indicators` or an Audience-mapper fallback).

7. **Major · ARCH §7 "Read-only course tools also work without a token": mixing anonymous and authenticated use on one `/mcp` URL is fragile.**
   - **Problem:**
     - MCP authorisation applies at the HTTP-transport level for the whole server, and clients usually decide whether to run OAuth when they connect.
     - Under 2026-07-28, "List endpoints … no longer vary per-connection", so anonymous callers see the write tools too.
     - They then hit a 401 in the middle of a conversation, which many clients surface as an error instead of starting OAuth.
   - **Fix:** Use two endpoints.
     - `/mcp/public`: no auth, read-only tools only.
     - `/mcp`: OAuth required, all tools.
     - The `/connect` page offers both. Update AGT-02, ARCH §5 and ARCH §7 accordingly.

8. **Major · REQ SEC-03, ARCH §11 (Redis "not now"): per-IP rate limits will throttle the campus.**
   - **Problem:**
     - During registration week, thousands of students on DTU's network share a few NAT/eduroam egress IPs, so "catalogue 20 requests/s per IP" is exactly the peak-load case the limit would break.
     - Behind a CDN, the client IP is also the CDN's unless the real client IP from the CDN's header is trusted explicitly.
     - Per-replica in-memory counters let a client use N× the limit.
   - **Fix:**
     - SEC-03: anonymous catalogue reads have no per-IP limit, because the CDN absorbs them.
     - Per-IP limits apply only to uncached expensive calls (free-text semantic search, recommend, MCP `tools/call`), at about 10 per second with a high burst, and DTU egress ranges are allow-listed.
     - Write and assistant limits are per user or token.
     - State which header holds the real client IP, and trust only the CDN's IP ranges.

9. **Major · ARCH §4.3 / §4.5 and the §9 capacity math: the embedding model swap breaks the memory, latency and scoring assumptions.**
   - **Problems:**
     - `multilingual-e5-small` has about 118M parameters (mostly its 250k-token vocabulary). At fp32 that is about 470 MB per ONNX session, against ARCH §9's "About 250 MB per worker … 8 workers ≈ 2 GB".
     - It has 12 layers against MiniLM's 6, so the "15 ms on CPU" figure is roughly doubled.
     - e5 requires `query: ` / `passage: ` prefixes.
     - Its cosine scores bunch up around 0.7–1.0, so the 0.99 near-duplicate threshold (REC-02), the 0.6 fusion weight and v1's "% match = score×100" all have to be re-tuned.
     - The default of 4 intra-op threads per ONNX session × 4 workers oversubscribes 2 vCPU.
   - **Fix:**
     - ARCH §4.5: "int8-quantised ONNX (about 120 MB), `query:`/`passage:` prefixes, `intra_op_num_threads=1`. The duplicate threshold, fusion weight and display score are recalibrated on the 50-query set; the UI shows rank or relative match, not raw cosine."
     - Redo the §9 memory line with the measured RSS.

10. **Major · ARCH §4.3 search, §9 "400–800 requests/s per vCPU": a semantic embedding on every query is the real bottleneck.**
    - **Problem:**
      - Each free-text query costs one ONNX inference (≥15 ms, more with e5), so one vCPU handles about 30–60 per second.
      - Search-as-you-type and MCP `search_courses` traffic is not cacheable at the CDN (MCP uses POST).
      - The per-worker LRU cache doesn't help across workers.
    - **Fix:**
      - Autocomplete and `search_courses` with `mode="auto"` use code and fuzzy matching only.
      - The semantic stage runs only on submit, or when the query has ≥3 words and fuzzy matching scores low.
      - Add "semantic queries/s" to the load test, and add a p95 target for it to PERF-02.

11. **Major · ARCH §4.2 / §4.6 / §9: ISR and snapshot swapping are incoherent across replicas.**
    - **Problems:**
      - Self-hosted Next.js keeps its ISR cache per instance. A revalidation hook called on one replica doesn't refresh the others, and the CDN keeps its own copy for `s-maxage`.
      - API workers pick up a new release by polling every 30 s. The pipeline's revalidation call can reach the API before the workers swap, so pages regenerate with the old release and stay that way.
      - Rollback (PIPE-07) doesn't purge anything.
      - Search cursors can span a swap.
    - **Fix:**
      - On publish or rollback: (1) wait until `/readyz` on every api replica reports the new release id; (2) call revalidation on every web replica (or use a shared filesystem ISR cache handler); (3) purge the CDN by tag `release`.
      - Course pages also get time-based `revalidate: 3600` as a backstop.
      - Cursors encode the release id and restart on a mismatch.

12. **Major · REQ PLAN-01/PLAN-05/PLAN-10 (P0, M2), ARCH §3.2: the anonymous local planner has no defined validation or undo path.**
    - **Problem:**
      - In M2 there are no accounts, so plans live in localStorage.
      - Validation is defined only as `domain.plan.validate(plan, snapshot)` "computed when the plan is read", which applies to server plans.
      - Undo is built on the server table `plan_event`.
      - Left as is, the team will re-implement clash and prerequisite logic in TypeScript, which contradicts OPS-02 ("UI, REST, MCP and assistant all call it").
    - **Fix:**
      - Add a stateless read tool, `validate_plan(plan: PlanJSON | plan_id) -> issues[]`, exposed over REST, MCP and `/v1/tools`. The local planner calls it on every change.
      - Undo for local plans is a client-side stack of plan snapshots (see finding 20).
      - Replace `check_prerequisites` and `check_schedule` with this one tool.

13. **Major · UX J1 step 2, REQ PLAN-01: anonymous plans will be silently lost.**
    - **Problem:**
      - Safari (ITP) deletes script-writable storage, including localStorage, for sites the user hasn't interacted with for 7 days.
      - Students plan twice a year, so most anonymous plans on iPhones will be gone by their next visit.
      - Clearing site data or switching devices also loses them.
    - **Fix:** Add to PLAN-01:
      - "Anonymous plans can be exported as a URL-encoded share link and as JSON."
      - "After the first plan is created, show 'Saved in this browser only, may be deleted after 7 days of inactivity on Safari: sign in or copy link'."
      - Track anonymous plan loss as a metric (plans with no return visit).

14. **Major · REQ SHR-01/SHR-08, ARCH §3.2 `slug unique`: unlisted plans need privacy rules that aren't specified.**
    - **Problems:**
      - Slug generation isn't specified. If it comes from the title (likely, given that "editable title" is next to Share), unlisted plans can be guessed.
      - Unlisted pages and their OG images have no `noindex`.
      - When a plan goes back to private, the cached `/p/{slug}` and OG image (CDN plus ISR) keep serving it.
    - **Fix:** Add a requirement SHR-09:
      - "Unlisted slugs are random, ≥ 96 bits, base62."
      - "Unlisted and private pages send `X-Robots-Tag: noindex`, are left out of the sitemap, and are served with `Cache-Control: private, no-store`. Only public plans may be CDN-cached."
      - "A visibility change purges the slug and the OG image."
      - Allow slug rotation ("reset link").

15. **Major · REQ PRIV-04 vs UX §7 vs SHR-06: the k-anonymity threshold is defined inconsistently, and one version is easy to game.**
    - **Problem:** PRIV-04 sets the threshold at "10 or more distinct users"; UX §7 says "10 or more distinct plans". One user can create 10 plans and re-identify a single student's plan. Including "private plans when the user has opted in" also needs a consent flag that is in no table.
    - **Fix:**
      - Standardise on "≥ 10 distinct `owner_id`s, counting at most one plan per owner".
      - Add `app_user.aggregate_opt_in bool default false` to ARCH §3.2.
      - Update UX §7 to match.

16. **Major · REQ CAT-01/CAT-05, missing FR: the v1 predecessor back-fill is not carried over.**
    - **Problem:**
      - v1 copies grade and evaluation history from the predecessor course(s) named in "Previous Course" into the successor wherever the successor has no data (csv_creator.py:99-117).
      - This changes the course-page stats of every renamed or renumbered course, and so DQ-02 parity too.
      - No FR keeps or drops it. The code also has a latent crash: `df_target.loc[course]` runs before the membership check (:106-107).
    - **Fix:** Add a requirement CAT-17:
      - "Stats for a course include terms inherited from its `previous_code` chain (parsed per format_info.py:284-302) where it has no own data for that term. Inherited terms are labelled 'as 0xxxx' on the course page."
      - Add a regression test for a course that has no row of its own.

17. **Major · REQ §3 priorities vs ARCH §13 milestones: the priority labels don't match what is delivered when.**
    - **Problems:**
      - AGT-01 is P0 and says "Every capability in §3.1–3.7 is available as an MCP tool", but plan write tools are AGT-03 (P1, M3).
      - In the other direction, P1 items are in scope for M2: PLAN-08 exports and PLAN-11 compare (ARCH:322 "compare, exports"), CRS-09 "Similar" in J1 step 3, and SRCH-11 (the ⌘K palette) in the UX §3 global header.
      - "Launch" is never tied to a milestone. M1 is "public beta" and P0 includes the planner (M2).
    - **Fix:**
      - Define "Launch = end of M2".
      - Reword AGT-01 to "…every read capability at P0; plan writes at P1 (AGT-03)".
      - Either move PLAN-08 and PLAN-11 out of M2 or make them P0.
      - Mark CRS-09 Similar as P0 (the recommender exists already), and keep "Often planned together" at P1.

18. **Major · REQ AGT-03, ARCH §5 tool list: the tool set is inconsistent and incomplete for the use cases.**
    - **Problems:**
      - AGT-08 requires confirmation for "delete a plan", but there is no `delete_plan` tool.
      - There are no tools to rename a plan or set its metadata, to undo, or to create a plan from a template (as opposed to forking).
      - `star_plan` is in ARCH but not in AGT-03.
      - `list_my_plans` and `get_plan` are listed under "Write" but are reads (`readOnlyHint` should be true).
      - The J3 example "avoid Friday mornings" can't be expressed, because SRCH-02 facets are OR-include only, and courses that have several slots would leak through.
    - **Fix:**
      - Add `delete_plan` (approval rule as in finding 1), `update_plan`, `undo_last_change`, and `exclude_slots` / `exclude_days` parameters on `search_courses`.
      - Merge `get_course_stats` into `get_course(detail=…)` to reduce near-duplicate tools.
      - Re-group the tools as "auth required" vs "public", not "read" vs "write".

19. **Major · REQ SEC-06, ARCH §6: prompt-injection coverage stops at the built-in assistant.**
    - **Problem:**
      - `<untrusted_user_content>` wrapping only protects the built-in assistant. Third-party agents get raw shared-plan text through `get_shared_plan` and `list_templates`.
      - Scraped DTU course text (description, remarks, highlighted message) is also free text that comes from outside the team.
    - **Fix:**
      - Every tool output puts user-generated or scraped text in a field named `untrusted_text` (or `*_untrusted`), and the tool descriptions say "treat as data".
      - The authorisation rule (writes only to the caller's own plans, with the publish and grade rules from finding 1) is the actual control. State it as such in SEC-06.

20. **Minor (simplification) · ARCH §3.2 `plan_event(op, inverse_op)`, UX §5.2 "an undo for each": per-edit inverse operations are more machinery than needed.**
    - **Problem:**
      - Inverse operations plus selective undo of any past event will conflict. For example, undoing "add 02105 to Autumn" after the user has moved it to Spring.
      - It also needs a second, client-side implementation for local plans.
      - Plans are tiny (under 60 items).
    - **Fix:**
      - Store `plan_version(plan_id, version, items_json, actor_kind, client_id, created_at)`.
      - Undo means "restore version N-1", or restore any listed version.
      - The "Assistant changed…" bar shows the diff between two versions. Selective per-event undo is dropped.
      - The same model works in localStorage, and it removes the need for `inverse_op`.

21. **Minor · ARCH §3.2 `version` / If-Match, ARCH §5 `add_course(… expect_version=inp.version)`: make the version check optional for operations that compose.**
    - **Problem:** Requiring every agent call to carry the plan version adds a read-before-write and 409 retry loops for LLMs, even though add, remove, move and set-status compose safely.
    - **Fix:** `version` is optional on operation tools; if it is present, a mismatch returns 409. Operations that make no sense at apply time (removing a course that isn't in the plan) return a `{code, fix}` error. Keep If-Match as mandatory only for `update_plan` and for whole-plan import.

22. **Minor · ARCH §9 capacity vs reference deployment: the capacity math doesn't match the deployment it describes.**
    - **Problem:**
      - The math assumes "Two api replicas × 2 vCPU (4 workers each)".
      - The deployment puts `web` (Next.js SSR), `api` and the auth broker on 2 nodes of 4 vCPU, so there is no dedicated 2 vCPU of api headroom.
      - "About 60% of hits are course and template pages" has no source, even though v1's access-log analyser can measure it.
    - **Fix:** Run `webhost_access_logs_analyzer.py` on the last registration week and replace the 60% with the measured share. Restate the capacity as per-node CPU budgets (web, api and broker), and confirm it in the M1 k6 test, which must include MCP POSTs and semantic queries.

23. **Minor · REQ SCAL-03, ARCH §4.1: "DB down → catalogue keeps working" is false whenever a worker restarts.**
    - **Problem:** Workers build the Snapshot from Postgres at startup. A crash restart, autoscale or rolling deploy during a database outage gives a worker that never becomes ready.
    - **Fix:** On publish, also write the serialised Snapshot (msgpack or npz plus JSON) to the shared volume. Workers load the latest file if Postgres is unreachable.

24. **Minor · ARCH §7 web sign-in: the session cookie will overflow.**
    - **Problem:** The plan says "httpOnly … session cookie containing the broker's tokens". Keycloak and Zitadel access, refresh and ID JWTs together often exceed the 4 KB cookie limit.
    - **Fix:** Store server-side sessions in Postgres (`web_session(id, user_id, refresh_token_enc, expires_at)`), and make the cookie an opaque id. Alternatively, use encrypted cookie chunking.

25. **Minor · ARCH §1 / SCAL-02: the auth broker is a single point of failure during registration weeks.**
    - **Problem:** SCAL-02 promises 99.9% during registration weeks. A single broker instance means new web logins and MCP token refreshes fail while it is down.
    - **Fix:** Run 2 broker replicas behind the load balancer (both Keycloak and Zitadel support this with a shared database), or state explicitly that availability excludes sign-in.

26. **Minor · REQ SHR-07, UX §7 "Reporting a plan hides it after 3 reports": auto-hide can be abused.**
    - **Problem:** Three accounts can take down any plan, including curated ones. There is also no rule on display names, so a user can call themselves "DTU Official" or a teacher's name.
    - **Fix:**
      - Auto-hide needs 3 reports from accounts older than 7 days and never applies to curated plans.
      - Rate-limit reports per user.
      - Reserve display-name words ("DTU", "official", "admin", staff names from `person`).
      - Use a profanity list for both Danish and English.

27. **Minor · REQ UX-01, UX §5.2: drag-and-drop has no pointer alternative on desktop.**
    - **Problem:** The "Move to…" menu exists only on mobile. WCAG 2.2 SC 2.5.7 (Dragging Movements, which is AA) requires a non-drag alternative on every viewport, and the menu also serves keyboard users.
    - **Fix:** UX §5.2: "Each course card has a 'Move to…' menu on all viewports; dragging is an enhancement."

28. **Minor · UX §4 / §5: empty, error and offline states are unspecified.**
    - **Problem:** No behaviour is defined for:
      - a course with no grades or no evaluations (new courses: what do the J1 tiles show?)
      - an "Unspecified" schedule
      - the API being down (do the planner and local plan still work?)
      - assistant tool failures
      - an empty gallery (UX:335 mentions only seeding)
      - first run of `/plan`
      - offline use. REQ §6 relies on "A responsive PWA" but no FR requires a manifest or offline support.
    - **Fix:** Add UX §5.5, "States", with one line each:
      - Tiles show "No data yet (new course)".
      - Schedule shows "Schedule not published".
      - The planner works offline from localStorage and shows "Checks paused, server unreachable".
      - The assistant panel shows the failed tool step.
      - The gallery shows "Curated only".
      - Add a PWA FR, or drop the PWA claim.

29. **Minor · REQ §5 bug list: missing v1 bug — no grade data shows as "0% fail".**
    - **Problem:** Courses with no grade data display "(0% fail)" on cards and sort as 0.
    - **Evidence:** `website_consts.py` (percent_failed branch): `if dct[key] == GradeConsts.grade_none: dct[key] = "0"`.
    - **Fix:** Add §5 item 19: "No grade data is shown as 'No data', never as 0% fail." This matters because SRCH-09 now counts real zeros.

30. **Minor · REQ SRCH-06 / UX tiles: "sign-ups" is the wrong name.**
    - **Problem:** STUDENTS_PER_SEMESTER is derived from grade counts (format_grades.py:163-198), not from registrations. The v1 UI calls it "Signups" / "N ppl", and REQ repeats "sign-ups".
    - **Fix:** Rename it to "students per term (graded)" in CAT-05, SRCH-06 and on the methodology page.

31. **Minor · REQ PIPE-05/PIPE-06/DQ: v1's data-quality rules aren't carried over.**
    - **Problem:** These v1 rules are missing from the requirements:
      - An evaluation term is dropped if its answer counts don't sum to "N besvarelser" (±2 tolerated, course 01005 exempt) or if the answer labels don't match (scrape_evaluations.py:217-240).
      - Grade pages with only 2 tables mean the grades are hidden because there are ≤3 students (scrape_grades.py:73-76).
      - Orbit photo matching uses name variants, an email check and manual overrides (scrape_orbit.py:22-53,127-258).
    - **Fix:**
      - Add these as named parser rules under PIPE-05.
      - Treat "hidden (≤3)" as distinct from "no data" in CAT-11.
      - Put the orbit overrides into `vocabulary_map(kind='person_slug')` under CAT-08.

32. **Minor · ARCH §13 M0 exit criterion: "all 18 regression tests pass" can't be met at M0.**
    - **Problem:** Bugs 7, 8, 16 and 18 are about course-page rendering and HTTP status codes, and bugs 1–3 and 6 are search and recommender behaviour. None of those exist at M0.
    - **Fix:** M0 covers the pipeline and metric bugs (9–15). M1 covers the rest. Update the exit criteria accordingly.

33. **Minor (over-engineering) · ARCH §5 entry points: there are two HTTP surfaces for the same tool calls.**
    - **Problem:** `POST /v1/tools/{name}` and `POST /api/v1/{name}` do the same thing.
    - **Fix:** Keep `/api/v1/{name}` only. `GET /v1/tools` returns the OpenAI tool JSON with a note that each call goes to `/api/v1/{name}`. That is one auth path, one rate limiter and one OpenAPI spec.

34. **Minor (over-engineering) · ARCH §10: the monitoring stack is heavy for the team.**
    - **Problem:** A Prometheus server plus alert rules is more to run than a student-council team needs.
    - **Fix:** Start with Sentry (errors and performance, which includes per-transaction RED), an uptime checker, structured logs, and one Postgres-backed admin page for pipeline, LLM cost and quotas. Add Prometheus when the load test or an incident shows a need, and list that in §11.

35. **Minor · REQ "P0 = parity" vs E-origin items rated P1/P2: launch removes v1 features.**
    - **Problem:** SRCH-09 summary stats (P1), SRCH-10 random course (P2), CAT-14 data export (P1) and ADM-03 bot/human report (P1) all exist in v1. Also, `/course/xxxxx` (random course) isn't covered by the ARCH §12 URL migration.
    - **Fix:** Either make them P0 (all are cheap) or say in §1 that they are intentionally dropped at launch. Add `/course/xxxxx` → 302 `/random` to ARCH §12.

36. **Minor · REQ AGT-07 vs ARCH §1/§7: the two docs disagree on who is the authorisation server.**
    - **Evidence:**
      - AGT-07: "the app is the authorisation server"
      - ARCH:44: the broker is the "OAuth 2.1 authorisation server"
    - **Fix:** AGT-07: "The auth broker is the authorisation server; the api is the resource server and serves `/.well-known/oauth-protected-resource`."

37. **Nit · REQ §5 bug 4: the bug is misstated.**
    - **Problem:** `/search_courses` is already case-insensitive and finds letter codes (recommender.py:105, `case=False`). The bugs are:
      - the navbar submit path doesn't upper-case its input and requires exactly 5 characters (search.py:8-20)
      - `/course/<id>` is case-sensitive (course_database.py:38)
      - extraction uses `\d{5}` (recommender.py:152)
    - **Fix:** Reword bug 4 as those three bugs, so the regression tests target the right code paths.

38. **Nit · REQ REC-02: the origin label is wrong.**
    - **Problem:** REC-02 says the existing behaviour (E) excludes "point-lock courses". v1 doesn't; `COURSE_POINT_LOCK` isn't even in `course_df.csv` and no code in `website/` references it.
    - **Fix:** Label point-lock exclusion as N.

39. **Nit · REQ §1, CRS-07/CRS-09: ID references and a claim need correcting.**
    - **Problems:**
      - The text references "NFR-PRIV-04" (CRS-09), "NFR-COST" (SEC-03, AGT-11) and "(NFR-COST)", but the defined IDs are PRIV-04 and COST-01..03.
      - §1 says "Every request re-parses JSON files on disk", but `/course/<id>` uses the in-memory dataframe (course_database.py:19-27).
    - **Fix:** Use the defined IDs, and say "browse, home and recommend requests".

---

I did not check:
- COST-01's €150/month
- PERF-06's 2-hour full-refresh estimate
- the UX copy
- the Python MCP SDK's support for 2026-07-28 (its exact version needs checking)
- whether ChatGPT and Claude support CIMD today (finding 6 makes this an M0 spike)

Zitadel's DCR support could not be confirmed either way.
