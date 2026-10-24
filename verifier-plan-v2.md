# Verifier Plan v2 (merged)

Merges the implementation plan (pipeline, shrinker, Codex tasks) with the verification and security plan (trust labels, reproduction rules, sandbox controls, release gates). Trimmed to what a 5-hour sprint can actually ship, with the rest parked as post-sprint gates.

---

## 1. Principles

1. **Trust over coverage.** A wrong "verified" label kills the product. A missing one doesn't.
2. **Every result states its evidence level.** No blurring AI opinion and executed proof.
3. **Fail closed.** Unsupported problem, language, or OS means `not_supported`, not a guess.
4. **Local only.** No internet-facing code execution. Ever, until a separate hardened design exists.

---

## 2. Trust labels

Every API response carries a `trust_label`. The UI must render it.

| Label | Meaning | When used |
|---|---|---|
| `ai_guidance` | Model explanation from supplied context | Hints, approach feedback. Never say "tested" or "proven". |
| `constraint_assessment` | Complexity vs. parsed or cataloged limits | Show assumptions, flag unknown constraints. |
| `examples_checked` | Checked only against provided examples | Never implies general correctness. |
| `likely_counterexample` | Disagreement with an **AI-generated** reference that passed the examples | Tier 2 results. Reproduced, but the reference is unvetted. |
| `verified_counterexample` | Disagreement with a **hand-authored, vetted** oracle, reproduced | Tier 1 results only. |
| `could_not_verify` | Oracle, runner, or contract didn't support the request | Explain why, offer a next step. |

**Rule:** `no_mismatch` is never shown as "your code is correct." Wording: "No counterexample found in N tests (seed S). This is not a proof."

---

## 3. Two oracle tiers (the merge)

The two source plans disagreed: one used LLM-generated brute forces (broad but unreliable), the other used hand-authored adapters (trusted but narrow). Use **both, through the same pipeline**. Only the source of the spec and the label differ.

| | Tier 1: adapters | Tier 2: generated |
|---|---|---|
| Spec source | Hand-written, in `adapters/<slug>.py` | One LLM call, JSON schema output |
| Coverage | 5 problems for the sprint | Any in-scope problem |
| Oracle trust | Vetted against examples and known-wrong solutions | Passed official examples only |
| Label on hit | `verified_counterexample` | `likely_counterexample` |
| Demo role | The 60-second demo uses Tier 1 | Shown as "works on other problems too" |

**Optional Tier 2 upgrade (if time):** generate two independent brute forces. If both agree on every input and on the examples, say so in the result ("cross-checked reference"). Still labeled `likely_counterexample`, but with higher confidence in the copy.

### Adapter / Spec shape (identical for both tiers)

```python
Spec(
  method="coinChange",
  oracle_code="class Solution: ...",    # brute force, obviously correct
  gen_code="def gen(rng, size): ...",   # seeded, small inputs
  validate_code="def validate(args): ...",  # encodes constraints
  normalize_code=None,                  # optional
  examples=[{"args": {...}, "expected": ...}],
  trust="trusted" | "unvetted",
)
```

Codex writes the Tier 1 adapters. **You verify each one**: oracle passes official examples, oracle disagrees with a known-wrong solution on a small input, `validate` rejects out-of-constraint inputs.

---

## 4. Pipeline

```
request {slug, statement, code}
  |
  v
[0] Preconditions (fail closed -> could_not_verify / not_supported)
    - Python, class Solution, JSON-serializable args only
    - code matches method contract (ast check)
    - spec exists (adapter, cache, or generate)
  |
  v
[1] Oracle sanity: oracle passes official examples in this build
    (Tier 2: regenerate up to 2x, else could_not_verify)
  |
  v
[2] Generate N=200 inputs from a recorded seed, filter with validate()
  |
  v
[3] Run oracle and candidate in SEPARATE sandboxed processes
  |
  v
[4] Compare (normalize if provided). First mismatch, crash, or timeout.
  |
  v
[5] Shrink (ddmin). Accept a candidate only if validate() holds AND mismatch persists.
  |
  v
[6] REPRODUCE: rerun oracle + candidate on the final shrunk input.
    Report only if the mismatch reproduces. Otherwise fall back to the unshrunk input,
    or could_not_verify if that fails to reproduce too.
  |
  v
[7] Report
```

### Result schema

```json
{
  "status": "found | crash | timeout | no_mismatch | could_not_verify | not_supported",
  "trust_label": "verified_counterexample | likely_counterexample | could_not_verify | ...",
  "input": {},
  "expected": null,
  "got": null,
  "tests_run": 200,
  "seed": 1234,
  "minimized": true,
  "reproduced": true,
  "limitations": ["Reference is AI-generated; passed 3/3 official examples."],
  "message": "..."
}
```

Every `found` result must include input, expected, actual, `tests_run`, `minimized`, and `limitations`. Exceptions, timeouts, and invalid output are distinct statuses, never collapsed into "wrong answer."

---

## 5. Sandbox controls (local MVP)

- One **separate process per execution** (oracle, candidate, generator). The parent backend never executes user source directly.
- LLM-generated code (oracle, gen, validate) is untrusted and runs in the sandbox too.
- `python -I`, new process group, wall-clock timeout, then `os.killpg`.
- `RLIMIT_CPU`, `RLIMIT_AS`, `RLIMIT_FSIZE`, `RLIMIT_NPROC`.
- Fresh empty temp cwd, **empty environment**, no secrets inherited.
- No network where the OS supports it (`unshare -n` on Linux), plus socket and subprocess blocking in the harness as a second layer.
- Caps on stdout, stderr, input size, and test-case count.
- **Fail closed** on unsupported OS, runtime, or contract.
- Don't persist user code by default. Persist only non-sensitive metadata (slug, seed, status).

