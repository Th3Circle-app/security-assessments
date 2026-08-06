# Contributions & disclosures

A running record of what came out of these assessments: the vulnerabilities reported
privately to maintainers, and the open-source security contributions made along the
way. This is the paper trail behind the findings in the [log](README.md) — real PRs,
real disclosures, dated.

## Open-source contributions

| Date | Project | Contribution | Status |
|------|---------|--------------|--------|
| 2026-08-05 | [AlexSciFier/neonlink](https://github.com/AlexSciFier/neonlink) (~400★) | Added a `SECURITY.md` disclosure policy so vulnerabilities can be reported through a private channel — [PR #126](https://github.com/AlexSciFier/neonlink/pull/126) | Open |
| 2026-08-05 | [magiccode1412/magicpush](https://github.com/magiccode1412/magicpush) (~130★) | Added a `SECURITY.md` disclosure policy — [PR #33](https://github.com/magiccode1412/magicpush/pull/33) | Open |

Every policy PR here was opened on a project I had actually assessed — not sprayed
across unrelated repos. A disclosure policy is only worth adding where there's a real
reason to have a private channel, and in both cases above there was: a confirmed
finding to report.

## Private vulnerability disclosures

| Finding | Project | Class | Reported | Status |
|---------|---------|-------|----------|--------|
| [001](findings/001-ssrf-notification-service.md) | notification service *(redacted)* | SSRF · CWE-918 | privately, via maintainer channel | Awaiting fix |
| [005](findings/005-unauth-ssrf-response-disclosure.md) | bookmark manager *(redacted)* | Unauthenticated SSRF · CWE-918 | disclosure in progress (policy PR opened as the channel) | In progress |

Confirmed findings stay redacted in this repo until a fix ships or the disclosure
window closes. Each entry is un-redacted with the full named write-up once it's safe
to publish.

---

*Updated 2026-08-05.*
