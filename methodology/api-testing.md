# API Testing Methodology

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

API testing aims to build a complete map of an API's endpoints, authentication model, and object relationships, then systematically test each endpoint against the OWASP API Security Top 10. Unlike traditional web application testing, APIs frequently lack a visible UI to guide exploration, so a disciplined discovery phase matters even more than in web testing.

## Tools

| Purpose | Tool | Notes |
|---|---|---|
| Documentation discovery | Manual search for `/swagger.json`, `/openapi.json`, `/api-docs`, Postman public workspaces | Often the fastest path to a complete endpoint map |
| Traffic capture | Burp Suite / OWASP ZAP | Capture all traffic while using the application/mobile app normally to build an organic site map |
| Object-level authorization testing | Autorize (Burp extension) | Automates cross-account authorization testing across a crawled map |
| Schema-aware fuzzing | `schemathesis` (if OpenAPI spec available) | Generates test cases directly from the API schema |
| Manual request crafting | Burp Repeater / Postman | For precise, iterative testing of specific hypotheses |
| Token/JWT inspection | `jwt_tool`, Burp JWT extension | See `checklists/jwt.md` |

## Manual Workflow

1. **Documentation discovery.** Search for an OpenAPI/Swagger spec, GraphQL schema, or Postman collection. If found, this becomes your primary endpoint map.
2. **Organic mapping.** Regardless of documentation availability, use the application (web or mobile) normally through an intercepting proxy to capture real traffic — documented specs are frequently incomplete or stale.
3. **Authentication model mapping.** Identify how authentication works: session cookie, bearer token, API key, mutual TLS. Note token issuance, refresh, and expiry behavior.
4. **Object relationship mapping.** For each resource type (users, orders, documents, etc.), note the identifier format and how ownership/access relationships are expressed.
5. **Systematic Top 10 pass.** Work through `checklists/api-security.md` methodically against every distinct endpoint category, not just the ones that seem obviously interesting.
6. **Cross-account testing.** Create at least two test accounts at different privilege levels (and, if the platform is multi-tenant, in different tenants) to test object- and function-level authorization thoroughly.
7. **Version and shadow endpoint sweep.** Check for older API versions and undocumented endpoints referenced in client-side code but absent from official documentation.

## Automation

```bash
# If an OpenAPI spec is available, generate schema-aware test cases
schemathesis run https://target.example/openapi.json \
  --checks all \
  --header "Authorization: Bearer $TOKEN" \
  --rate-limit 5

# Cross-account object-level authorization spot check (example loop)
for endpoint in orders invoices documents; do
  echo "Testing $endpoint..."
  curl -s -H "Authorization: Bearer $ACCOUNT_A_TOKEN" \
    "https://target.example/api/$endpoint/$ACCOUNT_B_OBJECT_ID" \
    -o "results/$endpoint-crossaccount.json"
done
```

Automated schema-based fuzzing is effective for surfacing input-validation classes of bugs (injection, type confusion) but is weak at detecting authorization issues, which require the tester to understand *who should* be able to access *what* — a judgment automation cannot make alone. Treat automation as a first pass, not a substitute for the manual cross-account workflow.

## Reporting

Map each confirmed finding to the specific OWASP API Security Top 10 2023 category it represents, and use that mapping explicitly in the report's OWASP Mapping section — this significantly speeds up triage for programs that categorize findings this way. Use `report-templates/idor.md` for object-level authorization findings (the most common API finding by volume), `report-templates/authorization.md` for function-level findings, and the general API Security template for anything spanning multiple categories.

## Best Practices

- Maintain a spreadsheet or notes file mapping every endpoint to its tested status per Top 10 category — API surfaces are large enough that memory alone will miss coverage gaps.
- Test both the documented and undocumented (JS-discovered) endpoints with equal rigor; undocumented endpoints are frequently less reviewed internally.
- When testing object-level authorization, always test in both directions (Account A → Account B's objects, and Account B → Account A's objects) in case the check is asymmetric.
- Re-test previously "clean" endpoints after any application update noticed during a long engagement; authorization regressions are common.

## Common Mistakes

- Testing only the happy-path documented endpoints and skipping JS-discovered or versioned duplicates.
- Using only one test account, making object-level authorization testing impossible.
- Treating a 403/401 response as proof of proper authorization without checking whether the same object is reachable via a slightly different path, method, or parameter format.
- Ignoring GraphQL endpoints because they don't fit the REST-oriented checklist structure — use `checklists/graphql.md` specifically rather than skipping them.

## Example

Testing a fictional logistics platform's API, `api.parcelroute.example`, documentation discovery surfaces a public Swagger UI at `/api-docs`, revealing 40 documented endpoints. Organic traffic capture while using the mobile app reveals 12 additional undocumented endpoints under `/api/internal/`, including `/api/internal/routes/{id}/gps-log`. Cross-account testing with two courier-role test accounts confirms Account A can retrieve Account B's GPS log history via this endpoint with no ownership check — a Broken Object Level Authorization finding (API1:2023) with a meaningful privacy impact, reported using `report-templates/idor.md`, and found specifically because the methodology's organic mapping step surfaced an endpoint the official documentation omitted entirely.
