# AGENTS.md

## Repo Role

`platform-kro` is a user-facing platform API repository.

It owns:

- the executable KRO `ResourceGraphDefinition` manifests
- the user-facing API documentation for those platform contracts
- migration examples and validated limitations for consumers

It is not just an internal staging area for YAML experiments.

## Required Process For API Changes

For every meaningful API-surface change:

1. update the executable RGD
2. update the matching API README under `rgds/`
3. update the contract or design doc under `docs/`
4. update the relevant migration spike or example if the change affects usage
5. verify through the Flux consumer path in
   `/Users/francesco/repos/bewatec/hetzner_playground`
6. document any rejected or deferred capability explicitly in user-facing docs

## Documentation Standard

Prefer user-facing language first.

Every feature or limitation should be documented in terms of:

- what the API user can express
- what is intentionally deferred
- what was validated on a live cluster
- what examples or overlays demonstrate the behavior

Do not leave important behavior only in commit history or chat context.

## Validation Standard

Local validation is necessary but not sufficient.

Before calling an API change done, prefer this sequence:

- YAML parse succeeds
- `kubectl kustomize` succeeds for the repo and relevant examples
- the `platform-kro` repo is pushed
- Flux reconciles the shared RGDs in the consumer cluster
- the resulting API is exercised with a live example instance

If KRO or the cluster rejects a capability, revert or narrow the executable
schema and document the limitation clearly.

## Schema Direction

Prefer additive changes until live validation proves the field shape is stable.

When choosing between a richer but brittle schema and a smaller verified one:

- keep the executable contract conservative
- keep the user docs explicit about what is deferred
- do not over-claim compatibility with raw Kubernetes shapes when KRO cannot
  validate them in practice
