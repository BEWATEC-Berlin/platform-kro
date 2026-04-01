# AppDelivery

## Scope

`AppDelivery` is the developer-facing CI and promotion-intent contract for one
application workload.

It exists to keep delivery concerns out of raw Tekton, Kargo, Flux, and
BuildKit manifests while still giving platform consumers a stable API to
declare:

- where source code lives
- which shared pipeline class should be used
- which image repository should receive published artifacts
- which cache transport defaults should apply
- whether promotion is enabled and where it starts
- which rollout strategy is intended downstream

## What It Creates

Today the executable contract is intentionally conservative.

`AppDelivery` creates one generated `ConfigMap`:

- `<spec.name>-delivery-contract`

That `ConfigMap` materializes the stable scalar fields of the delivery contract
for consumer repos and operators. The full contract remains on the
`AppDelivery` resource itself.

Boolean and structured fields remain on the `AppDelivery` resource itself. The
generated `ConfigMap` is intentionally limited to scalar string data that can
be materialized safely by the first executable RGD shape.

The status surface currently exposes only:

- `contractConfigMapName`

## Boundaries

`AppDelivery` is not a CI engine and not a mutable delivery-state store.

It does not create:

- Tekton `Pipeline`, `PipelineRun`, `Trigger`, or `EventListener` resources
- Kargo `Project`, `Stage`, or promotion resources
- Flux `Kustomization` resources
- mutable `release.yaml` state per environment
- per-run `PromotionArtifact` or `PromotionRecord` history

Those remain consumer-repo responsibilities.

This split is intentional:

- `platform-kro` owns the developer-facing API
- consumer repos such as `hetzner_playground` own executable control-plane
  integration
- a later dedicated delivery repo can own mutable promotion state without
  changing the app-facing contract

## Current Contract Shape

The first executable `AppDelivery` contract exposes these buckets:

- `appRef`
- `repo`
- `pipelineClass`
- `registry`
- `quality`
- `publish`
- `build`
- `promotion`
- `rollout`
- `requiredSecrets`

The contract is intentionally smaller than the raw platform implementation.

For example, `AppDelivery` records:

- `pipelineClass`
- `dockerfile`
- a migration-time `dockerfileAsset` when the build input is not the repo default
- a Node `lintCommand` when the repo does not expose the conventional entrypoint
- `publishQualitySource`
- cache import/export defaults
- first real deployment stage

It does not expose:

- Tekton task wiring
- BuildKit daemon addresses
- Kargo stage internals
- Flux path layout

## Deferred

These capabilities are intentionally deferred until a live consumer path proves
they belong in the developer-facing API:

- direct pipeline trigger policy per Git event
- detailed Sonar event lists and coverage exemptions per branch class
- promotion approvals and Kargo-specific stage internals
- mutable delivery state or environment digests
- executable rollout verification details beyond the coarse rollout strategy

## Consumer Model

The intended model is:

1. developers declare `AppDelivery`
2. a consumer repo renders or reconciles its CI and promotion plumbing from
   that contract
3. CI emits immutable build evidence
4. a separate delivery actor moves mutable promotion state across environments

That keeps the API surface stable even if the implementation changes from a
local writer to Kargo later.
