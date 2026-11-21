# DTU Course Analyzer: what users see and do today (front-end inventory)

Repo: /Users/puddu/Documents/GitHub/dtu-course-browser. Paths below are relative to `website/`. Read-only extraction.
Stack: Flask + Jinja, server-rendered pages. Bootstrap **5.0.0-alpha1** with BS4-style `data-toggle` attributes, plus jQuery 3.2.1, Popper 2.9.2 and Font Awesome 4.0.3, all from CDNs. Most JS is inline in the templates. `static/index.js` is 0 bytes. `static/navbar.css` is 77 bytes and no template links to it.

---

## 0. Which template serves which route

| Route | Template | Live? |
|---|---|---|
| `/` GET (POST = navbar search) | `home.html` (views.py:14-29) | yes |
| `/browse?…` GET/POST | `browse.html` (browse.py:12-38). With no args it redirects to `?dk=language&eng=language` | yes |
| `/course/<id>` GET/POST | `course.html`, or `404_invalid_course.html` if the course is unknown (course_database.py:29-72). `/course/xxxxx` opens a random course | yes |
| `/random/` | 302 redirect to a random `/course/<id>` (course_database.py:74-78). The navbar links to `/random` (no trailing slash), so it relies on Flask's slash redirect | yes |
| `/faq` | `faq.html` (misc.py:11) | yes |
| `/recommender` | `recommender.html` (recommender.py:68) | yes, linked in navbar |
| `/recommender_v1`, `/recommender_v2` | `recommender_v1.html` (standalone, no base), `recommender_v2.html` | routed, not linked |
| `/wip` | `wip.html` (layout sandbox from 2022, fake cards and an unsplash image) | routed, linked from FAQ ("Show me something cool") |
| `/test` | `test.html` (standalone "Hello World" with a demo nested dropdown) | routed, not linked |
| JSON endpoints | `/search_courses`, `/extract_courses`, `/get_study_line_courses`, `/recommend`, `/course_info/<id>`, `/cv_helper` (recommender.py:91-380) | used by JS |
| none | `index.html` (Bootstrap 5.1 cheatsheet copy), `recommender_v2x.html`, `recommender_v3.html`, `login.html`, `login_elif_test.html` (invalid Jinja `{% elif %}` at :8), `sign_up.html`, `404_empty_search.html`, `404_not_found.html` | dead |

- There is no 404 error handler: `404_not_found.html` is not wired, so unknown URLs get Flask's default 404 page.
- There are no accounts. The login and sign-up templates exist but have no routes.

---

## 1. Pages: purpose, layout, components

### 1.1 Shared layout and navbar (`base.html`)
- `<html lang="en">`. The `<title>` is fixed to "DTU Course Analyzer" on every page (base.html:9). Child templates define `{% block title %}`, but base.html never uses it.
- The meta description is "Browse and analyze courses at DTU…" (:7). Google Analytics gtag `G-53NMP6CHX6` runs on every page (:12-19), with no cookie or consent banner.
- Navbar (:123-162):
  - Uses `navbar-expand-md` with the brand red `rgb(153,0,0)` / `#990000` (:56).
  - The brand logo is `rocket.png` with `alt="Bootstrap"` (:126-128).
  - Links: Home (always `active`/`aria-current` on every page, :136), FAQ, Recommender, "Browse all" (`/browse?dk=language&eng=language`), a search box (fixed 200px, :148), and Random.
  - Below `md` the hamburger toggler collapses the menu (:130). The toggler has no aria-label.
- Navbar search autocomplete (:150-152, JS :188-246):
  - Fires on `oninput` and `onfocus` with a 25ms debounce, calling `GET /search_courses?query=` (substring match on code, English name and Danish name; at most 20 results, recommender.py:91-141).
  - The dropdown shows up to 10 rows (code + name). The code comment says "Limit to 5" (:220).
  - Clicking a row puts the code in the input and submits the form (POST).
  - Pressing Enter submits the typed text instead. `submit_search_field` (search.py:7-20) only accepts exactly 5 characters. Anything else flashes an error and redirects to `/course/invalid_search`, which renders "Course not found".
  - The dropdown is mouse-only: no arrow-key navigation, no ARIA combobox/listbox roles. Results go in via `innerHTML` without escaping (:223-226).
  - Clicking outside closes the dropdown (:241-245).
- Flash messages (:166-182): a dismissible alert, warning style for errors and success style otherwise. Home renders flashes a second time (home.html:60-68).
- Content is wrapped in `<div class="container">` (:185). There is no footer anywhere.

