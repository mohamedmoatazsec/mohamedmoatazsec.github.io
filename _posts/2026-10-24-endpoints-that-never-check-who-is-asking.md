---
title: "The quietest API bug: endpoints that never check who's asking"
description: A data-returning endpoint that forgot authentication leaks records in bulk. Here's the one-line test that finds it, and why "it's only tracking data" is the wrong defence.
date: 2026-10-24 09:00:00 +0300
category: Field notes
tags: [api-security, broken-access-control, information-disclosure, owasp-api]
---

The most dangerous API endpoints I find are not the ones with clever bugs. They are the ones
that answer everyone. No token, no cookie, no session — just a URL that returns data to whoever
asks. This is a note on a pattern I keep seeing on production targets, written without naming
the program it came from.

<!--more-->

## The one-line test

Before I look for anything subtle in an API, I ask every data-returning endpoint one question:
*does it actually require a session, or does it just assume the browser will send one?* The two
are not the same, and the gap between them is where this bug lives.

The test is to make the request with the credentials deliberately stripped:

```js
// Ask the endpoint with NO cookies at all. If it still answers with data,
// it never required a session — the browser was just sending one by habit.
const res = await fetch("https://app.example.com/api/<path>", { credentials: "omit" });
console.log(res.status, Array.isArray(await res.clone().json()));
```

A `200 OK` with a full JSON body to a request that sent nothing is the whole finding. There is
no filter to bypass and no token to forge, because there was never a check in the first place.
The endpoint was protected only by the fact that nobody had tried removing the cookies.

## What I found

On a recent production target — an enterprise portal, reported through a private program whose
name I'm withholding — an internal "tracking/export" endpoint behaved exactly like that. One
unauthenticated request returned **thousands of records** in a single response: no pagination, no
rate limit, no authentication of any kind.

The records were not harmless counters. They carried activity associated with distinct user
identifiers, and a sibling endpoint on the same API returned a mix of **internal staff email
addresses, external contact email addresses, business registration identifiers, and the
relationship between the two** — effectively a small graph of who-deals-with-whom, available to
an anonymous visitor. I confirmed the counts, took the minimum evidence needed to prove the
exposure, and stopped. Reading one page proves the bug; enumerating the dataset would have been
the attack.

## Why "it's only tracking data" is the wrong defence

The most common pushback on findings like this is that the endpoint "only" returns analytics or
tracking data, not the crown jewels. That misreads the risk in three ways:

- **Identifiers are PII when they identify people.** An email address is personal data on its
  own. A pseudonymous ID becomes personal the moment it can be joined to one.
- **Scale changes the category.** One leaked record is an incident. Thousands returned in a
  single unauthenticated request, with no rate limiting, is a dataset — and datasets get scraped,
  correlated and sold.
- **Relationships are the valuable part.** A list of staff emails is a phishing target list. A
  list of staff emails *linked to* the external contacts they work with is a phishing script.

The severity of an information-disclosure bug is not the sensitivity of one field. It is what an
attacker can assemble from the whole response, at the scale the endpoint will hand it over.

## How this gets built by accident

Nobody decides to publish this data. It happens when authorisation lives in the wrong layer. A
front-end only ever calls the endpoint from a logged-in page, so in testing the request always
*happens* to carry a session, and the endpoint is shipped trusting that it always will. The
check that should sit on the server — "who is asking, and are they allowed this?" — was never
written, because the UI made it look unnecessary.

This is OWASP API Security's **Broken Object Level Authorization** and **Broken Function Level
Authorization** in their plainest form: the server returns objects without confirming the caller
is entitled to them.

## Remediation

- **Require authentication and authorisation at the API layer, per request.** Never rely on the
  caller being a logged-in page. The endpoint must establish identity itself and return only what
  that identity is entitled to — or `401` / `403`.
- **Audit the siblings.** If one route under `/api/...` skipped the check, the ones next to it
  were probably written by the same hand on the same assumption. Finding one of these is a reason
  to review the whole surface, not to patch a single path.
- **Scope internal and export endpoints tightly.** "Tracking", "export" and "all-*" endpoints are
  the ones that return data in bulk, which makes them the ones that hurt most when they are open.
- **Rate-limit data-returning endpoints** so that even a correctly authorised caller cannot pull
  the entire dataset in one request.

The fix is rarely hard. The lesson is where the check belongs: authorisation that lives in the
interface is decoration. The server has to ask who's calling, every time, because eventually
someone will call it with the cookies turned off.
