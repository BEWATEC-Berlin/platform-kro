# Dev Slice

This example models a thin first slice for local or development validation:

- one `Namespace` for `platform-demo`
- one `App` instance for `payment`
- one `AppDelivery` instance for `payment`
- references to separately managed `DatabaseCluster` and `CacheCluster`

The example is intentionally small. It is meant to validate the API boundary,
not to represent the full Astronomy Shop deployment.

The backing services are intentionally not instantiated in this directory. The
goal is to keep the `App` example one layer above the backing-service APIs.

The referenced `DatabaseCluster` and `CacheCluster` are expected to exist in the
same namespace as the `App` instance. The example now creates that namespace
explicitly so both the `App` and `AppDelivery` resources can be applied without
falling back to `default`.

The `AppDelivery` instance intentionally describes delivery intent only. It
does not create Tekton or Kargo resources directly; it materializes a generated
delivery-contract `ConfigMap` for consumer repos.
