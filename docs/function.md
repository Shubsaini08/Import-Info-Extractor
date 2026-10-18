# Function Reference

Every function and constant in `key.py`, in file order.

---

## Regex constants

### `COMPLETE_EMAIL_RE`
```python
r"[A-Za-z0-9._%+\-]+@[A-Za-z0-9.\-]+\.[A-Za-z]{2,63}"
```
Practical "looks like an email" pattern. Not RFC-complete. See [EXTRACTION.md](EXTRACTION.md).

### `CANDIDATE_RE`
```python
r"[A-Za-z0-9._%+\-]+\s*@[A-Za-z0-9.\-]*"
```
Permissive: allows an empty domain (`jane@`). The domain class excludes whitespace so matches cannot run across words.

### `NOISE_RE`
```python
r"^\s*@\s*$"
```
Rejects a lone `@`.

---

## `extract_emails(page_text: str) -> Tuple[List[str], List[str]]`

| | |
|---|---|
| **Purpose** | Find complete and incomplete emails in a text blob |
| **Input** | Plain text from BeautifulSoup |
| **Output** | `(complete, incomplete)` — each de-duplicated within the page, first-seen order |

**Logic**
1. Iterate `COMPLETE_EMAIL_RE.finditer`. For each match: strip whitespace and `.,;:()[]{}"'` from the ends, re-check with `fullmatch`, record `(start, end)` span, add to `complete` if its lower-case form is new.
2. Define inner helper `overlaps_complete(start, end)` — true if the interval intersects any recorded complete span (`start < e and end > s`).
3. Iterate `CANDIDATE_RE.finditer`. Skip when it overlaps a complete span, is empty/noise, fully matches the complete pattern, lacks `@`, or has no alphanumeric char before `@`. Otherwise add if new (case-insensitive).

**Edge cases**
- `jane@ or john@example.com` → complete: `john@example.com`; incomplete: `jane@`.
- `ops@example` (no TLD) → incomplete.
- `Sales@X.com` and `sales@x.com` → first spelling kept.
- Trailing `,` in `a@b.com,` is removed from the stored value (span still includes it, harmlessly).

---

## `build_driver(headless: bool = True) -> webdriver.Chrome`

Creates Chrome with the options in [CONFIGURATION.md](CONFIGURATION.md) and a driver installed by `ChromeDriverManager().install()`.

**Failure modes:** Chrome missing, version mismatch, no network to download the driver. Called *outside* `try/finally`.

---

## `login(driver) -> bool`

1. Exit(1) if `USERNAME`/`PASSWORD` empty.
2. Open `LOGIN_URL`; wait ≤15 s for `#login_username`.
3. `clear()` + `send_keys()` both fields; click `.lgin-btn a`.
4. Wait until URL no longer contains `/login` **or** a *visible, non-empty* error element appears.
5. If still on `/login`: print error text, return `False`.
6. Extra ≤15 s wait for URL to leave `/login`; return `True`.

**Inner helper `_visible_error_elements(d)`** — checks `is_displayed()` **and** non-empty text, because Angular renders hidden toast containers up front. Swallows `StaleElementReferenceException`.

**Limitations:** success = "URL left /login" (heuristic); only `TimeoutException` is caught.

---

## `wait_for_results_ready(driver, timeout=DEFAULT_TIMEOUT) -> None`

Waits until a `<table>` or `<pagination-controls>` exists, then tries to wait (≤5 s each) for these to become invisible: `.loading`, `.spinner`, `.overlay`, `.ngx-spinner-overlay`, `[class*='loading']`, `[class*='spinner']`. Spinner timeouts are ignored.

**Raises:** `TimeoutException` if neither table nor pagination appears.
**Caveat:** a table existing ≠ all rows loaded.

---

## `get_pagination_controls(driver)`

Returns the `<pagination-controls>` element or `None`.

---

## `_small_screen_page_ratio(driver, container=None) -> Optional[Tuple[int,int]]`

Reads `li.small-screen` via `execute_script("return arguments[0].textContent;")`, parses `N / TOTAL`. Returns `(current, total)` or `None`.
**Why JS:** the element is CSS-hidden at desktop widths; `.text` would be empty.

