# `cc-accounting-middleware` Paper Mapping

## Purpose

This document maps the current `cc-accounting-middleware` Kustomize setup to
the current `App` contract without doing a live migration yet.

It is the second Wave 1 paper migration template.

## Current Resource Shape

Current Kustomize resources:

- one `Deployment`
- one `Service`
- one PVC
- one `ExternalSecret` that produces JWT key files
- one generated config map
- one Traefik `IngressRoute` in integration

## Current Runtime Contract

The deployment currently uses:

- one PVC mount at `/data`
- one `emptyDir` mount at `/tmp`
- one secret volume mount at `/keys`
- one env var from `fieldRef`: `POD_IP`
- one generated config map import via `envFrom`
- one HTTP app port `8080`
- one metrics port `9234`

The generated config-map values currently include:

- `SERVER_PORT`
- `SPRING_PROFILES_ACTIVE`
- `SPRING_BOOT_ADMIN_CLIENT_URL`
- `SPRING_BOOT_ADMIN_CLIENT_INSTANCE_SERVICE_URL`
- `SPRING_BOOT_ADMIN_CLIENT_INSTANCE_MANAGEMENT_BASE_URL`

The application properties also assume:

- JWT signing key at `/keys/jwt_private.key`
- JWT verification key at `/keys/jwt_public.key`
- Quarkus management port `9234`
- readiness and liveness endpoints under `/q/health/*`

## Direct Mapping To `App`

Deployment basics:

- image -> `spec.image`
- replicas -> `spec.replicas`
- resources -> `spec.runtime.resources`
- image pull secret `regcred` -> `spec.runtime.imagePullSecrets`
- node selector -> `spec.runtime.nodeSelector`
- topology spread -> `spec.runtime.topologySpread`
- `POD_IP` env -> `spec.config.env` with `fieldRef`
- generated config-map values -> `spec.config.data`
- readiness probe `/q/health/ready` on metrics port intent -> partial fit only
- liveness probe `/q/health/live` on metrics port intent -> partial fit only

Service:

- primary HTTP service port `8080` -> `spec.port`
- cluster service -> `spec.service.type`

Storage:

- one PVC at `/data` -> `spec.storage.*`

Secret generation:

- generated JWT secret object -> partial fit with `spec.externalSecret.*`

Exposure:

- integration ingress -> `spec.route.*` with `HTTPRoute`

## Current Gaps Against `App`

The current `App` contract does not yet express these parts cleanly:

- `emptyDir` mount at `/tmp`
- secret volume mount for JWT keys at `/keys`
- pod security context with `runAsUser`, `runAsGroup`, and `fsGroup`
- container security context beyond the current restricted preset
- health probes on the metrics port instead of the fixed named `http` port
- dual-port service shape where metrics is a first-class second service port
- service annotations for Prometheus and New Relic scraping

These are not small details. They are part of the runtime contract of this
service.

## Recommended Migration Stance

This repo should stay a paper mapping for now.

A first live migration spike should not happen until one of these is true:

1. `App` gains a deliberate model for secret and ephemeral volume mounts plus
   the required security context fields
2. the service is intentionally adapted to a narrower runtime model and that
   adaptation is accepted as a real product change

## Minimal Adapted `App` Shape

A narrowed migration candidate could still express:

- image
- replicas
- resources
- image pull secret
- node selector
- topology spread
- `POD_IP` fieldRef env
- generated config data
- PVC-backed `/data` storage
- `HTTPRoute` instead of Traefik ingress

But that would still leave these runtime elements outside the contract:

- `/tmp` `emptyDir`
- `/keys` secret volume
- exact current security context
- metrics-port probes and service shape

That is enough to say the repo is not yet a clean `App` migration target.

## Recommendation

Keep `cc-accounting-middleware` in the Wave 1 design corpus, but do not use it
as the first live migration spike.

Use it instead as the pressure test for future work on:

- volume model expansion beyond one PVC
- pod and container security context support
- multi-port service and probe support

## Sources

- `cc-accounting-middleware/kustomize/base/deployment.yaml`
- `cc-accounting-middleware/kustomize/base/kustomization.yaml`
- `cc-accounting-middleware/kustomize/base/pvc.yaml`
- `cc-accounting-middleware/kustomize/base/secrets.yaml`
- `cc-accounting-middleware/kustomize/overlays/integration/*`
- `cc-accounting-middleware/src/main/resources/application.properties`
