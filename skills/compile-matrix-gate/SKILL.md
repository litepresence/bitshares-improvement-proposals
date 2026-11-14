# Skill: compile-matrix-gate

Use when gating a track split program: run `tooling/build_matrix.py`, check the 7
configuration rows, check log headers, report GO/NO_GO per configuration. Wraps
the matrix driver; never resolves conflicts itself.

## Procedure

1. Clear `/tmp/graphene-tmp*` before any full-binary run.
2. Run `python3 tooling/build_matrix.py`. Capture the summary markdown path.
3. For each of the 7 configurations (F, G, H, FG, FH, GH, FGH):
   - Read its log under `build/logs/matrix-<date>/<name>.log`.
   - Verify the header carries `sha256sum` and `git rev-parse HEAD`.
   - Verify the log ends with `*** No errors detected`.
   - Verify exit code 0 and a recorded case count.
   - Verify the pin suite (`registry_wire_pin_tests`) appears in the run output.
4. A configuration with a merge conflict, configure failure, build failure, or
   non-zero exit is NO_GO for that row. Do not auto-resolve; report the conflict
   text verbatim.
5. Emit the GO/NO_GO table. Any NO_GO row blocks the program exit gate.

## Evidence emitted

`build/matrix-<date>.md` (driver output) plus `build/compile-matrix-gate-<date>.md`
with the per-row verdict, header verification, and the first failing line for any
NO_GO row.

## Red flags

- Calling a row green without reading its log header.
- Auto-resolving a merge conflict in the matrix branch.
- Running two full binaries concurrently (OOM on this box; -j4 max).
- Trusting the driver's summary row without re-checking the underlying log.
