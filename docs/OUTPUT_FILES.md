# Output Files

## `emails.txt`
One **complete** email-like string per line, discovery order, original casing of first sighting, case-insensitively unique.

## `incomplete_emails.txt`
One **incomplete/broken-looking** candidate per line (e.g. `jane@`, `ops@example`). Treat as leads for manual review, not as contacts.

## `last_successful_page.txt`
A single integer. Written after the two output files for each processed page.

## Write behavior

- `write_lines()` opens with `"w"` → **truncates then rewrites** the full accumulated list every page.
- Three separate writes per page: complete → incomplete → checkpoint. Not atomic.
- A fresh run (`--start-page` unset or 1) **starts empty and overwrites** previous files.
- Resume (`--start-page > 1`) loads existing files first and merges.

## Checkpoint semantics

| What it means | What it does *not* mean |
|---|---|
| The script reached the checkpoint write for that page | Every earlier page was correctly scraped |
| | The page was complete (rows could still have been loading) |
| | The run will auto-resume (you must pass `--start-page`) |

If a page fails inside the per-page `try/except`, the loop still tries to advance. Pages can therefore be skipped without stopping the run; check the console for `Error scraping page`.

## Recommended practice

- One working directory per commodity: `mkdir coffee && cd coffee && python3 ../key.py -C coffee`.
- Back up files before resuming.
- Do not commit outputs; they contain third-party contact data.
- Treat debug HTML/PNG as sensitive.

## Possible future format

```csv
run_id,commodity,page,company,email,status,found_at
2026-10-18T10:00Z,coffee,12,Acme Imports,sales@acme.com,complete,2026-10-18T10:05:11Z
```
Row-level capture would let you connect an address to the shipment/company it came from.
