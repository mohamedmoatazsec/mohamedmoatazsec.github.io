---
title: "CAPTCHA isn't a fix for an account-existence oracle"
description: A password-recovery flow answered differently for real and fake accounts, and returned recovery metadata before anyone proved they owned the account. The CAPTCHA in front of it changed nothing.
date: 2026-10-24 17:00:00 +0300
category: Field notes
tags: [user-enumeration, information-disclosure, authentication, account-recovery]
---

A password-recovery form is supposed to give the same answer to everyone: "if that account
exists, we've sent it instructions." When it answers *differently* depending on whether the
account is real, it becomes an oracle — a way for an anonymous visitor to confirm who has an
account. I found one recently where a CAPTCHA was treated as the control that made this safe. It
wasn't. Target withheld; both test addresses were mine.

<!--more-->

## What an account-existence oracle is

An oracle is any observable difference that answers a yes/no question you shouldn't be able to
ask. For account recovery, the question is "does an account exist for this email?" The difference
that answers it can be anything the server does differently for the two cases:

```http
POST /api/recover/lookup        → 200  { "status": "ok", ...recovery metadata... }   # exists
POST /api/recover/lookup        → 500  { "error": "user_not_found" }                 # does not
```

Here the difference was not subtle — a different HTTP status *and* a different body. Submitting an
address that had an account returned `200` with recovery metadata attached. Submitting an address
with no account returned a `500` with `user_not_found`. From the outside, those two responses are
a reliable "yes" and "no" for any email you care to try.

Worse than the yes/no: the "yes" response carried **recovery-state metadata** — masked name
fields, which recovery channels existed, sharing and recovery-attempt flags — returned *before*
the visitor had proved they owned the account at all. The oracle tells you an account exists; the
metadata starts describing it.

## The CAPTCHA was doing a different job

The flow was gated by a CAPTCHA, and that is clearly why the behaviour was considered acceptable.
But a CAPTCHA and an information-disclosure bug solve different problems:

- A **CAPTCHA** raises the cost of *automation*. It makes enumerating ten million addresses slow
  and annoying.
- An **account-existence oracle** is a problem of *information*. The difference in the response is
  still there whether you solve one CAPTCHA by hand or a million with a farm.

For targeted abuse, automation is not the threat model. An attacker who wants to know whether one
specific person — a named executive, a journalist, a person of interest — has an account here
will happily solve one CAPTCHA to find out. The anti-automation control does nothing about the
single, deliberate, high-value query. Rate-limiting and CAPTCHA reduce the *volume* of a leak;
they do not close one.

## Impact

- **Enumeration of who has an account**, one deliberate query at a time, which feeds targeted
  phishing ("we know you bank/host/work here").
- **Pre-authentication disclosure of recovery state**, which tells an attacker which recovery
  avenues to attack and gives them account details before proving any control.

I demonstrated the difference with two addresses I control — one enrolled, one owned but never
registered — and stopped there. I did not attempt recovery or takeover; the finding is the
disclosure, not an account compromise.

## Remediation

- **Return one response for both cases.** Same status code, same body, same timing:
  "If an account exists for that address, we've sent recovery instructions." The server knows
  which case it is; the client must not be able to tell.
- **Hold recovery metadata server-side** until the visitor proves control of an enrolled recovery
  channel. Nothing describing the account should appear before that proof.
- **Keep timing uniform.** A uniform body still leaks if "exists" takes 200 ms and "doesn't"
  takes 20 ms. Equalise the work, or defer it behind the proof step.
- **Treat CAPTCHA and rate-limiting as depth, not as the fix.** They belong in front of this
  endpoint — but as a cost control, layered on top of a flow that already gives nothing away.
  They are not a substitute for the responses being indistinguishable.

The one-line version, for anyone building a recovery flow: if an outsider can tell a real account
from a fake one by the response they get, you have an oracle — and no amount of anti-automation in
front of it changes what the response says.
