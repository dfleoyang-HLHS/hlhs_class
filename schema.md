# Data schema reference

This documents the Excel input, the `schedule_data.json` output, and the
three optional config files that feed `build_json.py`. Real, working
examples for all three configs live alongside this file in `assets/`
(`*.example.json`) — copy one and edit it rather than writing from scratch.

## 1. Excel input

One row per (class, subject, day, period-block). Expected columns (exact
header text, standard Taiwan senior-high 手排課 export):

| Column | Meaning |
|---|---|
| 班級代碼 | Class code, e.g. `101` |
| 班級名稱 | Class display name (often identical to the code) |
| 科目代碼 | Subject code |
| 科目名稱 | Subject name, e.g. `英語文` |
| 星期 | Day of week, `1`–`5` (Mon–Fri) |
| 首節 | Start period of this block |
| 末節 | End period of this block (equals 首節 for a single-period class, greater for a double/triple period) |
| 教師代碼 | Teacher code (blank for some electives/activities) |
| 教師名稱 | Teacher name |
| 教室代碼 | Room code (often blank — most classes meet in the homeroom) |
| 教室名稱 | Room name |
| 協同教師代碼 | Comma-separated teacher codes co-teaching this same slot (blank for most rows) |

A row with 首節=5, 末節=6 means one class spanning two consecutive periods.
`build_json.py` expands this into two per-period records (one for period 5,
one for period 6) rather than keeping it as a single 2-period block — a
per-period record renders correctly in the timetable without needing
rowspan logic, and a merged/rowspanned cell visually reads as "only the
first period has this class" at a glance, which is the opposite of what
you want.

Two conventions worth checking for in a new school's export:
- **A non-class row mixed into the same sheet** (e.g. class code `901` for
  department meetings / staff research groups). These have a teacher and a
  real time slot but aren't a homeroom class — `NON_CLASS_CODES` in
  `build_json.py` excludes them from the class picker while keeping them
  in the schedule so they still show up on the *teacher's* page.
- **A homeroom-period subject** (default name `導師時間`) that reliably has
  exactly one teacher per class — this is how `build_json.py` derives each
  class's homeroom teacher. If a school's export doesn't have this exact
  subject name, pass `--homeroom-subject` with whatever it's called there.

## 2. `schedule_data.json` output shape

```
{
  "meta": {"source": "...xlsx", "row_count": N, "class_count": N, "teacher_count": N},
  "classes": [{"code": "101", "name": "101", "homeroom_teacher_code": "116", "homeroom_teacher_name": "黃怡婷"}, ...],
  "teachers": [{"code": "116", "name": "黃怡婷"}, ...],
  "rooms": [{"code": "701", "name": "會議1"}, ...],
  "schedule": [ ...records... ],
  "elective_groups": [ ...see elective_groups.json below... ]
}
```

Each `schedule` record:

```
{
  "class_code": "101", "class_name": "101",
  "subject_code": "1202", "subject_name": "英語文",
  "day": 1, "start_period": 1, "end_period": 1,
  "block_start_period": 1, "block_end_period": 1,
  "teacher_code": "7022", "teacher_name": "吳秉寰",
  "room_code": null, "room_name": null,
  "co_teacher_codes": null, "co_teacher_names": null, "co_teachers": null,
  "is_rotation_partner": false, "rotation_peer": null, "rotation_tracks": null
}
```

`co_teachers` / `rotation_tracks` / `rotation_peer` are populated only when
the corresponding config applies to that slot (see below). The page's JS
checks these fields directly, so don't rename them without also updating
`template.html`.

## 3. `schedule_corrections.json` — fixing bad source rows

Use this for actual mistakes in the spreadsheet, not for 跑班 (see below for
that). The classic case: a block's 首節/末節 spans periods that don't all
really belong to that subject, so after expansion it silently overlaps a
different, genuinely-scheduled subject at the same period. You'll typically
find these because `build_json.py` (or a look at the rendered page) shows
two different subjects stacked in a cell where there should be one.

```json
{
  "remove": [
    {"class_code": "206", "day": 2, "period": 3, "subject_name": "數學A",
     "reason": "首節2/末節7 誤植成一整段，跟同時段的體育課重疊"},
    {"class_code": "901", "day": 2, "period": 4, "subject_name": "行政會議", "match_teacher_code": "1",
     "reason": "使用者指示這個教師代碼不列入資料庫；這節每位與會者各自獨立一列，移除他自己那列不影響其他人"}
  ],
  "modify": [
    {"class_code": "206", "day": 3, "period": 7, "match_subject_name": "每週團體活動時間",
     "set_subject_name": "數學A", "set_teacher_code": "310", "set_teacher_name": "姜姿戎",
     "reason": "使用者確認這節課實際是數學A"},
    {"class_code": "901", "day": 2, "period": 2, "match_subject_name": "行政會議", "match_teacher_code": "1",
     "set_subject_name": "行政會議", "set_teacher_code": null, "set_teacher_name": null,
     "reason": "同一位教師主持、但這節是單一列 + 協同教師代碼欄位列出17位與會者；移除代碼但保留會議與其他與會者"}
  ],
  "exclude_teachers": ["1"],
  "add": [
    {"class_code": "101", "day": 1, "period": 8, "subject_name": "數學輔",
     "teacher_code": "7304", "teacher_name": "林仲威", "co_teacher_codes": "7305",
     "reason": "依教師課表PDF新增，原始試算表缺漏這筆"}
  ]
}
```

