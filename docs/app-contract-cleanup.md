# `App` Contract Cleanup

## Purpose

This document records how the current transitional `App` contract should be
cleaned up before a future breaking normalization to
`workload / config / network / storage`.

It is a design note for repo consumers and maintainers. It is not executable
schema.

## Current Transitional Shape

Today the executable API still exposes:

- top-level identity and workload fields such as `image`, `port`, `replicas`
- `config`
- `runtime`
- `service`
- `storage`
- `serviceMonitor`
- `disruptionBudget`
- `autoscaling`
- `route`
- `externalSecret`
- `certificate`

That is acceptable while the field set is still settling.

## Fields Likely To Survive

These concepts should remain, even if their location changes:

- app identity fields
- config data and env projection
- runtime resources
- readiness and liveness probes
- restricted security preset
- service exposure
- `HTTPRoute`
- PVC-backed single-volume storage
- `ServiceMonitor`
- `PodDisruptionBudget`

## Fields Likely To Move

Target movement:

- `image`, `replicas`, `runtime.*`, `autoscaling`, `disruptionBudget`,
  `serviceMonitor` move under `workload`
- `config.*`, `externalSecret`, and dependency bindings move under `config`
- `service.*`, `route`, and `certificate` move under `network`
- `storage.*` stays under `storage` but should become a clearer subobject

## Fields That Need A Decision Before Normalization

- should `serviceMonitor` live under `workload.observability` or another
  nested name
- should `externalSecret` stay app-owned or become only a reference
- should `dependencies` remain top-level in `config` or be modeled as a
  separate block
- should the current restricted security preset remain the only supported
  security model in `App`

## Fields That Should Not Be Carried Forward Blindly

Avoid preserving transitional shapes just because they exist today:

- top-level flat runtime knobs that clearly belong together
- compatibility fields kept only for earlier schema revisions
- fields that imply broad raw Kubernetes compatibility when the live KRO path
  has already rejected that assumption

## Preconditions For A Breaking Cleanup

Do not normalize the executable schema yet unless all of the following are
true:

1. the current additive field set has been pressure-tested by real repo
   mappings
2. at least one real migration has been exercised live
3. the secret model is stable enough that consumers will not immediately need
   another breaking move
4. storage behavior is stable for the intended Wave 1 scope

## Practical Rule

Until those preconditions are met:

- keep the executable schema conservative
- keep additive changes only
- prefer better docs over premature breaking cleanup

That is the current posture of `platform-kro`.
