# platform-kro

`platform-kro` is the source-of-truth repo for the application-facing platform
APIs built with KRO for the platform PoC.

As of 2026-03-24, the repo direction is no longer to keep the API surface
intentionally minimal. The target is a fuller application platform API layer
that can absorb the repeated workload patterns currently expressed directly in
the `cc-*` application repositories.

The retained core API set is still:

- `DatabaseCluster`
- `CacheCluster`
- `App`

Those APIs should now evolve into migration-ready platform contracts rather
than remain thin wrappers around a single demo slice.

## Why This Repo Exists

This repo keeps KRO API work separate from consumer repos such as
`hetzner_playground`.

That separation is deliberate:

- platform API work can progress without clashing with cluster bootstrap work
- API docs and schema evolution live with the definitions
- consumer repos stay focused on installation, operators, and environments
- application migration can be planned against a stable API roadmap

## Repo Layout

- `rgds/`: ResourceGraphDefinition manifests and API-specific documentation
- `examples/`: example instances for a thin demo slice
- `docs/`: repo-level architecture and consumer guidance

Key repo-level documents:

- `docs/architecture.md`: target platform direction and API boundaries
- `docs/app-v2-contract.md`: concrete `App` v2 contract direction
- `docs/app-user-guide.md`: user-facing `App` usage guide
- `docs/app-config-and-secrets.md`: config, secret, and rollout model
- `docs/app-migration-cookbook.md`: step-by-step migration approach
- `docs/wave-planning.md`: Wave 1 exit criteria and Wave 2 entry criteria
- `docs/app-contract-cleanup.md`: transitional contract cleanup direction
- `docs/cc-epg-service-mapping.md`: first paper migration template
- `docs/cc-accounting-middleware-mapping.md`: second paper migration template
- `docs/data-service-boundaries.md`: retained backend API decisions
- `docs/cc-kustomize-gap-analysis.md`: gap analysis against the current
  `cc-*` kustomize repos

## User-Facing Docs

Start here when evaluating or consuming the platform APIs:

- `rgds/app/README.md`: user-facing `App` API guide
- `rgds/databasecluster/README.md`: user-facing `DatabaseCluster` API guide
- `rgds/cachecluster/README.md`: user-facing `CacheCluster` API guide
- `docs/app-user-guide.md`: how to model an HTTP app with `App`
- `docs/app-config-and-secrets.md`: how config and secret projection works
- `docs/app-migration-cookbook.md`: how to migrate from existing Kustomize
- `docs/cc-epg-service-mapping.md`: first paper migration example
- `docs/cc-accounting-middleware-mapping.md`: second paper migration example
- `docs/app-v2-contract.md`: target `App` contract and validated limitations
- `docs/wave-planning.md`: migration wave criteria and current status
- `docs/cc-kustomize-gap-analysis.md`: migration-gap view against the current
  `cc-*` estate
- `examples/otmicro-currency-spike/README.md`: concrete migration spike and
  overlay pattern
- `examples/restricted-nginx-spike/README.md`: restricted-policy `App` example
- `examples/cc-epg-service-paper/README.md`: paper `App` example for `cc-epg-service`

## Change Process

This repo is a user-facing platform API repository, not just an internal YAML
scratchpad.

Every meaningful API change should land with:

- the executable RGD update
- user-facing README updates for the affected API
- contract and gap-analysis updates in `docs/`
- an example or migration spike update when the change affects real usage
- live verification through the Flux consumer path in `hetzner_playground`
- an explicit note when a capability was attempted but deferred because KRO or
  cluster validation rejected it

## Current State

This repo is still an initial scaffold.

- the RGDs are first-pass drafts
- the current contracts do not yet cover most real workload patterns from the
  `cc-*` repositories
- consumer integration is expected to happen from repos such as
  `hetzner_playground`, but the API surface must expand first

The next milestone is to turn the retained APIs into a platform contract that
can absorb a meaningful subset of the existing application estate without
relying on `Ingress`, `IngressRoute`, or `IngressRouteTCP` in the target
application API.

The intended layering is:

- `DatabaseCluster` and `CacheCluster` provide backing-service contracts
- `App` becomes the main HTTP application-facing abstraction
- Gateway API `HTTPRoute` is the standard exposure model for HTTP applications
- non-HTTP exposure should be handled by dedicated future APIs rather than
  reintroducing ingress-specific CRDs
