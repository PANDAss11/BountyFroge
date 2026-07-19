# API Security Checklist

Covers REST and general JSON/HTTP API testing, structured around the OWASP API Security Top 10 2023. For protocol-specific depth, pair with `checklists/graphql.md`, `checklists/jwt.md`, and `checklists/oauth.md`.

## Table of Contents

- [Manual Tests](#manual-tests)
- [Automated Tests](#automated-tests)
- [Common Misconfigurations](#common-misconfigurations)
- [Burp Suite Tips](#burp-suite-tips)
- [Edge Cases](#edge-cases)
- [Common Bypasses](#common-bypasses)
- [References](#references)

## Manual Tests

### Object-Level Authorization (API1:2023)
- [ ] For every endpoint accepting an object identifier, test access with a second, unrelated account's session
- [ ] Test both read (`GET`) and write (`PUT`/`PATCH`/`DELETE`) operations for the same authorization gap
- [ ] Test nested/related object access (e.g., `GET /orders/{id}/items` — does ownership of the parent object get re-verified, or only checked at the top level?)

### Authentication (API2:2023)
- [ ] Confirm every endpoint that should require authentication actually rejects unauthenticated requests
- [ ] Test API key/token handling: are keys transmitted in URLs (leaked via logs/referrer headers) rather than headers?
- [ ] Confirm token expiry is enforced server-side, not just assumed from issuance time client-side

### Property-Level Authorization (API3:2023)
- [ ] Test for excessive data exposure: does the API return fields the client doesn't display but a modified request could reveal (internal notes, other users' partial data, cost/margin fields)?
- [ ] Test for mass assignment: submit additional, undocumented fields in a write request (`"role": "admin"`, `"isVerified": true`, `"balance": 999999`) and check whether they're accepted

### Resource Consumption (API4:2023)
- [ ] Test whether pagination limits (`page_size`, `limit`) can be set to an arbitrarily large value, causing excessive resource consumption
- [ ] Test for missing rate limiting on expensive endpoints (search, export, report generation)
- [ ] Test file upload endpoints for missing size limits

### Function-Level Authorization (API5:2023)
- [ ] Enumerate admin/internal endpoints (via JS analysis, API documentation, or naming pattern guessing) and test access with a standard-privilege account
- [ ] Test whether HTTP method switching bypasses a function-level check (e.g., a check exists for `POST /admin/users` but not `GET /admin/users`)

### Business Flow Consumption (API6:2023)
- [ ] Identify sensitive business flows (purchase, referral bonus claim, coupon redemption) and test for automatable abuse (scripted bulk execution)
- [ ] Test whether flows intended to be human-paced (e.g., "one claim per day") are enforceable server-side or trivially replayable

### Server-Side Request Forgery (API7:2023)
- [ ] Identify any parameter that causes the server to fetch a URL and test per `payloads/ssrf.md`

### Security Misconfiguration (API8:2023)
- [ ] Check for verbose error messages leaking stack traces, internal paths, or library versions
- [ ] Check for missing security headers (`Content-Security-Policy`, `X-Content-Type-Options`, `Strict-Transport-Security`)
- [ ] Check for permissive CORS configuration (see `checklists/cors.md`)
- [ ] Check for exposed debug/admin interfaces left enabled in production (Swagger UI with try-it-out enabled against production data, GraphQL introspection enabled, framework debug pages)

### Inventory Management (API9:2023)
- [ ] Identify and test older API versions still live alongside the current version (`/v1/`, `/api/v1/` often lack fixes applied to `/v2/`)
- [ ] Identify undocumented/shadow endpoints via JS bundle analysis and mobile app reverse engineering

### Unsafe Consumption of APIs (API10:2023)
- [ ] If the application integrates third-party APIs, test whether responses from those integrations are trusted without validation (e.g., a webhook receiver that doesn't verify the sender's signature)

## Automated Tests

### Broken Object Level Authorization Scanning
```bash
# Example workflow using a captured baseline request and swapping the object ID/session
curl -s -H "Authorization: Bearer $ACCOUNT_A_TOKEN" \
  https://target.example/api/orders/1043 | jq .
```

### Mass Assignment Fuzzing
```bash
ffuf -u https://target.example/api/users/me \
     -X PATCH \
     -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
     -d '{"FUZZ": true}' \
     -w mass-assignment-fields.txt
```

### Rate Limit Verification
```bash
for i in $(seq 1 50); do
  curl -s -o /dev/null -w "%{http_code}\n" \
    -H "Authorization: Bearer $TOKEN" \
    https://target.example/api/search?q=test
done | sort | uniq -c
```

## Common Misconfigurations

| Misconfiguration | Why It Happens | How to Spot It |
|---|---|---|
| Object-level auth checked on `GET` but not `DELETE` | Access control added incrementally, only to the endpoint that prompted concern | Test every HTTP verb for the same resource, not just the one you noticed first |
| Internal fields returned in API responses | Backend model serialized directly to JSON without an explicit response schema | Compare full response body against what the UI actually renders |
| API versioning without deprecation | Old version kept live indefinitely for backward compatibility | Older version lacks a security fix present in the current version |
| GraphQL introspection enabled in production | Default framework setting never disabled for production builds | `{__schema{types{name}}}` query succeeds against production |

## Burp Suite Tips

- Use the **Autorize** extension to automate object-level authorization testing across an entire crawled site map by replaying every request with a lower-privilege session token.
- Use **Param Miner** to discover hidden/undocumented parameters that may enable mass assignment.
- Use **JSON Web Tokens** extension alongside the JWT checklist when APIs are JWT-authenticated.
- Configure a dedicated Burp project per API version if testing multiple versions in parallel, to avoid conflating findings.

## Edge Cases

- [ ] Test batch/bulk endpoints (`POST /api/orders/batch`) for per-item authorization — bulk endpoints frequently check authorization once for the request but not per contained object
- [ ] Test API behavior when `Content-Type` doesn't match the actual body format
- [ ] Test whether internal/administrative API versions are reachable from a different subdomain or port not covered by the primary rate limiter

## Common Bypasses

- **Object ID type confusion**: some frameworks treat `123`, `"123"`, and `123.0` differently in authorization middleware vs. the data-fetching layer — test type variants.
- **Verb tunneling**: test `X-HTTP-Method-Override` header to reach a differently-authorized code path for the same logical operation.
- **Trailing slash / case variation**: `/api/Admin/Users` or `/api/admin/users/` may bypass a case-sensitive or exact-path authorization rule while still routing to the same handler.

## References

- OWASP API Security Top 10 2023: https://owasp.org/API-Security/editions/2023/en/0x00-header/
- OWASP API Security Testing Guide: https://owasp.org/www-project-api-security/
- PortSwigger Web Security Academy — API testing: https://portswigger.net/web-security/api-testing
