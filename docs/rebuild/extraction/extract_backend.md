# DTU Course Analyzer: web backend behaviour (extracted from the code)

Repo: /Users/puddu/Documents/GitHub/dtu-course-browser (branch main, commit 8e71e43). I only read files; nothing was changed.
Paths are relative to the repo root. Templates are cited where they hold the behaviour, because many "backend" rules live in Jinja and inline JS.

---

## 0. App bootstrap

- `website_launch.py:4-7`: `create_app()` then `app.run(debug=True)`, which is the Werkzeug dev server with the debugger and reloader on. Line 8 holds a commented-out public IP and port 8080. Production runs on PythonAnywhere (WSGI file not in the repo).
- `website/__init__.py:7-38`: a Flask app factory.
  - The static folder is `website/static` (line 9).
  - `SECRET_KEY` is read from `./secretkey.txt`. If that file is missing it falls back to the literal `'default_secret_key'` and prints a warning (lines 12-21). The session is used only for `flash()` messages.
  - It registers 5 blueprints, all at `url_prefix='/'`: views, misc, browse, course_database, recommender (lines 23-36).
  - There is no `errorhandler`, no `MAX_CONTENT_LENGTH`, no CSRF, no rate limiting, no caching headers, no logging config, and no `robots.txt` or sitemap (grep found none).
- Import side effects at startup:
  - `course_database.py:19-27` loads `course_df.pkl` into the global `df` and builds `course_set`.
  - `recommender.py:55-57` (`ENABLE_EAGER_LOADING=True`) builds `FilterRecommendations`: it loads the pkl again, loads the ONNX model and tokenizer, loads the embeddings cache and the filtered df cache, and loads `json_filter_dct.json`.
- File resolution pattern used everywhere: try `dtu_course_project/website/static/pandas_df/<file>` (the PythonAnywhere path, `file_name_consts.py:25,43`), and on `FileNameConsts` `FileNotFoundError` fall back to the relative `website/static/pandas_df/<file>` (`file_name_consts.py:42`). Examples: `course_database.py:19-26`, `context_dicts.py:8-47`, `recommend_utils.py:161-168,566-582`, `recommender.py:31-41`. Both are relative to the CWD, so the app only works when started from a particular directory.

## 1. Routes

The global search box sits in the navbar (`base.html:148-154`). Its form has no `action`, so it POSTs to the current URL. That is why almost every route accepts POST and starts with `if POST: return submit_search_field(form['search_field_input'])`.

| # | Method | URL | Handler (file:line) | Params | Returns |
|---|---|---|---|---|---|
| 1 | GET, POST | `/` | `views.home` views.py:14-29 | Query args are read through `current_args()` but never used by `home.html` (wasted work). POST: `search_field_input` | `home.html` with `course_lists` (a hard-coded "Course sample" of 21 IDs, `context_dicts.py:110-126`), `dicts_to_display`, `data()` (all 16 per-course dicts), `args`, `last_update` |
| 2 | GET, POST | `/browse` | `browse.browse_courses` browse.py:12-38 | Filter args `?<valueCode>=<category>` (inverted key/value, see §2), `sort_by`, `sort_reverse`. POST = search | If there are no query args, a 302 to `/browse?dk=language&eng=language` (line 22-23). Otherwise `browse.html` with the filtered and sorted list, `data()`, `summary_stats`, `args`, `last_update` |
| 3 | GET, POST | `/course/<string:course_number>` | `course_database.route_to_course` course_database.py:29-72 | Path segment, case-sensitive and not normalised. POST = search | Known ID: `course.html` with `data` (one df row as a dict) and `semesters`. `xxxxx`: a random course rendered at the same URL (line 38-43). Unknown: `404_invalid_course.html` **with HTTP 200** (soft 404, line 72) |
| 4 | GET | `/random/` | course_database.py:74-78 | none | 302 to `/course/<random id>`. `/random` (no slash) gets a 308 from Flask strict_slashes. The navbar links `/random` |
| 5 | GET, POST | `/faq` | misc.py:11-15 | POST = search | static `faq.html` |
| 6 | GET, POST | `/test` | misc.py:17-21 | POST = search | `test.html`, a public dev page |
| 7 | GET, POST | `/wip` | misc.py:23-28 | POST = search | `wip.html` with hard-coded fake grade data for course "12345". Public dev page |
| 8 | GET, POST | `/recommender` | recommender.py:68-73 | POST = search | `recommender.html` with `study_lines = sorted(InfoConsts.study_lines.values_raw)` (~75 names) |
| 9 | GET, POST | `/recommender_v1` | recommender.py:76-81 | same | `recommender_v1.html` (legacy UI, still live) |
| 10 | GET, POST | `/recommender_v2` | recommender.py:83-88 | same | `recommender_v2.html` (legacy UI, still live) |
| 11 | GET | `/search_courses` | recommender.py:91-141 | `query` (stripped, upper-cased), `include_historical=true/false` | JSON `{success, results:[{course_id,name,danish_name,language,ects,is_historical}]}`, at most 20 matches (+1 historical) |
| 12 | POST | `/extract_courses` | recommender.py:144-188 | JSON `{text}` | JSON `{success, courses:[{course_id,name,is_historical}], count}` |
| 13 | POST | `/get_study_line_courses` | recommender.py:191-220 | JSON `{study_line}` (the raw study-line name) | JSON `{success, courses:[{course_id,name}]}` |
| 14 | POST | `/recommend` | recommender.py:223-364 | JSON `{basket_items:[{type:'course'|'text', value, label?}], filters:{cat:[values]}, n_recommendations}` | JSON `{success, recommendations[], unlocked_courses[], basket_items, filters_applied}` |
| 15 | GET | `/course_info/<course_id>` | recommender.py:367-377 | path id (upper-cased) | JSON `{success, info:{course_id,name,ects,institute,description[:200]}}`. Not called by the current `recommender.html` (dead or legacy) |
| 16 | POST | `/cv_helper` | recommender.py:380-404 | JSON `{course_ids:[]}` | JSON `{success, text}`: for each ID, `"--- Course X ---\n<context>"` or "No context available.", joined by blank lines |
| 17 | GET | `/static/<path>` | Flask built-in | none | Everything under `website/static`, which includes **all data files**: `pandas_df/course_df.pkl` (9 MB), `extended_pkl.pkl` (21 MB), `cv_helper.json` (6.5 MB), the embeddings pickles, `model_onnx/model.onnx` (23 MB), `all-MiniLM-L6-v2/`, and `csv_files/*.csv` plus a zip |

