---
layout: post
title: "How do I add key commitment to my AEAD scheme in 2023?"
date: 2023-02-07
url: 2023/02/07/aead-key-commitment.html
---

While working on my master's thesis, I investigated some recently proposed constructions that turn AEADs into key-committing or fully-committing ones. Key commitment has recently gotten a lot more attention and I, therefore, expect this post to be outdated quite soon, as new research emerges. This post serves as a quick collection of personal notes and pointers, that maybe could help someone looking to add key commitment to their AEAD schemes today. There exist constructions proposed in earlier work, but the ones covered herein are the ones I focused on primarily.

An implementation of UtC+HtE and CTX for ChaCha20-Poly1305 with BLAKE2b is available here: [https://github.com/brycx/CAEAD](https://github.com/brycx/CAEAD)


### Key-committing or fully committing?
The first question you need to figure out is whether you only want a key-committing AE or fully committing one.

If an AEAD is key-committing, it means that it commits to the input `K`, such that a ciphertext and its authentication tag decrypt under the key `K` only.

A fully committing AE will commit to the entire input: `(K, N, AD, M)`.

If you deal with protocols where you need message franking, you would for example require a fully-committing AEAD.

Both constructions mentioned in this post are generic, meaning they can add commitment on top of any AEAD scheme and not just, say, AES-GCM.

### UtC, RtC and HtE by Bellare and Hoang

These transformations have been described in the 2022 paper ["Efficient Schemes for Committing Authenticated Encryption"](https://eprint.iacr.org/2022/268) by Mihir Bellare and Viet Tung Hoang. They define two generic constructions that turn either a nonce-based AEAD (nAEAD) into a key-committing scheme or a misuse-resistant AEAD (MRAE) into a key-committing scheme.


#### UtC (UNAE-then-Commit)
UtC adds key commitment to any nonce-based AEAD scheme. It uses what Bellare and Hoang call a _committing PRF_ (`F`), to derive a commitment block `P` and subkey `L` from the key and nonce. The subkey `L` is what is used as the key for the underlying AEAD and `P` is appended to the ciphertext.

```
(P, L) ← F(K, N)
C ← AEAD(L, N, A, M)
C ← P || C
```

Bellare and Hoang propose a specific instantiation of a _committing PRF_ based on AES, in their paper.

#### RtC (MRAE-then-Commit)
RtC adds key commitment to any misuse-resistant AEAD (MRAE) scheme. RtC utilizes a committing PRF, just as UtC does, but additionally incorporates a collision-resistant PRF `H`.

```
(P, L) ← F(K, N)
C ← MRAE(L, N, A, M)
T ← H(P, C[1 : n])
```

We again get a commitment `P`, but this time it's used to generate the hash output `T`, which is appended to the ciphertext. We assume there that the ciphertext from MRAE is at least `n` bits long, where `n` is the output length of `H`.


#### HtE (Hash-then-Encrypt)
This transform takes any key-committing schemes (what Bellare and Hoang call CMT-1) and turns it into a fully committing scheme (what Bellare and Hoang call CMT-4). This means this can be put on top of both UtC and RtC.

```
L ← H(K, (N, A))
C ← CMT-1-AE(L, N, ε, M)
```

We derive a subkey `L` which commits to the additional data and use this to encrypt with a CMT-1 scheme (ε is simply an empty string).


### CTX by Chan and Rogaway
CTX is described in the 2022 paper ["On Committing Authenticated Encryption"](https://eprint.iacr.org/2022/1260) by John Chan and Phillip Rogaway. This construction only applied to nAEADs, but is much simpler than UtC while achieving full commitment equivalent to UtC+HtE, for the cost of a single PRF pass.

```
C, T ← nAEAD(K, N, A, M)
T* ← H(K, N, A, T)
C || T* 
```

CTX takes the authentication tag produced by the MAC, eg. Poly1305, and replaces it with an alternative tag produced by a collision-resistant hash function. This alternative tag replaces the original tag entirely.


### Takeaways

- CTX gives full commitment at the cost of a single PRF pass, compared to two in the case of UtC+HtE and three in the case of RtC+HtE.

- UtC and RtC add a commitment block to the ciphertext. As modeled in the paper, this requires an additional verification routine on top of the underlying authentication tag verification. This means, a possible side-channel is introduced, about whether or not some additional data passed was correct. The actual impact on real-world protocols of this is unknown to me, however.

- CTX replaces the original authentication tag entirely, so there exists only one authentication tag verification routine, as is the case with original AEADs.

- If you work with an nAEAD, the straightforward choice in terms of simplicity and security provided seems to be CTX. If you work with an MRAE, then the only choice is RtC.

- Any hash function used for HtE or CTX __must not__ be vulnerable to length-extension attacks (for example SHA-256).

- The security of the commitment schemes reduces to the collision resistance of the PRFs used. Thus, choosing a large enough output size for the PRF is important. Chan and Rogaway suggest 160-bit, but the reason for this is unclear.

- It is a lot easier to simply provide a fully-committing AEAD instead of deciding whether you need that or only key commitment. Especially when designing an API for a cryptographic library, I believe it is better not to leave this decision to the user and simply force the use of a fully committing scheme.

### Errata

- Correction to what key commitment implies (previously said to commit to the nonce which was incorrect). Thanks to Neil Madden for pointing this out. 
