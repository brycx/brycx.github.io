---
layout: post
title:  "'orion' - yet another attempt at pure-Rust cryptography"
date:   2018-09-25
---

### What is it?

`orion` is another attempt at cryptography implemented in pure Rust. Its main focus
is usability. This is in part achieved by providing [thorough documentation](https://docs.rs/orion) of the library.
High-level abstractions are also provided, which are an attempt at guiding users towards safe usage
of the lower-level functionality of the library. Additionally, types used throughout the library, especially in the high-level interfaces,
are designed to increase misuse-resistance.

`orion` itself forbids the use of so-called "unsafe" code, meaning that all
memory-safety guarantees provided by Rust are enforced at compile-time
(see [rust-lang docs](https://doc.rust-lang.org/book/second-edition/ch19-01-unsafe-rust.html)).
Using no unsafe code also has its drawbacks, such as if you need near-complete
constant-time execution. This will in most cases require the use of assembly
and therefore unsafe Rust code too.

Even though `orion` forbids unsafe code, some of its dependencies do not.
For example, the [subtle crate](https://crates.io/crates/subtle) provides
constant-time comparisons, for which it uses inline assembly.

### Who is it for?

`orion` is not an attempt to replace existing libraries. Libraries such as
[*ring*](https://github.com/briansmith/ring) and those under the
[RustCrypto](https://github.com/RustCrypto) organization all have their own advantages.
`orion` is simply meant as a possible alternative that people can try out. They can then use it
 if they find it fitting for their specific use-case and if it meets their security requirements.

If you're a developer (regardless of prior experience) and looking to work on cryptography-related projects or just
want to work on something in Rust, `orion` could very well be for you. See the note at the bottom about contributing.

I would NOT recommend `orion` to someone looking for a production-ready library
and I have tried to make this as clear as possible in the [README of the repository](https://github.com/brycx/orion#security).

### Where is it?
The project is [hosted on GitHub](https://github.com/brycx) and the
crate is [published on crates.io](https://crates.io/crates/orion).

More detailed information about the library is available in the [project wiki](https://github.com/brycx/orion/wiki).

### Roadmap
`orion` is still under development, and as such the API is still changing. To get `orion` to a more "mature" state, it also needs to be used more by other projects.

Before a stable version of `orion` is released, an audit will be done.
This audit may not cover all of `orion`, depending on my financial situation.
If it does not, the scope of the audit will be determined at the given point
in time. A release date for a stable version has not been set.


### Pros:
- `orion`'s lower-level functionality is available for use on embedded devices.
More specifically, the high-level API is not supported in a `no_std` context.
- Hopefully, `orion` is easy to use.
- Ensures Rust's memory-safety guarantees by forbidding unsafe code. This does however not apply to `orion`s dependencies.
- Sometimes, easier dependency management compared to approaches like RustCrypto.
- Actively maintained compared to [rust-crypto](https://github.com/DaGenix/rust-crypto).
- Because `orion` is still new and under development, it is more flexible in terms of potential contributors having a greater influence on where the project should be heading.

### Cons:
- `orion` has been developed by someone who has no professional background in
cryptography and who was inexperienced with implementing
cryptographic algorithms when he started working on `orion`.
- `orion` has not been audited (a quick audit was done
[back in April](https://github.com/brycx/orion/issues/3), but the library has
changed so much since then, it effectively renders the audit useless).
- `orion` still lacks implementations of many popular/common cryptographic algorithms.
- `orion` is not decoupled like `RustCrypto`. This means you have the complete
library as a dependency, even if you only want to use some of its functionality.
- Due to forbidding unsafe code and therefor also assembly,
"constant-time" code, that `orion` itself implements, cannot at all be guaranteed to be constant-time. However, currently all
  constant-time operations are provided by external libraries that are using inline assembly for this.
- `orion` is not as "mature" as other alternatives available.

### Contributing
Contributions of any kind are most welcome! You can report issues and
suggest new features or improvements through the [public repository](https://github.com/brycx/orion).
