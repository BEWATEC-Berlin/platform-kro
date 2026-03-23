# App

## Scope

`App` is the generic workload API for the PoC.

It is intentionally broader than a plain deployment wrapper. It can own the
common runtime wiring that would otherwise repeat across services.

It should sit above backing-service APIs rather than force each example to
define databases or caches inline.

## Responsibilities

- `Deployment`
- `Service`
- `ServiceAccount`
- `ConfigMap`
- dependency references to `DatabaseCluster` and `CacheCluster` through
  `externalRef`
- optional `HorizontalPodAutoscaler`
- optional `HTTPRoute`
- optional `ExternalSecret`
- optional `Certificate`

## Boundaries

- no `Ingress` in the first pass
- no dedicated `Gateway` per app by default
- no cluster-wide issuer or secret-store management
- no attempt to model every app-specific setting in the first version
- backing services should usually be instantiated outside the app example

## Binding Model

The first pass assumes backing-service APIs exist in the same namespace as the
`App`.

When enabled, `App` binds to:

- `DatabaseCluster` for service names and secret names
- `CacheCluster` for service and endpoint projection

The deployment receives standardized environment variables for:

- database cluster and service names
- database name and secret name
- database username, password, and URI from the referenced secret
- cache cluster, service, endpoint, and port
