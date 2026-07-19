# Burp Suite Configuration and Extensions

Recommended Burp Suite setup referenced throughout the checklists in this repository.

## Table of Contents

- [Recommended Extensions](#recommended-extensions)
- [Project Configuration](#project-configuration)
- [Match and Replace Rules](#match-and-replace-rules)
- [Session Handling Rules](#session-handling-rules)
- [Suggested Workflow](#suggested-workflow)

## Recommended Extensions

Install via the BApp Store (Extender tab):

| Extension | Purpose | Referenced By |
|---|---|---|
| Autorize | Automated object/function-level authorization testing across a crawled site map | `checklists/api-security.md`, `methodology/api-testing.md` |
| Param Miner | Discovers hidden/unlinked parameters and headers | `checklists/api-security.md` |
| JSON Web Tokens | Decodes, edits, and re-signs JWTs inline in Repeater/Proxy | `checklists/jwt.md` |
| Logger++ | Advanced request/response logging with filtering, useful for recon traffic capture | `checklists/recon.md` |
| Turbo Intruder | High-throughput request sending, useful for race condition testing | `checklists/race-condition.md` (planned) |
| GraphQL Raider | GraphQL-aware scope and injection point identification | `checklists/graphql.md` (planned) |
| Backslash Powered Scanner | Heuristic-based injection point discovery independent of specific payload signatures | General injection testing |

## Project Configuration

For large-scope engagements:

1. Create a dedicated Burp project file per engagement (`Project > New Project on Disk`) rather than using a temporary in-memory project — this preserves your site map and Logger history if Burp needs to be restarted.
2. Set **Target > Scope** explicitly to the in-scope hosts before crawling or scanning, and enable **"Show only in-scope items"** in the site map to reduce noise.
3. Under **Proxy > Options**, disable interception for static asset extensions (`.js`, `.css`, `.png`, `.woff`) to reduce Proxy interception noise during manual testing — re-enable selectively when JavaScript file review is the active task.

## Match and Replace Rules

Useful rules referenced by the authentication and authorization checklists, configured under **Proxy > Options > Match and Replace**:

**Strip a specific cookie to test unauthenticated behavior on every request:**
- Type: Request header
- Match: `Cookie: session=[^;]+;?\s*`
- Replace: (empty)

**Swap session tokens to test cross-account authorization quickly (toggle on/off during Autorize-style manual spot checks):**
- Type: Request header
- Match: `Authorization: Bearer .*`
- Replace: `Authorization: Bearer <ACCOUNT_B_TOKEN>`

## Session Handling Rules

For applications with short-lived tokens or CSRF tokens that invalidate Repeater replay, configure a **Session Handling Rule** (`Project options > Sessions`) that:

1. Runs a macro to re-authenticate and capture a fresh token when a request matches a "session expired" response pattern.
2. Applies automatically to Repeater and Intruder tabs used for the affected host, so long testing sessions don't require constant manual re-authentication.

## Suggested Workflow

1. Configure scope and Match/Replace rules at the start of the engagement, per the recon methodology.
2. Browse the application manually with Proxy intercept off, building an organic site map.
3. Use Autorize with two configured session tokens (different privilege levels) while re-browsing key flows, to get automated authorization coverage as a first pass.
4. Use Repeater for manual, iterative testing of specific hypotheses identified from the checklists.
5. Export the Logger++ log at the end of each testing session as part of your engagement evidence trail.
