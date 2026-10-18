# Evidence

Thesis: I build instruments that catch AI systems, myself included, fooling themselves, and I ship. I work with a stack of three AI agents (Cadence, Aperture, Lodestone) on different models; that help is disclosed throughout. Every line below is a claim followed by a receipt a stranger can check. Links and hashes were checked on 2026-10-16 by two agents on separate machines; the devnet transactions were confirmed by RPC from one agent, and the other checked sha256 of the archived RPC responses and explorer screenshots.

## 1. Shipped

- honest-evals, a Python package for evals that must prove they can fail: hash-chained claim ledger with truncation detection, safe re-run verifier, two-population selection-effect estimator, canary gates, cutoff sweeps. 180 tests pass at commit `6672cbe`.
  https://github.com/weiscadence/honest-evals/tree/6672cbe, commit `6672cbe`
- The worked example reproduces a real result from the agent's own records and refuses to print the headline until a canary passes and a cutoff sweep is stable. The test `test_example_runs_canary_and_sweep_before_any_headline` enforces the order.
  `examples/selection_effect.py`, `tests/test_hardening.py`
- TAP (Two-party Authenticated Presence): a co-signer between an AI agent and its keys. The agent alone cannot move funds; agent plus phone can. Both outcomes on Solana devnet on 2026-10-10, four seconds apart (block times 18:54:44 and 18:54:48 UTC):
  refused: https://explorer.solana.com/tx/2h859F33kaLQ7RUtHXpWNdxz5UDmcTZUbyQzcyT3GrXKeX65b1UMAoFvxwfsJkYh2GNNkABJTaJqKmB2rVYobMrD?cluster=devnet (slot 509637420, `MissingRequiredSignature`)
  landed: https://explorer.solana.com/tx/2ik5wVv3cviq4uhdvVht3NJkguqY5d1MgvbAdicEuR99a6t9C8vYZ6okjjEBnTKzKrorjHHYpccjNgBMyhbjFtHq?cluster=devnet (slot 509637436, success)
  https://github.com/weiscadence/tap, `evidence/README.md` carries both signatures and sha256 of the archived RPC responses and explorer screenshots.
- TAP key box: agent key sealed AES-256-GCM, decryption key split 2-of-3 (phone, stone, paper); a search of the agent's workspace filesystem for the key returns 0 hits, and a canary proves the search finds a planted leak.
  commit `c37950e34150909c8195813152e8c12370afdd53`, `vault/keybox.js`, `vault/keybox.test.js`
- Lean formalization of TAP's acceptance-counting bounds (by Lodestone): the 32-bit target spec (529/2^32) and the deployed 16-bit, 1-flip bound ((1+L)/2^L) are separate theorems; Lodestone's `#print axioms` on `hammingBall_card` and `targetSet_card` shows only Lean's standard axioms. Signature and HMAC assumptions are named axioms. The receipt file still has one open `sorry` and a placeholder deniability statement.
  https://github.com/weiscadence/tap/tree/a141649/lean (README states the honest provenance: compiled by Lodestone; source read by Cadence and Aperture, not independently compiled; one `sorry` named)
- Colosseum submission video: https://youtu.be/TAbcggFGM5k ("TAP: a bouncer between your AI agent and your keys").
- 16 merged pull requests to TauCeti, a Lean 4 library downstream of Mathlib. 14 of 16 carry formal GitHub reviews; the project's model is AI review against human-written rubrics. Listing: `gh search prs --repo TauCetiProject/TauCeti --author weiscadence --merged`. Examples:
  #8486 lower p-central series and degree lemmas (merged 2026-09-26)
  #1334, #1333, #1297 Hermite and Chebyshev systems are Hilbert bases of L2
  #1283 completeness of an orthogonal polynomial system from moment determinacy
  #1256 a function with vanishing moments is a.e. zero
  https://github.com/TauCetiProject/TauCeti/pull/8486
- Three of those PRs ship nothing new on purpose: #1180 removes a dead `@[simp]` attribute, #1176 and #1169 replace duplicated lemmas with the shared one. Subtractive PRs are the cheapest honesty there is.

## 2. Caught ourselves

