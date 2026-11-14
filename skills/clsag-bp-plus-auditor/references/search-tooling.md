# Search Patterns and Tooling (reference)

Source: `SKILL.md` v2.0.0 Search Patterns + Tooling sections, moved byte-identical in v2.1.0.

## Search Patterns

Use targeted searches to locate high risk code.

```bash
rg -n "verify.*proof|proof.*verify" src
rg -n "bulletproof_plus|bp_plus" src
rg -n "clsag|CLSAG" src
rg -n "key_image|keyImage" src
rg -n "outPk|pseudoOut|txnFee" src
rg -n "scalarmult8|isInMainSubgroup|hash_to_point|hash_to_p3" src
rg -n "CHECK_AND_ASSERT|throw|return false" src/ringct
rg -n "assert\(" src/ringct
rg -n "memcpy|memmove|reinterpret_cast|const_cast" src/ringct
rg -n "resize|reserve|data\(\)|begin\(\)|end\(\)" src/ringct
```

Use these as starting points. Do not rely only on pattern matching.

## Tooling

Use manual review first. Use tools to extend coverage.

| Tool or method | Purpose |
|---|---|
| Manual code review | Primary consensus and proof logic analysis |
| `rg`, `glob`, file inventory | Locate proof and consensus surface |
| Unit tests | Confirm intended valid and invalid behavior |
| Sanitizers | Detect memory safety and undefined behavior |
| Differential testing | Compare independent implementations |
| Fuzzing | Find parser and verifier edge cases |
| Static analysis | Surface suspicious patterns, not final verdicts |
| Formal proof review | Compare implementation against security arguments |
| Cryptoscope or similar | Static crypto usage review if available |
| Picus or similar | Formal or symbolic verification if available |
| Independent Rust or C implementation | Cross validation if available |

Record every tool run, version, command, and relevant output. Record false
negative risk for each tool.

## Dynamic Validation Procedures

### Build and Unit Tests

Use the project build system. Record exact commands.

Example:

```bash
cmake -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build -j
ctest --test-dir build --output-on-failure
```

If the project uses another system, use equivalent commands and record them.

### Sanitizer Runs

If permitted, run relevant test targets under sanitizers.

Examples:

```bash
cmake -B build-asan -DCMAKE_BUILD_TYPE=Debug -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined"
cmake --build build-asan -j
ctest --test-dir build-asan --output-on-failure
```

Look for:

- heap overflow,
- stack overflow,
- out of bounds read,
- uninitialized read,
- signed integer overflow,
- null dereference,
- misaligned access,
- leaked proof object that affects consensus state.

### Differential Testing

Where an independent implementation exists, compare behavior.

Check:

- valid proofs verify in both implementations,
- invalid proofs reject in both implementations,
- serialization round trips match,
- transcript hashes match,
- point validation rules match,
- error conditions match consensus expectations.

If no independent implementation exists, state that differential validation
was not performed.

### Fuzzing

If fuzzing infrastructure exists, target:

- transaction deserialization,
- proof deserialization,
- point decoding,
- scalar decoding,
- vector length parsing,
- verifier entry points,
- batch verification wrappers.

If fuzzing is not possible, propose seed corpus and harness locations.
