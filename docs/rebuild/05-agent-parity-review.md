# Review: agent-parity proposal (agent-parity-proposal.md)

Scope checked: the proposal against agent-parity-evaluation.md and docs/rebuild 01 (§3.3–3.9, §4), 02 (§3, §5, §6, §7, §13), 03 (§2, §4, §5). Ordered most severe first.

1. **Blocker. Copying a plan can publish it or carry someone else's grades, with no approval.**
   - Evidence: proposal `create_plan(... source?: {template_id}|{plan_id}|{shared_slug}|{plan_json}|{items[]})`: "as a duplicate of an own plan, as a fork of a shared plan ... or from a full plan JSON". It never says what happens to `visibility`, `share_grades`, `slug`, `grade` or `status`. 02 §3.2 stores all of these on `plan` and `plan_item`. AGT-08 makes publish and share_grades owner-approval actions.
   - Problem:
     - A naive "duplicate" of a public plan that has `share_grades=true` creates a second public plan with grades, without approval.
     - A fork by `shared_slug` (when the source shares grades), or a pasted `plan_json`, puts another person's grades and "passed/failed" statuses into the forker's account. They then also show up in that user's ACC-04 export.
   - Fix: state the rule in create_plan. Every new plan starts `private`, with `share_grades=false`, a fresh slug and no `hidden_at`.
     - For `shared_slug` and `template_id`, drop `grade` and reset `status` to `planned` (keep `suggested`).
     - The `plan_json` schema has no visibility, share_grades, slug or forked_from fields (`additionalProperties: false`).
     - Refuse sources that are hidden or private and not owned by the caller.
     - Add this case to the M3 prompt-injection test.

2. **Major. `share_grades` inside an atomic `edit_plan` batch contradicts "one call = one change".**
   - Evidence: `edit_plan` ops include `set_meta {... share_grades?}` and "`share_grades=true` goes through owner approval". Principle 3 says "A batch of operations is atomic". `set_plan_visibility(plan_id, visibility, reset_link=false)` mixes a direct action (reset) with a gated one (unlisted or public).
   - Problem: a batch with 30 adds plus `share_grades=true` either:
     - holds back all 30 adds pending approval, or
     - applies only part of the batch, which breaks atomicity.

     `set_plan_visibility(visibility="unlisted", reset_link=true)` has the same ambiguity.
   - Fix:
     - Remove `share_grades` from `edit_plan`, so `edit_plan` never touches gated fields.
     - Replace `set_plan_visibility` with `set_sharing(plan_id, visibility?, share_grades?, reset_link?)`. Rule: if any gated field is in the call, the whole call becomes one `pending_approval`. Otherwise it applies directly.
     - Update the AGT-08 text: going private, turning `share_grades` off and resetting the link apply directly.

3. **Major. Plan-bearing public tools as REST GET put plans and grades in URLs and logs, and plan access on `/mcp/public` is not specified.**
   - Evidence:
     - Proposal section header: "Public: no account ... REST GET where read-only". 02 §5: "Public read tools become `GET /api/v1/{name}?…` (cacheable)".
     - These tools take `plan_json|plan_id`: `validate_plan`, `export_plan`, `search_courses` (via `filter.fits`) and `recommend_courses`. The local planner calls `validate_plan` "on every change" (02 §3.2).
     - PLAN-01: "Exported links and JSON never include grades". The proposal's own reason for `make_plan_link` is to keep plans out of "our server, CDN or logs".
   - Problem:
     - A local plan, grades included, ends up in query strings, which reach CDN and access logs. The URLs can get long (a 60-item plan plus `programme_rules`), and the responses can't be cached anyway.
     - `plan_id` on `/mcp/public` has no defined authorisation. `plan://{id}` ids are handed out by "Open in my AI" (03 J5), so `export_plan(plan_id)` could return a plan that has since gone private, or return grades that `share_grades=false` should hide.
   - Fix:
     - Any tool that takes a plan is `POST` and never cacheable.
     - Public tools drop `grade` on input; no check needs it.
     - `plan_id` is accepted only on `/mcp` and only for the owner. `/mcp/public` takes `plan_json`, `plan_link` (item 4) or a shared `slug`, and output for a slug strips grades unless `share_grades` is on.
     - Tool arguments are never logged (02 §10 already logs only the tool name; state this as a rule).

