## Coding philosophy

These are principles, not rules. Read the situation and apply judgment - don't follow mechanically.

**Solveit is collaborative, not autonomous.** Treat the dialog as an interactive workspace with a human in the loop, not as a background IDE. Prefer proposing the smallest next useful step over doing a large task automatically. Use tools only when they materially help the current agreed step. For code, usually show the user a small cell to run rather than executing or editing on their behalf, unless they clearly asked for that.

**Keep agency with the user.** Before changing dialogs, files, APIs, or structure, explain the intended change and stop for confirmation. Avoid "while I'm here" cleanup or speculative refactors. The best Solveit flow is: understand, propose, confirm, change, verify, summarize.

**One visible step at a time.** Make each step inspectable: read the relevant context, make one small change, run or verify it, then report what happened. If the next move depends on taste, design direction, or task priority, ask instead of guessing.

**Explore before implementing.** Before writing any code, understand the structure of what already exists. Read the relevant dialogs, map the hierarchy, identify the pattern. Code written without this understanding usually needs to be thrown away.

**Design by dialogue.** Don't specify a complete API upfront. Propose a direction, sketch the minimal version, let the right shape emerge from use. Ask before deciding.

**Incremental and empirical.** One change at a time. Run it, see the output, then proceed. Never make multiple changes before verifying the first one worked.

**Data over code.** When something is being repeated or configured, reach for a data structure before reaching for a function or class. Configuration as data, logic as infrastructure.

## Python export

Never edit `.py` files directly - always edit the `.ipynb` dialog and export.

To export all notebooks:
```bash
nbdev-export
```

To export a single notebook:
```bash
nbdev-export --path fs.ipynb
```


## Programming rhythm

Don't abstract until the pattern is clear. Start with the smallest working expression and extend one idea at a time. Functions emerge from working code; they don't precede it.

A cell is a checkpoint, not a step. Extend it while refining the same question; start a new one when building on top of what's proven.

### Avoid - API designed before anything works
```python
def parse_csv(path: str, delimiter: str = ',', skip_header: bool = True) -> list[dict]:
    with open(path) as f:
        rows = f.readlines()
    ...
```

### Good - pattern first, name it last
```python
# cell 1 - smallest concrete thing
line = "ABC,10,5.00"
line.split(',')

# cell 2 - extend to multiple rows
rows = ["ABC,10,5.00", "DEF,3,12.00"]
[o.split(',') for o in rows]

# cell 3 - pattern is clear, now name it
def parse_rows(rows): return [o.split(',') for o in rows]
```

Once all pieces work, propose consolidating into a function - show the code first, wait for confirmation before adding it. Delete exploration cells after consolidating.

## Coding workflow

Before any dialog edit, show the proposed change and wait for explicit confirmation. **Never call a tool in the same response as a proposed change.** The response must end after the proposal.

### Avoid - proposal and edit in the same response
> Here's the proposed change: `def CustomerPage(slug):...`
> [immediately calls msg_str_replace]

### Good - proposal only, then stop
> Here's the proposed change: `def CustomerPage(slug):...`
> [response ends - wait for user confirmation before any tool call]

Before any git commit, show the proposed commit message and wait for explicit confirmation.

## Dialog manipulation

Never use `NotebookEdit`, direct JSON editing, or raw `.ipynb` manipulation.

`termskill` is the primary tool for all dialog reads and edits; `sic` is for quick shell one-liners only.

### `dialoghelper.termskill` - primary tool

Use for any in-place message edits (string replace, line replace, Python-code edits) and for multi-step workflows where pinning the dialog saves repetition. All functions are `async`.

Dialog names use a **leading `/`**: `/projects/dunkin/<path>` (no `.ipynb`).

```python
# One-liner (single async call)
python3 -c "
import asyncio
from dialoghelper.termskill import view_dlg
print(asyncio.run(view_dlg('/projects/dunkin/pages/inventory')))
"

# Multi-step workflow - pin dialog once, then chain calls
python3 << 'EOF'
import asyncio
from dialoghelper.termskill import set_dialog, view_dlg, add_msg, msg_str_replace, msg_replace_lines

async def main():
    set_dialog('/projects/dunkin/pages/inventory')
    msgs = await view_dlg()          # view all messages
    id1 = await add_msg('note text', msg_type='note')
    await msg_str_replace(id1, 'old', 'new')   # in-place edit
    await msg_replace_lines(id1, 3, 5, 'replacement\n')

asyncio.run(main())
EOF
```

Key functions:
- `set_dialog('/projects/dunkin/<path>')` - pin dialog for subsequent calls
- `view_dlg(dname)` - concise XML view of all messages
- `find_msgs(re_pattern, dname=...)` - search by content/type; `dname` is keyword-only
- `add_msg(content, msg_type, id, placement, dname)` - add message; placement: `add_after`/`add_before`/`at_start`/`at_end`
- `update_msg(id, content, dname)` - replace full message content
- `msg_str_replace(id, old_str, new_str)` - targeted string replacement
- `msg_strs_replace(id, old_strs, new_strs)` - multiple replacements in one call
- `msg_replace_lines(id, start, end, new_content)` - replace line range (1-based)
- `msg_insert_line(id, insert_line, new_str)` - insert after line number
- `msg_del_lines(id, start, end)` - delete line range
- `msg_python(id, code)` - edit content via Python (`text` var holds content, last expr is new content)
- `del_msg(id, dname)` - delete message
- `update_msg(id, is_exported=1, dname)` - mark a cell as exported (use this, not add_msg)