### 1.2 Home `/` (`home.html`)
Sections, top to bottom:
1. **Hero** (:29-37): `<br>` ×4, an h1 "DTU Course Analyzer {{current_year}}" (renders "2026-2027"), and the lead line "A software project by Jonatan Rasmussen s183649". There is an empty red `<p>` at :35. A commented-out "NEW FEATURE" Recommender banner sits at :39-56.
2. **Quick navigation card** (:70-107): fixed `width: 26rem`.
   - Three stacked buttons: "Recommender tool" (purple `rgb(121,35,142)`), "Browse all courses" (red), "View random course" (orange `rgb(252,118,52)`).
   - On the right is the author photo `me.png` (968 KB, `loading="lazy"`, alt "Quick navigation").
   - Footer: "…Or scroll down to use the filter. You can also read the FAQ".
3. **Course search filter panel** (:111-191): fixed `width: 42rem`, `<form method="get" action="/browse">`. Each row is a label `<th>` plus checkboxes (full list in §2.1). Submit button: "Apply search filter" (:1185).
4. **"Course sample"** (:1195-1305):
   - h1 "Course sample", then "Pro-tip: Click on a card to see detailed course statistics!".
   - Grid `row-cols-1 row-cols-xxl-3`, so three columns only at ≥1400px and one column below that.
   - Shows 21 hard-coded courses (context_dicts.py:110-124: 02105, 02003, 02456, 02476, 34126, 02180, 42620, 10605, 63852, 02451, 38106, 38110, 38103, 42893, 02810, 62999, 27020, 01001, 10420, 25102, 02402) as course cards (§3.1).
5. **CTA buttons** (:1307-1324): "Click here to browse all DTU's 1700+ courses!" and "You can also check out the FAQ page!". Both open in a new tab (`target="_blank"`).
6. **Dead modal** `#modal_search_filter` (:1326-1388): leftover Bootstrap demo content ("Popover in a modal", "Save changes"). Nothing triggers it.
7. **End of page** (:1390-1394): "Return to top of page" and "Last updated: {{last_update.last_updated}}" (Config: "30/09/2026").
8. **Schedule legend modal** `#schedule_legend_modal` (:1396-1492), opened by "[?]" next to "Schedule:". It maps each schedule icon to day and morning/afternoon plus the DTU slot code: E1A…E5A are mornings 08-12, E1B…E5B are afternoons 13-17, Tue 18-22 is the evening slot, and "Off-schedule / Flexible / Unknown". It only shows E (autumn) codes; F (spring) codes are not explained.

### 1.3 Browse / results `/browse?…` (`browse.html`)
- The same hero h1 is repeated. The lead line here adds "DTU-student" (:8-10).
- **Results and sort card** (:18-86), `max-width: 24rem`:
  - Header: "N course(s) matched your search criteria".
  - Sort buttons ID, Name, Rating, Workload, Grade. These are plain `<a>` links to `{{args.args}}&sort_by=…`; every one carries `aria-current="page"`, so none is shown as the active sort.
  - "Reverse Sort Order" switch: `toggleSortReverse()` adds or removes `sort_reverse=true` and reloads the page (:44-48, :371-382).
  - "All sorting options" dropdown: ID, Name, Rating, Workload, Grade, ECTS, Signups, Schedule, Failrate, Evaluation votes, Rating: Learning, Rating: Motivation, Rating: Feedback (:55-77).
  - "Adjust filters" goes to `/` (:79). The filter form there is not pre-filled with the current filters.
  - When there are 0 results, all of the sort UI and the "Adjust filters" button are hidden (`{% if length != 0 %}` :24). The user sees only "0 courses matched" and no next step.
- **Summary statistics card** appears both before and after the list (:96-157, :280-340):
  - Average students per semester; average grade with fail rate.
  - Average rating out of 5.00, with workload "(not part of avg. rating)".
  - "I have learned a lot", "Motivating activities", "Access to feedback", each marked "(⅓ part of avg. rating)".
- **Cards**: the same card markup as Home (§3.1), except Danish courses show "(🇩🇰)" instead of "(DK)" (:202-204).
- **Load more** (:347-353, JS :385-453):
  - The server renders every matched card. JS then hides everything after the first 32 with `d-none`.
  - "Load More (n)" reveals 30 more with a 0.5s fade. "Load All Remaining" reveals the rest. Status text reads "Showing X of Y items".
  - This is not pagination: the full HTML ships on every load (§4.4).
- End of page: "Return to top of page" and "Last updated".

