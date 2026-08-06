# How I use AI in an application-security workflow (honestly)

Every finding in this repo was produced with an LLM (Claude) in the loop. This page
is about *where* the model helps, where it actively hurts, and the discipline that
keeps "AI-assisted" from meaning "AI-hallucinated." If you're evaluating me for a
role that involves AI in a security pipeline, this is the part to read — because the
failure mode of AI security tooling is confident nonsense, and the whole skill is
the harness that catches it.

## The one rule

**The model never decides whether a finding is real. The running instance does.**

An LLM will tell you code is vulnerable when it isn't, and tell you a patch works
when it doesn't — both fluently, both with a plausible paragraph attached. Neither
statement is evidence. Evidence is: fire the request at a live instance and read the
response; then, after the fix, fire the same request and watch it fail. If I can't
produce that second response, there is no finding, no matter how good the model's
explanation sounds.

## Where the model earns its keep

| Task | Why AI helps | The check on it |
|---|---|---|
| **Reading an unfamiliar codebase fast** | Surfaces where user input reaches a sink (fetch, SQL, template, file read) across a repo I've never seen, in minutes | I confirm every claimed sink by reading that exact file |
| **Drafting the exploit request** | Turns "this looks reachable" into a concrete payload to actually fire | The payload only counts once it lands against a running target |
| **Writing the fix** | Produces a minimal, idiomatic patch in the project's own style | Applied to a throwaway copy; the attack is re-run against it |
| **Classifying + writing up** | Maps to CWE/OWASP, drafts the report in a consistent shape | I own severity and the disclosure decision |

## Where the model actively hurts — and the guardrail

- **It invents boundaries that aren't there.** It called a single-user app's URL
  fetch an "SSRF" (entry 002) because it pattern-matched the shape. The guardrail is
  the boundary question — *which lower-privilege actor drives this?* — which is mine,
  not the model's.
- **It over-reports.** Left alone it would file every arbitrary-fetch as CWE-918,
  including an uptime monitor whose job is arbitrary fetch (entry 003). The guardrail
  is the three-question filter (is-it-the-point / is-it-blind / is-it-gated).
- **It claims patches work.** It will describe a fix as "now blocks the attack"
  without anything having run. The guardrail is verify-by-re-running — the same
  discipline as [redteam-loop](https://github.com/Th3Circle-app/redteam-loop)'s
  `verify` stage, which only reports "closed" after re-firing the exact attack.

## The shape, end to end

```
model reads repo ──► candidate sinks ──► [ME: which are reachable, by whom?]
       │                                          │
       ▼                                          ▼
model drafts payload ──► [fire at live instance] ──► landed? ── no ──► not a finding
       │                                                          
       ▼ yes                                                      
model drafts fix ──► [apply to copy · re-fire exact attack] ──► still lands? ── yes ──► fix rejected
       │                                                                                     
       ▼ refused                                                                             
[ME: severity · responsible disclosure to maintainer] ──► entry in this repo
```

The model is inside almost every box. It is inside **none** of the decisions — what's
reachable, what's real, what's fixed, what gets disclosed and when. That division is
the entire methodology, and it's the one I'd bring to a team's AI security tooling:
let the model do the fast, wide, tireless reading and drafting; put a human and a
running target on every gate where being confidently wrong has a cost.
