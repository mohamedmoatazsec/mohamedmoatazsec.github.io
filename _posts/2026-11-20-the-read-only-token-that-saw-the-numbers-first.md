---
title: "The read-only token that saw the quarterly numbers first"
description: A non-expiring CMS auth token was hard-coded in a public JavaScript bundle. It was read-only — but scoped to the whole corporate asset store, so it could list draft quarterly reports uploaded to the CMS before their market-regulated release.
date: 2026-11-20 23:00:00 +0200
category: Write-up
tags: [secrets-in-code, information-disclosure, access-control, scope, api]
report:
  program: Public program (name withheld pending disclosure)
  platform: Intigriti
  vulnerability: Hard-coded non-expiring CMS token with over-scoped read access (CWE-798 / CWE-200)
  severity: Medium
  cvss: "5.3 (AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N)"
  status: Reported — in mediation
  reported: 2026-11-10
---

A permanent CMS auth token for a company's production content system was hard-coded
in a client-side JavaScript bundle that anyone could download from a public cloud
bucket. The token had no expiry, had been sitting in the bucket across dozens of
builds for years, and — though read-only — was scoped to the *entire* corporate asset
store. That scope is the finding: it could list draft quarterly financial reports the
moment they were uploaded to the CMS, days before those reports were released to the
market. I'm withholding the company, the token and every identifier that would let
someone reach it while the report is in mediation.

<!--more-->

## Summary

The front-end for one application loaded a headless-CMS (GraphCMS/Hygraph-style)
**permanent auth token** embedded directly in its compiled JS bundle. The bundle was
served from a public object-storage bucket, so the token was retrievable by anyone,
unauthenticated. The token carried **no expiry** and had been present across many
successive builds for roughly three years.

It was read-only — but it was scoped to the shared CMS project behind the corporate
site, which gave it draft-stage read access to the whole asset store. Without the
token, the asset query returns `403 not allowed`; with it, the same query returns
every asset, including documents that were uploaded as unpublished drafts and never
published through the CMS — among them quarterly financial reports staged before their
regulated public release.

## The application

A headless CMS backs the corporate site. The front-end talks to the CMS's GraphQL
API, and for that it was shipped a *permanent* auth token rather than a short-lived,
narrowly-scoped one. Two things turned a convenience into an exposure:

- **The token was in client code, in a public bucket.** A secret that ships to the
  browser is not a secret; a secret in a world-readable bucket is a published one.
- **The token's scope was never narrowed.** A token issued for one application's own
  content models should not be able to read the finance/corporate asset store at all.
  On a shared CMS project, this one could.

## Finding it

Secrets in front-end bundles are a known class, so the asset store was the thing worth
checking: static analysis of the bundle surfaced a long JWT-shaped string. Decoding it
(header and payload only — the parts anyone can decode) showed a CMS management
audience, a project identifier, and, tellingly, **no `exp` claim**. A token with no
expiry, in a file anyone can fetch, is a standing key for as long as it is valid.

## Proof chain

All identifiers — the token, the bucket, the project ID, the bundle URL and every
asset URL — are withheld. I prove scope with a control, not with the sensitive data
itself.

1. **The token is retrievable with no authentication.** It's a static string inside a
   publicly downloadable bundle. Decoding its header and payload shows a CMS audience
   and no expiry.

2. **Confirm the store is protected — without the token.** The asset query, sent
   anonymously, is refused:

   ```json
   { "errors": [ { "message": "not allowed",
                   "extensions": { "code": "403", "path": ["assets"] } } ] }
   ```

3. **Same query, with the token — the store opens.** Authenticated with the leaked
   token, the identical query returns assets, including draft documents whose
   `publishedAt` is `null` (never published through the CMS — the token is the only API
   path to them). I confirmed the *existence, names and metadata* of pre-release
   financial reports this way. I did **not** download them (see below).

4. **Scope and lifetime.** The same token lists the full asset store — far beyond the
   one application's own content. Bundle listings show the token present from the
   earliest build in the bucket through the latest, spanning years.

## Impact

- **Pre-publication access to inside information.** The token lists draft assets the
  moment they're uploaded. For two quarters this year, that meant the quarterly report
  was enumerable roughly a couple of days and, in the other case, under a day before
  its market-regulated release — the kind of information that moves a listed issuer's
  securities.
- **Broad confidential-document exposure.** The whole corporate asset store is
  listable with direct download URLs, including internal signed policies and material
  never published through the CMS.
- **Persistence.** No expiry, publicly retrievable for years across dozens of builds;
  revocation is manual.
- **Scope misconfiguration.** A client-side token for one application could read a
  finance/corporate asset store on a shared CMS project — a boundary that was never
  drawn.

**What I did not do — stated plainly.** No mutation was attempted (a single harmless
write test was correctly denied with `403`). No financial report was downloaded — I
performed only a size/type check on one file that was *already public* — and nothing
was modified. Management-API access was tested and is not granted. The proven finding
is unauthenticated, over-scoped *read/list* access via a leaked permanent token, not
tampering.

## Remediation

- **Revoke the token now, and audit asset-URL access logs** around the two
  pre-release windows.
- **Re-issue a token scoped to the one application's own models** — or move that
  application to a separate CMS project. Never grant a client-side token draft-stage or
  asset-store read on a project that holds corporate or financial material.
- **Disable public object listing on the bucket** and review the other applications in
  it for embedded credentials.
- **Set an expiry on all CMS tokens and rotate them.** A permanent token in client code
  is a credential you have already published.

## Timeline

| Date       | Event             |
|------------|-------------------|
| 2026-11-10 | Reported          |
| —          | In mediation      |

## Lessons

"Read-only" did a lot of reassuring in the original framing, and it was the wrong thing
to focus on. The damage here wasn't write access — it was *what* the read-only token
could read, and *when*. A permanent credential shipped to the browser, in a public
bucket, scoped to a shared project that also holds pre-release financial filings: each
of those was individually ordinary, and together they put market-moving documents one
anonymous query away. When you find a token in a bundle, decode it for the expiry, then
ask the only question that matters — what is it allowed to see, and should it be?
