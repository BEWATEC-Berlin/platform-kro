# CacheCluster

## Scope

`CacheCluster` is the stable cache API for the PoC.

Current implementation:

- plain StatefulSet-based fallback

Planned direction:

- keep the contract stable
- allow the internal implementation to switch to an operator later if one is
  accepted

## Responsibilities

- create the cache workload
- expose stable service outputs
- give apps one connection contract regardless of the backing implementation

## Non-Goals

- generalized queue semantics
- HA cache topologies in the first pass
