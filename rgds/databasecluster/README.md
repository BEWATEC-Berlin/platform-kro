# DatabaseCluster

## Scope

`DatabaseCluster` is the retained PostgreSQL database API for the current
platform migration wave.

Current implementation:

- PostgreSQL only
- backed by CloudNativePG

Direction:

- keep the current status contract stable
- grow additive PostgreSQL operational fields as needed
- do not fold MariaDB into this API in the current wave
- introduce a separate retained MariaDB API later if the platform needs one

## Responsibilities

- create the PostgreSQL operator CR
- expose stable connection outputs
- give apps a consistent service and secret naming model

## Non-Goals

- schema migration orchestration
- application bootstrap SQL
- generic cross-engine abstractions before they are proven
- backup policy abstraction beyond a small amount of input selection
