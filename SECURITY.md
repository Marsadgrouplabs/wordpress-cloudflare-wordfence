<p align="center"><b>English</b> · <a href="SECURITY.ar.md">العربية</a></p>

# Security in WordPress + Cloudflare + Wordfence

## Who is the publisher?

[Marsad Labs](https://labs.marsadgroup.co) is the product arm of Marsad Group, a UK-registered marketing agency. [About us](https://labs.marsadgroup.co/about-us/) · [Privacy policy](https://labs.marsadgroup.co/privacy-policy/)

## What does this skill do with your data?

| Question | Answer |
|---|---|
| **What does it read?** | The site settings you give it access to. |
| **What does it change on its own?** | It asks before changing nameservers, SSL Strict or any setting that could lock you out of the site, and before deleting server files. |
| **Passwords** | The skill is not allowed to type passwords. You enter them yourself. |
| **Code** | Text instructions and references only, with no scripts that run on your machine. |

## Scan result

- **Tool:** SkillSpector, an automated scan that does not send the files to any external model.
- **Automated risk score:** 35/100, rated medium.
- **Alerts:** 0 critical, 1 high, 5 medium, 0 low.
- **Main cause of the alerts after manual review:** The alerts come from security rules that mention paths like `/.env` in order to block them, not to read them.
- **Review limits:** we reviewed the critical and high alerts by hand. Scanning reduces risk but is not a full guarantee, so check the permissions before using any skill.

## Verify the file

Every release on the Releases page includes a `SHA256SUMS.txt` file. Compare the hash of your download with the one listed there:

```powershell
Get-FileHash .\wordpress-cloudflare-wordfence.zip -Algorithm SHA256
```

## Found a security problem?

Send it to `Labs@marsadgroup.co` and please do not disclose it publicly until it is fixed.
