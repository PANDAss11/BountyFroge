# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/) for tagged releases (release tags refer to content milestones, not software versions).

## [Unreleased]

### Added
- Initial repository scaffold: report templates, checklists, methodology guides, payload references, and GitHub community files.
- Report templates: XSS, SQL Injection, IDOR, SSRF.
- Checklists: Recon, Authentication, API Security.
- Methodology guides: Reconnaissance, API Testing.
- Payload references: XSS, SSRF.
- Fictional-company example report demonstrating full template usage.
- GitHub Actions workflows for Markdown linting, link checking, and spell checking.

### Planned
- Remaining report templates: CSRF, SSTI, XXE, LFI, RCE, Open Redirect, Host Header Injection, Race Condition, Cache Poisoning, GraphQL, OAuth, JWT, Business Logic, File Upload, Authorization, general Authentication depth pass.
- Remaining checklists: Authorization, JWT, OAuth, REST, SOAP, GraphQL, AWS, Azure, GCP, Docker, Kubernetes, Web, Mobile, CI/CD, SAML, CORS, CSP, Rate Limiting, Session Management, Password Reset, Business Logic.
- Remaining methodology guides: Subdomain Enumeration, Web Testing, Authentication Testing, Authorization Testing, GraphQL Testing, Cloud Testing, JavaScript Analysis, Source Code Review.
- Remaining payload references: SSTI, XXE, Open Redirect, Host Header Injection, SQLi, CRLF, CORS, GraphQL, JWT.
- Additional nuclei template pack and Burp Suite configuration exports.
- Additional fictional-company example reports covering GraphQL and cloud misconfiguration scenarios.

## [0.1.0] - 2026-07-17

### Added
- Project initialized under MIT License.
- Core community health files: `CONTRIBUTING.md`, `SECURITY.md`, `CODE_OF_CONDUCT.md`.
- Base documentation set under `docs/`: getting started, methodology overview, reporting guide, FAQ.
- Repository branding assets (logo, banner, social preview) as SVG.

[Unreleased]: https://github.com/bountyforge/BountyForge/compare/v0.1.0...HEAD
[0.1.0]: https://github.com/bountyforge/BountyForge/releases/tag/v0.1.0
