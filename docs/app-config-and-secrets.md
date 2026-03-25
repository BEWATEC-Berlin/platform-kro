# `App` Config And Secrets

## Purpose

This document defines the user-facing config and secret model for `App`.

The goal is to give platform consumers a stable way to express application
configuration without exposing raw Kustomize generator behavior as the contract.

## Supported Config Inputs

### `config.data`

Use `config.data` for plain key-value application configuration that should be
written into the generated app `ConfigMap`.

Example:

```yaml
spec:
  config:
    data:
      LOG_LEVEL: info
      SERVER_PORT: "8080"
```

### `config.revision`

Use `config.revision` as the explicit rollout token.

The generated pod template is stamped with
`platform.connectedcare.io/config-revision`, so changing this field forces a
rollout even when config object names remain stable.

Example:

```yaml
spec:
  config:
    revision: prod-2026-03-25-1
```

### `config.env`

Use `config.env` for explicit container env entries.

Supported forms:

- literal `value`
- `valueFrom.fieldRef`
- `valueFrom.resourceFieldRef`
- `valueFrom.secretKeyRef`
- `valueFrom.configMapKeyRef`

Example:

```yaml
spec:
  config:
    env:
      - name: OTEL_SERVICE_NAME
        valueFrom:
          fieldRef:
            apiVersion: v1
            fieldPath: metadata.labels['app.kubernetes.io/component']
      - name: SERVER_PORT
        value: "8080"
```

### `config.envFromConfigMaps`

Use `config.envFromConfigMaps` to append explicit config-map refs to the
generated `envFrom` list.

### `config.envFromSecrets`

Use `config.envFromSecrets` to append explicit secret refs to the generated
`envFrom` list.

## Generated Order

The generated container `envFrom` order is:

1. generated app `ConfigMap`
2. entries from `config.envFromConfigMaps`
3. entries from `config.envFromSecrets`

That order is intentional and should be treated as the current contract.

## What To Put Where

Use `config.data` for:

- plain app config
- environment-specific literals
- flags and endpoints that are not secrets

Use `config.env` for:

- explicit variable names the app expects
- `fieldRef` wiring
- single-key secret or config-map projections

Use `config.envFromConfigMaps` for:

- existing config maps already managed outside `App`
- migration cases where the source config object already exists

Use `config.envFromSecrets` for:

- existing opaque secrets already managed outside `App`
- migration cases where the whole secret should be imported

Use `externalSecret` for:

- generating one secret object from an external secret store as part of the
  `App` graph

## Recommended Secret Pattern

Preferred first-pass pattern:

1. use `externalSecret` when the app should own generation of one target secret
2. reference the resulting secret with `config.envFromSecrets` or
   `config.env[].valueFrom.secretKeyRef`
3. keep secret-store ownership outside the app contract

This keeps the app-facing API explicit while avoiding raw secret values in the
manifest.

## Current Limitations

Not currently supported:

- generic raw `config.envFrom`
- `envFrom.prefix`
- raw Kubernetes `EnvFromSource` passthrough
- multiple generated config maps
- checksum or hash generation from config content inside KRO

These were deferred because the live KRO validation path in this environment
did not accept the broader union shape reliably.

## Migration From `configMapGenerator`

Old pattern:

- `configMapGenerator` literals in Kustomize
- stable name or hash-based rollout behavior

Current `App` pattern:

- move literals into `config.data`
- move explicit env vars into `config.env`
- move external refs into `config.envFromConfigMaps` or `config.envFromSecrets`
- use `config.revision` when rollout must be forced

## Paper Example: `cc-epg-service`

For `cc-epg-service`, the current config split is:

Plain config:

- `LOG_LEVEL`
- `NODE_ENV`
- `SERVER_PORT`
- `AWS_S3_REGION`
- `AWS_S3_BUCKET_NAME`
- `CRON_JOB_CACHE_UPDATE`
- `AUTH_AUDIENCE`
- `AUTH_ISSUER`
- `AUTH_JWKS_URI`
- `AUTH_INHOSPITAL_API_GATEWAY`

Secret-backed values:

- `AWS_S3_ACCESS_KEY_ID`
- `AWS_S3_SECRET_ACCESS_KEY`

New Relic:

- whole-secret import remains a good candidate for `config.envFromSecrets`

## Related Docs

- `docs/app-user-guide.md`
- `docs/app-migration-cookbook.md`
- `docs/cc-epg-service-mapping.md`
