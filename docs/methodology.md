# Methodology Overview

This document explains the philosophy behind the methodology guides in `methodology/`, and how they relate to the checklists and payload references elsewhere in the repository.

## Table of Contents

- [Why a Documented Methodology Matters](#why-a-documented-methodology-matters)
- [The BountyForge Testing Loop](#the-bountyforge-testing-loop)
- [Depth vs. Coverage](#depth-vs-coverage)
- [How Methodology, Checklists, and Payloads Relate](#how-methodology-checklists-and-payloads-relate)
- [Evidence Standards](#evidence-standards)
- [When to Stop](#when-to-stop)

## Why a Documented Methodology Matters

Ad hoc testing finds obvious bugs. A documented methodology finds the bugs that matter — the ones buried three parameters deep in an internal API, or hidden behind a business logic assumption the developers never questioned. Every methodology guide in this repository exists to convert "poke around and see what happens" into a repeatable process that produces consistent coverage regardless of who is running it.

A second, quieter benefit: documented methodology makes your reports more credible. A triager reading "I followed a structured authorization test matrix covering horizontal and vertical privilege escalation across all identified roles" trusts the finding more than one that reads "I tried some stuff."

## The BountyForge Testing Loop

Every methodology guide in this repository follows the same six-phase loop, even though the specific activities in each phase differ by testing category:

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Recon &   │ --> │   Mapping   │ --> │  Hypothesis │
│  Discovery  │     │             │     │ Generation  │
└─────────────┘     └─────────────┘     └──────┬──────┘
                                                 │
┌─────────────┐     ┌─────────────┐             │
│  Reporting  │ <-- │ Validation  │ <-----------┘
│             │     │             │
└─────────────┘     └─────────────┘
```

1. **Recon & Discovery** — Identify assets, endpoints, technologies, and entry points in scope.
2. **Mapping** — Build a structured understanding of how those assets relate: authentication boundaries, data flow, trust zones.
3. **Hypothesis Generation** — For each mapped component, ask "what could go wrong here" using the relevant checklist as a prompt list rather than a script to follow blindly.
4. **Validation** — Confirm or disprove each hypothesis with the minimum necessary payloads, using the detection guidance in `payloads/` to avoid false positives.
5. **Reporting** — Document confirmed findings using the matching template in `report-templates/`.
6. **Retest** — After a fix is deployed, re-run the specific validation step (not the entire methodology) to confirm remediation.

## Depth vs. Coverage

Two failure modes show up repeatedly in bug bounty submissions:

| Failure Mode | Symptom | Fix |
|---|---|---|
| **Coverage without depth** | Running an automated scanner across the whole scope and submitting every flagged item without validation | Use scanners for triage, not conclusions. Validate manually before writing anything up. |
| **Depth without coverage** | Spending days on one endpoint while ignoring 90% of the attack surface | Time-box initial recon and mapping before diving deep on any single finding. |

The methodology guides are structured to force both: a mapping phase that forces coverage, and a validation phase that forces depth before a finding is considered reportable.

## How Methodology, Checklists, and Payloads Relate

These three content types are deliberately separated by purpose:

- **Methodology** (`methodology/`) answers "in what order, and why."
- **Checklists** (`checklists/`) answer "what specifically should I check."
- **Payloads** (`payloads/`) answer "what exact input do I send, and how do I know it worked."

A methodology document will reference the relevant checklist and payload files rather than duplicating their content. If you find yourself needing a payload that isn't linked from the methodology you're following, check `payloads/` directly — coverage there is broader than any single methodology's citations.

## Evidence Standards

Every validated hypothesis needs evidence before it becomes a reported finding. Minimum evidence standards used throughout this repository:

- **Reflected/stored injection**: full HTTP request and response showing the payload and its effect, plus a screenshot of triggered behavior (alert box, DOM mutation, database error).
- **Access control issues**: paired requests showing the same resource accessed with two different privilege levels, with response bodies or status codes diffed.
- **Blind/out-of-band issues**: interaction log from your OOB listener (Burp Collaborator, interactsh, or equivalent) with timestamps correlated to the triggering request.
- **Business logic issues**: a narrative walkthrough plus request/response pairs for each step in the abused workflow.

## When to Stop

Stop testing a hypothesis and move to reporting once you have:

1. Reproduced the issue at least twice, ideally from a clean session.
2. Ruled out the most obvious alternative explanation (caching artifact, client-side-only effect with no server impact, browser-specific quirk with no security relevance).
3. Determined the realistic impact — not the theoretical worst case, but what an attacker could actually achieve given the preconditions.

Continuing to probe a confirmed vulnerability for additional impact is reasonable within scope and rules of engagement, but should not delay reporting a confirmed, in-scope, high-severity finding.
