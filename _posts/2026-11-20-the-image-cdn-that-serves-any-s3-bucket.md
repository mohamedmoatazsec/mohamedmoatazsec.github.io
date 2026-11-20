---
title: "The image CDN that would serve anyone's S3 bucket as first-party"
description: Two image hostnames fronted a generic S3 endpoint and forwarded the path unchanged. S3 read the first path segment as a bucket name — so any public bucket in the world was served under the brand's own domain, with script execution and parent-domain cookie access.
date: 2026-11-20 22:30:00 +0200
category: Write-up
tags: [subdomain-takeover, s3, xss, cdn, cookie-scope, broken-access-control]
report:
  program: Public program (name withheld pending disclosure)
  platform: HackerOne
  vulnerability: Arbitrary content/script execution via path-style S3 origin on a first-party host (CWE-79 / CWE-16)
  severity: Medium
  cvss: "6.1 (AV:N/AC:L/PR:N/UI:R/S:C/C:L/I:L/A:N)"
  status: Reported — under review
  reported: 2026-11-14
---

Two image hostnames on a well-known brand's domain sat behind a CDN whose origin was
the *generic* Amazon S3 endpoint — not one specific bucket. The CDN passed the
request path through unchanged, and S3 reads the first path segment of a generic-
endpoint request as the bucket name. The result: anyone could create a free S3
bucket and have its contents served under the brand's own hostnames — as HTML, with
no security headers — giving script execution in those origins and read/write access
to cookies scoped to the parent domain. I'm withholding the brand, the real
hostnames and the live proof bucket while the report is under review.

<!--more-->

## Summary

The two hosts — call them `image.brand.example` and `image-s.brand.example` — fronted
a CDN whose origin was the path-style S3 endpoint (`s3.amazonaws.com`) rather than a
single virtual-hosted bucket (`<bucket>.s3.amazonaws.com`). Because the path was
forwarded verbatim, a request for `https://image.brand.example/<bucket>/<object>`
reached S3 as a request for object `<object>` in bucket `<bucket>`. Any **public**
bucket on AWS was therefore reachable through the brand's hostnames, and objects came
back with the content type the bucket owner chose — `text/html` included — and
without `Content-Security-Policy`, `X-Content-Type-Options` or `Content-Disposition`.
That turns a static image host into an origin where an attacker runs arbitrary
JavaScript and touches `.brand.example` cookies.

## The application

Nothing about the feature was unusual: an image/asset host behind a CDN. The whole
bug is one origin-configuration choice. A CDN pointed at S3 can be wired two ways:

- **Virtual-hosted origin** — origin is `mybucket.s3.amazonaws.com`. The bucket is
  fixed; the path only ever selects an object *inside* `mybucket`.
- **Path-style origin** — origin is `s3.amazonaws.com`. Now the first path segment
  *is* the bucket selector, and whoever controls the path controls the bucket.

With the second wiring and an unchanged forwarded path, the host no longer serves
"our images." It serves "whatever bucket the URL names."

## Finding it

The tell was a response header: the hosts returned `Server: ESA` (the CDN) but the
body and error shapes were S3's. The quickest confirmation of a path-style origin is
to ask for a bucket that cannot exist and read the error. If the service answers with
S3's own `NoSuchBucket` XML naming your made-up segment as the bucket, the path is
selecting buckets — the origin is generic, not pinned.

## Proof chain

Bucket names and the real hostnames are redacted. A single harmless HTML file proves
script execution; nothing else was needed.

1. **Stand up a control.** Create a public bucket `attacker-bucket` with one file,
   `poc.html`, that prints `document.domain`. Served from S3 directly it reports its
   own origin:

   ```http
   GET /poc.html HTTP/1.1
   Host: attacker-bucket.s3.amazonaws.com
   ```
   ```
   document.domain = attacker-bucket.s3.amazonaws.com
   ```

2. **Serve the same object through the brand host.** Same bucket, same file, reached
   via the first path segment:

   ```http
   GET /attacker-bucket/poc.html HTTP/1.1
   Host: image.brand.example
   ```
   ```http
   HTTP/1.1 200 OK
   Content-Type: text/html
   Server: ESA
   ```
   ```
   document.domain = image.brand.example
   ```

   The script runs, and it reports the **brand's** origin — not S3's. The second host
   behaves identically.

