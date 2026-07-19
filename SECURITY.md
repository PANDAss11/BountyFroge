# Security Policy

BountyForge is a documentation and reference repository — it does not ship a running service, API, or application binary. That said, several classes of "security issue" are still meaningful in this context, and we take reports about them seriously.

## Scope

We consider the following in scope for this policy:

| Category | In Scope | Notes |
|---|---|---|
| Malicious content injected into the repo | Yes | e.g., a nuclei template or script that would harm a user running it as documented |
| Supply-chain risk in `.github/workflows/` | Yes | Compromised or overly-permissive CI actions |
| Payloads that could cause unintended damage if copy-pasted as-is | Yes | e.g., a "detection" payload that is actually destructive |
| Inaccurate technical guidance | No — use a normal issue | Not a security vulnerability, just open a documentation issue |
| Vulnerabilities in third-party tools we reference (Burp, nuclei, etc.) | No | Report to the upstream maintainer |
| Vulnerabilities discovered *using* content from this repo, in a third party's live system | No | Report directly to that organization or its bug bounty program, not to us |

## Reporting a Vulnerability

Do not open a public GitHub issue for security concerns that could be actively exploited if disclosed publicly (for example, a CI workflow that leaks secrets).

Instead:

1. Email **security@bountyforge-project.example** with a description of the issue, affected file(s), and potential impact.
2. Encrypt sensitive details using our PGP key (fingerprint published at `docs/pgp-key.asc` once GPG signing is configured for the project) if the report involves credential or token exposure.
3. You will receive an acknowledgment within **72 hours**.
4. We aim to provide a remediation timeline within **7 days** of triage.

For anything that is a content-accuracy issue rather than a genuine security risk to users of this repository (e.g., "this CVSS vector looks wrong," "this bypass is outdated"), please open a normal GitHub issue using the Bug Report template instead — that gets it fixed faster and doesn't need the private channel.

## Disclosure Policy

We follow coordinated disclosure:

- Reporter and maintainers agree on a disclosure timeline once the issue is confirmed.
- Default embargo period is 30 days from confirmation, extendable by mutual agreement if a fix requires broader changes (e.g., rotating a compromised CI secret).
- Credit is given to the reporter in the `CHANGELOG.md` unless anonymity is requested.

## Supported Content

Only content on the `main` branch is actively maintained. Forks and archived branches are not covered by this policy.

## Responsible Use Reminder

Everything in this repository — templates, checklists, payloads, and methodologies — is written for use against systems you are explicitly authorized to test, such as your own infrastructure, CTF environments, or targets covered by a bug bounty or penetration testing engagement's written scope. Running payloads or techniques from this repository against systems without authorization is illegal in most jurisdictions under computer misuse statutes. BountyForge maintainers are not responsible for misuse of published content.
