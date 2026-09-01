# Guidon Go Engineering Standard

## Purpose

This file defines the Go implementation standard for Guidon.

Guidon Go code should follow the same engineering direction used by Iron Signal Systems File Intelligence (FI): explicit, focused, reviewable code with strong platform behavior and minimal unnecessary abstraction.

The Go module lives under:

```text
/go
```

Current approved Go baseline:

```text
Go 1.27.0
```

## Repository Layout

Prefer focused packages under `go/internal/`.

Examples:

```text
internal/repository/
internal/journal/
internal/windows/
```

Do not create catch-all packages such as `util`, `common`, `helpers`, `framework`, `platform`, or `misc` merely to hold unrelated functionality.

Package boundaries should represent real engineering responsibilities.

## Source Licensing

Every Go source file must contain the Guidon source-review header:

```go
// Copyright (c) 2026 John Joseph Wood. All rights reserved.
// Use of this source code is governed by the Guidon
// Source Review License, Version 1.0, found in the repository root LICENSE file.
```

Scripts, Makefiles, and other applicable source artifacts should carry the equivalent appropriate notice.

## General Coding Style

Prefer obvious code over clever code.

Use `switch` for discrete alternatives, classifications, and state handling.

Use `if` for guards, ranges, boolean conditions, and compound predicates.

Keep control flow shallow where practical.

Prefer early validation and explicit return paths.

Avoid unnecessary indirection and unnecessary interfaces.

Introduce an interface only when a real implementation boundary, interchangeable implementation, or test boundary justifies one.

Do not introduce generic factories, registries, dependency-injection frameworks, reflection-driven behavior, generalized callback systems, or abstract plugin mechanisms unless the actual current requirement needs them.

Do not build abstractions merely because a future phase might eventually need them.

## Function and Type Organization

Keep source files organized and easy to review.

Prefer a consistent order such as:

```text
constants
sentinel errors
types
exported functions
internal functions
```

Within each logical section, keep names in predictable alphabetical order where doing so does not break necessary semantic grouping.

Keep tightly coupled lifecycle functions together when separating them solely for alphabetization would make the code harder to follow.

Export only what another package actually needs.

Exported names must have meaningful comments where Go tooling expects them.

Do not create oversized files containing unrelated responsibilities merely to reduce file count.

Do not fragment simple packages into excessive one-function files without a real platform or responsibility boundary.

## Dependencies

Prefer the Go standard library.

For operating-system interfaces, prefer the appropriate `golang.org/x/sys` package when it exposes the required native API correctly.

Add third-party dependencies only when they provide a concrete capability that would otherwise require significant custom code or security-sensitive reimplementation.

Do not add frameworks for convenience.

Every new dependency should be reviewable for:

```text
purpose
maintenance status
license
security impact
transitive dependency cost
supported operating systems
```

## CGO

Prefer pure Go and native operating-system API/syscall bindings.

Do not introduce CGO merely because a C library is convenient.

CGO may be used only when a required supported-platform capability cannot reasonably be implemented through Go/native system interfaces and the dependency is justified.

Any CGO introduction requires explicit review of:

```text
build portability
memory-safety boundary
deployment dependencies
supported target operating system(s)
cryptographic/FIPS implications where applicable
```

## Error Handling

Errors should identify what failed.

Wrap errors when operation context materially improves diagnosis.

Use sentinel errors only when callers genuinely need classification.

Do not create large error taxonomies without a demonstrated consumer.

Do not swallow durability failures, integrity failures, security failures, authorization failures, or cleanup failures that affect state.

If a cleanup failure occurs after successful publication or another meaningful state transition, preserve the completed transition and report the cleanup problem separately where necessary.

Do not reduce a multi-stage operation to one generic error when Guidon knows which stage actually occurred.

Preserve native error/result information long enough to diagnose the actual platform failure.

Do not prematurely convert distinct native failures into one generic `operation failed` result.

Translation into Guidon error/state vocabulary should retain the underlying native cause where safe and useful.

## Explicit State

Model meaningful Guidon state explicitly.

Do not use a single `Success bool` where the system knows more.

Preserve distinctions such as:

```text
received
durable
published
publication durable
verified
not_performed
mismatch
unavailable
cleanup pending
already present
```

A zero value must not accidentally mean a successful verification or completed security operation.

Where the architecture distinguishes stages, implementation state should allow those distinctions to remain observable.

## Platform Targeting

