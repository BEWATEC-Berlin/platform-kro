# `App` User Guide

## Purpose

This guide explains how to use `App` as the primary HTTP workload API in
`platform-kro`.

It is written for platform consumers, not for KRO implementation work.

## When To Use `App`

Use `App` when all of the following are true:

- the workload is a single HTTP application
- the main runtime shape is one `Deployment`
- exposure should use Gateway API `HTTPRoute`
- storage, if any, fits a single generated PVC mount

Do not use `App` when any of the following are true:

- the workload is primarily stateful and should be a `StatefulSet`
- the workload exposes non-HTTP traffic
- the repo needs multiple first-class workloads as one application contract
- the storage model needs multiple claims, existing claims, or non-PVC volumes

## What `App` Creates

Always created:

- `ServiceAccount`
- `ConfigMap`
- `Deployment`
- `Service`

Optionally created:

- `PersistentVolumeClaim`
- `PodDisruptionBudget`
- `ServiceMonitor`
- `HorizontalPodAutoscaler`
- `HTTPRoute`
- `ExternalSecret`
- `Certificate`

## Minimum Example

```yaml
apiVersion: platform.connectedcare.io/v1alpha1
kind: App
metadata:
  name: demo
  namespace: demo
spec:
  name: demo
  namespace: demo
  image: ghcr.io/example/demo:1.0.0
  port: 8080
```

## Common Fields

Identity:

- `name`
- `namespace`
- `component`
- `partOf`
- `version`

Runtime:

- `image`
- `replicas`
- `runtime.resources`
- `runtime.imagePullPolicy`
- `runtime.imagePullSecrets`
- `runtime.serviceAccountName`
- `runtime.nodeSelector`
- `runtime.priorityClassName`
- `runtime.tolerations`
- `runtime.topologySpread`
- `runtime.readinessProbe`
- `runtime.livenessProbe`
- `runtime.restrictedSecurity`

Config:

- `config.data`
- `config.revision`
- `config.env`
- `config.envFromConfigMaps`
- `config.envFromSecrets`

Networking:

- `service.type`
- `service.annotations`
- `route.*`
- `serviceMonitor.*`

Storage:

- `storage.enabled`
- `storage.size`
- `storage.mountPath`
- `storage.accessMode`
- `storage.className`
- `storage.volumeMode`

Availability:

- `disruptionBudget.enabled`
- `disruptionBudget.maxUnavailableCount`
- `autoscaling.*`

## Overlay Pattern

The supported overlay pattern is:

1. patch `spec.config.data` for environment-specific config
2. patch `spec.config.env` for explicit env values
3. patch `spec.config.revision` when the config change must force rollout
4. patch runtime and route fields per environment as needed

See:

- `examples/otmicro-currency-spike/overlays/dev`
- `examples/restricted-nginx-spike`

## Validated Behaviors

Validated live in the development cluster as of 2026-03-25:

- explicit env and env-from projection
- restricted container security preset
- HTTP readiness and liveness probes
- `ServiceMonitor`
- `PodDisruptionBudget`
- single PVC mount with explicit storage class and volume mode
- rollout forcing via `config.revision`

## Known Limits

Still intentionally deferred:

- raw pod or container `securityContext` passthrough
- startup probes
- gRPC probes
- exec probes
- generic raw `envFrom` union shape
- multiple PVCs or existing-claim mounting
- `emptyDir`, secret, projected, and config volumes
- `Ingress`, `IngressRoute`, and `IngressRouteTCP`
- non-HTTP routing

## Where To Look Next

- `rgds/app/README.md`
- `docs/app-config-and-secrets.md`
- `docs/app-migration-cookbook.md`
- `docs/app-v2-contract.md`
