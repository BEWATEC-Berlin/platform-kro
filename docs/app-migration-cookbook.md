# `App` Migration Cookbook

## Purpose

This document explains how to migrate an existing Kustomize-managed HTTP
service into `App` without committing to a live rollout too early.

## Recommended Sequence

1. do a paper mapping first
2. identify what stays inside `App`
3. identify what stays outside `App`
4. identify any genuine API gaps
5. only then create a live migration spike

This avoids mixing contract design with cluster debugging.

## Quick Fit Check

Good `App` migration candidates:

- one main `Deployment`
- one HTTP `Service`
- one route
- mostly env/config/secret wiring
- at most one simple PVC mount

Bad `App` migration candidates:

- multiple tightly coupled workloads
- raw `StatefulSet`
- TCP exposure
- several volumes and claim types
- complex init-container and sidecar requirements

## Mapping Table

| Kustomize pattern | `App` field | Keep outside `App` when |
| --- | --- | --- |
| image override | `spec.image` | never |
| replica patch | `spec.replicas` | never |
| service type/annotations | `spec.service.*` | never |
| HTTP ingress intent | `spec.route.*` | route is not HTTP |
| config map generator literals | `spec.config.data` | config should stay outside app ownership |
| explicit env vars | `spec.config.env` | env needs unsupported raw probe or volume coupling |
| imported config maps | `spec.config.envFromConfigMaps` | config object lifecycle is outside the app |
| imported secrets | `spec.config.envFromSecrets` | secret lifecycle is outside the app |
| external secret generation | `spec.externalSecret.*` | secret generation should stay platform- or team-owned |
| PDB | `spec.disruptionBudget.*` | policy is more complex than current narrow contract |
| service monitor | `spec.serviceMonitor.*` | monitoring needs labels/endpoints beyond current contract |
| one PVC mount | `spec.storage.*` | storage needs more than one claim or non-PVC volume types |
| resource requests/limits | `spec.runtime.resources` | never |
| readiness/liveness HTTP probe | `spec.runtime.readinessProbe` / `livenessProbe` | probe is not HTTP |
| restricted pod-security shape | `spec.runtime.restrictedSecurity` | workload needs custom security context |

## Paper Mapping Checklist

For each repo:

1. list generated resources
2. mark which are direct app runtime concerns
3. map each app runtime concern to `App`
4. mark the remainder as:
   - separate platform API
   - raw manifest kept outside `App`
   - current gap

## Live Migration Checklist

Only do this after the paper mapping is stable.

1. create one example `App` manifest
2. render with `kubectl kustomize`
3. push `platform-kro` if the contract changed
4. reconcile Flux in the consumer cluster
5. create the example app in an isolated namespace
6. verify:
   - `App` `Ready=True`
   - deployment available
   - route or service behavior
   - config and secret projection
   - policy compliance
7. document what still had to stay outside `App`

## What Not To Do

Do not:

- force `Ingress` or Traefik resources into the `App` contract
- add raw Kubernetes passthrough fields just to satisfy one repo
- call a migration successful when large runtime pieces still sit beside `App`

## Current Best Paper Candidate

The best current paper migration candidate is `cc-epg-service`.

Why:

- HTTP service
- one deployment
- one service
- overlay-driven config
- observability and PDB already align with the current `App` surface

See `docs/cc-epg-service-mapping.md`.
