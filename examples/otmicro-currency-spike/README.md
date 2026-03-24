# `otmicro_currency` Spike

This example adapts `otmicro_currency` from the SkelOps OpenTelemetry demo repo
onto the current `App` contract.

It is intentionally a migration spike.
It validates what the current `App` API can already express and records the
remaining gaps without forcing the contract to over-claim.

## What This Example Verifies

- namespace creation
- `App` instance creation
- generated `ServiceAccount`
- generated `ConfigMap`
- generated `Deployment`
- generated `Service`
- deployment availability in a live cluster

## Important Adaptation

The source deployment uses explicit env entries, including a `fieldRef` for
`OTEL_SERVICE_NAME`.

The current `App` API does not yet support explicit env projection, so this
example flattens the static env set into `config.data` and relies on the
existing generated `ConfigMap` plus `envFrom` wiring.

That is acceptable for this spike because the workload is simple and does not
need secrets or dynamic field references to start.
