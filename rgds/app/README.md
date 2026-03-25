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
User-facing guides for actual usage live in:

- `docs/app-user-guide.md`
- `docs/app-config-and-secrets.md`
- `docs/app-migration-cookbook.md`
- `docs/cc-epg-service-mapping.md`
- `docs/wave-planning.md`

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
- `runtime.priorityClassName`
- `runtime.tolerations`
- `runtime.topologySpread`
- `runtime.readinessProbe`
- `runtime.livenessProbe`
- `runtime.restrictedSecurity`
- `runtime.resources`
- `service.type`
- `service.annotations`
- `serviceMonitor.path`
- `serviceMonitor.interval`
- `storage.mountPath`
- `storage.size`
- `storage.className`
- `storage.volumeMode`
- `disruptionBudget.maxUnavailableCount`

Planned next responsibilities should be reasoned about in these buckets:

- `workload`: richer runtime controls, scaling, resilience, and
  cluster-validated HTTP probe models and disruption budgets
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
- `config.envFromConfigMaps` for explicit referenced config maps
- `config.envFromSecrets` for explicit referenced secrets

`config.env[].valueFrom.resourceFieldRef` is supported without `divisor`. The
live KRO validation path in this cluster rejected `divisor` as not structurally
compatible with the expected Deployment env schema.

`runtime.priorityClassName`, `runtime.tolerations`, and
`runtime.topologySpread` give `App` a first usable placement story for HTTP
workloads. `runtime.restrictedSecurity` adds a first cluster-validated
restricted-pod-security preset for workloads that can run non-root.

`runtime.topologySpread` is a validated generated preset rather than a full raw
Kubernetes `topologySpreadConstraints` passthrough. Full affinity-style
placement remains future design work.

Whole-object pod and container `securityContext` support is intentionally not
in the executable `App` contract yet. A live KRO validation pass on 2026-03-24
rejected the attempted optional-object interpolation for the generated
`Deployment` security-context fields, which left the `App`
`ResourceGraphDefinition` inactive. That richer passthrough capability remains
deferred until a cluster-valid rendering pattern is proven.

`runtime.restrictedSecurity` is the narrower supported path. When enabled, it
renders a fixed container `securityContext` with:

- `allowPrivilegeEscalation: false`
- `capabilities.drop: ["ALL"]`
- `runAsNonRoot: true`
- `seccompProfile.type: RuntimeDefault`
- optional `readOnlyRootFilesystem` from the App spec

This was verified live on 2026-03-24 in the development cluster with a
non-root image. It satisfies the restricted Kyverno policy that previously
blocked the test workload.

`runtime.readinessProbe` and `runtime.livenessProbe` are the supported probe
path in the current `App` contract. They render HTTP GET probes against the
container's named `http` port, with the probe path and timing values owned by
the `App` spec. Startup probes, gRPC probes, exec probes, and raw Kubernetes
probe passthrough remain deferred.

`disruptionBudget` is the current disruption-budget path. It creates an optional
`policy/v1` `PodDisruptionBudget` that selects the `App` pods by the stable
`app.kubernetes.io/name` and `app.kubernetes.io/instance` labels. The current
contract intentionally stays narrow and exposes only integer `maxUnavailableCount`. The older string `maxUnavailable` field remains in the schema for compatibility with earlier revisions, but the integer count field is the supported path.

`serviceMonitor` is the current observability path. It creates an optional
`monitoring.coreos.com/v1` `ServiceMonitor` that selects the `App` service by
the same stable labels and scrapes the named `http` port. The current contract
intentionally stays narrow and exposes only `path`, `interval`, and
`scrapeTimeout`.

`storage` is the current deployment-shaped persistence path. It creates one
optional `PersistentVolumeClaim` and mounts it into the application container.
The current contract intentionally stays narrow and exposes `size`,
`mountPath`, one `accessMode`, explicit `className`, and explicit `volumeMode`.

The validated path in the development cluster uses an explicit storage class
selection instead of relying on admission defaulting. That matters because the
default storage-class admission plugin mutates the claim spec, which leaves the
`App` stuck in `ResourcesReady=False` with `cluster mutated`.

This is enough for the first Wave 1 PVC case, but it is not yet a general
volume model. Existing claims, multiple mounts, secret volumes, and empty-dir
patterns remain outside the current `App` contract. The main operational
failure modes are a missing or wrong `storage.className`, container images that
cannot write to the mounted path without extra ownership handling, and stale
`App` instances created before the single-deployment graph change. Those older
instances may need to be recreated so KRO can manage a clean resource graph.

`config.revision` is available as an explicit rollout token. It is stamped onto
the pod template as `platform.connectedcare.io/config-revision`, so overlays can
force a rollout by patching one short field without depending on Kustomize
`configMapGenerator` name hashing.

Generic additive `config.envFrom` support is still deferred. A live KRO
validation pass on 2026-03-24 rejected the merged generic `EnvFromSource`
shape needed to combine user refs with the generated config-map `envFrom`
entry while keeping the resource graph active.

The supported normalization path is narrower:

- `config.envFromConfigMaps` appends explicit config-map refs
- `config.envFromSecrets` appends explicit secret refs

That keeps the contract additive and avoids the unsupported generic `prefix`
and union typing path.

The intended pattern is:

- patch `spec.config.data` in overlays for environment-specific values
- patch `spec.config.revision` when that config change must force a rollout

A concrete example is included under
`examples/otmicro-currency-spike/overlays/dev`.
