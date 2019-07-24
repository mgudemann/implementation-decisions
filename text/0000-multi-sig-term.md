- Feature Name: native multi-sig term
- Start Date: 2019-07-23
- RFC PR: (leave this empty)

# Summary
[summary]: #summary


A description of a multi-signature scheme for Shelley, including binary
representation of native multi-signature terms. In order to be language
agnostic, it is based on the standardized Concise Binary Object Representation
(CBOR) format[1].

[1]  https://tools.ietf.org/html/rfc7049

# Motivation
[motivation]: #motivation

The multi-signature terms allow to support multi-signature schemes used for
transactions and delegation certificates in Shelley. They allow for a native
implementation of multi-sig without relying on an external scripting
language. Different multi-signature terms can be used for value addresses and
stake addresses which is important for cases where for some participating
parties regulation might affect delegation.

The resulting multi-sig terms have a compact binary representation and can be
visualized in a easy to understand hierarchical tree-like structure. The
multi-signature terms can be seen as a script-like DSL which is interpreted in
order to validate them, therefore we also use the name multi-sig _script_ in
this document.

# Decision
[Decision]: #decision

A multi-sig term describes which combination of signatures of a transaction is
required to:

 - spend a multi-sig locked transaction output
 - allow the withdrawal of a multi-sig locked rewards account
 - allow changing the status of a multi-sig delegation reference

## Format of Multi-Sig

The recursive structure of a multi-sig term is of the form:

 - `RequireSignature kh` where `kh` is a hashed key
 - `RequireAllOf [msigs]` where `[msigs]` is a list of multi-sig terms
 - `RequireAnyOf [msisg]` where `[msigs]` is a list of multi-sig terms
 - `RequireMOf m [msigs]` where `m` is a natural and `[msigs]` is a list of
   multi-sig terms

This forms a tree-like structure where each leaf is of the form
`RequireSignature kh`. Leafs of the form `RequireAllOf` or `ReuquireMOf 0` with
an empty argument list of multi-sig terms are redundant in the sense that they
evaluate to `True`. Leaf terms of the form `RequireAnyOf` or `RequireMOf m` with
`m>0` with an empty list of multi-sig terms is also redundant and evaluates to
`False`.

### Examples

Assume we have key hashes for different participants: `aliceKH`, `bobKH`,
`carlKH` and `dariaKH`. Then the simplest multi-signature term requires
only Alice's signatures and looks as follows:

```haskell
aliceOnly = RequireSignature aliceKH
```

A multi-signature term that requires Alice's or Bob's signature looks as
follows:

```haskell
aliceOrBob = RequireAnyOf [RequireSignature aliceKH, RequireSignature bobKH]
```

A script requiring both signatures looks as follows:

```haskell
aliceAndBob = RequireAllOf [RequireSignature aliceKH, RequireSignature bobKH]
```

Finally for an 2 out of 3 scheme for Alice, Bob and Carl, the multi-signature
term looks as follows:

```haskell
aliceBobCarl2oo3 = RequireMOf 2 [RequireSignature aliceKH, RequireSignature bobKH,
RequireSignature carlHK]
```

The terms can be combined recursively, e.g., requiring a signature of Bob and
Alice or of Carl and Daria is expressed as follows:

```haskell
aliceAndBobOrCarlAndDaria = RequireAnyOf
    [ RequireAllOf [RequireSignature aliceKH, RequireSignature bobKH]
    , RequireAllOf [RequireSignature carlKH, RequireSignature dariaKH]]
```

`RequireAnyOf [msigs]` and `RequireMOf 1 [msigs]` are logically equivalent, just
as `RequireAllOf [msigs]` and `RequireMOf (length msisg) [msigs]`. From a point
of readability and to a lesser part also for encoding size, it makes sense to
have the special case terms for _any_ and _all_. It also provides a useful
extension point for later additions, e.g., hash locks and time locks.

# Technical explanation
[technical-explanation]: #technical-explanation

Multi-signature terms can be used in value addresses and stake addresses. They
can be used to lock unspent transaction outputs, withdrawal accounts and
delegation certificates.

## Binary Representation

The binary translation uses the standardized CBOR format. Each of the possible
sub-terms is tagged with an identifier, standard list encoding with start and
break tags is used for a list of sub-terms and key hashes are encoded as an
array of data items with the length depending on the specific hash
algorithm. Using `SHA256` this length is `2` bytes for length encoding and `32`
bytes for the hash. In CBOR CDDL:

