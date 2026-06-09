# outlook-excel-tasks

Evaluation tasks for the **outlook-excel** multi-app environment in
[gym-browser-use](https://github.com/patronus-ai/gym-browser-use).

## Environment

This task set targets a single Docker image that bundles two web apps:

- **Outlook web** (`cua_email`) — `http://cua_email.web`
- **Spreadsheet web** (`cua_spreadsheet`) — `http://cua_spreadsheet.web`

Both run together, share the same headless Chrome instance and MCP server,
and expose their state via per-app SQLite DBs and the spreadsheet's
`/api/state` diff endpoint.

## Tasks

5 cross-app evaluation tasks designed to target Claude **Opus 4.8** with a
< 50% pass-rate band:

| ID | Theme | Cross-app pattern |
|----|-------|-------------------|
| `00001-invoice-amount-to-cell` | Extract 5 fields from invoice email → spreadsheet rows | email read → 9 cells (incl. ISO date conversion + signature trap) |
| `00002-budget-table-to-spreadsheet` | Recreate budget table with `=SUM` formula | email read → 12 cells + formula |
| `00003-count-sender-emails` | Count emails per sender + ratio formula | inbox counting → 4 counts + `=MAX/total*100` |
| `00004-move-to-finance-and-log` | Move email between folders AND log action in spreadsheet | email mutation + spreadsheet log |
| `00005-combined-budget-invoice-sum` | Read two emails, sum amounts with formula | 2 emails read → 6 cells + `=SUM` |

Each task contains:

- A natural-language `prompt` for the agent
- An `available_tools` list (here: `browser_*` only)
- A list of `rewards` mixing SQL queries (email DB), HTTP diff against
  `/api/state` (spreadsheet), and action triggers (screenshot)

## Layout

```
outlook-excel-tasks/
├── README.md
├── tasks.jsonl                        # Combined runtime artifact (5 lines)
└── tasks/
    ├── 00001-invoice-amount-to-cell/definition.yaml
    ├── 00002-budget-table-to-spreadsheet/definition.yaml
    ├── 00003-count-sender-emails/definition.yaml
    ├── 00004-move-to-finance-and-log/definition.yaml
    └── 00005-combined-budget-invoice-sum/definition.yaml
```

## Regenerating `tasks.jsonl`

After editing any `definition.yaml`:

```bash
python3 -c "
import yaml, json, os
out = []
for d in sorted(os.listdir('tasks')):
    p = os.path.join('tasks', d, 'definition.yaml')
    if os.path.exists(p):
        out.append(yaml.safe_load(open(p)))
with open('tasks.jsonl', 'w') as f:
    for obj in out:
        f.write(json.dumps(obj, ensure_ascii=False) + '\n')
print(f'Wrote {len(out)} tasks')
"
```

## Pass-rate targeting

Each task is hardened for **< 50% pass-rate on Opus 4.8** (visual mode,
no hints). Key hardening levers:

- Multi-step coordination across both apps (state changes in both)
- Precision constraints: ISO date format, exact text matching, formula-vs-typed-number
- Negative-state checks: nothing in the email DB or spreadsheet outside
  declared cells changes
- Trap pairs: e.g. signature name in body vs `From:` header in inbox list

Empirical pass-rate to be confirmed when `ANTHROPIC_API_KEY` is available.
