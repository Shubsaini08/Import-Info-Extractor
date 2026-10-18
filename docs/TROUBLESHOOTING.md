# Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `ERROR: USERNAME/PASSWORD are not set.` | Env vars not exported | Set `IMPORTKEY_USERNAME`/`IMPORTKEY_PASSWORD` in the same shell; confirm code uses `os.getenv` |
| `Login form elements were not found in time` | Selectors changed, slow site, blocked | Run `--headed`, inspect `#login_username`, `#login_password`, `.lgin-btn a` |
| `Login error message on page: ...` | Wrong credentials / locked account / CAPTCHA | Verify manually in a browser |
| Login "succeeds" but results are empty | URL left `/login` but session not authorized | Check account plan/permissions; run `--headed` |
| `SessionNotCreatedException` / driver mismatch | Chrome and ChromeDriver versions differ | Update Chrome; `pip install -U selenium webdriver-manager` |
| Driver download error / `x-deny-reason` | Network/proxy restrictions | Allow the driver download domains or install a matching driver manually |
| `Timed out waiting for results table / pagination` | Slow load, wrong commodity URL, layout change | Increase `DEFAULT_TIMEOUT`; open the URL manually |
| `Detected total pages: 1` unexpectedly | Pagination markup changed | Inspect `<pagination-controls>`; update selectors |
| Starts at page 1 with `--start-page N` | Jump failed → fallback | Use `--headed`, check Go-to controls; consider patching to abort on failure |
| `Could not confirm navigation to page N` + debug files | Next and Go-to both failed | Open the HTML/PNG; check for overlay, changed selectors, session expiry |
| Many pages show `0 new complete` | Navigation not changing content, or duplicates | Verify page indicator in `--headed` mode |
| Output file empty | No addresses visible in rendered text | Check page in browser; addresses may be hidden/gated/images |
| Crash in container | Sandbox/shm issues | Keep `--no-sandbox` and `--disable-dev-shm-usage`; allocate more shared memory |
| `Error scraping page N` | Exception in parse/write | Fix cause; re-run that page range with `--start-page N --end-page N` |
| Merged results from wrong commodity | Shared filenames | Use a separate directory per commodity |

## Debug recipe

```bash
python3 key.py -C coffee --headed --start-page 1 --end-page 3 --delay 2
```
Watch the browser; compare each `Page X/Y` log with what you see.

## Re-scraping a failed page

```bash
python3 key.py -C coffee --start-page 12 --end-page 12
```
(`start-page > 1` merges with existing files.)
