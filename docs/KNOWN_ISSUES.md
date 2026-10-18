# Known Issues and Weaknesses

Findings from reading the supplied source — not claims about the live site.

| # | Severity | Issue | Impact | Suggested fix |
|---|---|---|---|---|
| 1 | **Critical** | Hard-coded credentials despite env-var comment | Account exposure | Rotate password; use `os.getenv` |
| 2 | **High** | Failed jump to `--start-page` falls back to page 1 | Wasted run; misleading resume | Abort, or click Next until reaching start; verify indicator |
| 3 | **High** | Page-wide text extraction, no row context | Can't tie email to company/shipment; noise | Parse table rows; store structured records |
| 4 | Medium | Page signature proves change, not correct page | Wrong-page transitions accepted | Assert indicator == target page |
| 5 | Medium | Per-page `except Exception` then continue | Pages silently skipped | Track failed pages; retry; stop or report |
| 6 | Medium | Three non-atomic writes | Inconsistent output/checkpoint after crash | Write to temp + `os.replace`; or SQLite |
| 7 | Medium | Shared filenames across commodities | Mixed data on resume | Per-commodity output dir/filenames |
| 8 | Low–Med | Selectors and pagination logic are markup-specific | Breaks on redesign | Centralize selectors; add smoke test |
| 9 | Low | `detect_total_pages` may understate or return 1 | Partial run reported as complete | Warn loudly; cross-check with indicator |
| 10 | Low | Login success = URL left `/login` | False positives | Check an authenticated-only element |
| 11 | Low | `build_driver` outside `try/finally` | Possible leak on driver failure | Move inside `try` |
| 12 | Low | Unused imports `os`, `json` | Clutter (`os` needed after fix #1) | Remove `json` |
| 13 | Low | No validation of `--start-page`/`--end-page`/`--delay` | Odd inputs accepted | Validate in `parse_args` |
| 14 | Low | Delay applies only after transitions | Not a rate limiter | Add jitter/backoff |
| 15 | Info | Regexes are not validators | False confidence | Label outputs "candidates"; verify separately |
| 16 | Info | Checkpoint is informational | Manual resume required | Auto-resume from checkpoint per commodity |
| 17 | Info | Debug files can hold sensitive data | Data leak | Restrict/delete |
