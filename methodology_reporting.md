# Pentest Report Writing Methodology

Source material: `/home/max/thm2/writingpentestreports/NOTES.md`

## Report Structure

1. **Executive Summary** — business stakeholders, no jargon, risk language
2. **Findings & Recommendations** — security team, attack chains, systemic issues, links between vulns
3. **Vulnerability Write-Ups** — one per finding (see template below)
4. **Appendices** — Scope + Artefacts

---

## Executive Summary Must Cover

- **Overview** — what was tested, type of system, goals, scope, coverage achieved
- **Results** — what was found, overall security posture
- **Impact** — real-world consequences if issues remain unaddressed
- **Remediation Direction** — high-level next steps, rough effort (quick fixes vs. major investment)

---

## Per-Vulnerability Write-Up Template

```
Title:
  Short, descriptive (e.g. "Unauthenticated SQL Injection in Login Form")

Risk Rating:
  Rate in isolation — as if no other vulnerabilities exist.
  Use CVSS or client's own risk matrix.

Summary:
  2-3 sentences. What is the vulnerability and what is its potential impact, in plain language.

Background:
  Context for a reader who isn't a security expert. Explain the vulnerability class,
  why it exists, and why it matters. Developers reading this need enough context
  to understand the root cause, not just the symptom.

Technical Details & Evidence:
  - Exact endpoint / parameter / feature where the issue was found
  - Steps to reproduce
  - Requests, responses, payloads
  - Screenshots or code snippets
  - Any assumptions required (e.g. requires valid account, only affects admin workflow)

Impact:
  Contextualise to THIS client's environment. Don't copy-paste generic impact.
  Ask: what could a real attacker realistically do here, given this specific system?
  E.g. if the app uses tokens not cookies, don't say "steal session cookie."

Remediation Advice:
  - First recommendation MUST address the root cause (not just mitigate)
  - Tailor to their tech stack if known (show a code example in their language)
  - Additional defence-in-depth measures can follow, but flag they can't stand alone
  - E.g. for SQLi: parameterisation is the fix; input validation is defence-in-depth only

References: (optional)
  Links to vendor docs, CVEs, or guidance that support the fix
```

---

## Appendices

### Assessment Scope
- What was originally scoped vs. what was actually tested
- Coverage percentage and reasons for any gaps
- Flags whether a follow-up assessment is needed for uncovered areas

### Assessment Artefacts
- List every file uploaded, account created, or change made during testing
- Include location, what it is, and how to remove it
- Critical: webshells, test accounts, modified configs — if forgotten, they become real incidents

---

## Writing Standards

- **Past tense** — "The vulnerability was identified during authentication testing"
- **No first person** — no I, we, our, us. Write as a neutral observer
- **No informal language** — no slang, no "we pwned", no contractions
- **Be objective** — facts only, no exaggeration, no assumed intent
- **Be consistent** — same terminology throughout, don't switch between synonyms
- **Mask sensitive data** — blur real passwords/PII in screenshots

---

## QA Checklist (before submitting)

- [ ] Every finding has evidence (request/response, screenshot, or output)
- [ ] Risk ratings are consistent and rated in isolation
- [ ] Remediation addresses root cause, not just mitigation
- [ ] Impact is contextualised to this client — not generic
- [ ] No first-person language anywhere
- [ ] No sensitive data (real passwords, PII) unredacted
- [ ] Scope appendix reflects actual coverage vs. planned
- [ ] Artefacts appendix lists everything left on the target
- [ ] No orphaned references or broken links
- [ ] Read aloud — if it sounds unclear, rewrite it

---

## Golden Rule

**If it's not in the report, it didn't happen.**

Every finding needs a write-up and evidence. Undocumented findings don't count.