### 1.4 Course detail `/course/<id>` (`course.html`)
1. **Tab strip** (:6-13): the active tab is the course code; the second tab is "DTU's website" → `https://kurser.dtu.dk/course/{{COURSE}}`, opening in the same tab.
2. **Title**: h5 "{{COURSE}} {{NAME}}" (:22). This is the only heading and it is small.
3. **Card grid** `row-cols-1 row-cols-xl-2` (:26). Each card starts with a grey bar image `colors/grey.png` (alt "...").
   - **Card A, stats** (`max-width: 36rem`, :29-256):
     - Basic info table: Danish title, language, ECTS points, course type, sign-ups ("N students per semester"), schedule (`TIME_OF_WEEK|safe`, e.g. "F4A (Spring, Tues 13-17)<br>E4A (Autumn…)").
     - Grade stacked bar with "[?]" (§3.2).
     - Average grade "(N grades)"; failed students "X% (+Y% absent)".
     - Workload burden: score plus a skull image (clicking opens Semester breakdown). "Overworked students X% (a% mildly + b% severely)".
     - Average rating with "(N evaluations)", shown in red with "– low sample size!" when there are 1-9 evaluations. Below it, three sub-ratings with star images (clicking opens Semester breakdown).
     - Footer: "Last updated: {{LAST_UPDATED}}", in Danish date format, e.g. "04. maj, 2026".
   - **Card B, people and exam** (`max-width: 24rem`, :261-439):
     - Main responsible: name, photo (orbit.dtu.dk webp, falls back to `blank_photo.png` via onerror/onload), and an underlined list of their other courses linking to `/course/<id>`.
     - Exam type; hand-ins (`ASSIGNMENTS`).
     - Buttons "Grade distribution" and "Semester breakdown" (modals).
     - Up to 4 co-responsibles in the same layout.
     - Footer: Last updated.
4. **"Additional information:" card** (`max-width: 54rem`, :448-697). Each row is shown only when its value is not `'None'`:
   - Description.
   - Location (`CAMPUS_RAW`).
   - Prerequisites, up to three rows that are all labelled "Prerequisites:": legacy recommended, recommended, mandatory. Each has a "[Show graph]" toggle. If both mandatory and recommended are None it shows "No prerequisites required", even when the legacy row is also shown.
   - Subsequent courses with "[Show graph]".
   - Scope and form; Study lines; Webpage (`HOME_PAGE`); Exam info (`EXAM_TYPE_RAW`); Aid (`EXAM_AID_RAW`); Duration; Objectives; Content; Teacher's note (`REMARKS`); Special remark (`HIGHLIGHTED_MESSAGE`).
5. **Modals** (§3.2): Semester breakdown (:699-795) and Grade distribution (:798-1213).
6. **Prerequisite/subsequent graph** (inline JS :1361-1776, CSS :1219-1359):
   - Data is fetched lazily from `raw.githubusercontent.com/Marcrulo/DTU-courses-extension/.../id_to_name.json` and `graphs.json` (:1375-1376).
   - Layered columns of course codes with SVG arrow lines. Hovering a code dims everything except that node and its neighbours, and shows a CSS tooltip with the course name.
   - Graphs with more than 50 nodes show a "Large graph warning – Show anyway" button.
   - If any layer has more than 6 nodes, the graph opens in a blurred 180px preview with "▼ Click to expand full graph ▼".
   - Credit line: "Made in collaboration with DTU-courses-extension".
   - Spinner text: "Loading graph...". Error states: "Error loading graph data." and "No graph data available…".

### 1.5 Recommender `/recommender` (`recommender.html`)
- Hero: h1 "DTU Course Recommender", lead "Find courses similar to ones you've taken or are interested in." (:363-368).
- **Left card, "Input your Preferences:"** (`max-width: 36rem`, :373-479):
  - Button "📋 Study-Planner toolset" opens the Bulk Import modal.
  - Box 1, inputs:
    - "Search by Name or ID": autocomplete that includes historical courses.
    - "Or describe your interests": a textarea plus "Add" (Ctrl+Enter also adds).
    - "Or import from study line": a `<select>` of all study lines plus "Add".
  - Box 2, Top 5: placeholder "Your top 5 course recommendations will be displayed here." After a run it lists the top 5 as code link + name + red "NN% match" pill.
  - Box 3, "✅ Prerequisites Met": hidden until the first run. Text "You have the prerequisites for N subsequent courses." with a "View Unlocked Courses" modal button.
  - Box 4, view toggles: "Browse N additional suggestions:" with 🎠 Carousel / 📜 List / 📊 Analyzer (default).
- **Right card, "Course Basket"** (`max-width: 24rem`, :483-522):
  - Subtitle "Courses similar to these will be found."
  - "📄 CV Helper" button.
  - Basket list ("Basket is empty."); each item has a × to remove it. Text items are italic, historical items are styled separately.
  - Footer: "⚙️ Configure Filters" (opens the modal) and "Get Recommendations" (red). While loading, the button reads "Analyzing, please wait..." and is disabled.
- **Results area** (:529-563):
  - Carousel: prev/next arrows, dots (hidden when there are more than 15 pages), "Showing a-b of N". Shows 1, 2 or 3 per view at ≤768 / ≤992 / wider.
  - List: max 36rem wide, 10 at a time, "Load More (N remaining)".
  - Analyzer: course-card grid (§3.3), 30 at a time.
  - Errors appear in a red `alert-danger` div (:565).
