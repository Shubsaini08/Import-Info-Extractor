# Architecture

## 1. What the program is

A single-file, synchronous, browser-driven scraper. It is **not** an API client, an email verifier, a contact-enrichment tool, or an outreach tool.

## 2. Component map

```
┌────────────────────────────────────────────────────────────────┐
│ CLI layer         main() · parse_args()                        │
├────────────────────────────────────────────────────────────────┤
│ Orchestrator      scrape_commodity()                           │
├──────────────┬──────────────┬───────────────┬──────────────────┤
│ Browser/Auth │ Page parsing │ Pagination    │ Persistence      │
│ build_driver │ BeautifulSoup│ get_pagination│ write_lines      │
│ login        │ extract_     │ _controls     │ read_lines       │
│              │ emails       │ detect_total_ │ read_checkpoint  │
│              │              │ pages         │ write_checkpoint │
│              │              │ get_page_     │                  │
│              │              │ signature     │                  │
│              │              │ advance_to_   │                  │
│              │              │ page ...      │                  │
├──────────────┴──────────────┴───────────────┴──────────────────┤
│ Diagnostics       dump_navigation_debug()                      │
└────────────────────────────────────────────────────────────────┘
```

`scrape_commodity()` coordinates nearly everything; helpers each do one job. Failures fall into separate categories:

| Category | Example | Effect |
|---|---|---|
| Navigation failure | Next arrow selector stops matching | Pages are missed; extraction code is fine |
| Extraction failure | Regex misses an address format | Browser visits all pages but data is incomplete |
| Persistence failure | Disk full | Data lost despite correct scraping |
| Auth failure | Wrong password / changed login form | Run aborts before scraping |

## 3. Dependencies and their role

| Dependency | Role |
|---|---|
| `selenium` | Drives Chrome: find elements, click, type, read DOM |
| `webdriver-manager` | Downloads/locates ChromeDriver |
| `beautifulsoup4` | Parses `driver.page_source`, produces plain text |
| `re` | Email patterns, pagination text parsing |
| `argparse` | CLI |
| `typing` | Type hints |
| `urllib.parse.quote` | URL-encodes commodity |
| `time` | Inter-page delay |
| `sys` | Exit codes |
| `os`, `json` | Imported but **unused** in the supplied code (`os` is needed once you adopt env-var credentials) |

## 4. Data flow

```
Chrome DOM ──page_source──▶ BeautifulSoup ──get_text(" ")──▶ one big string
     │                                                            │
     │                                                  extract_emails()
     │                                                            │
     │                                         (complete[], incomplete[])  per page
     │                                                            │
     │                                          merge into run-wide lists
     │                                          using lower-case "seen" sets
     │                                                            │
     └──────────── next page ◀── advance_to_page() ◀── write files + checkpoint
```

State held in memory for the whole run:

| Variable | Type | Purpose |
|---|---|---|
| `all_complete` | `List[str]` | Ordered complete addresses |
| `all_complete_seen` | `Set[str]` | Lower-case keys for O(1) duplicate checks |
| `all_incomplete` | `List[str]` | Ordered incomplete candidates |
| `all_incomplete_seen` | `Set[str]` | Lower-case keys |

List = order + original casing; set = speed.

## 5. Key design decisions

1. **Real browser, not HTTP** — required for an Angular SPA.
2. **Explicit waits, not `sleep`** — `WebDriverWait` polls conditions.
3. **Scoped selectors** — pagination logic searches inside `<pagination-controls>` to avoid unrelated numbers.
4. **`textContent` over `.text`** for the `N / TOTAL` marker — it is hidden by CSS on wide viewports, and Selenium's `.text` ignores hidden text.
5. **Verify, don't trust clicks** — page signatures confirm the content changed.
6. **Chained navigation** — arrow first, "Go to page" second.
7. **Fail-stop pagination** — if both methods fail, stop instead of looping on the same page.
8. **Persist every page** — limits loss on crashes.
9. **Keyboard events over `.clear()`** — keeps Angular two-way bindings in sync.
10. **Span-overlap check** — avoids double-reporting complete addresses as incomplete.

## 6. Error-handling strategy

| Layer | Mechanism |
|---|---|
| Login | Returns `False` on timeout/error text; caller exits with code 1 |
| Results load | `TimeoutException` → message → `sys.exit(1)` |
| Per-page extraction | Broad `try/except Exception`, logs and **continues** |
| Navigation | Returns `False` → loop `break`s |
| Browser | `try/finally: driver.quit()` |

Caveat: `build_driver()` is called *before* the `try`, so a failure there cannot reach `finally`.

## 7. Concurrency model

None. Single process, single browser, single thread. Throughput is bounded by page load time + `--delay`.

## 8. Extension points

- Replace page-wide text extraction with row-level parsing (see [DEVELOPMENT.md](DEVELOPMENT.md)).
- Swap text files for SQLite/CSV with a `commodity`, `page`, `run_id` column.
- Move constants into a config file or CLI flags.
