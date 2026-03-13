# Security Audit — apexapm.com / apexkpm.net (excluding ALDB)

**Date:** 2026-03-13  
**Scope:** ALGSIntel, Assets, CC, Pro-League, Qualifiers, Website, Partners  
**Auditor:** GitHub Copilot

---

## CRITICAL

### ~~CRIT-1~~ RESOLVED — Hardcoded database credentials in source control
**Files:**
- `ALGSIntel/includes/database.php` — RESOLVED
- `CC/includes/database.php` — RESOLVED
- `Pro-League/includes/database.php` — RESOLVED
- `Qualifiers/includes/database.php` — RESOLVED

All four files now read credentials from `DB_USER`/`DB_PASS` environment variables (with `DB_HOST`/`DB_NAME` also env-configurable). Server returns 500 and exits if the vars are absent. The credentials `Dad_Is_Bored`/`boredisdad` should be changed on the server, and `SetEnv DB_USER` / `SetEnv DB_PASS` directives added to each VirtualHost config.

---

### CRIT-2 — ~~Discord OAuth client secret hardcoded in source control~~ RESOLVED
**File:** `ALGSIntel/admin/discord-config.php`

Fixed. Hardcoded fallback values removed. Credentials are now read exclusively from `DISCORD_CLIENT_ID` and `DISCORD_CLIENT_SECRET` environment variables. Server will return 500 and refuse to start if either is absent. The exposed secret must still be rotated in the Discord Developer Portal.

---

### CRIT-3 — ~~Session fixation on admin login~~ RESOLVED
**File:** `ALGSIntel/admin/discord-callback.php`

Fixed. `session_regenerate_id(true)` is now called before any session variables are written on successful authentication.

---

## HIGH

### HIGH-1 — ~~XSS via `addslashes()` in inline JS event handlers (dashboard)~~ RESOLVED
**File:** `ALGSIntel/admin/dashboard.php`

Fixed. All five Delete button inline handlers now use `json_encode()` instead of `addslashes()` for correct JS string escaping.

---

### ~~HIGH-2~~ RESOLVED — XSS via `addslashes(htmlspecialchars())` in inline JS (profile pages)
**Files:**
- `CC/profile.php` (was line 645)
- `Pro-League/profile.php` (was line 635)
- `Qualifiers/profile.php` (was line 634)

**Issue:** Player name from the database is embedded into a JavaScript string literal using `addslashes(htmlspecialchars())`:

```php
link.download = '<?= addslashes(htmlspecialchars($playerName)) ?>-profile.png';
```

`htmlspecialchars()` is the correct function for HTML context but the wrong one for a JS string literal. A name containing a single quote followed by JS code would not be correctly neutralised.

**Required fix:** Use `json_encode($playerName . '-profile.png')` — no surrounding quotes:

```php
link.download = <?= json_encode($playerName . '-profile.png') ?>;
```

---

### HIGH-3 — ~~No CSRF protection on admin delete action~~ RESOLVED
**Files:** `ALGSIntel/admin/dashboard.php`, `ALGSIntel/admin/api.php`

Fixed. A per-session CSRF token (`bin2hex(random_bytes(32))`) is generated in `dashboard.php` on first load and stored in `$_SESSION['csrf_token']`. It is appended to every delete URL and verified server-side in `api.php` using `hash_equals()` before any deletion proceeds.

---

### HIGH-4 — ~~`tweetLink` field accepts any URL scheme (stored XSS / javascript: URI)~~ RESOLVED
**File:** `ALGSIntel/admin/api.php`

Fixed. Both the `add` and `edit` cases now validate that `tweetLink` matches `^https?://` before passing it to the data layer. Any non-HTTP value is rejected with an error redirect.

---

## MEDIUM

### MED-1 — ~~Missing security headers on CC, Pro-League, Qualifiers, ALGSIntel~~ RESOLVED
**Files:** `CC/.htaccess`, `Pro-League/.htaccess`, `Qualifiers/.htaccess`, `ALGSIntel/.htaccess` (created)

Fixed. Added `X-Content-Type-Options`, `X-Frame-Options`, `X-XSS-Protection`, and `Referrer-Policy` headers to all four sites. `ALGSIntel/.htaccess` was created as it did not previously exist.

---

### MED-2 — ~~Access token stored in session~~ RESOLVED
**File:** `ALGSIntel/admin/discord-callback.php`

Fixed. The `$_SESSION['discord_access_token']` and `$_SESSION['discord_token_expires']` lines have been removed. This also resolves MED-4 as the expiry tracking was only relevant to the now-removed token.

---

### MED-3 — ~~SQL schema files committed to repository~~ DISMISSED
All ApexAPM repositories are private. Schema files are intentionally version-controlled for use in migration/DDL scripting and are never served by the web server. No action required.

---

### MED-4 — ~~Token expiry tracked but never enforced~~ RESOLVED
**File:** `ALGSIntel/admin/discord-callback.php`

Resolved as a consequence of MED-2. Both the token and its expiry timestamp have been removed from the session.

---

## LOW

### LOW-1 — ~~Client-side only game logic in Partners/adiuvant~~ RESOLVED (accepted risk)
**Files:** `Partners/adiuvant/js/slot-machine.js`, `Partners/adiuvant/js/blackjack.js`

Entertainment-only with no monetary or account implications. No server component warranted. Accepted as low risk.

---

### LOW-2 — ~~Error message leaks "Please verify your credentials and try again"~~ RESOLVED
**Files:** All four `includes/database.php` files

Fixed. PDO catch block now echoes `Service temporarily unavailable.` in all four files.

---

## Summary Table

| ID | Severity | File(s) | Issue |
|----|----------|---------|-------|
| CRIT-1 | ~~Critical~~ RESOLVED | 4x `includes/database.php` | Hardcoded DB credentials |
| CRIT-2 | ~~Critical~~ RESOLVED | `ALGSIntel/admin/discord-config.php` | Hardcoded OAuth secret — rotate secret in Discord portal |
| CRIT-3 | ~~Critical~~ RESOLVED | `ALGSIntel/admin/discord-callback.php` | Session fixation |
| HIGH-1 | ~~High~~ RESOLVED | `ALGSIntel/admin/dashboard.php` | XSS via `addslashes()` in JS event handler |
| HIGH-2 | ~~High~~ RESOLVED | 3x `profile.php` | XSS via `addslashes(htmlspecialchars())` in JS string |
| HIGH-3 | ~~High~~ RESOLVED | `ALGSIntel/admin/dashboard.php` + `api.php` | No CSRF protection on delete |
| HIGH-4 | ~~High~~ RESOLVED | `ALGSIntel/admin/api.php` | No URL scheme validation on `tweetLink` |
| MED-1 | ~~Medium~~ RESOLVED | `CC/`, `Pro-League/`, `Qualifiers/`, `ALGSIntel/` `.htaccess` | Missing security headers |
| MED-2 | ~~Medium~~ RESOLVED | `ALGSIntel/admin/discord-callback.php` | OAuth access token stored in session unnecessarily |
| MED-3 | ~~Medium~~ DISMISSED | 4x `.sql` files | Private repo — non-issue |
| MED-4 | ~~Medium~~ RESOLVED | `ALGSIntel/admin/discord-callback.php` | Token expiry stored but never enforced (resolved via MED-2) |
| LOW-1 | ~~Low~~ RESOLVED | `Partners/adiuvant/js/` | Client-side-only game logic (accepted risk) |
| LOW-2 | ~~Low~~ RESOLVED | 4x `includes/database.php` | PDO error leaks credential hint |
