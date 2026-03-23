# DatabaseCluster

## Scope

`DatabaseCluster` is the stable database API for the PoC.

Current implementation:

- PostgreSQL only
- backed by CloudNativePG

Planned direction:

- keep the contract generic enough to support a later MariaDB-backed
  implementation
- do not claim multi-engine support until a second backend exists

## Responsibilities

- create the database operator CR
- expose stable connection outputs
- give apps a consistent service and secret naming model

## Non-Goals

- schema migration orchestration
- application bootstrap SQL
- backup policy abstraction beyond a small amount of input selection
