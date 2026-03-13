# Security Audit — apexapm.com / apexkpm.net (excluding ALDB)

**Date:** 2026-03-13  
**Scope:** ALGSIntel, Assets, CC, Pro-League, Qualifiers, Website, Partners  
**Auditor:** GitHub Copilot

---

## CRITICAL

### CRIT-1 — Hardcoded database credentials in source control
**Files:**
- `ALGSIntel/includes/database.php` lines 17-19
- `CC/includes/database.php` lines 17-19
- `Pro-League/includes/database.php` lines 17-19
- `Qualifiers/includes/database.php` lines 17-19

**Issue:** Username `Dad_Is_Bored` and password `boredisdad` are hardcoded in plain text. The same credential pair is shared across all four databases. Anyone with repository access has full database access.

**Required fix:** Remove the hardcoded values. Read credentials from environment variables (`DB_USER`, `DB_PASS`) set in the server's Apache `VirtualHost` config (`SetEnv`) or PHP-FPM pool config. Fail hard at boot if the env vars are not set rather than falling back to plaintext.

---

### CRIT-2 — Discord OAuth client secret hardcoded in source control
**File:** `ALGSIntel/admin/discord-config.php` lines 15-16

**Issue:** `DISCORD_CLIENT_SECRET` has a hardcoded fallback value (`5UFr5XvZECgRbwhCt_PXymJNPpQmYU_P`). If `$_ENV['DISCORD_CLIENT_SECRET']` is not set, the real secret is used. Anyone with repository access can impersonate the Discord OAuth application, forge OAuth flows, and gain admin access to the LFT dashboard.

```php
// current — falls back to the real secret
define('DISCORD_CLIENT_SECRET', $_ENV['DISCORD_CLIENT_SECRET'] ?? '5UFr5XvZECgRbwhCt_PXymJNPpQmYU_P');
```

**Required fix:** Remove the hardcoded fallback entirely. Fail hard if the env var is absent. Rotate the secret in the Discord Developer Portal immediately — it has already been committed and must be considered compromised.

---

### CRIT-3 — Session fixation on admin login
**File:** `ALGSIntel/admin/discord-callback.php` around line 46

**Issue:** After a successful OAuth authentication, the session is marked as authenticated (`$_SESSION['lft_admin_logged_in'] = true`) but `session_regenerate_id(true)` is never called. An attacker who can fix a victim's session ID before login retains that session ID and inherits the admin session post-login.

**Required fix:** Call `session_regenerate_id(true)` immediately before writing any `$_SESSION` values on successful login.

---

## HIGH

### HIGH-1 — XSS via `addslashes()` in inline JS event handlers (dashboard)
**File:** `ALGSIntel/admin/dashboard.php` lines 435, 496, 572, 635, 700

**Issue:** All five Delete buttons use `addslashes()` to embed database values into a JS string literal inside an HTML `onclick` attribute:

```php
onclick="confirmDelete('coach', <?= $coach['id'] ?>, '<?= addslashes($coach['name']) ?>')"
```

`addslashes()` only escapes `'`, `"`, `\`, and NUL. It does not prevent HTML injection. A name value containing `</button><script>alert(1)</script>` would break out of the attribute context. Although input is stored by an admin, it is still a stored XSS vector.

**Required fix:** Replace `addslashes($x)` with `json_encode($x)` (no surrounding quotes — `json_encode` adds them). This correctly handles all JavaScript string escaping in all contexts.

```php
// correct
onclick="confirmDelete('coach', <?= $coach['id'] ?>, <?= json_encode($coach['name']) ?>)"
```

---

### HIGH-2 — XSS via `addslashes(htmlspecialchars())` in inline JS (profile pages)
**Files:**
- `CC/profile.php` line 645
- `Pro-League/profile.php` line 635
- `Qualifiers/profile.php` line 634

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

### HIGH-3 — No CSRF protection on admin delete action
**Files:** `ALGSIntel/admin/dashboard.php` (delete buttons), `ALGSIntel/admin/api.php` (delete case)

**Issue:** Delete is triggered via a plain GET request: `api.php?action=delete&type=coach&id=1`. There is no CSRF token. Any page the admin visits while authenticated can silently issue this GET request and delete entries.

```php
// dashboard.php — no token in the URL
window.location.href = `api.php?action=delete&type=${type}&id=${id}`;

// api.php — no token verified
case 'delete':
    $id = (int) ($_GET['id'] ?? 0);
    // proceeds directly to deletion
```

**Required fix:** Generate a CSRF token on session start, store it in `$_SESSION['csrf_token']`, include it as a hidden field in all forms and as a query parameter in delete links, and verify it server-side before any state-changing action.

---

### HIGH-4 — `tweetLink` field accepts any URL scheme (stored XSS / javascript: URI)
**Files:** `ALGSIntel/admin/api.php` (add and edit cases), `ALGSIntel/includes/lft-data.php`

**Issue:** The `tweetLink` field accepts any string. There is no server-side validation that the value is an `http://` or `https://` URL before storage. The output in both `ALGSIntel/index.php` and `ALGSIntel/admin/dashboard.php` renders it as an `<a href="...">` only if `preg_match('/^https?:\/\//i', ...)` passes, but this check is applied at render time, not at write time. A non-HTTP value can be stored and then if the render-time check is ever removed or bypassed, a `javascript:` URI could execute.

**Required fix:** Validate that `tweetLink` matches `^https?://` in `api.php` before passing it to the data layer, and reject the request if it does not.

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
| CRIT-1 | Critical | 4x `includes/database.php` | Hardcoded DB credentials |
| CRIT-2 | Critical | `ALGSIntel/admin/discord-config.php` | Hardcoded OAuth secret with fallback |
| CRIT-3 | Critical | `ALGSIntel/admin/discord-callback.php` | Session fixation — no `session_regenerate_id()` |
| HIGH-1 | High | `ALGSIntel/admin/dashboard.php` | XSS via `addslashes()` in JS event handler (5 instances) |
| HIGH-2 | High | 3x `profile.php` | XSS via `addslashes(htmlspecialchars())` in JS string |
| HIGH-3 | High | `ALGSIntel/admin/dashboard.php` + `api.php` | No CSRF protection on delete |
| HIGH-4 | High | `ALGSIntel/admin/api.php` | No URL scheme validation on `tweetLink` |
| MED-1 | ~~Medium~~ RESOLVED | `CC/`, `Pro-League/`, `Qualifiers/`, `ALGSIntel/` `.htaccess` | Missing security headers |
| MED-2 | ~~Medium~~ RESOLVED | `ALGSIntel/admin/discord-callback.php` | OAuth access token stored in session unnecessarily |
| MED-3 | ~~Medium~~ DISMISSED | 4x `.sql` files | Private repo — non-issue |
| MED-4 | ~~Medium~~ RESOLVED | `ALGSIntel/admin/discord-callback.php` | Token expiry stored but never enforced (resolved via MED-2) |
| LOW-1 | ~~Low~~ RESOLVED | `Partners/adiuvant/js/` | Client-side-only game logic (accepted risk) |
| LOW-2 | ~~Low~~ RESOLVED | 4x `includes/database.php` | PDO error leaks credential hint |
