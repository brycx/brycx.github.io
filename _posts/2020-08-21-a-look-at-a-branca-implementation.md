---
layout: post
title: "Rolling your own crypto gone wrong: A look at a .NET Branca implementation"
date: 2020-08-22
---

### Introduction

Some time back, I was looking at token authentication formats to authenticate some API calls. I didn't even attempt to look at JWT & Co. for [multiple reasons](https://paragonie.com/blog/2017/03/jwt-json-web-tokens-is-bad-standard-that-everyone-should-avoid). I landed between [PASETO](https://paseto.io/) and [Branca](https://branca.io/).


I chose Branca for its simplicity. I needed authenticated API calls with a shared symmetric key. Both Branca and PASETO implemented this using XChaCha20-Poly1305, but PASETO also supports asymmetric authentication, which I didn't need. I was quite pleased by looking at how straight-forward Branca [made it](https://github.com/tuupola/branca-spec#token-format):
```
Version (1B) || Timestamp (4B) || Nonce (24B) || Ciphertext (*B) || Tag (16B)
```

Simply construct a header and encrypt and authenticate the payload using XChaCha20-Poly1305, with the header as the AAD.

Back then, there was only one [.NET implementation](https://github.com/thangchung/branca-dotnet), which targeted .NET Core whereas I needed .NET Framework. I took a quick look: there were no test vectors and used ChaCha20-Poly1305 instead of XChaCha20-Poly1305. It was only available GitHub, so I thought it may just be a toy project for fun. I dropped it and forgot about it. 

Fast forward a couple of days ago, I returned to find [a new](https://github.com/scottbrady91/IdentityModel) .NET Core implementation. It was also published as a NuGet, which got my hopes up - might be a polished implementation that I could get working on .NET Framework.

[ScottBrady.IdentityModel](https://www.nuget.org/packages/ScottBrady.IdentityModel/) is a relatively new NuGet, with three releases in total. Its first release was at the beginning of May this year and the latest was at the beginning of this August. It uses [BouncyCastle](https://www.bouncycastle.org/csharp/index.html) for cryptographic implementations and offers both PASETO and Branca.

Note: All code discussed is based on the master branch at [4ff8a06](https://github.com/scottbrady91/IdentityModel/commit/4ff8a06719bd83a4129f45d2ce92f1891a51bd01). I'll also be referring to this NuGet as just IdentityModel throughout the rest of this post.


### Inspection

#### Tokens.SecurityTokenException: Invalid message authentication code

I pulled down IdentityModel in a new project and took some [test vectors](https://github.com/tuupola/branca-js/blob/master/test.js) from the JS reference implementation, which is linked in the specification for Branca.

```c#
static void TestBranca(string expectedToken, string expectedPayload) 
{
    var handler = new BrancaTokenHandler();
    var key = Encoding.UTF8.GetBytes("supersecretkeyyoushouldnotcommit");

    try
    {
        var actualToken = handler.CreateToken(expectedPayload, key);
        var actualPayload = handler.DecryptToken(expectedToken, key);
    }
    catch (Exception e)
    {
        Console.WriteLine("FAILED: \nexpectedToken: {0}\nexpectedPayload: {1}\nexception: {2}\n", expectedToken, expectedPayload, e.Message);
    }
}

static void Main(string[] args)
{
    TestBranca("870S4BYjk7NvyViEjUNsTEmGXbARAX9PamXZg0b3JyeIdGyZkFJhNsOQW6m0K9KnXt3ZUBqDB6hF4", "Hello world!");
    TestBranca("89i7YCwtsSiYfXvOKlgkCyElnGCOEYG7zLCjUp4MuDIZGbkKJgt79Sts9RdW2Yo4imonXsILmqtNb", "Hello world!");
    TestBranca("875GH234UdXU6PkYq8g7tIM80XapDQOH72bU48YJ7SK1iHiLkrqT8Mly7P59TebOxCyQeqpMJ0a7a", "Hello world!");
}
```

Running the above tests gave me:

```
FAILED: 
expectedToken: 870S4BYjk7NvyViEjUNsTEmGXbARAX9PamXZg0b3JyeIdGyZkFJhNsOQW6m0K9KnXt3ZUBqDB6hF4
expectedPayload: Hello world!
exception: Invalid message authentication code

FAILED: 
expectedToken: 89i7YCwtsSiYfXvOKlgkCyElnGCOEYG7zLCjUp4MuDIZGbkKJgt79Sts9RdW2Yo4imonXsILmqtNb
expectedPayload: Hello world!
exception: Invalid message authentication code

FAILED: 
expectedToken: 875GH234UdXU6PkYq8g7tIM80XapDQOH72bU48YJ7SK1iHiLkrqT8Mly7P59TebOxCyQeqpMJ0a7a
expectedPayload: Hello world!
exception: Invalid message authentication code
```

I was already off to a good start.

#### Nonce generation
Starting at the top of the file containing the Branca implementation, comes `CreateToken()`. The first thing is nonce generation:
```c#
var nonce = new byte[24];
RandomNumberGenerator.Create().GetBytes(nonce);
```

It uses the [System.Security.Cryptography.RandomNumberGenerator.GetBytes](https://docs.microsoft.com/en-us/dotnet/api/system.security.cryptography.randomnumbergenerator.getbytes?view=netcore-3.1) method, which is intended for cryptographic purposes, so that checks out. 

#### Unauthenticated ciphertext

After the nonce is generated, the header is created according to the specification. No problem there. Then comes the encryption:

```c#
keyMaterial = new KeyParameter(key);
var parameters = new ParametersWithIV(keyMaterial, nonce);

var engine = new XChaChaEngine();
engine.Init(true, parameters);
```

I'm not familiar with BouncyCastle, so I checked its source to see what `KeyParameter` and `ParametersWithIV` were doing. They were simply wrappers for the parameters.

`XChaChaEngine()` was not from BouncyCastle however, but implemented in IdentityModel:

```c#
using Org.BouncyCastle.Crypto.Engines;

namespace ScottBrady.IdentityModel.Crypto
{
    public class XChaChaEngine : ChaChaEngine
    {
        public XChaChaEngine() : base(20) { }

        public override string AlgorithmName => "XChaCha20";

        protected override int NonceSize => 24;
    }
}
```

After initializing the `XChaChaEngine`, the payload is "encrypted and authenticated": 

```c#
var plaintextBytes = Encoding.UTF8.GetBytes(payload);
var ciphertext = new byte[plaintextBytes.Length + 16];

engine.ProcessBytes(plaintextBytes, 0, plaintextBytes.Length, ciphertext, 0);

var poly = new Poly1305();
poly.Init(keyMaterial);
poly.BlockUpdate(header, 0, header.Length);
poly.DoFinal(ciphertext, plaintextBytes.Length);
```

This is __not a XChaCha20-Poly1305 construction__. There is no padding of the AAD nor the ciphertext during authentication. Neither is there any authentication of their length. All this is specified in the [draft RFC](https://github.com/bikeshedders/xchacha-rfc) and the [RFC for ChaCha20-Poly1305](https://tools.ietf.org/html/rfc8439). Actually, this does not even authenticate the ciphertext since `DoFinal()` writes the current tag into `ciphertext`. The __ciphertext can be modified without invalidating the token__.

```c#
var handler = new BrancaTokenHandler();
var key = Encoding.UTF8.GetBytes("supersecretkeyyoushouldnotcommit");
var actualToken = handler.CreateToken("Test", key);
var decoded = Base62.Decode(actualToken);
decoded[decoded.Length - 17] ^= 1; // Last byte before the Poly1305 tag
Console.WriteLine("{0}", handler.DecryptToken(Base62.Encode(decoded), key).Payload);
```

Running this will return `Tesu` instead of `Test`. Thereby, __IdentityModel allows attackers to arbitrarily modify the payload of a Branca token__. 

After searching BouncyCastle, I found no XChaCha20 implementation but a ChaCha20-Poly1305, which had the following fields:

```c#
private readonly ChaCha7539Engine mChacha20;
private readonly IMac mPoly1305;
```

As you might have noticed, `ChaCha7539Engine` is not the same engine that is implemented by `XChaChaEngine` in IdentityModel. Turns out, IdentityModel uses the ChaCha20 variant with a 64-bit nonce, instead of the 96-bit nonce required by the IETF version of ChaCha20. Both ChaCha20-Poly1305 and XChaCha20-Poly1305 require the IETF variant of ChaCha20. Taking a look at `ChaChaEngine` from BouncyCastle, there is no HChaCha20 being used to calculate a subkey, if the nonce length is set to 24 as in `XChaChaEngine`. Therefore, all we're left with is the original ChaCha20 from DJB, using an 8-byte nonce, meaning `engine.Init(true, parameters)` only loads 8 bytes of the 24-byte nonce that has been generated.

The Branca specification makes it [pretty clear](https://github.com/tuupola/branca-spec) how to encrypt the payload:
> 4. Encrypt the user given payload with IETF XChaCha20-Poly1305 AEAD with user-provided secret key. Use the header as the additional data for AEAD.

It doesn't have to be made as complicated as the above code from IdentityModel. If one reads the draft RFC, or looks at another implementation, it eventually becomes clear that XChaCha20-Poly1305 is "just" a combination of HChaCha20 and ChaCha20-Poly1305.

#### Forgeable tokens
Let's return to the attempt of authenticating the header and ciphertext. Specifically, this line:
```c#
poly.Init(keyMaterial);
```

`keyMaterial` is __the same key__ that was used to initialize the `XChaChaEngine`.

> The sender must not use crypto_onetimeauth to authenticate more than one message under the same key. Authenticators for two messages under the same key should be expected to reveal enough information to allow forgeries of authenticators on other messages. 

(From [NaCl](https://nacl.cr.yp.to/onetimeauth.html))

As NaCls documentation states, any given key used with Poly1305 may __only be used once__ otherwise, an attacker could forge future authenticators. This is a problem since Branca might be used in contexts like authenticating API calls, where long-lived API keys are used. __IdentityModel allows attackers to forge API tokens__.

This would not be a problem in IdentityModel, had it at least used ChaCha20-Poly1305 from BouncyCastle to attempt the Branca implementation. ChaCha20-Poly1305 uses the first 32 bytes of the first keystream-block (64 bytes), of the internal ChaCha20 state, as the Poly1305 one-time key. So if a nonce is unique for every time ChaCha20-Poly1305 is used with any given key (which it __MUST__), the Poly1305 key will also be unique.

Of course, IdentityModel should use XChaCha20-Poly1305, not only because that is what the Branca specification defines, but also because it's not safe to randomly generate nonces for ChaCha20 or ChaCha20-Poly1305 (see [/x/crypto](https://godoc.org/golang.org/x/crypto/chacha20poly1305)). This limitation was the motivation behind XChaCha20-Poly1305 (see [draft RFC](https://github.com/bikeshedders/xchacha-rfc/blob/master/draft-irtf-cfrg-xchacha-rfc-03.txt)).


#### Constant-time MAC comparison
Any decent ChaCha20-Poly1305 or XChaCha20-Poly1305 implementation will compare the Poly1305 MACs in constant-time, to not reveal information via a timing side-channel. This, unfortunately, is not the case for IdentityModel either:

```c#
var headerMac = new byte[16];
[..]
if (!headerMac.SequenceEqual(tag)) throw new SecurityTokenException("Invalid message authentication code");
```

BouncyCastle uses constant-time comparison with ChaCha20-Poly1305 and provides the comparison function as a utility:

```c#
if (!Arrays.ConstantTimeAreEqual(MacSize, mMac, 0, mBuf, resultLen))
{
    throw new InvalidCipherTextException("mac check in ChaCha20Poly1305 failed");
}
```

### Summary

I'm a big fan of "rolling your own crypto" and here I'm talking about implementing known algorithms. I do it myself. I even think making it available on GitHub or similar, to ask for feedback, is good (if users are warned that no security can be expected).

However, a problem arises when projects that don't even uphold the bare minimum of testing test vectors, are published with no warnings at all. Had there been used test vectors in this case, it wouldn't have left IdentityModel completely broken.

#### Found a mistake?
Please [reach out](https://brycx.github.io/contact/).