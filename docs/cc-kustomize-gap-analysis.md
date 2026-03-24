# `cc-*` Kustomize Gap Analysis

## Purpose

This document compares the current `platform-kro` API surface to the consumer
workload patterns already present in the `cc-*` repositories that contain
`kustomize/` folders.

The goal is not to recreate every existing manifest one-to-one. The goal is to
identify:

- which repeated patterns should move into platform APIs
- which repos are good first migration candidates
- which workload classes still require raw manifests or new APIs

This analysis reflects the platform direction agreed on 2026-03-24:

- `platform-kro` should become a fuller app platform API layer
- HTTP exposure should standardize on Gateway API `HTTPRoute`
- the platform API layer should not introduce `Ingress`, `IngressRoute`, or
  `IngressRouteTCP`

## Scope

Twelve sibling `cc-*` repos currently contain `kustomize/` folders and are in
scope for this analysis:

- `cc-accounting-middleware`
- `cc-backend-api`
- `cc-e2e-identity-management`
- `cc-epg-service`
- `cc-infotainment-pin-service`
- `cc-kis-federated-service`
- `cc-limesurvey-cfg`
- `cc-mirth-connect-cfg`
- `cc-patient-request-service`
- `cc-purchase-manager`
- `cc-shop`
- `cc-third-party-token-service`

Fourteen sibling `cc-*` repos currently do not expose `kustomize/` folders and
are out of scope for this document.

## Current `platform-kro` Coverage

Today `platform-kro` publishes only three retained APIs:

- `App`
- `DatabaseCluster`
- `CacheCluster`

Current coverage is still narrow:

- `App` creates a single `Deployment`, `Service`, `ServiceAccount`,
  `ConfigMap`, and selected optional resources such as `HTTPRoute`,
  `ExternalSecret`, `HorizontalPodAutoscaler`, and `Certificate`
- `DatabaseCluster` is currently a thin CloudNativePG wrapper
- `CacheCluster` is currently a simple StatefulSet-backed cache wrapper

That is enough for a demo slice, but not enough to absorb most of the existing
`cc-*` kustomize estate.

## Common Patterns Found In The `cc-*` Repos

The current `cc-*` kustomize repos repeatedly use these workload patterns:

- `Deployment` plus `Service`
- environment-specific overlays
- generated `ConfigMap` values
- `ExternalSecret`, `SealedSecret`, and raw `Secret` usage
- PostgreSQL `Cluster` resources plus backup objects
- cache operator resources such as Dragonfly
- `PodDisruptionBudget`
- `CronJob`
- observability resources such as `ServiceMonitor` or `EndpointMonitor`
- PVC-backed applications
- multi-workload applications
- stateful application workloads
- Traefik-specific ingress resources and legacy `Ingress`

These patterns are the migration corpus that should drive API design.

## Core Gaps

### 1. `App` Is Too Thin For The Existing HTTP Workloads

The current `App` contract still leaves several repeated workload concerns
outside the executable platform API:

- startup probes and non-HTTP probe types beyond the current HTTP readiness/liveness model
- richer pod and container security context support beyond the current restricted preset
- richer pod placement controls such as full affinity policies beyond the current validated spread preset
- broader observability bindings beyond the current `ServiceMonitor` path
- optional PVC attachments for deployment-shaped workloads
- broader secret and config composition beyond the current `config.data`, generated config-map `envFrom`, and explicit `config.env` model

Without those features, most HTTP services still need large raw deployment
manifests beside the platform API.

### 2. Exposure Must Move To `HTTPRoute`, But Not Every Existing Workload Is HTTP

For HTTP services, the direction is clear:

- stop modeling `Ingress`
- stop modeling Traefik `IngressRoute`
- migrate to `HTTPRoute`

However, some repos currently expose non-HTTP traffic through
`IngressRouteTCP`. Those cannot be translated into `HTTPRoute`.

The practical consequence is:

- `App` should standardize on `HTTPRoute` for HTTP workloads
- non-HTTP traffic needs a separate future API, likely based on Gateway API
  `TCPRoute` or `TLSRoute`
- those workloads should stay out of the first `App` migration wave

### 3. The Current Data-Service APIs Do Not Yet Cover The Existing Estate

Current repos need more than the present first-pass abstractions:

- PostgreSQL is used with operator-specific backup and restore flows
- MariaDB exists in the repo set
- Dragonfly exists as a cache/data-store pattern

`DatabaseCluster` and `CacheCluster` should grow only where the resulting
contracts remain coherent. If they cannot stay coherent, the platform should
add dedicated APIs instead of hiding incompatible behavior behind vague fields.

### 4. Stateful And Multi-Workload Applications Need Their Own Story

Some repos do not fit a single deployment-shaped `App`:

- a repo may contain multiple tightly coupled deployments
- a repo may require application-level persistent volumes
- a repo may require a real `StatefulSet`

Those cases likely need one of:

- a richer composition model above `App`
- a dedicated `StatefulApp` API
- temporary retention of raw manifests until the contract is clear

### 5. Batch And Operational Attachments Need Explicit Treatment

Several repos include recurring jobs, one-off backup resources, and monitoring
attachments. Those should not be left ambiguous.

The platform needs an explicit position on:

