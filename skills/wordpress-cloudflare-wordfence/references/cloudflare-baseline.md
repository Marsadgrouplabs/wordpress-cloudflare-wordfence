# Cloudflare Free baseline for a WordPress site

Placeholders: `{{HOSTS}}` = `"example.com" "www.example.com"` (add subdomains only if WordPress runs there). Never copy values from another site.

## 1. Before anything: DNS review (read-only first)
1. Public DNS now: NS, A, www, MX, mail, smtp, SPF, DKIM (selector may not be `default`), DMARC, `_mailchannels`, TXT, SRV.
2. Add zone on Free. Do **not** change Nameservers yet.
3. Compare Cloudflare's scan with the real source (cPanel Zone Editor or current DNS host): Name, Type, Content, MX priority. Read SPF/DKIM/DMARC in full. A name being present does not prove the value is right. The scan misses subdomains and records (sub, `_mailchannels` happened before): add by hand.
4. Proxy status:

| Record | Proxy |
|---|---|
| root + www serving the site | Proxied (after origin cert/IP check) |
| mail, smtp, imap, pop, ftp | DNS Only |
| cpanel, whm, webmail, webdisk, cpcontacts, cpcalendars, autoconfig, autodiscover | DNS Only |
| MX, TXT, SRV, DKIM/verification CNAME | not proxiable; keep values |
| `localhost` leftover | not needed; don't copy automatically |

5. **smtp trap:** a `CNAME smtp -> root` ends on Cloudflare IPs once root is proxied. Replace with `A smtp -> real mail server IP`, DNS Only, no leftover CNAME. Confirm the IP in cPanel.
6. `_mailchannels` TXT only if the host uses it (format: `v=mc1 auth=<host-id>`). Verify against the host's own zone, also after the move: `nslookup -type=TXT _mailchannels.<domain> <old-host-nameserver>` (works without cPanel). If the host answers and Cloudflare doesn't, add the same value.
   - Also check `smtp`/`ftp` CNAMEs by resolving them (`nslookup smtp.<domain> 1.1.1.1`): if they return the real server IP, leave them; replace with `A` only if they resolve to Cloudflare addresses.
7. Sub-subdomains (`www.sub.example.com`): Universal SSL doesn't cover them. Check Edge coverage before proxying.
8. DNSSEC: user checks at registrar. If an old DS exists, disable and remove it, wait for TTL, before Nameservers change.

## 2. SSL/TLS
1. cPanel > SSL/TLS Status: validity, names, chain for every used name.
2. Set **Full (strict)** only after the origin cert covers the names (ask the user before switching). 526 = fix origin cert; Full is a temporary fallback with a date to return to Strict. Never Flexible.
3. Edge: Always Use HTTPS ON, Min TLS 1.2, TLS 1.3 ON, Automatic HTTPS Rewrites ON, Opportunistic Encryption ON, HSTS OFF during onboarding.
4. AutoSSL / wildcard renewal via DNS-01 may fail after DNS moves to Cloudflare: note the renewal date as a follow-up.

## 3. Speed
HTTP/2 + HTTP/3 ON, Early Hints ON, Rocket Loader **OFF**, Speed Brain OFF, Cloudflare Fonts OFF, 0-RTT OFF, Polish/APO/Smart Hints: don't buy or enable. RUM optional. Don't use "enable all available settings".

## 4. Caching
Caching Level **Standard**. API trap: dashboard "Standard" = API value `aggressive`; `basic` = "No Query String"; `simple` = "Ignore Query String". Read the API value through this mapping before calling it drift, Browser Cache TTL: respect origin headers, Crawler Hints ON, Always Online OFF, Dev Mode OFF, Smart Tiered Cache ON if offered, Cache Reserve OFF, Cache Rules/Page Rules: none unless the site already has them (audit existing before changing). No Cache Everything. Stale CSS after Elementor edits: regenerate Elementor CSS, then purge specific URLs, not everything.

## 5. Security settings
Bot Fight Mode **OFF** if any MCP/API/OAuth integration exists (it can't be bypassed on Free). Under Attack OFF, Browser Integrity Check ON, Email Obfuscation ON, Hotlink OFF, AI Labyrinth OFF, Replace Insecure JS Libraries OFF, Leaked Credentials Detection optional (Wordfence handles it).
Network: IPv6 ON, WebSockets ON, gRPC OFF, Pseudo IPv4 Off, IP Geolocation ON. Redirects: no extra HTTP->HTTPS rule; www->root only if canonical, keep path/query, check no loop. Don't buy Argo, Load Balancing, Waiting Room.

## 6. Custom rules (latest versions)
Path: Security > Security rules > Custom rules. Replace `{{HOSTS}}`. Keep to existing rules: update, don't duplicate. WordPress in a subfolder needs different paths.

**Protect WordPress Login** (Managed Challenge)
```
(http.host in {{{HOSTS}}}) and (http.request.uri.path eq "/wp-login.php")
```
Don't challenge all of wp-admin, admin-ajax, wp-cron, wp-json.

**Block Sensitive Files** (Block)
```
(http.host in {{{HOSTS}}}) and (
 (http.request.uri.path contains "/.env") or
 (http.request.uri.path contains "/.git/") or
 (ends_with(http.request.uri.path, "/.git")) or
 (http.request.uri.path contains "/wp-config.php") or
 (ends_with(http.request.uri.path, "/debug.log")) or
 (ends_with(http.request.uri.path, "/error_log")))
```

**Block WP Installer** (Block; only for an already-installed site)
```
(http.host in {{{HOSTS}}}) and (http.request.uri.path eq "/wp-admin/install.php")
```
Catches repeated `install.php?step=1` probes that come through Cloudflare addresses.

**Block Suspicious PHP Probes** (Block; optional by logs, but cheap)
```
(http.host in {{{HOSTS}}}) and (
 (http.request.uri.path in {"/aaa.php" "/2.php" "/root.php" "/menu.php" "/cache.php" "/admin.php" "/class.api.php" "/block-bindings.php"}) or
 (ends_with(http.request.uri.path, "/cache.php") and not starts_with(http.request.uri.path, "/wp-content/") and not starts_with(http.request.uri.path, "/wp-includes/")) or
 (http.user_agent contains "Mozlila") or
 (http.user_agent contains "Moblie Safari") or
 (http.user_agent contains "Bulid/"))
```
No `wp-signup.php` (official path). Check the site doesn't legitimately use these names.

**Rate Limit WordPress Login** (Rate limiting rules, Block)
- Match: host in `{{HOSTS}}` and path eq `/wp-login.php`
- Characteristics: `cf.colo.id` + `ip.src` (what Free accepted); Requests 10; Period 10 seconds; Action Block; Duration 10 seconds (Free's limits; check what is offered today).

## 7. Verify after Active (curl, from this machine)
- `http://root` -> 301 to https; `https://root` -> 200 with `server: cloudflare`; `www` -> one redirect to canonical, no loop.
- `/wp-login.php` -> 403 with `cf-mitigated: challenge`.
- `/.env`, `/dev/.env`, `/.git/config`, `/wp-config.php`, `/wp-content/debug.log`, `/wp-admin/install.php`, `/cache.php` -> 403.
- `/`, `/wp-json/`, `/robots.txt` -> 200. Sitemap 404 is the site's own (check the real sitemap name).
- A 406 from curl on plain http may be the origin reacting to curl's User-Agent: retest with a browser UA before concluding.
- Mail: MX/SPF/DKIM/DMARC visible, smtp = A DNS Only. The local resolver may cache old answers: `ipconfig /flushdns`.
Never print contents of a sensitive file if one unexpectedly returns 200: report the path and fix.
