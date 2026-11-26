---
name: wordpress-cloudflare-wordfence
description: Connect a WordPress site to Cloudflare Free and harden it with Wordfence, end to end and the same way every time. Use when the user wants to onboard a new WordPress domain on Cloudflare (DNS review, SSL, speed, cache, security rules, rate limits), set or audit Wordfence firewall / brute force / rate limiting settings, review attacks in Wordfence and block repeating patterns on Cloudflare and/or Wordfence, or says things like "اربط الموقع بكلاودفلير", "ظبط ووردفينس", "حماية الموقع من البوتات", "راجع الاتاكس".
---

# WordPress + Cloudflare Free + Wordfence

Same baseline for every site, with per-site questions where a blind default would break something.
Detail lives in `references/`; read the file for the phase you are in, not all of them.

## Who does what

- **Claude:** everything inside Cloudflare (Cloudflare API MCP first, Chrome as fallback), reading public DNS, curl tests, Wordfence settings through Chrome once the user is logged in to wp-admin.
- **Official Cloudflare tools (not ours):** the Cloudflare API MCP (the `search` and `execute` tools, or the `plugin:cloudflare:cloudflare` plugin) and Cloudflare's own skills (Workers, Wrangler, Durable Objects, web performance) are published by Cloudflare. Setup: https://developers.cloudflare.com/agent-setup/prompt.md, skills: https://github.com/cloudflare/skills. We use them as they are and don't ship or change them.
- **User:** logins and passwords, 2FA/verification codes, backups, changing Nameservers at the registrar, DNSSEC at the registrar, final functional tests (mail, forms, Elementor save).
- Never type passwords. Ask before: Nameservers change, SSL Strict, any Wordfence change that could lock the user out, deleting files on the server.
- Log every change as `old value -> new value` (goes into the site card, see `references/acceptance-and-report.md`).

## Phase 0: intake (ask first, one message)

Fill the site card. Do not start before these are answered:

1. Domain, canonical (www or not), registrar, hosting (cPanel?), origin IP.
2. Mail on the domain? Where? Subdomains? Other WordPress installs on the same hosting account (this causes Wordfence "Existing WAF Installation Detected").
3. Integrations: Elementor MCP / AI connector, REST/OAuth, Zapier or webhooks, Jetpack, mobile apps, XML-RPC users. **Decides Application Passwords, Bot Fight Mode, XML-RPC.**
4. WooCommerce / payments / membership logins? **Decides login-failure limit and invalid-username lockout.**
5. Wordfence Free or Premium? Does the user have a static IP (almost always: leave Allowlist empty).
6. Cloudflare access: Cloudflare API MCP authorized (`plugin:cloudflare:cloudflare` or the API MCP that exposes `cloudflare.request`)? Otherwise Chrome logged in.

Application Passwords is **optional per site**: ask "does anything connect to this site with an Application Password?". No -> disable in Wordfence. Yes or unsure -> leave allowed and note it in the card.

## Phases

1. **Prepare + DNS (no site impact):** add zone on Free, compare Cloudflare scan against the real DNS (cPanel Zone Editor / public DNS), fix mail records. Read `references/cloudflare-baseline.md` sections 1-2.
2. **Settings + rules while the zone is still Pending:** SSL, speed, cache, security, custom rules, rate limit. Sections 3-6 of the same file. Nothing is live until Nameservers change.
3. **User changes Nameservers** (after backup and DNSSEC check). Wait for Active, then verify with the curl tests in the baseline file.
4. **Wordfence:** update plugin first, then follow `references/wordfence-baseline.md` top to bottom. Real-IP setting before Rate Limiting.
5. **Scan + attack review:** run the Wordfence scan, read the attacks, then `references/attack-review.md`. Check `references/problem-rules-library.md` first (known problem -> known rule). Decide per pattern: Cloudflare, Wordfence, or both; fold new paths into existing rules (Free plan has few custom rules). New pattern solved -> add it to the library. Outdated plugins/themes: **note only**, never update them in this job.
6. **Acceptance + report:** `references/acceptance-and-report.md`. Tell the user what Claude verified and what only the user can test.

## Modes

- **New site:** phases 0-6 above.
- **Audit existing site** (already on Cloudflare/Wordfence, e.g. old zones that drifted): read-only first. Pull the live Cloudflare settings and rules and the Wordfence options, diff them against the baselines, and report differences as `current -> baseline`. Apply changes only after the user approves each group. Never delete a rule or setting that is not in the baseline without asking: it may exist for a reason.
- **Review only:** phase 5 on a site that is already set up (run after day 1, after plugin/integration/rule changes, or when the user asks "راجع الاتاكس").

## Rollback

Keep the old value of everything changed. If something breaks, undo the last change only (disable the last rule or revert the last setting), then retest. Reverting Nameservers is not the first fix for a WAF problem (propagation, DNSSEC, old-zone records). Turning the proxy off can fail if the origin cert is not browser-trusted.

## Hard rules

- Plugin/theme updates are never part of this job: record them as notes.

- Bot Fight Mode OFF when any MCP/API/OAuth integration exists.
- Never allowlist Cloudflare IPs in Wordfence. Never block whole countries because of one IP.
- Do not blanket-block POST, `wp-json`, `wp-admin`, `admin-ajax`, `wp-cron`, `/.well-known/`, or XML-RPC without checking integrations.
- Do not copy IPs, Nameservers, SPF/DKIM/DMARC, `_mailchannels` or cert names from another site. Read them from this site.
- A clean scan or 0 events is evidence, not proof. Say so in the report.
- Do not reproduce contents of exposed secret files (debug.log, .env) in chat.

