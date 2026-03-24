# CacheCluster

## Scope

`CacheCluster` is the retained generic cache API for the current platform
migration wave.

Current implementation:

- plain StatefulSet-based fallback

Direction:

- keep the consumer-facing contract stable
- allow the internal implementation to switch to an operator later if one is
  accepted
- allow Dragonfly-backed implementations if they still fit the retained cache
  contract

## Responsibilities

- create the cache workload
- expose stable service outputs
- give apps one connection contract regardless of the backing implementation

## Non-Goals

- generalized queue semantics
- backend-branded cache APIs before the app-facing contract requires them
- HA cache topologies in the first pass
