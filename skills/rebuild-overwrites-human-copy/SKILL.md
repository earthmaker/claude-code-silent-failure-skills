---
name: rebuild-overwrites-human-copy
description: Before re-running a builder, check whether a person has the output file open or has edited it — regeneration looks safe, but it erases whatever work a human is doing in that file.
when_to_use: Right before running an artifact builder (build_*.py, slide/document/figure generators); when about to execute a handoff note's "re-measure first" steps verbatim; and when regenerating an output the user opened for review.
---

# Rebuilding erases the human's copy

Re-running a build reads as safe: same input, same output. The assumption breaks not at the output but at the
**output path** — if a person has that file open and is editing it, even a deterministic builder wipes their edits.
The only recovery routes are the editor's memory and cloud version history, neither of which the builder knows about.

It happened twice on one day: a slide deck was overwritten while its PowerPoint lock file existed, leaving the user
looking at a stale version; and a report being edited in a word processor was overwritten.

## Three lines before you build

```bash
ls -la <out>/                       # folder mtime, file sizes and times
ls <out>/'~$'* 2>/dev/null          # editor lock files (PowerPoint, Word)
pgrep -il 'PowerPoint|Microsoft Word|Excel|Keynote|Pages|Hwp'   # broad on purpose: a false hit costs a question, a miss costs the file
```

If any line hits, do one of these instead of building:

| Situation | Do instead |
|---|---|
| Only need to re-measure | Use the builder's `--check` or a validator — **read only**. Count slides/characters without building |
| Need to ship new content | Write under **another name** (`_v2`, date prefix). Leave the human's copy alone |
| Must replace the human's copy | Ask them to close and reopen; preserve their copy under another name first |

🔴 **Lock files are indicators, not clutter.** A subagent once "cleaned up" a `~$…pptx`. The content survived, but the
first indicator was gone and the next build could overwrite an open file. Don't delete files you didn't create in an output folder.

## The discriminator — folder mtime

Editors usually save by writing a temp file and renaming it over the original, which **updates the folder's mtime**.
Builders usually write in place (open/truncate/write), which doesn't. So if the file is newer than the folder but the folder's
mtime is stuck at some other time, **a person saved at that time**.

Example — the folder mtime stopped at 00:01, the overwriting build ran at 09:38, and the file had shrunk from 15.07 MB to 14.86 MB.
That comparison diagnosed the accident afterwards.

## Some editors leave no lock file

Some editors (for example Hancom Office HWP on macOS) don't hold a file handle or leave a lock file (`lsof` shows nothing);
they just keep the document in memory. So `lsof` and lock-file checks can't prove "not open" — **if the process is running, assume it's open.**

That property is also a recovery route: if the editor still has the document, **Save As** from the editor restores it.
Tell the user that first; cloud version history comes second.

## Handoff notes turn this into a trap

If a handoff note (a file one working session leaves for the next) has a **build** as its first step, the next session runs it without question. The report accident happened exactly
that way: the note began "rebuild the submission and check it", and the output path was the user's working copy.

→ When writing re-measurement steps in a handoff, **separate read commands from write commands**. Re-measuring should be read-only;
if a build is required, add "first confirm no one is editing that file" next to it.

## Promote it to a guard where you can

If a builder stamps its output (build time, hash) and verifies the stamp before writing, it can refuse to overwrite a
human-edited copy **with an error**. That turns silent destruction into a visible failure. Guards are per builder, so start with the one that bit you.

```python
import hashlib, json, pathlib, sys

def guarded_write(out_path, data: bytes, force=False):
    out = pathlib.Path(out_path)
    stamp = out.with_suffix(out.suffix + ".buildhash")
    if out.exists() and not force:
        if not stamp.exists():   # first run with the guard: we can't tell who wrote this file
            sys.exit(f"{out} exists but has no build stamp — it may be a human copy. Refusing without --force.")
        if hashlib.sha256(out.read_bytes()).hexdigest() != json.loads(stamp.read_text())["sha256"]:
            sys.exit(f"{out} changed since the last build — someone may have edited it. Refusing without --force.")
    out.write_bytes(data)
    stamp.write_text(json.dumps({"sha256": hashlib.sha256(data).hexdigest()}))
```

## Related

- `shipped-config-mismatch` — when a handoff's prescription was measured on a different baseline than what ships now
