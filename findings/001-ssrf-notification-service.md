# 001 · Server-Side Request Forgery in a self-hosted notification service

**Class:** CWE-918 Server-Side Request Forgery
**Verdict:** Confirmed · reproduced · fixed · fix verified
**Severity (proposed):** Medium–High (authenticated; High on cloud hosts via metadata)
**Disclosure:** Reported privately to the maintainer. Target name and working
reproduction are withheld from this public repo until a fix ships or the disclosure
window closes.

> **Why this entry is redacted.** The finding is real and verified, but the software
> is running in production for other people right now. Publishing a working
> internal-network exploit before the maintainer has patched would be the exact
> irresponsible-disclosure behavior a security team is hired to prevent. What's
> below is everything that demonstrates the work — the class, the reasoning, the
> fix, and the verification — with the two details that would arm an attacker
> (which project, the exact request) held back.

---

## What it is

A self-hosted, **multi-user** notification service lets each user configure outbound
"push" channels. Two channel types send an HTTP request to a **destination URL taken
directly from user-supplied config**. The only validation is a `new URL()` format
check. There is no restriction on the destination *host* anywhere in the codebase —
no allowlist, no private-range check, nothing.

So an authenticated, **non-admin** user can make the server issue HTTP requests to
addresses the user cannot reach directly:

- **loopback** (`127.0.0.1`) and the host's own internal ports
- **private ranges** (`10/8`, `172.16/12`, `192.168/16`) — other containers,
  internal admin panels, databases with HTTP interfaces
- the **cloud metadata endpoint** (`169.254.169.254`) — on AWS/GCP/Azure this is
  the path to IAM credentials and instance secrets

## Why it's a real finding (the boundary check)

This is the step that separates it from entry 002 in this repo. The app **does**
isolate non-admin users from each other — I verified that a second, low-privilege
user receives a `404` for another user's resources. So there *is* a trust boundary,
and the app clearly intends to enforce it. SSRF lets that same low-privilege user
pivot to the server's internal network, which the boundary is supposed to prevent.
That's a privilege escalation of *reach*, not a feature. The boundary exists, is
intended, and this routes around it.

## Impact

- Read cloud instance metadata → potential credential theft on a cloud host.
- Reach internal-only services and other containers on the same network.
- Port-scan / fingerprint the internal network via response timing and errors.

## The fix

A reusable egress guard that resolves the destination host and refuses private,
loopback, link-local, and reserved ranges, called before every outbound request to
a user-supplied URL. The shape (language-neutral):

```
assertPublicUrl(rawUrl):
  parse rawUrl            → reject if not valid http/https
  host = url.hostname
  if host is a literal IP and isBlockedIp(host)   → reject
  if host is a name:
      addrs = DNS-resolve(host, all)
      if any(isBlockedIp(a) for a in addrs)       → reject
  return ok

isBlockedIp(ip):
  IPv4:  127/8, 10/8, 172.16/12, 192.168/16, 169.254/16, 0/8, >=224/4  → blocked
  IPv6:  ::1, fe80::/10 (link-local), fc00::/7 (unique-local), ::      → blocked
```

Wired in as one line at the top of each channel's `send()`:

```
await assertPublicUrl(this.url)   // refuse internal targets before the request
```

**Known limitation, stated honestly:** validating the host and then connecting
leaves a DNS-rebinding (TOCTOU) window. Closing it fully means pinning the
connection to the already-resolved public IP via a custom lookup/agent. The guard
above closes the direct-IP and DNS-name cases — the practical exposure — and the
report to the maintainer names the rebinding gap explicitly rather than pretending
the one-liner is complete.

## Verification

With the guard wired into the affected channel, the exact reproduction that
previously reached an internal listener now returns an **HTTP 400 rejection** and
the internal listener receives **zero** requests — the outbound call is refused
before it is made. That second run, against the patched code, is what turns
"I wrote a fix" into "the fix holds."

## Mapping

| | |
|---|---|
| OWASP 2021 | A10 Server-Side Request Forgery |
| CWE | 918 |
| Reachability | authenticated, lowest privilege role |
| Trust boundary crossed | user → server internal network / cloud metadata |