4. **Major. The anonymous round trip is one-way, and an import link can overwrite a student's only local plan.**
   - Evidence:
     - `make_plan_link` encodes, but nothing decodes.
     - ACC-01: "one local plan work[s] without logging in".
     - `make_plan_link`: "Opening it shows the plan in the planner as a local plan with 'Save in this browser'".
     - Whole-year template shifting exists only server-side in the sign-in tool `create_plan` (SHR-03).
   - Problem:
     - At launch, an agent can't overhaul an existing local plan: a student who pastes their "Copy link" URL gets an agent that can't read it.
     - Opening an agent-made link while a local plan exists has no defined behaviour. Replacing the plan silently destroys the student's only copy; PLAN-01 says "never merged silently".
     - An anonymous agent can't instantiate a starter template for a given start term, which is the J1 starting point.
   - Fix:
     - Accept `plan_link` wherever `plan_json` is accepted; the server decodes it.
     - The import page shows a preview with "Replace my plan" (the current plan is kept as a local version, so Undo restores it) and "Cancel". From M3 it also offers "Save to my account".
     - `get_shared_plan(slug|template_id, start_term?)` returns a shifted, grade-free `plan_json`.
     - The capability matrix states the limit honestly: anonymous agents hand over new versions by link; they can't push edits into the browser.

5. **Major. Ops address plan items by course code alone, which breaks retakes.**
   - Evidence: ops `remove {code}`, `move {code, start_term}` and `set_status {code, ...}`. 02 §3.2 has `plan_item(plan_id, course_code, start_term, ...)` with no item id. PLAN-03 statuses include `failed`.
   - Problem: a failed course retaken later is two items with the same code. Every code-addressed op is then ambiguous, and `replace_items` can't round-trip such a plan.
   - Fix: give `plan_item` an `item_id`, or address items by `(code, term)` with `term` optional when the code is unique. Return `AMBIGUOUS_ITEM` with a `fix` that lists the terms.

6. **Major. All-or-nothing batches have no definition of "invalid", and the existing error example would make one unavailable course reject a whole full-plan batch.**
   - Evidence:
     - Proposal: "If any op is invalid, nothing is applied".
     - 02 §5 error example: "`TERM_NOT_OFFERED: 02105 is not offered in 2028-spring`".
     - 02 §3.2: removing a course that isn't in the plan returns an error.
     - PLAN-12: unknown schedules must "never block". PLAN-05 lists "course not offered in that term" as a live check.
   - Problem: a 30–100-op batch built from scraped study regulations, which often name discontinued or future-year codes, fails completely on one not-offered course or one already-removed item. That is worse for agents than the per-call model it replaces.
   - Fix: keep all-or-nothing (it is right for one-step undo), but define the error set narrowly:
     - unknown code
     - malformed term
     - missing or ambiguous target for move or update
     - exact duplicate (same code and term)
     - over the size limits

     Every PLAN-05 check is returned as an issue and never blocks. `remove` of an absent item is a no-op with a note. Report all failing ops in one response.

7. **Major. `expect_version` is optional even for whole-plan replacement.**
   - Evidence: proposal `edit_plan(..., expect_version?, ...)` with `replace_items`. 02 §3.2: "`update_plan`, whole-plan import and version restore require `If-Match`."
   - Problem: an agent's `replace_items` while the student is dragging in the planner silently discards the student's edit. Undo can recover it, but nothing tells the student it happened.
   - Fix: require `expect_version` when the ops contain `replace_items` or `set_meta`, and on `restore_plan_version`. It stays optional for add, remove and move, as today.

8. **Major. The "about 4 calls per full plan" claim is wrong, because `fits` is single-term.**
   - Evidence: proposal "with batch tools a full plan needs about 4 (get_me, search/recommend, edit_plan or create_plan, validate)" and `filter.fits = {plan_json|plan_id, term}`. 02 §6: "at most 8 tool calls per turn".
   - Problem:
     - Filling 6 terms takes one `recommend_courses` call per term, plus get_me, create or edit, and validate: about 9 calls, over the cap.
     - `validate` is redundant, because `edit_plan` already returns issues.
     - The built-in assistant has no web fetch, so the regulations-driven workflow can't run inside it at all.
   - Fix: make `fits` multi-term: `fits_terms[]`, where an omitted list means all plan terms. Each result returns `fits_terms[]` (the terms it fits) rather than being filtered to one term. This also gives the §5.1 search "clash badge" for free. Restate the realistic call count, and say the regulations workflow is for external agents only.

9. **Major. `sort="popular_in_programme"` leaks sub-k counts (PRIV-04).**
   - Evidence: PRIV-04 says aggregates appear "only for groups of 10 or more distinct plan owners". The proposal's `search_courses` has `sort="popular_in_programme"` (k-anonymous).
   - Problem:
     - A full ordering also ranks courses planned by 1–9 owners. Rank order inside a small programme, combined with `within_codes` and narrow filters, reveals counts below the threshold, and one user can probe it by toggling their own opted-in plan.
     - At launch there is no data at all: accounts come at M3 and aggregates at M4.
   - Fix: only courses with at least 10 distinct owners in that programme get a popularity rank. All other courses come after them in the default sort order, with no rank shown. Drop this sort from the launch tool list and ship it at M4.

