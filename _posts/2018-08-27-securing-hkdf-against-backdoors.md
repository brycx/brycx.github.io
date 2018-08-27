---
layout: post
title:  "Securing HKDF - backdoor resistance using salts"
date:   2018-08-27
---

### About
In the paper "Backdoored Hash Functions: Immunizing HMAC and HKDF" by Marc Fischlin, Christian Janson and Sogol Mazaheri (Ref 1), there are two proposed solutions. The first is to make HMAC resistant to a backdoored compression function of the hashing primitive, using a random cascade construction that replaces the compression function.

The second is to make HKDF backdoor-resistant using random salts in the compression function of the hashing primitive. A method first introduced by Shai Halevi and Hugo Krawczyk (Ref 2). In this post I will focus on the implementation of making HKDF backdoor-resistant and talk about some drawbacks I think will prevent it from being adopted into existing protocols.

I won't be going into details about the specific definitions, proofs, etc. These are all available in the papers listed in the "References" section. "Backdoored Hash Functions: Immunizing HMAC and HKDF" also discusses the likelihood of- and how to plant backdoors in collision-resistant hash functions.

On the definition of a backdoor:
> "_A backdoored hash function is a function which is designed by an adversary together with a short backdoor
key, whose knowledge allows for violating the security of the hash function._" (Ref 1: p. 3).

### The proposed solution

> "_Immunizing HKDF boils down to hardening HMAC and therefore the round function h._" (Ref 1: p. 12)

The proposed solution is to change out the compression function `h(x,y)` of the hashing primitive, to one that takes `h(x, y ^ salt)` (Ref 1: p. 13), where `^` denotes the exclusive-OR(XOR) operation. Simply put: to XOR a random salt with each input passed to the compression function.

According to the paper:
> "This alleviates the necessary assumption for the compression function h from collision resistance to some kind of second-preimage resistance." (Ref 1: p. 12).

This proposed solution does not make HMAC or the hashing function itself backdoor-resistant. This **only applies to HMAC when it is being used as a key derivation function** (Ref 1: p. 13).

The following should be noted:
  - The salt has to be a random.
  - The salt needs to be unique for every HMAC call that is processed during the HKDF expansion step.
  - Because the salt is processed in the compression function, the salt needs to be of SHA-* blocksize length.

### Implementing the proposed solution
I have been trying to implement this proposed solution myself, based only on SHA512 and called it [sHKDF](https://github.com/brycx/sHKDF). Short for "salted-HMAC-Based Key Derivation Function". I've been using [RustCrypto's sha2 implementation](https://github.com/RustCrypto/hashes). This allowed for easy additions to the existing interface, such as adding the extended compression function to be used:
```rust
pub fn compress512_with_salt(state: &mut [u64; 8], block: &Block, salt: &[u8]) {
    let mut block_u64 = [0u64; BLOCK_LEN];

    assert!(salt.len() == 128);
    assert!(block.len() == salt.len());

    let mut salted_block = [0u8; 128];

    for (idx, itm) in salted_block.iter_mut().enumerate() {
        *itm = salt[idx] ^ block[idx];
    }

    read_u64v_be(&mut block_u64[..], &salted_block);
    sha512_digest_block_u64(state, &block_u64);
}
```
Apart from this, only ~27 extra LOC were required to make a struct that could
handle initialization, updating and finalization with a salt.

The HMAC and HKDF implementations that are in the sHKDF implementation, have been fully or partly used from my other project [orion](https://github.com/brycx/orion). Currently sHKDF  has the users generate the salts, used for each HMAC call. The sHKDF expansion function will index the passed salts based on iterations, where `idx` denotes iteration count starting at `0`:
```rust
let mut hmac = init(prk, &special_salt[idx * 128..(idx + 1) * 128]);
```

The authors of "Strengthening Digital Signatures via Randomized Hashing", Shai Halevi and Hugo Krawczyk, mentioned in the section "Specification Details" that due the way Merkle-Damgård message padding works, and that in real implementations the XOR with the random salt would likely happen before message padding, a more robust approach should be considered. If the XOR with the random salt were to happen before the message padding, then the message padding would potentially result in having the last blocks' zero bits not XOR'ed with the random salt but remain zero.

The more robust approach would be a two-layer padding approach, which simply pads the message blocks with zero bits, XOR's with the random salt and then passes it to the original hashing function. This is accomplished with sHKDF, as message padding is done on each `input` call when using the hashing function.

Shai Halevi and Hugo Krawczyk also link to a paper called “Randomized Hashing: Specification and Deployment", to be written by themselves and listed to be "_in preperation_" ("Strengthening Digital Signatures via Randomized Hashing" seems to be from 2006). I unfortunately wasn't able to find this paper anywhere.

### Drawbacks and limitations
As the above shows, it is a rather easy solution to implement, though it does have some drawbacks.

**Salt length and exchange**: The requirement of a unique salt for each HMAC call in the expansion step can become a problem. Since each salt is used to XOR one block, of 128 bytes in case of SHA512, the salt can never be below 128 bytes. On top of that, the salt length always increases by a factor of itself. As soon as you overstep the first iteration count, with a derived key length of 65 bytes for example, you need an additional 128 bytes for the salts, since two iterations are now processed. This can especially become tiring when using this with protocols such as the [Signal messaging protocol](https://www.signal.org/docs/specifications/doubleratchet/), where you work with KDF chains resulting in multiple HKDF calls. The only feasible approach I see, is if you could establish shared session-salts, such that the session salts would be re-used for each HKDF call in a given session. The impact on maintaining backdoor resistance with session based salts is unknown to me.

"Backdoored Hash Functions: Immunizing HMAC and HKDF" talks about the problems using the salted HKDF approach in TLS 1.3 PSK mode.

**Precomputation**: Precomputation of the HMAC `key XOR opad` and `key XOR ipad` is not possible here either, since each input to the hash functions are XOR'ed with a unique random salt. This most likely is not too much of a concern regarding HKDF, but if this approach is applicable and were to be used with PBKDF2, the slowdown would be significant. In this case it would be interesting to see how much overhead the first proposed solution would give in this case, when relying on random cascade constructions.

### More
If you have found a mistake or think I'm doing something horribly wrong, please do not hesitate to [reach out](https://brycx.github.io/contact). Any feedback is greatly appreciated. All code for this implementation can be found [here](https://github.com/brycx/sHKDF).


### References
- Ref 1: "Backdoored Hash Functions: Immunizing HMAC and HKDF",  Marc Fischlin and Christian Janson and Sogol Mazaheri, [online](https://eprint.iacr.org/2018/362.pdf), [Accessed] 26th August, 2018.
- Ref 2: "Strengthening Digital Signatures via Randomized Hashing", Shai Halevi and Hugo Krawczyk, [online](https://www.iacr.org/archive/crypto2006/41170039/41170039.pdf), [Accessed] 26th August 2018.
