# Import-Info-Extractor

> 📧 **Feeling puzzled? Email us for the code and a complete setup guide:** **[Freightdataus@gmail.com](mailto:Freightdataus@gmail.com)**

**Repository:** https://github.com/Shubsaini08/Import-Info-Extractor

## key.py — ImportKey Shipment Email Scraper

A Selenium-based command-line scraper that logs into **importkey.com**, opens the U.S. import shipment results for a commodity (e.g. `coffee`), walks through every results page, extracts **complete** and **incomplete/broken-looking** email strings from the rendered page, de-duplicates them, and saves progress to disk after every page.

> **Read this first:** the output is a list of *email-like text strings*. It is **not** a verified, deliverable, or company-linked contact list. See [Limitations](#limitations) and [docs/SECURITY.md](docs/SECURITY.md).

---

## Table of contents

1. [Features](#features)
2. [Why Selenium](#why-selenium)
3. [Requirements](#requirements)
4. [Installation](#installation)
5. [Configuration (credentials)](#configuration-credentials)
6. [Usage](#usage)
7. [Command-line reference](#command-line-reference)
8. [Output files](#output-files)
9. [How it works (short version)](#how-it-works-short-version)
10. [Project layout](#project-layout)
11. [Limitations](#limitations)
12. [Responsible use](#responsible-use)
13. [Documentation index](#documentation-index)
14. [Support](#support)
15. [License](#license)

---

## Features

- Logs into ImportKey with a real Chrome browser (handles the Angular single-page app).
- Opens `https://importkey.com/result/shipment/{commodity}?domain=usimport`.
- Auto-detects total page count from the pagination UI (several fallbacks).
- Extracts two classes of strings:
  - **Complete** emails, e.g. `sales@example.com`
  - **Incomplete** candidates, e.g. `jane@` or `ops@example`
- Case-insensitive de-duplication across all pages.
- **Verified page transitions**: confirms the page content actually changed before scraping again.
- Two navigation strategies: the "next" arrow, then the manual "Go to page" box.
- Saves results and a checkpoint **after every page**.
- Resume support via `--start-page` (merges with earlier output).
- Bounded runs via `--end-page`; polite pacing via `--delay`.
- Writes debug HTML + screenshot when navigation fails.

## Why Selenium

importkey.com is a JavaScript-rendered Angular app. `requests.get()` returns an empty shell with no login form or results table. Selenium drives a real browser that executes the JavaScript; BeautifulSoup then parses the fully rendered `driver.page_source`.

## Requirements

| Requirement | Notes |
|---|---|
| Python | 3.8+ (uses `typing` generics such as `List`, `Tuple`) |
| Google Chrome | Installed locally; must match the downloaded driver |
| Internet access | Needed for the site and for `webdriver-manager` to fetch ChromeDriver |
| Python packages | `selenium`, `webdriver-manager`, `beautifulsoup4` |
| An ImportKey account | You must hold valid credentials and be permitted to use them this way |

## Installation

```bash
git clone https://github.com/Shubsaini08/Import-Info-Extractor.git
cd Import-Info-Extractor

python3 -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate

pip install selenium webdriver-manager beautifulsoup4
```

Optional `requirements.txt`:

```text
selenium>=4.10
webdriver-manager>=4.0
beautifulsoup4>=4.12
```

## Configuration (credentials)

The header comment of `key.py` says credentials come from environment variables, **but the supplied version hard-codes them as string literals.** That is a security problem (see [docs/SECURITY.md](docs/SECURITY.md)).

**Required actions**

1. **Rotate the password** that was in the source file. Treat it as exposed.
2. Replace the two assignments in `key.py`:

```python
import os   # already imported

USERNAME = os.getenv("IMPORTKEY_USERNAME")
PASSWORD = os.getenv("IMPORTKEY_PASSWORD")
```

3. Set the variables before running:

```bash
# macOS / Linux
export IMPORTKEY_USERNAME="you@example.com"
export IMPORTKEY_PASSWORD="your-new-password"

# Windows PowerShell
$env:IMPORTKEY_USERNAME = "you@example.com"
$env:IMPORTKEY_PASSWORD = "your-new-password"
```

`login()` already exits with an error if either value is empty, so no other change is required. Full details: [docs/CONFIGURATION.md](docs/CONFIGURATION.md).

## Usage

```bash
python3 key.py -C coffee                          # full run, headless
python3 key.py -c coffee                          # -c is an alias of -C
python3 key.py -C coffee --headed                 # visible Chrome (debugging)
python3 key.py -C coffee --start-page 10 --end-page 50
python3 key.py -C coffee --delay 1.5              # 1.5 s pause between pages
```

Running without `-C/-c` prints a usage hint and exits with status 1.

### Resuming an interrupted run

1. Read `last_successful_page.txt` (say it contains `37`).
2. Re-run with the next page:

```bash
python3 key.py -C coffee --start-page 38
```

Because `--start-page` is greater than 1, the script loads existing `emails.txt` and `incomplete_emails.txt` and **merges** new results into them. The checkpoint is informational only; **you** choose the start page.

> ⚠ Resume uses the same output filenames regardless of commodity. Do not resume `coffee` results with a `tea` run. Use separate working directories per commodity.

## Command-line reference

| Flag | Meaning | Default |
|---|---|---|
| `-C`, `-c`, `--commodity` | Commodity to search (stripped and lower-cased) | *required* |
| `--headed` | Show the Chrome window | headless |
| `--start-page N` | First page to scrape; `N > 1` triggers resume/merge mode | 1 |
| `--end-page N` | Last page to scrape (clamped to detected total) | detected total |
| `--delay SECONDS` | Sleep after each successful page transition | `0.0` |

## Output files

| File | Content |
|---|---|
| `emails.txt` | One complete email per line (accumulated, de-duplicated) |
| `incomplete_emails.txt` | One incomplete candidate per line |
| `last_successful_page.txt` | Last page number for which writes were attempted |
| `debug_nav_fail_page_N.html/.png` | Created only when navigation to page N fails |

Files are **overwritten** on a fresh run. Details: [docs/OUTPUT_FILES.md](docs/OUTPUT_FILES.md).

## How it works (short version)

```
parse args → build Chrome → login → open results URL → wait for table
→ detect total pages → (jump to start page) →
   loop: read page_source → BeautifulSoup text → extract emails
         → merge/dedupe → write files + checkpoint
         → advance to next page (arrow, else Go-to-page) → verify change
→ driver.quit()
```

Deep dives: [docs/workflow.md](docs/workflow.md), [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md), [docs/function.md](docs/function.md).

## Project layout

```
Import-Info-Extractor/
├── key.py
├── emails.txt                   # generated
├── incomplete_emails.txt        # generated
├── last_successful_page.txt     # generated
├── debug_nav_fail_page_*.html   # generated on failure
├── debug_nav_fail_page_*.png    # generated on failure
└── docs/                        # this documentation set
```

## Limitations

- Output is regex-matched text; **not validated** for syntax, domain existence, mailbox existence, or deliverability.
- Whole-page text extraction: no link between an address and the shipment row, company, or date.
- Page may contain addresses unrelated to shipments (account info, navigation, hidden DOM text).
- Failed page jump to `--start-page` silently falls back to page 1.
- Selectors are tied to the site's current markup; a redesign can break login or pagination.
- Outputs are not commodity-specific; checkpoint and files are not written atomically.

Full list with severity: [docs/KNOWN_ISSUES.md](docs/KNOWN_ISSUES.md).

## Responsible use

Automated access and bulk collection of contact data may be restricted by ImportKey's Terms of Service and by laws such as CAN-SPAM, CASL, GDPR/UK-GDPR, and similar regimes. You are responsible for confirming you are allowed to scrape, store, and contact these addresses. See [docs/SECURITY.md](docs/SECURITY.md#compliance-and-responsible-use).

## Documentation index

| Document | Purpose |
|---|---|
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | Components, data flow, design decisions |
| [docs/workflow.md](docs/workflow.md) | Step-by-step runtime flow with diagrams |
| [docs/function.md](docs/function.md) | Every function: signature, purpose, logic, edge cases |
| [docs/EXTRACTION.md](docs/EXTRACTION.md) | Regexes, span-overlap logic, de-duplication |
| [docs/PAGINATION.md](docs/PAGINATION.md) | Page detection, signatures, navigation fallbacks |
| [docs/CONFIGURATION.md](docs/CONFIGURATION.md) | Constants, env vars, browser options |
| [docs/OUTPUT_FILES.md](docs/OUTPUT_FILES.md) | Output formats, checkpoint semantics |
| [docs/SECURITY.md](docs/SECURITY.md) | Credential handling, data protection, compliance |
| [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) | Symptoms, causes, fixes |
| [docs/KNOWN_ISSUES.md](docs/KNOWN_ISSUES.md) | Bugs and weaknesses ranked by severity |
| [docs/DEVELOPMENT.md](docs/DEVELOPMENT.md) | Testing, improvement roadmap, contributing |

## Support

Stuck on installation, Chrome/driver errors, login, or resuming a run? Check [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) first. If you're still puzzled, **email us for the code and a complete setup guide:** [Freightdataus@gmail.com](mailto:Freightdataus@gmail.com). You can also open an issue at https://github.com/Shubsaini08/Import-Info-Extractor/issues (never paste credentials or scraped data into an issue).

## License

Released under the [MIT License](LICENSE).

The license covers this source code only. It grants no rights to ImportKey's website, content, or data, and it does not override ImportKey's Terms of Service or any law that applies to collecting or contacting the addresses you extract.
