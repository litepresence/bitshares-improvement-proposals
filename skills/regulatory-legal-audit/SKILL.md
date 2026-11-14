# Skill: regulatory-legal-audit

Map what the law does to the design before a regulator explains it live. This is risk
inventory, not legal advice — mark it as such in any output.

## Checklist (every item, in order)

1. **Asset classification per token.** For every new token (stable, reserve coin, reward
   claims, LP shares, bond receipts): who profits from whose efforts? A junior tranche
   marketed on yield from others' activity is securities-shaped in most jurisdictions —
   say which tokens carry that shape and what mitigates it (utility use, no marketing of
   yield, decentralization of control).
2. **Privacy features vs sanctions/AML regimes.** Ring signatures + default-blind wallets
   against the Tornado Cash precedent (sanctioned mixer, developer prosecution) and
   exchange-delisting practice. Distinguish transfer privacy (higher scrutiny) from
   market/protocol transparency (retained). Name the listing deltas: which integrations get
   harder, which are unaffected, what the opt-out path proves.
3. **Custody and money-transmission surface.** Bonded vault operators holding external BTC:
   which roles look like custodians, money transmitters, or fiduciaries, and in what
   jurisdiction the operator sits matters more than where the chain sits. Threshold custody
   spreads but does not erase this. Oracles publishing prices: data-vendor vs benchmark-
   administrator framing (EU BMR direction).
4. **Governance liability.** Named roles (halt-holder, auditor, arbitrator, worker recipient):
   liability concentrates on names. Prefer role-definition by threshold vote over named
   individuals wherever possible, and say where a name is unavoidable.
5. **Sanctions-proofing the safety tools.** Freeze/halt/seize functions are dual-use: the
   same op that stops an exploit can censor a user. Define who may invoke against whom, with
   what appeal, and what prevents mission creep. A halt with no appeal bound is a finding.
6. **Disclosure posture.** What the docs promise vs what the code enforces, in plain-language
   risk statements a holder can read pre-deposit. Missing or misleading risk disclosure is a
   finding against the BSIP, not a footnote.
7. **Namespace and symbol claims.** Governance-controlled namespaces and asset symbols
   (reference-asset creation, registered symbols like the USD door) are claims about
   authority, not just identifiers: who may create them, what they imply to a holder, and
   what listing venues read from them. Misleading symbol scope is a finding.
8. **Governance-controlled parameters.** Every fee, cap, or schedule a vote can change is a
   regulatory surface: name who can change it, the range, and what a holder's disclosure
   says about it being changeable. An undisclosed committee-controlled parameter that moves
   user economics is a finding.

## Output

Per-token classification notes, per-feature regulatory deltas, custody map, role-liability
list, halt-appeal rules, and required disclosure sentences. Flag, don't resolve — resolution
needs counsel, but the flags must exist before community review.

## Machine-Readable Output (required)

```json
{"skill":"regulatory-legal-audit","auditor_model":"<exact model string — mandatory, AGENTS.md §11>","branch":"<branch>","tip":"<commit>",
 "findings":[{"id":"RL-01","severity":"Critical|Important|Minor|Note","check":"classification|custody|role-liability|halt-appeal|disclosure","verdict":"PASS|FAIL|QUALIFIED|NOT_COVERED","evidence":"file:line","fix":"smallest concrete fix"}],
 "disclosures":["<required plain-language disclosure sentence per token/feature>"],
 "gate":"GO|GO_WITH_CONDITIONS|NO_GO","conditions":["..."],"not_covered":["features with no classification"]}
```

Severity (precedence: the round brief's severity rule overrides this one;
when they conflict, file at the program severity and state both readings):
`Critical` = a missing required disclosure or an unappealable halt touching
user funds (**at the program discipline: funds/consensus/invariant loss
reachable by an unprivileged or lightly-privileged actor; a legally-shaped
risk needing counsel is Important, not Critical, unless funds are already
reachable**). `Important` = a custody or liability ambiguity needing counsel
before community review. `Minor`/`Note` = wording. Flag, do not resolve —
resolution needs counsel; the flag must exist.
