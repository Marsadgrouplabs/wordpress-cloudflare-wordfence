# Wordfence baseline (apply per WordPress install, each one separately)

Merged from the original Wordfence review and settings tested on live sites.
Order matters: update -> real IP -> firewall -> brute force -> rate limiting -> login security -> scan.

Change one thing, save, confirm it stuck after reload. Record old values.
Dropdowns in the Wordfence UI are easy to mis-select (a login-failures value once became 500 by accident): re-read the value after every selection, then again after save + reload.

## 0. Update first
Update Wordfence to the current release from the Plugins page before touching settings. Do not trust version numbers quoted in chats; read the Plugins page.

## 1. Real visitor IP (do this before Rate Limiting)
`All Options > General Wordfence Options > How does Wordfence get IPs`

- Behind Cloudflare: `CF-Connecting-IP`. Compare "Your IP with this setting" with the user's real public IP (curl ifconfig.me from the same machine; mind VPN and IPv4/IPv6). Match = keep.
- Not behind Cloudflare: leave Automatic.
- If the X-Real-IP warning shows and the two values match, keep CF-Connecting-IP.
- If they differ, stop and check the proxy chain with the host. Do not pick a header at random.
- If this is wrong, Rate Limiting can block Cloudflare's own IPs and take the site down.

## 2. Firewall
| Setting | Value |
|---|---|
| Status | Enabled and Protecting. Learning Mode only temporarily during a big change |
| Protection level | Extended Protection. If "Existing WAF Installation Detected": find the other install (same account / parent folder) before pressing Optimize; follow the host wizard; save a settings export first |
| Real-Time IP Blocklist | Premium only; 0% on Free is normal |
| Delay IP and country blocking until after WordPress loads | OFF (troubleshooting only) |
| Allowlisted IP addresses (bypass all rules) | Empty |
| Allowlisted services | Only services actually used (Google, Semrush/Ahrefs-type tools the user uses, uptime monitor in use). Not all of them |
| Immediately block IPs that access these URLs | Empty, unless a repeating attack pattern is seen (see attack-review.md) |
| Ignored IPs for WAF alerting | Empty |
| Rules (SQLi, XSS, traversal...) | Do not disable. For a false positive, allowlist that exact URL + parameter only |
| Allowlisted URLs | Review. Disable unjustified ones one at a time, test, then delete. Keep the legitimate (e.g. Insert Headers and Footers). Never delete all at once |
| Monitor background requests (front-end and admin) | ON |

## 3. Brute force protection
| Setting | Value | Notes |
|---|---|---|
| Enable | ON | |
| Lock out after login failures | **10** for sites with few admins; **20** for sites with many real users (members, customers) | Cloudflare already rate-limits wp-login, so 10 is safe behind it |
| Lock out after forgot-password attempts | **5** | |
| Count failures over | **4 hours** | |
| Lockout duration | **4 hours** | |
| Immediately lock out invalid usernames | **ON** for business sites with a handful of users; **OFF** if customers/members type usernames | Before enabling: list the real usernames and ask whether anyone still logs in with an old/renamed/deleted username (accounts get replaced). One wrong username = the whole IP locked for the lockout duration. After enabling, tell the team the exact usernames |
| Immediately block these usernames | `admin`, `administrator`, `root`, `webmaster`, only if none is a real user | extra layer |
| Prevent use of passwords leaked in breaches | ON, admins and publishers | |
| Enforce strong passwords | ON, admins and publishers | |
| Don't let WordPress reveal valid users in login errors | ON | |
| Prevent users registering "admin" | ON | |
| Prevent discovery of usernames (author scan / REST / oEmbed / sitemap) | ON; if an integration returns 401/403 on `/wp-json/wp/v2/users`, test before keeping | |
| Disable WordPress Application Passwords | **ASK per site.** No integration uses them -> ON. Elementor MCP / connector / Zapier / apps use them or unsure -> OFF and record why | |
| Block POST with blank User-Agent and Referer | OFF until forms, webhooks and CRM posts are tested; enable later only if tests pass | |
| Check password strength on profile update | ON | |
| Participate in Wordfence Security Network | ON | |

