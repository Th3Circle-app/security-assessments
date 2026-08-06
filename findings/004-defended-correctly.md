# 004 · SSRF in a self-hosted RSS reader — correctly defended, no finding

**Class checked:** CWE-918 Server-Side Request Forgery
**Verdict:** **No finding.** The target implements SSRF defense correctly — including
the DNS-rebinding case that most implementations miss.

---

## Why this was a promising target

A self-hosted, **multi-user** RSS reader (Google-Reader-compatible auth, per-user
subscriptions). Adding a feed hands the server a user-controlled URL and the server
fetches it on a schedule. That's the textbook SSRF profile — the same one that made
entry 001 real: multi-user boundary + server-side fetch of a user-supplied URL. I
expected a finding.

## What I found instead

Every outbound fetch in the codebase routes through a single choke point
(`utils/fetchURL.js` → `fetchWithOutboundRequestSafeguard`). There is no second
`axios`/`got`/`fetch` sink that skips it — I checked, because a guard is only as good
as its coverage, and the common real bug is *one fetch path that forgot to call it*.
Here, there is exactly one path and it is guarded.

The guard itself is not the naive version:

- A `node:net` **BlockList** covering loopback, private ranges, link-local, and the
  `169.254.0.0/16` cloud-metadata range.
- Crucially, it **checks the resolved DNS result and pins the connection to that
  vetted IP** via a custom `undici` connector — so a hostname that passes the check
  and then re-resolves to an internal address (DNS rebinding / TOCTOU) cannot slip
  through. The connect happens against the address that was actually validated.

## Why this entry exists

This is the control that entry 001's proposed fix explicitly flags as its *known
limitation*. 001 closes the direct-IP and DNS-name cases with a one-liner and tells
the maintainer the rebinding window is still open; this project closes the rebinding
window too, the correct way. Putting them side by side is the point:

- **001** — the boundary is real and the guard is absent → finding.
- **004** — the boundary is identical and the guard is present *and complete* → no
  finding, and I can say precisely why it's complete.

Recognizing a correct, well-covered control — and being able to name the specific
advanced case (rebinding) it defends — is the same competence as finding the hole.
A security engineer who can only spot absence, and can't recognize a good fix when
it's there, floods the team with noise. Logged as the positive control for this repo.

## The rule this encodes

> A guard is worth exactly its coverage. Before trusting one, find *every* outbound
> sink and confirm each goes through it; then check whether the guard validates the
> **connected** address, not just the **requested** hostname. This target passes both
> tests. That's what "done right" looks like.
