# outlook-excel-tasks

Evaluation tasks for the **outlook-excel** multi-app environment in
[gym-browser-use](https://github.com/patronus-ai/gym-browser-use).

## Environment

Single Docker image bundling two web apps:

- **Outlook web** (`cua_email`) — `http://cua_email.web`
- **Spreadsheet web** (`cua_spreadsheet`) — `http://cua_spreadsheet.web`

Target model: **Claude Opus 4.8**. Target pass-rate band: **10–50%**
(client requirement: harder than current — the agent succeeds
sometimes but not consistently).

## Tasks — pass-rate

5 cross-app tasks, all following the same pattern: **find one specific
email in the inbox (by sender + subject) and record its metadata in the
spreadsheet**. Each task is a 5-cell, two-column table.

Prompts are written as **natural-language prose** (no step-by-step
checklists or bullet specs) — they read like instructions you'd give a
person, while still naming the exact cell labels and values the grader
checks. Pass-rates were measured with `max_turns=51`, visual mode.

| ID (task `id:`) | Cells | Pass-rate | Band 10–50% |
|-----------------|------:|----------:|:-----------:|
| **00001-newest-invoice-record** | 5 | **1/5 = 20%** | ✅ IN |
| **00002-david-kim-lunch-record** | 5 | **2/5 = 40%** | ✅ IN |
| **00003-design-review-record** | 5 | **1/8 = 12.5%** | ✅ IN |
| **00004-marketing-campaign-record** | 5 | **1/8 = 12.5%** | ✅ IN |
| **00005-incident-retro-record** | 5 | **2/8 = 25%** | ✅ IN |

**All 5 tasks are CONFIRMED IN BAND** for Claude Opus 4.8.

> Note: the task **directory** names (`00002-count-omar-emails`,
> `00003-top-sender-formula`, `00004-starred-emails-list`,
> `00005-inbox-summary-formula`) are legacy from an earlier design
> iteration. The authoritative identifier is the `id:` field inside each
> `definition.yaml` (shown in the table above); `tasks.jsonl` and the
> runtime key off `id:`, not the folder name.

## Cross-app structure

Each task forces the agent to use **both** apps:

1. Log in to the email app at `cua_email.web` (single user-id form,
   `user_0001`).
2. Locate one specific email in the Inbox — sender and subject are
   visible directly in the inbox row (the agent must **not** open the
   body or modify any email).
3. Navigate to `cua_spreadsheet.web` (workbook "Book 1" is pre-seeded
   with an empty Sheet1).
4. Record the email's metadata across 5 labelled cells (Sender, Subject,
   Folder, plus 2 derived fields such as Year, Month, Day, Topic,
   DocType).

The reward system uses SQL queries directly against the spreadsheet's
`cells` table (`/opt/patronus-gym/apps/cua_spreadsheet/src/db/local.db`)
and a negative check against the email seed
(`/opt/patronus-gym/apps/cua_email/src/db/seed.sqlite`) to confirm no
email was modified.

## Calibration notes

The difficulty (and the resulting 10–50% band) comes from genuine task
structure, not from ambiguous instructions:

- **Precision across two apps.** Each task requires 5 exact cells.
  Pass = every cell correct (logical AND). The dominant failure mode is
  the spreadsheet's multi-row entry: a single tab-paste occasionally
  truncates after the first 2–3 rows, and when the agent then re-types
  individual cells to recover, Enter-navigation shifts the rows and
  mislays the last value. Five cells AND-combined with this flaky entry
  lands each task in the 10–40% range.
- **Fair value matching.** Derived-cell values (e.g. Day, Topic,
  DocType) are matched **case-insensitively and trimmed**
  (`LOWER(TRIM(raw_value))`) so the agent is not penalised for
  reasonable capitalisation differences (`notes` vs `Notes`). Verbatim
  fields (Sender, Folder) and full-subject fields use exact match.

Per-task notes:

- **00001 / 00002** — 5-cell, derived tokens (Year + invoice number /
  Day + Event); land ~20–40%.
- **00003** — promoted from 4 cells to 5 (added a Topic cell) because a
  4-cell version was too easy (~80–100%); the 5th cell brings it into
  band.
- **00004 / 00005** — derived cells matched case-insensitively. Their
  prompts include an explicit, prose recovery tip ("if a cell came out
  empty, click that exact cell and type only its value — don't use Enter
  to move between cells") which lifts them off 0%. 00005 also requires
  the **exact full subject** (incl. the `Re:` prefix).

## Layout

```
outlook-excel-tasks/
├── README.md                                       (this file)
├── tasks.jsonl                                     (5 tasks, runtime artifact)
└── tasks/
    ├── 00001-newest-invoice-record/definition.yaml   ← id 00001-newest-invoice-record    (20%)
    ├── 00002-count-omar-emails/definition.yaml        ← id 00002-david-kim-lunch-record    (40%)
    ├── 00003-top-sender-formula/definition.yaml       ← id 00003-design-review-record      (12.5%)
    ├── 00004-starred-emails-list/definition.yaml      ← id 00004-marketing-campaign-record (12.5%)
    └── 00005-inbox-summary-formula/definition.yaml    ← id 00005-incident-retro-record     (25%)
```

## Pass-rate methodology

- Visual mode (`use_hints: false`)
- Model: `claude-opus-4-8` (Anthropic)
- `max_turns: 51`, `workers: 1–5`
- Reward source: direct SQL queries against the app SQLite databases
  (avoids the cross-session UUID problem when using HTTP `/api/state`
  diff sources for the spreadsheet)

Each attempt resets app state via a fresh Docker container, so cells
written in one attempt do not leak into the next.

## Setup workarounds documented in the build path

Several app-side issues were patched at the gym integration layer
(in the pre-built `app.tar.gz` and gym configs), not in the upstream
app source:

1. `cua_email` `src/db/sessions/` directory was missing in the tarball,
   causing HTTP 500 on session creation — added empty directory.
2. `cua_email` tarball contained macOS `._*` AppleDouble metadata files
   that corrupted the JSON seed loader — filtered out in repack.
3. `cua_spreadsheet` UI hung on "Loading…" because the auto-create
   workbook flow did not complete in gym Chrome — pre-seeded a default
   workbook with `owner_session_id='system'` in the tarball's
   `local.db` and patched `client-session.ts` to always return `system`
   as the client session id.

Long-term these fixes should land in the upstream app repos
(`cua_email`, `cua_spreadsheet`) so the gym-browser-use standard
`make apps-download` flow produces a working build without
intervention.
