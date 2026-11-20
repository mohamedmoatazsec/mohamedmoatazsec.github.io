---
title: "The sign-in wall the client is allowed to switch off"
description: An AI shopping agent gated guests behind a sign-in wall after a few messages. The flag that turned the wall on was sent by the browser — swap one value and the server dropped the gate and loaded the signed-in tool set.
date: 2026-11-20 21:00:00 +0200
category: Write-up
tags: [broken-access-control, authentication, feature-flags, ai-agent, api-security]
report:
  program: Public program (name withheld pending disclosure)
  platform: HackerOne
  vulnerability: Broken authentication via client-controlled feature flag (CWE-284 / CWE-807)
  severity: Medium
  cvss: "5.3 (AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N)"
  status: Reported — under review
  reported: 2026-11-10
---

A consumer AI shopping agent lets anyone chat with it as a guest, then puts up a
"Sign in to continue" card after a handful of messages. That card is the whole free
tier's boundary. It turned out the server decided whether to show it based on a
feature flag the browser sends in the request body — and any client can set that
flag to the value used for signed-in sessions. One swap, and the gate stopped
applying and the authenticated agent skill loaded for a user who had never logged
in.

<!--more-->

## Summary

The agent enforces a guest turn limit: after roughly five messages, an
unauthenticated user gets an HTTP `422` with `guest_turn_limit_exceeded` and the UI
shows a sign-in wall. That decision is driven by a client-supplied feature flag in
the message body. With the guest flag present the server returns `422`; replace it
with the default (signed-in) flag — a value the client picks freely — and the same
unauthenticated session gets `200`, a full agent response, and the authenticated
buyer-agent skill loaded server-side. The sign-in card is a client-side check only;
the server does not independently verify authentication when the default flag is
supplied.

I withhold the program name, the production hostname and the live request-minting
steps while the report is under review.

## The application

The agent runs as a streaming chat. The browser opens an authenticated stream to the
agent host and POSTs each user message as JSON. Alongside the message content, the
body carries a `features` array — the client's view of which feature flags are
active for this session. Two of the values that matter:

- `shop_agent/guest` — the guest experience, subject to the turn limit.
- `shop_agent/default` — the experience used for normal (signed-in) sessions.

The turn limit, the sign-in wall and the set of tools the agent is allowed to call
are all downstream of which of those the request claims.

## Finding it

The sign-in wall behaves like a hard server boundary from the UI: you hit it, you're
stuck. But the request that *produces* the `422` is worth reading before you trust
it. The turn-limit decision wasn't keyed off the session identity — it was keyed off
a flag sitting in the request body next to the message text. Anything the client
sends, the client controls. So the test writes itself: send the same guest message
twice, once with each flag, and compare.

## Proof chain

The smallest A/B that proves the gate is client-controlled: one unauthenticated
session, one message, two flag values. Hostnames, tokens and session identifiers are
redacted; the stream-URL minting step is omitted while the report is open.

1. **Reach the wall as a guest.** In a private window, no sign-in, chat until the
   turn limit trips. The gated request returns:

   ```http
   POST /api/messages  (agent host)

   { "message": { "content": "…", "features": ["shop_agent/guest", …] } }
   ```
   ```http
   HTTP/1.1 422 Unprocessable Entity

   {"error":{"message":"Sign in to continue.","code":"guest_turn_limit_exceeded"}}
   ```

2. **Replay the same message with the default flag.** Identical session, identical
   content, one value changed:

   ```http
   POST /api/messages  (agent host)

   { "message": { "content": "…", "features": ["shop_agent/default", …] } }
   ```
   ```http
   HTTP/1.1 200 OK
   Content-Type: text/event-stream

   … tool_call_start: load_buyer_agent_skill …
   ```

3. **Confirm it reaches the thread.** Reload the conversation. The bypass turn and
   the agent's reply now sit in the thread above the still-present "Sign in to
   continue" card — while the session is still unauthenticated (the Sign-in button
   is still showing).

4. **Confirm the expanded surface, not just the status code.** Ask the agent what it
   can do, again with the default flag. It returns `200` and lists capabilities meant
   for signed-in buyers — past-order lookup, package tracking, asynchronous support —
   offered to the unauthenticated user.

## Impact

An unauthenticated user bypasses the guest turn limit by changing one client-supplied
flag, and in doing so causes the server to load the authenticated buyer-agent skill
and advertise its tool set.

**What this proves:** the turn gate is not enforced server-side — same session, `422`
for the guest flag and `200` for the default flag. The authenticated skill loads for
an unauthenticated caller (confirmed by the `load_buyer_agent_skill` tool-call event
in the stream), and the agent then offers support-ticket creation and lists order
lookup, package tracking and support as available.

**What it does not prove — and I'm not claiming it.** No order data or customer
information was returned during testing. The data-access tools appear to be bound to
the caller's identity, so the proven impact is the turn-limit bypass and the exposure
of the authenticated tool set to an anonymous user — not account data access. Severity
reflects that: Medium, CVSS 5.3.

## Remediation

- **Decide authorisation on the server, from the session — never from a client flag.**
  Feature flags describe presentation and rollout; they must not be the thing that
  decides whether an auth gate applies. The turn limit and the signed-in skill set
  should be gated on the authenticated identity the server establishes for the
  request, and a guest session must get the guest result regardless of what flags the
  body carries.
- **Treat the flag array as untrusted input.** Resolve the effective feature set
  server-side for the session; don't let the request assert its own.
- **Audit sibling flag-driven branches.** If one capability boundary was keyed off a
  client-sent flag, others written the same way probably exist. The fix is a pattern
  review, not a single `if`.

## Timeline

| Date       | Event             |
|------------|-------------------|
| 2026-11-10 | Reported          |
| —          | Under review      |

## Lessons

Feature flags are a client-side concept that quietly became a security boundary. The
moment "which experience is this" and "is this user allowed past the gate" are the
same value — and that value rides in the request body — the gate belongs to whoever
is holding the browser. When you find an auth wall, don't just check whether you can
climb it. Check who gets to decide it's there.
