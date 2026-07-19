# Frequently Asked Questions

## General

**Is BountyForge affiliated with HackerOne, Bugcrowd, or Intigriti?**
No. BountyForge is an independent, community-maintained reference project. It is not affiliated with, endorsed by, or officially connected to any bug bounty platform.

**Is this repository suitable for beginners?**
Partially. It assumes basic familiarity with HTTP, web application concepts, and command-line tooling. Complete beginners should pair it with a structured learning resource (PortSwigger Web Security Academy is a strong free option) before relying on it as a primary reference.

**Can I use this content commercially, e.g., in a paid training course?**
Yes, under the terms of the MIT License — attribution is appreciated but not strictly required beyond what the license specifies. See `LICENSE`.

## Scope and Ethics

**Can I use these payloads against any website I want to test?**
No. Every payload, checklist, and methodology in this repository assumes you have explicit authorization to test the target — through a bug bounty program's published scope, a signed penetration testing agreement, or infrastructure you own. Testing without authorization is illegal in most jurisdictions regardless of intent.

**Does BountyForge include zero-day exploits?**
No. The project explicitly excludes undisclosed vulnerabilities and working exploits for unpatched CVEs. See `CONTRIBUTING.md` for the full content policy.

**A payload in this repo didn't work against my target. Is it broken?**
Not necessarily. Payload behavior is highly environment-dependent (WAF configuration, framework version, encoding context). Check the "Bypasses" section of the relevant payload file, and confirm you're testing the right injection context (HTML body vs. attribute vs. JavaScript string, for example).

## Using the Templates

**Do I have to fill in every section of a report template?**
Yes, or explicitly mark a section "Not applicable" with a one-line reason. Reviewers and triagers expect the full structure; missing sections read as incomplete work rather than a deliberate choice.

**Can I remove the CVSS section if the program uses its own severity scale?**
Adapt rather than remove — include your CVSS score alongside a note mapping it to the program's scale if they use something nonstandard (e.g., a P1–P5 scale). This preserves a portable, objective reference point.

**The report templates feel long for a simple finding. Can I shorten them?**
For genuinely simple, low-severity findings, brevity in each section is fine — a one-sentence "Attack Scenario" is acceptable if that's all the finding warrants. Keep the section headings, but do not pad content to hit an arbitrary length; equally, do not delete sections just because the finding is simple.

## Tooling

**Do I need Burp Suite Professional, or does Community edition work?**
Most checklists and methodologies work fine with Community edition. Professional adds features referenced in a few places (the full Scanner, Collaborator without the public server's rate limits) — these are called out explicitly where relevant, and manual alternatives are noted.

**Are the nuclei templates safe to run against production systems?**
Most are read-only detection checks, but some involve out-of-band interaction or timing-based tests that can be noisy. Read `nuclei/README.md` and each template's metadata before running it against anything outside a lab environment, and always confirm the target is in your authorized scope.

**Why isn't there a Metasploit module directory?**
BountyForge focuses on web application and API security testing methodology, not exploitation framework modules. Fully weaponized exploitation tooling is out of scope for this project; see `CONTRIBUTING.md` for the reasoning.

## Contributing

**I found an outdated CVE reference or broken link. What do I do?**
Open an issue using the Bug Report template, or submit a pull request directly if the fix is small (updated link, corrected CWE number).

**Can I contribute a checklist for a framework/technology not yet listed?**
Yes — open an issue first describing what you'd like to add so scope can be agreed, then follow the standards in `CONTRIBUTING.md`.

**How do I get maintainer / merge access?**
Sustained, high-quality contributions over time are the path. There's no formal application process; active contributors are invited directly.
