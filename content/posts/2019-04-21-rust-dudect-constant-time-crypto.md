---
layout: post
title:  "Rust, dudect and constant-time crypto in debug mode"
date:   2019-04-21
---
### Introduction

The following are observations from when I started testing my own [pure-Rust crypto library](https://github.com/brycx/orion), including its dependencies, for constant-time execution. Starting with a short introduction to [dudect](https://eprint.iacr.org/2016/1123.pdf) and how it can be used to test code for timing-based side-channel vulnerabilities. Then discussing the process of [discovering a short-circuit](https://github.com/dalek-cryptography/subtle/issues/43) that resulted in variable-time execution, in [dalek-cryptography's subtle library](https://github.com/dalek-cryptography/subtle) and how this seems to relate to Rust codegen option `opt-level`.

__DISCLAIMER__: The tests and analysis presented in this post were not done by a professional - take everything with a grain of salt. 

### dudect

Excerpts from the [original papers abstract](https://eprint.iacr.org/2016/1123.pdf): 

> "[...] a tool to assess whether a piece of code runs in constant time or not on a given platform. We base our approach on leakage detection techniques, resulting  in  a  very  compact,  easy  to  use  and  easy  to  maintain tool."

> "Contrary to others, our solution requires no modeling of hardware behavior.  Our solution can be used in black-box testing, yet benefits from implementation details if
available. We show the effectiveness of our approach by detecting several  variable-time  cryptographic  implementations." 

dudect uses Welch's t-tests to detect variances in execution time, which is used to detect leakage in code that is supposed to be constant-time.

#### Welch's t-tests

> "In statistics, Welch's t-test, or unequal variances t-test, is a two-sample location test which is used to test the hypothesis that two populations have equal means."

([Wiki source](https://en.wikipedia.org/wiki/Welch%27s_t-test))

In terms of dudect, Welch's t-test is used to test the hypothesis that two input-classes produce two sets of timing measurements, that have equal means. 

#### Threshold

The threshold is the maximum t-value that is considered acceptable. The paper [suggests](https://eprint.iacr.org/2016/1123.pdf) that: 
> "A t value larger than 4.5 provides strong statistical evidence that the distributions are different"

When calculating the t-value, two means are subtracted: `M1 - M2`. Measurements can report negative t-values, this is however of little concern to us since this will just indicate that `M1 < M2`. Therefore, t-values should be within the acceptable threshold when `t > -4.5` or `t < 4.5` (see also this [question](https://www.researchgate.net/post/Why_have_I_got_a_negative_T_value)).

__Note:__ Throughout the rest of this post I'll also use the term "`max t`", which will refer to the same statistical t-value discussed here.

#### Number of samples

Based on what is tested, it should be assessed how many samples are to be generated. In some cases, a larger number of samples are required to detect any timing difference, as the author of the paper describes [here](https://github.com/oreparaz/dudect/blob/master/README.md):

> "Once we have around 20k measurements, the timing leak starts to be detectable, and we can conclude that the implementation is not constant time"

#### Inputs for measuring

When testing functions such as constant-time comparisons, it's often sufficient to use random input-pairs. Though, in some cases when looking for timing difference it might yield better results, or even be required to detect leakage, to use specially crafted test vectors.

Implementations such as Poly1305 will only be variable-time for some inputs. Poly1305 implementations that use big-num libraries are most likely not constant-time, where variable-time execution could arise from the internal operations (multiplication, modulo, etc):

> "For example, if the multiplication is performed as a separate operation from the modulus, the result will sometimes be under 2^256 and sometimes be above 2^256."

[RFC 8439](https://tools.ietf.org/html/rfc8439#section-4)

Creating tailored test vectors would involve identifying on which inputs such behavior occurs. Perhaps a good place to start would be using [Google Wycheproof test vectors for ChaCha20Poly1305](https://github.com/google/wycheproof/blob/master/testvectors/chacha20_poly1305_test.json) with dudect, which include some edge-cases for Poly1305.

This [paper](https://csrc.nist.gov/csrc/media/events/non-invasive-attack-testing-workshop/documents/09_jaffe.pdf) describes some specially crafted test vectors for leakage detection in RSA implementations, using Welch's t-tests.

The length of the input should also be considered. Testing a constant-time comparison function using arrays with a single element won't yield any results at all. However, measuring, for example, two arrays of length 128 where the first element differs, would have a much better chance of revealing differences in timing.

### Testing orion

All the tests can be examined in the [repository orion-dudect](https://github.com/brycx/orion-dudect). To run dudect tests, I use the wonderful [dudect-bencher](https://github.com/rozbb/dudect-bencher/) crate.

The parts of orion that are currently tested for constant-time execution, are as follows:
- The constant-time comparison function `secure_cmp`, that is provided as a wrapper around subtle's `ct_eq`.
- The macro that implements `PartialEq`, in constant-time, for all newtypes.
- The Poly1305 implementation.

Testing specifics:
- The number of samples generated (ie. number of input-pairs) was 1e6.
- dudect-bencher input-pairs were either both random or one random and one all 0's.
- A threshold of 4.5.
- Testing was done with Rust `1.34` and latest nightly.
- Crate versions: orion `0.13.4`, subtle `2.0.0`, dudect-bencher `0.3.0` and orion-dudect at commit [9dcb019](https://github.com/brycx/orion-dudect/commit/9dcb0193441ae3fb91571b978ea9ce98bb5ad29d). 

### Short-circuit in dalek-cryptography's subtle library

`secure_cmp` was the first interface I tested and the test-runner for that is shown below, where `u` and `v` are a single input-pair.

```
// Run timing
for (class, (u, v)) in classes.into_iter().zip(inputs.into_iter()) {
        runner.run_one(class, || secure_cmp(&u[..], &v[..]));
}
```

The first results I got were:
```
running 1 bench
bench test_secure_cmp seeded with [0xe3fcea9a, 0x2a24a9d8, 0x336618e6, 0x625fc346]
bench test_secure_cmp ... : n == +0.690M, max t = +704.74089, max tau = +0.84869, (5/tau)^2 = 34
```

I spent two days inspecting the code in both subtle, dudect-bencher and orion, but couldn't find anything amiss. The fourth day, I realized __I had only been testing in debug-mode__. Running the tests in release gave:
```
running 1 bench
bench test_secure_cmp seeded with [0x760ce3a5, 0x97c6bd44, 0x6baee54f, 0xcbd63a51]
bench test_secure_cmp ... : n == +0.228M, max t = -1.32524, max tau = -0.00277, (5/tau)^2 = 3246511
```
After having a look through all the code again, I noticed `debug_assert!` statements in subtle (a `debug_assert!` call is not included in release builds), which used the `||` short-circuit operator:

```
debug_assert!(input == 0u8 || input == 1u8);
debug_assert!(source.0 == 0u8 || source.0 == 1u8);
```

After changing these to strict evaluations (`|`), the `max t` values reported decreased to about 20.
```
debug_assert!((input == 0u8) | (input == 1u8));
debug_assert!((source.0 == 0u8) | (source.0 == 1u8));
```

This was still outside of the threshold, but to make sure using strict evaluations actually was the solution, I extracted the `secure_cmp` test, into a separate one that tested subtle's `ct_eq`. This resulted in `max t` values of ~2, within the threshold.

#### `debug_assert!` and `assert!`

I had assumed that since `debug_assert!` is left out in release builds, using the short-circuit evaluation in an `assert!` statement (an `assert!` call is included in release builds), the same timing results would be reported in debug and release mode. 

I ran tests, with all the short-circuiting `debug_assert!`s being changed to `assert!`s, assuming that the variable-time execution would persist in release builds as well. These were the results that were reported:
```
running 1 bench
bench test_secure_cmp seeded with [0xc1df4b3f, 0x7e07c3db, 0xbe9f442f, 0x7dc6dfd4]
bench test_secure_cmp ... : n == +0.048M, max t = +1.54205, max tau = +0.00702, (5/tau)^2 = 506736
```
My assumption was, that the compiler was using the information about `input` and `source.0` to optimize the code, resulting in variable-time execution. Seeing these results I assume this is not entirely the case, which led me to think that perhaps it is some of the debug information, that is included in debug builds. To tests this, I specified `profile` attributes in orion-dudect:

```
[profile.release]
debug = false
debug-assertions = false

[profile.dev]
debug = false
debug-assertions = false
```
Toggling any of these on and off individually made no difference. One difference between release and debug are the codegen options passed to LLVM. debug uses `opt-level = 0` by default and release uses `opt-level = 3`. Testing release with `opt-level = 3/2/1` made no difference, but with `opt-level = 0`, still using `assert!`, reported:
```
running 1 bench
bench test_secure_cmp seeded with [0x3aaf907c, 0xcc37be69, 0x30af1b90, 0xd613fd43]
bench test_secure_cmp ... : n == +0.556M, max t = +1428.01869, max tau = +1.91513, (5/tau)^2 = 6
``` 
Replacing the `assert!`s with strict evaluations had the same result on timing, as when testing `debug_assert!`s. This had made me spend some time trying to figure out what exact codegen parameters are used with LLVM, when setting `opt-level = 1`, to identify the parameters that made the compiler ignore the short-circuit evaluations. I haven't been able to find information on what parameters are passed and could therefore not proceed with this. The only thing I could take away from this was that __with an `opt-level = 0`, the compiler uses results of short-circuit evaluations in assert-statements__. 

#### `Result` and constant-time evaluations

The `max t` values for `secure_cmp` were still outside of threshold after applying strict evaluations to subtle's `debug_assert!`s.

When toggling the `opt-level`, as with subtle, the same behavior was observed - only `opt-level = 0` in both debug and release mode resulted in `max t` values outside of the threshold. 

`secure_cmp` currently returns a `Result`, like so:
```
if a.ct_eq(b).into() {
    Ok(true)
} else {
    Err(errors::UnknownCryptoError)
}
```

Testing `secure_cmp` with `is_ok()` and `is_err()`, instead of not using the `Result`, gave the following:
```
running 1 bench
bench test_secure_cmp seeded with 0xcf4880125e45af1b
bench test_secure_cmp ... : n == +0.563M, max t = -63.07522, max tau = -0.08409, (5/tau)^2 = 3535
``` 

Again, toggling the `opt-level`s showed the exact same results as before. Changing `secure_cmp` to return only a `bool`, directly from `.into()`, showed the exact same results as when testing subtle's `ct_eq` separately.

Perhaps the compiler can somehow use the information, arising from subtle's `From<> for bool` conversion, when calling `.into()` combined with an `Ok(true)` return? I honestly have no idea as to what the cause of this is. The only thing I could take away from this was that __using `Result` in code that is expected to run in constant-time, using `opt-level = 0`, can result in variable-time execution__.

### Summary

A short summary of my overall takeaway:

- All Rust code should be tested in both debug and release mode.
- Testing Rust code for constant-time execution should be done with all `opt-level` values.
- Rust code that is expected to run in constant-time must __never__ be compiled with `opt-level = 0`.
- `assert!` and `debug_assert!` should be avoided, with `opt-level = 0`, in code that needs to execute in constant time.
- `Result` return-types should be avoided, with `opt-level = 0`, in code that needs to execute in constant time.


As the introduction mentioned: all of the above are merely observations, and not very detailed ones. Furthermore, the tests are based on only one measurement tool. Testing for constant-time execution should preferably combine uses of both dudect, [ctgrind](https://github.com/agl/ctgrind) and other relevant ones. 

There's also a new tool in Rust called [SideFuzz](https://github.com/phayes/sidefuzz), which currently combines differential fuzzing, genetic algorithms and dudect for leakage detection. I'm quite excited to try it out and see how it develops.

#### Found a mistake?
Please [reach out](https://brycx.github.io/contact/).