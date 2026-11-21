# DTU Course Analyzer — Data Pipeline Behaviour Extraction

Repo: /Users/puddu/Documents/GitHub/dtu-course-browser (read-only analysis). All `file:line` refs are relative to repo root.

---

## 0. Big picture

- Pipeline = 4 manual steps run in-process by `main.py` (main.py:12-22):
  1. `AllInOneScraper.run_all_scrape_scripts()` — scrape everything to `scraped_data/` ("THIS TAKES HOURS. Async url scraping is NOT IMPLEMENTED", main.py:12).
  2. `CsvCreator.create_csv()` — clean/format/merge into `website/static/csv_files/*.csv` + `website/static/pandas_df/*.pkl|*.json` (main.py:16).
  3. `run_recommender_embeddings_update()` — rebuild sentence embeddings (main.py:19).
  4. `website_launch_main()` — Flask dev server `app.run(debug=True)` (website_launch.py:4-7).
- Pinned deps, Python 3.11.7: Flask 3.0.3, pandas 2.2.2, numpy 1.26.3, beautifulsoup4 4.12.3, lxml 5.2.2, requests 2.31.0, scikit-learn 1.5.2, onnxruntime 1.24.1, tokenizers 0.22.1 (requirements.txt:1-11). Exact versions required because committed .pkl files are pandas-version sensitive (main.py:7-9).
- No scheduler, no CLI args, no env vars. Everything is driven by `website/global_constants/config.py` class attributes. Each scraper module is also runnable standalone via `if __name__ == "__main__"` with a hardcoded "quick test" (e.g. scrape_info.py:275-276, scrape_grades.py:148-149).
- Hosting target: PythonAnywhere; paths prefixed `dtu_course_project/` for prod (file_name_consts.py:25,32,36,43).

---

## 1. Data sources scraped

General HTTP characteristics (all scrapers)
- Library: `requests` (sync, sequential, single thread); `pd.read_html(url)` (urllib under the hood) for grades; `urllib.request.urlopen` for eval pages.
- No auth anywhere; all public pages. Header `Accept-Language: en` on most requests (scrape_archive.py:57,86; scrape_info.py:82,84,104-105; scrape_evaluations.py:74; scrape_orbit.py:236). No User-Agent set.
- Timeout 10 s on `requests` calls (utils.py:39; scrape_archive.py:57; scrape_info.py:82; scrape_evaluations.py:74; scrape_orbit.py:236; scrape_ppics.py:70). NO timeout for `pd.read_html(url)` (scrape_grades.py:60) or `urllib.request.urlopen` (scrape_evaluations.py:46).
- Retry pattern: recursive "try once more" via `is_timeout` flag = max 2 attempts (e.g. scrape_archive.py:72-74). Info & eval scrapers add a 60 s `time.sleep` then another 2-attempt round if page is empty (scrape_info.py:42-47, 50-56; scrape_evaluations.py:35-40) → up to 4 attempts. Orbit sleeps 5 s before its retry (scrape_orbit.py:239-241).
- Rate limiting: none for kurser/karakterer/evaluering. Orbit: 0.5 s sleep between name variants and between teachers (scrape_orbit.py:166,177). Image downloader: 0.2 s per image (scrape_ppics.py:79).
- Parallelism: none ("Async url scraping is NOT IMPLEMENTED", main.py:12).
- Logging: `Utils.logger` → `logs/<name>.log` via `logging.basicConfig` (utils.py:251-288); log names `scrape_logs`, `format_logs` (file_name_consts.py:20-21). `logs/` is gitignored.

### 1.1 Course archive (course list per academic year) — `scrape_archive.py`
- URL A: `https://kurser.dtu.dk/archive/{academic_year}` e.g. `2024-2025` (scrape_archive.py:56). Parses: split page at last `</strong>`, BeautifulSoup `table.table`, every `<a>` text = 2-digit "course code" prefix (`01`, `02`, `10`, …) (scrape_archive.py:61-70). `raise_for_status()` used.
- URL B: `https://kurser.dtu.dk/archive/{academic_year}/coursecode/{code}` (scrape_archive.py:85). Parses `table.table` rows (skip header), rows with exactly 2 `<td>` → `{course_number: course_name}` (scrape_archive.py:90-99). No `raise_for_status()`.
- Academic years derived from configured semesters: E→same year start, F→previous year start (utils.py:77-90); unique sorted (utils.py:66-75).
- Output: nested dict `{ "2024-2025": {"01005": "Advanced Engineering Mathematics 1", ...}, ...}` → `scraped_data/archived_courses.json` (scrape_archive.py:24-38; scrape_all.py:36-38). Current file: 7 years, 1704-1760 courses/year.
- On failure after retry: logs error and returns `None` (implicit) → downstream `len(None)` crash (scrape_archive.py:72-76, 101-105, 36).
- English names here are the authoritative `NAME` column in the final CSV (csv_creator.py:78,149).

### 1.2 Course info (DTU Course Base) — `scrape_info.py`
- Per course per academic year:
  - Info page `https://kurser.dtu.dk/course/{academic_year}/{course}` (scrape_info.py:78). Cookie-priming: new `requests.Session()` per call, GET twice (first returns a JS reload page + cookie, second returns real content) (scrape_info.py:79-86).
  - Responsibles page `https://kurser.dtu.dk/course/{course}/info` (scrape_info.py:101) — NOT year-specific (always current). Same double-GET.
- Extraction from info page:
  - "Primary df": `pd.read_html(page)`; expects exactly 3 tables, concatenates them as 2-col key/value → dict (scrape_info.py:114-141). Keys are DTU's row labels (see §3.3). If ≠3 tables → warning, nothing extracted. If no tables (`ValueError`) → refetch once and retry parse (scrape_info.py:122-128).
  - String-split extractions (`_parse_from_page_source`, which first normalises whitespace and converts `<ul><li>`/`</li><li>` into `<br />- `) (scrape_info.py:240-256):
    - Study lines: between `var lines = ` and `;` (JS array literal string) → key `Study lines` (scrape_info.py:151-158).
    - `Last updated</div>` … `<` → `Last updated` (scrape_info.py:161-168).
    - `General course objectives</div>` … `<div class=` (scrape_info.py:171-178).
    - `Learning objectives</div>` … `<div class=` (scrape_info.py:181-188).
    - `>Content</div>` … `<div class=` (scrape_info.py:191-198).
    - `>Remarks</div>` … `<div class=`; default `"None"` (scrape_info.py:201-206).
    - Highlighted (red) message: `<div class="row"><div class="col-xs-12">` … `</`; default `"None"` stored under custom key `Highlighted Message` (scrape_info.py:209-214).
