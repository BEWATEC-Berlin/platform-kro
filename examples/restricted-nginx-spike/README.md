# `restricted-nginx` Spike

This example verifies the narrow `runtime.restrictedSecurity` path in `App`.

It uses a non-root-friendly image and enables the restricted security preset so
that the generated Deployment satisfies the development cluster's restricted
Kyverno policy.

## What This Example Verifies

- `App` instance creation under `platform.connectedcare.io/v1alpha1`
- generated Deployment with a restricted-compliant container `securityContext`
- generated HTTP readiness and liveness probes against the named `http` port
- generated `PodDisruptionBudget` for the `App` pods
- generated `ServiceMonitor` for the same service labels and `http` port
- merged env-from projection from the generated app config map, one referenced config map, and one referenced secret
- deployment availability in the development cluster without a policy exception

## Render Command

```bash
kubectl kustomize examples/restricted-nginx-spike
```