Always call `view_msg(id)` immediately before any line-based edit - never rely on line numbers from earlier in the conversation. To verify a cell was exported correctly, use `view_msg(id)` - never `grep` or `json.load` on the `.ipynb` file directly.


#### Common mistakes

**`find_msgs` - `dname` is a keyword arg, not positional.** Passing it positionally silently binds it to `msg_type`, leaving `dname=''` and triggering a stack-scan that fails outside a Solveit kernel:
```python
# wrong - second arg goes to msg_type, not dname
await find_msgs('PickerBtn', '/projects/dunkin/components')

# correct
await find_msgs('PickerBtn', dname='/projects/dunkin/components')
```

**`add_msg` without an `id` chains relative to the current message.** Multiple calls without capturing the return value insert messages in reverse order. Always capture and forward the id:
```python
# wrong - second message ends up before the first
await add_msg('first', msg_type='note')
await add_msg('second', msg_type='note')

# correct
id1 = await add_msg('first', msg_type='note')
id2 = await add_msg('second', msg_type='note', id=id1, placement='add_after')
```

### `sic` - quick shell one-liners

Use for one-off lookups, listing messages, and executing cells. No Python boilerplate needed.

Auth: `SOLVEIT_TOKEN` is set; no `--url` or `--token` flags needed.
Dialog names use **no leading `/`**: `projects/dunkin/<path>`.

```bash
sic                              # list namespaces
sic dialog --help                # list all dialog methods

sic dialog.messages --name projects/dunkin/<path> | jq   # list messages with IDs
sic dialog.add_msg 'content' --name projects/dunkin/<path> --msg_type note
sic dialog.add_msg 'content' --name projects/dunkin/<path> --msg_type note --placement add_before --id <id>
sic message.exec --name projects/dunkin/<path> --id <id>   # execute a cell
sic message.delete --name projects/dunkin/<path> --id <id>
sic client.create_dialog --name projects/dunkin/<path>
```

**sic arg rules:**
- Dialog name → `--name projects/dunkin/<path>` (NOT a positional arg)
- Message content → first positional arg, NOT `--content` (causes "multiple values" error)
- `msg_type` values: `note`, `code`, `prompt`, `raw` (NOT `md`)
- Adding a `prompt` message does NOT trigger the AI - must follow with `message.exec`

### Stopping and restarting a dialog

Always use `sic` to stop and restart dialogs:

```bash
# Stop, then run all cells (restart)
sic dialog.stop --name projects/dunkin/main && sic dialog.run_all --name projects/dunkin/main

# Run all cells without stopping first
sic dialog.run_all --name projects/dunkin/main
```

Dialog name uses **no leading `/`** (same as all `sic` commands).

## Git commit messages

Follow Linux kernel patch style:
- Subject: `subsystem: short imperative description` (50 chars max, no period)
- Blank line between subject and body
- Body: explains *why*, not *what*; wrap at 72 chars

Subsystem is typically the file/module (e.g. `fs`, `pages/admin`, `views/dftable`).

### Avoid
```
Updated weekly report to fix date range issue.
```

### Good
```
pages/weekly: drop ref_mo default to fix stale date range

Prior default silently used last month when ref_mo was omitted,
causing the report to show stale data after month rollover.
```

## UI terminology

Use Swift/iOS naming conventions for UI components:
- **Cell** not Card
- **Section** not Group
- **Header** not Title bar

## Naming

Commonly used names should be short; rare ones can be longer.

Standard short names:
- `o` - object in comprehensions
- `i` - index
- `k`, `v` - dict key, value
- `x` - primary input; also first lambda parameter (`y`, `z` for subsequent)
- Domain abbreviations are fine (`tfm`, `det`, `coord`, etc.)

### Prefixes and suffixes

- `is_` - boolean (e.g. `is_active`)
- `n_` - count (e.g. `n_items`)
- `to_` - type conversion (e.g. `to_pct`)
- `_s` suffix - plural/collection (e.g. `tfms`, `cols`)
- `2` infix - conversion between types (e.g. `name2idx`)

### Avoid
```python
def filter_dictionary(dictionary, filter_function):
    return {key: value for key, value in dictionary.items() if filter_function(key, value)}
```

### Good
```python
def filter_dict(d, func):
    return {k:v for k,v in d.items() if func(k,v)}
```

### Avoid
```python
def get_first_element(iterable, filter_func=None, negate=False):
    iterator = iter(iterable)
```

### Good
```python
def first(x, f=None, negate=False):
    x = iter(x)
```

## Python style

No automatic formatters (autopep8, yapf, linters) - hand formatting preserves domain intent.

**One line, one idea.** Every line should express exactly one complete semantic concept - no more, no less.

```python
return b if a is None else a                       # null coalescing is one idea
return ''.join(s.title().split('_'))               # transform chain is one idea
for k,v in d.items(): (fs,ts)[f(k,v)][k] = v      # partition step is one idea
```

Never align parameters or arguments with extra spaces for visual column alignment.

Optimize for vertical space - PEP8 is not a concern. Prefer single-line functions where readable.

Prefer double quotes for strings.

### Good
```python
f"Imported → {dst.split('/')[-1]}"
def _cust_badge(s): return Span(cls="border px-2 text-xs rounded-lg text-gray-400 border-gray-400")(s)
```

### Avoid
```python
f"Imported → {dst.split(\"/\")[-1]}"
def _cust_badge(s):
    return Span(cls="border px-2 text-xs rounded-lg text-gray-400 border-gray-400")(s)
```