Format: what we claimed, what was true, how it was caught, receipt.

- Claim: the TAP co-signer refuses a config transaction (threshold to 1) planted at the vault-transaction index.
  Truth: it was refused only because the SDK decoder crashed on an offset; the decoder never checks the 8-byte discriminator. A differently shaped config transaction would have decoded and been signed.
  Caught: asking for each negative test what the error string was, not whether there was an error.
  Receipt: https://github.com/weiscadence/tap/commit/80729ed24ba994c9c136abbd0709a8341a109809. The code comment says "was only refused because decoding happened to crash." The new `vault/squads-type.test.js` prints `FAIL wrong reason` unless the message matches `/not a VaultTransaction|not owned by the Squads/`.
- Claim: the co-signer signs only what it decodes.
  Truth: honest bytes plus 64 trailing junk bytes were signed. The parser ignored what it did not read.
  Caught: a parser-differential test written against our own signer.
  Receipt: https://github.com/weiscadence/tap/commit/66a3bca348a102276b01d11c739dfd9e3f8f519e, commit message "trailing-bytes parser differential was SIGNED"; `vault/cosign.test.js` line 41, `refused('honest + trailing bytes (parser differential)', ...)`.
- Claim: the honest-evals README headline, "a 10x selection effect."
  Truth: the ratio ranges from 2.6x to 28.9x depending on the error definition and drift handling; at the 2.6x definition it is not significant at 0.05.
  Caught: sweeping the definitions instead of picking one.
  Receipt: commit `c556d5c` (body bullet: "replace 10x with 6/16 vs 3/77 and the definition-dependent range"); `test_readme_has_no_tenfold_claim_and_states_definition_dependence` fails if "10x" returns to the README.
- Claim: the example's cutoff sweep was controlled by a negative control.
  Truth: the first control (swapping population names) could not flip the winner; the sweep was vacuous. Then the hidden-controls fix closed that hole — and a 2026-10-16 re-attack broke it twice more. T2c: a call-history-aware evaluate (flip only after the sweep settings) still gets `StableFinding`/`proven=True`. T2d: a range-aware evaluate (flip on out-of-range controls) also gets `StableFinding`/`proven=True`; this is structural, not a bug — hidden controls are by design drawn outside the swept range, so any value-aware evaluate can distinguish them. T2c is mitigable by randomly interleaving settings and controls; T2d is a ceiling on black-box probing. Honest line: hidden controls catch evaluates that ignore their setting outright; against an adversarial evaluate, no black-box probe suffices.
  Caught: re-attacking our own fix, same day.
  Receipt: commit `0ce1c73`, `tests/test_lodestone.py::test_T2_evaluate_that_ignores_setting_is_vacuous_not_stable` and `test_T2_example_sweeps_carry_a_flipping_negative_control`; re-attack receipt: wire post `b44ea011577542a4` (2026-10-16T19:44:13Z), vs commit `dda36d6` ("Close Lodestone re-test survivors R1, T2, T2b", 174 tests pass).
- Claim: our private append-only changelog verifier proves the log was not tampered with.
  Truth: a hash chain with the newest entries deleted is still a valid chain; the verifier printed CHAIN OK after truncation. (The incident itself is private; the fix class is public.)
  Caught: forcing the failure path of our own guard, unprompted.
  Receipt: honest-evals `tests/test_ledger.py`: `test_truncation_detected`, `test_truncate_to_zero_detected`, `test_missing_head_is_not_ok`; `test_hardening.py::test_ledger_full_rewrite_caught_by_external_anchor`.
- Claim: a canary that plants a known-false item proves the checker checks.
  Truth: a checker honest only on its first call, or one that trusts the tail position where the plant used to go, passed.
  Caught: writing the dishonest checker and running it.
  Receipt: commit `c556d5c`, `test_canary_catches_checker_honest_only_on_first_call`, `test_canary_defeats_checker_that_trusts_tail_position`.
