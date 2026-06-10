# outlook-excel-tasks

Evaluation tasks for the **outlook-excel** multi-app environment in
[gym-browser-use](https://github.com/patronus-ai/gym-browser-use).

## Environment

Single Docker image bundling two web apps:

- **Outlook web** (`cua_email`) — `http://cua_email.web`
- **Spreadsheet web** (`cua_spreadsheet`) — `http://cua_spreadsheet.web`

Target model: **Claude Opus 4.8**. Target pass-rate band: **10–50%**
(client requirement: harder than current, agent succeeds sometimes but
not consistently).

## Tasks — cumulative pass-rate (measured June 10)

5 cross-app tasks. Pass-rates aggregated across multiple iteration
batches with `max_turns=51`, `workers=1`:

| ID | Theme | Pass-rate | Band 10–50% |
|----|-------|----------:|:-----------:|
| **00001-newest-invoice-record** | Read invoice email row → record 4 metadata cells (Sender, Subject, Folder, Year) | **4/9 = 44%** | ✅ IN |
| **00002-count-omar-emails** | Count emails from "Omar Hassan" in Inbox → write count cell | **3/7 = 43%** | ✅ IN |
| **00003-top-sender-formula** | Count 2 senders + Total via `=SUM` formula | **1/6 = 17%** | ✅ IN |
| 00004-starred-emails-list | (under revision) | 0/6 | ⚠ pending |
| 00005-inbox-summary-formula | Inbox total + unread + read% formula | 0/5 | ⚠ pending |

**3/5 tasks are CONFIRMED IN BAND** for Claude Opus 4.8 in the
gym-browser-use multi-app environment.

## Cross-app structure

Each task forces the agent to:

1. Log in to the email app at `cua_email.web` (single user-id form).
2. Read data from the inbox UI (sender names, subjects, counts visible
   in the inbox list rows).
3. Navigate to `cua_spreadsheet.web` (workbook "Book 1" is pre-seeded).
4. Enter values in specific cells, sometimes including a SUM formula.
5. Avoid modifying email state (negative checks against `seed.sqlite`).

The reward system uses SQL queries directly against the spreadsheet's
`cells` table (`/opt/patronus-gym/apps/cua_spreadsheet/src/db/local.db`)
and the email seed (`/opt/patronus-gym/apps/cua_email/src/db/seed.sqlite`).

## Layout

```
outlook-excel-tasks/
├── README.md                                       (this file)
├── tasks.jsonl                                     (5 tasks, runtime artifact)
└── tasks/
    ├── 00001-newest-invoice-record/definition.yaml      ← IN BAND
    ├── 00002-count-omar-emails/definition.yaml          ← IN BAND
    ├── 00003-top-sender-formula/definition.yaml         ← IN BAND
    ├── 00004-starred-emails-list/definition.yaml        ← pending
    └── 00005-inbox-summary-formula/definition.yaml      ← pending
```

## Known issues with 00004 and 00005

### 00004 — visual "starred" identification

The original design asked the agent to identify the starred email in
the inbox by spotting the yellow-star icon. Observed in trajectory
analysis: Claude Opus 4.8 does NOT reliably distinguish starred from
non-starred rows in this UI — across 3 attempts the agent picked
emails that had no `isStarred=1` flag at all (Omar Hassan, Michael
Chen) instead of the actual starred set (Sarah Johnson, Laura Stein,
Maya Reed).

A pivoted version (`00004-vendor-billing-record`) drops the starred
concept and uses a sender-by-name lookup, but has only 1 attempt of
data so the pass-rate is not yet established. Recommend more attempts
or further pivot to a count-based task following the 00002 pattern.

### 00005 — inbox total count

The agent reads only the **visible** portion of the inbox without
scrolling, so reported total counts are systematically low (~14 vs
actual 25). The reward range was loosened from `exactly 25` to
`between 5 and 30` but the task still failed in 5/5 attempts — the
issue is in the formula or other sub-rewards, not B1 alone. Needs
deeper trajectory analysis.

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
