---
name: school-schedule-query-site
description: Turn a school's "班級手排課" (class timetable) Excel export into a structured JSON database plus a responsive, self-contained HTML query webpage with a class-view and teacher-view, cross-linked, mobile+desktop responsive, supporting "跑班"/rotating-elective overrides, a color-coded elective-group reference table, and homeroom-teacher display. Use this whenever the user uploads or references a school class-schedule Excel file (columns like 班級代碼/班級名稱/科目代碼/科目名稱/星期/首節/末節/教師代碼/教師名稱/教室代碼/教室名稱/協同教師代碼) and wants a lookup/query tool, a way to browse by class or by teacher, or to publish/share/deploy such a page (including to GitHub Pages) — even if they just say "help me build a course search site" or "查詢系統" without naming this skill. Also use it for follow-up requests on a site already built this way: adding elective/rotation groups, fixing a mislabeled schedule row, adding cross-navigation, or making the page more mobile-friendly.
---

# School schedule query site

Builds a two-mode (班級 / 教師) timetable lookup page from a school's raw
排課 Excel export. The pipeline is: **Excel → `schedule_data.json` →
HTML page**, glued together by two scripts in `scripts/`. Everything here
is school-agnostic — no class list, teacher name, or elective arrangement
is hardcoded anywhere in the bundled scripts or template. All of that comes
from the Excel file plus whatever config JSON you write for that school's
specific quirks.

## When a school's data has real complexity, don't skip it

This skill exists because a straightforward "read Excel, make a table"
build is *not* what most real schools' data looks like. Expect all of the
following, and don't treat them as edge cases to work around — they're the
actual job:

- **Rows the spreadsheet couldn't represent cleanly.** A period where a
  class splits into 3–4 tracks in different rooms shows up as one row with
  no teacher. A grade-wide rotating elective looks identical across many
  classes. This skill's `rotation_overrides.json` mechanism exists
  specifically for this — read `references/schema.md` §4 before assuming a
  gap is unfixable.
- **Actual mistakes in the source file.** A block's 首節/末節 spanning too
  many periods, silently overlapping a real class. Don't paper over an
  overlap by picking one side — verify against another source (a teacher
  confirming their own schedule, a printed timetable, common sense about
  which subject fits the rest of that day) or ask the user.
- **Teachers with no code anywhere in the sheet.** Some instructors (native
  language teachers, an NET, a co-teach-only aide) never appear as a
  primary 教師代碼. They still need to show up correctly in *their own*
  teacher-view page — see `references/schema.md` §4 and §6.
- **A rotation partner teacher's own page is easy to forget.** If a
  "生物選修" row's other tracks are only recorded as extra fields on the
  biology record, the IT/physics/chemistry teachers' own timetables will
  show nothing for that slot. `build_json.py` generates synthetic partner
  records automatically from `rotation_overrides.json` — you don't need to
  write these by hand, but you do need to actually supply the override so
  they get generated.

If the user reports "my teacher's schedule doesn't show this class" or "my
class shows this course twice" or "why is this teacher double-booked",
those are signals to go fix the underlying config file, not to patch the
generated HTML by hand.

## Workflow

### 1. Get the Excel file and check the columns

Ask for the file if it's not already provided or referenced. Peek at its
columns (`pandas.read_excel(path).columns`, or ask the user to confirm from
a screenshot) against the 12 expected in `references/schema.md` §1. If a
header differs slightly, either rename it in the Excel file or edit the
`COL_*` constants at the top of `scripts/build_json.py` for this run —
don't build a generic column-mapping system, this is a stable convention
and a one-line edit is simpler.

Ask the user for the **school's name and a short badge abbreviation**
(e.g. 花蓮高中 / HLHS) if it's not obvious from the filename or from
context — these go into the page header/title/footer.

### 2. Run the build (first pass, no configs)

```bash
python scripts/build_json.py --excel "path/to/school.xlsx" --output schedule_data.json
python scripts/package.py --template assets/template.html --data schedule_data.json \
  --school-name "花蓮高中" --school-abbr "HLHS" \
  --source-label "班級手排課（2026-09-07 匯出）" \
  --out-dir . --basename schedule
```

This alone produces a working page — every class and teacher's *regular*
schedule, cross-linked, mobile-responsive. Do this before asking the user
about elective groups or rotations; it gives you (and them) a working
baseline to spot gaps in, rather than trying to anticipate every quirk of
their school's data up front.

### 3. Look for what needs a config file

