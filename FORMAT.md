# arc - disk format

arc archives are tar archives compressed with gzip and then encrypted
with the XChaCha20 + Poly1305 authenticated encryption mode using a
key derived in one of three ways:

  1. from a password using the Argon2 KDF
  2. from a static-ephemeral ECDH key exchange
  3. from a random key split into n shards

The on-disk format begins with a 1-byte disk format version V followed
by a 1-byte archive type T, then a type-specific number of bytes, a
16-byte Poly1305 authentication tag, a 24-byte cryptographically secure
random nonce, and finally the encrypted data.

All numbers are stored in little-endian format.

## Password Archive Format

The 32-byte XChaCha20Poly1305 key is generated by applying the Argon2
KDF to a user-supplied password and 32 bytes of cryptographically
secure random salt.

    T = 1
    I = uint32 number of iterations
    M = uint32 memory usage

    ┌─┬─┬────┬────┬───────────────────────────────┐
    │V│T│I   │M   │Salt                           │
    ├─┴─┴────┴────┴─┬───────────────────────┬─────┴─────────────┐
    │Tag            │Nonce                  │Data···············│
    ├───────────────┴───────────────────────┴───────────────────┤
    │···························································│
    └───────────────────────────────────────────────────────────┘

## Curve448 Archive Format

The 32-byte XChaCha20Poly1305 key results from applying BLAKE2b to the
shared secret derived from a X448 ECDH key exchange with an ephemeral
private key and static public key. The corresponding ephemeral public
key is embedded in the archive.

    T = 2

    ┌─┬─┬───────────────────────────────────────────────────────┐
    │V│T│Ephemeral Public Key                                   │
    ├─┴─┴───────────┬───────────────────────┬───────────────────┤
    │Tag            │Nonce                  │Data···············│
    ├───────────────┴───────────────────────┴───────────────────┤
    │···························································│
    └───────────────────────────────────────────────────────────┘

## Shard Archive Format

The 32-byte XChaCha20Poly1305 key is cryptographically secure random
bytes split into n shards using Shamir's Secret Sharing algorithm.
One archive is generated for each n and k archives must be present to
recreate the key and decrypt the archive.

    T = 3
    n = shard number

    ┌─┬─┬─┬───────────────────────────────┐
    │V│T│n│Shard                          │
    ├─┴─┴─┴─────────┬─────────────────────┴─┬───────────────────┐
    │Tag            │Nonce                  │Data···············│
    ├───────────────┴───────────────────────┴───────────────────┤
    │···························································│
    └───────────────────────────────────────────────────────────┘

## Curve448 Key Format

arc curve448 public and private keys are encrypted with XChaCha20+
Poly1305 using a key derived from applying the Argon2 KDF to a
password and 32 bytes of cryptographically secure random salt.

    T = public = 1, private = 2
    I = uint32 number of iterations
    M = uint32 memory usage

    ┌─┬─┬────┬────┬───────────────────────────────┐
    │V│T│I   │M   │Salt                           │
    ├─┴─┴────┴────┴─┬───────────────────────┬─────┴─────────────┐
    │Tag            │Nonce                  │Key················│
    ├───────────────┴───────────────────────┴───────────────────┤
    │···························································│
    └───────────────────────────────────────────────────────────┘

Private keys use a user-supplied password while public keys use an
empty string.