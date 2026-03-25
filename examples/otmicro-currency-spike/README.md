# `otmicro_currency` Spike

This example adapts `otmicro_currency` from the SkelOps OpenTelemetry demo repo
onto the current `App` contract.

It is intentionally a migration spike.
It validates what the current `App` API can already express and records the
remaining gaps without forcing the contract to over-claim.

## What This Example Verifies

- namespace creation through the base kustomization
- `App` instance creation under `platform.connectedcare.io/v1alpha1`
- generated `ServiceAccount`
- generated `ConfigMap`
- generated `Deployment`
- generated `Service`
- deployment availability in the development cluster

## Important Adaptation

The source deployment uses explicit env entries, including a `fieldRef` for
`OTEL_SERVICE_NAME`.

This spike keeps those runtime values in `spec.config.env`, because that is the
closest current `App` mapping for this workload. The example does not use the
restricted security preset, because the source image still runs as root.

The source service also does not speak normal HTTP on port `8080`, so the spike
keeps readiness and liveness probes disabled. The current `App` HTTP probe
contract remains useful for HTTP workloads, but it is not the right fit for
this service.

## Overlay Pattern

This example includes a reusable base in [base/kustomization.yaml](/Users/francesco/repos/bewatec/platform-kro/examples/otmicro-currency-spike/base/kustomization.yaml#L1)
and an overlay in [overlays/dev/kustomization.yaml](/Users/francesco/repos/bewatec/platform-kro/examples/otmicro-currency-spike/overlays/dev/kustomization.yaml#L1)
with a patch in [overlays/dev/patch-app-currency.yaml](/Users/francesco/repos/bewatec/platform-kro/examples/otmicro-currency-spike/overlays/dev/patch-app-currency.yaml#L1).

That overlay does two things:

- patches `spec.config.env` with environment-specific runtime values
- bumps `spec.config.revision` to force a rollout when those config changes must
  restart pods

Render commands:

```bash
kubectl kustomize examples/otmicro-currency-spike
kubectl kustomize examples/otmicro-currency-spike/base
kubectl kustomize examples/otmicro-currency-spike/overlays/dev
```
