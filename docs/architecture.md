# Architecture

## Purpose

`platform-kro` owns the application-facing platform APIs built with KRO. It
does not own cluster bootstrap.

As of 2026-03-24, the platform direction is to evolve this repo into a fuller
application platform API layer instead of keeping the API set intentionally
thin.

Supporting design documents:

- `docs/app-v2-contract.md`
- `docs/data-service-boundaries.md`
- `docs/cc-kustomize-gap-analysis.md`

Consumers such as `hetzner_playground` are responsible for:

- installing KRO
- installing dependent operators and controllers
- configuring cluster-specific resources such as Gateway controllers,
  `cert-manager`, and `external-secrets`
- instantiating the APIs defined here

## Boundaries

This repo should stay provider-neutral.

Keep these concerns out of the API layer:

- Hetzner-specific naming
- cluster bootstrap
- environment-specific DNS
- secret store setup
- gateway controller installation

Keep these concerns out of scope for the first migration wave:

- direct support for `Ingress`, `IngressRoute`, or `IngressRouteTCP`
- provider-specific traffic CRDs
- a dedicated `Gateway` per application by default

## API Set

### `DatabaseCluster`

- wraps the PostgreSQL operator contract behind a smaller stable API
- stays PostgreSQL-focused in the current migration wave
- should evolve into a platform database contract rather than remain a thin
  CloudNativePG pass-through
- a future MariaDB retained API should be separate if the contract is needed

### `CacheCluster`

- starts as a plain StatefulSet-based cache contract
- should evolve into a stable cache service contract that can absorb more of
  the current application estate
- may swap to an operator-backed implementation without changing the
  consumer-facing API
- may absorb Dragonfly-backed implementations if the app-facing contract stays
  coherent

### `App`

- sits one abstraction layer above backing-service APIs such as
  `DatabaseCluster` and `CacheCluster`
- is the main HTTP application-facing abstraction
- should be organized internally around four concern buckets:
  `workload`, `config`, `network`, and `storage`
- should stay a single application API instead of being split into separate
  app-adjacent APIs for each Kubernetes concern area
- may create optional `HTTPRoute`, `ExternalSecret`, `Certificate`, and other
  common application-facing resources that are repeated across consumer repos
- should not create dedicated `Gateway` resources by default

The concern-bucket split is an internal contract organization, not a signal to
create separate top-level APIs such as `AppNetwork` or `AppStorage`.

### `AppDelivery`

- is the delivery-facing companion contract to `App`
- should own developer-facing CI and promotion intent, not raw Tekton or Kargo
  resources
- should stay above mutable environment release state
- should allow consumer repos to render or reconcile their chosen delivery
  control-plane implementation without changing the app-facing contract
- should remain separate from `App` so runtime and delivery concerns do not
  collapse into one overloaded API

## Exposure Model

The standard exposure model for HTTP applications is Gateway API
`HTTPRoute`.

That means:

- the target platform API layer should not introduce `Ingress`
- the target platform API layer should not introduce Traefik-specific
  `IngressRoute` resources
- migration work should convert HTTP exposure to `HTTPRoute`

Some current consumer repos expose non-HTTP traffic through
`IngressRouteTCP`. Those workloads cannot be modeled with `HTTPRoute`. They
need one of the following:

- a separate future API using Gateway API `TCPRoute` or `TLSRoute`
- temporary retention of raw manifests outside the `App` migration wave

They should not drive `App` back toward ingress-specific CRDs.

## Target Platform Coverage

The target API layer should absorb the repeated application patterns found in
the `cc-*` kustomize repos. For `App`, the preferred internal organization is:

- `workload`: runtime behavior, scaling, resilience, placement, and rollout
  controls
- `config`: application configuration, secret projection, and dependency wiring
- `network`: service exposure and HTTP routing
- `storage`: deployment-shaped persistence only

This organization is preferred over a flat field set because it keeps `App`
coherent as the contract grows.

The target API layer should not attempt to collapse every workload into a
single API if the contracts become incoherent. Where needed, the platform
should add new first-class APIs instead of overloading `App`.

Candidate future APIs if the current estate requires them:

- `StatefulApp` for stateful application workloads
- `ScheduledTask` for recurring `CronJob`-style workloads
- a dedicated non-HTTP exposure API for `TCPRoute` or `TLSRoute`

## Consumption Model

The intended model is:

1. `platform-kro` publishes the RGDs and example instances
2. a consumer repo vendors or references the RGDs
3. the consumer repo installs KRO and required operators
4. platform or environment manifests instantiate `DatabaseCluster` and
   `CacheCluster` where needed
5. application-facing manifests instantiate `App` objects that bind to those
   backing-service APIs and expose HTTP services through `HTTPRoute`
6. application-facing manifests may also instantiate `AppDelivery` objects
   that declare CI and promotion intent for those workloads
7. workloads that do not fit the current retained APIs stay on raw manifests
   until a coherent platform API exists for them

## Immediate Next Steps

- keep the current RGD shape transitional while the normalized `App` structure
  is refined in docs
- continue expanding `App` around the Wave 1 HTTP workloads
- keep `DatabaseCluster` PostgreSQL-focused in the current wave
- keep `CacheCluster` generic and evaluate Dragonfly behind that contract
- keep non-HTTP ingress patterns out of `App`
- introduce new APIs only where the workload class is genuinely different