**Boundary:** supported Python on a controlled local machine only. Public hosted execution requires a container or nsjail/gVisor design, network isolation, quotas, queues, rate limits, audit logs, and independent review. Out of scope for this sprint.

---

## 6. Privacy and API hygiene (minimum for the sprint)

- Send code only on an explicit user action. Show what's in the request.
- Strict Pydantic schemas, request-size limits, enumerated verdict/level/language values, reject unknown fields.
- CORS limited to the extension origin.
- API keys in env only, never returned to the client.
- Never log raw code, prompts, or keys.
- Escape all rendered text. Never inject pasted code or model output as HTML.
- Model output: validate against schema, retry once to repair, reject output that claims verification that wasn't performed.

---

## 7. Build order (5 hours)

| Time | Task | Owner |
|---|---|---|
| 0:00-0:30 | Contract with `trust_label` + result schema, mock endpoint returning canned Coin Change result | Codex |
| 0:30-1:30 | `harness.py`, `sandbox.py`, hostile-code tests | **You** (Codex writes tests) |
| 1:00-2:00 | 5 Tier 1 adapters + golden set | Codex, **you verify** |
| 1:30-2:15 | `spec_gen.py` (Tier 2) + oracle sanity + cache | Codex, you review the prompt |
| 2:15-3:00 | `compare.py`, `orchestrator.py` with reproduce step | Codex |
| 3:00-3:45 | `shrink.py` | **You** |
| 3:45-4:15 | Real `/counterexample`, replace mock | Codex |
| 4:15-5:00 | Integration with A and B, demo rehearsal | all |

Hand-write: `harness.py`, `sandbox.py`, `shrink.py`. Everything else is Codex plus review.

Tier 1 problems: coin-change, house-robber, maximum-subarray, jump-game, longest-increasing-subsequence. (Two Sum is a poor fit: multiple valid answers need a custom checker. Keep it out.)

---

## 8. Tests and evaluation (right-sized)

**Unit**
- Sandbox: infinite loop, memory bomb, `import os; os.system`, huge stdout, fork bomb are all contained.
- Compare: normalize, float tolerance, exception vs wrong output vs timeout are distinct.
- Shrink: result still validates, still fails, and is not larger than the input.
- Spec validation: every adapter's oracle passes its own examples.

**Golden set (5 Tier 1 + 3 Tier 2 problems)** Each has: examples, oracle, correct solution, known-wrong solution, expected minimal counterexample.

**Sprint thresholds**

| Metric | Threshold |
|---|---|
| Counterexample reproducibility | 100% of reported cases reproduce in the harness |
| Correct solutions | `no_mismatch` for every golden correct solution |
| Wrong solutions | `found` for every golden wrong solution |
| Shrink quality | Minimal or within a small constant of known minimum |
| Result completeness | Every `found` includes input, expected, actual, tests_run, minimized, limitations |
| Hostile code | All containment tests pass |
| Failure states | Backend error, timeout, `could_not_verify` each have a usable UI state |

Classify every failure by layer before fixing it: adapter defect, oracle defect, runner defect, generator defect, shrinker defect, prompt defect, schema defect, UI defect. Fix the layer, not just the example.

**End-to-end smoke test**
1. Paste Coin Change plus a greedy candidate.
2. Run counterexample search. Expect `verified_counterexample` with `[1,3,4]`, amount 6, expected 2, got 3.
3. Confirm the UI shows the label, `tests_run`, and limitations.
4. Kill the backend. Confirm the panel stays usable.

---

## 9. Release gates

**Sprint demo gate (this weekend)**
- [ ] Paste-in flow works without page scraping
- [ ] No hint level auto-escalates
- [ ] UI labels guidance vs. verification correctly
- [ ] At least 3 Tier 1 adapters reproduce seeded bugs
- [ ] No raw code or keys in logs
- [ ] Hostile-code tests pass

**Private beta gate (post-sprint)**
- 20-problem eval set meets thresholds, including hint-level compliance and approach classification
- Local-data disclosure and delete-all control
- Adapter degrades cleanly to manual input
- Request limits, CORS, and error handling tested
- Security review confirms no public arbitrary-code execution path

**Public execution gate (not planned)**
- Hardened sandbox independently reviewed, with quotas, rate limits, monitoring, incident response, documented threat model, and matching privacy terms

---

## 10. Cut from the source plan, and why

| Cut | Reason |
|---|---|
| 20-problem eval set | 8 is enough to prove the pipeline in a sprint. 20 is the beta gate. |
| Catalog cycle detection, startup validation | The KB is ~50 hand-checked entries. Validate with one script, not a system. |
| Anonymous schema-failure telemetry | Nothing to send telemetry to yet. |
| Frontend runtime parsers, saved-HTML adapter fixtures | Person A's scope, post-demo. |
| Hint-level policy classifier | Person B's scope. Spot-check by hand for the sprint. |
| Full private-beta and public gates | Kept as written, but explicitly not part of the 5 hours. |

## 11. Added to the source plan

| Added | Reason |
|---|---|
| Two oracle tiers with different labels | Gets both trust and coverage without lying about confidence. |
| `likely_counterexample` label | Tier 2 shouldn't claim "verified." |
| Dual-oracle agreement (optional) | Cheap way to raise confidence in Tier 2. |
| Shrinker design (ddmin, validate-gated) | The source plan said "shrink" but not how. |
| Codex task split and timeline | The source plan had no schedule. |
