---
title: "How I document a finding: the proof chain"
description: The rule I apply to every bug bounty submission, and the checklist behind it.
date: 2026-10-17 18:00:00 +0300
category: Methodology
tags: [methodology, bug-bounty, reporting]
---

I have one rule for bug bounty reports: nothing goes out until the full proof chain is
documented and confirmed working, request to response. Not "this parameter looks injectable".
Not "the same endpoint probably accepts another user's ID". A chain I can replay from a clean
session and that a triager can replay without asking me anything.

<!--more-->

This sounds obvious. It isn't, judging by how many reports get bounced for "needs more info".
Here is what the rule means in practice, and the checklist I run before I press submit.

## What a proof chain is

A proof chain is the smallest ordered set of HTTP exchanges that takes someone from *nothing* to
*the impact*. Every link in it is a real request I sent and a real response I received, pasted
in full, with the one or two lines that matter called out.

For an authorisation bug that usually looks like this:

1. Create account A and account B.
2. As A, perform the legitimate action and capture the request.
3. As A, replay that request with B's identifier.
4. Show the response contains B's data, and show (from B's session) that the data really is B's.

Step 4 is the one people skip. A `200 OK` with some JSON in it proves nothing on its own. The
report has to tie the leaked object to a victim the attacker could not otherwise see.

## The checklist

Before a report leaves my drafts folder, every line below has a yes.

**Reproducibility**

- Does the chain work from a fresh session, with no cookies or tokens left over from testing?
- Did I reproduce it twice, at least a few minutes apart, to rule out a transient state?
- Are the two test accounts mine, created for this program, and named in the report?
- Is every request shown in full: method, path, headers that matter, body? No `...` in the
  middle of the part that proves the bug.

**Impact**

- Can I state the impact in one sentence a product manager would understand?
  "An attacker with any free account can download any other customer's invoices" is impact.
  "IDOR on `/api/v2/invoices/{id}`" is a location.
- Is the severity I am claiming the severity the program's policy would assign, not the one I
  would like? If the program uses CVSS, I fill the vector in and let the number be what it is.
- What are the preconditions? Authenticated or not, any plan or a paid one, user interaction or
  none. Overstating this is the fastest way to lose a triager's trust.

**Scope and conduct**

- Is the asset in scope today, not in a cached copy of the policy from last month?
- Did I stop at proof? Reading one record of my own second account is proof. Enumerating
  other customers is not research, it is the attack.
- Did I avoid anything that could degrade the service: no automated bursts against
  authentication, no deleting, no mass requests to confirm what one request already showed?

**Remediation**

- Have I written a fix the owning team can act on, in their terms?
  For the invoice case: "Resolve the invoice by `id` *and* the authenticated account; return
  404 when the pair does not exist, the same way the UI already does." That is one sentence they
  can turn into a ticket.
- Did I mention the general class so they can look for siblings? If one object type skipped the
  ownership check, the others deserve a look.

## Why I bother

Three reasons, in order of how much they matter to me.

**It gets bugs fixed faster.** A report that needs no clarification goes from "new" to
"triaged" in one pass. Every round of questions is a day or a week the vulnerability stays live.

**It is the job.** On an engagement, the deliverable is the report, not the finding. A finding
the client cannot reproduce after you leave has no value to them. Bug bounty is practice for
that, with real stakes.

**It keeps me honest.** Writing the chain out is where I find out whether I actually have a
bug or just a suspicious response. More than once the write-up is what showed me the "leak" was
my own account's data coming back through a path I had not noticed.

## The template I use

Plain Markdown, the same shape every time, so the triager's eye knows where to look:

```markdown
## Summary
One paragraph: what, where, who can exploit it, what they get.

## Severity
Claimed severity + CVSS vector, with a sentence on the preconditions.

## Steps to reproduce
1. ...
2. ...
(every step has the request and the relevant response lines)

## Proof
The response that shows victim data, with the victim-side screenshot
or request that proves ownership.

## Impact
The one-sentence version, then the realistic worst case.

## Remediation
The fix in the team's terms, plus the class of bug to look for elsewhere.
```

Nothing clever. The value is in refusing to submit until every section is full.

If you hunt and your reports keep coming back with questions, try running the checklist above
against your last three submissions. The gaps are usually in *proof of ownership* and
*preconditions*, and both are fixable in ten minutes before you submit instead of three days
after.
