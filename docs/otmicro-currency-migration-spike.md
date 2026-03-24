# `otmicro_currency` Migration Spike

## Purpose

This document maps `otmicro_currency` from
`/Users/francesco/repos/skelops/ot_microservices` onto the current `App`
contract in `platform-kro` and records what can be verified today.

The goal is not to prove that `App` is finished.
The goal is to pressure-test the current contract against a real, simple,
deployment-shaped workload.

## Source Workload

Source manifests:

- ServiceAccount in
  `/Users/francesco/repos/skelops/ot_microservices/otmicro_currency/kustomize/base/01-serviceaccount-currency.yaml`
- Service in
  `/Users/francesco/repos/skelops/ot_microservices/otmicro_currency/kustomize/base/02-service-currency.yaml`
- Deployment in
  `/Users/francesco/repos/skelops/ot_microservices/otmicro_currency/kustomize/base/03-deployment-currency.yaml`

Key source characteristics:

- single `Deployment`
- single container
- single `Service`
- one port on `8080`
- static image `ghcr.io/open-telemetry/demo:2.1.3-currency`
- static environment variables except for one `fieldRef`-based label lookup
- memory limit only
- no route, no backing service APIs, no external secrets

## Mapping To `App`

### Directly Representable

These source fields map directly onto the current `App` contract:

- workload identity via `name`, `component`, `partOf`, and `version`
- image via `image`
- service port via `port`
- replicas via `replicas`
- service account override via `runtime.serviceAccountName`
- image pull policy via `runtime.imagePullPolicy`
- resource limits and requests via `runtime.resources`
- cluster-internal service exposure via `service.type`

### Adapted To Fit The Current Contract

The current `App` contract now supports explicit user-owned `config.env`
projection.

For this spike, the source deployment environment is represented directly in
`config.env`. That includes `valueFrom.fieldRef` for `OTEL_SERVICE_NAME`, so the
example no longer needs to flatten normal env entries into `config.data`.

### Intentionally Not Preserved One-To-One Yet

The following source details are not modeled exactly in the current spike:

- `opentelemetry.io/name` label
- `revisionHistoryLimit`
- `fieldRef`-based `OTEL_SERVICE_NAME`

The `fieldRef` case is now modeled directly through `config.env[].valueFrom.fieldRef`.

## Why `otmicro_currency` Was Chosen

`otmicro_currency` is a useful first migration spike because it is simple enough
for the current `App` contract while still surfacing real design pressure.

It is a better first target than:

- `otmicro_accounting`, which already needs an init container
- `otmicro_flagd`, which already needs multiple containers, init containers,
  and volumes
- stateful services such as `otmicro_opensearch`

## Verification Scope

The verification target for this spike is:

1. `ResourceGraphDefinition` update applies to a live KRO-enabled cluster
2. the example `App` instance reconciles successfully
3. generated `ServiceAccount`, `ConfigMap`, `Deployment`, and `Service` appear
4. the `currency` deployment becomes available in the test namespace

A full feature-parity claim is intentionally out of scope because the current
contract still lacks gRPC-native health probes and other richer workload knobs.

## Live Verification Result

The live cluster verification on 2026-03-24 used Kubernetes context
`k8s-dev-oidc` with KRO `v0.8.5` installed.

The first verification pass exposed a real platform issue:

- the null-based conditional probe rendering pattern made the `App`
  `ResourceGraphDefinition` inactive
- because the RGD was inactive, the `App` CRD schema on the cluster did not pick
  up the new fields and the example `App` instance was rejected by the API

That first probe implementation was then removed from the executable RGD and
reworked. The current repo now supports HTTP readiness and liveness probes with
a cluster-validated object-or-empty-map rendering pattern, but richer probe
types remain deferred.

At that point in the repo history, disruption-budget support was still
deferred. As of 2026-03-24 later in the same validation cycle, the consumer
cluster RBAC was widened and `App` now supports a narrow optional
`PodDisruptionBudget` path using `disruptionBudget.maxUnavailable`.

The rest of the migration spike remained valid and could then be verified end to
end.

## Current Contract Gaps Exposed By This Spike

This spike now confirms that the next meaningful `App` gaps are:

- richer probe support for startup, gRPC, or exec health checks
- broader pod and container `securityContext` support beyond the current restricted preset
- multi-container and init-container support

## Example Artifact

The concrete migration spike artifact is in:

- `examples/otmicro-currency-spike/namespace.yaml`
- `examples/otmicro-currency-spike/app-currency.yaml`
- `examples/otmicro-currency-spike/kustomization.yaml`
- `examples/otmicro-currency-spike/README.md`
