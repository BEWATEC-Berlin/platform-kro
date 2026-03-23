# Architecture

## Purpose

`platform-kro` owns a small set of application-facing platform APIs built with
KRO. It does not own cluster bootstrap.

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

## API Set

### `DatabaseCluster`

- wraps the PostgreSQL operator contract behind a smaller stable API
- should remain generic enough to allow a later MariaDB-backed implementation
- only PostgreSQL is implemented in the first pass

### `CacheCluster`

- starts as a plain StatefulSet-based cache contract
- can later swap to an operator-backed implementation without changing the
  consumer-facing API

### `App`

- sits one abstraction layer above backing-service APIs such as
  `DatabaseCluster` and `CacheCluster`
- owns common workload wiring for runtime and exposure concerns
- should stay smaller than a full application platform
- may create optional `HTTPRoute`, `ExternalSecret`, and `Certificate`
- should not create dedicated `Gateway` resources by default

## Consumption Model

The intended model is:

1. `platform-kro` publishes the RGDs and example instances
2. a consumer repo vendors or references the RGDs
3. the consumer repo installs KRO and required operators
4. platform or environment manifests instantiate `DatabaseCluster` and
   `CacheCluster` where needed
5. application-facing manifests instantiate `App` objects that bind to those
   backing-service APIs

## Immediate Next Steps

- validate the draft RGDs against a cluster with KRO installed
- tighten the API contracts based on the first consumer slice
- decide how `hetzner_playground` should consume this repo
