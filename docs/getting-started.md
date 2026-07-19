# Getting Started with BountyForge

This guide walks through setting up BountyForge locally, understanding how the repository is organized, and using it effectively during an engagement or bug bounty program.

## Table of Contents

- [Who This Is For](#who-this-is-for)
- [Prerequisites](#prerequisites)
- [Cloning the Repository](#cloning-the-repository)
- [Repository Layout](#repository-layout)
- [A Typical Workflow](#a-typical-workflow)
- [Using Report Templates](#using-report-templates)
- [Using Checklists](#using-checklists)
- [Using Payload References](#using-payload-references)
- [Integrating with Burp Suite](#integrating-with-burp-suite)
- [Integrating with nuclei](#integrating-with-nuclei)
- [Next Steps](#next-steps)

## Who This Is For

BountyForge is written for:

- Bug bounty hunters submitting to programs on HackerOne, Bugcrowd, Intigriti, or private VDPs.
- Penetration testers who need consistent, client-ready report structures.
- Application security engineers building internal testing checklists.
- Security researchers learning a structured methodology for a new vulnerability class.

It assumes working knowledge of HTTP, basic web application architecture, and command-line tooling. It does not teach programming or networking fundamentals from scratch.

## Prerequisites

To get the most out of the repository day-to-day, have the following available:

- A current version of [Burp Suite](https://portswigger.net/burp) (Community or Professional) or an equivalent intercepting proxy.
- [nuclei](https://github.com/projectdiscovery/nuclei) if you intend to use the template pack under `nuclei/`.
- A Markdown-capable editor (VS Code, Obsidian, or similar) for filling out report templates.
- Git, for cloning and keeping the repository up to date.

## Cloning the Repository

```bash
git clone https://github.com/bountyforge/BountyForge.git
cd BountyForge
```

To stay current with new templates and checklist updates, pull periodically:

```bash
git pull origin main
```

If you maintain local customizations to templates (e.g., your own report header with your handle and contact details), keep them in a separate branch or a personal fork so upstream pulls do not conflict:

```bash
git checkout -b my-customizations
# edit templates
git add . && git commit -m "chore: personalize report header"
```

## Repository Layout

```
BountyForge/
├── report-templates/   # Fill-in-the-blank templates per vulnerability class
├── checklists/          # Manual + automated test checklists per testing category
├── methodology/         # End-to-end testing workflows (recon through reporting)
├── payloads/             # Annotated payload collections with detection guidance
├── references/          # Curated external references (RFCs, advisories, talks)
├── examples/              # Fully worked example reports (fictional companies)
├── nuclei/                 # Custom nuclei templates aligned with checklist content
├── burp/                    # Burp Suite configuration and extension notes
└── docs/                     # This documentation set
```

## A Typical Workflow

1. **Scope review.** Read the program's scope and rules of engagement before touching anything else.
2. **Recon.** Follow `methodology/recon.md` to build an asset inventory.
3. **Checklist pass.** Pick the relevant checklist(s) under `checklists/` for the technology stack in scope (e.g., `api-security.md` for a REST API, `authentication.md` for a login flow).
4. **Deep dive.** When a checklist item flags something interesting, pull the matching methodology document for a structured approach, and the matching payload reference under `payloads/` for concrete test strings.
5. **Validate.** Confirm the finding is reproducible and not a false positive before writing it up.
6. **Report.** Copy the relevant file from `report-templates/` and fill it in completely. Use `examples/` as a reference for tone and level of detail.
7. **Submit.** Follow the program's disclosure process. Never publish details of an unresolved finding publicly.

## Using Report Templates

Each file under `report-templates/` is a Markdown skeleton with every section a mature bug bounty program expects: severity, CVSS, CWE, OWASP mapping, reproduction steps, and remediation guidance.

```bash
cp report-templates/xss.md my-reports/acme-corp-stored-xss.md
```

Fill in every section — do not delete sections you think are optional. If a section genuinely does not apply (for example, "Preconditions" for an unauthenticated finding), write "None" rather than removing the heading; reviewers on the other end expect a consistent structure.

## Using Checklists

Checklists are meant to be worked through linearly during testing, not read once. Treat each `- [ ]` item as a literal task:

```markdown
- [ ] Test password reset token for predictability
- [ ] Confirm token single-use enforcement
- [ ] Confirm token expiry window
```

Copy the checklist into your engagement notes and check items off as you go so you have an audit trail of what was actually tested versus assumed safe.

## Using Payload References

Payload files under `payloads/` are organized by vulnerability class and always include **detection guidance** — how to tell whether a payload succeeded, not just the payload string itself. Read the "Detection" and "Expected Behaviour" sections before firing payloads at a target; blind payload spraying against production systems is poor practice and can violate program rules.

## Integrating with Burp Suite

See `burp/README.md` for:

- Recommended extensions per testing category.
- Match/replace rules referenced by several checklists (e.g., for testing authorization by swapping session tokens).
- Suggested project configuration for large-scope engagements.

## Integrating with nuclei

See `nuclei/README.md` for template usage:

```bash
nuclei -u https://target.example -t nuclei/ -severity medium,high,critical
```

> **Warning**
> Only run nuclei templates against hosts explicitly covered by your engagement scope. Some templates are intentionally noisy (timing-based checks, out-of-band interaction probes) and can trigger WAF alerting or rate-limit bans if scope is misjudged.

## Next Steps

- Read [`methodology.md`](methodology.md) for the philosophy behind how testing workflows in this repo are structured.
- Read [`reporting-guide.md`](reporting-guide.md) before writing your first report — small formatting choices materially affect how fast triage teams respond.
- Skim the [`faq.md`](faq.md) for answers to common setup and scoping questions.
