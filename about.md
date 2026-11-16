---
layout: page
title: About
description: Penetration tester and bug bounty researcher, based in Cairo.
permalink: /about/
---

I find security vulnerabilities in web applications and APIs, and report them so they get fixed
before someone else finds them.

I'm based in Cairo, Egypt, and work as an independent security researcher on
[HackerOne](https://hackerone.com/mohamedmoatazsec) and YesWeHack, testing in-scope production
applications for programs across fintech, SaaS, hosting and consumer brands. I hold the
**eWPTX** (Web Application Penetration Tester eXtreme) and **eJPT** certifications from INE, and
I'm currently working toward the **eAIS** (AI Systems Security Specialist) to extend the same
work into AI-powered systems.

## What I test for

Authentication and authorisation flaws (IDOR, broken access control), injection, XSS and CSRF,
business-logic abuse and API security issues, mapped against the OWASP Top 10 and the OWASP API
Security Top 10. Most of what I find lives in the gap between what the UI lets you do and what
the API behind it actually enforces.

## How I report

Every submission ships with a full proof chain, request to response, an impact statement a
non-security engineer can follow, and remediation guidance the owning team can act on. The
longer version is in [How I document a finding]({{ '/writeups/2026/the-proof-chain/' | relative_url }}).

## Tooling

Burp Suite, Nmap, SQLMap, ffuf and Gobuster, Metasploit, Wireshark, Kali Linux, and Python for
everything that should be repeatable. The repeatable parts became
[SecureScope](https://github.com/mohamedmoatazsec/SecureScope), an open-source web security
posture assessment CLI.

## Background

BSc in Management Information Systems, Egyptian Russian University (2025). My graduation
project was a Laravel-based university learning platform with authentication, role-based access
control and document management, which is where I first spent real time on how authorisation is
implemented in a codebase rather than how it is described in a spec.

## Elsewhere

- GitHub: [github.com/mohamedmoatazsec](https://github.com/mohamedmoatazsec)
- LinkedIn: [linkedin.com/in/mohamedmoatazsec](https://www.linkedin.com/in/mohamedmoatazsec)
- HackerOne: [hackerone.com/mohamedmoatazsec](https://hackerone.com/mohamedmoatazsec)

I'm open to penetration testing, application security and security consulting roles, in Cairo
or remote. The quickest way to reach me is a LinkedIn message.
