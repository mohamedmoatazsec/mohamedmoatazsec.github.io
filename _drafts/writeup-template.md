---
title: "IDOR in /api/v2/invoices exposed every customer's billing history"
description: One sentence a product manager would understand. This becomes the meta description and the card excerpt lede.
category: Write-up
tags: [idor, broken-access-control, api]
# The `report` block renders the "Report summary" panel at the top of the post
# and the severity badge on the cards. Delete any line you do not want shown.
report:
  program: Private program (name withheld)
  platform: HackerOne
  vulnerability: Insecure Direct Object Reference (CWE-639)
  severity: High            # Critical | High | Medium | Low | Informational
  cvss: "7.5 (AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:N/A:N)"
  status: Resolved
  reported: 2026-01-10
  resolved: 2026-01-24
  disclosed: 2026-03-01
  bounty: Undisclosed
---

<!--
  To publish: move this file to _posts/YYYY-MM-DD-slug.md, set `date:` if you want
  a time other than midnight, and make sure the program's disclosure policy allows
  the level of detail below. The URL becomes /writeups/YYYY/slug/.

  Everything above the "more" marker (the HTML comment after the first paragraph)
  is the excerpt shown on the cards.
-->

Two or three sentences: what the application does, what was broken, and who could
abuse it. Lead with the impact, not the endpoint.

<!--more-->

## Summary

One paragraph: what the bug is, where it lives, who can exploit it and what they get.

## The application

Enough context to follow the rest: the feature involved, the roles that exist, and
how the API behind the UI is structured. Skip anything the reader does not need.

## Finding it

How the behaviour first looked suspicious. Keep the narrative honest; dead ends are
useful to other hunters.

## Proof chain

The smallest ordered set of requests that takes an attacker from nothing to the
impact. Every link is a real request and the relevant lines of its response.

1. **Create two accounts.** `attacker@` and `victim@`, both on the free plan.
2. **As the victim, generate the object.** Capture the request and note the ID.

   ```http
   GET /api/v2/invoices/48213 HTTP/1.1
   Host: app.example
   Authorization: Bearer <victim token>
   ```

3. **As the attacker, replay with the victim's ID.**

   ```http
   GET /api/v2/invoices/48213 HTTP/1.1
   Host: app.example
   Authorization: Bearer <attacker token>
   ```

   ```http
   HTTP/1.1 200 OK
   Content-Type: application/json

   {"id": 48213, "customer": "victim@…", "total": "…"}
   ```

4. **Prove ownership.** Show, from the victim's side, that the data returned is the
   victim's and that the attacker could not otherwise see it.

## Impact

The one-sentence version, then the realistic worst case. State the preconditions
plainly: authentication required or not, which plan, user interaction or none.

## Remediation

What the owning team should change, in their terms, plus the general class so they
can look for siblings elsewhere in the codebase.

## Timeline

| Date       | Event                          |
|------------|--------------------------------|
| 2026-01-10 | Reported                       |
| 2026-01-11 | Triaged                        |
| 2026-01-24 | Fix deployed, verified         |
| 2026-03-01 | Disclosure agreed with program |

## Lessons

What this one taught me, or what I would do differently next time.
