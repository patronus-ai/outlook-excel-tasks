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

## Tasks — pass-rate (measured June 11)

5 cross-app tasks, all following the same pattern: **find one specific
email in the inbox (by sender + subject) and record its metadata in the
spreadsheet**. Pass-rates aggregated across multiple batches with
`max_turns=51`, `workers=1`, visual mode.

| ID (task `id:`) | Cells | Pass-rate | Band 10–50% |
|-----------------|------:|----------:|:-----------:|
| **00001-newest-invoice-record** | 5 | **4/10 = 40%** | ✅ IN |
| **00002-david-kim-lunch-record** | 5 | **4/10 = 40%** | ✅ IN |
| **00003-design-review-record** | 4 | **4/10 = 40%** | ✅ IN |
| **00004-marketing-campaign-record** | 5 | **1/10 = 10%** | ✅ IN |
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
4. Record the email's metadata across 4–5 labelled cells (Sender,
   Subject, Folder, plus 1–2 derived fields such as Year, Month, Day,
   Topic, DocType).

The reward system uses SQL queries directly against the spreadsheet's
`cells` table (`/opt/patronus-gym/apps/cua_spreadsheet/src/db/local.db`)
and a negative check against the email seed
(`/opt/patronus-gym/apps/cua_email/src/db/seed.sqlite`) to confirm no
email was modified.

## Calibration notes

The difficulty (and the resulting 10–50% band) comes from two sources,
not from ambiguous instructions:

- **Precision across two apps.** Each task requires 4–5 exact cells.
  Pass = every cell correct (logical AND). The dominant failure mode is
  that the spreadsheet's multi-row tab-paste occasionally truncates
  after the first 2–3 rows, leaving the last cells empty — this is what
  naturally produces a ~40% pass-rate for a clean 5-cell task.
- **Fair value matching.** Derived-cell values (e.g. Day, Topic,
  DocType) are matched **case-insensitively and trimmed**
  (`LOWER(TRIM(raw_value))`) so the agent is not penalised for
  reasonable capitalisation differences (`notes` vs `Notes`). Verbatim
  fields (Sender, Folder) and full-subject fields use exact match.

Per-task calibration:

- **00001 / 00003** — exact match on unambiguous derived tokens (Year,
  Month, invoice number); stable at 40%.
- **00002 / 00004 / 00005** — derived cells matched case-insensitively.
- **00005** additionally requires the **exact full subject** (incl. the
  `Re:` prefix), which centres it at 25% rather than sitting on the 50%
  boundary.
- **00004** sits at the lower edge (10%): the longer multi-word values
  make the paste-truncation bite more often. It is solvable (≥1/10) and
  comfortably under the `<50%` requirement.

## Layout

```
outlook-excel-tasks/
├── README.md                                       (this file)
├── tasks.jsonl                                     (5 tasks, runtime artifact)
└── tasks/
    ├── 00001-newest-invoice-record/definition.yaml   ← id 00001-newest-invoice-record   (40%)
    ├── 00002-count-omar-emails/definition.yaml        ← id 00002-david-kim-lunch-record   (40%)
    ├── 00003-top-sender-formula/definition.yaml       ← id 00003-design-review-record     (40%)
    ├── 00004-starred-emails-list/definition.yaml      ← id 00004-marketing-campaign-record (10%)
    └── 00005-inbox-summary-formula/definition.yaml    ← id 00005-incident-retro-record     (25%)
```

## Pass-rate methodology

- Visual mode (`use_hints: false`)
- Model: `claude-opus-4-8` (Anthropic)
- `max_turns: 51`, `workers: 1`
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
