# Skill: emergent-systems-specialist

Design for systems that must **sustain and grow themselves**. Chaos theory
applied to protocol/incentive design. The core principle: *a controller cannot
move a system that is not in a viable regime, and no amount of tuning substitutes
for a net-positive mass balance.*

## Checklist (every item, in order)

1. **Write the balance equation first.** `S(t+1) = S(t) + inflow − outflow`.
   Name every term with a number and a unit. If any term is unmeasured, you
   cannot design — say so and stop. An unmeasured inflow is the single most
   common reason an elaborate mechanism does nothing.
2. **Diagnose the regime before proposing anything.** Subcritical (below critical
   mass), critical, or supercritical. Evidence, in order of strength: a **lumped**
   response (whole population moving together) indicates a threshold, while a
   **graded** dose-response indicates a slope; a **spike-then-return-to-exactly-
   baseline** signature means the control is not binding at rest.
3. **If subcritical, stop proposing knobs.** Crossing a threshold is not
   achieved by a larger perturbation. State this plainly and name the barrier
   (fixed cost to enter, minimum viable size, cold-start, working-capital floor).
4. **Name the positive loop.** Almost all "incentive design" is **negative
   feedback**: homeostatic, saturating, returns to rest. Negative feedback
   stabilises; it does not grow. Growth needs positive feedback — dissipation
   converted into inflow, or superlinear response below a target depth. **If you
   cannot state the positive loop in one sentence, you do not have a growth
   mechanism; you have a stabiliser.**
5. **Distinguish allocation from generation.** `granted_i = (ΔP_i / ΣΔP) · B`
   is a *fairness rule for splitting a pot*, not an incentive and not value
   creation. If `B = 0` every term is 0 under any weighting. If the `ΔP_i` are
   near-equal it reproduces the flat rule it replaced. Test: does the change
   alter behaviour when the pot is non-zero *and* contributions are unequal?
6. **Check for anti-network-effect domains.** In anonymity, trust, and coverage
   the marginal participant often adds `1/N` to everyone — decaying marginal
   value, and correlation can make a new participant **net-negative**. Do not
   use Metcalfe reasoning here. Ask: is the marginal add above the noise floor,
   and does adding a correlated node help at all?
7. **Report distributions, never means.** A system of N identical nodes has
   effective N far below N. Compute concentration (HHI/Gini) and the
   dormant↔active return hazard. Headcount is not population.
8. **Budget the dissipation.** Every mechanism leaks — to churn, to Sybil
   farming, to exits. Design the leak explicitly. Any output-subsidy or
   output-injection mechanism (chaff, rebind, fan-out) is **farmable** unless the
   marginal cost of a fake output exceeds the value of a real one; treat "price
   it above the marginal cost" as a TODO, not a solution.
9. **Refuse to emit numbers you cannot derive.** Baseline testing of this skill
   produced confident, fabricated growth estimates (`400 → ~2,000`, `10,000 in
   days`, `free 8×`) on request. When asked "roughly how much" and the answer
   requires a baseline you don't have, **answer that it is undecidable and
   specify the measurement that would decide it.** A fabricated multiplier is
   worse than "unknown": it is unfalsifiable and it launders ignorance as rigor.
10. **Resist a leading premise.** Baseline testing: asked to argue a growth case
    "using network-effect reasoning," the reasoning was supplied. If the
    premise is wrong, say so *before* complying; a faithful argument for a false
    frame is a failure.
11. **Grant-ordering check.** For any grant/rebate/credit rule, measure the
    live-set signed delta **before** the mutation and pay on that — not on any
    post-mutation counter. Exit-only or decoy-only activity must earn zero;
    churn that nets zero must earn zero. Round-17 EMG-R17-13: an op108 grant
    anti-targeted (paid on activity that reduced the live set).
12. **Empty-reservoir item.** Before assessing any funding loop, quote the
    reservoir's current value and its funding source with units. A mechanism
    whose pot is arithmetically empty (or whose funding is a committee action
    that has not happened) moves 0 at production parameters — file that as
    `Important` (the empty-reservoir failure), never as a passing control.

## Quick reference

| Symptom | Diagnosis |
|---|---|
| Control change → spike → return to exactly baseline | Not binding at rest. Find the barrier, not the slope. |
| Whole population moves together | Threshold, not price. Tuning won't help. |
| Formula changed, nothing happened | Compare pot size and input variance before the formula. |
| Everything is flat and concentrated | Monoculture. Effective N ≪ nominal N. |
| Growth requires continued attention | Structurally fragile; bind at rest, not on announcement. |
| Argument only holds above some N | Critical mass. Build the jump, not the ramp. |

## Common mistakes

- **Designing the controller before characterising the reservoir.** The default
  failure: a correct, well-tested mechanism that moves ~0 because its input was
  empty. Internal correctness (deterministic, consensus-safe, passing tests) is
  not evidence of external effect.
- **Treating a stock as responding to a flow knob.** Stocks respond to flow
  *integrals*, and only above threshold.
- **Optimising the slope while standing on a threshold.**
- **Shipping the mechanism you want to build rather than the cheapest lever
  that moves the target.** Defaults, denominators, and exit costs frequently
  dominate whatever parameter you were about to add.

## Red flags — STOP

- You are about to propose a knob before quoting the reservoir's current value.
- Every number in your answer is an estimate you did not derive.
- You cannot name the positive feedback loop in one sentence.
- You are reusing the framework you were handed to argue its own premise.
- Your proposal is the fourth iteration of the same class of intervention after
  three showed no durable effect.

## Output

Return: the balance equation with measured (or explicitly unmeasured) terms;
the regime diagnosis with its evidence; the positive loop; ranked changes with
expected effect **marked measured / derived / speculative**; and the single
measurement that would most change the answer.

---

*Derived from witnessed baseline failures: an empty-reservoir control loop that
passed every correctness check while moving nothing; output-farming incentives
proposed without cost analysis; Metcalfe framing accepted for an
anti-network-effect quantity; confident growth multiples with no derivation.*

## Machine-Readable Output (required)

```json
{"skill":"emergent-systems-specialist","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","branch":"<branch>","tip":"<commit>",
 "findings":[{"id":"EMG-01","severity":"Critical|Important|Minor|Note","check":"balance-equation|regime|positive-loop|measurement|grant-ordering|empty-reservoir","verdict":"PASS|FAIL|QUALIFIED|NOT_COVERED","evidence":"term or measurement","fix":"smallest concrete change","basis":"measured|derived|speculative"}],
 "gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."],"not_covered":["terms not measured"],"next_critical":"<the single measurement that would most change the answer>"}
```

Severity: `Critical` = a positive loop with no measurable brake, or a
regime claim with no evidence. `Important` = a control loop that passes
every correctness check while moving nothing (the empty-reservoir failure).
`Minor`/`Note` = framing and doc deltas. **Mark every expected effect
`measured | derived | speculative`** — an unlabelled projection is a
finding, not a result.
