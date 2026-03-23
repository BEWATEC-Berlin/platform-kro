# platform-kro

`platform-kro` is the source-of-truth repo for the small KRO API surface used
by the platform PoC.

The current scope is intentionally narrow:

- `DatabaseCluster`
- `CacheCluster`
- `App`

Everything else should stay on normal Kubernetes resources, Helm releases, or
operator-native CRs until these three APIs prove their value.

## Why This Repo Exists

This repo keeps KRO API work separate from consumer repos such as
`hetzner_playground`.

That separation is deliberate:

- platform API work can progress without clashing with cluster bootstrap work
- API docs and schema evolution live with the definitions
- consumer repos stay focused on installation, operators, and environments

## Repo Layout

- `rgds/`: ResourceGraphDefinition manifests and API-specific documentation
- `examples/`: example instances for a thin demo slice
- `docs/`: repo-level architecture and consumer guidance

## Current State

This is an initial scaffold.

- the RGDs are first-pass drafts
- the contracts are intentionally simple
- consumer integration is expected to happen from repos such as
  `hetzner_playground`

The first milestone is to make the retained APIs concrete enough to consume in a
single thin slice, not to solve every platform concern immediately.

The intended layering is:

- `DatabaseCluster` and `CacheCluster` provide backing-service contracts
- `App` consumes those contracts as the higher application-facing abstraction