Search submit (`search.py:7-20`):
- The input is stripped. Empty: flash "empty search", then redirect to `/course/invalid_search`. Length <5 or >5: flash an error, then the same redirect. Exactly 5 characters: redirect to `/course/<input>`.
- The input is **not upper-cased**, so `ku181` fails even though `KU181` exists (195 of 1746 IDs are alphanumeric, e.g. `41B02`, `KU181`, `MA200`).
- There is no name search on the server. Name search happens only through the JS autocomplete calling `/search_courses` (`base.html:189-238`). It debounces for 25 ms, shows up to 10 results, and clicking one fills in the ID and submits the form.

Error handling:
- An unknown URL gets Flask's default 404 page. `404_not_found.html` and `404_empty_search.html` are never used.
- An invalid course renders `404_invalid_course.html` with status 200. It links to `https://kurser.dtu.dk/course/<input>` unless the input is `invalid_search` (`404_invalid_course.html:10-12`) and says that only courses from `{{last_update.current_year}}` can be searched.
- JSON endpoints wrap everything in `try/except Exception` and return `{'success': False, 'error': str(e)}` **with HTTP 200** (recommender.py:139-141, 186-188, 218-220, 362-364, 376-377, 402-404). This leaks exception text and also swallows Flask's 415/400 errors, which are `HTTPException`s and therefore subclasses of `Exception`.
- Flash messages render in `base.html:165-182`: category `error` shows a yellow alert, anything else green. `home.html:60-67` renders them a second time, without category.

## 2. Search and filtering (browse page)

### Filter state and URL format
- All state lives in the query string. Nothing goes into the session, cookies or localStorage on the server side.
- The format is inverted: the **key is the value code and the value is the category**, e.g. `/browse?dk=language&eng=language&bsc=course_type&Mon 8-12=timetable`. These come from the checkbox `name`/`value` pairs in the home form (`home.html:118-1190`), submitted with GET to `/browse`.
- Parsing (`context_dicts.py:49-69`):
  - `filter_dct = json_filter_dct.json`, re-read from disk on every call (`get_filter_dct`, lines 165-168).
  - Each `(key=value_code, val=category)` pair is accepted only if `filter_dct[category][value_code]` exists. Unknown pairs are silently ignored.
  - Course lists are **concatenated (OR) within a category** and **intersected (AND) across categories**.
  - If there are no valid filters the result is an empty set, so the page says "0 courses matched".
- `request.args.to_dict()` keeps only the first value for a repeated key. Because the key is the value code, two categories that share a code clash: `offschedule` exists in both `time_of_week` and `timetable`.
- `current_args` (`context_dicts.py:170-184`) rebuilds a canonical `?k=v&k=v` string that holds only the valid filter pairs (no `sort_by`, no `sort_reverse`). Templates use it as the base for the sort links: `browse.html:31-75` builds `href="{{args.args}}&sort_by=..."`. Values are not URL-encoded (e.g. `Mon 8-12`), and if `args.args` is empty the link becomes the relative `&sort_by=x`, which resolves to a broken path.

### Filter dimensions (keys from json_filter_dct.json; built by `csv_creator.py:216-255` from `InfoConsts.*.values_url`)

