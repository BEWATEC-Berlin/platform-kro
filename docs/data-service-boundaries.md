# Data Service Boundaries

## Purpose

This document records the current boundary decisions for `DatabaseCluster` and
`CacheCluster` so `App` expansion does not get blocked by unresolved backend
questions.

These are repo design decisions as of 2026-03-24. They are intended to guide
migration work, not to claim that the final platform surface is frozen.

## Decision 1: `DatabaseCluster` Stays PostgreSQL-Focused In The Current Wave

`DatabaseCluster` remains the retained PostgreSQL contract for the first major
migration wave.

Rationale:

- the current manifest already maps directly to CloudNativePG
- the current `cc-*` repos using PostgreSQL are enough to drive immediate work
- trying to unify PostgreSQL and MariaDB too early would hide materially
  different operator contracts behind vague fields

Consequence:

- `DatabaseCluster` can grow additive operational knobs for PostgreSQL
- it should keep the current status contract stable
- it should not claim cross-engine parity yet

## Decision 2: MariaDB Should Become A Separate Future API

MariaDB should not be folded into `DatabaseCluster` in the current wave.

Rationale:

- the MariaDB operator resources have different semantics around users,
  replication, backups, and tuning
- the only current MariaDB-heavy consumer in the scanned repo set is
  `cc-limesurvey-cfg`
- that repo is already outside the first `App` migration wave

Consequence:

- if MariaDB becomes a retained platform API, it should be introduced as a
  separate contract such as `MariaDBCluster`
- `DatabaseCluster` should not be overloaded with engine switches until there
  is a proven need for a higher-level cross-engine abstraction

## Decision 3: `CacheCluster` Stays Generic And May Absorb Dragonfly-Like Backends

`CacheCluster` should remain the retained generic cache contract.

Rationale:

- the consumer-facing need is usually a stable cache endpoint and secret model,
  not a backend-specific API
- Dragonfly may still fit the retained cache abstraction if the exposed app
  contract remains single-endpoint and cache-oriented
- keeping one cache API is simpler than introducing backend-branded cache APIs
  too early

Consequence:

- `CacheCluster` can grow additive operational fields such as resources,
  persistence, auth, and replication knobs if they remain backend-agnostic
- if a backend requires clearly different semantics, that is the point to split
  the API rather than now

## Decision 4: Backup And Restore Stay Operator-Native In The Current Wave

Backup and restore behavior should remain mostly operator-native for now.

Rationale:

- the current repo set already shows materially different backup objects and
  restore workflows
- forcing them into a generic platform abstraction now would add complexity
  faster than it would remove it

Consequence:

- platform APIs may later accept a small amount of policy input
- detailed backup schedules, restore workflows, and one-off recovery procedures
  should remain outside the first platform API wave

## Decision 5: Status Contracts Must Stay Stable Even If Implementations Change

The application-facing status surface should stay stable even if the internal
backing implementation changes.

For `DatabaseCluster`, the important stable outputs are:

- read-write service name
- read-only service name where available
- application secret name
- database name

For `CacheCluster`, the important stable outputs are:

- service name
- endpoint
- port

These status fields should remain the main compatibility layer for `App`.