- **Modals**:
  - Similarity Breakdown (:1604-1634): one row per basket item with its match %; above 70% is green bold, 41-70% dark, ≤40% muted.
  - CV Helper (:1636-1661): "Copy course data for AI purposes", a read-only textarea from `/cv_helper`, a "Copy Text" button, and "Copied to clipboard!" feedback.
  - Unlocked Courses (:1663-1684): list of code + name linking to the course page.
  - Bulk Import "Import courses from Study-Planner" (:1686-1716): placeholder "Go to studieplan.dtu.dk and copy-paste all of your prior course enrollments…", "Extract to Basket" (Ctrl+Enter extracts and also runs recommendations), with feedback text.
  - Filter Recommendations (:1718-2089): see §2.4.
- Older recommender versions, for reference only:
  - v1: standalone page without the navbar. "Search Courses" / "Paste Course List" tabs, course-type filter, links go straight to kurser.dtu.dk (recommender_v1.html:513-600, :943, :986).
  - v2: a "Step 1: Input Method / Step 2: Your Basket / Step 3: Filter Recommendations" wizard layout (recommender_v2.html:412-800). The step framing is a pattern worth keeping.

### 1.6 FAQ `/faq` (`faq.html`)
- h1 "🤔 Frequently Asked Questions 🤔", then the course-card legend shown only as an image (`course_card_explainer.png`, 258 KB, alt "Bootstrap", max 800px, :14-15).
- 17 Q&As as h2 + p, all fully expanded with no accordion or search. Topics:
  - Who are you; what is this site; how to use it; why you made it; how long it took.
  - Is the data up to date ("Summer Exams 2026; 1-2 updates/yr"); how you got the data.
  - GitHub; raw data; missing data.
  - **How the 5-star rating is calculated**: mean of 3 evaluation questions on a 1-5 scale; 3 stars = [2.75, 3.25]; colour hard-coded, 4.0 green / 3.5 yellow / 3.0 orange.
  - **How the 5-skull workload is calculated**: from the "time spent" evaluation question; "More" and "Much more" count as overworked.
  - Future plans; other universities; bugs; contact; fun fact; /wip; "May I leave now?".
- The usage answer still says "use the filter in the Home tab" (:33).

### 1.7 Error pages
- `404_invalid_course.html`:
  - "🤔 Error! Course not found in database! 🤔".
  - "Search input must be a 5-digit course number. You cannot use words in your search."
  - "Only courses available during {{current_year}} can be searched…"
  - A link to `https://kurser.dtu.dk/course/{{course}}`, unless the search was invalid (:6-12).
- The other two 404 templates are unused.

---

## 2. Interactive elements

### 2.1 Home filter form → GET `/browse` (home.html:118-1187)
Each checkbox's `name` is the filter key and its `value` is the category. The backend ORs keys within a category and ANDs across categories (context_dicts.py:49-69). An unchecked category applies no restriction. If nothing at all is checked, the result is empty.

| Row | Controls (name=value) | Defaults |
|---|---|---|
| Language | `dk=language` "🍺Danish", `eng=language` "🚀English" | unchecked |
| Type | `beng` 🐇BEng, `bsc` 🐢BSc, `msc` 👩‍🎓MSc, `phd` 🧙Ph.D. (value `course_type`) | BEng, BSc, MSc checked |
| Season | `autumn` 🍂, `spring` 🌱, `jan` ❄️January, `jun` ☀️June, `jul` ⛱️July, `aug` 🌻August (value `semester_period`) | none |
| Schedule [?] | `Mon 8-12`…`Fri 8-12`, `Mon 13-17`…`Fri 13-17`, `offschedule`, `Tue 18-22`, `3-week` (value `timetable`). The first 12 labels are only tiny PNG grid icons with no alt text (:221-275) | none |
| Exam type | `written` ✍, `oral` 🦜, `reporthandin` 📑"No exam" | none |
| Exam details | `passfail=gradetype` 💯, `digitalaid=aid` 💻"All aid (computers allowed)", `noassign=assignments` 🙏"No hand-ins before exam", `internal=examiner` 🔑 | none |
| Study lines | Three collapse toggles (BEng / BSc / MSc), 73 checkboxes in total (value `study_lines`). Each group includes "Show courses not linked to any study line" (`no_studyline`) | collapsed |
| Institutes | Collapse toggles "Institutes" and "Locations". Institutes: 01 Applied Mathematics, 02 Computer Science, 10 Physics, 12, 22, 23, 25, 26, 27, 29, 30, 34, 41, 42, 46, 47, 62 BEng, KU. Locations: `lyngby`, `ballerup`, `elsewhere` (value `campus`) | collapsed |

- **Silent broken filters**: four study-line checkboxes use names the backend doesn't know. Checking them filters nothing.
  - `BENG_CE` "BEng, Civil Engineering" (:411). Backend has BENG_CE1/CE2.
  - `BENG_ITE2` "BEng, IT and Economics" (:467). Backend has BENG_ITE.
  - `BSC_BI` "BSc, Biotechnology" (:563). Backend has BSC_B.
  - `BSC_MT1` "BSc, Mathematics and Technology" (:675). Backend has BSC_MT.
