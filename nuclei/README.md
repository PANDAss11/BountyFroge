# nuclei Templates

Custom [nuclei](https://github.com/projectdiscovery/nuclei) templates aligned with the checklists and methodologies in this repository. These supplement, not replace, the official [nuclei-templates](https://github.com/projectdiscovery/nuclei-templates) community catalog — install that catalog too for broad CVE and misconfiguration coverage.

## Usage

```bash
# Update nuclei and the official template catalog first
nuclei -update
nuclei -update-templates

# Run BountyForge's custom templates against an authorized target
nuclei -u https://target.example -t nuclei/ -severity medium,high,critical

# Combine with the official catalog
nuclei -u https://target.example -t nuclei/ -t ~/nuclei-templates/ -severity medium,high,critical
```

> **Warning**
> Only run these templates against hosts explicitly covered by your engagement's authorized scope. Several templates below use out-of-band interaction checks or timing-based detection, which can generate alerts on the target's defensive monitoring even when non-destructive.

## Template Index

| Template | Detects | Technique |
|---|---|---|
| `exposed-swagger-docs.yaml` | Publicly accessible API documentation at common paths | Path probing (`/swagger.json`, `/openapi.json`, `/api-docs`) |
| `ssrf-metadata-oob.yaml` | SSRF via cloud metadata endpoint, blind variant | Out-of-band interaction on parameters matching common URL-fetch field names |
| `graphql-introspection-enabled.yaml` | GraphQL introspection left enabled | POSTs an introspection query, checks for a schema in the response |
| `git-directory-exposed.yaml` | Exposed `.git` directory leaking source history | Path probing for `/.git/HEAD` with content-type verification |
| `host-header-injection-reflected.yaml` | Host header value reflected unsanitized in response body | Injects a marker value in the `Host` header, checks for reflection |

## Writing New Templates

New templates should follow the [official nuclei template writing guide](https://docs.projectdiscovery.io/templates/introduction) and, per `CONTRIBUTING.md`, must be non-destructive by default (no state-changing requests without an explicit `-dast` style opt-in flag pattern matching nuclei's own conventions for active testing templates).

Minimum required template metadata fields for any submission to this directory:

```yaml
id: template-id-here
info:
  name: Human-readable name
  author: your-handle
  severity: info | low | medium | high | critical
  description: One or two sentences on what this detects and why it matters.
  reference:
    - https://link-to-relevant-advisory-or-cheatsheet
  classification:
    cwe-id: CWE-XXX
  tags: relevant,comma,separated,tags
```

Test every new template against both a confirmed-vulnerable local lab instance and a confirmed-patched instance before submitting, to minimize false positive/negative rates. Include the lab setup you used to validate it in the pull request description.
