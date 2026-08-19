# Proxy and ClusterSecretStore validation

## Overview

When `networkPolicyProvisioning` is `Managed` and proxy URLs are configured on
`ExternalSecretsConfig` or `ExternalSecretsManager`, the operator restricts operand
egress to the configured proxy via `eso-sys-allow-proxy-egress`. Operand pods
receive `HTTP_PROXY`, `HTTPS_PROXY`, and `NO_PROXY` environment variables.

All outbound provider traffic during `ClusterSecretStore` validation must honor
those variables. A provider that performs a direct TCP connectivity check while
other calls use the proxy will time out under managed egress policy.

## Akeyless provider (OCPBUGS-91653)

Prior to the fix, the Akeyless provider probed `akeylessGWApiURL` through the
proxy during client construction but `Validate()` used a direct network check.
That mismatch caused `ClusterSecretStore` reconciliation to time out in
proxy-only environments.

The fix routes validation through the same proxied HTTP authentication path used
during secret synchronization and preserves proxy settings when custom CA
bundles are configured.

Operand fix: https://github.com/sakshiep1/external-secrets/tree/OCPBUGS-91653

## Testing a patched operand image

Until a catalog release ships the fixed operand, patch the subscription to use a
custom image built from the fixed external-secrets branch:

```bash
oc -n <operator-namespace> patch subscription openshift-external-secrets-operator \
  --type='merge' -p '{"spec":{"config":{"env":[{"name":"RELATED_IMAGE_EXTERNAL_SECRETS","value":"<image>"}]}}}'
```

Verify:

1. Operand pods have proxy environment variables set.
2. `ClusterSecretStore` reports `Ready=True`.
3. Network capture shows provider traffic routed through the proxy, not direct egress.
