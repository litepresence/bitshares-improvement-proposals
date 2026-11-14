# CLSAG/BP+ Checklists (reference)

Source: `SKILL.md` v2.0.0 Phase 4 + Cryptographic Invariant Checklist, moved byte-identical in v2.1.0. Load alongside Phase 4; work every applicable list, do not sample.

## CLSAG Review Checklist

- [ ] Ring public keys are parsed and validated before use.
- [ ] Ring size bounds are enforced before indexing.
- [ ] Ring index calculations cannot underflow or overflow.
- [ ] Duplicate ring entries are handled according to consensus rules.
- [ ] Key image is parsed as a valid point.
- [ ] Key image identity check is present where required.
- [ ] Key image subgroup check is present where required.
- [ ] Key image uniqueness is enforced by the chain or mempool layer.
- [ ] The verifier does not assume uniqueness from the signature alone.
- [ ] The CLSAG challenge transcript includes all binding public inputs.
- [ ] The message being signed includes fee and relevant commitments where
      required.
- [ ] Scalar values are reduced or checked before use.
- [ ] Point reconstruction uses the correct base points.
- [ ] `scalarmult8` or equivalent cofactor handling is used correctly.
- [ ] Multiexponentiation handles zero scalars and identity points safely.
- [ ] No early return can skip a decisive verification step.
- [ ] No debug flag or test flag can disable verification.
- [ ] Failure paths return false or fail closed.

## BP+ Review Checklist

- [ ] Number of commitments is bounded before expensive work.
- [ ] Number of inner product proof elements is bounded.
- [ ] Bit width or digit width matches consensus rules.
- [ ] Aggregation size limits are enforced where required.
- [ ] If bounds are delegated to callers, every caller enforces them.
- [ ] All proof points are validated before use.
- [ ] All scalar values are validated or reduced before use.
- [ ] The verifier rejects malformed lengths instead of truncating.
- [ ] The verifier rejects invalid encodings instead of coercing them.
- [ ] The transcript includes commitments, proof statement, and public fee.
- [ ] The transcript does not omit any field that can vary without invalidating
      the proof.
- [ ] Challenge derivation is deterministic.
- [ ] Inverse operations are guarded against zero or invalid inputs.
- [ ] Batch verification combines all equations correctly.
- [ ] Batch verification returns false if any component fails.
- [ ] Batch verification randomness is not attacker controlled.
- [ ] Proof verification is not skipped for zero value outputs unless consensus
      explicitly allows it.
- [ ] Amount extraction happens only after proof verification.
- [ ] Range proof covers every confidential output in the transaction.
- [ ] Range proof does not cover commitments that are not present.

## Pedersen Commitment and Balance Checklist

- [ ] Input pseudoOuts are summed correctly.
- [ ] Output commitments are summed correctly.
- [ ] Fee is converted to a point contribution correctly.
- [ ] Fee scalar cannot overflow.
- [ ] Fee is not negative unless consensus explicitly allows it.
- [ ] Balance check uses exact point equality.
- [ ] Balance check is not replaced by scalar equality unless proven safe.
- [ ] Fee zero case is handled correctly.
- [ ] Coinbase or reward outputs do not bypass balance rules unless intended.
- [ ] Burn outputs are handled according to consensus rules.
- [ ] No commitment can be opened to two different values under the same
      verifier path.

## Serialization and Parsing Checklist

- [ ] Vector lengths are bounded before allocation.
- [ ] Varint decoding rejects overflow.
- [ ] Integer width conversions are checked.
- [ ] Byte buffers are not indexed before length checks.
- [ ] Point decompression failures are handled.
- [ ] Non canonical encodings are rejected if consensus requires it.
- [ ] Optional fields cannot hide mandatory proof data.
- [ ] Old transaction versions are parsed according to fork rules.
- [ ] New transaction versions cannot be downgraded silently.
- [ ] Parser errors fail closed.
- [ ] Exceptions or error codes cannot be swallowed by callers.

## Consensus Boundary Checklist

- [ ] Every transaction path calls proof verification.
- [ ] Mempool and block validation use the same proof rules.
- [ ] RPC submission cannot bypass verification.
- [ ] Test hooks are not compiled into production builds.
- [ ] Compile flags such as `NDEBUG` do not remove consensus checks.
- [ ] Assertions are not the only defense for consensus critical rules.
- [ ] Key image used set is checked before acceptance.
- [ ] Key image used set is checked across mempool and chain state.
- [ ] Reorg behavior does not allow key image replay unless intended.
- [ ] Fork version gating cannot disable proof checks unintentionally.

## FFI and Cross Language Checklist

Use this if the target includes Rust, C, or another bridge around proof code.

- [ ] FFI inputs are validated on both sides.
- [ ] Pointers are checked for null and valid length.
- [ ] Ownership rules are clear and deterministic.
- [ ] Rust panics do not unwind into C++ consensus code.
- [ ] C++ exceptions do not cross FFI boundary.
- [ ] Error codes are checked by every caller.
- [ ] Serialization format matches exactly across languages.
- [ ] Scalar and point validation rules match across languages.
- [ ] Allocator failure is handled safely.
- [ ] FFI tests cover malformed input, empty input, and oversized input.

## Cryptographic Invariant Checklist

- [ ] Curve point operations use the correct curve and representation.
- [ ] Point addition and scalar multiplication are correct for edge cases.
- [ ] Cofactor handling is correct where required.
- [ ] Subgroup membership checks are enforced where required.
- [ ] Identity point checks are enforced where required.
- [ ] Hash to point domain separation is correct and matches the protocol.
- [ ] Pedersen generators are independent and correctly derived.
- [ ] Pedersen commitments are binding under the implemented arithmetic.
- [ ] Fiat-Shamir transcript includes all public inputs that affect soundness.
- [ ] Fiat-Shamir transcript is not malleable in a way that changes binding.
- [ ] Range proof covers the intended bit width.
- [ ] Range proof covers all required commitments.
- [ ] Balance invariant holds as an exact point equation.
- [ ] Key image uniqueness is enforced by persistent consensus state.
- [ ] Key image validity is enforced before use.
- [ ] Batch verification preserves fail closed behavior.
