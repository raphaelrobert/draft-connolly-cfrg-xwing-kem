---
title: "X-Wing: general-purpose hybrid post-quantum KEM"
abbrev: xwing
category: info

docname: draft-connolly-cfrg-xwing-kem-latest
submissiontype: IRTF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date: 2024-01-22
consensus: true
v: 3
area: "IRTF"
workgroup: "Crypto Forum"
keyword:
 - post quantum
 - kem
 - PQ/T hybrid
venue:
  group: "Crypto Forum"
  type: "Research Group"
  mail: "cfrg@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/search/?email_list=cfrg"
  github: "dconnolly/draft-connolly-cfrg-xwing-kem"
  latest: "https://dconnolly.github.io/draft-connolly-cfrg-xwing-kem/draft-connolly-cfrg-xwing-kem.html"

author:
 -
    fullname: Deirdre Connolly
    organization: SandboxAQ
    email: durumcrustulum@gmail.com

 -
    fullname: Peter Schwabe
    organization: MPI-SP & Radboud University
    email: peter@cryptojedi.org

 -
    ins: B.E. Westerbaan
    fullname: Bas Westerbaan
    organization: Cloudflare
    email: bas@cloudflare.com

normative:
  RFC2119:

informative:
  I-D.driscoll-pqt-hybrid-terminology:
  I-D.ounsworth-cfrg-kem-combiners:
  FIPS202:
    target: https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.202.pdf
    title: 'FIPS 202: SHA-3 Standard: Permutation-Based Hash and
Extendable-Output Functions'
    author:
      -
        ins: National Institute of Standards and Technology
  MLKEM:
    target: https://csrc.nist.gov/pubs/fips/203/ipd
    title: 'FIPS 203 (Initial Draft): Module-Lattice-Based Key-Encapsulation Mechanism Standard'
    author:
      -
        ins: National Institute of Standards and Technology
  RFC9180:
  RFC7748:
  HYBRID: I-D.stebila-tls-hybrid-design
  XYBERHPKE: I-D.westerbaan-cfrg-hpke-xyber768d00
  XYBERTLS: I-D.tls-westerbaan-xyber768d00
  TLSIANA: I-D.ietf-tls-rfc8447bis
  PROOF:
    target: https://eprint.iacr.org/2024/039
    title: "X-Wing: The Hybrid KEM You’ve Been Looking For"
    author:
      -
        ins: M. Barbosa
      -
        ins: D. Connolly
      -
        ins: J. Duarte
      -
        ins: A. Kaiser
      -
        ins: P. Schwabe
      -
        ins: K. Varner
      -
        ins: B.E. Westerbraan

--- abstract

This memo defines X-Wing, a general-purpose post-quantum/traditional
hybrid key encapsulation mechanism (PQ/T KEM) built on X25519 and
ML-KEM-768.

--- middle

# Introduction {#intro}

## Warning: ML-KEM-768 has not been standardised

X-Wing uses ML-KEM-768, which has not been standardised yet. Thus X-Wing
is not finished, yet, and should not be used, yet.

## Motivation {#motivation}

There are many choices that can be made when specifying a hybrid KEM:
the constituent KEMs; their security levels; the combiner; and the hash
within, to name but a few. Having too many similar options are a burden
to the ecosystem.

The aim of X-Wing is to provide a concrete, simple choice for
post-quantum hybrid KEM, that should be suitable for the vast majority
of use cases.

## Design goals {#goals}

By making concrete choices, we can simplify and improve many aspects of
X-Wing.

* Simplicity of definition. Because all shared secrets and cipher texts
  are fixed length, we do not need to encode the length. Using SHA3-256,
  we do not need HMAC-based construction. For the concrete choice of
  ML-KEM-768, we do not need to mix in its ciphertext, see {{secc}}.

* Security analysis. Because ML-KEM-768 already assumes the Quantum Random
  Oracle Model (QROM), we do not need to complicate the analysis
  of X-Wing by considering stronger models.

* Performance. Not having to mix in the ML-KEM-768 ciphertext is a nice
  performance benefit. Furthermore, by using SHA3-256 in the combiner,
  which matches the hashing in ML-KEM-768, this hash can be computed in
  one go on platforms where two-way Keccak is available.

We aim for "128 bits" security (NIST PQC level 1). Although at the
moment there is no peer-reviewed evidence that ML-KEM-512 does not reach
this level, we would like to hedge against future cryptanalytic
improvements, and feel ML-KEM-768 provides a comfortable margin.

We aim for X-Wing to be usable for most applications, including
specifically HPKE {{RFC9180}}.

## Not an interactive key-agreement