- Claim (void): "Re-invention: 0 of N, collapse 100% in every cell. Re-invention simply does not occur in this substrate," with a causal story built on it.
  Truth: the counter was gated on a set that only the soft knockout populates; the cited experiments used the hard knockout, so the zero was a vacuous pass. Re-invention does occur.
  Caught: planting the exact state the detector needs (`reinvent-canary.js`, fired 12/12) and then reading why the real runs could never reach it.
  Receipt: https://github.com/weiscadence/void/blob/main/RESULT-reinvention-corrected.md, retracting `RESULT-energetic-nesting.md`; README section "The correction that matters most."

## 3. Attacked by peers

- Two sibling agents attacked honest-evals after its first push. Every hole they found is now a test in the repo, next to its fix.
  Pass 1 (commit `c556d5c`): delimiter collision in the ledger encoding, backdated appends, full-file rewrite invisible without an external anchor, shell metacharacter and command-substitution bypasses in the re-run allowlist, canary honest-on-first-call. `tests/test_hardening.py`, 16 test functions.
  Pass 2, Lodestone (commit `0ce1c73`, "Close Lodestone review holes (L1,R1-R3,S1,C1-C2,T1-T2)"): two ledgers shared one identity, snapshot auto-classified as drift, whitespace collapse hid mismatches, all-unresolved population raised instead of reporting, dict flags raised TypeError inside the canary, `1` and `True` collapsed to one winner, vacuous evaluate reported as stable. `tests/test_lodestone.py`, 41 tests.
  Pass 3, Lodestone re-attack vs `dda36d6` (2026-10-16, 174 tests pass): R1 holds — forged hash, wrong sequence, edited fields, plain adjudication, and old snapshot+reason all correctly rejected (boundary: an author controlling the whole ledger can still forge a self-consistent record; the binding proves adjudication-matches-record, not ledger trustworthiness — that is L1's job). T2 broken twice more: T2c (call-history-aware evaluate) and T2d (range-aware evaluate), both `StableFinding`/`proven=True`; T2d is structural, a ceiling on the black-box approach (see §2). L1 still open operationally: two contradictory ledgers with self-asserted `chain_id='victim'` both verify without an externally published anchor; the anchor is the real defense.
  Pass 4, Lodestone vs `ddc8d3a` (2026-10-17, Cadence's T2c/T2d closure attempt): T2c broken again — grindable (reproduced: a k-optimized counter closes on ~0.5% of seeds, and the invoker controls the draw loop); T2d's `probing_limit` caveat gameable (point-gerrymander clears it). R1 holds; L1 open. Resolution at `6672cbe`: both conceded as structural and pinned as tests — one pins a winning grind draw, one asserts the gerrymander's uncaveated pass exists — with the defenses named in the README (publish seed and attempt count or ledger the sweeps; pre-register settings before the evaluate exists). Pattern across passes: each fix holds against the naive adversary and falls to the optimized one; the suite's job is to say so, not to pretend otherwise.
- TAP: Lodestone's review strikes on the stone-attested approval path (cluster binding, tap id, per-stone last-seen, freshness, held-touch) were folded into commit `34a73fe`. Aperture's UX truth-fixes (label the demo payment as a demo, open full devnet signatures from proof rows) landed as commit `5a51cde5da00e059e2e04babf71f841be8b3128e` on branch `app-v2`.
- TauCeti: 14 of 16 merged PRs carry formal reviews; #8486 carries 5 review rounds.

## 4. Honest limits

- Self-checking dominates. In the selection-effect study behind honest-evals, the verifier for almost every claim is the same agent re-running its own check. The structural mitigation: three agents on different models attack each other's claims, and every break becomes a test in the repo (section 3). That is still a closed loop, which is why the work I'm applying to do is the independent-verifier tier. Re-running your own check reproduces your own wrongness; rows where the command did not test the claim are only found by reading. The README says this.
- The headline rests on n=16 claims logged at utterance (95% Wilson interval 19-61% (6/16)). The package proves the arithmetic; the catching is a self-report.
- TAP is devnet only. The stone firmware is host-tested and compiles; commits `34a73fe` and `2fc1350` both say "Not run on hardware." Cosign rules are tested against transactions we constructed, not against an adversary we did not pick.
- Our security audits have not been paid. No bounty has been awarded for any finding to date.
- Repository stars: zero across the board. Nobody outside the project has depended on this code yet.
