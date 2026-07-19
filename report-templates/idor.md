# Insecure Direct Object Reference (IDOR) — Vulnerability Report Template

> Copy this file, rename it to match your finding, and complete every section. Replace all bracketed placeholders.

## 1. Executive Summary

[One paragraph for a non-technical stakeholder. State which resource is exposed, what identifier is manipulable, and what an attacker can access or modify as a result. Example: "The order details endpoint returns any user's order data when supplied with a sequential order ID, without verifying the requesting user owns that order. An attacker can enumerate order IDs to read other customers' names, addresses, and partial payment details."]

## 2. Severity

| Field | Value |
|---|---|
| Overall Severity | [Critical / High / Medium] |
| CVSS v3.1 Vector | `AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:N/A:N` |
| CVSS v3.1 Score | [e.g., 7.5] |
| Vulnerability Type | Insecure Direct Object Reference / Broken Object Level Authorization |

### CVSS Justification

| Metric | Value | Reasoning |
|---|---|---|
| Attack Vector (AV) | Network | Exploitable via direct HTTP requests |
| Attack Complexity (AC) | Low | Sequential or guessable identifiers; no special conditions |
| Privileges Required (PR) | [Low / None] | [State the minimum privilege needed — usually a standard authenticated account] |
| User Interaction (UI) | None | No victim action required |
| Scope (S) | [Unchanged / Changed] | Changed if access crosses a tenant/organization boundary in a multi-tenant system |
| Confidentiality (C) | [Low / High] | Depends on sensitivity of exposed data (PII, financial, health) |
| Integrity (I) | [None / High] | High if the IDOR permits modification (update/delete), not just read |
| Availability (A) | None | Typically not applicable |

## 3. CWE Classification

- **CWE-639**: Authorization Bypass Through User-Controlled Key
- Secondary: **CWE-284** (Improper Access Control)

## 4. OWASP Mapping

- OWASP Top 10 2021: **A01:2021 – Broken Access Control**
- OWASP API Security Top 10 2023: **API1:2023 – Broken Object Level Authorization** (primary mapping for API-driven IDOR)

## 5. Affected Component

| Field | Value |
|---|---|
| URL | `[https://target.example/api/orders/{id}]` |
| Parameter / Path Segment | `[id]` |
| HTTP Method | `[GET / PUT / DELETE]` |
| Identifier Type | [Sequential integer / UUID (if predictable via other leak) / Base64-encoded integer] |
| Authentication Required | Yes — [role/tier] |
| Object Type Exposed | [Order / Invoice / Message / Document / User Profile] |

## 6. Description

