# `otmicro_currency` Migration Spike

## Purpose

This document maps `otmicro_currency` from
`/Users/francesco/repos/skelops/ot_microservices` onto the current `App`
contract in `platform-kro` and records what was verified on 2026-03-25.

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
- explicit env entries, including `valueFrom.fieldRef`
- memory-only limit in the source manifest
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
- cluster-internal service exposure via `service.type`
- explicit environment projection via `config.env`
- `valueFrom.fieldRef` for `OTEL_SERVICE_NAME`

### Adapted To Fit The Current Contract

The current `App` contract always emits a generated `ConfigMap`, even though the
source manifests do not use one.

The source resource shape also needed a small normalization:

- the `App` contract emits both requests and limits, not only a memory limit
- the generated container is always named `app`
- the generated service port is always named `http`

Those are acceptable migration differences for this spike because they do not
change the basic workload behavior.

### Not Preserved One-To-One

The following source details are not modeled exactly in the current spike:

- `opentelemetry.io/name` labels and selectors
- `app.kubernetes.io/environment` from the source overlays
- exact service and port names such as `tcp-service` and `service`
- `revisionHistoryLimit`
- exact resource shape with only `limits.memory`

## Why `otmicro_currency` Was Chosen

`otmicro_currency` is a useful first migration spike because it is simple enough
for the current `App` contract while still surfacing real design pressure.

It is a better first target than:

- `otmicro_accounting`, which already needs an init container
- `otmicro_flagd`, which already needs multiple containers, init containers,
  and volumes
- stateful services such as `otmicro_opensearch`

## Live Verification Result

The live cluster verification on 2026-03-25 used Kubernetes context
`k8s-dev-oidc` with KRO `v0.8.5` installed through the Flux consumer path in
`hetzner_playground`.

The verified example is the base manifest in:

- `examples/otmicro-currency-spike/base/app-currency.yaml`
- `examples/otmicro-currency-spike/base/namespace.yaml`
- `examples/otmicro-currency-spike/base/kustomization.yaml`

Verified live outcome:

1. `App/currency` reconciled successfully in namespace `platform-kro-currency`
2. generated `ServiceAccount`, `ConfigMap`, `Deployment`, and `Service` existed
3. `Deployment/currency` became `Available`
4. `App/currency` reported `Ready=True`
5. the generated container used image `ghcr.io/open-telemetry/demo:2.1.3-currency`
6. the generated env list preserved the explicit source values and `fieldRef`

The live `App` annotation confirmed the managed child-resource set as:

- `ConfigMap`
- `Deployment.apps`
- `Service`
- `ServiceAccount`

## Important Limits Exposed By This Spike

This spike also confirmed two real limits that still matter for later work:

- the current `App` HTTP probe path is not suitable for this service, because
  `otmicro_currency` does not answer normal HTTP on port `8080`
- the restricted security preset is not usable for this image, because the
  image still runs as root

That means this workload can be migrated as a basic internal service, but it is
not a proof that the current `App` contract covers gRPC-native or stricter
restricted-policy workloads without more design work.

## Example Artifact

The concrete migration spike artifact is in:

- `examples/otmicro-currency-spike/base/kustomization.yaml`
- `examples/otmicro-currency-spike/base/app-currency.yaml`
- `examples/otmicro-currency-spike/base/namespace.yaml`
- `examples/otmicro-currency-spike/overlays/dev/kustomization.yaml`
- `examples/otmicro-currency-spike/overlays/dev/patch-app-currency.yaml`
- `examples/otmicro-currency-spike/README.md`