- Extraction from responsibles page: split on `<div class="row" style="margin:20px">`; blocks 1..5 → name between `<b>`/`</b>`, pic between `<img src="` / `"` (scrape_info.py:218-271). Up to 5 people: main + 4 co-responsibles; missing → `"NO_DATA"` (dtu_consts.py:57). Pic URLs are `https://www.dtubasen.dtu.dk/showimage.aspx?id=NNN`. Names contain HTML entities (e.g. `M&#248;ller`).
- Only columns listed in `InfoConsts.scrape_info_column_names` are kept (scrape_info.py:32; info_consts.py:825; utils.py:189-199).
- Output per year: `scraped_data/scraped_info_{YYYY}_{YYYY}.pkl` + `.csv` (scrape_all.py:53-58; utils.py:211-229). Iterates the course list for THAT year from the archive (scrape_all.py:56).

### 1.3 Grades — `scrape_grades.py`
- URL: `https://karakterer.dtu.dk/Histogram/1/{course}/{exam_period}` where exam_period = `Summer-20YY` for `FYY`, `Winter-20YY` for `EYY` (scrape_grades.py:33; utils.py:92-105).
- `pd.read_html(url, header=0)`: ≥3 tables → table[2], index `Karakter`, first value column → `{grade_label: count}`; keys stringified + `.capitalize()` (scrape_grades.py:60-72). Exactly 2 tables → grades hidden (≤3 students) → `{}` (scrape_grades.py:73-76). Fewer → warning `{}`.
- Fix-up: int-parsed labels `"0"`→`"00"`, `"2"`→`"02"` (scrape_grades.py:95-99).
- Label mapping (scrape_grades.py:107-119): `12,10,7,4,02,00,-3` → `GRADE_*`; `Bestået`→`PASSED`; `Ikke bestået`→`FAILED`; `Ej mødt`→`ABSENT`; `Syg`/`Ej mødt`/`Ikke Godkendt` summed into `ABSENT` (scrape_grades.py:129-130, 142-143); `Godkendt` always expected 0, warns if not (scrape_grades.py:133-136). Missing label → `"No data"` string (scrape_grades.py:123-125).
- Course list = union of ALL archived course numbers across all years (scrape_all.py:47; utils.py:128-137) → ~2817 rows × 14 semesters ≈ 39k requests.
- Retries on `HTTPError`, `IndexError`, `RequestException` (scrape_grades.py:83-88). 404 → `{}`.
- Output: `scraped_data/scraped_grades.pkl` + `.csv`, wide: `COURSE` + `{SEM}_{GRADE}` for 14 sems × 10 grades = 141 cols (scrape_grades.py:28,41-43; utils.py:44-64).

### 1.4 Course evaluations — `scrape_evaluations.py`
- Search: `POST https://evaluering.dtu.dk/CourseSearch` form `{courseNumber, termUid:"", SearchButton:"Søg"}` (scrape_evaluations.py:71-76).
- Parse search result (scrape_evaluations.py:83-110):
  - Split on `href="/kursus/`; chars `[6:12]` must be 6 digits = evaluation id → URL `https://evaluering.dtu.dk/kursus/{course}/{id}`; term from `<div class="Term">` e.g. `F 23`/`E-23` → normalised `F23` (requires `term[0]∈{F,E}` and `term[2:4]` digits).
  - `"Ingen kurser matchede s"` → no evaluations.
  - `"Der er 6 undervisningsperioder:"` → server redirected straight to the single evaluation; term parsed from the page header (supports `F19/E19`, `Jan 19`→E18, `Jun/Jul/Aug 19`→F19, `Winter 19`→E19, `Summer 19`→F19; else `X00`) and URL marked `""` (scrape_evaluations.py:103-106, 134-159).
- Eval page: `urllib.request.urlopen(url).read()` (scrape_evaluations.py:46) → BeautifulSoup(lxml) text of `id="Results"`, split into non-empty lines (scrape_evaluations.py:115-131).
- Extraction (scrape_evaluations.py:162-246):
  - Format detection: line[1] == `Jeg synes, at…` ⇒ post-Sep-2019 questionnaire.
  - 4 questions (post-2019, Danish): learning `jeg har lært meget i dette kursus.`; motivation `undervisningsaktiviteterne motiverer mig til at arbejde med stoffet.`; feedback `jeg i løbet af kurset har haft mulighed for at få feedback …`; workload `Jeg mener, at den tid jeg har brugt på kurset er`. Pre-2019 has different wording and "material is good" instead of motivation (scrape_evaluations.py:183-186).
  - 5 answer counts read at fixed line offsets after the question: post-2019 `[2,6,10,14,18]`, response total at offset 21; pre-2019 `[15,11,8,5,2]` (reversed so that 1=disagree) and workload `[2,5,8,11,15]`, total at 18 (scrape_evaluations.py:204-213).
  - Orientation: index 0 = "Helt uenig" (strongly disagree) / "Meget mindre" (much less time) … index 4 = strongly agree / much more time.
  - Validation: next line must equal expected first answer label, else whole semester discarded (scrape_evaluations.py:217-225); sum of counts must equal "N besvarelser" (±2 tolerated with warning; course `01005` exempt) else discarded (scrape_evaluations.py:227-240).
  - Result per semester: `{SEM}_LEARNING|MOTIVATION|FEEDBACK|WORKLOAD = [c1,c2,c3,c4,c5]` (Python list stored in cell).
- Course list = all archived courses across all years (scrape_all.py:41-44).
- Output: `scraped_data/scraped_evals.pkl` + `.csv`, 1 + 4×14 = 57 cols; missing = None/empty (scrape_evaluations.py:30,56-58).

### 1.5 Orbit (DTU research portal) profile pictures — `scrape_orbit.py`
- Input: `scraped_data/scraped_info_{year}.csv` (must already exist) (scrape_orbit.py:68-75). Collects unique responsible names from the 5 name columns of courses in that year (HTML-unescaped, excluding `NO_DATA`) (scrape_orbit.py:77-96).
- Expected email aliases per teacher: regex `([a-zA-Z0-9._-]+)@` over `Responsible` + `Course co-responsible` raw text of their courses (scrape_orbit.py:99-121).
- URL: `https://orbit.dtu.dk/en/persons/{slug}/` (scrape_orbit.py:235), slug = name lowercased with spaces→`-`, URL-quoted (scrape_orbit.py:141,191).
- Matching strategy per teacher (scrape_orbit.py:127-174):
  - Hardcoded `MISSING_LIST` (5 names) → `""` (scrape_orbit.py:22-29,131-134).
  - Hardcoded `MANUAL_MAPPING` (14 name→slug overrides) (scrape_orbit.py:38-53).
  - Otherwise try: full name, then fallbacks: drop last name, drop first name, drop each middle name (scrape_orbit.py:209-229); each also with `-2` suffix.
  - 404 detection: page contains `Page not found` or `The page does not exist` (scrape_orbit.py:245-247).
  - Email verification: alias between `class="email">` and `<` must be in expected aliases (if both known); skipped for `SKIP_EMAIL_CHECK` (2 names) (scrape_orbit.py:32-35,197-202,249-258).
  - Image: `files-asset/{id}` → `https://orbit.dtu.dk/files-asset/{id}.jpg?w=160&f=webp`; `no-portrait-` → default; fallback default `https://orbit.dtu.dk/assets/no-portrait-473c6d005990baa1f418d9c668dcd4ec.png` (scrape_orbit.py:19,261-280).