| Category (URL value) | Value codes | Meaning (info_consts.py) | Exposed in home form? |
|---|---|---|---|
| `language` | `dk`, `eng` | :81-90 | yes (default browse link = both) |
| `course_type` | `bsc`, `msc`, `beng`, `phd` | :135-144. BScMSc maps to MSc, Deltidsdiplom to BEng, Parttime master to MSc | yes; beng, bsc and msc are pre-checked (home.html:146-160) |
| `semester_period` | `spring`, `autumn`, `jan`, `jun`, `jul`, `aug` | :307-320 | yes |
| `timetable` (schedule block, day and time) | `Mon 8-12`, `Thu 13-17`, `Mon 13-17`, `Thu 8-12`, `Tue 8-12`, `Fri 13-17`, `Tue 13-17`, `Fri 8-12`, `Wed 8-12`, `Wed 13-17`, `Tue 18-22`, `3-week`, `offschedule` | :276-304 (merges E/F blocks) | yes, all 13 |
| `time_of_week` (exact block) | `F1A`…`F5B`, `F7`, `E1A`…`E5B`, `E7`, `W3january`, `W3june`, `W3july`, `W3august`, `offschedule` | :229-273 | no (URL only) |
| `exam_type` | `written`, `oral`, `reporthandin` | :147-169 | yes |
| `gradetype` | `sevenstep`, `passfail` | :93-105 | only `passfail` |
| `aid` | `digitalaid`, `analogaid`, `noaid` | :197-206 | only `digitalaid` |
| `assignments` | `noassign`, `reports`, `exercises`, `experiments` | :172-194 | only `noassign` |
| `examiner` | `internal`, `external` | :108-120 | only `internal` |
| `campus` | `lyngby`, `ballerup`, `elsewhere` | :123-132 | yes |
| `course_duration` | `13week`, `3week`, `16week`, `26week`, `xweek` | :209-226 | no (URL only) |
| `study_lines` | 76 codes, e.g. `BENG_AE`, `BSC_CE1`, `MSC_WE`, `no_studyline` (made by `shorten_study_lines`, :404-426, which keeps the uppercase letters and adds 1/2 suffixes to duplicates) | :323-429 | yes, in collapsible BEng/BSc/MSc groups. **4 checkbox names do not exist in the JSON and are silently ignored**: `BENG_CE`, `BENG_ITE2`, `BSC_BI`, `BSC_MT1` (verified by script) |
| `institute` (department) | `01`, `02`, `10`, `12`, `22`, `23`, `25`, `26`, `27`, `29`, `30`, `31`, `34`, `41`, `42`, `46`, `47`, `62`, `KU` | :697-737. Department is derived from the course-number prefix | yes |

Dimensions that do **not** exist as filters: ECTS, grade or pass-rate ranges, evaluation score ranges, workload ranges, free-text search. Those appear only as sort keys. Filtering is purely set membership on precomputed boolean (one-hot) columns.

### Sorting (`context_dicts.py:71-102`, `website_consts.py:58-79`)
- `sort_by` must be one of 20 keys in `json_as_sort_catagory`. Otherwise the order is `course_numbers.json` insertion order, i.e. ascending ID (lines 85-89).
- The order of a valid key is the **insertion order of the pre-sorted dict** in `course_data.json[sort_by]`. That dict was sorted ascending by value at build time (`website_consts.py:131-156`). For numeric columns (grade, fail, workload, rating, learning, motivation, feedback), string placeholders ("–", "P/F") are mapped to -0.1 so they sort as lowest.
- The boolean decides whether the order is reversed to get descending:
  - reversed (best or highest first): ects, signups, grade, fail, workload, rating, workload_tier, rating_tier, votes, eval_learning, eval_motivation, eval_feedback
  - not reversed (ascending): name, type, language, season, schedule, exam, responsible
- Then `sort_reverse` (key present, any value) reverses the final list (line 78-79). The toggle JS adds or removes `sort_reverse=true` (`browse.html:371-381`).
- UI sort options (`browse.html:31-75`): ID (default), Name, Rating, Workload, Grade, ECTS, Signups, Schedule, Failrate, Evaluation votes, Rating: learning, Rating: motivation, Rating: feedback, and an "Adjust filters" link back to `/`.
- Bug: `sort_by=course_numbers` is a "valid" key, but `course_numbers` is not nested in `course_data.json`, so the arrange order is empty and **0 courses are displayed** (lines 95-96 combined with 15-18).
- Bug: `sort_by=course_responsible` sorts by **teacher photo URL**.

### Pagination
- There is none on the server. All matched cards (up to 1746) are rendered into the HTML.
- `browse.html:383-454` hides cards on the client: the first 32 are shown, then "Load More (30)" and "Load All Remaining" buttons. The controls appear only when there are more than 29 results (browse.html:347).

### Summary stats (`context_dicts.py:186-211`, shown at browse.html:94-155)
- These are averages over the displayed courses for: rating, workload, learning, motivation, feedback (2 decimals), grade, failrate (1 decimal), signups (integer).
- Only values that are int or float **and not 0** are counted, so 0% fail courses are excluded (an upward bias), and "P/F" and "–" are excluded.
- Each stat calls a loader that re-parses `course_data.json` (8 more full parses).

### Browse and home course card (`home.html:1196-1305`; browse uses the same layout)
- Card fields: ID and name, linking to `/course/<id>`; teacher photo (Orbit URL or `blank_photo.png`, with an onerror/onload fallback for ≤10px images, lazy-loaded); course type, plus "(DK)" if Danish; season ("Fall"/"Spring"/"January"/…/"2+ Timeslots"/"No data") plus a schedule icon `assets/schedules/<schedule>_Schedule.png` when Fall or Spring; ECTS; exam ("Written exam", "Oral exam", "Exam project"); signups ("ppl"); average grade plus "% fail"; rating stars plus a "(!)" low-sample warning when 1-9 votes; workload; a rating tier image `ratings/<tier>-star.png`; a vote count with size styling; a workload tier image `ratings/<tier>-skull.png`.
- The value shortenings applied at build time are in `website_consts.py:158-246`:
  - course type: BSc+MSc becomes "BSc/MSc"
  - season: Autumn becomes "Fall"; multiple values become "2+ Timeslots"
  - schedule: mapped to icon keys `Mon_8_12`, `Multi_…`, `Multi_Unknown`
  - grades and scores: rounded to 1 decimal, with "P/F" and "–"
  - fail %: one decimal under 10, otherwise an integer; "No data" becomes "0"
  - exam: "Written/oral exam" becomes "Oral exam"; none or unknown becomes "Exam project"

