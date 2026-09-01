# Exact-Byte Encoding v1

## Purpose

Guidon signs, hashes, stores, compares, and journals exact byte sequences. This contract defines the accepted text/JSON profile so independent Guidon components cannot interpret the same security-relevant bytes differently.

## Governing rule

> **Cryptographic verification is performed over the exact accepted byte sequence. Parsing must not silently change the byte sequence being verified.**

Guidon does not require JSON canonicalization for v1 exact-byte artifacts.

## Text encoding

All v1 JSON artifacts covered by this contract use:

```text
UTF-8
no BOM
```

Malformed UTF-8 is rejected.

Implementations must not transparently transcode UTF-16, Latin-1, or another encoding into UTF-8 and then treat the transformed bytes as the original signed/hashed artifact.

## JSON profile

Security-relevant Guidon v1 JSON artifacts use a strict JSON parser profile.

The parser MUST:

- reject duplicate object member names at every nesting level;
- reject malformed JSON;
- reject malformed UTF-8;
- reject non-finite numeric values such as NaN or Infinity;
- enforce the schema-defined type for every field;
- enforce schema-defined numeric ranges and integer requirements;
- enforce required fields;
- reject values outside controlled enums;
- enforce maximum artifact size before unbounded allocation; and
- enforce implementation limits for nesting depth, string length, collection count, and other parser-resource boundaries.

A parser implementation must not rely on a library behavior that silently accepts duplicate keys using `first wins`, `last wins`, or map overwrite semantics.

## Unknown fields

Unknown-field behavior is schema specific.

For security-authority artifacts such as Job v1, authorization assertions, signatures, recovery-point commit artifacts, trust changes, deletion authorization, and recovery authorization, unknown fields MUST be rejected unless the schema version explicitly defines an extension container and its interpretation.

For observational/factual payloads where forward-compatible extension is intentionally permitted, unknown fields may be preserved or ignored only according to that artifact's explicit schema contract.

A parser must never silently accept an unknown field that can alter authority, scope, target, identity, cryptographic interpretation, destructive behavior, or recovery semantics.

## Exact-byte hashing

When a Guidon contract states that an artifact is hashed as exact bytes:

```text
hash = SHA-256(exact accepted bytes)
```

The implementation does not:

- parse and reserialize before hashing;
- normalize whitespace;
- reorder object members;
- normalize Unicode;
- normalize line endings;
- rewrite escaped characters; or
- change number formatting.

Two semantically equivalent JSON documents with different byte encodings are different exact-byte artifacts and therefore have different SHA-256 values.

## Exact-byte signatures

When a Guidon artifact is signed, the signature input is defined by Signature v1 and binds the exact artifact bytes through the artifact SHA-256 plus an explicit Guidon signature domain/purpose.

The verifier validates the exact received/stored bytes before accepting the signature relationship.

## Serialization by Guidon producers

Guidon-owned producers should emit one deterministic serialization form for operational consistency, testing, diffability, and reproducibility. Deterministic producer serialization does not change the governing integrity rule: the exact emitted bytes remain authoritative.

Producer serialization SHOULD:

- use UTF-8 without BOM;
- use one stable object-member ordering per schema implementation;
- use one stable numeric representation;
- use one stable escaping strategy; and
- avoid insignificant formatting variation.

This is an implementation discipline, not a substitute for exact-byte verification.

## Limits

Each artifact schema MUST define or inherit bounded limits appropriate to its role.

Before Phase 1 network exposure, implementations must have explicit limits for at least:

```text
maximum request/artifact bytes
maximum JSON nesting depth
maximum string bytes
maximum array/object member count
maximum identifier length
maximum declared object-transfer length
```

Rejecting an over-limit artifact is a factual protocol result and does not fall back to an unbounded parser path.

## Security acceptance tests

At minimum, every security-relevant parser must test:

- duplicate keys at top level and nested levels;
- malformed UTF-8;
- BOM presence;
- truncated JSON;
- wrong field types;
- unknown security-relevant fields;
- out-of-range integers;
- deeply nested input;
- oversized strings/arrays/documents;
- semantically equivalent but byte-different JSON; and
- signature/hash verification against the exact original bytes rather than a reserialized form.

## Versioning

A future encoding/profile change requires a new explicit schema/profile version. Guidon must not silently reinterpret existing v1 exact-byte artifacts under a later parser profile.