Guidon components target the operating system on which they are intended to run.

Do not require platform-specific code to compile or operate on unrelated operating systems merely to create artificial cross-platform compatibility.

Examples:

```text
Repository / Journal appliance code
    -> target FreeBSD when the defined deployment platform is FreeBSD

Windows endpoint/service code
    -> target Windows

Linux endpoint/service code
    -> target Linux

Shared domain/protocol code
    -> may remain platform-neutral when behavior is genuinely common
       and the package is actually consumed by multiple platforms
```

Target the platform intentionally.

If a component lives on FreeBSD, design it for FreeBSD.

If a component lives on Windows, design it for Windows.

If a component lives on Linux, design it for Linux.

Do not weaken, emulate, or abstract away useful native platform behavior solely so the same implementation can compile on another operating system.

Portability follows deployment responsibility. It is not a goal by itself.

A platform implementation must never return success merely to satisfy a common interface when the required property was not established.

If a capability required by the operation is unsupported on the target platform, return an explicit unsupported/not-established result and fail closed where the operation requires that capability.

Compile-time portability is not more important than truthful behavior.

## Native Platform Engineering

Guidon is cross-platform at the product and domain-semantics level where multiple platform-specific components exist.

It is not cross-platform by forcing each operating system through identical implementation mechanics.

Use the strongest appropriate native operating-system interface for the platform where the component lives.

Platform-specific mechanics may differ. Guidon semantics may not.

### FreeBSD

FreeBSD-specific code should use appropriate FreeBSD and OpenZFS behavior for filesystem durability, Jails, native system interfaces, storage behavior, and process/service boundaries.

Do not assume Linux behavior is identical to FreeBSD.

Repository and Journal code that lives on the FreeBSD appliance should be engineered and tested for FreeBSD rather than made artificially portable to Windows or Linux.

### Linux

Linux-specific code should use Linux-native interfaces and syscalls where platform-specific behavior matters.

Do not force Linux-specific behavior through Windows or FreeBSD abstractions merely for symmetry.

Linux components should be validated on Linux.

### Windows

Windows is a first-class Guidon platform.

Code whose responsibility exists specifically on Windows should be designed for Windows and use Windows-native operating-system facilities rather than being forced through a lowest-common-denominator Unix-style abstraction.

Windows-owned implementation should normally live under `internal/windows/` and/or files using the `_windows.go` suffix.

Do not place Windows-native syscall/API behavior in a generic cross-platform file merely because the function signature is shared.

Shared domain types may remain platform-neutral when their meaning is genuinely common.

Preferred Windows direction:

- use documented Windows APIs and native system calls where appropriate;
- use Go's Windows support and `golang.org/x/sys/windows` where it provides the required API correctly;
- use direct DLL/native syscall bindings where necessary and justified;
- use explicit `//go:build windows` files for Windows-only implementation where appropriate;
- preserve native Windows handles, error codes, structures, flags, and behavior long enough to interpret them correctly;
- validate native structure sizes, returned byte counts, buffer boundaries, and API results before interpreting returned data;
- isolate `unsafe` usage to the native API boundary;
- use `runtime.KeepAlive` where syscall pointer lifetime requires it; and
- convert native API results into Guidon domain types only after validating the operating-system result.

Follow the same engineering direction currently used by FI for Windows-native collection and control boundaries.

## Windows API Preference

Do not replace an appropriate supported Windows API with:

```text
PowerShell execution
cmd.exe execution
parsing command-line utility output
WMI/CIM when a more direct supported API is appropriate
POSIX emulation
generic filesystem abstractions that discard Windows semantics
```

External commands are acceptable only when no appropriate supported programmatic interface exists or when the command itself is the defined supported interface.

Such use must be intentional, bounded, and documented.

Examples of preferred Windows-native directions include:

```text
filesystem / NTFS
    -> CreateFileW
    -> DeviceIoControl / FSCTL
    -> native file IDs
    -> volume APIs
    -> reparse information
    -> security descriptors

services
    -> Service Control Manager APIs

certificates / key material
    -> Windows certificate store APIs
    -> CNG / CryptoAPI where appropriate

VSS
    -> supported VSS APIs / COM interfaces

Event Log
    -> Windows Event Log API

identity / tokens / SIDs
    -> native Windows security/token APIs
```

Do not parse utility output when the operating system exposes the required structured API directly.

## Native Boundary Safety

Platform-native code must validate operating-system output before turning it into Guidon facts.

