# Reviewing attacks and deciding where to block

Do this after the rules have been live for a day or more, and again after plugin/integration/rule changes. Goal: block harmful requests while the site works, not a zero counter.

## Where to look
- Wordfence: Dashboard recent attacks, Firewall > Blocking, Live Traffic (if enabled), Scan results. Record time, Host, URI, Method, Action, Reason/Rule, IP, User-Agent.
- Cloudflare: Security > Events (Free shows samples; a missing row doesn't prove nothing passed). Look only at the period after the deploy; older events predate the rules.

## Reading signals correctly
200 on wp-login.php is only an HTTP response, not a login. 403 = refused by some layer: find which. 503 from Wordfence may be a block: open the Reason. "Human" label in Wordfence can be a spoofed User-Agent. A foreign IP alone is not an attack. admin-ajax/wp-cron/POST volume is often normal. `server.empty` alone is not a compromise.

## Decision table
| Pattern seen | Action |
|---|---|
| Same sensitive/probe paths from many IPs (`/.env`, `/.git`, `/cache.php`, odd `.php` names) | Cloudflare custom rule (add to the existing rule, not a new one) |
| Garbled User-Agent strings (`Mozlila`, `Bulid`, `Moblie`) across probes | Cloudflare UA condition in the probes rule |
| `install.php?step=1` on a schedule, even from Cloudflare addresses | Block WP Installer (already in baseline) |
| One noisy IP hammering 404s | Leave to Wordfence 404 rate limit; don't write a rule per IP |
| Scan finds a public log/backup file | Cloudflare block for that filename + delete the file on the server + disable its cause (`WP_DEBUG_LOG`). Hide-file in Wordfence only after asking |
| Repeated hits on one specific URL that does not exist and is not a plugin path | Wordfence "Immediately block IPs that access these URLs" (honeypot) for that URL |
| Same pattern also shows in Wordfence after Cloudflare | Normal for requests that pass Cloudflare; consider both layers. If direct-to-origin access is suspected, ask the host to review origin protection; don't lock the whole server to Cloudflare IPs (mail and other sites) |
| Legit request blocked (form, Elementor, MCP) | Fix the specific reason: allowlist that URL+param in Wordfence, or narrow the Cloudflare condition. Never disable whole rule families |

## Guardrails
- Check the pattern is not used by the site before blocking (search plugins, sitemap, forms).
- Free plan has few custom rules: merge, don't multiply. Avoid endless per-name lists.
- Don't block countries for one IP; don't allowlist Cloudflare IPs in Wordfence.
- After a new or changed rule: test a bad path (expect 403) and legit paths `/`, `/wp-json/`, `/robots.txt` (expect 200); then re-check Events.
- Report what was found, what was blocked where, and what was deliberately left alone.
- Before writing a new rule, look the pattern up in `problem-rules-library.md`. After solving a new one, add it there (see that file's last section).