- One shared `requests.Session` (scrape_orbit.py:125). No status-code check (relies on text).
- Output: `scraped_data/scraped_orbit_{YYYY}_{YYYY}.json` = `{teacher_name: image_url}` (scrape_all.py:61-66). Current year ≈1158 teachers.

### 1.6 Profile picture downloader — `scrape_ppics.py` (NOT wired into main pipeline)
- Reads `scraped_orbit_{website_current_year}.json`, downloads each URL with streaming GET into `scraped_images/{sanitized_name}.jpg|.png` (`.png` if URL contains `no-portrait`) (scrape_ppics.py:22-76). Name sanitized: strip `\/*?:"<>|`, spaces→`_` (scrape_ppics.py:58). Skip existing unless `REDOWNLOAD_EXISTING_IMAGES=True` (scrape_ppics.py:19,62-65). 0.2 s delay. Prints summary counts. `scraped_images/` gitignored. Output not consumed by pipeline (FileNameConsts.path_of_profile_pics `website/static/assets/profile_pics/` exists but unused in pipeline).

### 1.7 Study lines
- Not a separate source: extracted from the `var lines = [...]` JS literal on each course info page (scrape_info.py:151-155), then normalised in `format_study_lines.py` (§3.5). Only populated for the most recent year on DTU's site (csv_creator.py:173-174).
- `studieordning_link.txt` contains only `https://sdb.dtu.dk/2015` — referenced nowhere in code (dead).

### 1.8 Access logs (offline, not scraped) — see §5.

---

## 2. Orchestration

### 2.1 Config knobs (`website/global_constants/config.py`)
- `course_semesters = ['E20','F21','E21','F22','E22','F23','E23','F24','E24','F25','E25','F26','E26','F27']` (config.py:4) — drives semester columns, exam periods, academic years (2020-2021 … 2026-2027).
- `course_years = '2026-2027'` (config.py:5) — "current" year: which archive slice / info pkl / orbit json feeds the website CSV. Must equal most recent year on DTU site (scrape_all.py:117-118).
- `data_null_value = None` (config.py:6); `data_decimal_precision = 2` (averages) (config.py:7); `data_percental_precision = 1` (percentages) (config.py:8).
- `website_current_year = course_years`; `website_last_updated = "30/09/2026"` (manually edited string shown on site) (config.py:10-11; website/context_dicts.py:143).
- `use_orbit_profile_pictures = True` (config.py:13) — replace dtubasen pics with Orbit URLs in CSV.
- Feature flags `feature_flag_scrape_archive|evals|grades|info|orbit` all True (config.py:16-20).
- FileNameConsts: folder names, file stems, paths (file_name_consts.py:5-46). `create_file_for_specific_year = ""` suffix knob (file_name_consts.py:28).

### 2.2 Step order in `run_all_scrape_scripts` (scrape_all.py:30-66)
1. Print banner with semesters & year (scrape_all.py:33,110-119).
2. Archive → `archived_courses.json` (all years).
3. Evals for all archived course numbers (all years, union) × all semesters → `scraped_evals.*`.
4. Grades for same list → `scraped_grades.*`.
5. Info: loop academic years; course list for that year → `scraped_info_{Y1}_{Y2}.*`.
6. Orbit: loop academic years → `scraped_orbit_{Y1}_{Y2}.json` (depends on step 5 csv).
- Steps 3-6 read the archive JSON from disk (so step 2 can be skipped via flag to reuse an old archive).

### 2.3 Alternative mode `run_all_scrape_scripts_one_semester_at_a_time` (scrape_all.py:68-106)
- Per semester: archive per academic year (only when year changes) → `archived_courses_{Y1}_{Y2}.json`; grades → `scraped_grades_{SEM}.*`; evals → `scraped_evals_{SEM}.*`; info per year. No orbit.
- Explicitly incompatible with CsvCreator file names (scrape_all.py:70-71). Yet the per-year `archived_courses_{Y1}_{Y2}.json` it produces IS consumed by the CV helper (csv_creator.py:276). Repo contains these per-semester files (scraped_data/scraped_grades_F24.csv etc.).
- Bug: `course_numbers` is only set inside the archive branch; if archive flag off → NameError (scrape_all.py:83-91,96).

### 2.4 `CsvCreator.create_csv` (csv_creator.py:31-71) — 4 parts
1. Website CSV with fixed column set `PREMADE_COLUMNS` → `website/static/csv_files/course_df.csv` + `website/static/pandas_df/course_df.pkl` (csv_creator.py:33-42).
2. Extended CSV with ALL generated keys → `website/static/csv_files/extended_csv.csv` + `website/static/pandas_df/extended_pkl.pkl` (csv_creator.py:44-54). Recomputes the whole merge a second time.
3. Website JSONs (in `website/static/pandas_df/`): `course_numbers.json`, `course_data.json`, `json_filter_dct.json` (csv_creator.py:56-62; website_consts.py:100-156; csv_creator.py:214-255).
4. `cv_helper.json` (LLM context per course, all years) (csv_creator.py:65-67, 257-320).