```
msig =
       [0, keyhash]          ; The case: RequireSignature sig
     / [1, [ *msig ]]        ; The case: RequireAllOf msigs
     / [2, [ *msig ]]        ; The case: RequireAnyOf msigs
     / [3, uint, [ *msig ]]  ; The case: RequireMOf m msigs

keyhash = bytes .size 32
```

For example, a `RequireSignature kh` term for an example key for Alice is
encoded as follows (36 bytes), first in CBOR diagnostic notation and then as raw
bytes:

```haskell
[0, h'C9C65B1126BCF95573B9C5CF99828B01ABF5E492DBF3B0A4D46444E8ABB30E8F']
```

```
82  # list(2)
   00  # int(0)
   58 20 c9 c6 5b 11 26 bc f9 55 73 b9 c5 cf 99 82
   8b 01 ab f5 e4 92 db f3 b0 a4 d4 64 44 e8 ab b3
   0e 8f  # bytes(32)
```

analogously, the multi-sig term `RequireSignature kh` for another example key for Bob
is encoded as:

```haskell
[0, h'08392E1F27777F46594D585DD92186CE405FA7DD369DAFBD3F58D7B9998237C9']
```

```
82  # list(2)
   00  # int(0)
   58 20 08 39 2e 1f 27 77 7f 46 59 4d 58 5d d9 21
   86 ce 40 5f a7 dd 36 9d af bd 3f 58 d7 b9 99 82
   37 c9  # bytes(32)
```

and for an example key for Carl:

```haskell
[0, h'DB108819E4ACCF1D11924CA5B9069654D01E116DA66437D0971C54DB1A46082D']
```

```
82  # list(2)
   00  # int(0)
   58 20 db 10 88 19 e4 ac cf 1d 11 92 4c a5 b9 06
   96 54 d0 1e 11 6d a6 64 37 d0 97 1c 54 db 1a 46
   08 2d  # bytes(32)
```

These can be combined for the following multi-signature terms:

- Alice and Bob (76 bytes)

```haskell
[ 2,
  [ [0, h'C9C65B1126BCF95573B9C5CF99828B01ABF5E492DBF3B0A4D46444E8ABB30E8F']
  , [0, h'08392E1F27777F46594D585DD92186CE405FA7DD369DAFBD3F58D7B9998237C9']]]
```

```
82  # list(2)
   02  # int(2)
   9f  # list(*)
      82  # list(2)
         00  # int(0)
         58 20 c9 c6 5b 11 26 bc f9 55 73 b9 c5 cf 99 82
         8b 01 ab f5 e4 92 db f3 b0 a4 d4 64 44 e8 ab b3
         0e 8f  # bytes(32)
      82  # list(2)
         00  # int(0)
         58 20 08 39 2e 1f 27 77 7f 46 59 4d 58 5d d9 21
         86 ce 40 5f a7 dd 36 9d af bd 3f 58 d7 b9 99 82
         37 c9  # bytes(32)
   ff  # break
```

- 2 out of 3 for Alice, Bob and Carl (113 bytes)

```
[ 3
, 2
, [ [0, h'C9C65B1126BCF95573B9C5CF99828B01ABF5E492DBF3B0A4D46444E8ABB30E8F']
  , [0, h'08392E1F27777F46594D585DD92186CE405FA7DD369DAFBD3F58D7B9998237C9']
  , [0, h'DB108819E4ACCF1D11924CA5B9069654D01E116DA66437D0971C54DB1A46082D']]]
```

```
83  # list(3)
   03  # int(3)
   02  # int(2)
   9f  # list(*)
      82  # list(2)
         00  # int(0)
         58 20 c9 c6 5b 11 26 bc f9 55 73 b9 c5 cf 99 82
         8b 01 ab f5 e4 92 db f3 b0 a4 d4 64 44 e8 ab b3
         0e 8f  # bytes(32)
      82  # list(2)
         00  # int(0)
         58 20 08 39 2e 1f 27 77 7f 46 59 4d 58 5d d9 21
         86 ce 40 5f a7 dd 36 9d af bd 3f 58 d7 b9 99 82
         37 c9  # bytes(32)
      82  # list(2)
         00  # int(0)
         58 20 db 10 88 19 e4 ac cf 1d 11 92 4c a5 b9 06
         96 54 d0 1e 11 6d a6 64 37 d0 97 1c 54 db 1a 46
         08 2d  # bytes(32)
   ff  # break
```