- `CronJob`-style recurring tasks
- backup and restore objects
- `ServiceMonitor` and `EndpointMonitor`
- namespace creation and environment bootstrapping boundaries

## Repo Matrix

| Repo | Current profile | Fit with current APIs | Main gaps before migration |
| --- | --- | --- | --- |
| `cc-accounting-middleware` | Simple HTTP service with PVC, `ExternalSecret`, `IngressRoute` | Low to medium | `App` needs PVC attachments, richer runtime options, and `HTTPRoute` migration |
| `cc-backend-api` | HTTP service with PostgreSQL, `ExternalSecret`, `CronJob`, PDB, backups | Low | `App` needs richer deployment controls; platform needs clear PostgreSQL backup model and recurring task story |
| `cc-e2e-identity-management` | Namespace plus `CronJob` overlays | Low | likely needs a dedicated `ScheduledTask`-style API rather than `App` |
| `cc-epg-service` | HTTP service with `ExternalSecret`, monitoring, PDB | Medium | `App` needs `HTTPRoute` migration and validation against the actual monitoring shape |
| `cc-infotainment-pin-service` | HTTP service with PostgreSQL plus backup and restore flows | Low | same as `cc-backend-api`, plus clearer restore and operational workflow boundaries |
| `cc-kis-federated-service` | HTTP service with mixed secret patterns and monitoring | Medium | `App` needs secret model normalization, `HTTPRoute` migration, and validation against the actual monitoring shape |
| `cc-limesurvey-cfg` | App plus MariaDB, PVC, backup, DB user, `IngressRouteTCP` | Very low | current APIs do not cover MariaDB shape, stateful storage, or TCP exposure |
| `cc-mirth-connect-cfg` | Stateful application with PVC and raw secrets | Very low | likely needs `StatefulApp` or raw manifests until a stateful API exists |
| `cc-patient-request-service` | HTTP service with PostgreSQL, Dragonfly, monitoring, backups, `IngressRoute` | Low | platform needs richer `App`, clearer cache contract, observability support, and `HTTPRoute` migration |
| `cc-purchase-manager` | HTTP service with PostgreSQL, `CronJob`, backups, PDB | Low | same work as `cc-backend-api` plus recurring-task support |
| `cc-shop` | Multi-workload app with two deployments, PVs, PVCs, and legacy ingress | Very low | needs composition above `App`, persistent volume support, and `HTTPRoute` migration |
| `cc-third-party-token-service` | TCP service with `ExternalSecret`, PDB, `IngressRouteTCP` | Very low | cannot move to `App` plus `HTTPRoute`; needs a non-HTTP route API or temporary exclusion |

## Recommended First Migration Waves

### Wave 1: Best Candidates For `App` Expansion

These repos look like the best first design corpus for `App`:

- `cc-epg-service`
- `cc-kis-federated-service`
- `cc-accounting-middleware`

Why:

- they are primarily application workloads
- they are closer to the single-service HTTP model
- they force useful expansion of `App` without immediately requiring
  stateful-app or multi-workload composition support

### Wave 2: HTTP Apps With Backing Services And Batch Attachments

These repos should shape the second round of API design:

- `cc-backend-api`
- `cc-infotainment-pin-service`
- `cc-patient-request-service`
- `cc-purchase-manager`

Why:

- they require `App` plus stronger database and operational contracts
- they will force decisions about backups, recurring tasks, and monitoring

### Wave 3: Repos That Likely Need New APIs

These repos should not be used to force awkward fields into `App`:

- `cc-e2e-identity-management`
- `cc-limesurvey-cfg`
- `cc-mirth-connect-cfg`
- `cc-shop`
- `cc-third-party-token-service`

Why:

- they contain workload shapes that are not naturally a single HTTP app
- some require stateful APIs
- some require non-HTTP routing
- some are better modeled as composition or dedicated task APIs

## Proposed Design Workstreams

### Workstream 1: `App` V2 For HTTP Workloads

Expand `App` to cover the common HTTP workload contract:

- richer container spec
- probes
- resources
- image pull secrets
- richer pod placement controls
- env and secret projection
- optional monitoring attachment beyond the current `ServiceMonitor` path
- `HTTPRoute` as the only built-in HTTP exposure primitive

### Workstream 2: Data-Service Contract Decisions

Make explicit decisions on:

- whether MariaDB extends `DatabaseCluster` or becomes a separate API
- whether Dragonfly extends `CacheCluster` or becomes a separate API
- which backup and restore responsibilities belong in platform APIs versus
  operator-native manifests

### Workstream 3: New API Candidates

Evaluate whether the repo set justifies:

- `StatefulApp`
- `ScheduledTask`
- non-HTTP route APIs based on `TCPRoute` or `TLSRoute`
- a composition API for multi-workload applications

## Recommended Immediate Actions

1. Expand the `App` design around the Wave 1 repos.
2. Keep `HTTPRoute` as the only built-in HTTP exposure primitive.
3. Exclude `Ingress`, `IngressRoute`, and `IngressRouteTCP` from the target
   platform API layer.
4. Treat TCP-exposed repos as a separate design track, not as exceptions that
   weaken `App`.
5. Decide the database and cache API boundaries before trying to migrate the
   heavier repos.
