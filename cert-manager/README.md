# cert-manager

Issues and auto-rotates in-cluster TLS certificates. Here it backs the **OpenTelemetry operator's
admission webhook**: the operator requests a cert through cert-manager (a self-signer `Issuer` + a
`Certificate`), and cert-manager's CA-injector keeps the webhook's `caBundle` in sync with the served
cert — automatically. This replaces the operator's self-signed *auto-generate* certs, which drifted
out of sync under GitOps (the serving cert and the webhook `caBundle` stopped matching, so the
`Instrumentation` webhook rejected everything with an "unknown authority" TLS error).

Managed by Argo CD via [`app/chart.yaml`](./app/chart.yaml): the `cert-manager` chart (`v1.21.3`,
CRDs included) into the `cert-manager` namespace. It runs at **sync-wave -5** so it — and its own
webhook — are ready before any tool that requests a cert. Bump `targetRevision` + push to upgrade.

## Using it from another tool
Point an operator's webhook at cert-manager instead of self-signed certs, e.g. the OTel operator
([`../observability/otel-operator-values.yaml`](../observability/otel-operator-values.yaml)):

```yaml
admissionWebhooks:
  certManager:
    enabled: true
  autoGenerateCert:
    enabled: false
```
