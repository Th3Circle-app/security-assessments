# Security assessments of open-source tools

A running log of application-security assessments I run against self-hosted,
open-source web apps: how I pick a target, what I check, what I found, and — just
as often — why a thing that *looked* like a vulnerability wasn't one. Every entry
is a real assessment against real running code, not a writeup of someone else's CVE.

The point of this repo is not a trophy wall. It's the **method**: a repeatable,
AI-assisted process for standing a target up, finding the trust boundary, and
deciding — with evidence — whether a control actually holds. The non-findings are
here on purpose. Deciding that something is *safe* and being able to say why is the
same skill as finding the hole, and it's the half most writeups leave out.

---

## How I work

I use an LLM (Claude) the way a senior engineer uses a sharp junior: it reads a
codebase fast, drafts the exploit and the fix, and never gets to decide whether a
finding is real. That last call — is this actually reachable, does it actually
cross a trust boundary, is the fix actually verified — is mine, and it's made
against a running instance, never against the model's opinion.

The loop for each target:

```
1. Pick a target      small, active, multi-user web app that fetches or
                      renders user-controlled input (URLs, files, templates)
2. Stand it up        run it locally with a second, low-privilege user, so
                      "cross a trust boundary" is a thing I can actually test
3. Map the boundary   where does user input reach a sink? (outbound fetch,
                      file read, SQL, template, deserialization)
4. Reproduce          fire the attack at the running instance; a finding is
                      the response, not a hunch. No repro, no finding.
5. Fix + verify       write the patch, apply it to a throwaway copy, re-run
                      the exact attack, confirm it now fails
6. Disclose privately  report to the maintainer through a private channel
                      before anything is published here
```

Steps 4 and 5 are the whole thing. An LLM will happily tell you code is vulnerable
when it isn't, and happily tell you a patch works when it doesn't. The defense
against both is the same: run the attack, read the real response, then run it again
after the fix. This is the generalized version of the loop in
[redteam-loop](https://github.com/Th3Circle-app/redteam-loop) — pointed at other
people's code instead of my own.

---

## Log

| # | Target | Class | Verdict | Notes |
|---|--------|-------|---------|-------|
| 001 | self-hosted notification service *(name withheld — disclosure in progress)* | SSRF · CWE-918 | **Confirmed, fixed, verified** | Low-priv user drives server-side requests to internal / cloud-metadata addresses. Fix written and verified; target name and working repro withheld until the maintainer has patched. → [findings/001-ssrf-notification-service.md](findings/001-ssrf-notification-service.md) |
| 005 | self-hosted bookmark manager *(name withheld — disclosure in progress)* | SSRF · CWE-918 | **Confirmed, fixed, verified** | **Unauthenticated**, and the fetched internal page's title/description are **returned to the caller** (response disclosure, not blind). Reproduced against a running instance; fix + verification below. → [findings/005-unauth-ssrf-response-disclosure.md](findings/005-unauth-ssrf-response-disclosure.md) |
| 002 | single-user reminder app | SSRF (alleged) | **Not a vulnerability** | Fetches user URLs with no guard — but single-user, so there is no trust boundary to cross. Documented so I don't "find" it again. → [findings/002-single-user-no-boundary.md](findings/002-single-user-no-boundary.md) |
| 003 | self-hosted uptime monitor | Blind SSRF (alleged) | **Not a defensible finding** | Fetching arbitrary URLs is the product's entire function, the response is never returned to the caller (blind), and the action is gated behind an explicit manage-monitors permission. → [findings/003-by-design-and-gated.md](findings/003-by-design-and-gated.md) |
| 004 | self-hosted RSS reader | SSRF checked | **No finding — defended correctly** | Same profile as 001 (multi-user, server fetches user URLs) but every fetch routes through one guard that blocks private/metadata ranges *and pins the connection to the vetted DNS result* — closing the rebinding gap 001 leaves open. The positive control. → [findings/004-defended-correctly.md](findings/004-defended-correctly.md) |

Confirmed findings stay redacted here until they're fixed or the disclosure window
closes, whichever comes first. Responsible disclosure is not a formality — for a
security engineer it's the actual job, and publishing a working exploit against
software people are running today is the opposite of it.

The disclosures and the open-source security contributions that came out of these
assessments — including a `SECURITY.md` policy PR to a project that had no private
reporting channel — are tracked in **[contributions.md](contributions.md)**.

---

## Why the non-findings matter

Three of the five entries above are "no." That ratio is the honest one, and it's the
part I'd want a hiring manager to read.

- **002** looks textbook-exploitable — user-supplied URL, server fetches it, zero
  SSRF guard. But the app has exactly one user, who is also the admin. There is no
  lower-privilege actor to escalate from, so there is no boundary being crossed.
  "Vulnerable-looking code" and "vulnerability" are different claims.
- **003** is a real outbound-fetch-of-arbitrary-URLs — in an uptime monitor, whose
  entire reason to exist is fetching arbitrary URLs you tell it to. The response
  never comes back to you (blind), and you need an explicit permission to add a
  monitor at all. Reporting that as a bug would waste a maintainer's time and mark
  me as someone who pattern-matches instead of thinks.

- **004** has the *exact* profile that made 001 real — multi-user app, server fetches
  user-supplied URLs — but every fetch path goes through one guard that blocks
  internal ranges and pins the connection to the already-validated IP, closing even
  the DNS-rebinding case. No finding, and I can name precisely why the control is
  complete. Recognizing a correct fix is the same skill as finding its absence.

Knowing when *not* to file is what separates a signal from noise. A finder who
reports 002 and 003 is a finder a maintainer learns to ignore — and one who can't
recognize 004's guard as complete is a finder who files bypasses that don't exist.

---

## Related work

- **[redteam-loop](https://github.com/Th3Circle-app/redteam-loop)** — the automated
  detect → classify → patch → verify loop this process generalizes.
- **[tenant-isolation-postgres](https://github.com/Th3Circle-app/tenant-isolation-postgres)**
  — the same "execute the exploit and assert it fails" discipline, applied to my own
  Postgres schema.
- **[th3circle-architecture](https://github.com/Th3Circle-app/th3circle-architecture)**
  — the OWASP-mapped security model of the SaaS these techniques were built against.

Assessments by Harrison C. Songolo · [xkaii.studio/security](https://xkaii.studio/security)
