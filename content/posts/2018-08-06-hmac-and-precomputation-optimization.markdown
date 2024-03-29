---
layout: post
title:  "HMAC optimization - slight performance improvements through precomputation"
date:   2018-08-06
url: 2018/08/06/hmac-and-precomputation-optimization.html
---

### Background

I've been working on one [project](https://github.com/brycx/orion) called `orion`, where I've
implemented the HMAC algorithm. That project is written in Rust and depends on
the `std` library, which means that I'm unable to run that code on embedded devices.

After a couple of times where I had been reviewing my own code, I started to notice some
patterns in the HMAC implementation that seemed to be repetitive. So I set out
to implement HMAC again, without the repetitive patterns and something that could be
run on embedded devices. [This implementation](https://github.com/brycx/rigel) is called `rigel` and is based on HMAC-SHA512, but the
tweaks can be applied to HMAC with any SHA variant.

I set the following goals for my project:
1. Keep as much data as possible in a single array to avoid unnecessary allocations
2. Achieve better, or at least the same performance as the current alternatives available
(The only alternatives I found that supported embedded devices are [RustCrypto](https://github.com/RustCrypto/MACs) and
[*ring*](https://github.com/briansmith/ring). I consider `rust-crypto` abandoned.)
3. Have as few dependencies as possible

### The improved implementation

HMAC is defined as follows ([RFC 2104](https://tools.ietf.org/html/rfc2104)):

```
H(K XOR opad, H(K XOR ipad, text))
```

where:
- `ipad = 0x36`
- `opad = 0x5C`


We know that the blocksize for the hash function `H` (SHA512 in this case) is 128 and the output size is 64. `K` is padded against a length of blocksize, so `K XOR opad` would result in an array with length 128. Since the output size of the hash function `H` is 64 `H(K XOR ipad, text)` would result in an array with length 64. So an array of length 192 should be able to hold everything.

As the RFC mentions, `K` should be padded with zeroes if the length of `K` is less than the blocksize. The padding of zeroes can be skipped:
```rust
fn pad_key_to_ipad(key: &[u8]) -> [u8; 192] {

    let mut padded_key = [0x36; 192];

    if key.len() > 128 {
        padded_key[..64].copy_from_slice(&sha2::Sha512::digest(&key));

        for itm in padded_key.iter_mut().take(64) {
            *itm ^= 0x36;
        }
    else {
        for idx in 0..key.len() {
            padded_key[idx] ^= key[idx];
        }
    }
    padded_key
}
```
You can start out with an array of `0x36 * 192`. This means that if the length of `K` is less than the blocksize, you can simply iterate through the key and XOR each value with the corresponding index value of the `padded_key` array. This yields the result of `K XOR ipad`. If the length of `K` is greater than the blocksize, simply hash `K` using `H`, copy the result into the first 64 bytes of the array and XOR this with the `ipad`.

The padding of zeroes and XORing this with the `ipad` has been precomputed, because naturally `0x00 XOR 0x36 = 0x36`.

Next we need to append the message before hashing, so the following is placed in the main function, using the function `pad_key_to_ipad()` from the key-padding steps before:

```rust
    let mut buffer: [u8; 192] = pad_key_to_ipad(key);
    // First 128 bytes is the ipad
    hash_ipad.input(&buffer[..128]);
    hash_ipad.input(message);
    buffer[128..].copy_from_slice(&hash_ipad.result());
```

So the last 64 bytes of the array `buffer` are now the result of `H(K XOR ipad, text)`. The last step would be to compute the result of `K XOR opad`. Since the XOR function satisfies the associativity property, this can be done rather easily.

The definition of associative property from [Wikipedia](https://en.wikipedia.org/wiki/Associative_property) is as follows:
> Formally, a binary operation ∗ on a set S is called associative if it satisfies the associative law:
> (x ∗ y) ∗ z = x ∗ (y ∗ z) for all x, y, z in S.

Remember that the first 128 bytes of the `buffer` array are the result of `K XOR ipad`. Let's call these 128 bytes `ires`. To retrieve the key `K` from `ires`, we could inverse: `ires XOR ipad` and then compute the `opad` by `K XOR opad`. These two operations are needless however, since it can be done in one operation: `ires XOR 0x6A`, where `0x6A = (ipad XOR opad)`. This is possible because of the associative property: `(ires XOR ipad) XOR opad = ires XOR (ipad XOR opad)`:

```rust
    // Make first 128 bytes the opad
    for idx in buffer.iter_mut().take(128) {
        *idx ^= 0x6A;
    }
```
Now we can simply hash the content of the `buffer` array to compute the MAC.

#### Streaming API - Update [09-08-2018]
I got a very good comment about using this approach along with a streaming API. I have updated the repository to include such an implementation and updated the benchmarks accordingly.

#### Dropping another 64 bytes of allocation with no performance loss - Update [10-08-2018]
I noticed that the extra 64 bytes of allocation for the `ires` was excessive. Previously the `ires` had been stored in the `buffer` array. Instead of this, the SHA512 hashing struct responsible for retrieving the MAC can be updated with the `opad` and `ires` sequentially. Because of this, you only need an array of `128` bytes to store the necessary data. Before these last 64 bytes had been dropped, the benchmarks for `rigel` were:
```rust
test rigel_one_shot ... bench:       2,094 ns/iter (+/- 182)
test rigel_stream   ... bench:       2,174 ns/iter (+/- 121)
```
Now they are:
```rust
test rigel_one_shot ... bench:       2,093 ns/iter (+/- 42)
test rigel_stream   ... bench:       2,161 ns/iter (+/- 58)
```
The repository and performance benchmarks have been updated.  

### Performance - Update [31-01-2019]
Some time has passed since this blogpost was written. Since then, I've [integrated](https://github.com/RustCrypto/MACs/commit/6ad7bb09b221a093260352e5b6f6f95b179e9b1a) some of the key setup ideas into RustCrypto, which made it into the `0.6.3` release of the [hmac crate](https://crates.io/crates/hmac).

[*ring*](https://github.com/briansmith/ring) has also [updated](https://github.com/briansmith/ring/commit/a051eb6e3283cd941d48ae5358ac4d8fd35d3e89) its HMAC key setup process based on some of the ideas from this [blogpost](https://github.com/brycx/rigel/pull/27). These changes made it into the `0.14.4` release of *ring*.

My other project [orion](https://github.com/brycx/orion) has also grown quite a bit, so that is also included in the new benchmarks for comparison.

The old benchmarks (from 09-08-2018) were as follows:

```rust
test RustCrypto     ... bench:       2,727 ns/iter (+/- 91)
test rigel_one_shot ... bench:       2,093 ns/iter (+/- 42)
test rigel_stream   ... bench:       2,161 ns/iter (+/- 58)
test ring           ... bench:       3,357 ns/iter (+/- 96)
```
> This was benchmarked on a MacBook Air 1,6 GHz Intel Core i5, 4GB.

The new benchmarks (from 31-01-2019) are as follows:
```rust
test RustCrypto     ... bench:       2,168 ns/iter (+/- 141)
test orion          ... bench:       2,207 ns/iter (+/- 52)
test rigel_one_shot ... bench:       2,077 ns/iter (+/- 53)
test rigel_stream   ... bench:       2,127 ns/iter (+/- 36)
test ring           ... bench:       1,463 ns/iter (+/- 37)
```
> This was benchmarked on a MacBook Air 1,6 GHz Intel Core i5, 4GB.

*ring* seems to have gained a big performance improvement with its key setup process, adopting some of the ideas in this blogpost. However, this isn't to say that adopting them has resulted in the above performance improvements alone. Much could have changed since the last benchmarks.

There are some things that should be noted about these benchmarks and the current implementation of rigel. The first is, they are focused on benchmarking the key setup steps of HMAC. It does process data in the benches (once), but it doesn't cover use cases such as re-using keys for different input. RustCrypto, *ring*, and orion will outperform rigel on that matter.

The second thing is, even though "embedded" has been mentioned once or twice in this post, rigel has never been tested/benchmarked on actual embedded devices. The only meaning behind me mentioning embedded was focusing on not using needless allocations and supporting `no_std`.

### More

All code for this implementation can be found [here](https://github.com/brycx/rigel).
