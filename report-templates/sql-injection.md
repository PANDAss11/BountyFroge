# SQL Injection — Vulnerability Report Template

> Copy this file, rename it to match your finding, and complete every section. Replace all bracketed placeholders.

## 1. Executive Summary

[One paragraph for a non-technical stakeholder. State the vulnerable component, the injection type, and the realistic worst-case impact — typically unauthorized data access, and in severe cases data modification or command execution via the database layer.]

## 2. Severity

| Field | Value |
|---|---|
| Overall Severity | [Critical / High] |
| CVSS v3.1 Vector | `AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:H` |
| CVSS v3.1 Score | [e.g., 9.8] |
| Vulnerability Type | SQL Injection ([In-band / Blind Boolean / Blind Time-based / Out-of-band]) |

### CVSS Justification

| Metric | Value | Reasoning |
|---|---|---|
| Attack Vector (AV) | Network | Exploitable via standard HTTP request |
| Attack Complexity (AC) | Low | Reliable, repeatable payload — no race condition |
| Privileges Required (PR) | [None / Low] | [State whether authentication is needed] |
| User Interaction (UI) | None | No victim interaction required |
| Scope (S) | [Unchanged / Changed] | Changed if the database underlies multiple applications/tenants beyond the tested app |
| Confidentiality (C) | High | Full or partial database read access demonstrated |
| Integrity (I) | [None / High] | High if write access (UPDATE/INSERT/DELETE) was confirmed |
| Availability (A) | [None / High] | High if the injection allows resource exhaustion or destructive statements |

## 3. CWE Classification

- **CWE-89**: Improper Neutralization of Special Elements used in an SQL Command ('SQL Injection')

## 4. OWASP Mapping

- OWASP Top 10 2021: **A03:2021 – Injection**
- OWASP API Security Top 10: **API8:2023 – Security Misconfiguration** (if the injection point is an API parameter rather than a traditional web form)

## 5. Affected Component

| Field | Value |
|---|---|
| URL | `[https://target.example/api/path]` |
| Parameter | `[parameter name]` |
| HTTP Method | `[GET / POST / PUT]` |
| Injection Point | [URL parameter / JSON body field / HTTP header / Cookie / Sort/Filter parameter] |
| Database Engine (if identified) | [MySQL / PostgreSQL / MSSQL / Oracle / SQLite — note how it was fingerprinted] |
| Authentication Required | [Yes / No — specify role] |

## 6. Description

[Explain why the vulnerability exists: unparameterized query construction via string concatenation, an ORM method that falls back to raw SQL for a specific feature (e.g., dynamic sort/order-by clauses), or a stored procedure receiving unsanitized input. Note the injection technique that succeeded (error-based, UNION-based, boolean-blind, time-blind, out-of-band) and why — e.g., error messages were suppressed, ruling out error-based extraction and requiring a blind technique instead.]

## 7. Preconditions

- [e.g., None — the endpoint is unauthenticated]
- [e.g., Requires a valid low-privilege API key]
- [Any WAF/filter bypass required, and how it was achieved]

## 8. Attack Scenario

1. Attacker identifies the `sort` parameter on the product listing endpoint reflects into the SQL `ORDER BY` clause without parameterization.
2. Attacker confirms injection via a boolean-based blind technique, since the application suppresses database error messages.
3. Attacker extracts the database version, current user, and schema names using conditional time-delay responses.
4. Attacker enumerates the `users` table and extracts email addresses and password hashes column by column.
5. Attacker cross-references extracted email addresses with a credential-stuffing list to attempt account takeover on a subset of accounts.

## 9. Reproduction Steps

1. Send the baseline request to the endpoint and confirm the normal (non-error) response.
2. Inject a single quote (`'`) into the target parameter and observe a change in application behavior (error, altered result set, or timing shift).
3. Confirm the injection is boolean-controllable by sending a always-true and an always-false conditional payload and diffing the responses.
4. Escalate to data extraction using the technique appropriate to the confirmed injection type (see Section 10).
5. Extract a limited, clearly labeled proof-of-concept dataset — do not perform bulk exfiltration beyond what is necessary to prove impact.

## 10. Proof-of-Concept Payloads

**Confirmation (boolean-based blind):**
```
?sort=name AND 1=1   →  200 OK, normal result set
?sort=name AND 1=2   →  200 OK, empty result set
```

