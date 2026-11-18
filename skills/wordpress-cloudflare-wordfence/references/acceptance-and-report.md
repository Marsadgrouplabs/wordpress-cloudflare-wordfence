# Site card, acceptance checklist, report

## Site card (fill in phase 0, update as you go)
| Item | Value |
|---|---|
| Domain / canonical | |
| Registrar / DNSSEC at start | |
| Hosting / origin IP / other WP installs on account | |
| Nameservers before / after | |
| Mail provider, MX, smtp target | |
| Subdomains and what they do | |
| Elementor / forms / WooCommerce / payments | |
| Integrations (MCP, REST, OAuth, webhooks, Jetpack, XML-RPC) | |
| Application Passwords decision + reason | |
| Wordfence plan (Free/Premium), version | |
| Real-IP method + matched IP | |
| Brute-force profile used (10 or 20, invalid-username lockout) | |
| Cloudflare settings changed: old -> new | |
| Custom rules deployed (names) | |
| Open follow-ups (cert renewal, DNSSEC, updates, deleted files) | |

## Acceptance checklist
Claude verifies (mark with evidence):
- [ ] DNS for every used service; TXT values complete; smtp = A, DNS Only
- [ ] root/www/needed subdomains open over HTTPS with covering certs, no loop
- [ ] Custom rules return 403 on test paths; legit paths 200; Events show the rule name
- [ ] Wordfence: Enabled and Protecting + Extended, real IP matches, values saved after reload
- [ ] Allowlisted URLs reviewed; no Cloudflare IPs allowlisted
- [ ] Scan run; Administrators list checked
- [ ] Events reviewed after functional tests

User tests (Claude cannot):
- [ ] Mail send and receive (domain -> external and back)
- [ ] Contact form submits and the notification arrives
- [ ] Elementor opens and saves; change visible to visitors
- [ ] Incognito: wp-login passes the challenge and logs in
- [ ] MCP/connector reads and saves a draft (if used); WooCommerce cart/checkout (if used)
- [ ] 2FA enrolled for admins

## Report format (end of the job)
1. Done (grouped: Cloudflare, Wordfence), with old -> new for changed values.
2. Verified by Claude vs left for the user.
3. Open items with owner and date.
4. Anything deliberately not changed and why (e.g. Application Passwords kept for the connector).
5. Notes (not actioned): outdated plugins/themes found by the scan (names and versions), for the user to handle separately.
6. New patterns learned this job, added to `problem-rules-library.md` (or "none").
Then the line: `Skills used: ...`
