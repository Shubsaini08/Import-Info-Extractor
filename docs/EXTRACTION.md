# Email Extraction Deep Dive

## 1. Pipeline

`driver.page_source` → `BeautifulSoup(..., "html.parser")` → `get_text(separator=" ", strip=True)` → `extract_emails()`.

The separator `" "` prevents adjacent text nodes from fusing (`foo` + `bar@x.com` → `foo bar@x.com`). Structure (rows, columns) is lost.

## 2. Complete regex

```
[A-Za-z0-9._%+\-]+  @  [A-Za-z0-9.\-]+  \.  [A-Za-z]{2,63}
 local part            domain labels       TLD letters
```

Matches: `sales@example.com`, `dispatch+east@example.co.uk`, `freight_ops@example-logistics.com`.
Does not match: quoted local parts, IDN/Unicode addresses, numeric-only TLDs.
May accept invalid strings such as `a..b@x..com` shapes depending on context.

## 3. Why `finditer`

Gives match objects with `.start()`/`.end()`. Offsets power the overlap check.

## 4. Punctuation stripping

```python
email = m.group(0).strip().strip(".,;:()[]{}\"'")
if COMPLETE_EMAIL_RE.fullmatch(email): ...
```
`strip` only affects both ends. It does not repair malformed addresses.

## 5. Case-insensitive de-duplication

```python
if email.lower() not in complete_seen:
    complete_seen.add(email.lower()); complete.append(email)
```
First spelling wins. Dedupe happens twice: per page (in `extract_emails`) and across pages (in `scrape_commodity`).

## 6. Span overlap

Two intervals `[start, end)` and `[s, e)` overlap when:

```
start < e  and  end > s
```
Without it, the permissive regex would re-find every complete email and list it as "incomplete".

## 7. Candidate regex and the whitespace fix

```
[A-Za-z0-9._%+\-]+  \s*  @  [A-Za-z0-9.\-]*
```

- Domain part may be **empty** → catches `jane@`.
- `\s*` before `@` tolerates `jane @x`.
- Domain class **excludes whitespace**. An earlier version allowed whitespace after `@`, so `reach jane@ or call john@example.com` became one long match that overlapped `john@example.com` and was discarded — silently losing `jane@`. The current pattern stops right after `@`, yielding two clean matches.

Side effect: `\s*` before `@` plus the local-part class means `jane smith@example.com` may yield partial candidates depending on match start; overlap with the complete span usually removes them.

## 8. Candidate filter order

1. Overlaps a complete span → reject
2. Empty or `NOISE_RE` → reject
3. Fully matches complete regex → reject (defensive)
4. No `@` → reject
5. No alphanumeric before first `@` → reject
6. Already seen (lower-case) → reject

## 9. What counts as "incomplete"

| Example | Why |
|---|---|
| `jane@` | no domain |
| `ops@example` | no TLD |
| `info@mail.` | trailing dot, no TLD |
| `sales@-bad` | invalid domain shape |

Expect noise: text like `user@` in UI copy, handles (`@twitter` has no local part so is rejected), and truncated cells.

## 10. Limits

- No MX/SMTP/syntax validation → **never treat as verified**.
- Whole-page text may include non-shipment addresses (account/UI/hidden DOM).
- No association with company, shipment, date, or page number.
- Obfuscated forms (`name [at] domain [dot] com`) are not detected.
- HTML entities/`mailto:` attributes are not parsed; only visible-text nodes are scanned.

## 11. Quick test harness

```python
from key import extract_emails

text = "Contact sales@acme.com, or jane@ for rates. ops@acme John@Example.com john@example.com"
complete, incomplete = extract_emails(text)
assert complete == ["sales@acme.com", "John@Example.com"]
assert incomplete == ["jane@", "ops@acme"]
```
