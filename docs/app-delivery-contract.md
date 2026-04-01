# `AppDelivery` Contract

## Purpose

`AppDelivery` is the application-facing delivery contract that sits beside
`App`.

It exists because delivery concerns are real platform concerns, but they should
not leak raw Tekton, Kargo, Flux, or BuildKit shapes into developer-facing
manifests.

The contract is intended to answer:

- where does the source live
- which shared pipeline class applies
- where is the image published
- what is the expected cache and publish policy
- which real deployment stage receives the first promoted release
- what rollout posture is intended downstream

## Why Separate From `App`

`App` remains the runtime-facing HTTP workload API.

`AppDelivery` is different in lifecycle and responsibility:

- `App` describes runtime shape
- `AppDelivery` describes CI and promotion intent

Keeping those separate avoids turning `App` into a single god object for both
runtime and delivery concerns.

## Boundaries

`AppDelivery` should stay above implementation details.

It should own:

- repository identity
- pipeline class selection
- image destination
- build-cache policy defaults
- publish policy defaults
- a small number of string-safe app-specific build inputs such as `dockerfileAsset`
- a small number of language-specific command exceptions such as a Node `lintCommand`
- promotion enablement and first stage
- coarse rollout policy
- required secret names

It should not own:

- Tekton task structure
- trigger internals
- BuildKit daemon configuration
- Kargo stage internals
- Flux path layout
- mutable promoted digests

## Current Executable Shape

The first executable `AppDelivery` RGD is intentionally small.

It creates one generated `ConfigMap` that materializes the scalar parts of the
contract for consumer repos.

That is enough to prove:

- the contract belongs in `platform-kro`
- consumers can bind to a stable API
- the app-facing schema can stay smaller than the raw control-plane shape

It is not meant to be the final consumer mechanism.

## Consumption Direction

The intended long-term flow is:

1. `platform-kro` publishes `AppDelivery`
2. a consumer repo such as `hetzner_playground` instantiates `AppDelivery`
3. the consumer repo renders or reconciles Tekton and later Kargo wiring
4. CI emits immutable build evidence
5. mutable delivery state moves to a dedicated delivery repo

## Deferred

The following are intentionally not in the executable contract yet:

- Kargo-specific promotion approvals
- environment-specific release-state paths
- delivery-repo Git mutation rules
- per-run evidence history
- detailed rollout verification contracts

Those remain implementation concerns until the live consumer integration
proves they belong in the stable API.
