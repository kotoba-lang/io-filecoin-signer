# io-filecoin-signer

`filecoin.signer` — **secp256k1 signing for Filecoin**, the `f1` path, on both
runtimes. The seam [`io-filecoin`](https://github.com/kotoba-lang/io-filecoin)
leaves open, filled in for the one case that needs no new cryptography.

`io-filecoin` pins no curve on purpose, so the same message code runs on a
JVM, in a Worker or behind a WASM capability import, and a private key can
stay in a Keychain. This repo is a separate dependency for the same reason: a
caller who wants a curve pinned asks for it.

## Usage

```clojure
(require '[filecoin.signer :as signer]
         '[filecoin.message :as msg])

(signer/address-string priv)        ; => "f1…" — the account this key controls
(signer/sign-message priv message)  ; => a signed message, type 1
(signer/verify-message signed)      ; => recovers the signer, compares to :from

(signer/signer priv)                ; => an ISigner for io-filecoin's seam
```

## What differs from Ethereum — only the last step

The curve is the same, the signature is the same 65 bytes. The chains part
company at two places, and both are silent when wrong:

|  | Ethereum | Filecoin `f1` |
|---|---|---|
| address | keccak-256(X‖Y)[12:] | BLAKE2b-160(**`0x04`**‖X‖Y) |
| digest | keccak-256 of the payload | BLAKE2b-256 **of the message CID** |

**The `0x04` matters.** Ethereum hashes the 64-byte key; Filecoin hashes the
65-byte SEC1 uncompressed form, prefix included. Omit it and you get a
perfectly well-formed `f1` address for an account nobody controls — nothing
complains until a transfer disappears into it.

**The digest is hashed twice.** The message CID is already a BLAKE2b-256, and
secp256k1 signs BLAKE2b-256 *of* that. Signing the CID directly gives a
signature of the right length that verifies against nothing.

## What this cannot sign

- **`f3` (BLS12-381).** No implementation of that curve exists anywhere in
  this workspace.
- **Delegated (type 3)**, the FEVM path. A delegated signature covers the
  RLP-encoded unsigned Ethereum transaction, keccak-hashed — not the message
  CID. It is not even the same shape:

  ```
  EIP-1559 (type 2)   r ‖ s ‖ v          65 bytes, v ∈ {0,1}
  legacy EIP-155      0x02 ‖ r ‖ s ‖ v   67 bytes on mainnet, since
                                         v = chainId·2 + 35 = 663 takes two
  ```

  Both shapes appear in this repo's mainnet vectors. `eth-crypto` now has RLP,
  keccak and EIP-1559 signing on both runtimes, so building this is possible;
  what is missing is the exact Filecoin↔EthTransaction round-trip lotus
  requires, and getting it wrong yields a signature the network rejects rather
  than an error here.

## Verification

The vectors are **mainnet's**, and they are checked in the one direction that
cannot be faked: **recovery**. Every signature in them was produced by
somebody else's wallet, over a message the network accepted. Recovering the
signer and getting back `From` exercises CBOR field order, the message CID,
the second BLAKE2b-256, ECDSA recovery, the `0x04` prefix and BLAKE2b-160 in a
single comparison — and nothing this library computes is an input to it.

The delegated vectors are there to assert a **negative**: recovery over the
message digest must *not* return the sender, because a delegated signature
covers something else. That is ADR-2607299300's correction, checked against
the network rather than against a reading of lotus — and the network turned
out to say something stronger than the ADR did, since one of the two shapes is
not 65 bytes at all.

RFC-6979 makes signing deterministic, so the pinned signature bytes are a
**cross-implementation** check: `eth-crypto` implements the nonce separately
on each runtime (BigInteger + javax.crypto; js/BigInt + a hand-written
SHA-256), and both suites assert the same bytes.

**44 assertions, green on both.**

```sh
clojure -M:test        # JVM
npm run test:cljs      # nbb
npm run vectors        # re-snapshot mainnet (not run in CI)
```

The test key is the EIP-155 worked example's, which is public by construction
and controls nothing. A generated key would make the pinned signatures
unreproducible, and a real one has no place in a test file.

## Scope — read this before using it

**Nothing here has been used to send a transaction.** The signatures are
verified by recovering them, which is the same check a node makes, but no
message signed by this code has been submitted to a network. A working client
also needs gas and nonce (`Filecoin.GasEstimateMessageGas`,
`Filecoin.MpoolGetNonce` — `filecoin.rpc` builds those request bodies) and a
transport, which is still nobody's job in this family of repos.
