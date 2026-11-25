# Verified Security Work

This is a portfolio of Hack The Box work and security engineering research. Every HTB result below was
verified through the official API (`authUserSolve` / `Congratulations`) before being written up. No flag
is claimed without an accepted submission. CVE claims below were verified against NVD on 25 November 2026.

**Target track:** DevSecOps / security engineering
**Focus:** reproducible analysis, verification discipline, and defensive lessons

## Hard challenges

| Challenge | Category | Result | Writeup |
|-----------|----------|--------|---------|
| Raining Primes | Crypto | owned, API verified 2026-11-24 | [01-raining-primes.md](writeups/01-raining-primes.md) |
| Retention | Misc | owned, API verified 2026-11-24 | [02-retention.md](writeups/02-retention.md) |
| Kernel Monarch | Reversing | owned, API verified 2026-11-24 | [03-kernel-monarch.md](writeups/03-kernel-monarch.md) |

Rank at time of writing: Hack The Box Prodigy, 171 challenges and 19 Sherlocks owned, all API verified.

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

## Verified MCP threat landscape

Six ecosystem CVEs confirmed against NVD. Two earlier score and attribution errors in working notes were
corrected from the primary records.

| CVE | CVSS | Flaw |
|-----|------|------|
| CVE-2025-6514 | 9.6 | `mcp-remote` OS command injection |
| CVE-2025-49596 | 9.4 | MCP Inspector unauthenticated stdio RCE |
| CVE-2025-54136 | 8.8 | Cursor persistent code execution via a trusted MCP config |
| CVE-2026-33032 | 9.8 | nginx-ui unauthenticated MCP message endpoint |
| CVE-2026-19516 | 9.1 | Grafana MCP SSRF through `X-Grafana-URL` |
| CVE-2026-15583 | 8.6 | Grafana MCP confused-deputy service-account token leak |

Protocol-level risks: no capability attestation, unauthenticated bidirectional sampling, and implicit
trust propagation across servers. OWASP's MCP Top 10 ranks prompt injection, tool poisoning and supply
chain compromise first.

## Security engineering practice

- **Deterministic tool plane:** 1417 tools in 70 live categories, called by direct HTTP POST to a local
  tool API, never through a chat webhook. Categories include bugbounty, penetration testing, forensics,
  reverse engineering, OSINT, network, wireless and infrastructure.
- **Isolation:** gVisor `runsc` strict profile for untrusted servers, no network, read-only root filesystem,
  all Linux capabilities dropped, no new privileges, bounded CPU, memory and PIDs, memory-backed scratch.
- **Pre-deploy:** mcp-scan, Ramparts, Cisco MCP Scanner, Trivy and Grype for static and supply-chain review.
- **Red team:** Promptfoo, garak, DeepTeam, AgentDojo and mcp-context-protector for tool poisoning,
  prompt injection and agent supply-chain risk.
- **Runtime:** Lasso Gateway and Docker MCP Gateway for secret masking and egress control.
- **State and queues:** authenticated Redis on loopback port 6380 with strict systemd sandboxing.
- **Secret leakage:** four detection boundaries, pre-tool argument scanning, post-tool result redaction,
  egress DLP and read-then-write sequence alerts. Credentials should never enter the context window;
  detection is the safety net.
- **Delivery pipeline:** lint schemas, block CI on tool-definition findings, scan images at build, check
  configuration and pin versions at deploy, control egress and log every call.

## Method

1. Download and read the challenge source or binary.
2. Build a local clone of the vulnerable logic where possible.
3. Prove the attack locally against a fake flag or generated key.
4. Run the verified chain exactly once against the live service.
5. Submit the flag through the official API and confirm ownership.
6. Stop the instance.
7. Write up the technical chain and the defensive lesson.

## Note on scope

This portfolio contains only technical challenge material and public vulnerability research. Nothing from
any private or sacred project document is included, and nothing here is sent to external services beyond
the official Hack The Box API and public vulnerability databases.