Traditionally most protocols use a Diffie-Hellman (DH) style
non-interactive key-agreement.  In many cases, a DH key agreement can be
replaced by the interactive key-agreement afforded by a KEM without
change in the protocol flow.  One notable example is TLS {{HYBRID}}
{{XYBERTLS}}.  However, not all uses of DH can be replaced in a
straight-forward manner by a plain KEM.

## Not an authenticated KEM {#auth}

In particular, X-Wing is not, borrowing the language of {{RFC9180}}, an
*authenticated* KEM.

## Comparisons

### With HPKE X25519Kyber768Draft00

X-Wing is most similar to HPKE's X25519Kyber768Draft00
{{XYBERHPKE}}. The key differences are:

* X-Wing uses the final version of ML-KEM-768.

* X-Wing hashes the shared secrets, to be usable outside of HPKE.

* X-Wing has a simpler combiner by flattening DHKEM(X25519) into the
  final hash.

* X-Wing does not hash in the ML-KEM-768 ciphertext.

There is also a different KEM called X25519Kyber768Draft00 {{XYBERTLS}}
which is used in TLS. This one should not be used outside of TLS, as it
assumes the presence of the TLS transcript to ensure non malleability.

### With generic combiner

The generic combiner of {{I-D.ounsworth-cfrg-kem-combiners}} can be
instantiated with ML-KEM-768 and DHKEM(X25519). That achieves similar
security, but:

* X-Wing is more performant, not hashing in the ML-KEM-768 ciphertext,
  and flattening the DHKEM construction, with the same level of
  security.

* X-Wing has a fixed 32 byte shared secret, instead of a variable shared
  secret.

* X-Wing does not accept the optional counter and fixedInfo arguments.

# Requirements Notation

{::boilerplate bcp14-tagged}

# Conventions and Definitions

This document is consistent with all terminology defined in
{{I-D.driscoll-pqt-hybrid-terminology}}.

The following terms are used throughout this document to describe the
operations, roles, and behaviors of HPKE:

- `concat(x0, ..., xN)`: returns the concatenation of byte
  strings. `concat(0x01, 0x0203, 0x040506) = 0x010203040506`.
- `random(n)`: return a pseudorandom byte string of length `n` bytes produced by
  a cryptographically-secure random number generator.

# Cryptographic Dependencies {#base-crypto}

X-Wing relies on the following primitives:


* ML-KEM-768 post-quantum key-encapsulation mechanism (KEM) {{MLKEM}}:

  - `ML-KEM-768.KeyGen()`: Randomized algorithm to generate an
    ML-KEM-768 key pair `(pk_M, sk_M)` of an encapsulation key `pk_M`
    and decapsulation key `sk_M`.
    Note that `ML-KEM-768.KeyGen()` returns the keys in reverse
    order of `GenerateKeyPair()` defined below.
  - `ML-KEM-768.Encaps(pk_M)`: Randomized algorithm to generate `(ss_M,
    ct_M)`, an ephemeral 32 byte shared key `ss_M`, and a fixed-length
    encapsulation (ciphertext) of that key `ct_M` for encapsulation key
    `pk_M`.
  - `ML-KEM-768.Decap(ct_M, sk_M)`: Deterministic algorithm using the
    decapsulation key `sk_M` to recover the shared key from `ct_M`.

  To generate deterministic test vectors, we also use

  - `ML-KEM-768.KeyGenDerand(seed)`: Same as `ML-KEM-768.KeyGen()`,
    but derandomized as follows.
    `seed[0:32]` is used for `d` in line 1 of algorithm 12 from {{MLKEM}}
    and `seed` is 64 bytes. `seed[32:64]` is used for `z` in line 1 of
    algorithm 15.
  - `ML-KEM-768.EncapsDerand(pk_M, seed)`: Same as `ML-KEM-768.Encaps()`
    but derandomized as follows.
    `seed` is 32 bytes and used for `m` in line of 1 algorithm 16.

* X25519 elliptic curve Diffie-Hellman key-exchange defined in {{Section 5 of RFC7748}}:

  - `X25519(k,u)`: takes 32 byte strings k and u representing a
    Curve25519 scalar and curvepoint respectively, and returns
    the 32 byte string representing their scalar multiplication.
  - `X25519_BASE`: the 32 byte string representing the standard base point
    of Curve25519. In hex
    it is given by `0900000000000000000000000000000000000000000000000000000000000000`.

Note that 9 is the standard basepoint for X25519, cf {{Section 6.1 of RFC7748}}.


* Symmetric cryptography.

  - `SHAKE128(message, outlen)`: The extendable-output function (XOF)
    defined in Section 6.2 of {{FIPS202}}.
  - `SHA3-256(message)`: The hash defined in Section 6.1 of {{FIPS202}}.