- **`remove`** deletes one record identified by (class, day, period,
  subject name) — it errors out if that doesn't match exactly one record,
  so a typo in the matcher fails loudly instead of silently doing nothing
  or (worse) silently deleting the wrong one.
- **`modify`** finds one record the same way (matching on its *current*
  subject name via `match_subject_name`) and overwrites its subject/
  teacher/room. Setting `set_teacher_code`/`set_teacher_name` to `null`
  blanks the teacher while keeping the row (and its `co_teacher_codes`) —
  useful for "this meeting's host shouldn't be in the database, but the
  other 17 attendees still need to see it on their own timetable."
- **Both** accept an optional `match_teacher_code` when (class, day,
  period, subject name) alone isn't unique — this happens with standing
  meetings where every attendee has their *own* independent row (no
  `協同教師代碼` aggregation at all) rather than one row plus a co-teacher
  list; without it in that situation `build_json.py` refuses to guess and
  exits with an error naming how many rows it found.
- **`exclude_teachers`** is a flat list of teacher codes to drop from the
  teacher picker/list entirely. It does *not* touch any schedule record
  that still names that code — pair it with a `remove` or `modify` (as
  above) so nothing dangles pointing at a teacher that no longer exists.
- **`add`** covers a class period that's missing a row altogether — not a
  bad row to fix, not a 跑班 placeholder, genuinely absent. The classic
  source: a school's official per-teacher timetable (e.g. a PDF) shows a
  session that never made it into the 排課 Excel at all. Only `class_code`,
  `day`, `period`, and `subject_name` are required; everything else
  (`teacher_code`/`teacher_name`/`room_code`/`room_name`/
  `co_teacher_codes`) is optional and defaults to blank.

## 4. `rotation_overrides.json` — 跑班 (rotating electives)

Some periods aren't really one subject — the class splits into several
parallel tracks in different rooms with different teachers (e.g. a
"生物選修" period that's actually biology/IT/physics/chemistry running at
once, or a "數學選修" period split into A組/B組). The source spreadsheet
usually just has ONE row for this with no teacher, because it can't
represent a fan-out.

Each override targets one existing (class, day, period) record — the
`period` here is the **start** period, matching that record's
`start_period`. If the original class-level record has that subject name
across several periods (like the 生物選修 case: two separate periods, or
even the same one repeated on different days), you generally need one
override entry **per period**, not one for the whole week — write out
each occurrence explicitly rather than trying to make the schema imply a
recurrence.

```json
{"class_code": "305", "day": 2, "period": 3,
 "teacher_code": "513", "teacher_name": "吳曙序", "room_name": "選修一",
 "subject_name": "生物選修",
 "rotation_tracks": [
   {"subject": "資訊", "teacher_code": "7006", "teacher_name": "王宇武", "room": "語言"}
 ]}
```

- `teacher_code`/`teacher_name`/`room_name` become the *primary* track on
  the existing record (shown as the main subject/teacher/room in that
  cell).
- `subject_name` is optional — omit it to keep the record's existing
  subject text (e.g. 生物選修 was already specific enough); set it when the
  original name was a generic placeholder and a track-specific name reads
  better, e.g. renaming a bare "數學選修" to "數學選修（A組）" so the primary
  and rotation side read symmetrically.