3. **Confirm the root cause.** Request a non-existent bucket and read S3's own error:

   ```http
   GET /this-bucket-does-not-exist-7731/x.html HTTP/1.1
   Host: image.brand.example
   ```
   ```xml
   <Error><Code>NoSuchBucket</Code>
     <BucketName>this-bucket-does-not-exist-7731</BucketName></Error>
   ```

   An unrelated third-party public bucket resolves the same way, which shows the
   reach isn't limited to buckets the tester owns.

4. **Check the headers.** The HTML response carried no `Content-Security-Policy`, no
   `X-Content-Type-Options: nosniff` and no `Content-Disposition`. Nothing told the
   browser to treat the response as a download or to refuse to execute it.

5. **Read parent-domain cookies.** From a script running on `image.brand.example`,
   enumerate cookie *names* visible to the origin. Several were present — analytics
   and visitor identifiers, an anti-bot token — all set by sibling applications on
   `.brand.example` and readable here. Only names were recorded; values were not
   touched.

6. **Write a parent-domain cookie and affect a sibling app.** Set a cookie scoped to
   `.brand.example` from the image origin:

   ```js
   document.cookie = 'lg_mode=pocAAA; domain=.brand.example; path=/; secure'
   ```

   A different application on the same parent domain reads `lg_mode` as its language
   setting; with the planted value it returned `HTTP 400` for that visitor until the
   cookie was cleared. Clearing it (`max-age=0`) restored the app. That's one concrete
   instance of a general primitive: any sibling app that trusts a parent-domain cookie
   without validating it can be influenced from the image host.

## Impact

An unauthenticated attacker needs only a free public S3 bucket and a link the victim
opens:

- **Script execution in two first-party origins.** Arbitrary JavaScript on the
  brand's own image hostnames.
- **Phishing on genuine brand URLs.** A fake login or payment page is served from a
  real brand hostname, with a valid brand certificate and HSTS — which defeats
  "check the address bar" and URL-reputation filters.
- **Read access to parent-domain cookies.** Analytics identifiers and anti-bot tokens
  scoped to `.brand.example` are readable by the attacker's script.
- **Write access to parent-domain cookies.** The attacker plants cookies every
  `*.brand.example` app receives; one such cookie broke a sibling app for the visitor
  until cookies were cleared.
- **Arbitrary third-party content under the brand's name** — malware or illegal
  material served from a trusted hostname.

**What I did not find — stated plainly.** No authentication or session cookie was
readable from these origins (the account session lives on a different parent domain),
and the API gateway did not accept these origins for cross-origin requests. The proven
impact is first-party script execution, parent-domain cookie read/write, and the
phishing surface that follows — not account-session theft.

## Remediation

- **Pin the origin to one bucket.** Configure the CDN origin as a virtual-hosted
  bucket (`<bucket>.s3.amazonaws.com`) or a fixed path prefix, so the request path can
  no longer choose the bucket. This is the fix; the rest is defence in depth.
- **Serve only expected types.** For anything that isn't an image, add
  `X-Content-Type-Options: nosniff`, `Content-Security-Policy: default-src 'none'; sandbox`,
  and `Content-Disposition: attachment`.
- **Validate cookies at the consumer.** The sibling app should check `lg_mode` against
  its allowed values and fall back to a default rather than erroring — and that lesson
  generalises to every app trusting a parent-domain cookie.

## Timeline

| Date       | Event             |
|------------|-------------------|
| 2026-11-14 | Tested & reported |
| —          | Under review      |

## Lessons

"It's just an image host" is how this gets shipped. The severity didn't come from the
asset host at all — it came from the host sharing a cookie namespace with login,
analytics and anti-bot on the same parent domain. A path-style S3 origin quietly
handed that shared origin to anyone with a bucket. When you see a CDN in front of S3,
the question isn't whether it caches images. It's which bucket the path is allowed to
name — and what else lives on the domain it's borrowing.
