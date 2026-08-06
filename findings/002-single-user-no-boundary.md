# 002 · "SSRF" in a single-user reminder app — not a vulnerability

**Class alleged:** CWE-918 Server-Side Request Forgery
**Verdict:** **Not a vulnerability.** No trust boundary exists to cross.

---

## The tempting version

A self-hosted reminder / nudge app accepts a user-supplied URL and the server
fetches it. No SSRF guard anywhere — no private-range check, no allowlist. Pattern-
matched against entry 001 in this repo, it looks like the same bug: user URL →
server fetch → internal network reachable.

## Why it isn't one

SSRF is a *privilege* problem, not a *fetch* problem. The exploit is "a lower-
privilege actor makes the server reach something they couldn't reach themselves."
That requires two privilege levels.

This app has **one user**, who is also the administrator and the operator of the
box. There is no second, lower-privilege actor. The only person who can point the
server at `127.0.0.1` is the same person who owns `127.0.0.1`. Making the server
fetch an internal address is indistinguishable from that user opening the address
themselves — no boundary is crossed because there is no boundary.

## The rule this encodes

> Server-fetches-user-URL is a *precondition* for SSRF, not SSRF. The finding only
> becomes real when a lower-privilege user can drive the fetch. In a single-tenant,
> single-user app, that actor doesn't exist.

Filing this would be a false positive — the kind that teaches a maintainer to stop
reading your reports. It's logged here so the process doesn't re-flag it on a later
pass, and as the contrast case that makes entry 001's boundary check meaningful:
001 is real *because* I confirmed a second low-privilege user is isolated, and this
one is not *because* that user can't exist.

## Would it ever become real?

Yes, and this is worth stating so the judgment is legible: if the app grew multi-
user support (shared instance, invited users, any role below admin), the exact same
code would become a genuine SSRF the moment a non-admin could reach the fetch. The
code isn't "safe" in the abstract — it's safe *given the current single-user trust
model*. That distinction is the finding.