- Alice and Bob or Carl (116 bytes)

```
[ 2
, [ [ 1
    , [ [0, h'C9C65B1126BCF95573B9C5CF99828B01ABF5E492DBF3B0A4D46444E8ABB30E8F']
      , [0, h'08392E1F27777F46594D585DD92186CE405FA7DD369DAFBD3F58D7B9998237C9']]]
  , [0, h'DB108819E4ACCF1D11924CA5B9069654D01E116DA66437D0971C54DB1A46082D']]]
```

```
82  # list(2)
   02  # int(2)
   9f  # list(*)
      82  # list(2)
         01  # int(1)
         9f  # list(*)
            82  # list(2)
               00  # int(0)
               58 20 c9 c6 5b 11 26 bc f9 55 73 b9 c5 cf 99 82
               8b 01 ab f5 e4 92 db f3 b0 a4 d4 64 44 e8 ab b3
               0e 8f  # bytes(32)
            82  # list(2)
               00  # int(0)
               58 20 08 39 2e 1f 27 77 7f 46 59 4d 58 5d d9 21
               86 ce 40 5f a7 dd 36 9d af bd 3f 58 d7 b9 99 82
               37 c9  # bytes(32)
         ff  # break
      82  # list(2)
         00  # int(0)
         58 20 db 10 88 19 e4 ac cf 1d 11 92 4c a5 b9 06
         96 54 d0 1e 11 6d a6 64 37 d0 97 1c 54 db 1a 46
         08 2d  # bytes(32)
   ff  # break
```

## Multi-Sig term Size

The term size can be calculated from the CDDL description above, by recursively
summing up the sizes of the sub-terms. Every key hash requires `34` bytes, so
the encoded keys generally make up the largest part of the size of the encoded
terms. Currently we use indefinite length lists, but as the expected number of
elements in each list is rather low, having an explicit length marker would
allow using a single byte start marker with the length included.

## Script Hashing

The hash of a multi-sig term is computed on a tagged version of the binary
representation. This allows for easy extension with other multi-signature
schemes using different scripting languages in the future without the need to
change the ledger rules.

Therefore when hashing the example script for the required signature of Alice
and Bob above, we prefix the CBOR encoded script with an additional tag `1` and
vkey addresses with a tag `0`. Other tags will be used for other scripting
languages.

## Validation of Multi-Sig Terms

The validation of a multi-signature script is done recursively over the
structure of the term. The validator function also requires the set of
key hashes `vkhs` for which the corresponding key has correctly signed the
transaction.

The validation works as follows

- The term `RequireSignature kh` is evaluated as
```validate (RequireSignature kh) vkhs = kh ∈ vhks```
- The term `RequireAllOf [msigs]` is evaluated as
```validate (RequireAllOf) [msigs] = all (λt. validate t vkhs) [msigs]```
- The term `RequireAnyOf [msigs]` is evaluated as
```validate (RequireAnyOf) [msigs] = any (λt. validate t vkhs) [msigs]```
- The term `RequireMOf m [msigs]` is evaluated as
```validate (RequireMOf m [msigs]) vkhs = m <= sum [if validate t vkhs then 1
else 0 | t <- msigs]```

# Drawbacks/Implications
[drawbacks]: #drawbacksimplications
[implications]: #drawbacksimplications

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The approach to use a script-like DSL for multi-signature schemes allows for an
easy extension point in two dimensions:

 1. additional features, e.g., hash locks and time locks, by extending the terms
 2. integration of other script languages

The specification gives an outline how another script language can be integrated
which is a first step in the direction of full smart contract support with
Plutus. The required changes to the ledger rules and therefore also to the
implementation will be trivial for multi-signature using simplified Plutus (no
data and redeemer scripts) and minimal for full Plutus support.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

The presented scheme does not resolve the problem of the communication that is
necessary for parties to agree and collectively sign a transaction for a
multi-signature scheme.

In the current form there is no overall size limit of the multi-signature
scripts, and there is no relation to transaction fees.
