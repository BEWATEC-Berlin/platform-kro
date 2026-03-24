# App

## Scope

`App` is the generic workload API for the PoC.

It is intentionally broader than a plain deployment wrapper. It can own the
common runtime wiring that would otherwise repeat across services.

It should sit above backing-service APIs rather than force each example to
define databases or caches inline.

As of 2026-03-24, `App` is expected to grow into the main HTTP application API
for the platform migration effort. It should absorb the repeated HTTP service
patterns from the `cc-*` repos instead of remaining only a thin demo wrapper.

The current contract direction is documented in `docs/app-v2-contract.md`.

## Preferred Shape

The preferred long-term `App` shape is organized around four concern buckets:

- `workload`
- `config`
- `network`
- `storage`

This is the preferred internal structure for the `App` contract.
It is not a recommendation to split `App` into separate top-level APIs such as
`AppConfig`, `AppNetwork`, or `AppStorage`.

## Transitional State

The current RGD is transitional.

It still carries older top-level fields and additive transitional blocks such as:

- `runtime`
- `service`

That transitional shape is acceptable while the target contract is being
stabilized, but new design work should be evaluated against the normalized
`workload / config / network / storage` target.

## Labels

`App` now stamps the recommended Kubernetes `app.kubernetes.io/*` labels across
managed resources.

Always stamped:

- `app.kubernetes.io/name`
- `app.kubernetes.io/instance`
- `app.kubernetes.io/managed-by`

Stamped from optional spec fields when provided:

- `app.kubernetes.io/component`
- `app.kubernetes.io/part-of`
- `app.kubernetes.io/version`

Stable selectors use only:

- `app.kubernetes.io/name`
- `app.kubernetes.io/instance`

## Responsibilities

Current first-pass responsibilities:

- `Deployment`
- `Service`
- `ServiceAccount`
- `ConfigMap`
- config-driven rollout token through `config.revision`
- dependency references to `DatabaseCluster` and `CacheCluster` through
  `externalRef`
- optional `HorizontalPodAutoscaler`
- optional `HTTPRoute`
- optional `ExternalSecret`
- optional `Certificate`

Additive v2 responsibilities implemented in this repo revision:

- top-level `component`
- top-level `partOf`
- top-level `version`
- recommended Kubernetes app labels
- `runtime.imagePullSecrets`
- `runtime.imagePullPolicy`
- `runtime.serviceAccountName`
- `runtime.deploymentAnnotations`
- `runtime.podAnnotations`
- `runtime.nodeSelector`
- `runtime.resources`
- `service.type`
- `service.annotations`

Planned next responsibilities should be reasoned about in these buckets:

- `workload`: richer runtime controls, scaling, resilience, and
  cluster-validated models for probes and disruption budgets
- `config`: config and secret projection patterns used repeatedly across app
  repos, including explicit env/envFrom projection and later secret/config
  composition refinements
- `network`: service exposure and `HTTPRoute` wiring
- `storage`: only where the workload still fits a deployment-shaped app

## Boundaries

- no `Ingress`
- no Traefik `IngressRoute`
- no dedicated `Gateway` per app by default
- no cluster-wide issuer or secret-store management
- do not overload `App` to cover clearly stateful or non-HTTP workloads if a
  separate API would be cleaner
- backing services should usually be instantiated outside the app example

## Binding Model

The first pass assumes backing-service APIs exist in the same namespace as the
`App`.

When enabled, `App` binds to:

- `DatabaseCluster` for service names and secret names
- `CacheCluster` for service and endpoint projection

The current deployment wiring references backing services through `externalRef`, but
automatic database/cache environment-variable injection is deferred until it can
be implemented without breaking deployment materialization in KRO.

Explicit container environment projection is available through:

- `config.env` for literal `value` and `valueFrom` entries, including
  `fieldRef`, `resourceFieldRef`, `secretKeyRef`, and `configMapKeyRef`
- `config.envFrom` for additional name-based `configMapRef` and `secretRef` sources

The current `config.envFrom` shape intentionally supports name-based refs only.
Kubernetes `prefix` support is still deferred until KRO can express the merged
list without invalidating the resource graph.

`config.env[].valueFrom.resourceFieldRef` is supported without `divisor`. The
live KRO validation path in this cluster rejected `divisor` as not structurally
compatible with the expected Deployment env schema.

`config.revision` is available as an explicit rollout token. It is stamped onto
the pod template as `platform.connectedcare.io/config-revision`, so overlays can
force a rollout by patching one short field without depending on Kustomize
`configMapGenerator` name hashing.

The intended pattern is:

- patch `spec.config.data` in overlays for environment-specific values
- patch `spec.config.revision` when that config change must force a rollout

A concrete example is included under
`examples/otmicro-currency-spike/overlays/dev`.