**Database fingerprinting (time-based blind, PostgreSQL example):**
```
?sort=name; SELECT CASE WHEN (1=1) THEN pg_sleep(5) ELSE pg_sleep(0) END--
```

**Data extraction (UNION-based, once column count is confirmed):**
```
?id=-1 UNION SELECT username, password_hash, NULL FROM users LIMIT 1--
```

> **Warning**
> Do not run destructive statements (`DROP`, `DELETE`, `UPDATE` without a `WHERE` clause) against a live target, even to "prove" write access. Demonstrate write capability with a reversible, clearly-scoped change agreed with the program if write impact needs proof, or rely on the read-access demonstration plus a written statement of the theoretical write capability based on observed privileges.

## 11. HTTP Request

```http
GET /api/products?sort=name%3B%20SELECT%20CASE%20WHEN%20(1%3D1)%20THEN%20pg_sleep(5)%20ELSE%20pg_sleep(0)%20END--&order=asc HTTP/1.1
Host: target.example
Cookie: session=<session-token>
Accept: application/json
```

## 12. HTTP Response

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 512
X-Response-Time: 5023ms

{
  "products": [...]
}
```

[Note the elevated `X-Response-Time` or measured round-trip latency of ~5000ms, matching the injected `pg_sleep(5)`, confirming time-based blind injection.]

## 13. Evidence

- [ ] Baseline vs. injected response diff (boolean-based)
- [ ] Timing measurements across at least 3 trials for time-based confirmation, to rule out network jitter
- [ ] Screenshot or terminal output of extracted schema/table names (redact actual user PII beyond what's needed to prove access)
- [ ] Database version and engine fingerprint
- [ ] Out-of-band interaction log, if OOB technique was used (e.g., via `xp_dirtree` on MSSQL or `UTL_HTTP` on Oracle)

## 14. Impact

- **Confidentiality**: [Full/partial database read access — specify tables/columns accessed]
- **Integrity**: [State whether write access was confirmed, and its scope]
- **Availability**: [State whether resource-exhaustion or destructive statements are plausible given confirmed privileges]
- **Business Impact**: [e.g., exposure of the full user credential table enables mass account takeover; exposure of payment-adjacent tables may trigger regulatory disclosure obligations under PCI-DSS]

## 15. Risk Assessment

| Factor | Assessment |
|---|---|
| Likelihood | [High — reliable, scriptable exploitation] |
| Exploitability | [High — no rate limiting observed; automatable with sqlmap] |
| Detectability by Defender | [Low if error messages are suppressed and no query logging/anomaly detection is in place] |
| Blast Radius | [Entire user table / entire database / potentially the underlying host if stacked queries and elevated DB privileges are present] |

## 16. Remediation

### Short-Term
- Deploy a WAF rule targeting the specific injection pattern as an immediate mitigation.
- Rate-limit or temporarily disable the vulnerable endpoint if exploitation is trivial and automatable.

### Long-Term
- Replace string-concatenated SQL with parameterized queries / prepared statements universally — including for `ORDER BY` and other clauses that don't support standard bind parameters (use an allow-list of valid column names instead, mapped server-side).
- Apply the principle of least privilege to the database account used by the application; the account should not have rights to system tables, cross-database queries, or command execution functions (`xp_cmdshell`, `UTL_HTTP`, etc.) unless explicitly required.
- Enable query logging and anomaly detection (e.g., alerting on `UNION SELECT` patterns or abnormal query execution time) to catch exploitation attempts against similar future bugs.
- Conduct a codebase-wide audit for other instances of the same anti-pattern (dynamic query construction via string concatenation), since a single instance often indicates a systemic issue.

## 17. References

- OWASP SQL Injection Prevention Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html
- OWASP Top 10 2021 — A03:2021 Injection: https://owasp.org/Top10/A03_2021-Injection/
- CWE-89: https://cwe.mitre.org/data/definitions/89.html
- PortSwigger Web Security Academy — SQL injection: https://portswigger.net/web-security/sql-injection

## 18. Report Notes

| Field | Value |
|---|---|
| Reported By | [handle] |
| Date Reported | [YYYY-MM-DD] |
| Program / Client | [name] |
| Report ID | [platform-assigned ID, if applicable] |
| Extraction Scope | [Note explicitly how much data was extracted for PoC purposes and confirm no bulk exfiltration occurred] |
| Retest Status | [Pending / Confirmed Fixed / Reopened] |