# X-Wing Construction

## Encoding and sizes {#encoding}

X-Wing encapsulation key, decapsulation key, ciphertexts and shared secrets are all
fixed length byte strings.

 Decapsulation key (private):
 : 2464 bytes

 Encapsulation key (public):
 : 1216 bytes

 Ciphertext:
 : 1120 bytes

 Shared secret:
 : 32 bytes

## Key generation

An X-Wing keypair (decapsulation key, encapsulation key) is generated as
follows.

~~~
def GenerateKeyPair():
  (pk_M, sk_M) = ML-KEM-768.KeyGen()
  sk_X = random(32)
  pk_X = X25519(sk_X, X25519_BASE)
  return concat(sk_M, sk_X, pk_X), concat(pk_M, pk_X)
~~~

`GenerateKeyPair()` returns the 2464 byte secret decapsulation key `sk`
and the 1216 byte encapsulation key `pk`.

Here and in the balance of the document for clarity we use
the `M` and `X`subscripts for ML-KEM-768 and X25519 components respectively.

### Key derivation {#derive-key-pair}

For testing, it is convenient to have a deterministic version
of key generation. An X-Wing implementation MAY provide the following
derandomized variant of key generation.

~~~
def GenerateKeyPairDerand(seed):
  (pk_M, sk_M) = ML-KEM-768.KeyGenDerand(seed[0:64])
  sk_X = seed[64:96]
  pk_X = X25519(sk_X, X25519_BASE)
  return concat(sk_M, sk_X, pk_X), concat(pk_M, pk_X)
~~~

`seed` must be 96 bytes.

`GenerateKeyPairDerand()` returns the 2464 byte secret encapsulation key
`sk` and the 1216 byte decapsulation key `pk`.

## Combiner {#combiner}

Given 32 byte strings `ss_M`, `ss_X`, `ct_X`, `pk_X`, representing the
ML-KEM-768 shared secret, X25519 shared secret, X25519 ciphertext
(ephemeral public key) and X25519 public key respectively, the 32 byte
combined shared secret is given by:

~~~
def Combiner(ss_M, ss_X, ct_X, pk_X):
  return SHA3-256(concat(
    XWingLabel,
    ss_M,
    ss_X,
    ct_X,
    pk_X
  ))
~~~

where XWingLabel is the following 6 byte ASCII string

~~~
XWingLabel = concat(
    "\./",
    "/^\",
)
~~~

In hex XWingLabel is given by `5c2e2f2f5e5c`.

## Encapsulation {#encaps}

Given an X-Wing encapsulation key `pk`, encapsulation proceeds as follows.

~~~
def Encapsulate(pk):
  pk_M = pk[0:1184]
  pk_X = pk[1184:1216]
  ek_X = random(32)
  ct_X = X25519(ek_X, X25519_BASE)
  ss_X = X25519(ek_X, pk_X)
  (ss_M, ct_M) = ML-KEM-768.Encaps(pk_M)
  ss = Combiner(ss_M, ss_X, ct_X, pk_X)
  ct = concat(ct_M, ct_X)
  return (ss, ct)
~~~

`pk` is a 1216 byte X-Wing encapsulation key resulting from `GeneratePublicKey()`

`Encapsulate()` returns the 32 byte shared secret `ss` and the 1120 byte
ciphertext `ct`.

### Derandomized

For testing, it is convenient to have a deterministic version
of encapsulation. An X-Wing implementation MAY provide
the following derandomized function.

~~~
def EncapsulateDerand(pk, eseed):
  pk_M = pk[0:1184]
  pk_X = pk[1184:1216]
  ek_X = eseed[32:64]
  ct_X = X25519(ek_X, X25519_BASE)
  ss_X = X25519(ek_X, pk_X)
  (ss_M, ct_M) = ML-KEM-768.EncapsDerand(pk_M, eseed[0:32])
  ss = Combiner(ss_M, ss_X, ct_X, pk_X)
  ct = concat(ct_M, ct_X)
  return (ss, ct)
~~~

`pk` is a 1216 byte X-Wing encapsulation key resulting from `GeneratePublicKey()`
`eseed` MUST be 64 bytes.

`EncapsulateDerand()` returns the 32 byte shared secret `ss` and the 1120 byte
ciphertext `ct`.


## Decapsulation {#decaps}

~~~
def Decapsulate(ct, sk):
  ct_M = ct[0:1088]
  ct_X = ct[1088:1120]
  sk_M = sk[0:2400]
  sk_X = sk[2400:2432]
  pk_X = sk[2432:2464]
  ss_M = ML-KEM-768.Decapsulate(ct_M, sk_M)
  ss_X = X25519(sk_X, ct_X)
  return Combiner(ss_M, ss_X, ct_X, pk_X)