Skim the generated data for blank-teacher rows on subjects that clearly
have real students in real rooms (usually "XX選修" or "XX跑班"-sounding
subject names) — that's the signature of a rotating elective needing
`rotation_overrides.json`. If the user mentions a specific class or
teacher's schedule looking wrong, that's `schedule_corrections.json`. If
they want a summary table of grouped electives at the page bottom (nice to
have, not required for the page to work), that's `elective_groups.json`.

Read `references/schema.md` for the exact shape of each, and copy the
matching `assets/*.example.json` as a starting point rather than writing
one from scratch — they're real, working examples, not toy fragments.

Re-run the build with whichever configs apply:

```bash
python scripts/build_json.py --excel "path/to/school.xlsx" --output schedule_data.json \
  --corrections schedule_corrections.json \
  --rotations rotation_overrides.json \
  --electives elective_groups.json \
  --supplemental-teachers supplemental_teachers.json
python scripts/package.py --template assets/template.html --data schedule_data.json \
  --school-name "花蓮高中" --school-abbr "HLHS" --source-label "..." \
  --out-dir . --basename schedule
```

`build_json.py` fails loudly (non-zero exit, clear message) if a
correction or override doesn't match a real record, or matches more than
one — treat that as a sign to re-check your (class, day, period, subject)
targeting, not something to silence.

**Before shipping any rotation config**, cross-check every teacher/day/
period combination it introduces against that teacher's *existing*
schedule for a conflict (same teacher, same slot, a different class they
were already scheduled for). See `references/schema.md` §4's last section
for how — this is what catches a genuine data error in the source sheet
before it reaches the user, and it has caught real mistakes in practice.
If you find one, don't silently prefer one source over the other — tell
the user what you found and ask.

### 4. Verify locally before handing anything over

```bash
python scripts/serve_preview.py --dir . --port 8792
```

Then open `http://localhost:8792/schedule_standalone.html` in a browser
tool and sanity-check: pick a class, pick a teacher, click a cross-link,
and if the harness's browser can actually emulate a narrow viewport, check
mobile width too. `serve_preview.py` sets the UTF-8 content-type header
that a plain `python -m http.server` doesn't — without it, Chinese text
renders as mojibake in most local testing setups, which will send you
chasing a phantom encoding bug in the actual data (this happened during
this skill's own development — the underlying JSON was fine, the local
test server was silently guessing the wrong encoding).

### 5. Hand over both outputs, and know which is which

- **`schedule.html`** (bare, no `<!DOCTYPE>`/`<html>`/`<head>`/`<body>`) —
  publish this one with the Artifact tool, which wraps bare content in its
  own skeleton (including a viewport meta tag) at publish time.
- **`schedule_standalone.html`** (a complete document with its own charset
  + viewport meta) — this is the one to hand the user as a download, and
  the one they should deploy anywhere else (GitHub Pages, a school
  intranet server, emailed as an attachment). **Never hand out the bare
  file for self-hosting** — without the viewport meta tag a phone browser
  renders it at a fake ~980px desktop width and then shrinks it to fit,
  so every bit of the responsive CSS silently never engages. This exact
  failure happened once already: the page looked fine on a quick desktop
  check, then came back from the user's actual phone squeezed and
  illegible, and the eventual root cause was the missing meta tag, not
  the CSS. Publish and send the standalone file from the very first
  version, not just once someone reports a broken phone.

Send `schedule_data.json` alongside it if the user wants the raw data too
(e.g. "以 JSON 方式建立查詢資料庫" is itself a common ask) — it's a legitimate
deliverable on its own, not just an intermediate build artifact.

### 6. Keep the two HTML outputs in sync going forward

Any later change (adding a rotation group, fixing a class, adding a UI
feature) means re-running `build_json.py` then `package.py` — always
regenerate **both** `schedule.html` and `schedule_standalone.html` and
republish/resend both, even if only one of them seemed relevant to the
change. They're generated from the same template and the same data; let
them drift and the next bug report will be "it works on the link you sent
but not the file I have," which is much harder to debug than it needs to
be.

If the change is to `assets/template.html` itself (a new UI feature, a
styling fix), make it in the skill's own copy under
`C:\Users\user\.claude\skills\school-schedule-query-site\assets\template.html`
so future runs of this skill — for this school or a different one — start
from the improved version, not just the one-off copy sitting in whatever
working directory this build happened in.

## Reference

- `references/schema.md` — full schema for the Excel input, the
  `schedule_data.json` output, and all four config files, with the
  reasoning behind each field (not just its shape).
- `assets/*.example.json` — real, working config examples (from Hualien
  Senior High's build) to copy and adapt, not hardcoded defaults.
