---
title: "One operator, one open redirect: when || skips your allowlist"
description: A redirect validator checked every URL against an allowlist — and a single `||` meant three of them were never checked at all. A code-review note from a real finding.
date: 2026-10-24 13:00:00 +0300
category: Field notes
tags: [open-redirect, logic-flaw, javascript, client-side]
---

An allowlist is only as good as the condition that enforces it. On a recent target I found a
redirect validator that checked four URLs against a 28-domain allowlist and still let three of
them through unchecked — not because the allowlist was wrong, but because of one logical
operator. Here's the pattern, generalised, with the target withheld.

<!--more-->

## The setup

A single-page app took several URLs from the query string, stored them in `sessionStorage`, and
later handed one to `window.open()` when the user clicked a button at the end of a flow. Passing
user-supplied URLs into a navigation sink is fine *if* every one of them is validated first. The
app did validate — it had a proper allowlist check, `isAllowed(url)`, applied against a list of
legitimate domains. The bug was in how the check was wired up.

## The bug, generalised

Reduced to its shape, the validation looked like this:

```js
// primary + three alternates all come from the query string.
if (
  isAllowed(primary) ||
  (alt1 && alt2 && alt3 && isAllowed(alt1) && isAllowed(alt2) && isAllowed(alt3))
) {
  store({ primary, alt1, alt2, alt3 }); // all four saved for later navigation
}
```

Read the condition the way JavaScript evaluates it. `||` short-circuits: if the left side is
true, the right side is **never evaluated**. So the instant `primary` is an allowed domain, the
whole condition is true — and `alt1`, `alt2`, `alt3` are stored *without `isAllowed` ever being
called on them*.

The allowlist was correct. The 28 domains were fine. But three of the four inputs could be
anything, as long as the first one was legitimate. An attacker sets `primary` to a real allowed
domain to satisfy the left side, and points the alternates at their own URL. All four get stored;
the one that reaches `window.open()` is attacker-controlled.

## Finding it

This is not something you see from the outside — it lives in the client-side JavaScript. The
chain I followed:

1. **Notice the sink.** The flow ended in a `window.open()` call on a stored value. Any time
   user input reaches a navigation sink, the validation in between is worth reading.
2. **Read the (minified) source.** Un-minified enough to see the condition, and the `||` sitting
   between the single-URL check and the AND-chain for the rest.
3. **Prove the store, not just the theory.** Load the crafted URL, then read `sessionStorage` in
   the console and confirm the unvalidated alternates were saved — the bypass happens at page
   load, before any user action.
4. **Prove the navigation.** Complete the flow, click through, and confirm the browser lands on
   the controlled URL from the stored value.

Proving the *store* separately from the *navigation* matters. It shows exactly where the control
fails (validation at load) and that the final redirect is a consequence, not a coincidence.

## Why an open redirect is worth reporting

Open redirect gets dismissed as low severity, and on its own it often is. The impact here is
**trust transfer**: the link starts on a legitimate, recognisable domain, so it passes the
glance test — the victim sees the real brand in the address bar and in any preview. After a few
screens, their own click sends them somewhere else entirely. That is a phishing primitive: a
credential-harvesting or payment page that arrives wearing the brand's own domain as the referrer.
It also makes a tidy first link in a chain — redirecting into an OAuth flow, or to a page that
itself does something more interesting.

## Remediation

Validate every input independently, before storing any of them:

```js
if (isAllowed(primary) && isAllowed(alt1) && isAllowed(alt2) && isAllowed(alt3)) {
  store({ primary, alt1, alt2, alt3 });
}
```

Then the broader guidance:

- **Don't let one input vouch for another.** Each value that can reach the sink gets its own
  check. `||` between validations almost always means something is going unvalidated on one side.
- **Validate at the sink, not only at the door.** Even with the fix above, re-checking the value
  immediately before `window.open()` / `location = ...` closes the gap if stored state is ever
  set another way.
- **Prefer allowlists of paths over domains where you can.** Most of these redirect params only
  ever need to point back into the app itself; a relative-path allowlist removes the external
  redirect entirely.

The whole finding is one operator in the wrong place. That is what makes it a good reminder: the
allowlist being *present* is not the same as the allowlist being *enforced on every input*, and
the difference can be a single `||`.
