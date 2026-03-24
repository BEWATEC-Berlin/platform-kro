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

The current `App` API now supports explicit `config.env`, so this example
models the source deployment environment more directly. The `OTEL_SERVICE_NAME`
field-ref is represented through `valueFrom.fieldRef`, while the remaining
static values stay as explicit literal env entries.

The generated `ConfigMap` path is still available through `config.data`, but it
is no longer required just to express normal container environment variables.

## Overlay Pattern

`spec.config.data` is easy to override in normal Kustomize overlays because it is
just ordinary YAML on the `App` resource.

This example includes a reusable base in [base/kustomization.yaml](/Users/francesco/repos/bewatec/platform-kro/examples/otmicro-currency-spike/base/kustomization.yaml#L1)
and an overlay in [overlays/dev/kustomization.yaml](/Users/francesco/repos/bewatec/platform-kro/examples/otmicro-currency-spike/overlays/dev/kustomization.yaml#L1)
with a patch in [overlays/dev/patch-app-currency.yaml](/Users/francesco/repos/bewatec/platform-kro/examples/otmicro-currency-spike/overlays/dev/patch-app-currency.yaml#L1).

That overlay does two things:

- patches `spec.config.data` with environment-specific values
- bumps `spec.config.revision` to force a rollout when those config changes must
  restart pods

Render commands:

```bash
kubectl kustomize examples/otmicro-currency-spike
kubectl kustomize examples/otmicro-currency-spike/base
kubectl kustomize examples/otmicro-currency-spike/overlays/dev
```