~~~

`ct` is the 1120 byte ciphertext resulting from `Encapsulate()`
`sk` is a 2464 byte X-Wing decapsulation key resulting from `GenerateKeyPair()`

`Decapsulate()` returns the 32 byte shared secret.

## Use in HPKE

X-Wing satisfies the HPKE KEM interface as follows.

The `SerializePublicKey`, `DeserializePublicKey`,
`SerializePrivateKey` and `DeserializePrivateKey` are the identity functions,
as X-Wing keys are fixed-length byte strings, see {{encoding}}.

`DeriveKeyPair()` is given by

~~~
def DeriveKeyPair(ikm):
  return GenerateKeyPairDerand(SHAKE128(ikm, 96))
~~~

where the HPKE private key and public key are the X-Wing decapsulation
key and encapsulation key respectively.

The argument `ikm` to `DeriveKeyPair()` SHOULD be at least 32 octets in
length.  (This is contrary to {{RFC9180}} which stipulates it should be
at least Nsk=2432 octets in length.)

`Encap()` is `Encapsulate()` from {{encaps}}.

`Decap()` is `Decapsulate()` from {{decaps}}.

X-Wing is not an authenticated KEM: it does not support `AuthEncap()`
and `AuthDecap()`, see {{auth}}.

Nsecret, Nenc, Npk, and Nsk are defined in {{iana}}.

## Use in TLS 1.3

For the client's share, the key_exchange value contains
the X-Wing encapsulation key.

For the server's share, the key_exchange value contains
the X-Wing ciphertext.

# Security Considerations {#secc}

Informally, X-Wing is secure if SHA3 is secure, and either X25519 is
secure, or ML-KEM-768 is secure.

More precisely, if SHA3-256, SHA3-512, SHAKE-128, and SHAKE-256 may be
modelled as a random oracle, then the IND-CCA security of X-Wing is
bounded by the IND-CCA security of ML-KEM-768, and the gap-CDH security
of Curve25519, see {{PROOF}}.

The security of X-Wing relies crucially on the specifics of the
Fujisaki-Okamoto transformation used in ML-KEM-768: the X-Wing
combiner cannot be assumed to be secure, when used with different
KEMs. In particular it is not known to be safe to leave
out the post-quantum ciphertext from the combiner in the general case.

# IANA Considerations {#iana}

This document requests/registers a new entry to the "HPKE KEM Identifiers"
registry.

 Value:
 : TBD (please)

 KEM:
 : X-Wing

 Nsecret:
 : 32

 Nenc:
 : 1120

 Npk:
 : 1216

 Nsk:
 : 2464

 Auth:
 : no

 Reference:
 : This document

Furthermore, this document requests/registers a new entry to the TLS
Named Group (or Supported Group) registry, according to the procedures
in {{Section 6 of TLSIANA}}.

 Value:
 : TBD (please)

 Description:
 : X-Wing

 DTLS-OK:
 : Y

 Recommended:
 : Y

 Reference:
 : This document

 Comment:
 : PQ/T hybrid of X25519 and ML-KEM-768


# TODO

- Which validation do we want to require?


--- back

# Implementations

- Go

  - [CIRCL](https://github.com/cloudflare/circl/pull/471)

  - [Filippo](https://github.com/FiloSottile/mlkem768)

- Rust

  - [xwing-kem.rs](https://github.com/rugo/xwing-kem.rs])

    Note: implements the older `-00` version of this memo at the time of
    writing.


# Machine-readable specification {#S-spec}

For the convenience of implementors, we provide a reference specification
in Python. This is a specification; not production ready code:
it should not be deployed as-is, as it leaks the private key by its runtime.

## xwing.py

~~~~
{::include ./spec/xwing.py}
~~~~

## x25519.py

~~~~
{::include ./spec/x25519.py}
~~~~

## mlkem.py

~~~~
{::include ./spec/mlkem.py}
~~~~

# Test vectors # TODO: replace with test vectors that re-use ML-KEM, X25519 values

~~~~
{::include ./spec/test-vectors.txt}
~~~~

# Acknowledgments

TODO acknowledge.

# Change log

> **RFC Editor's Note:** Please remove this section prior to publication of a
> final version of this document.

## Since draft-connolly-cfrg-xwing-kem-01

- Add list of implementations.

- Miscellaneous editorial improvements.

- Add Python reference specification.

- Correct definition of `ML-KEM-768.KeyGenDerand(seed)`.

## Since draft-connolly-cfrg-xwing-kem-00

- A copy of the X25519 public key is now included in the X-Wing
  decapsulation (private) key, so that decapsulation does not
  require separate access to the X-Wing public key. See #2.
