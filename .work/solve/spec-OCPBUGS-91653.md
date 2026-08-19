# Implementation Plan: OCPBUGS-91653

## Problem
ClusterSecretStore reconciliation times out in proxy-restricted clusters because the Akeyless provider validates connectivity with a direct TCP check (`NetworkValidate`) while `akeylessGWApiURL` probing uses HTTP through the proxy. Managed egress network policy (`eso-sys-allow-proxy-egress`) blocks the direct validation path.

## Root Cause
- Operand: `Validate()` in `providers/v1/akeyless` used `esutils.NetworkValidate` (direct connection)
- Operand: `getAkeylessHTTPClient` with custom CA replaced transport without preserving `http.DefaultTransport.Proxy`

## Solution
### Operand (external-secrets)
1. Change `Validate()` to call `TokenFromSecretRef` (same proxied HTTP auth path as sync)
2. Clone `http.DefaultTransport` when setting custom TLS for CA bundles
3. Add unit tests for proxy preservation and validation path

### Operator (external-secrets-operator)
- No controller changes required; proxy injection and network policy are correct
- Ship fix by deploying an operand image built from the fixed external-secrets commit
- Customers can test via `RELATED_IMAGE_EXTERNAL_SECRETS` subscription patch until a catalog release includes the fixed operand

## Verification
- `go test ./providers/v1/akeyless/...` in external-secrets fork
- `make verify` / `make test` in operator repo (no regressions)
- Live: ClusterSecretStore Ready + tcpdump shows all traffic via proxy

## Operand fix branch
- Repo: https://github.com/sakshiep1/external-secrets
- Branch: `OCPBUGS-91653`
