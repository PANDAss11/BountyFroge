# Reconnaissance Methodology

## Table of Contents

- [Objectives](#objectives)
- [Tools](#tools)
- [Manual Workflow](#manual-workflow)
- [Automation](#automation)
- [Reporting](#reporting)
- [Best Practices](#best-practices)
- [Common Mistakes](#common-mistakes)
- [Example](#example)

## Objectives

Reconnaissance exists to answer one question completely before any exploitation testing begins: **what is actually in scope, and what does it look like from the outside?** A thorough recon phase produces:

1. A complete list of in-scope hosts (domains, subdomains, IP ranges).
2. A technology fingerprint per host (framework, server software, CDN/WAF presence).
3. A mapped attack surface per application (endpoints, parameters, authentication boundaries).
4. A prioritized target list, ranking assets by apparent sensitivity and likely testing yield.

Skipping or rushing this phase is the single most common reason experienced testers miss high-value findings — the bug is often on the forgotten staging subdomain, not the polished flagship application.

## Tools

| Purpose | Tool | Notes |
|---|---|---|
| Passive subdomain enumeration | `subfinder`, `crt.sh` | No direct interaction with target infrastructure |
| Active subdomain enumeration | `amass enum -active`, `assetfinder` | Generates DNS traffic against target-adjacent infrastructure |
| Live host probing | `httpx` | Fast, supports tech-detection and title extraction |
| Port scanning | `naabu`, `nmap` | `naabu` for speed across large ranges, `nmap` for deep service fingerprinting on a filtered set |
| Screenshotting | `gowitness`, `aquatone` | Enables rapid visual triage across hundreds of hosts |
| Content discovery | `ffuf`, `feroxbuster` | Directory/file brute-forcing |
| JS analysis | `katana`, manual review | Endpoint and secret extraction from client-side bundles |
| Wordlists | SecLists (`Discovery/`, `DNS/`) | Curated, actively maintained wordlist collection |

## Manual Workflow

1. **Scope confirmation.** Re-read the program scope. Build an explicit include/exclude list before running any tool.
2. **Organizational mapping.** Identify parent company, subsidiaries, and known acquisitions — these frequently extend scope under wildcard rules and often carry weaker security posture.
3. **Subdomain enumeration**, passive first, then active, merging and deduplicating results.
4. **Live host verification and screenshotting** to enable rapid visual categorization (login pages, admin panels, marketing sites, APIs, error pages indicating dead infrastructure).
5. **Technology fingerprinting** per live host.
6. **Manual application walkthrough** of the highest-priority 3–5 targets: register an account, explore every visible feature, and note every distinct endpoint and parameter observed.
7. **JavaScript review** for endpoints, feature flags, and hardcoded values not visible through UI navigation alone.
8. **Prioritization.** Rank targets by: authentication presence (auth'd apps tend to have richer attack surface), apparent age/maintenance level (legacy-looking UI often correlates with legacy, unpatched code), and data sensitivity implied by the application's purpose.

## Automation

A minimal automated recon pipeline, chained via shell:

```bash
#!/usr/bin/env bash
set -euo pipefail

DOMAIN="target.example"
OUT="recon-$DOMAIN"
mkdir -p "$OUT"

subfinder -d "$DOMAIN" -all -silent -o "$OUT/subs.txt"
httpx -l "$OUT/subs.txt" -silent -title -tech-detect -status-code -o "$OUT/live.txt"
naabu -list "$OUT/subs.txt" -top-ports 1000 -silent -o "$OUT/ports.txt"
katana -list "$OUT/subs.txt" -jc -silent -o "$OUT/endpoints.txt"
gowitness file -f "$OUT/live.txt" --destination "$OUT/screenshots/"

echo "Recon complete. Review $OUT/live.txt and $OUT/screenshots/ first."
```

Automation should produce a triage-ready output set, not a final answer. Every automated result still needs manual review before it informs testing priorities.

## Reporting

Recon findings are rarely reported as standalone vulnerabilities, but two categories often are:

- **Exposed non-production environments** (staging/dev hosts reachable from the public internet with weaker or absent authentication) — typically reported as a Security Misconfiguration finding.
- **Information disclosure via recon artifacts** (exposed `.git` directories, backup files, verbose error pages revealing internal paths or software versions) — reported individually per instance, referencing `report-templates/` as appropriate for the specific disclosure type.

Maintain a running recon notes document per engagement recording every host, its purpose, and its priority ranking — this becomes the map you continually return to as testing progresses.

## Best Practices

- Re-run subdomain enumeration periodically during a long-running engagement; infrastructure changes, and new subdomains appear.
- Treat screenshots as a triage aid, not a substitute for visiting interesting hosts directly.
- Keep raw tool output archived per engagement; you will frequently need to re-check "did I already see this host" weeks later.
- Respect documented rate limits during active enumeration — aggressive scanning is a common cause of program relationship problems independent of any vulnerability found.

## Common Mistakes

- Treating recon as a one-time phase rather than continuous through the engagement.
- Over-relying on a single subdomain enumeration tool; different tools draw from different data sources and results vary meaningfully.
- Ignoring "boring" looking hosts (internal tools, status pages, old marketing microsites) — these are frequently under-tested by other researchers and disproportionately likely to contain unpatched issues.
- Running noisy active scans against hosts before confirming they are actually in scope.

## Example

A recon pass against a fictional target, `globex-cloud.example`, surfaces 340 subdomains via passive enumeration. Live-host probing narrows this to 89 responsive hosts. Screenshotting reveals `metrics-internal.globex-cloud.example` returns a Grafana login page with default branding — a strong signal of an internally-intended tool exposed publicly. JavaScript review of the primary application at `app.globex-cloud.example` reveals a hardcoded reference to `/api/v1/internal/debug` in a bundled but unused code path, which manual testing confirms is still live and unauthenticated — an information disclosure finding reported using the appropriate template, entirely a product of thorough recon rather than any exploitation technique.
