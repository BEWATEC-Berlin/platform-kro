# Wave Planning

## Purpose

This document defines the migration waves for `platform-kro` in operational
terms.

The goal is to stop treating wave boundaries as vague intent.

## Wave 1 Goal

Wave 1 is successful when `App` can absorb a meaningful subset of current
HTTP service repos without large parallel deployment manifests.

Wave 1 is not just “more fields were added.”

## Wave 1 Candidate Repos

- `cc-epg-service`
- `cc-kis-federated-service`
- `cc-accounting-middleware`

## Wave 1 Exit Criteria

All of the following should be true:

1. at least two real Wave 1 repos are mapped cleanly to `App`
2. at least one of those mappings is exercised live in the development cluster
3. normal HTTP route, config, secret, monitoring, disruption-budget, and
   resource requirements can be expressed without raw deployment manifests
4. the current storage path is proven sufficient for at least one real
   deployment-shaped PVC case
5. the remaining gaps are clearly outside ordinary HTTP app modeling

## Wave 2 Entry Criteria

Wave 2 starts when Wave 1 exit criteria are met and the remaining high-value
repos are mainly blocked by backing-service and operational contracts.

Wave 2 should focus on:

- PostgreSQL contract refinement
- backup and restore boundaries
- recurring task story
- stronger cache contract decisions

## Wave 2 Candidate Repos

- `cc-backend-api`
- `cc-infotainment-pin-service`
- `cc-patient-request-service`
- `cc-purchase-manager`

## Wave 3 Candidate Repos

These should not shape `App` directly:

- `cc-e2e-identity-management`
- `cc-limesurvey-cfg`
- `cc-mirth-connect-cfg`
- `cc-shop`
- `cc-third-party-token-service`

These are more likely to need:

- `ScheduledTask`
- `StatefulApp`
- composition above `App`
- non-HTTP route APIs
- a separate MariaDB API

## Current Status As Of 2026-03-25

Wave 1 `App` capabilities already validated live:

- explicit env and env-from refs
- restricted security preset
- readiness and liveness probes
- `ServiceMonitor`
- `PodDisruptionBudget`
- single-PVC storage with explicit class and mode
- `HTTPRoute` support in the contract

Wave 1 still needs:

- at least one real `cc-*` service migration spike through GitOps
- at least one more paper mapping after `cc-epg-service`
- explicit documentation of what remains outside `App` for those repos

## Recommended Next Order

1. paper-map `cc-epg-service`
2. paper-map `cc-accounting-middleware`
3. choose one of them for the first live migration spike
4. only then begin Wave 2 design pressure from PostgreSQL-heavy repos