Validate, as applicable:

```text
returned lengths
buffer offsets
structure versions
structure sizes
integer ranges
handles
native result codes
termination conditions
encoding
```

Do not trust a native buffer merely because the operating-system call returned success.

Keep privileged/native packages narrow.

Parsing that does not require privileged access should generally remain outside the privileged native-call boundary.

## Filesystem and Durability Engineering

`write()` success is not durability.

Do not report a durability-dependent state before all required target-platform durability operations complete.

Durability-sensitive code must clearly distinguish stages such as:

```text
bytes written
file synchronized
file closed
directory synchronized
object published
publication directory synchronized
cleanup completed
```

Do not hide durability behavior behind vaguely named helpers.

Platform-specific durability mechanisms may differ, but they must establish the same required Guidon semantic before the same Guidon state is reported.

Do not silently continue after `fsync`/equivalent failure, directory-sync/equivalent failure, storage full, I/O error, or publication conflict.

Preserve interrupted artifacts when the recovery/reconciliation contract requires them.

## Immutable Object Handling

For content-addressed immutable Repository objects:

```text
receive
    -> calculate SHA-256 independently
    -> require exact declared size
    -> cross incoming durability boundary
    -> publish without clobbering an existing object
```

If the final object already exists:

```text
open existing object
    -> independently verify expected size
    -> independently hash exact bytes
    -> reuse only if identity matches
```

Never overwrite an existing immutable object to resolve an integrity conflict.

A conflicting object must remain observable as a conflict.

## Security Engineering

Do not log or intentionally persist:

```text
private keys
OTP values
TOTP secrets
complete plaintext Job bodies
temporary recovery passwords
decrypted sensitive control artifacts
```

Keep cryptographic operations purpose-specific.

Avoid generic APIs such as `Sign(data)`, `EncryptAnything(data)`, or `Execute(command)` when the architecture requires constrained operations such as `SignJournalReceipt`, `VerifyJournalReceipt`, `EncryptDurableJob`, or `VerifyRecoveryAuthorization`.

Do not weaken authority separation for implementation convenience.

## Tests

Tests should live with the package.

Use clear FI-style naming and focused tests.

Test both successful behavior and failure behavior.

For security-sensitive or durability-sensitive operations, refusal/failure tests are required alongside happy-path tests.

Examples:

```text
short input
long input
hash mismatch
existing-object conflict
storage failure
durability failure
malformed native buffer
wrong native structure size
unexpected API result
restart with interrupted artifact
```

Tests involving immutable-object conflict must verify that the original object was not overwritten.

Tests should establish what Guidon claims, not merely whether a function returned nil.

## Required Validation

Before a Go change is considered ready for review, run as applicable:

```text
gofmt
go test ./...
go vet ./...
```

Validate each component against its intended supported operating system(s).

Examples:

```text
FreeBSD Repository / Journal
    -> GOOS=freebsd GOARCH=amd64 build/compile checks as applicable
    -> native runtime testing on FreeBSD
    -> OpenZFS/Jail/durability testing on FreeBSD

Windows service / endpoint
    -> GOOS=windows GOARCH=amd64 build/compile checks as applicable
    -> native Windows runtime tests
    -> Windows API/security/service/VSS/NTFS tests on Windows

Linux service / endpoint
    -> GOOS=linux GOARCH=amd64 build/compile checks as applicable
    -> native Linux runtime tests
```

A genuinely shared package should be tested across the operating systems on which that package is actually intended to run.

Do not require FreeBSD-only Repository code to compile for Windows or Windows-only VSS code to compile for Linux.

Cross-compilation does not prove runtime behavior such as filesystem durability, directory synchronization, OpenZFS behavior, Jail boundaries, Windows FSCTL behavior, Windows service behavior, VSS behavior, native security-token behavior, power-loss behavior, or ENOSPC behavior.

If a required target/runtime check could not be executed, state that explicitly.

Do not substitute successful cross-compilation for a runtime claim.

## Change Review

Before finishing a Go implementation slice, verify:

- code follows the applicable Guidon architecture/contracts;
- no state was made broader or more positive than what was established;
- platform-native behavior remains appropriate for where the component lives;
- no security boundary was weakened;
- no future-phase abstraction was introduced unnecessarily;
- durability behavior is explicit;
- failure behavior is tested;
- formatting/tests/vetting were performed where possible; and
- intended target-platform builds remain intact.