## 3. Course detail page (`course_database.py:29-72`, `course.html`)

### Data preparation
- `data = df.loc[id].to_dict()` (line 45). The df is indexed by `COURSE` (`csv_creator.py:174`).
- `semesters` (lines 48-56): for each of `Config.course_semesters` (`E20,F21,…,E26,F27`, `config.py:4`) it builds `SEMESTER_NAME` ("Autumn 2020"/"Spring 2021") and copies every key containing that code with the prefix stripped (e.g. `E25_AVERAGE_GRADE` becomes `AVERAGE_GRADE`). It uses a substring match.
- Per-semester fields come from `csv_columns_consts.py:14-18`: TOTAL_STUDENTS, AVERAGE_GRADE, PERCENT_FAILED, GRADE_12…GRADE_MINUS_3, LEARNING_VOTES, WORKLOAD/LEARNING/MOTIVATION/FEEDBACK_AVERAGE_SCORE.
- NaN floats are turned into the string `'None'` (lines 59-61). Real Python `None` values are not converted, so templates comparing `!= 'None'` would print "None".
- The 5 `*_RESPONSIBLE_*_COURSES` strings are split on `<br />` into lists (lines 65-67). They are built by `format_study_lines.create_teacher_course_lst`.

### Sections rendered (course.html)
- **Sub-nav**: tabs for the course ID and "DTU's website", linking `https://kurser.dtu.dk/course/<id>` (:6-13). The title is `<ID> <NAME>` (:22).
- **Card 1, basics** (:35-60): Danish title, language, ECTS, course type, sign-ups ("students per semester"), schedule (`TIME_OF_WEEK|safe`).
- **Card 1, grades** (:66-87):
  - average grade with the total number of grades
  - failed % plus a computed absent % = `100*ABSENT/TOTAL_STUDENTS`, rounded to 1 decimal
  - a CSS stacked bar of the grade distribution (12, 10, 7, 4, 02, 00, -3, passed, failed, absent) that opens a modal
- **Card 1, workload** (:180-198):
  - `WORKLOAD_AVERAGE_SCORE` plus a skull tier image
  - "Overworked students" % = `100*(WORKLOAD_4_STAR + WORKLOAD_5_STAR)/LEARNING_VOTES`, split into mildly and severely. The denominator is LEARNING_VOTES, not WORKLOAD_VOTES.
- **Card 1, evaluations** (:200-240):
  - average rating plus the evaluation count, with a red "low sample size!" when 0 < MOTIVATION_VOTES < 10
  - "I have learned a lot", "Motivating activities", "Access to feedback", each with a score and a star tier image
- **Card 1 footer**: LAST_UPDATED.
- **Card 2**:
  - the main responsible: name, photo with a fallback, and links to the other courses they are responsible for
  - exam type
  - hand-ins (ASSIGNMENTS)
  - "Grade distribution" and "Semester breakdown" modal buttons
  - co-responsible 1-4, each shown only if `!= 'NO_DATA'`, with name, photo and course list (:265-435)
- **Additional information** (:452-690). Each row is shown only when the field is not `'None'`, and all of them use `|safe` (raw HTML from DTU):
  - Description
  - Location (CAMPUS_RAW)
  - Prerequisites: three variants, Legacy recommended, Recommended and Mandatory. Each has a "[Show graph]" toggle. "No prerequisites required" is shown when Mandatory and Recommended are both 'None'.
  - Subsequent courses: reverse prerequisites computed at build time (`csv_creator.py:176-202`) by regex `\b\d{5}\b` over all prerequisite texts. They come with a graph toggle.
  - Scope and form
  - Study lines, hidden when equal to the "Not linked…/Unknown" sentinel
  - Webpage (HOME_PAGE)
  - Exam info (EXAM_TYPE_RAW)
  - Aid (EXAM_AID_RAW)
  - Duration (EXAM_DURATION)
  - Objectives. Bug: the condition checks `data.OBJECTIVES`, which is undefined and therefore always true, but it renders `LEARNING_OBJECTIVES`.
  - Content
  - Teacher's note (REMARKS)
  - Special remark (HIGHLIGHTED_MESSAGE)
- **Semester breakdown modal** (:699-790):
  - **Historical trend table** with columns Semester, Grade, Failed %, Students. A semester row is shown only if its TOTAL_STUDENTS > 5% of the all-time total. The footer row is "All semesters".
  - A second table of evaluation scores per semester (A = workload, B = learning, C = motivation, D = feedback, plus votes), shown for rows with LEARNING_VOTES > 0, plus an all-semesters row.
- **Grade distribution modal** (:800-1210): per-grade horizontal bars with counts and percentages.
- **Prerequisite graph** (:1361-1781):
  - It is client-side and fetches **third-party JSON from raw.githubusercontent.com/Marcrulo/DTU-courses-extension** (`id_to_name.json`, `graphs.json`) on first toggle.
  - It filters nodes by level (≤0 for prerequisites, ≥0 for subsequent).
  - It shows a "large graph warning" above 50 nodes and a blurred preview when the largest layer has more than 6 nodes.
  - It credits the DTU-courses-extension.
- `assets/dist/js/gradegraph.js` is referenced at :1216 but **does not exist in the repo**, so every course page view triggers a 404.
- Fields in the df that the page never shows: TIMETABLE_PLACEMENT, INSTITUTE, TOTAL_SEMESTERS, PERCENT_PASSED, SEMESTER_PERIOD, RATING_TIER.

