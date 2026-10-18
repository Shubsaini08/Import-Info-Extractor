# Configuration

## Constants in `key.py`

| Constant | Value | Meaning |
|---|---|---|
| `LOGIN_URL` | `https://importkey.com/login` | Login page |
| `RESULTS_URL_TEMPLATE` | `https://importkey.com/result/shipment/{commodity}?domain=usimport` | Results page; `{commodity}` is URL-encoded via `quote()` |
| `USERNAME`, `PASSWORD` | *(literals in the supplied file)* | **Replace with environment variables** |
| `EMAILS_FILE` | `emails.txt` | Complete emails |
| `INCOMPLETE_EMAILS_FILE` | `incomplete_emails.txt` | Incomplete candidates |
| `CHECKPOINT_FILE` | `last_successful_page.txt` | Last page checkpoint |
| `DEFAULT_TIMEOUT` | `25` seconds | Generous explicit-wait timeout |

## Credentials

```python
USERNAME = os.getenv("IMPORTKEY_USERNAME")
PASSWORD = os.getenv("IMPORTKEY_PASSWORD")
```

Optional `.env` workflow (requires `pip install python-dotenv` and two extra lines at the top of `key.py`):

```python
from dotenv import load_dotenv
load_dotenv()
```

```dotenv
# .env  — never commit this file
IMPORTKEY_USERNAME=you@example.com
IMPORTKEY_PASSWORD=change-me
```

```gitignore
.env
emails.txt
incomplete_emails.txt
last_successful_page.txt
debug_nav_fail_page_*
.venv/
```

## Chrome options (in `build_driver`)

| Option | Why |
|---|---|
| `--headless=new` | No visible window (disabled by `--headed`) |
| `--no-sandbox` | Needed in some containers/root environments; weakens isolation |
| `--disable-dev-shm-usage` | Avoids small `/dev/shm` crashes in Docker |
| `--window-size=1366,900` | Desktop layout so desktop pagination renders |
| `user-agent=...Chrome/125...` | Hard-coded UA string; may not match installed Chrome |

## Timeouts and waits

| Where | Duration |
|---|---|
| Login form presence | 15 s |
| Login resolution | 15 s |
| Results ready | 25 s (`DEFAULT_TIMEOUT`) |
| Each spinner-invisibility check | 5 s (×6 selectors) |
| Quick wait after "next" click | `max(8, timeout // 3)` = 8 s |
| Wait after "Go to page" | 25 s |

## Selectors that depend on site markup

If the site changes, these are what break.

| Purpose | Selector |
|---|---|
| Username / password | `#login_username`, `#login_password` |
| Login button | `.lgin-btn a` |
| Login error | `.toast, .alert-danger, .error-message` |
| Pagination wrapper | `<pagination-controls>` |
| Page ratio | `li.small-screen` (text like `7 / 42`) |
| Current page fallbacks | `li.current`, `.current`, `li.active`, `.active` |
| Next arrow | `a.pagination-next`, `.pagination-next a`, `.pagination-next`, `a[aria-label='Next']`, `li.pagination-next a` |
| Go-to container | `div.d-flex.mt-2.ml-2` |
| Go-to input | `input[style*='width: 50px']` |
| Go button | `span.gotopagebutton` |

## Suggested improvements to configuration

- Per-commodity output directory: `out/<commodity>/`.
- CLI flags for timeout, output directory, and file format.
- Derive the user-agent from the installed Chrome version or drop the override.
