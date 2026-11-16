---
title: "SecureScope 2.0: a rewrite around evidence-backed findings"
description: The tool went from a header checker to an evidence-backed scanner with grades, a CSP evaluator, SARIF output and a CI gate. What changed, and the one idea it's all built on.
date: 2026-10-24 17:30:00 +0300
category: Tooling
tags: [securescope, tooling, python, csp, sarif, ci]
---

[SecureScope](https://github.com/mohamedmoatazsec/SecureScope) is my open-source web security
posture scanner: point it at a site you're authorised to test and it reviews the transport layer
and every security-relevant response header the way a reviewer would. It's now at **2.0**, and
it's less an update than a rewrite. Here's what changed and the single idea the whole thing is
built on.

<!--more-->

## One idea: every result is an evidence-backed finding

The 1.x tool produced a report. The 2.0 tool produces *findings*, and everything else derives
from them. Each finding carries a stable id, a severity, a confidence level, the **exact evidence
it saw**, a fix written for whoever owns the configuration, and CWE / OWASP / MDN references:

```text
HIGH   CSP allows inline scripts ('unsafe-inline')   name=Content-Security-Policy
       CSP-UNSAFE-INLINE · csp                        directive=script-src
```

Once a result is a structured finding with a stable id, everything downstream is just a view of
it — the console table, the grade, the HTML report, the SARIF file, the baseline diff. Nothing is
special-cased. That stable id (`CSP-UNSAFE-INLINE`, `HSTS-SHORT-MAX-AGE`, …) is also what
baselines and SARIF rules key on, so a finding means the same thing everywhere.

## A real CSP evaluator, not a presence check

The biggest single addition. SecureScope 1.x checked whether headers were *there*. 2.0 checks
whether they're configured so they actually protect anyone. For Content-Security-Policy that
means evaluating the policy: missing or report-only policies, no script restriction,
`'unsafe-inline'` without nonces or hashes, `'unsafe-eval'`, wildcard and scheme-only script
sources, missing `object-src` / `base-uri` / `form-action`, duplicate and unknown directives, no
reporting endpoint — and it understands `'strict-dynamic'` so it doesn't cry wolf on a modern
policy. The same "quality, not presence" rule now runs across HSTS, clickjacking protection,
cookies, CORS and TLS.

## Grades

Each target gets a 0–100 score and a letter grade, computed the simple way so it can be explained
in one sentence: start at 100, subtract 40 per critical, 20 per high, 10 per medium and 4 per low;
informational findings are free. `A+ ≥ 95`, down to `F`. A grade is not a substitute for reading
the findings — it's a way to tell, across many targets or across time, whether things are getting
better or worse.

## Reports, and a CI gate

The report writers now cover **JSON** (the canonical format, with a published schema), a
self-contained **HTML** report with severity filters and dark/light mode, **SARIF 2.1.0** for
GitHub code scanning, **Markdown** for PR comments and job summaries, and **CSV** for ticketing.
`-f` is repeatable, so one run can write several.

The part I use most is the CI gate:

```bash
securescope app.example.com --baseline reports/last-good.json --fail-on high
```

`--fail-on` exits non-zero when a finding is at or above a severity; `--baseline` diffs against a
previous JSON report so that **only new findings fail the build**. That's what makes it adoptable
on an existing site: run once, commit the report as the baseline, gate on regressions, and burn
the baseline down at your own pace instead of failing every pipeline on day one. The repo doubles
as a GitHub Action (`uses: mohamedmoatazsec/SecureScope@v2`) that writes the Markdown report to
the job summary and uploads SARIF to the security tab.

## The scanner's own attack surface

A security tool points itself at hostile hosts, so the target's response is attacker-controlled
input. 2.0 (and the 2.0.1 hardening release) treats it that way:

- **Everything from the target is escaped in every output** — console, HTML, Markdown and CSV
  (including CSV formula injection). This started as the 1.1 HTML fix; now it's a rule across the
  board, pinned by tests.
- **Redirects are followed by hand, at most five hops, and never into loopback, link-local or
  private address space** — otherwise a scanner running in CI would be a way to reach cloud
  metadata endpoints and internal services. A refused hop is reported as
  `REDIRECT-TO-INTERNAL-ADDRESS`.
- **Response bodies are capped at 2 MB and evidence at 1000 characters**, so a target can't
  exhaust memory or bloat a report.

It still never sends state-changing requests, and the noisier recon (ports, subdomains, WHOIS) is
opt-in — the default scan is HTTP and TLS only.

## Install

```bash
pip install securescope
securescope app.example.com --fail-on high -f sarif -f html -o reports/
```

Python 3.9+, MIT-licensed, 75 offline tests, and a `SECURITY.md` for reporting an issue in the
tool itself. The code is at
[github.com/mohamedmoatazsec/SecureScope](https://github.com/mohamedmoatazsec/SecureScope).

As always: run it only against systems you own or have explicit, written permission to test, and
treat findings as configuration observations to verify, not as proof of exploitability.