How the build side defines the metrics (from the FAQ, `faq.html:70-80`):
- **Rating**: the mean of three evaluation questions mapped to 1-5, averaged across semesters, then bucketed into half stars.
- **Workload**: the "time spent" question mapped to 1-5. "More" counts as overworked and "Much more" as severely overworked.

## 4. Recommender

### Model and embeddings (`recommend_utils.py`)
- The encoder is **all-MiniLM-L6-v2 exported to ONNX** (`pandas_df/model_onnx/model.onnx`, 23 MB, plus `tokenizer.json`). It runs with `onnxruntime` and the HF `tokenizers` library, not torch (:10-11, :20-22, :44-100).
  - The tokenizer pads and truncates to **256 tokens** (:84-85).
  - Inference feeds `input_ids`, `attention_mask` and `token_type_ids` (:142-149), then **mean pooling** (:109-118), then **L2 normalisation** (:155-156).
- Course document text (`combine_course_text`, :235-256) concatenates the course description, learning objectives, content, remarks, legacy recommended prerequisites, recommended prerequisites, and "mandatory prerequisites".
  - Bug: line 245 passes the `Info` object instead of `.key_df`, so **mandatory prerequisites are never included**.
  - Preprocessing (:226-233): strip HTML, lowercase, keep only `[a-zA-Z\s]` (Danish letters, digits and course numbers are removed), collapse whitespace.
  - Only courses with more than 10 characters of text are kept (:284-290). With the 256-token truncation, mostly the description and objectives end up in the vector.
- Precomputed embeddings: `recommended_course_embeddings_cache.pkl` (5.4 MB, which suggests ~1.7k × 384 float64) plus `recommended_filtered_courses_df_cache.pkl` (the df with a positional index after `reset_index`). Both are built offline by `recommender_update.py:3-10` (`fit(force_recreate=True)`).
  - On PythonAnywhere (detected by the existence of the PA path, :170-172) `fit()` runs in cache-only mode and raises if the cache is missing (:264-276).
  - Locally it creates and saves the cache if it is missing (:278-296).
- Startup (`FilterRecommendations.initialize_course_recommender`, :554-563): the model is loaded or the cache is loaded, then `fit()`.
  - Afterwards `courses_df` is the **filtered cache df**, which differs from `course_database.df`. Courses without text do not exist for the recommender or for `/search_courses`.

### `/recommend` pipeline (recommender.py:223-364 + recommend_utils.py:396-489)
1. Input is the `basket_items` list of `{type:'course'|'text', value}`. An empty list returns `success:false` with the message "Please add courses or text…".
2. **Historical-course interception** (recommender.py:238-257):
   - A `course` item whose ID is not in the current df but is a key of `cv_helper.json` (2817 courses across all archived years) is replaced by a `text` item. The text is the cv_helper context (English name, Danish title, description, content, objectives), labelled "Historical Course X".
   - Unknown IDs with no context are dropped.
   - The full 6.5 MB `cv_helper.json` is re-read for this request.
3. `recommend_hybrid` (:396-489):
   - A course item uses its precomputed vector, labelled "Course <id> - <name>".
   - A text item is preprocessed, must be longer than 2 characters, and is encoded **live with ONNX for each item, on the request thread**. Its label is "input '<first 20 chars>…'".
   - `final = 0.5*mean(course_vecs) + 0.5*mean(text_vecs)` when both kinds are present, otherwise the single centroid (:445-453). There is no re-normalisation, but cosine similarity is scale-invariant.
   - `similarity_score = cosine(final, every course)` (:455-457).
   - `similarity_breakdown`, for each candidate: the cosine to **each** input vector, as a list `{name, score}` sorted descending (:459-467).
   - It excludes the input course IDs, and excludes near-duplicates whose max cosine to any input course is ≥ 0.99, which catches renamed or cloned courses (:473-483).
   - It sorts descending and returns everything (`return_all_ranked=True`).
4. **Filters** (recommender.py:268-274; `FilterRecommendations.apply_filters` :499-528):
   - The input is `{category: [value_codes]}` using the same `json_filter_dct` as browse. Lists are OR within a category and AND across categories.
   - Unknown categories or values are skipped with a warning.
   - If no valid filter remains it returns an empty set, and the endpoint answers "Filters excluded all results."
5. **Unlocked courses** (recommender.py:276-304):
   - For each input course ID, `SUBSEQUENT_COURSES` (a comma string) is split and unioned, then the inputs are subtracted.
   - Result: `[{id, name}]` sorted by ID. It ignores filters and historical inputs.
6. **Output formatting** (recommender.py:306-353):
   - `head(n_recommendations)`, where the client always sends 9001, i.e. everything.
   - The description is HTML-stripped and truncated to 200 characters plus "...".
   - Stats are joined from `data()`: responsible, course_type, language, season, schedule, exam, signups, grade, fail, rating, votes, workload, rating_tier, workload_tier.
   - Also included: score (float), ects, institute.
   - The client shows the top 5 with "% match" = `round(score*100)` capped at 100, then a list or carousel of the rest, a similarity breakdown modal and an "unlocked courses" modal (`recommender.html:905-1000`).

### Other recommender endpoints
- `/search_courses`:
  - a case-insensitive `str.contains` over COURSE, NAME and DANISH_NAME of the filtered df, **in regex mode (the pandas default)**
  - first 20 rows in df order, with no relevance ranking
  - when `include_historical=true` and the query is exactly 5 digits found in cv_helper, it appends "(Course no longer exists)"
  - the navbar calls it without `include_historical`; the recommender calls it with `true` (`recommender.html:610`)
