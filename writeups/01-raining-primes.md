# Raining Primes (Hack The Box, Hard Crypto)

**Status:** owned and API verified, 2026-11-24
**Result:** `HTB{5cb474c854fb15b1cf85561c70debf96}`
**Skills:** approximate GCD, continued fractions, RSA key derivation, algebraic factorization, AES-ECB, verification-driven exploitation

## Summary

The service generates a secret 640-bit prime `r`, then builds RSA primes of the form
`p = a*r + b` with 384-bit `a` and 256-bit `b`. It also lets a player update a 256-bit
AES key through a public-key transform, then returns `AES(key, flag)` wrapped in RSA.

The solve has four independent stages. Each stage was proved against a local clone of
the challenge before being run against the live service.

1. Recover `r` from leaked primes with a continued-fraction AGCD attack.
2. Force the AES key to 32 zero bytes through the key-update oracle.
3. Factor the RSA modulus algebraically using the recovered `r`, with no general-purpose factoring.
4. Decrypt RSA, then AES-ECB with the known zero key.

## Stage 1: recovering r

Option 1 prints fresh primes from `gen_prime()`:

```
p_i = a_i * r + b_i
a_i < 2^384, b_i < 2^256, r is a 640-bit prime
```

For two samples `p1/p2`, the convergent `c1/c2` of the rational `p1/p2` can match the
unknown ratio `a1/a2` when the residues `b_i` are small relative to `r`. Six samples
give fifteen pairs, which is enough to find a convergent where:

- `c1` and `c2` are at most 390 bits,
- `rc = p1 // c1` is exactly 640 bits,
- every collected sample satisfies `(sample % rc).bit_length() <= 264`.

That last check is the verification gate. A false candidate fails it immediately.
This recovered `r` on the first live attempt after being validated locally.

## Stage 2: forcing the AES key to a known value

`update_key` rebuilds the public key from the current 256 key bits. For each bit:

```
m_i = 2 * random_b_i + key_bit_i
k0_i = random_a_i * r + m_i
```

The player supplies `k1_i` with `r < k1_i <= 2^384 * r`, and the server computes:

```
new_bit_i = (k0_i * k1_i mod r) mod 2
          = (m_i * k1_i mod r) mod 2
```

Choosing `k1_i = r + t` with small `t` gives `new_bit_i = (m_i * t) mod 2`:

- `t = 1` preserves every bit,
- `t = 2` forces every bit to zero,
- `t = 2r - 1` complements every bit.

Sending `k1_i = r + 2` for all 256 positions sets the AES key to 32 zero bytes.
This is a clean example of a malleable key-update design: an attacker-controlled
transform over a secret, combined with a parity-only output, lets an attacker choose
the resulting key without learning the original one.

## Stage 3: factoring n without factoring

With `r` known, the RSA modulus can be split algebraically. Write:

```
n = (a_p*r + b_p)(a_q*r + b_q)
  = A*r^2 + M*r + B
```

where `A = a_p*a_q`, `M = a_p*b_q + a_q*b_p`, and `B = b_p*b_q`.

Because `b_p*b_q < 2^512 < 2^640 ~ r`, the value `B = n mod r` is exact. Also
`A = floor(n / r^2) - k` for a small `k` in the range 0 to 7. For the correct `k`:

```
M = (n - A*r^2 - B) / r
M^2 - 4*A*B = (a_p*b_q - a_q*b_p)^2
```

The discriminant is a perfect square, so `D = isqrt(M^2 - 4AB)` is exact. Then
`gcd(A, (M+D)/2)` recovers `a_p`, and the matching offset gives `b_p`, so
`p = a_p*r + b_p`. Four candidate combinations are tried, and `n % p == 0` is the
verification test. Live, the correct branch used `k = 1` and produced a prime factor
immediately.

This avoids the 512-bit semiprime factorization of `B` entirely. Local experiments
confirmed that `sympy.factorint` on `B` was intractable, which is why the algebraic
route was worth deriving.

## Stage 4: decryption

```
d = pow(e, -1, (p-1)*(q-1))
m = pow(c, d, n)
aes_ciphertext = m.to_bytes(...)
flag = unpad(AES.new(b'\\x00'*32, AES.MODE_ECB).decrypt(aes_ciphertext), 16)
```

## Verification and engineering notes

- Every stage was first reproduced locally with `rp_full_test.py`, which generated its
  own `r`, `p`, `q`, key, and ciphertext. The local chain printed `decrypt ok True`
  before any live traffic was sent.
- The live solve is in `rp_solve_live.py`. It collected six primes, recovered `r`,
  forced the key, requested the ciphertext, factored `n`, and printed the flag in one connection.
- The flag was submitted through `POST /api/v4/challenge/own` and accepted with
  `Congratulations`; `authUserSolve` then read `true`. The container was stopped.

## Defensive takeaways

- Do not reuse the same hidden divisor across many published values. Approximate-GCD
  attacks turn a modest sample set into a full secret recovery.
- A key-update API that reveals parity of a secret-derived product is a key-recovery
  primitive, even if the original key never leaves the server.
- Algebraic structure in RSA primes matters. If `p` and `q` are generated from a shared
  secret plus small residues, knowing the secret can be enough to factor `n`.
- Verification discipline: a false AGCD candidate is rejected by a cheap residue check,
  and a false factorization is rejected by `n % p == 0`. These negative checks are what
  keep the solve honest.
