# Reporting Guide

A technically correct finding with a poorly written report gets triaged slowly, downgraded, or closed as informative. This guide covers how to write reports that get read, understood, and paid promptly.

## Table of Contents

- [Why Report Quality Changes Outcomes](#why-report-quality-changes-outcomes)
- [Title Writing](#title-writing)
- [Severity and CVSS](#severity-and-cvss)
- [Writing Reproduction Steps](#writing-reproduction-steps)
- [Evidence: Screenshots, Requests, and Video](#evidence-screenshots-requests-and-video)
- [Impact Statements](#impact-statements)
- [Remediation Advice](#remediation-advice)
- [Common Mistakes That Slow Down Triage](#common-mistakes-that-slow-down-triage)
- [Handling Triager Pushback](#handling-triager-pushback)
- [Duplicate and N/A Outcomes](#duplicate-and-na-outcomes)

## Why Report Quality Changes Outcomes

Triage teams read dozens to hundreds of reports weekly. A report that is unambiguous, reproducible on the first attempt, and correctly scoped gets a faster response than one requiring back-and-forth to even confirm what was tested. Report quality does not change whether a bug exists, but it strongly changes how quickly and how favorably it is assessed.

## Title Writing

A good title states the vulnerability class, the affected component, and the mechanism, in that order of specificity:

- Weak: "XSS on website"
- Better: "Stored XSS in profile bio field via unsanitized markdown rendering"

Avoid severity claims in the title ("Critical RCE!!!") — let the CVSS score and impact section carry that argument.

## Severity and CVSS

Use CVSS v3.1 unless the program specifies otherwise. Score each vector component deliberately rather than defaulting to the highest plausible value:

| Metric | Common Mistake |
|---|---|
| Attack Vector (AV) | Marking Network when exploitation requires local network position |
| Privileges Required (PR) | Marking None when the flow requires an authenticated session |
| User Interaction (UI) | Marking None for an attack that requires a victim to click a crafted link |
| Scope (S) | Forgetting to mark Changed when the vulnerability crosses a security boundary (e.g., a web app bug leading to underlying host compromise) |

Programs frequently adjust submitted CVSS scores during triage. Justify your score in one or two sentences so the triager understands your reasoning even if they disagree with the final number.

## Writing Reproduction Steps

Reproduction steps should be numbered, testable by someone with no prior context, and free of assumed state:

```markdown
1. Log in as a standard user at https://app.target.example/login.
2. Navigate to Settings → Profile → Bio.
3. Enter the payload below into the Bio field and save:
   `<img src=x onerror=alert(document.domain)>`
4. Log out, then log in as a different user.
5. Navigate to the first user's public profile at /u/<username>.
6. Observe the alert box fires, confirming stored XSS executes in the context of any viewer.
```

Number every step. State exact URLs, exact field names, and exact payloads. Do not write "then try some XSS payloads" — pick the specific payload that worked and document it exactly.

## Evidence: Screenshots, Requests, and Video

- Include the raw HTTP request (from Burp's Repeater or equivalent), not a paraphrase.
- Include the raw HTTP response, or the relevant excerpt if the full response is large.
- Screenshot the triggered behavior, not just the payload sitting in a text field.
- For anything timing-dependent (race conditions) or interaction-dependent (CSRF, clickjacking), a short screen recording resolves ambiguity that screenshots cannot.

Redact anything sensitive that isn't relevant to the finding (other users' real data, your own session tokens if the report will be shared beyond the triage team) but never redact the actual proof-of-concept payload or the response fields demonstrating impact.

## Impact Statements

State impact in terms of what an attacker can *actually* achieve, grounded in the preconditions you demonstrated:

- Weak: "This could lead to account takeover."
- Better: "An attacker with the ability to post a comment (available to any registered user with no special privileges) can execute arbitrary JavaScript in the browser of any visitor viewing that comment, including administrators, enabling session token theft against the admin panel demonstrated at `/admin`."

If an attack chain requires multiple steps or preconditions, state them explicitly rather than letting the reader assume the easiest possible path.

## Remediation Advice

Provide both a short-term mitigation and a long-term fix where they differ:

- **Short-term**: input validation, WAF rule, feature flag disable.
- **Long-term**: architectural fix — parameterized queries instead of a blocklist, context-aware output encoding instead of a regex filter, a proper authorization check instead of a client-side role hide.

Generic remediation ("sanitize user input") is less useful than specific remediation ("use `DOMPurify` with a strict allow-list on the server before storage, and re-encode on output using the templating engine's autoescaping rather than manual string concatenation").

## Common Mistakes That Slow Down Triage

- Submitting a scanner's raw output without manual validation.
- Vague reproduction steps that assume the triager already knows the application layout.
- Missing the affected URL or endpoint entirely.
- Conflating a low-impact finding with a high-severity title to try to force a higher payout — this damages credibility for future submissions.
- Reporting the same root cause across dozens of endpoints as dozens of separate reports instead of one report noting the pattern (unless the program explicitly wants per-instance reports).

## Handling Triager Pushback

If a triager downgrades severity or requests more information:

- Respond with additional evidence or a clarified attack scenario, not with an appeal to authority ("I've been doing this for 10 years").
- If you believe the CVSS scoring is wrong, walk through the vector string component by component and explain your reasoning.
- Escalation processes exist on most platforms for genuine disagreements after good-faith discussion — use them as a last resort, not a first response.

## Duplicate and N/A Outcomes

Not every report will be accepted. If marked duplicate, ask (politely, once) whether the program can share the report ID or rough submission date so you can calibrate future testing. If marked N/A or informative and you disagree, provide the specific missing evidence rather than re-arguing severity in the abstract. Programs generally respect researchers who accept a well-reasoned "no" gracefully, and that reputation compounds across future submissions.
