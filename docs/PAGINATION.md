# Pagination Deep Dive

The site uses Angular's `ngx-pagination`. The script combines reading, navigation, and verification.

## 1. Markup assumptions

```html
<pagination-controls>
  <ul>
    <li class="small-screen"> 7 / 42 </li>        <!-- CSS-hidden on desktop -->
    <li class="current"><span>7</span></li>
    <li class="pagination-next"><a ...>Next</a></li>
    <!-- when disabled: <li class="pagination-next disabled"><span aria-disabled="true">…</span></li> -->
  </ul>
</pagination-controls>

<div class="d-flex mt-2 ml-2">
  <input style="width: 50px">
  <span class="gotopagebutton">Go &gt;</span>
</div>
```

## 2. Reading the page position

| Source | Reliability | Note |
|---|---|---|
| `li.small-screen` `textContent` (`N / TOTAL`) | Highest | Gives both current and total; read via JS because it's hidden |
| Highest numeric link in container | Medium | Can understate total if only a window of links is shown |
| Container text regex | Medium | `N / TOTAL` or `page N of TOTAL` |
| Whole page-source regex | Low | Only if container is missing; logs a warning |
| Default `1` | Fail-soft | May cause a 1-page "successful" run |

## 3. Page signature

```
"{indicator}||{first_row[:200]}||{last_row[:200]}||n={row_count}"
```
- Header row skipped (`rows[1:]`).
- Falls back to `body.text[:300]` without a table.
- Detects *change*, not *correct target*. If a click lands on page 9 instead of 8, the check still passes.

## 4. Navigation strategies

### A. Next arrow (`click_next_page`)
Selector list (first usable wins): `a.pagination-next`, `.pagination-next a`, `.pagination-next`, `a[aria-label='Next']`, `li.pagination-next a`.
Disabled detection: own class, parent class, `aria-disabled`.
Click: scroll into view → normal click → JS click on interception/stale.

### B. Go-to-page (`go_to_page_via_input`)
Select-all + Delete + type digits using real key events so Angular's model updates; JS value + `input`/`change` events as backup; click → JS click → Enter.

### C. Chaining (`advance_to_page`)
A (wait 8 s) → B (wait 25 s) → debug dump → `False`.

## 5. Timing

| Wait | Seconds |
|---|---|
| Next-arrow confirmation | `max(8, 25//3)` = 8 |
| Go-to confirmation | 25 |
| `wait_for_results_ready` after change | 25 |
| `--delay` | user-defined, after success |

## 6. Failure patterns

| Symptom | Likely cause |
|---|---|
| "Detected total pages: 1" on a large result set | Pagination markup changed; fallback hit |
| Stops after N pages with debug files | Both navigation methods failed |
| Starts at page 1 despite `--start-page` | Jump not confirmed → fallback |
| Duplicate counts of 0 new emails for many pages | Navigation not changing content, or same data repeated |
| Stale element exceptions | Angular re-rendered during interaction; usually retried by fallbacks |

## 7. Using the debug artifacts

`debug_nav_fail_page_N.html` first section is the pagination `outerHTML`. Compare to the selectors in [CONFIGURATION.md](CONFIGURATION.md) and update accordingly. The PNG shows overlays/modals blocking clicks. Delete both when done: they may include sensitive page content.

## 8. Recommended hardening

- After navigation, parse the indicator and assert it equals the target page.
- On jump failure, stop (or click Next repeatedly to reach the start page) instead of using page 1.
- Retry transient failures with a bounded budget; log failed pages for manual review.