## 4. Rate limiting
Enable ON. Match **each field by its label** (Wordfence splits bots/humans and request types) before typing numbers.

| Field | Value |
|---|---|
| Verified Google crawlers | Not rate limited (never "anyone claiming to be Google") |
| If anyone's requests exceed | 120 per minute -> Throttle |
| Crawler's page views exceed | 120 per minute -> Throttle |
| Crawler's 404s exceed | 30 per minute -> Block |
| Human's page views exceed | 120 per minute -> Throttle |
| Human's 404s exceed | 30 per minute -> Block |
| How long an IP is blocked | 30 minutes |
| Allowlisted 404 URLs | Keep defaults (favicon, apple-touch-icon...) |

Before trusting the 404 block: open a few key pages and check there are no broken images/files that would 404 more than a few times per page view. After Elementor edit sessions and MCP jobs, check Live Traffic for blocks.

## 5. Login Security
- 2FA for every administrator (passkeys if the installed version offers them).
- XML-RPC: do not block blindly. If no Jetpack/publishing app/integration uses it, disable XML-RPC authentication; otherwise leave and note.
- Review the Administrators list and remove users that should not exist.

## 6. Scan and alerts
- Run a full scan now; schedule it.
- Alert email goes to the site owner (site-specific, never copied).
- Fix security findings: exposed `debug.log` (block on Cloudflare, delete the file, turn off `WP_DEBUG_LOG`), modified core files (inspect before Repair/Delete).
- **Outdated plugins/themes: note only.** List them in the report under "Notes (not actioned)" with names. Do not update them as part of this job; the user decides separately (backup first, especially Elementor add-ons).
- Pressing HIDE FILE edits `.htaccess`: ask first.
- WooCommerce logs and Elementor CSS changing is usually normal: match timing with the user's work.

## Driving the Wordfence UI through Chrome (what worked)
- Each WordPress install has its own Wordfence: do `example.com` and `sub.example.com` separately (own login, own settings, own scan).
- Pages: Firewall options `admin.php?page=WordfenceWAF&subpage=waf_options`; Global options `page=Wordfence&subpage=global_options`; Login Security `page=WFLS#top#settings`; Scan `page=WordfenceScan`; Live Traffic `page=WordfenceTools&subpage=livetraffic`. Save button is at the top right (about 1382,62 on a 1482-wide frame).
- The page text shows values of selects but NOT checkbox states: screenshot the section (scroll by JS `scrollIntoView` on a label) to read checkboxes.
- Selects are custom lists: click the select, scroll the list, click the option, then zoom to confirm the value before Save.
- Global options: the IP radios are inputs named `wf-howgetIPs` (values `HTTP_CF_CONNECTING_IP`, `REMOTE_ADDR`, ...). `radio.click()` via JS works even while the section is collapsed. Element refs from `find` reset after every navigation; call `find` again.
- After Save: reload and re-read every changed value.
- Lockouts and automatic blocks: Firewall > Blocking > switch on "Show Wordfence Automatic Blocks". The row's Reason says why (e.g. "Used an invalid username 'x' to try to sign in"). To release: tick the row > UNBLOCK > confirm. Prefer this over Allowlisting the IP (an allowlisted IP bypasses every firewall rule, and most visitors' IPs change).
- Users list (`users.php`) shows 2FA status per user; Login Security settings show which roles require 2FA. Never change 2FA requirements for real people without asking: unenrolled admins get locked out when the grace period ends.

## Final state to verify after reload
Firewall Enabled and Protecting + Extended; IP method correct; brute-force values saved; rate-limit values saved; Allowlists as decided; 2FA on; scan done.
