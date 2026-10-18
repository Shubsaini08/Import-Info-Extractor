# Development Guide

## Code style notes
- Type hints throughout; helper functions are small and single-purpose.
- Comments explain *why* (Angular events, hidden CSS, regex history). Keep that habit.

## Testing

### Unit-testable (no browser)
`extract_emails` and the regexes.

```python
import pytest
from key import extract_emails

@pytest.mark.parametrize("text,complete,incomplete", [
    ("a@b.com", ["a@b.com"], []),
    ("see a@b.com,", ["a@b.com"], []),
    ("jane@ or john@example.com", ["john@example.com"], ["jane@"]),
    ("ops@example", [], ["ops@example"]),
    ("A@B.com a@b.com", ["A@B.com"], []),
    ("@@@ @", [], []),
])
def test_extract(text, complete, incomplete):
    assert extract_emails(text) == (complete, incomplete)
```

### Browser-dependent
Use saved HTML fixtures of the pagination and login pages and run helpers against a local static page via Selenium, or mock `driver` with `unittest.mock`. Cover: expired session, login error, empty results, changed pagination markup, stale elements, duplicates, failed navigation, interrupted writes.

### Manual smoke test
```bash
python3 key.py -C coffee --headed --end-page 3 --delay 2
```

## Improvement roadmap

1. **Security first** — env credentials, rotate password.
2. **Explicit navigation state** — assert indicator == target; abort on jump failure.
3. **Recoverability** — page status table (ok / parse-failed / load-failed / write-failed); bounded retries; failed-page report.
4. **Structured records** — per-row company, date, details, page, email.
5. **Separate extraction from qualification** — raw candidates → normalize → verify → score.
6. **Reproducible runs** — run ID, timestamps, per-commodity directories, requested vs completed range.
7. **Robust persistence** — atomic replace or SQLite; checkpoint only after successful writes; auto-resume.
8. **CLI polish** — `--output-dir`, `--timeout`, `--format csv`, validation.
9. **Logging** — Python `logging`, file + console, summary of failed pages.

## Suggested patch: abort on failed start-page jump

```python
else:
    print(f"Could not confirm jump to page {first_page}; aborting.")
    sys.exit(1)
```
(replace the two `current_page = 1` fallbacks).

## Suggested patch: atomic writes

```python
def write_lines(path, lines):
    tmp = path + ".tmp"
    with open(tmp, "w", encoding="utf-8") as f:
        f.writelines(l + "\n" for l in lines)
    os.replace(tmp, path)
```

## Contributing
1. Fork and branch.
2. Keep secrets out of commits.
3. Add tests for extraction changes.
4. Update the relevant doc in `docs/` with any behavior change.
