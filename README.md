# Verified Security Work

This is a local portfolio of Hack The Box work. Every result below was verified through
the HTB API (`authUserSolve` / `Congratulations`) before being written up. No flag is
claimed without an accepted submission.

**Target track:** DevSecOps / security engineering
**Focus:** reproducible analysis, verification discipline, and defensive lessons

## Hard challenges

| Challenge | Category | Result | Writeup |
|-----------|----------|--------|---------|
| Raining Primes | Crypto | owned, API verified 2026-11-24 | [01-raining-primes.md](writeups/01-raining-primes.md) |
| Retention | Misc | owned, API verified 2026-11-24 | [02-retention.md](writeups/02-retention.md) |
| Kernel Monarch | Reversing | owned, API verified 2026-11-24 | [03-kernel-monarch.md](writeups/03-kernel-monarch.md) |

## What these demonstrate

- Cryptanalysis with a reproducible local proof before live exploitation (AGCD, continued
  fractions, algebraic RSA factorization).
- Security-policy analysis: finding and exploiting a certification/deployment state
  mismatch in a verifier, which is the same class of bug as sandbox and admission-control
  bypasses.
- Kernel-level reversing: understanding ftrace syscall hooks, rootkit self-hiding, and
  recovering artifacts from kernel memory through `/proc/kcore`.
- Verification discipline: every exploit carries a cheap, specific negative check
  (`n % p == 0`, exact flag byte match, API acceptance) so a plausible-looking result
  cannot be mistaken for a correct one.
- Clean incident hygiene: challenge containers are stopped immediately after the flag is
  submitted.

## Method

1. Download and read the challenge source or binary.
2. Build a local clone of the vulnerable logic where possible.
3. Prove the attack locally against a fake flag or generated key.
4. Run the verified chain exactly once against the live service.
5. Submit the flag through the official API and confirm ownership.
6. Stop the instance.
7. Write up the technical chain and the defensive lesson.

## Note on scope

This portfolio contains only technical challenge material. Nothing from any private or
sacred project document is included, and nothing here is sent to external services beyond
the official Hack The Box API.
