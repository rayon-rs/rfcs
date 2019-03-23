# Summary

- Add on-device storage for key material, e.g. to perform signatures, encryption/decryption, or static passwords
- Expose these keys and associated operations using a dedicated CTAPHID vendor command CRYPTOKI
- When in doubt, follow the general principles of PKCS#11 ("Cryptoki") for easier mapping to host consumers
- Extend `solo` Python tool to exercise the new command
- Add cross-platform PKCS#11 library exposing the API

# Motivation

We would like to extend Solo functionality beyond the FIDO2 use case. Example use cases are

- sign documents, Git commit, emails
- store and use SSH keys
- encrypt or decrypt files with a symmetric or public key
- (potentially) store and use WireGuard keys

In all cases, there is a private key involved that is not directly exposed (TBD whether keys are strictly generated on the device and never leave it, or whether things like certificate chains, loading keys, extracting wrapped keys are supported).

# Guide-level explanation

How does this affect public SoloKeys products? If this is purely an
implementation detail, it's ok to note that it doesn't affect SoloKeys
users very much.

# Implementation notes

## Storage

Let's use flash page 109 + 110 from the spare storage, this gives 4096 bytes.

TBD how big an individual slot needs to be, we don't want to support huge keys such as 4096 bit RSA, so slots should not be huge.

E.g. if a slot is 512 bytes (for comparison, a FIDO2 resident key is 360 bytes in total), we'd get 8 slots

## Low-level interface / Transport

We have an existing CTAPHID transport, let's use it. In analogy to CTAP2 `authenticatorClientPIN`, we can use for instance CTAPHID 0x55 as `CTAPHID_CRYPTOKI`, together with subcommands corresponding to Cryptoki operations

Currently, of the custom command 0x40 - 0x7F, we use 0x50, 0x51, 0x52, 0x60, and 0x70.

## High-level interface(s)

The standard way to expose "Portable computing devices such as (...) smart diskettes" is shared library exposing C functions from PKCS#11. It seems relatively easy to write such a `solo-pkcs11.so`, even in Rust, for instance Krypton does it [here](https://github.com/kryptco/kr/tree/master/pkcs11shim).

The `solo` tool itself can directly speak CTAPHID.

### Git

Git can be configured with `git config --global gpg.program <something>`  to use any program for signing commits, as long as that program accepts what is to be signed on stdin, and outputs OpenPGP compatible output (ASCII armored etc.) to stdout. Withoutboats implements such a program (storing keys on host) [here](https://github.com/withoutboats/bpb). In our case, that program could even be built on `solo` and directly speak CTAPHID. It would be a bit nicer to have it in Rust, so it can be easily distributed in binary format for Windows, Linux and macOS.

### SSH

For SSH, we have a bit of a temporary problem. Unless we want to implement huge RSA keys, the only reasonable choice is ed25519 keys. Current PKCS#11 (v2.4) does not have this. Version 3.0, slated for release in 2019, should have it, there is a [draft](https://www.oasis-open.org/committees/document.php?document_id=62198&wg_abbrev=pkcs11), and [SoftHSM]( https://github.com/opendnssec/SoftHSMv2) implements it. So we'd be working ahead to some degree. We could fall back on ECDSA keys, apart from the trust issue.

# Rationale and alternatives

The way that Yubico does it is to integrate with https://github.com/OpenSC/OpenSC and pretend it is an actual smart card, which entails all the complexity of GPG, PIV, CCID, etc. This all does not seem to be very well documented.

One could also forgo either of CTAPHID or PKCS#11 as interfaces, I believe it is good to stick to pre-existing standards and their extension points if possible.

If we don't do this, someone will have to come up with an alternative plan :)

# Unresolved questions

Details to be added as discussion evolves.

# References

- CTAP2 protocol: https://fidoalliance.org/specs/fido-v2.0-rd-20180702/fido-client-to-authenticator-protocol-v2.0-rd-20180702.html#commands
- Cryptoki usage guide: http://docs.oasis-open.org/pkcs11/pkcs11-ug/v2.40/cn02/pkcs11-ug-v2.40-cn02.html
- Rust Cryptoki shim: https://github.com/kryptco/kr/tree/master/pkcs11shim
- Software Cryptoki device: https://github.com/opendnssec/SoftHSMv2
- Git GPG shim: https://github.com/withoutboats/bpb