- `/extract_courses`:
  - regex `\b\d{5}\b` over pasted text, e.g. a studieplan.dtu.dk enrolment list (`recommender.html:1703`)
  - each code is de-duplicated and classified as current (with its name) or historical
  - **alphanumeric IDs (41B02, KU181) are never found**
- `/get_study_line_courses`:
  - loads `extended_pkl.pkl` (21 MB, lazy and cached globally, :31-41)
  - accepts any column name and returns rows where `df[col] == 1`; study lines are one-hot columns named by the raw study-line name
  - it is used to "import all courses of my study line" into the basket
- `/cv_helper`:
  - re-reads the 6.5 MB JSON **on every request**
  - returns concatenated per-course context text, which the UI shows in a modal with a copy-to-clipboard button: "Copy course data for AI purposes … provide it as context to an LLM (like ChatGPT) to help tailor your CV" (`recommender.html:1559-1660`)
- Client filter UI bugs, which are frontend but change backend results (`recommender.html:862-880`, :1739-1830):
  - the language checkboxes send `"Danish"`/`"English"`, but the filter dict keys are `dk`/`eng`. Language alone therefore returns "Filters excluded all results"; combined with other filters it is silently ignored.
  - `semester_period` reads only the first 4 checkboxes, so July and August are ignored.
  - only language, course_type, semester_period, timetable and exam_type are offered.

### Unused recommender code
- `CourseRecommender.recommend_courses`, `recommend_by_text`, `clear_cache`
- `FilterRecommendations.get_filtered_recommendations`, `get_available_filter_options`, `get_courses_by_filters`, `get_text_based_recommendations` (recommend_utils.py:328-394, 530-552, 584-595)

### Abandoned approach (old_recommender_stuff/)
The first recommender (`recommend_courses.py`) used `sentence_transformers.SentenceTransformer('all-MiniLM-L6-v2')` with PyTorch. It also imported TF-IDF, CountVectorizer, LDA and NLTK stopwords, which suggests earlier bag-of-words and topic-model experiments. It had separate JS/CSS front-end files (`recommender_core.js`, `recommender_display.js`, `recommender_search.js`). It was abandoned because the torch and sentence-transformers install took about 1.3 GB in `~/.local`, far over PythonAnywhere's disk quota of about 200 MB. `note_for_later.txt` records the decision to drop torch and run the same model through ONNX Runtime plus `tokenizers`, which is what `recommend_utils.py` does now. `recommend_questions_to_ask.md` is a user-testing script (list vs carousel, raw-text vs search input, filters, Netflix-style carousels).

## 5. Home page, browse, FAQ, quick navigation, historical courses

- **Home** (`home.html`):
  - The title is "DTU Course Analyzer {{current_year}}" (`Config.course_years='2026-2027'`), with the byline "A software project by Jonatan Rasmussen s183649".
  - A recommender advert card is present but commented out (:39-56).
  - **Quick navigation** (:70-104): buttons for Recommender tool (`/recommender`), Browse all courses (`/browse?dk=language&eng=language`) and View random course (`/random`), a lazy-loaded `assets/brand/me.png` image, and the hint "…Or scroll down to use the filter. You can also read the FAQ".
  - **Course search filter form** (:110-1190): GET to `/browse`, with the checkboxes listed in §2 and a submit button "Apply search filter".
  - A "Course sample" grid of 21 hard-coded course IDs (`context_dicts.py:111-115`). Missing IDs are dropped with a console print.
  - Footer links: Browse all and FAQ.
  - "Last updated: {{last_updated}}" (`Config.website_last_updated="30/09/2026"`).
  - A schedule icon legend modal (:1397-1490).
  - A leftover demo modal containing "Popover in a modal" (:1327-1390).
- **Browse**: see §2. It shows the "N course(s) matched your search" header, the sort buttons and dropdown, a reverse switch, the "Adjust filters" link back to `/`, summary stats, cards and "Return to top of page".
- **FAQ**: static HTML with 19 questions: who, what, how to use, why, time spent, data freshness (summer exams 2026, updated 1-2×/year), data source (scraping kurser.dtu.dk, no API), GitHub, raw data CSV, missing data, star-rating method, skull-workload method, future plans, other universities, bugs, contact, fun fact, something cool, "May I leave now?". The `faq_dict()` in `context_dicts.py:337-350` is unused placeholder text ("Bla bla").
- **Historical (discontinued) courses**:
  - They have no browse page.
  - The course page only covers current-year courses; others get the invalid-course page with a link to DTU.
  - Historical courses exist only inside the recommender: `/search_courses?include_historical=true`, `/extract_courses`, basket interception and `/cv_helper`, all through `cv_helper.json`.
  - Grade and evaluation data of a predecessor course is merged into its successor at build time through DTU's "Previous Course" field (`csv_creator.py:99-113`).
- **Random**: `/random/` redirects. `/course/xxxxx` renders a random course without changing the URL.

## 6. Data loading, memory, concurrency

### Files (all under `website/static/pandas_df/`, built offline by `csv_creator.py`)

