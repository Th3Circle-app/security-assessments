# 005 · Unauthenticated SSRF with response disclosure in a self-hosted bookmark manager

**Class:** CWE-918 Server-Side Request Forgery
**Verdict:** Confirmed · reproduced · fixed · fix verified
**Severity (proposed):** High (unauthenticated; response content returned; High→Critical on cloud hosts via metadata)
**Disclosure:** Being reported privately to the maintainer. Target name and working
reproduction withheld here until a fix ships or the disclosure window closes.

> Redacted for the same reason as 001: the software is running for real users now, and
> publishing a working unauthenticated internal-network exploit before the maintainer
> has patched is the opposite of the job. Class, reasoning, fix, and verification below;
> the "which project" and the exact request are held back.

---

## What it is

A self-hosted, multi-user bookmark manager has a "fetch the metadata for this URL"
endpoint: you hand it a URL, the server fetches the page, parses the `<head>`, and
returns the title, description, and icon. Two failures stack:

1. **No SSRF guard.** The destination host is never restricted — only a
   `new URL()` + http/https protocol check. The server will fetch loopback, private
   ranges, and the cloud metadata IP (`169.254.169.254`).
2. **No auth on the endpoint.** Every sibling route on the app requires a session;
   this one was registered with no auth gate at all, and the global request hook only
   *populates* session context, it doesn't reject anonymous callers. So the fetch is
   reachable with **no login**.

Stacked, that's an **unauthenticated** attacker driving server-side requests to
internal addresses **and reading the parsed response back**.

## Why it's worse than a blind, authenticated SSRF

Two properties make this more severe than entry 001:

- **Unauthenticated.** 001 needed a low-privilege account. This needs nothing — the
  endpoint answers anonymous requests. The trust boundary crossed is "anyone on the
  internet → the server's internal network."
- **Response disclosure.** The parsed `<title>` and `<meta description>` of the
  fetched internal page are returned in the HTTP response. This isn't blind SSRF you
  infer from timing — you read internal pages' metadata directly, and on a cloud host
  the metadata endpoint's content comes back the same way.

## Reproduction (sanitized)

With no session cookie, POST the metadata endpoint a URL pointing at an internal
listener. Observed against a running instance:

- **HTTP 200**, response body containing the internal page's real `title` and `desc`
  — proof the response is disclosed, not blind.
- The internal listener logs the request — proof the server made the outbound call.
- Pointing the URL at `169.254.169.254/latest/meta-data/` makes the server attempt
  the cloud-metadata endpoint (off-cloud it errors; on a cloud host it returns the
  parsed content).

## The fix — two layers

Neither layer alone is enough, so both go in:

1. **Egress guard** (`assertPublicUrl`) — resolve the host, reject private/loopback/
   link-local/reserved ranges (incl. `169.254.0.0/16`), called before every
   outbound fetch of a user-supplied URL. Same shape as 001's guard.
2. **Require authentication** on the fetch endpoints — add the same session
   `preHandler` the app's other routes already use, so the surface isn't reachable
   anonymously even if the guard ever regresses.

**Known limitation, stated honestly:** the guard validates the host then connects,
leaving a DNS-rebinding (TOCTOU) window; closing it fully means pinning the connect
to the resolved public IP. The auth layer independently removes anonymous reach, so
the two together are defense-in-depth rather than a single point of failure.

## Verification

With the guard wired in, the same unauthenticated reproduction returns **HTTP 500
"Refused: URL targets a private/reserved address"**, the internal listener receives
**zero** requests, and a legitimate public URL still returns HTTP 200 with a parsed
title — the attack is closed and normal use is intact. That second run against the
patched code is the finding; the first was the hypothesis.

## Mapping

| | |
|---|---|
| OWASP 2021 | A10 Server-Side Request Forgery |
| CWE | 918 |
| Reachability | **unauthenticated** |
| Trust boundary crossed | anonymous internet caller → server internal network / cloud metadata |
| Disclosure | response content (title/desc/icon) returned to caller — not blind |
