# `App` V2 Contract

## Purpose

This document defines the next contract shape for `App` based on the Wave 1
HTTP workloads identified in the `cc-*` kustomize analysis.

The design goal is to expand `App` without turning it into an unstructured
mirror of raw Kubernetes YAML. The contract should remain additive,
migration-friendly, and understandable as an application API.

## Design Principles

- keep the existing top-level API stable while the contract is still moving
- normalize the target shape around concern buckets rather than a flat field set
- keep `HTTPRoute` as the only built-in HTTP exposure primitive
- keep stateful and non-HTTP workloads out of `App` unless the contract remains
  coherent
- split future workload classes into new APIs instead of overloading `App`
- use Kubernetes-style concern grouping internally without exposing a menu of
  fragmented top-level APIs to users
- stamp the recommended Kubernetes `app.kubernetes.io/*` labels consistently
  across App-managed resources

## Target Shape

The target `App` shape should be organized around four concern buckets:

- `workload`
- `config`
- `network`
- `storage`

This is the preferred long-term shape because it matches how application
operators reason about the resources while still preserving `App` as a single
application-facing API.

## Proposed Spec Skeleton

```yaml
spec:
  name: payment
  namespace: platform-demo
  component: api
  partOf: astroshop
  version: 2.1.3-payment

  workload:
    image: ghcr.io/example/payment:1.0.0
    replicas: 2
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 500m
        memory: 512Mi
    imagePullSecrets: []
    imagePullPolicy: IfNotPresent
    serviceAccountName: payment
    deploymentAnnotations: {}
    podAnnotations: {}
    nodeSelector: {}
    probes:
      readiness: {}
      liveness: {}
    autoscaling:
      enabled: false
    disruptionBudget:
      enabled: false

  config:
    data: {}
    env: []
    envFrom: []
    externalSecrets: []
    dependencies:
      database:
        enabled: true
        databaseClusterName: payment-db
      cache:
        enabled: true
        cacheClusterName: cart-cache

  network:
    service:
      type: ClusterIP
      port: 8080
      annotations: {}
    httpRoute:
      enabled: false
      gatewayName: ""
      gatewayNamespace: ""
      hostnames: []
      pathPrefix: /
    certificate:
      enabled: false

  storage:
    volumes: []
```

This is a target shape, not a requirement that the current RGD adopt every
field immediately.

## Recommended Labels

`App` should stamp the Kubernetes recommended application labels on the
resources it manages.

Always present:

- `app.kubernetes.io/name`
- `app.kubernetes.io/instance`
- `app.kubernetes.io/managed-by`

Present when provided in the `App` spec:

- `app.kubernetes.io/component`
- `app.kubernetes.io/part-of`
- `app.kubernetes.io/version`

Selector stability matters. The stable selector set should use only:

- `app.kubernetes.io/name`
- `app.kubernetes.io/instance`

The other recommended labels should be stamped on resources, but not used as
selector keys.

## Why This Shape Is Better

### Better Than A Flat Spec

A flat spec becomes hard to navigate once `App` covers real workloads. Fields
for pods, routes, secrets, autoscaling, and persistence start to mix together
without a clear model.

### Better Than Splitting Into Many App-Adjacent APIs

The buckets should stay inside `App` rather than becoming separate top-level
APIs such as `AppNetwork` or `AppStorage`.

That would be a poor split because:

- one logical application would require multiple objects
- users would have to understand composition for ordinary app deployment
- the platform would start to feel like raw Kubernetes grouped by sidebar menu
  instead of an application abstraction

### Still Compatible With Future API Splits

This concern-bucket shape does not block separate future APIs where the
workload class is genuinely different.

Good future top-level API splits remain:

- `StatefulApp`
- `ScheduledTask`
- non-HTTP route APIs using `TCPRoute` or `TLSRoute`
- composition APIs for multi-workload applications if the need becomes real

## Bucket Definitions

### `workload`

This bucket owns the runtime and rollout behavior of the application workload.

Candidate contents:

- image
- replicas
- resource requests and limits
- image pull secrets and pull policy
- deployment and pod annotations
- service account selection
- node selector
- affinity
- topology spread constraints
- pod and container security context
- probes
- autoscaling
- disruption budget

### `config`

This bucket owns application configuration and dependency binding.

Candidate contents:

- config map style data
- secret projections
- external secret references
- dependency references to `DatabaseCluster` and `CacheCluster`

### `network`

This bucket owns app-facing networking.

Candidate contents:

- service shape, annotations, and service port
- `HTTPRoute`
- optional certificate wiring for the HTTP route path
- future network policy hooks if they become common enough

This bucket should not include:

- `Ingress`
- `IngressRoute`
- `IngressRouteTCP`

### `storage`

This bucket owns deployment-shaped persistence only.

Candidate contents:

- PVC-backed mounts for application data where the workload still fits a
  deployment-shaped API
- storage class and access mode controls

This bucket should not be stretched to cover a clearly stateful application
contract. That is where `StatefulApp` should begin.

## Transitional State In This Repo

The current `App` RGD is a transitional implementation, not the final target
shape.

Today it still exposes older top-level blocks such as:

- `image`
- `port`
- `replicas`
- `dependencies`
- `config`
- `autoscaling`
- `route`
- `externalSecret`
- `certificate`

