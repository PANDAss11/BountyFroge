# Curated References

External references, organized by category, that inform and supplement the content in this repository. Prefer these primary/authoritative sources over secondary blog posts when researching a topic further.

## Table of Contents

- [Standards and Classification](#standards-and-classification)
- [Cheat Sheets](#cheat-sheets)
- [Learning Platforms](#learning-platforms)
- [Tooling Documentation](#tooling-documentation)
- [Disclosed Report Archives](#disclosed-report-archives)
- [Wordlists](#wordlists)

## Standards and Classification

| Resource | Use |
|---|---|
| [OWASP Top 10](https://owasp.org/www-project-top-ten/) | Web application risk category reference used throughout `report-templates/` |
| [OWASP API Security Top 10](https://owasp.org/API-Security/editions/2023/en/0x00-header/) | API-specific risk categories, primary reference for `checklists/api-security.md` |
| [CWE (Common Weakness Enumeration)](https://cwe.mitre.org/) | Vulnerability classification used in every report template |
| [CVSS v3.1 Specification](https://www.first.org/cvss/v3.1/specification-document) | Scoring methodology reference for the Severity section of every report template |
| [NIST SP 800-63B](https://pages.nist.gov/800-63-3/sp800-63b.html) | Authoritative digital identity/authentication guidance |

## Cheat Sheets

| Resource | Use |
|---|---|
| [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/) | Prevention guidance referenced in nearly every report template's Remediation section |
| [PortSwigger Web Security Academy](https://portswigger.net/web-security) | Interactive labs and reference material per vulnerability class |
| [HackTricks](https://book.hacktricks.xyz/) | Broad technique reference, useful for cross-checking edge cases |

## Learning Platforms

| Resource | Use |
|---|---|
| PortSwigger Web Security Academy | Free, hands-on labs for nearly every vulnerability class covered in this repository |
| PentesterLab | Paid, deeper exercises including source-code-review-driven exploitation |
| TryHackMe / HackTheBox | Broader offensive security skill-building, less API/web-bounty-specific |

## Tooling Documentation

| Tool | Documentation |
|---|---|
| Burp Suite | https://portswigger.net/burp/documentation |
| nuclei | https://docs.projectdiscovery.io/tools/nuclei/overview |
| ffuf | https://github.com/ffuf/ffuf |
| sqlmap | https://github.com/sqlmapproject/sqlmap/wiki |
| jwt_tool | https://github.com/ticarpi/jwt_tool |

## Disclosed Report Archives

For studying real-world report writing and severity reasoning (never copy content directly — see `CONTRIBUTING.md`):

- [HackerOne Hacktivity](https://hackerone.com/hacktivity) — publicly disclosed reports across many programs
- [Bugcrowd Disclosure Program listings](https://bugcrowd.com/programs) — where programs opt into public disclosure

## Wordlists

| Resource | Use |
|---|---|
| [SecLists](https://github.com/danielmiessler/SecLists) | The canonical wordlist collection for discovery, fuzzing, and payload testing referenced throughout `checklists/` |
| [Assetnote Wordlists](https://wordlists.assetnote.io/) | Continuously regenerated wordlists derived from internet-wide scanning, useful for content discovery |
