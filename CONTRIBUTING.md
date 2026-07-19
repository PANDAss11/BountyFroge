# Contributing to BountyForge

Thank you for considering a contribution. BountyForge is maintained as a reference library for bug bounty hunters, penetration testers, and application security engineers. Its value depends entirely on the accuracy, clarity, and defensive framing of what gets merged, so contributions go through a deliberate review process.

## Table of Contents

- [Ground Rules](#ground-rules)
- [What We Accept](#what-we-accept)
- [What We Do Not Accept](#what-we-do-not-accept)
- [Getting Started](#getting-started)
- [Style Guide](#style-guide)
- [Report Template Standards](#report-template-standards)
- [Checklist Standards](#checklist-standards)
- [Payload Standards](#payload-standards)
- [Commit Conventions](#commit-conventions)
- [Pull Request Process](#pull-request-process)
- [Review Criteria](#review-criteria)
- [Recognition](#recognition)

## Ground Rules

1. All content must be authorized-testing oriented. Every checklist, methodology, or payload entry should read as guidance for testing systems you are authorized to test, not as a generic exploitation script.
2. No content may target a real, named, non-consenting organization. Example reports use fictional companies only (see [Report Template Standards](#report-template-standards)).
3. No zero-days, no undisclosed vulnerabilities, no working exploits for unpatched CVEs. If you want to contribute research tied to a real CVE, it must already be public and you must cite the advisory.
4. Cite your sources. Techniques adapted from a conference talk, blog post, or advisory should link back to the original.
5. Write for a reader who is competent but not already an expert in the specific topic. Assume they understand HTTP and general web security, but not the nuance of the bug class you are documenting.

## What We Accept

- New or improved report templates for vulnerability classes not yet covered, or refinements to existing ones (clearer CVSS guidance, better remediation language, corrected CWE mappings).
- Checklists for testing categories (cloud providers, frameworks, protocols) with concrete manual and automated test steps.
- Methodology documents describing a repeatable testing workflow.
- Payload collections with explanations of *why* a payload works and how to detect success, not just raw strings.
- Corrections: broken links, outdated CVSS vectors, deprecated tool flags, inaccurate OWASP mappings.
- Tooling integrations: nuclei templates, Burp Suite extension configurations, automation scripts that support the methodologies already documented.

## What We Do Not Accept

- Payloads or scripts whose only purpose is unauthorized access, denial of service, or data destruction.
- Content copied verbatim from HackerOne, Bugcrowd, or Intigriti reports. Disclosed reports may be referenced and linked, never reproduced.
- Malware, backdoors, or persistence mechanisms framed as "post-exploitation examples."
- Marketing content for commercial products. Tool mentions should be neutral and functional.
- Low-effort submissions: single-line payload additions without context, checklists copy-pasted from another repository without attribution.

## Getting Started

```bash
# Fork and clone
git clone https://github.com/<your-username>/BountyForge.git
cd BountyForge

# Create a topic branch
git checkout -b add/graphql-batching-checklist

# Make your changes, then run the local lint pass described below
```

Before opening a pull request, run a spell check and a Markdown lint pass locally. The same checks run in CI (`.github/workflows/`) and will block merge on failure.

```bash
npx markdownlint-cli2 "**/*.md"
npx cspell "**/*.md"
```

## Style Guide

- Use ATX-style headings (`#`, `##`, `###`), never Setext (`===`, `---`) underlines.
- One `H1` per document, matching the file's purpose.
- Wrap prose naturally; do not hard-wrap at a fixed column.
- Use fenced code blocks with an explicit language tag (` ```http `, ` ```bash `, ` ```json `).
- Use tables for structured comparisons (severity matrices, tool comparisons, bypass lists).
- Use GitHub-flavored admonitions consistently:
  - `> **Note**` for supplementary context.
  - `> **Warning**` for actions that can cause damage, trigger WAF bans, or violate program scope if misapplied.
- Avoid marketing language ("revolutionary," "cutting-edge," "game-changing"). Write like a technical reviewer, not a press release.
- Avoid absolute claims ("this always works," "guaranteed bypass"). Security behavior is environment-dependent; qualify claims accordingly.

## Report Template Standards

Each report template under `report-templates/` must include, in order:

1. Executive Summary
2. Severity and CVSS vector (v3.1, with justification for each metric)
3. CWE classification
4. OWASP Top 10 / API Security Top 10 mapping where applicable
5. Affected component
6. Description
7. Preconditions
8. Attack scenario narrative
9. Reproduction steps (numbered, testable)
10. Example HTTP request/response pair
11. Evidence section (what a screenshot or log excerpt should show)
12. Business impact
13. Risk assessment
14. Remediation guidance (short-term and long-term)
15. References
16. Reviewer/report notes

Templates should target 200–300 lines. Shorter is acceptable if the vulnerability class genuinely does not need more; padding is not acceptable.

## Checklist Standards

Each checklist must separate **manual tests** from **automated tests**, list **common misconfigurations**, include **Burp Suite-specific tips** where relevant, and end with a **references** section linking to primary sources (vendor docs, OWASP, RFCs).

## Payload Standards

Every payload entry needs:

- **Purpose** — what condition it detects or exploits.
- **Usage** — where in the request it belongs (parameter, header, path segment).
- **Detection** — how to confirm the payload had an effect (response diffing, timing, out-of-band callback).
- **Expected behavior** — what a vulnerable vs. non-vulnerable target does differently.
- **Known bypasses / variants** — filter evasion notes, with the underlying reason the bypass works.

Do not submit payload lists without this context. A bare list of strings has no educational value and will be closed.

## Commit Conventions

We use Conventional Commits:

```
feat(payloads): add DOM-based XSS sink reference table
fix(templates): correct CVSS vector for SSRF template
docs(checklists): add CORS preflight edge cases
chore(ci): bump markdownlint-cli2 version
```

## Pull Request Process

1. Open an issue first for anything larger than a typo fix or single payload addition, so scope can be agreed before you invest time.
2. Keep PRs focused — one checklist, one template, or one coherent set of related payloads per PR.
3. Fill out the pull request template completely, including the "Testing / Verification" section describing how you validated the content (lab environment, CTF, disclosed report reference).
4. A maintainer will review within 7 days. Expect at least one round of feedback on tone, accuracy, or structure.
5. Once approved, a maintainer will squash-merge.

## Review Criteria

Maintainers evaluate submissions against:

| Criterion | What we check |
|---|---|
| Accuracy | Technical claims are correct and current |
| Authorization framing | Content assumes and reinforces authorized testing only |
| Completeness | No placeholder sections, no "TODO," no unfinished tables |
| Originality | Not copy-pasted from another repository or disclosed report |
| Clarity | Understandable to a mid-level security researcher |
| Formatting | Passes markdownlint and cspell CI checks |

## Recognition

Contributors are listed in the README credits section after their first merged PR. Significant sustained contributors may be invited to join as maintainers with merge rights.

Questions can be opened as a GitHub Discussion or a `question`-labeled issue.