### 2.5 `_build_combined_df` (csv_creator.py:74-175)
- Inputs: archive JSON (all + current-year numbers & names), `scraped_grades.pkl`, `scraped_evals.pkl`, `scraped_info_{course_years}.pkl`, `scraped_orbit_{course_years}.json` (optional; warns if missing) (csv_creator.py:76-92).
- Rows = current-year courses only (Config.course_years) (csv_creator.py:77,99). Today 1746 rows.
- Pass 1 — predecessor back-fill: for each course whose `PREVIOUS_COURSE` lists course numbers (parsed by splitting on ` and `, `/`, `,`; parts with `-` skipped; digits or alnum-with-digit kept (format_info.py:284-302)), for every semester column in grade_df and eval_df that is empty/None/"No data" in the current course, copy the predecessor's non-empty value (csv_creator.py:99-117). Mutates the loaded DataFrames in memory.
- Pass 2 — per course: `GradeFormatter.format_grades`, `EvalFormatter.format_evaluations`, `InfoFormatter.format_info`, merge (`{**grades, **evals, **info}`) (csv_creator.py:119-142).
- Orbit injection: for 5 responsible slots, `*_PIC = orbit_dict[name.strip()]` if non-empty and not `"None"`, else `"NO_DATA"` (csv_creator.py:124-141). Overrides the dtubasen pic.
- Columns: COURSE, NAME, then PREMADE_COLUMNS (or, extended mode, keys of the FIRST course's dict) (csv_creator.py:143-147). Missing keys → None (csv_creator.py:150-154).
- Post: teacher→courses columns (`create_teacher_course_lst`) and `SUBSEQUENT_COURSES` (csv_creator.py:163-166).
- Sanity check: warns for any non-semester column whose values are all identical (special hint for STUDY_LINES) (csv_creator.py:168-174).
- Idempotency: overwrite-only; deterministic given same inputs. No incremental/delta logic; no checkpointing in scrapers (a crash mid-scrape loses all progress for that file since save happens only at end — scrape_grades.py:41-43, scrape_evaluations.py:56-58, scrape_info.py:63-65, scrape_orbit.py:179-180).

### 2.6 Runtime / volume estimate
- Evals: ~2.8k POSTs + up to 14 GETs each. Grades: ~2.8k × 14 ≈ 39k page loads (no timeout). Info: 7 years × ~1.74k × 4 GETs (two double-GETs) ≈ 49k. Orbit: 7 years × ~1.1k teachers × (1..2×N variants) with 0.5 s sleeps. → "hours" (main.py:12). Formatting: O(courses × columns) with row-by-row `pd.concat` (quadratic-ish) (csv_creator.py:155-159; utils.py:197-198).

### 2.7 Files produced (summary)
| File | Format | Producer |
|---|---|---|
| scraped_data/archived_courses.json | JSON {year:{num:name}} | scrape_archive |
| scraped_data/archived_courses_{Y1}_{Y2}.json | JSON (single year) | alt mode only |
| scraped_data/scraped_evals.{pkl,csv} | wide, list-valued cells | scrape_evaluations |
| scraped_data/scraped_grades.{pkl,csv} | wide, int or "No data" | scrape_grades |
| scraped_data/scraped_{evals,grades}_{SEM}.{pkl,csv} | per semester | alt mode only |
| scraped_data/scraped_info_{Y1}_{Y2}.{pkl,csv} | 1 row/course, DTU label columns | scrape_info |
| scraped_data/scraped_orbit_{Y1}_{Y2}.json | {name:url} | scrape_orbit |
| scraped_images/*.jpg/png | images | scrape_ppics (manual) |
| website/static/csv_files/course_df.csv, pandas_df/course_df.pkl | 1746×288 | CsvCreator part 1 |
| website/static/csv_files/extended_csv.csv, pandas_df/extended_pkl.pkl | 1746×1130 | CsvCreator part 2 |
| website/static/pandas_df/course_numbers.json | {num:num} | WebsiteConsts |
| website/static/pandas_df/course_data.json | {dict_name:{num:value}} sorted | CsvCreator part 3 |
| website/static/pandas_df/json_filter_dct.json | {filter:{value:[nums]}} | CsvCreator part 3 |
| website/static/pandas_df/cv_helper.json | {num: context string} | CsvCreator part 4 |
| website/static/pandas_df/recommended_course_embeddings_cache.pkl | pickled np.ndarray (N×384 float64) | recommender |
| website/static/pandas_df/recommended_filtered_courses_df_cache.pkl | pickled DataFrame | recommender |
| logs/scrape_logs.log, logs/format_logs.log | text | Utils.logger |
| website/static/csv_files/years_2020-2027.zip | manual archive | not produced by code |

---

## 3. Data model

Conventions: `{SEM}` ∈ Config.course_semesters (E=autumn/efterår, F=spring/forår, 2-digit year). Null sentinels vary: `None`, `"No data"` (grades & evals), `"No data available"` (info not yet assigned), `"Unknown data value"`, `"NO_DATA"` (responsibles), string `"None"` (remarks/highlight/str(None)), `"–"`, `"P/F"` (website JSON).

### 3.1 Raw grades (scraped_grades) — per course
- `COURSE` + `{SEM}_{G}` for G in `GRADE_12, GRADE_10, GRADE_7, GRADE_4, GRADE_02, GRADE_00, GRADE_MINUS_3, PASSED, FAILED, ABSENT` (grade_consts.py:23-24). Value: int count or `"No data"`.

### 3.2 Formatted grades (format_grades.py:47-198) — per course
- Per semester (non-null counts summed): `{SEM}_{G}` counts (0 when missing) (format_grades.py:58-73).
- Per semester & overall stats (`_create_statistics_dict`, format_grades.py:147-161):
  - `TOTAL_STUDENTS = Σ all 10 categories` (incl. pass/fail/absent).
  - `AVERAGE_GRADE` (format_grades.py:87-114): numeric = 12,10,7,4,02,00,-3 with weights `[12,10,7,4,2,0,-3]`; if total any-grade count == 0 → `"No data"`; elif `numeric_count*10 <= PASSED+FAILED` → `"Pass/Fail"`; else `round(Σw·n / numeric_count, 2)`.
  - `PERCENT_PASSED/FAILED/ABSENT` (format_grades.py:116-145): passed = 12+10+7+4+02+PASSED; failed = 00+(-3)+FAILED; absent = ABSENT; total = sum; each `round(100*x/total, 1)`; `"No data"` if total 0.
- Overall (no prefix): same stats + summed counts across all semesters (format_grades.py:75-78).
- `STUDENTS_PER_SEMESTER` & `TOTAL_SEMESTERS` (`_number_of_semesters`, format_grades.py:163-198): over semesters with TOTAL_STUDENTS ≠ 0: `avg_students = mean(students)`, `avg_not_passed = mean(1 - PERCENT_PASSED/100)`; a semester is "real" (not re-exam-only) if `students > avg_students * (0.05 + avg_not_passed) * 1.5`; TOTAL_SEMESTERS = count of real; STUDENTS_PER_SEMESTER = `int(round(mean(real students)))`; both 0 if none.

### 3.3 Raw info (scraped_info_{year}) — columns = DTU labels (dtu_consts.py:8-59, info_consts.py:776-825)
`COURSE, Danish title, Point( ECTS ), Language of instruction, Evaluation, Location, Course type, Type of assessment, Aid, Duration of Course, Schedule, Study lines, Recommended Academic prerequisites, Academic prerequisites, Mandatory Prerequisites, General course objectives, Learning objectives, Content, Remarks, Highlighted Message, Last updated, Scope and form, Exam duration, Home page, Previous Course, Registration Sign up, Not applicable together with, Green challenge participation, Department, Department involved, External Institution, Date of examination, Responsible, Course co-responsible, Institute (always empty), Main responsible name, Main responsible pic, Co responsible name 1..4, Co responsible pic 1..4` (45 cols).

### 3.4 Formatted info (format_info.py:28-267 driven by InfoConsts.info_to_format)
Generic algorithm `_look_for_info` (format_info.py:129-267) per Info definition (key_raw, key_df, key_url, values_raw, values_df, values_url, add_raw):
- Init `key_df = "No data available"`, every `values_df` one-hot = 0.
- If `add_raw`: `{key_df}_RAW = str(raw)`; for `Type of assessment` the raw string is normalised (`Oral examination and`→`Oral exam and`, etc.) and `<br />` inserted after each known exam type phrase (format_info.py:161-185).
- Matching (`_is_start_of_string_equal`, format_info.py:74-127): default = raw starts with candidate value (prefix match). `Duration of Course`: exact equality only. `Schedule`: expands combined slots (`Spring F1 (Mon 8-12, Thurs 13-17)` → `F1A … and F1B …`, etc., F7/E7 prefixing), splits on double space / ` and `, exact match per token. `Type of assessment`: prefix after normalisation.
- Each matched value → one-hot `values_df[i] = 1` and appended to a label list; label list is de-duplicated and joined by `<br />` (STUDY_LINES, TIME_OF_WEEK) or ` + ` (others) into `key_df`; numeric strings converted to int/float (format_info.py:249-266).
- No value matched: STUDY_LINES → `"Not linked to any study lines"` + `NO_STUDYLINE=1`; CAMPUS → `CAMPUS_OTHER=1`; TIME_OF_WEEK/SEMESTER_PERIOD → `"Unspecified"`; others → `"Unknown data value"` (format_info.py:220-231).
- Free-text fields (values empty): `key_df = str(raw)` (→ `"None"` when null); responsible names HTML-unescaped (format_info.py:213-216).
- ECTS fix: if int(ECTS) > 21 → divide by 10 (e.g. `75`→`7.5`) (format_info.py:49-72). Likely caused by `pd.read_html` default `thousands=','` turning `7,5` into 75.
- INSTITUTE derived from first two chars of course number (not from scraped Department) via table; unknown → `"Partner University"` (format_info.py:233-242, 269-281).

Field catalogue (key_df / source / allowed values → one-hot columns / URL short-names) (info_consts.py):
- `DANISH_NAME` ← Danish title (free text) (info_consts.py:56-66).
- `ECTS_POINTS` ← Point( ECTS ) (info_consts.py:68-78).
- `LANGUAGE` ← Language of instruction; Danish/English → cols `Danish`,`English`; url `dk`,`eng` (info_consts.py:80-90).
- `GRADE_TYPE` ← Evaluation; `7 step scale…`→`SEVEN_STEP_SCALE`, `pass / not pass…`→`PASS_OR_FAIL`; url `sevenstep`,`passfail` (info_consts.py:92-105; dtu_consts.py:83-86).
- `EXAMINER` ← Evaluation; internal/external → `EXAMINER_INTERNAL|EXTERNAL`; url `internal`,`external` (info_consts.py:107-120).
- `CAMPUS` ← Location; `Campus Lyngby|Campus Ballerup` → `CAMPUS_LYNGBY|CAMPUS_BALLERUP`, else `CAMPUS_OTHER`; `CAMPUS_RAW`; url `lyngby,ballerup,elsewhere` (info_consts.py:122-132).
- `COURSE_TYPE` ← Course type; `BSc,MSc,BScMSc,BEng,Ph.D.,Deltidsdiplom,Parttime master` → `BSc,MSc,MSc,BEng,Ph.D.,BEng,MSc` (e.g. "BSc + MSc"); url `bsc,msc,beng,phd` (info_consts.py:134-144; dtu_consts.py:63).
- `EXAM_TYPE` ← Type of assessment; 14 DTU values → `Written exam|Oral exam|Report hand-in`; `EXAM_TYPE_RAW`; url `written,oral,reporthandin` (info_consts.py:146-169; dtu_consts.py:69-82).
- `ASSIGNMENTS` ← Type of assessment → `No mandatory assignments|Mandatory reports|Mandatory exercises|Mandatory experiments`; url `noassign,reports,exercises,experiments` (info_consts.py:171-194).
- `EXAM_AID` ← Aid; `All Aid|Written works of reference are permitted|No Aid` → `All aid|No computers|No aid`; `EXAM_AID_RAW`; url `digitalaid,analogaid,noaid` (info_consts.py:196-206).
- `COURSE_DURATION` ← Duration of Course (exact); → `DURATION_13_WEEKS|3_WEEKS|16_WEEKS|26_WEEKS|UNIQUE`; url `13week,3week,16week,26week,xweek` (info_consts.py:208-226; dtu_consts.py:87-95).
- `TIME_OF_WEEK` ← Schedule; 28 DTU slots → labels like `F1A (Spring, Mon 8-12)`, `E7 (Autumn, Tues 18-22)`, `3-week January`, `Special schedule`; url `F1A…E7, offschedule, W3january…` (info_consts.py:228-273; dtu_consts.py:96-123).
- `TIMETABLE_PLACEMENT` ← Schedule; season-agnostic slot labels `1A Mon 8-12`…`7 Tues 18-22`, `3-week`; url `Mon 8-12`… (info_consts.py:275-304).
- `SEMESTER_PERIOD` ← Schedule; `Spring|Autumn|January|June|July|August` (e.g. "Spring + Autumn"); url `spring,autumn,jan,jun,jul,aug` (info_consts.py:306-320).
- `STUDY_LINES` ← Study lines; 74 canonical names (BEng/BSc/MSc, info_consts.py:327-401) + `NO_STUDYLINE`; `STUDY_LINES_RAW`; url = acronyms built from uppercase letters with `BENG_/BSC_/MSC_` prefixes, duplicates numbered (info_consts.py:403-427).
- Free text (no one-hot): `LEGACY_RECOMMENDED_PREREQUISITES` (Recommended Academic prerequisites), `RECOMMENDED_PREREQUISITES` (Academic prerequisites), `MANDATORY_PREREQUISITES`, `COURSE_DESCRIPTION` (General course objectives), `LEARNING_OBJECTIVES`, `COURSE_CONTENT` (Content), `REMARKS`, `HIGHLIGHTED_MESSAGE`, `LAST_UPDATED` (e.g. "04. maj, 2026"), `SCOPE_AND_FORM`, `EXAM_DURATION`, `HOME_PAGE`, `PREVIOUS_COURSE`, `SIGN_UP_LOCATION` (Registration Sign up), `COURSE_POINT_LOCK` (Not applicable together with, e.g. "02323.02403.02593.62670"), `GREEN_CHALLENGE`, `DEPARTMENT_RESPONSIBLE` (Department), `DEPARTMENTS_INVOLVED`, `EXTERNAL_INSTITUTES`, `EXAM_DATE` (Date of examination), `COURSE_MAIN_RESPONSIBLE_CONTACT` (Responsible, incl. email/building), `COURSE_CO_RESPONSIBLE_CONTACT` (info_consts.py:432-693).
- `INSTITUTE` (derived): prefixes `01,02,10,56,12,22,23,CB,25,26,27,29,28,88,30,31,34,41,42,38,46,63,47,62,MA,LY,KU` → 19 institute names (one-hot) (info_consts.py:695-737).
- Responsibles: `MAIN_RESPONSIBLE_NAME/PIC`, `CO_RESPONSIBLE_{1..4}_NAME/PIC` (info_consts.py:754-774).

### 3.5 Study-line normalisation (format_study_lines.py:30-207)
- Parse JS literal: split on `'`, take odd segments (format_study_lines.py:37-42).
- ~100 ordered string replacements: NBSP→space, `&`→`and`, typo fixes, category prefixes (`Polytechnical foundation (BSc),`, `Programme specific course (MSc)`, `Technological specialization course…`, `General competence course…`, `Mandatory course, Bachelor of Engineering` etc.) → `BSc,`/`MSc,`/`BEng,`; generic `Basic Natural Science Courses|Technology Core Courses|Projects and general subjects` → `BSc, Design of Sustainable Energy Systems`; renames to official names per year (2024/25, 2025/26, 2026/27 blocks) (format_study_lines.py:46-183).
- Drop empty / bare `BSc|MSc|BEng`; result dict `{study_line: [course_numbers]}`; sorted keys (format_study_lines.py:186-207).
- Unknown names trigger warnings telling the maintainer to edit `info_consts.py` or this file (format_info.py:204-210).

### 3.6 Teachers (format_study_lines.py:209-269)
- `teachers_dct[name] = "num1<br />num2…"` over all 5 responsible slots (excluding `NO_DATA`).
- Adds `MAIN_RESPONSIBLE_COURSES`, `CO_RESPONSIBLE_{1..4}_COURSES` = that teacher's course list string, or `NO_DATA`.

### 3.7 Subsequent courses (csv_creator.py:177-203)
- Regex `\b\d{5}\b` over RECOMMENDED + LEGACY_RECOMMENDED + MANDATORY prereq text; if course A mentions B (B in current-year set) → A ∈ subsequents(B). `SUBSEQUENT_COURSES` = sorted, `", "`-joined, else `"No other course requires this as a prerequisite"`.

### 3.8 Evaluations (format_evaluations.py:45-176)
- Types: WORKLOAD, LEARNING, MOTIVATION, FEEDBACK, and derived RATING (format_evaluations.py:26).
- Per semester and overall, per type: star counts `{prefix}{TYPE}_{1..5}_STAR`; RATING counts = element-wise SUM of LEARNING + MOTIVATION + FEEDBACK counts (WORKLOAD excluded) (format_evaluations.py:66-73).
- Stats per (scope,type) (format_evaluations.py:93-176):
  - `VOTES = Σ counts`.
  - `AVERAGE_SCORE = round(Σ k·count_k / votes, 2)`, k=1..5; `"No data"` if 0 votes. (A workload exaggeration transform exists but is disabled, format_evaluations.py:120-131.) Workload scale: 1 = much less time than expected, 3 = as expected, 5 = much more.
  - `UPVOTE_RATIO = round((count_4 + count_5)/votes, 1)` (0–1), `"No data"` if 0.
  - `TIER` (0–9): 0 if no data; thresholds `<1.25→1, <1.75→2, <2.25→3, <2.75→4, <3.25→5, <3.75→6, <4.25→7, <4.75→8, else 9`.
- Overall stats are computed over pooled counts across all semesters (not the mean of semester means) (format_evaluations.py:81-86).

### 3.9 Final website CSV `course_df.csv` (1746 rows × 288 cols) — column order (csv_columns_consts.py:21-56)
- `COURSE, NAME`, then:
- Basic: `DANISH_NAME, LANGUAGE, ECTS_POINTS, COURSE_TYPE, STUDENTS_PER_SEMESTER, SEMESTER_PERIOD, EXAM_TYPE, ASSIGNMENTS, TIME_OF_WEEK, LAST_UPDATED`.
- Grades (overall): `GRADE_12, GRADE_10, GRADE_7, GRADE_4, GRADE_02, GRADE_00, GRADE_MINUS_3, PASSED, FAILED, ABSENT, AVERAGE_GRADE, TOTAL_STUDENTS, PERCENT_PASSED, PERCENT_FAILED, PERCENT_ABSENT` (listed twice in the constant; deduped by dict).
- Evals (overall): `{RATING,LEARNING,MOTIVATION,FEEDBACK,WORKLOAD}_{TIER,AVERAGE_SCORE,VOTES}`, `WORKLOAD_4_STAR, WORKLOAD_5_STAR`.
- Responsibles: 5×(NAME, PIC).
- Content: `COURSE_DESCRIPTION, SCOPE_AND_FORM, EXAM_TYPE_RAW, EXAM_AID_RAW, CAMPUS_RAW, TIMETABLE_PLACEMENT, EXAM_DURATION, HOME_PAGE, LEARNING_OBJECTIVES, COURSE_CONTENT, REMARKS, LEGACY_RECOMMENDED_PREREQUISITES, RECOMMENDED_PREREQUISITES, MANDATORY_PREREQUISITES, SUBSEQUENT_COURSES, HIGHLIGHTED_MESSAGE, STUDY_LINES, TOTAL_SEMESTERS, INSTITUTE`.
- Per-semester (14 each): `{SEM}_TOTAL_STUDENTS, _AVERAGE_GRADE, _PERCENT_FAILED, _GRADE_12…_GRADE_MINUS_3, _LEARNING_VOTES, _WORKLOAD_AVERAGE_SCORE, _LEARNING_AVERAGE_SCORE, _MOTIVATION_AVERAGE_SCORE, _FEEDBACK_AVERAGE_SCORE` (csv_columns_consts.py:14-18).
- Appended: `MAIN_RESPONSIBLE_COURSES, CO_RESPONSIBLE_{1..4}_COURSES`.
- Example (02402): AVERAGE_GRADE 5.49, TOTAL_STUDENTS 3367, PERCENT_PASSED 73.7, STUDENTS_PER_SEMESTER 561, TOTAL_SEMESTERS 6, RATING_AVERAGE_SCORE 3.64 (4103 votes), WORKLOAD 3.12.

### 3.10 Extended CSV `extended_csv.csv` (1746 × 1130)
- All keys from the first course's merged dict: overall grades + stats, `STUDENTS_PER_SEMESTER, TOTAL_SEMESTERS`; per eval type `{TYPE}_{VOTES,AVERAGE_SCORE,UPVOTE_RATIO,TIER,1..5_STAR}`; every info key incl. all one-hot columns (named by values_df, e.g. `Danish`, `F4A (Spring, Tues 13-17)`, `BSc, Data Science and Management`, `Computer Science`), all `_RAW`, all free-text fields; per-semester versions of all grade & eval metrics (60 suffixes × 14 sems); responsibles + course lists + SUBSEQUENT_COURSES.
- `COLUMNS_TO_COPY_OVER` (planner subset) is defined but unused (csv_columns_consts.py:58-66); `planner_csv/planner_pkl` names unused (file_name_consts.py:39,46).

### 3.11 Website JSONs
- `course_numbers.json`: `{num: num}` (website_consts.py:104).
- `course_data.json` (website_consts.py:100-156): keys `course_english_names, course_ects, course_type, course_language, course_season, course_schedule, course_signups, course_grade, course_fail, course_exam, course_workload, course_rating, course_workload_tier, course_rating_tier, course_votes (=MOTIVATION_VOTES), course_responsible (=MAIN_RESPONSIBLE_PIC), course_eval_learning, course_eval_motivation, course_eval_feedback`; each `{num: value}` sorted by value (strings sort to -0.1 for numeric columns).
  - Value renames (website_consts.py:158-246): course_type `BSc + MSc`→`BSc/MSc`; season `Autumn`→`Fall`, no-data→`No data`, combos→`2+ Timeslots`; schedule → tokens `Mon_8_12`, `Multi_Mon_8_12_Thurs_13_17`, else `Multi_Unknown`; averages: `Pass/Fail`→`P/F`, `No data`→`–`, numbers rounded to 1 dp; fail%: `No data`→`"0"`, <10 → 1 dp else int; exam: `Written/oral exam`→`Oral exam`, report/unknown→`Exam project`.
  - Sort direction flags per dict (website_consts.py:60-79).
- `json_filter_dct.json` (csv_creator.py:214-255): `{info.key_url: {value_url: [course nums with one-hot==1]}}` for every Info with values: language, gradetype, examiner, campus, course_type, exam_type, aid, assignments, course_duration, time_of_week, timetable, semester_period, study_lines, institute.
- `cv_helper.json` (csv_creator.py:257-320): for every year oldest→newest, for every course in `scraped_info_{year}.csv`: `"Name English: …\nDanish Title: …\nGeneral Course Objectives: …\nContent: …\nLearning Objectives: …"` (HTML stripped, whitespace collapsed); newer years overwrite; English name from `archived_courses_{year}.json` else `"Unknown Course Name"`; crashes on missing required columns. Includes historical courses (~2.8k). Consumed by website/recommender.py:24-27,380-387.

### 3.12 Orbit data
- `{teacher_name (unescaped): image_url | "" | default-no-portrait URL}` per academic year (scrape_orbit.py:123-180).

---

## 4. Recommender / embeddings update

- Entry: `recommender_update.run_recommender_embeddings_update()` → `CourseRecommender(eager_load=True).fit(force_recreate=True)` (recommender_update.py:3-11). Lazy import so a missing onnxruntime doesn't break main.py (recommender_update.py:5-7).
- Input: `website/static/pandas_df/course_df.pkl` (prod path tried first) (website/recommend_utils.py:160-168).
- Model: `all-MiniLM-L6-v2` (sentence-transformers) exported to ONNX at `website/static/pandas_df/model_onnx/{model.onnx, tokenizer.json}` (recommend_utils.py:20-22,44-100); original HF folder `pandas_df/all-MiniLM-L6-v2/` also committed (~90 MB safetensors). ONNX export is NOT part of the pipeline (manual).
- Tokenizer (`tokenizers` lib): BERT uncased; truncation 128 tokens, fixed padding 128 (tokenizer.json). Inputs `input_ids, attention_mask, token_type_ids` (recommend_utils.py:139-152).
- Pooling: attention-masked mean pooling + L2 normalisation → 384-dim float64 (recommend_utils.py:109-158).
- Text input per course (recommend_utils.py:235-256): concat of preprocessed `COURSE_DESCRIPTION, LEARNING_OBJECTIVES, COURSE_CONTENT, REMARKS, LEGACY_RECOMMENDED_PREREQUISITES, RECOMMENDED_PREREQUISITES` (+ MANDATORY intended, see bug). Preprocess: null/`"None"`/`"NO_DATA"`→""; strip tags; lowercase; replace every non `[a-zA-Z\s]` char with space; collapse spaces (recommend_utils.py:226-233).
- Courses with combined text ≤10 chars are dropped; df reset_index (recommend_utils.py:280-288).
- One `encode_batch` call for all texts (recommend_utils.py:291).
- Outputs: `recommended_course_embeddings_cache.pkl` (pickled ndarray, ~5.4 MB) and `recommended_filtered_courses_df_cache.pkl` (filtered df) in `website/static/pandas_df/` (or PythonAnywhere path) (recommend_utils.py:24-25,180-198).
- On PythonAnywhere (`dtu_course_project/website/static/pandas_df/` exists) fit is cache-only; `force_recreate` ignored; raises if cache missing (recommend_utils.py:263-275).
- Runtime use (context): cosine similarity between mean of input course embeddings and all courses, excluding inputs (recommend_utils.py:328-357); text and hybrid modes exist (recommend_utils.py:372-490).

---

## 5. Access log analyzer (`webhost_access_logs_analyzer.py`)

- Manual offline script for PythonAnywhere nginx access logs downloaded to `logs/manual_webhost_logs/access/` (webhost_access_logs_analyzer.py:13-14).
- Config: hardcoded `START_DATE='2026-02-18 00:00:00'`, `END_DATE='2026-02-28 23:59:59'` (UTC) (webhost_access_logs_analyzer.py:18-19); bot regex `bot|spider|crawl|slurp|chatgpt|amazon|bing|yandex|semrush|ahrefs|openai|seo` (webhost_access_logs_analyzer.py:22).
- Parsing: reads `.gz` (gzip text) and files whose name contains `.log`; regex for combined log format → ip, timestamp, method, path, status, size, referrer, user_agent (webhost_access_logs_analyzer.py:29-95). Progress every 100k lines.
- Analysis (webhost_access_logs_analyzer.py:100-163): parse timestamps (`%d/%b/%Y:%H:%M:%S %z`, coerce, drop NaT); filter date window; split bots by UA regex (case-insensitive); human page views exclude static (`.png|jpg|css|js|ico|woff2?` suffix or `/static/` prefix).
- Reports printed: total hits, bot hits and %, human page views, est. unique humans (unique IPs among human hits incl. static), top 10 human page paths, top 10 `/course/<id>` paths (prefix stripped).
- IPv4-only regex (`[\d\.]+`) → IPv6 lines silently dropped.

---

## 6. Known weaknesses, bugs, fragility

### 6.1 Bugs (concrete)
- `format_study_line_scrape` compares builtin `type == 'list'` instead of `format_type` → always returns the dict (format_study_lines.py:204). Works by accident because `in dict`/`len(dict)` behave like the list.
- Recommender omits MANDATORY_PREREQUISITES: passes the `Info` object instead of `.key_df` (recommend_utils.py:243).
- Recommender preprocessing deletes digits and non-ASCII letters (æ, ø, å) → course numbers in prerequisites and Danish text lost (recommend_utils.py:231).
- csv_creator previous-course back-fill does `df_target.loc[course]` BEFORE checking membership → KeyError if absent (csv_creator.py:106-108). Also `pd.isna` check commented out → NaN cells never filled (csv_creator.py:114).
- Grade label mapping: scraped keys are `.capitalize()`d (scrape_grades.py:70) so `"Ikke Godkendt"` in the mapping can never match (would be `"Ikke godkendt"`) (scrape_grades.py:119,129). Unmapped labels (`Syg`, `Godkendt`, …) when missing produce stray `{SEM}_ILL/APPROVED/REJECTED = "No data"` keys that are silently dropped.
- Schedule: replacement creates `Spring F7 (Tues 18-22)` but allowed value list has `Spring E7 (Tues 18-22)` → spring evening slot never matches (format_info.py:97; dtu_consts.py:107). Replacing `E7 (Tues 18-22)` also re-prefixes already-prefixed `Autumn E7` → `Autumn Autumn E7…` (format_info.py:98).
- Missing info values become the literal string `"None"` (str(None)) instead of a null sentinel because every scraped column exists (with None) so the "not found" branch is unreachable (format_info.py:191,213-214; visible in course_df.csv e.g. `MANDATORY_PREREQUISITES = None`).
- `Utils.print_progress` uses 1-based `iteration` to index `course_numbers` (off by one; IndexError possible) and its "no elements" check is dead since dict always has COURSE (utils.py:162-168).
- `run_all_scrape_scripts_one_semester_at_a_time`: `course_numbers` undefined if archive flag off or year repeats from a fresh start (scrape_all.py:83-106).
- Archive scraper returns `None` on double failure → crash at `len(None)` (scrape_archive.py:36,72-76).
- `Utils.get_course_numbers/get_course_names` read `scraped_data/course_numbers.json` which isn't produced there (it's in website/static/pandas_df) → `format_study_lines.__main__` broken (utils.py:139-153; format_study_lines.py:305).
- Access-log filter `('access' and '.log') in filename` evaluates to `'.log' in filename` (webhost_access_logs_analyzer.py:58).
- Extended CSV columns come from the first course's dict only; keys only present for later courses are silently dropped (csv_creator.py:143-147).
- Institute derived from course-number prefix, contradicts scraped Department (02402 → "Computer Science" while Department = "01 … Applied Mathematics and Computer Science") (format_info.py:233-236).
- Course responsibles scraped from non-versioned `/course/{course}/info` → historical years get today's teachers (scrape_info.py:101).
- cv_helper depends on per-year `archived_courses_{Y}.json`, which the main scrape path no longer writes (only alt mode) → new years get "Unknown Course Name" (csv_creator.py:276,299 vs scrape_all.py:37).

### 6.2 Crash risks (no error handling)
- `urllib.request.urlopen(evaluation_url)` without try/timeout → any network error aborts the whole multi-hour run (scrape_evaluations.py:46).
- `scraped_data.index(question)` raises ValueError if questionnaire wording changes; `scraped_data[1]` IndexError on empty page (scrape_evaluations.py:167,203).
- `pd.read_html(url)` for grades: no timeout; `ValueError` (no tables) and `URLError`/socket errors not caught (scrape_grades.py:60,83).
- No incremental saving/checkpointing; results written only at the end of each scraper.

### 6.3 Fragility (HTML/text coupling)
- String-split parsing of HTML (`</div>`, `<div class=`, `var lines = `, `<div class="row"><div class="col-xs-12">`, `<div class="row" style="margin:20px">`, `<b>`, `<img src="`) (scrape_info.py:151-271).
- Assumes exactly 3 tables on info page (scrape_info.py:129); table index 2 + column `Karakter` on grade page (scrape_grades.py:61-66).
- Eval parsing: fixed line offsets, Danish question strings, `href="/kursus/` + 6-digit id slicing `[6:12]`, Danish marker strings `Ingen kurser matchede s`, `Der er 6 undervisningsperioder:` (scrape_evaluations.py:89-106,173-213).
- Orbit: slug guessing, text-based 404 detection, hardcoded MISSING_LIST/SKIP_EMAIL_CHECK/MANUAL_MAPPING of named people (scrape_orbit.py:22-53).
- ~100 hand-maintained study-line string replacements, appended per academic year (format_study_lines.py:46-183) + hardcoded canonical list of 74 study lines (info_consts.py:327-401); unknown ones only produce console warnings.
- Hardcoded DTU vocabularies (schedule slots, exam types, aids, durations, course types, institute prefixes) (dtu_consts.py:63-123; info_consts.py:700-734).
- ECTS ×10 heuristic (format_info.py:60-72).
- Two-request cookie priming per page (kurser.dtu.dk JS reload) with new Session each time (scrape_info.py:79-86).
- Mixed null sentinels across stages (`None`, `"No data"`, `"No data available"`, `"Unknown data value"`, `"NO_DATA"`, `"None"`, `"–"`, `-0.1` sort key) (§3).
- Website `_create_dct_as_json` sort silently no-ops on mixed types (website_consts.py:150 comment).
- Pickles tied to exact pandas version (main.py:7-9).

### 6.4 Hardcoded / manual
- Semester list and current year edited by hand each release (config.py:4-5); `website_last_updated` hand-typed (config.py:11).
- Access-log date window hardcoded (webhost_access_logs_analyzer.py:18-19).
- Course `01005` exempted from eval vote-sum check (scrape_evaluations.py:229).
- PythonAnywhere paths hardcoded (`dtu_course_project/…`) and environment detection by path existence (file_name_consts.py:25-43; recommend_utils.py:170-172).
- Image downloader not integrated; `profile_pics` path unused; `studieordning_link.txt` unused; `COLUMNS_TO_COPY_OVER`/planner names unused.

### 6.5 Performance
- Fully sequential I/O; ~100k+ HTTP requests; grades/evals scraped for ALL historical course numbers × ALL 14 semesters every run (no caching of immutable past semesters) (scrape_all.py:41-50).
- Row-by-row `pd.concat` in scrapers and CSV build (utils.py:197-198; csv_creator.py:155-159).
- `_build_combined_df` executed twice (premade + extended) including re-loading pkls and re-running formatters (csv_creator.py:39,50).
- Archive JSON re-opened from disk on every `Utils.get_*` call (utils.py:114-137).
- `iterrows` loops (csv_creator.py:183,296; recommend_utils.py:245; scrape_orbit.py:110).
- 60 s sleeps on transient failures (scrape_info.py:43,52; scrape_evaluations.py:36).

### 6.6 Duplication / code smell
- `GRADE_COLUMNS` block duplicated in PREMADE_COLUMNS (csv_columns_consts.py:25-34).
- Same retry/progress/save boilerplate repeated in every scraper; three different progress printers (utils.py:161-177; scrape_orbit.py:283-286).
- Duplicate replacement lines in study-line fixes (e.g. `MSc,, ` ×3, `Programme-specific course (MSc)` ×2) (format_study_lines.py:50,104-105,163,183).
- `quick_test_scrape_for_debugging_please_ignore` per module instead of tests; no automated tests at all.
