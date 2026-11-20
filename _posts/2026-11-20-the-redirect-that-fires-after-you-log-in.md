---
title: "The redirect that fires after you log in"
description: A procurement login page read a 'pagerefer' value from the URL and, on a successful login, assigned it straight to location.href with no validation — turning a link to the real login page into DOM XSS or an open redirect that triggers the moment the victim signs in.
date: 2026-11-20 22:45:00 +0200
category: Write-up
tags: [dom-xss, open-redirect, client-side, authentication, logic-flaw]
report:
  program: Public program (name withheld pending disclosure)
  platform: HackerOne
  vulnerability: DOM-based XSS and open redirect via unvalidated post-login redirect parameter (CWE-79 / CWE-601)
  severity: Medium
  cvss: "6.1 (AV:N/AC:L/PR:N/UI:R/S:C/C:L/I:L/A:N)"
  status: Reported — under review
  reported: 2026-11-14
---

The login page of a procurement platform read a `pagerefer` value out of the query
string and, inside the success handler of the login request, assigned it directly to
`location.href` — no check on scheme, no check on host. So a link to the *genuine*
login page, on the real hostname, with a valid certificate, would run attacker-chosen
JavaScript in that origin — or bounce the user to any external site — the instant the
victim finished signing in. I'm withholding the brand and the real hostname while the
report is under review.

<!--more-->

## Summary

On a successful login, the page picked where to send the user next. It preferred one
internal variable, then fell back to a `pagerefer` query parameter, then to a default
path. The `pagerefer` branch assigned the raw parameter value to `location.href`
without validating it. A value of `javascript:…` executes in the page's origin after
login (DOM XSS); a value of `//evil.example` sends the freshly authenticated user off
site (open redirect). The link the victim clicks is the real login page — nothing
about the URL looks off until they've already signed in.

## The application

A standard username/password login page for an invitation-only procurement portal.
After the login request returns success, an inline script decides the redirect. The
query parser it uses is permissive: it pulls the value after `pagerefer=` and only
stops at `&`, `=` or `#`, so `:` and `/` — everything you need for `javascript:` or a
protocol-relative URL — pass straight through.

## Finding it

Reading the page source before trusting the flow is the whole technique here. The
redirect logic was in plain sight in an inline script, and the shape that matters is
this (reconstructed, not the target's verbatim code):

```js
function getSearchParam(key) {
  var s = window.location.search || '';
  var m = s.match(new RegExp('\\b' + key + '=([^&=#]+)', 'mi'));
  return m ? m[1] : '';
}

// inside the login success handler:
if (queryurl) {
  location.href = queryurl;
} else if (getSearchParam('pagerefer') !== '') {
  location.href = getSearchParam('pagerefer');   // unvalidated sink
} else {
  location.href = '/home/index.html';
}
```

A user-controlled value flowing into `location.href` with no allowlist is both a
`javascript:`-URI XSS sink and an open-redirect sink. The only question left was
whether the success branch could be reached.

## Proof chain

I do not have an account on this platform (access is by invitation), and I did not
attempt to bypass or attack the login. To prove the page's *own code* reaches the
sink, I simulated a successful-login response locally in my browser — no request was
sent to the server.

1. **Load the real login page with the payload in the URL.** A harmless payload that
   only writes the page's own origin name:

   ```
   https://portal.brand.example/usercenter/login.html?pagerefer=javascript:document.write('XSS-PoC-on-'+document.domain)
   ```

   The genuine login page loads; the parameter is not filtered.

2. **Simulate the success response locally.** In the page's DevTools console, stub the
   login request so it returns the same shape a real success returns, then call the
   page's own login function:

   ```js
   $.ajax = o => o.success({ state: 1, data: 0 });   // local stub, no network
   $("input[name='uName']").val('demo'); $("input[name='pWord']").val('demo');
   Login();
   ```

3. **The sink fires in the real origin.** The success branch assigns the `pagerefer`
   value to `location.href`, the payload executes, and the document is replaced with
   `XSS-PoC-on-portal.brand.example` — the address bar still on the real hostname. A
   teammate with any valid account can confirm the full chain by opening the step-1
   link and signing in normally.

4. **Open-redirect variant.** The same code path, with
   `?pagerefer=//external.example`, sends the user to an external site after login.

## Impact

The attacker needs only to get a victim to open a link to the genuine login page and
sign in as they normally would.

- **JavaScript execution in an authenticated session.** The payload runs immediately
  after the victim's own successful login, inside the portal origin — so it can read
  what that user sees and act as that user on the procurement platform.
- **Credential phishing on a real hostname.** Because the link is the real login page
  on a real hostname with a valid certificate, the injected script can show a
  convincing "session expired, sign in again" prompt right after login and capture the
  re-entered credentials.
- **Open redirect.** The same parameter bounces a freshly logged-in user to any
  external site.

**Limits of what I demonstrated — stated plainly.** The successful-login step was
simulated locally because I have no account, so I did not confirm which specific
actions or data are reachable after login, or whether the session cookie is readable
by script. What is proven is that the page's own success handler sends an unvalidated
URL parameter into `location.href`.

## Remediation

- **Validate the redirect target before use.** Accept only a same-origin relative path
  — must begin with a single `/`, reject `//`, `\`, and any value containing a scheme
  such as `javascript:` — or map `pagerefer` to a server-side allowlist of
  destinations. Apply the identical check to the other redirect variable in the same
  branch.
- **Never assign a user-controlled value to `location.href`** without that check. It is
  simultaneously an XSS and an open-redirect sink.

## Timeline

| Date       | Event             |
|------------|-------------------|
| 2026-11-14 | Tested & reported |
| —          | Under review      |

## Lessons

"Open redirect" undersells it. The same unvalidated assignment that bounces a user off
site will also run `javascript:` in the origin — and because it only fires *after* a
real login on the real page, every trust signal a careful user relies on (correct
hostname, valid padlock, their own successful sign-in) is present before the payload
runs. A post-login "return URL" is still attacker input. Validate where you send
people, every time.
