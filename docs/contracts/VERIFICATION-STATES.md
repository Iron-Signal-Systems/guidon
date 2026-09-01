# Verification State Contract

Guidon uses narrow, explicit recovery/verification states. These states are not interchangeable and do not imply later states.

The locked vocabulary is:

```text
RECEIVED
STORED
VERIFIED
COMMITTED
RECONSTRUCTION_VERIFIED
RESTORED
VALIDATED
```

Guidon never advances a state because previous facts merely suggest the next state is possible. The defining action/check must actually have been performed.

## RECEIVED

`RECEIVED` means the complete expected transfer was observed by the receiving Guidon component.

It does **not** mean:

- durable storage completed;
- integrity verification completed;
- object publication completed; or
- recovery-point commitment occurred.

## STORED

`STORED` means the exact bytes crossed the defined synchronous durability boundary.

For Repository objects, this requires the complete bytes, finalized SHA-256, and the required durable-storage operation to have completed such that Guidon is willing to rely on the stored bytes surviving process/host restart without sender recreation.

`STORED` does not imply later verification, recovery-point commitment, reconstructability, restoration, or workload validation.

## VERIFIED

`VERIFIED` is never an unqualified boolean.

A verification must state:

```text
target/subject
method
expected value/condition
observed value/condition
result
```

Examples include exact SHA-256 comparison, signature verification, manifest-reference existence, or a defined protocol/workload check.

`VERIFIED` means only that the named verification was actually performed and produced the recorded result.

## COMMITTED

`COMMITTED` is primarily a recovery-point lifecycle state.

A recovery point is `COMMITTED` only after all defined commit requirements have actually completed, including object/manifest durability and verification, required immutable preparation history, Journal attestation/receipt durability, and durable publication/commit marker.

`COMMITTED` does not mean the recovery point has ever been reconstructed or restored.

## RECONSTRUCTION_VERIFIED

`RECONSTRUCTION_VERIFIED` replaces the earlier word `RECONSTRUCTABLE`.

The old wording was predictive. Guidon does not predict that data "should be reconstructable" merely because objects appear present.

`RECONSTRUCTION_VERIFIED` means Guidon actually performed the defined reconstruction and the defined reconstruction verification succeeded.

The record must identify what was reconstructed, where/how it was reconstructed, and which checks were performed.

## RESTORED

`RESTORED` means the actual requested restore write/reconstruction completed at the defined target.

It does **not** imply that:

- content hashes match unless that check was performed;
- supported metadata was validated;
- a filesystem/application successfully opened;
- a database is internally consistent;
- Windows booted; or
- a workload is operational.

Those require separate verification/validation records.

## VALIDATED

`VALIDATED` is the strongest of these generic states and must remain workload/check specific.

It means the defined post-recovery validation checks were actually performed and every required check for that validation profile satisfied its defined acceptance condition.

Guidon reports the exact checks. It does not replace them with a broad claim such as:

```text
healthy
fully good
production ready
```

unless a future contract gives such a term an exact testable meaning.

## Historical versus current integrity

Verification is time/context specific.

A recovery point may have historical facts such as:

```text
VALIDATED on 2026-02-01
```

and later experience an unclean Repository/storage event.

The historical validation remains true as an event that occurred. It does not prove the currently stored bytes survived a later power loss.

After a triggering failure event, the recovery point's current stored-data integrity is `pending` until the new verification epoch produces fresh facts.

A post-power-loss object/hash pass does not automatically create a new `VALIDATED` state because no workload restore/validation was necessarily performed.

## Failure/mismatch

A failed verification is not represented by pretending the verification did not happen.

For example:

```text
verification method: SHA-256
expected: AAA
observed: BBB
result: mismatch
```

is an important immutable fact.

The same applies to signature mismatch, missing required object, invalid chain, unavailable dependency, or post-recovery validation failure.

## No generic success substitution

Avoid using a single primary `SUCCESS`/`FAILURE` status where Guidon knows the precise stage/result.

For example:

```text
object received: yes
object stored: yes
object integrity comparison: match
recovery point commit: not_performed
reason: Journal attestation unavailable
```

is more useful and truthful than:

```text
backup failed
```

or:

```text
backup succeeded
```

without detail.
