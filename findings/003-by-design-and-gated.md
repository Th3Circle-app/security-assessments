# 003 · "Blind SSRF" in a self-hosted uptime monitor — not a defensible finding

**Class alleged:** CWE-918 Server-Side Request Forgery (blind)
**Verdict:** **Not a defensible finding.** By-design behavior, blind, and
permission-gated. Three independent reasons, any one of which is disqualifying.

---

## The tempting version

A self-hosted uptime monitor lets a user add a "monitor" pointed at any URL, and the
server periodically fetches it. You can point it at `169.254.169.254` or an internal
host, and the server will dutifully request it. SSRF, surely.

## Why it isn't a finding — three reasons, each sufficient

**1. It's the product.** An uptime monitor's entire function is "fetch the arbitrary
URL I give you and tell me if it's up." Fetching arbitrary user-specified URLs isn't
a bypass of the intended behavior — it *is* the intended behavior. Reporting it is
like reporting that a web browser fetches URLs.

**2. It's blind.** The response body is never returned to the caller. You get up/down
and timing, not content. Cloud-metadata SSRF is dangerous because you *read the
credentials back*. A monitor that only tells you whether a host answered leaks far
less — at most coarse internal-reachability by timing, which for a tool whose job is
"tell me if this host answers" is, again, the feature.

**3. It's gated.** Adding a monitor requires an explicit manage-monitors permission.
It isn't reachable by the lowest-privilege user; it's an action granted to operators
who are, by design, trusted to say "watch this URL."

## The honest engineering judgment

Could a maintainer add metadata-endpoint and private-range blocking as a hardening
option? Sure — it's a reasonable defense-in-depth toggle, and a good writeup would
*offer* it that way. But "a reasonable optional hardening" is a feature request, not
a vulnerability report. Framing by-design, blind, permission-gated behavior as a
CWE-918 finding inflates the severity of a design tradeoff the maintainer already
made on purpose.

## The rule this encodes

> Before filing SSRF, ask three questions: is the fetch *the point of the tool*, does
> the caller *get the response back*, and is the action *available to a low-privilege
> actor*? A "yes / no / no" — as here — is not a report you send. It's a hardening
> suggestion at most, and often not even that.

Logged so the process doesn't mistake a monitor's core loop for a bug on a later pass.
