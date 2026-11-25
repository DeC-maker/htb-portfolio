# Retention (Hack The Box, Hard Misc)

**Status:** owned and API verified, 2026-11-24
**Result:** `HTB{th3_v3r1f13r_n3v3r_54w_4ft3r_th3_f1rst_sc4n}`
**Skills:** policy verifier bypass, state-machine reasoning, certification/deployment divergence, TOCTOU, defensive engineering

## Summary

Retention is a compliance-range simulator. A field engineer submits a small scan program
in a custom assembly-like language. A verifier must certify the program as safe before a
rig deploys it. The flag lives in a restricted memory vault at addresses 40 through 87.
The verifier rejects any program that reads the vault, either by literal address or
during a certification run.

The bug is not in the parser. It is a mismatch between the state model used for
certification and the state model used for deployment. A certified program can read the
whole vault while never touching it during certification.

## The two execution modes

Both modes run the same program 650 times, but they differ in one argument passed to the
interpreter:

- Certification: `run_deployment(..., verify_mode=True)`. Registers persist across scans.
  A `cleared` set is created once, so each named `BLOCK` executes at most once in the
  entire certification run.
- Deployment: the rig calls `exec_stmts` directly with `cleared=None`, so every `CALL`
  executes on every scan. Registers also persist across scans in the deployment loop.

That single difference is the whole challenge. A block that increments a register runs
once in certification but 650 times in deployment.

## The exploit program

```asm
BLOCK inc
ADD p 1
ENDBLOCK
CALL inc
IF p == 1
SET bank 1
ENDIF
IF p == 33
SET bank 2
ENDIF
MOV off p
SUB off 1
READ v [bank,off]
OUT v
```

Certification:

- `p` becomes 1 on the first scan and is never incremented again, because `inc` is in
  `cleared`.
- `bank` is set to 1 by `IF p == 1`, and `IF p == 33` never fires.
- `off = p - 1 = 0`, and `READ [bank,off]` resolves to address `1*32 + 0 = 32`.
- Address 32 is outside the vault. Every certification read is safe, so the program is
  accepted.

Deployment:

- `p` increments on every scan: `p = scan number`.
- `bank` is set to 1 at scan 1 and to 2 at scan 33, then persists.
- `off = p - 1`; the memory lookup applies `off % 32` and `bank % 32`.
- Scans 9 through 32 walk bank 1 offsets 8 through 31, which are addresses 40 through 63.
- Scans 33 through 56 walk bank 2 offsets 0 through 23, which are addresses 64 through 87.
- `OUT v` emits one flag byte per scan.

The 48 flag bytes arrive in scan order from scan 9 to scan 56. Reading `outs[8:56]` from
the API response gives the flag. The first eight outputs are zeros because the address
is below the vault, which is itself a useful confirmation that the state machine behaved
as predicted.

## Verification discipline

- The exploit was built and tested against the downloaded `vm.py`, `rig.py`, and
  `verifier.py` with the local fake flag. The local run printed `MATCH True` before any
  live submission.
- Live, `POST /api/submit` returned `accepted: true` and `scan program certified safe for
  deployment`, with 650 scan records. The flag was extracted from the `outs` fields.
- The flag was submitted through `POST /api/v4/challenge/own` and accepted. The container
  was then stopped.

## Defensive takeaways

This is a textbook certification/deployment divergence. The verifier proved safety under
a state model that did not match production. Concrete lessons for real systems:

- A policy verifier must model the production execution semantics, including loop counts,
  callback multiplicity, and which state is reset between iterations.
- "Each block runs once" is an optimization, not a security boundary. Any certification
  that relies on one-shot execution must also prove that production cannot call the same
  block repeatedly.
- Computed addresses must be range-checked at execution time, not only at parse time.
  The static check here only examined literal `READ` addresses, so `[bank,off]` reads
  were invisible to it.
- Register values that persist across iterations become covert counters. Anything that
  changes between certification and deployment can be used as a branch oracle.
- Test the negative case. The local fake flag plus the exact byte-offset assertion
  (`MATCH True`) caught an earlier bug where the bank was reset every scan and the same
  25 bytes were emitted repeatedly.

## Relevance to security engineering

This challenge maps directly to CI/CD policy bypasses, sandbox escape through state
confusion, and verification gaps between a test harness and a production runtime. The
same reasoning applies when reviewing a Terraform plan gate, a Kubernetes admission
controller, or a static analysis gate that assumes single-pass execution.