- `rotation_tracks` is a list (usually 1–3 entries) of the *other*
  simultaneous tracks. Each becomes: (a) a line shown on the class's
  timetable cell alongside the primary track, and (b) a **synthetic
  schedule record** for that partner teacher, so their own timetable also
  shows the session (otherwise it would only exist folded into the primary
  record and be invisible on the partner teacher's page). Synthetic
  records are marked `is_rotation_partner: true` and carry a
  `rotation_peer` pointing back at the primary track — the class view
  filters these out (they're already shown via the primary record's
  `rotation_tracks`, so keeping them too would double-count that class's
  weekly period count).

**Multi-class rotations**: when several classes combine for the same
track (e.g. two classes' biology students merge into one class taught
once), just give each class its own override entry with the *same*
teacher_code/room for that shared track. The teacher's own timetable will
then correctly show every one of those classes stacked in that one time
slot — that's a feature, not a bug, since it's genuinely one physical
session serving several classes.

**Teachers with no code anywhere in the spreadsheet**: a track's teacher
might never appear as a primary 教師代碼 in the whole sheet (they only ever
teach this one rotation slot). There's no real code to reuse — invent a
synthetic one (e.g. `L01`, `L02`, ...) and add it to
`supplemental_teachers.json` (see below) so the page's teacher list and
search can find them.

**Free-choice electives with no per-class attribution**: some "跑班" slots
are genuinely open enrollment across a whole grade (e.g. a menu of 10+
elective offerings that any of 6+ homerooms' students can individually
pick from) rather than a clean combination of a couple of classes. If the
source data (or a school PDF) gives you the full menu — course, teacher,
room — but *not* which students/classes actually ended up in which
section, don't guess an attribution. Instead give every eligible class its
own override with a **null primary** (`"teacher_code": null,
"teacher_name": null, "room_name": null`, subject_name left as whatever
generic placeholder the source already used, e.g. "多元選修") and the
*entire* menu as `rotation_tracks` — the page skips rendering the (empty)
primary line and just lists every option. This is honest about the
uncertainty rather than fabricating a specific class for each course. On
the teacher side, each offering's teacher will then show every eligible
class stacked at that slot (e.g. all 6 classes) — over-inclusive, but
correctly reflects "we know these are the eligible classes, not the exact
split," which is what was actually knowable from the source.

**Before finalizing overrides**: cross-check every (teacher_code, day,
period) pair used across all overrides — both the primary tracks and every
`rotation_tracks` entry — against the *rest* of the schedule for that same
teacher_code. If a rotation teacher is already scheduled to teach a
completely different class at that same slot, something is wrong (either
the rotation data or the existing schedule row) — don't silently pick a
side; surface it and ask. A one-off Python check (`build_json.py` already
adds `is_rotation_partner` to every record, so this is a simple filter +
group-by) catches this before it ships.

## 5. `elective_groups.json` — the reference table at the page bottom

Purely a display aid — a color-coded card grid summarizing which teacher
and room each class's elective group maps to, shown below the timetable.
It doesn't feed the "跑班" cell rendering (that comes from
`rotation_overrides.json`); the two are usually kept in sync by hand
because they answer different questions ("what's the deal with 305's
biology elective" vs. "show me every class in every rotation on one
table").

```json
{
  "elective_groups": [
    {
      "id": "native_language_g1",
      "title": "高一本土語",
      "note": "依班級分四種語別小班上課",
      "color": "amber",
      "classes": [
        {"class_code": "101", "tracks": [
          {"name": "閩南語", "teacher": "徐堃明", "room": "101教室"},
          {"name": "客語文", "teacher": "溫李秋", "room": "選修四"}
        ]}
      ]
    },
    {
      "id": "math_grouping_g2",
      "title": "高二數學",
      "note": "分組教學",
      "color": "teal",
      "classes": [
        {"class_code": "201", "groups": [
          {"name": "數A組", "teacher": "邱培杰", "room": "201教室"},
          {"name": "數B組", "teacher": "姜姿戎", "room": "選修四"}
        ]}
      ]
    }
  ]
}
```

Use `"tracks"` or `"groups"` interchangeably per class (the page's JS reads
whichever is present) — pick whichever reads more naturally for that
group. `color` must be one of `amber` / `teal` / `plum` / `sky` (the four
accent tokens defined in `template.html`); if a school ends up needing a
5th distinct rotation category, add a new CSS variable pair there
(`--<name>` / `--<name>-soft`, in all three theme blocks) and register it
in `GROUP_COLOR_VARS` in the script section, following the existing four
as a pattern.

## 6. `supplemental_teachers.json`

A flat `{code: name}` object for teachers who never appear as a primary
教師代碼 anywhere in the spreadsheet (only as a 協同教師代碼 or only inside a
`rotation_tracks` entry). Real leftover codes (like a co-teacher code that
happens to already exist in the source data) don't need an entry here —
only teachers with **no** code in the source at all do.

**Prefer a real code over inventing one.** A teacher missing from the
排課 Excel often still has an official code the school uses elsewhere —
check for it in any other export the school can provide before falling
back to a synthetic code (e.g. `L01`, `L02`, ...): a per-teacher timetable
PDF, an HR/staff roster, an ID-card system export. This actually happened
during this skill's own development — six teachers (native-language
instructors only teaching one rotating elective) got synthetic `L01`–`L06`
codes because they weren't in the 排課 Excel at all, but turned out to have
real codes (`7801`, `7802`, ...) once the school's official
per-teacher-timetable PDF was checked. If the user later hands you such a
document, cross-reference every teacher's code against it (see the
worked example in this skill's own project history — extract each page's
`教師: <code> <name>` line with `pdfplumber`, not by rendering every page as
an image, which doesn't scale much past a handful of pages) and correct
any synthetic codes you find real ones for — update both
`supplemental_teachers.json` and every matching `teacher_code` inside
`rotation_overrides.json`, then rebuild.