- **Supported by the backend but missing from the UI** (`static/pandas_df/json_filter_dct.json`):
  - Study lines BENG_NAOE, MSC_SES, MSC_SET, MSC_SFA; institute 31.
  - `examiner=external`; `aid=analogaid|noaid`; `assignments=reports|exercises|experiments`; `gradetype=sevenstep`.
  - `course_duration=13week|3week|16week|26week|xweek`.
  - `time_of_week` DTU slot codes F1A…E5B, E7/F7, W3january…
- Missing controls: no ECTS filter, no min-rating / max-workload / grade sliders, no text or keyword search, no reset/clear button, no "N results" preview before submitting, and no count of active filters.
- The labels "Language:" and "Type:" are `<th>` elements, not `<legend>`/`<fieldset>`, so the checkbox groups have no group label.

### 2.2 Browse sorting (browse.html)
- All sorting is links that reload the page. The current sort isn't highlighted.
- If the URL has only `sort_by` and no filters, `args.args` is empty and the links become `&sort_by=…`, which resolves to a broken relative URL (current_args, context_dicts.py:170-184).

### 2.3 Course page
- Bootstrap modals: "[?]" next to Grades, the clickable grade bars, the clickable star/skull images, and the two buttons.
- "[Show graph]" / "[Hide graph]" toggles are `<small onclick>`. They are not buttons, so they can't be reached by keyboard.
- Graph hover highlight is mouse-only.
- Clickable course links: co-responsible course lists and graph nodes. The prerequisite text itself is **plain text, not links** (sample: "01001/01003 , Latest at the same semester").
- `HOME_PAGE` and URLs inside `EXAM_TYPE_RAW` are rendered as plain text, not anchors (sample `https://01004.compute.dtu.dk/`).

### 2.4 Recommender
- Autocomplete: 10ms debounce, max 10 rows, skips courses already in the basket, Enter picks the first result, Ctrl+Enter picks and submits (:1491-1509).
- Study-line import: POST `/get_study_line_courses` adds every course on the line. Feedback: "Successfully added N courses from X" for 4s, otherwise `alert()`.
- Bulk extract: POST `/extract_courses`. Error feedback: "No valid 5-digit course codes found!".
- Validation and errors use browser `alert()` in several places: "Please enter a longer description." (:687), "Please select a study line first." (:742), "Add at least one course…" for the CV Helper (:1541).
- Filters modal:
  - Language uses names `Danish`/`English`, unlike Home's `dk`/`eng`.
  - Type defaults to BEng/BSc/MSc checked.
  - Season (6), Schedule (13, with a collapsible legend), Exam type (3).
  - **Bug**: `addFilter('semester_period','semester_period',4)` only reads 4 of the 6 season boxes, so **July and August are ignored** (:878 vs :1824-1829).
  - The modal has only "Close"; filters take effect on the next "Get Recommendations". There's no "N filters active" indicator, and no study-line, institute, campus or exam-detail filters.
- Request: POST `/recommend` with `{basket_items, n_recommendations: 9001, filters}`, which returns every result in one response.
- Nothing persists: basket, results and filters are JS variables only. Reloading loses everything. No localStorage, no URL state, no share link.

### 2.5 JS libraries and techniques
- No chart library. The grade bars are hand-made float spans, presumably sized by `static/assets/dist/js/gradegraph.js` (course.html:1216). That file is **missing from the repo**: `dist/` is gitignored, so a fresh deploy gets wrong bar sizes. Production screenshots show proportional bars, so the file exists on the server.
- The prerequisite graph is custom SVG `<line>` elements drawn over an HTML table.
- Lazy loading: native `loading="lazy"` on card photos and `me.png`, plus the client-side "Load more" on browse and recommender. There's no infinite scroll.
- Autocomplete is hand-rolled `fetch` against `/search_courses`.

---

## 3. Course data and how it's shown

### 3.1 Course card (home.html:1206-1298, browse.html:161-275; legend only in faq image)
- A 20px red bar image (`colors/red.png`, alt "red-color") on top.
- The whole card is one empty `<a class="stretched-link">` with no accessible name; there's also a stray `</a>` at home:1213.
- Title: h6 "{{code}} {{english name}}".
- Two-column icon grid (the emoji are `<th>` row headers):
  - 🎓 course type (+ "(DK)" or "(🇩🇰)" for Danish courses).
  - 🕗 season, plus a schedule grid icon when Fall/Spring. Possible values include "January" and "2+ Timeslots".
  - 🏅 "N pts" ECTS.
  - 📚 exam (Written exam / Oral exam / Exam project …).
  - 👩‍🎓 "N ppl" sign-ups per semester.
  - ❌ average grade "(X% fail)", or "P/F".
  - ⭐ "X stars", with a red "(!)" `title` warning when there are 1-9 evaluations.
  - ☠️ "X workload". "–" means no data.
