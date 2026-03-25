# `cc-epg-service` `App` Example

This example translates `cc-epg-service` into the current `App` contract.

It is a paper migration example. It is intended to render cleanly and to show
how the service would be modeled with `App`, but it is not yet treated as a
live-validated migration.

## What This Example Shows

- base `App` modeling for `cc-epg-service`
- environment-specific overlay patching for integration
- config values moved from `configMapGenerator` into `spec.config.data`
- secret imports moved into `spec.config.envFromSecrets`
- `HTTPRoute` replacing the old ingress intent

## Important Limit

The source deployment mounts an `emptyDir` at `/opt/npm-cache`.

The current `App` contract does not support `emptyDir`, so this example does
not model that mount. This is the main remaining runtime gap for this service.

## Render Commands

```bash
kubectl kustomize examples/cc-epg-service-paper
kubectl kustomize examples/cc-epg-service-paper/base
kubectl kustomize examples/cc-epg-service-paper/overlays/integration
```

## Related Docs

- `docs/cc-epg-service-mapping.md`
- `docs/app-migration-cookbook.md`
