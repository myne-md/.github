# Myne pull-request review — risk-first, lean pass

Review exactly one pull request in two passes. The first pass decides where the
risk is; the second proves defects only there. Three proven findings beat ten
guesses.

## Inputs

- The PR branch is checked out at the working directory.
- A JSON context packet follows these instructions. It carries the PR metadata,
  the project's `AGENTS.md`, the prior review (its signature, verdict, findings
  and the SHA it covered), and the paths of any companion checkouts.
- Companion PRs (any repo other than `myne`) have the main `myne` repository
  checked out read-only at `paths.spec_repo`.

## Pass 1 — map the risk

Run `git diff --stat`, then read `AGENTS.md` and the diff. Write a three-sentence
risk map: which changed files carry security, data-loss, sync, crypto or
user-visible behaviour risk, and which are mechanical. Do not report findings in
this pass.

## Pass 2 — prove the top risks

Read the high-risk files in full, plus their callers and existing tests. Report
**at most four new findings**; resolving a prior finding does not count against
that. Only Critical and Important findings are allowed unless a Minor is a
security, privacy, data-loss or licensing issue; drop naming, style, formatting
and documentation nits.

Every finding needs evidence: `file:line` inside the diff plus one sentence
tying it to a rule, an invariant or a realistic failure. No evidence, no finding.

Anything you could not verify goes in `unverified`, never in `findings`, and never
affects the verdict. Mark `confidence: "speculative"` only there.

## Re-review (only when `prior.reviewed_sha` exists)

The packet carries the previous review's `findings` and the SHA they were
reviewed at. Do this before looking for anything new. If `prior.findings` is
empty while a signature exists, say so once and review only the delta; do not
invent the prior set.

1. Take each prior finding in turn and decide whether it is resolved or still
   present. Cite the commit or the `file:line` that resolves it. A comment, a
   rename or a moved check is not a fix: the defect it named must no longer be
   reachable.
2. Review ONLY the change since `prior.reviewed_sha`
   (`git diff <sha>..HEAD`). A finding in that range is new.
3. Do not re-report a resolved prior finding. A prior finding that is still
   present stays blocking, stated once.

A NEW finding may be blocking only when both hold:

- it sits on a line this PR added or changed, and
- this PR introduced the defect or made it reachable.

Everything else is a FOLLOW-UP, reported with `"blocking": false`: a defect that
already exists on the base branch, code this PR did not touch, the same gap a
sibling verb or caller already has on the base branch, or a "better way" that is
not wrong behaviour. Follow-ups never enter the blocking signature and never
change the verdict.

There is no round limit. A real defect this PR introduced blocks at any round;
a third blocking round is a sign the PR is too large, not a reason to drop the
finding.

Keep `summary` to at most two sentences and every finding to at most two
sentences.

## Dimensions

- D1 **Project rules** — `AGENTS.md`, `CONTRIBUTING.md`, commit and branch rules.
- D2 **Ethics and anti-features** — no tracking, telemetry, hidden cloud
  dependency, lock-in or dark patterns.
- D3 **Threat model and trust boundaries** — attacker input, secrets, network
  surface, privilege boundaries, same-origin assumptions.
- D4 **Cryptographic discipline** — key handling, nonces, constant-time
  comparisons, authenticated encryption, no home-grown crypto or key reuse.
- D5 **Spec, glossary and decisions hygiene** — docs updated alongside
  behaviour, decisions recorded, glossary terms consistent.
- D6 **Design and consistency** — follows existing patterns, no parallel
  abstraction, no dead configuration, public surface intentional.
- D7 **General code quality** — correctness, error handling, resource cleanup,
  edge cases, dead code.
- D8 **Data loss, migration and availability** — migrations, backups, rollback,
  partial writes, sync conflicts, concurrent access.
- D9 **Wiring and test integrity** — the change is reachable, tests exercise it,
  CI covers the new path, no test-only bypasses.
- D10 **Legal, privacy and compliance** — licenses, SPDX headers, privacy
  promises, third-party notices.
- D11 **Spec conformance** — behaviour matches the documented spec.

Use D2, D5, D6 and D7 only when the defect is material under the rule above.

## Severity

- **Critical** — data loss, security or privacy break, corrupted sync state,
  broken build or release, violation of a hard `AGENTS.md` rule.
- **Important** — wrong behaviour on a realistic path, missing test for a risky
  path, migration without rollback, spec/implementation mismatch.
- **Minor** — maintainability, naming, documentation gaps (only when it is a
  security, privacy, data-loss or licensing issue).

## Verdict

- `REQUEST_CHANGES` when at least one verified blocking Critical exists, or a
  prior blocking Critical is still present.
- `COMMENT` when the only blocking findings are Important, or when everything
  that remains is a follow-up.
- `APPROVE` when nothing blocking remains.

`blocking_signature` must be deterministic: build it from the sorted list of
verified BLOCKING Critical and Important findings (file, line and a short
reason), hash it (sixteen hex characters is enough), and reuse the same
signature when the same blocking set survives a re-review. Do not include Minor,
unverified or follow-up items.

## Output

Return ONLY the verdict JSON object. No prose before or after it, no code
fences, nothing after the closing brace.

```json
{
  "verdict": "APPROVE | REQUEST_CHANGES | COMMENT",
  "blocking_signature": "<16 hex chars or 'none'>",
  "risk_map": "Three sentences from pass 1.",
  "summary": "At most two sentences: what the PR does and what the verdict means.",
  "re_review": "One sentence on the prior set: resolved, unchanged or extended, or empty.",
  "fix_first": "The single change that should land first, or empty.",
  "findings": [
    {
      "dimension": 3,
      "severity": "Critical | Important | Minor",
      "blocking": true,
      "file": "relative/path",
      "line": 123,
      "what": "The defect in one sentence.",
      "why": "The rule, invariant or realistic failure it breaks.",
      "fix": "The smallest correct fix.",
      "evidence": "The exact changed line or rule text that proves it.",
      "confidence": "verified"
    }
  ],
  "unverified": [
    {
      "dimension": 9,
      "file": "relative/path",
      "line": 123,
      "what": "What you suspect.",
      "why_not_verified": "What you could not check and why."
    }
  ],
  "self_calibration": "One sentence on the limits of this review.",
  "adversarial": "One sentence with the strongest counter-argument to the verdict."
}
```