- Right column: the responsible teacher's photo (alt "100%x280"; external orbit.dtu.dk URL; falls back to `blank_photo.png` on error or images narrower than 10px).
- Bottom row:
  - Star bar image `ratings/{0-9}-star.png`: 10 half-star tiers, green/yellow/orange by tier.
  - Evaluation count in parentheses. The font shrinks with nested `<small>` for 100+ and 1000+; it turns red when under 10.
  - Skull bar image `ratings/{0-9}-skull.png`, coloured green to orange.
- There's no "add to plan", compare or bookmark action. The match % badge appears only in the recommender version of the card.

### 3.2 Course page visuals
- **Grade stacked bar**, 16px (course.html:69-170). Segment colours:
  - Pass `#01D93F`.
  - 7-step grades: 12 `#244710`, 10 `#1B791F`, 7 `#01D93F`, 4 `#FDBF02`, 02 `#FE7E03`, 00 `#D44B33`, -3 `#8E1600`.
  - Fail `#8E1601`, Absent `#989898`.
  - **Pass and 7 share a colour, and -3 and Fail are effectively the same colour.**
  - The % labels inside the bars use fully transparent colours (`#xxxxxx00`), so they are invisible. Values appear only in the `title` tooltip ("Grade 12").
- **Grade distribution modal**: a table of colour swatch + grade + % + mini bar + (count), with a Total row (:810-1205).
- **Semester breakdown modal**:
  - Table 1: semester (Autumn/Spring YYYY, E20…F27) × Grade / Failed / Students. Semesters with under 5% of the course's total students are hidden. Footer row "All semesters".
  - Table 2: evaluation categories A workload, B learned, C motivating, D feedback, plus Votes, per semester (:711-788).
- **Rating images**: tier PNGs with no alt text, height 36. Clicking opens the semester breakdown, but nothing tells the user that.
- **Data fields not shown on the course page** even though they're in the data: `SEMESTER_PERIOD` (only visible inside the schedule text), `INSTITUTE`, `TIMETABLE_PLACEMENT`, `TOTAL_SEMESTERS`, `RATING_VOTES`, `WORKLOAD_VOTES`. Exam date isn't shown anywhere.
- **Bug**: "Objectives" checks `data.OBJECTIVES`, a column that doesn't exist (only `LEARNING_OBJECTIVES` exists), so the row always renders. It shows "None" when the data is missing (:642-648).
- A long list of scraped HTML is rendered with `|safe`: TIME_OF_WEEK, DESCRIPTION, CAMPUS_RAW, prerequisites, SUBSEQUENT, SCOPE, STUDY_LINES, HOME_PAGE, EXAM_*, CONTENT, REMARKS, HIGHLIGHTED_MESSAGE.

### 3.3 Recommender cards
- **Analyzer card** (recommender.html:1140-1271): the same layout as §3.1, plus a red "NN% match" badge in the title. It always shows "(DK)" for Danish courses.
- **Carousel/List card** (:1080-1137):
  - Code link, then "NN% match" and a "View breakdown" button.
  - Bold name; a subtitle line "🎓 type 🏅 ECTS | 📚 exam | 🕗 season [schedule icon]". "2+ Timeslots" is relabelled "Multiple semesters".
  - Description clamped to 4 lines; "View Course ↗" button.

---

## 4. Responsiveness, accessibility, theming, language, performance

### 4.1 Responsive layout and mobile
- The viewport meta tag is present.
- Fixed widths overflow on phones: the Quick-nav card is `width: 26rem` (416px) and the filter card is `width: 42rem` (672px) (home.html:73, :114). The filter rows also use fixed `th 9rem` and `td 7.8-10.6rem` widths.
- Card grids switch to 3 columns only at `xxl` (≥1400px). Below that, a laptop at 1366px shows **a single 26rem-wide column** with the rest of the screen empty.
- The course page stacks its two cards below `xl` (1200px). Its cards are capped at 36, 24 and 54rem.
- The navbar collapses below 768px. The search box is fixed at 200px.
- Hover-only interactions don't work on touch: graph tooltips and neighbour highlighting, and the `title` tooltips on grades and the low-sample warning.
- Schedule icons and the "[?]" triggers are very small tap targets.
- Only the recommender carousel adapts itself (media queries in recommender.html CSS at 992 and 768px). The rest relies on Bootstrap's grid.
- Vertical spacing uses stacks of `<br>` (4 to 8 at a time) throughout.

### 4.2 Accessibility
- **Images**:
  - The schedule checkboxes' labels are only `<img>` with no alt (home.html:221-275; recommender filter modal), so screen readers announce nothing for 12 of the 13 options.
  - Star and skull rating images have no alt (home.html:1285/1289, course.html:197-246).
  - The brand logo and the FAQ legend both have `alt="Bootstrap"` (base.html:127, faq.html:15). Teacher photos have `alt="100%x280"`; grey bars have `alt="..."`.
  - The only explanation of the card icons is a raster image.