10. **Major. Caller-supplied `programme_rules` adds a P2 feature at launch and makes our checker look authoritative on rules nobody vetted.**
    - Evidence: `validate_plan(plan_json|plan_id, programme_rules?)` ("mandatory codes, minimum ECTS per category, total ECTS"). CAT-16 is P2. The workflow has the agent scrape sdb.dtu.dk itself.
    - Problem:
      - The rules come from untrusted web content that the agent read. Our server then says "requirement met" on our domain, and a `source` label doesn't stop a student from trusting a degree-requirement verdict.
      - It also fixes a rules schema now that CAT-16 will later have to stay compatible with, which goes against the "no over-engineering" stance.
    - Fix: drop `programme_rules`. Have `validate_plan` return ECTS per term and in total (PLAN-06 already does), plus an optional `groups: {label: codes[]}` that returns ECTS sums per group with no pass/fail verdict. Labels come back as `*_untrusted`.

11. **Major. Tool schemas rely on nested unions that weaker models and strict function-calling handle badly.**
    - Evidence:
      - `plan_json|plan_id` appears in 5 tools.
      - `source` is a union of 5 shapes.
      - `ops[]` is a union of 6 differently shaped objects.
      - `CourseFilter.fits` nests a whole plan inside a filter.
      - AGT-05 exports the same schemas as OpenAI tools, and AGT-06 targets "vLLM or Ollama".
    - Problem: `oneOf`/`anyOf` and deep nesting are where tool-call accuracy drops. OpenAI strict mode and some providers restrict or reject them, small local models get them wrong, and schemas for 20 tools take a lot of tokens.
    - Fix: flatten the schemas.
      - Use sibling optional fields with a server-side "exactly one of" check: `plan_id?`, `plan_json?`, `plan_link?`; `from_template_id?`, `from_plan_id?`, `from_shared_slug?`, `items?`.
      - Ops become flat objects with an `op` enum.
      - Merge `move`, `set_status` and the missing suggested toggle (item 13) into one `update {code, term?, to_term?, status?, grade?, suggested?}`.
      - Move `fits_*` out of `CourseFilter` into top-level parameters.
      - Add a CI check that the exported schemas pass OpenAI strict mode and one local model.

12. **Major. Batch reads make each public call up to 100 times more expensive, and some of them bypass the per-IP limit.**
    - Evidence:
      - `get_courses(codes[1..100], detail=full)`, `within_codes[]` (≤100), and `programme_rules` with no size limit.
      - SEC-03: "Cached anonymous catalogue reads have no per-IP limit". Arbitrary code lists never hit the cache, yet a REST GET is not an "expensive call", so it has no limit.
      - DTU egress ranges are allow-listed.
    - Problem:
      - 100 full courses is a response of about a megabyte, which also floods the agent's context, against principle 5.
      - "A write call counts once regardless of ops" turns 5 writes/s into 500 ops/s, each write a full `plan_version` snapshot.
    - Fix:
      - Cost-weighted limits: 1 unit plus 1 per 10 codes, plus 1 per embedding.
      - `detail=full` takes at most 10 codes, `summary` up to 100.
      - Cap plan size (100 items, the same limit for `replace_items`), plans per user and request body size.
      - The web client sorts `codes[]` before calling, so the CDN cache key is stable.

13. **Minor. Coverage gaps: UI capabilities with no tool or output field.**
    - **ECTS target:** the evaluation's audit row "ECTS target: missing" is not addressed. PLAN-05 says "(target 30; warn below 25 or above 35, configurable)", but no plan field holds it. Either state that it is deployment config, or add `ects_target` to `set_meta`/`plan_json` (part-time students need it).
    - **`suggested` after creation:** it can only be set on `add`, so the J2 "swap" flow can't clear it. Fixed by `update` (item 11).
    - **SRCH-05 and SRCH-09:** facet counts, "remove X to get N" suggestions and result-set summary stats have no output in `search_courses`. Add `include=[facets, summary]` and a `relax_suggestions` field on zero results.
    - **CRS-07 naming:** "successors" mixes up the courses this one unlocks with the replacement for a discontinued code (03 §5.3). Use `unlocks_codes[]` and `successor_codes[]`, and expose `prereq_hints_untrusted[]`.
    - **CRS-01:** no list of the academic years available for a code.
    - **SHR-06 and SHR-04:** no "N students planned this" count; `list_templates` lacks the "specialisation" filter that SHR-04 names.
    - **Pending approvals:** `get_plan` doesn't show them, so an agent can't tell whether the owner approved.
    - **Validation fixes:** the panel's one-click fixes ("Move 02105 to Autumn", 03 §5.2) have no machine-readable equivalent. Add `suggested_ops` to each issue.
    - **SHR-03:** realignment warnings when a template is created for a different start term are not specified for `create_plan`.
    - **Filter vocabulary:** agents have no way to discover valid `CourseFilter` values (departments, slots, teachers). Fold `list_study_lines` into `get_dataset_info().vocabularies`, which also saves a tool.