| File | Size | When it is loaded |
|---|---|---|
| `course_df.pkl` | 9.0 MB | At import (course_database.py:19-27). Kept as a global df indexed by COURSE |
| `recommended_filtered_courses_df_cache.pkl` | 9.1 MB | At import, through the recommender. It replaces the first pkl load inside the recommender, after that pkl has been loaded once (recommend_utils.py:34, 218) |
| `recommended_course_embeddings_cache.pkl` | 5.4 MB | At import |
| `model_onnx/model.onnx` + tokenizer | 23 MB + 0.7 MB | At import (eager) |
| `json_filter_dct.json` | 240 KB | At import inside the recommender, and **again from disk on every browse/home request**: `get_filter_dct` is called twice per browse (filter + `current_args`) and once per home |
| `course_data.json` | 711 KB | **Never cached.** Each of the 16 accessor functions (`name()`, `ects()`, …) opens and parses the whole file and returns one sub-dict (`context_dicts.py:8-30,219-334`) |
| `course_numbers.json` | 31 KB | Per browse request with the default sort |
| `extended_pkl.pkl` | 21 MB | Lazily, on the first `/get_study_line_courses`. Kept in a global cache |
| `cv_helper.json` | 6.5 MB | Key set cached globally after the first use. The **full file is re-parsed per `/cv_helper` request** and per `/recommend` request that contains a historical course |

### Per-request disk and CPU cost
- `/` costs 16 full `json.load`s of `course_data.json` plus 1 of the filter dict.
- `/browse` costs 16 (`data()`) + 8 (`summary_stats`) + 1 (sort) = **about 25 parses of a 711 KB JSON (~18 MB of JSON)**, plus 2 of the filter dict. It then renders up to 1746 full cards.
- `/recommend` costs 16 `course_data.json` parses, plus one ONNX inference per text item, plus an N×k cosine matrix and Python loops over ~1.7k rows (`iterrows`, building the breakdown lists), plus a `df.copy()` of the whole df for each request (recommend_utils.py:469). The response holds about 1.7k recommendations, each with a breakdown list, because the client asks for 9001.
- `/search_courses` runs 3 regex `str.contains` scans over ~1.7k rows on every keystroke, after a 25 ms debounce.

### Memory
These are estimates; I did not measure them because pandas is not installed locally.
- Per process: about 2 copies of the ~9 MB course df in object-dtype pandas form (likely tens of MB each), the embeddings (~5 MB), the ONNX session (~100 MB order), optionally the extended df (~21 MB pickle, likely more than 100 MB in memory), plus the Flask and numpy/sklearn imports.

### Concurrency model
- There is no app-level concurrency code. Locally it is the Flask dev server (threaded by default in Flask 3). On PythonAnywhere it is uWSGI with a fixed number of workers: each worker process imports the app and **loads the model and data again for each worker**.
- Global mutable state:
  - `_recommendation_filter`, `_historical_courses_cache`, `_extended_df_cache` (recommender.py:19-21)
  - `df` and `course_set` (course_database.py:19-27)
  - These are lazy and have no locks. Racing initialisations are benign, but they duplicate the heavy load.
- If eager init fails, `get_recommender()` retries the **full init on every request** (recommender.py:59-65).
- All work is synchronous on the request thread. There is no background queue or caching layer.

## 7. SEO, meta, analytics, static assets, lazyload

- `base.html:4-9`:
  - `<meta name="description" content="Browse and analyze courses at DTU (Technical University of Denmark)">`
  - `author = "Jonatan Rasmussen, s183649"`
  - a **hard-coded `<title>DTU Course Analyzer</title>` on every page**. Child templates define `{% block title %}Home{% endblock %}`, but base.html has no title block, so per-page or per-course titles are dead.
  - No canonical, OpenGraph, structured data, robots or sitemap. The invalid-course page returns a soft 404 (200).
- Analytics: Google Analytics 4 gtag `G-53NMP6CHX6` (`base.html:11-19`), loaded on every page with no consent banner.
- Third-party front-end assets: Bootstrap **5.0.0-alpha1** CSS/JS from stackpath, Font Awesome 4.0.3, Popper 2.9.2, jQuery 3.2.1 (all CDN, no SRI) (`base.html:27-40`). The course page also fetches JSON from raw.githubusercontent.com.
- Local assets:
  - `static/assets/brand` (favicon.png, rocket.png logo, me.png, blank_photo.png)
  - `colors/*.png` (card header strips)
  - `ratings/<tier>-star.png` and `<tier>-skull.png` (tiers 0-9)
  - `schedules/<key>_Schedule.png`
  - `screenshots/` (unreferenced)
  - `static/index.js` (empty), `navbar.css` (7 lines)
- Lazyload:
  - native `loading="lazy"` on the navbar logo (`base.html:127`), the home card colour strip and teacher photos (`home.html:1210,1237,1240`), the quick-nav image (`home.html:96`) and the browse cards (recent commits a37e29e, 754e69f)
  - teacher photos come from `orbit.dtu.dk/files-asset/...?w=160&f=webp` (hotlinked)
  - an onerror/onload fallback swaps in `blank_photo.png`

## 8. Weaknesses

### Scalability
- Uncached JSON re-parsing on every request: about 25 parses of 711 KB per `/browse`, 16 per `/` and per `/recommend`, and 6.5 MB per `/cv_helper`. This is the main CPU hotspot and is easily fixed by loading once at startup.
- Browse renders every matched card on the server; the full list is about 1746 cards. Pagination is only CSS hiding, so HTML size and render time grow with the result set.
- `/recommend` is unbounded:
  - any number of text basket items, each a synchronous ONNX inference
  - no request-size limit (no `MAX_CONTENT_LENGTH`)
  - the response is about 1.7k items with nested breakdowns
  - a `df.copy()` and Python `iterrows` loops on every call
  - A few concurrent requests can saturate a PythonAnywhere worker. There is no rate limiting.
