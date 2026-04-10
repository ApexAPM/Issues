# Data Proxy Encryption — Pre-Implementation Report

## Problem
All `data_proxy.php` responses are plaintext JSON, visible and copy-pasteable in the browser Network tab by any authenticated user.

---

## Affected Endpoints

| `type=` param | Called from |
|---|---|
| `maps` | JS/maps.js |
| `building_floors` | JS/maps.js |
| `entities` | JS/maps.js |
| `ring_data` | JS/maps.js |
| `damage_data` | JS/maps.js |
| `pl_bans` | JS/pl_bans.js |
| `pl_comps` | pl_comps.php |
| `pl_weapons` | pl_weapons.php |
| `lobby_tracker` | JS/lobby_tracker.js |
| `pick_rates` | JS/legends.js |
| `recoil` | JS/recoil.js |
| `weapons` | JS/weapons.js |
| `ttk` | JS/ttk.js |
| `team_comps` | JS/team_comps.js |

---

## Proposed Solution — Per-Request AES-256 Encrypted Payloads

### How it works
1. PHP generates a per-request AES-256-GCM key derived from the session token + a random nonce on every page load
2. The nonce is embedded into the page alongside the existing `window._dataToken`
3. `data_proxy.php` encrypts the JSON payload with the derived key before sending — Network tab shows ciphertext only
4. JS decrypts using the browser-native Web Crypto API (`SubtleCrypto`) in memory before the data is used
5. Scrapers and crawlers hitting the proxy directly get useless encrypted bytes — without the session-bound nonce they cannot decrypt

### Files that need to change

**PHP (server-side)**
- `ALDB/includes/data_token.php` — add key derivation + nonce generation
- `ALDB/includes/data_proxy.php` — encrypt output before echo on every case

**PHP pages (embed nonce into page)**
- `ALDB/maps.php`
- `ALDB/pl_bans.php`
- `ALDB/pl_comps.php`
- `ALDB/pl_weapons.php`
- `ALDB/lobby_tracker.php`
- `ALDB/pl_maps.php`
- `ALDB/pl_lobby.php`
- `ALDB/recoil.php`
- `ALDB/team_comps.php`
- `ALDB/weapons.php`
- `ALDB/ttk.php`

**JavaScript (decrypt before use)**
- `ALDB/JS/maps.js`
- `ALDB/JS/pl_bans.js`
- `ALDB/JS/lobby_tracker.js`
- `ALDB/JS/legends.js`
- `ALDB/JS/recoil.js`
- `ALDB/JS/weapons.js`
- `ALDB/JS/ttk.js`
- `ALDB/JS/team_comps.js`
- `ALDB/pl_comps.php` (inline fetch)
- `ALDB/pl_weapons.php` (inline fetch)

One shared JS decrypt utility function will be added (either as a new file or injected via a shared include).

---

## Known Limitation
A logged-in user who sets a JS breakpoint *after* the in-memory decrypt call can still inspect the decrypted data object. This is a fundamental browser constraint and cannot be solved at the application layer. The encryption stops casual Network tab copy-paste and all automated scrapers/crawlers.