14. **Minor. `make_plan_link` makes a privacy claim that isn't true, and the import page is a spoofing surface.**
    - Evidence: "it is never sent to our server, CDN or logs". Yet the tool call itself sends the plan to our server, and the import page will call `validate_plan`.
    - Problem: anyone can craft `…/plan/import#p=` with a title such as "DTU: re-verify your login at …", and it renders on our domain. A tiny fragment can also inflate to megabytes (a decompression bomb), and the format has no version.
    - Fix:
      - Reword the claim to "not in URLs, logs or CDN keys".
      - The import page renders imported text as plain text: no linkification, title ≤80 characters. Simplest: drop the description from links entirely.
      - Show a banner: "This plan came from a link".
      - Drop unknown codes, cap the inflated size at 64 KB, and prefix the payload `v1.`.

15. **Minor. Untrusted text gets laundered into the user's own plan.**
    - Evidence: SEC-06 marks as untrusted only "user-generated text (shared plan titles and descriptions)". `create_plan(source=shared_slug|plan_json)` copies titles and descriptions into the user's own plan.
    - Problem: once copied, an injected description comes back from `get_plan` and `get_me` as the user's own, apparently trusted, text, including to the built-in assistant.
    - Fix: put every plan's free text, own plans included, in `*_untrusted`, and drop `description` on fork and import.

16. **Minor. The AGT-13 claim that the web client uses only registry endpoints "by construction" is false, and two endpoints lose their home.**
    - Evidence: "the web client may call only registry-generated endpoints (true by construction)". But 02 §6 polls `GET /api/v1/plan_version`, 02 §3.2 posts to `/api/v1/import_plan`, and there are also the approve, assistant SSE, token, report and admin endpoints. The proposal's tool list has neither `plan_version` nor `import_plan`.
    - Fix: the CI test checks the capability matrix plus an explicit allow-list of non-tool endpoints. `plan_version` stays a non-tool endpoint (it serves rendering, not a capability). `import_plan` becomes `create_plan(from plan_json)`. Add assistant conversation management (PRIV-05) to the web-only list.

17. **Minor. The milestone list pulls P1 and P2 work into launch.**
    - Evidence: "all public tools at launch (M1/M2): ... export_plan ...". But PLAN-08 is P1 and its UI arrives in M3. Also:
      - PDF needs a web render, and for `plan_json` that means putting the plan in a URL or storing it.
      - iCal needs term dates.
      - Popularity data comes at M4.
      - MCP prompts are P1 (AGT-04), yet `build-full-plan` carries the launch workflow.
    - Fix:
      - `export_plan` moves to M3 and covers iCal only. Markdown and CSV are presentation: the agent already has the data, and the UI can render them in the browser.
      - Popularity sort and `often_planned_with` ship at M4.
      - Put the workflow guidance in tool descriptions and `next_steps`, which work from M1.
      - Update the §13 "Later" row, which still lists SRCH-04.

18. **Minor. `fits` merges two different requirements.**
    - Evidence: SRCH-04 is "no schedule clash". PLAN-07 is "free slots and met prerequisites". The proposal has one `fits` that means "free slots and met prerequisites".
    - Problem: the search filter would wrongly hide courses that fit the schedule but whose prerequisites are planned for the same term. It is also undefined which items count as met (earlier terms only? status other than failed?).
    - Fix: `fits` means no clash and offered in the term. Add a separate `prereqs_met: bool`, counted from items in earlier terms whose status is not `failed`.

19. **Minor. `edit_plan` output is still too large for a full plan.**
    - Evidence: "Writes return `{version, diff, issues}`".
    - Problem: a 30-course plan routinely has dozens of recommended-prerequisite and ECTS warnings, and each write resends all of them.
    - Fix: return `issues_delta` (new and resolved), counts by kind and the blocking count. The full list only with `return_plan=true` or from `validate_plan`.

20. **Nit. Annotations and leftover old tool names in the docs.**
    - **Annotations:** `edit_plan` can wipe a plan (`replace_items`, `remove`), so it needs `destructiveHint=true` and `idempotentHint=false`. The old `add_course` was idempotent; a batch is not.
    - **Docs still naming removed tools:**
      - 02 §3.2 version rules: add/remove/move/set_status/update_plan
      - the 02 §5 sketch: `add_course`
      - AGT-02, AGT-03 and the AGT-04 prompt `compare-courses`, which names the removed `compare_courses`
      - 03 J3 step 2: "`add_course` ×3"
    - **`preset` vs `sort`:** fold `preset` into `sort`; SRCH-06 already treats presets as sorts.
    - **`filter.teacher`:** say whether it takes a name or a person id.