- `/search_courses` runs on every keystroke (25 ms debounce) with regex scans and no caching.
- Every uWSGI worker holds its own copy of the model, embeddings and dataframes, and initialises them eagerly at import, so cold starts are slow.
- Both pickled dataframes are loaded although the recommender then replaces its copy. The data is effectively loaded twice.

### Security
- **Regex injection / ReDoS** in `/search_courses`: `str.contains(query)` defaults to `regex=True` (recommender.py:104-108). An invalid pattern raises an error, which is caught; a catastrophic pattern burns CPU.
- **Stored-XSS trust**: about 20 scraped DTU HTML fields render with `|safe` (`course.html:57,465,477,489,506,524,556,573,585,597,609,621,633,645,657,669,681`). Any HTML or JS in kurser.dtu.dk content, or in a tampered pickle, executes. Several client templates use `innerHTML` with server data: the navbar autocomplete with `course.name` (`base.html:218-229`), recommender cards, and the basket display, which is self-XSS from user text.
- Exception messages are returned to clients (`str(e)`), always with HTTP 200.
- `SECRET_KEY` falls back to the public `'default_secret_key'` (`__init__.py:16`). The impact is limited to forging flash cookies.
- `app.run(debug=True)` in the launcher exposes the Werkzeug debugger (remote code execution) if it is ever bound to a public interface; the commented-out line 8 shows that intent.
- Raw data and model files are publicly downloadable through `/static/pandas_df/...` (pickles, ONNX, cv_helper), which costs bandwidth. Pickles are loaded from local disk, so the server is safe as long as that folder is not writable.
- No CSRF protection. That is low risk, since the only POSTs are search redirects and idempotent JSON reads.
- `/get_study_line_courses` accepts an arbitrary column name. That is harmless because the data is public.
- There is no input validation on the `basket_items` structure: a `KeyError` or `TypeError` becomes a 200 error JSON. `int(n_recommendations)` raises on non-integers.
- CDN scripts are loaded without SRI. Bootstrap 5 alpha and jQuery 3.2.1 have known CVEs. Google Analytics loads without consent, a GDPR concern for an EU audience.

### Bugs
- The search box does not upper-case input, so alphanumeric IDs fail when typed in lower case (`search.py:8`).
- `/extract_courses` regex `\d{5}` misses alphanumeric course IDs (recommender.py:152).
- Mandatory prerequisites are never embedded (`recommend_utils.py:245`, which passes the `Info` object instead of `key_df`).
- Recommender language filter value mismatch, and July/August ignored (`recommender.html:876-877,1739`).
- 4 home study-line checkboxes have no JSON key (`BENG_CE`, `BENG_ITE2`, `BSC_BI`, `BSC_MT1`); they are silently ignored.
- `sort_by=course_numbers` returns 0 courses; sort by responsible sorts by photo URL.
- Sort links are broken when `args.args` is empty, and values are not URL-encoded (`browse.html:31-75`).
- Code collisions between categories (`offschedule`) because the URL key is the value code.
- Course page: the `data.OBJECTIVES` condition is always true; real `None` values are printed as "None"; the overworked % uses LEARNING_VOTES as the denominator; `gradegraph.js` returns 404.
- The invalid-course page returns 200 instead of 404. The 404 templates are unused.
- `summary_stats` excludes zeros, which biases fail-rate and signup averages upward, and raises a `KeyError` if a course is missing from a sub-dict.
- `get_course_info` slices `COURSE_DESCRIPTION[:200]` even when it is NaN (recommend_utils.py:368).
- `recommend_hybrid` relies on `courses_df` having a 0..N-1 index that is aligned with the embeddings rows (:416-417, :483). That only holds for the cache df.
- The navbar autocomplete has a comment saying "Limit to 5" while the code uses 10. `/search_courses` has no relevance ranking (e.g. "0" matches everything).
- `home()` computes `args` but `home.html` never uses it.
- Paths are CWD-relative, so the app breaks when started from another directory.

### Dead code and leftovers
- Routes: `/test`, `/wip`, `/recommender_v1`, `/recommender_v2`, `/course_info/<id>` (no caller in the current UI).
- Templates: `index.html` (Bootstrap example), `login.html`, `login_elif_test.html`, `sign_up.html`, `recommender_v2x.html`, `recommender_v3.html`, `404_not_found.html`, `404_empty_search.html`. The `{% block title %}` blocks are dead too.
- Functions: `context_dicts.faq_dict`, `get_name_of_list`, the commented "Jonatan's passed courses" lists, the unused recommender methods listed in §4, `CourseRecommender.clear_cache` and `save_embeddings_cache` (used only offline).
- Unused imports: `browse.py:4-5` (`pd`, `flash`, `render_template`…), `views.py:8-9`, `misc.py:5` (`get_filter_dct`), and `course_database.py:17`, a stray comment.
- Empty `static/index.js`; the `screenshots/` assets are unreferenced; the home "Popover in a modal" demo modal; the commented-out recommender advert.
- `website_consts.create_website_data_dct` and `csv_columns_consts` are build-time only and live inside the `website` package. `csv_columns_consts.py:7` imports `utils` from the repo root, which couples the web package to the scraper.