---

## `detect_total_pages(driver) -> int`

Priority:
1. `N / TOTAL` ratio.
2. Largest numeric text among `a, span, li, button` inside the container.
3. Regex in container text: `N / TOTAL`, then `page N of TOTAL`.
4. No container: warn, regex over `driver.page_source`.
5. Return `1`.

Always `>= 1`. **Risk:** fallback 2 can understate total when pagination shows a sliding window; fallback 5 can cause a one-page "success".

---

## `get_current_page_indicator(driver) -> str`

Returns `"N/TOTAL"` from the ratio if available; else the first non-empty text from `li.current`, `.current`, `li.active`, `.active`, `[class*='current']`, `[class*='active']`; else `""`. No numeric validation on fallbacks.

---

## `get_page_signature(driver) -> str`

Builds `"{indicator}||{first_row[:200]}||{last_row[:200]}||n={row_count}"`.
Falls back to `body.text[:300]` if no table rows. Used only to detect **change**, not to prove which page is shown.

---

## `get_go_to_page_controls(driver) -> (input_el, go_btn)`

Looks in `div.d-flex.mt-2.ml-2` for `input[style*='width: 50px']` and `span.gotopagebutton`; falls back to page-wide search; returns `(None, None)` if absent.

---

## `click_next_page(driver) -> bool`

Tries five selectors for the next control (scoped to pagination container if present). Skips elements marked disabled (own class, parent class, or `aria-disabled="true"`). Scrolls into view and clicks; on `ElementClickInterceptedException`/stale, retries with JS click. Returns `True` if a click was issued (not that navigation succeeded).

---

## `go_to_page_via_input(driver, page_number) -> bool`

1. Click input; `Ctrl+A`, `Delete`, type digits (real key events; avoids `.clear()` which may not notify Angular).
2. If the value differs, set via JS and dispatch `input` + `change` events.
3. Scroll to Go button; click → JS click → `Enter` fallback.

Returns `True` if an attempt was issued.

---

## `dump_navigation_debug(driver, target_page) -> None`

Writes `debug_nav_fail_page_{N}.html` (pagination outerHTML, go-input outerHTML, full page source) and `debug_nav_fail_page_{N}.png`. **May contain sensitive page content — handle accordingly.**

---

## `advance_to_page(driver, target_page, prev_signature, timeout) -> bool`

1. `click_next_page` → wait up to `max(8, timeout//3)` s for signature change → `wait_for_results_ready` → `True`.
2. Else `go_to_page_via_input(target)` → wait `timeout` s for change → `True`.
3. Else dump debug, return `False`.

---

## `write_lines(path, lines)` / `read_lines(path)`

`write_lines`: opens `"w"` (truncates), one line each. `read_lines`: returns non-blank lines, `[]` on `OSError` (missing and unreadable look identical).

## `read_checkpoint()` / `write_checkpoint(page_number)`

Integer in `last_successful_page.txt`. Read returns `None` on error; write prints a warning on `OSError`.

---

## `scrape_commodity(commodity, headless, start_page, end_page, delay) -> None`

The orchestrator. See [workflow.md](workflow.md) for the step-by-step.

Key behaviors:
- `resuming = bool(start_page and start_page > 1)` → loads prior output, merges.
- Compares checkpoint+1 with `--start-page` and prints a note on mismatch (does not enforce).
- Login failure → `sys.exit(1)`; results timeout → `sys.exit(1)`.
- `first_page = max(1, start_page or 1)`; `last_page = min(total_pages, end_page or total_pages)`.
- If jump to start page cannot be confirmed → `current_page = 1` (**risk**).
- Per page: parse, extract, merge, write, checkpoint — all inside `try/except Exception` (log + continue).
- Navigation failure → `break`.
- `finally: driver.quit()`.

---

## `parse_args()`

Defines `-C/-c/--commodity`, `--headed`, `--start-page`, `--end-page`, `--delay`. `commodity` is declared `required=False` and checked manually so a friendly usage message prints, then `sys.exit(1)`.

## `main()`

`commodity = args.commodity.strip().lower()` then calls `scrape_commodity(...)` with `headless=not args.headed`.
