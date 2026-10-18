# Security, Privacy and Compliance

## 1. Credentials — critical

The supplied `key.py` contains a username and password as string literals, while its comments claim environment variables are used.

**Consequences**
- Anyone with the file can log in as that account.
- Committing/sharing/pasting the file (including into chats or tickets) exposes it.
- Git history keeps it even after deletion.

**Do now**
1. **Change the ImportKey password** (and any other account where it was reused).
2. Remove literals; use `os.getenv("IMPORTKEY_USERNAME")` / `os.getenv("IMPORTKEY_PASSWORD")`.
3. If the file was ever committed, purge history (`git filter-repo` / BFG) *and* still treat the password as burned.
4. Add `.env` and output files to `.gitignore`.
5. Prefer a secrets manager (OS keychain, 1Password CLI, cloud secret store) for shared or server use.

This documentation set intentionally does not reproduce the credentials.

## 2. Debug artifacts

`debug_nav_fail_page_N.html` contains the full page source and `.png` a screenshot. They may include account details and third-party data. Store securely, never publish, delete after debugging.

## 3. Chrome flags

`--no-sandbox` weakens browser isolation. Only use it where required (some containers); avoid on a normal desktop.

## 4. Data handling

Collected addresses are personal/business contact data.
- Store with restricted permissions (`chmod 600 *.txt`).
- Keep only what you need; define retention.
- Do not share raw outputs broadly.

## 5. Compliance and responsible use

This tool automates access to a subscription website and harvests contact strings. Before use, confirm:

| Area | Check |
|---|---|
| **Terms of Service** | Does ImportKey allow automated access, scraping, bulk export, and your intended use? Many services prohibit it. |
| **Account rights** | Are you authorized to use this account in this way? |
| **Email law** | CAN-SPAM (US), CASL (Canada), PECR/UK-GDPR, GDPR/ePrivacy (EU) restrict unsolicited email, especially to individuals. B2B rules vary by country. |
| **Data protection** | Lawful basis, notice, opt-out and deletion handling for personal data. |
| **Load** | Use `--delay`, avoid parallel runs, respect any rate limits. Note: `--delay` is not permission to scrape. |

If unsure, consult legal counsel or ask the data provider about licensed export/API options, which are also more reliable than scraping.

## 6. Quality caveat

Never send outreach to `incomplete_emails.txt` entries or assume `emails.txt` entries are deliverable. Validate with a proper verification process and honor opt-outs.
