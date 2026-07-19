# Reconnaissance Checklist

Use this checklist at the start of any engagement to build a complete asset inventory before deep-diving into specific vulnerability classes. Work through it linearly; check off items as completed and record findings in your engagement notes, not just in your head.

## Table of Contents

- [Manual Tests](#manual-tests)
- [Automated Tests](#automated-tests)
- [Common Misconfigurations](#common-misconfigurations)
- [Burp Suite Tips](#burp-suite-tips)
- [Edge Cases](#edge-cases)
- [Common Bypasses](#common-bypasses)
- [References](#references)

## Manual Tests

### Scope and Program Review
- [ ] Read the program's scope document in full before running any tooling
- [ ] Note explicitly out-of-scope assets, testing types, and rate limits
- [ ] Identify whether wildcard domains (`*.target.example`) are in scope
- [ ] Confirm whether the program permits automated scanning at all, and at what rate

### Organizational Footprinting
- [ ] Identify the target's parent company, subsidiaries, and recent acquisitions (M&A activity often leaves legacy, poorly-maintained infrastructure in scope)
- [ ] Review the company's engineering blog, job postings, and conference talks for technology stack hints
- [ ] Check public GitHub/GitLab organizations for exposed repositories, and review commit history for accidentally committed secrets
- [ ] Review the company's SSL/TLS certificate transparency logs for subdomains not found via other methods

### Manual Endpoint Discovery
- [ ] Browse the primary application manually, noting every distinct endpoint, parameter, and workflow
- [ ] Review client-side JavaScript bundles for hardcoded API endpoints, feature flags, and internal hostnames
- [ ] Check mobile application binaries (if in scope) for embedded API endpoints and hardcoded credentials/keys
- [ ] Review `robots.txt`, `sitemap.xml`, and `.well-known/` for disclosed paths
- [ ] Check archived versions of the site (Wayback Machine) for deprecated but still-live endpoints

## Automated Tests

### Subdomain Enumeration
- [ ] Passive enumeration via certificate transparency (`crt.sh`), DNS aggregators, and search engine dorking
- [ ] Active enumeration via tools such as `subfinder`, `amass`, and `assetfinder`, cross-referenced against each other
- [ ] Brute-force enumeration against a curated wordlist for high-value targets where passive/active enumeration alone seems incomplete

```bash
subfinder -d target.example -all -o subs_subfinder.txt
amass enum -passive -d target.example -o subs_amass.txt
cat subs_subfinder.txt subs_amass.txt | sort -u > subs_all.txt
```

### Live Host Verification
- [ ] Resolve all discovered subdomains and filter to live hosts
- [ ] Screenshot live hosts in bulk for rapid visual triage

```bash
cat subs_all.txt | httpx -silent -o live_hosts.txt
cat live_hosts.txt | gowitness file -f - --destination screenshots/
```

### Port and Service Discovery
- [ ] Run a fast full-port scan against in-scope IP ranges to identify non-standard listening services
- [ ] Follow up with service/version detection on discovered open ports
- [ ] Cross-reference exposed services against known CVEs for the identified version

```bash
naabu -host target.example -p - -o ports.txt
nmap -sV -iL live_hosts.txt -oA nmap_scan
```

### Technology Fingerprinting
- [ ] Fingerprint web server, framework, CMS, and JavaScript libraries in use per host
- [ ] Note version numbers where disclosed (response headers, error pages, static asset paths)

```bash
httpx -l live_hosts.txt -tech-detect -o tech_stack.txt
```

### Content and Parameter Discovery
- [ ] Directory/file brute-forcing against each live host with a curated wordlist
- [ ] Parameter discovery against identified endpoints
- [ ] JavaScript file crawling and endpoint extraction

```bash
ffuf -u https://FUZZ.target.example -w subs_all.txt -mc 200,301,302,403
katana -u https://target.example -jc -o endpoints.txt
```

## Common Misconfigurations

| Misconfiguration | Why It Happens | How to Spot It |
|---|---|---|
| Exposed staging/dev subdomains | Separate environment provisioned without the same access controls as production | Subdomain naming patterns (`dev-`, `staging-`, `uat-`, `test-`) surfaced in enumeration |
| Exposed cloud storage buckets | Bucket created for a specific feature, permissions never tightened | Search for `target-*`, `*-target`, `target-assets`, `target-backup` bucket name patterns |
| Forgotten legacy subdomains from acquisitions | M&A integration deprioritizes decommissioning old infrastructure | Certificate transparency logs referencing acquired company names |
| Exposed internal admin panels on public IPs | Internal tool deployed without network segmentation | Port scan results showing admin panel banners on non-standard ports |
| API documentation left publicly accessible | Swagger/OpenAPI docs deployed alongside the API without access restriction | `/swagger.json`, `/openapi.json`, `/api-docs` paths |

## Burp Suite Tips

- Use **Target > Scope** to define the exact in-scope hosts before crawling, to avoid accidentally sending requests to out-of-scope assets.
- Enable **Burp's passive Content Discovery** while manually browsing to surface additional endpoints without active scanning traffic.
- Use the **Logger** extension (or built-in Logger++ equivalent) to capture every request made during manual recon for later review and for building a complete site map.
- Import discovered subdomains as a target list, then use **Engagement Tools > Discover Content** for semi-automated endpoint discovery within scope.

## Edge Cases

- [ ] Check for IPv6-only or dual-stack hosts that may not appear in IPv4-focused scans
- [ ] Check for subdomains resolving only from specific geographic regions (geo-fenced infrastructure)
- [ ] Check for internal hostnames leaked in TLS certificate Subject Alternative Names even if not publicly resolvable
- [ ] Check for GraphQL endpoints that don't follow REST-style naming conventions and are easily missed by path-based brute-forcing (`/graphql`, `/api/graphql`, `/v1/graphql`, `/query`)

## Common Bypasses

- **WAF-fronted subdomain identification**: compare response fingerprints (headers, error pages, TLS certificate) between the apex domain and discovered subdomains to identify hosts sitting behind a different (or no) WAF.
- **Cloudflare/CDN origin discovery**: check historical DNS records (predating CDN adoption), SSL certificate transparency logs for origin hostnames, and misconfigured subdomains that bypass the CDN entirely (e.g., `direct.target.example`, `origin.target.example`).
- **Rate-limit evasion during enumeration**: distribute requests across multiple source IPs only if explicitly permitted by program rules; otherwise throttle to the documented rate limit rather than attempting evasion.

## References

- OWASP Testing Guide — Information Gathering: https://owasp.org/www-project-web-security-testing-guide/
- ProjectDiscovery tool documentation (subfinder, httpx, naabu, katana): https://docs.projectdiscovery.io/
- crt.sh certificate transparency search: https://crt.sh/