It also includes additive transitional blocks:

- `runtime`
- `service`
- `availability`

Those transitional blocks are useful because they let the contract grow without
an immediate breaking schema rewrite.

## Implemented Transitional Field Sets

### Already Implemented

The current transitional RGD already implements:

- `runtime.imagePullSecrets`
- `runtime.nodeSelector`
- `runtime.resources`
- `service.type`

### Implemented In This Next Transitional Round

This repo revision adds another safe additive field set:

- `runtime.imagePullPolicy`
- `runtime.serviceAccountName`
- `runtime.deploymentAnnotations`
- `runtime.podAnnotations`
- `service.annotations`
- top-level `component`
- top-level `partOf`
- top-level `version`
- recommended Kubernetes app labels stamped across managed resources

These fields were chosen because they are common in the scanned HTTP workloads
and fit the current deployment-shaped `App` without introducing awkward merge
semantics or new CRD dependencies.

This transitional round keeps probe design at the documentation level only.

A live cluster verification on 2026-03-24 against KRO `v0.8.5` showed that the
first null-based conditional probe rendering pattern is not accepted by KRO
static validation. The attempted expression produced a `map` or `null` branch,
which KRO rejected for `Deployment` probe fields during `ResourceGraphDefinition`
validation.

Explicit `env` and `envFrom` projection is now implemented. The current `App`
contract emits a generated config-map `envFrom` entry, supports additive
`config.envFrom` sources, and allows `config.env` entries with literal `value`,
`valueFrom.fieldRef`, `valueFrom.secretKeyRef`, and
`valueFrom.configMapKeyRef`. `config.data` still defaults to an empty map so
`App` can materialize when no config literals are supplied. `config.revision`
is available as an explicit rollout token and is stamped onto the pod template
so overlays can trigger deployment rollout without depending on generated
config-map name changes. The intended Kustomize pattern is to patch
`config.data` normally in overlays and patch `config.revision` only when the
config change must force a rollout. Automatic database/cache environment-
variable injection remains deferred because the first cross-resource
interpolation pattern did not produce a reliable `Deployment` in live KRO
verification.

### Deliberately Deferred Again

These fields are still deferred to a later transitional round:

- executable probe support in the RGD
- executable `PodDisruptionBudget` support in the RGD
- observability CRDs such as `ServiceMonitor`
- deployment-shaped persistence

They remain deferred because they add external CRD coupling, require cleaner
merge semantics with the current generated environment wiring, still need a
cluster-validated rendering pattern, or depend on controller RBAC that is not
yet guaranteed in the target cluster baseline.

## Compatibility Mapping

The current schema should evolve toward the target shape using a deliberate
mapping.

| Current field | Target bucket | Target field |
| --- | --- | --- |
| `component` | `workload` or shared app identity | to be decided during normalization |
| `partOf` | shared app identity | to be decided during normalization |
| `version` | shared app identity | to be decided during normalization |
| `image` | `workload` | `workload.image` |
| `replicas` | `workload` | `workload.replicas` |
| `runtime.resources` | `workload` | `workload.resources` |
| `runtime.imagePullSecrets` | `workload` | `workload.imagePullSecrets` |
| `runtime.imagePullPolicy` | `workload` | `workload.imagePullPolicy` |
| `runtime.serviceAccountName` | `workload` | `workload.serviceAccountName` |
| `runtime.deploymentAnnotations` | `workload` | `workload.deploymentAnnotations` |
| `runtime.podAnnotations` | `workload` | `workload.podAnnotations` |
| `runtime.nodeSelector` | `workload` | `workload.nodeSelector` |
| `autoscaling` | `workload` | `workload.autoscaling` |
| `config.data` | `config` | `config.data` |
| `config.env` | `config` | `config.env` |
| `config.envFrom` | `config` | `config.envFrom` |
| `config.revision` | `config` | `config.revision` |
| `dependencies` | `config` | `config.dependencies` |
| `externalSecret` | `config` | `config.externalSecrets` or `config.externalSecret` |
| `port` | `network` | `network.service.port` |
| `service.type` | `network` | `network.service.type` |
| `service.annotations` | `network` | `network.service.annotations` |
| `route` | `network` | `network.httpRoute` |
| `certificate` | `network` | `network.certificate` |

This mapping should guide the next contract refactor.

## What To Implement Next

The next `App` work should follow this order:

1. continue adding fields conservatively in the transitional schema
2. group those additions conceptually under `workload`, `config`, `network`,
   and `storage` in docs and examples
3. only perform a breaking schema move to the normalized shape once the field
   set is stable enough to justify it

The next likely transitional round after this one should focus on cluster-validated designs for probes and
disruption budgets.

## Mapping To The `cc-*` Repos

Wave 1 repos that should shape `App` next:

- `cc-epg-service`
- `cc-kis-federated-service`
- `cc-accounting-middleware`

Wave 2 repos that should extend the contract after that:

- `cc-backend-api`
- `cc-infotainment-pin-service`
- `cc-patient-request-service`
- `cc-purchase-manager`

These repos should not be used to force awkward `App` fields:

- `cc-e2e-identity-management`
- `cc-limesurvey-cfg`
- `cc-mirth-connect-cfg`
- `cc-shop`
- `cc-third-party-token-service`
