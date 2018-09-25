---
layout: post
title:  "'orion' - yet another attempt at pure-Rust cryptography"
date:   2018-09-25
---

### What is it?

`orion` is another attempt at cryptography implemented in pure Rust. Its main focus
is usability. This is in part achieved by providing a [thorough documentation](https://docs.rs/orion/0.7.1/orion/) of the library.
[High-level](https://docs.rs/orion/0.7.1/orion/default/index.html) abstractions
are also provided, which are an attempt at guiding the users towards safe usage
of the lower-level functionality of the library.

`orion` forbids the use of so-called "unsafe" code, meaning that all
memory-safety guarantees provided by Rust are enforced at compile-time
(see [rust-lang docs](https://doc.rust-lang.org/book/second-edition/ch19-01-unsafe-rust.html)).
Using no unsafe code also has its drawbacks, such as if you need near-complete
constant-time execution. This will in most cases require the use of assembly
and therefor unsafe Rust code too.

### Who is it for?

`orion` is not an attempt to replace existing libraries. Libraries such as
[*ring*](https://github.com/briansmith/ring) and those under the
[RustCrypto](https://github.com/RustCrypto) organization all have their own advantages.
`orion` is simply meant as a possible alternative that people can try out. They can then use it
 if they find it fitting for their specific use-case and if it meets their security requirements.

If you're a developer (regardless of prior experience) and looking to work on cryptography-related projects or just
want to work on something in Rust, `orion` could very well be for you. See the note at the bottom about contributing.

I would NOT recommend `orion` to someone looking for a production-ready library
and I have tried to make this as clear as possible in the [README of the repository](https://github.com/brycx/orion#warning).

### Where is it?
The project is [hosted on GitHub](https://github.com/brycx) and the
crate is [published on crates.io](https://crates.io/crates/orion).

### Road map

One of the things on the road map for `orion` includes adding the Poly1305 algorithm,
which will also be used in the high-level API to provide an AEAD construct (IETF ChaCha20_Poly1305)
for encrypting/decrypting data and by that also increasing usability greatly.

Before a stable version  of `orion` is released, an audit will be done.
This audit may not cover all of `orion`, depending on my financial situation.
If it does not, the scope of the audit will be determined at the given point
in time. A release date for a stable version has not been set.


### Pros:
- `orion`'s lower-level functionality is available for use on embedded devices.
More specifically, the high-level API is not supported in a `no_std` context.
- Hopefully, `orion` is easy to use.
- Ensures Rust's memory-safety guarantees by forbidding unsafe code.
- Sometimes, easier dependency management compared to approaches like RustCrypto.
- Actively maintained compared to [rust-crypto](https://github.com/DaGenix/rust-crypto).

### Cons:
- `orion` has been developed by someone who has no professional background in
cryptography and who was inexperienced with implementing
cryptographic algorithms when he started working on `orion`.
- `orion` has not been audited (a quick audit was done
[back in April](https://github.com/brycx/orion/issues/3), but the library has
changed so much since then, it effectively renders the audit useless).
- `orion` still lacks implementations of many popular/common cryptographic algorithms.
- `orion` is not decoupled like `RustCrypto`. This means you have the complete
library as dependency, even if you only want to use some of its functionality.
- Due to forbidding unsafe code and therefor also assembly,
"constant-time" code cannot at all be guaranteed to be constant-time.
- `orion` is not as "mature" as other alternatives available.

### Contributing
Contributions of any kind are most welcome! You can report issues and
suggest new features or improvements through the [public repository](https://github.com/brycx/orion).