[Explain the missing authorization check: the endpoint retrieves the object by the supplied identifier without verifying it belongs to (or is otherwise accessible by) the authenticated requester. Note the identifier's predictability — sequential integers are trivially enumerable; UUIDs are not directly guessable but may be discoverable via another leak point (e.g., returned in a list endpoint, or present in an email/notification link).]

## 7. Preconditions

- A valid authenticated session for **any** user (Account A)
- Knowledge or guessability of a target object's identifier belonging to a **different** user (Account B)
- [Note any additional constraint, e.g., the object must be in a specific status]

## 8. Attack Scenario

1. Attacker registers a standard account (Account A) and creates one object of the relevant type (e.g., places an order) to learn the identifier format.
2. Attacker observes the identifier is a sequential integer incrementing by one per object created platform-wide.
3. Attacker requests the same endpoint with adjacent identifier values, substituting their own session token.
4. The server returns full object data for identifiers belonging to other users, with no ownership check.
5. Attacker scripts sequential enumeration across a range of identifiers to harvest data at scale.

## 9. Reproduction Steps

1. Log in as Account A. Create an object of the relevant type and note its returned identifier (e.g., `1042`).
2. Log in as Account B (a second, unrelated test account) and create a second object, noting its identifier (e.g., `1043`) to confirm sequential/predictable numbering.
3. While authenticated as Account A, send a request to the object endpoint using **Account B's** identifier (`1043`).
4. Observe the response returns Account B's object data in full, despite the request being authenticated as Account A.
5. Repeat with a `PUT`/`DELETE` request if write access is suspected, using a disposable test object to confirm modification/deletion capability without affecting real user data.

## 10. HTTP Request

```http
GET /api/orders/1043 HTTP/1.1
Host: target.example
Cookie: session=<Account-A-session-token>
Accept: application/json
```

## 11. HTTP Response

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 384

{
  "order_id": 1043,
  "owner_id": "user_88213",
  "owner_email": "victim@example.com",
  "shipping_address": "42 Example Ave, Springfield",
  "items": [ { "sku": "SKU-1029", "qty": 1 } ],
  "payment_last4": "4242"
}
```

[Note: the response above confirms Account A received Account B's private order data despite no ownership relationship — the core evidence of the IDOR.]

## 12. Evidence

- [ ] Paired requests: Account A creating/owning object `1042`, Account B creating/owning object `1043`
- [ ] Cross-account request (Account A's session, Account B's object ID) with full response showing unauthorized data
- [ ] If write access is present: before/after state of a disposable test object showing successful cross-account modification
- [ ] Screenshot of both accounts' dashboards side by side, if useful to visually confirm the data crossover

## 13. Impact

- **Confidentiality**: [Exposure scope — e.g., all users' names, addresses, and last-4 payment digits, enumerable across the full identifier space]
- **Integrity**: [If applicable — ability to modify or cancel other users' orders]
- **Availability**: [Typically None, unless mass deletion is possible]
- **Business Impact**: [e.g., regulatory exposure under data protection law given PII disclosure at scale; reputational risk if exploited and disclosed publicly before remediation]

## 14. Risk Assessment

| Factor | Assessment |
|---|---|
| Likelihood | [High — trivial to discover once one authenticated account exists] |
| Exploitability | [High — fully scriptable enumeration] |
| Detectability by Defender | [Low, unless rate-limiting or anomaly detection flags sequential cross-account access patterns] |
| Blast Radius | [Every object of this type platform-wide, bounded by the identifier space] |

## 15. Remediation

### Short-Term
- Add a rate limit on the endpoint to slow bulk enumeration while a proper fix is developed.
- If feasible short-term, switch to non-sequential identifiers (UUIDv4) as a stopgap — note this reduces but does not eliminate risk if IDs leak elsewhere.

### Long-Term
- Enforce object-level authorization on every access to the resource: verify the authenticated user's ID matches the object's owner (or the user holds an explicit grant/role permitting access) before returning or modifying data.
- Apply this check centrally (middleware / a shared authorization layer) rather than per-endpoint, to prevent the same gap from recurring on other object types.
- Add automated tests asserting that Account A cannot access Account B's objects, for every object type with an ownership concept, as a regression guard.
- Log and alert on access patterns consistent with enumeration (e.g., a single account requesting many sequential IDs in a short window).

## 16. References

- OWASP API Security Top 10 2023 — API1:2023 Broken Object Level Authorization: https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/
- OWASP Top 10 2021 — A01:2021 Broken Access Control: https://owasp.org/Top10/A01_2021-Broken_Access_Control/
- CWE-639: https://cwe.mitre.org/data/definitions/639.html
- PortSwigger Web Security Academy — Access control vulnerabilities: https://portswigger.net/web-security/access-control

## 17. Report Notes

| Field | Value |
|---|---|
| Reported By | [handle] |
| Date Reported | [YYYY-MM-DD] |
| Program / Client | [name] |
| Report ID | [platform-assigned ID, if applicable] |
| Test Accounts Used | [Account A / Account B identifiers — confirm both are researcher-controlled test accounts] |
| Retest Status | [Pending / Confirmed Fixed / Reopened] |