- **Keyboard**:
  - "[?]" triggers, graph toggles and clickable rating images are `<small>`/`<img>` elements with `data-toggle` or `onclick`. None is focusable.
  - The navbar autocomplete has no arrow-key support. The basket remove control "×" is a `<span onclick>`.
  - Card links have no accessible name: an empty `stretched-link`.
- **Meaning carried by colour or emoji alone**:
  - Grade colours repeat (Pass = 7, -3 ≈ Fail), and the green/red scale is a problem for colour-blind users.
  - Emoji act as row headers; screen readers read them as "graduation cap", "skull and crossbones" and so on.
  - The low-sample warning lives only in a `title` tooltip.
- **Contrast**:
  - `navbar-dark` links (white at 55% opacity on #990000) look faint pink in the screenshots and likely fail WCAG AA.
  - Lots of `text-muted` text nested in several `<small>` tags, plus xx-small text in bars.
- **Structure**:
  - Heading levels jump around: the course page has only an h5 title; FAQ questions are h2 under a display h1.
  - Every page has the same `<title>`, and the nav always marks Home as current.
  - Recommender errors use `alert()` or an alert div with no `aria-live`.
  - Modals use BS alpha markup (`.close`), with `aria-label` on the close buttons.

### 4.3 Dark mode and language
- There's no dark mode: no `prefers-color-scheme` rule anywhere, and colours are hard-coded (`#990000`, white card footers, grey PNG bars).
- The UI is English only (`lang="en"`), with no language switcher.
- Danish content leaks into the English UI: the Danish course title on the course page, Danish-format "Last updated" dates ("04. maj, 2026"), Danish marker "(DK)" vs "(🇩🇰)" on different pages, and a beer emoji 🍺 for Danish.
- Course names on cards are English only. Search matches Danish names, but results show the English name.

### 4.4 Performance
- **Blocking resources in `<head>` on every page, with no defer**: Bootstrap 5.0.0-alpha1 CSS + JS (stackpath), Font Awesome 4.0.3 CSS (unused except in `wip.html`), Popper, jQuery 3.2.1, and async GTM. That's about 6 third-party requests before content.
- **Browse all** (the navbar default): the server renders every course card. The card template is about 6.6 KB before data, times about 1,746 courses, so roughly 10 MB of HTML; JS then hides all but 32.
  - Each card also requests `red.png`, a schedule png, a star png and a skull png. These are small and cached.
  - Teacher photos are lazy and hotlinked from orbit.dtu.dk.
- Home makes about 21 photo requests plus `me.png` (968 KB, lazy but above the fold). The FAQ legend image is 258 KB.
- The course page graph pulls two JSON files from raw.githubusercontent.com at runtime. Their size is unknown, they aren't cached by this app, and they depend on a third-party repo.
- `gradegraph.js` is referenced but not in the repo (§2.5).
- Search fires a request almost on every keystroke (25ms / 10ms debounce).
- `/recommend` asks for 9001 results, i.e. everything in one response.
- **Image asset categories** (`static/assets/`):
  - `brand/`: logos, favicon, rocket, `blank_photo`, `me`, `course_card_explainer`.
  - `colors/`: 11 single-colour bar PNGs, about 150 B each; only red and grey are used.
  - `ratings/`: 0-9 star and 0-9 skull tier bars.
  - `schedules/`: 17 tiny week-grid icons.
  - `screenshots/`: 5 marketing screenshots from 2024.
- `static/csv_files` (about 33 MB) and `pandas_df` (pickles, ONNX model) are server data, not served to pages.

---

## 5. UX pain points, especially for a new student building a semester plan

Journey: "a new DTU student lands from an official kurser.dtu.dk course page and needs to build their semester or study plan."

### 5.1 Landing
- The way in is `/course/<id>`. Nothing on that page says what the site is, how the ratings work, or where to go next. The only heading is a small h5, and the breadcrumb is a "DTU's website" tab back to the official page.
- Nothing summarises the course at a glance in plain words (good/bad, hard/easy, fits my schedule). The data is spread across two cards, two modals and a long "Additional information" card.
- The meaning of stars and skulls is only in the FAQ image. The page doesn't explain the 1-5 evaluation scale or "workload 3.77".
- Prerequisite codes can't be clicked in text form. The graph needs a click, loads from a third-party source, and works by mouse hover.

### 5.2 No planning tools at all
- **No plan or semester builder**: no "add to plan" or "save" button on cards, the course page or recommender results.
- **No schedule conflict check**, although every course has a DTU slot code (E1A…F5B, `TIME_OF_WEEK`) and the schedule-icon system already exists. There's no week-grid view of chosen courses.
- **No ECTS total or tracking** (30 ECTS per semester), and no ECTS filter.
- **No prerequisite check** against what the student has passed or planned. The recommender's "Prerequisites Met / Unlocked" covers only basket courses.
- **No study-programme view**: study lines are only a filter. There's no "my programme requires X, Y, Z" or polytechnical foundation / mandatory / elective split.
- **Nothing is saved or shareable**: no accounts (login/sign-up templates are dead), no localStorage, no export (iCal, PDF, CSV) and no shareable plan URL. The recommender basket is lost on reload.
- **No side-by-side comparison** of two or more courses.
- **No timeline across semesters** (autumn → January → spring → June), even though `semester_period` and the 3-week periods exist in the data.
- **No exam dates or exam-collision check**.

### 5.3 Finding courses
- Filtering is split in two: Home has the full filter panel; the Recommender has a different, smaller filter set in a modal, with different names (`dk` vs `Danish`) and the July/August bug.
- After filtering, "Adjust filters" goes back to Home with every choice reset, so each refinement starts from scratch.
- Getting 0 results is a dead end (§1.3).
- Search accepts only exact 5-digit codes on submit. Typing a name and pressing Enter shows the error "Search input must be a 5-digit course number. You cannot use words in your search", even though the autocomplete above does match names.
- Four study-line filters silently do nothing (§2.1); the site's own course count is "1700+" in places.
- Sorting covers only one dimension at a time and doesn't show which sort is active. Rating, workload, grade and pass rate can't be combined, e.g. "high rating and low workload".
- Default Browse shows every course, about 10 MB, and the 1-column layout below 1400px makes scanning slow.
- The course sample on Home is a fixed list of the author's picks. It isn't personalised or labelled as such.

### 5.4 Trust and information gaps
- Low-sample warnings exist (the red (!) for fewer than 10 evaluations), but the mean hides how spread out the ratings are.
- There's no "data as of" per metric, and the dates are mixed-language and inconsistent: Config 30/09/2026, FAQ "Summer Exams 2026", screenshots from 2023/2024.
- Semester breakdown hides semesters with under 5% of students without saying so.
- Teacher photos hotlinked from orbit can break, and the fallback is a blank silhouette.

### 5.5 Visual and design debt
- Personal branding ("A software project by Jonatan Rasmussen s183649", a 1 MB author photo in the quick nav) competes with the task.
- Emoji everywhere (🍺 Danish, 🐢 BSc, 🧙 PhD, ☠️ workload) feels unserious for an academic planning tool.
- Many grey bar images, `<br>` spacing and inconsistent max-widths (24/26/36/42/54rem).
- Leftover demo UI: the `modal_search_filter` "Popover in a modal" modal, `/wip`, `/test`, the Bootstrap cheatsheet `index.html`, and three dead recommender versions.
- The framework mixes Bootstrap 5 alpha with BS4 class names (`ml-3`, `font-weight-bold`, `badge-pill`, `text-right`, `input-group-append`, `btn-block`).

---

## 6. External links and where they appear
- **kurser.dtu.dk**:
  - Course page tab "DTU's website" → `https://kurser.dtu.dk/course/{{COURSE}}` (course.html:11).
  - Course-not-found page → `https://kurser.dtu.dk/course/{{course}}` (404_invalid_course.html:11).
  - FAQ → `https://kurser.dtu.dk/archive/volumes` (faq.html:53), plus example URLs shown as `<code>`.
  - recommender_v1 cards: "Visit Official Page" / "View Course Details" (recommender_v1.html:943, :986, :1031).
- **studieplan.dtu.dk**: mentioned only in the Bulk Import textarea placeholder, not as a link (recommender.html:1703).
- **orbit.dtu.dk**: teacher photos (data `*_PIC`), shown on cards and the course page.
- **Course home pages** (e.g. `01004.compute.dtu.dk`): shown as plain text in the "Webpage:" row, not linked (course.html:600).
- **Evaluations and grades**: the data is scraped from DTU evaluation and grade pages, but **the site never links to the original evaluation or grade pages**.
- **GitHub**:
  - Marcrulo/DTU-courses-extension: JSON fetched at runtime and credit links (course.html:1375-1376, :1469, :1545).
  - JonatanRasmussen/dtu-course-browser, repo and CSV folder (faq.html:58, :63, :93).
- **Reddit** announcement post (faq.html:93, :98); **YouTube** "Best of Mat 1" (faq.html:103); **mailto** jonatan.j.k.rasmussen@gmail.dk (faq.html:98).
- **Google Tag Manager / Analytics** on every page (base.html:12); **unsplash** image on /wip (wip.html:218).
- **CDNs**: stackpath.bootstrapcdn.com, cdnjs.cloudflare.com (Font Awesome, jQuery), cdn.jsdelivr.net (Popper) (base.html:27-40).

---

Not checked: I didn't run the app, so the ~10 MB Browse figure is an estimate (template size × course count), and I didn't measure the size of the GitHub graph JSON. Production behaviour of `gradegraph.js` is inferred from the screenshots. I skimmed rather than fully read `wip.html`, `index.html` and the unrouted recommender v2x/v3 because they're dead or demo pages; the live recommender was read in full.
